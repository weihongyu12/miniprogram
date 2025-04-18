---
lang: zh-cmn-Hans-CN
title: Cookbook
---

# Cookbook

## 计算属性和侦听器

如果代码中，存在依赖响应式状态的复杂逻辑，可以使用 **计算属性（computed）** 和 **侦听器（watch）** 简化代码。

由于小程序本身不带计算属性（computed）和侦听器（watch）功能，需要通过 `miniprogram-computed` 引入相关的 behaviors，提供框架扩展支持。

```js{1,5,12-19,21-28}
import { behavior as computedBehavior } from 'miniprogram-computed';
import otherBehavior from './behavior/otherBehavior';

Component({
  behaviors: [otherBehavior, computedBehavior],
  data: {
    a: 1,
    b: 1,
    pow: 0,
  },

  // 计算属性
  computed: {
    sum(data) {
      // 注意： computed 函数中不能访问 this ，只有 data 对象可供访问
      // 这个函数的返回值会被设置到 this.data.sum 字段中
      return data.a + data.b + data.c // data.c 为自定义 behavior 数据段
    },
  },

  // 侦听器
  watch: {
    'a, b': function (a, b) {
      this.setData({
        pow: a ** b,
      })
    },
  },

  methods: {
    onTap() {
      this.setData({
        a: this.data.b,
        b: this.data.a + this.data.b,
      });
    },
  },
});
```

## Lodash 的使用

[Lodash](https://lodash.com/) 是一个 JS 工具库，可以在微信小程序中使用。

使用 Lodash 时，由于小程序不提供全局环境，需要手动 polyfill。

```js
// utlis/lodash-miniprogram-polyfill.js

global.Object = Object;
global.Array = Array;
global.DataView = DataView;
global.Date = Date;
global.Error = Error;
global.Float32Array = Float32Array;
global.Float64Array = Float64Array;
global.Function = Function;
global.Int8Array = Int8Array;
global.Int16Array = Int16Array;
global.Int32Array = Int32Array;
global.Map = Map;
global.Math = Math;
global.Promise = Promise;
global.RegExp = RegExp;
global.Set = Set;
global.String = String;
global.Symbol = Symbol;
global.TypeError = TypeError;
global.Uint8Array = Uint8Array;
global.Uint8ClampedArray = Uint8ClampedArray;
global.Uint16Array = Uint16Array;
global.Uint32Array = Uint32Array;
global.WeakMap = WeakMap;
global.clearTimeout = clearTimeout;
/* eslint-disable-next-line no-restricted-properties */
global.isFinite = Number.isFinite;
global.parseInt = parseInt;
global.setTimeout = setTimeout;
```

由于完整版的 Lodash 包非常大（构建后大概 400KB），并且微信小程序不支持 Tree-Shaking，因此不要使用全量版本的 Lodash

**使用 Lodash 的[独立方法包](https://lodash.com/per-method-packages)**，可以有效减少包的体积。

:::danger 反面例子 👎
缺少必要的 polyfill

```js
import throttle from 'lodash.throttle';
```

使用全量的 lodash 包

```js
import './utils/lodash-miniprogram-polyfill';
import { throttle } from 'lodash';
```
:::

:::tip 正面例子 👍
```js
import './utils/lodash-miniprogram-polyfill';
import throttle from 'lodash.throttle';
```
:::

注意，在使用中，有部分方法不能使用箭头函数，否则会改变 `this` 的指向

:::danger 反面例子 👎
使用箭头函数，会改变 `this` 的指向。

```js
import './utils/lodash-miniprogram-polyfill';
import throttle from 'lodash.throttle';

Page({
  onChange: throttle(() => {
    // ...
  }, 300),
});
```
:::

:::tip 正面例子 👍
使用 `function` 函数，不会改变 `this` 的指向。

```js
import './utils/lodash-miniprogram-polyfill';
import throttle from 'lodash.throttle';

Page({
  onChange: throttle(function () {
    // ...
  }, 300),
});
```
:::



