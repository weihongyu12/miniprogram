---
description: 微信小程序云函数原理、与自建后端的能力对比、使用限制与混用方案的边界划分
---

# 云函数

## 什么是云函数

云函数是微信云开发提供的「无服务器函数」：你写一段 Node.js（也支持 Python/PHP），部署到微信云上，由微信托管运行环境与扩缩容。小程序端通过 `wx.cloud.callFunction` 调用，函数内部可直接用 `cloud.openapi.*` 调微信开放接口，**免 access_token 管理**。

它和「自建后端」不是同一层东西，是「微信侧适配层」的候选方案之一。

## 与自建后端的能力对比

| 维度 | 云函数 | 自建后端（Java/Node/Go + 服务器） |
|------|--------|--------------------------|
| 运维 | 零运维，微信托管 | 自己部署、监控、扩容 |
| access_token 管理 | 框架自动管理 | 自己实现获取/刷新/缓存 |
| 调用微信开放接口 | `cloud.openapi.*` 直连 | 自己签名请求 |
| 数据库 | 配套云数据库（NoSQL） | 自由选型（MySQL/PG/...） |
| 文件存储 | 配套云存储 | 自建 OSS/CDN |
| 冷启动 | 有，几百 ms ~ 数秒 | 无（常驻进程） |
| 单次执行时限 | 默认 20s，最长 60s | 无限制 |
| 本地调试 | 链路长、模拟麻烦 | 随便断点 |
| 单价模型 | 按调用次数 + 运行时长 | 按服务器规格 |
| 跨端复用 | 锁死微信生态 | 与端无关 |
| 业务复杂度上限 | 低~中 | 任意 |

## 混用方案的边界

云函数与自建后端**不是二选一**，可以混用：云函数做「微信侧适配」，自建后端做「业务事实源」。

### 决策表

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 订阅消息推送 | 云函数 | `cloud.openapi.subscribeMessage.send` 免 token |
| 生成小程序码 / 二维码 | 云函数 | `cloud.openapi.wxacode.*` 免 token，可直存云存储 |
| 内容安全检测 | 云函数 | `security.msgSecCheck` / `imgSecCheck` 直调 |
| 客服消息自动回复 | 云函数 | 微信事件直接推送，免回调 URL 鉴权 |
| 定时任务（订阅消息、对账通知） | 云函数 + 云定时器 | 省自建调度服务 |
| 上传文件后处理（缩略图、水印） | 云函数 | 配合云存储链路最顺 |
| 微信登录 `code2Session` | 自建后端 | 要签发自有 token、写用户表 |
| 支付核心流程 | 自建后端 | 统一下单、回调验签、订单状态更新 |
| 业务数据 CRUD | 自建后端 | 业务库在后端 |
| 长流程任务（对账、批量导入） | 自建后端 | 云函数超时限制 |

### 边界口诀

> **凡是「调微信接口」+「不需要碰业务库」的活给云函数；凡是「要碰业务库 / 要和用户体系联动」的活给自建后端。**

### 具体落点示例

| 业务动作 | 落在哪 | 说明 |
|---------|-------|------|
| 支付完成后更新订单状态 | 自建后端 | 碰订单库 |
| 支付完成后发订阅消息 | 云函数 | 后端通过 HTTP 触发云函数，云函数只调微信接口 |
| 用户上传头像生成缩略图 | 取决于文件存储位置 | 云存储用云函数，自建 OSS 用后端 |
| 用户登录 | 自建后端 | 要签发 token、写用户表 |
| 推广员生成专属小程序码 | 云函数 | 免 token，存云存储后返回 fileID |

## 使用限制

### 1. 冷启动

云函数长时间未被调用后，实例会被回收，下次调用需重新拉起容器。冷启动通常几百 ms ~ 数秒，**首次请求**或**流量突增**时明显。

**对策**：

- 避免在云函数里做重初始化（数据库连接、SDK 加载放函数外）
- 对延迟敏感的场景可使用「预置并发」（需付费）
- 关键路径上云函数**不要同步等待**用户请求，改为后端异步触发

### 2. 执行超时

| 类型 | 默认 | 最长 |
|------|------|------|
| 普通云函数 | 20s | 60s |
| 定时触发器触发的云函数 | 20s | 60s |

超时会被强制中断，**不能用于长流程任务**。批量处理要拆分，或改用自建后端。

### 3. 并发与实例

- 单个实例**串行处理**请求，并发请求会拉起新实例
- 实例数有上限（基础版 50、专业版可调高）
- 实例间**不共享内存**，状态不能放函数内

**对策**：

- 状态放云数据库或自建后端，不放函数级变量
- 高并发场景下用 `cloud.openapi` 调微信接口时要考虑微信侧 QPS 限制

### 4. 包大小

单个云函数代码包限制 **50MB**，包含 `node_modules`。引入大依赖前先评估。

### 5. 调试与日志

- 日志在云开发控制台**长期保留**且**所有协作者可见**
- 不能断点调试生产环境，本地调试用开发者工具的「云开发 → 本地调试」
- `cloud.openapi` 在本地 Node 环境无法直接调用，必须在云环境内运行

:::danger
云函数日志中**不要打印敏感数据**：openid、手机号、token、身份证号等。建议在日志输出前做脱敏。
:::

## 配置

### 1. 初始化

在小程序入口 `app.ts` 的 `onLaunch` 中初始化云开发：

```ts
// app.ts
App({
  onLaunch() {
    if (!wx.cloud) {
      console.error('请使用 2.2.3 或以上的基础库');
      return;
    }

    wx.cloud.init({
      env: 'your-env-id',       // 云环境 ID
      traceUser: true,          // 记录用户访问
    });
  },
});
```

:::warning[环境 ID]
`env` 不要写死在生产代码里。推荐用 `wx.getAccountInfoSync()` 的 `miniProgram.envVersion` 区分 `develop`/`trial`/`release`，分别对应不同云环境。
:::

### 2. 目录结构

云函数代码独立于小程序代码，放在项目根的 `cloudfunctions/`：

```
miniprogram/
├── src/                      # 小程序源码
│   ├── app.ts
│   ├── pages/
│   └── services/
│       ├── http/             # 自建后端调用
│       └── cloud/            # 云函数调用封装
│           ├── callFunction.ts
│           └── qrcode.ts
├── cloudfunctions/           # 云函数源码
│   ├── sendOrderNotify/
│   │   ├── index.js
│   │   └── package.json
│   └── genQrcode/
│       ├── index.js
│       └── config.json       # 定时触发器配置
└── project.config.json
```

### 3. project.config.json

声明云函数目录，让开发者工具识别：

```json
{
  "cloudfunctionRoot": "cloudfunctions/",
  "miniprogramRoot": "src/"
}
```

### 4. services/cloud 封装

所有云函数调用收口在 `services/cloud/`，与 `services/http/` 并列，详见 [云函数](../../cookbook/cloud-function/)。

## 部署与版本管理

| 操作 | 方式 |
|------|------|
| 上传单个云函数 | 开发者工具右键云函数目录 → 上传并部署 |
| 上传所有云函数 | 开发者工具 → 云开发 → 上传所有云函数 |
| CI 部署 | `tcb fn deploy <name> --envId <env>`（[CloudBase CLI](https://docs.cloudbase.net/cli/intro)） |

每个云函数上传后会生成**版本号**，可在云开发控制台回滚。生产环境推荐：

- 灰度发布：先上传到 `trial` 环境，验证后切到 `release`
- 别名管理：用版本别名，回滚时切换别名指向即可

:::warning[云函数与小程序代码不同步]
云函数部署是独立动作，不会随小程序上传一起发版。**小程序发版前必须先部署依赖的云函数**，否则线上会出现 `FunctionName parameter could not be found` 错误。
:::

## 计费

云函数按「调用次数 + 运行时长 + 资源使用」计费，与自建后端的「按服务器规格」模型不同：

| 项目 | 免费额度（基础版） | 超出后单价 |
|------|---------------|-----------|
| 调用次数 | 4 万次/月 | ¥0.0133 / 千次 |
| 运行时长 | 4 万 GB-秒/月 | ¥0.00017 / GB-秒 |
| 出网流量 | 5 GB/月 | ¥0.8 / GB |

**评估建议**：

- 日均调用 < 1 万次：基本免费额度内
- 高频场景（每次点击都触发云函数）：要算调用次数成本
- 长耗时任务（如视频处理）：运行时长费用会飙升

## 参见

- [云函数（官方）](https://developers.weixin.qq.com/miniprogram/dev/wxcloud/guide/functions.html)
- [云函数 API](https://developers.weixin.qq.com/miniprogram/dev/wxcloud/reference-sdk-api/functions/)
- [CloudBase CLI](https://docs.cloudbase.net/cli/intro)
- [云函数定时触发器](https://developers.weixin.qq.com/miniprogram/dev/wxcloud/guide/functions/timer.html)
