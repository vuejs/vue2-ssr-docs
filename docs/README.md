# Vue.js Server-Side Rendering Guide

::: tip NOTE
This guide requires the following minimum versions of Vue and supporting libraries:

- vue & vue-server-renderer 2.3.0+
- vue-router 2.5.0+
- vue-loader 12.0.0+ & vue-style-loader 3.0.0+

If you have previously used Vue 2.2 with SSR, you will notice that the recommended code structure is now [a bit different](./guide/structure.md) (with the new [runInNewContext](./api/#runinnewcontext) option set to `false`). Your existing app should continue to work, but it's recommended to migrate to the new recommendations.
:::

## What is Server-Side Rendering (SSR)?

Vue.js is a framework for building client-side applications. By default, Vue  components produce and manipulate DOM in the browser as output. However, it is also possible to render the same components into HTML strings on the server, send them directly to the browser, and finally "hydrate" the static markup into a fully interactive app on the client.

A server-rendered Vue.js app can also be considered "isomorphic" or "universal", in the sense that the majority of your app's code runs on both the server **and** the client.

## Why SSR?

Compared to a traditional SPA (Single-Page Application), the advantage of SSR primarily lies in:

- Better SEO, as the search engine crawlers will directly see the fully rendered page.

  Note that as of now, Google and Bing can index synchronous JavaScript applications just fine. Synchronous being the key word there. If your app starts with a loading spinner, then fetches content via Ajax, the crawler will not wait for you to finish. This means if you have content fetched asynchronously on pages where SEO is important, SSR might be necessary.

- Faster time-to-content, especially on slow internet or slow devices. Server-rendered markup doesn't need to wait until all JavaScript has been downloaded and executed to be displayed, so your user will see a fully-rendered page sooner. This generally results in better user experience, and can be critical for applications where time-to-content is directly associated with conversion rate.

There are also some trade-offs to consider when using SSR:

- Development constraints. Browser-specific code can only be used inside certain lifecycle hooks; some external libraries may need special treatment to be able to run in a server-rendered app.

- More involved build setup and deployment requirements. Unlike a fully static SPA that can be deployed on any static file server, a server-rendered app requires an environment where a Node.js server can run.

- More server-side load. Rendering a full app in Node.js is obviously going to be more CPU-intensive than just serving static files, so if you expect high traffic, be prepared for corresponding server load and wisely employ caching strategies.

Before using SSR for your app, the first question you should ask is whether you actually need it. It mostly depends on how important time-to-content is for your app. For example, if you are building an internal dashboard where an extra few hundred milliseconds on initial load doesn't matter that much, SSR would be an overkill. However, in cases where time-to-content is absolutely critical, SSR can help you achieve the best possible initial load performance.

## SSR vs Prerendering

If you're only investigating SSR to improve the SEO of a handful of marketing pages (e.g. `/`, `/about`, `/contact`, etc), then you probably want __prerendering__ instead. Rather than using a web server to compile HTML on-the-fly, prerendering simply generates static HTML files for specific routes at build time. The advantage is setting up prerendering is much simpler and allows you to keep your frontend as a fully static site.

If you're using webpack, you can easily add prerendering with the [prerender-spa-plugin](https://github.com/chrisvfritz/prerender-spa-plugin). It's been extensively tested with Vue apps - and in fact, [the creator](https://github.com/chrisvfritz) is a member of the Vue core team.

## About This Guide

This guide is focused on server-rendered Single-Page Applications using Node.js as the server. Mixing Vue SSR with other backend setups is a topic of its own and briefly discussed in a [dedicated section](./guide/non-node.md).

This guide will be very in-depth and assumes you are already familiar with Vue.js itself, and have decent working knowledge of Node.js and webpack. If you prefer a higher-level solution that provides a smooth out-of-the-box experience, you should probably give [Nuxt.js](https://nuxtjs.org/) a try. It's built upon the same Vue stack but abstracts away a lot of the boilerplate, and provides some extra features such as static site generation. However, it may not suit your use case if you need more direct control of your app's structure. Regardless, it would still be beneficial to read through this guide to better understand how things work together.

As you read along, it would be helpful to refer to the official [HackerNews Demo](https://github.com/vuejs/vue-hackernews-2.0/), which makes use of most of the techniques covered in this guide.

Finally, note that the solutions in this guide are not definitive - we've found them to be working well for us, but that doesn't mean they cannot be improved. They might get revised in the future - and feel free to contribute by submitting pull requests!
