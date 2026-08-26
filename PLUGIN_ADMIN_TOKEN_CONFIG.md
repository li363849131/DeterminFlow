# DeterminFlow 插件管理令牌配置指南

## 问题描述
添加插件时报错：
```
API Error 403: {"detail":"远程 Plugin 管理默认关闭；请配置 DETERMINFLOW_PLUGIN_ADMIN_TOKEN 并使用 Bearer token"}
```
或
```
API Error 401: {"detail":"Plugin 管理令牌无效"}
```

## 解决方案

### 1. 配置环境变量

在 `/root/DeterminFlow/.env` 文件中添加：

```bash
# Plugin admin token for remote plugin management operations
DETERMINFLOW_PLUGIN_ADMIN_TOKEN=v0QaFTx_thdzSZ2NmxuFJu25AZp_aeDGKaXk1YoOiBg
```

### 2. 更新 docker-compose.yml

在 `docker-compose.yml` 的 `environment` 部分添加：

```yaml
environment:
  # ... 其他环境变量 ...
  DETERMINFLOW_PLUGIN_ADMIN_TOKEN: "${DETERMINFLOW_PLUGIN_ADMIN_TOKEN:-}"
```

### 3. 修复 Workflow Executor 启动问题

如果遇到 `TimeoutError: Workflow Executor endpoint was not published` 错误：

**方法1：切换到 inline 模式（推荐）**

在 `.env` 文件中修改：
```bash
DETERMINFLOW_WORKFLOW_EXECUTOR_MODE=inline
DETERMINFLOW_WORKFLOW_EXECUTOR_COUNT=1
```

**方法2：清理锁文件**

```bash
# 停止容器
docker compose down

# 删除锁文件
rm -f /root/DeterminFlow/data/system/workflow-executor*.lock

# 重启
docker compose up -d
```

### 4. 重启服务

```bash
cd /root/DeterminFlow

# 停止服务
docker compose down

# 启动服务
docker compose up -d

# 查看日志
docker logs determinflow-app-1 -f
```

### 5. 在Web界面使用令牌

1. 访问 `http://localhost:30014`
2. 使用密码 `gjp` 登录
3. 进入插件管理页面
4. 找到"管理令牌"输入框（带钥匙图标）
5. 输入令牌：`v0QaFTx_thdzSZ2NmxuFJu25AZp_aeDGKaXk1YoOiBg`
6. 现在可以添加插件源和安装插件了

## 添加插件源示例

**官方插件源：**
- 名称：`official`
- Git URL：`https://github.com/alikon-art/DeterminFlow-Plugins.git`
- 分支/标签：`main`

## 验证配置

```bash
# 检查容器环境变量
docker exec determinflow-app-1 printenv | grep PLUGIN

# 应该输出：
# DETERMINFLOW_PLUGIN_ADMIN_TOKEN=v0QaFTx_thdzSZ2NmxuFJu25AZp_aeDGKaXk1YoOiBg

# 检查服务健康状态
curl http://localhost:30014/healthz

# 应该返回：
# {"status":"ok"}
```

## 注意事项

1. **令牌安全**：这个令牌用于插件管理授权，请妥善保管
2. **令牌存储**：前端输入的令牌只保存在页面内存中，刷新后需要重新输入
3. **本地访问**：如果服务端未配置token且从localhost访问，可以留空
4. **远程访问**：从远程访问必须配置并使用token

## 生成新令牌

如需生成新的管理令牌：

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

将生成的令牌更新到 `.env` 文件和 Web 界面即可。

## 故障排查

### 容器不断重启
- 检查是否有锁文件冲突：`find /root/DeterminFlow/data -name "*.lock"`
- 删除锁文件后重启

### 端口访问问题
- 确认容器端口映射：`docker ps | grep determinflow`
- 应该看到：`0.0.0.0:30014->8020/tcp`

### Token验证失败
- 确认环境变量已加载：`docker exec determinflow-app-1 printenv | grep PLUGIN`
- 确认Web界面输入的token与配置文件一致
- 注意token中不要有多余的空格或换行符

---

**配置完成！** 现在可以正常使用插件管理功能了。
