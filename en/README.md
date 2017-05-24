# Vue.js 服务器端渲染指南

> **注意：** 本指南需要最低为如下版本的 Vue，以及以下 library 支持：
> - vue & vue-server-renderer >= 2.3.0
> - vue-router >= 2.5.0
> - vue-loader >= 12.0.0 & vue-style-loader >= 3.0.0

> 如果先前已经使用过 Vue 2.2 的服务器端渲染(SSR)，您应该注意到，推荐的代码结构现在[略有不同](./structure.md)（使用新的 [runInNewContext](./api.md#runinnewcontext) 选项，并设置为 `false`）。现有的应用程序可以继续运行，但建议您迁移到新的推荐规范。

## 什么是服务器端渲染(SSR)?

Vue.js 是构建客户端应用程序的框架。默认情况下，可以在浏览器中输出 Vue 组件，进行生成 DOM 和操作 DOM。然而，也可以将同一个组件渲染为服务器端的 HTML 字符串，将它们直接发送到浏览器，最后将静态标记"混合"为客户端上完全交互的应用程序。

服务器渲染的 Vue.js 应用程序也可以被认为是"同构"或"通用"，因为应用程序的大部分代码都可以在**服务器**和**客户端**上运行。

## 为什么使用服务器端渲染(SSR)?

与传统 SPA（Single-Page Application - 单页应用程序）相比，服务器端渲染(SSR)的优势主要在于：

- 更好的 SEO，由于搜索引擎爬虫抓取工具可以直接查看完全渲染的页面。

  请注意，截至目前，Google 和 Bing 可以很好对同步 JavaScript 应用程序进行索引。在这里，同步是关键。如果您的应用程序初始展示 loading 菊花图，然后通过 Ajax 获取内容，抓取工具并不会等待异步完成后再行抓取页面内容。也就是说，如果 SEO 对你的站点至关重要，而你的页面又是异步获取内容，则你可能需要服务器端渲染(SSR)解决此问题。

- 更快的内容到达时间(time-to-content)，特别是对于缓慢的网络情况或运行缓慢的设备。无需等待所有的 JavaScript 都完成下载并执行，才显示服务器渲染的标记，所以你的用户将会更快速地看到完整渲染的页面。通常可以产生更好的用户体验，并且对于那些「内容到达时间(time-to-content)与转化率直接相关」的应用程序而言，服务器端渲染(SSR)至关重要。

使用服务器端渲染(SSR)时还需要有一些权衡之处：

- 开发条件所限。浏览器特定的代码，只能在某些生命周期钩子函数(lifecycle hook)中使用；一些外部扩展库(external library)可能需要特殊处理，才能在服务器渲染应用程序中运行。

- 涉及构建设置和部署的更多要求。与可以部署在任何静态文件服务器上的完全静态单页面应用程序(SPA)不同，服务器渲染应用程序，需要处于 Node.js server 运行环境。

- 更多的服务器端负载。在 Node.js 中渲染完整的应用程序，显然会比仅仅提供静态文件的 server 更加大量占用 CPU 资源(CPU-intensive - CPU 密集)，因此如果您预料在高流量环境(high traffic)下使用，请准备相应的服务器负载，并明智地采用缓存策略。

在对您的应用程序使用服务器端渲染(SSR)之前，您应该问第一个问题是否真的需要它。这主要取决于内容到达时间(time-to-content)对应用程序的重要程度。例如，如果您正在构建一个内部仪表盘，初始加载时的额外几百毫秒并不重要，这种情况下去使用服务器端渲染(SSR)将是一个小题大作之举。然而，内容到达时间(time-to-content)要求是绝对关键的指标，在这种情况下，服务器端渲染(SSR)可以帮助您实现最佳的初始加载性能。

## SSR vs Prerendering

If you're only investigating SSR to improve the SEO of a handful of marketing pages (e.g. `/`, `/about`, `/contact`, etc), then you probably want __prerendering__ instead. Rather than using a web server to compile HTML on-the-fly, prerendering simply generates static HTML files for specific routes at build time. The advantage is setting up prerendering is much simpler and allows you to keep your frontend as a fully static site.

If you're using Webpack, you can easily add prerendering with the [prerender-spa-plugin](https://github.com/chrisvfritz/prerender-spa-plugin). It's been extensively tested with Vue apps - and in fact, [the creator](https://github.com/chrisvfritz) is a member of the Vue core team.

## About This Guide

This guide is focused on server-rendered Single-Page Applications using Node.js as the server. Mixing Vue SSR with other backend setups is a topic of its own and is not covered in this guide.

This guide will be very in-depth and assumes you are already familiar with Vue.js itself, and have decent working knowledge of Node.js and webpack. If you prefer a higher-level solution that provides a smooth out-of-the-box experience, you should probably give [Nuxt.js](http://nuxtjs.org/) a try. It's built upon the same Vue stack but abstracts away a lot of the boilerplate, and provides some extra features such as static site generation. However, it may not suit your use case if you need more direct control of your app's structure. Regardless, it would still be beneficial to read through this guide to better understand how things work together.

As you read along, it would be helpful to refer to the official [HackerNews Demo](https://github.com/vuejs/vue-hackernews-2.0/), which makes use of most of the techniques covered in this guide.

Finally, note that the solutions in this guide are not definitive - we've found them to be working well for us, but that doesn't mean they cannot be improved. They might get revised in the future - and feel free to contribute by submitting pull requests!

***

> 原文：https://ssr.vuejs.org/en/