---
title: 读懂oauth协议
date: 2021-09-06 23:10:37
author: yuxuan
img: https://image.fyxemmmm.cn/blog/images/ins-122.jpg
top: false
hide: false
cover: false
coverImg: 
toc: true
mathjax: false
summary: oauth 单点登录授权详解
categories: 协议
tags:
  - oauth
  - 单点登录
---
今天，我想登陆豆瓣，看看电影评论，陶冶陶冶情操。

但是，我从来没注册过豆瓣账号，而我又懒得再注册一个，怎么办呢？

我打开豆瓣的官网，笑了，原来豆瓣早就为我这种懒人想到了办法。

## 懒人三步  

**第一步**：在豆瓣官网点击用 QQ 登陆。

![](https://image.fyxemmmm.cn/blog/images/oauth-0.jpg)

**第二步**：跳转到 qq 登录页面输入用户名密码，然后点授权并登录。

![](https://image.fyxemmmm.cn/blog/images/oauth-1.jpg)

**第三步**：跳回到豆瓣页面，成功登录。

![](https://image.fyxemmmm.cn/blog/images/oauth-17.png)

太方便了！

但这短短的几秒钟，可不简单，我来给你说说。

## 上帝视角看发生了什么   

这几秒钟之内发生的事情，在**外行的用户**视角看来，就是在豆瓣官网上输了个 qq 号和密码就登录成功了。

在一些**细心的用户**视角看来，页面经历了从豆瓣到 qq，再从 qq 到豆瓣的两次页面跳转。

但作为一群**专业的程序员**，我们还应该从上帝视角来看这个过程。

![](https://image.fyxemmmm.cn/blog/images/oauth-3.jpg)

**第一步：在豆瓣官网点击用 qq 登录**

当你点击用 qq 登录的小图标时，实际上是向豆瓣的服务器发起了一个请求。

http:// www.douban.com/leadToAuthorize

豆瓣服务器会响应一个重定向地址，指向 qq 授权登录的页面地址。

http:// www.qq.com/authorize

当然，这个重定向地址还附带了一个回调地址，这是在 QQ 那边登陆成功后需要跳回的豆瓣地址。

http://www.qq.com/authorize?
callback=www.douban.com/callback

这跳回的地址是必然的嘛，不然 QQ 怎么知道在我这边登陆成功后我要干嘛，上杆子找人家 QQ 授权的网站那么多。

这部分的流程是黄色的这部分。

![](https://image.fyxemmmm.cn/blog/images/oauth-4.jpg)

**第二步：跳转到 qq 登录页面输入用户名密码，然后点授权并登录**

上一步，浏览器接到重定向地址

http://www.qq.com/authorize?
callback=www.douban.com/callback

自然没什么好说的，乖乖访问过去。

这回访问的就是 QQ 的页面了。

![](https://image.fyxemmmm.cn/blog/images/oauth-5.jpg)

用户输入 QQ 号和密码，点击授权并登陆，这里走 QQ 服务器自己的校验逻辑，与豆瓣毫无关系。

若校验成功，会响应给浏览器一个重定向地址

www.douban.com/callback

没错，就是上一步传给 QQ 服务器的 callback 参数！

但除了这个地址外，还附上了一个 code，我们叫它**授权码**。

www.douban.com/callback?code=xxx

这个 code 是豆瓣服务唯一关心的事情，至于你那边如何校验用户，无所谓，只要最终能给我一个 code 码，我就认为这个用户在你那里登陆成功了。

这部分的流程是黄色的这部分。

![](https://image.fyxemmmm.cn/blog/images/oauth-6.jpg)

**第三步：跳回到豆瓣页面，成功登录**

这一步背后的过程其实是最繁琐的，但对于用户来说是完全感知不到的。

用户在 QQ 登录页面点击授权登陆后，就直接跳转到豆瓣首页了，但其实经历了很多隐藏的过程。

首先接上一步，QQ 服务器在判断登录成功后，使页面重定向到之前豆瓣发来的 callback 并附上 code 授权码。

www.douban.com/callback?code=xxx

浏览器接到重定向，乖乖发起请求，此时请求的是**豆瓣服务器**。

豆瓣服务器收到请求后，对 QQ 服务器发起了两次请求：

\1. 用拿到的 code 去换 token

\2. 再用拿到的 token 换取用户信息

这个 code 和 token 都是有失效时间的，也因此保证了只要不在短时间内泄漏出去，就不会有安全风险。

拿到用户信息之后，就返回给了浏览器。注意此时的浏览器上是豆瓣的首页，豆瓣也因此可以将你的个人信息展示出来。

![](https://image.fyxemmmm.cn/blog/images/oauth-17.png)

这部分的流程是黄色的这部分。

![](https://image.fyxemmmm.cn/blog/images/oauth-8.jpg)

至此，整个过程结束。

这个破玩意，就叫做 **OAuth 2.0 协议**。

这个流程目的是让大家从全局了解 oauth2.0 协议实际上发生了什么，并仅仅以 oauth 的其中一种模式，**授权码模式**进行讲解。

如想了解更多模式，以及每次的请求和响应的标准齐全的参数，推荐读一下阮一峰的文章。

http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html

##  **为啥要这么跳来跳去**    

为什么，要这么麻烦呢？跳来跳去的。

其实之所以有这个协议，我总结起来有两点原因：

**懒 + 不信任**

**懒**是指用户懒。

如果用户不那么懒，直接在豆瓣上新注册一个账号就好了。

**不信任**是什么意思呢？

如果用户信任豆瓣网站，那完全可以让用户在豆瓣网站输入 QQ 的用户名和密码，由豆瓣传给 QQ 服务器做校验，并返回用户信息。

![](https://image.fyxemmmm.cn/blog/images/oauth-9.jpg)

但这是不可能的，你愿意把你的 QQ 号和密码给豆瓣看到？

更甚者，如果 QQ 信任豆瓣，用户也信任豆瓣，那 QQ 也可以把自己的数据库直接拷贝给豆瓣，然后豆瓣就可以完全自己拥有一套 QQ 用户数据了，也就可以让用户使用 QQ 登录。

![](https://image.fyxemmmm.cn/blog/images/oauth-10.jpg)

当然，这也是不可能的。

所以就有了 OAuth 这种协议，你进行第三方授权时（文中的QQ），用户名和密码是不经过目标服务器的（文中的豆瓣），这保证了**授权的安全性**。

第三方授权服务器只给目标服务器返回有时效性的 code 和 token，目标服务器通过这个去第三方资源服务器，换取用户信息，这达成了**拿到用户信息**的目的。

所以总的来说，oauth 协议，就是由于三者（用户、目标、第三方）相互**不信任**，又想使用第三方服务器的**授权功能**，以及获取第三方服务器存储的**用户信息**，而产生的一个办法。

这个破玩意，就叫做 **OAuth 2.0 协议**。

哦，上面好像说过了。

##  ![](https://image.fyxemmmm.cn/blog/images/oauth-11.jpg)**coding**  

了解了上述过程后，代码自然就不难写了。

这里我实现了一个极简版的 oauth2.0 用于体验这个过程，大家可以参考下。

项目结构非常简单，只有两个模块，分别是豆瓣和QQ，分别启动即可。

![](https://image.fyxemmmm.cn/blog/images/oauth-12.jpg)

最终效果也非常简单清晰，下面请忍受 low 逼的显示效果

**第一步**，登陆豆瓣页面。

![](https://image.fyxemmmm.cn/blog/images/oauth-13.jpg)

**第二步**，使用 QQ 页面进行授权。

![](https://image.fyxemmmm.cn/blog/images/oauth-14.jpg)

**第三步**，授权成功跳回豆瓣首页。

![](https://image.fyxemmmm.cn/blog/images/oauth-15.jpg)