---
description: 微信小程序 npm 支持原理，与 Web npm 的差异、构建方式与使用限制
---

# npm

## 小程序 npm 与 Web npm 的差异

Web 项目中，`npm install` 后直接 `import` 即可，打包工具会处理依赖。

小程序不是浏览器环境，没有 webpack 这层打包。微信官方提供的“[npm 支持](https://developers.weixin.qq.com/miniprogram/dev/devtools/npm.html)”是一个**受控的 npm 子集**：

| 对比项 | Web | 小程序 |
|--------|-----|--------|
| 依赖位置 | `node_modules/` | `miniprogram_npm/`（构建产物） |
| 包格式 | 标准 npm | 标准 npm，但**不能含原生模块**（如 `fs`、`child_process`） |
| Tree-shaking | 支持 | ❌ 不支持（整包打入） |
| 构建方式 | 打包工具自动 | 需在开发者工具或 CI 中"构建 npm" |
| 包大小 | 无限制 | 受小程序包大小限制（主包 2MB，总包 20MB） |

:::warning
小程序的“构建 npm” ≠ Web 的“打包构建”。它只是把 `node_modules/` 中的包按小程序格式整理到 `miniprogram_npm/`，**不会做 Tree-shaking、压缩或转译**。
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

- `packNpmManually: true`：手动指定 `package.json` 与产物目录
- `packageJsonPath`：项目根的 `package.json`
- `miniprogramNpmDistDir`：构建产物 `miniprogram_npm/` 输出位置（一般是 `src/`）

### 2. package.json

正常声明依赖即可：

```json
{
  "dependencies": {
    "dayjs": "^1.11.21",
    "miniprogram-computed": "^8.0.0",
    "mobx-miniprogram": "^6.12.3",
    "mobx-miniprogram-bindings": "^6.0.0"
  },
  "devDependencies": {
    "miniprogram-api-typings": "^5.2.1",
    "miniprogram-automator": "^1.0.0",
    "miniprogram-ci": "^2.1.31",
    "miniprogram-simulate": "^1.6.1",
    "jest": "^30.0.0",
    "eslint": "^8.57.1",
    "eslint-config-airbnb-base": "^19.0.4",
    "stylelint": "^17.13.0",
    "stylelint-config-twbs-bootstrap": "^16.1.0",
    "husky": "^9.0.0",
    "lint-staged": "^17.0.0"
  }
}
```

### 3. 构建 npm

#### 方式 A：开发者工具

工具栏 → 工具 → 构建 npm。完成后会生成 `src/miniprogram_npm/`。

#### 方式 B：miniprogram-ci（CI/脚本）

```js
// scripts/build-npm.mjs
import ci from 'miniprogram-ci';
import path from 'node:path';
import { fileURLToPath } from 'node:url';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

const result = await ci.packNpmManually({
  packageJsonPath: path.resolve(__dirname, '../package.json'),
  miniprogramNpmDistDir: path.resolve(__dirname, '../src/'),
  ignores: ['miniprogram-ci'],
});
console.log('pack done:', result);
```

```bash
node scripts/build-npm.mjs
```

## 使用 npm 包

构建完成后，代码中直接 `require` / `import`：

```ts
// src/pages/home/index.ts
import dayjs from 'dayjs';

Page({
  onLoad() {
    const now = dayjs().format('YYYY-MM-DD');
    console.log(now);
  },
});
```

:::warning
小程序不会自动找 `node_modules/`，只识别 `miniprogram_npm/`。如果忘了构建 npm，运行时会报 `Cannot find module 'xxx'`。
:::

## npm 使用的限制与建议

### 1. 严格控制包大小

小程序无 Tree-shaking，依赖会整包打包：

- 优先用独立方法包（如 `lodash.throttle`）或轻量库（如 `dayjs`）
- 大体积库建议放入分包

### 2. 禁用原生 Node 模块

不支持 `fs`、`path`、`child_process` 等原生模块；引入前需检查依赖。

### 3. 需手动 Polyfill 全局对象

小程序无 `global`、`window` 等对象；若第三方库存在依赖，需在使用前自行注入全局变量并引入：

```js
// utils/polyfill.js
global.Object = Object;
// 注入后在入口文件处 import './utils/polyfill' 即可
```

### 4. 依赖变更需重新构建

每次执行 `npm install` 升级依赖后，必须重新“构建 npm”才能生效；建议配置 npm scripts 简化流程。

### 5. 勿将 miniprogram_npm 提交 Git

为保持代码仓库整洁，建议在 `.gitignore` 中加入构建产物，并在本地微信开发者工具或者 CI 环节中通过脚本自动重新构建：

```gitignore
src/miniprogram_npm/
```

## 参见

- [npm 支持](https://developers.weixin.qq.com/miniprogram/dev/devtools/npm.html)
- [miniprogram-ci packNpm](https://developers.weixin.qq.com/miniprogram/dev/devtools/ci.html)
- [CI/CD](../../pipeline/ci/)
