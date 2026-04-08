# Pydantic-AI 代码探索 Todo

> 目标：系统性地从各个维度理解 pydantic-ai 的代码设计与实现，涵盖执行引擎、模型抽象、工具系统、持久化、评估框架等核心领域。

---

## 架构总览

```
pydantic-ai/
├── pydantic_ai_slim/pydantic_ai/   # 核心包
│   ├── agent/                       # Agent 抽象与规格
│   ├── models/                      # 模型抽象层（18+ 种模型）
│   ├── toolsets/                    # 工具集系统
│   ├── providers/                   # Provider 配置层
│   ├── profiles/                    # 模型能力 Profiles
│   ├── capabilities/                # 能力声明与组合
│   ├── durable_exec/                # 持久化执行（DBOS/Temporal/Prefect）
│   ├── embeddings/                  # 嵌入模型
│   ├── ui/                          # UI 协议（AG-UI/Vercel AI）
│   └── _agent_graph.py              # 核心执行图（★ 最重要的文件）
├── pydantic_graph/                  # 通用图执行引擎
└── pydantic_evals/                  # 评估框架
```

---

## 一、核心 Agent 执行系统

> **入口**：`_agent_graph.py`（1915行）是最核心的文件，理解整个执行生命周期的钥匙

- [ ] **1.1 Agent Graph 执行引擎**
  - [ ] 阅读 `_agent_graph.py` 全文，理解 `GraphAgentState`、`GraphAgentDeps` 的状态结构
  - [ ] 追踪 `UserPromptNode → ModelRequestNode → CallToolsNode` 的三节点执行流
  - [ ] 理解 `EndStrategy`（`early` vs `exhaustive`）的区别和适用场景
  - [ ] 分析 `build_run_context()` 如何组装运行时上下文
  - [ ] 研究 `capture_run_messages()` 的消息捕获机制

- [ ] **1.2 Agent 抽象与规格**
  - [ ] 阅读 `agent/abstract.py`，理解 `AbstractAgent` 接口设计
  - [ ] 阅读 `agent/spec.py`，理解 Agent 规格声明
  - [ ] 阅读 `agent/wrapper.py`，理解包装模式
  - [ ] 对比 `agent/` 与 `_agent_graph.py` 的分工边界

- [ ] **1.3 运行上下文与状态**
  - [ ] 阅读 `_run_context.py`，理解 `RunContext` 和 `set_current_run_context()`
  - [ ] 阅读 `run.py`（530行），理解顶层 run API 的组织方式
  - [ ] 理解 `ContextVar` 在并发场景下的状态隔离机制
  - [ ] 分析重试状态（`retries`）如何在多轮中传递

- [ ] **1.4 消息系统**
  - [ ] 阅读 `messages.py`（2480行，最长文件），梳理完整的消息类型体系
  - [ ] 绘制 `ModelMessage` → `ModelRequest` / `ModelResponse` 继承关系图
  - [ ] 理解 `ToolCallPart`、`ToolReturnPart`、`RetryPromptPart` 的语义
  - [ ] 分析 `_history_processor.py`，理解历史压缩/截断机制

---

## 二、模型抽象层

> **关键文件**：`models/__init__.py`（1561行），定义了所有模型必须遵守的接口契约

- [ ] **2.1 模型接口定义**
  - [ ] 阅读 `models/__init__.py`，理解 `Model`、`AgentModel`、`StreamedResponse` 核心抽象
  - [ ] 理解 `ModelRequestContext` 的组成和传递路径
  - [ ] 分析 `ModelResponse` 和 `StreamTextResponse`/`StreamStructuredResponse` 的区分

- [ ] **2.2 主要模型实现**
  - [ ] 阅读 `models/openai.py`，作为参考实现，理解完整 Model 实现模式
  - [ ] 对比 `models/anthropic.py`，理解工具调用格式差异
  - [ ] 阅读 `models/gemini.py`，理解 Google 特有的流式处理方式
  - [ ] 阅读 `models/function.py`，理解用于测试的函数式 Mock 模型

- [ ] **2.3 模型包装与组合**
  - [ ] 阅读 `models/wrapper.py`，理解透明代理模式
  - [ ] 阅读 `models/fallback.py`，理解多模型 Fallback 链的实现
  - [ ] 阅读 `models/instrumented.py`，理解 OTel 追踪注入机制
  - [ ] 阅读 `models/mcp_sampling.py`，理解通过 MCP 协议代理模型请求

- [ ] **2.4 并发与速率控制**
  - [ ] 阅读 `models/concurrency.py`，理解并发限制的包装器设计

---

## 三、工具与 Toolset 系统

> **关键文件**：`toolsets/abstract.py` 定义了工具集接口，`tools.py` 定义了工具执行上下文

- [ ] **3.1 Tool 基础定义**
  - [ ] 阅读 `tools.py`（595行），理解 `RunContext`、`ToolDefinition`、`ToolKind` 的设计
  - [ ] 理解 `DeferredToolCallResult` / `DeferredToolResult` 的延迟执行模式
  - [ ] 理解 `ToolApproved`/`ToolDenied` 审批机制

- [ ] **3.2 Toolset 抽象与实现**
  - [ ] 阅读 `toolsets/abstract.py`，理解 `AbstractToolset` 生命周期
  - [ ] 阅读 `toolsets/function.py`，理解函数工具集的注册机制
  - [ ] 阅读 `toolsets/combined.py`，理解多工具集组合
  - [ ] 阅读 `toolsets/filtered.py`、`prefixed.py`、`renamed.py`，理解工具集装饰器模式

- [ ] **3.3 高级 Toolset 特性**
  - [ ] 阅读 `toolsets/approval_required.py`，理解人工审批工具调用流程
  - [ ] 阅读 `toolsets/deferred_loading.py`，理解惰性加载 Toolset
  - [ ] 阅读 `toolsets/_dynamic.py`，理解动态工具集（运行时决定工具集合）
  - [ ] 阅读 `toolsets/_tool_search.py`，理解工具搜索/语义检索机制
  - [ ] 阅读 `toolsets/external.py`，理解外部工具集集成

- [ ] **3.4 Tool Manager**
  - [ ] 阅读 `_tool_manager.py`，理解 `ToolManager` 如何统一管理工具调用、验证、重试
  - [ ] 分析 `ValidatedToolCall` 的验证流程

---

## 四、输出与结果系统

> **关键设计**：pydantic-ai 的结构化输出是核心卖点，基于 Pydantic 的 Schema 生成

- [ ] **4.1 输出规格设计**
  - [ ] 阅读 `output.py`，理解 `OutputDataT` 和 `OutputSpec` 的泛型设计
  - [ ] 阅读 `_output.py`，理解内部输出处理实现
  - [ ] 理解 `output.py` 中 `NativeOutput`、`ToolOutput` 两种输出模式

- [ ] **4.2 结果处理**
  - [ ] 阅读 `result.py`，理解 `RunResult`、`StreamedRunResult` 的结构
  - [ ] 理解流式结果的迭代协议和资源管理

- [ ] **4.3 函数 Schema 生成**
  - [ ] 阅读 `_function_schema.py`，理解从 Python 函数类型注解自动生成 JSON Schema
  - [ ] 分析与 Pydantic 模型的集成点

- [ ] **4.4 重试机制**
  - [ ] 阅读 `retries.py`，理解重试策略和 `ToolRetryError` 触发机制
  - [ ] 追踪重试从 `ToolRetryError` 到 `RetryPromptPart` 的完整流程

---

## 五、Provider 与 Profiles 系统

> **设计理解**：Provider 负责认证/连接配置，Profile 负责模型能力声明，两者相互独立

- [ ] **5.1 Provider 系统**
  - [ ] 阅读 `providers/openai.py`，理解 Provider 的职责边界（认证、base_url）
  - [ ] 对比 `providers/anthropic.py` 和 `providers/bedrock.py`，理解差异
  - [ ] 阅读 `providers/litellm.py`，理解统一代理 Provider 的实现

- [ ] **5.2 Profiles 系统**
  - [ ] 阅读 `profiles/abstract.py`，理解 `ModelProfile` 接口
  - [ ] 阅读几个具体 Profile（`openai.py`、`anthropic.py`、`google.py`），理解能力声明的内容
  - [ ] 理解 Profile 如何影响工具格式、上下文长度、Thinking 等行为

- [ ] **5.3 Capabilities 系统**
  - [ ] 阅读 `capabilities/abstract.py`，理解 `AbstractCapability` 抽象
  - [ ] 阅读 `capabilities/toolset.py`、`capabilities/web_search.py` 等具体实现
  - [ ] 理解 `capabilities/hooks.py` 的钩子机制
  - [ ] 理解 `capabilities/combined.py` 的能力组合

---

## 六、pydantic_graph 图执行引擎

> **基础设施层**：pydantic-ai 的 Agent 运行时建立在 pydantic_graph 之上

- [ ] **6.1 核心图抽象**
  - [ ] 阅读 `pydantic_graph/graph.py`，理解 `Graph` 的构建和执行
  - [ ] 阅读 `pydantic_graph/nodes.py`，理解 `BaseNode`、`End` 节点设计
  - [ ] 理解 `GraphRunContext` 的运行时上下文组成

- [ ] **6.2 Graph Builder（Beta）**
  - [ ] 阅读 `pydantic_graph/beta/` 目录，理解 `GraphBuilder` 的动态图构建 API
  - [ ] 对比 `Graph`（静态）和 `GraphBuilder`（动态）的使用场景

- [ ] **6.3 图状态持久化**
  - [ ] 阅读 `pydantic_graph/persistence/in_mem.py`，理解内存持久化
  - [ ] 阅读 `pydantic_graph/persistence/file.py`，理解文件持久化
  - [ ] 理解持久化接口（`_utils.py`），为 Durable Execution 做铺垫

- [ ] **6.4 Mermaid 可视化**
  - [ ] 阅读 `pydantic_graph/mermaid.py`，理解图结构可视化生成原理

---

## 七、Durable Execution（持久化执行）

> **高级特性**：让 Agent 在宕机/重启后可以从断点恢复

- [ ] **7.1 DBOS 集成**
  - [ ] 阅读 `durable_exec/dbos/_agent.py`，理解 DBOS 工作流 Agent 包装
  - [ ] 阅读 `durable_exec/dbos/_model.py`，理解 DBOS 幂等模型调用
  - [ ] 阅读 `durable_exec/dbos/_mcp.py`，理解 DBOS + MCP 的组合

- [ ] **7.2 Temporal 集成**
  - [ ] 阅读 `durable_exec/temporal/_workflow.py`，理解 Temporal Workflow 映射
  - [ ] 阅读 `durable_exec/temporal/_agent.py`，理解 Temporal Activity 包装
  - [ ] 对比 DBOS 和 Temporal 在 API 设计上的差异

- [ ] **7.3 Prefect 集成**
  - [ ] 阅读 `durable_exec/prefect/` 目录，理解 Prefect Flow 集成模式

---

## 八、MCP（Model Context Protocol）集成

> **生态扩展**：通过 MCP 接入外部工具和模型

- [ ] **8.1 MCP 核心**
  - [ ] 阅读 `_mcp.py`，理解 MCP 连接管理的核心抽象
  - [ ] 阅读 `mcp.py`，理解面向用户的 MCP API

- [ ] **8.2 MCP Toolset**
  - [ ] 阅读 `toolsets/fastmcp.py`，理解 FastMCP 工具集集成
  - [ ] 理解 MCP 工具如何被转换为 pydantic-ai 的 `ToolDefinition`

- [ ] **8.3 MCP Sampling（反向）**
  - [ ] 阅读 `models/mcp_sampling.py`，理解 pydantic-ai 作为 MCP 的 Sampling 后端
  - [ ] 理解双向 MCP（作为客户端 + 作为服务端）的设计意图

---

## 九、可观测性与追踪

> **生产级监控**：OpenTelemetry 原生支持，Logfire 深度集成

- [ ] **9.1 OpenTelemetry 集成**
  - [ ] 阅读 `_instrumentation.py`，理解 OTel Tracer 的初始化和注入方式
  - [ ] 阅读 `_otel_messages.py`，理解消息到 OTel Span 属性的映射
  - [ ] 理解 `models/instrumented.py` 如何无侵入地包装任意 Model

- [ ] **9.2 Logfire 集成**
  - [ ] 搜索代码中对 `logfire` 的引用，理解与 Logfire SDK 的集成点
  - [ ] 理解 `durable_exec/temporal/_logfire.py` 的特殊处理

- [ ] **9.3 Instrumentation Settings**
  - [ ] 理解 `InstrumentationSettings` 如何控制追踪粒度（包含/排除哪些数据）

---

## 十、评估框架 pydantic_evals

> **质量保证**：系统化评估 LLM 应用质量的完整框架

- [ ] **10.1 数据集与评估器**
  - [ ] 阅读 `pydantic_evals/dataset.py`，理解评估数据集的组织
  - [ ] 阅读 `pydantic_evals/evaluators/evaluator.py`，理解评估器接口
  - [ ] 阅读 `pydantic_evals/evaluators/llm_as_a_judge.py`，理解 LLM-as-a-Judge 实现

- [ ] **10.2 评估生命周期**
  - [ ] 阅读 `pydantic_evals/lifecycle.py`，理解评估的完整生命周期
  - [ ] 阅读 `pydantic_evals/generation.py`，理解测试数据生成
  - [ ] 阅读 `pydantic_evals/online.py`，理解在线评估（生产环境采样）

- [ ] **10.3 报告系统**
  - [ ] 阅读 `pydantic_evals/reporting/analyses.py`，理解统计分析
  - [ ] 阅读 `pydantic_evals/reporting/render_numbers.py`，理解数字可视化
  - [ ] 阅读 `pydantic_evals/otel/` 目录，理解评估结果的 OTel 上报

---

## 十一、UI 接口与流式协议

> **前端集成**：支持主流 AI UI 框架的流式协议

- [ ] **11.1 AG-UI 协议**
  - [ ] 阅读 `ui/ag_ui/app.py`，理解 AG-UI 服务器端点实现
  - [ ] 阅读 `ui/ag_ui/_event_stream.py`，理解事件流格式
  - [ ] 阅读 `ui/ag_ui/_adapter.py`，理解 pydantic-ai 消息到 AG-UI 事件的转换

- [ ] **11.2 Vercel AI SDK 集成**
  - [ ] 阅读 `ui/vercel_ai/response_types.py`，理解 Vercel AI 响应格式
  - [ ] 阅读 `ui/vercel_ai/_event_stream.py`，理解 Vercel AI 流式格式
  - [ ] 理解 `ui/vercel_ai/request_types.py` 中的请求模型

- [ ] **11.3 A2A 协议（Agent-to-Agent）**
  - [ ] 阅读 `_a2a.py`，理解 A2A 协议支持，理解多 Agent 通信机制
  - [ ] 阅读 `ag_ui.py`，理解顶层 AG-UI 接口

---

## 十二、嵌入模型系统

- [ ] **12.1 嵌入抽象**
  - [ ] 阅读 `embeddings/base.py`，理解 `EmbeddingModel` 接口
  - [ ] 阅读 `embeddings/settings.py`，理解嵌入配置
  - [ ] 阅读 `embeddings/result.py`，理解嵌入结果结构

- [ ] **12.2 嵌入实现**
  - [ ] 阅读 `embeddings/openai.py`，作为参考实现
  - [ ] 阅读 `embeddings/sentence_transformers.py`，理解本地嵌入支持
  - [ ] 阅读 `embeddings/instrumented.py`，理解 OTel 追踪包装

---

## 十三、CLI 与开发者工具

- [ ] **13.1 CLI 工具**
  - [ ] 阅读 `_cli/` 目录，理解 `pydantic-ai` 命令行工具功能
  - [ ] 阅读 `_cli/web.py`，理解 CLI 的 Web UI

- [ ] **13.2 直接调用接口**
  - [ ] 阅读 `direct.py`，理解不通过 Agent 直接调用模型的 API

- [ ] **13.3 内置工具**
  - [ ] 阅读 `builtin_tools.py`，理解内置工具（如图像生成）的设计
  - [ ] 阅读 `common_tools/` 目录（tavily.py、exa.py、web_fetch.py 等），理解开箱即用的工具

---

## 十四、指令与模板系统

- [ ] **14.1 System Prompt 管理**
  - [ ] 阅读 `_system_prompt.py`，理解系统提示的组装和动态注入
  - [ ] 阅读 `_instructions.py`，理解指令系统

- [ ] **14.2 模板引擎**
  - [ ] 阅读 `_template.py`，理解提示模板的变量替换机制
  - [ ] 阅读 `format_prompt.py`，理解提示格式化工具

---

## 十五、安全与基础设施

- [ ] **15.1 安全防护**
  - [ ] 阅读 `_ssrf.py`，理解 SSRF（服务端请求伪造）防护机制

- [ ] **15.2 通用工具**
  - [ ] 阅读 `_utils.py`，理解项目内部公用工具函数
  - [ ] 阅读 `_uuid.py`，理解 UUID v7 的生成和使用（时序排序）
  - [ ] 阅读 `exceptions.py`，梳理完整的异常体系
  - [ ] 阅读 `usage.py`，理解 Token 用量统计和汇总机制
  - [ ] 阅读 `settings.py`，理解 `ModelSettings` 的配置项

---

## 探索优先级建议

```
★★★ 必读（理解核心执行流）
  → _agent_graph.py
  → messages.py
  → models/__init__.py
  → tools.py + _tool_manager.py
  → toolsets/abstract.py

★★  重要（理解系统设计）
  → output.py + result.py
  → agent/abstract.py
  → pydantic_graph/graph.py + nodes.py
  → _instrumentation.py

★   扩展（按需深入）
  → 各模型实现（openai.py 为首选）
  → durable_exec/（持久化执行）
  → pydantic_evals/（评估框架）
  → ui/（前端协议）
```
