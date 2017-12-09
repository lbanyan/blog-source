---
title: 生产环境MySQL备份表的一种实现
date: 2017-09-27 17:36
tags:
    - MySQL
    - 备份
---

### MySQL表备份实现
可以在服务器上定时跑脚本来实现MySQL表的定时备份。
``` bash
#!/bin/bash
#day=`date +%Y%m%d`
day=`date +%Y%m%d "-d-1 days"`
time=`date +%s%N`
now=$time/1000000
table_name=RequestLog$day
/usr/local/mysql/bin/mysql -Drise_db -e "create table $table_name like RequestLog"
/usr/local/mysql/bin/mysql -Drise_db -e "insert into $table_name select * from RequestLog"
if [ $? -eq 0 ];then
 /usr/local/mysql/bin/mysql -Drise_db -e "delete from RequestLog where createTime < $now"
fi
#每天凌晨4点执行
#0 4 * * * root /data/mlrequestlog/backup_ml_request_log.sh >/tmp/backup_ml_rquest_log.log 2>&1
```

### 题外话
如果需要清空表，可以使用truncate，相比于delete，速度快，且不是事务方式。

### 参考文档
[MySQL 删除数据 Delete 语句 、Truncate 语句](http://www.xiaoxiaozi.com/2009/09/03/1427/)
