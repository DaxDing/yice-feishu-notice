# yice 飞书通知 API 合同

服务端真源：yice 主仓 `project/yice-notification`（README、`config/notification_catalog.v1.json`、`config/callers.v1.json`，仅维护者可见）。本页是调用方视角的摘要；字段与服务不一致时以服务和 `GET /v1/catalog` 为准。

## 入口与鉴权

| 调用位置 | `NOTIFICATION_API_URL` |
|---|---|
| 外网（出口 IP 已加白名单） | `http://218.244.157.197:8772` |
| 同一 VPC 的 ECS（如 Task） | `http://172.23.214.40:8772` |
| Info ECS 本机 | `http://127.0.0.1:8772` |

- 每个调用方一个 token：请求头 `Authorization: Bearer <token>`（或 `X-Notification-Token`）。token 只放环境变量或密钥文件。
- token 只在声明的来源 IP 有效；外网来源还要在安全组白名单里。明文 HTTP：消息里不放密码、密钥等机密。
- 发送人是企业应用 bot「伊策 - 小助理」。
- 可选请求头 `X-Request-Id`（`[A-Za-z0-9._:-]{1,128}`）会写进服务端审计日志，便于对账；响应头 `X-Request-Id` 是服务端生成的请求号。

## 端点

| 方法 / 路径 | 权限范围 | 用途 |
|---|---|---|
| `GET /health` | 无 | 存活、配置与发送配速 |
| `GET /v1/whoami` | 任意 | 当前调用方、权限、限速、允许来源 |
| `GET /v1/catalog` | 任意 | 模板、登记目标（不含 ID） |
| `POST /v1/notifications/preview` | `notifications:send` | 渲染预览：不发送、不落库、会解析收件人 |
| `POST /v1/notifications` | `notifications:send` | 同步发送一次 |
| `GET /v1/notifications/{id}` | `notifications:send` | 查询本调用方的一条（有 `notifications:read` 可查全部） |
| `GET /v1/notifications?status=&caller=&limit=&cursor=` | `notifications:send` | 倒序列表，`limit` ≤ 100，`next_cursor` 翻页 |
| `POST /v1/events` | `events:write` | 记录事件到时间线，不发送 |

## 请求体（preview 与 send 相同）

```json
{
  "template": "ops.text.v1",
  "target": {"name": "张三"},
  "data": {"body": "数仓 2026-10-21 已完成"},
  "idempotency_key": "task-warehouse-20261021-zhangsan"
}
```

### target 四选一

| 写法 | 含义 | 约束 |
|---|---|---|
| `{"ref": "<登记名>"}` | 服务端登记的目标 | 名单见 `GET /v1/catalog` |
| `{"name": "张三"}` | Info 通讯录在职成员姓名 | 去首尾空格后完全一致；同名多人 409，查无此人 404；通讯录每天 08:00 同步 |
| `{"open_id": "ou_…"}` | 飞书用户 open_id（本企业应用「伊策 - 小助理」下的 ID，Info / Task 存的就是它） | 未知或离职用户由飞书拒收，记为 failed |
| `{"chat_id": "oc_…"}` | 飞书群 ID | bot 必须已在群里，否则 failed |

### template 与 data

| template | 飞书格式 | data | 说明 |
|---|---|---|---|
| `demo.markdown.v1` | post · Markdown | `{"title": "…", "body": "…"}` | 渲染为 `## {title}` 加正文；支持加粗、链接、列表、代码，表格显示为纯文本 |
| `ops.text.v1` | 纯文本 | `{"body": "…"}` | 告警、短句 |
| `delivery.rich.v1` | post · 富文本 | `{"title": "…", "content": [[行元素…], …]}` | 行元素仅 `{"tag":"text","text":"…"}` 与 `{"tag":"a","text":"…","href":"https://…"}`；1–128 行、每行 1–32 个元素、每段文字 1–4000 字 |

- `data` 字段必须与模板占位符一致；值可为字符串、数字、布尔，列表渲染为 `- 项` 逐行，对象渲染为 JSON。
- 上限：请求体 256,000 字节；渲染文本 32,000 字符；飞书消息体 30,000 UTF-8 字节。

### idempotency_key

8–192 位 `A-Z a-z 0-9 . _ : -`，首字符为字母或数字。它是这条消息的业务身份：同一调用方、同一键、同一内容的重复请求回放原记录，不会重复发送；同键不同内容或属于其他调用方返回 409。

## 响应与处理

发送响应体含 `notification_id`、`caller`、`template`、`target`、`status`、`message_id`、`chat_id`、`error_code`、`error_message`、`created_at`、`updated_at`、`replayed`。

| HTTP | status / code | 含义 | 调用方动作 |
|---|---|---|---|
| 200 | `sent` | 飞书已受理（不代表已读） | 完成 |
| 202 | `unknown` / `queued` / `sending` | 结果未确认 | 同一请求原样重发（回放）或 `GET` 原 ID；**禁止换键** |
| 502 | `failed` | 明确未送达；`error_code` 给原因 | 修正原因后用**新键**重发；`feishu_rate_limited` 稍等再用新键 |
| 400 / 404 / 422 | `notification_invalid`、`target_invalid`、`template_not_found`、`target_not_found`、`recipient_not_found`、`recipient_name_invalid`、`invalid_request` | 请求有误，未落库 | 改请求 |
| 409 | `idempotency_conflict` | 键已用于别的内容或别的调用方 | 是新消息就换新键；是同一条就恢复原内容 |
| 409 | `recipient_ambiguous` | 同名多人 | 改用 `open_id` 或登记目标 |
| 429 | `throttled` | 服务配速排队超过等待上限，未落库 | 几秒后同一请求原样重发 |
| 429 | `rate_limited` | 超过本调用方每分钟发送上限，未落库 | 按 `Retry-After` 后原样重发 |
| 401 | `token_required` / `unauthorized` | 缺 token 或 token 无效 | 检查 `NOTIFICATION_API_TOKEN` |
| 403 | `source_forbidden` / `scope_forbidden` | 来源 IP 未声明 / 权限范围不足 | 找维护者更新声明 |
| 503 | `directory_not_configured`、`directory_unavailable`、`transport_not_configured`、`storage_unavailable` | 依赖不可用 | `storage_unavailable` 按 202 处理，其余稍后重试 |
| 连接超时 | — | 多为来源 IP 不在安全组白名单（不是 token 问题） | 找维护者加白名单；send 超时按 202 处理 |

## 并发与配速

可以并发请求。服务对飞书统一配速：同时在途最多 16 条、整体每秒 40 条、同一收件人每秒 4 条；排队超过 10 秒返回 429 `throttled`。每个调用方另有每分钟发送上限（`GET /v1/whoami` 的 `sends_per_minute`）。实测不同收件人约 25 条/秒，单个群约 4 条/秒：同一群的批量内容合并成一条消息。
