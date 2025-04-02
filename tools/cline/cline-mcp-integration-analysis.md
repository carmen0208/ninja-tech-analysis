# Cline与MCP集成分析

## 1. MCP服务器如何被集成到Cline客户端

### 通信协议

Cline客户端通过**Model Context Protocol (MCP)**与MCP服务器集成。从代码中可以看出，集成主要基于以下协议组件：

- **传输层**：使用的是`StdioClientTransport`，这是一种基于标准输入/输出的通信机制。从`src/services/mcp/McpHub.ts`中可以看到：

```typescript
const transport = new StdioClientTransport({
    command: config.command,
    args: config.args,
    env: {
        ...config.env,
        ...(process.env.PATH ? { PATH: process.env.PATH } : {}),
    },
    stderr: "pipe",
})
```

- **客户端库**：Cline使用`@modelcontextprotocol/sdk/client`库来建立与MCP服务器的连接：

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js"
```

- **连接过程**：通过`client.connect(transport)`方法建立与MCP服务器的连接，然后可以使用`client.request()`方法发送请求。

### 配置集成

Cline通过一个名为`cline_mcp_settings.json`的配置文件来管理MCP服务器：

1. 每个MCP服务器在配置文件中有一个条目，指定了启动服务器的命令、参数和环境变量等。
2. `McpHub`类负责监视这个配置文件的变化，并根据配置自动管理MCP服务器的连接。
3. 连接成功后，Cline会从MCP服务器获取其提供的工具列表、资源列表和资源模板列表。

## 2. 用户提问时的工具选择与执行流程

当用户提问时，LLM (如Claude)基于提问内容和它所知道的可用工具来决定使用哪个MCP工具。

### 工作流程

1. **系统提示注入**：从`src/core/prompts/system.ts`可以看到，系统提示中包含了MCP服务器的信息：

```typescript
${
    mcpHub.getMode() !== "off"
        ? `
====

MCP SERVERS

The Model Context Protocol (MCP) enables communication between the system and locally running MCP servers that provide additional tools and resources to extend your capabilities.

# Connected MCP Servers

When a server is connected, you can use the server's tools via the \`use_mcp_tool\` tool, and access the server's resources via the \`access_mcp_resource\` tool.
...
```

2. **工具选择**：当LLM处理用户请求时，它会分析需要完成的任务，并从系统提示中提供的可用工具列表中选择适当的工具。

3. **工具调用**：LLM会创建一个`use_mcp_tool`格式的请求，指定服务器名称、工具名称和参数：

```xml
<use_mcp_tool>
<server_name>服务器名称</server_name>
<tool_name>工具名称</tool_name>
<arguments>
{
  "参数1": "值1",
  "参数2": "值2"
}
</arguments>
</use_mcp_tool>
```

4. **执行流程**：从`src/core/Cline.ts`中可以看到执行流程：
   - 首先，检查是否需要用户批准工具调用
   - 之后，调用`mcpHub.callTool(server_name, tool_name, parsedArguments)`执行工具
   - 处理返回结果并展示给用户

### 实例代码

关键执行代码在`src/core/Cline.ts`的第2591-2642行：

```typescript
// 检查工具是否需要自动批准
if (this.shouldAutoApproveTool(block.name) && isToolAutoApproved) {
    this.removeLastPartialMessageIfExistsWithType("ask", "use_mcp_server")
    await this.say("use_mcp_server", completeMessage, undefined, false)
    this.consecutiveAutoApprovedRequestsCount++
} else {
    // 需要用户批准
    showNotificationForApprovalIfAutoApprovalEnabled(
        `Cline wants to use ${tool_name} on ${server_name}`
    )
    this.removeLastPartialMessageIfExistsWithType("say", "use_mcp_server")
    const didApprove = await askApproval("use_mcp_server", completeMessage)
    if (!didApprove) {
        break
    }
}

// 执行工具
await this.say("mcp_server_request_started") 
const toolResult = await this.providerRef
    .deref()
    ?.mcpHub?.callTool(server_name, tool_name, parsedArguments)

// 处理结果
const toolResultPretty = (toolResult?.isError ? "Error:\n" : "") +
    toolResult?.content
        .map((item) => {
            if (item.type === "text") {
                return item.text
            }
            if (item.type === "resource") {
                const { blob, ...rest } = item.resource
                return JSON.stringify(rest, null, 2)
            }
            return ""
        })
        .filter(Boolean)
        .join("\n\n") || "(No response)"
await this.say("mcp_server_response", toolResultPretty)
```

### 实际工具调用示例

系统提示中提供了几个例子，比如：

```xml
<use_mcp_tool>
<server_name>weather-server</server_name>
<tool_name>get_forecast</tool_name>
<arguments>
{
  "city": "San Francisco",
  "days": 5
}
</arguments>
</use_mcp_tool>
```

这个例子中，LLM决定使用名为`weather-server`的MCP服务器中的`get_forecast`工具来获取旧金山未来5天的天气预报。

## 3. 多MCP服务器调用的情况

Cline支持在一个用户会话中调用多个不同的MCP服务器。这是通过迭代工作流程实现的。

### 多服务器调用的实现

1. **服务器独立性**：每个MCP服务器有独立的连接和状态管理。从`McpHub.ts`中可以看到：

```typescript
// Each MCP server requires its own transport connection and has unique capabilities, configurations, and error handling. Having separate clients also allows proper scoping of resources/tools and independent server management like reconnection.
const client = new Client(...)
```

2. **服务器查找和调用**：调用工具时，根据服务器名称找到对应的连接：

```typescript
async callTool(serverName: string, toolName: string, toolArguments?: Record<string, unknown>): Promise<McpToolCallResponse> {
    const connection = this.connections.find((conn) => conn.server.name === serverName)
    if (!connection) {
        throw new Error(`No connection found for server: ${serverName}...`)
    }
    
    // ... 调用工具
    return await connection.client.request(...)
}
```

3. **迭代式工作流**：系统提示中明确指出了迭代工作流程：

```
3. If multiple actions are needed, use one tool at a time per message to accomplish the task iteratively, with each tool use being informed by the result of the previous tool use. Do not assume the outcome of any tool use. Each step must be informed by the previous step's result.
```

### 多服务器调用实例

例如，一个用户可能需要完成一个涉及天气信息和GitHub操作的任务：

1. 首先，LLM可能会调用`weather-server`服务器的工具：
   ```xml
   <use_mcp_tool>
   <server_name>weather-server</server_name>
   <tool_name>get_forecast</tool_name>
   <arguments>{"city": "San Francisco", "days": 5}</arguments>
   </use_mcp_tool>
   ```

2. 然后，根据天气信息，可能需要在GitHub上创建一个issue：
   ```xml
   <use_mcp_tool>
   <server_name>github.com/modelcontextprotocol/servers/tree/main/src/github</server_name>
   <tool_name>create_issue</tool_name>
   <arguments>{
     "owner": "octocat",
     "repo": "hello-world",
     "title": "Weather alert: Rain expected in SF",
     "body": "Based on forecast data, we need to prepare for rain in SF."
   }</arguments>
   </use_mcp_tool>
   ```

在这个流程中：
1. 每个工具调用都是独立的操作
2. 后续工具的参数可能依赖于前一个工具的结果
3. 每次工具调用会涉及用户的确认步骤（除非设置了自动批准）
4. LLM会根据每个工具的执行结果调整后续的策略

## 总结

1. **集成协议**：Cline通过Model Context Protocol与MCP服务器集成，主要使用STDIO作为传输层。

2. **工具选择流程**：
   - LLM基于系统提示中列出的可用工具和用户问题决定使用哪个工具
   - 通过`use_mcp_tool`格式化对工具的调用
   - 工具调用通过`McpHub.callTool()`方法执行
   - 结果返回给用户

3. **多服务器调用**：
   - 每个MCP服务器有独立的连接和状态管理
   - 用户会话中可以顺序调用多个不同服务器的工具
   - 工具调用采用迭代式工作流，每次一个工具，结果影响后续工具的选择和参数

这种架构设计使Cline能够灵活地扩展其功能，允许开发者创建自定义的MCP服务器来提供专门的工具和资源，而不需要修改Cline的核心代码。
