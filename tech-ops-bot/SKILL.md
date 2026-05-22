---
name: tech-ops-bot
description: 企业微信/飞书 IT运维机器人 — 群消息监控、故障分类、知识库问答、每日日报、Web驾驶舱。支持企微自建应用或飞书机器人。
version: 1.0
category: productivity
tags: [企业微信, 飞书, IT运维, 故障管理, 知识库, 自动化, 机器人]
license: MIT
homepage: https://github.com/winterliu6/hermes-skills
issues: https://github.com/winterliu6/hermes-skills/issues
---

# IT运维机器人 (Tech Ops Bot)

## 核心架构

```
用户电脑 (24h开机)
  ├── Python服务
  │   ├── 企微/飞书API对接 (收/发消息)
  │   ├── 故障分类引擎 (关键词规则)
  │   ├── 知识库引擎 (SQLite检索)
  │   ├── 日报生成器 (每天定时推送)
  │   └── Web驾驶舱 (Flask+ECharts)
  └── 本地存储
      ├── data/*.db (SQLite - 故障/知识/统计)
      ├── export/*.xlsx (周报导出)
      └── config/*.yaml (门店/人员/设备)
```

**数据安全**: 仅与官方API通信，所有数据本地存储，不外传。

## 支持平台

| 平台 | 接入方式 | 优点 |
|------|---------|------|
| 企业微信 | 自建应用（推荐） | 官方API不封、语音自动转文字 |
| 飞书 | 自建应用 | 能力丰富、支持多语言 |
| 企微群机器人 | Webhook | 简单，但只能发不能收 |

## 企微自建应用接入

### vs 其他方案

| 方式 | 能力 | 稳定性 |
|------|------|--------|
| 群机器人(wenook) | 只能发，不能收 | 稳但无用 |
| 个人微信(itchat) | 能收能发 | 封号风险 |
| **企微自建应用** | 能收能发，语音自动转文字 | 官方API不封 |

### 配置步骤

1. 企微后台 → 应用管理 → 自建 → 创建应用
2. 获取 `AgentId` 和 `Secret`
3. 应用拉进客户群(外部群)
4. 填到 `config/wework.yaml`

### 回调 vs 轮询

- **回调**: 需要公网URL，消息实时
- **轮询**: 不依赖公网，每分钟拉一次（企微消息有1分钟延迟，够用）

初期建议轮询，稳定后再升级回调。

## 故障分类引擎

**不用AI，用关键词规则** — 稳定、可控、零成本：

```python
规则示例:
  ["画面", "闪烁", "花屏", "黑屏"] → "放映-画面故障"
  ["没声音", "无声", "爆音"]       → "放映-声音故障"
  ["好了", "搞定", "修好了"]       → "__RESOLVED__" (关闭故障)
  ["网络", "上不了", "断网"]       → "网络故障"
```

会话合并(去重)：同一店同一厅30分钟内的消息自动归并为一个故障记录。

### 支持的故障类型（示例）

- 放映-画面故障 / 放映-声音故障
- 放映-播放服务器故障 / 放映-投影机故障 / 放映-TMS故障
- 网络故障 / 票务系统故障
- IT-服务器故障 / IT-安全事件 / IT-办公设备
- 放映-大屏LED故障 / 日常巡检

## 消息处理流程

收到群消息后的处理顺序：

1. **去除 @ 机器人文本**，提取纯消息内容
2. **特殊指令优先判断**：驾驶舱地址 / 故障解决确认
3. **故障分类**：`classify_message()` → 得到故障类型标签
4. **知识库查询**：`search_knowledge()` → 有结果直接回复可能原因+解决方案
5. **联网搜索**（知识库无记录时）：用搜狗搜索，提取摘要片段回复
6. **写入故障记录**：`add_fault()` → 写 SQLite
7. **回复**：发送知识库方案或搜索结果，**不要只回"收到！问题已记录"**

关键原则：**必须先尝试给出解决方案，再记录问题**。

## 知识库

### 知识库转换管线

用户可能提供PDF/Excel/Word文件（非markdown）。转换方案：

```
知识库目录/
├── 原始文件/        ← 用户丢原始文件进去
│   ├── 票务手册.pdf
│   ├── 运维规范.xlsx
│   └── 故障记录.docx
├── convert_to_md.py ← 一键转换脚本
├── 票务手册.md      ← 自动转换结果
├── 运维规范.md
└── 故障记录.md
```

### 转换脚本依赖

```bash
pip install pymupdf openpyxl python-docx
```

## 日报/周报

- 每天定时自动发Markdown日报到群(昨日故障汇总)
- 每周导出Excel周报(明细+人员统计+排行)
- 发送方式: 企微 `message/send` API (markdown消息类型)

## Web驾驶舱

- Flask + ECharts 单页应用
- 只绑内网IP（外网不可访问）
- 展示: 故障总数/今日新增/月度处理/平均时长 → 类型分布→门店排行→趋势→最近故障列表→知识库搜索

## 方案B：Hermes Agent + soul.md（轻量方案）

如果直接用Hermes Agent接企微（而非写Python服务），架构更轻量：

```
企微回调URL → Hermes Gateway → soul.md控制行为
  ├─ 没@我 → 静默记录SQLite（报障日志）
  ├─ @了我 → 搜索知识库 → 回复
  └─ cron → 查昨日记录 → 发日报
```

### soul.md 核心结构

```markdown
## 核心职责
1. 日常群聊监控（静默模式）
   - 群里的报障消息自动记录本地SQLite
   - 不回复非@我的消息
2. 技术客服（被@时触发）
   - 查询本地知识库
   - 按问题匹配解决方案，步骤化回答
3. 每日工作简报（定时）
   - 昨日问题总数、已解决、未解决
```

## 飞书接入（备选）

如果企微无法落地，可用飞书替代：

1. 飞书开放平台 → 创建企业自建应用
2. 开启机器人能力
3. 配置回调URL到内网Flask服务
4. 同样支持消息收发、群管理

详见 `references/feishu-bot-reference.md`

## 本地目录结构

```
E:/tech-ops-bot/           ← 项目根目录
├── config/
│   ├── wework.yaml       ← 企微配置（corp_id/agent_id/secret）
│   ├── shops.yaml        ← 门店/分店清单
│   ├── staff.yaml        ← 人员-区域对应
│   └── equipment.yaml    ← 厅-设备对应
├── data/
│   ├── records.db        ← 故障/消息记录
│   ├── knowledge.db      ← 知识库
│   └── stats.db          ← 统计数据
├── data/knowledge/       ← 维修手册、故障代码表
├── export/               ← 周报Excel
├── scripts/
│   ├── main.py           ← 主服务入口
│   ├── database.py       ← SQLite CRUD
│   ├── classifier.py    ← 故障分类引擎
│   ├── reporter.py       ← 日报/周报生成
│   ├── wework_api.py    ← 企微API
│   └── run_web.py        ← Flask启动
├── web/app.py            ← 驾驶舱Flask+ECharts
└── logs/
```

## 快速开始

1. 安装依赖：`pip install flask eCharts pyyaml requests schedule`
2. 复制 `config/wework.yaml.example` → `wework.yaml`，填入企微凭证
3. 启动主服务：`python scripts/main.py`
4. 启动驾驶舱：`python scripts/run_web.py`
5. 访问 `http://localhost:5000` 查看驾驶舱
