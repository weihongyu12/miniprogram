---
description: 微信小程序 npm 支持原理，与 Web npm 的差异、构建方式与使用限制
---

# npm

## 小程序 npm 与 Web npm 的差异

Web 项目中，`npm install` 后直接 `import` 即可，打包工具（webpack/vite）会处理依赖。

小程序不是浏览器环境，没有 webpack 这层打包。微信官方提供的"npm 支持"是一个**受控的 npm 子集**：

| 对比项 | Web | 小程序 |
|--------|-----|--------|
| 依赖位置 | `node_modules/` | `miniprogram_npm/`（构建产物） |
| 包格式 | 标准 npm | 标准 npm，但**不能含原生模块**（如 `fs`、`child_process`） |
| Tree-shaking | 支持 | ❌ 不支持（整包打入） |
| 构建方式 | 打包工具自动 | 需在开发者工具或 CI 中"构建 npm" |
| 包大小 | 无限制 | 受小程序包大小限制（主包 2MB，总包 20MB） |

:::warning
小程序的"构建 npm" ≠ Web 的"打包构建"。它只是把 `node_modules/` 中的包按小程序格式整理到 `miniprogram_npm/`，**不会做 Tree-shaking、压缩或转译**。
:::

## 配置

### 1. project.config.json

```json
{
  "setting": {
    "packNpmManually": true,
    "packNpmRelationList": [
      {
        "packageJsonPath": "./package.json",
        "miniprogramNpmDistDir": "./src"
      }
    ]
  }
}
```

- `packNpmManually: true`：手动指定 package.json 与产物目录
- `packageJsonPath`：项目根的 `package.json`
- `miniprogramNpmDistDir`：构建产物 `miniprogram_npm/` 输出位置（一般是 `src/`）

### 2. package.json

正常声明依赖即可：

```json
{
  "dependencies": {
    "dayjs": "^1.11.10",
    "mobx-miniprogram": "^6.12.0",
    "mobx-miniprogram-bindings": "^6.0.0"
  },
  "devDependencies": {
    "miniprogram-api-typings": "^4.0.0",
    "miniprogram-simulate": "^1.6.0",
    "miniprogram-automator": "^1.0.0",
    "miniprogram-ci": "^1.9.0"
  }
}
```

### 3. 构建 npm

#### 方式 A：开发者工具

工具栏 → 工具 → 构建 npm。完成后会生成 `src/miniprogram_npm/`。

#### 方式 B：miniprogram-ci（CI/脚本）

```js
// scripts/build-npm.js
const path = require('path');
const ci = require('miniprogram-ci');

(async () => {
  const result = await ci.packNpmManually({
    packageJsonPath: path.join(__dirname, '../package.json'),
    miniprogramNpmDistDir: path.join(__dirname, '../src/'),
    ignores: ['miniprogram-ci'],
  });
  console.log('pack done:', result);
})();
```

```bash
node scripts/build-npm.js
```

## 使用 npm 包

构建完成后，代码中直接 `require` / `import`：

```ts
// src/pages/home/home.ts
import dayjs from 'dayjs';

Page({
  onLoad() {
    const now = dayjs().format('YYYY-MM-DD');
    console.log(now);
  },
});
```

:::warning[路径]
小程序不会自动找 `node_modules/`，**只识别 `miniprogram_npm/`**。如果忘了构建 npm，运行时会报 `Cannot find module 'xxx'`。
:::

## npm 使用的限制与建议

### 1. 包大小要克制

小程序不支持 Tree-shaking，整包打入。一个全量 lodash（~400KB）会让主包直接超限。

**建议**：

- 用独立方法包：`npm i lodash.throttle`，而不是 `npm i lodash`
- 优先选轻量库：`dayjs` 替代 `moment`
- 体积大的库放分包

### 2. 不能用原生 Node 模块

`fs`、`path`、`child_process`、`crypto`（Node 版）等都不可用。引入前看包的 dependencies。

### 3. 全局对象 polyfill

部分库依赖 `global`、`window`、`document` 等全局对象，小程序运行时没有。需手动 polyfill：

```js
// utils/polyfill.js
global.Object = Object;
global.Array = Array;
global.Promise = Promise;
// ...
```

在使用前引入：

```js
import './utils/polyfill';
import throttle from 'lodash.throttle';
```

### 4. 升级依赖要重新构建

每次 `npm install` 升级依赖后，必须重新跑"构建 npm"，否则 `miniprogram_npm/` 还是旧版本。

可在 `package.json` 加 script 串起来：

```json
{
  "scripts": {
    "build:npm": "node scripts/build-npm.js"
  }
}
```

### 5. miniprogram_npm 是否提交 git

两种做法：

| 做法 | 优点 | 缺点 |
|------|------|------|
| 提交 | clone 后立即可跑，无需构建 | 仓库体积大，依赖版本与代码可能不同步 |
| 不提交 | 仓库干净 | clone 后必须执行构建 npm |

**推荐不提交**，在 `.gitignore` 中加入：

```
src/miniprogram_npm/
```

CI 中通过 `npm run build:npm` 重新构建。

## 参见

- [npm 支持（官方）](https://developers.weixin.qq.com/miniprogram/dev/devtools/npm.html)
- [miniprogram-ci packNpm](https://developers.weixin.qq.com/miniprogram/dev/devtools/ci.html)
- [CI/CD](../../pipeline/ci/)
