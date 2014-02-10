---
layout: post
title: "Hadoop Domain socket"
description: "domain socket"
category: hadoop
tags: [hadoop, rpc, native]
---
{% include JB/setup %}

HDFS 本地读
---

Domain Socket介绍
---

配置
---

```xml
<property>
   <!--是否开启本地读-->
    <name>dfs.client.read.shortcircuit</name>
    <value>true</value>
</property>
<property>
    <name>dfs.domain.socket.path</name>
    <value>/home/hadoop/cluster-data/dfs_domain_socket_path</value>
</property>
<property>
    <name>dfs.client.read.shortcircuit.skip.checksum</name>
    <value>true</value>
</property>

```

性能测试
---
测试工具：Hadoop自带的TestDFSIO进行顺序读测试

+ 读的Buffer Size均为1MB
+ Mapper数与文件数量一致
+ 吞吐量 = 读写总尺寸 / 累加上每个Mapper的执行时间
+ IO平均速率 = 所有Mapper读写速率的平均数
+ IO平均速率标准差： 反应每个Mapper读写速率与平均速率之间的波动情况
+ 执行时长：该MapReduce的耗时

测试环境

+ 2台测试服务器，一台部署jobtraker, 另外一台部署namenode、datanode、tasktracker.
+ java version "1.6.0_32"  Java(TM) SE Runtime Environment (build 1.6.0_32-b05) OpenJDK (Taobao) 64-Bit Server VM (build 20.0-b12-internal, mixed mode)
+ Linux r65g02003.cm10 2.6.18-164.11.1.el5 #1 SMP Wed Jan 6 13:26:04 EST 2010 x86_64 x86_64 x86_64 GNU/Linux
+ Mem: 188G cpu core: 32  Intel(R) Xeon(R) CPU E5-2690 0 @ 2.90GHz
+ bin/hadoop jar hadoop-0.19.1-dc-mapred-test.jar TestDFSIO -read -nrFiles $nrFiles -fileSize $fileSize

**hadoop version : yunti-100.9 patch domain socket**

nrFiles=2 fileSize=10240M

|socket type |checksum|clean system cache|吞吐量|IO平均速率|IO平均率标准差|执行时长|
|:-------------|:---:|:---:|:-----|:-----|:-----|:-----|
|Tcp socket| Y | N |610.7234448619312 |610.764892578125 |5.018315856084518|40.383|
|Tcp socket| Y | Y |567.7376431125773|567.8120727539062|6.500964152545038|42.3|
|Tcp socket| N | N |1521.0932857991681 |1522.7047119140625 |49.536454401904145|28.351|
|Tcp socket| N | Y |694.9440108585002 |694.9439697265625 |0.06277488816014715 |40.286 |
|Domain Socket| N | N  |3688.0965244012245 |3688.21142578125 |20.602881755691918 |28.364 |
|Domain Socket| N | Y | 2534.0262311309084| 2534.076416015625|11.323329607719455 |26.349 |
|Domain Socket| Y | Y | 585.1094223187247|585.2426147460938|8.827549782090745|42.377|
|Domain Socket| Y | N | 611.5988771426865| 611.7811279296875|10.562682380879258 |38.323 |

之所以在master服务器上部署datanode及tasktracker，主要是master服务器是ssd存储，磁盘的性能不会成为测试的瓶颈，进而将关注点集中在 本地读与网络读的性能上来。

结果对比：

+ 在打开checksum的情况下， 否开启本地读对性能的影响基本在测试误差范围之内。
+ 在关闭checksum的情况下， 开启本地读的性能会有很大的提升（清除系统缓存的情况下，`提升`了2.6倍，不清除系统缓存，提升了1.4倍）
+ tcp socket传输数据，checksum在关闭后，性能提升了1.5倍
+ local read:，checksum在关闭后，性能提升了5倍

[更详细的测试结果][testdomainsocket] 

当磁盘不是瓶颈时，checksum是瓶颈
当checksum关闭后，网络会是瓶颈。
local read 开启后，可以大量的节省网络的开销。

对于Hbase集群，开启本地读+skipchecksum后，性能会有很大的提升。
考虑到磁盘文件损坏的可能性，可考虑：

1. 对不同数据采取不同的读策略，对数据完整性要求高的开启checksum， 对数据完整性要求不高的关闭checksum
2. 调整datanode 校验block checksum的周期。


[unix domain socket]: http://zh.wikipedia.org/wiki/Unix_domain_socket
[testdomainsocket]:/assets/other/testdomainsocket.tar.gz

