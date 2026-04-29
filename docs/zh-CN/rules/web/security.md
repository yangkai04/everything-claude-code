> 本文件在 [common/security.md](../common/security.md) 的基础上，补充 Web 专项安全内容。

# Web 安全规则（Web Security Rules）

## 内容安全策略（Content Security Policy，CSP）

生产环境必须配置 CSP。

### 基于 Nonce 的 CSP

使用每次请求的随机 nonce 为脚本授权，而非使用 `'unsafe-inline'`。

```text
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{RANDOM}' https://cdn.jsdelivr.net;
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  img-src 'self' data: https:;
  font-src 'self' https://fonts.gstatic.com;
  connect-src 'self' https://*.example.com;
  frame-src 'none';
  object-src 'none';
  base-uri 'self';
```

根据项目实际情况调整来源（origins），不要原封不动地照搬此配置块。

## XSS 防护

- 永远不要注入未经净化的 HTML
- 避免使用 `innerHTML` / `dangerouslySetInnerHTML`，除非已先净化
- 转义动态模板中的值
- 在绝对必要时，使用经过验证的本地净化库（sanitizer）处理用户 HTML

## 第三方脚本（Third-Party Scripts）

- 异步加载
- 从 CDN 提供时使用 SRI（子资源完整性）
- 每季度审查一次
- 对关键依赖，在可行时优先自托管（self-hosting）

## HTTPS 与安全响应头

```text
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

## 表单（Forms）

- 改变状态的表单需要 CSRF 保护
- 提交端点需要限流（rate limiting）
- 客户端和服务端双重验证
- 优先使用蜜罐（honeypot）或轻量反滥用控制，而非默认的重型 CAPTCHA
