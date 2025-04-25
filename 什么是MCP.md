
# Model Context Protocol TypeScript SDK 文档

## 概述
Model Context Protocol（MCP）允许应用程序以标准化方式为LLM提供上下文，将上下文提供与实际LLM交互的关注点分离。这个TypeScript SDK实现了完整的MCP规范，便于：
- 构建可连接到任何MCP服务器的MCP客户端
- 创建暴露资源、提示和工具的MCP服务器
- 使用stdio和Streamable HTTP等标准传输方式
- 处理所有MCP协议消息和生命周期事件

## 安装
```bash
npm install @modelcontextprotocol/sdk
```

## 快速入门
以下是创建一个暴露计算器工具和数据的简单MCP服务器的示例：
```typescript
import { McpServer, ResourceTemplate } from "@modelcontextprotocol/sdk/server/mcp.js"; 
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js"; 
import { z } from "zod"; 

// 创建MCP服务器
const server = new McpServer({
  name: "Demo",
  version: "1.0.0" 
}); 

// 添加加法工具
server.tool("add",
  { a: z.number(), b: z.number() },
  async ({ a, b }) => ({
    content: [{ type: "text", text: String(a + b) }]
  }) 
); 

// 添加动态问候资源
server.resource(
  "greeting",
  new ResourceTemplate("greeting://{name}", { list: undefined }),
  async (uri, { name }) => ({
    contents: [{
      uri: uri.href,
      text: `Hello, ${name}!`
    }]
  }) 
); 

// 在标准输入输出上开始接收和发送消息
const transport = new StdioServerTransport(); 
await server.connect(transport);
```

## 什么是MCP？
Model Context Protocol（MCP）允许您构建以安全、标准化方式向LLM应用程序暴露数据和功能的服务器。可以将其视为专门为LLM交互设计的Web API。MCP服务器可以：
- 通过资源（Resources）暴露数据（类似GET端点，用于将信息加载到LLM的上下文中）
- 通过工具（Tools）提供功能（类似POST端点，用于执行代码或产生副作用）
- 通过提示（Prompts）定义交互模式（LLM交互的可重用模板）

## 核心概念
### 服务器（Server）
McpServer是与MCP协议交互的核心接口，处理连接管理、协议合规和消息路由：
```typescript
const server = new McpServer({
  name: "My App",
  version: "1.0.0" 
});
```

### 资源（Resources）
资源用于向LLM暴露数据，类似于REST API中的GET端点，提供数据但不应执行大量计算或产生副作用：
```typescript
// 静态资源
server.resource(
  "config",
  "config://app",
  async (uri) => ({
    contents: [{
      uri: uri.href,
      text: "App configuration here"
    }]
  }) 
); 

// 带参数的动态资源
server.resource(
  "user-profile",
  new ResourceTemplate("users://{userId}/profile", { list: undefined }),
  async (uri, { userId }) => ({
    contents: [{
      uri: uri.href,
      text: `Profile data for user ${userId}`
    }]
  }) 
);
```

### 工具（Tools）
工具允许LLM通过服务器执行操作，与资源不同，工具预期执行计算并产生副作用：
```typescript
// 带参数的简单工具
server.tool(
  "calculate-bmi",
  {
    weightKg: z.number(),
    heightM: z.number()
  },
  async ({ weightKg, heightM }) => ({
    content: [{
      type: "text",
      text: String(weightKg / (heightM * heightM))
    }]
  }) 
); 

// 带外部API调用的异步工具
server.tool(
  "fetch-weather",
  { city: z.string() },
  async ({ city }) => {
    const response = await fetch(`https://api.weather.com/${city}`);
    const data = await response.text();
    return {
      content: [{ type: "text", text: data }]
    };
  } 
);
```

### 提示（Prompts）
提示是可重用的模板，帮助LLM与服务器有效交互：
```typescript
server.prompt(
  "review-code",
  { code: z.string() },
  ({ code }) => ({
    messages: [{
      role: "user",
      content: {
        type: "text",
        text: `Please review this code:\n\n${code}`
      }
    }]
  }) 
);
```

## 运行服务器
TypeScript中的MCP服务器需要连接到传输层以与客户端通信，启动服务器的方式取决于所选的传输方式。

### stdio
适用于命令行工具和直接集成：
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js"; 
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js"; 

const server = new McpServer({
  name: "example-server",
  version: "1.0.0" 
}); 

// 设置服务器资源、工具和提示...

const transport = new StdioServerTransport(); 
await server.connect(transport);
```

### Streamable HTTP
适用于远程服务器，设置处理客户端请求和服务器到客户端通知的Streamable HTTP传输层。

#### 有会话管理
```typescript
import express from "express"; 
import { randomUUID } from "node:crypto"; 
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js"; 
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js"; 
import { isInitializeRequest } from "@modelcontextprotocol/sdk/types.js";

const app = express(); 
app.use(express.json()); 

// 按会话ID存储传输层的映射
const transports: { [sessionId: string]: StreamableHTTPServerTransport } = {}; 

// 处理客户端到服务器通信的POST请求
app.post('/mcp', async (req, res) => {
  const sessionId = req.headers['mcp-session-id'] as string | undefined;
  let transport: StreamableHTTPServerTransport;

  if (sessionId && transports[sessionId]) {
    transport = transports[sessionId];
  } else if (!sessionId && isInitializeRequest(req.body)) {
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      onsessioninitialized: (sessionId) => {
        transports[sessionId] = transport;
      }
    });

    transport.onclose = () => {
      if (transport.sessionId) {
        delete transports[transport.sessionId];
      }
    };
    const server = new McpServer({
      name: "example-server",
      version: "1.0.0"
    });

    // 设置服务器资源、工具和提示...

    await server.connect(transport);
  } else {
    res.status(400).json({
      jsonrpc: '2.0',
      error: {
        code: -32000,
        message: 'Bad Request: No valid session ID provided',
      },
      id: null,
    });
    return;
  }

  await transport.handleRequest(req, res, req.body); 
}); 

// 处理GET和DELETE请求的可重用处理程序
const handleSessionRequest = async (req: express.Request, res: express.Response) => {
  const sessionId = req.headers['mcp-session-id'] as string | undefined;
  if (!sessionId || !transports[sessionId]) {
    res.status(400).send('Invalid or missing session ID');
    return;
  }
  
  const transport = transports[sessionId];
  await transport.handleRequest(req, res); 
}; 

// 通过SSE处理服务器到客户端通知的GET请求
app.get('/mcp', handleSessionRequest); 
// 处理会话终止的DELETE请求
app.delete('/mcp', handleSessionRequest); 

app.listen(3000);
```

#### 无会话管理（无状态）
适用于简单用例：
```typescript
const app = express(); 
app.use(express.json()); 

app.post('/mcp', async (req: Request, res: Response) => {
  try {
    const server = getServer(); 
    const transport: StreamableHTTPServerTransport = new StreamableHTTPServerTransport({
      sessionIdGenerator: undefined,
    });
    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
    res.on('close', () => {
      console.log('Request closed');
      transport.close();
      server.close();
    });
  } catch (error) {
    console.error('Error handling MCP request:', error);
    if (!res.headersSent) {
      res.status(500).json({
        jsonrpc: '2.0',
        error: {
          code: -32603,
          message: 'Internal server error',
        },
        id: null,
      });
    }
  } 
}); 

app.get('/mcp', async (req: Request, res: Response) => {
  console.log('Received GET MCP request');
  res.writeHead(405).end(JSON.stringify({
    jsonrpc: "2.0",
    error: {
      code: -32000,
      message: "Method not allowed."
    },
    id: null
  })); 
}); 

app.delete('/mcp', async (req: Request, res: Response) => {
  console.log('Received DELETE MCP request');
  res.writeHead(405).end(JSON.stringify({
    jsonrpc: "2.0",
    error: {
      code: -32000,
      message: "Method not allowed."
    },
    id: null
  })); 
}); 

const PORT = 3000; 
app.listen(PORT, () => {
  console.log(`MCP Stateless Streamable HTTP Server listening on port ${PORT}`); 
});
```

这种无状态方法适用于：
- 简单API包装器
- 每个请求独立的RESTful场景
- 无共享会话状态的水平扩展部署

## 测试和调试
要测试服务器，可以使用MCP Inspector，更多信息请参阅其README。

## 示例
### 回显服务器
演示资源、工具和提示的简单服务器：
```typescript
import { McpServer, ResourceTemplate } from "@modelcontextprotocol/sdk/server/mcp.js"; 
import { z } from "zod"; 

const server = new McpServer({
  name: "Echo",
  version: "1.0.0" 
}); 

server.resource(
  "echo",
  new ResourceTemplate("echo://{message}", { list: undefined }),
  async (uri, { message }) => ({
    contents: [{
      uri: uri.href,
      text: `Resource echo: ${message}`
    }]
  }) 
); 

server.tool(
  "echo",
  { message: z.string() },
  async ({ message }) => ({
    content: [{ type: "text", text: `Tool echo: ${message}` }]
  }) 
); 

server.prompt(
  "echo",
  { message: z.string() },
  ({ message }) => ({
    messages: [{
      role: "user",
      content: {
        type: "text",
        text: `Please process this message: ${message}`
      }
    }]
  }) 
);
```

### SQLite资源管理器
展示数据库集成的更复杂示例：
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js"; 
import sqlite3 from "sqlite3"; 
import { promisify } from "util"; 
import { z } from "zod"; 

const server = new McpServer({
  name: "SQLite Explorer",
  version: "1.0.0" 
}); 

// 创建数据库连接的辅助函数
const getDb = () => {
  const db = new sqlite3.Database("database.db");
  return {
    all: promisify<string, any[]>(db.all.bind(db)),
    close: promisify(db.close.bind(db))
  }; 
}; 

server.resource(
  "schema",
  "schema://main",
  async (uri) => {
    const db = getDb();
    try {
      const tables = await db.all(
        "SELECT sql FROM sqlite_master WHERE type='table'"
      );
      return {
        contents: [{
          uri: uri.href,
          text: tables.map((t: {sql: string}) => t.sql).join("\n")
        }]
      };
    } finally {
      await db.close();
    }
  } 
); 

server.tool(
  "query",
  { sql: z.string() },
  async ({ sql }) => {
    const db = getDb();
    try {
      const results = await db.all(sql);
      return {
        content: [{
          type: "text",
          text: JSON.stringify(results, null, 2)
        }]
      };
    } catch (err: unknown) {
      const error = err as Error;
      return {
        content: [{
          type: "text",
          text: `Error: ${error.message}`
        }],
        isError: true
      };
    } finally {
      await db.close();
    }
  } 
);
```

## 高级用法
### 动态服务器
可以在服务器连接后添加、更新或删除工具、提示和资源，这将自动发出相应的`listChanged`通知：
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js"; 
import { z } from "zod"; 

const server = new McpServer({
  name: "Dynamic Example",
  version: "1.0.0" 
}); 

const listMessageTool = server.tool(
  "listMessages",
  { channel: z.string() },
  async ({ channel }) => ({
    content: [{ type: "text", text: await listMessages(channel) }]
  }) 
); 

const putMessageTool = server.tool(
  "putMessage",
  { channel: z.string(), message: z.string() },
  async ({ channel, message }) => ({
    content: [{ type: "text", text: await putMessage(channel, string) }]
  }) 
); 

putMessageTool.disable(); 

const upgradeAuthTool = server.tool(
  "upgradeAuth",
  { permission: z.enum(["write", "admin"]) },
  async ({ permission }) => {
    const { ok, err, previous } = await upgradeAuthAndStoreToken(permission);
    if (!ok) return {content: [{ type: "text", text: `Error: ${err}` }]};

    if (previous === "read") {
      putMessageTool.enable();
    }

    if (permission === 'write') {
      upgradeAuthTool.update({
        paramSchema: { permission: z.enum(["admin"]) }, 
      });
    } else {
      upgradeAuthTool.remove();
    }
  } 
); 

const transport = new StdioServerTransport(); 
await server.connect(transport);
```

### 低级服务器
如需更多控制，可以直接使用低级Server类：
```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js"; 
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js"; 
import {
  ListPromptsRequestSchema,
  GetPromptRequestSchema 
} from "@modelcontextprotocol/sdk/types.js"; 

const server = new Server(
  {
    name: "example-server",
    version: "1.0.0"
  },
  {
    capabilities: {
      prompts: {}
    }
  } 
); 

server.setRequestHandler(ListPromptsRequestSchema, async () => {
  return {
    prompts: [{
      name: "example-prompt",
      description: "An example prompt template",
      arguments: [{
        name: "arg1",
        description: "Example argument",
        required: true
      }]
    }]
  }; 
}); 

server.setRequestHandler(GetPromptRequestSchema, async (request) => {
  if (request.params.name !== "example-prompt") {
    throw new Error("Unknown prompt");
  }
  return {
    description: "Example prompt",
    messages: [{
      role: "user",
      content: {
        type: "text",
        text: "Example prompt text"
      }
    }]
  }; 
}); 

const transport = new StdioServerTransport(); 
await server.connect(transport);
```

### 编写MCP客户端
SDK提供高级客户端接口：
```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js"; 
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js"; 

const transport = new StdioClientTransport({
  command: "node",
  args: ["server.js"] 
}); 

const client = new Client(
  {
    name: "example-client",
    version: "1.0.0"
  } 
); 

await client.connect(transport); 

// 列出提示
const prompts = await client.listPrompts(); 
// 获取提示
const prompt = await client.getPrompt({
  name: "example-prompt",
  arguments: {
    arg1: "value"
  } 
}); 
// 列出资源
const resources = await client.listResources(); 
// 读取资源
const resource = await client.readResource({
  uri: "file:///example.txt" 
}); 
// 调用工具
const result = await client.callTool({
  name: "example-tool",
  arguments: {
    arg1: "value"
  } 
});
```

### 向上游代理授权请求
可以将OAuth请求代理到外部授权提供程序：
```typescript
import express from 'express'; 
import { ProxyOAuthServerProvider, mcpAuthRouter } from '@modelcontextprotocol/sdk'; 

const app = express(); 
const proxyProvider = new ProxyOAuthServerProvider({
    endpoints: {
        authorizationUrl: "https://auth.external.com/oauth2/v1/authorize",
        tokenUrl: "https://auth.external.com/oauth2/v1/token",
        revocationUrl: "https://auth.external.com/oauth2/v1/revoke",
    },
    verifyAccessToken: async (token) => {
        return {
            token,
            clientId: "123",
            scopes: ["openid", "email", "profile"],
        }
    },
    getClient: async (client_id) => {
        return {
            client_id,
            redirect_uris: ["http://localhost:3000/callback"],
        }
    } 
}); 

app.use(mcpAuthRouter({
    provider: proxyProvider,
    issuerUrl: new URL("http://auth.external.com"),
    baseUrl: new URL("http://mcp.example.com"),
    serviceDocumentationUrl: new URL("https://docs.example.com/"), 
}));
```

这种设置允许您：
- 将OAuth请求转发到外部提供程序
- 添加自定义令牌验证逻辑
- 管理客户端注册
- 提供自定义文档URL
- 在委托给外部提供程序的同时控制OAuth流程

## 向后兼容性
### 客户端兼容性
需要同时支持Streamable HTTP和旧版SSE服务器的客户端：
```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js"; 
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js"; 
import { SSEClientTransport } from "@modelcontextprotocol/sdk/client/sse.js"; 

let client: Client|undefined = undefined; 
const baseUrl = new URL(url); 
try {
  client = new Client({
    name: 'streamable-http-client',
    version: '1.0.0'
  });
  const transport = new StreamableHTTPClientTransport(
    new URL(baseUrl)
  );
  await client.connect(transport);
  console.log("Connected using Streamable HTTP transport"); 
} catch (error) {
  console.log("Streamable HTTP connection failed, falling back to SSE transport"); 
  client = new Client({
    name: 'sse-client',
    version: '1.0.0'
  });
  const sseTransport = new SSEClientTransport(baseUrl);
  await client.connect(sseTransport);
  console.log("Connected using SSE transport"); 
}
```

### 服务器兼容性
需要同时支持Streamable HTTP和旧版客户端的服务器：
```typescript
import express from "express"; 
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js"; 
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js"; 
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js"; 

const server = new McpServer({
  name: "backwards-compatible-server",
  version: "1.0.0" 
}); 

const app = express(); 
app.use(express.json()); 

const transports = {
  streamable: {} as Record<string, StreamableHTTPServerTransport>,
  sse: {} as Record<string, SSEServerTransport> 
}; 

// 现代Streamable HTTP端点
app.all('/mcp', async (req, res) => {
  // 处理现代客户端的Streamable HTTP传输
  // 实现如“有会话管理”示例所示...
}); 

// 旧版SSE客户端的遗留SSE端点
app.get('/sse', async (req, res) => {
  const transport = new SSEServerTransport('/messages', res);
  transports.sse[transport.sessionId] = transport;
  
  res.on("close", () => {
    delete transports.sse[transport.sessionId];
  });
  
  await server.connect(transport); 
}); 

// 旧版客户端的遗留消息端点
app.post('/messages', async (req, res) => {
  const sessionId = req.query.sessionId as string;
  const transport = transports.sse[sessionId];
  if (transport) {
    await transport.handlePostMessage(req, res, req.body);
  } else {
    res.status(400).send('No transport found for sessionId');
  } 
}); 

app.listen(3000);
```

注意：SSE传输已被弃用，推荐使用Streamable HTTP，现有SSE实现应计划迁移。

## 文档
- [Model Context Protocol文档](链接)
- [MCP规范](链接)
- [示例服务器](链接)

## 贡献
欢迎在GitHub上提交问题和拉取请求：https://github.com/modelcontextprotocol/typescript-sdk。

## 许可证
本项目采用MIT许可证，详情请参阅LICENSE文件。
