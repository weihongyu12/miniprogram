---
description: 微信小程序 Skyline 实践示例，页面启用、组件适配与常见问题排查
---

# Skyline

Skyline 工作原理、渲染差异与迁移指南见 [Skyline](../../reference/skyline/)。

## 在页面启用 Skyline

页面配置中指定渲染引擎：

```json
// pages/scroll/scroll.json
{
  "renderer": "skyline",
  "componentFramework": "glass-easel"
}
```

全局启用则在 `app.json` 中配置：

```json
{
  "lazyCodeLoading": "requiredComponents",
  "renderer": "skyline",
  "componentFramework": "glass-easel",
  "rendererOptions": {
    "skyline": {
      "defaultDisplayBlock": true,
      "defaultContentBox": true,
      "disableABTest": true
    }
  }
}
```

## worklet 滚动驱动动画

通过 `scroll-view` 的滚动位置驱动元素缩放，全程在 UI 线程执行，不受 JS 阻塞：

```xml
<!-- pages/scroll/scroll.wxml -->
<scroll-view scroll-y id="scroller" style="height: 100vh;">
  <view class="header" style="transform: scale({{scale}});">
    <text>滚动时缩放</text>
  </view>
  <view style="height: 2000rpx;"></view>
</scroll-view>
```

```ts
// pages/scroll/scroll.ts
Page({
  data: {
    scale: 1,
  },

  onLoad() {
    wx.createSelectorQuery()
      .select('#scroller')
      .node()
      .exec((res) => {
        const scroller = res[0].node;
        // 把滚动位置驱动到 scale，全程 UI 线程，不掉帧
        scroller.applyAnimationStyle({
          scrollY: (val) => {
            'worklet';
            return { scale: 1 - Math.min(val / 200, 0.5) };
          },
        });
      });
  },
});
```

:::warning[worklet 声明]
worklet 函数体必须以 `'worklet';` 开头声明，且不能闭包捕获外部变量（除少量基本类型字面量）。开发者工具中需在"详情 → 本地设置"勾选"编译 worklet 代码"。
:::

## 页面配置对应的 WXSS

Skyline 下样式支持更严格，`*` 通配符不支持，需显式声明类名：

```scss
/* pages/scroll/scroll.wxss */
.header {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 200rpx;
  background: #07c160;
}
```
