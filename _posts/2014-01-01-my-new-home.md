---
layout: post
title: "我的blog新家"
description: ""
category: other
tags: [杂谈]
---
{% include JB/setup %}

以前一直在博客园上维护自己的blog，用live writer写日志同步， 虽说用着还不错，但埋在内心深处的得瑟一直怂恿着我要自己搭个博客。

domain
---
之前的.com域名到期了，在maddogdomain上购买了新的.me域名，7.9美刀一年。在dnspod上注册了个帐号，然后将域名的dns指向了dnspod. dnspod做的挺不错，界面简单、操作清晰，还免费提供了不少好功能。之所以要将dns server指向dnspod，主要还是由于maddogmomain上不支持redcreen.me的cname. 顺便说一句的是，dnspod还可以绑定微信，可以在微信上的服务帐号上操作，微信真心强大啊。

hosting
--
找个靠谱的hosting还真是费脑筋，最开始好友@oldratlee推荐了diandian, diandian做的还不错，支持markdown，但由于绑定域名还要发布文章超过一定的数量，真是够汗的。

后来在openshift上弄了个免费的空间，安装了wordpress, 功能上都挺不错的，只是访问速度慢的难以接受。

最终还是把家安在了github page上了，通过jeykll 可以满足我建个blog的所有功能上的需求，同时github Page访问快、无存储及流量上的限制，相当不错啊。 本地编写post， 用git管理的方式 也足以体现“不倒腾不是好程序员”的精神。
