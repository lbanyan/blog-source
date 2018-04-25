---
title: 常用技巧
date: 2017-11-05 22:05
---

### Linux

#### 日志定位

less warning: may be a binary file
日志中可能含有控制字符
```
less -f -r test.log
grep 'ERROR' --color test.log
less -f -r test.log | grep 'ERROR' | grep '2018-04-24 23:55'
```

#### redis-cli

中文乱码
```
redis-cli -h **** -p **** -a **** --raw
```
