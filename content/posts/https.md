---
title: "HTTPS"
date: 2024-03-01T16:51:45+08:00
draft: false
tags: ["HTTP", "面试"]
---
## 连接建立过程

基本流程如下：

1. 客户端向服务器索要并验证服务器的公钥
2. 双方协商产生对称密钥
3. 双方采用对称密钥进行加密通信

TLS 的握手阶段涉及四次通信，使用不同的密钥交换算法，流程也会有所不同。下面以 RSA 密钥交换算法为例，介绍 HTTPS 连接建立过程。

![握手流程](https://cdn4.codesign.qq.com/materials/2024/03/01/GD5Oj24EPoJE03Z3eAX01/uvghofdlvqne8h0r/a32256fa-5a0f-4d32-b45e-d0fa99979e24.webp)

### Client Hello

客户端向服务器发送一个 Client Hello 消息，包含以下信息：

- 客户端支持的 TLS 版本
- 客户端生成的一个随机数（Client Random）
- 客户端支持的加密算法，如 RSA、DH、ECDH 等

### Server Hello

服务器收到 Client Hello 消息后，向客户端发送一个 Server Hello 消息，包含以下信息：

- 服务器选择的 TLS 版本
- 服务器生成的一个随机数（Server Random）
- 服务器选择的加密算法
- 服务器的数字证书

### 客户端回应

客户端收到 Server Hello 消息后，会执行以下操作：

1. 通过浏览器或者 OS 中的 CA 公钥验证服务器的数字证书
2. 从服务器的数字证书中提取公钥，用于后续的通信
3. 生成一个随机数（Pre-Master Secret），并使用服务器的公钥加密该随机数
4. 向服务器发送消息，包含以下信息
   1. 加密后的 Pre-Master Secret
   2. 加密算法改变通知，表示后续的通信将使用对称密钥加密
   3. 客户端的 Finished 消息，包含一个验证数据，用于验证双方是否使用相同的密钥

### 服务器回应

服务器收到客户端的 Pre-Master Secret 后，会执行以下操作：

1. 使用私钥解密 Pre-Master Secret
2. 使用 Client Random、Server Random 和 Pre-Master Secret 生成对称密钥（Master Secret）
3. 向客户端发送消息，包含以下信息
   1. 加密算法改变通知，表示后续的通信将使用对称密钥加密
   2. 服务器的 Finished 消息，包含一个验证数据，用于验证双方是否使用相同的密钥

