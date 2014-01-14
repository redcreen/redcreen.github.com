---
layout: post
title: "alibaba service framework practice"
description: "服务框架实践与探索"
category: dubbo 
tags: [dubbo, rpc, soa]
---
{% include JB/setup %}

2011年杭州Qcon dubbo分享资料

<iframe src="http://www.slideshare.net/slideshow/embed_code/11962358" width="800" height="600" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC;border-width:1px 1px 0;margin-bottom:5px" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="https://www.slideshare.net/ShawnQian/alibaba-service-framework-practice" title="Alibaba Service Framework Practice" target="_blank">Alibaba Service Framework Practice</a> </strong> from <strong><a href="http://www.slideshare.net/ShawnQian" target="_blank">Shawn Qian</a></strong> </div>

[PPT下载][ptt_download]


1. 服务框架实践与探索阿里巴巴(B2B) 技术部钱霄（@shawnqianx）2011/10/23
2. Overview• 承载每天10亿次的调用• 管理超过1000个的服务• 部署在阿里巴巴整个站点
3. 提纲• 应用开发的挑战• 服务框架的演进• 一些总结分享• 问答交流
4. 应用开发技术的变迁• Alibaba B2B的Web应用 – 1999/2000年 使用Perl CGI开发 – 2001年开始 改用Java技术 • 2001年Servlet/JSP开发 • 2002年Java EE技术 • 2003年 基于Turbine MVC框架开发 • 2004年 使用轻量级容器 • 2005年 自制的WebX框架成为应用开发的首选
5. 应用结构的变化• 发展初期：规模小，JEE技术管用• 高速成长：膨胀，巨无霸应用开始出现• 寻求变革：拆 分应用，独立服务• 持续优化：聚 合服务，管控治理
6. 挑战• 业务不断发展，应用规模日趋庞大 – 巨型应用的开发维护成本高，部署效率降低 – 应用数量膨胀，数据库连接数变高• 访问量逐年攀升，服务器数不断增加 – 数据连接增加，数据库压力增大 – 网络流量增加，负载均衡设备压力增大• 对性能，可靠性的要求越来越高
7. 对策• 拆分 – 对巨型系统进行梳理，垂直拆分成多个独立的Web 系统。• 剥离 – 抽取共用的服务，提供远程调用接口，与应用共生• 独立 – 甄别核心的服务，独立搭建集群，提供专门服务。• 均衡 – 减少专业负载均衡设备使用，应用自行支持分布式 调用/调度。
8. 通讯• 进程内  进程间• 节点内  节点间• RPC 是一切的基础
9. 远程调用的变化• EJB@Alibaba B2B的年代 – 享受容器级的db事务连接池等服务，及透明的 分布式调用• RPC@Alibaba B2B – RMI/Hessian – XML-RPC/WS• 定制的框架 – Dubbo
10. 重新造轮子？• 需要吗？ – 不仅仅是RPC • LB/FailOver/Routing/QoS等功能，是达成治理所必 需的，但一般的开源方案少有提供。 – 稳定性/兼容性考虑 • 开源方案并不完美，用好她们，付出的代价也不 低。 – 集团作战需要规范 • 大量的应用并存，大规模的开发团队，需要统一的 规范指引。
11. 初始目标• 零入侵• 高性能• 高可靠/适应高并发的环境• 模块化设计• 从底层支持服务化
12. 实践• 最初的尝试 - Dubbo 0.9
13. 重点1 – 核心功能抽象 Spring IntegrationService Remote Service Stub RR Strategy RND Strategy Registry Pub/Sub Depends LoadBalance / FailOver RPC Abstraction DBO RPC RPC Java RMI Hessian v2 TBRemoting Communication 示意图
14. 重点2 -软负载initasyncsync Registry 2.subscribe 1.register 3.notify 4.invoke Consumer Provider 5.count Monitor 示意图
15. 重点3 – OSGi化？
16. 取舍• Dubbo 0.9 的选择 – Spring Bean XML配置方式来暴露/注册服务 – 用内部的TB-Remoting作为通讯框架 – 用Hessian v2作为首选的序列化方式 – 用Spring-DM/OSGi作为模块化的基础 – 简易的服务注册中心，支持订阅推送• 放弃 – 多协议/多通讯框架/异步调用…
17. 三个月后…
18. 逐渐成长 – 1.0• Dubbo1.0 版本 – 放弃对Spring-DM/OSGi的支持 – 增加独立的服务管理中心，提供初步的服务治 理能力。 – 调用数据的监控与展示
19. 关于OSGiSpring-DM Server的一些不适应： 1.遗留应用的迁移，需要付出很高代价 2.为了处理OSGi/non-OSGi不同的情况，框架代 码变复杂。 3.针对ClassLoading的问题的特殊处理，非常不 优雅。 4.使用DMServer后，被框架Bundle与业务 Bundle的互相依赖及启动顺序问题所困扰，未 能妥善解决。 5.构建及调试不够完善。
20. 幸亏…• Spring-DM隔离了OSGi的API• 设计初期有预留伏笔，框架模块没有完全 切换到OSGi Bundle风格• 3天时间脱离DM Server，使用Spring完成 Bootstrap.
21. 野蛮生长…• Dubbo 1.0迅速推广，覆盖了大部分的关键 应用• 成为B2B内部服务调用的首选实现• 遭遇第一次大规模故障• 支持热线打爆
22. 新的要求…• 被要求支持更多的使用场景 – 支持专用服务协议的调用(memcached etc…) – 多种模式的异步调用• 要求完善的监控 – 服务状态/性能/调用规模等多方面多维度的统 计及分析…• 治理功能 – 服务分组 – 流量分离 –…
23. 新的挑战…• 如何抵抗功能膨胀？ – “通用”==“难用”，如何取舍？ – 精力有限，团队忙不过来了！• 如何抵抗架构的衰退？ – 新需求的加入… – 新人的加入…
24. 新的旅程 – 2.x• Dubbo 2 – 重构 – 对RPC框架的重新审视 – 对模块化机制的重构 • 以JDK SPI机制替代原有的Spring Bean组装 – 扩展扩展扩展 • 支持更多的通讯框架(Mina/Netty/Grizzly…) • 支持更多的序列化方式(Hessian/JSON/PB…) • 支持更多的远程调用协议(DBO/RMI/WS) • 完整的异步调用支持 – 服务注册中心持续增强 • 分组/路由/QoS/监控
25. 重点1 – 协议优化 Stub Codec Serialization Implement Client Header Body Server Thread Pool Transporter• Transporter – mina, netty, grizzy…• Serialization – dubbo, hessian2, java, json…• ThreadPool – fixed, cached
26. 重点2 – 负载均衡增强FailoverFailfast Cluster Directory RegistryFailsafe StaticFailback merge list List<Invoker>Forking route Script Invoker Router invoke Condition select List<Invoker> Random LoadBalance RoundRobin• 在客户端处理 invoke Invoker LeastActive – 容错 Invoker – 路由 – 软负载均衡
27. 重点3 - Dogfooding• 注册中心和监控中心也是普通RPC服务 – 为此，需支持回调操作： • 生成回调参数的反向代理： • subscribe(URL url, NotifyListener listener) • listener.notify(list) refer RegistryServiceProxy subscribe RegistryService refer UserServiceProxy UserService invoke
28. 重点4 – 更多调用方式• 完整的异步调用支持 – 基于内部的消息中间件，实现可靠的异步调用• 并行调用(Fork/Join) – 利用API，应用可以同时发起多个远程请求 – 虽然比较简单，但的确管用！
29. 重点5 – 插件机制调整• 简化插件机制 – 基于JDK的SPI机制扩展 – 不再依赖Spring• 区分API/SPI – API给使用者。 – SPI给扩展者
30. 其他增强• 远程调用的本地短路 – 允许：缓存或远端故障时，本地短路• 调用的Cookie传递 – 某些隐式传参的场合（鉴权等）• 诊断功能 – 自带远程调试诊断功能 (Diagnosing via Telnet)
31. Business Dubbo Framework Consumer Provider Start Interface Class Inherit Init Call Depend Service User API Dubbo 2 于是… Interface Implement Config refer get invoke export export ReferenceConfig ServiceConfig invoke Proxy getProxy getInvoker Proxy ProxyFactory Invoker Registry notify getRegistry NotifyListener Registry RegistryFactory invoke notify subscribe register getRegistry new RegistryDirectory RegistryProtocol Cluster list getRouter merge Filter Directory Cluster list merge invoke route invokeRPC getRouter Invoker Router RouterFactory select LoadBalance Contributor SPI Monitor invoke getMonitor count getMonitor MonitorFilter Monitor MonitorFactory Protocol invoke refer export invoke Filter Invoker Protocol Exporter Filter invoke export invoke refer DubboInvoker DubboProtocol DubboExporter invoke DubboHandler Exchange request connect bind reply connect bind ExchangeClient Exchanger ExchangeSerever ExchangeHandler Transport send connect bind received received connect bindRemoting Client Transporter Server ChannelHandler decode wrap wrap encode Codec ChannelHandlerWrapper write read Serialize serialize deserialize getExecutor ObjectOutput Serialization ObjectInput ThreadPool
32. Dubbo 2 一些数据• 部署范围： – 运行在200+个产品中 – 为1000+个服务提供支持 – 涉及数千台服务器• 繁忙程度： – 最繁忙单个应用： 4亿次/天 – 累计：10亿次/天
33. Dubbo2 性能数据CPU : E5520 @ 2.27GHz *2内存: 24G网卡: 1GOS: RedHat EL 6.1Linux Kernel: 2.6.32-131.0.15.el6.x86_64
34. 某服务调用情况Credit to TB LogStat 
35. 后续方向 – 我们在路上• 服务治理 – 服务的版本管理 – 优雅升降级• 资源管理 – 服务容器及服务自动部署 – 统一管理集群资源• 开发阶段增强 – IDE支持
36. 一些总结• 框架的入侵性 – 支持Spring的Bean配置 - 包括业务Service Bean的暴露及框架的运行时参数配置 – 也支持API编程方式的暴露服务，及API配置框 架的运行时参数。 – 远程服务的特性 决定了不可能完全无入侵
37. 一些总结 – 续• 框架的可配置性 – 约定优于配置 • Convention over Configuration – 配置方式对等 • XML == Java
38. 一些总结 – 续• 框架的扩展性 – 微内核设计风格，框架由一个内核及一系列核 心插件完成。 – 平等对待第三方的SPI扩展。 • 第三方扩展可以替代核心组件,实现关键的功能 – 区分API与SPI • API面向使用者，SPI面向扩展者
39. 一些总结 – 续• 模块间的解耦 – 事先拦截 • 在关键环节点允许配置类似ServletFilter的强类型的 拦截器。 – 事后通知 • 允许注册消息监听器，框架在执行关键操作后，回 调用户代码。Stub Filter Invoker Protocol Filter Invoker Impl Listener Listener
40. 一些总结 – 三方库• 三方库的采纳 – 严格控制三方库依赖的规模 • 传递依赖会对应用程序的依赖管理造成很大的负担 • 核心代码尽可能少的依赖三方库。 • 必须考虑三方库不同版本的冲突问题。 – 隔离三方库的不稳定/不兼容 • 内联
41. 一些总结 – 性能优化• 性能及优化 – 环境优化 • 升级Linux内核开启 ReceivePacketSteering, 测试 表明包处理吞吐量提升明显 • JVM GC tuning
42. IRQ Balancing 42
43. 一些总结 – 优化 续• 性能及优化 – 代码优化 • 锁粒度细化 • 善用无锁数据结构(Lock-free data structure) • 使用对SMP优化的同步机制(Java Concurrent Lib)
44. 一些总结 – 优化 续• 考量性价比，避免过度优化 – 莫钻牛角尖，充分够用即可。 • 拿90分容易，拿99分难 • 比应用足够快，就够了。 – 优化，通常意味着牺牲未来的可能性。
45. 一些总结 – NIO框架• 支持多个NIO框架是挑战 – Mina/Netty的差异只会在细节中体现 • 内存使用表现上的差异 • 适配Codec和Serialization的行为差异 • 线程处理上的差异：Netty一次请求派发两个事件， 导致需两倍线程处理
46. Minas vs. Netty MemoryMina Netty 48
47. 一些总结 – 线程模型• 线程模型选择权留给应用 – IO线程池与业务线程池的隔离 – 固定线程数 vs 可变线程数
48. 更多分享 尽在技术博客…• 实现的健壮性• 防痴呆设计• 泛化式扩展与组合式扩展• 常见但易忽略的编码细节• 负载均衡扩展接口重构
49. 真正的分享 - 开源！• Dubbo2框架核心以 Apache v2协议开源 – 久经考验的代码 – 符合开源社区口味的开 发流程 – 完善的单元测试 – 必要的文档 开源模块一览
50. 资源• 访问 http://code.alibabatech.com/ – 了解更多关于 Alibaba B2B 开源项目的信息• Blog http://code.alibabatech.com/blog/• Follow Us @dubbo
51. Credit & Thanks to• The Dubbo Team• PupaQian• BlueDavy & the HSF Team• ……
52. Q&A
53. End


[ptt_download]: /assets/doc/Shawn-BuildServiceFramework-rev02.pdf