---
layout: post
title: "Hadoop rpc优化- 高并发请求"
description: ""
category: hadoop 
tags: [hadoop,rpc,p1故障]
---
{% include JB/setup %}

前言：让程序员痛苦的事情有很多，比如晚上睡的香香的，报警短信把你叫醒了。让程序员幸福的事也有，比如你成了植物人，报警短信响了，你很有可能会苏醒过来。蛋疼的事情也不少，比如问题解决了，却还找不到原因...

言归正传，问题还要从13年11月17日及11月20日的两次namenode慢引发的P1故障说起。

namenode2在11.17日早上8点左右出现连接异常，服务变慢。

问题现象描述
===

    
![getprotocolversion][getprotocolversion]    
出现问题时getprotocolversion会有大量的增加，当这个上去多时，NN一般都会比较慢，这个问题一直都有。
![netstat][net2]    

```bash
cat /proc/sys/net/ipv4/tcp_max_syn_backlog
16384
```

namenode jstack

```java
"Socket Reader#5 for port 9000" prio=10 tid=0x000000005e8bd000 nid=0x273 waiting for monitor entry [0x0000000044b10000]
   java.lang.Thread.State: BLOCKED (on object monitor)
    at org.apache.hadoop.ipc.Server.closeConnection(Server.java:1305)
    - waiting to lock <0x00002aacbc85e1f8> (a java.util.Collections$SynchronizedList)
    at org.apache.hadoop.ipc.Server.access$1200(Server.java:71)
    at org.apache.hadoop.ipc.Server$Listener.doRead(Server.java:667)
    at org.apache.hadoop.ipc.Server$Listener$Reader.run(Server.java:448)
    - locked <0x00002aacae0fa6c0> (a org.apache.hadoop.ipc.Server$Listener$Reader)

   Locked ownable synchronizers:
    - None
"Socket Reader#4 for port 9000" prio=10 tid=0x000000005e8bb000 nid=0x272 runnable [0x0000000044a0f000]
   java.lang.Thread.State: RUNNABLE
    at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
    at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:210)
    at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:65)
    at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:69)
    - locked <0x00002aacae0fb608> (a sun.nio.ch.Util$2)
    - locked <0x00002aacae0fb620> (a java.util.Collections$UnmodifiableSet)
    - locked <0x00002aacae0f84c0> (a sun.nio.ch.EPollSelectorImpl)
    at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:80)
    at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:84)
    at org.apache.hadoop.ipc.Server$Listener$Reader.run(Server.java:437)
    - locked <0x00002aacae0f0e08> (a org.apache.hadoop.ipc.Server$Listener$Reader)

   Locked ownable synchronizers:
    - None
```

```java
"IPC Server listener on 9000" daemon prio=10 tid=0x0000000059115000 nid=0x4da1 waiting for monitor entry [0x0000000044d37000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at org.apache.hadoop.ipc.Server$Listener$Reader.registerChannel(Server.java:481)
        - waiting to lock <0x00002aabfad62be0> (a org.apache.hadoop.ipc.Server$Listener$Reader)
        at org.apache.hadoop.ipc.Server$Listener.doAccept(Server.java:628)
        at org.apache.hadoop.ipc.Server$Listener.run(Server.java:563)
```

当出现问题时， namenode 9000端口的5个reader线程，有4个都是BLOCKED状态，基本上都是block在closeConnection上； listener线程也经常会出现BLOCKED在registerChannel上

问题分析
===

问题重现
===

代码优化
===

性能测试
===




[getprotocolversion]: /assets/images/rpcimprove/getprotocolversion.jpg
[net2]: /assets/images/rpcimprove/net2.png
[net3]: /assets/images/rpcimprove/net3.png