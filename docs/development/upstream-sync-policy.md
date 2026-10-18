# 上游同步与 Fork 开发策略（Chatwoot）

日期：2026-10-18
状态：生效
规范来源：kt-agent-skills/oss-fork-maintenance

## 仓库角色

- `upstream` = https://github.com/chatwoot/chatwoot.git（只读，禁止 push）
- `origin` = https://github.com/KTAIorg/chatwoot.git（KT fork，GitHub fork 网络内）

本仓库是公开 fork。Chatwoot 社区版代码本来就是公开的 MIT 许可。客户对话、联系人、手机号、cookie、token 只进部署环境的数据库，不进 Git。

GitHub 不允许把公开仓库的 fork 改成私有还保留 fork 关系。需要切断 fork 网络时，先确认 `git merge-base` 仍然找得到共同祖先，再单独决策。

## 分支模型

上游默认分支是 `develop`，稳定版用 tag。KT 不把 `develop` 当生产线。

| 分支 | 职责 |
|---|---|
| `develop` | 只镜像 `upstream/develop`。禁止 KT 提交 |
| `main` | KT 生产线。上游 release tag + 白名单定制。当前基线 `v4.18.0`（`9f920b549c14491a4e587687a3eed5d21c6ccc7d`） |
| `upstream-sync` | 跟上游的缓冲线。在这里合并目标 tag、解冲突 |
| `gitops-prod` | 部署纳入 Argo CD 后再建立。本文件生效时不存在 |

同步目标用 release tag，不用 `develop` HEAD。`v4.18.0` 之后 `develop` 已再前进，那些提交等下一个评估过的 tag。

## 许可证边界

根目录 `LICENSE`：`enterprise/` 以外是 MIT Expat。`enterprise/LICENSE` 要求生产使用付费订阅，并按坐席计数。

- 运行社区版时设置 `DISABLE_ENTERPRISE=true`。仓库里存在 `enterprise/` 目录时，`ChatwootApp.enterprise?` 会把进程当成带企业代码的安装。
- 不设置 `INSTALLATION_PRICING_PLAN=premium` 或 `enterprise`。
- 不启用、不对外提供 Captain、SSO/SAML、SLA、自定义品牌、角色权限、语音。这些在自建付费计划里。
- 不修改 `enterprise/` 来绕过订阅。未购买席位前，生产镜像使用上游 FOSS 构建，不用 EE 镜像。
- 镜像来源是上游发布的社区版镜像，或本仓库日后单独的 KT workflow。不改 `publish_foss_docker.yml` / `publish_ee_docker.yml` 往 `chatwoot/chatwoot` 推镜像。

## KT 定制边界（白名单，超出需 PR 论证）

Day 0 没有业务代码定制。允许将来改动的只有下面四域。

1. 认证 / kt-identity：登录走 kt-identity 的 OIDC。优先独立认证门面或上游已有的 OAuth 配置，不新写用户体系。身份主键用 `sub = kt_account_id`。登录成功不等于有坐席权限，坐席要显式授权。
2. 支付 / ktpay：本仓库不做坐席计费，此域为空。不接 Chatwoot 的 Stripe 计费。
3. 品牌：坐席和客户界面显示「KT 客服」。仓库名保持 `chatwoot`，产品名不带 Chatwoot。品牌只改安装名、发件名、帮助中心标题这类配置。未购买自定义品牌能力前，不改企业版品牌开关。
4. CI / 部署：KT 自己新增的 workflow 和 `deploy/`。部署钉 release tag。

对话正文留在 Chatwoot 的 Postgres。向外只发会话编号、分配、满意度这类事件。

## 同步频率

- 安全修复：上游发布后即时评估并同步。
- 常规：每个上游 release 评估一次。
- 保底：每季度至少一次，防止生产线停在旧 tag。

## 冲突优先级

安全修复以上游为准 > 上游已内建的能力则删掉对应 KT 定制 > 白名单域定制保留并重放 > 纯风格以上游为准。

## 操作

```bash
git fetch upstream --tags
git switch upstream-sync
git merge vX.Y.Z
# 解冲突后
# PR: upstream-sync -> main
```

PR 描述写明：上游 from → to、冲突摘要、白名单定制是否受影响、验证结果。

合入 `main` 之后，把 `develop` 快进到当时的 `upstream/develop`，保持镜像，不把 KT 提交带进 `develop`。

## 给审查者和 Agent

仓库钩子不允许在 `docs/`、`adr/` 之外新增说明文件，因此审查约定写在这里，不单放 `.github/copilot-instructions.md`。

- KT 新增文档和 PR 用简体中文。上游原有英文文档保持英文。
- 审查先看改动有没有越出白名单，以及下一次 tag 合并会不会被卡住。
- 拒绝把付费企业版能力带进社区版部署。
- 拒绝把明文客户数据写进日志、诊断包、Bus 或 Git。
- 涉及部署、数据库、鉴权的变更要写验证方式和回滚方式。

## 验证清单

部署和业务改动发生前，下面各项保持未执行。一旦改白名单域或升级 tag，逐项做：

- [ ] 上游该 tag 的 build + test
- [ ] 认证门面：kt-identity 登录、无权限账号被拒绝
- [ ] 品牌显示为「KT 客服」，仓库与界面都不把产品叫成 Chatwoot 的换皮
- [ ] `DISABLE_ENTERPRISE=true`，付费能力未打开
- [ ] 数据库 migration 先在 test 环境演练
- [ ] 目标 tag 的安全修复已包含
- [ ] 生产先灰度，再全量

## 同步记录

| 日期 | 上游 from → to | 冲突摘要 | 验证 | 操作者 |
|---|---|---|---|---|
| 2026-10-18 | 无 → v4.18.0 (`9f920b549`) | 无。Day 0 只增加本文、ADR 和 Copilot 指令 | 未部署，未跑上游测试 | Day 0 采纳 |
