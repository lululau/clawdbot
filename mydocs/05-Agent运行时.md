# 05-Agent运行时

## Pi代理集成

OpenClaw使用`@mariozechner/pi-agent-core`作为AI运行时。

### 核心依赖

```typescript
import {
  createAgent,
  type Agent,
  type Tool
} from '@mariozechner/pi-agent-core';
```

## 文件结构

```
src/agents/
├── pi-embedded-runner/       # Pi嵌入运行器
├── pi-embedded-subscribe/   # 事件订阅
├── model-selection/         # 模型选择
├── model-auth/             # 模型认证
├── pi-tools/               # 工具适配
├── context/                # 上下文管理
├── session-utils/          # 会话工具
└── subagent-registry/       # 子Agent注册
```

## Agent启动

```typescript
// src/agents/pi-embedded-runner/subscribe.ts
async function startAgent(sessionId: string): Promise<AgentHandle> {
  // 1. 加载模型配置
  const modelConfig = await resolveModel(sessionId);
  const authProfile = await resolveAuth(modelConfig.modelId);

  // 2. 创建Agent实例
  const agent = await createAgent({
    model: modelConfig.modelId,
    apiKey: authProfile.apiKey,
    systemPrompt: await loadSystemPrompt(sessionId),
    tools: await loadTools(sessionId),
    onEvent: handleAgentEvent
  });

  // 3. 订阅事件
  agent.subscribe('message.delta', handleMessageDelta);
  agent.subscribe('tool.result', handleToolResult);
  agent.subscribe('error', handleAgentError);

  // 4. 运行
  await agent.run({
    input: 'Start conversation',
    sessionId
  });

  return agent;
}
```

## 上下文管理

```typescript
// src/agents/context/
async function buildContext(sessionId: string): Promise<AgentContext> {
  const session = await sessionManager.get(sessionId);

  // 1. 系统提示词
  const systemPrompt = await loadSystemPrompt();
  const soul = await loadSoul(sessionId);

  // 2. 会话历史
  const history = await session.getRecentMessages(20);
  const compressed = await compressHistory(history);

  // 3. 用户配置
  const config = getConfig().agents?.defaults;

  return {
    sessionId,
    systemPrompt: `${systemPrompt}\n\nPersonality:\n${soul}`,
    history: compressed,
    config
  };
}
```

## 模型选择

### 提供商支持

```typescript
// src/agents/model-selection/
const providers = {
  anthropic: 'anthropic',
  openai: 'openai',
  openresponses: 'openresponses',
  ollama: 'ollama',
  together: 'together',
  // ... 更多提供商
};

async function selectModel(sessionConfig: SessionConfig): Promise<ModelInfo> {
  // 1. 解析模型ID
  const [provider, model] = parseModelId(sessionConfig.model);

  // 2. 检查认证
  const auth = await getAuthProvider(provider);
  const apiKey = await auth.getApiKey(sessionConfig.model);

  // 3. 验证模型
  const available = await listAvailableModels(provider, apiKey);
  if (!available.includes(model)) {
    throw new Error(`Model ${model} not available`);
  }

  return { provider, model, apiKey };
}
```

### 认证轮换

```typescript
// src/agents/model-auth/
class AuthProfileRotator {
  private profiles: AuthProfile[] = [];
  private currentIndex = 0;

  async rotate(): Promise<AuthProfile> {
    // 跳过当前profile
    this.currentIndex = (this.currentIndex + 1) % this.profiles.length;
    return this.profiles[this.currentIndex];
  }

  async verifyAll(): Promise<Map<string, boolean>> {
    const results = new Map();
    for (const profile of this.profiles) {
      const valid = await testApiKey(profile);
      results.set(profile.id, valid);
    }
    return results;
  }
}
```

## 工具调用

### 工具注册

```typescript
// src/agents/pi-tools/
interface PiToolAdapter {
  name: string;
  description: string;
  parameters: any;
  
  toPiTool(): Tool;
}

// 注册内置工具
const builtInTools: PiToolAdapter[] = [
  {
    name: 'bash',
    description: 'Execute shell commands',
    parameters: {
      type: 'object',
      properties: {
        command: { type: 'string' },
        directory: { type: 'string' }
      }
    },
    toPiTool(): () => ({
      name: 'bash',
      description: this.description,
      input_schema: { type: 'string', description: 'Command to execute' },
      function: async (input: string) => executeCommand(input)
    })
  }
  ],
  {
    name: 'browser',
    description: 'Control browser via CDP',
    parameters: { type: 'object', properties: { url: { type: 'string' } } },
    toPiTool(): () => ({ /* ... */ })
  }
];
```

### 工具执行

```typescript
// 执行工具调用
async function invokeTool(
  toolName: string,
  params: any,
  context: AgentContext
): Promise<ToolResult> {
  // 1. 查找工具
  const tool = toolRegistry.get(toolName);
  if (!tool) {
    throw new ToolNotFoundError(toolName);
  }

  // 2. 权限检查
  if (!context.session.hasPermission(tool)) {
    throw new ToolPermissionError(`Tool ${toolName} not allowed`);
  }

  // 3. 执行
  const result = await tool.function(params, context);

  // 4. 保存结果
  await saveToolResult(context.sessionId, toolName, result);

  return result;
}
```

## 流式响应处理

### 消息分块

```typescript
// src/agents/pi-embedded-subscribe/handlers/messages.ts
agent.subscribe('message.delta', async (event) => {
  const { sessionId, delta } = event;

  // 1. 发送到Gateway
  await gateway.sendEvent({
    type: 'chat.event',
    event: 'message.delta',
    data: { sessionId, delta }
  });

  // 2. 发送到渠道
  const session = sessionManager.get(sessionId);
  const channel = channelRegistry.get(session.channel);
  await channel.send({
    sessionId,
    text: delta
  });
});
```

### 工具流式

```typescript
// src/agents/pi-embedded-subscribe/handlers/tools.ts
agent.subscribe('tool.result', async (event) => {
  const { sessionId, tool, result } = event;

  // 格式化工具输出
  const formatted = formatToolOutput(tool, result);

  // 流式发送
  for (const chunk of formatChunks(formatted)) {
    await gateway.sendEvent({
      type: 'chat.event',
      event: 'message.delta',
      data: { sessionId, delta: chunk }
    });
  }
});
```

## 会话管理

### 会话存储

```typescript
// src/agents/session-utils/
interface SessionStore {
  get(sessionId: string): Promise<Session | null>;
  create(session: CreateSessionParams): Promise<Session>;
  update(sessionId: string, updates: Partial<Session>): Promise<void>;
  delete(sessionId: string): Promise<void>;
  list(filter?: SessionFilter): Promise<Session[]>;
}

// SQLite存储
const db = new SessionDatabase('~/.openclaw/sessions.db');
```

### 会话状态

```typescript
enum SessionState {
  Active = 'active',
  WaitingForTool = 'waiting_for_tool',
  Thinking = 'thinking',
  Compacting = 'compacting',
  Closed = 'closed'
}

async function setSessionState(
  sessionId: string,
  state: SessionState
): Promise<void> {
  await sessionStore.update(sessionId, { state });
  await notifyStateChange(sessionId, state);
}
```

## 子Agent系统

### 子Agent注册

```typescript
// src/agents/subagent-registry/
interface SubAgent {
  id: string;
  parentId?: string;
  config: AgentConfig;
  agent: Agent;
}

class SubAgentRegistry {
  private agents = new Map<string, SubAgent>();

  async spawn(config: SpawnSubAgentParams): Promise<SubAgent> {
    const id = generateSubAgentId();
    const agent = await createAgent(config);
    
    const subAgent: SubAgent = {
      id,
      parentId: config.parentId,
      config,
      agent
    };

    this.agents.set(id, subAgent);
    return subAgent;
  }

  async communicate(
    fromId: string,
    toId: string,
      message: string
  ): Promise<void> {
    const from = this.agents.get(fromId);
    const to = this.agents.get(toId);
    await from.agent.sendMessage(to.id, message);
  }
}
```

## 错误处理

### Agent错误

```typescript
class AgentError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly recoverable: boolean = false
  ) {
    super(message);
  }
}

// 错误类型
class ToolExecutionError extends AgentError {
  constructor(tool: string, error: Error) {
    super(`Tool execution failed: ${tool}`, 'TOOL_EXECUTION_ERROR', true);
    this.tool = tool;
    this.originalError = error;
  }
}

class ModelAuthError extends AgentError {
  constructor(model: string, error: Error) {
    super(`Model auth failed: ${model}`, 'MODEL_AUTH_ERROR');
    this.model = model;
  }
}
```

### 重试逻辑

```typescript
// src/agents/pi-embedded-runner/guard.ts
async function executeWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  let lastError: Error;

  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      if (!isRetryable(error)) {
        continue;
      }
    }
  }

  throw lastError;
}
```

## 学习重点

1. **Pi代理集成**: 理解Agent创建和配置
2. **工具系统**: 掌握工具注册、执行和流式处理
3. **上下文管理**: 学习系统提示词和历史压缩
4. **会话管理**: 理解Session生命周期和状态
5. **子Agent**: 理解多Agent架构

## 相关文件

- [src/agents/pi-embedded-runner/](../../src/agents/pi-embedded-runner/)
- [src/agents/pi-embedded-subscribe/](../../src/agents/pi-embedded-subscribe/)
- [src/agents/model-selection/](../../src/agents/model-selection/)
- [src/agents/pi-tools/](../../src/agents/pi-tools/)
- [src/agents/session-utils/](../../src/agents/session-utils/)

## 下一步

- [06-插件系统.md](./06-插件系统.md) - Plugin架构
- [07-配置管理.md](./07-配置管理.md) - 配置系统
