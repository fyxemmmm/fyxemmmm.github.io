---
title: http2详解 [下]
date: 2021-08-26 21:11:57
author: yuxuan
img: https://image.fyxemmmm.cn/blog/fj/images/fj-16.jpg
top: false
hide: false
cover: false
coverImg: 
toc: false
mathjax: false
summary: http2协议详解 下篇
categories: 协议
tags:
  - http2
---

# 8. http2的世界

那么当http2被广泛采用的时候，世界将会成什么样呢？或者说，它会被真正的采用吗？

## 8.1. http2会如何影响普通人？

到目前为止，http2还没被大范围部署使用，我们也无法确定到底会发生什么变化，但至少可以参考SPDY的例子和曾经做过的实验来进行大概的估计。

http2减少了网络往返传输的数量，并且用多路复用和快速丢弃不需要的流的办法来完全避免了head of line blocking(线头阻塞)的困扰。

它也支持大量并行流，所以即使网站的数据分发在各处也不是问题。

合理利用流的优先级，可以让客户端尽可能优先收到更重要的数据。

所有这些加起来，我认为页面载入时间和站点的响应速度都会更快。简而言之，它们都代表着更好的web体验。

但到底能变得多快，到底提升有多大呢？我认为目前很难说清楚。毕竟这些技术依然在早期阶段，我们还无法看见客户端和服务器实现这些并真正受益于它所提供的强大功能。

## 8.2. http2会如何影响web开发？

近年来，web开发者、web开发环境为HTTP 1.1存在的一些问题提供了一部分临时的解决方案。其中的一部分我已在上文中简单的介绍了，不妨简单的回忆一下。

很多工具和开发者可能会默认使用这些方案，但它们其中的一部分也许会损害到http2的性能，或者至少让我们无法真正利用到http2新提供的强大威力。Spriting和内联应该是http2里面最不需要的了。因为http2更倾向于使用更少的连接，所以Sharding甚至会伤害到http2的性能。

这里的问题在于：对于网站的开发者而言，在短期内开发和部署同一套前端来支持HTTP 1.1和http2的客户端访问并获得最大性能将会是一个挑战。

考虑到这些问题，我认为彻底发掘http2的潜力还有很长一段路要走。

## 8.3. http2的各种实现

在这样一篇文章中详细说明每个实现细节注定乏味且毫无意义，我将用更通用的术语来解释实际的场景，并在此给大家提供一个http2的[实现列表](https://github.com/http2/http2-spec/wiki/Implementations)作为参考。

在http2的早期就已经有大量的实现。并且在http2标准化工作期间，这个数量还持续增长。截至我写这篇文档的时候，共有40种实现已记录在案，他们中的大多数都实现了最新的草案。

### 8.3.1. 浏览器

Firefox一直紧跟最新的协议，Twitter也紧追不舍提供了基于http2的服务。2014年4月期间<!--『从2014年4月开始』会不会更符合原意？-->，Google在少数测试服务器上提供http2支持。从同年5月开始，开发版的Chrome支持http2。Microsoft也在他们的产品预发布会上展示了支持http2的下一代浏览器。Safari (iOS 9 以及 Mac OS X El Capitan) 和 Opera也都表态它们将会支持http2。

### 8.3.2 服务器

事实上，已经有不少的服务器实现了http2。

时下最流行的Nginx自1.9.5(发布于2015年9月22号)版本后提供了对http2的支持并且取缔了原来的SPDY模块(因此SPDY和http2无法同时运行在同一个Nginx服务器实例中)。

而Apache HTTPD服务器也实现了一个名为[mod_http2](https://httpd.apache.org/docs/2.4/mod/mod_http2.html)的http2模块，并与2015年10月9号在2.4.17的版本中发布。

此外，[H2O](https://h2o.examp1e.net/), [Apache Traffic Server](https://trafficserver.apache.org/), [nghttp2](https://nghttp2.org/), [Caddy](https://caddyserver.com/) 以及 [LiteSpeed](https://www.litespeedtech.com/products/litespeed-web-server/overview) 也都发布了可以工作于http2下的服务器。

### 8.3.3 其他

curl和libcurl支持未加密的http2并借助某些TLS库支持了TLS版本。

Wireshark同样支持了http2, 所以用它来分析http2网络数据流着实是再好不过的了。

## 8.4. 对http2的常见批评

在制定协议的讨论过程中往往存在许多争议，甚至会有不少人认为这样的协议最终会以失败告终。这里我想提一些常见的对协议的批评以及我的解释：

### 8.4.1. “这个协议是Google设计制定的”

江湖上有太多传言暗示着这个世界越来越被Google所控制，但事实显然并非如此。这个协议是IETF制定的，就跟过去30年间很多其他协议一样。但不得不承认，SPDY是Google非常出色的成果。它不仅仅证明了开发一个新协议的可行性，还充分展现了新协议所能带来的好处。

而Google也公开[声明](https://blog.chromium.org/2015/02/hello-http2-goodbye-spdy.html)了他们会在2016年移除Chrome里对SPDY和NPN的支持，并且极力推动服务器迁移至HTTP/2。2016年2月他们[声明](https://blog.chromium.org/2016/02/transitioning-from-spdy-to-http2.html)了SPDY和NPN会在Chrome 51被移除.

### 8.4.2. “这个协议只在浏览器上有用”

在一定意义上，这是对的。开发http2的其中一个主要原因就是修复HTTP pipelining。如果在你的应用场景里本来就不需要pipelining，那么确实很有可能http2对你没有太大帮助。虽然这并不是唯一的提升，但显然这是非常重要的一个。

一旦当某些服务意识到在一个连接上建立多路复用流的强大威力时，我认为会有越来越多的程序采用http2。

小规模的REST API和采用HTTP 1.x的简单程序可能并不会从迁移到http2中获得多大的收益。但至少，迁移至http2对绝大部分用户来讲几乎是没有坏处的。

### 8.4.3. “这个协议只对大型网站有用”

完全不是这样。因为缺乏内容分发网络，小网站的网络延迟往往较高，而多路复用的能力可以极大的改善在高网络延迟下的体验。大型网站往往已经将内容分发到各处，所以速度其实已经非常快了。

### 8.4.4. “TLS让速度变得更慢”

这个评价在某种程度上是对的。虽然TLS的握手确实增加了额外的开销，但也有越来越多的方案提出来减少TLS往返的时间。使用TLS而不是纯文本带来的开销是显著的，有可观证据表明，和传输同样的流量相比，TLS会消耗更多的CPU和其他资源。具体影响有多大以及怎么影响是一个和具体测量有关的课题。更多的例子可以参看[istlsfastyet.com](https://istlsfastyet.com/)。

Telecom和一些其他网络服务商，例如ATIS开放网络联盟，表示为了为卫星、飞机等提供的快速网络体验，他们需要一些[不加密的流量](https://www.atis.org/openweballiance/docs/OWAKickoffSlides051414.pdf )来提供caching，压缩和其他技术。

由于http2并不强制要求使用TLS，所以我们不应该为此担心。

如今，很多互联网使用者都希望TLS能被更广泛的使用来保护用户隐私。

实验也证明了通过使用TLS能比用在80端口实现一个新的基于文本的协议更容易成功。因为当前已经有太多中间商使用该方案，所以凡是基于80端口的协议，都很可能被理所当然的当作HTTP 1.1。

最后，得益于http2可以在单一连接上提供多路复用的流，正常使用普通浏览器也可以减少TLS握手的次数，所以使用HTTPS仍然会比HTTP 1.1更快。

### 8.4.5. “不基于ASCII是没法忍受的”

是的，如果我们可以直接读出协议内容，那么调试和追踪都会变得更为简单。但是基于文本的协议更容易产生错误，造成更多解析的问题。

假如你真的无法接受二进制协议，那么你也很难在HTTP 1.x中处理TLS和压缩。因为其实这些技术已经被使用了很久了。

### 8.4.6. “它根本没有比HTTP/1.1快”

当然，到底该如何定义和衡量“快”就是另外一个话题了，但在SPDY的时代，已经有很多实验证明了该协议会让浏览器载入页面变得更快（例如华盛顿大学的[“SPDY有多快？”](https://www.usenix.org/system/files/conference/nsdi14/nsdi14-paper-wang_xiao_sophia.pdf)和Hervé Servy的[“评估启用SPDY后的Web服务器的性能”](https://www.neotys.com/blog/performance-of-spdy-enabled-web-servers/)），同样这些实验也可以被用来证明http2。我期待能有越来越多的诸如此类的测试实验结果发布。而这篇文章[httpwatch.com的一个简单测试](https://blog.httpwatch.com/2015/01/16/a-simple-performance-comparison-of-https-spdy-and-http2/)亦能证明HTTP/2名副其实。<!-- 那一句“我也期待XX”放在那怪怪的 -->


### 8.4.7. “它违反了网络分层”

你确定这也是反对的理由么？网络分层并不是不可侵犯的。如果我们在制定http2的时候已经踏入了灰色地带，那我们当然可以尝试在限制内制定出更好更高效的协议。

### 8.4.8. “它并没有修复很多HTTP/1.1的短板”

确实是这样。兼容HTTP/1.1的范式是我们的目标之一，所以一些老的HTTP功能仍然被保留。例如一些常用的协议头、可怕的cookies、验证头等等。但保留这些范式的好处就是我们在升级到新协议的时候少掉很多工作，也不需要重写很多底层的东西。Http2其实只是一个新的帧层。<!-- 那个"可怕的"cookies该怎么翻译好？ -->

## 8.5. http2会被广泛部署吗？

现在讨论这个议题还言之尚早，但我仍然要在这里做出我的预估。

很多怀疑论者会以“看看IPv6现在的德性”为让我们回想起这个经历了10多年才开始慢慢被采用的协议。但http2毕竟不是IPv6。它是一个建立在TCP之上的使用基于原有HTTP协议升级过后的机制、端口号和TLS等的协议。大部分路由器或者防火墙不需要为此而进行更改。

Google向世界展示了他们的SPDY，证明了像这样的新协议也能在足够短的时间内拥有多种实现，并且能被浏览器和服务所采用。虽然如今支持SPDY服务器端数量在1%以内，但通过这些服务器所交换的数据却要大很多。很多非常流行的网站现在也有提供SPDY支持。

我认为建立在SPDY的基本范式之上的http2会被更广泛的部署，其中一个主要的原因是：它是一个IETF制定的协议。而SPDY则因为背负了“它是Google的协议”这个恶名，导致它的发展总是畏首畏脚。

在它首次发布的幕后有很多大型浏览器支持。来自Firefox，Chrome，Safari，Internet Explorer和Opera的代表宣布了他们会发布支持http2特性的浏览器，并且他们已经演示了一些能正常运作的实现。

也有很多像Google，Twitter和Facebook这样的服务器运营者希望尽快支持http2，也同样希望可以快点在主流服务器实现中出现对http2的支持（例如Apache HTTP Server和nginx）。而[H2o](https://github.com/h2o/h2o)作为一个极具潜力的新生HTTP服务器，已经支持了http2。

那些大型代理程序开发者，例如HAProxy、Squid和Varnish也表示出了他们对支持http2的兴趣。

纵观2015年，http2的流量正在逐步上升。9月初，Firefox 40中http2流量占据了所有HTTP流量中的13%，HTTPS中的27%。与此同时，Google表示约有18%的流量来自HTTP/2。值得注意的是，Google同时也在实验其他协议，这也使得http2的使用量暂时比正常值低一些。


# 9. Firefox里的http2

Firefox紧跟着草案，并且很早之前就实现了http2的测试实现。在http2协议开发的时候，客户端和服务器需要采用同一的协议草案版本，进行测试也变得比较繁琐。所以请一定注意你的客户端和服务器支持的是一样的版本。

## 9.1. 首先，确保它已被启用

从发布于2015年1月13日的Firefox 35之后，http2支持是默认开启的。

在地址栏里进入'about:config'，再搜索一个名为“network.http.spdy.enabled.http2draft”的选项，确保它被设置为`true`。Firefox 36添加了一个“network.http.spdy.enabled.http2”的配置项，并默认设置为*true*。后者控制的是“纯”http2版本，而前者控制了启用／禁用通过http2草案版本进行协商。从Firefox 36之后，这两者都默认为true。

## 9.2. 仅限TLS

请记住Firefox只在TLS上实现了http2。你只会看到http2只在`https://`的网站里得到支持。

## 9.3. 透明！ <!--这个标题改成： 一切都是透明的  怎么样-->

![transparent http2 use](https://image.fyxemmmm.cn/blog/images/h210.png)

在UI上，没有任何元素标明你正在使用http2。但想确认也并不复杂，一种方法是启用“Web developer->Network”，再查看响应头里面服务器发回来的内容。这个响应是“HTTP/2.0”，并且Firefox也插入了一个自己头“X-Firefox-Spdy:”，如上面截图所示。

你在这里看到的头文件是网络工具把二进制的http2格式转换成类似HTTP 1.x显示方式的文本格式。

## 9.4. 图形化HTTP/2

有一些Firefox的插件可以图形化HTTP/2，比如[“HTTP/2 and SPDY Indicator”](https://addons.mozilla.org/en-US/firefox/addon/http2-indicator/)。

# 10. Chromium里的http2

Chromium团队并且很早之前就已经在dev和beta分支里面实现并支持了HTTP/2。从2015年1月27日发布的Chrome 40起，http2已经默认为一些用户启用该功能。虽然刚开始用户的数量会很少，但会慢慢增加。

Chrome 51移除了SPDY的支持来为http2铺路。在2016年2月的一篇[博客](https://blog.chromium.org/2016/02/transitioning-from-spdy-to-http2.html)里面有如下一段话：

> “在Chrome里有超过25%的资源是通过HTTP/2来传输的，而SPDY只有不到5%。考虑到如此大范围的采用，自5月15日，也就是HTTP/2 RFC的周年纪念日起，Chrome将不再支持SPDY。”

## 10.1. 首先，确保它已被启用

在地址栏里进入`chrome://flags/#enable-spdy4`，如果没有被enable的话，点击"enable"启用它。

## 10.2. TLS-only

请记住Chrome只在TLS上实现了http2。你只会在以`https://`做前缀的网站里得到http2的支持。

## 10.3. 图形化HTTP/2

有一些Chrome的插件可以图形化HTTP/2，比如[“HTTP/2 and SPDY Indicator”](https://chrome.google.com/webstore/detail/spdy-indicator/mpbpobfflnpcgagjijhmgnchggcjblin)。

## 10.4. QUIC

Chrome正在试验QUIC，所以或多或少稀释了HTTP/2的份额。


# 11. curl中的http2

[curl项目](https://curl.haxx.se/)从2013年9月就开始对http2提供实验性的支持。

为了遵从curl的要旨，我们尽可能全方位地支持http2。curl通常被用作一个网站连接测试工具，希望这项使命也能在http2上被得以延续。

curl使用一个叫做[nghttp2](https://nghttp2.org/)的库来提供http2帧层的支持。curl依赖于nghttp2 1.0以上版本。

请注意当前linux curl和libcurl并没有默认启用对HTTP/2协议的支持。

## 11.1. 跟HTTP 1.x非常相似

curl会在内部把收到的http2头部转换为HTTP1.x风格的头部再呈现给用户，这样一来，它们就和目前的HTTP非常类似。这也使得无论是用curl还是HTTP，转换都非常容易。<!-- TOREVIEW -->类似地，curl会用相同的方式对发出的HTTP头部做转换，即发给curl的HTTP 1.x风格头部会在被发送到http2服务器之前完成转换。这使得户无需关心底层到底使用的是哪个版本的HTTP协议。

## 11.2. 不安全的纯文本

curl通过升级头部支持基于标准TCP的http2. 当发起一个使用http2的HTTP请求，如果可能，curl会请求服务器把连接升级到http2.

## 11.3. TLS和相关库

curl可以使用许多不同TLS的底层库来提供TLS支持，http2也得这样。TLS兼容http2的挑战来自于对ALPN以及一些NPN扩展的支持。

基于最新版本的OpenSSL或NSS编译curl可以同时获得ALPN和NPN支持。而使用GnuTLS或PolarSSL只能得到ALPN。

## 11.4. 命令行中使用

无论是用纯文本还是通过TLS，必须使用`--http2`参数来让curl使用http2。在未使用该参数的默认情况下，curl会使用HTTP/1.1。

## 11.5. libcurl参数

### 11.5.1 启用HTTP/2

应用程序和从前一样使用`https://`或者`http://`风格的URL，但你可以通过将`curl_easy_setopt`的`SURLOPT_HTTP_VERSION`参数设置为`CURL_HTTP_VERSION_2`来使libcurl尝试使用http2。它将优先尽可能地使用http2，如果不行的话，会继续使用HTTP 1.1。

### 11.5.2 多路复用

正如libcurl想尽可能量维持以前的用法，你需要通过[CURLMOPT_PIPELINING](https://curl.haxx.se/libcurl/c/CURLMOPT_PIPELINING.html)参数为你的程序启用HTTP/2多路复用功能。不然的话，它会保持一个连接只发送一个请求。

另一个需要注意的小细节是，当你通过libcurl同时请求多个传输的时候，请使用多接口模式。这样能使应用程序能同时启用任意数量的传输。如果你宁愿让libcurl等待也要把它们放到同一个连接来传输的话，请使用[CURLOPT_PIPEWAIT](https://curl.haxx.se/libcurl/c/CURLOPT_PIPEWAIT.html)参数。

### 11.5.3 服务器推送

libcurl 7.44.0及其后续版本开始支持HTTP/2服务器推送功能。你可以通过在[CURLMOPT_PUSHFUNCTION](https://curl.haxx.se/libcurl/c/CURLMOPT_PUSHFUNCTION.html)参数中设定一个推送回调来激活该功能。如果应用程序接受了该推送，它将为CURL建立一个新的传输，以便接受内容。<!-- 最后一句需要review -->

