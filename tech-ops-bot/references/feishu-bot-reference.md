# 飞书机器人接入参考

## 飞书 vs 企微对比

| 能力 | 企微 | 飞书 |
|------|------|------|
| 自建应用收发消息 | ✅ | ✅ |
| 群机器人 | ✅ | ✅ |
| 语音转文字 | ✅（API自带） | ✅（API自带） |
| 回调URL | 需公网 | 需公网 |
| 个人外部群 | ❌（需认证） | ✅（支持外部群） |

**结论**：如果企微外部群无法落地，飞书是最佳替代方案。

## 飞书自建应用接入步骤

1. 打开 https://open.feishu.cn/app → 创建企业自建应用
2. 在「应用功能」→「机器人」开启机器人能力
3. 添加应用能力：「网页」（用于驾驶舱）
4. 获取 `App ID` 和 `App Secret`
5. 配置请求地址（回调URL），指向内网Flask服务
6. 发布应用

## 飞书消息收发API

### 获取tenant_access_token

```
POST https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal
Body: {"app_id": "CLI_xxx", "app_secret": "xxx"}
```

### 发送消息

```
POST https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id
Headers: Authorization: Bearer {tenant_access_token}
Body: {"receive_id": "oc_xxx", "msg_type": "text", "content": "{"text":"消息内容"}"}
```

## 内网穿透方案

回调URL需要公网可达，推荐方案：

| 方案 | 成本 | 稳定性 |
|------|------|--------|
| natapp.cn | 免费/低价 | 一般 |
| ngrok | 免费 | 一般 |
| Cloudflare Tunnel | 免费 | 较好 |
| 自购云服务器+frp | 较低 | 好 |

## 已知坑

1. 飞书机器人无法主动@外部群成员（只能回复消息）
2. 飞书消息有频率限制，大规模推送需加队列
3. 飞书API返回的错误码需要查文档：https://open.feishu.cn/document/uAjLw4CM/ukTMukTMukTM/error-code
