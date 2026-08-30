---
title: TECH-04-星图 API 设计
version: 1.0-draft
created: 2026-08-30
status: draft
---

# 星图 API 设计 v1

## 1. 原则

公共 API 只提供平台定义为公开的数据，不绕过网页权限、隐私设置、许可证或风险限制。API 不用于站外镜像、搜索索引或推荐站。

## 2. 公开资源

- `communities`：社群页面、状态、公开主题和公开入口；
- `themes`：正式主题和公开标签；
- `relations`：已确认或公开的社群/条目关系；
- `activities`：公开线上活动；
- `posts`：公开帖子、公开回复摘要、作者化名和“已注销”状态；
- `wiki/pages`：发现页、正式条目、消歧页、重定向页；
- `wiki/revisions`：公开版本、差异、公开贡献者、审核阶段和脚注。

不提供私下核验材料、举报证据、申请记录、行为日志、隐藏关系、受限来源、高风险隐藏历史和未公开作品细节。

## 3. 通用规则

- 所有响应包含 `license`、`published_at`、`revision_id`、`visibility` 和删除/合并状态；
- 受限资源返回状态和原因类别，不返回原文；
- 使用分页、稳定 ID 和 `updated_since` 参数；
- 软删除资源返回 `410 Gone` 或删除状态，不静默复活；
- API 版本使用 `/api/v1/` 前缀。

## 4. API 密钥

用户在设置中创建密钥。密钥只完整显示一次，可设置名称、权限范围、过期时间和撤销。数据库只保存密钥哈希。

每个密钥记录最近使用时间、调用次数和违规状态，但不保存完整请求参数中的个人内容。

## 5. Webhook

用户可以订阅公开社群、主题或 Wiki 页面更新。Webhook 请求带时间戳、事件 ID 和签名；接收方必须校验签名和去重。失败采用指数退避重试，超过次数后暂停并通知用户。

## 6. 限流与滥用

按 API 密钥、关联用户、IP 和资源设置频率限制。发现批量抓取、站外镜像、搜索索引、骚扰或绕过删除限制时，可以暂停密钥、限制账户并保留审计记录。

## 7. 示例端点

```text
GET  /api/v1/communities
GET  /api/v1/communities/{id}
GET  /api/v1/themes
GET  /api/v1/wiki/pages/{id}
GET  /api/v1/wiki/pages/{id}/revisions
GET  /api/v1/wiki/pages/{id}/relations
GET  /api/v1/activities
GET  /api/v1/posts/{id}
POST /api/v1/webhooks
DELETE /api/v1/webhooks/{id}
```

