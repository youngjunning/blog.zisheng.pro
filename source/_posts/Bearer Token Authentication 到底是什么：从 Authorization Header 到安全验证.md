---
title: Bearer Token Authentication 到底是什么：从 Authorization Header 到安全验证
date: 2026-09-17 15:43:17
description: Bearer Token 是 HTTP API 最常见的访问凭证，却经常被误认为 JWT、OAuth 2.0 或普通 API Key。本文从 RFC 6750 出发，讲清 Authorization Header、Access Token、JWT 与 Refresh Token 的边界，并给出一套可运行的 Node.js 验证、Scope 授权和错误响应实现。
categories:
  - [软件工程]
tags:
  - Bearer Token
  - OAuth 2.0
  - HTTP
  - JWT
  - API Security
  - Node.js
cover: /images/bearer-token-authentication.webp
---

最近在给桌面 Agent 接入 API 时，我遇到一个很基础、也很容易混淆的问题：请求头里写的 `Bearer` 到底叫什么，它是在证明用户身份，还是只是在携带一段 Token？

标准写法是：

```http
Authorization: Bearer <access_token>
```

这里没有连字符。`Bearer` 后面必须是空格，再跟 Access Token。它对应的标准名称是 **Bearer Token Authentication**，更准确地说，是 HTTP `Authorization` Header 中的 Bearer Authentication Scheme。

<!-- more -->

## 一句话总结

**Bearer Token 的核心规则是“谁持有，谁就能使用”。** Resource Server 通常不要求调用方再证明自己掌握某个私钥，因此 Token 一旦泄露，攻击者就可能直接重放。

这也是 Bearer Token 最重要的工程边界：它很好用，但安全性高度依赖 HTTPS、短有效期、最小权限、安全存储和严格验证。

## Bearer 到底是什么意思

[RFC 6750](https://www.rfc-editor.org/rfc/rfc6750.html) 对 Bearer Token 的定义很直接：任何持有 Token 的参与方，都可以用它访问 Token 对应的资源，不需要额外证明自己持有某个密码学密钥。

可以把它理解为一张临时门禁卡：

1. Authorization Server 签发门禁卡。
2. Client 保存门禁卡，并在访问 API 时出示。
3. Resource Server 检查门禁卡是否有效、是否过期、是否属于自己、权限是否足够。
4. 检查通过后，Resource Server 返回受保护资源。

这个比喻只解释“持有即使用”。实际系统仍然要检查 Token 的签名、Issuer、Audience、有效期和 Scope，不能看到一段字符串就放行。

## 正确的请求格式

客户端应优先把 Access Token 放进 `Authorization` Header：

```http
GET /api/orders/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IjIwMjYtMDkifQ...
```

用 `curl` 调用时：

```bash
curl https://api.example.com/api/orders/123 \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

在浏览器或 Node.js 中使用 `fetch`：

```ts
const response = await fetch('https://api.example.com/api/orders/123', {
  headers: {
    Authorization: `Bearer ${accessToken}`,
  },
});

if (!response.ok) {
  throw new Error(`Request failed: ${response.status}`);
}

const order = await response.json();
```

RFC 6750 还描述了 Form Body 和 URI Query 两种传递方式，但同一个请求不能同时使用多种方式。URI Query 尤其不推荐：URL 可能进入浏览器历史、反向代理、监控系统、访问日志和 Referer，泄露面远大于 Header。

```text
不推荐：GET /api/orders/123?access_token=xxx
推荐：  Authorization: Bearer xxx
```

## Bearer、JWT、OAuth 2.0 不是同一个概念

Bearer 经常和 JWT、OAuth 2.0 一起出现，但三者不在同一层。

| 名称 | 所在层次 | 解决的问题 |
| --- | --- | --- |
| Bearer | HTTP Token 使用方式 | Client 如何向 Resource Server 出示 Token |
| Access Token | 授权凭证 | 当前 Client 被允许访问什么资源、持续多久 |
| JWT | Token 编码格式 | 如何在一个可签名的数据结构中携带 Claims |
| OAuth 2.0 | 授权框架 | Client 如何取得授权并获得 Access Token |
| OpenID Connect | 身份协议 | 在 OAuth 2.0 之上表达用户登录和身份信息 |

Bearer Token 可以是 JWT，也可以是一段不可读的随机字符串。后者通常称为 Opaque Token，Resource Server 需要通过 Token Introspection 或内部查询确认它的状态。

反过来也成立：JWT 不一定是 Bearer Access Token。ID Token、内部事件签名和一次性声明都可以使用 JWT 格式。Resource Server 不能因为字符串长得像 `xxx.yyy.zzz`，就把它当作合法 Access Token。

## Bearer 与其他鉴权方式有什么区别

| 方式 | 请求中携带什么 | 主要特点 | 常见场景 |
| --- | --- | --- | --- |
| Bearer Token | Access Token | 持有即可使用，适合短期授权 | OAuth API、移动端、Agent、服务调用 |
| Basic Authentication | 用户名和密码的 Base64 编码 | 简单，但每次请求都暴露长期凭证 | 内部工具、兼容旧系统 |
| API Key | 应用或调用方的固定 Key | 易于接入，身份与权限模型通常较粗 | 开放 API、服务集成 |
| Session Cookie | 服务端 Session ID | 浏览器自动携带，需要处理 CSRF | 传统 Web 登录 |
| mTLS / DPoP | Token 加客户端密钥证明 | 被盗 Token 不一定能被单独重放 | 高安全 API、金融与关键基础设施 |

Bearer Token Authentication 只说明 Token 怎样被呈现，不自动回答 Token 怎样签发、撤销、刷新和绑定用户。完整系统至少还需要 Authorization Server、Resource Server、权限模型和密钥管理。

## 一次完整的 Access Token 生命周期

### 1. Client 获取 Access Token

在 OAuth 2.0 中，Client 通过 Authorization Code、Client Credentials 等流程向 Authorization Server 换取 Access Token。返回结果通常类似：

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 900,
  "scope": "orders:read orders:write",
  "refresh_token": "opaque-refresh-token"
}
```

`access_token` 用于调用 Resource Server；`refresh_token` 只发给 Authorization Server 换取新的 Access Token，不应该被发送到业务 API。

### 2. Client 调用 Resource Server

```http
Authorization: Bearer <access_token>
```

Client 不应把 Token 写进 URL、日志、错误上报或公开配置。跨服务转发时也不能默认原样透传，需要确认下游服务是否属于 Token 的 Audience。

### 3. Resource Server 验证 Token

对于 JWT Access Token，至少检查：

| 检查项 | 防止什么问题 |
| --- | --- |
| 签名与允许的算法 | Token 被伪造、篡改或算法降级 |
| `iss` | 接受了不受信任的签发方 |
| `aud` | 把签给其他 API 的 Token 拿来重放 |
| `exp`、`nbf` | 使用已过期或尚未生效的 Token |
| `scope` / permission | Token 有效，但没有当前操作权限 |
| Token 类型 | 把 ID Token、Refresh Token 当成 Access Token |

只执行 Base64 Decode 没有任何安全意义。Decode 只能读取 Claims，不能证明 Claims 没被修改。

### 4. Resource Server 返回标准错误

缺少或无效 Token 通常返回 `401 Unauthorized`；Token 有效但权限不足返回 `403 Forbidden`。RFC 6750 还定义了 `WWW-Authenticate` Header：

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="api", error="invalid_token"
```

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer realm="api", error="insufficient_scope", scope="orders:write"
```

这比只返回一段模糊的 JSON 更利于 Client 判断应该重新登录、刷新 Token，还是提示用户缺少权限。

## Node.js 实战：验证 JWT Bearer Token

这个最小 Resource Server 使用 Express 和 [`jose`](https://github.com/panva/jose)，从受信任的 JWKS 地址获取公钥，并验证 JWT 的签名、Issuer 和 Audience。

### 1. 安装依赖

```bash
npm install express jose
npm install -D typescript tsx @types/express
```

准备环境变量：

```bash
export AUTH_ISSUER="https://auth.example.com/"
export AUTH_AUDIENCE="https://api.example.com"
export AUTH_JWKS_URL="https://auth.example.com/.well-known/jwks.json"
```

JWKS 地址必须来自配置或可信的 Issuer Metadata，不能读取 Token Header 里的任意 `jku` 后直接请求。Token Header 属于攻击者可控输入。

### 2. 编写 Bearer Middleware

```ts
import express, { type NextFunction, type Request, type Response } from 'express';
import { createRemoteJWKSet, jwtVerify, type JWTPayload } from 'jose';

const issuer = process.env.AUTH_ISSUER!;
const audience = process.env.AUTH_AUDIENCE!;
const jwksUrl = process.env.AUTH_JWKS_URL!;

const JWKS = createRemoteJWKSet(new URL(jwksUrl));

type AuthenticatedRequest = Request & {
  auth?: JWTPayload;
};

function readBearerToken(header: string | undefined): string | undefined {
  if (!header) return undefined;
  const match = /^Bearer +([^\s]+)$/i.exec(header.trim());
  return match?.[1];
}

async function authenticateBearer(
  req: Request,
  res: Response,
  next: NextFunction,
) {
  const token = readBearerToken(req.header('authorization'));

  if (!token) {
    res.setHeader('WWW-Authenticate', 'Bearer realm="api"');
    return res.status(401).json({ error: 'unauthorized' });
  }

  try {
    const { payload } = await jwtVerify(token, JWKS, {
      issuer,
      audience,
      algorithms: ['RS256'],
      requiredClaims: ['exp'],
      clockTolerance: 5,
    });

    (req as AuthenticatedRequest).auth = payload;
    return next();
  } catch {
    res.setHeader(
      'WWW-Authenticate',
      'Bearer realm="api", error="invalid_token"',
    );
    return res.status(401).json({ error: 'invalid_token' });
  }
}

const app = express();

app.get('/api/profile', authenticateBearer, (req, res) => {
  res.json({ subject: (req as AuthenticatedRequest).auth?.sub });
});

app.listen(3000, () => {
  console.log('Resource Server listening on http://localhost:3000');
});
```

`jwtVerify` 会完成签名验证，并根据配置检查标准 Claims。生产代码还应校验所使用 Authorization Server 对 Access Token 规定的 `typ`、Scope Claim 名称和其他约束；不同 Provider 的 Claims 契约不完全相同。如果双方采用 [RFC 9068](https://www.rfc-editor.org/rfc/rfc9068.html) 的 JWT Access Token Profile，还应要求 `typ=at+jwt`，防止把 ID Token 当成 Access Token。

### 3. 增加 Scope 授权

身份验证通过不等于拥有所有权限。可以在同一条路由上继续检查 Scope：

```ts
function requireScope(requiredScope: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    const auth = (req as AuthenticatedRequest).auth;
    const scopes = String(auth?.scope || '')
      .split(' ')
      .filter(Boolean);

    if (!scopes.includes(requiredScope)) {
      res.setHeader(
        'WWW-Authenticate',
        `Bearer realm="api", error="insufficient_scope", scope="${requiredScope}"`,
      );
      return res.status(403).json({ error: 'insufficient_scope' });
    }

    return next();
  };
}

app.delete(
  '/api/orders/:id',
  authenticateBearer,
  requireScope('orders:delete'),
  (req, res) => {
    res.status(204).end();
  },
);
```

实际项目还需要把 `sub` 映射到用户或服务主体，并在业务层检查资源归属。例如拥有 `orders:write` 不代表可以修改任意用户的订单。

### 4. 验证三条关键分支

```bash
# 缺少 Token：预期 401，并返回 WWW-Authenticate: Bearer
curl -i http://localhost:3000/api/profile

# 无效 Token：预期 401 + invalid_token
curl -i http://localhost:3000/api/profile \
  -H "Authorization: Bearer invalid-token"

# 有效但缺少 Scope：预期 403 + insufficient_scope
curl -i -X DELETE http://localhost:3000/api/orders/123 \
  -H "Authorization: Bearer $READ_ONLY_ACCESS_TOKEN"
```

验证时不要只看 JSON Body。HTTP Status 与 `WWW-Authenticate` Header 同样属于协议契约。

## 生产环境最容易犯的错误

### 把 JWT Decode 当成 Verify

任何人都能构造 JWT Payload。Resource Server 必须验证签名，并固定可信 Issuer、Audience 与允许算法。

### 把 Token 放进 URL

URL 会穿过太多系统。即使业务日志主动脱敏，浏览器历史、代理、CDN、APM 或错误追踪仍可能留下完整 Query String。

### 让 Access Token 永不过期

长期 Bearer Token 接近长期密码。优先使用短期 Access Token，并为 Refresh Token 配置 Rotation 或 Sender Constraint。

### 只验证“有效”，不验证“给谁”

签名有效只说明 Token 来自某个密钥，不说明它是签给当前 API 的。`aud` 校验用于阻止 Token Redirect 和跨服务误用。

### 在日志里打印整个 Header

应用日志、网关日志和异常上下文都应脱敏 `Authorization`、Cookie、Refresh Token 等字段。排障通常只需要 Token 的哈希摘要、`kid`、Issuer、Subject 和错误类型。

### 用设备 ID 代替 Bearer Token

设备序列号、随机安装 ID 和浏览器指纹可以帮助审计来源，但它们通常不是服务端签发、可过期、可限制 Scope 的安全凭证。它们回答“可能来自哪个安装实例”，不能独立证明“调用方被授权访问什么”。

## 2026 年还要继续用 Bearer Token 吗

Bearer Token 仍然是主流且实用的 API 访问方式。它的优势是协议简单、工具链成熟、跨语言互操作好。多数普通业务 API，只要做到 HTTPS、短有效期、Audience 限制、最小 Scope、可靠密钥轮换和日志脱敏，就能建立清晰的安全边界。

高价值数据、金融操作或泄露后影响很大的 API，需要进一步考虑 Sender-Constrained Access Token。[RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.html) 建议使用 mTLS 或 DPoP 等机制，把 Token 绑定到特定发送方，从而降低被盗 Token 单独重放的风险。

## 要不要用：我的判断框架

### 值得用

系统已经有 OAuth 2.0 / OIDC Provider，需要为 Web、移动端、CLI、Agent 或服务间调用提供统一 API 授权；团队能够管理 Token 生命周期、Scope、Audience 和密钥轮换。

### 可以简化

单体 Web 应用只有浏览器页面和同域后端，成熟的 Server Session + `Secure`、`HttpOnly`、`SameSite` Cookie 可能更直接。不要为了“前后端分离”机械引入一套不完整的 OAuth 系统。

### 需要升级

Token 一旦泄露就可能造成重大损失，或 API 跨越不可信终端和网络边界时，应评估 DPoP、mTLS、BFF、Token Exchange 和更严格的 Audience 隔离。Bearer 的易用性来自“持有即可使用”，风险也来自同一个性质。

我的判断是：**Bearer Token 适合做短期、最小权限、可验证的访问凭证，不适合充当永不过期的万能身份。**

## 参考资料

1. [RFC 6750：The OAuth 2.0 Authorization Framework: Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html)
2. [RFC 9700：Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html)
3. [RFC 9068：JSON Web Token Profile for OAuth 2.0 Access Tokens](https://www.rfc-editor.org/rfc/rfc9068.html)
4. [`jose`：JWT Verification 与 Remote JWKS](https://github.com/panva/jose/blob/main/docs/jwks/remote/functions/createRemoteJWKSet.md)

> 本文使用 [writting-skill](https://github.com/zisheng-ai/writting-skill) 辅助写作。项目已开源，欢迎在 GitHub 点个 Star。
