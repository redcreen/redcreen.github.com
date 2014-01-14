---
layout: post
title: "java garbage collection fail"
description: ""
category: java
tags: [java,gc]
---
{% include JB/setup %}

在gc log中我们可能会遇到下面两种日志，并且这两种情况都会引发FullGC，进而造成造成应用暂停。

promotion failed
===

```
2013-10-10T06:11:47.654+0800: 1444544.329: [GC 1444544.329: [ParNew (promotion failed): 3774912K->3682402K(3774912K), 1.2662220 secs]1444545.596: [CMS: 128047712K->89062768K(153092096K), 811.7515150 secs] 131716601K->89062768K(156867008K), [CMS Perm : 22265K->22258K(37136K)], 813.0179170 secs] [Times: user=815.14 sys=2.21, real=812.89 secs]
```

> "ParNew (promotion failed)" means, there are some objects from young generation will be promoted to old generation, but there is not enough space. Maybe the old space is almost full, or maybe a promoted object is too huge, and there is not enough continue space. 

> from [java-gc-parnew-promotion-failed-concurrent-mode-failure]

"ParNew (promotion failed)" 表示当对象要从新生代晋升到老生代时，老生代没有足够的空间。具体分为两种情况：

+ 老生代满了，没有剩余空间
+ 要晋升的对象太大，而老生代没有足够的连续空间。

第一种情况比较容易理解，也容易判断，当出发fullgc后，如果老生代依然占用很高， dump出内存，然后用mat分析一下内存，排查是否有内存泄露或是确定就是老生代内存设置过小。


第二种情况要从cms gc的实现方式来说起了。详见[understanding-java-garbage-collection]和[What the Heck's a Concurrent Mode][what_the_heck_s_a]. 简单来讲，由于cms不停机，不能对内存压缩，cms会把死对象放在free-list中，然后在从free-list分配新对象。free-list中由于管理的是不连续的内存片段，因此成为**内存碎片**。如果free-list中都是小的内存片段， 这时如果要分配一个大的对象，就会造成分配失败， 进而引发FullGc, 停机回收并压缩内存。

concurrent mode failure
===

```
2013-10-09T17:03:50.334+0800: 1397267.009: [GC 1397267.010: [ParNew (promotion failed): 3647305K->3684179K(3774912K), 0.8413060 secs]1397267.851: [CMS2013-10-09T17:04: 
21.516+0800: 1397298.192: [CMS-concurrent-sweep: 30.519/31.363 secs] [Times: user=34.84 sys=0.18, real=31.35 secs] 
(concurrent mode failure): 135995125K->89169200K(153092096K), 840.6461240 secs] 139547450K->89169200K(156867008K), [CMS Perm : 22262K->22256K(37136K)], 841.4875980 se 
cs] [Times: user=841.80 sys=1.63, real=841.35 secs] 

```
concurrent mode failure产生的原因是在执行CMS GC过程中，同时有对象要放入老生代， 而此时老生代空间不足。（空间不足的情况同上）


可以看到`promotion failed` 和 `concurrent mode failure`的主要区别在于promotion fail发生的时间不同。


内存碎片
===

为了能够查看内存碎片的分配，taobao jdk 6u32中提供了观测CMS碎片信息的功能。

统计CMS gc时old gen中小于等于某些指定阈值的碎片的大小总和，使用`-XX:CMSFragSize`来指定这些阈值（按从小到大顺序用逗号隔开），如“`-XX:CMSFragSize=8,128,256,1k`”，会在gc log中输出该信息和当前used region的大小：

```
[CMS-concurrent-sweep: CMS fragmentation 8 number:0, size:0]
[CMS-concurrent-sweep: CMS fragmentation 128 number:1721, size:78461]
[CMS-concurrent-sweep: CMS fragmentation 256 number:1975, size:133759]
[CMS-concurrent-sweep: CMS fragmentation 1k number:6652, size:1359142]
[CMS-concurrent-sweep: CMS used region size:471365712]
```

内存碎片的产生是由CMS GC的实现原理决定的，解决的方法依情况不同而不定。 可以尝试：

+ 加大old区内存
+ 设置-XX:CMSInitiatingOccupancyFraction=NN 为更小的值。
+ 加大Young区，减小内存晋升的几率
+ 减少应用程序大内存分配的操作。
+ 优化应用程序，避免频繁分配大内存。

参考文章：

[jstat显示的full GC次数与CMS周期的关系][jstat_fullgc_cms]

[jstat_fullgc_cms]: http://rednaxelafx.iteye.com/blog/1108768 
[java-gc-parnew-promotion-failed-concurrent-mode-failure]: http://stackoverflow.com/questions/12323124/java-gc-parnew-promotion-failed-concurrent-mode-failure
[what_the_heck_s_a]: https://blogs.oracle.com/jonthecollector/entry/what_the_heck_s_a
[understanding-java-garbage-collection]: http://www.cubrid.org/blog/dev-platform/understanding-java-garbage-collection/