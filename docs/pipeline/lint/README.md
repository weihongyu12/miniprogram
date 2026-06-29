---
description: 微信小程序 Lint 配置实践，ESLint 全局变量声明、stylelint 规则与 husky 集成
---

# Lint

ESLint、stylelint 的安装、husky + lint-staged 的 Git 工作流集成与 Web 项目完全一致，此处不再重复。本文档仅说明小程序场景下 Lint 配置的特有部分。

## ESLint：小程序全局变量

小程序运行时注入了一批全局 API（`wx`、`App`、`Page`、`Component` 等），需在 ESLint 配置中声明，避免 `no-undef` 报错：

```js
// .eslintrc.js
module.exports = {
  env: {
    browser: true,
    es2021: true,
    wx: true,        // 小程序环境
    jest: true,
  },
  globals: {
    wx: 'readonly',
    App: 'readonly',
    Page: 'readonly',
    Component: 'readonly',
    Behavior: 'readonly',
    getCurrentPages: 'readonly',
    getApp: 'readonly',
  },
};
```

## stylelint：WXSS 特有规则

小程序 WXSS 与 CSS 有差异，配置时需注意：

```js
// stylelint.config.js
module.exports = {
  rules: {
    // rpx 单位支持
    'unit-no-unknown': [true, { ignoreUnits: ['rpx'] }],
    // 小程序自定义组件样式
    'selector-type-no-unknown': [true, { ignoreTypes: ['page'] }],
  },
};
```

:::warning[WXSS 限制]
- 不支持 `*` 通配符
- 不支持属性选择器 `[attr]`（部分基础库支持）
- 单位支持 `rpx`、`px`、`vh`、`vw` 等
- 不支持 `@media` 的部分特性
:::

## lint-staged 配置

小程序的 lint-staged 配置需覆盖 `.wxml` 文件：

```js
// lint-staged.config.js
module.exports = {
  '*.{js,ts}': ['eslint --fix', 'prettier --write'],
  '*.{wxss,scss,css}': ['stylelint --fix', 'prettier --write'],
  '*.{wxml,json,md}': ['prettier --write'],
};
```

## 小程序特有规则建议

| 规则 | 错误级别 | 说明 |
|------|---------|------|
| `no-console` | warn（开发）/ error（生产） | 避免日志泄露 |
| `@typescript-eslint/no-floating-promises` | error | `wx.xxx` 异步 API 必须处理 Promise |
| `no-async-promise-executor` | error | 避免在 Promise 中使用 async |
