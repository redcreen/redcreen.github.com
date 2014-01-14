---
layout: post
title: "Application engine security architecture"
description: ""
category: paas 
tags: [security, PAAS]
---
{% include JB/setup %}

安全容器

![container][container] 

安全体系

![security][security] 

网络层
---
网络层的目的是：给应用进程最小的网络权限。

1. 网络设置只针对ownerid范围是100000-3000000000的进程（这个ownerid = appversion+100000， 加100000是t4在生成instance时ip的规则）；
这样即保证了isv的进程网络有严格设置，又可以不用担心管理进程受影响。
2. 网络设置采用白名单方式，只对OUTPUT限制，目前的规则是，output只允许rds指定ip+port，cache指定ip+port，top api指定port
3. OUTPUT对于ownerid在100000-3000000000的进程的默认规则必须是reject，drop会导致jetty启动变慢，主要是因为jetty在启动时要尝试一个网络连接（具体是哪个还需排查，之前调试时遇到过，没什么实际用途，但增加了网络设置后如果是drop那会等待到那个网络连接超时.）
4. feturl功能（暂时未提供）由于需要访问网络，需要在iptables上增加相应的功能，但考虑到iptables规则变更会较多，并且规则数量会变得很多，建议将这个功能在服务中心上实现，isv通过调用服务的方式使用这个功能，iptables上只增加服务中心地址即可。
5. iptablesctl中实现了check功能（iptables check)，后续需要在监控系统中配置上去。
6. iptables目前使用的版本是v1.4.18，服务器默认安装版本v1.3.5，不支持conntrack，需要升级（我是用的手动编译的方式升级的，后续需要考虑自动化，或让pe升级默认版本）
7. 网络层的限制是针对app的，即使是同一个isv开发的不同应用之间也不能相互访问

系统层
---

系统层的目标是：将每个应用进程限定在独立的安全容器上，做到应用间及应用与系统间强制隔离。
系统层的主要工作分为2层, 由下到上：

+ kernal patch （王志通提供，主要是进程权限的提升，比如通过setuid等,具体可咨询高阳或王志通）
+ T4 instance (细节实现可参考文档：http://baike.corp.taobao.com/index.php/Kernel/documents/tae 或咨询高阳)
+ selinux： 通过设置selinux策略，阻止jni，未授权网络连接、未授权文件访问，阻止程序listen端口等
+ capablity instance ：通过去除所有的capaalibility，禁止了所有的特权操作.
+ 资源限制 （cpu mem 文件句柄数）,通过限制instance的资源，进而限制应用进程可使用的资源
+ 普通用户权限：instance中的root用户也降为普通用户，即使通过系统漏洞拿到了instance的root权限，也毫无用途
+ 单instance单进程：每个instance中只启动一个java进程，从java进程的角度来看，这个“系统”中只有这一个进程，因此很自然的起到了应用隔离的作用。

注：

1. 系统层的安全极为重要，kernal patch是最后一道防线，需要增强测试及监控，部署系统需要支持物理机的下线，在全面放开java应用前，需要确保系统层的安全设置及监控达到安全要求。
2. selinux中也做了端口上的限制，因此在iptables上增加规则的同时，也需要修改selinux规则，并重新编译。（此处可考虑让高阳增加一个shell的接口，简化操作）
3. T4 instance创建需要将近20s，其中10s用于selinux新规则的编译，10s用于增加对应的端口规则，为了提升创建和销毁的性能，增加了预创建机制：安装完T4后，先预创建128个instance，instance端口范围为[16384+128), 在这个端口范围内，instance创建时会直接从预创建的instance中根据端口取出一个使用，如果指定的端口不在欲创建的端口范围内，则会编译并创建一个instanc[只是这个过程会比较长。
4. 为了避免临时端口与jetty监听的端口[16384，+128）冲突，设置ip_local_port_range为32768~61000（iptablesctl.sh start时会自动设置掉），如果后面需要暴漏dubbo服务，可在 16512~32767 端口范围内选取一个合适的区间.


JVM层
---

JVM层的目标是提高突破JVM沙箱的门槛.
主要工作分为3层：

1. java policy file ：白名单授权，由于ssh等开源框架需要createClassLoader、反射等权限，因此也可很容易的通过createClassLoader定义高权限的类来突破java沙箱， java policy file在安全上的作用已经很小，主要是避免开发者不小心调用了危险指令。
2.运行时字节码修改：通过java instrument，在加载类时通过asm对字节码进行修改，对方法体的头尾进行字节码增强，增加安全功能。
3. jdk源码修改：修改ClassLoader Policy ProtectionDomain ProcessImpl等rt.jar中的类就行修改，去除Runtime.exec System.loadLibrary 等功能.
需要注意的是classLoader中判断loadlibrary时用到了类名判断（if fromclass startWith (sun.) || startWtich(java.) ，为了防止用户定义java或sun开头的类绕过此防护，在jetty-plus.xml中设置了sysclass，这样sun、java开头的类就必须由我们定义的classLoader加载.

注：

1. jetty policy提供了动态load policy的功能，但当前版本上存在不少的解析bug，并且动态policy对生产环境没什么帮助，因此采用java提供的默认policy解析器
2. 第2层与第3层从本质上来讲是一致的，都是通过修改java 安全沙箱的默认行为，增强安全校验的目的。两层都加旨在提高突破门槛。
3. jvm层的安全防护措施需要对java security有足够的了解。
4. 后续可考虑给不同的业务定制不同的开发框架，减少开源框架的使用，将权限收得更紧. 

PHP层
---

1. 通过配置com.caucho.quercus.QuercusModule，以白名单的方式设置PHP API白名单
2. 可以扩展statment，限制应用循环次数
3. 建议将quercus升级到4.0.35, 功能更加完善，避免在使用时遇坑

应用层
---

1. 防sql注入 - 在druid数据源上配置了 wall filter
a)druid数据源对jndi数据源的配置支持有问题：1) 内部参数传递有问题，所以通过jndi配置后，datasource上配置的参数部分失效 2)每次getDatasource都会new datasource，不符合预期，修改后的做法是：写了一个apapter类，避开此问题.
b)每个app有自己的帐号，同一个app的不同版本同享同一个帐号。
2. css 、html 过滤 - TBMLFilter
3. xss - CajaFilter + TBmlFilter
4. csrf - CsrfTokenValidateFilter + TBmlFilter
5. HTTP Header Injection 
a) Location Header白名单检查 
b) Header 在输出时对\r \n 字符做检查并过滤, 避免header注入
c) response.sendError(code, "content"), 对errorcontent进行过滤，防止伪造钓鱼页面（error处理不经过filter，因此需要单独处理）
6. cookie
a) 容器默认禁止cookie （Header SetCookie没有检查，需要处理）
b) 配置cookie开启后，用户写的cookie名称全部增加一个前缀：tae_${appname}_ , 需要注意限制cookie的个数及长度，避免cookie攻击
7. session
a)session采用分布式session，具体实现为调用open api中的cache接口存储在tair中，失效时间是30m
b)由于session缓存的存储不是持久式的，为了避免cache被换出（LRU), session的存储与cache的存储分为两个空间，对应于tair的两个用户. session本身的量比较小，不用担心session本身的换出问题。
c) 考虑到分布式实现，禁用方法：getSessionContext，getAttributeNames，getValueNames，invalidate
d) 另需注意，由于实现是分布式session，放入session中的对象必须实现序列化接口java.io.Serializable
8. cache
tae 对 tair的访问都是经过统一的网关（由tair提供，架构与rds类似），该网关做帐号验证. 每个app 都有自己的帐号。同一个app的不同版本共享同一个帐号。
9. form表单提交
a) method自动转换为post，并在action上增加token
b) 提交时比对session中token与参数中的token，避免重复提交及csrf攻击
10. 文件上传
目前禁止文件上传，限制方式：
通过tbml禁用form表单上的encrept（即使用户突破限制也没有关系，由于没有接入分布式文件系统，上传的文件只能存放在一台服务器，没有实际意义.)
11. 调试版本登录功能
调试版本登录功能主要目的是防止调试中的内容被他人看到，这个安全级别要求不高，实现方式为验证cookie中的nickname是应用配置文件（部署系统自动生成）中的author是否相同
即使有人恶意修改nickname，访问到了调试版本的内容，没有什么危害性。
12. 后台管理登录功能
部分应用存在后天管理功能，isv通过后台管理功能对数据进行维护. 后台管理功能等登录验证的安全级别要求很高，一旦恶意用户拿到了他人app的管理权限，app的内容有被篡改的风险。
实现方案步骤：
1)定义一个规范，所有后台管理功能都放到admin目录
2)在nginx上做功能扩展，对所有admin目录的访问，验证用户身份
a) 先验证用户登录状态是否正确 （访问tbsession提供的http接口，获取用户session信息，并与cookie中的信息做比对）
b) 验证登录的用户是否为app的author （访问容器定制的http接口，获取author，并对session中的nickname比对） 
注：【优化项】author信息可考虑直接存到路由信息中，也可考虑放到session中
13. header中敏感信息过滤功能
考虑到java沙箱被攻破，java应用可获取到taobao域名下用户的cookie信息，在nginx上扩展增加header过滤功能，将请求中的敏感信息过滤掉，也可对response中header信息做检查

其他
---

ip白名单的可靠性：为了获取访客的ip地址，一般需要

```java
        String ip = request.getHeader("X-Forwarded-For");
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("WL-Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("HTTP_CLIENT_IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("HTTP_X_FORWARDED_FOR");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
```

这里需要注意，如果用户用curl http://addr -H "WL-Proxy-Client-IP:X-Forwarded-For:1.2.3.4",这时拿到的ip地址会是1.2.3.4, realip, 如果白名单比较是按照前缀匹配，比如1.2.开头，就会被跳过。

改进方法：

```java
        String ip = request.getHeader("WL-Proxy-Client-IP");
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("X-Forwarded-For");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("HTTP_CLIENT_IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("HTTP_X_FORWARDED_FOR");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
```

WL-Proxy-Client-IP放在第一位是因为nginix上设置了

`proxy_set_header        WL-Proxy-Client-IP $remote_addr;`

[container]: /assets/images/tae/container.jpg
[security]: /assets/images/tae/security.jpg