---
sidebar_position: 2
description: 微信小程序注释规范，涵盖 WXML、WXSS 注释及小程序场景下的注释实践
---

# 注释规范

JavaScript / TypeScript 的注释规范（单行、多行、JSDoc 函数注释）与 Web 项目完全一致，此处不再重复。本文档仅说明小程序特有的 WXML、WXSS 注释规范，以及小程序场景下的注释实践。

## WXML 注释规范

WXML 中的注释使用 `<!-- -->`，注释内容前后各留一个空格。

```html
<!-- 这是一个 WXML 注释 -->
<view class="container">
  <!-- 用户信息展示区 -->
  <text>{{userInfo.name}}</text>
</view>
```

:::warning[注意]
WXML 注释会被打包进产物，**不要在注释中写入敏感信息**（如密钥、内部接口路径）。
:::

## WXSS 注释规范

WXSS 注释使用 `/* */`，与 CSS 一致。

```scss
/* 全局变量 */
$primary-color: #07c160;

/* 按钮基础样式 */
.button {
  /* 主按钮 */
  &--primary {
    background-color: $primary-color;
  }
}
```

## 小程序特有注释场景

除通用注释场景外，小程序开发中以下场景特别需要注释：

### 平台兼容性 Hack

小程序在不同基础库版本、不同平台（iOS / Android / 开发者工具）上行为可能不一致，遇到兼容性问题时必须注释说明：

```javascript
// HACK: iOS 下 wx.previewImage 在 8.0.30 版本存在闪退，降级使用 webview 预览
```

### 平台 API 限制说明

```javascript
// NOTE: wx.setStorage 单个 key 上限 1MB，总容量 10MB，大文件需分片或直传 OSS
```

### 废弃 API 替换标记

```javascript
// TODO: wx.getUserInfo 已废弃，待迁移到 button open-type="chooseAvatar" + input type="nickname"
```
