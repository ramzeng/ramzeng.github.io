---
title: "API 重放攻击防御"
date: 2023-08-27T22:49:52+08:00
draft: false
tags: ["API", "安全"]
---
## 前置
API 签名：是指在 API 请求中加入签名参数，服务端根据签名参数校验请求是否合法的一种方式。API 签名的实现方式有很多种，本文不做讨论。

## 介绍
API 重放攻击是指攻击者通过截获合法用户的请求，然后再次发送给服务端，从而达到攻击目的的一种攻击方式。

## Timestamp 方案
```bash
curl example.com/api?timestamp=xxx&signature=xxx
```
### 流程
- 发起请求时，将当前时间戳作为参数传递给服务端
- 服务端校验时间戳是否在合理范围内

### 问题
无法保证请求仅一次有效，服务端允许的合理范围内，可以多次发起请求

## Nonce 方案
```bash
curl example.com/api?nonce=xxx&signature=xxx
```
### 流程
- 发起请求时，将随机字符串作为参数传递给服务端
- 服务端校验随机字符串是否已经使用过

### 问题
服务端需要保存使用过的随机字符串，存在数据膨胀问题

## Timestamp + Nonce 方案
```bash
curl example.com/api?timestamp=xxx&nonce=xxx&signature=xxx
```
### 流程
- 发起请求时，将当前时间戳和随机字符串作为参数传递给服务端
- 服务端校验时间戳是否在合理范围内，随机字符串是否已经使用过

此方案可以保证请求仅一次有效，且数据膨胀问题可控，服务端仅需保存一段时间内的随机字符串即可。