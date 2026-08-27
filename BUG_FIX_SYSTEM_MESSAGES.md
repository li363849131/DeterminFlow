# Bug 修复：Anthropic API "multiple non-consecutive system messages" 错误

## 问题描述

在使用 DeterminFlow 调用 Claude 模型时，出现以下错误：

```
anthropic.BadRequestError: Error code: 400 - {'type': 'error', 'error': {'type': 'invalid_request_error', 'message': 'messages: Received multiple non-consecutive system messages; system messages must be consecutive and must appear before all user and assistant messages'}}
```

**症状：**
- 模型调用失败，提示"模型调用失败，请稍后再试"
- 上下文限制显示错误（128K 而不是 1M）

## 根本原因

1. **多个 System 消息未合并**：在消息压缩和截断过程中，代码会保留多个 `SystemMessage`，但 Anthropic API 要求所有 system 消息必须合并为一个，并且必须在最开始。

2. **上下文限制配置错误**：`config/models_config.json` 中 `maxContextTokens` 被设置为 128000（128K），但 Claude Sonnet 5 实际支持 1M 上下文。

## 修复内容

### 1. 添加 `_merge_system_messages()` 函数

在 `src/core/utils.py` 中添加了新函数，用于合并多个 SystemMessage：

```python
def _merge_system_messages(messages: list[BaseMessage]) -> list[BaseMessage]:
    """
    合并多个 SystemMessage 为一个，避免 Anthropic API 报错。

    Anthropic API 要求：
    - 所有 system 消息必须合并为一个
    - system 消息必须在最开始

    Args:
        messages: 消息列表

    Returns:
        合并后的消息列表
    """
    if not messages:
        return messages

    system_msgs = []
    other_msgs = []

    for msg in messages:
        if isinstance(msg, SystemMessage):
            system_msgs.append(msg)
        else:
            other_msgs.append(msg)

    # 如果没有或只有一个 system 消息，直接返回
    if len(system_msgs) <= 1:
        return messages

    # 合并所有 system 消息的内容
    merged_content = "\n\n".join(
        str(msg.content) if msg.content else ""
        for msg in system_msgs
    ).strip()

    if not merged_content:
        # 如果所有 system 消息都是空的，返回非 system 消息
        return other_msgs

    # 创建一个新的合并后的 SystemMessage
    merged_system = SystemMessage(content=merged_content)

    # 返回：合并的 system 消息 + 其他消息
    return [merged_system] + other_msgs
```

### 2. 在三个关键函数中调用合并逻辑

修改了以下三个函数，在返回消息列表前调用 `_merge_system_messages()`：

- `_sanitize_tool_pairs()` - 第107-111行
- `_sanitize_tool_pairs_strict()` - 第183-187行  
- `truncate_messages()` - 第288-292行

### 3. 修复上下文限制配置

修改 `config/models_config.json`：

**修改前：**
```json
"maxContextTokens": 128000
```

**修改后：**
```json
"maxContextTokens": 1000000
```

应用到两个 provider（sui-xiang 和 anthropic）。

## 部署步骤

1. **修改代码**：
   ```bash
   # 已修改 /root/DeterminFlow/src/core/utils.py
   ```

2. **修改配置**：
   ```bash
   # 已修改 /root/DeterminFlow/config/models_config.json
   ```

3. **复制到容器并重启**：
   ```bash
   docker cp /root/DeterminFlow/src/core/utils.py determinflow-app-1:/app/src/core/utils.py
   docker restart determinflow-app-1
   ```

4. **验证**：
   ```bash
   docker ps --filter name=determinflow-app-1
   # 应该显示 (healthy)
   ```

## 验证方法

1. **检查服务状态**：
   ```bash
   docker logs determinflow-app-1 --tail 20 | grep "startup complete"
   ```
   应该看到：`Application startup complete.`

2. **检查上下文限制**：
   在 DeterminFlow Web 界面中，现在应该显示"上限 1.0M"而不是"128.0K"

3. **测试模型调用**：
   尝试发送一个请求，应该不再出现"multiple non-consecutive system messages"错误

## 修复时间

- 2026-08-27
- 修复文件：`src/core/utils.py`, `config/models_config.json`
- Docker 容器已重启并应用修复

## 技术细节

**为什么会有多个 System 消息？**

在消息压缩和历史截断时，代码会：
1. 提取所有 SystemMessage
2. 保留 checkpoint（summary）
3. 保留最近的对话

然后用 `system_msgs + checkpoint + kept` 重组，这可能导致多个 SystemMessage 并存。

**Anthropic API 的要求：**
- 只能有一个 system 消息
- system 消息必须在最开始
- system 消息必须连续，不能被其他类型消息分隔

修复方案是在所有消息处理函数的最后，统一调用 `_merge_system_messages()` 合并多个 system 消息为一个。
