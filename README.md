# yice-feishu-notice

用企业 bot「伊策 - 小助理」给飞书个人或群发消息。本仓库是一个 Agent Skill：装进 Claude Code / Codex 后，对 AI 说「给张三发飞书：数仓已完成」，它会先预览再发送。

## 开通

向**仓库管理员**申请 token：说明你是谁、用在哪个项目，不需要提供 IP。你会收到一段安装口令：

```text
请安装飞书通知 skill：按 https://github.com/DaxDing/yice-feishu-notice 的 README「AI 安装步骤」操作，我的 token 是 <token>
```

把口令原样发给你的 AI，它会自己装好、验证。以后更新 skill：对 AI 说「按 README 重新执行 AI 安装步骤第 1 步」。

## AI 安装步骤

写给 AI：按顺序执行，任何一步失败就停下，告诉用户原因。

1. 克隆本仓库，把 `yice-feishu-notice/` 复制到 skills 目录（Claude Code：`~/.claude/skills/`；Codex：`~/.codex/skills/`），旧版本整个替换。
2. 新建 `~/.config/yice-feishu-notice/env`（权限 600）：

   ```bash
   export NOTIFICATION_API_URL='http://218.244.157.197:8772'
   export NOTIFICATION_API_TOKEN='<用户给的 token>'
   ```

   在 shell 启动文件（zsh：`~/.zshrc`；bash 和 Windows Git Bash：`~/.bashrc`）末尾加一行，已有就不加：

   ```bash
   [ -f ~/.config/yice-feishu-notice/env ] && . ~/.config/yice-feishu-notice/env
   ```

3. 验证：

   ```bash
   . ~/.config/yice-feishu-notice/env && curl -sS -m 15 "$NOTIFICATION_API_URL/v1/whoami" -H "Authorization: Bearer $NOTIFICATION_API_TOKEN"
   ```

   - `"ok":true`：已接通，告诉用户返回的 `caller`。
   - 超时：网络或代理问题，检查后重试，仍不通请用户联系仓库管理员。
   - 401：token 不对，请用户核对口令。
4. token 不打印到对话，不提交进任何仓库。

## 发消息

先预览（不发送），再用同一请求体发送：

```bash
curl -sS -X POST "$NOTIFICATION_API_URL/v1/notifications/preview" -H "Authorization: Bearer $NOTIFICATION_API_TOKEN" -H 'Content-Type: application/json' --data @m.json
curl -sS -X POST "$NOTIFICATION_API_URL/v1/notifications" -H "Authorization: Bearer $NOTIFICATION_API_TOKEN" -H 'Content-Type: application/json' --data @m.json
```

```json
{"template": "ops.text.v1", "target": {"name": "张三"}, "data": {"body": "数仓已完成"}, "idempotency_key": "myapp-warehouse-20261021"}
```

- **target**：`{"name": "通讯录姓名"}`（完全一致；同名用 open_id）｜`{"open_id": "ou_…"}`｜`{"chat_id": "oc_…"}`（先把 bot 拉进群）｜`{"ref": "登记名"}`（`GET /v1/catalog` 查）。
- **template**：`ops.text.v1` 纯文本 `{"body"}`｜`demo.markdown.v1` Markdown `{"title","body"}`｜`delivery.rich.v1` 富文本 `{"title","content":[[行元素]]}`，行元素只有 `{"tag":"text","text"}` 和 `{"tag":"a","text","href"}`。
- **idempotency_key**：8–192 位字母数字和 `. _ : -`。同键只发一次：超时或不确定就同键重发；要发新消息就换键。

## 返回码

- **200 `sent`**：飞书已接收。
- **202 `unknown` / `sending`**：未确认，同键重发。
- **502 `failed`**：没送达，看 `error_code`，修正后换键重发。
- **400 / 404 / 422**：请求写错（如查无此人）。
- **409**：同名多人，或幂等键已用在别的内容上。
- **429**：太快（每分钟 60 条），稍后同键重发。
- **401 / 403**：token 不对 / 权限不够，找仓库管理员。

完整接口见 [references/api.md](yice-feishu-notice/references/api.md)。

## 规则

- **token 是个人凭证**：不转发、不进代码仓库和群聊；怀疑泄露找仓库管理员换。
- **不放机密**：接口走明文 HTTP，消息里不放密码、密钥。
- **别刷屏**：同一个人或群每秒约 4 条，批量内容合并成一条。
- **对方没收到**：消息在对方和「伊策 - 小助理」的单聊里；不要换键重发。
- **查无此人**：姓名要和通讯录完全一致；当天入职的第二天才能查到。
