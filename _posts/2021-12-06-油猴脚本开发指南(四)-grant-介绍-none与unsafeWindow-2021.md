---
layout:     post
title:      "油猴脚本开发指南(四)-grant-介绍-none与unsafeWindow, 2021"
subtitle:   "欢迎使用"
date:       2021-12-06 12:31:20
author:     "Ruer"
header-img: "img/bg/hello_world.jpg"
catalog: true
tags:
    - TamperMonkey
---

## 主要内容

介绍一下油猴脚本的 grant 属性,说明 none 和 unsafeWindow.

## grant

这个属性可用来申请 GM_* 函数和 unsafeWindow 权限.相当于放在脚本 header 里面告诉油猴扩展,你需要用些什么东西,然后它就会给你相应的权限.

更加详细的列表:
tampermonkey [文档地址](https://www.tampermonkey.net/documentation.php#_grant)
tampermonkey 可申请 api [文档地址](https://www.tampermonkey.net/documentation.php#api)

## none 和 unsafeWindow

简单来说 none 就是直接运行在前端页面中,否则就是运行在一个沙盒环境,需要使用 unsafeWindow 去操作前端的元素.

除了 GM_* 函数外,还有两个特殊的权限,就是 none 和 unsafeWindow.默认的情况下,你的脚本运行在油猴给你创建的一个沙盒环境下,这个沙河环境无法访问到前端的页面,也就无法操作前端的一些元素等.如果在页面最前方声明:"//@grant none",那么油猴就会将你的脚本直接放在前端的上下文中执行,这是的脚本上下文(window)就是前端的上下文.但是这样的话就无法使用 GM_* 等函数,无法与油猴交互,使用一些更强的功能.

所以一般写脚本的时候是使用 unsafeWindow 与前端交互,而不使用"//@grant none",这样就可以使用 grant 去申请油猴的一些更强的函数功能.这时候的脚本上下文(window)是沙盒的上下文,而不是前端的上下文.

在沙盒环境中,有一些 window 的操作也无法处理,需要使用 grant 来获取,例如:"// @grant window.onurlchange"(TamperMonkey文档中的)

```JavaScript
// ==UserScript==
...
// @grant window.onurlchange
// ==/UserScript==

if (window.onurlchange === null) {
    // feature is supported
    window.addEventListener('urlchange', (info) => ...);
}
```

这样的作法是为了避免恶意网页可以直接的使用 GM_* 函数,也可以避免被网页检测到 GM_* 插件的存在.
