---
sidebar_position: 5
toc_min_heading_level: 2
toc_max_heading_level: 4
description: 微信小程序接口请求规范，涵盖域名配置、请求封装、错误处理等小程序场景特有约定
---

# RESTful API 规范

RESTful API 的设计规范（HTTP 方法、状态码、响应格式、错误码等）与 Web 项目完全一致，此处不再重复。本文档仅说明小程序场景下接口请求的特有约定。

## 域名配置

:::warning[小程序强制要求]
微信小程序在正式环境中**只允许 HTTPS 请求**，且域名必须在小程序管理后台配置 `request 合法域名`。开发阶段可在开发者工具中勾选"不校验合法域名"。
:::

在小程序管理后台 → 开发管理 → 开发设置 → 服务器域名 中配置：

- `request 合法域名`：HTTPS 接口域名
- `uploadFile 合法域名`：文件上传域名
- `downloadFile 合法域名`：文件下载域名

:::tip
每月可修改域名 50 次，建议提前规划好域名结构。
:::

## wx.request 封装

小程序原生 `wx.request` 是回调风格，需在 `services/http` 层统一封装为 Promise，并处理 Token 注入、错误拦截。

```ts
// services/http/request.ts
interface RequestConfig {
  url: string;
  method?: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  data?: Record<string, unknown>;
  header?: Record<string, string>;
}

const BASE_URL = 'https://api.example.com/api';

export const request = <T = unknown>(config: RequestConfig): Promise<T> => {
  const token = wx.getStorageSync('token');
  return new Promise((resolve, reject) => {
    wx.request({
      url: `${BASE_URL}${config.url}`,
      method: config.method ?? 'GET',
      data: config.data,
      header: {
        'Content-Type': 'application/json',
        Authorization: token ? `Bearer ${token}` : '',
        ...config.header,
      },
      success: (res) => {
        if (res.statusCode === 401) {
          // token 失效，跳转登录
          wx.removeStorageSync('token');
          wx.reLaunch({ url: '/pages/login/login' });
          return reject(res);
        }
        if (res.statusCode >= 200 && res.statusCode < 300) {
          resolve(res.data as T);
        } else {
          // 统一错误提示
          const error = res.data as { message?: string };
          wx.showToast({
            title: error?.message || '请求失败',
            icon: 'none',
          });
          reject(res);
        }
      },
      fail: reject,
    });
  });
};
```

### 错误处理建议

在 `services/http` 拦截器中统一处理错误响应：

| 状态码 | 处理方式 |
|-------|---------|
| `401` | 清除本地 token，跳转登录页 |
| `429` | 提示"操作过于频繁" |
| `5xx` | 提示"服务异常，请稍后重试" |
| 业务错误码 | 根据 `message` 字段 toast 提示 |

## 文件上传：wx.uploadFile

:::warning[小程序限制]
小程序**不支持 `FormData` 对象**，`wx.uploadFile` 通过 `filePath` + `name` + `formData` 参数传递文件与附加字段。`formData` 只能是简单键值对，不能嵌套对象。
:::

```js
wx.uploadFile({
  url: 'https://api.example.com/api/file',
  filePath: tempFilePath,
  name: 'file',
  formData: { bizType: 'avatar' },
  success(res) {
    const data = JSON.parse(res.data);
  }
});
```

详细的文件上传方案参见 [文件上传](../../features/file-upload/)。

## Token 管理

小程序的 Token 存储与 Web 不同，使用 `wx.setStorageSync` 而非 `localStorage`：

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

在 `services/http` 拦截器中自动注入 Token，避免在每个页面手动拼接。详见 [认证](../../features/authentication/)。
