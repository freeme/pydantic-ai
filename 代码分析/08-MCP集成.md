# 八、MCP（Model Context Protocol）集成

> **核心文件**：`mcp.py`（公开 API）、`_mcp.py`（消息格式转换）、`toolsets/fastmcp.py`、`models/mcp_sampling.py`

---

## 8.1 MCP 双向集成概述

pydantic-ai 对 MCP 协议有两个方向的集成：

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP 双向集成                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  方向一：pydantic-ai 作为 MCP 客户端（消费工具）             │
│  ┌──────────────┐       MCP        ┌───────────────────┐   │
│  │ pydantic-ai  │ ──call_tool()──▶ │  MCP Server       │   │
│  │  (client)    │ ◀──tool_result── │  (工具提供方)      │   │
│  └──────────────┘                  └───────────────────┘   │
│                                                             │
│  方向二：pydantic-ai 作为 MCP 服务端（提供 Sampling）        │
│  ┌──────────────┐       MCP        ┌───────────────────┐   │
│  │ pydantic-ai  │ ◀─create_msg()── │  MCP Client       │   │
│  │  (sampling)  │ ──model_result──▶│  (请求 LLM)       │   │
│  └──────────────┘                  └───────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 8.2 MCPServer — MCP 客户端（方向一）

**文件**：`mcp.py`

`MCPServer` 系列类将 MCP 服务端的工具暴露为 pydantic-ai 的 Toolset：

```python
# 基类
class MCPServer(AbstractToolset, ABC):
    """Abstract base class for MCP server integrations."""

# 三种传输方式
class MCPServerStdio(MCPServer):
    """通过 stdio 连接本地进程（如 CLI 工具）。"""
    command: str
    args: list[str] = []
    env: dict[str, str] | None = None

class MCPServerHTTP(MCPServer):
    """通过 HTTP 连接 MCP 服务（Streamable HTTP）。"""
    url: str
    headers: dict[str, str] | None = None

class MCPServerSSE(MCPServer):
    """通过 Server-Sent Events 连接 MCP 服务（兼容旧版）。"""
    url: str
```

### 连接生命周期

```
MCPServer.__aenter__()
  ├── 创建 MCP 客户端会话
  ├── 建立传输连接（stdio/http/sse）
  └── 初始化：列出服务端工具

for_run(ctx) → 为每次 Agent 运行返回新实例（或共享）

get_tools(ctx) → 获取 MCP 服务端工具列表
  ├── client.list_tools() → [mcp.types.Tool]
  └── 转换为 ToolsetTool（包含 JSON Schema）

call_tool(name, args, ctx, tool) → 执行工具
  ├── client.call_tool(name, args) → [mcp.types.Content]
  └── 将 MCP 内容格式转换为 pydantic-ai 格式

MCPServer.__aexit__()
  └── 关闭 MCP 连接
```

### Sampling 反向回调

MCPServer 在连接初始化时注册 Sampling 处理器，使 MCP 服务端可以通过 Agent 调用 LLM：

```python
async def _sampling_handler(
    params: mcp_types.CreateMessageRequestParams,
    session: ServerSession,
    agent_context: dict,
) -> mcp_types.CreateMessageResult:
    # 将 MCP 采样请求转换为 pydantic-ai 模型调用
    messages = _mcp.map_from_mcp_params(params)
    response = await model_request(model, messages, ...)
    return _to_mcp_result(response)
```

---

## 8.3 FastMCPToolset — FastMCP 客户端集成

**文件**：`toolsets/fastmcp.py`

`FastMCPToolset` 是基于 [FastMCP](https://github.com/jlowin/fastmcp) 客户端的工具集实现，支持更灵活的 MCP 客户端连接方式：

```python
class FastMCPToolset(AbstractToolset):
    client: Client[Any]  # FastMCP Client

    def __init__(
        self,
        client: Client | ClientTransport | FastMCP | AnyUrl | Path | MCPConfig | str,
        *,
        max_retries: int = 1,
        tool_error_behavior: Literal['model_retry', 'error'] = 'model_retry',
        include_instructions: bool = False,
        id: str | None = None,
    ):
```

**支持的连接类型**：
- `FastMCP` / `FastMCP1Server`：本地内存连接（测试友好）
- `AnyUrl`：远程 HTTP 地址
- `Path`：本地脚本路径（stdio 连接）
- `MCPConfig`：MCP 配置对象
- `str`：URL 字符串或命令行

**引用计数生命周期**：多个 Agent 共享同一 FastMCPToolset 时，通过引用计数 `_running_count` 管理连接：

```python
async def __aenter__(self) -> Self:
    async with self._enter_lock:
        if self._running_count == 0:
            # 首次 enter 才真正建立连接
            self._exit_stack = AsyncExitStack()
            await self._exit_stack.enter_async_context(self.client)
        self._running_count += 1
    return self

async def __aexit__(self, *args) -> bool | None:
    async with self._enter_lock:
        self._running_count -= 1
        if self._running_count == 0:
            # 最后一个 exit 才真正断开连接
            await self._exit_stack.aclose()
```

---

## 8.4 消息格式转换（_mcp.py）

MCP 和 pydantic-ai 各有自己的消息格式，`_mcp.py` 负责双向转换：

### pydantic-ai → MCP（用于 Sampling）

```python
def map_from_pai_messages(pai_messages: list[ModelMessage]) -> tuple[str, list[SamplingMessage]]:
    """将 pydantic-ai 消息历史转换为 MCP Sampling 格式。
    
    Returns: (system_prompt, sampling_messages)
    """
    # SystemPromptPart + InstructionPart → system_prompt（合并为字符串）
    # UserPromptPart → SamplingMessage(role='user', content=TextContent)
    # ModelResponse.TextPart → SamplingMessage(role='assistant', content=TextContent)
```

### MCP → pydantic-ai（用于 MCP Sampling Model）

```python
def map_from_mcp_params(params: CreateMessageRequestParams) -> list[ModelMessage]:
    """将 MCP create_message 请求转换为 pydantic-ai 消息列表。"""
    # systemPrompt → SystemPromptPart
    # user messages → UserPromptPart
    # assistant messages → ModelResponse.TextPart
```

---

## 8.5 MCPSamplingModel — pydantic-ai 作为 MCP Sampling 服务端（方向二）

**文件**：`models/mcp_sampling.py`

`MCPSamplingModel` 实现了 `Model` 接口，但不直接调用外部 API，而是通过 MCP Sampling 协议将请求委托给 MCP 客户端：

```python
@dataclass
class MCPSamplingModel(Model):
    """通过 MCP Sampling 协议调用 LLM。"""
    session: ServerSession  # MCP 服务器会话

    async def request(
        self,
        messages: list[ModelMessage],
        model_settings: ModelSettings | None,
        model_request_parameters: ModelRequestParameters,
    ) -> ModelResponse:
        # 1. 将 pydantic-ai 消息转换为 MCP 格式
        system_prompt, sampling_messages = _mcp.map_from_pai_messages(messages)

        # 2. 通过 MCP session 请求采样（MCP 客户端负责实际 LLM 调用）
        result = await self.session.create_message(
            sampling_messages,
            max_tokens=...,
            system_prompt=system_prompt,
            model_preferences=...,
        )

        # 3. 将 MCP 结果转换回 ModelResponse
        return ModelResponse(parts=[_mcp.map_from_sampling_content(result.content)])
```

**使用场景**：当 pydantic-ai 嵌入到另一个 AI 系统（作为 MCP 服务端）时，可以通过 MCP Sampling 让上层系统提供 LLM 能力，而无需 pydantic-ai 自己持有 API Key。

---

## 8.6 TOOL_SCHEMA_VALIDATOR — 工具 Schema 验证

MCP 协议中，工具的参数 Schema 格式与 pydantic-ai 的 `ToolDefinition` 格式需要互相转换：

```python
# 在 FastMCPToolset 中
from pydantic_ai.mcp import TOOL_SCHEMA_VALIDATOR

# 验证 MCP 工具的 inputSchema 是否符合 pydantic-ai 期望的格式
validated_schema = TOOL_SCHEMA_VALIDATOR.validate_python(mcp_tool.inputSchema)
```

---

## 8.7 工具内容转换

MCP 工具返回的内容（`mcp.types.Content`）被转换为 pydantic-ai 的 `ToolReturn` 格式：

```
MCP Content 类型              →  pydantic-ai 格式
────────────────────────────────────────────────
TextContent(type='text')      →  str
ImageContent(type='image')    →  BinaryContent(is_image=True)
BlobResourceContents          →  BinaryContent
TextResourceContents          →  str
EmbeddedResource              →  取决于内部类型
ResourceLink                  →  dict（资源链接信息）
```

---

## 8.8 elicitation — MCP 人工交互

pydantic-ai 支持 MCP 的 Elicitation 功能（服务端请求用户输入）：

```python
class MCPServer:
    elicitation_handler: ElicitationFnT | None = None
    """处理 MCP 服务端发起的用户交互请求。"""
```

---

## 8.9 MCPError — 统一错误处理

```python
class MCPError(RuntimeError):
    message: str
    code: int
    data: dict[str, Any] | None

    @classmethod
    def from_mcp_sdk(cls, error: mcp_exceptions.McpError) -> MCPError:
        """将 MCP SDK 的错误转换为 pydantic-ai 的错误格式。"""
```

工具调用错误处理：
- `tool_error_behavior='model_retry'`（默认）：`ToolError` → `ModelRetry`（工具重试）
- `tool_error_behavior='error'`：`ToolError` → 直接抛出异常

---

## 总结

```
设计亮点
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 双向集成：
   - 客户端方向：MCP 服务器工具 → pydantic-ai 工具
   - 服务端方向：pydantic-ai → MCP Sampling 后端

2. 三种传输协议：
   stdio（本地进程）、HTTP/Streamable HTTP、SSE
   统一的 MCPServer 接口屏蔽传输差异

3. FastMCPToolset 的引用计数：
   多个 Agent 共享同一 MCP 服务连接
   避免重复建立连接的开销

4. 格式转换隔离（_mcp.py）：
   消息格式转换逻辑集中在一处
   MCP ↔ pydantic-ai 消息互转

5. 错误可配置：
   tool_error_behavior 控制 MCP 工具错误是触发重试还是报错
   对 LLM 友好（默认 model_retry）vs. 严格错误传播
```
