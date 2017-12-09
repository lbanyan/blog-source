---
title: MySQL导入csv乱码
date: 2017-04-07 20:54
tags:
    - MySQL
    - csv
    - 乱码
---

### 解决办法

1. 确认csv编码格式，常见的为gbk编码
2. MySQL导入向导中设置数据源编码格式
![](/img/mysql_import_csv_garbled.png)
3. 此过程不会影响mysql数据表的编码格式，此处仅表示该文件的编码格式
