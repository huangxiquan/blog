---
title: Android WebView Cookie
date: 2018-09-05 20:43:05
tags: android webview
---
虽然现在的webview很少需要我们关心cookie的管理,但是了解一下还是有必要的.
<!--more-->
当开启webview的缓存,网页的cookie会被存在数据库,路径为data/data/包名/app_webview/Cookies.
{%asset_img android-webivew-cookie-path.jpg Cookie.db%}
将Cookies文件导出,用Navicat Premium(数据库工具)打开.

可以看到有三条记录.看到android API文档上的一句话.我做了这个实验.

这句话的大慨意思是如果host_key,name,path 三个字段值如果一样,则被认为是同一个记录,会覆盖之前的记录.所以我分别打开了如下三个网页,网页中都进行了cookie的设置.
* 192.168.84.140/a.html     cookie: "hello world"
* 192.168.84.140/b/b.html   cookie: "javascript"
* 192.168.84.140/c.html     cookie: "key=hello world"

以上是webview cookie的缓存规则,那么它又是如何取的呢?
当我第二次去打开这三个页面,我分别取了cookie,结果如下:
* a.html cookie: "hello world;key=hello world"
* b.html cookie: "javascript;hello world;key=hello world"
* c.html cookie: "hello world;key=hello world"

所以可以得出结论:webview会取该路径和它的上层路径上的所有cookie.例如host/b/b.html 会去取host/和host/b的cookie.
以上就是webview自动管理cookie的过程.如果你想要自己管理,webview提供了CookieManager供开发者使用.
