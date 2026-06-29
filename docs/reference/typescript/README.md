---
description: 微信小程序 TypeScript 实践，项目配置、类型约束与常见陷阱
---

# TypeScript

本文档旨在规范小程序开发中的 TypeScript 使用习惯，确保项目类型安全且易于维护。完整的 `Page` / `Component` / `App` / `Behavior` 写法示例请参考 [TypeScript](../../cookbook/typescript/)。

## 为什么小程序需要 TypeScript

原生小程序的选项式 API 严重依赖字符串字段名（`data`、`setData`、`properties`），使用 JS 开发时极易出现难以察觉的运行时错误：

- **拼写错误**：`this.data.xxx` 拼写错误直到运行时才暴露
- **接口脆弱**：`wx.request` 返回值缺乏类型定义
- **传值不匹配**：组件 `properties` 与父组件传递的值类型不一致
- **文档缺失**：跨模块共享的数据结构缺乏类型约束，逻辑难以追踪

TypeScript 在编译期即可拦截上述问题。微信开发者工具从 `1.05.2109101` 起[原生支持 TS](https://developers.weixin.qq.com/miniprogram/dev/devtools/compilets.html)，**无需额外的构建任务**。

## 工作原理（核心机制）

微信小程序原生 TS 编译由开发者工具内置的 `@babel/plugin-transform-typescript` 插件处理。**请务必注意：该插件仅进行“类型剥离”（strip types），不执行类型检查**。

- **编译时**：移除类型注解，输出 JavaScript
- **编译错误**：编译阶段不会阻断，也不会报错

**这意味着**：

- 类型错误不会导致小程序无法运行（避免了构建卡死）
- **类型安全完全依赖 IDE 检查**。不能因为编译通过就认为代码类型安全，必须配合 IDE 提示或在 CI 中手动检查

:::warning
严禁因“工具没报错”而忽视类型检查。必须通过 CI 运行 `tsc --noEmit` 进行全量类型校验。
:::

## 项目配置

### 1. 开启编译器插件

在 `project.config.json` 的 `setting.useCompilerPlugins` 中添加 `"typescript"`：

```json
{
  "setting": {
    "useCompilerPlugins": ["typescript", "sass"]
  }
}
```

### 2. tsconfig.json 配置

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

### 3. 类型声明包

安装官方 API 类型定义：

```bash
npm install -D miniprogram-api-typings
```

:::tip[更新声明文件]
如遇 API 类型过时，在开发者工具目录树的 `typings/types/wx` 上右键，选择“更新声明文件”。
:::

## 类型管理策略

### 1. 后端接口类型（自动管理）

本项目已接入 Orval 自动生成后端接口模型。所有类型定义存放在 `src/api/model` 目录下。

- **原则**：严禁手动编写接口类型。
- **使用方式**：直接从对应模块导入。

```ts
import type { LoginResponse } from '@/api/model';

const { token } = await request<LoginResponse>({ /* ... */ });
```

### 2. 前端全局业务类型（手动管理）

对于非后端返回的、纯前端共享的业务领域类型，统一维护在 `typings/` 目录下。

```
typings/
├── business.d.ts     # 业务领域类型定义
└── index.d.ts
```

:::warning[避坑指南]
在 `.d.ts` 文件中，请避免在顶层使用 `import` 或 `export`。因为这会使文件被视为“模块（Module）”，导致定义的 namespace 失去全局作用域。正确的做法是使用 `declare namespace` 直接声明，以便在任意页面直接调用。
:::

## 常见陷阱与最佳实践

### 1. setData 类型不安全

原生 `setData` 不约束入参字段。建议编写一个辅助函数进行包装，或在 Strict 模式下通过 IDE 严格检查。

### 2. properties 默认值

`properties` 定义中 `value` 的类型推断有限。编写时务必确保 `value` 与定义的 `type` 类型一致，并手动进行校对。

### 3. this.data 与 setData

`this.data` 是只读快照。直接修改 `this.data.xxx = ...` **不会触发页面更新**。必须使用 `this.setData`。

### 4. wx API 的 Promise 化

优先使用 Promise 风格的 API，以提升异步代码的可读性：

```ts
// ✅ 推荐
const { code } = await wx.login();

// ❌ 尽量避免旧的回调风格，除非处理复杂交互
```

## CI 中的类型校验

由于开发者工具仅剥离类型，CI/CD 流程中必须显式增加类型检查步骤：

```json
// package.json
{
  "scripts": {
    "type-check": "tsc --noEmit"
  }
}
```

在 CI 配置文件中添加：

```yaml
lint:
  script:
    - npm run lint
    - npm run type-check
```

## 参见

- [原生支持 TypeScript](https://developers.weixin.qq.com/miniprogram/dev/devtools/compilets.html)
- [miniprogram-api-typings](https://www.npmjs.com/package/miniprogram-api-typings)
- [项目目录规范](../../specification/directory/)
