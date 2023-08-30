---
title: "Web Security"
date: 2023-08-31T00:09:23+08:00
draft: true
tags: ["Web 安全"]
---
## XSS
- 输入验证：确保用户输入数据格式符合预期
- 输出转义：对特殊字符进行转义，防止恶意脚本执行
- HttpOnly：保护 Cookie 信息不被 JavaScript 读取
- Content Security Policy (CSP)：限制浏览器可加载资源
- 限制输入长度：降低恶意脚本执行风险

## CSRF
- 使用 CSRF 令牌：为用户会话生成唯一令牌，验证请求合法性 
- 验证 Referer：检查请求来源，确保可信 
- 使用 SameSite Cookie：限制第三方网站携带 Cookie 
- 使用双重验证：对敏感操作增加验证码或邮箱验证 
- 避免 GET 请求执行敏感操作：使用 POST 请求

## SQL 注入
- 使用预编译语句：使用参数化查询，避免拼接 SQL 语句
- 过滤输入：对用户输入数据进行过滤，确保数据格式符合预期

## SSRF
- 检查协议：限制请求协议，非 HTTP / HTTPS 协议拒绝访问
- 检查 IP：限制请求 IP，避免访问内网
- 检查域名：解析 IP，处理 30x 跳转
