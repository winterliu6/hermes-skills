---
name: zabbix-ops
description: Zabbix运维自动化 — 多门店IT基础设施监控、API自动化、安全审计、日常运维报表、跳板机批量操作。
version: 1.0
category: devops
tags: [Zabbix, 监控, 自动化, IT运维, 安全审计, 跳板机, API]
license: MIT
homepage: https://github.com/winterliu6/hermes-skills
issues: https://github.com/winterliu6/hermes-skills/issues
---

# Zabbix 运维自动化

## 概述

本技能涵盖多门店IT基础设施（260+主机）的 Zabbix 监控运维全流程：API自动化、安全审计、日常报表、告警路由、跳板机批量操作脚本部署。

适用于：影城/零售/连锁等有多门店、多系统IT基础设施的企业。

## 环境要求

- Zabbix Server 5.x 及以上
- Admin级别账号（API + Web UI）
- 跳板机：Zabbix Server可SSH到达各被监控主机
- `sshpass`（Linux跳板机用）
- VPN或专线（跳板机到各门店网络）

## 核心工作流

### 1. 批量主机发现与指纹识别

通过Zabbix API获取所有主机，结合TCP指纹识别设备类型：

```python
import socket, json

def get_banner(ip, port=22, timeout=3):
    try:
        s = socket.socket()
        s.settimeout(timeout)
        s.connect((ip, port))
        banner = s.recv(1024).decode(errors="ignore")
        s.close()
        return banner
    except:
        return ""

# 指纹库
fingerprints = {
    "FortiGate": ["FortiGate", "FortiOS"],
    "TMS": ["TMS", "TheatreManagement"],
    "Linux": ["SSH", "linux"],
    "Windows": ["RDP", "smb"],
    "HP_Switch": ["HP ", "ProCurve"],
    "Cisco": ["Cisco", "IOS"],
}
```

### 2. 告警自动路由

根据告警关键词自动路由到不同处理人：

```python
ALERT_ROUTES = {
    "放映.*": ["tech-cinema@company.com"],
    "服务器.*": ["tech-server@company.com"],
    "网络.*": ["tech-network@company.com"],
    "安全.*": ["security@company.com"],
}

def route_alert(alert_msg):
    for pattern, recipients in ALERT_ROUTES.items():
        if re.search(pattern, alert_msg):
            return recipients
    return ["tech-general@company.com"]
```

### 3. 跳板机批量执行

```bash
# 测试连通性
export SSHPASS='password'
sshpass -e ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 user@host 'hostname'

# 批量推送脚本到所有主机
for ip in $(cat hosts.txt); do
    sshpass -e scp -o StrictHostKeyChecking=no script.sh root@$ip:/tmp/
    sshpass -e ssh -o StrictHostKeyChecking=no root@$ip "bash /tmp/script.sh"
done
```

### 4. 每日运维报表（API自动生成）

```python
import requests, json
from datetime import datetime, timedelta

ZBX_URL = "https://zabbix.company.com/api_jsonrpc.php"
ZBX_USER = "admin"
ZBX_PASS = "password"

def zbx_api(method, params):
    r = requests.post(ZBX_URL, json={
        "jsonrpc": "2.0",
        "method": method,
        "params": params,
        "id": 1,
        "auth": zbx_login()
    }, verify=False)
    return r.json()["result"]

def zbx_login():
    return zbx_api("user.login", {"user": ZBX_USER, "password": ZBX_PASS})["result"]

# 获取昨日告警
def get_yesterday_alerts():
    yesterday = (datetime.now() - timedelta(days=1)).strftime("%Y-%m-%d")
    return zbx_api("alert.get", {
        "time_from": f"{yesterday} 00:00:00",
        "time_till": f"{yesterday} 23:59:59"
    })
```

### 5. 安全审计

- 检查Zabbix用户列表，清理离职员工账号
- 检查guest账号是否启用（安全风险）
- 检查API token有效期
- 扫描弱口令主机
- 检查暴露在公网的Zabbix Server

```python
# 检查暴露公网
def check_zbx_public_exposure():
    r = requests.get("https://zabbix.company.com/zabbix.php?action=dashboard.view")
    return "cloudflare" not in r.text.lower()
```

## 常见设备默认密码

| 设备类型 | 用户名 | 默认密码 |
|---------|--------|---------|
| FortiGate | admin | （首次登录需设置）|
| TMS | tms | （安装时设置）|
| Linux | root | （安装时设置）|
| HP Switch | admin | （安装时设置）|
| VMware ESXi | root | （安装时设置）|

## 已知坑

1. **sshpass的`^`问题**：密码首字符是`^`时，`-p`参数会漏掉。用 `SSHPASS` 环境变量代替。
2. **system.run[]被禁用**：Zabbix agent的`system.run[]`默认被禁用，需在`zabbix_agentd.conf`中启用，且Zabbix Server需开启active checks。
3. **Zabbix API每分钟限速**：大量请求时加`sleep(0.1)`缓冲。
4. **Windows主机WMI问题**：防火墙或权限问题导致WMI监控失败时，可用`zabbix_get`手动测试。

## 参考资料

- Zabbix API文档: https://www.zabbix.com/documentation/current/manual/api
- pyzabbix库: `pip install pyzabbix`
