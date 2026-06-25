---
description: 微信小程序 TypeScript 实践，项目配置、类型约束与常见陷阱
---

# TypeScript

完整的 Page / Component / App / Behavior 写法示例见 [TypeScript](../../cookbook/typescript/)。

## 为什么小程序值得用 TypeScript

原生小程序的 `Page` / `Component` 选项式 API 大量依赖字符串字段名（`data`、`setData`、`properties`、`methods`），用 JS 开发时常见痛点：

- `this.data.xxx` 拼写错误运行时才暴露
- `wx.request` 返回值无类型，需要手写 interface
- properties 类型与父组件传值不匹配
- 跨页面/组件共享的数据结构无文档

TypeScript 在编译期就能拦截这些问题。微信开发者工具从 `1.05.2109101` 起原生支持 TS，**无需额外构建配置**。

## 工作原理（重要）

小程序原生 TS 编译由开发者工具内置的 `@babel/plugin-transform-typescript` 插件完成。**该插件只做"类型剥离"（strip types），不做类型检查**：

- 编译时：移除类型注解，输出 JS
- 类型错误：仅在编辑器（IDE / VSCode）中提示，**编译过程不报错、不阻断**

这意味着：

- 类型错误不会让小程序无法运行（好处：不会因类型问题卡住构建）
- 但也意味着 TS 的类型保护**依赖编辑器**，不能只靠编译器把关

:::warning
不能因为"工具没报错"就以为类型正确。强烈建议配合 IDE，或在 CI 中运行 `tsc --noEmit` 做真正的类型检查。
:::

## 项目配置

### 1. 开启 TS 编译插件

`project.config.json` 的 `setting.useCompilerPlugins` 加入 `"typescript"`：

```json
{
  "setting": {
    "useCompilerPlugins": ["typescript", "sass"]
  }
}
```

支持同时开启 `typescript`、`less`、`sass` 三种编译插件。

### 2. tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "moduleResolution": "Node",
    "strict": true,
    "noImplicitAny": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "lib": ["ES2020"],
    "types": ["miniprogram-api-typings"],
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*.ts", "typings/**/*.d.ts"],
  "exclude": ["node_modules", "miniprogram_npm"]
}
```

要点：

- `target` 建议用 ES2020，小程序运行时支持现代 ES 语法
- `types` 引入 `miniprogram-api-typings`（见下节）
- `paths` 配合 `@/` 别名，与 [分层架构](../../getting-started/) 一致

### 3. 安装类型声明包

```bash
npm install -D miniprogram-api-typings
```

`miniprogram-api-typings` 提供 `wx.*`、`App`、`Page`、`Component`、`Behavior`、`getCurrentPages`、`getApp` 等全部小程序 API 的类型定义。

:::tip[更新声明文件]
从模板创建的 TS 项目，遇到 API 类型过时时，可在开发者工具目录树 `typings/types/wx` 上右键 → "更新声明文件"。
:::

## 常见陷阱

### 1. setData 类型不安全

小程序原生 `setData` 不做字段类型校验：

```ts
this.setData({ count: 'oops' });  // TS 不报错（运行时才出错）
```

原因：`Page` / `Component` 泛型只约束 `data` 字段，不约束 `setData` 入参。

**应对**：在 strict 模式下编码，依赖编辑器实时检查；高级用法可包装一层类型安全的 `setData`。

### 2. properties 默认值类型推断

`properties: { step: { type: Number, value: 1 } }` 中，`value` 字段的类型不会被严格约束（写成字符串也不会报错）。需手动校对。

### 3. this.data 与 setData 数据流

`this.data` 在 `methods` 内是只读快照，**修改 `this.data.xxx = ...` 不生效**。必须用 `setData`：

```ts
// ❌ 错误：直接修改不触发渲染
this.data.count += 1;

// ✅ 正确
this.setData({ count: this.data.count + 1 });
```

### 4. wx API 的回调风格与 Promise

`wx.xxx` 既支持 callback 风格，也支持 Promise（基础库 2.10.2+）。在 TS 中优先用 Promise/async：

```ts
// ✅ 推荐
const { code } = await wx.login();

// 旧式回调也可用，但需注意类型
wx.login({
  success: (res) => { /* res.code: string */ },
});
```

### 5. 跨页面共享类型

将业务相关的类型定义放在 `typings/` 目录，统一管理：

```
typings/
├── api.d.ts          # 后端接口响应类型
├── business.d.ts     # 业务领域类型
└── index.d.ts
```

```ts
// typings/api.d.ts
declare namespace API {
  interface LoginResponse {
    token: string;
    refreshToken: string;
    isNewUser: boolean;
  }
}
```

使用时无需 import：

```ts
const { token } = await request<API.LoginResponse>({ /* ... */ });
```

## CI 中的类型检查

由于开发者工具内置编译器不做类型检查，CI 中应运行 `tsc --noEmit`：

```json
// package.json
{
  "scripts": {
    "type-check": "tsc --noEmit"
  }
}
```

CI 中加入：

```yaml
lint:
  script:
    - npm run lint
    - npm run type-check
    - npm run lint:style
```

## 参见

- [原生支持 TypeScript（官方）](https://developers.weixin.qq.com/miniprogram/dev/devtools/compilets.html)
- [miniprogram-api-typings](https://github.com/wechat-miniprogram/api-typings)
- [TypeScript](../../cookbook/typescript/)
- [项目目录规范](../../specification/directory/)
