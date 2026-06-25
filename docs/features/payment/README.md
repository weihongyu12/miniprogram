---
description: 微信小程序支付业务实现方案，wx.requestPayment 调起与订单状态同步
---

# 支付

本章节记录小程序支付业务的实现方案。

## 子文档

- [微信支付](./weixinpay/) - 小程序支付调起、订单状态同步

## 架构分层

```mermaid
graph TD
    P[pages/payment] --> S[services/payment]
    S --> WX[wx.requestPayment]

    style P fill:#e1f5fe,stroke:#03a9f4
    style S fill:#fff3e0,stroke:#ff9800
    style WX fill:#c8e6c9,stroke:#4caf50
```

| 层 | 职责 |
|----|------|
| `services/payment` | 封装 `wx.requestPayment` 为 Promise，并编排创建订单、轮询订单状态、支付结果回调 |
| `pages/payment` | 支付 UI、用户交互，调用 `services/payment` |
