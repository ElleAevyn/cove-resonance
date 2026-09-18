# Cove Resonance

> 不搬 AI，给 AI 修路。

Cove Resonance 是一个面向 **MCP Apps / ChatGPT** 的双向消息桥实验项目。它把外部实时事件送进当前对话，并把模型回复按原路送回外部应用。

当前重点场景是 **网易云音乐「一起听」ChatRoom ⇄ Cove Bridge ⇄ ChatGPT**：用户仍然在官方网易云客户端里听歌、发消息；Bridge 负责实时转发、上下文补充、回复路由与唤醒。

## 现在已经能做什么

- 网易云一起听 ChatRoom 文本消息实时进入 Bridge。
- ChatGPT 回复可经 `cove_bridge_reply` 回到原 ChatRoom。
- Conversation / State 两条流分离：
  - Conversation 保序，不丢聊天消息。
  - State 采用 latest-state-wins，避免陈旧播放状态堆积。
- 单轮 required reply backpressure，避免多条对话互相串线。
- SSE wake、短期单次 session token、EventSource Listener 已实现；当前仍在做长时间运行与 reconnect 边界的端到端稳定性验收。
- 同一网易云 `messageId` 与同一 Bridge `eventId` 的去重保护已实现；当前仍在继续验证 reconnect / ACK 边界。
- 自动把过长回复拆成 2–5 个自然聊天气泡。
- 换歌时把整首可用歌词作为隐藏模型上下文注入。
- NIM 实时 ChatRoom 已跑通；播放事件解码已具备，播放状态主链路仍在继续优化。

## 架构

```text
网易云官方客户端
        │
        │ Listen Together / NIM ChatRoom
        ▼
┌──────────────────────────┐
│        Cove Bridge       │
│                          │
│  Conversation Stream     │
│  State Stream            │
│  Reply Route             │
│  SSE Wake                │
└────────────┬─────────────┘
             │ HTTPS / MCP
             ▼
┌──────────────────────────┐
│  ChatGPT + Listener App  │
│                          │
│  ui/update-model-context │
│  ui/message              │
│  cove_bridge_reply       │
└──────────────────────────┘
```

核心原则：

> 对话要记忆，状态要新鲜。

SSE 只负责“敲门”，不直接承载聊天正文。真正的事件仍通过 `cove_bridge_sync` 从队列取出，因此重连不会绕开 reservation、幂等和 reply lock。

## 5 分钟启动

要求：

- Node.js 22
- npm
- 一台能被 ChatGPT 访问的 HTTPS 服务
- 你自己的网易云账号 Cookie（需要 `MUSIC_U`）
- 支持 MCP Apps 的 ChatGPT 环境

```bash
git clone https://github.com/yanceydaisy/cove-resonance.git
cd cove-resonance

cp .env.example .env
npm install
npm test
npm run build
```

至少配置：

```dotenv
BRIDGE_PUBLIC_ORIGIN=https://bridge.example.com
NETEASE_COOKIE=MUSIC_U=...
TOGETHER_ENABLED=true
```

然后：

```bash
set -a
source .env
set +a
npm start
```

默认：

- Health: `http://127.0.0.1:8787/`
- MCP: `http://127.0.0.1:8787/mcp`
- External event ingest: `POST /events`

公网部署请在前面放 Caddy / Nginx / 其他 HTTPS reverse proxy。

部署教程：**[docs/GETTING_STARTED.zh-CN.md](docs/GETTING_STARTED.zh-CN.md)**

技术实现 / 移植教程：**[docs/ARCHITECTURE_FOR_AGENTS.zh-CN.md](docs/ARCHITECTURE_FOR_AGENTS.zh-CN.md)**

给编码 Agent 的入口：**[AGENTS.md](AGENTS.md)**

## 安全边界

请把这些当成硬规则：

- **不要提交网易云 Cookie。**
- **不要提交 `BRIDGE_INGEST_TOKEN`。**
- `NETEASE_COOKIE` 只放服务器环境变量或 root-only 文件。
- Listener SSE token 是短期、单次消费 token，不要改成长效凭证。
- 如果使用 noVNC / Chrome DevTools，5901 / 6080 / 9222 必须只绑定 loopback，不要直接暴露公网。
- 如果在 VPS 上托管 ChatGPT Listener，请使用 ChatGPT 官方支持的地区。

## 当前实验状态

已验证环境：

- Ubuntu 22.04
- Node.js 22
- `node-nim@10.10.13`
- NetEase NIM ChatRoom realtime
- MCP Apps Widget
- Caddy HTTPS reverse proxy

已实现、仍在稳定性验收：

- SSE realtime wake 的长时间运行 / reconnect 边界。
- Listener ACK 与重复投递保护的极端网络场景。

优化方向（未宣称完成）：

- NIM playback realtime 成为播放状态主数据源。
- SQLite 持久化 Conversation / reply route。
- crash-safe reply journal。
- Listener 自动恢复 / watchdog。
- multi-listener semantics。
- 更完整的一键部署脚本。
- 支持地区 VPS 的单机一体化部署。

## 测试

```bash
npm test
npm run build
```

目前测试覆盖队列、回复幂等、ChatRoom 编解码、歌词、播放状态、SSE session、Widget 生成脚本语法与重复消息抑制。

## 项目定位

这不是网易云客户端替代品，也不是另起一个聊天前端。

目标始终是：

```text
用户继续待在原来的应用里
            +
AI 继续待在官方 ChatGPT 里
            +
Bridge 只负责把路修通
```

## Credits

消息桥最初结构参考：

- [wynsyl1014/mcp-app-message-bridge](https://github.com/wynsyl1014/mcp-app-message-bridge)
- [wuxiandudang-hash/ncm-listen-together](https://github.com/wuxiandudang-hash/ncm-listen-together)

MCP App 相关行为以 MCP Apps SDK / Host 实际能力为准。
