---
title: Linux 获取ms毫秒级时间戳
---

``` bash
echo $(($(date +%s%N)/1000000))
```

