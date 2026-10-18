# ADR 0001：采用 Chatwoot 社区版做 KT 客服台

状态：已决策
日期：2026-10-18

## 背景

KT 需要一套自托管客服台：网站在线聊天、邮件、WhatsApp、Telegram、LINE 进同一个坐席收件箱，并保留分配、协作和帮助中心。

通用能力是全渠道收件箱。KT 独有的部分是身份（kt-identity）、AI 出口（New API）和部署。桌面端已有 Bug 反馈和客服入口，`kt-customer-bot` 负责获客，都不是坐席共享收件箱。

2026-10-18 对照过仍在维护的同类项目：Zammad、FreeScout、Frappe Helpdesk、LibreDesk。Chatwoot 社区版星数最多（约 3.8 万），渠道覆盖 KT 正在用的 WhatsApp、Telegram、LINE，社区版 MIT。LibreDesk 的 Go 技术栈更贴近 KT，但 WhatsApp 尚未进入正式版，也没有 Telegram 和 LINE，许可证是 AGPL-3.0。

## 决策

采用 Chatwoot 社区版，方式是 wrap。

- 真 fork：`KTAIorg/chatwoot`，上游 `chatwoot/chatwoot`，保留 GitHub fork 网络和共同祖先。
- 生产线钉 release tag。Day 0 基线是 `v4.18.0`，不跟 `develop`。
- KT 只写认证、品牌、CI、部署。收件箱、渠道、坐席界面用上游。
- 对坐席和客户显示「KT 客服」。仓库名保持 `chatwoot`。
- 生产使用社区版，`DISABLE_ENTERPRISE=true`。不把 `enterprise/` 的付费能力编译进对外服务。

## 理由

- 渠道覆盖和社区活跃度没有另一个开源项目同时超过它。
- MIT 允许私有定制。AGPL 候选项目会要求把网络服务上的修改公开。
- 深度改 Rails 核心会在每月一次的上游 release 上持续冲突。new-api 的快照 fork 已经证明，没有共同祖先就补不进安全修复。
- Captain、SSO、SLA、自定义品牌属于付费计划。需要时购买对应席位，或在认证门面和 New API 上做薄封装。不在 fork 里重写这些功能。

## 非目标

- 不把本仓库做成 Chatwoot Cloud 的多租户克隆，也不对外做换皮服务。
- 不在本仓库重写 WhatsApp、Telegram、LINE 协议。渠道用 Chatwoot 官方收件箱。
- 不在 Day 0 部署到 ACK，不登记域名，不把仓库写进 `repos-inventory.yaml` 的已纳管服务。部署时再走 GitOps onboarding，并在那时把分类登记为 `service`。
- 不上传明文聊天、联系人、手机号、cookie、token。

## 约束

- 定制超出白名单必须在 PR 里说明理由。见 `docs/development/upstream-sync-policy.md`。
- 安全修复以上游 tag 为准。
- 域名分配走 `kt-edge-dns` 的 policy 和 inventory，不在本 ADR 里指定主机名。
- 仓库是公开 fork。密钥和对话数据不入库。

## 待定问题

- 认证门面的具体落点：独立服务，还是上游 OAuth 配置。
- 坐席规模是否值得购买 Chatwoot Premium / Enterprise，用来打开 Captain 或 SSO。
- test 环境的 Postgres、Redis 和 FOSS 镜像由哪次 onboarding PR 引入。
