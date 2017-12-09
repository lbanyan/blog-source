---
title: Linux 获取ms毫秒级时间戳
date: 2017-05-10 19:36
tags:
    - Linux
    - 时间戳
---

``` bash
echo $(($(date +%s%N)/1000000))
```

