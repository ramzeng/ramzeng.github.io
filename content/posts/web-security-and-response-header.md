---
title: "安全相关的 HTTP 响应头"
date: 2024-03-18T11:15:17+08:00
draft: false
tags: ["安全", "面试"]
---
## Strict-Transport-Security

Strict-Transport-Security 是一个安全相关的 HTTP 响应头，它可以让网站要求浏览器只通过 HTTPS 访问它，以防止中间人攻击。它的值是一个时间，单位是秒。例如：

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

## X-Frame-Options

X-Frame-Options 是一个安全相关的 HTTP 响应头，它可以用来控制网站在 iframe 中的显示情况。它有三个值：

- DENY：表示页面不允许在 iframe 中显示
- SAMEORIGIN：表示页面可以在相同域名下的 iframe 中显示
- ALLOW-FROM uri：表示页面可以在指定来源的 iframe 中显示

## X-XSS-Protection

X-XSS-Protection 是一个安全相关的 HTTP 响应头，它可以用来控制浏览器的 XSS 防护机制。它有三个值：

- 0：表示关闭浏览器的 XSS 防护机制
- 1：表示开启浏览器的 XSS 防护机制，如果检测到 XSS 攻击，尝试清除不安全的部分，然后加载页面
- 1; mode=block：表示开启浏览器的 XSS 防护机制，并且如果检测到 XSS 攻击，浏览器会停止加载页面

## X-Content-Type-Options

X-Content-Type-Options 是一个安全相关的 HTTP 响应头，它可以用来控制浏览器的 MIME 类型嗅探行为。它有一个值：

- nosniff：表示浏览器不会执行 MIME 类型嗅探，即使服务器返回的 MIME 类型是错误的，浏览器也不会改变它

## Content-Security-Policy
TODO
