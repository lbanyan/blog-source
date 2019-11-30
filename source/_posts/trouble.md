---
title: 故障定位分析及解决
date: 2019-11-19 21:24
---

### 案例一

线上用户公钥服务，收到如下告警：
``` java
告警分派:
严重 健康拨测
2019-11-13 15:31:46: 任务名(secretkey-jingwei.huya.com_16033),域名(secretkey-jingwei.huya.com),路径(/), 健康拨测： ip：10.185.32.105, port:8080, 多次拨测失败(3次)

【此告警需要处理，请点击下方我的告警或登录告警平台进行操作】
```
收到告警的第一反应应该是当前是否有人在发版，近期是否是有代码变更。此处均没有。
到监控系统查询WEB质量情况，关注的点由前到后分为（故障发生前后时段）：请求总数、请求\响应时间、分机请求数、分机请求响应时间、URI请求分布、URI返回码分布、接入层WEB专区\Nginx统计数据等。
从这里看出本告警是由于该服务/downloadUserKey接口负载过高导致，如下图所示：
![](/img/trouble/downloadUserKey.png)
但该接口之前以做过优化，增加了JVM和DB间的Redis缓存，以及JVM缓存。初步怀疑存在缓存穿透问题。

<!--more-->

使用Arthas trace命令对该接口进行分析，分析结果如下：
在缓存未命中下的执行情况：
![](/img/trouble/trace_no_cache.png)
假设40ms处理一个，则1s可以处理25个，4台机，单机4核，即使跑满，也只能达到400qps，这还未算服务器自身损耗。
缓存命中时的执行情况：
![](/img/trouble/trace_cache.png)
跟相关研发同学沟通后，发现其缓存设计问题，其设置了缓存失效时间，但在缓存失效后，未主动拉新，导致上述问题发生。

修复后，问题依然出现。
使用Jmeter对该系统进行压测，压测结果如下：
并发100时，压测结果：
![](/img/trouble/secretkey_stress1.png)
并发200时，压测结果：
![](/img/trouble/secretkey_stress2.png)
发现系统可达到1300qps以上，那理论上完全可以抗住当前的压力，我们看一下压测当时和平时的监控情况：
![](/img/trouble/secretkey_req.png)
![](/img/trouble/secretkey_req1.png)
![](/img/trouble/secretkey_req2.png)
![](/img/trouble/secretkey_req3.png)
DB情况：
![](/img/trouble/secretkey_db.png)
![](/img/trouble/secretkey_db1.png)
单机监控：
![](/img/trouble/secretkey_server1.png)
![](/img/trouble/secretkey_server2.png)
![](/img/trouble/secretkey_server3.png)
![](/img/trouble/secretkey_server4.png)
![](/img/trouble/secretkey_server5.png)
![](/img/trouble/secretkey_server6.png)
查看操作系统连接数限制
``` bash
sh-4.3# ulimit -n
1048576
```
查看当前连接数
``` bash
sh-4.3# netstat -nat|wc -l
27
```

```bash
sh-4.3# ps aux | grep java
root         45  3.9  1.9 17115352 1304796 pts/0 Sl+ 20:43   1:59 /usr/local/java/jre/bin/java -Djava.util.logging.config.file=/usr/local/tomcat/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Xmx2688M -Xms2688M -Xmn960M -XX:MaxMetaspaceSize=512M -XX:MetaspaceSize=512M -XX:+UseConcMarkSweepGC -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70 -XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses -XX:+CMSClassUnloadingEnabled -XX:+ParallelRefProcEnabled -XX:+CMSScavengeBeforeRemark -Djdk.tls.ephemeralDHKeySize=2048 -Djava.endorsed.dirs=/usr/local/tomcat/endorsed -classpath /usr/local/tomcat/bin/bootstrap.jar:/usr/local/tomcat/bin/tomcat-juli.jar -Dcatalina.base=/usr/local/tomcat -Dcatalina.home=/usr/local/tomcat -Djava.io.tmpdir=/usr/local/tomcat/temp org.apache.catalina.startup.Bootstrap start
```

GC相关
``` bash
ID             NAME                                       GROUP                         PRIORITY      STATE          %CPU          TIME           INTERRUPTED   DAEMON
769            Timer-for-arthas-dashboard-21dee698-865d-4 system                        10            RUNNABLE       41            0:0            false         true
206            http-apr-8080-exec-163                     main                          5             RUNNABLE       35            0:0            false         true
171            http-apr-8080-exec-128                     main                          5             RUNNABLE       17            0:0            false         true
36             http-apr-8080-Poller                       main                          5             TIMED_WAITING  2             1:0            false         true
38             http-apr-8080-Acceptor-0                   main                          5             RUNNABLE       1             0:45           false         true
31             Abandoned connection cleanup thread        main                          5             TIMED_WAITING  0             0:15           false         true
757            AsyncAppender-Worker-arthas-cache.result.A system                        9             WAITING        0             0:0            false         true
755            Attach Listener                            system                        9             RUNNABLE       0             0:0            false         true
27             C3P0PooledConnectionPoolManager[identityTo main                          5             TIMED_WAITING  0             0:0            false         true
28             C3P0PooledConnectionPoolManager[identityTo main                          5             TIMED_WAITING  0             0:4            false         true
29             C3P0PooledConnectionPoolManager[identityTo main                          5             TIMED_WAITING  0             0:4            false         true
30             C3P0PooledConnectionPoolManager[identityTo main                          5             TIMED_WAITING  0             0:4            false         true
35             ContainerBackgroundProcessor[StandardEngin main                          5             TIMED_WAITING  0             0:2            false         true
3              Finalizer                                  system                        8             WAITING        0             0:0            false         true
21             GC Daemon                                  system                        2             TIMED_WAITING  0             0:0            false         true
2              Reference Handler                          system                        10            WAITING        0             0:0            false         true
5              Signal Dispatcher                          system                        9             RUNNABLE       0             0:0            false         true
32             Thread-8                                   main                          5             TIMED_WAITING  0             0:0            false         true
48             ajp-apr-8009-Acceptor-0                    main                          5             RUNNABLE       0             0:0            false         true
50             ajp-apr-8009-AsyncTimeout                  main                          5             TIMED_WAITING  0             0:1            false         true
Memory                                used        total        max         usage        GC
heap                                  380M        2592M        2592M       14.67%       gc.parnew.count                             12
par_eden_space                        172M        768M         768M        22.47%       gc.parnew.time(ms)                          1461
par_survivor_space                    96M         96M          96M         100.00%      gc.concurrentmarksweep.count                0
cms_old_gen                           111M        1728M        1728M       6.46%        gc.concurrentmarksweep.time(ms)             0
nonheap                               69M         93M          1776M       3.91%
code_cache                            16M         39M          240M        6.74%
metaspace                             48M         49M          512M        9.38%
compressed_class_space                5M          5M           1024M       0.50%
direct                                39M         39M          -           100.00%
Runtime
os.name                                                                                 Linux
os.version                                                                              4.9.144-huya
java.version                                                                            1.8.0_112
java.home                                                                               /usr/local/java/jre
systemload.average                                                                      1.22
processors                                                                              24
uptime                                                                                  83663s
```

``` bash
sh-4.3# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
SYN_RECV 14
CLOSE_WAIT 105
ESTABLISHED 902
TIME_WAIT 439
```

```bash
sh-4.3# netstat -n | awk '/^tcp/'
tcp        0      0 10.185.34.137:50212     10.64.34.42:6970        ESTABLISHED
tcp        0      0 10.185.34.137:8080      10.68.9.209:24268       SYN_RECV
tcp        0      0 10.185.34.137:8080      10.68.33.157:47608      SYN_RECV
tcp        0      0 10.185.34.137:8080      10.68.9.207:48366       SYN_RECV
tcp        0      0 10.185.34.137:8080      10.68.33.249:38658      SYN_RECV
tcp        0      0 10.185.34.137:44134     10.64.34.42:6970        TIME_WAIT
tcp        0      0 10.185.34.137:8080      10.207.112.232:53292    TIME_WAIT
tcp        0      0 10.185.34.137:8080      10.68.9.209:24106       SYN_RECV
tcp        0      0 10.185.34.137:8080      10.68.9.207:48648       SYN_RECV
tcp        0      0 10.185.34.137:8080      10.68.9.208:45416       SYN_RECV
tcp        0      0 10.185.34.137:8080      10.68.33.156:56876      SYN_RECV
tcp        0      0 10.185.34.137:8080      10.216.230.60:56042     TIME_WAIT
tcp        0      0 10.185.34.137:8080      10.216.220.42:35334     ESTABLISHED
tcp        0      0 10.185.34.137:43034     10.64.34.42:6970        TIME_WAIT
tcp        0      0 10.185.34.137:45248     10.64.34.42:6970        TIME_WAIT
tcp        0      0 10.185.34.137:8080      10.68.33.157:50630      SYN_RECV
tcp        0      0 10.185.34.137:8080      10.68.33.156:57362      SYN_RECV
tcp        0      0 10.185.34.137:8080      10.68.33.249:38944      SYN_RECV
tcp        0      0 10.185.34.137:8080      10.68.9.208:45190       SYN_RECV
tcp        0      0 10.185.34.137:41862     10.64.34.42:6970        TIME_WAIT
tcp        0      0 10.185.34.137:8080      10.68.9.207:54932       SYN_RECV
tcp        0      0 10.185.34.137:51466     10.64.34.42:6970        ESTABLISHED
tcp        0      0 10.185.34.137:8080      10.68.33.250:47962      SYN_RECV
tcp        0      0 10.185.34.137:48956     10.64.34.42:6970        ESTABLISHED
tcp        0      0 10.185.34.137:47670     10.64.34.42:6970        ESTABLISHED
```


``` bash
sh-4.3# tcpdump host 10.206.118.109
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
12:36:44.519595 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 1439, win 32, options [nop,nop,TS val 985558793 ecr 988778448], length 0
12:36:44.519677 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 1444, win 32, options [nop,nop,TS val 985558793 ecr 988778448], length 0
12:36:47.500310 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [P.], seq 756:1094, ack 1444, win 32, options [nop,nop,TS val 985559539 ecr 988778448], length 338: HTTP: GET /ws/secretKey/downloadUserKey?passport=wudaqing1 HTTP/1.1
12:36:47.500495 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.34892: Flags [P.], seq 1444:2163, ack 1094, win 31, options [nop,nop,TS val 988779195 ecr 985559539], length 719: HTTP: HTTP/1.1 200 OK
12:36:47.500542 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.34892: Flags [P.], seq 2163:2168, ack 1094, win 31, options [nop,nop,TS val 988779195 ecr 985559539], length 5: HTTP
12:36:47.507821 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 2163, win 33, options [nop,nop,TS val 985559541 ecr 988779195], length 0
12:36:47.507976 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 2168, win 33, options [nop,nop,TS val 985559541 ecr 988779195], length 0
12:36:49.807399 IP 10.206.118.109.59156 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [P.], seq 1103:1526, ack 2209, win 64, options [nop,nop,TS val 985560115 ecr 988778442], length 423: HTTP: GET /ws/secretKey/downloadUserKey?passport=liuzheng1 HTTP/1.1
12:36:49.807590 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.59156: Flags [P.], seq 2209:2929, ack 1526, win 49, options [nop,nop,TS val 988779772 ecr 985560115], length 720: HTTP: HTTP/1.1 200 OK
12:36:49.807640 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.59156: Flags [P.], seq 2929:2934, ack 1526, win 49, options [nop,nop,TS val 988779772 ecr 985560115], length 5: HTTP
12:36:49.814826 IP 10.206.118.109.59156 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 2929, win 65, options [nop,nop,TS val 985560117 ecr 988779772], length 0
12:36:49.814831 IP 10.206.118.109.59156 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 2934, win 65, options [nop,nop,TS val 985560117 ecr 988779772], length 0
12:36:51.001125 IP 10.206.118.109.57498 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [S], seq 2531574600, win 42340, options [mss 1424,sackOK,TS val 985560414 ecr 0,nop,wscale 11], length 0
12:36:51.001155 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.57498: Flags [S.], seq 3531506922, ack 2531574601, win 43440, options [mss 1460,sackOK,TS val 988780070 ecr 985560414,nop,wscale 11], length 0
12:36:51.009187 IP 10.206.118.109.57498 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 1, win 21, options [nop,nop,TS val 985560416 ecr 988780070], length 0
12:36:51.009193 IP 10.206.118.109.57498 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [P.], seq 1:426, ack 1, win 21, options [nop,nop,TS val 985560416 ecr 988780070], length 425: HTTP: GET /ws/secretKey/downloadUserKey?passport=jiangwei HTTP/1.1
12:36:51.009230 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.57498: Flags [.], ack 426, win 22, options [nop,nop,TS val 988780072 ecr 985560416], length0
12:36:51.009584 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.57498: Flags [P.], seq 1:720, ack 426, win 22, options [nop,nop,TS val 988780072 ecr 985560416], length 719: HTTP: HTTP/1.1 200 OK
12:36:51.009631 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.57498: Flags [P.], seq 720:725, ack 426, win 22, options [nop,nop,TS val 988780072 ecr 985560416], length 5: HTTP
12:36:51.017606 IP 10.206.118.109.57498 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 720, win 22, options [nop,nop,TS val 985560418 ecr 988780072], length0
12:36:51.017615 IP 10.206.118.109.57498 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 725, win 22, options [nop,nop,TS val 985560418 ecr 988780072], length0
12:36:51.845709 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [P.], seq 1094:1515, ack 2168, win 33, options [nop,nop,TS val 985560625 ecr 988779195], length 421: HTTP: GET /ws/secretKey/downloadUserKey?passport=zhusong HTTP/1.1
12:36:51.845863 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.34892: Flags [P.], seq 2168:2885, ack 1515, win 31, options [nop,nop,TS val 988780281 ecr 985560625], length 717: HTTP: HTTP/1.1 200 OK
12:36:51.845919 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.34892: Flags [P.], seq 2885:2890, ack 1515, win 31, options [nop,nop,TS val 988780281 ecr 985560625], length 5: HTTP
12:36:51.853207 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 2885, win 34, options [nop,nop,TS val 985560627 ecr 988780281], length 0
12:36:51.853212 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 2890, win 34, options [nop,nop,TS val 985560627 ecr 988780281], length 0
12:37:00.076391 IP 10.206.118.109.59156 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [P.], seq 1526:1955, ack 2934, win 65, options [nop,nop,TS val 985562683 ecr 988779772], length 429: HTTP: GET /ws/secretKey/downloadUserKey?passport=zhangshuisen HTTP/1.1
12:37:00.076509 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.59156: Flags [P.], seq 2934:3657, ack 1955, win 50, options [nop,nop,TS val 988782339 ecr 985562683], length 723: HTTP: HTTP/1.1 200 OK
12:37:00.076557 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.59156: Flags [P.], seq 3657:3662, ack 1955, win 50, options [nop,nop,TS val 988782339 ecr 985562683], length 5: HTTP
12:37:00.083725 IP 10.206.118.109.59156 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 3657, win 66, options [nop,nop,TS val 985562685 ecr 988782339], length 0
12:37:00.083733 IP 10.206.118.109.59156 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 3662, win 66, options [nop,nop,TS val 985562685 ecr 988782339], length 0
12:37:01.189295 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.43338: Flags [F.], seq 215, ack 341, win 22, options [nop,nop,TS val 988782617 ecr 985557762], length 0
12:37:01.196717 IP 10.206.118.109.43338 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [F.], seq 341, ack 216, win 22, options [nop,nop,TS val 985562963 ecr 988782617], length 0
12:37:01.196737 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.43338: Flags [.], ack 342, win 22, options [nop,nop,TS val 988782619 ecr 985562963], length0
12:37:02.321233 IP 10.206.118.109.44408 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [S], seq 1500094426, win 42340, options [mss 1424,sackOK,TS val 985563244 ecr 0,nop,wscale 11], length 0
12:37:02.321269 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.44408: Flags [S.], seq 2922143463, ack 1500094427, win 43440, options [mss 1460,sackOK,TS val 988782900 ecr 985563244,nop,wscale 11], length 0
12:37:02.328459 IP 10.206.118.109.44408 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 1, win 21, options [nop,nop,TS val 985563246 ecr 988782900], length 0
12:37:02.328464 IP 10.206.118.109.44408 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [P.], seq 1:338, ack 1, win 21, options [nop,nop,TS val 985563246 ecr 988782900], length 337: HTTP: GET /ws/secretKey/downloadUserKey?passport=chendechang HTTP/1.1
12:37:02.328513 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.44408: Flags [.], ack 338, win 22, options [nop,nop,TS val 988782902 ecr 985563246], length0
12:37:02.328883 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.44408: Flags [P.], seq 1:723, ack 338, win 22, options [nop,nop,TS val 988782902 ecr 985563246], length 722: HTTP: HTTP/1.1 200 OK
12:37:02.328933 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.44408: Flags [P.], seq 723:728, ack 338, win 22, options [nop,nop,TS val 988782902 ecr 985563246], length 5: HTTP
12:37:02.336134 IP 10.206.118.109.44408 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 723, win 22, options [nop,nop,TS val 985563248 ecr 988782902], length0
12:37:02.336140 IP 10.206.118.109.44408 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 728, win 22, options [nop,nop,TS val 985563248 ecr 988782902], length0
12:37:04.465353 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [P.], seq 1515:1847, ack 2890, win 34, options [nop,nop,TS val 985563780 ecr 988780281], length 332: HTTP: GET /ws/secretKey/downloadUserKey?passport=huhang HTTP/1.1
12:37:04.465595 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.34892: Flags [P.], seq 2890:3607, ack 1847, win 32, options [nop,nop,TS val 988783436 ecr 985563780], length 717: HTTP: HTTP/1.1 200 OK
12:37:04.465641 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.34892: Flags [P.], seq 3607:3612, ack 1847, win 32, options [nop,nop,TS val 988783436 ecr 985563780], length 5: HTTP
12:37:04.472930 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 3607, win 34, options [nop,nop,TS val 985563782 ecr 988783436], length 0
12:37:04.472935 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 3612, win 34, options [nop,nop,TS val 985563782 ecr 988783436], length 0
12:37:06.670428 IP 10.206.118.109.59156 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [P.], seq 1955:2284, ack 3662, win 66, options [nop,nop,TS val 985564331 ecr 988782339], length 329: HTTP: GET /ws/secretKey/downloadUserKey?passport=wangle HTTP/1.1
12:37:06.670571 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.59156: Flags [P.], seq 3662:4378, ack 2284, win 50, options [nop,nop,TS val 988783987 ecr 985564331], length 716: HTTP: HTTP/1.1 200 OK
12:37:06.670617 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.59156: Flags [P.], seq 4378:4383, ack 2284, win 50, options [nop,nop,TS val 988783987 ecr 985564331], length 5: HTTP
12:37:06.677684 IP 10.206.118.109.59156 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 4378, win 67, options [nop,nop,TS val 985564333 ecr 988783987], length 0
12:37:06.677761 IP 10.206.118.109.59156 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 4383, win 67, options [nop,nop,TS val 985564333 ecr 988783987], length 0
12:37:09.315702 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [P.], seq 1847:2184, ack 3612, win 34, options [nop,nop,TS val 985564992 ecr 988783436], length 337: HTTP: GET /ws/secretKey/downloadUserKey?passport=liuwenxu HTTP/1.1
12:37:09.315942 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.34892: Flags [P.], seq 3612:4330, ack 2184, win 32, options [nop,nop,TS val 988784649 ecr 985564992], length 718: HTTP: HTTP/1.1 200 OK
12:37:09.315996 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.34892: Flags [P.], seq 4330:4335, ack 2184, win 32, options [nop,nop,TS val 988784649 ecr 985564992], length 5: HTTP
12:37:09.323353 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 4330, win 35, options [nop,nop,TS val 985564994 ecr 988784649], length 0
12:37:09.323393 IP 10.206.118.109.34892 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 4335, win 35, options [nop,nop,TS val 985564994 ecr 988784649], length 0
12:37:11.371036 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.57498: Flags [F.], seq 725, ack 426, win 22, options [nop,nop,TS val 988785162 ecr 985560418], length 0
12:37:11.379069 IP 10.206.118.109.57498 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [F.], seq 426, ack 726, win 22, options [nop,nop,TS val 985565508 ecr 988785162], length 0
12:37:11.379103 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.57498: Flags [.], ack 427, win 22, options [nop,nop,TS val 988785164 ecr 985565508], length0
12:37:14.370958 IP 10.206.118.109.59156 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [P.], seq 2284:2617, ack 4383, win 67, options [nop,nop,TS val 985566256 ecr 988783987], length 333: HTTP: GET /ws/secretKey/downloadUserKey?passport=yuyang2 HTTP/1.1
12:37:14.371187 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.59156: Flags [P.], seq 4383:5100, ack 2617, win 51, options [nop,nop,TS val 988785912 ecr 985566256], length 717: HTTP: HTTP/1.1 200 OK
12:37:14.371237 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.59156: Flags [P.], seq 5100:5105, ack 2617, win 51, options [nop,nop,TS val 988785912 ecr 985566256], length 5: HTTP
12:37:14.378376 IP 10.206.118.109.59156 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 5100, win 67, options [nop,nop,TS val 985566258 ecr 988785912], length 0
12:37:14.378381 IP 10.206.118.109.59156 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 5105, win 67, options [nop,nop,TS val 985566258 ecr 988785912], length 0
12:37:16.071697 IP 10.206.118.109.34712 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [S], seq 3397934625, win 42340, options [mss 1424,sackOK,TS val 985566681 ecr 0,nop,wscale 11], length 0
12:37:16.071728 IP secretkey-secretkey-wx-42061-7v6sq.hy.http-alt > 10.206.118.109.34712: Flags [S.], seq 67059285, ack 3397934626, win 43440, options [mss 1460,sackOK,TS val988786338 ecr 985566681,nop,wscale 11], length 0
12:37:16.079076 IP 10.206.118.109.34712 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [.], ack 1, win 21, options [nop,nop,TS val 985566683 ecr 988786338], length 0
12:37:16.079082 IP 10.206.118.109.34712 > secretkey-secretkey-wx-42061-7v6sq.hy.http-alt: Flags [P.], seq 1:329, ack 1, win 21, options [nop,nop,TS val 985566683 ecr 988786338], length 328: HTTP: GET /ws/secretKey/getUid?passport=huangkiajia HTTP/1.1
```

QPS不高，并发高导致

服务接口已调优->JVM已调优

考虑问题是否出在Tomcat这里

docker Tomcat路径
docker tomcat路径
/usr/local/tomcat/conf

线上使用了默认配置
Tomcat 7 文档
https://tomcat.apache.org/tomcat-7.0-doc/config/http.html
acceptCount、maxConnections、maxThreads

更改dockerfile，以对Tomcat配置优化
``` bash
#2019/11/30更新：Tomcat调优
FROM registry-haiyan.local.huya.com/library/tomcat:7.0.75-jdk1.8.0_112-ubuntu16.04-1.0.4

ENV SCRIPT_URL http://athena-registry-package.hys3.huya.com/scripts/athena_clean_data.sh
ENV SCRIPT_DIR /scripts/athena_clean_data.sh
ENV APP_NAME secretkey

RUN cd /etc/apt \
&& wget -O sources.list http://athena-registry-package.hys3.huya.com/ubuntu/sources.list

RUN apt-get clean \
&& apt-get update \
&& apt-get -y install jq \
&& rm -rf /var/lib/apt/lists/*

RUN set -eux; \
mkdir -p /scripts; \
cd /scripts; \
wget -O ROOT.war http://athena-artifacts.hys3.huya.com/maven/secretkey-633-1/20191128221708-23-rfbf444be57015ddb0c54ef1d69a4e2e34159f8fb/secretkey-admin-1.0.0-SNAPSHOT.war; \
wget -O $SCRIPT_DIR $SCRIPT_URL \
&& chmod +x $SCRIPT_DIR; \
wget -O cmd_init.sh http://athena-registry-package.hys3.huya.com/scripts/cmd_init.sh; \
sed -i "s#appBase=\"webapps\"#appBase=\"/data/app/secretkey\"#g" /usr/local/tomcat/conf/server.xml; \
sed -i "s#protocol=\"HTTP/1.1\"#protocol=\"org.apache.coyote.http11.Http11NioProtocol\" acceptCount=\"1000\" maxThreads=\"1000\" minSpareThreads=\"100\" maxConnections=\"1000\" maxHttpHeaderSize=\"8192\" tcpNoDelay=\"true\" compression=\"on\" compressionMinSize=\"2048\" disableUploadTimeout=\"true\" enableLookups=\"false\" URIEncoding=\"UTF-8\"#g" /usr/local/tomcat/conf/server.xml; \
chmod 755 $JAVA_HOME/bin;
```

优化后结果截图：



### 案例二

混合云交付平台告警：
``` java
告警分派:
警告 web质量运营告警
告警级别:告警 
域名:autoidc-jingwei.huya.com 
Note:autoidc-jingwei.huya.com4XX请求数每分钟大于60 
最近3个值为:70.00 , 341.00 , 146.00 
Timestamp:2019-11-15 09:32:00 
https://skynet-jingwei.huya.com/web.html#/domain/autoidc-jingwei.huya.com/summary?type=webproxy

【此告警需要处理，请点击下方我的告警或登录告警平台进行操作】
```
根据之前的分析路数，查询到下列异常统计数据：
![](/img/trouble/autoidc_5XX.png)
![](/img/trouble/autoidc_4XX.png)
确认是WEB专区问题，跟相关运维同学沟通，确认在该时段，深圳机房WEB专区磁盘高载，导致其服务质量下降，影响到后端服务。

TODO 故障分析流程
代码变动->发布变动->web质量->硬件监控->JVM相关

### 案例三
混合云交付平台性能调优
主机信息：
``` java
CPU: E3-1231 / 3.40GHz / 8核
MEM: 16G
```
旧JVM配置：
``` java
-Xms1024m -Xmx4048m -Duser.timezone=Asia/Shanghai -Xmn1512m -XX:PermSize=192m -Xss256k -XX:+DisableExplicitGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:+UseCMSCompactAtFullCollection -XX:LargePageSizeInBytes=128m -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70 -Djava.awt.headless=true -Djava.net.preferIPv4Stack=true -Duser.timezone=Asia/Shanghai -Dfile.encoding=UTF-8
```
新JVM配置：
``` java
-Xmx8000M -Xms8000M -XX:MaxMetaspaceSize=512M -XX:MetaspaceSize=512M -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:+ParallelRefProcEnabled -Dfile.encoding=UTF-8 -Duser.timezone=Asia/Shanghai -XX:-RestrictContended
```
调整后，请求延时和响应延时均大幅下降40%，见下图：
![](/img/trouble/autoidc_jvm.png)
