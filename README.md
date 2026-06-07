[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) [![Feishu](https://img.shields.io/badge/Platform-Feishu-blue.svg)](https://www.feishu.cn)

# Meeting Flow · 会议纪要自动化

> **说一句话，会议纪要自动写好。** 录音上传 → 豆包 ASR 转写 → AI 生成纪要 → 回填飞书文档 + 多维表格，全程无需人工干预。
<img width="1693" height="929" alt="1" src="https://github.com/user-attachments/assets/46ddd7e2-fc15-4849-b411-451ed5313052" />
<img width="1536" height="1024" alt="3" src="https://github.com/user-attachments/assets/98695a95-0849-4d88-bfc3-0348f644d8c1" />
<img width="1536" height="1024" alt="2" src="https://github.com/user-attachments/assets/d76edc33-53e7-4d5f-b3c4-0284380bd12d" />

---

## 整体流程

```
你私聊说"记会议纪要"
       ↓
Bot 弹飞书卡片（表单链接）
       ↓
填写表单：会议名称 + 模板选择 + 上传录音
       ↓
提交到飞书多维表格
       ↓
多维表格自动化 → 消息通知到工作群 → @Bot
       ↓
Bot 在群里收到通知 → 从 bitable 获取待处理记录
       ↓
获取录音公网直链 → 豆包 ASR 转写（带说话人分离）
       ↓
读取对应 prompt skill → AI 生成纪要
       ↓
创建飞书文档 + 写 Obsidian + 回填 bitable + 群里通知完成
```

---

## 功能亮点

- 🎙️ **一句话触发** — 私聊 Bot 说"记会议纪要"，弹出表单卡片，上传录音即可
- 🔊 **说话人分离** — 豆包 ASR 自动识别不同发言人，谁说了什么一目了然
- 📝 **三种纪要模板** — 商务版 / 详细版 / 行动项版，按场景选择
- 🔄 **全自动闭环** — 提交后无需任何操作，处理完成自动在群里通知
- 📊 **多维表格追踪** — 每条记录状态可视化：待处理 → 处理中 → 已完成
- 📄 **双重存档** — 纪要同时写入飞书文档和本地 Obsidian

---

## 架构概览

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   飞书 Bot    │ ←→  │   openclaw   │ ←→  │  飞书多维表格  │
│  (消息收发)    │     │  (流程编排)    │     │ (录音+状态追踪) │
└──────────────┘     └──────┬───────┘     └──────────────┘
                            │
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
       ┌────────────┐ ┌─ ─ ─ ─ ─┐ ┌─ ─ ─ ─ ─┐
       │  豆包 ASR   │ │ Codex CLI│ │ Obsidian │
       │ (语音转文字) │ │(纪要生成) │ │(本地备份) │
       └────────────┘ └─ ─ ─ ─ ─┘ └─ ─ ─ ─ ─┘
                           可选          可选
```

> 💡 虚线框为可选组件。未安装 Codex CLI 时，openclaw 内置 AI 直接生成纪要；不使用 Obsidian 则跳过本地备份步骤。

| 组件 | 用途 | 必须？ |
|------|------|--------|
| openclaw + 飞书 Bot | 消息收发、事件驱动 | ✅ |
| 飞书多维表格 + 表单 | 录音收集、状态追踪 | ✅ |
| 豆包 ASR（火山引擎） | 录音转写 + 说话人分离 | ✅ |
| Codex CLI | 沙箱内 AI 生成纪要 | ⬜ 可选 |
| Obsidian | 本地 Markdown 备份 | ⬜ 可选 |

---

## 快速开始

完整配置大约需要 **15 分钟**，详见 **[SETUP.md](SETUP.md)**。

### 你需要准备

1. **飞书开放平台应用** — 创建机器人，获取 App ID / Secret
2. **火山引擎 ASR** — 开通录音文件识别，获取 App ID / Token
3. **一个飞书多维表格** — 配置帮你自动创建

### 三步跑通

```bash
# 1. 克隆项目
git clone git@github.com:joyye-spec/meeting-flow.git
cd meeting-flow

# 2. 跟着 SETUP.md 走完配置（agent 引导，11 步自动化）

# 3. 私聊 Bot 说"记会议纪要"，上传测试录音验证链路
```

---

## 项目结构

```
📂 meeting-flow
├── README.md           ← 项目介绍（你在看）
├── SETUP.md            ← 11 步配置指南（agent 引导版）
├── SKILL.md            ← Skill 核心逻辑（5 阶段流水线）
└── assets/
    ├── workflow.png    ← 整体流程图
    ├── architecture.png← 系统架构图
    └── state.png       ← 记录状态流转图
```

---

## 一条记录的生命周期

```
 用户提交表单
      ↓
 [状态: 空]  → 等待触发
      ↓
 [状态: 处理中]  → ASR 转写中 / AI 生成中
      ↓
 [状态: 已完成]  → 纪要链接已回填
      ↓
 [状态: 失败]    → 异常卡死，下次自动重试
```

---

## License

MIT
