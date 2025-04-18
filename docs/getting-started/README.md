---
lang: zh-cmn-Hans-CN
title: 指南
description: 架构指南和总览
---

# 架构

:::tip
目前仅支持微信小程序，对于其他小程序平台暂时不予考虑。
:::

![前端架构](./assets/architecture.png?as=webp)

| 特性/平台          | 微信小程序                                                                   |
|----------------|-------------------------------------------------------------------------|
| 开发方式           | 原生微信小程序                                                                 |
| NPM 支持         | ✔️                                                                      |
| UI 组件/框架       | [TDesign MiniProgram](https://tdesign.tencent.com/miniprogram/overview) |
|                | [Vant Weapp](https://vant-ui.github.io/vant-weapp/)                     |
| TypeScript     | ☑️可选                                                                    |
| ESLint         | airbnb                                                                  |
| Sass           | ✔️                                                                      |
| stylelint      | Bootstrap                                                               |
| 单元测试           | 🚧工作进行中                                                                 |
|                | Jest + MiniProgram Simulate                                             |
| MiniProgram CI | ✔️                                                                      |
| 云开发            | ❌不支持                                                                    |

- 使用原生小程序开发方式，降低学习成本
- 支持 npm 使用第三方工具包
- 严格的代码检查工具，提升代码维护性：包括 ESLint、stylelint
- 提供 CI 支持自动化构建和上传
- 支持使用 TypeScrip 和 Sass，提供更好的开发体验
- 支持小程序自动化测试

:::warning
对于 UI 组件，推荐使用 [TDesign MiniProgram](https://tdesign.tencent.com/miniprogram/overview) 或 [Vant Weapp](https://vant-ui.github.io/vant-weapp/)，可以根据实际情况进行选择。
:::

## 技术运用

### 环境

- **Node.js**：用于运行命令和安装 npm 依赖
- **npm**：项目相关依赖包，同时提供命令行进行关联

:::tip 参见
- [npm 支持](https://developers.weixin.qq.com/miniprogram/dev/devtools/npm.html)
:::

### 基础技术

- **微信小程序**：微信小程序基础开发技术，包括 WXML、WXSS、WXS 等
- **ECMAScript 6**：简称 ES6，又称 ECMAScript 2015，后续版本随年份命名，是 JavaScript 的标准规范
- **TypeScript**：微软出品的编程语言，需要转化为 JS 执行，为 JS 提供静态类型和强类型

:::tip 参见
- [微信开发者文档](https://developers.weixin.qq.com/miniprogram/dev/framework/)
- [企业微信开发者文档](https://developer.work.weixin.qq.com/document/path/92455)
- [原生支持 TypeScript](https://developers.weixin.qq.com/miniprogram/dev/devtools/compilets.html)
:::

### 工具

- **ESLint**：JS 语法检查工具，避免一些编程时的错误，同时能让团队编程风格统一
- **Sass**：CSS 预处理器，为 CSS 提供编程能力
- **Jest**：单元测试工具，可以测试 JS 函数和小程序组件
- **MiniProgram CI**：提供自动化的小程序构建和上传功能

:::tip 参见
- [单元测试](https://developers.weixin.qq.com/miniprogram/dev/framework/custom-component/unit-test.html)
- [小程序 CI](https://developers.weixin.qq.com/miniprogram/dev/devtools/ci.html)
:::

### 小程序框架扩展

| 包名                                                                                           | 作用                           |
|----------------------------------------------------------------------------------------------|------------------------------|
| [miniprogram-api-promise](https://github.com/wechat-miniprogram/miniprogram-api-promise)     | 将小程序的 API 转化为 Promise        |
| [miniprogram-computed](https://github.com/wechat-miniprogram/computed)                       | 计算属性 computed 和监听器 watch 的实现 |
| [mobx-miniprogram](https://github.com/wechat-miniprogram/mobx)                               |                              |
| [mobx-miniprogram-bindings](https://github.com/wechat-miniprogram/mobx-miniprogram-bindings) | MobX 绑定辅助库，提供全局状态管理功能        |

### 小程序扩展组件

| 包名                                                                                          | 作用                           |
|---------------------------------------------------------------------------------------------|------------------------------|
| [wxml-to-canvas](https://github.com/wechat-miniprogram/wxml-to-canvas)    | 小程序内通过静态模板和样式绘制 canvas ，导出图片，可用于生成分享图等场景       |

### JS 库

| 包名                                 | 作用                              |
|------------------------------------|---------------------------------|
| [dayjs](https://day.js.org/zh-CN/) | 轻量化的时间/日期处理工具，提供时间日期格式化、计算操作等功能 |
| [lodash.*](https://lodash.com/)    | JS 工具函数集，提供诸如数据类型判断、转换、节流、防抖等函数 |
