---
title: "OpenClaw：把 AI 助手装进你的聊天软件"
published: 2026-03-24
description: "OpenClaw 是一个自托管的多渠道 AI 网关，让你通过 Telegram、WhatsApp、Discord 等聊天软件与 AI 助手对话。本文介绍它的核心架构、安装部署和实际使用体验。"
image: ""
tags: ["AI", "Self-Hosted", "Telegram", "OpenClaw"]
category: "技术"
---

## OpenClaw 是什么

一句话：**把 AI 编程助手塞进你的聊天软件里。**

OpenClaw 是一个开源的自托管网关（Gateway），它在你自己的服务器上运行，连接 WhatsApp、Telegram、Discord、iMessage 等聊天平台和 AI 模型（Claude、GPT、Gemini 等），让你随时随地用手机给 AI 发消息——就像跟朋友聊天一样。

```
聊天软件 (Telegram/WhatsApp/Discord...)
        ↓
    OpenClaw Gateway（你的服务器）
        ↓
    AI 模型 (Claude/GPT/Gemini...)
```

**不是又一个聊天机器人框架。** OpenClaw 的定位是「AI Agent 的多渠道网关」——它不只是转发消息，而是提供完整的会话管理、工具调用、多代理路由、定时任务、记忆系统等能力。

## 为什么需要它

你可能会问：ChatGPT 不是有 App 吗？Claude 不是有网页版吗？为什么还需要这个？

**几个真实场景：**

- 你想用 **Telegram** 随时给 AI 发消息，让它帮你管理服务器、查天气、搜资料
- 你想让 AI 助手能 **执行 shell 命令**、读写文件、浏览网页——不只是聊天
- 你想要一个 **记住上下文** 的助手，跨对话记得你之前说过什么
- 你想用 **自己的 API Key**，不受官方客户端的限制
- 你想在 **多个平台** 同时使用同一个助手——手机 Telegram、电脑 Discord、甚至 iMessage

OpenClaw 满足以上所有需求，而且数据全在你自己的服务器上。

## 核心特性

### 🔗 多渠道支持

内置支持 WhatsApp、Telegram、Discord、iMessage，通过插件还可以扩展到 Mattermost、Matrix、Slack 等。一个 Gateway 进程可以同时连接多个渠道。

### 🤖 多模型 & 多代理

支持 35+ 模型提供商：Anthropic (Claude)、OpenAI (GPT)、Google (Gemini)、以及各种自托管模型（Ollama、vLLM 等）。可以配置多个 AI 代理，每个有独立的会话空间和工具集。

### 🛠️ Agent 工具链

这才是 OpenClaw 真正强大的地方：

- **Shell 执行**：AI 可以直接在服务器上运行命令
- **浏览器自动化**：控制 Chromium 浏览网页、截图、填表
- **网页搜索**：集成 Brave、Perplexity、Grok 等搜索引擎
- **文件读写**：读取和编辑服务器上的文件
- **定时任务（Cron）**：设置定期执行的自动化任务
- **记忆系统**：向量搜索 + 日记式记忆管理
- **子代理**：可以派生隔离的子任务

### 📱 移动节点

iOS 和 Android 节点可以配对到 Gateway，提供摄像头、屏幕录制、位置获取等设备级能力。

### 🎨 Skills 技能系统

类似插件的技能系统，可以给 AI 助手添加专业能力。社区有现成的技能包可以直接安装。

## 安装部署

### 环境要求

- Node.js 24（推荐）或 Node.js 22.16+
- 一个 AI 模型提供商的 API Key
- 一台服务器（VPS、树莓派、或者你的笔记本都行）

### 快速安装

```bash
# 一键安装
curl -fsSL https://openclaw.ai/install.sh | bash

# 运行引导设置
openclaw onboard --install-daemon

# 检查状态
openclaw gateway status
```

引导程序会带你选择模型提供商、配置 API Key，整个过程大约 2 分钟。

### 连接 Telegram

Telegram 是最快的接入方式：

1. 在 Telegram 找 **@BotFather**，创建一个 Bot，拿到 Token
2. 编辑配置文件 `~/.openclaw/openclaw.json`：

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "你的Bot Token",
      allowFrom: [你的Telegram用户ID],
    },
  },
}
```

3. 重启 Gateway，给你的 Bot 发消息——AI 就会回复你了

### 配置模型

OpenClaw 支持自定义反向代理，适合国内用户：

```json5
{
  models: {
    providers: {
      anthropic: {
        baseUrl: "https://你的反代地址",
        apiKey: "你的API Key",
        models: [
          {
            id: "claude-opus-4-6",
            name: "Claude Opus 4.6",
            contextWindow: 500000,
          },
        ],
      },
    },
  },
}
```

## 实际使用体验

我已经用 OpenClaw + Claude Opus 4.6 跑了一周。说说真实感受：

### 优点

- **随时可用**：手机 Telegram 发消息就能用，比开网页方便太多
- **工具能力强**：AI 能直接操作服务器，装软件、写脚本、管理文件
- **记忆系统好用**：向量搜索 + 日记系统，AI 能记住之前的对话和决策
- **Cron 任务**：定时同步记忆、检查服务器状态，完全自动化
- **开源自控**：数据在自己服务器上，API Key 自己管理

### 不足

- **文档偏英文**：中文文档不够完善
- **部分工具不稳定**：比如 `message` tool 发图片偶尔报错
- **配置项多**：初始配置有一定门槛，需要对 JSON5 格式比较熟悉
- **资源占用**：Gateway 本身不重，但浏览器自动化等功能需要额外资源

### 适合谁

- 有自己服务器的开发者
- 想要「随身 AI 助手」的 Power User
- 对数据隐私有要求的用户
- 想折腾 AI Agent 自动化的爱好者

## 总结

OpenClaw 不是一个完美的产品，但它解决了一个真实的问题：**让 AI 助手真正融入你的日常通讯工具**。

如果你已经有一台 VPS，有 AI 模型的 API Key，花 5 分钟装一个试试。当你在地铁上用 Telegram 让 AI 帮你查看服务器状态的时候，你会觉得这 5 分钟花得值。

---

- 官网：[openclaw.ai](https://openclaw.ai)
- 文档：[docs.openclaw.ai](https://docs.openclaw.ai)
- GitHub：[github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)
- 社区 Discord：[discord.com/invite/clawd](https://discord.com/invite/clawd)
