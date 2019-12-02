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
收到告警的第一反应应该是当前是否有人在发版，近期是否有代码变更。此处均没有。
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
检查GC情况：
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
基本正常。
使用Jmeter对该系统进行压测，压测结果如下：
并发100时，压测结果：
![](/img/trouble/secretkey_stress1.png)
并发200时，压测结果：
![](/img/trouble/secretkey_stress2.png)
发现系统可达到1300qps以上，那理论上完全可以抗住当前的压力，我们看一下压测当时和平时的监控情况，11月28日 00:41为平时峰值情况：
![](/img/trouble/secretkey_req.png)
![](/img/trouble/secretkey_req1.png)
分钟级请求数峰值为10106次，请求响应时间均超过11s。11月28日 20:54为压测时峰值情况：
![](/img/trouble/secretkey_req2.png)
![](/img/trouble/secretkey_req3.png)
分钟级请求次数远远高于10106次，达到117909次，但请求响应时间仅为43ms。
DB监控数据正常，在平时峰值情况下单机负载极低，压测时单机负载符合预期：
![](/img/trouble/secretkey_server5.png)
![](/img/trouble/secretkey_server6.png)
单机TCP连接数：
![](/img/trouble/secretkey_server3.png)
![](/img/trouble/secretkey_server4.png)
怀疑TCP瞬时连接过高，可能导致服务端无法及时响应。如果是这样的话，之前的压测没有出现此情况也能解释的通，即压测并发不高。为了稳定期间，线上服务暂时不进行高并发压测。
检查操作系统TCP连接数是否受限：
``` bash
sh-4.3# ulimit -n
1048576
```
这个是ok的。
在平时峰值情况下，检查TCP连接情况：
``` bash
sh-4.3# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
SYN_RECV 14
CLOSE_WAIT 105
ESTABLISHED 902
TIME_WAIT 439
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
tcp        0      0 10.185.34.137:8080      10.206.118.109:54932    SYN_RECV
tcp        0      0 10.185.34.137:51466     10.64.34.42:6970        ESTABLISHED
tcp        0      0 10.185.34.137:8080      10.68.33.250:47962      SYN_RECV
......
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
......
```
证实了高峰时段瞬时大量TCP连接，通过tcpdump日志可以看出，JVM处理每个任务基本在2ms以内。
本服务使用的是[Tomcat 7](https://tomcat.apache.org/tomcat-7.0-doc/config/http.html)，尝试Tomcat调优。
Docker默认Tomcat路径：/usr/local/tomcat，查看本服务使用的Tomcat配置server.xml，发现其使用的是默认配置，根据[Tomcat官方文档](https://tomcat.apache.org/tomcat-7.0-doc/config/http.html)，不使用线程池的默认情况下，Tomcat的maxConnections受acceptCount限制，acceptCount默认值为100。当瞬时连接1000时，Tomcat是无法处理过来的，因为它的最大连接设置100。
更改dockerfile，以对Tomcat配置优化：
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
优化前后结果对比，优化前：
![](/img/trouble/secretkey_req4.png)
![](/img/trouble/secretkey_req5.png)
优化后：
![](/img/trouble/secretkey_req6.png)
![](/img/trouble/secretkey_req7.png)
请求响应时间从11s下降到22ms，响应速度提升了5000倍。
#### 总结
这是一个特殊的并发过大，QPS不大的场景。经历了这么多种优化的尝试，也是因为这种场景过于特殊。通常我们遇到的场景都是QPS过大的场景，一般在代码、缓存、JVM层面优化，服务水平扩展来解决这种问题。本例是通过优化Tomcat的maxConnections达到了优化的效果。
一般的调优顺序应为应用调优->JVM调优->Tomcat调优->OS调优等，故障分析流程应为代码变动->发布变动->WEB质量->硬件监控->JVM相关->网络相关等逐级定位，对于涉及DB的，也需要观察其监控信息是否符合预期。

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
