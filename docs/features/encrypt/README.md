---
description: 微信小程序敏感数据加密实践，AES 双向加密传输与数据脱敏方案
---

# 敏感数据加密

对敏感数据采用 AES 加密传输，支持前端到后端和后端到前端的双向加密通信，避免数据泄露。

:::warning
采用 AES 加密传输属于**纵深防御**，在传输数据时，应该确保**优先使用 HTTPS**。小程序在正式环境强制 HTTPS，本规范作为额外防护层。
:::

## 基础理论

### 敏感数据的分类与特征

| 数据类别 | 具体字段示例 | 识别策略推荐 |
|---------|------------|------------|
| **个人身份信息** | 身份证号、手机号、邮箱、家庭住址、姓名 | 格式固定，优先使用**正则表达式**进行自动化值匹配 |
| **金融与交易信息** | 银行卡号、账户余额、交易流水、薪资 | 部分格式固定（如银行卡号可用正则），其余依赖**手动字段标记** |
| **认证与凭据信息** | 登录密码、支付密码、API Key、JWT Token | 具有明确的语义特征，优先通过**字段名（Key）匹配**或手动标记 |

### 自动化识别与手动标记的结合

采用"正则值检测"与"字段字典映射"相结合的纯函数手段，精准定位复杂业务对象中的敏感数据：

```ts
// services/crypto/detect.ts
const sensitivePatterns = {
  idCard: /^[1-9]\d{5}(18|19|20)\d{2}((0[1-9])|(1[0-2]))(([0-2][1-9])|10|20|30|31)\d{3}[0-9Xx]$/,
  phone: /^1[3-9]\d{9}$/,
  bankCard: /^\d{13,19}$/,
  email: /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/,
};

const sensitiveFields: Record<string, string> = {
  'user.address': 'PII',
  'payment.balance': 'FINANCIAL',
  'auth.password': 'AUTH',
  'auth.apiKey': 'AUTH',
};

/**
 * 敏感字段检测管道
 */
export const isSensitiveData = (fieldPath: string, value: unknown): boolean => {
  if (sensitiveFields[fieldPath]) return true;
  if (/(password|pwd|secret|token)/i.test(fieldPath)) return true;
  if (typeof value === 'string') {
    return Object.values(sensitivePatterns).some((pattern) => pattern.test(value));
  }
  return false;
};
```

### 表现层：数据脱敏处理

对于非网络传输场景（如日志打印、页面只读展示），使用数据脱敏（Masking）保护隐私：

```ts
// services/crypto/mask.ts

/**
 * 身份证号脱敏：保留前6位和后4位
 * 示例：110101199001011234 -> 110101********1234
 */
export const maskIdCard = (idCard: string): string => {
  if (!idCard || idCard.length !== 18) return idCard;
  return idCard.replace(/(\d{6})\d{8}(\d{4})/, '$1********$2');
};

/**
 * 手机号脱敏：保留前3位和后4位
 * 示例：13800138000 -> 138****8000
 */
export const maskPhone = (phone: string): string => {
  if (!phone || phone.length !== 11) return phone;
  return phone.replace(/(\d{3})\d{4}(\d{4})/, '$1****$2');
};

/**
 * 银行卡号脱敏：只保留后4位
 * 示例：6225880123456789 -> **** **** **** 6789
 */
export const maskBankCard = (cardNumber: string): string => {
  if (!cardNumber) return cardNumber;
  const lastFour = cardNumber.slice(-4);
  const maskedLength = Math.max(0, cardNumber.length - 4);
  const maskedGroups = Math.ceil(maskedLength / 4);
  return `${'**** '.repeat(maskedGroups)}${lastFour}`;
};

/**
 * 邮箱脱敏：保留第一个字符和域名
 * 示例：zhangsan@example.com -> z*******@example.com
 */
export const maskEmail = (email: string): string => {
  if (!email || !email.includes('@')) return email;
  const [username, domain] = email.split('@');
  if (username.length <= 1) return email;
  return `${username[0]}${'*'.repeat(username.length - 1)}@${domain}`;
};

/**
 * 姓名脱敏：保留姓氏
 * 示例：张三 -> 张*，李四五 -> 李**
 */
export const maskName = (name: string): string => {
  if (!name || name.length <= 1) return name;
  return `${name[0]}${'*'.repeat(name.length - 1)}`;
};
```

## 端实现

### 与 Web 场景的差异

| 维度 | Web 场景 | 小程序场景 |
|------|---------|----------|
| 加密 API | `crypto.subtle`（Web Crypto API） | `crypto-js`（npm 包） |
| RSA 密钥对生成 | `crypto.subtle.generateKey` | `crypto-js` 不支持 RSA，需后端生成或引入 `jsencrypt` |
| Base64 转换 | `btoa` / `atob` | `wx.arrayBufferToBase64` / `wx.base64ToArrayBuffer` |

:::warning[小程序限制]
小程序不支持 `crypto.subtle`（Web Crypto API），需引入纯 JS 实现的加密库：

- **AES**：使用 [crypto-js](https://github.com/brix/crypto-js)
- **RSA**：使用 [jsencrypt](https://github.com/travist/jsencrypt)（如需 RSA-OAEP）

由于 RSA 在小程序中性能较差，**推荐简化方案**：使用固定 AES 密钥 + 后端定期轮换，而非每次请求生成临时密钥对。
:::

### 简化方案：固定 AES 密钥

适用于大多数小程序场景，避免 RSA 性能开销：

```ts
// services/crypto/aes.ts
import CryptoJS from 'crypto-js';

// 密钥从服务端动态获取（启动时拉取，定期轮换）
let AES_KEY = '';

export const setAesKey = (key: string): void => {
  AES_KEY = key;
};

export const encrypt = (data: Record<string, unknown>): string => {
  const jsonStr = JSON.stringify(data);
  return CryptoJS.AES.encrypt(jsonStr, AES_KEY).toString();
};

export const decrypt = (ciphertext: string): Record<string, unknown> => {
  const bytes = CryptoJS.AES.decrypt(ciphertext, AES_KEY);
  const jsonStr = bytes.toString(CryptoJS.enc.Utf8);
  return JSON.parse(jsonStr);
};
```

### 在 http 拦截器中集成

```ts
// services/http/interceptor.ts
import { encrypt, decrypt, isSensitiveData } from '@/services/crypto';

const SENSITIVE_PATHS = ['/api/user/profile', '/api/payment/confirm'];

const isSensitiveRequest = (url: string, data: unknown): boolean => {
  if (SENSITIVE_PATHS.some((path) => url.includes(path))) return true;
  if (data && typeof data === 'object') {
    return Object.entries(data).some(([key, value]) =>
      isSensitiveData(key, value),
    );
  }
  return false;
};

export const encryptRequest = (
  url: string,
  data: Record<string, unknown>,
): Record<string, unknown> => {
  if (!isSensitiveRequest(url, data)) return data;
  return { encrypted: encrypt(data) };
};

export const decryptResponse = (response: unknown): unknown => {
  if (response && typeof response === 'object' && 'encrypted' in response) {
    return decrypt((response as { encrypted: string }).encrypted);
  }
  return response;
};
```

### 在 services 业务模块中使用

```ts
// services/auth/profile.ts
import { request } from '@/services/http';

export const updateProfile = async (profile: {
  name: string;
  idCard: string;
  phone: string;
}) => {
  // 拦截器会自动识别敏感字段并加密
  return request({
    url: '/user/profile',
    method: 'PUT',
    data: profile,
  });
};
```

## 安全风险一览

| 风险点 | 业务隐患 | 防范方案 |
|--------|---------|---------|
| **中间人攻击 (MITM)** | 攻击者伪造服务器公钥拦截解密 | 小程序强制 HTTPS，双向加密作为纵深防御 |
| **重放攻击 (Replay)** | 恶意第三方拦截合法密文重复发送 | 加密明文中添加 **时间戳** + **Nonce**，后端校验窗口外请求 |
| **密钥泄露** | AES 密钥硬编码在前端代码中 | 密钥从服务端动态获取，定期轮换；**禁止硬编码** |
| **IV 密钥复用** | 同一 AES 密钥使用相同 IV | 每次加密重新生成 12 字节 IV |

## 性能优化

### 密钥复用策略

- **会话复用**：启动时拉取一次 AES 密钥，在会话生命周期内复用，避免每次请求都握手
- **避免大对象加密**：仅对敏感字段加密，非敏感字段明文传输，减少加解密开销

### 异步处理

`crypto-js` 是同步阻塞的，加密大对象时会卡 UI 线程。对于大对象，建议：

- 拆分为多个小对象分别加密
- 或仅加密敏感字段，非敏感字段明文传输

## 参考资料

- [crypto-js](https://github.com/brix/crypto-js)
- [jsencrypt](https://github.com/travist/jsencrypt)
- [小程序安全指南](https://developers.weixin.qq.com/miniprogram/dev/framework/security.html)
