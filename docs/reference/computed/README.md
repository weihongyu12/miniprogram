---
description: miniprogram-computed 工作原理与适用场景，为 Component 补齐计算属性与侦听器能力
---

# 计算属性

完整的 computed / watch 代码示例见 [计算属性](../../cookbook/computed/)。

## 什么是 miniprogram-computed

[miniprogram-computed](https://github.com/wechat-miniprogram/computed) 是微信官方提供的小程序扩展包，为 `Component` 补齐 **计算属性（computed）** 和 **侦听器（watch）** 两种能力。

原生小程序的 `Component` 不带这两种能力，当组件内存在依赖 `data` 派生的字段时，只能手动在每次 `setData` 后重新计算并写入，容易遗漏。`miniprogram-computed` 通过 behavior 注入自动追踪依赖、自动更新的机制解决这个问题。

## 为什么需要

原生小程序的常见痛点：

- **派生字段需手动维护**：`fullName` 依赖 `firstName` + `lastName`，每次改 `firstName` 都要记得同步更新 `fullName`
- **多字段联动易遗漏**：`a` 和 `b` 任一变化都需触发某段逻辑，散落在多个方法中难以集中管理
- **跨页面共享计算成本高**：没有统一入口

`computed` 自动追踪依赖并更新派生字段；`watch` 集中监听字段变化执行副作用。两者配合可显著降低组件内状态联动的复杂度。

## 两个核心能力

| 能力 | 作用 | 类比 |
|------|------|------|
| `computed` | 声明依赖 `data` 的派生字段，依赖变化时自动重算 | Vue 的 computed |
| `watch` | 监听指定字段变化，执行副作用（如请求、setData） | Vue 的 watch |

## 工作原理

`miniprogram-computed` 通过 behavior 实现。组件声明 `behaviors: [computedBehavior]` 后：

1. **computed 追踪依赖**：首次渲染时执行 computed 函数，记录其访问的 data 字段作为依赖
2. **自动重算**：依赖字段变化时，自动重新执行对应 computed 函数，并把返回值写入 `this.data`
3. **watch 触发**：监听的字段变化时，调用对应回调

## computed 的关键限制

:::warning[computed 函数中不能访问 this]
computed 函数签名是 `computed(data)`，**只能通过 `data` 参数访问当前数据，不能通过 `this` 访问 methods、properties 等**。

原因：computed 在 data 变更的同步流程中执行，此时 `this` 上的方法等可能尚未就绪，访问 `this` 会破坏响应式追踪的纯函数特性。
:::

```js
// ❌ 错误：computed 中访问 this
computed: {
  sum(data) {
    return data.a + this.properties.step;  // this 不可用
  },
},

// ✅ 正确：只依赖 data
computed: {
  sum(data) {
    return data.a + data.b;
  },
},
```

如果派生值依赖 properties，应把 properties 的值同步到 data，或在 computed 中通过 `data` 访问（properties 字段也会出现在 data 中）。

## watch 的使用

watch 的 key 支持单个字段或逗号分隔的多个字段：

```js
watch: {
  // 单字段
  a: function (newVal) { /* ... */ },

  // 多字段：任一变化都触发，按顺序接收新值
  'a, b': function (a, b) { /* ... */ },
},
```

watch 中可以访问 `this`（与 computed 不同），常用于触发请求或联动 setData。

## 与 mobx-miniprogram 的关系

两者解决不同层级的问题，互不冲突，可同时使用：

| 对比项 | miniprogram-computed | mobx-miniprogram |
|--------|---------------------|------------------|
| 作用域 | 组件级，基于组件 `data` | 全局，跨页面共享 store |
| computed 依赖来源 | 组件 data 字段 | store 状态 |
| 适用场景 | 组件内派生字段、联动逻辑 | 跨页面共享状态、用户信息、购物车 |

:::tip
- 组件内的派生字段用 `miniprogram-computed`
- 跨页面共享的全局状态用 `mobx-miniprogram`（见 [状态管理](../state-management/)）
:::

## 适用场景

### ✅ 推荐

- 表单组件：`总价 = 单价 × 数量`、`是否可提交` 等派生字段
- 联动逻辑：A 字段变化需触发请求或更新 B 字段
- 复杂组件：存在多个互相依赖的 data 字段

### ⚠️ 谨慎

- 简单组件：只有一两个字段，引入 behavior 反而增加心智负担
- 需要访问 `this` 的计算逻辑：computed 不支持，应改用 methods 手动调用

## 参见

- [miniprogram-computed（GitHub）](https://github.com/wechat-miniprogram/computed)
- [计算属性](../../cookbook/computed/)
- [状态管理](../state-management/)
