---
title: "OpenClaw 一周深度体验：一个自托管 AI 网关的真实面貌"
published: 2026-03-24
description: "从 3 月 17 日部署到今天，我用 OpenClaw + Claude Opus 4.6 跑了整整一周。这不是一篇官方文档翻译，而是一个真实用户踩过的坑、发现的惊喜、以及冷静的评价。"
image: ""
tags: ["AI", "Self-Hosted", "Telegram", "OpenClaw", "深度体验"]
category: "技术"
---

## 写在前面

OpenClaw 是一个开源的自托管 AI 网关——把 Claude、GPT、Gemini 这些 AI 模型连接到你的 Telegram、WhatsApp、Discord，让你随时随地用聊天软件和 AI 对话。

从 3 月 17 日部署至今刚好一周。这篇不是功能列表的复读，而是**真实使用中发现的好与坏**。

## 架构一句话

```
聊天软件 → OpenClaw Gateway（你的服务器）→ AI 模型
```

Gateway 是中枢，所有消息经过它路由。它不只是消息转发器——它管理会话、调用工具、执行代码、记忆上下文。

## 真正让我惊喜的地方

### 记忆系统

这是 OpenClaw 最被低估的能力。它不是简单的上下文窗口——而是一个**三层记忆架构**：

- **短期**：当前会话的压缩摘要（compaction 自动管理）
- **中期**：日记文件（`memory/YYYY-MM-DD.md`），AI 主动写入
- **长期**：`MEMORY.md`，精华蒸馏

配合**向量语义搜索**（`memory_search`），AI 能回忆起几天前的对话细节。这不是花哨的 feature——当你第三天问「之前那个代理叫什么端口来着」，它真的能翻出来。

我设了两个 cron 任务：每天 14:00 和 22:00 自动同步日记，凌晨 4:00 整理压缩旧日记到周报。**AI 自己管理自己的记忆**，不需要人工维护。

### Compaction（上下文压缩）

长对话不会爆上下文。OpenClaw 的 compaction 机制会在接近窗口上限时自动压缩历史，保留关键信息。我把 `contextWindow` 从 1M 调到了 500K——因为实测 Claude Opus 4.6 在 256K 内注意力准确率 93%，到 1M 降到 76%（MRCR v2 基准）。500K 是甜蜜点：够大不会频繁压缩，又不会因为 context rot 丢失注意力。

### 工具能力是真·Agent 级别

这不是「帮你搜搜网页」的水平。一周下来，我的 AI 助手做了这些事：

- **安装和配置 SS 代理**：配置 4 条代理线路，写 systemd 服务，测速选优
- **CF 过盾研究**：测试 scrapling、CloakBrowser、Camoufox 三种反检测浏览器，找到最优方案
- **分析 GitHub 钓鱼仓库**：从混淆的 JS 代码还原完整攻击链（Base91 解码 + RCE payload 分析）
- **给手机写代理配置**：生成 Mihomo/Surfing 模块的 YAML 配置，含 Google Play 规则和 DNS 分流
- **部署这个博客**：从选型到建站到 Vercel 部署到 DNS 配置，全程自动

这些不是演示用例——是真实的生产力。`exec` 工具让 AI 直接操作服务器，`browser` 工具可以自动化 Chromium，`web_search` + 自定义搜索技能提供信息获取能力。

### Cron 定时任务

不只是提醒闹钟。你可以让 AI 定期执行复杂任务——用便宜的模型（我用 Gemini Flash）跑定时脚本，主模型只处理交互对话。这是**真正的模型分工**：

- 日常对话：Claude Opus 4.6 Fast（质量高）
- Cron 任务 + 多模态识别：Gemini 3 Flash（便宜够用）

### 技能系统（Skills）

Markdown 格式的技能文件，可以教 AI 新能力。我装了 15 个技能：搜索、画图（NovelAI/Gemini）、网页保存、PDF 解析、GitHub Issues 管理……社区（ClawHub）也有现成的可以装。

但要注意：**ClawHub 上有恶意技能**。安全研究人员发现了超过 1100 个恶意技能分发 Atomic Stealer。安装前一定要审查内容。

## 踩过的坑（这些文档不会告诉你）

### 安全问题是真的严重

这是必须正视的事实。2026 年 2 月曝出的 **CVE-2026-25253**（CVSS 8.8）是个一键 RCE 漏洞：

- 攻击者在恶意网页里用 JS 连接你本地的 `localhost:18789`
- 通过 WebSocket 偷取 Gateway token
- 拿到 token = 拿到完整的 shell 权限

**绑定 localhost 也没用**——攻击通过受害者的浏览器中转。超过 13.5 万个 OpenClaw 实例暴露在公网上。Microsoft 的安全博客直接说「对大多数环境，合适的决定可能是不部署它」。

虽然 v2026.2.25+ 已修复，但这暴露了一个根本问题：**OpenClaw 的安全模型假设只有你自己在用**。如果你部署在 VPS 上、暴露了端口、或者连接了聊天渠道——攻击面会大幅扩展。

**建议**：一定要升级到最新版、设置 `gateway.bind: "loopback"`、配置 `allowFrom` 白名单、不要在公网暴露 Gateway 端口。

### message 工具有 bug

`message` tool 的 schema 设计有缺陷：

- `buttons` 参数是全局 required 的（即使你不需要按钮），需要传空数组 `[]` 绕过
- `edit` 动作不会自动推断当前聊天，需要显式传 `target`
- 主动发送图片文件（`filePath`）偶尔报 `Channel is unavailable`——但同一个 channel 正常收发文字消息
- 工具失败时会自动给用户发 ⚠️ 错误通知，暴露内部错误信息

### 流式输出的 NO_REPLY 陷阱

OpenClaw 是流式输出的——文字生成多少发多少。如果 AI 先输出了一段文字，最后又加了 `NO_REPLY`（表示不需要回复），前面的文字已经发到聊天软件了，后面检测到 NO_REPLY 又去删除……用户看到的就是**消息闪烁然后消失**。

这个坑很隐蔽，文档也没提。

### contextWindow 不是越大越好

默认可以设到 1M，但 Claude Opus 4.6 在大上下文窗口下注意力会衰减。实测 MRCR v2 基准：256K 准确率 93%，1M 降到 76%。**不要因为能设大就设大**——500K 是我测出来的平衡点。

### 配置门槛不低

配置文件是 JSON5 格式（`~/.openclaw/openclaw.json`），功能强大但字段巨多。模型配置要手动写 provider、API、contextWindow、cost……没有 GUI，没有向导（`openclaw onboard` 只覆盖基础配置）。反代用户还得自己处理 API 格式兼容性。

### dev 版和 stable 版差异

dev 版（git clone）占 2.7GB，npm stable 版 579MB，功能相同。如果你不需要改源码，**用 npm 版**，省空间也方便更新。

### Compaction 不触发 memory flush

这个我翻源码才确认的：手动执行 `/compact` 不会触发 pre-compaction 的 memory flush。只有**自动 compaction**（上下文快满时）才会先 flush 再压缩。如果你依赖 memory flush 来保存重要信息，别手动 compact。

## 和替代品的比较

调研了几个同类项目：

| 项目 | 语言 | 特点 | 适合谁 |
|------|------|------|--------|
| **OpenClaw** | Node.js | 功能最全，生态最大，但安全问题多 | 能折腾的开发者 |
| **nanobot** | Python | 4K 行代码，26.8K 星，轻量极简 | 想要简单可控的用户 |
| **ZeroClaw** | Rust | <5MB 二进制，WASM 沙箱 | 资源受限环境 |
| **LoongClaw** | Rust | 团队协作向，权限管理完善 | 企业/团队 |

OpenClaw 的优势是生态和功能全面性，劣势是复杂度和攻击面。如果你只需要简单的聊天转发，nanobot 可能更合适。

## 实际资源占用

在我的 VPS（Debian 12, 2C4G）上：

- Gateway 进程本身很轻，~100MB 内存
- 但浏览器自动化（patchright/Chromium）可以吃到 3GB
- 加上 xvfb、SS 代理等辅助进程，日常占用约 1.5-2GB
- 磁盘：npm 版本 ~580MB + workspace + 浏览器缓存，总共约 3-4GB

## 写给想尝试的人

### 最快上手路径

```bash
# 安装
curl -fsSL https://openclaw.ai/install.sh | bash

# 引导配置
openclaw onboard --install-daemon

# 连接 Telegram（最快的渠道）
# 找 @BotFather 创建 Bot → 拿 Token → 写入配置 → 重启
```

### 安全清单

- [ ] 升级到 v2026.2.25+（修复 CVE-2026-25253）
- [ ] `gateway.bind` 设为 `loopback`
- [ ] 配置 `channels.telegram.allowFrom`（只允许你自己的 ID）
- [ ] 不要在公网暴露 18789 端口
- [ ] 审查任何第三方技能的内容再安装
- [ ] 定期 `openclaw update`

### 模型配置建议

如果用反代（国内用户大概率需要）：

```json5
{
  models: {
    providers: {
      anthropic: {
        baseUrl: "https://你的反代地址",
        apiKey: "你的Key",
        models: [{
          id: "claude-opus-4-6-fast",
          contextWindow: 500000,  // 不要设1M
          maxTokens: 128000,
        }],
      },
    },
  },
}
```

## 结论

OpenClaw 不是一个开箱即用的产品。它是一个**强大但粗糙的工具**，适合愿意花时间配置和维护的开发者。

一周下来，它帮我完成了大量本来需要手动 SSH 到服务器操作的事情，记忆系统让跨会话的连续性工作成为可能，Cron 任务实现了真正的自动化。这些是实实在在的生产力提升。

但安全问题、工具稳定性、配置复杂度——这些不是小事。如果你不愿意花时间理解它的安全模型和配置体系，**不建议部署在有敏感数据的环境**。

**评分：7/10** — 功能强大，生态活跃，但安全和稳定性还需要打磨。

---

- 官网：[openclaw.ai](https://openclaw.ai)
- 文档：[docs.openclaw.ai](https://docs.openclaw.ai)
- GitHub：[github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)
- 安全公告：[CVE-2026-25253](https://thehackernews.com/2026/02/openclaw-bug-enables-one-click-remote.html)
