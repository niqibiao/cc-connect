# Fork: Additional Features & Configuration

English | [中文](#fork-额外功能与配置说明)

This fork adds the following features on top of the upstream [cc-connect](https://github.com/chenhg5/cc-connect). For general usage, see the main [README](./README.md).

---

## Group Chat Summary (Agent-Driven)

Passively record all group chat messages and let the AI agent summarize them on demand — no slash command needed. The agent naturally decides when to call `cc-connect chatlog` based on the user's request.

### How It Works

```
Group Chat Messages (all members)
        │
        ▼
  cc-connect engine ──► in-memory ChatLog (per-chat ring buffer, max 500)
        │
   user @bot: "summarize the chat"
        │
        ▼
  AI Agent ──► cc-connect chatlog  (reads history)
        │
        ▼
  Summary returned to group chat
        │
  Agent asks: "Clear the history?"
        │
   user: "yes"
        │
        ▼
  AI Agent ──► cc-connect chatlog-clear
```

- Messages are stored **in memory only** and cleared on process restart.
- Each group chat has an independent ring buffer (max 500 messages). Messages from different groups are never mixed.
- Both @bot messages and non-@bot messages are recorded, but only @bot messages trigger the agent.
- The agent decides on its own when to fetch and clear the chat log — no hardcoded `/summary` command.

### Supported Platforms

| Platform | Status | Notes |
|----------|--------|-------|
| Feishu (Lark) | ✅ Supported | Requires permission changes (see below) |
| Others | 🔜 Planned | Requires platform-level `IsGroup` + passive message support |

### Feishu Configuration

Two settings must be changed in the [Feishu Developer Console](https://open.feishu.cn/app):

#### 1. Receive All Group Messages

By default, Feishu bots only receive messages that @mention the bot. To record all group messages for summarization:

1. Go to **App Features → Bot** in the developer console
2. Find **Message Receive Mode** (消息接收模式)
3. Change from "Only receive @bot messages" to **"Receive all group messages"** (接收群聊中所有消息)
4. Save and publish a new version

#### 2. Contact Permission (for User Names)

Without this permission, the chat log will show raw Open IDs instead of real user names:

1. Go to **Permissions & Scopes** in the developer console
2. Search for `contact:user.base:readonly`
3. Enable it and submit for approval (if your app is internal, approval is automatic)

### CLI Tools

These tools are primarily used by the AI agent automatically, but can also be run manually for debugging:

```bash
# Retrieve chat history
cc-connect chatlog                          # all recorded messages
cc-connect chatlog -n 100                   # last 100 messages
cc-connect chatlog -s "feishu:oc_xxx:ou_xxx" # specify session key

# Clear chat history
cc-connect chatlog-clear
cc-connect chatlog-clear -s "feishu:oc_xxx:ou_xxx"

# Common flags
#   -p, --project <name>    Target project (if multi-project)
#   -s, --session <key>     Session key (default: CC_SESSION_KEY env)
#       --data-dir <path>   Data directory (default: ~/.cc-connect)
```

### API Endpoints

The cc-connect internal API server (Unix socket) exposes:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/chatlog?session_key=...&n=...` | GET | Retrieve recent chat log entries |
| `/clear_chatlog` | POST | Clear all entries for a chat (`{"session_key":"...","project":"..."}`) |

### Agent Integration

The agent's system prompt is automatically extended with instructions for `cc-connect chatlog` and `cc-connect chatlog-clear`. When a user asks for a summary, the agent:

1. Calls `cc-connect chatlog` to fetch the message history
2. Produces a summary
3. Asks the user whether to clear the history
4. If confirmed, calls `cc-connect chatlog-clear`

No additional configuration is needed — just ensure the Feishu permissions above are set.

### Example

In a Feishu group chat:

```
Alice: Let's discuss the API design
Bob:   I think we should use REST
Carol: GraphQL might be better for our use case
Bob:   Good point, let's compare both approaches
  ...
You:   @bot summarize the discussion
Bot:   Here's a summary of the recent discussion:
       - The team discussed API design approaches
       - Bob suggested REST, Carol proposed GraphQL
       - The team agreed to compare both approaches

       Would you like me to clear the chat history?
You:   yes
Bot:   Chat history cleared.
```

---

## Technical Details

### Message Recording

- A new `IsGroup` field on `Message` determines whether a message is recorded to the chat log. This is set by the platform layer (e.g., Feishu checks `chatType == "group"`).
- Passive messages (`Passive=true`) are recorded but do **not** trigger the AI agent — they are silently stored.
- The chat key is derived from the session key by stripping the user part: `"feishu:chatID:userID"` → `"feishu:chatID"`.

### Feishu User Name Resolution

- User names are resolved via the Feishu Contact API (`/open-apis/contact/v3/users/{open_id}`)
- Results are cached in a `sync.Map` for the lifetime of the process
- API calls have a 5-second timeout to prevent hangs
- Falls back to the raw Open ID if the API call fails

---

# Fork: 额外功能与配置说明

[English](#fork-additional-features--configuration) | 中文

本 Fork 在上游 [cc-connect](https://github.com/chenhg5/cc-connect) 基础上新增了以下功能。通用用法请参考主 [README](./README.zh-CN.md)。

---

## 群聊消息总结（Agent 驱动）

被动记录所有群聊消息，由 AI Agent 按需总结 —— 无需手动输入斜杠命令。Agent 根据用户请求自动判断何时调用 `cc-connect chatlog`。

### 工作原理

```
群聊消息（所有群成员）
        │
        ▼
  cc-connect 引擎 ──► 内存 ChatLog（每群独立环形缓冲区，最多 500 条）
        │
   用户 @bot："总结一下群聊"
        │
        ▼
  AI Agent ──► cc-connect chatlog（读取历史）
        │
        ▼
  总结结果发回群聊
        │
  Agent 询问："需要清空历史记录吗？"
        │
   用户："清空吧"
        │
        ▼
  AI Agent ──► cc-connect chatlog-clear
```

- 消息仅存储在**内存中**，进程重启后清空。
- 每个群聊有独立的环形缓冲区（最多 500 条消息），不同群之间的消息不会混淆。
- @bot 和非 @bot 的消息都会被记录，但只有 @bot 的消息才会触发 Agent。
- Agent 自行决定何时获取和清空聊天记录 —— 没有硬编码的 `/summary` 命令。

### 支持的平台

| 平台 | 状态 | 说明 |
|------|------|------|
| 飞书 (Lark) | ✅ 已支持 | 需要修改权限设置（见下文） |
| 其他平台 | 🔜 计划中 | 需要平台层支持 `IsGroup` 和被动消息 |

### 飞书配置

需要在[飞书开发者后台](https://open.feishu.cn/app)修改两项设置：

#### 1. 接收所有群聊消息

飞书机器人默认只接收 @机器人 的消息。要记录群内所有消息以支持总结功能：

1. 进入开发者后台 → **应用功能 → 机器人**
2. 找到 **消息接收模式**
3. 从"仅接收 @机器人的消息"改为 **"接收群聊中所有消息"**
4. 保存并发布新版本

#### 2. 通讯录权限（用于显示用户名）

没有此权限，聊天记录将显示 Open ID 而非真实用户名：

1. 进入开发者后台 → **权限管理**
2. 搜索 `contact:user.base:readonly`
3. 启用并提交审核（企业内部应用自动通过）

### CLI 工具

这些工具主要由 AI Agent 自动调用，也可以手动运行用于调试：

```bash
# 获取聊天记录
cc-connect chatlog                          # 所有已记录的消息
cc-connect chatlog -n 100                   # 最近 100 条
cc-connect chatlog -s "feishu:oc_xxx:ou_xxx" # 指定 session key

# 清空聊天记录
cc-connect chatlog-clear
cc-connect chatlog-clear -s "feishu:oc_xxx:ou_xxx"

# 常用参数
#   -p, --project <name>    目标项目（多项目时使用）
#   -s, --session <key>     Session key（默认读取 CC_SESSION_KEY 环境变量）
#       --data-dir <path>   数据目录（默认 ~/.cc-connect）
```

### API 端点

cc-connect 内部 API 服务器（Unix socket）提供：

| 端点 | 方法 | 说明 |
|------|------|------|
| `/chatlog?session_key=...&n=...` | GET | 获取最近的聊天记录 |
| `/clear_chatlog` | POST | 清空指定群聊的记录（`{"session_key":"...","project":"..."}`) |

### Agent 集成

Agent 的系统提示词会自动扩展 `cc-connect chatlog` 和 `cc-connect chatlog-clear` 的使用说明。当用户请求总结时，Agent 会：

1. 调用 `cc-connect chatlog` 获取消息历史
2. 生成总结
3. 询问用户是否需要清空历史记录
4. 如果确认，调用 `cc-connect chatlog-clear`

无需额外配置 —— 只需确保上述飞书权限已设置。

### 示例

在飞书群聊中：

```
Alice: 我们讨论一下 API 设计
Bob:   我觉得用 REST 比较好
Carol: GraphQL 可能更适合我们的场景
Bob:   有道理，我们比较一下两种方案
  ...
你:    @bot 总结一下讨论内容
Bot:   以下是近期讨论的总结：
       - 团队讨论了 API 设计方案
       - Bob 建议使用 REST，Carol 提议 GraphQL
       - 团队同意对比两种方案

       需要清空聊天记录吗？
你:    清空吧
Bot:   聊天记录已清空。
```

---

## 技术细节

### 消息记录机制

- `Message` 结构体新增 `IsGroup` 字段，决定消息是否被记录到聊天日志。该字段由平台层设置（如飞书检查 `chatType == "group"`）。
- 被动消息（`Passive=true`）会被记录，但**不会**触发 AI Agent —— 仅静默存储。
- Chat key 通过去除 session key 中的用户部分得到：`"feishu:chatID:userID"` → `"feishu:chatID"`。

### 飞书用户名解析

- 通过飞书通讯录 API（`/open-apis/contact/v3/users/{open_id}`）解析用户名
- 结果通过 `sync.Map` 缓存，进程生命周期内有效
- API 调用设有 5 秒超时，防止挂起
- API 调用失败时回退到原始 Open ID
