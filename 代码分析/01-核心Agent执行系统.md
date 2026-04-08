# 一、核心 Agent 执行系统

> **核心文件**：`_agent_graph.py`（1915行）、`run.py`（530行）、`messages.py`（2480行）、`_run_context.py`

---

## 1.1 整体架构概览

pydantic-ai 的 Agent 执行系统建立在 `pydantic_graph` 的有向图引擎之上。每一次 Agent 运行，本质上是在一个三节点的 State Machine 中流转：

```
                  ┌─────────────────────────────────────────────┐
                  │           Agent Run (GraphRun)              │
                  └─────────────────────────────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │         UserPromptNode              │
                    │  - 组装 SystemPrompt                │
                    │  - 处理 DeferredToolResults         │
                    │  - 重评估 dynamic system prompts    │
                    └──────────────────┬──────────────────┘
                                       │
              ┌────────────────────────▼────────────────────────┐
              │              ModelRequestNode                    │
              │  - 调用 model.request() 或 request_stream()     │
              │  - 封装 Capability 中间件链                      │
              │  - 处理流式/非流式两条路径                        │
              └────────────────────────┬────────────────────────┘
                                       │
              ┌────────────────────────▼────────────────────────┐
              │              CallToolsNode                       │
              │  - 执行函数工具（function tools）                │
              │  - 处理输出验证和重试                            │
              │  - 确定是否继续循环或 End                        │
              └──────────────┬─────────────────┬────────────────┘
                             │                  │
                     继续循环                  结束
              ┌──────────────▼──┐      ┌───────▼────────────────┐
              │ ModelRequestNode│      │  End(FinalResult)       │
              └─────────────────┘      └────────────────────────┘
```

循环直到：输出验证通过、达到最大重试次数、或超出 Token 使用限制。

---

## 1.2 状态数据结构

### GraphAgentState — 运行时状态（跨节点持久化）

```python
@dataclasses.dataclass(kw_only=True)
class GraphAgentState:
    message_history: list[ModelMessage]      # 完整消息历史
    usage: RunUsage                           # Token 用量汇总
    retries: int = 0                          # 当前全局重试次数
    run_step: int = 0                         # 当前执行步骤（每次 ModelRequest + 1）
    run_id: str                               # UUID v7，时序可排序的唯一 Run ID
    metadata: dict[str, Any] | None          # 用户附加的元数据
    last_max_tokens: int | None              # 最后一次请求的 max_tokens（错误信息用）
    last_model_request_parameters: ...       # OTel Span 属性用
```

**关键方法**：
- `check_incomplete_tool_call()`：检测因 token 截断导致的不完整工具调用（`finish_reason == 'length'`），并抛出 `IncompleteToolCall`
- `increment_retries()`：递增重试计数器，超过上限后抛出 `UnexpectedModelBehavior`

### GraphAgentDeps — 依赖/配置（整个运行期间不变）

```python
@dataclasses.dataclass
class GraphAgentDeps(Generic[DepsT, OutputDataT]):
    user_deps: DepsT                   # 用户自定义依赖（注入工具函数）
    prompt: str | Sequence[UserContent] | None
    model: Model                       # 要使用的模型实例
    get_model_settings: Callable       # 动态获取 ModelSettings
    usage_limits: UsageLimits          # Token 使用限制
    max_result_retries: int            # 最大输出重试次数
    end_strategy: EndStrategy          # 'early' 或 'exhaustive'
    output_schema: OutputSchema        # 输出结构定义
    output_validators: list            # 输出验证器
    root_capability: AbstractCapability # 能力中间件链
    builtin_tools: list               # 内置工具（如图像生成）
    tool_manager: ToolManager         # 工具管理器
    tracer: Tracer                    # OTel Tracer
```

---

## 1.3 EndStrategy：两种结束策略

```python
EndStrategy = Literal['early', 'exhaustive']
```

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `'early'`（默认） | 发现第一个有效输出就立即终止，跳过剩余工具调用 | 输出工具调用先于函数调用出现时，立即返回 |
| `'exhaustive'` | 执行完所有工具调用后，再检查是否有有效输出 | 需要副作用工具全部执行完毕的场景 |

**`early` 的实现**（在 `CallToolsNode._handle_final_result` 中）：
```python
# 发现 output tool 调用时
if end_strategy == 'early' and output_tool_calls:
    # 立即返回，不处理剩余工具
    return SetFinalResult(result)
```

---

## 1.4 UserPromptNode — 节点1

**职责**：构建发送给模型的第一个 `ModelRequest`。

**核心执行流**：
```
UserPromptNode.run()
  ├── 从 capture_run_messages() 恢复或新建消息列表
  ├── 处理 DeferredToolResults（用户注入的工具结果）
  │     └── → 直接跳到 CallToolsNode（跳过 ModelRequest）
  ├── 重评估 dynamic system prompts
  └── 构建 ModelRequest
        ├── _sys_parts()   → SystemPromptPart + 动态系统提示
        └── UserPromptPart → 用户输入
```

**恢复运行的逻辑**（`user_prompt is None` 时）：
```
if 历史最后一条是 ModelRequest:
    → 重用已有 ModelRequest，is_resuming_without_prompt = True
elif 历史最后一条是 ModelResponse 且有 tool_calls:
    → 直接转到 CallToolsNode 执行未完成的工具调用
elif 历史最后一条是 ModelResponse 但无 tool_calls:
    → 获取 instructions，若有则创建新 ModelRequest
```

---

## 1.5 ModelRequestNode — 节点2

**职责**：调用模型 API，处理流式/非流式两条路径。

### 非流式路径（`_make_request`）

```
_make_request()
  ├── _prepare_request()
  │     ├── 获取 model_settings
  │     ├── 准备 tool_defs（经过 Capability.prepare_tools 过滤）
  │     └── 构建 ModelRequestParameters
  ├── root_capability.wrap_model_request()   ← Capability 中间件链
  │     └── → model.request(messages, settings, params)
  └── _finish_handling(response)
        ├── 追加 response 到 message_history
        └── → CallToolsNode(response)
```

### 流式路径（`stream` contextmanager）

流式路径使用了精妙的协程协调机制（两个 `asyncio.Event`）：

```
stream() contextmanager
  ├── _prepare_request()
  ├── 创建 stream_ready 和 stream_done 两个 Event
  ├── 异步任务 wrap_task:
  │     └── _streaming_handler()
  │           ├── model.request_stream(...) 打开流
  │           ├── stream_ready.set()  ← 通知 stream 已就绪
  │           └── await stream_done.wait()  ← 等待消费完毕
  ├── await 直到 stream_ready 或 wrap_task 完成
  ├── yield AgentStream  ← 提供给调用方消费
  └── 消费完毕后 stream_done.set() → 让 wrap_task 继续关闭流
```

这种设计保证了流的上下文管理器能在 `yield` 后正确关闭，同时也支持 Capability 层对模型请求进行拦截（`SkipModelRequest`）。

**`SkipModelRequest` 机制**：Capability 可以短路（short-circuit）整个模型调用，直接返回预先准备的 `ModelResponse`（用于缓存、测试等场景）。

---

## 1.6 CallToolsNode — 节点3

**职责**：处理模型返回的工具调用，执行工具，决定是继续还是结束。

```
CallToolsNode.run()
  ├── 判断 EndStrategy 和工具类型
  │     ├── output_tools → SetFinalResult（early策略）
  │     └── function_tools → ToolManager.execute()
  ├── 对每个 ToolCallPart 执行:
  │     ├── tool_manager.call_tool(tool_name, args, ctx)
  │     │     ├── 工具验证（Pydantic schema）
  │     │     ├── 执行函数
  │     │     └── 返回 ToolReturnPart 或 RetryPromptPart
  │     └── 汇总结果到 ModelRequest
  ├── 输出验证
  │     ├── 成功 → End(FinalResult)
  │     └── 失败 → 增加 retries，返回 ModelRequestNode（携带错误信息）
  └── 确定下一个节点
        ├── 有未处理工具 → ModelRequestNode
        └── 无待处理工具且有有效输出 → End
```

---

## 1.7 RunContext — 运行上下文（传递给工具）

`RunContext[DepsT]` 是工具函数、输出验证器、系统提示函数能够访问到的"快照"：

```python
@dataclasses.dataclass(repr=False, kw_only=True)
class RunContext(Generic[RunContextAgentDepsT]):
    deps: RunContextAgentDepsT           # 用户依赖
    model: Model                         # 当前模型
    usage: RunUsage                      # 当前 Token 用量
    agent: AbstractAgent | None
    prompt: str | Sequence[UserContent] | None
    messages: list[ModelMessage]         # 完整消息历史
    retries: dict[str, int]             # 每个工具的重试次数
    tool_call_id: str | None            # 当前工具调用 ID
    tool_name: str | None               # 当前工具名
    retry: int = 0                       # 当前工具的重试次数
    max_retries: int = 0                 # 当前工具的最大重试次数
    run_step: int = 0                    # 当前步骤
    run_id: str | None
    model_settings: ModelSettings | None  # 已解析的模型设置
```

**`ContextVar` 隔离机制**：
```python
_CURRENT_RUN_CONTEXT: ContextVar[RunContext | None] = ContextVar(...)

@contextmanager
def set_current_run_context(run_context: RunContext) -> Iterator[None]:
    token = _CURRENT_RUN_CONTEXT.set(run_context)
    try:
        yield
    finally:
        _CURRENT_RUN_CONTEXT.reset(token)
```

这确保了即使多个 Agent 并发运行，每个协程都能看到自己的 `RunContext`，通过 `token` 机制保证正确恢复。

---

## 1.8 消息系统（messages.py）

消息体系分为两大类：

### 消息类型层级

```
ModelMessage (抽象概念)
├── ModelRequest（用户/工具 → 模型）
│   parts:
│   ├── SystemPromptPart     系统提示
│   ├── UserPromptPart       用户输入（支持多模态）
│   ├── ToolReturnPart       工具执行结果
│   ├── BuiltinToolReturnPart 内置工具结果
│   ├── RetryPromptPart      工具失败重试信息
│   └── InstructionPart      指令（静态/动态）
│
└── ModelResponse（模型 → 用户）
    parts:
    ├── TextPart             文本回复
    ├── ThinkingPart         思考过程（Reasoning 模型）
    ├── ToolCallPart         函数工具调用
    ├── BuiltinToolCallPart  内置工具调用
    └── FilePart             文件内容
```

### 流式事件类型（Delta 增量）

```
ModelResponseStreamEvent（流式事件）
├── PartStartEvent      新 Part 开始
├── PartDeltaEvent      Part 内容增量
│   ├── TextPartDelta
│   ├── ThinkingPartDelta
│   └── ToolCallPartDelta
├── PartEndEvent        Part 结束
├── FinalResultEvent    最终结果事件
├── FunctionToolCallEvent  工具调用事件
└── FunctionToolResultEvent 工具结果事件
```

### 多模态内容支持

`UserPromptPart.content` 可以是：
- `str`（文本）
- `Sequence[UserContent]`，其中 `UserContent` 包括：
  - `str`
  - `ImageUrl` / `BinaryImage`（图像）
  - `AudioUrl`（音频）
  - `VideoUrl`（视频）
  - `DocumentUrl`（文档：PDF、Word、Excel 等）
  - `BinaryContent`（原始二进制）

---

## 1.9 capture_run_messages — 消息捕获机制

```python
@contextmanager
def capture_run_messages() -> Iterator[list[ModelMessage]]:
    """在流式运行中捕获消息的 contextmanager。

    Example:
        with capture_run_messages() as messages:
            result = await agent.run_sync('prompt')
        # messages 包含完整消息历史
    """
```

**内部机制**：
- 使用 `_CAPTURED_RUN_MESSAGES: ContextVar[_RunMessages | None]` 存储捕获列表
- `UserPromptNode.run()` 在首次执行时，将 `ctx.state.message_history` 重指向被捕获的列表
- 这样在流式 `agent.run()` 返回前，外部也能实时看到消息历史

---

## 1.10 AgentRun — 顶层迭代接口（run.py）

```python
async with agent.iter('prompt') as agent_run:
    async for node in agent_run:
        # 可以观察每个节点的执行
        print(type(node).__name__)
    print(agent_run.result.output)
```

`AgentRun` 包装了底层的 `GraphRun`，提供：
- 异步迭代节点的接口
- `.result` 属性（`AgentRunResult`）
- `.usage()` Token 统计
- `.run_id` 唯一标识

`AgentRunResult` 提供：
- `.output`：最终输出（泛型 `OutputDataT`）
- `.all_messages()`：完整消息历史
- `.new_messages()`：本次运行产生的新消息
- `.all_messages_json()`：JSON 序列化（用于持久化对话历史）

---

## 1.11 build_run_context — 上下文组装

```python
def build_run_context(ctx: GraphRunContext[GraphAgentState, GraphAgentDeps]) -> RunContext[DepsT]:
    """从 Graph 运行时上下文组装 RunContext。"""
    return RunContext(
        deps=ctx.deps.user_deps,
        model=ctx.deps.model,
        usage=ctx.state.usage,
        messages=ctx.state.message_history,
        run_step=ctx.state.run_step,
        run_id=ctx.state.run_id,
        # ... 其他字段
    )
```

每次需要 `RunContext` 时（如调用工具、动态系统提示）都通过此函数创建，确保包含最新的运行时状态。

---

## 总结

```
关键设计决策
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 图驱动执行     → 每次 Agent 运行 = 一次 Graph Traversal
                    节点 = 原子执行单元，状态 = 消息历史

2. 状态/依赖分离  → GraphAgentState（可变，跨节点传递）
                    GraphAgentDeps（不变，整个 Run 期间固定）

3. Capability 中间件 → 模型请求可被拦截/包装（缓存、日志、测试）
                       通过 wrap_model_request 钩子实现

4. 两种结束策略   → early（性能优先）vs exhaustive（副作用完整）

5. ContextVar 隔离 → 并发安全的 RunContext 访问
                     通过 token-based reset 保证异步安全

6. 流式协程协调   → asyncio.Event 实现 stream 生产者/消费者协调
                     wrap_task + _streaming_handler 两协程协作
```
