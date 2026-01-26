# Clawdbot 安装指南

本指南将帮助您从零开始安装和配置 Clawdbot，包括 iMessage、Discord 和 Slack 三个消息渠道的详细配置说明。

---

## 目录

1. [系统要求](#系统要求)
2. [安装 Clawdbot](#安装-clawdbot)
   - [快速安装（推荐）](#快速安装推荐)
   - [手动全局安装](#手动全局安装)
   - [从源码安装（开发者）](#从源码安装开发者)
3. [初始配置](#初始配置)
4. [iMessage 配置](#imessage-配置)
5. [Discord 配置](#discord-配置)
6. [Slack 配置](#slack-配置)
7. [DM 配对安全机制](#dm-配对安全机制)
8. [常见问题排查](#常见问题排查)

---

## 系统要求

- **Node.js**: >= 22
- **操作系统**: macOS、Linux 或 Windows（通过 WSL2）
- **pnpm**: 仅在从源码构建时需要
- **iMessage 特别要求**: 仅支持 macOS，需要 Messages 应用已登录

> **Windows 用户注意**: 强烈建议使用 WSL2（推荐 Ubuntu）。原生 Windows 未经充分测试，可能存在兼容性问题。

---

## 安装 Clawdbot

### 快速安装（推荐）

使用官方安装脚本，会自动安装 CLI 并运行初始配置向导：

```bash
curl -fsSL https://clawd.bot/install.sh | bash
```

**Windows（PowerShell）**:

```powershell
iwr -useb https://clawd.bot/install.ps1 | iex
```

安装完成后，如果跳过了初始配置，运行：

```bash
clawdbot onboard --install-daemon
```

### 手动全局安装

如果您已经安装了 Node.js：

```bash
# 使用 npm
npm install -g clawdbot@latest

# 或使用 pnpm
pnpm add -g clawdbot@latest
```

> **macOS 用户注意**: 如果全局安装了 libvips（通过 Homebrew），且 `sharp` 安装失败，请使用：
> ```bash
> SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install -g clawdbot@latest
> ```

安装完成后运行初始配置：

```bash
clawdbot onboard --install-daemon
```

### 从源码安装（开发者）

```bash
git clone https://github.com/clawdbot/clawdbot.git
cd clawdbot
pnpm install
pnpm ui:build    # 首次运行会自动安装 UI 依赖
pnpm build
clawdbot onboard --install-daemon
```

> **提示**: 如果没有全局安装，可以通过 `pnpm clawdbot ...` 运行命令。

---

## 初始配置

### 配置向导

运行初始配置向导：

```bash
clawdbot onboard --install-daemon
```

向导会引导您完成以下配置：

1. **Gateway 模式**: 本地或远程
2. **认证方式**: OAuth（推荐）或 API 密钥
3. **消息渠道**: WhatsApp、Telegram、Discord、Slack、iMessage 等
4. **配对默认设置**: 安全的 DM 访问控制
5. **后台服务**: 安装为系统服务（launchd/systemd）

### 配置文件位置

- **主配置文件**: `~/.clawdbot/clawdbot.json`
- **工作空间**: `~/clawd`（技能、提示词、记忆等）
- **凭证存储**: `~/.clawdbot/credentials/`
- **会话数据**: `~/.clawdbot/agents/<agentId>/sessions/`

### 验证安装

```bash
# 检查状态
clawdbot status

# 健康检查
clawdbot health

# 诊断问题
clawdbot doctor
```

---

## iMessage 配置

> **重要**: iMessage 仅支持 macOS，需要 Messages 应用已登录 Apple ID。

### 前提条件

1. macOS 系统，Messages 应用已登录
2. 安装 `imsg` CLI 工具
3. 授予全磁盘访问权限（访问 Messages 数据库）
4. 授予自动化权限（发送消息时）

### 安装 imsg

```bash
brew install steipete/tap/imsg
```

### 基本配置

在 `~/.clawdbot/clawdbot.json` 中添加：

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/<你的用户名>/Library/Messages/chat.db"
    }
  }
}
```

### 配置说明

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `enabled` | 启用/禁用 iMessage 渠道 | `false` |
| `cliPath` | imsg 可执行文件路径 | - |
| `dbPath` | Messages 数据库路径 | - |
| `dmPolicy` | DM 访问策略：`pairing`/`allowlist`/`open`/`disabled` | `pairing` |
| `groupPolicy` | 群组策略：`open`/`allowlist`/`disabled` | `allowlist` |
| `includeAttachments` | 是否包含附件 | `false` |
| `mediaMaxMb` | 媒体大小上限（MB） | `16` |
| `textChunkLimit` | 文本分块大小（字符） | `4000` |

### 使用独立 Bot macOS 用户（可选）

如果您希望 Bot 使用独立的 iMessage 身份（保持个人 Messages 干净）：

1. **创建独立 Apple ID**（例如：`my-cool-bot@icloud.com`）

2. **创建 macOS 用户**（例如：`clawdbot`）并登录

3. **在该用户下打开 Messages 并登录 Bot Apple ID**

4. **启用远程登录**：系统设置 → 通用 → 共享 → 远程登录

5. **安装 imsg**（在 Bot 用户下）：
   ```bash
   brew install steipete/tap/imsg
   ```

6. **配置 SSH 免密登录**：
   ```bash
   ssh <bot-macos-user>@localhost true
   ```

7. **创建 SSH 包装脚本**（`chmod +x`）：

   ```bash
   #!/usr/bin/env bash
   set -euo pipefail
   exec /usr/bin/ssh -o BatchMode=yes -o ConnectTimeout=5 -T <bot-macos-user>@localhost \
     "/usr/local/bin/imsg" "$@"
   ```

8. **更新配置**：

   ```json5
   {
     channels: {
       imessage: {
         enabled: true,
         accounts: {
           bot: {
             name: "Bot",
             enabled: true,
             cliPath: "/path/to/imsg-bot",
             dbPath: "/Users/<bot-macos-user>/Library/Messages/chat.db"
           }
         }
       }
     }
   }
   ```

### 远程 Mac 配置（通过 Tailscale）

如果 Gateway 运行在 Linux 主机上，但 iMessage 需要在 Mac 上运行：

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.clawdbot/scripts/imsg-ssh",
      remoteHost: "bot@mac-mini.tailnet-1234.ts.net",
      includeAttachments: true,
      dbPath: "/Users/bot/Library/Messages/chat.db"
    }
  }
}
```

SSH 包装脚本（`~/.clawdbot/scripts/imsg-ssh`）：

```bash
#!/usr/bin/env bash
exec ssh -T bot@mac-mini.tailnet-1234.ts.net imsg "$@"
```

### 首次运行注意事项

首次发送/接收消息可能需要在 *Bot macOS 用户* 中批准 GUI 提示（自动化 + 全磁盘访问）。如果 `imsg rpc` 卡住或退出：

1. 登录该用户（屏幕共享可以帮助）
2. 运行一次 `imsg chats --limit 1` 或 `imsg send ...`
3. 批准提示
4. 重试

---

## Discord 配置

### 前提条件

1. Discord 账户
2. Discord 开发者门户访问权限

### 步骤 1：创建 Discord 应用和 Bot

1. 访问 [Discord 开发者门户](https://discord.com/developers/applications)
2. 点击 **New Application**，输入应用名称
3. 进入 **Bot** → **Add Bot**
4. 复制 **Bot Token**（这是您需要配置的 token）

### 步骤 2：启用 Gateway Intents

在 **Bot** → **Privileged Gateway Intents** 中启用：

- ✅ **Message Content Intent**（必需，用于读取消息内容）
- ✅ **Server Members Intent**（推荐，用于成员/用户查找）

> **注意**: 不启用 Message Content Intent 会导致 Bot 连接但无法响应消息。

### 步骤 3：生成邀请链接

在 **OAuth2** → **URL Generator** 中：

**Scopes（范围）**:
- ✅ `bot`
- ✅ `applications.commands`（原生斜杠命令需要）

**Bot Permissions（权限）**:
- ✅ View Channels
- ✅ Send Messages
- ✅ Read Message History
- ✅ Embed Links
- ✅ Attach Files
- ✅ Add Reactions（可选但推荐）

复制生成的 URL，在浏览器中打开，选择服务器并安装 Bot。

### 步骤 4：获取 ID

1. Discord 桌面/网页版 → **用户设置** → **高级** → 启用 **开发者模式**
2. 右键点击获取 ID：
   - 服务器名称 → **复制服务器 ID**（Guild ID）
   - 频道（如 `#general`）→ **复制频道 ID**
   - 用户 → **复制用户 ID**

### 步骤 5：配置 Clawdbot

**方式一：通过环境变量**（推荐用于服务器）：

```bash
export DISCORD_BOT_TOKEN="your-bot-token"
```

**方式二：通过配置文件**：

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "YOUR_BOT_TOKEN"
    }
  }
}
```

### 完整配置示例

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "YOUR_BOT_TOKEN",
      groupPolicy: "allowlist",
      dm: {
        enabled: true,
        policy: "pairing",        // pairing | allowlist | open | disabled
        allowFrom: ["123456789012345678", "username"]
      },
      guilds: {
        "*": { requireMention: true },  // 全局默认需要 @提及
        "YOUR_GUILD_ID": {
          requireMention: false,
          users: ["YOUR_USER_ID"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["USER_ID"],
              skills: ["search", "docs"],
              systemPrompt: "保持回答简短。"
            }
          }
        }
      },
      textChunkLimit: 2000,
      maxLinesPerMessage: 17,
      mediaMaxMb: 8,
      actions: {
        reactions: true,
        messages: true,
        threads: true,
        pins: true,
        search: true,
        memberInfo: true,
        roleInfo: true,
        channelInfo: true
      }
    }
  }
}
```

### 配置说明

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `enabled` | 启用/禁用 Discord 渠道 | `true`（有 token 时） |
| `token` | Bot Token | - |
| `groupPolicy` | 服务器频道策略：`open`/`allowlist`/`disabled` | `allowlist` |
| `dm.enabled` | 启用 DM | `true` |
| `dm.policy` | DM 策略：`pairing`/`allowlist`/`open`/`disabled` | `pairing` |
| `dm.allowFrom` | DM 允许列表 | - |
| `guilds` | 服务器规则（按 ID 或 slug） | - |
| `textChunkLimit` | 文本分块大小 | `2000` |
| `maxLinesPerMessage` | 每条消息最大行数 | `17` |
| `mediaMaxMb` | 媒体大小上限（MB） | `8` |
| `historyLimit` | 上下文历史消息数 | `20` |

### 验证配置

1. 启动 Gateway：`clawdbot gateway --verbose`
2. 在服务器频道发送：`@YourBot hello`
3. 检查日志和响应

---

## Slack 配置

Slack 支持两种模式：**Socket Mode**（默认）和 **HTTP Mode**。

### Socket Mode（推荐）

#### 步骤 1：创建 Slack 应用

1. 访问 [Slack API](https://api.slack.com/apps)
2. 点击 **Create New App** → **From scratch**
3. 输入应用名称，选择工作空间

#### 步骤 2：启用 Socket Mode

1. 进入 **Socket Mode** → 开启
2. 进入 **Basic Information** → **App-Level Tokens** → **Generate Token and Scopes**
3. 添加 scope `connections:write`
4. 复制 **App Token**（`xapp-...`）

#### 步骤 3：配置 OAuth 权限

进入 **OAuth & Permissions** → **Bot Token Scopes**，添加以下权限：

**必需的 Bot Token Scopes**:
- `chat:write` - 发送消息
- `channels:history`, `groups:history`, `im:history`, `mpim:history` - 读取历史
- `channels:read`, `groups:read`, `im:read`, `mpim:read` - 读取频道信息
- `users:read` - 用户查找
- `app_mentions:read` - 读取 @提及
- `reactions:read`, `reactions:write` - 表情反应
- `pins:read`, `pins:write` - 置顶消息
- `emoji:read` - 自定义表情
- `commands` - 斜杠命令
- `files:read`, `files:write` - 文件操作
- `im:write` - 打开 DM

点击 **Install to Workspace**，复制 **Bot User OAuth Token**（`xoxb-...`）

#### 步骤 4：配置事件订阅

进入 **Event Subscriptions** → 启用事件，订阅以下 Bot Events：
- `app_mention`
- `message.channels`, `message.groups`, `message.im`, `message.mpim`
- `reaction_added`, `reaction_removed`
- `member_joined_channel`, `member_left_channel`
- `channel_rename`
- `pin_added`, `pin_removed`

#### 步骤 5：启用 App Home

进入 **App Home** → 启用 **Messages Tab**（允许用户与 Bot 发 DM）

#### 步骤 6：配置 Clawdbot

**方式一：通过环境变量**（推荐）：

```bash
export SLACK_APP_TOKEN="xapp-..."
export SLACK_BOT_TOKEN="xoxb-..."
```

**方式二：通过配置文件**：

```json5
{
  channels: {
    slack: {
      enabled: true,
      appToken: "xapp-...",
      botToken: "xoxb-..."
    }
  }
}
```

#### 步骤 7：邀请 Bot 到频道

在 Slack 中，将 Bot 邀请到需要它响应的频道。

### HTTP Mode（Events API）

适用于 Gateway 可通过 HTTPS 公开访问的服务器部署。

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: "xoxb-...",
      signingSecret: "your-signing-secret",
      webhookPath: "/slack/events"
    }
  }
}
```

在 Slack 应用配置中：
- **Event Subscriptions** → Request URL：`https://your-gateway/slack/events`
- **Interactivity & Shortcuts** → 同上
- **Slash Commands** → 同上

### 完整配置示例

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      groupPolicy: "allowlist",
      dm: {
        enabled: true,
        policy: "pairing",           // pairing | allowlist | open | disabled
        allowFrom: ["U123", "U456"],
        groupEnabled: false,
        groupChannels: ["G123"],
        replyToMode: "all"
      },
      channels: {
        "C123": { allow: true, requireMention: true },
        "#general": {
          allow: true,
          requireMention: true,
          users: ["U123"],
          skills: ["search", "docs"],
          systemPrompt: "保持回答简短。"
        }
      },
      reactionNotifications: "own",
      replyToMode: "off",
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true
      },
      slashCommand: {
        enabled: true,
        name: "clawd",
        ephemeral: true
      },
      textChunkLimit: 4000,
      mediaMaxMb: 20,
      historyLimit: 50
    }
  }
}
```

### Slack 应用清单（快速创建）

使用以下清单快速创建应用：

```json
{
  "display_information": {
    "name": "Clawdbot",
    "description": "Slack connector for Clawdbot"
  },
  "features": {
    "bot_user": {
      "display_name": "Clawdbot",
      "always_online": false
    },
    "app_home": {
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "slash_commands": [
      {
        "command": "/clawd",
        "description": "Send a message to Clawdbot",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "chat:write",
        "channels:history",
        "channels:read",
        "groups:history",
        "groups:read",
        "groups:write",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "users:read",
        "app_mentions:read",
        "reactions:read",
        "reactions:write",
        "pins:read",
        "pins:write",
        "emoji:read",
        "commands",
        "files:read",
        "files:write"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "reaction_added",
        "reaction_removed",
        "member_joined_channel",
        "member_left_channel",
        "channel_rename",
        "pin_added",
        "pin_removed"
      ]
    }
  }
}
```

### 配置说明

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `enabled` | 启用/禁用 Slack 渠道 | `false` |
| `appToken` | App Token（`xapp-...`） | - |
| `botToken` | Bot Token（`xoxb-...`） | - |
| `mode` | 模式：`socket`/`http` | `socket` |
| `groupPolicy` | 频道策略：`open`/`allowlist`/`disabled` | `allowlist` |
| `dm.policy` | DM 策略 | `pairing` |
| `channels` | 频道规则（按 ID 或 `#name`） | - |
| `textChunkLimit` | 文本分块大小 | `4000` |
| `mediaMaxMb` | 媒体大小上限（MB） | `20` |
| `historyLimit` | 上下文历史消息数 | `50` |
| `replyToMode` | 回复线程模式：`off`/`first`/`all` | `off` |

### 回复线程模式说明

| 模式 | 行为 |
|------|------|
| `off` | 默认。在主频道回复。仅当触发消息在线程中时才在线程中回复。 |
| `first` | 第一条回复进入线程，后续回复在主频道。 |
| `all` | 所有回复都在线程中。 |

---

## DM 配对安全机制

Clawdbot 默认使用 **配对（Pairing）** 机制保护 DM 访问安全。

### 工作原理

1. 当配置了 `dmPolicy: "pairing"` 时，未知发送者会收到一个配对码
2. 消息不会被处理，直到您批准该配对码
3. 配对码在 1 小时后过期
4. 每个渠道默认最多 3 个待处理的配对请求

### 批准配对

```bash
# 列出待处理的配对请求
clawdbot pairing list telegram
clawdbot pairing list discord
clawdbot pairing list slack
clawdbot pairing list imessage

# 批准配对
clawdbot pairing approve telegram <CODE>
clawdbot pairing approve discord <CODE>
clawdbot pairing approve slack <CODE>
clawdbot pairing approve imessage <CODE>
```

### 配对状态存储位置

- 待处理请求：`~/.clawdbot/credentials/<channel>-pairing.json`
- 已批准列表：`~/.clawdbot/credentials/<channel>-allowFrom.json`

> **安全提示**: 这些文件控制谁可以访问您的助手，请妥善保管。

---

## 常见问题排查

### 通用诊断

```bash
# 运行诊断
clawdbot doctor

# 检查渠道状态
clawdbot channels status --probe

# 查看详细状态
clawdbot status --all
```

### `clawdbot` 命令未找到

```bash
# 检查 Node 和 npm
node -v
npm -v
npm prefix -g
echo "$PATH"

# 确保全局 npm bin 目录在 PATH 中
export PATH="$(npm prefix -g)/bin:$PATH"
```

### Discord Bot 连接但不响应

1. 确认启用了 **Message Content Intent**
2. 确认 Bot 有频道权限（View/Send/Read History）
3. 检查 `requireMention` 设置
4. 检查 Guild/Channel 允许列表

### Slack Bot 不响应

1. 确认 Bot 已被邀请到频道
2. 确认启用了所需的事件订阅
3. 检查 `groupPolicy` 和频道允许列表
4. 确认 tokens 正确

### iMessage 卡住或退出

1. 登录 Bot macOS 用户
2. 运行 `imsg chats --limit 1` 触发权限提示
3. 批准自动化和全磁盘访问权限
4. 重试

### Gateway 启动失败

```bash
# 检查端口占用
lsof -i :18789

# 强制启动
clawdbot gateway --force

# 查看日志
tail -f /tmp/clawdbot/gateway.log
```

---

## 更多资源

- [官方文档](https://docs.clawd.bot)
- [GitHub 仓库](https://github.com/clawdbot/clawdbot)
- [Discord 频道配置详情](https://docs.clawd.bot/channels/discord)
- [Slack 频道配置详情](https://docs.clawd.bot/channels/slack)
- [iMessage 频道配置详情](https://docs.clawd.bot/channels/imessage)
- [安全配置](https://docs.clawd.bot/gateway/security)
- [配对机制](https://docs.clawd.bot/start/pairing)
