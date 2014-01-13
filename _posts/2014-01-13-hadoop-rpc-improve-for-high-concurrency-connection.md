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

namenode2在11.17日早上8点左右出现连接异常，服务变慢。下面分别列出各部分的监控数据：

    
![getprotocolversion][getprotocolversion]    
![netstat][net2]    


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

"Socket Reader#3 for port 9000" prio=10 tid=0x000000005e8b9800 nid=0x271 waiting for monitor entry [0x000000004490e000]
   java.lang.Thread.State: BLOCKED (on object monitor)
    at org.apache.hadoop.ipc.Server.closeConnection(Server.java:1305)
    - waiting to lock <0x00002aacbc85e1f8> (a java.util.Collections$SynchronizedList)
    at org.apache.hadoop.ipc.Server.access$1200(Server.java:71)
    at org.apache.hadoop.ipc.Server$Listener.doRead(Server.java:667)
    at org.apache.hadoop.ipc.Server$Listener$Reader.run(Server.java:448)
    - locked <0x00002aacae0f8c40> (a org.apache.hadoop.ipc.Server$Listener$Reader)

   Locked ownable synchronizers:
    - None

"Socket Reader#2 for port 9000" prio=10 tid=0x000000005e962000 nid=0x270 waiting for monitor entry [0x000000004480d000]
   java.lang.Thread.State: BLOCKED (on object monitor)
    at org.apache.hadoop.ipc.Server.closeConnection(Server.java:1305)
    - waiting to lock <0x00002aacbc85e1f8> (a java.util.Collections$SynchronizedList)
    at org.apache.hadoop.ipc.Server.access$1200(Server.java:71)
    at org.apache.hadoop.ipc.Server$Listener.doRead(Server.java:667)
    at org.apache.hadoop.ipc.Server$Listener$Reader.run(Server.java:448)
    - locked <0x00002aacae0f62b8> (a org.apache.hadoop.ipc.Server$Listener$Reader)

   Locked ownable synchronizers:
    - None

"Socket Reader#1 for port 9000" prio=10 tid=0x000000005e960800 nid=0x26f runnable [0x000000004470c000]
   java.lang.Thread.State: RUNNABLE
    at java.util.LinkedList.remove(LinkedList.java:224)
    at java.util.Collections$SynchronizedCollection.remove(Collections.java:1580)
    - locked <0x00002aacbc85e1f8> (a java.util.Collections$SynchronizedList)
    at org.apache.hadoop.ipc.Server.closeConnection(Server.java:1306)
    - locked <0x00002aacbc85e1f8> (a java.util.Collections$SynchronizedList)
    at org.apache.hadoop.ipc.Server.access$1200(Server.java:71)
    at org.apache.hadoop.ipc.Server$Listener.doRead(Server.java:667)
    at org.apache.hadoop.ipc.Server$Listener$Reader.run(Server.java:448)
    - locked <0x00002aacae0f7a98> (a org.apache.hadoop.ipc.Server$Listener$Reader)

   Locked ownable synchronizers:
    - None

```

``` java
public void main() {
 int i = 0;
}

#comment
```

未完待续

[getprotocolversion]: /assets/images/rpcimprove/getprotocolversion.jpg
[net2]: /assets/images/rpcimprove/net2.png
[net3]: /assets/images/rpcimprove/net3.png