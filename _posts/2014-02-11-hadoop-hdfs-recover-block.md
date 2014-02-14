---
layout: post
title: "hadoop hdfs recover block"
description: ""
category: hadoop
tags: [hadoop,recovery]
---
{% include JB/setup %}

部分内容转自: www.cnblogs.com/ZisZ/p/3253570.html

#### 客户端发起的recover  block ####

HDFS客户端在写流水线（pipeline）的时候，如果遇到异常，会对写文件的最后一个block进行差错恢复（error recovery）；这个过程被称为recover block，或block recovery；

![client recover][block-recovery-client]

注：

+ targets：D1、D3、D4、D5；
+ syncList：D1、D3、D4；
+ successList：D3、D4；

对该过程的简单描述如下，配合上图：

1. 客户端C在写 D1->D2->D3->D4->D5流水线时，D2异常，C发起对文件最后一个block的recover block过程，根据一定的规则，选定D3为首DN（primaryDatanode），对targets中所有DN进行recover block；
1. D3向其它的DataNode收集该block的信息，这个过程中可能再遇到DataNode异常或不适合recover block，如D5，将其排除后，组织成syncList队列，并选定其中最短的长度作为；
1. D3向NameNode（N）发送消息请求更高的Block版本号；
1. D3向syncList中所有DN发送消息，要求其对该block进行更新（包括内存对象及磁盘文件的版本号、长度，可能引发磁盘文件被切短）；在这个过程中，可能再遇到DataNode异常，如D1，将其排除后，组织成successList队列；
1. D3向N发送消息，提交recover block的结果：删除旧的block对象，增加新的block对象；
1. 将successList返回给客户端，客户端根据该列表，重新构建流水线，继续写文件；



#### Namenode发起的recover  block ####

除客户端外，在某些特殊情况下，会由NameNode发起recover block过程；其通常是由recover lease过程引起的（详见Lease（租约）的“租约释放”部分）；

与客户端发起的recover block过程相比，不同之处在于：

+ recover block过程的启动过程不是由客户端请求，而是通过NameNode在响应DataNode的心跳时返回命令；
+ 客户端recover block通常不会关闭文件，以期完成恢复后继续写文件；而NameNode完成恢复后会关闭文件，以便客户端可以重新打开；

结合这两点，整个recover block过程的逻辑流程如下：
![namenode recover][block-recovery-server]


[block-recovery-client]: /assets/images/hadoop/block-recovery-client.png
[block-recovery-server]: /assets/images/hadoop/block-recovery-server.png
