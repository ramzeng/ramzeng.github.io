---
title: "Nginx 入门"
date: 2024-04-24T21:28:14+08:00
draft: true
tags: ["Nginx"]
---
## 应用场景
- 静态资源服务
- 反向代理服务
    - 负载均衡
    - 缓存
- API 服务
    - OpenResty

## Nginx 为什么会出现
- 低效的 Apache，一个连接一个进程
- 互联网数据量快速增长，需要更高效的服务器

## Nginx 主要优点
1. 高并发，高性能
2. 可拓展性好
3. 高可靠性
4. 热部署
5. BSD 许可证

## Nginx 的组成
- 二进制可执行文件
- nginx.conf 配置文件
- access.log 访问日志
- error.log 错误日志

## Nginx 命令
- nginx -s stop 立即停止服务
- nginx -s quit 优雅停止服务
- nginx -s reload 重新加载配置文件
- nginx -s reopen 重新打开日志文件
