

> gdbprof，是一款基于系统时间（而非cpu事件）的性能分析工具，它通过sampling的方式进行性能统计的。每隔period（默认为0.1秒），给gdb发送sigint信号，然后循环每个thread，根据该thread的call trace，统计函数的调用情况。
它是基于python实现的，中间用到了gdb的python api。<br>
>[gdbprof源码](https://github.com/markhpc/gdbprof)
>本文采用的[gdbprof优化版本](https://github.com/liupan1111/gdbprof)<br>
>[本人forked版本](https://github.com/yinminggang/gdbprof),随时欢迎


## 实验环境

搭建ceph-osd集群（本文用了3个osd.0、osd.1、osd.2）

服务端：server_host（osd）

客户端：client_host（mons、mgrs）

服务端需安装gdb，客户端需安装fio（都自行解决），下载[gdbprof](https://github.com/liupan1111/gdbprof)

## 实验过程

###（1）简单查看下集群是否正常：
**$ceph osd tree**
![](/includes/posts/2018-03-17-gdbprof-analysis-ceph-performance/img/ceph_cluster_health1.png)

**$ceph -s**
![](/posts/2018-03-17-gdbprof-analysis-ceph-performance/img/ceph_cluster_heath2.png)

###（2）PUSH/PRELOAD，推送/预加载初始 URL 路由所需的关键资源。

无论是 HTTP2 Server Push 还是 `<link rel="preload">`，其关键都在于，我们希望提前请求一些隐藏在应用依赖关系（Dependency Graph）较深处的资源，以节省 HTTP 往返、浏览器解析文档、或脚本执行的时间。比如说，对于一个基于路由进行 code splitting 的 SPA，如果我们可以在 webpack 清单、路由等入口代码（entry chunks）被下载与运行之前就把初始 URL，即用户访问的入口 URL 路由所依赖的代码用 Server Push 推送或 `<link rel="preload">` 进行提前加载。那么当这些资源被真正请求时，它们可能已经下载好并存在在缓存中了，这样就加快了初始路由所有依赖的就绪。

在多页应用中，每一个路由本来就只会请求这个路由所需要的资源，并且通常依赖也都比较扁平。饿了么移动站的大部分脚本依赖都是普通的 `<script>` 元素，因此他们可以在文档解析早期就被浏览器的 preloader 扫描出来并且开始请求，其效果其实与显式的 `<link rel="preload">` 是一致的。

![](/img/in-post/post-eleme-pwa/PUSH-link-rel-preload.jpg)

我们还将所有关键的静态资源都伺服在同一域名下（不再做域名散列），以更好的利用 HTTP2 带来的多路复用（Multiplexing）。同时，我们也在进行着对 API 进行 Server Push 的[实验](https://zhuanlan.zhihu.com/p/26757514)。



### 2. RENDER，渲染初始路由，尽快让应用可被交互

既然所有初始路由的依赖都已经就绪，我们就可以尽快开始初始路由的渲染，这有助于提升应用诸如首次渲染时间、可交互时间等指标。多页应用并不使用基于 JavaScript 的路由，而是传统的 HTML 跳转机制，所以对于这一部分，多页应用其实不用额外做什么。


### 3. PRE-CACHE，用 Service Worker 预缓存剩下的路由

这一部分就需要 [Service Worker][9] 的参与了，Service Worker 是一个位于浏览器与网络之间的客户端代理，它以可拦截、处理、响应流经的 HTTP 请求，使得开发者得以从缓存中向 web 应用提供资源而闻名。不过，Service Worker 其实也可以主动发起 HTTP 请求，在“后台” 预请求与预缓存我们未来所需要的资源。

![](/img/in-post/post-eleme-pwa/PRECACHE-future-routes.jpg)


我们已经使用 [Webpack][10] 在构建过程中进行 `.vue` 编译、文件名哈希等工作，于是我们编写了一个 webpack 插件来帮助我们收集需要缓存的依赖到一个“预缓存清单”中，并使用这个清单在每次构建时生成新的 Service Worker 文件。在新的 Service Worker 被激活时，清单里的资源就会被请求与缓存，这其实与 [SW-Precache 这个库的运行机制][11]非常接近。

**实际上，我们只对我们标记为“关键路由”的路由进行依赖收集。**你可以将这些“关键路由”的依赖理解为我们整个应用的 ["App Shell"][12] 或者说“安装包”。一旦它们都被缓存，或者说成功安装，无论用户是在线离线，我们的 web 应用都可以从缓存中直接启动。对于那些并不那么重要的路由，我们则采取在运行时增量缓存的方式。我们使用的 [SW-Toolbox][13] 提供了 LRU 替换策略与 TTL 失效机制，可以保证我们的应用不会超过浏览器的缓存配额。

### 4. LAZY-LOAD 按需懒加载、懒实例化剩下的路由

懒加载与懒实例化剩下的路由对于 SPA 是一件相对麻烦点儿的事情，你需要实现基于路由的 code splitting 与异步加载。幸运的是，这又是一件不需要多页应用担心的事情，多页应用中的各个路由天生就是分离的。

值得说明的是，无论单页还是多页应用，如果在上一步中，我们已经将这些路由的资源都预先下载与缓存好了，那么懒加载就几乎是瞬时完成的了，这时候我们就只需要付出实例化的代价。

---


这四句话即是 PRPL 的全部了。有趣的是，我们发现多页应用在实现 PRPL 这件事甚至比单页还要容易一些。那么结果如何呢？

![](/img/in-post/post-eleme-pwa/Lighthouse-before.png)

根据 Google 推出的 Web 性能分析工具 Lighthouse（v1.6），在模拟的 3G 网络下，用户的初次访问（无任何缓存）大约在 2 秒左右达到“可交互”，可以说非常不错。而对于再次访问，由于所有资源都直接来自于 Service Worker 缓存，页面可以在 1 秒左右就达到可交互的状态了。

但是，故事并不是这么简单得就结束了。在实际的体验中我们发现，**应用在页与页的切换时，仍然存在着非常明显的白屏空隙**，由于 PWA 是全屏运行的，白屏对用户体验所带来的负面影响甚至比以往在浏览器内更大。我们不是已经用 Service Worker 缓存了所有资源了吗，怎么还会这样呢？

![](/img/in-post/post-eleme-pwa/before-skeleton.jpg)
*从首页点击到发现页，跳转过程中的白屏*



## 多页应用的陷阱：重启开销


与 SPA 不同，在多页应用中，路由的切换是原生的浏览器文档跳转（Navigating across documents），这意味着之前的页面会被完全丢弃而浏览器需要为下一个路由的页面重新执行所有的启动步骤：重新下载资源、重新解析 HTML、重新运行 JavaScript、重新解码图片、重新布局页面、重新绘制……即使其中的很多步骤本是可以在多个路由之间复用的。这些工作无疑将产生巨大的计算开销，也因此需要付出相当的时间成本。

图中为我们的入口页（同时也是最重的页面）在 2 倍 CPU 节流模拟下的 profile 数据。即使我们可以将“可交互时间”控制在 1 秒左右，我们的用户仍然会觉得这对于“仅仅切换个标签”来说实在是太慢了。

![](/img/in-post/post-eleme-pwa/msite-Before-Optim.png)

### 巨大的 JavaScript 重启开销

根据 Profile，我们发现在首次渲染（First Paint）发生之前，大量的时间（900 毫秒）都消耗在了 JavaScript 的运行上（Evaluate Script）。几乎所有脚本都是阻塞的（Parser-blocking），不过因为所有的 UI 都是由 JavaScript/Vue 驱动的，倒也不会有性能影响。这 900ms 中，约一半是消耗在包括 Vue 运行时、组件、库等依赖的运行上，而另一半则花在了业务组件实例化时 Vue 的启动与渲染上。从软件工程角度来说，我们需要这些抽象，所以这里并不是想责怪 JavaScript 或是 Vue 所带来的开销。

**但是，在 SPA 中，JavaScript 的启动成本是均摊到整个生命周期的：** 每个脚本都只需要被解析与编译一次，诸如生成 Virtual DOM 等较重的任务可以只执行一次，像 Vue 的 ViewModel 或是 Virtual DOM 这样的大对象也可以被留在内存里复用。**可惜在多页应用里就不是这样了，我们每次切换页面都为 JavaScript 付出了巨大的重启代价。**

### 浏览器的缓存啊，能不能帮帮忙？

能，也不能。

V8 提供了[代码缓存（code caching）](http://v8project.blogspot.com/2015/07/code-caching.html)，可以将编译后的机器码在本地拷贝一份，这样我们就可以在下次请求同一个脚本时一次省略掉请求、解析、编译的所有工作。而且，对于缓存在 Service Worker 配套的 Cache Storage 中的脚本，会在第一次执行后就触发 V8 的代码缓存，这对于我们的多页切换能提供不少帮助。

另外一个你或许听过的浏览器缓存叫做“进退缓存”，Back-Forward Cache，简称 bfcache。浏览器厂商对其的命名各异，Opera 称之为 Fast History Navigation，Webkit 称其为 Page Cache。但是思路都一样，**就是我们可以让浏览器在跳转时把前一页留存在内存中，保留 JavaScript 与 DOM 的状态，而不是全都销毁掉。**你可以随便找个传统的多页网站在 iOS Safari 上试试，无论是通过浏览器的前进后退按钮、手势，还是通过超链接（会有一些不同），基本都可以看到瞬间加载的效果。

Bfcache 其实非常适合多页应用。但不幸的是，Chrome 由于内存开销与其多进程架构等原因目前并不支持。Chrome 现阶段仅仅只是用了传统的 HTTP 磁盘缓存，来稍稍简化了一下加载过程而已。对于 Chromium 内核霸占的 Android 生态来说，我们没法指望了。


## 为“感知体验”奋斗

尽管多页应用面临着现实中的不少性能问题，我们并不想这么快就妥协。一方面，我们尝试尽可能减少在页面达到可交互时间前的代码执行量，比如减少/推迟一些依赖脚本的执行，还有减少初次渲染的 DOM 节点数以节省 Virtual DOM 的初始化开销。另一方面，我们也意识到应用在感知体验上还有更多的优化空间。

Chrome 产品经理 Owen 写过一篇 [Reactive Web Design: The secret to building web apps that feel amazing](https://medium.com/@owencm/reactive-web-design-the-secret-to-building-web-apps-that-feel-amazing-b5cbfe9b7c50)，谈到两种改进感知体验的手段：一是使用骨架屏（Skeleton Screen）来实现瞬间加载；二是预先定义好元素的尺寸来保证加载的稳定。跟我们的做法可以说不谋而合。

为了消除白屏时间，我们同样引入了尺寸稳定的骨架屏来帮助我们实现瞬间的加载与占位。即使是在硬件很弱的设备上，我们也可以在点击切换标签后立刻渲染出目标路由的骨架屏，以保证 UI 是稳定、连续、有响应的。我录了[两个](https://youtu.be/K5JBGnMYO1s)[视频](https://youtu.be/w1ZbNsHmRjs)放在 Youtube 上，不过如果你是国内读者，你可以直接访问饿了么移动网站来体验实地的效果 ;) 最终效果如下图所示。

![](/img/in-post/post-eleme-pwa/after-skeleton.jpg)
*在添加骨架屏后，从发现页点回首页的效果*

这效果本该很轻松的就能实现，不过实际上我们还费了点功夫。


### 在构建时使用 Vue 预渲染骨架屏

你可能已经想到了，为了让骨架屏可以被 Service Worker 缓存，瞬间加载并独立于 JavaScript 渲染，我们需要把组成骨架屏的 HTML 标签、CSS 样式与图片资源一并内联至各个路由的静态 `*.html` 文件中。

不过，我们并不准备手动编写这些骨架屏。你想啊，如果每次真实组件有迭代（每一个路由对我们来说都是一个 Vue 组件）我们都需要手动去同步每一个变化到骨架屏的话，那实在是太繁琐且难以维护了。好在，[骨架屏不过是当数据还未加载进来前，页面的一个空白版本而已](https://www.lukew.com/ff/entry.asp?1797)。如果我们能将骨架屏实现为真实组件的一个特殊状态 —— “空状态”的话，我们理论上就可以从真实组件中直接渲染出骨架屏来。

而 Vue 的多才多艺就在这时体现出来了，我们真的可以用 [Vue.js 的服务端渲染模块](https://ssr.vuejs.org/en/) 来实现这个想法，不过不是用在真正的服务器上，而是在构建时用它把组件的空状态预先渲染成字符串并注入到 HTML 模板中。你需要调整你的 Vue 组件代码使得它可以在 Node 上执行，有些页面对 DOM/BOM 的依赖一时无法轻易去除得，我们目前只好额外编写一个 `*.shell.vue` 来暂时绕过这个问题。


### 关于浏览器的绘制（Painting）

HTML 文件中有标签并不意味着这些标签就能立刻被绘制到屏幕上，你必须保证页面的[关键渲染路径](https://developers.google.com/web/fundamentals/performance/critical-rendering-path/)是为此优化的。很多开发者相信将 script 标签放在 body 的底部就足以保证内容能在脚本执行之前被绘制，这对于能渲染不完整 DOM 树的浏览器（比如桌面浏览器常见的流式渲染）来说可能是成立的。但移动端的浏览器很可能因为考虑到较慢的硬件、电量消耗等因素并不这么做。**不仅如此，即使你曾被告知设为 `async` 或 `defer` 的脚本就不会阻塞 HTML 解析了，但这可不意味着浏览器就一定会在执行它们之前进行渲染。**

![](https://html.spec.whatwg.org/images/asyncdefer.svg)

首先我想澄清的是，根据 [HTML 规范 Scripting 章节](https://html.spec.whatwg.org/multipage/scripting.html)，`async` 脚本是在其请求完成后立刻运行的，因此它本来就可能阻塞到解析。只有 `defer`（且非内联）与最新的 `type=module` 被指定为“一定不会阻塞解析”。（不过 `defer` 目前也有点小问题……我们稍后会再提到）

**而更重要的是，一个不阻塞 HTML 解析的脚本仍然可能阻塞到绘制。**我做了一个简化的**“最小多页 PWA”**（Minimal Multi-page PWA，或 MMPWA）来测试这个问题，：我们在一个 `async`（且确实不阻塞 HTML 解析）脚本中，生成并渲染 1000 个列表项，然后测试骨架屏能否在脚本执行之前渲染出来。下面是通过 USB Debugging 在我的 Nexus 5 真机上录制的 profile：

![](/img/in-post/post-eleme-pwa/thisTick-&-Load.png)


是的，出乎意料吗？首次渲染确实被阻塞到脚本执行结束后才发生。究其原因，**如果我们在浏览器还未完成上一次绘制工作之前就过快得进行了 DOM 操作，我们亲爱的浏览器就只好抛弃所有它已经完成的像素，且一直要等待到 DOM 操作引起的所有工作结束之后才能重新进行下一次渲染。**而这种情况更容易在拥有较慢 CPU/GPU 的移动设备上出现。


### 黑魔法：利用 setTimeout() 让绘制提前

不难发现，骨架屏的绘制与脚本执行实际是一个竞态。大概是 Vue 太快了，我们的骨架屏还是有非常大的概率绘制不出来。于是我们想着如何能让脚本执行慢点，或者说，“懒”点。于是我们想到了一个经典的 Hack： `setTimeout(callback, 0)`。我们试着把 MMPWA 中的 DOM 操作（渲染 1000 个列表）放进 `setTimeout(callback, 0)` 里……

![](/img/in-post/post-eleme-pwa/nextTick-&-Load.png)

当当！首次渲染瞬间就被提前了。如果你熟悉浏览器的**事件循环模型（event loop）**的话，这招 Hack 其实是通过 setTimeout 的回调把 DOM 操作放到了事件循环的任务队列中以避免它在当前循环执行，这样浏览器就得以在主线程空闲时喘息一下（更新一下渲染）了。如果你想亲手试试 MMPWA 的话，你可以访问 [github.com/Huxpro/mmpwa](https://github.com/Huxpro/mmpwa) 或 [huangxuan.me/mmpwa/](https://huangxuan.me/mmpwa) 访问代码与 Demo。我把 UI 设计为了 A/B Test 的形式并改为渲染 5000 个列表项来让效果更夸张一些。

回到饿了么 PWA 上，我们同样试着把 `new Vue()` 放到了 `setTimeout` 中。果然，黑魔法再次显灵，骨架屏在每次跳转后都能立刻被渲染。这时的 Profile 看起来是这样的：

![](/img/in-post/post-eleme-pwa/msite-After-Optim.png)

现在，我们在 400ms 时触发首次渲染（骨架屏），在 600ms 时完成真实 UI 的渲染并达到页面的可交互。你可以拉上去详细对比下优化前后 profile 的区别。


### 被我 “defer” 的有关 `defer` 的 Bug

不知道你发现没有，在上图的 Profile 中，我们仍然有不少脚本是阻塞了 HTML 解析的。好吧让我解释一下，由于历史原因，我们确实保留了一部分的阻塞脚本，比如侵入性很强的 [lib-flexible](https://github.com/amfe/lib-flexible)，我们没法轻易去除它。不过，profile 里的大部分阻塞脚本实际上都设置了 `defer`，我们本以为他们应该在 HTML 解析完成之后才被执行，结果被 profile 打了一脸。

我和 [Jake Archibald](https://twitter.com/jaffathecake) [聊了一下](https://twitter.com/Huxpro/status/859842124849827841)，果然这是 Chrome 的 Bug：`defer` 的脚本被完全缓存时，并没有遵守规范等待解析结束，反而阻塞了解析与渲染。Jake 已经提交在 [crbug](https://bugs.chromium.org/p/chromium/issues/detail?id=717979) 上了，一起给它投票吧~

最后，是优化后的 Lighthouse 跑分结果，同样可以看到明显的性能提升。需要说明的是，能影响 Lighthouse 跑分的因素有很多，所以我建议你以控制变量（跑分用的设备、跑分时的网络环境等）的方式来进行对照实验。

![](/img/in-post/post-eleme-pwa/Lighthouse-after.png)

最后附上一张图，这张图当时是做给 Addy Osmani 的 I/O 演讲用的，描述了饿了么 PWA 是如何结合 Vue 来实现多页应用的 PRPL 模式，可以作为一个架构的参考与示意图。

![](/img/in-post/post-eleme-pwa/Architecture.png)


## 一些感想

### 多页应用仍然有很长的路要走

Web 是一个极其多样化的平台。从静态的博客，到电商网站，再到桌面级的生产力软件，它们全都是 Web 这个大家庭的第一公民。而我们组织 web 应用的方式，也同样只会更多而不会更少：多页、单页、Universal JavaScript 应用、WebGL、以及可以预见的 Web Assembly。不同的技术之间没有贵贱，但是适用场景的差距确是客观存在的。

[Jake](https://twitter.com/jaffathecake) 曾在 [Chrome Dev Summit 2016](https://youtu.be/J2dOTKBoTL4?list=PLNYkxOF6rcIBTs2KPy1E6tIYaWoFcG3uj) 上说过 "PWA !== SPA"。可是尽管我们已经用上了一系列最新的技术（PRPL、Service Worker、App Shell……），我们仍然因为多页应用模型本身的缺陷有着难以逾越的一些障碍。多页应用在未来可能会有“bfcache API”、Navigation Transition 等新的规范以缩小跟 SPA 的距离，不过我们也必须承认，时至今日，多页应用的局限性也是非常明显的。


### 而 PWA 终将带领 web 应用进入新的时代

即使我们的多页应用在升级 PWA 的路上不如单页的那些来得那么闪亮，但是 PWA 背后的想法与技术却实实在在的帮助我们在 web 平台上提供了更好的用户体验。

PWA 作为[下一代 Web 应用模型](https://zhuanlan.zhihu.com/p/25167289)，其尝试解决的是 web 平台本身的根本性问题：对网络与浏览器 UI 的硬依赖。因此，任何 web 应用都可以从中获益，这与你是多页还是单页、面向桌面还是移动端、是用 React 还是 Vue 无关。或许，它还终将改变用户对移动 web 的期待。现如今，谁还觉得桌面端的 web 只是个看文档的地方呢？

还是那句老话：让我们的用户，也像我们这般热爱 web 吧。

---

最后，感谢饿了么的王亦斯、任光辉、题叶，Google 的 Michael Yeung、DevRel 团队， UC 浏览器团队，腾讯 X5 浏览器团队在这次项目中的合作。感谢尤雨溪、陈蒙迪和 Jake Archibald 在写作过程中给予我的帮助。



[1]: https://twitter.com/vuejs/status/834087199008239619
[2]: https://developers.google.com/web/progressive-web-apps/
[3]: https://blog.twitter.com/2017/how-we-built-twitter-lite
[4]: https://medium.com/progressive-web-apps/building-flipkart-lite-a-progressive-web-app-2c211e641883
[5]: https://medium.com/engineering-housing/progressing-mobile-web-fac3efb8b454
[6]: https://shop.polymer-project.org/
[7]: https://developers.google.com/web/fundamentals/performance/prpl-pattern/
[8]: https://calendar.perfplanet.com/2013/big-bad-preloader/
[9]: https://w3c.github.io/ServiceWorker/v1/
[10]: https://webpack.github.io/
[11]: https://medium.com/@Huxpro/how-does-sw-precache-works-2d99c3d3c725
[12]: https://developers.google.com/web/updates/2015/11/app-shell
[13]: https://googlechrome.github.io/sw-toolbox/

