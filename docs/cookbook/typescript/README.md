---
description: 微信小程序 TypeScript 实践示例，Page、Component、App、Behavior 的类型写法
---

# TypeScript

工作原理、项目配置与常见陷阱见 [TypeScript](../../reference/typescript/)。

## Page

`Page<PageData>` 泛型约束 `data` 字段，`this.data` 自动推断类型。

```ts
// pages/profile/profile.ts
interface PageData {
  userInfo: WechatMiniprogram.UserInfo | null;
  loading: boolean;
}

Page<PageData>({
  data: {
    userInfo: null,
    loading: false,
  },

  async onLoad() {
    this.setData({ loading: true });
    try {
      const { userInfo } = await wx.getUserProfile({ desc: '用于完善资料' });
      this.setData({ userInfo });
    } finally {
      this.setData({ loading: false });
    }
  },
});
```

## Component

`Component` 接受两个泛型：`<Data, Properties>`。

```ts
// components/counter/counter.ts
interface Data {
  count: number;
}

interface Properties {
  step: number;
  initial: number;
}

Component<Data, Properties>({
  properties: {
    step: { type: Number, value: 1 },
    initial: { type: Number, value: 0 },
  },

  data: {
    count: 0,
  },

  lifetimes: {
    attached() {
      this.setData({ count: this.properties.initial });
    },
  },

  methods: {
    onIncrement() {
      this.setData({ count: this.data.count + this.properties.step });
    },
  },
});
```

## App

获取 app 实例时需要类型断言：

```ts
// app.ts
interface AppOption {
  globalData: {
    userInfo?: WechatMiniprogram.UserInfo;
    systemInfo?: WechatMiniprogram.SystemInfo;
  };
}

App<AppOption>({
  globalData: {},

  onLaunch() {
    const systemInfo = wx.getSystemInfoSync();
    this.globalData.systemInfo = systemInfo;
  },
});
```

```ts
// pages/home/home.ts
const app = getApp<AppOption>();
const systemInfo = app.globalData.systemInfo;  // 类型推断为 SystemInfo | undefined
```

## Behavior

```ts
// behaviors/share.ts
interface Data {
  shareTitle: string;
}

export const shareBehavior = Behavior<Data>({
  data: {
    shareTitle: '默认分享标题',
  },

  methods: {
    onShareAppMessage() {
      return {
        title: this.data.shareTitle,
        path: '/pages/home/home',
      };
    },
  },
});
```
