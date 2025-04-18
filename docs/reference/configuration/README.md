---
lang: zh-cmn-Hans-CN
title: 参考配置
description: 参考配置手册
---

# 参考配置

## 项目配置

```json
{
  "miniprogramRoot": "src/",
  "compileType": "miniprogram",
  "setting": {
    "es6": true,
    "enhance": true,
    "postcss": true,
    "minified": true,
    "minifyWXSS": true,
    "minifyWXML": true,
    "uglifyFileName": true,
    "ignoreUploadUnusedFiles": true,
    "autoAudits": true,
    "urlCheck": true,
    "compileHotReLoad": true,
    "preloadBackgroundData": false,
    "lazyloadPlaceholderEnable": false,
    "useStaticServer": false,
    "bigPackageSizeSupport": true,
    "babelSetting": {
      "ignore": [],
      "disablePlugins": [],
      "outputPath": ""
    },
    "useCompilerPlugins": [
      "typescript",
      "sass"
    ],
    "disableUseStrict": false,
    "uploadWithSourceMap": true,
    "packNpmManually": true,
    "packNpmRelationList": [
      {
        "packageJsonPath": "./package.json",
        "miniprogramNpmDistDir": "./src"
      }
    ],
    "coverView": true,
    "ignoreDevUnusedFiles": true,
    "checkInvalidKey": true,
    "showShadowRootInWxmlPanel": true,
    "useIsolateContext": true,
    "useMultiFrameRuntime": true,
    "useApiHook": true,
    "useApiHostProcess": true,
    "useLanDebug": false,
    "enableEngineNative": false,
    "showES6CompileOption": false,
    "newFeature": false,
    "nodeModules": true,
    "scopeDataCheck": false,
    "checkSiteMap": true,
    "userConfirmedBundleSwitch": false
  },
  "libVersion": "2.25.1",
  "appid": "wx768c7751fe2b26c3",
  "projectname": "众业达商城",
  "packOptions": {
    "ignore": [],
    "include": []
  }
}
```

## ESLint 配置

```bash
npm i eslint eslint-config-airbnb-base eslint-config-airbnb-typescript @typescript-eslint/parser @typescript-eslint/eslint-plugin eslint-plugin-import --save-dev
```

```js
// .eslintrc.js

const globals = {
  require: true,
  Page: true,
  wx: true,
  App: true,
  getApp: true,
  getCurrentPages: true,
  Component: true,
  getRegExp: true,
  Behavior: true,
};

module.exports = {
  parser: '@typescript-eslint/parser',
  extends: [
    'airbnb-base',
    'airbnb-typescript/base',
    'plugin:@typescript-eslint/recommended',
  ],
  globals,
};
```

## Jest 配置

```bash
npm i @babel/core @babel/preset-env babel-jest@27 jest@27 miniprogram-simulate --save-dev
```

```javascript
// jest.config.js

module.exports = {
  testEnvironment: 'jsdom',
  snapshotSerializers: ['miniprogram-simulate/jest-snapshot-plugin'],
};
```

```javascript
// babel.config.js

// 这里的 Babel 仅仅是为了 Jest 的运行，不会在小程序的构建中使用
module.exports = {
  presets: [
    [
      '@babel/preset-env',
      {
        targets: {
          node: true,
        },
      },
    ],
  ],
};
```
