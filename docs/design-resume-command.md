# /resume 命令设计方案

## 1. 需求背景

当前 Agentic 模式下，会话恢复是**隐式自动**的（基于 user_id + project_path 自动匹配最近的未过期会话）。用户无法：
- 查看自己的历史会话列表
- 手动选择恢复某个特定会话
- 跨项目目录恢复会话
- 在手机上随时继续之前的工作对话

Classic 模式虽有 `/continue`，但只能恢复最近一个，且必须在同一 project_path 下。

## 2. 核心用户场景

**场景 A：手机上继续工作**
> 在电脑上用 Bot 和 Claude 讨论了一个方案（项目 A），关了电脑。路上想继续聊，发 `/resume`，看到列表，选择那个会话继续。

**场景 B：切换回之前的会话**
> 中途被打断去做另一个项目。做完后发 `/resume`，选择之前的项目会话继续。

**场景 C：恢复过期会话（延伸需求）**
> 昨天的会话已过期被自动清理，但 Claude Code 侧的会话可能还在。用户希望能尝试恢复。（这个需要看 Claude Code 的 session 存活策略，先不做）

## 3. 交互设计

### 3.1 基本流程

```
用户发 /resume
    │
    ├── 有历史会话 → 展示 Inline Keyboard 列表
    │                   │
    │                   ├── 用户点选某个会话 → 设置 session_id → 回复确认
    │                   │
    │                   └── 用户点 "Cancel" → 取消操作
    │
    └── 无历史会话 → 回复提示 "No sessions to resume. Send a message to start."
```

### 3.2 会话列表展示

每个会话按钮显示关键信息（Telegram inline button text 限 64 字符）：

```
格式: [项目短名] · N msgs · 时间距离
示例: kf · 12 msgs · 2h ago
示例: rod · 3 msgs · 1d ago
```

列表按 `last_used` 降序排列，最多展示 **8 个**会话（Telegram inline keyboard 单列滑动体验合理上限）。

**布局方案：** 每行 1 个按钮（信息密度优先，手机上更易点击），最后一行放 "Cancel" 按钮。

```
┌─────────────────────────────┐
│  📂 kf · 12 msgs · 2h ago  │
├─────────────────────────────┤
│  📂 rod · 3 msgs · 1d ago  │
├─────────────────────────────┤
│  📂 kf · 8 msgs · 3d ago   │
├─────────────────────────────┤
│         ❌ Cancel           │
└─────────────────────────────┘
```

### 3.3 选中后的行为

1. 更新 `context.user_data["claude_session_id"]` 为选中的 session_id
2. 更新 `context.user_data["current_directory"]` 为该会话的 project_path
3. 编辑原消息为确认文本：`Resumed session in 📂 kf (12 msgs). Continue chatting.`
4. 用户下一条消息自动进入该会话

**不要**在 resume 时发送 prompt 给 Claude（不触发 `run_command`），因为：
- 用户可能只是切换，还没想好问什么
- 避免浪费 API 费用
- `/continue` 那种发 "Please continue where we left off" 的设计其实不好

### 3.4 过期会话的处理

- 展示列表时**包含**已过期但未被清理的会话（is_active=TRUE 但 is_expired=TRUE）
- 过期会话标记 `⏰` 图标：`⏰ kf · 8 msgs · 5d ago`
- 选中过期会话时，尝试恢复。如果 Claude Code 侧 session 已失效，会在下一次消息时自动降级为新会话（现有 facade.py 的 fallback 逻辑已处理）

### 3.5 /resume 带参数的快捷用法（可选增强）

```
/resume        → 展示列表
/resume last   → 直接恢复最近的会话（等价于 Classic 的 /continue）
/resume <id>   → 按 session_id 前缀匹配恢复
```

**建议 v1 只做 `/resume`（无参数），后续按需加参数。**

## 4. 技术方案

### 4.1 需要修改的文件

| 文件 | 修改内容 |
|------|---------|
| `src/bot/orchestrator.py` | 添加 `agentic_resume` handler + callback handler + 注册命令 |
| `src/claude/facade.py` | 添加 `get_resumable_sessions()` 方法（可选，也可直接用现有 `get_user_sessions`） |

**不需要修改的：**
- 数据库 schema（已有 sessions 表，字段够用）
- session_storage.py（`get_user_sessions` 已存在）
- session.py / sdk_integration.py（无需改动）

### 4.2 核心代码设计

#### 4.2.1 `agentic_resume` handler

```python
async def agentic_resume(
    self, update: Update, context: ContextTypes.DEFAULT_TYPE
) -> None:
    """Show resumable sessions as inline keyboard."""
    user_id = update.effective_user.id
    claude_integration = context.bot_data.get("claude_integration")

    if not claude_integration:
        await update.message.reply_text("Claude integration not available.")
        return

    # Get all user sessions (already sorted by last_used DESC)
    sessions = await claude_integration.get_user_sessions(user_id)

    # Filter: must have session_id, limit to 8
    resumable = [
        s for s in sessions
        if s.get("session_id")
    ][:8]

    if not resumable:
        await update.message.reply_text(
            "No sessions to resume. Send a message to start."
        )
        return

    # Build inline keyboard
    buttons = []
    for s in resumable:
        project_name = Path(s["project_path"]).name
        msgs = s["message_count"]
        expired = s.get("expired", False)
        time_ago = _format_time_ago(s["last_used"])
        icon = "⏰" if expired else "📂"
        label = f"{icon} {project_name} · {msgs} msgs · {time_ago}"

        buttons.append([
            InlineKeyboardButton(
                label,
                callback_data=f"resume:{s['session_id'][:32]}"
            )
        ])

    buttons.append([
        InlineKeyboardButton("❌ Cancel", callback_data="resume:cancel")
    ])

    await update.message.reply_text(
        "Select a session to resume:",
        reply_markup=InlineKeyboardMarkup(buttons),
    )
```

#### 4.2.2 Callback handler for `resume:*`

```python
async def _handle_resume_callback(
    self, update: Update, context: ContextTypes.DEFAULT_TYPE
) -> None:
    """Handle resume session selection."""
    query = update.callback_query
    await query.answer()

    data = query.data  # "resume:<session_id_prefix>" or "resume:cancel"

    if data == "resume:cancel":
        await query.edit_message_text("Cancelled.")
        return

    session_id_prefix = data.split(":", 1)[1]
    user_id = update.effective_user.id
    claude_integration = context.bot_data.get("claude_integration")

    # Find matching session by prefix
    sessions = await claude_integration.get_user_sessions(user_id)
    match = next(
        (s for s in sessions if s["session_id"].startswith(session_id_prefix)),
        None
    )

    if not match:
        await query.edit_message_text("Session not found or expired.")
        return

    # Set session context
    context.user_data["claude_session_id"] = match["session_id"]
    context.user_data["current_directory"] = Path(match["project_path"])
    context.user_data["force_new_session"] = False

    project_name = Path(match["project_path"]).name
    msgs = match["message_count"]

    await query.edit_message_text(
        f"Resumed session in 📂 {project_name} ({msgs} msgs). Continue chatting."
    )
```

#### 4.2.3 注册

在 `_register_agentic_handlers` 中添加：

```python
handlers = [
    ("start", self.agentic_start),
    ("new", self.agentic_new),
    ("resume", self.agentic_resume),      # <-- 新增
    ("status", self.agentic_status),
    ("verbose", self.agentic_verbose),
    ("repo", self.agentic_repo),
]

# Callback handler (已有 cd: pattern，新增 resume: pattern)
app.add_handler(
    CallbackQueryHandler(
        self._inject_deps(self._handle_resume_callback),
        pattern=r"^resume:",
    )
)
```

在 `get_bot_commands` 中添加：

```python
BotCommand("resume", "Resume a previous session"),
```

### 4.3 辅助函数

```python
def _format_time_ago(iso_time: str) -> str:
    """Format ISO time string to human-readable relative time."""
    dt = datetime.fromisoformat(iso_time)
    if dt.tzinfo is None:
        dt = dt.replace(tzinfo=UTC)
    delta = datetime.now(UTC) - dt

    if delta.total_seconds() < 60:
        return "just now"
    if delta.total_seconds() < 3600:
        return f"{int(delta.total_seconds() / 60)}m ago"
    if delta.total_seconds() < 86400:
        return f"{int(delta.total_seconds() / 3600)}h ago"
    return f"{delta.days}d ago"
```

### 4.4 Callback Data 长度限制

Telegram 的 `callback_data` 最大 64 字节。Claude 的 session_id 格式通常是 UUID（36 字符），加上 `resume:` 前缀（7 字符）= 43 字符，在限制范围内。

但为安全起见，截取 session_id 前 32 字符做前缀匹配（`resume:` + 32 = 39 字符）。

## 5. 需要确认的设计决策

### Q1: 是否包含过期会话？
**推荐：包含**。标记 `⏰` 图标，让用户知道可能恢复失败，但值得一试。

### Q2: 是否在 Classic 模式也加 /resume？
**推荐：先只做 Agentic 模式**。Classic 模式已有 `/continue`，且 Classic 模式用户可能更少。

### Q3: 同一项目多个会话如何区分？
**方案：** 显示第一条 prompt 的前 20 字符作为 "会话主题"。
但这需要额外查 messages 表，v1 可以先不做，用 `project_name + msg count + time_ago` 区分。

### Q4: 最大展示数量
**推荐：8 个**。考虑手机屏幕和操作便利性。

## 6. 实现步骤

1. 在 `orchestrator.py` 添加 `_format_time_ago` 辅助函数
2. 在 `orchestrator.py` 添加 `agentic_resume` handler
3. 在 `orchestrator.py` 添加 `_handle_resume_callback` handler
4. 在 `_register_agentic_handlers` 注册命令和 callback
5. 在 `get_bot_commands` 添加命令描述
6. 测试：无会话、有会话、过期会话、取消操作
7. 部署到 deploy 分支

## 7. 后续增强（v2）

- `/resume last` 快捷恢复
- 会话主题（第一条 prompt 摘要）展示
- 会话搜索（按关键词搜索历史会话）
- 会话归档和删除 UI
- 会话标签/备注功能
