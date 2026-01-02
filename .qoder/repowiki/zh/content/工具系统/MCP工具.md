# MCP工具

<cite>
**本文档中引用的文件**
- [mcp_add.py](file://examples/functionality/mcp/mcp_add.py)
- [mcp_multiply.py](file://examples/functionality/mcp/mcp_multiply.py)
- [main.py](file://examples/functionality/mcp/main.py)
- [main.py](file://examples/integration/alibabacloud_api_mcp/main.py)
- [oauth_handler.py](file://examples/integration/alibabacloud_api_mcp/oauth_handler.py)
- [_client_base.py](file://src/agentscope/mcp/_client_base.py)
- [_mcp_function.py](file://src/agentscope/mcp/_mcp_function.py)
- [_http_stateful_client.py](file://src/agentscope/mcp/_http_stateful_client.py)
- [_http_stateless_client.py](file://src/agentscope/mcp/_http_stateless_client.py)
- [_stdio_stateful_client.py](file://src/agentscope/mcp/_stdio_stateful_client.py)
- [_stateful_client_base.py](file://src/agentscope/mcp/_stateful_client_base.py)
- [_toolkit.py](file://src/agentscope/tool/_toolkit.py)
</cite>

## 目录
1. [MCP客户端协议](#mcp客户端协议)
2. [MCP工具函数封装](#mcp工具函数封装)
3. [工具包集成流程](#工具包集成流程)
4. [示例应用](#示例应用)
5. [认证与安全](#认证与安全)
6. [网络容错与性能优化](#网络容错与性能优化)

## MCP客户端协议

MCP（Model Calling Protocol）客户端协议通过`MCPClientBase`抽象基类定义了客户端通信的标准接口。该协议支持有状态（Stateful）和无状态（Stateless）两种通信模式，为不同类型的MCP服务提供了灵活的连接方式。

`MCPClientBase`作为所有MCP客户端的基类，定义了核心的通信协议。它通过抽象方法`get_callable_function`提供工具函数获取接口，允许客户端根据函数名称动态获取可调用的远程工具。该基类还包含`_convert_mcp_content_to_as_blocks`方法，负责将MCP协议的内容块转换为AgentScope系统内部使用的块结构，包括文本、图像、音频和视频等多媒体内容的转换。

有状态客户端通过`StatefulClientBase`基类实现，该类继承自`MCPClientBase`并提供了连接生命周期管理功能。有状态客户端需要显式调用`connect()`方法建立连接，并在使用完毕后调用`close()`方法释放资源。这种模式适用于需要维护会话状态的交互式服务，如浏览器自动化或需要上下文保持的复杂交互场景。

**Section sources**
- [mcp/_client_base.py](file://src/agentscope/mcp/_client_base.py#L18-L102)
- [mcp/_stateful_client_base.py](file://src/agentscope/mcp/_stateful_client_base.py#L16-L173)

## MCP工具函数封装

`MCPToolFunction`类负责将远程MCP服务封装为本地可调用的函数对象，实现了JSON Schema的自动转换和参数映射。该类通过`_extract_json_schema_from_mcp_tool`工具函数从MCP工具定义中提取JSON Schema，将远程服务的接口描述转换为本地可理解的函数签名。

`MCPToolFunction`的初始化过程接收MCP服务名称、工具定义、结果包装选项以及客户端生成器或会话对象。当`wrap_tool_result`参数为`True`时，工具调用结果会被包装成`ToolResponse`对象，其中包含转换后的AgentScope内容块和元数据。这种封装机制使得远程MCP服务可以像本地函数一样被调用，同时保持了结果格式的一致性。

工具函数的调用通过`__call__`方法实现，支持异步调用模式。调用时，系统会根据配置使用客户端生成器创建新的会话，或复用现有的会话对象来执行工具调用。这种设计既支持无状态的独立调用，也支持有状态的连续交互，为不同类型的MCP服务提供了统一的调用接口。

**Section sources**
- [mcp/_mcp_function.py](file://src/agentscope/mcp/_mcp_function.py#L14-L87)
- [mcp/_client_base.py](file://src/agentscope/mcp/_client_base.py#L40-L102)

## 工具包集成流程

`Toolkit.register_mcp_client`方法提供了MCP客户端的集成流程，支持细粒度的工具控制和配置。该方法允许开发者通过多种参数对MCP工具进行精确管理，包括工具过滤、预设参数映射和冲突处理策略。

工具过滤功能通过`enable_funcs`和`disable_funcs`参数实现。`enable_funcs`参数指定需要启用的工具函数列表，如果为`None`则启用所有可用工具；`disable_funcs`参数指定需要禁用的工具函数列表。这两个参数不能同时使用，且不能有重叠的函数名称，确保了工具选择的明确性和一致性。

预设参数映射通过`preset_kwargs_mapping`参数实现，这是一个字典结构，键为工具函数名称，值为需要预设的参数字典。这些预设参数不会出现在JSON Schema中，也不会暴露给代理模型，实现了敏感参数或固定配置的隐藏。`postprocess_func`参数允许指定后处理函数，在工具执行后对结果进行额外处理，支持同步和异步两种模式。

当工具名称发生冲突时，`namesake_strategy`参数提供了四种处理策略：`raise`（抛出异常）、`override`（覆盖现有工具）、`skip`（跳过注册）和`rename`（重命名新工具）。这种灵活的冲突处理机制确保了工具注册过程的健壮性。

**Section sources**
- [tool/_toolkit.py](file://src/agentscope/tool/_toolkit.py#L727-L865)
- [tool/_toolkit.py](file://src/agentscope/tool/_toolkit.py#L797-L818)

## 示例应用

MCP工具的典型应用包括基本数学服务的集成和复杂代理系统的构建。在数学服务示例中，`mcp_add.py`和`mcp_multiply.py`分别实现了加法和乘法服务，通过`FastMCP`框架创建MCP服务器。这些服务使用SSE和流式HTTP传输协议，展示了不同通信模式的应用。

在`main.py`示例中，通过创建`HttpStatefulClient`和`HttpStatelessClient`实例连接到不同的MCP服务器，然后使用`Toolkit.register_mcp_client`方法将这些客户端注册到工具包中。注册后，ReAct代理可以使用这些工具执行复杂的计算任务，如先乘法后加法的组合运算。

在`deep_research_agent`的Tavily搜索集成中，展示了MCP在信息检索场景的应用。通过`HttpStatelessClient`连接到Alibaba Cloud的MCP服务，使用OAuth认证机制确保安全访问。`oauth_handler.py`文件实现了完整的OAuth 2.0授权流程，包括回调服务器、令牌存储和重定向处理，为需要认证的MCP服务提供了参考实现。

**Section sources**
- [functionality/mcp/main.py](file://examples/functionality/mcp/main.py#L34-L111)
- [functionality/mcp/mcp_add.py](file://examples/functionality/mcp/mcp_add.py#L1-L17)
- [functionality/mcp/mcp_multiply.py](file://examples/functionality/mcp/mcp_multiply.py#L1-L17)
- [integration/alibabacloud_api_mcp/main.py](file://examples/integration/alibabacloud_api_mcp/main.py#L70-L102)
- [integration/alibabacloud_api_mcp/oauth_handler.py](file://examples/integration/alibabacloud_api_mcp/oauth_handler.py#L1-L267)

## 认证与安全

MCP系统通过多种机制确保通信的安全性。对于需要认证的服务，系统支持OAuth 2.0等标准认证协议。在Alibaba Cloud MCP集成示例中，`OAuthClientProvider`与`HttpStatelessClient`结合使用，实现了完整的授权码流程。

认证流程包括重定向用户到授权服务器、处理回调、交换授权码获取访问令牌等步骤。`InMemoryTokenStorage`类提供了令牌存储的参考实现，支持访问令牌和刷新令牌的持久化。`handle_redirect`和`handle_callback`函数处理浏览器重定向和回调，确保用户能够顺利完成授权过程。

对于不需要复杂认证的内部服务，系统依赖传输层安全（TLS）和API密钥等简单认证机制。`headers`参数允许在HTTP请求中添加自定义头部，如认证令牌或API密钥，为不同安全需求的MCP服务提供了灵活的认证选项。

**Section sources**
- [integration/alibabacloud_api_mcp/main.py](file://examples/integration/alibabacloud_api_mcp/main.py#L37-L59)
- [integration/alibabacloud_api_mcp/oauth_handler.py](file://examples/integration/alibabacloud_api_mcp/oauth_handler.py#L72-L267)

## 网络容错与性能优化

MCP系统通过多种策略实现网络容错和性能优化。客户端实现中包含超时机制，`timeout`参数控制HTTP请求的超时时间，默认为30秒，`sse_read_timeout`参数控制SSE读取的超时时间，默认为5分钟。这些超时设置防止了网络异常导致的无限等待。

对于有状态客户端，系统建议按照后进先出（LIFO）原则关闭连接，避免潜在的资源竞争问题。`AsyncExitStack`用于管理异步资源的生命周期，确保即使在异常情况下也能正确释放连接资源。

性能优化方面，系统实现了工具列表的缓存机制。`StatefulClientBase`中的`_cached_tools`属性缓存了`list_tools`的查询结果，避免了重复的网络请求。对于无状态客户端，虽然每次调用都创建新会话，但工具列表的缓存仍然减少了`get_callable_function`调用时的网络开销。

在高并发场景下，建议使用有状态客户端以减少连接建立的开销，并通过工具分组和激活机制控制代理可用的工具集，减少上下文长度和推理成本。

**Section sources**
- [mcp/_http_stateful_client.py](file://src/agentscope/mcp/_http_stateful_client.py#L31-L85)
- [mcp/_http_stateless_client.py](file://src/agentscope/mcp/_http_stateless_client.py#L30-L149)
- [mcp/_stateful_client_base.py](file://src/agentscope/mcp/_stateful_client_base.py#L46-L96)