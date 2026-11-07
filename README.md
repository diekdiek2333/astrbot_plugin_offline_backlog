# 离线消息补回（AstrBot 插件）

一个给 AstrBot + OneBot（aiocqhttp）用的离线消息补回插件。

机器人关机、重启、断线期间，QQ 不会把消息重新推给 OneBot。这个插件会在重新上线后读取平台聊天记录，把断点之后的消息合成一段上下文，重新注入 AstrBot 的正常对话流程，让机器人像刚看到手机一样补回。

可选功能：如果名单里的私聊在离线期间一条消息都没发，机器人会主动发一句“终于上线了”的招呼或抱怨。

---

## 功能

- 私聊离线消息补回
- 群聊离线消息补回（默认只补 @机器人 或包含唤醒词的消息）
- 按会话保存断点，避免重复补回
- 支持从最近消息向前、向后两个方向读取，覆盖 SnowLuma 关机期间没观察到的消息
- 支持“离线期间没人发消息”时主动打招呼
- 心跳记录真实离线时长，避免用“最后一条消息时间”误判
- 管理员可干跑查看将要补回的内容
- 所有补回都走 AstrBot 正常事件管线，人格、记忆、工具、分段回复仍然生效

---

## 安装

1. 把整个 `astrbot_plugin_offline_backlog` 文件夹放到：

   ```text
   AstrBot/data/plugins/astrbot_plugin_offline_backlog
   ```

2. 重启 AstrBot。
3. 打开 WebUI → 插件 → 离线消息补回，填写配置。
4. 确认 OneBot 平台（aiocqhttp）已经连接。

---

## 必填配置

### 私聊目标

```json
"private_targets": [
  "123456789",
  "987654321"
]
```

填 QQ 号，一行一个。只有这里填了的私聊才会被处理。

### 群聊目标

```json
"group_targets": [
  "123456789"
]
```

填群号。不想补群聊就留空。

### 显示名

```json
"target_labels": [
  "123456789=好友A",
  "987654321=好友B",
  "555555555=某个群"
]
```

格式是 `QQ号=名字`。名字只用于提示词里的称呼，不填也能工作。

### 群聊唤醒词

```json
"group_wake_words": [
  "机器人"
]
```

群聊默认只补 @机器人 或包含这些词的消息。留空则只认 @。

---

## 上线打招呼（可选）

如果希望“离线期间没人发消息”时机器人主动说一句，配置：

```json
"greeting_enabled": true,
"greeting_targets": [
  "123456789=好友A",
  "987654321=好友B"
],
"greeting_min_offline_minutes": 30,
"greeting_cooldown_hours": 12
```

规则：

- 只对 `greeting_targets` 里的**私聊**生效
- 这个人必须同时出现在 `private_targets` 里
- 离线至少 `greeting_min_offline_minutes` 分钟才触发
- 同一个人两次招呼至少间隔 `greeting_cooldown_hours` 小时
- 如果对方离线期间发过消息，走正常补回，不会额外打招呼
- 打招呼时禁用所有工具，只让模型说话，避免突然发说说、逛空间

`greeting_targets` 留空表示不开启。

---

## 提示词

### 补回提示词

```json
"reply_prompt": "[离线消息补回 · 系统任务]\n...\n{transcript}"
```

`{transcript}` 会替换成聊天记录。

### 打招呼提示词

```json
"greeting_prompt": "[上线打招呼 · 系统任务]\n...\n{label}"
```

`{label}` 会替换成对方名字。可以按自己的机器人人格改写。

---

## 管理员命令

- `/backlog`：干跑，只显示会补回多少条，不发送
- `/backlog send`：立即补回
- `/离线补回`：`/backlog` 的别名

命令需要 AstrBot 管理员权限。

---

## 工作原理

1. 开机后等待平台连接。
2. 读取每个目标的最近聊天记录。
3. 用上次保存的断点时间和消息锚点过滤。
4. 只取断点之后、最近 `max_backlog_age_hours` 小时内的消息。
5. 把聊天记录合成一个提示词，构造一条合成事件，提交给 AstrBot 正常事件总线。
6. 人格、记忆、工具、分段回复都照常生效。
7. 更新断点和锚点，避免下次重复补回。

上线招呼使用单独的合成事件，并带有 `offline_backlog_greeting` 标记，插件会在这个事件上清空工具列表。

---

## 注意

- 需要 `aiocqhttp` 平台；其他平台没有 OneBot 的 `get_friend_msg_history` / `get_group_msg_history` 接口。
- 群聊默认只补 @机器人 或唤醒词，避免把群友所有闲聊都补一遍。
- `first_run_lookback_minutes` 默认 0。第一次运行没有断点时不会回看；需要首次回看就把它改成 30 / 60。
- `max_backlog_age_hours` 默认 24，关机超过一天的旧消息不会补。
- 心跳每 60 秒写一次，所以强杀进程时离线时长最多误差约 1 分钟。
- 补回内容会进入正常对话历史，可能影响后续上下文长度；`max_messages_per_target` 和 `max_prompt_chars` 可以控制大小。
- 如果平台历史接口被限制、Cookie 失效或 SnowLuma 没连接，补回会失败并在日志里提示。
