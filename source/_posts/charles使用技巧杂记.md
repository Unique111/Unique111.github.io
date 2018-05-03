---
title: charles技巧使用杂记
date: 2018-04-21
categories: ['工具', 'charles']
tags:
---

最近在做一个需求，到了联调阶段，后端同学希望可以自己访问页面，测试测试他的接口；然而，发现通过ip直接访问我本地时，报了非法来源的错误。（以前是可以直接通过ip访问我本地的）

通过debug发现，是node server中使用到sso中间件报的错，大概是sso登录对域名的校验变严格了吧。

通过ip直接访问不行，貌似只能部署到test环境了，但是考虑到有点麻烦，毕竟，需求还在联调阶段，如果每次修改都去部署一次环境，还是很累的=='

于是，想到了一个方法，既然sso只是对域名有限制，那我可以伪造域名啊！！！机智如我哈哈哈哈~

<!-- more -->

只要在浏览器中，设置一个代理（代理到charles），然后在charles设置一个合法域名到我本地ip的映射。这样，表面上访问的是合法的域名，实际上却是访问我本地啦~

OK，问题解决。

具体步骤如下：

1、在charles设置端口；
Proxy => Proxy Settings => 设置Port的值
记得勾选“Enable transparent HTTP proxying”选项。

2、在chrome浏览器中安装switchysharp插件，用来设置代理；
![switchysharp](http://p1.meituan.net/scarlett/33f6ccc4ab62f7bac47cb7bd5762caae262125.png)

3、在charles中设置url的映射map；
Tools => Map Remote => 添加如下映射：
http://bd.maoyan.com => http://172.19.148.117:端口号

4、到此，就OK了。此时访问http://bd.maoyan.com，实际上就是访问我的本地http://172.19.148.117:端口号了。

可能遇到的问题：

1、在charles中抓到的包显示unknow，提示说证书有问题；
解决：重新安装下根证书。

