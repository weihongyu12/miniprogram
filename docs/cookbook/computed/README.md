---
description: miniprogram-computed 实践示例，computed 与 watch 的安装、引入与使用
---

# 计算属性

工作原理、限制与适用场景见 [计算属性](../../reference/computed/)。

## 安装与引入

```bash
npm install --save miniprogram-computed
```

然后"构建 npm"（见 [npm](../../reference/npm/)）。

## 基础 computed

声明依赖 `data` 的派生字段，依赖变化时自动重算：

```js
import { behavior as computedBehavior } from 'miniprogram-computed';

Component({
  behaviors: [computedBehavior],

  data: {
    a: 1,
    b: 1,
    // sum 不需要手动声明，computed 会自动写入 this.data.sum
  },

  computed: {
    sum(data) {
      // 注意：computed 函数中不能访问 this，只有 data 对象可供访问
      // 返回值会被设置到 this.data.sum
      return data.a + data.b;
    },
  },

  methods: {
    onTap() {
      this.setData({
        a: this.data.b,
        b: this.data.a + this.data.b,
      });
      // sum 会自动更新，无需手动 setData
    },
  },
});
```

```xml
<!-- sum 自动更新 -->
<view>a + b = {{sum}}</view>
```

## 侦听器 watch

监听字段变化执行副作用。watch 中**可以访问 this**：

```js
import { behavior as computedBehavior } from 'miniprogram-computed';

Component({
  behaviors: [computedBehavior],

  data: {
    a: 1,
    b: 1,
    pow: 0,
  },

  watch: {
    // 多字段：a 或 b 变化都触发，按顺序接收新值
    'a, b': function (a, b) {
      this.setData({
        pow: a ** b,
      });
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

## 结合 properties 的 computed

properties 传入的字段也会出现在 computed 的 `data` 参数中，可直接使用：

```js
import { behavior as computedBehavior } from 'miniprogram-computed';

Component({
  behaviors: [computedBehavior],

  properties: {
    price: { type: Number, value: 0 },
    count: { type: Number, value: 1 },
  },

  data: {
    discount: 1, // 折扣，组件内部维护
  },

  computed: {
    // 通过 data 访问 properties 与 data 字段
    total(data) {
      return data.price * data.count * data.discount;
    },
  },

  methods: {
    onDiscountChange(e) {
      this.setData({ discount: e.detail.value });
      // total 会自动重算
    },
  },
});
```

## TypeScript 写法

miniprogram-computed 自带类型声明，TS 下直接使用即可。为 computed 显式标注返回类型更清晰：

```ts
import { behavior as computedBehavior } from 'miniprogram-computed';

interface Data {
  a: number;
  b: number;
}

interface Properties {
  price: number;
  count: number;
}

Component<Data, Properties>({
  behaviors: [computedBehavior],

  properties: {
    price: { type: Number, value: 0 },
    count: { type: Number, value: 1 },
  },

  data: {
    a: 1,
    b: 1,
  } as Data,

  computed: {
    sum(data: Data & Properties): number {
      return data.a + data.b;
    },
    total(data: Data & Properties): number {
      return data.price * data.count;
    },
  },

  methods: {
    onTap() {
      this.setData({ a: this.data.a + 1 });
    },
  },
});
```

:::tip[类型合并]
computed 的 `data` 参数包含 `data` 与 `properties` 的字段，类型可标注为 `Data & Properties`，便于在函数内访问两者。
:::
