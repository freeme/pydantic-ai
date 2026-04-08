# 十、评估框架 pydantic_evals

> **核心文件**：`pydantic_evals/dataset.py`、`evaluators/`、`reporting/`、`generation.py`、`lifecycle.py`、`online.py`

---

## 10.1 设计目标

`pydantic_evals` 是一个独立的评估工具包，用于对任意"随机函数"（如 LLM 调用）进行测试、评分与报告。

```
pydantic_evals 核心流程
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

   Dataset (YAML/JSON)          Evaluator (评分规则)
        │                              │
        ▼                              │
   Case × N  ──── task(inputs) ────► output
        │                              │
        │         EvaluatorContext ◄───┘
        │              │
        ▼              ▼
   SpanTree      EvaluatorOutput (bool / float / str)
        │              │
        └──────┬────────┘
               ▼
         EvaluationReport (Rich 表格 / JSON)
```

---

## 10.2 核心数据模型

### Case — 单个测试用例

```python
@dataclass
class Case(Generic[InputsT, OutputT, MetadataT]):
    name: str | None            # 用例名称
    inputs: InputsT             # 传给 task 的输入
    metadata: MetadataT | None  # 附加元数据（供 Evaluator 使用）
    expected_output: OutputT | None   # 预期输出（可选）
    evaluators: list[Evaluator]       # 用例级别的 Evaluator（追加）
```

### Dataset — 用例集合

```python
class Dataset(BaseModel, Generic[InputsT, OutputT, MetadataT]):
    name: str | None
    cases: list[Case[InputsT, OutputT, MetadataT]]
    evaluators: list[Evaluator]         # 数据集级别 Evaluator（对所有用例生效）
    report_evaluators: list[ReportEvaluator]  # 聚合级别 Evaluator（对整个报告生效）
```

---

## 10.3 Evaluator 体系

### 评分结果类型

```python
EvaluationScalar = bool | int | float | str
# bool → 断言（Pass/Fail）
# int/float → 分数
# str → 标签/分类

EvaluatorOutput = (
    EvaluationScalar
    | EvaluationReason           # 带原因的标量
    | Mapping[str, EvaluationScalar | EvaluationReason]  # 多维度评分
)
```

### EvaluatorContext — 评分上下文

```python
@dataclass
class EvaluatorContext(Generic[InputsT, OutputT, MetadataT]):
    name: str | None            # 用例名称
    inputs: InputsT             # 任务输入
    metadata: MetadataT | None  # 元数据
    expected_output: OutputT | None  # 预期输出
    output: OutputT             # 实际输出 ← 核心
    duration: float             # 执行时间（秒）
    span_tree: SpanTree         # OTel Span 树（追踪数据）
    attributes: dict[str, Any]  # 任务执行期间设置的属性
    metrics: dict[str, int | float]  # 任务执行期间设置的指标
```

### 内置 Evaluator（common.py）

| 类名 | 用途 |
|------|------|
| `Equals` | 精确匹配某个值 |
| `EqualsExpected` | 精确匹配 `expected_output` |
| `Contains` | 字符串包含检查 |
| `IsInstance` | 类型检查 |
| `MaxDuration` | 执行时间限制 |
| `LLMJudge` | LLM 作为评判者 |
| `HasMatchingSpan` | 检查 Span 树中是否有匹配的 Span |

### 自定义 Evaluator

```python
from dataclasses import dataclass
from pydantic_evals.evaluators import Evaluator, EvaluatorContext

@dataclass
class ExactMatch(Evaluator):
    def evaluate(self, ctx: EvaluatorContext) -> bool:
        return ctx.output == ctx.expected_output

@dataclass
class SimilarityScore(Evaluator):
    threshold: float = 0.8
    def evaluate(self, ctx: EvaluatorContext) -> EvaluationReason:
        score = compute_similarity(ctx.output, ctx.expected_output)
        return EvaluationReason(
            value=score,
            reason=f"Similarity: {score:.2f}"
        )
```

---

## 10.4 LLM as a Judge

**文件**：`evaluators/llm_as_a_judge.py`

```python
class GradingOutput(BaseModel):
    reason: str
    pass_: bool      # 序列化为 'pass'
    score: float

# 四个便捷函数
judge_output(output, rubric, model)               # 仅评判输出
judge_output_expected(output, expected, rubric)    # 评判输出 vs 预期
judge_input_output(inputs, output, rubric)         # 评判输入 + 输出
judge_input_output_expected(inputs, output, expected, rubric)  # 完整评判
```

内部使用 `pydantic-ai` Agent：系统提示包含评判规则示例，输出类型为 `GradingOutput`，结果包含：
- `pass_`：是否通过
- `score`：0.0 ~ 1.0 的分数
- `reason`：判断依据

---

## 10.5 Dataset 的 evaluate() 流程

```python
async def evaluate(
    task: Callable[[InputsT], Awaitable[OutputT]],
    *,
    concurrency: int = 10,
    lifecycle: type[CaseLifecycle] | None = None,
) -> EvaluationReport:
```

每个 Case 的执行流程：

```
1. CaseLifecycle.setup()
         │
2. OTel context_subtree 开始捕获 Span
         │
3. task(inputs)  ← 执行实际任务
         │
4. CaseLifecycle.prepare_context(EvaluatorContext)
         │
5. 运行 Evaluator 列表（用例级 + 数据集级）
         │
6. CaseLifecycle.teardown(ReportCase | ReportCaseFailure)
         │
7. 聚合到 EvaluationReport
```

- 并发：使用 `anyio` TaskGroup，`concurrency` 参数控制同时运行的用例数
- 异常隔离：任意用例失败不影响其他用例，失败记录在 `ReportCaseFailure` 中
- Rich 进度条：通过 `rich.progress` 实时显示评估进度

---

## 10.6 CaseLifecycle — 生命周期钩子

```python
class CaseLifecycle(Generic[InputsT, OutputT, MetadataT]):
    """为每个用例提供生命周期钩子。每个用例创建一个新实例。"""

    async def setup(self) -> None:
        """任务执行前调用。可用于创建数据库、启动服务等。"""

    async def prepare_context(
        self, ctx: EvaluatorContext
    ) -> EvaluatorContext:
        """任务执行后、Evaluator 运行前调用。
        可以在此丰富 metrics 和 attributes。"""
        return ctx

    async def teardown(
        self, result: ReportCase | ReportCaseFailure
    ) -> None:
        """Evaluator 运行后调用。接收完整结果，可用于清理资源。"""
```

---

## 10.7 自定义评估属性与指标

在 task 执行期间，可以通过上下文变量注入评估数据：

```python
from pydantic_evals import set_eval_attribute, increment_eval_metric

async def my_task(inputs: str) -> str:
    result = await agent.run(inputs)

    # 记录属性（字符串/值）
    set_eval_attribute('model_used', 'gpt-5.2')
    set_eval_attribute('prompt_tokens', result.usage().input_tokens)

    # 累计指标（数值）
    increment_eval_metric('api_calls', 1)
    increment_eval_metric('cost', result.cost())

    return result.output
```

这些数据通过 `ContextVar` 传递，在 `EvaluatorContext.attributes` 和 `metrics` 中可访问。

---

## 10.8 SpanTree — OTel Span 分析

```python
class SpanTree:
    """OTel Span 的树形结构，用于评估中的追踪分析。"""

    def find(self, query: SpanQuery) -> list[SpanNode]:
        """查找匹配条件的 Span。"""

    def filter(self, query: SpanQuery) -> SpanTree:
        """过滤出匹配的子树。"""
```

`HasMatchingSpan` Evaluator 就是基于 SpanTree 的查询：

```python
@dataclass
class HasMatchingSpan(Evaluator):
    query: SpanQuery
    def evaluate(self, ctx: EvaluatorContext) -> bool:
        return len(ctx.span_tree.find(self.query)) > 0
```

---

## 10.9 EvaluationReport — 报告系统

**文件**：`reporting/`

```python
@dataclass
class EvaluationReport(Generic[InputsT, OutputT, MetadataT]):
    """评估报告，包含所有用例的结果。"""
    name: str
    cases: list[ReportCase | ReportCaseFailure]

    def print(self, *, verbose: bool = False) -> None:
        """使用 Rich 库打印表格报告。"""

    def to_json(self) -> str:
        """序列化为 JSON（用于 CI 断言或存档）。"""
```

报告特性：
- **Rich 表格**：彩色输出、对齐列、分数渲染
- **聚合统计**：Pass Rate、平均分数、时间分布
- **分析图表**：ConfusionMatrix、PrecisionRecall、LinePlot
- **JSON 序列化**：支持 CI 流程中的存档与比较

---

## 10.10 generate_dataset() — AI 生成测试用例

**文件**：`generation.py`

```python
async def generate_dataset(
    *,
    dataset_type: type[Dataset[InputsT, OutputT, MetadataT]],
    model: models.Model | str = 'openai:gpt-5.2',
    n_examples: int = 3,
    extra_instructions: str | None = None,
    path: Path | str | None = None,
) -> Dataset[InputsT, OutputT, MetadataT]:
```

使用 LLM 自动生成测试数据集：
1. 从 `Dataset` 的泛型参数提取 JSON Schema
2. 向 LLM 发送 Schema + 数量要求
3. 解析 LLM 输出为结构化 Dataset 对象
4. 可选保存到 YAML 文件

---

## 10.11 Online Evaluation — 生产环境在线评估

**文件**：`online.py`

```python
from pydantic_evals.online import evaluate

@evaluate(IsNonEmpty(), MaxDuration(timedelta(seconds=2)))
async def my_production_function(x: int) -> int:
    return x
```

- **装饰器模式**：不修改函数签名，在后台异步运行 Evaluator
- **随机采样**：可配置采样率，避免对所有请求都运行评估
- **结果上报**：评估结果通过 OTel 记录，可在监控平台查看
- **与 Dataset 共享 Evaluator**：同一套 Evaluator 可用于测试（Dataset.evaluate）和生产（online.evaluate）

---

## 10.12 YAML/JSON Dataset 文件格式

```yaml
# yaml-language-server: $schema=./test_cases_schema.json
name: uppercase_tests
cases:
  - name: simple_greeting
    inputs: "hello world"
    expected_output: "HELLO WORLD"
    evaluators:
      - EqualsExpected
  - name: already_upper
    inputs: "HELLO"
    expected_output: "HELLO"

evaluators:
  - MaxDuration:
      max_duration: 1.0
  - LLMJudge:
      rubric: "Output is a proper uppercase transformation"
      model: openai:gpt-5.2
```

Dataset 文件与 JSON Schema 联动（通过 `$schema` 字段），支持 IDE 自动补全。

---

## 总结

```
pydantic_evals 设计哲学
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 任务无关性：
   evaluate(task_fn) 接受任意 async 函数
   不仅限于 pydantic-ai Agent，可评估任何随机函数

2. 评分器即代码：
   Evaluator 是普通 Python 类，可以复用
   同一套 Evaluator 可用于 Dataset 测试和 Online 监控

3. OTel 原生集成：
   SpanTree 将追踪数据作为评估维度
   可以评判"是否使用了正确的工具"等追踪级别问题

4. AI 生成测试数据：
   generate_dataset() 利用 LLM 从类型定义生成测试用例
   降低了测试数据创建的人力成本

5. 报告即分析：
   EvaluationReport 不只是显示结果
   内置 ConfusionMatrix、PR Curve 等分析工具
```
