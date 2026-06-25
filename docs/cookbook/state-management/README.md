---
description: mobx-miniprogram 实践示例，Store 定义与 Page/Component 绑定
---

# 状态管理

状态管理原理、与 services 层关系与最佳实践见 [状态管理](../../reference/state-management/)。

## 定义 Store

Store 放在 `services/store/`，与分层架构一致：

```ts
// services/store/user.ts
import { observable, action } from 'mobx-miniprogram';

interface UserInfo {
  id: string;
  nickname: string;
  avatar: string;
}

export const userStore = observable({
  // 状态
  token: '' as string,
  userInfo: null as UserInfo | null,

  // 计算属性
  get isLoggedIn() {
    return !!this.token;
  },

  // action：所有状态修改必须经过 action
  setToken: action(function (this: any, token: string) {
    this.token = token;
  }),

  setUserInfo: action(function (this: any, userInfo: UserInfo) {
    this.userInfo = userInfo;
  }),

  clear: action(function (this: any) {
    this.token = '';
    this.userInfo = null;
  }),
});
```

## 绑定到 Component（behavior 绑定）

`storeBindingsBehavior` 适用于 `Component` 构造器：

```ts
// components/user-card/user-card.ts
import { storeBindingsBehavior } from 'mobx-miniprogram-bindings';
import { userStore } from '@/services/store/user';

Component({
  behaviors: [storeBindingsBehavior],

  storeBindings: {
    store: userStore,
    fields: {
      token: 'token',
      userInfo: 'userInfo',
      isLoggedIn: 'isLoggedIn',
    },
    actions: {
      clearUser: 'clear',
    },
  },

  methods: {
    onLogout() {
      this.clearUser();   // 调用 store 的 clear action
    },
  },
});
```

`fields` / `actions` 把 store 字段映射到组件的 `data` 和 `this`，可直接在 WXML 中使用：

```xml
<!-- components/user-card/user-card.wxml -->
<view wx:if="{{isLoggedIn}}">
  <text>{{userInfo.nickname}}</text>
  <button bindtap="onLogout">退出</button>
</view>
```

## 绑定到 Page（手工绑定）

Page 构造器不能用 behavior，必须用 `createStoreBindings` 手工绑定，并在 `onUnload` 中清理：

```ts
// pages/profile/profile.ts
import { createStoreBindings } from 'mobx-miniprogram-bindings';
import { userStore } from '@/services/store/user';

Page({
  onLoad() {
    this.storeBindings = createStoreBindings(this, {
      store: userStore,
      fields: ['token', 'userInfo', 'isLoggedIn'],
      actions: {
        setToken: 'setToken',
        clearUser: 'clear',
      },
    });
  },

  onUnload() {
    this.storeBindings.destroyStoreBindings();   // 必须清理，否则内存泄漏
  },
});
```

:::danger[内存泄漏]
`createStoreBindings` 必须在 `onUnload` 中调用 `destroyStoreBindings()`，否则每次进页面都会累积一份绑定。
:::

## TypeScript 接口（推荐）

TS 下用 `ComponentWithStore` / `BehaviorWithStore` 替代原构造器，自动处理类型：

```ts
// components/user-card/user-card.ts
import { ComponentWithStore } from 'mobx-miniprogram-bindings';
import { userStore } from '@/services/store/user';

ComponentWithStore({
  storeBindings: {
    store: userStore,
    fields: ['token', 'userInfo', 'isLoggedIn'] as const,
    actions: { clearUser: 'clear' } as const,
  },

  methods: {
    onLogout() {
      this.clearUser();
    },
  },
});
```

:::tip[as const]
`fields` 和 `actions` 末尾必须加 `as const`，否则类型会推断为 `string[]` 而非字面量联合类型，失去类型保护。
:::

## 多 store 同时绑定

`storeBindings` 也可以是数组，同时绑定多个 store：

```ts
storeBindings: [
  {
    store: userStore,
    fields: ['userInfo'],
    actions: ['setUserInfo'],
  },
  {
    store: cartStore,
    fields: ['cartCount'],
    actions: ['addToCart'],
  },
],
```

## services 层驱动 store

业务流程直接读写 store，store 成为唯一真源：

```ts
// services/auth/login.ts
import { userStore } from '@/services/store/user';

export const login = async () => {
  const code = await wxLogin();
  const { token, userInfo } = await request({ /* ... */ });

  userStore.setToken(token);
  userStore.setUserInfo(userInfo);
};
```
