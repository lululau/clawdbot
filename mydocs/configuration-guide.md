# Clawdbot 配置与使用指南

本文档详细介绍 Clawdbot 的配置选项和使用方法。如需安装说明，请参阅单独的安装文档。

## 目录

- [基本概念](#基本概念)
- [快速入门](#快速入门)
- [Gateway 配置](#gateway-配置)
- [模型配置](#模型配置)
- [消息频道配置](#消息频道配置)
- [Agent 配置](#agent-配置)
- [安全配置](#安全配置)
- [CLI 命令参考](#cli-命令参考)
- [聊天命令](#聊天命令)
- [Skills 技能系统](#skills-技能系统)
- [高级功能](#高级功能)
- [故障排除](#故障排除)

---

## 基本概念

### 架构概述

Clawdbot 采用 Gateway 架构，所有组件通过 WebSocket 连接到中央 Gateway：

```
WhatsApp / Telegram / Slack / Discord / Signal / iMessage / WebChat
                        │
                        ▼
         ┌───────────────────────────────┐
         │            Gateway            │
         │        (控制平面)              │
         │     ws://127.0.0.1:18789      │
         └──────────────┬────────────────┘
                        │
                        ├─ Pi agent (RPC)
                        ├─ CLI (clawdbot …)
                        ├─ WebChat UI
                        ├─ macOS 应用
                        └─ iOS / Android 节点
```

### 核心组件

- **Gateway**: WebSocket 服务器，管理频道、节点、会话和钩子
- **Agent**: AI 助手运行时，处理消息和执行工具
- **Channels**: 消息平台连接器（WhatsApp、Telegram 等）
- **Skills**: 可扩展的技能系统，教 Agent 使用工具
- **Nodes**: 设备节点（macOS/iOS/Android），提供本地功能

### 重要术语

- **DM (Direct Message)**: 私信/直接消息，指聊天平台上的私人一对一对话，与群组聊天相对
- **Gateway**: Clawdbot 的中央控制平面，通过 WebSocket 管理所有连接
- **Agent**: AI 助手实例，处理消息和执行任务
- **Channel**: 消息平台连接器（如 WhatsApp、Telegram）
- **Session**: 会话，代表一次对话的上下文
- **Skill**: 技能，可扩展的功能模块，教 Agent 如何使用工具
- **Node**: 设备节点，提供本地功能（如 macOS/iOS/Android）

### 配置文件位置

主配置文件：`~/.clawdbot/clawdbot.json`（支持 JSON5 格式，允许注释和尾随逗号）

其他重要路径：
- 凭证目录：`~/.clawdbot/credentials/`
- 会话日志：`~/.clawdbot/agents/<agentId>/sessions/*.jsonl`
- 认证配置：`~/.clawdbot/agents/<agentId>/agent/auth-profiles.json`
- 技能目录：`~/.clawdbot/skills/`
- 日志文件：`/tmp/clawdbot/clawdbot-YYYY-MM-DD.log`

---

## 快速入门

### 1. 运行配置向导

推荐使用交互式向导进行初始配置：

```bash
clawdbot onboard --install-daemon
```

向导将引导你完成：
- 模型和认证配置（OAuth 或 API 密钥）
- Gateway 设置
- 消息频道配置
- 后台服务安装

### 2. 启动 Gateway

如果已安装为服务，Gateway 应该已经在运行：

```bash
clawdbot gateway status
```

手动启动（前台运行）：

```bash
clawdbot gateway --port 18789 --verbose
```

### 3. 验证配置

```bash
clawdbot status        # 基本状态
clawdbot health        # 健康检查
clawdbot status --all  # 完整诊断报告
```

### 最小配置示例

```json5
{
  agents: { defaults: { workspace: "~/clawd" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } }
}
```

---

## Gateway 配置

### 基本选项

```json5
{
  gateway: {
    mode: "local",           // 必须设置为 "local" 才能启动
    port: 18789,             // WebSocket 端口
    bind: "loopback",        // 绑定模式：loopback | lan | tailnet | auto | custom
    auth: {
      mode: "token",         // 认证模式：token | password
      token: "your-token"    // 或使用环境变量 CLAWDBOT_GATEWAY_TOKEN
    }
  }
}
```

### 绑定模式说明

| 模式 | 说明 |
|------|------|
| `loopback` | 仅本地连接（默认，最安全） |
| `lan` | 局域网可访问 |
| `tailnet` | Tailscale 网络可访问 |
| `auto` | 自动检测 |
| `custom` | 自定义绑定地址 |

### Tailscale 集成

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: {
      mode: "serve"  // off | serve | funnel
    }
  }
}
```

- `serve`: Tailnet 内部 HTTPS 访问
- `funnel`: 公网 HTTPS 访问（需要密码认证）

### Gateway 服务管理

```bash
clawdbot gateway install    # 安装为系统服务
clawdbot gateway start      # 启动服务
clawdbot gateway stop       # 停止服务
clawdbot gateway restart    # 重启服务
clawdbot gateway uninstall  # 卸载服务
```

---

## 模型配置

### 模型选择优先级

1. 主模型（`agents.defaults.model.primary`）
2. 备选模型（`agents.defaults.model.fallbacks`，按顺序）
3. Provider 内部 auth failover

### 配置示例

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-5",
        fallbacks: [
          "anthropic/claude-sonnet-4-5",
          "openai/gpt-5.2"
        ]
      },
      // 图像模型（当主模型不支持图像时使用）
      imageModel: {
        primary: "anthropic/claude-sonnet-4-5"
      },
      // 模型白名单（可选，设置后仅允许列表中的模型）
      models: {
        "anthropic/claude-opus-4-5": { alias: "Opus" },
        "anthropic/claude-sonnet-4-5": { alias: "Sonnet" }
      }
    }
  }
}
```

### 认证配置

#### Anthropic（推荐 API 密钥或 setup-token）

```bash
# 使用 Claude Code 的 setup-token
claude setup-token
clawdbot models status

# 或者在向导中设置 API 密钥
clawdbot onboard --auth-choice anthropic-api-key
```

#### OpenAI Codex（OAuth）

```bash
clawdbot onboard --auth-choice openai-codex
```

### 模型管理命令

```bash
clawdbot models status                    # 查看当前模型状态
clawdbot models list --all                # 列出所有可用模型
clawdbot models set <provider/model>      # 设置主模型
clawdbot models set-image <provider/model> # 设置图像模型

# 备选模型管理
clawdbot models fallbacks list
clawdbot models fallbacks add <provider/model>
clawdbot models fallbacks remove <provider/model>

# 别名管理
clawdbot models aliases add opus anthropic/claude-opus-4-5
clawdbot models aliases list
```

### 聊天中切换模型

```
/model                    # 显示模型选择器
/model list               # 列出可用模型
/model 3                  # 选择编号 3 的模型
/model openai/gpt-5.2     # 直接指定模型
/model status             # 查看详细状态
```

---

## 消息频道配置

### 支持的频道

| 频道 | 说明 | 配置方式 |
|------|------|----------|
| WhatsApp | 最流行，需 QR 配对 | `clawdbot channels login` |
| Telegram | Bot API，支持群组 | Bot Token |
| Discord | 支持服务器、频道、私信 | Bot Token |
| Slack | 工作区应用 | Bot Token + App Token |
| Google Chat | HTTP webhook | Chat API |
| Signal | 隐私优先 | signal-cli |
| iMessage | macOS 专用 | 原生集成 |
| BlueBubbles | 推荐的 iMessage 方案 | BlueBubbles 服务器 |
| Microsoft Teams | 企业支持（插件） | Bot Framework |
| Matrix | 去中心化（插件） | Matrix 协议 |
| WebChat | Gateway 内置 | 无需配置 |

### WhatsApp 配置

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",              // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123"],      // DM 白名单
      groups: {
        "*": { requireMention: true }   // 群组需要 @提及
      }
    }
  }
}
```

登录 WhatsApp：

```bash
clawdbot channels login --channel whatsapp
# 扫描 QR 码完成配对
```

### Telegram 配置

```json5
{
  channels: {
    telegram: {
      botToken: "123456:ABCDEF",        // 或设置环境变量 TELEGRAM_BOT_TOKEN
      dmPolicy: "pairing",
      allowFrom: ["123456789"],         // 用户 ID
      groups: {
        "*": { requireMention: true }
      }
    }
  }
}
```

### Discord 配置

```json5
{
  channels: {
    discord: {
      token: "your-bot-token",          // 或设置环境变量 DISCORD_BOT_TOKEN
      dm: {
        policy: "pairing",
        allowFrom: ["user-id-1"]
      },
      guilds: {
        "guild-id": {
          requireMention: true,
          allowFrom: ["*"]              // 允许所有成员
        }
      }
    }
  }
}
```

### Slack 配置

```json5
{
  channels: {
    slack: {
      botToken: "xoxb-...",             // 或 SLACK_BOT_TOKEN
      appToken: "xapp-...",             // 或 SLACK_APP_TOKEN
      dm: {
        policy: "pairing"
      }
    }
  }
}
```

### 频道管理命令

```bash
clawdbot channels list                  # 列出配置的频道
clawdbot channels status --probe        # 检查频道健康状态
clawdbot channels add --channel telegram --token $TOKEN
clawdbot channels remove --channel discord
clawdbot channels logs --channel all    # 查看频道日志
```

### DM 配对审批

当 `dmPolicy` 设置为 `pairing` 时，未知发送者会收到配对码：

```bash
clawdbot pairing list whatsapp          # 列出待审批请求
clawdbot pairing approve whatsapp ABC123 # 批准配对请求
```

---

## Agent 配置

### 基本 Agent 配置

```json5
{
  agents: {
    defaults: {
      workspace: "~/clawd",             // 工作空间目录
      model: { primary: "anthropic/claude-opus-4-5" },
      timeoutSeconds: 600,              // 超时时间
      heartbeat: {
        every: "2h",                    // 心跳间隔
        prompt: "Check on things"       // 心跳提示
      }
    },
    list: [
      {
        id: "main",
        workspace: "~/clawd",
        identity: {
          name: "Clawd",
          emoji: "🦞",
          theme: "helpful lobster"
        }
      }
    ]
  }
}
```

### 多 Agent 配置

```json5
{
  agents: {
    defaults: {
      workspace: "~/clawd"
    },
    list: [
      {
        id: "personal",
        workspace: "~/clawd-personal",
        sandbox: { mode: "off" }        // 个人 Agent：无沙箱
      },
      {
        id: "work",
        workspace: "~/clawd-work",
        sandbox: { mode: "all" }        // 工作 Agent：全部沙箱
      }
    ]
  },
  routing: {
    agents: {
      "personal": {
        bindings: ["whatsapp:+15555550123"]
      },
      "work": {
        bindings: ["slack:work-account"]
      }
    }
  }
}
```

### Agent 管理命令

```bash
clawdbot agents list                    # 列出所有 Agent
clawdbot agents add personal --workspace ~/clawd-personal
clawdbot agents delete work --force

# 运行 Agent
clawdbot agent --message "Hello" --thinking high
clawdbot agent --agent work --message "Check tasks" --deliver
```

### 工作空间文件

Agent 工作空间支持以下提示文件：

- `AGENTS.md`: Agent 指令和项目上下文
- `SOUL.md`: 性格和行为定义
- `TOOLS.md`: 工具使用指南
- `skills/`: 技能目录

---

## 安全配置

### 安全审计

```bash
clawdbot security audit                 # 基本审计
clawdbot security audit --deep          # 深度审计（含 Gateway 探测）
clawdbot security audit --fix           # 自动修复常见问题
```

### DM 访问策略

| 策略 | 说明 |
|------|------|
| `pairing` | 默认，未知发送者需配对审批 |
| `allowlist` | 仅允许白名单用户 |
| `open` | 允许所有人（需显式设置） |
| `disabled` | 禁用 DM |

### 群组策略

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",         // 群组白名单模式
      groupAllowFrom: ["admin-id"],     // 群组内允许的用户
      groups: {
        "*": { requireMention: true }   // 需要 @提及才响应
      }
    }
  }
}
```

### 沙箱配置

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",               // off | all | non-main
        scope: "agent",                 // agent | session | shared
        workspaceAccess: "none"         // none | ro | rw
      }
    }
  }
}
```

沙箱模式：
- `off`: 无沙箱，工具在主机运行
- `all`: 所有会话都使用沙箱
- `non-main`: 非主会话（群组等）使用沙箱

### 工具权限

```json5
{
  agents: {
    list: [
      {
        id: "restricted",
        tools: {
          allow: ["read", "sessions_list"],
          deny: ["write", "edit", "exec", "browser"]
        }
      }
    ]
  }
}
```

### 安全最佳实践

1. **Gateway 认证**: 始终启用 token 或 password 认证
2. **DM 配对**: 使用 `pairing` 模式而非 `open`
3. **群组提及**: 在群组中启用 `requireMention`
4. **沙箱隔离**: 对不受信任的会话使用沙箱
5. **文件权限**: 确保 `~/.clawdbot` 权限为 700

---

## CLI 命令参考

### 全局选项

```bash
clawdbot [--dev] [--profile <name>] <command>

--dev           # 使用开发配置（~/.clawdbot-dev）
--profile <n>   # 使用指定配置（~/.clawdbot-<name>）
--no-color      # 禁用 ANSI 颜色
--json          # JSON 输出
```

### 配置命令

```bash
clawdbot config get <path>              # 获取配置值
clawdbot config set <path> <value>      # 设置配置值
clawdbot config unset <path>            # 删除配置值

# 示例
clawdbot config get browser.executablePath
clawdbot config set agents.defaults.heartbeat.every "2h"
clawdbot config set channels.whatsapp.groups '["*"]' --json
```

### 状态和诊断

```bash
clawdbot status                         # 基本状态
clawdbot status --all                   # 完整诊断
clawdbot status --deep                  # 深度探测
clawdbot status --usage                 # 显示用量

clawdbot health                         # Gateway 健康检查
clawdbot doctor                         # 配置诊断和修复
clawdbot doctor --fix                   # 自动修复
```

### 会话管理

```bash
clawdbot sessions                       # 列出会话
clawdbot sessions --active 60           # 活跃会话（60分钟内）
clawdbot sessions --json                # JSON 输出
```

### 日志查看

```bash
clawdbot logs                           # 查看日志
clawdbot logs --follow                  # 实时跟踪
clawdbot logs --limit 200               # 限制行数
clawdbot logs --json                    # JSON 格式
```

### 消息发送

```bash
clawdbot message send --target +15555550123 --message "Hello"
clawdbot message send --channel telegram --target chat-id --message "Hi"
```

### 浏览器控制

```bash
clawdbot browser status                 # 浏览器状态
clawdbot browser start                  # 启动浏览器
clawdbot browser stop                   # 停止浏览器
clawdbot browser tabs                   # 列出标签页
clawdbot browser screenshot             # 截图
clawdbot browser navigate <url>         # 导航到 URL
```

### 定时任务

```bash
clawdbot cron list                      # 列出定时任务
clawdbot cron add --name daily --every 24h --message "Daily check"
clawdbot cron enable <id>
clawdbot cron disable <id>
clawdbot cron rm <id>
```

### 节点管理

```bash
clawdbot nodes list                     # 列出节点
clawdbot nodes status                   # 节点状态
clawdbot nodes approve <requestId>      # 批准节点
clawdbot nodes invoke --node <id> --command <cmd>
```

---

## 聊天命令

在聊天中发送以 `/` 开头的命令：

### 状态和控制

| 命令 | 说明 |
|------|------|
| `/status` | 显示当前状态 |
| `/help` | 显示帮助 |
| `/commands` | 列出可用命令 |
| `/new` 或 `/reset` | 重置会话 |
| `/compact` | 压缩会话上下文 |
| `/stop` | 停止当前操作 |
| `/restart` | 重启 Gateway（需启用） |

### 模型和思考

| 命令 | 说明 |
|------|------|
| `/model` | 选择模型 |
| `/model <name>` | 切换到指定模型 |
| `/think <level>` | 设置思考级别：off\|minimal\|low\|medium\|high\|xhigh |
| `/verbose on\|off` | 详细输出 |
| `/reasoning on\|off` | 显示推理过程 |

### 会话设置

| 命令 | 说明 |
|------|------|
| `/usage off\|tokens\|full` | 使用量显示 |
| `/tts off\|always` | 语音合成 |
| `/activation mention\|always` | 群组激活模式 |
| `/elevated on\|off` | 提升权限模式 |
| `/queue <mode>` | 队列模式 |

### 技能调用

```
/skill <name> [input]    # 运行指定技能
/<skill-name> [input]    # 直接调用技能
```

### 配置命令（需启用）

```
/config show             # 显示配置
/config get <path>       # 获取值
/config set <path>=<value>  # 设置值
/config unset <path>     # 删除值
```

---

## Skills 技能系统

### 技能位置

技能按以下优先级加载：

1. 工作空间技能：`<workspace>/skills/`（最高优先级）
2. 本地技能：`~/.clawdbot/skills/`
3. 内置技能：随安装包提供

### 技能格式

每个技能是一个包含 `SKILL.md` 的目录：

```markdown
---
name: my-skill
description: 技能描述
metadata: {"clawdbot":{"requires":{"bins":["uv"],"env":["API_KEY"]}}}
---

技能使用说明...
```

### 技能配置

```json5
{
  skills: {
    entries: {
      "my-skill": {
        enabled: true,
        apiKey: "your-api-key",
        env: {
          API_KEY: "your-api-key"
        }
      }
    },
    load: {
      watch: true,              // 监视技能变化
      watchDebounceMs: 250
    }
  }
}
```

### 技能管理命令

```bash
clawdbot skills list                    # 列出技能
clawdbot skills info <name>             # 技能详情
clawdbot skills check                   # 检查技能状态
```

### ClawdHub（技能市场）

```bash
npx clawdhub                            # 搜索技能
clawdhub install <skill-slug>           # 安装技能
clawdhub update --all                   # 更新所有技能
```

---

## 高级功能

### 环境变量

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-..."
    },
    shellEnv: {
      enabled: true,            // 从 shell 导入环境变量
      timeoutMs: 15000
    }
  }
}
```

### 配置文件包含

```json5
// ~/.clawdbot/clawdbot.json
{
  gateway: { port: 18789 },
  agents: { "$include": "./agents.json5" },
  channels: { "$include": [
    "./channels/whatsapp.json5",
    "./channels/telegram.json5"
  ]}
}
```

### 心跳和自动化

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "2h",
        prompt: "Check on scheduled tasks",
        channels: ["telegram:main"]
      }
    }
  }
}
```

### Webhook 集成

```bash
# Gmail Pub/Sub
clawdbot webhooks gmail setup --account email@gmail.com
clawdbot webhooks gmail run
```

### 浏览器控制

```json5
{
  browser: {
    enabled: true,
    controlUrl: "http://127.0.0.1:18791",
    executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
  }
}
```

### 日志配置

```json5
{
  logging: {
    level: "info",
    file: "/tmp/clawdbot/clawdbot.log",
    consoleLevel: "info",
    consoleStyle: "pretty",         // pretty | compact | json
    redactSensitive: "tools"        // off | tools
  }
}
```

---

## 故障排除

### 常见问题

#### Gateway 无法启动

```bash
# 检查配置
clawdbot doctor

# 查看日志
clawdbot logs --limit 50

# 强制启动（杀死现有进程）
clawdbot gateway --force
```

#### 模型认证失败

```bash
# 检查认证状态
clawdbot models status

# 重新配置认证
clawdbot onboard --auth-choice setup-token
```

#### 频道连接问题

```bash
# 检查频道状态
clawdbot channels status --probe

# 查看频道日志
clawdbot channels logs --channel whatsapp

# 重新登录
clawdbot channels login --channel whatsapp
```

#### DM 没有响应

1. 检查 DM 策略是否为 `pairing`
2. 检查是否有待审批的配对请求：`clawdbot pairing list <channel>`
3. 检查白名单配置

#### 群组消息不响应

1. 检查 `requireMention` 是否启用
2. 检查群组白名单
3. 确认 @提及格式正确

### 诊断命令

```bash
clawdbot doctor                         # 配置诊断
clawdbot doctor --deep                  # 深度扫描
clawdbot status --all                   # 完整状态报告
clawdbot health                         # Gateway 健康检查
clawdbot security audit                 # 安全审计
```

### 重置选项

```bash
# 重置配置
clawdbot reset --scope config

# 重置全部（配置+凭证+会话）
clawdbot reset --scope full --yes

# 卸载服务和数据
clawdbot uninstall --all --yes
```

### 获取帮助

- 文档：https://docs.clawd.bot
- Discord：https://discord.gg/clawd
- GitHub Issues：https://github.com/clawdbot/clawdbot/issues

---

## 配置参考

### 完整配置示例

```json5
{
  // Gateway 配置
  gateway: {
    mode: "local",
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token",
      token: "${CLAWDBOT_GATEWAY_TOKEN}"
    }
  },

  // Agent 配置
  agents: {
    defaults: {
      workspace: "~/clawd",
      model: {
        primary: "anthropic/claude-opus-4-5",
        fallbacks: ["anthropic/claude-sonnet-4-5"]
      },
      timeoutSeconds: 600,
      sandbox: {
        mode: "non-main",
        scope: "agent"
      }
    },
    list: [
      {
        id: "main",
        identity: {
          name: "Clawd",
          emoji: "🦞"
        }
      }
    ]
  },

  // 频道配置
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15555550123"],
      groups: {
        "*": { requireMention: true }
      }
    },
    telegram: {
      botToken: "${TELEGRAM_BOT_TOKEN}",
      dmPolicy: "pairing"
    }
  },

  // 命令配置
  commands: {
    native: "auto",
    text: true,
    bash: false,
    config: false,
    debug: false
  },

  // 日志配置
  logging: {
    level: "info",
    redactSensitive: "tools"
  }
}
```

---

*本文档基于 Clawdbot 最新版本编写，如有更新请参考官方文档。*
