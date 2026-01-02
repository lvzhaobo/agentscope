# 追踪系统API

<cite>
**本文档中引用的文件**   
- [__init__.py](file://src/agentscope/tracing/__init__.py)
- [_setup.py](file://src/agentscope/tracing/_setup.py)
- [_trace.py](file://src/agentscope/tracing/_trace.py)
- [_attributes.py](file://src/agentscope/tracing/_attributes.py)
- [_extractor.py](file://src/agentscope/tracing/_extractor.py)
- [_converter.py](file://src/agentscope/tracing/_converter.py)
- [_utils.py](file://src/agentscope/tracing/_utils.py)
- [_run_config.py](file://src/agentscope/_run_config.py)
- [task_tracing.py](file://docs/tutorial/zh_CN/src/task_tracing.py)
</cite>

## 目录
1. [简介](#简介)
2. [追踪系统初始化](#追踪系统初始化)
3. [跨度创建与装饰器](#跨度创建与装饰器)
4. [自定义属性与事件记录](#自定义属性与事件记录)
5. [上下文传播机制](#上下文传播机制)
6. [第三方APM平台集成](#第三方apm平台集成)
7. [性能数据导出与采样](#性能数据导出与采样)
8. [敏感信息过滤](#敏感信息过滤)
9. [分布式追踪调试](#分布式追踪调试)

## 简介

AgentScope追踪系统基于OpenTelemetry实现，为多智能体应用提供全面的可观测性支持。该系统通过自动追踪关键组件（如LLM调用、智能体回复、工具调用等）来生成详细的追踪数据，帮助开发者分析系统性能、调试问题并优化应用行为。

追踪系统的核心功能包括：
- 自动追踪LLM调用、智能体交互、工具执行等关键操作
- 支持与Jaeger、Zipkin等第三方APM平台集成
- 提供丰富的API用于自定义属性添加和事件记录
- 实现跨智能体调用链的上下文传播
- 支持性能数据导出、采样率配置和敏感信息过滤

**Section sources**
- [__init__.py](file://src/agentscope/tracing/__init__.py#L1-L23)
- [task_tracing.py](file://docs/tutorial/zh_CN/src/task_tracing.py#L44-L72)

## 追踪系统初始化

追踪系统的初始化通过`setup_tracing`函数完成，该函数配置OpenTelemetry的追踪导出器端点。初始化过程包括创建追踪提供者、配置批量跨度处理器和设置导出器。

```mermaid
flowchart TD
Start([开始]) --> CheckEnabled{"追踪已启用?"}
CheckEnabled --> |否| Return[返回原始函数]
CheckEnabled --> |是| GetTracer[获取追踪器]
GetTracer --> CreateSpan[创建跨度]
CreateSpan --> Execute[执行函数]
Execute --> CheckResult{"结果是生成器?"}
CheckResult --> |是| WrapGenerator[包装生成器]
CheckResult --> |否| SetAttributes[设置属性]
SetAttributes --> CheckError{"发生错误?"}
CheckError --> |是| RecordError[记录错误]
CheckError --> |否| SetSuccess[设置成功状态]
RecordError --> End[结束]
SetSuccess --> End
```

**Diagram sources**
- [_setup.py](file://src/agentscope/tracing/_setup.py#L11-L39)
- [_trace.py](file://src/agentscope/tracing/_trace.py#L192-L319)

**Section sources**
- [_setup.py](file://src/agentscope/tracing/_setup.py#L11-L39)
- [_trace.py](file://src/agentscope/tracing/_trace.py#L69-L77)

## 跨度创建与装饰器

追踪系统提供了一系列装饰器用于创建和管理跨度，支持同步和异步函数以及生成器函数的追踪。

### 核心装饰器

```mermaid
classDiagram
class trace {
+name : str | None
+decorator(func : Callable) Callable
+wrapper(*args, **kwargs) Any
}
class trace_llm {
+func : Callable
+wrapper(self : ChatModelBase, *args, **kwargs) ChatResponse | AsyncGenerator
}
class trace_reply {
+func : Callable
+wrapper(self : AgentBase, *args, **kwargs) Msg
}
class trace_toolkit {
+func : Callable
+wrapper(self : Toolkit, tool_call : ToolUseBlock) AsyncGenerator[ToolResponse, None]
}
class trace_format {
+func : Callable
+wrapper(self : FormatterBase, *args, **kwargs) list[dict]
}
class trace_embedding {
+func : Callable
+wrapper(self : EmbeddingModelBase, *args, **kwargs) EmbeddingResponse
}
trace --> trace_llm : "继承"
trace --> trace_reply : "继承"
trace --> trace_toolkit : "继承"
trace --> trace_format : "继承"
trace --> trace_embedding : "继承"
```

**Diagram sources**
- [_trace.py](file://src/agentscope/tracing/_trace.py#L192-L649)
- [_attributes.py](file://src/agentscope/tracing/_attributes.py#L131-L150)

**Section sources**
- [_trace.py](file://src/agentscope/tracing/_trace.py#L192-L649)
- [_attributes.py](file://src/agentscope/tracing/_attributes.py#L131-L150)

### 装饰器使用示例

```mermaid
sequenceDiagram
participant User as "用户代码"
participant Decorator as "追踪装饰器"
participant Function as "目标函数"
participant Span as "OpenTelemetry跨度"
User->>Decorator : 调用被装饰的函数
Decorator->>Decorator : 检查追踪是否启用
alt 追踪已启用
Decorator->>Span : 创建新跨度
Span->>Function : 执行函数
Function-->>Span : 返回结果
alt 结果是生成器
Span->>Span : 包装生成器
else 非生成器结果
Span->>Span : 设置响应属性
end
Span->>Span : 设置成功状态
Span-->>Decorator : 返回结果
else 追踪未启用
Function-->>Decorator : 返回结果
end
Decorator-->>User : 返回结果
```

**Diagram sources**
- [_trace.py](file://src/agentscope/tracing/_trace.py#L208-L317)
- [_utils.py](file://src/agentscope/tracing/_utils.py#L15-L79)

## 自定义属性与事件记录

追踪系统支持添加自定义属性和记录事件，通过属性提取器和序列化工具实现复杂对象的JSON转换。

### 属性提取机制

```mermaid
classDiagram
class SpanAttributes {
+GEN_AI_CONVERSATION_ID : str
+GEN_AI_OPERATION_NAME : str
+GEN_AI_PROVIDER_NAME : str
+GEN_AI_REQUEST_MODEL : str
+GEN_AI_REQUEST_TEMPERATURE : str
+GEN_AI_REQUEST_TOP_P : str
+GEN_AI_REQUEST_MAX_TOKENS : str
+GEN_AI_RESPONSE_ID : str
+GEN_AI_RESPONSE_FINISH_REASONS : str
+GEN_AI_USAGE_INPUT_TOKENS : str
+GEN_AI_USAGE_OUTPUT_TOKENS : str
+AGENTSCOPE_FUNCTION_NAME : str
+AGENTSCOPE_FUNCTION_INPUT : str
+AGENTSCOPE_FUNCTION_OUTPUT : str
}
class OperationNameValues {
+FORMATTER : str
+INVOKE_GENERIC_FUNCTION : str
+CHAT : str
+INVOKE_AGENT : str
+EXECUTE_TOOL : str
+EMBEDDINGS : str
}
class ProviderNameValues {
+DASHSCOPE : str
+OLLAMA : str
+OPENAI : str
+ANTHROPIC : str
+GCP_GEMINI : str
+MOONSHOT : str
+AZURE_AI_OPENAI : str
+AWS_BEDROCK : str
}
SpanAttributes --> OperationNameValues : "使用"
SpanAttributes --> ProviderNameValues : "使用"
```

**Diagram sources**
- [_attributes.py](file://src/agentscope/tracing/_attributes.py#L8-L184)
- [_extractor.py](file://src/agentscope/tracing/_extractor.py#L52-L800)

**Section sources**
- [_attributes.py](file://src/agentscope/tracing/_attributes.py#L8-L184)
- [_extractor.py](file://src/agentscope/tracing/_extractor.py#L52-L800)

### 对象序列化

```mermaid
flowchart TD
Start([开始]) --> CheckType{"类型检查"}
CheckType --> |基本类型| Return[直接返回]
CheckType --> |列表/元组/集合| Loop1[遍历并递归转换]
CheckType --> |字典| Loop2[遍历键值对并递归转换]
CheckType --> |Msg/BaseModel/数据类| Return[返回repr]
CheckType --> |类且继承BaseModel| Return[返回repr]
CheckType --> |日期时间| ISOFormat[转换为ISO格式]
CheckType --> |时间间隔| TotalSeconds[转换为总秒数]
CheckType --> |枚举| Value[返回值并递归转换]
CheckType --> |其他| String[转换为字符串]
Loop1 --> Process[处理每个元素]
Process --> Convert[_to_serializable]
Convert --> Loop1
Loop2 --> Process2[处理每个键值对]
Process2 --> Convert2[_to_serializable]
Convert2 --> Loop2
```

**Diagram sources**
- [_utils.py](file://src/agentscope/tracing/_utils.py#L15-L79)
- [_converter.py](file://src/agentscope/tracing/_converter.py#L11-L126)

## 上下文传播机制

追踪系统通过OpenTelemetry的上下文传播机制实现跨智能体调用链的追踪，确保分布式环境下的调用链完整性。

### 上下文传播流程

```mermaid
sequenceDiagram
participant AgentA as "智能体A"
participant AgentB as "智能体B"
participant Context as "OpenTelemetry上下文"
participant Span as "当前跨度"
AgentA->>Span : 开始新跨度
Span->>Context : 将跨度设为当前
AgentA->>AgentB : 发送消息(包含上下文)
AgentB->>Context : 从消息中提取上下文
Context->>Span : 恢复跨度上下文
AgentB->>Span : 创建子跨度
Span->>Context : 将子跨度设为当前
AgentB->>AgentA : 返回响应(包含更新的上下文)
AgentA->>Context : 合并响应上下文
Context->>Span : 结束当前跨度
```

**Diagram sources**
- [_extractor.py](file://src/agentscope/tracing/_extractor.py#L447-L505)
- [_run_config.py](file://src/agentscope/_run_config.py#L66-L73)

**Section sources**
- [_extractor.py](file://src/agentscope/tracing/_extractor.py#L447-L505)
- [_run_config.py](file://src/agentscope/_run_config.py#L66-L73)

### 跨智能体调用链

```mermaid
graph TD
subgraph "调用链"
A[智能体A] --> |调用| B[智能体B]
B --> |调用| C[LLM服务]
B --> |调用| D[工具服务]
C --> |返回| B
D --> |返回| B
B --> |返回| A
end
subgraph "追踪上下文"
ContextA[上下文A] --> ContextB[上下文B]
ContextB --> ContextC[上下文C]
ContextB --> ContextD[上下文D]
ContextC --> ContextB
ContextD --> ContextB
ContextB --> ContextA
end
A --> ContextA
B --> ContextB
C --> ContextC
D --> ContextD
```

**Diagram sources**
- [_trace.py](file://src/agentscope/tracing/_trace.py#L369-L435)
- [_extractor.py](file://src/agentscope/tracing/_extractor.py#L447-L505)

## 第三方APM平台集成

追踪系统支持与多种第三方APM平台集成，通过OTLP协议将追踪数据导出到目标平台。

### 集成配置

```mermaid
classDiagram
class APMIntegration {
+setup_tracing(endpoint : str) None
+_get_tracer() Tracer
}
APMIntegration --> Jaeger : "支持"
APMIntegration --> Zipkin : "支持"
APMIntegration --> Langfuse : "支持"
APMIntegration --> ArizePhoenix : "支持"
APMIntegration --> AlibabaCloudMonitor : "支持"
class Jaeger {
+endpoint : str
+protocol : OTLP
}
class Zipkin {
+endpoint : str
+protocol : OTLP
}
class Langfuse {
+endpoint : str
+headers : dict
+protocol : OTLP
}
class ArizePhoenix {
+endpoint : str
+api_key : str
+protocol : OTLP
}
class AlibabaCloudMonitor {
+endpoint : str
+adapt_id : str
+protocol : OTLP
}
APMIntegration ..> Jaeger : "配置"
APMIntegration ..> Zipkin : "配置"
APMIntegration ..> Langfuse : "配置"
APMIntegration ..> ArizePhoenix : "配置"
APMIntegration ..> AlibabaCloudMonitor : "配置"
```

**Diagram sources**
- [_setup.py](file://src/agentscope/tracing/_setup.py#L11-L39)
- [task_tracing.py](file://docs/tutorial/zh_CN/src/task_tracing.py#L50-L63)

**Section sources**
- [_setup.py](file://src/agentscope/tracing/_setup.py#L11-L39)
- [task_tracing.py](file://docs/tutorial/zh_CN/src/task_tracing.py#L50-L63)

### 集成示例

```mermaid
flowchart LR
subgraph "AgentScope应用"
A[智能体] --> B[追踪SDK]
B --> C[OTLP导出器]
end
subgraph "APM平台"
D[Jaeger]
E[Zipkin]
F[Langfuse]
G[Arize-Phoenix]
H[阿里云云监控]
end
C --> |HTTPS| D
C --> |HTTPS| E
C --> |HTTPS| F
C --> |HTTPS| G
C --> |HTTPS| H
style A fill:#f9f,stroke:#333
style B fill:#bbf,stroke:#333
style C fill:#f96,stroke:#333
style D fill:#9f9,stroke:#333
style E fill:#9f9,stroke:#333
style F fill:#9f9,stroke:#333
style G fill:#9f9,stroke:#333
style H fill:#9f9,stroke:#333
```

**Diagram sources**
- [task_tracing.py](file://docs/tutorial/zh_CN/src/task_tracing.py#L50-L72)
- [_setup.py](file://src/agentscope/tracing/_setup.py#L22-L24)

## 性能数据导出与采样

追踪系统提供灵活的性能数据导出选项和采样率配置，以平衡追踪数据的完整性和系统性能。

### 数据导出配置

```mermaid
classDiagram
class ExportConfig {
+endpoint : str
+headers : dict
+timeout : int
+max_export_batch_size : int
+schedule_delay_millis : int
+max_queue_size : int
}
class SamplingConfig {
+sampling_rate : float
+sampling_strategy : str
+trace_id_ratio_based : float
+parent_based : bool
}
class PerformanceConfig {
+export_config : ExportConfig
+sampling_config : SamplingConfig
+enable_metrics : bool
+metrics_export_interval : int
}
PerformanceConfig --> ExportConfig : "包含"
PerformanceConfig --> SamplingConfig : "包含"
```

**Diagram sources**
- [_setup.py](file://src/agentscope/tracing/_setup.py#L26-L28)
- [_run_config.py](file://src/agentscope/_run_config.py#L66-L73)

**Section sources**
- [_setup.py](file://src/agentscope/tracing/_setup.py#L26-L28)
- [_run_config.py](file://src/agentscope/_run_config.py#L66-L73)

### 采样策略

```mermaid
flowchart TD
Start([开始]) --> CheckEnabled{"追踪已启用?"}
CheckEnabled --> |否| Skip[跳过追踪]
CheckEnabled --> |是| CheckSampling{"需要采样?"}
CheckSampling --> |否| CreateSpan[创建完整跨度]
CheckSampling --> |是| GenerateRandom[生成随机数]
GenerateRandom --> Compare{"随机数 < 采样率?"}
Compare --> |是| CreateSpan
Compare --> |否| Skip
CreateSpan --> Execute[执行函数]
Execute --> RecordResult[记录结果]
RecordResult --> End[结束]
Skip --> Execute
```

**Diagram sources**
- [_trace.py](file://src/agentscope/tracing/_trace.py#L69-L77)
- [_setup.py](file://src/agentscope/tracing/_setup.py#L11-L39)

## 敏感信息过滤

追踪系统提供敏感信息过滤机制，通过序列化工具自动处理敏感数据，确保追踪数据的安全性。

### 敏感信息过滤流程

```mermaid
flowchart TD
Start([开始]) --> CheckType{"类型检查"}
CheckType --> |基本类型| Return[直接返回]
CheckType --> |复杂对象| Serialize[序列化对象]
Serialize --> Filter{"包含敏感信息?"}
Filter --> |是| Redact[脱敏处理]
Filter --> |否| Encode[JSON编码]
Redact --> Encode
Encode --> End[返回结果]
subgraph "敏感信息检测"
S1[密码]
S2[密钥]
S3[令牌]
S4[个人信息]
end
subgraph "脱敏处理"
D1[替换为***]
D2[哈希处理]
D3[部分隐藏]
end
S1 --> Filter
S2 --> Filter
S3 --> Filter
S4 --> Filter
D1 --> Redact
D2 --> Redact
D3 --> Redact
```

**Diagram sources**
- [_utils.py](file://src/agentscope/tracing/_utils.py#L15-L79)
- [_extractor.py](file://src/agentscope/tracing/_extractor.py#L251-L257)

**Section sources**
- [_utils.py](file://src/agentscope/tracing/_utils.py#L15-L79)
- [_extractor.py](file://src/agentscope/tracing/_extractor.py#L251-L257)

## 分布式追踪调试

追踪系统提供完整的分布式追踪调试支持，包括错误标注、异常处理和调试工具。

### 错误标注策略

```mermaid
sequenceDiagram
participant Function as "目标函数"
participant Decorator as "追踪装饰器"
participant Span as "跨度"
participant Exception as "异常"
Function->>Decorator : 执行函数
alt 成功执行
Decorator->>Span : 设置成功状态
Span-->>Function : 返回结果
else 发生异常
Exception-->>Decorator : 抛出异常
Decorator->>Span : 设置错误状态
Span->>Span : 记录异常
Span->>Span : 结束跨度
Decorator-->>Function : 重新抛出异常
end
```

**Diagram sources**
- [_trace.py](file://src/agentscope/tracing/_trace.py#L80-L104)
- [_extractor.py](file://src/agentscope/tracing/_extractor.py#L371-L391)

**Section sources**
- [_trace.py](file://src/agentscope/tracing/_trace.py#L80-L104)
- [_extractor.py](file://src/agentscope/tracing/_extractor.py#L371-L391)

### 调试方法

```mermaid
classDiagram
class Debugging {
+_set_span_success_status(span : Span) None
+_set_span_error_status(span : Span, e : Exception) None
+_check_tracing_enabled() bool
+_trace_sync_generator_wrapper(res : Generator, span : Span) Generator
+_trace_async_generator_wrapper(res : AsyncGenerator, span : Span) AsyncGenerator
}
class ErrorHandling {
+try-except块
+异常记录
+跨度结束
+异常重新抛出
}
Debugging --> ErrorHandling : "实现"
class Logging {
+警告日志
+类型检查
+跳过追踪
}
Debugging --> Logging : "实现"
```

**Diagram sources**
- [_trace.py](file://src/agentscope/tracing/_trace.py#L80-L132)
- [_extractor.py](file://src/agentscope/tracing/_extractor.py#L396-L403)

**Section sources**
- [_trace.py](file://src/agentscope/tracing/_trace.py#L80-L132)
- [_extractor.py](file://src/agentscope/tracing/_extractor.py#L396-L403)