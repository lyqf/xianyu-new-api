# New-API + MC-Proxy-Server 使用指南

## 系统架构

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   客户端     │─────▶│   New-API   │─────▶│ MC-Proxy-   │
│  (Claude/    │      │   (网关)    │      │   Server    │
│  Cherry等)   │      │  localhost  │      │  localhost  │
│              │      │   :3000     │      │   :3999     │
└─────────────┘      └─────────────┘      └─────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │   MC CLI    │
                     │   后端      │
                     └─────────────┘
```

---

## 快速启动

### 1. 启动 MC-Proxy-Server

```bash
cd /Users/maceo/my_github_wp/mc-decode/mc-proxy-server

# 检查是否已在运行
curl -s http://localhost:3999/health

# 如未运行，执行启动
nohup npm start > logs/app.log 2>&1 &
echo $! > .mc-proxy-server.pid
```

### 2. 启动 New-API

```bash
cd /Users/maceo/origin_github_wp/new-api

# 检查是否已在运行
curl -s http://localhost:3000/api/status

# 如未运行，执行启动
nohup ./new-api --log-dir ./logs > ./logs/app.log 2>&1 &
echo $! > .new-api.pid
```

### 3. 验证服务状态

```bash
# 验证 MC-Proxy-Server
curl -s http://localhost:3999/health

# 验证 New-API
curl -s http://localhost:3000/api/status

# 验证模型列表
curl -s http://localhost:3000/v1/models \
  -H "Authorization: Bearer sk-newapi"
```

---

## 管理员使用指南

### 登录管理界面

1. 访问 http://localhost:3000
2. 使用账号登录：
   - 用户名：`admin`
   - 密码：`admin123456`

### 查看渠道（供应商）

```bash
# 通过数据库查询
sqlite3 one-api.db "SELECT id, name, type, base_url, status FROM channels;"
```

### 添加新渠道

```bash
# 示例：添加另一个 OpenAI 兼容渠道
sqlite3 one-api.db "
INSERT INTO channels (type, key, name, status, created_time, base_url, models, 'group')
VALUES (1, 'your-api-key', 'My Channel', 1, $(date +%s), 'https://api.example.com', 'gpt-4,gpt-3.5-turbo', 'default');
"

# 添加对应的 abilities
sqlite3 one-api.db "
INSERT INTO abilities ('group', model, channel_id, enabled, priority, weight)
VALUES ('default', 'gpt-4', LAST_INSERT_ROWID(), 1, 0, 100);
"
```

### 创建新用户

```bash
# 通过 API 注册
curl -X POST http://localhost:3000/api/user/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "newuser",
    "password": "userpassword",
    "display_name": "New User"
  }'
```

### 创建 API Token

```bash
# 直接插入数据库（简化方式）
sqlite3 one-api.db "
INSERT INTO tokens (user_id, key, name, created_time, status, unlimited_quota, 'group')
VALUES (2, 'usertoken123', 'User Token', $(date +%s), 1, 1, 'default');
"
```

### 查看系统状态

```bash
# 查看日志
tail -f logs/app.log

# 查看数据库中的关键统计
sqlite3 one-api.db "
SELECT 'Users' as metric, COUNT(*) as count FROM users
UNION ALL
SELECT 'Channels', COUNT(*) FROM channels
UNION ALL
SELECT 'Tokens', COUNT(*) FROM tokens
UNION ALL
SELECT 'Abilities', COUNT(*) FROM abilities;
"
```

### 重启服务

```bash
# 停止服务
kill $(cat .new-api.pid)

# 重新启动
nohup ./new-api --log-dir ./logs > ./logs/app.log 2>&1 &
echo $! > .new-api.pid
```

---

## 用户使用指南

### 获取 API 信息

| 项目 | 值 |
|------|-----|
| API Base URL | `http://localhost:3000/v1` |
| API Key | `sk-newapi` |
| 支持的模型 | claude-sonnet-4.5, deepseek-chat, glm-4.6 等 16 个模型 |

### 在客户端中配置

#### 1. Claude Code / Claude Desktop

```bash
# 设置环境变量
export ANTHROPIC_API_KEY="sk-newapi"
export ANTHROPIC_BASE_URL="http://localhost:3000/v1"

# 运行 Claude Code
claude
```

#### 2. Cherry Studio

1. 打开设置 → 模型服务
2. 选择「OpenAI 兼容」类型
3. 填写配置：
   - API 地址：`http://localhost:3000/v1`
   - API 密钥：`sk-newapi`
   - 模型：从下拉列表选择（如 `deepseek-chat`）

#### 3. Lobe Chat

```
设置 → OpenAI → API代理地址: http://localhost:3000/v1
设置 → OpenAI → API Key: sk-newapi
```

#### 4. Python OpenAI SDK

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:3000/v1",
    api_key="sk-newapi"
)

response = client.chat.completions.create(
    model="deepseek-chat",
    messages=[{"role": "user", "content": "Hello!"}]
)
print(response.choices[0].message.content)
```

#### 5. curl 命令行

```bash
# 非流式请求
curl http://localhost:3000/v1/chat/completions \
  -H "Authorization: Bearer sk-newapi" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "messages": [{"role": "user", "content": "Hello"}],
    "stream": false
  }'

# 流式请求（SSE）
curl http://localhost:3000/v1/chat/completions \
  -H "Authorization: Bearer sk-newapi" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "messages": [{"role": "user", "content": "Hello"}],
    "stream": true
  }'
```

### 查看可用模型

```bash
curl http://localhost:3000/v1/models \
  -H "Authorization: Bearer sk-newapi"
```

---

## 故障排除

### 问题："无效的令牌"

**原因**：Token 格式不正确

**解决**：
- 确保使用 `sk-newapi`（不带额外 `-` 后缀）
- 检查 token 是否存在于数据库：`sqlite3 one-api.db "SELECT * FROM tokens;"`

### 问题："用户额度不足"

**原因**：用户 quota 为 0

**解决**：
```bash
sqlite3 one-api.db "UPDATE users SET quota = 100000000 WHERE id = 1;"
```

### 问题："model_not_found"

**原因**：模型未在 abilities 表中启用

**解决**：
```bash
sqlite3 one-api.db "INSERT INTO abilities ('group', model, channel_id, enabled) VALUES ('default', 'your-model', 1, 1);"
```

### 问题：服务无法启动

**检查步骤**：
1. 检查端口占用：`lsof -i :3000` 或 `lsof -i :3999`
2. 检查日志：`tail -n 50 logs/app.log`
3. 检查数据库权限：`ls -la one-api.db`

### 问题：MC-Proxy-Server 返回错误

**检查步骤**：
1. 直接测试 mc-proxy-server：
   ```bash
   curl http://localhost:3999/health
   ```
2. 检查 mc-proxy-server 日志：
   ```bash
   tail -f /Users/maceo/my_github_wp/mc-decode/mc-proxy-server/logs/app.log
   ```

---

## 数据库常用操作

### 备份数据库

```bash
cp one-api.db one-api.db.backup.$(date +%Y%m%d)
```

### 重置用户密码

```bash
# 密码需要 MD5 加密（简化示例，实际需计算 MD5）
sqlite3 one-api.db "UPDATE users SET password = 'NEW_MD5_HASH' WHERE username = 'admin';"
```

### 查看使用统计

```bash
# 查看各用户使用额度
sqlite3 one-api.db "
SELECT username, quota, used_quota,
       ROUND(used_quota * 100.0 / quota, 2) as usage_percent
FROM users WHERE quota > 0;
"
```

### 清理日志

```bash
# 清空日志文件（保留文件）
> logs/app.log
```

---

## 安全注意事项

1. **生产环境**：
   - 修改默认密码
   - 使用 HTTPS
   - 配置防火墙限制访问
   - 定期备份数据库

2. **Token 管理**：
   - 定期轮换 API Token
   - 为不同用户分配不同 Token
   - 设置合理的额度限制

3. **网络隔离**：
   - MC-Proxy-Server 只需本地访问（localhost:3999）
   - New-API 可根据需要开放外部访问

---

## 附录：服务端口汇总

| 服务 | 端口 | 用途 |
|------|------|------|
| new-api | 3000 | API 网关和管理界面 |
| mc-proxy-server | 3999 | MC CLI 代理服务 |

## 附录：关键文件位置

| 文件 | 路径 |
|------|------|
| New-API 数据库 | `/Users/maceo/origin_github_wp/new-api/one-api.db` |
| New-API 日志 | `/Users/maceo/origin_github_wp/new-api/logs/app.log` |
| MC-Proxy 日志 | `/Users/maceo/my_github_wp/mc-decode/mc-proxy-server/logs/app.log` |
| MC-Proxy 配置 | `/Users/maceo/my_github_wp/mc-decode/mc-proxy-server/.env` |
