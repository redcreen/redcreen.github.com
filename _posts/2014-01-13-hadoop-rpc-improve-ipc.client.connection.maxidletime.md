---
layout: post
title: "Hadoop rpc优化 参数优化"
description: ""
category: hadoop 
tags: [hadoop, rpc]
---
{% include JB/setup %}



上线后新问题
===

hdfs启动后一切正常，当运维将参数`?`改回后，突然出现了102个deadnode


临时方案
---
重启有问题的datanode，其实也可以通过refresh datanodes恢复

问题分析
===

解决方案
===
