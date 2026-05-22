---
name: security-newsletter
description: 企业网络安全意识宣贯邮件自动化流水线。每周搜集安全情报，生成并发送安全意识宣贯邮件。
version: 1.1
category: productivity
tags: [安全, 网络安全, 邮件自动化, 企业安全, RSS, 威胁情报]
license: MIT
homepage: https://github.com/winterliu6/hermes-skills
issues: https://github.com/winterliu6/hermes-skills/issues
---

# 企业网络安全意识宣贯邮件

## 任务描述
每周搜集网络安全威胁情报，生成并发送企业网络安全意识宣贯邮件。

## 执行流程

### 1. 信息搜集

**安全媒体RSS源（双源策略）：**
- 嘶吼安全：`https://www.4hou.com/feed` — JS动态渲染，RSS是唯一入口；需过滤无关内容
- DarkReading：`https://www.darkreading.com/rss.xml` — 英文，每日更新，本周最新威胁；SSL需加 `context=ssl.create_default_context()`

**关键词（可根据企业行业定制）：**
- 勒索病毒、网络钓鱼、数据泄露
- 供应链攻击、0day漏洞
- 社工攻击、内部威胁

### 2. 邮件格式模板

```
Subject: [公司名称] · 网络安全意识宣贯

正文格式：
1. 开头：致公司全体同事 / 各部门负责人、全体同事
2. 威胁预警（外部情报，贴近本行业场景）：
   - ① [威胁名称]
     描述：<行业场景影响>
     防控措施：<具体可操作>
3. 安全排查与处置：
   - 排查要点（发现哪些情况立即断网上报）
   - 处置原则（立即断网、不支付赎金、保留现场）
4. 日常安全规范（5-6条）
5. 结尾：各部门负责人切实履行职责……IT部收尾
6. 落款：[公司名称] · IT部门 + 日期
```

### 3. 邮件内容要求

- **绝对禁止**：照搬之前发过的邮件内容，要搜最新情报
- **贴近本行业**：每个威胁描述都要落到本行业场景，不能泛泛而谈
- **防控建议**：必须具体可操作
- **处置简洁**：列出3-4条核心处置要点，不要大篇幅

### 4. 发送前检查清单

- [ ] 收件人列表正确
- [ ] 邮件标题符合格式
- [ ] 内容为本周最新情报，与上期无重复
- [ ] 日期自动更新
- [ ] 防控建议具体可操作

### 5. 技术实现

#### 邮件发送（Python示例）

```python
import smtplib, ssl
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

def send_newsletter(smtp_host, smtp_port, account, password,
                    to_addrs, subject, html_body):
    ctx = ssl.create_default_context()
    with smtplib.SMTP_SSL(smtp_host, smtp_port, context=ctx) as srv:
        srv.login(account, password)
        msg = MIMEMultipart("alternative")
        msg["Subject"] = subject
        msg["From"] = account
        msg["To"] = ", ".join(to_addrs)
        msg.attach(MIMEText(html_body, "html", "utf-8"))
        srv.sendmail(account, to_addrs, msg.as_string())
```

#### RSS采集（Python示例）

```python
import feedparser, ssl

def fetch_rss(url, keywords, max_items=10):
    ctx = ssl.create_default_context()
    feed = feedparser.parse(url, request_context=ctx)
    results = []
    for entry in feed.entries[:max_items]:
        if any(kw in entry.title or kw in entry.summary for kw in keywords):
            results.append({
                "title": entry.title,
                "link": entry.link,
                "summary": entry.summary[:200]
            })
    return results
```

#### 完整Cron任务示例

```python
import feedparser, ssl, smtplib, ssl
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from datetime import datetime

RSS_SOURCES = [
    {"url": "https://www.4hou.com/feed", "keywords": ["勒索", "钓鱼", "漏洞"]},
    {"url": "https://www.darkreading.com/rss.xml", "keywords": ["ransomware", "phishing", "breach"]},
]

SMTP_CONFIG = {
    "host": "smtp.example.com",
    "port": 465,
    "account": "it@example.com",
    "password": "YOUR_PASSWORD",  # 从环境变量或密钥文件读取
}

RECIPIENTS = ["colleague1@company.com", "colleague2@company.com"]

def main():
    items = []
    for src in RSS_SOURCES:
        ctx = ssl.create_default_context()
        feed = feedparser.parse(src["url"], request_context=ctx)
        for e in feed.entries[:5]:
            if any(kw in e.title or kw in (e.summary or "") for kw in src["keywords"]):
                items.append(e)
    # 生成邮件HTML并发送...
    html = generate_email_html(items)
    send_newsletter(SMTP_CONFIG, RECIPIENTS, "网络安全意识宣贯", html)
    print(f"发送成功，共{len(items)}条情报")

if __name__ == "__main__":
    main()
```

### 6. 定时任务

建议执行时间：每周二上午10点（避开周一上班高峰期）

Cron表达式：`0 10 * * 2`

### 7. 行业场景库示例（可根据实际业务扩展）

**通用企业场景：**
- 邮件服务器感染勒索病毒，邮件数据被锁
- OA系统被渗透，导致内部文件外泄
- 员工邮箱收到伪装领导的钓鱼邮件（转账诈骗）
- VPN被破译，攻击者横向渗透内网

**零售/电商：**
- 客服系统被钓鱼，导致会员信息外泄
- POS系统感染，支付数据被盗

**制造/工厂：**
- 工控系统被勒索，生产线停摆
- 第三方供应商被攻击，图纸外泄
