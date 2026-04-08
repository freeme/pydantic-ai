# 十三、CLI 与开发者工具

> **核心文件**：`_cli/__init__.py`、`_cli/web.py`、`models/test.py`、`models/function.py`

---

## 13.1 CLI 命令入口

pydantic-ai 提供 `clai` 命令行工具（通过 `pip install "pydantic-ai[cli]"` 安装）。

```
clai [选项] [提示词]        # 交互式对话或单次提问
clai web [选项]             # 启动 Web UI 界面
```

---

## 13.2 clai — 命令行聊天

**文件**：`_cli/__init__.py`

### 主要参数

| 参数 | 说明 |
|------|------|
| `prompt` | 直接提问（单次模式），省略则进入交互式模式 |
| `--model` / `-m` | 模型名称（如 `openai:gpt-5.2`），支持 Tab 补全 |
| `--agent` / `-a` | 加载指定 Agent（格式：`module:variable` 或 YAML/JSON 文件路径）|
| `--tool` / `-t` | 启用内置工具（如 `web_search`、`code_execution`）|
| `--no-stream` | 禁用流式输出 |
| `--version` | 显示版本信息 |

### 运行模式

**交互式模式**（无 `prompt` 参数）：
- 使用 `prompt-toolkit` 提供历史记录、自动补全
- 历史记录保存至 `~/.pydantic-ai/prompt-history.txt`
- 使用 `rich` 渲染 Markdown 输出

**单次模式**（有 `prompt` 参数）：
- 直接执行，输出到 stdout
- 适合脚本集成

### 系统提示词

CLI 模式下的系统提示词：
```python
@cli_agent.system_prompt
def cli_system_prompt() -> str:
    return f"""\
Help the user by responding to their request, the output should be concise
and always written in markdown.
The current date and time is {datetime.now()} {tzname}.
The user is running {sys.platform}."""
```

---

## 13.3 clai web — Web UI 界面

**文件**：`_cli/web.py`、`ui/_web/`

```bash
clai web                                    # 启动默认 Web UI
clai web --agent my_app:agent               # 加载指定 Agent
clai web --model openai:gpt-5.2 --port 7932 # 指定模型和端口
clai web --tool web_search                  # 启用内置工具
```

Web UI 特性：
- 默认运行在 `http://127.0.0.1:7932`
- 支持实时流式对话
- 可选择/切换模型
- 通过 `html_source` 参数自定义 UI HTML（默认从 CDN 加载）

### load_agent() — Agent 加载

```python
def load_agent(agent_path: str) -> Agent | None:
    """支持两种格式：
    - 'module:variable' → 动态导入（uvicorn 风格）
    - 'agent.yaml' / 'agent.json' → 从文件加载
    """
    path = Path(agent_path)
    if path.suffix in ('.yaml', '.yml', '.json'):
        return Agent.from_file(path)  # 文件格式
    else:
        return _import_string_adapter.validate_python(agent_path)  # 模块导入
```

---

## 13.4 TestModel — 测试专用模型

**文件**：`models/test.py`

`TestModel` 是专门为单元测试设计的确定性模型，**不发起实际 LLM 调用**。

```python
@dataclass
class TestModel(Model):
    __test__ = False  # 防止 pytest 误识别为测试类

    call_tools: list[str] | Literal['all'] = 'all'
    # 'all': 调用所有可用工具
    # list: 只调用指定工具

    custom_output_text: str | None = None
    # 直接返回此文本作为最终输出

    custom_output_args: Any | None = None
    # 用于输出工具的参数

    seed: int = 0
    # 随机种子（用于生成占位数据）

    last_model_request_parameters: ModelRequestParameters | None = None
    # 最近一次请求的工具定义（可用于断言验证）
```

### 使用场景

```python
from pydantic_ai.models.test import TestModel

# 1. 基本测试：调用所有工具，返回占位文本
agent = Agent(TestModel())

# 2. 控制输出内容
agent = Agent(TestModel(custom_output_text='The answer is 42'))

# 3. 控制工具调用
agent = Agent(TestModel(call_tools=['get_weather']))

# 4. 验证工具定义
model = TestModel()
result = await agent.run('test', model=model)
tools = model.last_model_request_parameters.function_tools
assert any(t.name == 'my_tool' for t in tools)
```

---

## 13.5 FunctionModel — 函数驱动模型

**文件**：`models/function.py`

`FunctionModel` 允许用本地 Python 函数控制模型行为，适合：
- 复杂测试逻辑
- 自定义响应生成
- 模拟特定错误场景

```python
from pydantic_ai.models.function import FunctionModel
from pydantic_ai.messages import ModelResponse, TextPart

def my_function(messages, info: AgentInfo) -> ModelResponse:
    """同步函数形式。"""
    last_message = messages[-1]
    return ModelResponse(parts=[TextPart(content='Mocked response')])

async def my_async_function(messages, info: AgentInfo) -> ModelResponse:
    """异步函数形式。"""
    ...

# 流式函数
async def my_stream_function(messages, info: AgentInfo):
    yield TextPart(content='Part 1')
    yield TextPart(content='Part 2')

model = FunctionModel(my_function)                    # 同步
model = FunctionModel(function=my_async_function)     # 异步
model = FunctionModel(stream_function=my_stream_function)  # 流式
```

### AgentInfo 参数

```python
@dataclass
class AgentInfo:
    """提供给函数模型的运行时信息。"""
    function_tools: tuple[ToolDefinition, ...]   # 可用工具定义
    allow_text_result: bool                       # 是否允许文本输出
    output_tools: tuple[ToolDefinition, ...]      # 输出工具定义
    model_settings: ModelSettings | None          # 模型设置
```

---

## 13.6 Tab 补全支持

CLI 通过 `argcomplete` 库提供 Tab 补全：

```python
import argcomplete
argcomplete.autocomplete(parser)  # 在解析前启用自动补全
```

模型名称补全基于 `KnownModelName` 的 Literal 类型，过滤掉无提供商前缀的别名：

```python
qualified_model_names = [n for n in get_literal_values(KnownModelName.__value__) if ':' in n]
```

---

## 13.7 Rich 输出美化

CLI 使用 `rich` 库渲染 Markdown：

```python
# 自定义代码块：避免背景色影响复制粘贴
class SimpleCodeBlock(CodeBlock):
    def __rich_console__(self, console, options) -> RenderResult:
        yield Text(self.lexer_name, style='dim')
        yield Syntax(code, self.lexer_name, theme=self.theme, background_color='default')
        yield Text(f'/{self.lexer_name}', style='dim')

# 自定义标题：左对齐、显示 # 标记
class LeftHeading(Heading):
    def __rich_console__(self, console, options) -> RenderResult:
        yield Text(f'{"#" * int(self.tag[1:])} {self.text.plain}', style=Style(bold=True))
```

---

## 13.8 PYDANTIC_AI_HOME 配置目录

```python
PYDANTIC_AI_HOME = Path.home() / '.pydantic-ai'
PROMPT_HISTORY_FILENAME = 'prompt-history.txt'
```

- 历史记录存储位置：`~/.pydantic-ai/prompt-history.txt`
- 使用 `prompt-toolkit` 的 `FileHistory` 跨会话保留历史

---

## 13.9 内置工具在 CLI 中的支持

```python
SUPPORTED_CLI_TOOL_IDS = sorted(
    bint.kind for bint in SUPPORTED_BUILTIN_TOOLS
    if bint not in BUILTIN_TOOLS_REQUIRING_CONFIG
)
```

不需要额外配置的内置工具（可在 CLI 中直接启用）：
- `web_search`
- `code_execution`
- `image_generation`

需要配置的工具（不在 CLI 的自动启用范围，如需要 API Key 或连接设置）则通过警告提示用户。

---

## 总结

```
CLI 与开发者工具体系
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. clai 命令（交互式 + 单次模式）：
   prompt-toolkit 提供历史记录和自动补全
   rich 渲染 Markdown，定制代码块和标题样式

2. clai web（图形化 Web UI）：
   一键启动对话 Web 界面，无需前端部署
   可加载自定义 Agent 和配置模型/工具

3. TestModel（确定性测试模型）：
   不发起实际 API 调用，完全可预测
   call_tools 控制哪些工具被调用
   last_model_request_parameters 暴露工具定义供断言

4. FunctionModel（函数驱动模型）：
   本地函数完全控制模型响应
   支持同步、异步、流式三种函数形式
   AgentInfo 提供工具定义等运行时上下文

5. 开发者体验优先：
   Tab 补全降低参数记忆负担
   argcomplete 集成 shell 补全
   load_agent 支持模块路径和文件两种加载方式
```
