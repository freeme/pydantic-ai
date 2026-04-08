# 五、Provider 与 Profiles 系统

> **核心文件**：`providers/__init__.py`、`providers/openai.py`、`providers/anthropic.py`、`profiles/__init__.py`、`profiles/openai.py`、`profiles/anthropic.py`

---

## 5.1 三层分离设计

pydantic-ai 使用三层分离来管理 LLM 的配置：

```
┌──────────────────────────────────────────────────────────────┐
│                    用户配置层                                 │
│                                                              │
│  Agent('openai:gpt-5.2', ...)                               │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐    │
│  │   Provider   │   │    Model     │   │   Profile    │    │
│  │  （认证/连接）│   │  （API调用）  │   │ （能力声明） │    │
│  │              │   │              │   │              │    │
│  │ 负责：       │   │ 负责：       │   │ 负责：       │    │
│  │ - API Key    │   │ - 消息格式   │   │ - 支持工具   │    │
│  │ - base_url  │   │ - 错误处理   │   │ - 输出模式   │    │
│  │ - HTTP Client│   │ - 流式处理   │   │ - Thinking   │    │
│  │ - 认证方式   │   │ - SDK 适配   │   │ - JSON Schema│    │
│  └──────────────┘   └──────────────┘   └──────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

**关键点**：
- **Provider** 解决"如何连接"（认证、URL）
- **Model** 解决"如何调用"（API 格式、消息转换）
- **Profile** 解决"模型能做什么"（能力声明）

---

## 5.2 Provider 抽象基类

**文件**：`providers/__init__.py`

```python
class Provider(ABC, Generic[InterfaceClient]):
    """提供认证后的 API 客户端。"""

    @property
    @abstractmethod
    def name(self) -> str: ...            # 提供商标识（'openai', 'anthropic'）

    @property
    @abstractmethod
    def base_url(self) -> str: ...        # API 基础 URL

    @property
    @abstractmethod
    def client(self) -> InterfaceClient:  # SDK 客户端实例
        ...

    @staticmethod
    def model_profile(model_name: str) -> ModelProfile | None:
        """根据模型名返回对应的 ModelProfile（可选实现）。"""
        return None
```

Provider 的泛型参数 `InterfaceClient` 是 SDK 客户端类型（如 `AsyncOpenAI`、`AsyncAnthropic`）。

---

## 5.3 OpenAIProvider — 参考实现

```python
class OpenAIProvider(Provider[AsyncOpenAI]):

    def __init__(
        self,
        base_url: str | None = None,   # 自定义 API URL（兼容 OpenAI 格式的服务）
        api_key: str | None = None,    # 从环境变量 OPENAI_API_KEY 读取
        openai_client: AsyncOpenAI | None = None,  # 直接传入已有客户端
        http_client: httpx.AsyncClient | None = None,  # 自定义 HTTP 客户端
    ) -> None: ...

    @staticmethod
    def model_profile(model_name: str) -> ModelProfile | None:
        return openai_model_profile(model_name)
```

**特殊处理**：如果没有 API Key 但指定了 base_url，则自动使用 `'api-key-not-set'` 占位（本地服务如 Ollama 不需要 API Key）。

**共享 HTTP 客户端**：通过 `cached_async_http_client(provider='openai')` 在同一进程内复用 HTTP 连接池。

---

## 5.4 AnthropicProvider — 多客户端类型支持

```python
AsyncAnthropicClient = (
    AsyncAnthropic
    | AsyncAnthropicBedrock    # AWS Bedrock 上的 Claude
    | AsyncAnthropicFoundry    # Microsoft Azure AI Foundry
    | AsyncAnthropicVertex     # Google Vertex AI 上的 Claude
)

class AnthropicProvider(Provider[AsyncAnthropicClient]):
    """支持 4 种不同部署环境的 Anthropic 客户端。"""
```

Anthropic Provider 的 `model_profile` 返回同时包含 `AnthropicModelProfile`（子类）和基础 `ModelProfile` 合并后的结果，包含 Anthropic 特有的 `json_schema_transformer`。

---

## 5.5 LiteLLMProvider — 统一代理

**文件**：`providers/litellm.py`

```python
class LiteLLMProvider(Provider[AsyncOpenAI]):
    """通过 LiteLLM 统一接入多种模型。"""

    @staticmethod
    def model_profile(model_name: str) -> ModelProfile | None:
        # 解析 'provider/model-name' 格式，路由到对应的 profile 函数
        provider_to_profile = {
            'anthropic': anthropic_model_profile,
            'openai': openai_model_profile,
            'google': google_model_profile,
            'mistral': mistral_model_profile,
            # ...
        }
```

LiteLLM 使用 OpenAI 兼容的 API 格式，`model_name` 采用 `'provider/model-name'` 格式（如 `'anthropic/claude-opus-4'`）。

---

## 5.6 AzureProvider — 企业部署

**文件**：`providers/azure.py`

```python
class AzureProvider(Provider[AsyncOpenAI]):
    """Azure OpenAI 和 Azure AI Foundry 的 Provider。"""
```

Azure 使用 `AsyncAzureOpenAI` 客户端，但实现了 OpenAI 兼容的接口，因此泛型参数仍为 `AsyncOpenAI`（使用结构类型兼容）。

Azure Provider 的 `model_profile` 包含对多种第三方模型（Llama、DeepSeek、Mistral）的 profile 映射，因为 Azure AI Foundry 支持这些第三方模型。

---

## 5.7 infer_provider_class — Provider 路由

```python
def infer_provider_class(provider: str) -> type[Provider[Any]]:
    """根据 provider 名称字符串找到对应的 Provider 类。

    支持的 provider 名称：
    'openai', 'anthropic', 'google-gla', 'google-vertex',
    'bedrock', 'azure', 'groq', 'mistral', 'xai', 'ollama',
    'deepseek', 'openrouter', 'fireworks', 'together', 'litellm', ...
    """
```

**别名处理**：
- `'vertexai'` → `'google-vertex'`（向后兼容）
- `'google'` → `'google-gla'`
- `'gateway/openai'` → `'openai'`（Gateway 前缀剥离）

---

## 5.8 ModelProfile — 模型能力声明（详细）

**文件**：`profiles/__init__.py`

`ModelProfile` 是一个 `dataclass`，所有字段都有默认值（`DEFAULT_PROFILE` 即全默认）：

```python
@dataclass(kw_only=True)
class ModelProfile:
    # 工具相关
    supports_tools: bool = True                    # 支持工具调用
    supports_json_schema_output: bool = False       # 支持原生 JSON Schema 输出
    supports_json_object_output: bool = False       # 支持 JSON 对象模式
    supports_image_output: bool = False             # 支持图像生成输出

    # 输出模式
    default_structured_output_mode: StructuredOutputMode = 'tool'
    prompted_output_template: str = "..."           # 提示模板 {schema} 占位符
    native_output_requires_schema_in_instructions: bool = False

    # JSON Schema 转换器（模型特有 Schema 限制）
    json_schema_transformer: type[JsonSchemaTransformer] | None = None

    # Thinking/Reasoning
    supports_thinking: bool = False                 # 支持配置 thinking
    thinking_always_enabled: bool = False           # 总是使用 thinking（o系列、R1）
    thinking_tags: tuple[str, str] = ('<think>', '</think>')

    # 其他
    ignore_streamed_leading_whitespace: bool = False  # 忽略流式前导空白
    supported_builtin_tools: frozenset[...]          # 支持的内置工具类型
```

### Profile 更新机制（update 方法）

```python
def update(self, profile: ModelProfile | None) -> Self:
    """用另一个 Profile 的非默认值覆盖当前 Profile。"""
    non_default_attrs = {
        f.name: getattr(profile, f.name)
        for f in fields(profile)
        if getattr(profile, f.name) != f.default  # 只覆盖非默认值
    }
    return replace(self, **non_default_attrs)
```

这允许"继承式"的 Profile 组合：子类 Profile 可以基于父类修改特定字段，而不是从头定义。

---

## 5.9 各厂商 Profile 详解

### OpenAIModelProfile

**文件**：`profiles/openai.py`

```python
@dataclass(kw_only=True)
class OpenAIModelProfile(ModelProfile):
    # Reasoning 参数映射
    openai_reasoning_effort_map: dict[ThinkingLevel, str] = field(
        default_factory=lambda: OPENAI_REASONING_EFFORT_MAP
    )
    # 系统提示角色（'system'|'developer'|'user'）
    system_prompt_role: OpenAISystemPromptRole = 'system'
    # 工具选择策略
    openai_responses_tool_choice: ...
    # Thinking 内容字段名（非标准扩展）
    openai_chat_thinking_field: str | None = None
    openai_chat_send_back_thinking_parts: Literal['auto', 'tags', 'field', False] = 'auto'
```

`openai_model_profile(model_name)` 函数根据模型名称返回对应配置：
- o 系列（`o1`, `o3`, `o4`）：`thinking_always_enabled=True`
- 其他：`supports_json_schema_output=True`

**`OpenAIJsonSchemaTransformer`**：处理 OpenAI 的 Schema 限制（如移除 `additionalProperties: false` 不兼容项）。

### AnthropicModelProfile

**文件**：`profiles/anthropic.py`

```python
@dataclass(kw_only=True)
class AnthropicModelProfile(ModelProfile):
    anthropic_supports_adaptive_thinking: bool = False  # Sonnet 4.6+ 自适应 thinking
    anthropic_supports_effort: bool = False              # Opus 4.5+ effort 参数
```

Anthropic 的 thinking 有两种实现：
- 旧版（`budget_tokens`）：`anthropic_supports_adaptive_thinking=False`
- 新版 Adaptive（Sonnet 4.6+）：`anthropic_supports_adaptive_thinking=True`

**Thinking Budget 映射**：
```python
ANTHROPIC_THINKING_BUDGET_MAP: dict[ThinkingLevel, int] = {
    True: 10000,
    'minimal': 1024,
    'low': 2048,
    'medium': 10000,
    'high': 16384,
    'xhigh': 32768,
}
```

---

## 5.10 JSON Schema 转换器

模型对 JSON Schema 的支持各有差异，`json_schema_transformer` 在工具定义发送前自动转换：

```python
class JsonSchemaTransformer:
    """将 JSON Schema 转换为模型兼容格式。"""
    def transform_function_tool(self, schema: JsonSchema) -> JsonSchema: ...
    def transform_output(self, schema: JsonSchema) -> JsonSchema: ...
```

**`OpenAIJsonSchemaTransformer`**：
- 移除 OpenAI 严格模式不支持的关键字
- 处理 `$defs` 内联展开

**`AnthropicJsonSchemaTransformer`**（在 `providers/anthropic.py` 中定义）：
- Anthropic 特有的 Schema 限制处理
- 工具参数格式适配

---

## 5.11 ThinkingLevel — 统一的 Thinking 配置

**文件**：`settings.py`

```python
ThinkingLevel = bool | Literal['minimal', 'low', 'medium', 'high', 'xhigh']
```

各厂商的 Thinking 参数差异很大，pydantic-ai 通过 `ThinkingLevel` 统一抽象：

| ThinkingLevel | OpenAI | Anthropic（旧） | Anthropic（自适应） |
|---------------|--------|----------------|---------------------|
| `True` | `reasoning_effort='medium'` | `budget_tokens=10000` | `{'type': 'adaptive'}` |
| `'high'` | `reasoning_effort='high'` | `budget_tokens=16384` | `effort='high'` |
| `'xhigh'` | `reasoning_effort='xhigh'` | `budget_tokens=32768` | — |
| `False` | `reasoning_effort='none'` | 禁用 | — |

---

## 5.12 环境变量约定

各 Provider 的 API Key 约定：

| Provider | 环境变量 |
|----------|---------|
| OpenAI | `OPENAI_API_KEY` |
| Anthropic | `ANTHROPIC_API_KEY` |
| Google | `GEMINI_API_KEY` / `GOOGLE_API_KEY` |
| Groq | `GROQ_API_KEY` |
| Mistral | `MISTRAL_API_KEY` |
| DeepSeek | `DEEPSEEK_API_KEY` |
| XAI | `XAI_API_KEY` |
| Bedrock | AWS 标准凭证（`AWS_ACCESS_KEY_ID` 等） |

---

## 总结

```
设计亮点
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Provider / Model / Profile 三层分离：
   每层职责清晰，可以独立变化：
   - 同一个 Model（如 OpenAIChatModel）可以通过不同 Provider 使用
     （OpenAI 官方、Azure、LiteLLM）
   - 同一个 Provider 可以服务不同的 Model（Chat vs Responses API）

2. Profile 的 update 机制：
   基于字段默认值的合并，允许子类 Profile 精确覆盖
   而不是全量重定义，支持"Profile 继承"

3. JSON Schema Transformer：
   各模型对 JSON Schema 的支持不同，通过 Transformer
   在 prepare_request 阶段自动修正，对用户透明

4. 统一 ThinkingLevel：
   跨越 OpenAI/Anthropic/Google 等不同 thinking API 的差异
   用户只需设置一个 level，框架映射到对应的 API 参数

5. 共享 HTTP 连接池：
   cached_async_http_client 确保同一进程内的多个模型实例
   共享 HTTP 连接池，避免连接数膨胀
```
