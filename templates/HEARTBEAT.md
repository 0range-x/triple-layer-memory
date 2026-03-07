# HEARTBEAT.md

## Session Token 检查（每次心跳执行）

检查当前 session 的 token 使用量。

如果达到 160k tokens（80%）：
1. 总结对话历史中的关键信息
2. 将总结写入 `memory/YYYY-MM-DD.md`
3. 输出提示：`⚠️ Session 上下文已达 80%，建议发送 [NEW_SESSION] 切换新会话`

如果达到 180k tokens（90%）：
1. 强制压缩并写入记忆
2. 输出警告：`🚨 Session 上下文已达 90%，请立即发送 [NEW_SESSION] 切换新会话，否则可能影响功能`

## Session 切换检测（防止记忆断层）

**检测 [NEW_SESSION] 标记：**

如果在最近的消息中检测到 `[NEW_SESSION]` 标记：
1. **立即触发主动压缩**：
   ```python
   from scripts.session_handoff import compress_current_session
   compress_current_session()
   ```

2. **生成交接上下文**：
   ```python
   from scripts.session_handoff import generate_handoff_context, save_handoff_context
   context = generate_handoff_context(channel=current_channel)
   save_handoff_context(context)
   ```

3. **输出确认信息**：
   ```
   ✅ 旧 session 已压缩，交接上下文已生成
   新 session 启动时会自动加载最近的记忆
   ```

**为什么这样做：**
- 避免记忆断层：切换前先压缩，确保内容不丢失
- 自动交接：新 session 启动时自动加载旧 session 的摘要
- 无缝衔接：用户无需手动操作，系统自动处理

## 定时任务执行规则

**核心原则：HEARTBEAT 不直接跑长任务**

所有定时任务必须通过 sub-agent 执行：

```python
# 示例：在 HEARTBEAT 中触发定时任务
if should_run_scheduled_task():
    # 不要直接执行任务，而是 spawn sub-agent
    spawn_subagent(
        task="执行定时任务：[任务描述]",
        mode="run",
        runtime="subagent",
        cleanup="delete"
    )
```

**为什么这样做：**
- 避免 heartbeat 超时导致任务中断
- 任务失败不影响主 session
- 可以并行执行多个任务
- 便于监控和日志追踪
