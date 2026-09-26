---
title: TECH-04-星图 API 设计
version: 1.1
created: 2026-08-30
status: confirmed
updated: 2026-09-05
owner: 新星团队
reviewers: 用户（本会话审议确认）
last_reviewed: 2026-09-05
effective_date: 2026-09-05
supersedes: 1.0-draft
previous_status: draft
---

# 星图 API 设计 v1.1


## 1. 定位与权威契约

本文件提供实现概览。API 认证、字段、分页、错误、生命周期、限流、Webhook 和兼容性规则统一由 [TECH-12](TECH-12-星图公共API与Webhook规范-v1.md) 维护；机器契约为 [OpenAPI](api/openapi.json)，Webhook 消息为 [JSON Schema](api/webhook-event.schema.json)。出现差异必须修订契约后发布，不在此重复定义另一套参数。

## 2. 实现分层

路由先做认证、关联主体限流和 scope 检查，再调用与网页共享的可见性策略；序列化只输出明确允许的公开字段。随机 ID 不代替授权，ETag、计数、关系与历史版本同样要鉴权。用户导出通过本人会话通道，公共读取密钥无导出私密资料的能力。

社群、主题、动态、活动、Wiki 和公开变化序列由独立查询服务读取，统一输出 schema；后台事件与业务变更同事务提交，异步投递不持有数据库写事务。列表、详情、批量、搜索、旧版本及下载均在输出前复查对象状态。

## 3. 功能阶段

TECH-17 的核心首发不开放公共 API。第二批只读 API 通过契约、删除和限流测试后开放；Webhook 在后续阶段通过出站、签名和重试测试后开放。关闭的路由返回 FEATURE_DISABLED，不签发未启用的 scope。功能关闭不删除已经存在的数据或审计记录。

## 4. 交付检查

开发实现必须以 OpenAPI 和事件 schema 为固定夹具执行字段、状态码、分页、删除、条件请求和签名测试；公开文档页只展示已启用端点。API 沙盒使用独立数据库、存储、密钥和 OAuth 客户端，只含合成数据。
