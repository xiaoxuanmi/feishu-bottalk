# feishu-bottalk

> 飞书多机器人互聊解决方案（基于 OpenClaw）

---

## 📌 背景

在飞书群聊中：

- 🤖 机器人之间**无法互相看到消息**
- 导致多 Agent 无法直接在飞书中协作

而在 OpenClaw 中：

- Agent 之间可以通过 `sessions_send` 通信
- 但 **不会同步到飞书群聊**

👉 结果：  
**人看不到 Agent 对话，Agent 也无法通过飞书交互**

---

## 🚀 解决方案

本项目提供一个 **双通道插件（feishu-bottalk）**：

当 Agent 发言时：

1. ✅ 通过 `sessions_send` → 发送给其他 Agent  
2. ✅ 同步发送 → 飞书群聊  

从而实现：

> **在飞书中“模拟机器人互相聊天”**

---

## 🧩 适用场景

- 多 Agent（如：研发 / 产品 / 老板）
- 同一 OpenClaw 实例
- 接入同一个飞书群

---

## ⚙️ 安装与配置

---

### 0️⃣ 前提条件

请先完成：

- ✅ OpenClaw 安装 + onboard
- ✅ 配置 LLM & gateway
- ✅ 安装官方飞书插件 `@openclaw/feishu`
- ✅ 在飞书开发者平台创建多个机器人

示例角色：

- `rd`（研发）
- `pm`（产品）

---

### 1️⃣ 创建 Agent

```bash
openclaw agents add rd
openclaw agents add pm
```

确认 `openclaw.json`：

```json
{
  "agents": {
    "list": [
      {
        "id": "rd",
        "workspace": "/root/.openclaw/workspace-rd"
      },
      {
        "id": "pm",
        "workspace": "/root/.openclaw/workspace-pm"
      }
    ]
  }
}
```

---

### 2️⃣ 绑定飞书机器人

#### 2.1 添加飞书账号

```json
"channels": {
  "feishu": {
    "accounts": {
      "rd": {
        "enabled": true,
        "appId": "...",
        "appSecret": "..."
      },
      "pm": {
        "enabled": true,
        "appId": "...",
        "appSecret": "..."
      }
    }
  }
}
```

---

#### 2.2 配置长连接

在飞书后台：

- 开启 WebSocket
- 添加事件：

```
im.message.receive_v1
```

---

#### 2.3 Agent 绑定关系

```json
"bindings": [
  {
    "agentId": "rd",
    "match": {
      "channel": "feishu",
      "accountId": "rd"
    }
  },
  {
    "agentId": "pm",
    "match": {
      "channel": "feishu",
      "accountId": "pm"
    }
  }
]
```

---

### 3️⃣ 安装插件

```bash
git clone https://github.com/xiaoxuanmi/feishu-bottalk
cd feishu-bottalk/extensions/feishu-bottalk

npm pack
openclaw plugins install feishu-bottalk-*.tgz
```

成功标志：

```
Installed plugin: feishu-bottalk
```

---

### 4️⃣ 开启插件权限（关键）

#### 4.1 Agent 允许插件

```json
"tools": {
  "allow": ["*", "group:plugins"]
}
```

---

#### 4.2 全局 tools 配置

⚠️ 必须删除：

```json
"tools": {
  "profile": "coding"
}
```

否则插件可能被过滤！

---

#### 4.3 启用 sessions

```json
"tools": {
  "sessions": {
    "visibility": "all"
  },
  "agentToAgent": {
    "enabled": true
  }
}
```

---

#### 4.4 插件白名单

```json
"plugins": {
  "allow": [
    "feishu",
    "feishu-bottalk"
  ]
}
```

---

### 5️⃣ Gateway 放行

```json
"gateway": {
  "tools": {
    "allow": [
      "sessions_send"
    ]
  }
}
```

---

## 🧠 Agent 行为约束（关键设计）

---

### AGENTS.md（通信规则）

```md
## Reply Protocol（必须遵守）

你只能做两件事：

1. 调用 `feishu-bottalk`
2. 输出 `NO_REPLY`

❌ 禁止：

- 直接回复文本
- 使用 sessions_send

✅ 正确：

- 想说话 → feishu-bottalk
```

---

### SOUL.md（角色定义）

```md
你是研发工程师 agent，负责技术实现与系统落地。

核心原则：

- 工程优先
- 稳定第一
- 问题导向
```

---

### IDENTITY.md

```md
- Name: 研发
- Emoji: 🔧
- Role: 技术解决方案提出者
```

---

## 🔁 工作机制

```
Agent 发言
   ↓
feishu-bottalk 插件
   ↓
├── sessions_send → 其他 Agent
└── 飞书群消息 → 人类可见
```

---

## ✅ 启动验证

```bash
openclaw gateway restart
```

查看日志：

```
feishu-bottalk: Registered feishu-bottalk tool
```

---

## 🧪 测试效果

- 多 Agent 在飞书群中可见对话
- 支持多轮互动
- 人类可观察全过程

---

## 🛠️ 故障排查

### ❌ Agent 看不到插件

👉 检查：

- tools.allow
- plugins.allow
- 是否删除 `"profile": "coding"`

---

### ❌ 角色错乱

👉 检查：

- bindings 配置
- accountId / agentId 是否一致

---

### ❌ sessions_send 404

👉 检查：

- gateway.tools.allow 是否包含 `sessions_send`

---

## 🔍 调试工具

```bash
./read_session {agentId}
```

作用：

- 查看 Agent ↔ LLM 实际交互
- 判断问题来源：
  - LLM 没响应
  - 返回 NO_REPLY
  - tool 调用失败

---

## 🎯 总结

本项目核心价值：

- ✅ 打通 Agent ↔ Agent 通信
- ✅ 同步飞书可视化
- ✅ 支持多角色协作（研发 / 产品 / 老板）