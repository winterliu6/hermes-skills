---
name: 自动发布
description: 将Hermes skill脱敏后发布到GitHub。包含：读取原始skill→脱敏处理→创建本地git仓库→建GitHub仓库→推送。
version: 1.0
category: productivity
tags: [github, git, skill发布, 脱敏, 自动化]
---

# Skill 发布到 GitHub 工作流

## 触发条件
用户要求将某个skill发布到GitHub时执行本工作流。

## 前置条件
- GitHub用户名已知（如不知道，通过 `https://api.github.com/user` + token 查询）
- GitHub Personal Access Token（classic），scope：`repo`
- 若仓库不存在，自动创建；若已存在，跳过创建

## 标准流程（6步）

### Step 1：读取原始skill内容

使用 `skill_view(name)` 获取完整内容，同时读取关联的 `references/` 文件。

若skill文件不在标准路径，先用以下命令定位：
```python
import os
base = "C:/Users/Administrator/AppData/Local/hermes"
for root, dirs, files in os.walk(base):
    for f in files:
        if f.endswith(".md") or f == "SKILL.md":
            full = os.path.join(root, f)
```

### Step 2：脱敏处理（必须全部执行）

**必须替换的敏感信息：**

| 类型 | 原始示例 | 替换为 |
|------|---------|--------|
| 公司/品牌名 | 某影城、Yaolai、jccinema | 通用词/留空 |
| 真实姓名 | 张三、李四 | 通用占位符 |
| 邮箱 | it@company.com | `it@company.com` |
| 手机号 | 13xxxxxxxxx | `13800000000` |
| 内网IP | 192.168.0.5 | `192.168.1.x` |
| API Key/Token | `sk-xxxx`、`ghp_xxxx` | 完全删除或 `YOUR_KEY_HERE` |
| 数据库路径 | `E:/project/...` | 通用路径如 `E:/your-project/` |

### Step 3：创建本地git仓库

```python
import os, subprocess

local_path = "E:/hermes/workspace/projects/github/hermes-skills"
os.makedirs(local_path, exist_ok=True)

subprocess.run(["git", "init"], cwd=local_path, check=True)
subprocess.run(["git", "config", "user.name", "YOUR_GITHUB_USERNAME"], cwd=local_path, check=True)
subprocess.run(["git", "config", "user.email", "YOUR_EMAIL@gmail.com"], cwd=local_path, check=True)
```

### Step 4：创建GitHub仓库（如不存在）

```python
import requests

token = "GH_TOKEN"
username = "YOUR_GITHUB_USERNAME"
headers = {"Authorization": f"token {token}", "Accept": "application/vnd.github.v3+json"}

r = requests.post(
    "https://api.github.com/user/repos",
    headers=headers,
    json={
        "name": "hermes-skills",
        "description": "Hermes Agent Skills - AI内容创作与自动化工作流集",
        "private": False,
        "auto_init": False
    },
    timeout=30
)
```

### Step 5：写入文件 + 提交 + 推送

```python
remote_url = f"https://{username}:{token}@github.com/{username}/hermes-skills.git"
subprocess.run(["git", "remote", "add", "origin", remote_url], cwd=local_path, check=True)
subprocess.run(["git", "add", "."], cwd=local_path, check=True)

r = subprocess.run(
    ["git", "commit", "-m", "Add [skill-name] skill (sanitized)"],
    cwd=local_path, capture_output=True, text=True
)

r2 = subprocess.run(
    ["git", "push", "-u", "origin", "master"],
    cwd=local_path, capture_output=True, text=True, timeout=60
)
```

### Step 6：更新README（追加而非覆盖）

每次新增skill后，在README.md的表格中追加一行。

## 错误处理

| 错误 | 处理方式 |
|------|---------|
| token无repo权限(403) | 重新生成classic token并勾选repo |
| 仓库已存在(422) | 跳过创建，直接推送 |
| 推送失败(401/403) | 检查token是否过期 |
