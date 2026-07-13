---
description: 微信小程序 npm 实践示例，以 dayjs 为例演示安装、构建与使用全流程
---

# npm

npm 与 Web 的差异、配置与使用限制见 [npm](../../reference/npm/)。

## 集成 dayjs 时间库

完整的"安装 → 构建 → 使用"流程，适用于任何第三方 npm 包。

### 1. 安装依赖

```bash
npm install --save dayjs
```

### 2. 构建 npm

在开发者工具中执行"工具 → 构建 npm"，或通过 CI 脚本：

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

构建后会生成 `src/miniprogram_npm/dayjs/`。

### 3. 在页面中使用

```ts
// src/pages/order/index.ts
import dayjs from 'dayjs';

Page({
  data: {
    createTime: '',
    relativeTime: '',
  },

  onLoad() {
    const now = dayjs();
    this.setData({
      createTime: now.format('YYYY-MM-DD HH:mm:ss'),
      relativeTime: now.fromNow(),
    });
  },
});
```

```xml
<!-- src/pages/order/index.wxml -->
<view>下单时间：{{createTime}}</view>
<view>相对时间：{{relativeTime}}</view>
```

:::warning
每次 `npm install` 升级 dayjs 后，必须重新"构建 npm"，否则 `miniprogram_npm/dayjs/` 还是旧版本。
:::

## 升级依赖后重新构建

在 `package.json` 中串联脚本，避免忘记构建：

```json
{
  "scripts": {
    "build:npm": "node scripts/build-npm.js"
  }
}
```

```bash
npm install dayjs@latest && npm run build:npm
```
