---
title: TECH-12-星图公共 API 与 Webhook 规范
version: 1.1
created: 2026-09-02
status: confirmed
updated: 2026-09-05
owner: 新星团队
reviewers: 用户（本会话审议确认）
last_reviewed: 2026-09-05
effective_date: 2026-09-05
supersedes: 1.0-draft
previous_status: confirmed
---

# 星图公共 API 与 Webhook 规范 v1.1


## 1. 范围与契约

本文件为 API/Webhook 唯一规则来源。机器契约为 [OpenAPI 3.1](api/openapi.json) 和 [Webhook JSON Schema](api/webhook-event.schema.json)。公开资源包括社群、主题、关系、活动、帖子和公开 Wiki 页面/版本/脚注；不包含用户主页、隐藏关系、申请、举报、原始证明、个人选票、账户 ID 或行为日志。

核心首发不开 API；第二批只读 API 和后续 Webhook 按 TECH-17 开关。不存在的功能不要提前向用户签发 scope。个人数据导出由本人会话授权，公共 API 不设 `export_manage`。

## 2. 认证、密钥和限流

除文档、版本和健康检查外，必须 `Authorization: Bearer <api_key>`。密钥随机生成至少 256 位熵，只显示一次、数据库只存哈希；默认 90 天有效，最长 365 天，可撤销。scope 为 `public_read`，后续启用 Webhook 时可增加 `webhook_manage`；不允许发帖、Wiki 编辑、投票或治理写入。账户注销或封禁立即撤销密钥。

每密钥和关联主体共享上限每分钟 60 次、每日 50,000 次，任一达到即限流，防止通过多密钥叠加；每人最多 5 个有效密钥。批量最多 100 个 ID，除一次请求限额外按对象计入每日 100,000 个对象预算。搜索、历史、关系每分钟各 10 次；批量每分钟 5 次。IP 使用宽松防护桶，不凭 IP 判定同一人。429 返回 Retry-After、错误码和请求 ID。24 小时日窗口使用 UTC。

默认关闭浏览器跨域；服务器端密钥不得嵌入浏览器页面。来源域名白名单最多 3 个 HTTPS 域，用于受控客户端，并非密钥保密或授权机制。若需公开浏览器应用应经后端代理，首版不提供把长期 Bearer 密钥公开在前端的接入方式。

## 3. 数据、状态与缓存

成功为 `data`、`meta`，错误为 `errors[]`、`meta.request_id`。使用 snake_case、随机 UUID、UTC ISO 8601；公开资源包含 `resource_type`、ID、版本、许可证、发布时间、更新时间、可见性和状态，非版本对象 `revision_id` 可空。个人化名可见，内部账户/关联标识不可见。

| HTTP | code | 行为 |
|---|---|---|
| 400 | INVALID_ARGUMENT / INVALID_CURSOR | 参数或游标格式无效 |
| 401 | UNAUTHENTICATED | 密钥无效/失效 |
| 403 | INSUFFICIENT_SCOPE | 操作超出 scope，不附私密对象信息 |
| 404 | NOT_FOUND / FEATURE_DISABLED | 不存在、从未公开、无权获知其存在或功能未开放 |
| 409 | CONFLICT | Webhook 状态/并发冲突 |
| 410 | RESOURCE_GONE / CURSOR_EXPIRED / SYNC_RESET_REQUIRED | 已公开资源删除、游标到期或增量水位过期 |
| 422 | UNSAFE_TARGET | Webhook 出站目标不安全 |
| 429 | RATE_LIMITED | 返回 Retry-After |
| 503 | TEMPORARILY_UNAVAILABLE | 维护或依赖故障，可重试 |

曾公开且允许保留墓碑的删除对象返回 410，错误 detail 中只有 ID、状态、时间、允许公开的原因类别和安全的合并目标；高风险或不能透露存在性的对象统一 404。墓碑不带标题、作者、正文、附件。批量逐项标记 ok/gone/not_found，不能通过批量绕过此规则。

授权与有效可见性检查必须先于 ETag/304 判断；权限撤回后不能返回 304 让客户端使用旧正文。使用 `Cache-Control: private, max-age=0, must-revalidate` 和 `Vary: Authorization`，受限/错误为 no-store。读取层立即过滤隐藏/删除对象，物理搜索索引最多 5 分钟完成清理。客户端规则见 SA3；已经离线下载的副本不能由技术强制追回。

## 4. 分页与增量

默认 20、最多 100 条，按 `(created_at, id)` 稳定排序；游标为签名的不透明字符串，服务端绑定过滤条件、资源、主体、高水位和最后排序键，有效 24 小时。不得包含裸内部账户标识。客户端改变过滤条件须重新分页；到期返回 CURSOR_EXPIRED。列表是实时公开视图，不保证被删除对象继续出现在旧页。

`GET /api/v1/changes` 是独立公开变更序列，只返回事件类型、对象和版本 ID、时间、API 查询地址，无正文。按严格单调序号和 ID 排序，游标冻结首请求高水位；`updated_since` 为 UTC 时间下界，按大于等于处理并允许重复，客户端按事件 ID 去重。删除/受限产生可公开的墓碑事件，私密对象从未公开不生成此事件。

变化序列保存 30 天；超期返回 SYNC_RESET_REQUIRED，客户端重新校验其已保存对象，不能把遗漏删除解释为仍公开。无全站全文导出接口，增量、批量与普通读取共用限额。随机 ID 和游标签名不能用来阻止已授权客户端复制，限制依赖条款和限流。

## 5. Webhook 目标与签名

仅订阅明确的公开对象，必须拥有 `webhook_manage`。目标 HTTPS、443 端口，无 URL 凭据和片段；创建、修改、每次投递/重试都解析并校验 IPv4/IPv6，拒绝本机、私网、保留、链路本地和云元数据地址。DNS 结果固定到实际拨号地址，TLS 仍验证原主机名；禁止跟随重定向，防 DNS 重绑定。连接 3 秒、总超时 10 秒，响应体最多 64 KB。

事件 schema_version 为 `1`，只含事件 ID、类型、公开对象 ID、资源类型、可空版本 ID、UTC 时间和相对 API 地址。每个订阅独立随机签名密钥，仅创建/轮换显示一次，服务端加密存储；轮换重叠 24 小时并携带 key_id。

请求头为 X-Starmap-Timestamp（Unix 秒）、X-Starmap-Event-ID、X-Starmap-Key-ID、X-Starmap-Signature（v1=十六进制摘要）。HMAC-SHA256 输入为 `timestamp + "." + event_id + "." + 原始请求体字节`。先校验签名和 5 分钟时间窗，再按订阅/事件 ID 去重，去重记录保留 7 天。重试重新签名时间戳，保持事件 ID 和请求体不变；不重新序列化签名前原始字节。

## 6. 投递、重试和暂停

至少一次投递，不保证事件到达顺序；客户端按对象版本查询最新状态。仅 2xx 算成功。初次失败后按 1 分钟、5 分钟、30 分钟、2 小时、12 小时各重试一次，共最多 6 次尝试；最后失败标记永久失败并暂停订阅。跨事件连续 10 次失败或自首次失败持续 24 小时也暂停，以先发生者为准。超时、重定向、TLS/DNS 安全检查失败均计失败；安全失败不连接被拒地址。

待发送事件保存 7 天，成功摘要 30 天；删除 Webhook 取消未发送任务，暂停不自动补发历史队列。人工恢复需确认目标并重新做安全校验，遗漏事件通过 changes 在权限内补取。所有者注销、密钥撤销或资源权限变化时停止相关投递并复核排队事件，不发送尚未公开的元数据。

## 7. 版本与沙盒

新增可选字段不改变旧字段含义，删除/语义变化升级 API 主版本并提前至少 6 个月公告；事件 schema 主版本变更需订阅者明确接受。schema、示例和兼容性测试随版本发布。未知可选字段可忽略，未知事件主版本停止处理并告警。

沙盒使用独立数据库、OSS、OAuth、SMTP 和密钥，仅合成数据；出口策略拒绝正式环境域名与地址。健康检查仅返回粗略状态，不泄露内部配置。

## 8. 许可证

代码 GPLv3 与内容许可证分开。Wiki 为 CC BY-SA 4.0；普通帖子逐项返回其 CC BY-SA 4.0、CC BY 4.0 或保留全部权利，不把所有 API 内容强制标成同一许可。保留署名、来源和许可证；API 访问限制不能撤销已授予的 CC 权利。
