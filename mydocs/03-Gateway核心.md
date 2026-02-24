# 03-Gateway核心

## 核心职责

Gateway是OpenClaw的**中央控制平面**,提供:
- WebSocket服务器(ws://127.0.0.1:18789)
- 会话生命周期管理
- 请求路由和分发
- 客户端认证和授权

## 文件结构

```
src/gateway/
├── server.impl.ts           # 核心实现
├── server.ts              # 服务器入口
├── server-http.ts         # HTTP端点
├── server.ws-runtime.ts    # WebSocket运行时
├── server-methods/        # 方法处理器
│   ├── chat.ts          # 聊天相关
│   ├── agent.ts         # Agent调用
│   ├── sessions.ts      # 会话管理
│   ├── tools.ts        # 工具调用
│   └── nodes.ts        # 节点管理
├── auth.ts               # 认证逻辑
├── client.ts             # 客户端管理
└── session-utils/       # 会话工具
```

## 启动流程

```typescript
// src/gateway/server.impl.ts
async function startGateway(options: GatewayOptions) {
  // 1. 配置加载
  const config = await loadConfig();

  // 2. 端口绑定
  const server = express();
  const wsServer = new WebSocket.Server({ noServer: true });

  // 3. HTTP服务器
  setupHttpEndpoints(server, config);

  // 4. WebSocket服务器
  setupWebSocketServer(wsServer, config);

  // 5. 附加到HTTP
  server.on('upgrade', handleWebSocketUpgrade);

  // 6. 会话恢复
  await restoreSessions();

  // 7. 节点连接
  await connectNodes();

  // 8. 启动监听
  server.listen(options.port, options.host);
}
```

## WebSocket协议

### 连接建立

```typescript
// 握手流程
Client → Gateway: {
  type: "gateway.hello",
  version: "1.0.0",
  authToken?: string,
  capabilities?: string[]
}

Gateway → Client: {
  type: "gateway.welcome",
  serverId: string,
  configHash: string,
  supportedMethods: string[]
}
```

### 方法调用

```typescript
// JSON-RPC 2.0风格
interface RpcRequest {
  id: number;
  method: string;
  params?: any;
}

interface RpcResponse {
  id: number;
  result?: any;
  error?: {
    code: number;
    message: string;
    data?: any;
  };
}

// 示例
await ws.send({
  id: 1,
  method: "chat.send",
  params: { sessionId: "main", message: "Hello" }
});
```

### 事件流

```typescript
// 服务器主动推送
interface EventStream {
  type: "chat.event" | "agent.event" | "session.event";
  event: string;
  data: any;
  id: string;
}

// 消息流式发送
{
  type: "chat.event",
  event: "message.delta",
  data: { text: "Hello" },
  id: "msg-123"
}

// 完整消息
{
  type: "chat.event",
  event: "message.complete",
  data: { fullText: "Hello!" },
  id: "msg-123"
}
```

## 会话管理

### Session数据结构

```typescript
interface Session {
  id: string;              // 唯一标识
  type: 'main' | 'group' | 'dm';
  channel: string;         // 渠道标识
  peer: string;            // 对等端标识
  state: SessionState;
  config: SessionConfig;
  createdAt: number;
  updatedAt: number;
}

enum SessionState {
  Active = "active",
  Idle = "idle",
  Compacting = "compacting",
  Closed = "closed"
}
```

### 生命周期

```typescript
// 创建
const session = await sessionManager.create({
  channel: "telegram",
  peer: "+1234567890",
  type: "dm"
});

// 激活
await sessionManager.activate(sessionId);

// 消息处理
await sessionManager.handleMessage(sessionId, message);

// 压缩(节省token)
await sessionManager.compact(sessionId);

// 关闭
await sessionManager.close(sessionId);
```

## 路由系统

```typescript
// src/gateway/server-methods.ts
interface MethodHandler {
  method: string;
  handler: (params: any, context: RequestContext) => Promise<any>;
  requireAuth?: boolean;
  requirePermission?: string[];
}

// 方法路由
const methodMap = new Map<string, MethodHandler>([
  ['chat.send', { handler: handleChatSend, requireAuth: true }],
  ['agent.invoke', { handler: handleAgentInvoke, requireAuth: true }],
  ['sessions.list', { handler: handleSessionsList, requireAuth: true }],
  ['tools.invoke', { handler: handleToolInvoke, requireAuth: true }],
  ['node.list', { handler: handleNodeList, requireAuth: true }],
  ['gateway.startup', { handler: handleStartup, requireAuth: false }],
  ['ping', { handler: handlePing, requireAuth: false }]
]);

// 请求分发
async function dispatchRequest(
  request: RpcRequest,
  client: ClientConnection
): Promise<RpcResponse> {
  const handler = methodMap.get(request.method);
  if (!handler) {
    return { id: request.id, error: { code: -32601, message: 'Unknown method' } };
  }

  // 权限检查
  if (handler.requireAuth && !client.authenticated) {
    return { id: request.id, error: { code: -32602, message: 'Unauthorized' } };
  }

  try {
    const result = await handler.handler(request.params, { client });
    return { id: request.id, result };
  } catch (error) {
    return { id: request.id, error: { code: -32603, message: error.message } };
  }
}
```

## 认证机制

### Token认证

```typescript
// src/gateway/auth.ts
interface AuthContext {
  clientId: string;
  authenticated: boolean;
  permissions: string[];
  session?: Session;
}

// 令牌验证
async function validateAuthToken(
  token: string
): Promise<AuthContext> {
  // 1. 配置中的令牌
  const configToken = getConfig().gateway?.authToken;

  // 2. 比对
  if (token !== configToken) {
    throw new Error('Invalid auth token');
  }

  // 3. 加载客户端配置
  const clientConfig = loadClientConfig(clientId);

  return {
    clientId: generateClientId(),
    authenticated: true,
    permissions: clientConfig.permissions || [],
  };
}
```

### 节点配对

```typescript
// 节点配对流程
1. Gateway发送配对码
2. 用户在设备上确认配对
3. Gateway验证配对
4. 建立信任连接
```

## 工具调用流程

```typescript
// 工具调用处理
async function handleToolInvoke(
  params: ToolInvokeParams,
  context: RequestContext
): Promise<ToolResult> {
  const { sessionId, tool, args } = params;
  const session = sessionManager.get(sessionId);

  // 1. 权限检查
  if (!session.hasPermission(tool)) {
    throw new ToolPermissionError(`Tool ${tool} not allowed`);
  }

  // 2. 工具查找
  const toolHandler = toolRegistry.get(tool);
  if (!toolHandler) {
    throw new ToolNotFoundError(tool);
  }

  // 3. 执行
  const result = await toolHandler.execute(args, {
    session,
    client: context.client
  });

  // 4. 结果格式化
  return formatToolResult(result);
}
```

## 客户端管理

```typescript
// src/gateway/client.ts
class ClientConnection {
  id: string;
  socket: WebSocket;
  authContext: AuthContext | null;
  subscriptions: Set<string>;

  // 发送事件
  sendEvent(type: string, event: string, data: any) {
    this.send({
      type,
      event,
      data,
      id: generateEventId(),
      timestamp: Date.now()
    });
  }

  // 批量发送(流式)
  sendStream(type: string, chunks: any[]) {
    for (const chunk of chunks) {
      this.send({ type, event: 'chunk', data: chunk });
    }
  }

  // 断开
  disconnect(reason: string) {
    this.socket.close(1000, reason);
  }
}
```

## 配置热重载

```typescript
// 监听配置变化
watchConfigFile(async (config) => {
  // 1. 验证配置
  const validation = validateConfig(config);
  if (!validation.valid) {
    logger.warn('Invalid config:', validation.errors);
    return;
  }

  // 2. 更新Gateway
  gateway.updateConfig(config);

  // 3. 通知客户端
  broadcastEvent('config.reloaded', { configHash: getConfigHash(config) });

  // 4. 重启依赖服务
  if (needsRestart(validation)) {
    await gateway.restart();
  }
});
```

## 学习重点

1. **WebSocket通信**: 理解JSON-RPC协议和事件流
2. **会话管理**: 掌握Session生命周期和状态
3. **路由系统**: 学习方法分发和权限检查
4. **认证流程**: 理解Token认证和节点配对
5. **工具集成**: 理解工具注册和调用机制

## 相关文件

- [src/gateway/server.impl.ts](../../src/gateway/server.impl.ts)
- [src/gateway/server-methods/](../../src/gateway/server-methods/)
- [src/gateway/auth.ts](../../src/gateway/auth.ts)
- [src/gateway/client.ts](../../src/gateway/client.ts)

## 下一步

继续学习:
- [04-Channels系统.md](./04-Channels系统.md) - 渠道实现
- [05-Agent运行时.md](./05-Agent运行时.md) - Agent集成
