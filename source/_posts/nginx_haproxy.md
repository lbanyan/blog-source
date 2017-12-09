---
title: HAproxy和Nginx对比
date: 2017-07-07 19:33
---

### 区别
从定位上来说，Nginx重点是web服务器，替换的是apache，同时具备lb的作用，HAproxy是单纯的lb，可以对照lvs进行比较。
相对于Nginx，HAproxy不支持server_name，实现泛域名解析较为麻烦。
