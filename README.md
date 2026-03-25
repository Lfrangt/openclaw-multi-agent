# OpenClaw 多 Agent 协作架构

> 这是 Khalil 的 OpenClaw 实战架构，供参考和复用。

---

## 核心思路

不要只有一个"全能 AI"，而是像公司一样分工：
- 每个 Telegram topic 是一个部门
- 每个部门有专属 agent 负责
- agent 之间通过文件或消息协作

---

## 架构总览

```
Telegram 无骨家族群
├── 💬 General (topic:18)     → 无骨宏基（主 agent）
├── 🤵 CEO (topic:1414)       → 无骨总裁（pulse-ceo agent）
├── 💻 Coding (topic:10)      → 无骨宏基（主 agent）
├── 🧠 Learning (topic:8)     → 无骨宏基（词汇推送）
├── 💪 Fitness (topic:1194)   → 无骨宏基（健康教练）
├── 📋 Tasks (topic:6)        → 无骨宏基
├── 💰 Business (topic:14)    → 无骨宏基
└── 📖 Books (topic:12)       → 无骨宏基
```

---

## Agent 1：无骨宏基（主 agent）

**角色：** 全能执行者 + 私人助理  
**Bot：** @wuguhongji_bot  
**负责：** 所有 topic（除 CEO）

**做什么：**
- Coding topic：用 Claude Code PTY 写代码、build、push
- Learning topic：推送学习内容（可自定义，如词汇、资讯等）
- Fitness topic：读 Apple Health 数据，给训练建议
- General topic：日常对话、信息搜索、文件处理

**配置文件：**
```
~/.openclaw/workspace/SOUL.md      # 人设和行为规范
~/.openclaw/workspace/MEMORY.md    # 长期记忆
~/.openclaw/workspace/NOW.md       # Session 交接快照
~/.openclaw/workspace/USER.md      # 用户信息
```

---

## Agent 2：无骨总裁（pulse-ceo agent）

**角色：** 产品 CEO + 商业策略  
**Bot：** @wuguCEO_bot  
**负责：** CEO topic only  
**Workspace：** `~/.openclaw/workspaces/pulse-ceo/`

**做什么：**
- 产品策略、ASO 优化、用户增长
- Review Coding agent 的完成情况
- 派发新任务到 Coding topic
- 维护产品路线图

**配置文件：**
```
~/.openclaw/workspaces/pulse-ceo/SOUL.md   # CEO 人设
~/.openclaw/workspaces/pulse-ceo/NOW.md    # 进度交接
```

---

## 两个 Agent 如何协作

### 核心机制：共享任务文件

```
~/.openclaw/workspace/pulse-tasks.json
```

```json
{
  "queue": [
    {
      "task": "实现用户引导页",
      "priority": "P0",
      "goal": "提升新用户留存",
      "acceptance": "3步引导，本地化，build通过",
      "assigned_at": "2026-03-20T10:00:00"
    }
  ],
  "completed": [...]
}
```

### 工作流程

```
CEO 在 CEO topic 分析产品进展
    ↓ 派任务（写入 pulse-tasks.json）
Coding topic 的宏基接到任务
    ↓ 用 Claude Code 写代码
    ↓ Build + commit + push
    ↓ 通知 CEO topic 完成
CEO review → 派下一个任务
    ↓
循环 ♻️
```

### 触发机制

**CEO → Coding：**
```bash
# CEO 写完任务后，注入到 Coding session
openclaw agent \
  --session-key "agent:main:telegram:group:GROUP_ID:topic:10" \
  --message "📋 CEO 新任务：[任务描述]" \
  --channel telegram --deliver --reply-to "GROUP_ID"
```

**Coding → CEO（git hook）：**
```bash
# .git/hooks/post-commit
openclaw agent \
  --session-key "agent:pulse-ceo:telegram:group:GROUP_ID:topic:CEO_TOPIC_ID" \
  --message "📊 Coding 完成：$(git log --oneline -1)" \
  --channel telegram --deliver --reply-to "GROUP_ID"
```

每次 git commit 自动通知 CEO，CEO 收到后 review + 派新任务，形成闭环。

---

## Cron 任务

| 名称 | 时间 | 负责 | 说明 |
|------|------|------|------|
| morning-brief | 每天 8:00 | 主 agent | 健康晨报 |
| daily-tech-digest | 每天 9:00 | 主 agent | HN/GitHub 技术摘要 |
| agent-loop-check | 每 10 分钟 | CEO agent | 检查任务进展 |

---

## 记忆系统

### 分层架构

```
SOUL.md         → 核心人设（每次都读）
USER.md         → 用户信息（每次都读）
NOW.md          → Session 交接（新 session 先读）
MEMORY.md       → 长期记忆（主 session 读）
memory/YYYY-MM-DD.md  → 每日日记
memory/knowledge.json → 结构化知识库（v2）
```

### 语义搜索

```bash
# 搜索历史记忆
python3 ~/.openclaw/workspace/memory/memory_search.py "PulseWatch 项目"

# 只更新索引
python3 ~/.openclaw/workspace/memory/memory_search.py --index-only
```

基于 OpenAI text-embedding-3-small，余弦相似度检索。

---

## 复用这套架构的步骤

### 1. 基础配置

```bash
# 在 SOUL.md 写自己的身份和规则
# 在 USER.md 写用户信息
# 在 AGENTS.md 配置启动流程
```

### 2. 创建第二个 Agent（如 CEO）

```bash
# 新建 agent
openclaw agents add my-ceo-agent \
  --workspace ~/.openclaw/workspaces/my-ceo/ \
  --non-interactive

# 配置 Telegram bot
openclaw config set channels.telegram.accounts.my-ceo.botToken "BOT_TOKEN"
openclaw config set channels.telegram.accounts.my-ceo.groupPolicy "open"

# 绑定到 bot
openclaw agents bind --agent my-ceo-agent --bind telegram:my-ceo
```

### 3. 路由 Topic 到指定 Agent

```bash
# topic 1414 的消息路由给 CEO agent
openclaw config set 'channels.telegram.groups.GROUP_ID.topics' \
  '{"1414":{"agentId":"my-ceo-agent"}}' --json
```

### 4. 设置 Git Hook（自动协作）

```bash
cat > ~/Projects/your-project/.git/hooks/post-commit << 'EOF'
#!/bin/bash
openclaw agent \
  --session-key "agent:my-ceo-agent:telegram:group:GROUP_ID:topic:CEO_TOPIC_ID" \
  --message "📊 Coding 完成：$(git log --oneline -1). 请 review 并派下一个任务" \
  --channel telegram --deliver --reply-to "GROUP_ID" &
EOF
chmod +x ~/Projects/your-project/.git/hooks/post-commit
```

---

## 关键踩坑

1. **Bot 消息不触发 session** — bot 自己发的消息不会触发对方的 session，要用 `openclaw agent --session-key` 注入
2. **requireMention 默认 true** — 新 agent 在群里默认需要 @ 才回复，要设成 false
3. **Topic 路由** — `channels.telegram.groups.GROUP_ID.topics` 要传 JSON 对象不是数组
4. **Privacy Mode** — BotFather 要先 `/setprivacy → Disable` 再把 bot 重新踢出加回群，否则收不到消息
5. **Claude Code PTY** — 用 `--dangerously-skip-permissions`，不要用 `--print`（需要 API key）

---

## 适合这套架构的场景

- 独立开发者：CEO agent 管产品，Coding agent 执行
- 学习者：Study agent 管课程计划，主 agent 执行
- 创作者：Editor agent 管内容方向，Writer agent 创作
- 任何需要"决策 + 执行"分工的场景

---

*文档基于 Khalil 的实际生产环境，OpenClaw 2026.3.13*
