---
title: SSH Forward 总结
date: 2018-01-10 17:36
tags:
    - SSH Forward
---

#### 本地端口转发

A、B、C 服务器（场景：A 无法访问 C 的 80，但是 B 可以访问 C 的 80，A 可以 SSH 到 B）
在A上执行命令：
ssh -Nf -L 8888:C的ip:80 B的IP    ## -Nf 使得ssh进入background
作用：
访问A的8888端口就相当于访问C的80端口（端口数据流通过B来转发）（A上的8888是ssh生成的）

#### 远程转发（场景：A 是内网电脑，B 为公网电脑，外网用户要访问到 A）
在 A 上执行命令：
ssh -fTNR 8888:0.0.0.0:80 B的IP
作用：
访问 B 的 8888 端口就相当于访问 A 的 80 端口（B 上的 8888 是 SSH 生成的）
ssh -fTNR 8888:0.0.0.0:80 125.90.88.109
ssh -fTNR *:8888:0.0.0.0:80 125.90.88.109

<font color='grey'>本文由牛逼的艺强提供~</font>