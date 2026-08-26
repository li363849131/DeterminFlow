# DeterminFlow 插件管理快速配置

## 一键配置命令

下次需要配置时，直接复制粘贴以下命令：

```bash
# 1. 进入DeterminFlow目录
cd /root/DeterminFlow

# 2. 停止服务
docker compose down

# 3. 配置环境变量（如果.env文件中没有）
if ! grep -q "DETERMINFLOW_PLUGIN_ADMIN_TOKEN" .env; then
    echo "" >> .env
    echo "# Plugin admin token for remote plugin management operations" >> .env
    echo "DETERMINFLOW_PLUGIN_ADMIN_TOKEN=v0QaFTx_thdzSZ2NmxuFJu25AZp_aeDGKaXk1YoOiBg" >> .env
fi

# 4. 切换到inline模式（避免executor超时）
sed -i 's/DETERMINFLOW_WORKFLOW_EXECUTOR_MODE=.*/DETERMINFLOW_WORKFLOW_EXECUTOR_MODE=inline/' .env
sed -i 's/DETERMINFLOW_WORKFLOW_EXECUTOR_COUNT=.*/DETERMINFLOW_WORKFLOW_EXECUTOR_COUNT=1/' .env

# 5. 清理锁文件
rm -f /root/DeterminFlow/data/system/workflow-executor*.lock

# 6. 启动服务
docker compose up -d

# 7. 等待服务启动
echo "等待服务启动..."
sleep 20

# 8. 验证
docker exec determinflow-app-1 printenv | grep DETERMINFLOW_PLUGIN_ADMIN_TOKEN
curl -s http://localhost:30014/healthz
```

## Web界面配置

访问 `http://localhost:30014`，在插件管理页面的"管理令牌"输入框中输入：

```
v0QaFTx_thdzSZ2NmxuFJu25AZp_aeDGKaXk1YoOiBg
```

## 完整配置说明

详见：`PLUGIN_ADMIN_TOKEN_CONFIG.md`
