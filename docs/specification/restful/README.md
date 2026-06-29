---
sidebar_position: 5
toc_min_heading_level: 2
toc_max_heading_level: 4
description: 微信小程序接口请求规范，涵盖域名配置、Orval 请求集成、错误处理等小程序场景特有约定
---

# RESTful API 请求规范

本项目遵循标准 [RESTful API 规范](https://weihongyu12.github.io/web/docs/specification/restful/) 进行接口设计。由于微信小程序的运行环境不同于标准浏览器，我们采用 Orval + 自定义 [`wx.request`](https://developers.weixin.qq.com/miniprogram/dev/api/network/request/wx.request.html) 实例 (Custom Mutator) 的架构来处理网络请求。

## 域名配置

:::warning[小程序强制要求]
微信小程序在正式环境中只允许 HTTPS 请求，且域名必须在小程序管理后台配置 request 合法域名。开发阶段可在开发者工具中勾选“不校验合法域名”。
:::

在小程序管理后台 → 开发管理 → 开发设置 → 服务器域名 中配置：

- **request 合法域名**：HTTPS 接口域名
- **uploadFile 合法域名**：文件上传域名
- **downloadFile 合法域名**：文件下载域名

:::warning
每月可修改域名 50 次，建议提前规划好域名结构。
:::

## Orval 客户端与 wx.request 封装

为了享受 OpenAPI 带来的类型安全和自动生成代码的便利（详见 [OpenAPI 规范](https://weihongyu12.github.io/web/docs/features/openapi/)），我们不直接在业务中使用原生的 `wx.request`。

通过在 Orval 配置中指定自定义的 mutator，我们可以将小程序的原生请求方法与自动生成的 API 客户端完美结合。

### 1. 编写自定义请求实例 (Custom Mutator)

创建 `services/http/request-instance.ts` 文件，将 `wx.request` 封装为符合 Orval 签名的 Promise 函数：

```ts
// services/http/request-instance.ts
import { tokenManager } from '../storage/token';
import { handleHttpError } from './error-handler';

// 根据小程序的当前运行环境，动态配置 BASE_URL
const getBaseUrl = (): string => {
  const { miniProgram } = wx.getAccountInfoSync();
  const { envVersion } = miniProgram;

  const baseUrlMap: Record<string, string> = {
    develop: 'https://dev-api.example.com',
    trial: 'https://test-api.example.com',
    release: 'https://api.example.com',
  };

  return baseUrlMap[envVersion] ?? baseUrlMap.release;
};

const BASE_URL = getBaseUrl();

export interface RequestConfig {
  url: string;
  method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  params?: Record<string, any>;
  data?: Record<string, any>;
  headers?: Record<string, string>;
}

export const requestInstance = <T>(config: RequestConfig): Promise<T> => {
  const token = tokenManager.get();

  return new Promise((resolve, reject) => {
    wx.request({
      url: `${BASE_URL}${config.url}`,
      method: config.method,
      // wx.request 会自动将 GET 请求的 data 转换为 query string
      data: config.method === 'GET' ? config.params : config.data,
      header: {
        'Content-Type': 'application/json',
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
        ...config.headers,
      },
      success: (res) => {
        // 遵循 RESTful 规范，利用 HTTP 状态码判断成功与否
        if (res.statusCode >= 200 && res.statusCode < 300) {
          resolve(res.data as T);
          return;
        }
        // 将状态码和响应体交给专门的错误处理器
        handleHttpError(res.statusCode, res.data);
        reject(res);
      },
      fail: (err) => {
        wx.showToast({ title: '网络连接失败', icon: 'error' });
        reject(err);
      },
    });
  });
};
```

### 2. 统一错误处理拦截

在上述的 `handleHttpError` 函数中，我们按照项目的 RESTful 标准集中处理状态码：

```ts
// services/http/error-handler.ts
export const handleHttpError = (statusCode: number, data: any): void => {
  const errorMessage = data?.message || '请求失败';

  switch (statusCode) {
    case 400:
      wx.showToast({ title: errorMessage, icon: 'none' });
      break;
    case 401:
      // Token 失效，清除凭证并跳转登录
      tokenManager.clear();
      wx.showToast({ title: '登录已过期，请重新登录', icon: 'none' });
      setTimeout(() => wx.reLaunch({ url: '/pages/login/login' }), 1500);
      break;
    case 403:
      wx.showToast({ title: '无权访问该资源', icon: 'none' });
      break;
    case 404:
      wx.showToast({ title: '请求的资源不存在', icon: 'none' });
      break;
    case 429:
      wx.showToast({ title: '操作过于频繁，请稍后再试', icon: 'none' });
      break;
    case 500:
    case 502:
    case 503:
    case 504:
      wx.showToast({ title: '服务器开小差了', icon: 'error' });
      break;
    default:
      wx.showToast({ title: errorMessage, icon: 'none' });
  }
};
```

### 3. 配置 Orval

在项目根目录的 `orval.config.ts` 中配置覆盖 mutator：

```ts
// orval.config.ts
export default {
  api: {
    input: './openapi.yaml',
    output: {
      mode: 'tags-split',
      target: 'src/api/generated',
      schemas: 'src/api/model',
      override: {
        mutator: {
          path: 'src/services/http/request-instance.ts',
          name: 'requestInstance',
        },
      },
    },
  },
};
```

配置完成后，业务代码只需直接导入 Orval 生成的函数，既享有类型提示，又底层默认使用了 `wx.request`。

## 文件上传：特别说明

:::warning[小程序限制与 Orval 冲突]
标准 OpenAPI 规范中的 `multipart/form-data` 通常会生成使用 `FormData` 对象的代码。但微信小程序不支持 `FormData`，只能使用特有的 [`wx.uploadFile`](https://developers.weixin.qq.com/miniprogram/dev/api/network/upload/wx.uploadFile.html) API。
:::

针对文件上传接口，请不要直接使用 Orval 生成的该接口函数。建议针对上传接口单独进行封装：

```ts
// services/http/upload.ts
export const uploadFile = (
  tempFilePath: string,
  formData?: Record<string, string>,
): Promise<unknown> => {
  const token = tokenManager.get();

  return new Promise((resolve, reject) => {
    wx.uploadFile({
      url: `${BASE_URL}/api/file/upload`,
      filePath: tempFilePath,
      name: 'file', // 后端接收文件的字段名
      formData, // 只能是简单键值对
      header: {
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
      },
      success: (res) => {
        if (res.statusCode >= 200 && res.statusCode < 300) {
          // wx.uploadFile 的 res.data 是字符串，需要手动 parse
          resolve(JSON.parse(res.data));
          return;
        }
        reject(res);
      },
      fail: reject,
    });
  });
};
```

详细的文件管理及上传业务方案参见[文件上传](../../features/file-upload/)。

## Token 管理

为了保证全局的鉴权逻辑统一，Token 的读写必须通过专用的管理器，底层使用 `wx.setStorageSync`：

```ts
// services/storage/token.ts
const TOKEN_KEY = 'auth_token';

export const tokenManager = {
  get(): string | null {
    return wx.getStorageSync(TOKEN_KEY) || null;
  },
  set(token: string): void {
    wx.setStorageSync(TOKEN_KEY, token);
  },
  clear(): void {
    wx.removeStorageSync(TOKEN_KEY);
  },
};
```
