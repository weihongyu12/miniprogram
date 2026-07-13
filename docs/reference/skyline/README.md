---
description: 微信小程序 Skyline 渲染引擎原理，与 WebView 渲染的差异及迁移指南
---

# Skyline

## 什么是 Skyline

Skyline 是微信自研的小程序渲染引擎，与传统的 WebView 渲染并存。它不再用 HTML/CSS 模拟，而是直接用客户端原生渲染管线绘制界面，目标是把小程序的渲染体验拉到接近原生 App 的水平。

## 为什么需要 Skyline

WebView 渲染在小程序场景下的痛点：

- 长列表滚动卡顿、白屏
- 动画掉帧（`setData → JS bridge → WebView 重绘` 链路较长）
- 复杂布局性能差

Skyline 的核心收益：

- **更流畅的滚动和动画**：worklet 动画在 UI 线程执行，不受 JS 阻塞
- **更快的首屏**：原生渲染管线，省去 WebView 初始化
- **更接近原生的交互**：手势系统、贴底弹层等
- **更精确的布局**：对 Flex、Grid 等现代布局支持更完整

## 渲染模式对比

| 对比项 | WebView | Skyline |
|--------|---------|---------|
| 渲染引擎 | 系统 WebView | 微信自研 |
| 性能 | 中等 | 高（更接近原生） |
| 组件支持 | 全部 | 大部分（少数不支持） |
| 样式支持 | 标准 CSS 子集 | 更严格的子集 + worklet |
| 动画 | setData 驱动 | worklet（UI 线程） |
| 兼容性 | 全版本 | 基础库 3.0.2+（Android/iOS）、3.11.3+（鸿蒙） |
| 是否推荐新项目 | 否 | 是 |

## 启用 Skyline

### 环境要求

- 微信客户端：8.0.40 或以上（Android/iOS）；鸿蒙 1.0.10+
- 基础库：3.0.2+（Android/iOS）；3.11.3+（鸿蒙）
- 开发者工具：Stable 1.06.2307260 或以上（建议 Nightly）

### 全局启用

`app.json`：

```json
{
  "lazyCodeLoading": "requiredComponents",
  "renderer": "skyline",
  "componentFramework": "glass-easel",
  "rendererOptions": {
    "skyline": {
      "defaultDisplayBlock": true,
      "defaultContentBox": true,
      "tagNameStyleIsolation": "legacy",
      "enableScrollViewAutoSize": true,
      "keyframeStyleIsolation": "legacy",
      "disableABTest": true
    }
  }
}
```

各字段含义：

| 字段 | 含义 |
|------|------|
| `renderer` | 渲染引擎：`skyline` 或 `webview` |
| `componentFramework` | 组件框架，必须为 `glass-easel` |
| `lazyCodeLoading` | 按需注入，必须为 `requiredComponents` |
| `defaultDisplayBlock` | inline 元素默认按 block 显示 |
| `defaultContentBox` | 默认使用 content-box 盒模型 |
| `disableABTest` | 关闭 We 分析的 A/B 实验，强制启用 Skyline |

### 页面级启用

也可只在某些页面用 Skyline，其他保留 WebView：

```json
// pages/home/index.json
{
  "renderer": "skyline",
  "componentFramework": "glass-easel"
}
```

:::warning[AB 实验]
默认情况下，即使配置了 Skyline，线上也会进入 We 分析的 A/B 实验，部分用户仍会落到 WebView。要强制启用 Skyline：

- 设置 `rendererOptions.skyline.disableABTest: true`，或
- 在 [We 分析](https://wedata.weixin.qq.com/) 配置白名单

开发期可用开发者工具模拟器左上角的"切换渲染模式"快速预览。
:::

### 开发者工具调试

切换 Skyline 后，需在 **详情 → 本地设置** 中勾选：

- ✅ 开启 Skyline 渲染调试
- ✅ 编译 worklet 代码（用到 worklet 动画时）

模拟器左上角会显示当前 renderer 为 `skyline`。

:::tip[热重载]
Skyline 暂不支持热重载，修改代码后需手动编译。
:::

## Skyline 的关键差异

### 1. WXSS 样式限制

Skyline 对 WXSS 的支持比 WebView 更严格，部分写法不兼容：

| 样式 | WebView | Skyline |
|------|---------|---------|
| `*` 通配符 | 支持 | ❌ 不支持 |
| 属性选择器 `[attr]` | 支持 | 部分支持 |
| `display: flex` | 支持 | 需配合 `defaultDisplayBlock` 或显式声明 |
| `position: sticky` | 支持 | 部分支持 |
| `backdrop-filter: blur` | 支持 | ❌ 不支持 |

详见 [Skyline WXSS 支持与差异](https://developers.weixin.qq.com/miniprogram/dev/framework/runtime/skyline/wxss.html)。

### 2. 基础组件差异

绝大多数基础组件在 Skyline 中支持，但部分组件的属性/事件有差异，少数组件不支持。详见 [Skyline 基础组件支持与差异](https://developers.weixin.qq.com/miniprogram/dev/framework/runtime/skyline/component.html)。

迁移前务必对照该表检查项目用到的组件。

### 3. worklet 动画

Skyline 引入 worklet 机制，动画在 UI 线程执行，不受 JS 阻塞。完整示例见 [Skyline](../../cookbook/skyline/)。

worklet 函数体内必须以 `'worklet';` 开头声明，且不能闭包捕获外部变量（除少量基本类型字面量）。

### 4. 手势系统

Skyline 提供了一组手势组件，弥补 WebView 中手势能力的不足：

- `worklet` 内置手势事件
- `scale-gesture-handler`、`pan-gesture-handler` 等手势组件

详见 [Skyline 手势系统](https://developers.weixin.qq.com/miniprogram/dev/framework/runtime/skyline/gesture.html)。

### 5. fallback 机制

在低版本微信或 PC 端，Skyline 会自动降级到 WebView。需保证页面在两种模式下都能正常显示。

:::tip[兼容性建议]
迁移到 Skyline 时，**不要一次性全量切换**。推荐路径：

1. 先在新建页面或独立分包中启用 Skyline
2. 验证通过后，逐步迁移其他页面
3. 全程通过开发者工具"切换渲染模式"对比两种模式下的表现
:::

## 何时使用 Skyline

### ✅ 推荐

- 新项目，从第一行代码就用 Skyline
- 长列表、瀑布流、复杂滚动场景
- 流畅动画要求高的页面（如商品详情、抽奖转盘）
- 视频弹幕、贴底弹层

### ⚠️ 谨慎

- 已有大型 WebView 项目：迁移成本高，需逐页适配
- 用到不支持的组件/样式：先评估替代方案
- PC 端为主的使用场景：fallback 到 WebView 时体验不一致

### ❌ 不推荐

- 强依赖 WebView-only 特性的页面（如某些 webview 组件嵌入 H5）
- 基础库版本低于 3.0.2 的旧客户端场景

## 迁移步骤（针对老项目）

1. **环境核查**：基础库版本、客户端版本、开发者工具版本是否达标
2. **组件/样式清单**：列出项目用到的所有组件和 WXSS 特性，对照 Skyline 差异表
3. **单页面试点**：选一个简单页面（如设置页）启用 Skyline，验证基本渲染
4. **逐步扩展**：按页面重要程度递增迁移，每迁一个页面都做 WebView/Skyline 双模验证
5. **真机预览**：通过开发者工具"快捷切换入口"在真机强制 Skyline 预览
6. **灰度发布**：在 We 分析配置 AB 实验，按比例放量

## 参见

- [Skyline 渲染引擎（官方）](https://developers.weixin.qq.com/miniprogram/dev/framework/runtime/skyline/)
- [从 WebView 迁移到 Skyline](https://developers.weixin.qq.com/miniprogram/dev/framework/runtime/skyline/migration/)
- [Skyline 更新日志](https://developers.weixin.qq.com/miniprogram/dev/framework/runtime/skyline/changelog.html)
- [Skyline 基础组件支持与差异](https://developers.weixin.qq.com/miniprogram/dev/framework/runtime/skyline/component.html)
- [Skyline](../../cookbook/skyline/)
