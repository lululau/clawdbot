# 04-Channels系统

## 架构概述

```
消息平台 → Channel接口 → 统一消息格式 → Gateway
```

## 核心接口

```typescript
// src/channels/base.ts
interface Channel {
  readonly name: string;
  readonly type: string;

  // 生命周期
  start(options: ChannelOptions): Promise<ChannelHandle>;
  stop(): Promise<void>;
  getStatus(): ChannelStatus;

  // 消息操作
  send(message: OutboundMessage): Promise<void>;
  sendTyping(peer: string, isTyping: boolean): void;
  sendPresence(presence: Presence): void;

  // 查询
  listConversations(): Promise<Conversation[]>;
  getConversation(id: string): Promise<Conversation>;
}

interface ChannelHandle {
  id: string;
  platform: string;
  isConnected: boolean;
}
```

## 渠道实现

### WhatsApp

**库**: @whiskeysockets/baileys

**关键文件**:
```
src/channels/whatsapp/
├── dock.ts              # WhatsApp连接管理
├── session.ts           # 会话处理
└── message.ts           # 消息格式化
```

**特点**:
- E2E加密
- 媒体支持(图片、文档、音频)
- 阅读回执
- 消息状态同步

### Telegram

**库**: grammY

**关键文件**:
```
src/channels/telegram/
├── bot.ts               # Bot封装
├── session.ts           # 会话处理
├── message.ts           # 消息处理
└── handlers/            # 命令处理
```

**特点**:
- Webhook/长轮询模式
- 富文本支持
- 内联按钮
- 文档/图片发送

### Slack

**库**: @slack/bolt

**特点**:
- Workspace事件
- Slash命令
- Modal/Block Kit
- 文件上传/下载

### Discord

**库**: discord.js

**特点**:
- 消息事件
- Slash命令
- 语音频道
- Thread支持

## 消息格式化

### 统一消息结构

```typescript
// src/channels/message.ts
interface UnifiedMessage {
  id: string;
  sessionId: string;

  // 发送者
  sender: {
    id: string;
    name: string;
    username?: string;
    avatar?: string;
  };

  // 内容
  content: string;
  attachments?: Attachment[];

  // 元数据
  metadata: {
    timestamp: number;
    replyTo?: string;
    isEdited: boolean;
    isFromBot?: boolean;
  };
}

interface Attachment {
  id: string;
  type: 'image' | 'video' | 'audio' | 'document';
  url: string;
  mimeType: string;
  size: number;
}
```

### 转换流程

```
WhatsApp Message → 解析 → 统一格式 → Gateway → Agent
Telegram Message  → 解析 → 统一格式 → Gateway → Agent
...
```

## 配置管理

### 渠道配置

```typescript
// src/config/channel-config.ts
interface ChannelConfig {
  enabled: boolean;
  allowFrom?: string[];
  allowlists?: Record<string, string[]>;

  // 群组设置
  groups?: Record<string, GroupConfig>;

  // 命令
  commands?: ChannelCommands;

  // 媒体
  mediaMaxMb?: number;
}

interface GroupConfig {
  requireMention?: boolean;
  commandPrefix?: string;
  readOnly?: boolean;
}
```

### 权限控制

```typescript
// src/channels/allow-from.ts
async function checkPermission(
  channel: string,
  peer: string,
  sessionType: SessionType
): Promise<PermissionResult> {
  // 1. 检查allowlist
  const allowed = isPeerAllowed(channel, peer);
  if (!allowed) {
    return { allowed: false, reason: 'not_in_allowlist' };
  }

  // 2. 检查DM配对
  if (sessionType === 'dm') {
    const paired = isPaired(channel, peer);
    if (!paired) {
      return { allowed: false, reason: 'needs_pairing' };
    }
  }

  // 3. 检查命令权限
  if (isAdminCommand()) {
    return { allowed: true, reason: 'admin_command' };
  }

  return { allowed: true };
}
```

## 错误处理

### 重试机制

```typescript
// src/channels/retry.ts
class RetryManager {
  private maxRetries = 3;
  private backoff = [1000, 2000, 5000];

  async execute<T>(
    fn: () => Promise<T>,
    context: string
  ): Promise<T> {
    for (let i = 0; i < this.maxRetries; i++) {
      try {
        return await fn();
      } catch (error) {
        if (this.isRetryable(error)) {
          await this.delay(this.backoff[i]);
        } else {
          throw error;
        }
      }
    }
    throw new MaxRetriesExceeded(context);
  }
}
```

## 扩展插件

### 插件接口

```typescript
// src/plugin-sdk/channel.ts
export interface ChannelPlugin {
  readonly name: string;
  readonly version: string;

  start(options: ChannelStartOptions): Promise<Channel>;
  stop(): Promise<void>;

  // 可选功能
  canHandleMessage?(message: any): boolean;
  transformMessage?(message: any): any;
}
```

### 插件加载

```typescript
// src/plugins/channel-loader.ts
async function loadChannelPlugins(): Promise<void> {
  const pluginDirs = [
    'extensions/msteams',
    'extensions/matrix',
    'extensions/zalo',
    'extensions/zalouser'
  ];

  for (const dir of pluginDirs) {
    const plugin = await importPlugin(dir);
    if (plugin) {
      channelRegistry.register(plugin);
    }
  }
}
```

## 学习重点

1. **Channel接口**: 理解统一抽象和消息格式
2. **具体实现**: 学习主要渠道的连接和消息处理
3. **权限控制**: 掌握allowlist和配对机制
4. **扩展机制**: 学习如何添加新渠道插件

## 相关文件

- [src/channels/](../../src/channels/)
- [src/config/channel-config.ts](../../src/config/channel-config.ts)
- [src/plugins/channel-*.ts](../../src/plugins/)

## 下一步

- [05-Agent运行时.md](./05-Agent运行时.md) - Agent集成
- [06-插件系统.md](./06-插件系统.md) - Plugin架构
