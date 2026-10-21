---
name: yice-feishu-notice
description: "经 yice 通知 API 用企业 bot「伊策 - 小助理」发飞书通知：按人名、open_id、群 chat_id 或登记目标发 Markdown / 纯文本 / 富文本；预览、发送、查记录；给新调用方开通 token 与来源 IP 白名单。Trigger on: 发飞书通知, 通知某人, 发到群, 通知 API, yice-notify, 接入通知, 开通通知权限, 加白名单, 通知发失败, 429/403。Does NOT trigger on: 看板推送与推送通知（→ feishu-push）、以个人身份发消息或读写飞书文档表格（→ feishu-call）。"
metadata:
  service: project/yice-notification
---

# yice 飞书通知

## 目标

消息经线上通知 API 发出，响应 `status=sent`，并留有可追溯的 `notification_id` / `message_id`；或者调用方拿到可用的 token，能从自己的来源 IP 调通 `whoami`。真实发送前先预览；发送需要本次任务的明确授权。

## 结果模板

所有人（包括 yice 自己）都直接调线上 API，不经本地工具。调用前需要两个环境变量：`NOTIFICATION_API_URL`（外网 `http://218.244.157.197:8772`；同一 VPC 用内网地址，见 api.md）和 `NOTIFICATION_API_TOKEN`（本调用方 token，开通时由维护者交付）；yice 本机两者都在 `project/yice-notification/.env`，其他人按 README 安装时写在 `~/.config/yice-feishu-notice/env`（环境变量没设置时先加载它）。一条通知写成请求体文件 `m.json`：

```json
{"template": "demo.markdown.v1", "target": {"name": "张三"},
 "data": {"title": "数仓完成", "body": "2026-10-21 全部到位"},
 "idempotency_key": "task-warehouse-20261021"}
```

先预览，再发送（两者请求体相同）：

```bash
curl -sS -X POST "$NOTIFICATION_API_URL/v1/notifications/preview" \
  -H "Authorization: Bearer $NOTIFICATION_API_TOKEN" -H 'Content-Type: application/json' --data @m.json
curl -sS -X POST "$NOTIFICATION_API_URL/v1/notifications" \
  -H "Authorization: Bearer $NOTIFICATION_API_TOKEN" -H 'Content-Type: application/json' --data @m.json
```

成功的标志是 HTTP 200 且响应 `status` 为 `sent`；其他状态码按 api.md 的应对表处理。查询用 `GET /v1/whoami`、`GET /v1/notifications/{id}`、`GET /v1/notifications?status=failed`，同样带上 token。

接入新调用方，对方只需提供服务端看到的出口 IP（同一 VPC 的机器给内网 IP）；调用方名、默认权限 `notifications:send` 由维护者定。维护者按 yice 主仓 `project/yice-notification/deploy/DEPLOY.md`「调用方维护」完成：在 `config/callers.v1.json` 声明（含 `sources`）、在 ECS `.env` 生成 token、`deploy.sh --execute`，外网来源再跑 `sync_whitelist.py --execute`，然后私聊发给对方一段安装口令：`请安装飞书通知 skill：按 https://github.com/DaxDing/yice-feishu-notice 的 README「AI 安装步骤」操作，我的 token 是 <token>`。对方的 AI 装好后调通 `whoami`，就是回读。

## 资料指南

- [references/api.md](references/api.md)：完整 HTTP 合同，包括入口地址、端点、`target` 四种写法、三个模板的 `data` 结构与上限、全部状态码与应对、并发配速。给 Info、Task 或外部系统写对接代码时读它；分享给别人时连同本文件一起给。
- 登记目标和模板以线上 `GET /v1/catalog` 为准；服务行为与配置的真源是 yice 主仓 `project/yice-notification/README.md`（仅维护者可见）。

## 红线

- **幂等键是消息的业务身份**：按「来源-事件-业务 ID」取稳定值。超时、202、503 `storage_unavailable` 时原样重发同一请求（服务会回放，不会重复发）；换键等于再发一条，会重复打扰收件人。
- **429 用原键重试，failed 才换键**：`throttled` / `rate_limited` 未落库，稍后原样重发；`failed` 是明确未送达，修正原因后才用新键重发。
- **先分清连不上的原因**：连接超时是来源 IP 不在安全组白名单；401 是 token 问题；403 `source_forbidden` 是 token 从未声明的 IP 出示。三者修法不同，不要靠换 token 解决超时。
- **群要先拉 bot**：发 `chat_id` 前，「伊策 - 小助理」必须已在群内，否则飞书拒收、记为 failed。
- **人名只认精确在职姓名**：同名返回 409，必须改用 `open_id` 或登记目标，不要猜部门挑一个。当天入职的人，通讯录次日 08:00 同步后才查得到。
- **明文 HTTP**：token 只放环境变量或私有文件，不进 URL、日志、聊天或 Git；消息内容不放密码、密钥。
- **批量看配速**：同一群每秒约 4 条；批量结果合并成一条消息发，不要逐行刷屏。
