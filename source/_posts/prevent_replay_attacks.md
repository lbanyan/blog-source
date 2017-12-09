---
title: 防止重放攻击
date: 2017-09-23 19:31
tags:
    - 重放攻击
---

### 背景
重放攻击，即攻击者在获取一次正确请求的URL及各参数后，使用该URL及参数对服务端多次请求的一种攻击行为。
例如在IaaS平台，用户创建云主机的行为，攻击者采取重放攻击，会导致该用户账户下出现大量恶意创建的云主机，该行为不仅影响用户的账户费用，同时对IaaS平台造成不稳定的可能性。

<!--more-->

### 解决方案
HTTP请求用于验证请求合法性的传参：
```
nonce:u27dj1ls09djzxpo 随机字符串 
secretId:12345678 API密钥ID 
signature:xxxxxxxxx 生成的签名 
timestamp:1445184000000 毫秒时间戳
```
在传参中增加一个nonce的随机字符串，服务端在处理了该nonce对应的请求后，后期如果出现重放攻击，将直接拒绝该nonce的请求。

### 题外话——签名验证
服务端会给每个用户分配一个secretId（密钥Id）和secretKey（私钥），用于在用户API请求时签名及服务端识别用户。用户每次发送请求前生成一个随机的字符串nonce和当前时间的毫秒级时间戳timestamp。
1. 将nonce（随机字符串）、密钥Id（secretId）、timestamp（时间戳）按照顺序拼接在一起组成一个 **被签名串** 。
2. 将API密钥的私钥 (secretKey) 作为key，生成 **被签名串** 的 HMAC-SHA256 **签名** ，
3. 将 **签名** 进行 Base64 编码

用伪代码描述如下：  
signature=Base64(HmacSha256(nonce+secretId+timestamp,secretKey))
注意：
服务端在做时间戳验证时常犯错误，应使用当前时间-timestamp的绝对值与设置的超时时间比较，以避免服务端时间和客户端时间不一致导致在时间戳这里验证时偶尔失败的情况出现。


