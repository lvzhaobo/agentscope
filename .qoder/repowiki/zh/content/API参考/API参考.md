# API参考

<cite>
**本文档中引用的文件**  
- [__init__.py](file://src/agentscope/__init__.py)
- [_version.py](file://src/agentscope/_version.py)
- [_run_config.py](file://src/agentscope/_run_config.py)
- [_logging.py](file://src/agentscope/_logging.py)
- [agent\__init__.py](file://src/agentscope/agent/__init__.py)
- [model\__init__.py](file://src/agentscope/model/__init__.py)
- [tool\__init__.py](file://src/agentscope/tool/__init__.py)
- [memory\__init__.py](file://src/agentscope/memory/__init__.py)
- [pipeline\__init__.py](file://src/agentscope/pipeline/__init__.py)
- [message\__init__.py](file://src/agentscope/message/__init__.py)
- [session\__init__.py](file://src/agentscope/session/__init__.py)
- [embedding\__init__.py](file://src/agentscope/embedding/__init__.py)
- [token\__init__.py](file://src/agentscope/token/__init__.py)
- [evaluate\__init__.py](file://src/agentscope/evaluate/__init__.py)
- [formatter\__init__.py](file://src/agentscope/formatter/__init__.py)
- [rag\__init__.py](file://src/agentscope/rag/__init__.py)
- [a2a\__init__.py](file://src/agentscope/a2a/__init__.py)
- [tracing\__init__.py](file://src/agentscope/tracing/__init__.py)
</cite>

## 目录
1. [简介](#简介)
2. [核心模块](#核心模块)
3. [Agent模块](#agent模块)
4. [模型模块](#模型模块)
5. [工具模块](#工具模块)
6. [记忆模块](#记忆模块)
7. [消息模块](#消息模块)
8. [会话模块](#会话模块)
9. [嵌入模块](#嵌入模块)
10. [令牌模块](#令牌模块)
11. [评估模块](#评估模块)
12. [流水线模块](#流水线模块)
13. [格式化模块](#格式化模块)
14. [RAG模块](#rag模块)
15. [A2A模块](#a2a模块)
16. [追踪模块](#追踪模块)
17. [配置与环境变量](#配置与环境变量)
18. [版本兼容性](#版本兼容性)

## 简介

AgentScope是一个多智能体系统框架，提供了一套完整的API来构建和管理智能体应用。该框架支持多种大语言模型、工具集成、记忆管理、消息传递和评估功能。通过`init()`函数进行初始化，可以配置项目名称、运行标识、日志路径等参数。

**Section sources**
- [__init__.py](file://src/agentscope/__init__.py#L72-L157)

## 核心模块

### 初始化函数

#### `init()`
初始化AgentScope库。

**函数签名**
```python
def init(
    project: str | None = None,
    name: str | None = None,
    run_id: str | None = None,
    logging_path: str | None = None,
    logging_level: str = "INFO",
    studio_url: str | None = None,
    tracing_url: str | None = None,
) -> None:
```

**参数说明**
- **project**: 项目名称
- **name**: 运行实例的名称
- **run_id**: 运行实例的唯一标识符
- **logging_path**: 日志文件保存路径
- **logging_level**: 日志级别（"INFO", "DEBUG", "WARNING", "ERROR", "CRITICAL"）
- **studio_url**: AgentScope Studio的URL
- **tracing_url**: 追踪端点的URL

**返回值**
无返回值

**异常情况**
- 当`logging_level`参数无效时抛出`ValueError`

**使用示例**
```python
from agentscope import init

init(
    project="my_project",
    name="my_run",
    logging_path="./logs",
    logging_level="INFO"
)
```

### 版本信息

#### `__version__`
获取当前AgentScope库的版本号。

**返回值**
返回字符串格式的版本号，当前版本为"1.0.10"

**使用示例**
```python
from agentscope import __version__
print(__version__)
```

**Section sources**
- [__init__.py](file://src/agentscope/__init__.py#L66)
- [_version.py](file://src/agentscope/_version.py#L4)

## Agent模块

Agent模块提供了智能体的核心功能，包括基础智能体类、ReAct智能体、用户智能体等。

### 类定义

#### `AgentBase`
智能体基类，所有智能体都应继承此类。

**属性**
- `name`: 智能体名称
- `sys_prompt`: 系统提示词

**方法**
- `reply(msg: Msg) -> Msg`: 处理消息并返回响应

#### `ReActAgentBase`
ReAct模式智能体基类，支持思考-行动循环。

**继承自**: `AgentBase`

**方法**
- `think()`: 生成思考过程
- `act()`: 执行动作
- `observe(msg: Msg)`: 观察环境

#### `ReActAgent`
具体的ReAct智能体实现。

**参数**
- `name`: 智能体名称
- `model_config_name`: 模型配置名称
- `sys_prompt`: 系统提示词
- `tool_set`: 工具集

#### `UserAgent`
用户智能体，用于与用户交互。

**方法**
- `override_class_input_method(input_handler)`: 重写输入方法

#### `A2AAgent`
A2A（Agent-to-Agent）智能体，用于智能体间通信。

### 用户输入类

#### `UserInputBase`
用户输入基类。

**方法**
- `get(prompt: str) -> UserInputData`: 获取用户输入

#### `TerminalUserInput`
终端用户输入实现。

#### `StudioUserInput`
AgentScope Studio用户输入实现。

**参数**
- `studio_url`: Studio URL
- `run_id`: 运行ID
- `max_retries`: 最大重试次数

**Section sources**
- [agent\__init__.py](file://src/agentscope/agent/__init__.py#L3-L26)

## 模型模块

模型模块提供了与各种大语言模型的集成。

### 基础类

#### `ChatModelBase`
聊天模型基类。

**方法**
- `chat(messages: List[Msg], **kwargs) -> ChatResponse`: 发送消息并获取响应

### 具体模型实现

#### `DashScopeChatModel`
阿里云通义千问模型。

**参数**
- `model_config_name`: 模型配置名称
- `api_key`: API密钥
- `generate_args`: 生成参数

#### `OpenAIChatModel`
OpenAI模型。

**参数**
- `model_config_name`: 模型配置名称
- `api_key`: API密钥
- `organization`: 组织ID
- `generate_args`: 生成参数

#### `AnthropicChatModel`
Anthropic模型。

#### `OllamaChatModel`
Ollama本地模型。

#### `GeminiChatModel`
Google Gemini模型。

#### `TrinityChatModel`
Trinity模型。

### 响应类

#### `ChatResponse`
聊天响应对象。

**属性**
- `text`: 响应文本
- `metadata`: 元数据
- `usage`: 使用统计

**Section sources**
- [model\__init__.py](file://src/agentscope/model/__init__.py#L3-L22)

## 工具模块

工具模块提供了各种实用工具函数和工具集。

### 工具函数

#### `execute_python_code(code: str) -> ToolResponse`
执行Python代码。

**参数**
- `code`: 要执行的Python代码

**返回值**
`ToolResponse`对象

#### `execute_shell_command(command: str) -> ToolResponse`
执行shell命令。

**参数**
- `command`: 要执行的shell命令

#### `view_text_file(path: str) -> ToolResponse`
查看文本文件内容。

**参数**
- `path`: 文件路径

#### `write_text_file(path: str, content: str) -> ToolResponse`
写入文本文件。

**参数**
- `path`: 文件路径
- `content`: 要写入的内容

#### `insert_text_file(path: str, content: str, position: int = 0) -> ToolResponse`
在指定位置插入文本。

**参数**
- `path`: 文件路径
- `content`: 要插入的内容
- `position`: 插入位置

### 多模态工具

#### `dashscope_text_to_image(prompt: str, **kwargs) -> ToolResponse`
通义万相文生图。

**参数**
- `prompt`: 文本提示
- 其他参数参考DashScope API

#### `dashscope_text_to_audio(text: str, **kwargs) -> ToolResponse`
通义听悟文本转语音。

#### `dashscope_image_to_text(image: str, **kwargs) -> ToolResponse`
通义千问视觉理解。

#### `openai_text_to_image(prompt: str, **kwargs) -> ToolResponse`
OpenAI DALL-E文生图。

#### `openai_text_to_audio(text: str, **kwargs) -> ToolResponse`
OpenAI文本转语音。

#### `openai_edit_image(image: str, mask: str, prompt: str, **kwargs) -> ToolResponse`
OpenAI图像编辑。

#### `openai_create_image_variation(image: str, **kwargs) -> ToolResponse`
OpenAI图像变体生成。

#### `openai_image_to_text(image: str, **kwargs) -> ToolResponse`
OpenAI视觉理解。

#### `openai_audio_to_text(audio: str, **kwargs) -> ToolResponse`
OpenAI语音转文本。

### 工具集

#### `Toolkit`
工具集类，用于管理和注册工具。

**方法**
- `register_tool(tool_func)`: 注册工具函数
- `get_tool(tool_name)`: 获取工具函数

**Section sources**
- [tool\__init__.py](file://src/agentscope/tool/__init__.py#L3-L44)

## 记忆模块

记忆模块提供了短期和长期记忆功能。

### 基础类

#### `MemoryBase`
记忆基类。

**方法**
- `add(msg: Msg)`: 添加消息
- `get(**kwargs) -> List[Msg]`: 获取消息
- `clear()`: 清除记忆

#### `LongTermMemoryBase`
长期记忆基类。

**继承自**: `MemoryBase`

### 具体实现

#### `InMemoryMemory`
内存记忆实现，用于短期记忆。

#### `Mem0LongTermMemory`
基于Mem0的长期记忆实现。

**参数**
- `api_key`: Mem0 API密钥
- `base_url`: Mem0服务地址

#### `ReMePersonalLongTermMemory`
个人长期记忆。

#### `ReMeTaskLongTermMemory`
任务长期记忆。

#### `ReMeToolLongTermMemory`
工具长期记忆。

**Section sources**
- [memory\__init__.py](file://src/agentscope/memory/__init__.py#L3-L22)

## 消息模块

消息模块提供了消息和内容块的定义。

### 消息类

#### `Msg`
消息对象。

**参数**
- `name`: 发送者名称
- `content`: 消息内容
- `role`: 角色（"system", "user", "assistant"等）
- `timestamp`: 时间戳

**属性**
- `id`: 消息ID
- `metadata`: 元数据

### 内容块

#### `ContentBlock`
内容块基类。

#### `TextBlock`
文本内容块。

**参数**
- `text`: 文本内容

#### `ThinkingBlock`
思考内容块，用于ReAct智能体的思考过程。

#### `ToolUseBlock`
工具使用内容块。

**参数**
- `name`: 工具名称
- `arguments`: 工具参数

#### `ToolResultBlock`
工具结果内容块。

**参数**
- `name`: 工具名称
- `content`: 工具执行结果

#### `ImageBlock`
图像内容块。

**参数**
- `source`: 图像源（Base64Source或URLSource）

#### `AudioBlock`
音频内容块。

#### `VideoBlock`
视频内容块。

### 源类型

#### `Base64Source`
Base64编码的源。

**参数**
- `data`: Base64编码的数据
- `media_type`: 媒体类型

#### `URLSource`
URL源。

**参数**
- `url`: URL地址

**Section sources**
- [message\__init__.py](file://src/agentscope/message/__init__.py#L3-L31)

## 会话模块

会话模块提供了会话管理功能。

### 基础类

#### `SessionBase`
会话基类。

**方法**
- `save()`: 保存会话
- `load()`: 加载会话
- `clear()`: 清除会话

### 具体实现

#### `JSONSession`
JSON格式的会话存储。

**参数**
- `session_id`: 会话ID
- `path`: 存储路径

**方法**
- `save_to_json(file_path: str)`: 保存到JSON文件
- `load_from_json(file_path: str)`: 从JSON文件加载

**Section sources**
- [session\__init__.py](file://src/agentscope/session/__init__.py#L3-L10)

## 嵌入模块

嵌入模块提供了文本和多模态嵌入功能。

### 基础类

#### `EmbeddingModelBase`
嵌入模型基类。

**方法**
- `embed(texts: List[str]) -> EmbeddingResponse`: 生成嵌入

### 响应类

#### `EmbeddingResponse`
嵌入响应对象。

**属性**
- `embeddings`: 嵌入向量列表
- `usage`: 使用统计
- `model`: 模型名称

#### `EmbeddingUsage`
嵌入使用统计。

**属性**
- `prompt_tokens`: 提示词元数
- `total_tokens`: 总词元数

### 具体实现

#### `DashScopeTextEmbedding`
通义千问文本嵌入。

**参数**
- `model_config_name`: 模型配置名称
- `api_key`: API密钥

#### `DashScopeMultiModalEmbedding`
通义千问多模态嵌入。

#### `OpenAITextEmbedding`
OpenAI文本嵌入。

#### `GeminiTextEmbedding`
Gemini文本嵌入。

#### `OllamaTextEmbedding`
Ollama文本嵌入。

### 缓存类

#### `EmbeddingCacheBase`
嵌入缓存基类。

**方法**
- `get(key: str) -> List[float]`: 获取缓存
- `set(key: str, value: List[float])`: 设置缓存

#### `FileEmbeddingCache`
文件嵌入缓存。

**参数**
- `cache_dir`: 缓存目录

**Section sources**
- [embedding\__init__.py](file://src/agentscope/embedding/__init__.py#L3-L27)

## 令牌模块

令牌模块提供了令牌计数功能。

### 基础类

#### `TokenCounterBase`
令牌计数器基类。

**方法**
- `count(text: str) -> int`: 计算文本的令牌数

### 具体实现

#### `GeminiTokenCounter`
Gemini令牌计数器。

#### `OpenAITokenCounter`
OpenAI令牌计数器。

#### `AnthropicTokenCounter`
Anthropic令牌计数器。

#### `HuggingFaceTokenCounter`
HuggingFace令牌计数器。

**Section sources**
- [token\__init__.py](file://src/agentscope/token/__init__.py#L3-L16)

## 评估模块

评估模块提供了模型和系统的评估功能。

### 基础类

#### `EvaluatorBase`
评估器基类。

**方法**
- `evaluate(solution: SolutionOutput) -> MetricResult`: 评估解决方案

#### `BenchmarkBase`
基准测试基类。

**方法**
- `run() -> SolutionOutput`: 运行基准测试

#### `MetricBase`
指标基类。

**方法**
- `compute() -> MetricResult`: 计算指标

### 具体实现

#### `RayEvaluator`
基于Ray的分布式评估器。

#### `GeneralEvaluator`
通用评估器。

#### `ACEBenchmark`
ACE基准测试。

#### `ACEAccuracy`
ACE准确率指标。

#### `ACEProcessAccuracy`
ACE过程准确率指标。

#### `ACEPhone`
ACE电话任务。

### 存储类

#### `EvaluatorStorageBase`
评估器存储基类。

**方法**
- `save(result: MetricResult)`: 保存结果
- `load() -> MetricResult`: 加载结果

#### `FileEvaluatorStorage`
文件评估器存储。

**参数**
- `storage_path`: 存储路径

### 其他类

#### `Task`
任务类。

#### `SolutionOutput`
解决方案输出类。

**Section sources**
- [evaluate\__init__.py](file://src/agentscope/evaluate/__init__.py#L3-L44)

## 流水线模块

流水线模块提供了复杂工作流的构建功能。

### 类实现

#### `SequentialPipeline`
顺序流水线。

**参数**
- `agents`: 智能体列表
- `exit_condition`: 退出条件

#### `FanoutPipeline`
扇出流水线。

**参数**
- `agents`: 智能体列表
- `aggregator`: 聚合函数

### 函数实现

#### `sequential_pipeline(agents: List, exit_condition=None)`
创建顺序流水线。

**参数**
- `agents`: 智能体列表
- `exit_condition`: 退出条件函数

#### `fanout_pipeline(agents: List, aggregator=None)`
创建扇出流水线。

**参数**
- `agents`: 智能体列表
- `aggregator`: 结果聚合函数

#### `stream_printing_messages(msg: Msg)`
流式打印消息。

**参数**
- `msg`: 消息对象

### 上下文管理

#### `MsgHub`
消息中心，用于智能体间的消息共享。

**参数**
- `name`: 名称
- `participants`: 参与者列表

**用法**
```python
with MsgHub(name="shared_memory", participants=[agent1, agent2]) as hub:
    # 在此上下文中，agent1和agent2共享消息
    pass
```

**Section sources**
- [pipeline\__init__.py](file://src/agentscope/pipeline/__init__.py#L3-L20)

## 格式化模块

格式化模块提供了不同模型的提示格式化功能。

### 基础类

#### `FormatterBase`
格式化器基类。

**方法**
- `format(messages: List[Msg]) -> List[Dict]`: 格式化消息列表

#### `TruncatedFormatterBase`
截断格式化器基类。

**继承自**: `FormatterBase`

**方法**
- `format_with_truncation(messages: List[Msg], max_tokens: int) -> List[Dict]`: 带截断的格式化

### 具体实现

#### `A2AFormatter`
A2A格式化器。

#### `AnthropicFormatter`
Anthropic模型格式化器。

#### `DashScopeFormatter`
通义千问模型格式化器。

#### `DeepSeekFormatter`
DeepSeek模型格式化器。

#### `GeminiFormatter`
Gemini模型格式化器。

#### `OllamaFormatter`
Ollama模型格式化器。

#### `OpenAIFormatter`
OpenAI模型格式化器。

**Section sources**
- [formatter\__init__.py](file://src/agentscope/formatter/__init__.py)

## RAG模块

RAG（检索增强生成）模块提供了知识检索功能。

### 基础类

#### `ReaderBase`
读取器基类。

**方法**
- `read(source: str) -> Document`: 读取源内容

#### `StoreBase`
存储基类。

**方法**
- `add(documents: List[Document])`: 添加文档
- `search(query: str, k: int = 10) -> List[Document]`: 搜索文档

### 读取器实现

#### `TextReader`
文本文件读取器。

#### `PDFReader`
PDF文件读取器。

#### `WordReader`
Word文档读取器。

#### `ImageReader`
图像读取器。

### 存储实现

#### `MilvusLiteStore`
Milvus Lite向量存储。

#### `QdrantStore`
Qdrant向量存储。

### 其他类

#### `Document`
文档类。

**属性**
- `content`: 内容
- `metadata`: 元数据
- `embedding`: 嵌入向量

#### `KnowledgeBase`
知识库类。

**方法**
- `add_document(doc: Document)`: 添加文档
- `query(question: str) -> List[Document]`: 查询知识库

#### `SimpleKnowledge`
简单知识类。

**Section sources**
- [rag\__init__.py](file://src/agentscope/rag/__init__.py)

## A2A模块

A2A（Agent-to-Agent）模块提供了智能体间通信功能。

### 解析器

#### `FileResolver`
文件解析器，从文件加载智能体配置。

#### `NacosResolver`
Nacos服务发现解析器。

#### `WellKnownResolver`
知名服务解析器。

### 基础类

#### `BaseResolver`
解析器基类。

**方法**
- `resolve(agent_id: str) -> AgentConfig`: 解析智能体配置

### 其他类

#### `A2AAgent`
A2A智能体类。

**Section sources**
- [a2a\__init__.py](file://src/agentscope/a2a/__init__.py)

## 追踪模块

追踪模块提供了系统追踪和监控功能。

### 基础类

#### `Trace`
追踪类。

**方法**
- `start_span(name: str)`: 开始跨度
- `end_span()`: 结束跨度

#### `Attributes`
属性类，用于存储追踪属性。

#### `Extractor`
提取器类，用于从上下文提取追踪信息。

#### `Converter`
转换器类，用于转换追踪数据格式。

### 函数

#### `setup_tracing(endpoint: str)`
设置追踪。

**参数**
- `endpoint`: 追踪端点URL

**Section sources**
- [tracing\__init__.py](file://src/agentscope/tracing/__init__.py)

## 配置与环境变量

### 配置选项

AgentScope支持通过`init()`函数进行配置，主要配置选项包括：

- **project**: 项目名称
- **name**: 运行名称
- **run_id**: 运行ID
- **logging_path**: 日志路径
- **logging_level**: 日志级别
- **studio_url**: Studio URL
- **tracing_url**: 追踪URL

### 环境变量

AgentScope会读取以下环境变量：

- **DASHSCOPE_API_KEY**: 通义千问API密钥
- **OPENAI_API_KEY**: OpenAI API密钥
- **ANTHROPIC_API_KEY**: Anthropic API密钥
- **GOOGLE_API_KEY**: Google API密钥
- **MEM0_API_KEY**: Mem0 API密钥

**Section sources**
- [__init__.py](file://src/agentscope/__init__.py#L72-L157)

## 版本兼容性

### 当前版本

当前版本为1.0.10，是稳定版本。

### 弃用警告

以下功能已被弃用：

- `run_dir`字段：在`init()`函数的注册数据中已被标记为废弃

### 兼容性说明

- 1.x版本之间保持向后兼容
- 重大变更将在2.0版本中引入
- 建议使用最新版本以获得最佳功能和安全性

**Section sources**
- [_version.py](file://src/agentscope/_version.py#L4)
- [__init__.py](file://src/agentscope/__init__.py#L126-L128)