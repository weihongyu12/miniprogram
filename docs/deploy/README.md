---
lang: zh-cmn-Hans-CN
title: 部署
---

# 部署

:::tip
小程序的上传和发布推荐使用 [MiniProgram CI](https://developers.weixin.qq.com/miniprogram/dev/devtools/ci.html)。
:::

## 命令行配置

```json
{
  "scripts": {
    "build": "node scripts/build.js",
    "deploy": "node scripts/deploy.js"
  },
  "devDependencies": {
    "miniprogram-ci": "^1.8.35"
  }
}
```

```bash
# 构建 npm
npm run build

# 上传代码
npm run deploy
```

## 构建 npm

```js
// scripts/build.js

const path = require('path');
const ci = require('miniprogram-ci');

(async () => {
  const packResult = await ci.packNpmManually({
    packageJsonPath: path.join(__dirname, '../package.json'),
    miniprogramNpmDistDir: path.join(__dirname, '../src/'),
    ignores: [],
  });

  console.log('pack done, packResult:', packResult);
})();
```

## 上传代码

```js
// scripts/deploy.js

const path = require('path');
const ci = require('miniprogram-ci');

(async () => {
  const project = new ci.Project({
    appid: '<YOUR_APPID>',
    projectPath: path.join(__dirname, '../'),
    privateKeyPath: '<YOUR_PROJECT_KEY>',
    type: 'miniProgram',
    ignores: ['node_modules/**/*'],
  });

  const uploadResult = await ci.upload({
    project,
    version: '<VERSION>',
    desc: '修复了一些已知问题',
    setting: {
      es6: true,
      es7: true,
      disableUseStrict: false,
      minify: true,
      codeProtect: true,
      autoPrefixWXSS: true,
    },
    onProgressUpdate: console.log,
  });

  console.log(uploadResult);
})();
```

:::tip
上传密钥可以在 “[微信公众平台](https://mp.weixin.qq.com/) - 开发 - 开发设置” 获取，并建议设置上传白名单 IP
:::
