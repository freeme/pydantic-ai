# 七、Durable Execution（持久化执行）

> **核心文件**：`durable_exec/dbos/`、`durable_exec/temporal/`、`durable_exec/prefect/`

---

## 7.1 设计目标

Durable Execution 解决的核心问题：**Agent 运行可能耗时很长（数分钟到数小时），进程崩溃后如何从断点恢复？**

```
普通 Agent 运行：
  process → [Agent.run()] → 结束
  ↓ 崩溃
  × 从头重来，所有工具调用结果丢失

Durable Execution：
  process → [Agent.run()] → 工作流引擎保存每一步 → 结束
  ↓ 崩溃
  ✓ 从最后一个成功步骤恢复，不重复调用工具
```

---

## 7.2 核心设计模式：WrapperAgent + WrapperModel

所有 Durable Execution 集成都采用相同的模式：

```
原始 Agent（正常使用）
    │
    ▼ 包装
DBOSAgent / TemporalAgent（在工作流引擎中使用）
    │
    ├── 模型调用 → DBOSModel / TemporalModel
    │   （将 model.request() 包装为工作流步骤/活动）
    │
    └── 工具调用 → DBOSToolset / TemporalWrapperToolset
        （将 toolset.call_tool() 包装为工作流步骤/活动）
```

**原始 Agent 不受影响**，包装后的 Agent 在工作流引擎中使用。

---

## 7.3 DBOS 集成

**文件**：`durable_exec/dbos/`

[DBOS](https://www.dbos.dev) 是一个基于数据库的持久化执行框架，每个步骤的结果存储在 PostgreSQL 中。

### DBOSAgent

```python
@DBOS.dbos_class()
class DBOSAgent(WrapperAgent, DBOSConfiguredInstance):

    def __init__(self, wrapped: AbstractAgent, *, name: str, ...):
        # 1. 将 wrapped.model 替换为 DBOSModel
        dbos_model = DBOSModel(wrapped.model, step_config=model_step_config)
        self._model = dbos_model

        # 2. 将所有 MCP Toolset 替换为 DBOSMCPServer
        dbos_toolsets = [toolset.visit_and_replace(dbosify_toolset) for toolset in wrapped.toolsets]

        # 3. 将 run() 方法包装为 DBOS 工作流
        @DBOS.workflow(name=f'{name}.run')
        async def wrapped_run_workflow(...):
            return await self._run_agent(...)
```

### DBOSModel

核心机制：将模型请求包装为 DBOS Step（幂等执行单元）：

```python
class DBOSModel(WrapperModel):

    def __init__(self, model: Model, step_config: StepConfig):
        # 将 request 包装为 DBOS step
        @DBOS.step(name=f'{prefix}__model.request', **step_config)
        async def wrapped_request_step(messages, settings, params) -> ModelResponse:
            return await super().request(messages, settings, params)
        self._dbos_wrapped_request_step = wrapped_request_step

        # 将 request_stream 包装为 DBOS step（流式→非流式）
        @DBOS.step(name=f'{prefix}__model.request_stream', **step_config)
        async def wrapped_request_stream_step(...) -> ModelResponse:
            async with super().request_stream(...) as sr:
                async for _ in sr: pass  # 消费完整个流
            return sr.get()  # 只保存最终 ModelResponse
```

**流式处理的关键**：DBOS Step 只能保存**确定性的输出**（JSON 可序列化），因此流式请求在步骤内被完全消费，最终以 `ModelResponse`（非流式）返回。这意味着在 DBOS 中，流式的中间事件不会被外部观察到，但 `event_stream_handler` 在步骤内仍然可以处理它们。

### DBOS 的幂等性保证

DBOS 使用步骤名称作为幂等键。如果步骤已执行成功，重放时直接返回存储的结果，不重新执行：

```
第一次运行：model.request() → 执行 → 存储结果
  ↓ 崩溃
重放：model.request() → 查询 DB → 返回已存储结果（不重新调用 API）
```

---

## 7.4 Temporal 集成

**文件**：`durable_exec/temporal/`

[Temporal](https://temporal.io) 是行业标准的工作流引擎，使用 Activity 作为持久化执行单元。

### TemporalAgent

```python
class TemporalAgent(WrapperAgent):

    def __init__(
        self,
        wrapped: AbstractAgent,
        *,
        name: str,
        models: Mapping[str, Model] | None = None,   # 多模型注册表
        provider_factory: TemporalProviderFactory | None = None,
        activity_config: ActivityConfig | None = None,  # 默认 Activity 配置
        model_activity_config: ActivityConfig | None = None,  # 模型专用配置
        toolset_activity_config: dict[str, ActivityConfig] | None = None,
        tool_activity_config: dict[str, dict[str, ActivityConfig | Literal[False]]] | None = None,
        run_context_type: type[TemporalRunContext] = TemporalRunContext,
    ):
```

**每个工具可以有独立的 Activity 配置**（超时、重试策略等），`tool_activity_config[toolset_id][tool_name]` 精确控制每个工具的 Activity 行为。

### TemporalModel — 将模型请求转为 Activity

```python
class TemporalModel(WrapperModel):
    """将模型请求包装为 Temporal Activity。"""
```

**与 DBOS 的关键区别**：
1. **多模型支持**：通过 `models` 注册表，可以在运行时选择模型（`run(model='model-id')`）
2. **Provider Factory**：`provider_factory` 允许在 Activity 执行时从 RunContext 注入 API Key（依赖注入场景）
3. **`_RequestParams`**：需要将 Pydantic 数据结构序列化为 Temporal 可传输的格式

```python
@dataclass
@with_config(ConfigDict(arbitrary_types_allowed=True))
class _RequestParams:
    messages: list[ModelMessage]
    model_settings: dict[str, Any] | None  # 注意：使用 dict 而非 ModelSettings 防止字段丢失
    model_request_parameters: ModelRequestParameters
    serialized_run_context: Any
    model_id: str | None = None
```

### TemporalWrapperToolset — 将工具调用转为 Activity

```python
class TemporalWrapperToolset(WrapperToolset):
    """将工具调用包装为 Temporal Activity。"""

    async def call_tool(self, name, tool_args, ctx, tool) -> Any:
        # 通过 Temporal 调度 Activity 执行工具
        return await workflow.execute_activity(
            self._activity_func,
            _ToolCallParams(name=name, tool_args=tool_args, serialized_ctx=...),
            **self.get_activity_config(name),
        )
```

### PydanticAIWorkflow — 注册点

```python
class PydanticAIWorkflow:
    """Temporal Workflow 基类，注册 TemporalAgent。"""
    __pydantic_ai_agents__: Sequence[TemporalAgent]
```

用户继承此类创建自己的 Temporal Workflow，`__pydantic_ai_agents__` 列表中的 Agent 会自动注册其 Activities。

---

## 7.5 Prefect 集成

**文件**：`durable_exec/prefect/`

[Prefect](https://www.prefect.io) 是数据工程领域的工作流编排工具，与 Temporal/DBOS 类似但更偏数据处理场景。

Prefect 集成采用相同的包装模式（`WrapperModel` + `WrapperToolset`），将模型请求和工具调用包装为 Prefect Task。

---

## 7.6 各系统对比

| 特性 | DBOS | Temporal | Prefect |
|------|------|----------|---------|
| 存储后端 | PostgreSQL | 内部数据库 | 文件/DB |
| 幂等机制 | 步骤名称 | Activity ID | Task ID |
| 工具粒度配置 | Step 级别 | 精确到单个工具 | Task 级别 |
| 多模型支持 | 单模型 | ✅（注册表） | — |
| 流式处理 | 步骤内消费 | Activity 内消费 | — |
| 序列化 | JSON（数据库） | JSON（Temporal） | — |

---

## 7.7 工具串行化要求

所有 Durable Execution 集成都要求工具参数和结果可以 JSON 序列化，因为：
1. 参数需要传递给工作流引擎
2. 结果需要保存在持久化存储中
3. 失败时从存储中恢复参数和结果

DBOS 使用 `'parallel_ordered_events'` 作为默认执行模式（而非 `'parallel'`），因为 DBOS 工作流需要**确定性的重放**，而并行执行的事件顺序可能不确定。

---

## 7.8 TemporalRunContext — 可序列化的 RunContext

Temporal Activities 是跨进程执行的，RunContext 需要能被序列化和反序列化：

```python
class TemporalRunContext(Generic[AgentDepsT]):
    """Temporal Activity 中的 RunContext，支持序列化。"""

    # 用户需要将 deps 类型标注为可序列化的
    deps: AgentDepsT

    @classmethod
    def deserialize(cls, serialized: Any) -> TemporalRunContext:
        """从序列化形式恢复 RunContext。"""
```

用户的 `deps` 类型必须是可序列化的（Pydantic 模型或基本类型），因为它需要跨越 Activity 边界传输。

---

## 总结

```
设计模式总结
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 装饰器模式：
   WrapperAgent 和 WrapperModel 完全保持接口兼容
   原始 Agent 和 Durable Agent 可以互换使用

2. 步骤/活动 = 幂等执行单元：
   每个模型请求、工具调用被包装为持久化步骤
   失败时从存储中恢复，不重复执行

3. 流式→非流式降级：
   流式模型请求在步骤内完全消费，只保存最终结果
   event_stream_handler 在步骤内仍可访问流式事件

4. 序列化约束：
   所有跨步骤的数据必须 JSON 可序列化
   RunContext.deps、工具参数/结果都需满足此要求

5. 确定性重放（Temporal/DBOS）：
   使用 'parallel_ordered_events' 而非 'parallel'
   保证工作流重放时事件顺序确定
```
