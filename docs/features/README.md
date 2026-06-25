---
description: 微信小程序业务功能模块清单，认证、支付、加密、上传、容灾的实现方案
---

# 业务功能

本章节记录小程序中常见业务模块的实现方案。每个模块都遵循 [分层架构](../getting-started/) 原则：业务流程编排与平台 API 调用统一收口在 `services` 层。

## 模块清单

| 模块 | 说明 | 对应 services 子模块 |
|------|------|----------------------|
| [认证](./authentication/) | 微信登录、手机号绑定、Token 管理 | `services/auth` |
| [支付](./payment/weixinpay/) | 微信支付调起、订单状态同步 | `services/payment` |
| [敏感数据加密](./encrypt/) | AES 双向加密、数据脱敏 | `services/crypto` |
| [文件上传](./file-upload/) | 基于 `wx.uploadFile` 的上传方案 | `services/http` |
| [接口容灾](./recovery/) | 主备域名、本地缓存兜底 | `services/http` + `services/storage` |

## 迁移说明

本章节内容多源自 Web 项目实践，针对小程序环境做了适配。与 Web 完全一致的部分不再重复，仅记录小程序特有的差异部分。

:::tip[基础理论 vs 端实现]
每个模块文档会明确区分：

- **基础理论**：业务流程、状态机、安全策略，可直接平移
- **端实现**：基于小程序 API 的具体代码，需重写
:::
