# RSS情报源列表

## 安全媒体源

### 中文源
- 嘶吼安全：`https://www.4hou.com/feed` — JS动态渲染，RSS是唯一入口；需过滤无关内容
- FreeBuf：`https://www.freebuf.com/feed` — 内容偏技术
- 安全客：`https://www.anquanke.com/feed`

### 英文源
- DarkReading：`https://www.darkreading.com/rss.xml` — 综合威胁情报，RSS稳定
- The Hacker News：`https://feeds.feedburner.com/TheHackersNews` — 每日热点
- BleepingComputer：`https://www.bleepingcomputer.com/feed/` — 恶意软件情报
- Krebs on Security：`https://krebsonsecurity.com/feed/` — 深度分析
- CISA Alerts：`https://www.cisa.gov/uscert/ncas/alerts.xml` — 官方漏洞预警

## 失效源记录

| 源 | 失效时间 | 原因 | 替代 |
|----|---------|------|------|
| Seebug RSS | 2025-Q3 | 停更 | 嘶吼/安全客 |
| NTI威胁情报中心 | 2025-Q4 | 域名解析失败 | DarkReading |

## 双源合并策略

1. 嘶吼（中文）负责国内威胁情报（勒索、钓鱼、国内漏洞）
2. DarkReading（英文）负责国际前沿（0day、供应链攻击、勒索家族追踪）
3. 合并时去重（标题相似度>80%视为同一事件）
4. 优先使用中文源描述（贴近国内读者），英文源作为补充背景

## 关键词过滤

### 必选关键词（情报相关）
勒索、钓鱼、漏洞、泄露、入侵、攻击、恶意、病毒、木马、勒索软件、0day、供应链

### 排除关键词（噪声过滤）
- OpenClaw、龙虾（AI工具滥用内容，占比过高）
- 会议、活动、招聘（与威胁情报无关）
- 无线通信、汽车、物联网（与本邮件主题无关的细分领域）
