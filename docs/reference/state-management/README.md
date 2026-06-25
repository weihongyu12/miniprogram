---
description: 微信小程序状态管理原理，mobx-miniprogram 与 services 层的关系及最佳实践
---

# 状态管理

完整的 Store 定义与绑定代码见 [状态管理](../../cookbook/state-management/)。

## 为什么小程序需要状态管理

原生小程序的状态传递链路有限：

- 父子组件：`properties` 向下，`triggerEvent` 向上
- 跨页面/跨组件：只能通过 `getApp().globalData` 或自己实现事件总线

当业务涉及多页面共享用户信息、购物车、订单状态时，纯靠 `properties` + `globalData` 会写出难以维护的"面条代码"。

`mobx-miniprogram` 把 MobX 的响应式状态管理能力带到小程序，配合 `mobx-miniprogram-bindings` 把状态绑定到 `Page` / `Component`。

## 两个包的关系

| 包 | 作用 |
|----|------|
| [`mobx-miniprogram`](https://github.com/wechat-miniprogram/mobx) | MobX 的小程序构建版，提供 `observable` / `action` / `computed` 等核心能力 |
| [`mobx-miniprogram-bindings`](https://github.com/wechat-miniprogram/mobx-miniprogram-bindings) | 把 MobX store 绑定到 Page/Component 的辅助库 |

:::warning
必须同时安装两个包。`mobx-miniprogram` 不绑定视图，`mobx-miniprogram-bindings` 不含 MobX 核心。
:::

需要基础库版本 ≥ 2.11.0。

## 安装与构建

```bash
npm install --save mobx-miniprogram mobx-miniprogram-bindings
```

然后在开发者工具或 CI 中"构建 npm"。详见 [npm](../npm/)。

## 与 services 层的关系

Store 应放在 `services/store/`，与 [分层架构](../../getting-started/) 一致：

```
services/
├── store/                 # mobx store 定义
│   ├── user.ts
│   ├── cart.ts
│   └── index.ts
├── auth/                  # 业务流程：登录、Token 刷新
├── http/
└── ...
```

`auth` / `http` 等 services 子模块可以**直接读写 store**（业务流程驱动状态变化），而 pages / components 通过 `mobx-miniprogram-bindings` 订阅 store：

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

这样业务流程与视图解耦，store 成为唯一的真源（single source of truth）。

## 最佳实践

- **状态修改只在 action 中进行**：避免直接 `store.xxx = ...`
- **store 按业务域拆分**：`userStore` / `cartStore` / `orderStore` 各管一摊，避免巨型 store
- **计算属性用 getter**：不要把派生状态存到 data 中
- **Page 必须清理绑定**：`onUnload` 中 `destroyStoreBindings()`
- **不要在 store 中放 UI 状态**：tab 选中态、表单输入等页面级状态留在 `data` 中，store 只放跨页面共享状态

## 参见

- [mobx-miniprogram](https://github.com/wechat-miniprogram/mobx)
- [mobx-miniprogram-bindings](https://github.com/wechat-miniprogram/mobx-miniprogram-bindings)
- [MobX 官方文档](https://mobx.js.org/)
- [状态管理](../../cookbook/state-management/)
