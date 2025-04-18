---
lang: zh-cmn-Hans-CN
title: 目录规范
description: 项目目录规范
---

# 目录规范

## 项目根目录

:::tip
通过在 `project.config.json` 设置 `miniprogramRoot` 可以改变源代码目录，并设置 `packNpmManually` 和 `packNpmRelationList` 改变 `miniprogram_npm` 的目录。
:::

```:no-line-numbers
project/
├── .husky/
├── scripts/
├── src/
│   └── ...
├── typings/
├── .eslintignore
├── .eslintrc.js
├── .gitignore
├── .stylelintignore
├── commitlint.config.js
├── lint-staged.config.js
├── package.json
├── package-lock.json
├── project.config.json
├── project.private.config.json
├── README.md
├── stylelint.config.js
└── tsconfig.json
```

- `.husky/`：Git Hooks 配置，参见 [husky](https://typicode.github.io/husky/)
- `scripts/`：构建和上传脚本
- `src/`：项目源代码
- `typings/`：TypeScript 类型代码，参见 [原生支持 TypeScript](https://developers.weixin.qq.com/miniprogram/dev/devtools/compilets.html)
- `.eslintignore`：忽略不需要 ESLint 检查的目录或文件
- `.eslintrc.js`：ESLint 配置
- `.gitignore`：忽略不需要 Git 提交的目录或文件
- `.stylelintignore`：忽略不需要 stylelint 检查的目录或文件
- `commitlint.config.js`：commitlint 配置
- `lint-staged.config.js`：Lint Staged 配置
- `package.json`
- `package-lock.json`
- `project.config.json`：小程序项目配置，参见 [项目配置文件](https://developers.weixin.qq.com/miniprogram/dev/devtools/projectconfig.html)
- `project.private.config.json`：本地小程序项目配置，请勿提交到 git 仓库
- `stylelint.config.js`：stylelint 配置
- `tsconfig.json`：TypeScript 配置

## `src` 目录

```:no-line-numbers
project/
└── src/
    ├── components/
    ├── custom-tab-bar/
    ├── miniprogram_npm/
    ├── packages/
    ├── pages/
    ├── styles/
    │   └── utilities.scss
    ├── utils/
    ├── app.js
    ├── app.json
    ├── app.wxss
    └── sitemap.json
```

- `components/`：项目公共组件
- `custom-tab-bar/`：自定义导航条代码，参见 [自定义 tabBar](https://developers.weixin.qq.com/miniprogram/dev/framework/ability/custom-tabbar.html)
- `miniprogram_npm/`：小程序构建的npm，参见 [npm 支持](https://developers.weixin.qq.com/miniprogram/dev/devtools/npm.html)
- `packages/`：分包代码
- `pages/`：主包页面代码
- `styles/`：全局样式文件，包括一个工具样式 `utilities.scss`
- `utils/`：工具JS函数
- `app.js`：小程序 App 实例，参见 [注册小程序](https://developers.weixin.qq.com/miniprogram/dev/framework/app-service/app.html)
- `app.json`：小程序全局配置，参见 [小程序全局配置](https://developers.weixin.qq.com/miniprogram/dev/framework/config.html#%E5%85%A8%E5%B1%80%E9%85%8D%E7%BD%AE)
- `app.wxss`：小程序全局样式，请谨慎使用
- `sitemap.json`：参见 [sitemap 配置](https://developers.weixin.qq.com/miniprogram/dev/framework/sitemap.html)
