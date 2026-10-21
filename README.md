# yice-feishu-notice

用一个 HTTP API，以企业 bot「伊策 - 小助理」的身份给飞书个人或群发消息。本仓库是一个 Agent Skill：装进 Claude Code / Codex 后，AI 会按 [SKILL.md](yice-feishu-notice/SKILL.md) 先预览再发送，自己处理重试和报错。不用 AI 的话，直接用 curl 调也可以。

## 开通

向维护者提供两样东西：

1. **出口 IP**：服务端看到的公网 IP，必须固定。查法：关掉代理后访问 <https://ip.sb>。家用宽带或代理的 IP 会变，变了就要重新报。和服务在同一阿里云 VPC 的机器给内网 IP。
2. **用途**：一句话说明谁在用、发什么。

维护者会回给你专属的 `NOTIFICATION_API_TOKEN`。token 只在你登记的 IP 上有效，每条消息都会记录是谁发的。默认权限是「发送 + 查自己发过的」，每分钟最多 60 条。

## 安装

```bash
git clone https://github.com/DaxDing/yice-feishu-notice.git
mkdir -p ~/.claude/skills && cp -R yice-feishu-notice/yice-feishu-notice ~/.claude/skills/
```

Codex 用户复制到 `~/.codex/skills/`。然后配置环境变量，比如写进 `~/.zshrc`（同一 VPC 的机器把地址换成内网 `http://172.23.214.40:8772`）：

```bash
export NOTIFICATION_API_URL='http://218.244.157.197:8772'
export NOTIFICATION_API_TOKEN='<你的 token>'
```

验证：下面这条命令返回 `"ok":true` 和你的调用方名，就说明接通了。

```bash
curl -sS "$NOTIFICATION_API_URL/v1/whoami" -H "Authorization: Bearer $NOTIFICATION_API_TOKEN"
```

装好后直接对 AI 说「给张三发飞书：数仓已完成」就行。

## 发一条消息

把请求写成 `m.json`，先预览（不会真发），确认后再发送，两次请求体相同：

```json
{
  "template": "ops.text.v1",
  "target": {"name": "张三"},
  "data": {"body": "2026-10-21 数仓已完成"},
  "idempotency_key": "myapp-warehouse-20261021-zhangsan"
}
```

```bash
curl -sS -X POST "$NOTIFICATION_API_URL/v1/notifications/preview" \
  -H "Authorization: Bearer $NOTIFICATION_API_TOKEN" -H 'Content-Type: application/json' --data @m.json
curl -sS -X POST "$NOTIFICATION_API_URL/v1/notifications" \
  -H "Authorization: Bearer $NOTIFICATION_API_TOKEN" -H 'Content-Type: application/json' --data @m.json
```

返回 HTTP 200 且 `"status":"sent"` 就是发出去了。

## 发给谁：target 四选一

| 写法 | 用法 | 注意 |
|---|---|---|
| `{"name": "张三"}` | 飞书通讯录里的在职姓名 | 必须完全一致（包括大小写）；同名多人会返回 409，这时改用 `open_id`；当天入职的人第二天 08:00 后才能查到 |
| `{"open_id": "ou_…"}` | 飞书用户 ID | 必须是「伊策 - 小助理」这个应用下的 open_id |
| `{"chat_id": "oc_…"}` | 飞书群 ID | 先把「伊策 - 小助理」拉进群，否则发不出去 |
| `{"ref": "<登记名>"}` | 维护者预先登记的目标 | 名单用 `GET /v1/catalog` 查 |

## 消息格式：template 三选一

| template | 飞书里显示为 | data | 适合 |
|---|---|---|---|
| `ops.text.v1` | 纯文本 | `{"body": "…"}` | 告警、一句话通知 |
| `demo.markdown.v1` | 带标题的 Markdown | `{"title": "…", "body": "…"}` | 结构化汇报；支持加粗、链接、列表、代码，表格会显示成纯文本 |
| `delivery.rich.v1` | 带标题的富文本 | `{"title": "…", "content": [[行元素…], …]}` | 需要可点击链接的汇总 |

Markdown 示例：

```json
{"template": "demo.markdown.v1", "target": {"name": "张三"},
 "data": {"title": "日报已更新", "body": "**项目 A** 已完成\n\n- 曝光：12,345\n- 详情：[打开看板](https://example.com)"},
 "idempotency_key": "myapp-daily-20261021-a"}
```

富文本示例：每个内层数组是一行，行元素只有文本 `{"tag":"text","text":"…"}` 和链接 `{"tag":"a","text":"…","href":"https://…"}` 两种。

```json
{"template": "delivery.rich.v1", "target": {"chat_id": "oc_xxx"},
 "data": {"title": "推送完成", "content": [
   [{"tag": "text", "text": "项目 A：成功 "}, {"tag": "a", "text": "查看", "href": "https://example.com/a"}],
   [{"tag": "text", "text": "项目 B：跳过（无数据）"}]
 ]},
 "idempotency_key": "myapp-push-20261021"}
```

`data` 里的值可以是字符串、数字或布尔值；传列表会渲染成逐行的 `- 项`。大小上限：请求体 256 KB，渲染后的文本 32,000 字符；富文本最多 128 行、每行最多 32 个元素。

## 幂等键与重试

`idempotency_key` 是这条消息的身份证：8–192 位，只能用字母、数字和 `. _ : -`，首字符是字母或数字，建议写成「来源-事件-业务 ID」。同一个键重复请求，服务只回放第一次的结果，不会重复发送。所以：

- **超时或结果不确定**：用同一个键原样重发，安全。
- **想再发一条新消息**：换一个新键。

## 返回码速查

| 返回 | 含义 | 怎么办 |
|---|---|---|
| 200 `sent` | 飞书已接收 | 完成 |
| 202 `unknown` / `sending` | 结果未确认 | 用同一个键原样重发，或按 `notification_id` 查询 |
| 502 `failed` | 明确没送达，`error_code` 写了原因 | 修正后用新键重发 |
| 400 / 404 / 422 | 请求写错，比如查无此人、模板不存在、字段不对 | 改请求 |
| 409 | 同名多人，或幂等键已经用在别的内容上 | 改用 `open_id`；或者换新键 |
| 429 | 发得太快（每分钟 60 条，或服务排队超过 10 秒） | 稍后用同一个键重发 |
| 401 | 缺 token 或 token 不对 | 检查 `NOTIFICATION_API_TOKEN` |
| 403 | 从没登记过的 IP 调用，或权限不够 | 找维护者更新 |
| 连接超时 | 出口 IP 不在白名单里，不是 token 的问题 | 把新 IP 报给维护者 |

查询接口：`GET /v1/whoami`（我是谁）、`GET /v1/notifications/{id}`（查一条）、`GET /v1/notifications?status=failed`（查失败记录）。完整的接口合同见 [references/api.md](yice-feishu-notice/references/api.md)。

## 并发与限速

可以并发调用。服务统一控制发往飞书的速度：整体每秒约 40 条，同一个人或同一个群每秒约 4 条。往同一个群批量发时，把内容合并成一条消息，不要逐行刷屏。

## 安全

- 这个接口走明文 HTTP：消息里不要放密码、密钥等机密。
- token 只放环境变量或私有文件，不要写进代码仓库、聊天记录或 URL。
- token 怀疑泄露时，立刻找维护者换一个。
