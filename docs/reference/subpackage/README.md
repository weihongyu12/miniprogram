---
description: 微信小程序分包机制，包大小限制、分包配置、独立分包、预下载与分包异步化的原理与最佳实践
---

# 分包

## 为什么需要分包

小程序的代码包在启动时由微信下载并注入。如果把所有页面、组件、npm 产物都塞进主包，会带来两个问题：

- **包体积超限**：主包超过 2MB 无法上传，整个小程序所有分包总和也有上限
- **启动慢**：主包越大，首次启动下载、初始化耗时越长，影响首屏体验

分包把不常用的页面按业务模块拆出，**首次启动只下载主包，分包按需下载**。这是控制小程序体积和启动性能的核心手段。

## 包大小限制

| 维度 | 限制 |
|------|------|
| 单个主包 / 分包 | 2MB |
| 整个小程序所有包总和 | 20MB（开通 `bigPackageSizeSupport` 后可放宽到 30MB） |

`bigPackageSizeSupport` 在[参考配置](../configuration/)的 `project.config.json` 中开启。**这只是允许上传，不会改变运行时的下载策略**，启动速度仍要靠合理拆分包来保证。

## 配置分包

在 `app.json` 中通过 `subPackages` 声明分包：

```json
{
  "pages": [
    "pages/home/index",
    "pages/login/index"
  ],
  "subPackages": [
    {
      "root": "packages/order",
      "name": "order",
      "pages": [
        "pages/list/index",
        "pages/detail/index"
      ]
    },
    {
      "root": "packages/marketing",
      "name": "marketing",
      "pages": [
        "pages/coupon/index"
      ]
    }
  ]
}
```

| 字段 | 含义 |
|------|------|
| `root` | 分包根目录（相对 `src/`） |
| `name` | 分包别名，用于预下载配置 |
| `pages` | 分包页面路径，相对 `root`，**不要带 `root` 前缀** |

对应目录结构：

```
src/
├── pages/                    # 主包页面
├── packages/                 # 所有分包根
│   ├── order/
│   │   └── pages/
│   │       ├── list/
│   │       └── detail/
│   └── marketing/
│       └── pages/
│           └── coupon/
└── app.json
```

:::warning[路径易错点]
分包页面的最终访问路径是 `/${root}/${page}`，例如 `packages/order/pages/list/index`。

- 在 `app.json` 的 `subPackages[].pages` 中**不要写 `root` 前缀**
- 在 `wx.navigateTo` 等 API 调用中**必须带 `root` 前缀**
:::

## 分包页面访问

分包页面与主包页面在使用上没有差别，跳转时使用完整路径即可：

```ts
// 主包页面跳转到分包页面
wx.navigateTo({ url: '/packages/order/pages/list/index' });

// 分包页面之间互相跳转
wx.navigateTo({ url: '/packages/marketing/pages/coupon/index' });

// 分包页面跳回主包页面
wx.switchTab({ url: '/pages/home/index' });
```

:::warning
`tabBar` 中配置的页面必须在主包中，**分包页面不能作为 `tabBar` 页面**。
:::

## 分包预下载

默认情况下，分包在用户首次进入该分包内页面时才下载。可以通过 `preloadRule` 配置预下载，让分包提前下载，进入时无网络等待：

```json
{
  "preloadRule": {
    "pages/home/index": {
      "network": "all",
      "packages": ["order", "marketing"]
    },
    "pages/login/index": {
      "network": "wifi",
      "packages": ["__APP__"]
    }
  }
}
```

| 字段 | 含义 |
|------|------|
| `network` | `all`（所有网络）或 `wifi`（仅 Wi-Fi） |
| `packages` | 预下载的分包 `name` 列表；`"__APP__"` 表示主包 |

:::tip[预下载策略]
- 同一页面可配置预下载多个分包
- 预下载大小总和**不超过 2MB**，否则不会触发预下载
- 推荐在用户高频入口（首页、tabBar 页）配置预下载，提前拉取关联分包
:::

## 独立分包

独立分包是特殊类型的分包，**不依赖主包即可运行**，能在主包未下载完时直接打开。

### 与普通分包的差异

| 维度 | 普通分包 | 独立分包 |
|------|----------|----------|
| 启动依赖 | 必须先加载主包 | 可独立启动 |
| 访问主包资源 | 可访问 | ❌ 不能访问 |
| App 实例 | 共享主包 | 独立（需自行处理 `getApp()`） |
| 全局数据 | 共享 `globalData` | 不共享 |
| 适用场景 | 普通业务模块 | 启动入口页、活动页、推广落地页 |

### 配置

在分包配置中加 `"independent": true`：

```json
{
  "subPackages": [
    {
      "root": "packages/promo",
      "name": "promo",
      "pages": ["pages/landing/index"],
      "independent": true
    }
  ]
}
```

### 限制

独立分包中：

- **不能 `require` / `import` 主包的任何代码**（包括 `services/`、`utils/`、`components/`）
- `getApp()` 可能返回 `undefined`，需做容错或使用 `wx.getEnterOptionsSync()` 拿启动参数
- 主包 `App.onLaunch` 不会执行，需在独立分包页面 `onLoad` 中自行初始化

```ts
// packages/promo/pages/landing/index.ts
Page({
  onLoad() {
    // 独立分包中 getApp() 可能为 undefined
    const app = getApp();
    if (app?.globalData?.token) {
      // 已登录态
    } else {
      // 走独立分包自己的初始化逻辑
    }
  },
});
```

:::warning[与项目架构的关系]
本项目遵循 pages → services 两层架构。独立分包**无法引用主包 `services/`**，因此独立分包内的业务流程需要在分包内部重新组织 services，或仅做轻量展示类页面（无复杂业务流程）。

如果独立分包需要复用主包业务能力（如登录、请求封装），应将这些能力下沉到分包内独立实现，**不要试图通过动态 `require` 绕过限制**。
:::

## 分包异步化

默认情况下，分包之间不能互相 `require`，分包也不能引用主包代码。基础库 2.11.1+ 引入「分包异步化」机制，支持：

- 分包引用主包代码
- 分包之间互相引用
- 主包引用分包代码

通过 `require.async` 异步加载：

```ts
// packages/order/pages/list/index.ts
Page({
  async onLoad() {
    // 异步引用主包的 services
    const { request } = await require.async('@/services/http');
    const data = await request({ url: '/api/orders' });
  },
});
```

并在 `app.json` 中通过 `subPackages` 的 `plugins` / 依赖声明配置（详见官方文档）。**分包异步化主要解决跨包复用问题，不是性能优化手段**，能不拆就尽量不拆。

## 分包拆分原则

### ✅ 推荐拆分

- **按业务域拆分**：订单、营销、用户中心、活动页等高内聚模块独立成包
- **大体积资源拆分**：富文本编辑器、图表库、地图等大依赖放在分包中
- **低频访问页面拆分**：设置页、帮助页、协议页等非首屏页面

### ⚠️ 谨慎

- **频繁互相跳转的模块**：分包间跳转有下载等待，体验不如主包内跳转
- **共享大量组件/服务的模块**：拆分后需要重复实现或用分包异步化，得不偿失
- **tabBar 相关页面**：必须在主包，无法拆分

### ❌ 不推荐

- **按文件类型拆分**（如「所有页面一包、所有组件一包」），破坏业务内聚
- **把 services 拆到分包**：services 是业务编排层，与页面强绑定，跨包引用会引入分包异步化复杂度

## 常见陷阱

### 1. 分包页面路径写错

```ts
// ❌ 在 app.json 的 subPackages.pages 中带了 root 前缀
{ "root": "packages/order", "pages": ["packages/order/pages/list/index"] }

// ✅ pages 相对 root
{ "root": "packages/order", "pages": ["pages/list/index"] }

// ❌ 跳转时漏了 root 前缀
wx.navigateTo({ url: '/pages/list/index' })

// ✅ 跳转用完整路径
wx.navigateTo({ url: '/packages/order/pages/list/index' })
```

### 2. 分包体积超限未察觉

开发者工具不会在编码期提示分包体积，**只有上传时才报错**。建议在 CI 中用 `miniprogram-ci` 的 `getCompiledResult` 检查分包大小（详见 [CI/CD](../../pipeline/ci/)）。

### 3. 主包过大未拆分

主包超 1.5MB 就要警惕。优先把非首屏页面、大体积 npm 包移入分包，并把分包内的 npm 依赖通过 `packNpmManually` 配置在分包内构建（见 [npm](../npm/)）。

### 4. 独立分包中误用主包资源

独立分包 `require` 主包代码会直接报错。任何对 `@/services/*`、`@/utils/*`、`@/components/*` 的引用都要确认分包类型。

### 5. 预下载过多导致首次启动变慢

`preloadRule` 配置的预下载总量过大，会拖慢主包启动后的网络请求队列。优先只预下载「下一步大概率进入」的分包。

## 参见

- [使用分包（官方）](https://developers.weixin.qq.com/miniprogram/dev/framework/subpackages/basic.html)
- [独立分包](https://developers.weixin.qq.com/miniprogram/dev/framework/subpackages/independent.html)
- [分包预下载](https://developers.weixin.qq.com/miniprogram/dev/framework/subpackages/preload.html)
- [分包异步化](https://developers.weixin.qq.com/miniprogram/dev/framework/subpackages/async.html)
- [参考配置](../configuration/)
- [npm](../npm/)
- [分层架构](../../getting-started/#分层架构)
- [CI/CD](../../pipeline/ci/)
