# 三、工具与 Toolset 系统

> **核心文件**：`tools.py`（595行）、`toolsets/abstract.py`、`_tool_manager.py`、`toolsets/` 目录

---

## 3.1 整体层次架构

```
                 ┌─────────────────────────────────────────┐
                 │           AbstractToolset               │
                 │  + get_tools() → dict[name, ToolsetTool]│
                 │  + call_tool(name, args, ctx, tool)     │
                 │  + get_instructions()                   │
                 │  + for_run() / for_run_step()           │
                 │  + __aenter__ / __aexit__ (资源管理)     │
                 └──────────────────┬──────────────────────┘
                                    │
          ┌─────────────────────────┼──────────────────────────┐
          │                         │                          │
  ┌───────▼──────┐        ┌─────────▼──────┐       ┌──────────▼──────┐
  │ FunctionToolset│      │  WrapperToolset │       │ CombinedToolset │
  │ 函数工具集    │        │  （装饰器基类）  │       │ （多个合并）     │
  │ MCPToolset    │        │               │        │                 │
  │ ExternalToolset│       │  过滤器链：    │        │                 │
  └───────────────┘        │  Filtered     │        │                 │
                           │  Prefixed     │        │                 │
                           │  Renamed      │        │                 │
                           │  Prepared     │        │                 │
                           │  ApprovalReq  │        │                 │
                           │  DeferredLoad │        │                 │
                           │  ToolSearch   │        │                 │
                           │  DynamicToolset│        │                │
                           └───────────────┘        └─────────────────┘
```

**ToolManager** 在每个 run step 中统一管理所有 Toolset，是执行层的入口。

---

## 3.2 ToolDefinition — 工具描述符

工具的"元信息"，传递给模型 API 告知其可用工具：

```python
@dataclass(repr=False, kw_only=True)
class ToolDefinition:
    name: str                               # 工具名称（唯一）
    parameters_json_schema: ObjectJsonSchema # 参数 JSON Schema
    description: str | None                 # 给模型看的描述
    outer_typed_dict_key: str | None        # 外层 TypedDict 键（输出工具用）
    strict: bool | None                     # 是否强制严格 Schema（OpenAI/Anthropic）
    sequential: bool = False                # 是否需要串行执行
    kind: ToolKind = 'function'             # 'function'|'output'|'external'|'unapproved'
    metadata: dict[str, Any] | None        # 不发给模型的内部元数据
    timeout: float | None                   # 执行超时（秒）
    defer_loading: bool = False             # 是否延迟加载（工具搜索用）
    prefer_builtin: str | None              # 内置工具本地 fallback 标识
```

### ToolKind 含义

| Kind | 含义 | 执行者 |
|------|------|--------|
| `'function'` | 普通函数工具，pydantic-ai 执行 | pydantic-ai 框架 |
| `'output'` | 输出工具，作为结构化结果返回 | pydantic-ai 框架 |
| `'external'` | 延迟工具，结果由外部提供 | 用户/外部系统 |
| `'unapproved'` | 需人工审批才能执行 | 需 ToolApproved 后才执行 |

---

## 3.3 Tool — 函数工具包装器

`Tool` 类封装了一个 Python 函数，包含：
- 函数引用
- Pydantic Schema（自动从类型注解生成）
- 执行配置（max_retries、timeout、sequential）
- 审批要求（requires_approval）

```python
tool = Tool(
    my_function,
    name="my_tool",
    description="...",
    max_retries=3,
    prepare=only_if_active,       # 动态决定是否注册
    args_validator=validate_args,  # 自定义参数验证
    strict=True,                   # 强制严格 Schema
    sequential=True,               # 串行执行
    requires_approval=True,        # 需人工审批
    timeout=30.0,                  # 30秒超时
)
```

**函数签名的两种形式**：

```python
# 形式一：带 RunContext（推荐，可访问依赖注入）
async def tool_with_ctx(ctx: RunContext[MyDeps], arg1: str, arg2: int) -> str:
    db = ctx.deps.database  # 访问注入的依赖
    return await db.query(arg1)

# 形式二：普通函数（简单工具）
def simple_tool(arg1: str, arg2: int) -> str:
    return f"{arg1}: {arg2}"
```

### ToolPrepareFunc — 动态工具定义

`prepare` 函数在每次运行时动态决定工具是否可用：

```python
def only_if_premium(ctx: RunContext[UserDeps], tool_def: ToolDefinition) -> ToolDefinition | None:
    if ctx.deps.user.is_premium:
        return tool_def  # 注册工具
    return None  # 不注册（对模型隐藏）
```

---

## 3.4 AbstractToolset — Toolset 抽象接口

**文件**：`toolsets/abstract.py`

### 核心生命周期

```
┌───────────────────────────────────────────────────────────────┐
│                    Toolset 生命周期                            │
├───────────────────────────────────────────────────────────────┤
│  1. for_run(ctx)       ← 每次 agent.run() 调用一次            │
│     ↓ 返回 run-scoped 的 toolset 实例                          │
│  2. __aenter__()       ← 进入上下文（设置网络连接等）           │
│     ↓                                                          │
│  3. for_run_step(ctx)  ← 每个 ModelRequestNode 调用一次        │
│     ↓ 返回 step-scoped 的 toolset 实例                         │
│  4. get_tools(ctx)     ← 获取当前 step 的可用工具              │
│     ↓ 返回 dict[name, ToolsetTool]                             │
│  5. call_tool(...)     ← 工具被模型调用时执行                  │
│     ↓                                                          │
│  6. get_instructions() ← 给模型的使用说明（每步可变）           │
│     ↓                                                          │
│  7. __aexit__()        ← 退出上下文（清理资源）                 │
└───────────────────────────────────────────────────────────────┘
```

### ToolsetTool — 工具运行时描述

```python
@dataclass
class ToolsetTool:
    toolset: AbstractToolset         # 来自哪个 Toolset（错误信息用）
    tool_def: ToolDefinition         # 工具描述符
    max_retries: int                 # 最大重试次数
    args_validator: SchemaValidator  # Pydantic Core 验证器
    args_validator_func: Callable | None  # 自定义验证函数
```

### Toolset 装饰器方法（流式 API）

`AbstractToolset` 提供了一系列方法直接返回装饰后的 Toolset：

```python
toolset = (
    MyToolset()
    .filtered(lambda ctx, tool: ctx.deps.user.can_use(tool.name))
    .prefixed("my_")
    .prepared(lambda ctx, defs: [replace(d, strict=True) for d in defs])
    .approval_required(lambda ctx, tool, args: tool.name in SENSITIVE_TOOLS)
    .defer_loading(tool_names=['search_web', 'read_file'])
)
```

---

## 3.5 WrapperToolset — 透明代理装饰器基类

类似 `WrapperModel`，`WrapperToolset` 是所有装饰器 Toolset 的基类：

```python
class WrapperToolset(AbstractToolset[AgentDepsT]):
    wrapped: AbstractToolset[AgentDepsT]

    async def get_tools(self, ctx) -> dict[str, ToolsetTool]:
        return await self.wrapped.get_tools(ctx)

    async def call_tool(self, name, tool_args, ctx, tool) -> Any:
        return await self.wrapped.call_tool(name, tool_args, ctx, tool)
    # ... 其他方法透传
```

子类只需覆盖想要拦截的方法，典型例子：

**FilteredToolset** — 按条件过滤工具：
```python
async def get_tools(self, ctx):
    all_tools = await self.wrapped.get_tools(ctx)
    return {name: tool for name, tool in all_tools.items()
            if self.filter_func(ctx, tool.tool_def)}
```

**PrefixedToolset** — 给工具名加前缀：
```python
async def get_tools(self, ctx):
    tools = await self.wrapped.get_tools(ctx)
    return {f"{self.prefix}{name}": tool for name, tool in tools.items()}
```

---

## 3.6 ApprovalRequiredToolset — 人工审批工具调用

**文件**：`toolsets/approval_required.py`

```python
class ApprovalRequiredToolset(WrapperToolset):
    approval_required_func: Callable[[RunContext, ToolDefinition, dict], bool]

    async def call_tool(self, name, tool_args, ctx, tool) -> Any:
        if not ctx.tool_call_approved and self.approval_required_func(ctx, tool.tool_def, tool_args):
            raise ApprovalRequired  # 中断执行，等待审批
        return await super().call_tool(name, tool_args, ctx, tool)
```

**审批流程**：
```
1. 首次调用 → ApprovalRequired 异常
2. 框架捕获 → 工具调用标记为 'unapproved' 种类
3. Agent 返回 DeferredToolRequests（包含待审批的工具调用）
4. 用户审批后，创建 DeferredToolResults 传入下一次 agent.run()
5. ctx.tool_call_approved = True → 工具正常执行
```

---

## 3.7 DynamicToolset — 运行时动态工具集

**文件**：`toolsets/_dynamic.py`

`DynamicToolset` 通过函数在运行时动态决定工具集：

```python
async def get_user_tools(ctx: RunContext[UserDeps]) -> AbstractToolset | None:
    if ctx.deps.user.role == 'admin':
        return admin_toolset
    return user_toolset

agent = Agent('openai:gpt-5.2', toolsets=[DynamicToolset(get_user_tools)])
```

**`per_run_step=True`（默认）**：每个 run step 重新评估工厂函数，允许工具集在对话中动态变化。

**生命周期管理**：`for_run_step()` 负责退出旧的内部 toolset、进入新的内部 toolset，管理 `__aenter__`/`__aexit__` 的切换。

---

## 3.8 ToolSearchToolset — 大规模工具集的语义搜索

**文件**：`toolsets/_tool_search.py`

当工具数量很多时（如 MCP 服务器提供大量工具），一次性传给模型所有工具定义会消耗大量 token。`ToolSearchToolset` 提供"按需发现"机制：

```
初始状态：
  visible_tools = [常用工具, search_tools]  ← 只暴露少量工具 + 搜索工具

模型调用 search_tools(keywords="...")：
  搜索引擎 → 按关键词匹配工具名和描述
  返回匹配的工具定义 → 模型发现新工具

下一步：
  visible_tools = [常用工具, search_tools, 发现的工具]  ← 工具集扩展
```

**标记工具为延迟加载**：
```python
Tool(my_func, defer_loading=True)      # 单个工具延迟加载
toolset.defer_loading(['tool_a', 'tool_b'])  # Toolset 级别标记
```

---

## 3.9 DeferredToolResults — 延迟工具执行

**文件**：`tools.py`

延迟执行设计用于"工具调用需要外部系统处理"的场景：

```
┌─────────────────────────────────────────────────────────┐
│ 1. agent.run() 返回 DeferredToolRequests                │
│    calls: [ToolCallPart(id='abc', name='query_db')]     │
│    approvals: [ToolCallPart(id='xyz', name='delete')]   │
│                                                          │
│ 2. 用户/外部系统处理工具调用:                            │
│    db_result = await query_database(...)                │
│    user_approved = await ask_user(...)                  │
│                                                          │
│ 3. agent.run(                                           │
│       deferred_tool_results=DeferredToolResults(         │
│           calls={'abc': ToolReturn('result')},          │
│           approvals={'xyz': True},  ← True = 批准       │
│       )                                                  │
│    )                                                     │
└─────────────────────────────────────────────────────────┘
```

**ToolApproved** 和 **ToolDenied** 控制审批结果：
- `ToolApproved(override_args=...)` — 批准，可修改参数
- `ToolDenied(message="...")` — 拒绝，附带拒绝原因

---

## 3.10 ToolManager — 统一执行层

**文件**：`_tool_manager.py`

`ToolManager` 是 `GraphAgentDeps` 中的关键组件，统一管理所有工具的验证和执行：

```python
@dataclass
class ToolManager(Generic[AgentDepsT]):
    toolset: AbstractToolset[AgentDepsT]   # 所有工具集（CombinedToolset）
    root_capability: AbstractCapability | None  # 用于钩子
    tools: dict[str, ToolsetTool] | None   # 缓存的工具（当前 step）
    failed_tools: set[str]                 # 本 step 已失败的工具
    default_max_retries: int = 1
```

### 执行流程

```
ToolManager.call_tool(tool_call_part, ctx)
  ├── 1. 验证阶段 validate_tool_call()
  │     ├── 查找工具（未知工具 → ToolRetryError）
  │     ├── Schema 验证（args_validator.validate_json()）
  │     │     ├── 验证失败 → ToolRetryError（ValidationError → 友好错误信息）
  │     │     └── 成功 → validated_args
  │     └── 自定义 args_validator_func（可抛 ModelRetry）
  │
  ├── 2. 执行阶段 execute_tool()
  │     ├── 更新 ctx（tool_name, tool_call_id, retry, max_retries）
  │     ├── 调用 toolset.call_tool(name, validated_args, ctx, tool)
  │     ├── 超时处理（asyncio.wait_for + timeout 参数）
  │     └── 异常转换：
  │           ModelRetry → RetryPromptPart（重试指令）
  │           其他异常 → UnexpectedModelBehavior
  │
  └── 3. 结果转换
        ├── 工具返回值 → ToolReturnPart（序列化为 JSON）
        └── ModelRetry → RetryPromptPart（告知模型重试）
```

### ValidatedToolCall — 验证与执行分离

```python
@dataclass
class ValidatedToolCall:
    call: ToolCallPart      # 原始工具调用
    tool: ToolsetTool | None
    ctx: RunContext
    args_valid: bool        # 验证是否通过
    validated_args: dict | None
    validation_error: ToolRetryError | None  # 验证失败的错误
```

将验证和执行分离，好处：
1. 可以先批量验证，再并行执行
2. 事件流中可以准确报告 `args_valid` 状态
3. 验证失败和执行失败可以分开处理

### 并行执行模式

```python
ParallelExecutionMode = Literal['parallel', 'sequential', 'parallel_ordered_events']
```

- `'parallel'`（默认）：所有工具并行执行（`asyncio.gather`）
- `'sequential'`：串行执行（`sequential=True` 的工具）
- `'parallel_ordered_events'`：并行执行，但事件按工具调用顺序输出

---

## 3.11 FunctionToolset — 内置函数工具集

**文件**：`toolsets/function.py`

`FunctionToolset` 是管理 `Tool` 对象的核心工具集，它是 `Agent.tool()` 装饰器背后的实现：

```python
@agent.tool
async def search(ctx: RunContext[Deps], query: str) -> str:
    # 框架自动创建 Tool(search) 并注册到 FunctionToolset
    ...
```

**Schema 缓存**：Pydantic Schema Validator 在工具注册时一次性创建，之后被缓存复用。

---

## 3.12 工具系统总结

```
设计亮点
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 装饰器模式（WrapperToolset）：
   过滤、前缀、重命名、审批等功能通过组合实现，
   无需修改原始 Toolset

2. 生命周期钩子（for_run / for_run_step）：
   允许工具集在 run 级别和 step 级别进行状态管理
   DynamicToolset 利用 for_run_step 实现动态工具集切换

3. 延迟工具执行：
   DeferredToolResults 将 Agent 的工具调用与外部系统解耦
   允许"暂停-继续"模式（human-in-the-loop）

4. 工具搜索（ToolSearchToolset）：
   解决大规模工具集的 token 效率问题
   工具先"隐藏"，模型通过搜索按需发现

5. 验证/执行分离（ValidatedToolCall）：
   先验证再执行，支持精确的事件流 + 并行执行优化

6. 类型安全：
   Tool 从 Python 类型注解自动生成 JSON Schema
   Pydantic Core SchemaValidator 确保运行时类型安全
```
