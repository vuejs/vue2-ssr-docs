# Mise en cache (En) <br><br> *Cette page est en cours de traduction française. Revenez une autre fois pour lire une traduction achevée ou [participez à la traduction française ici](https://github.com/vuejs-fr/vue-ssr-docs).*

Although Vue's SSR is quite fast, it can't match the performance of pure string-based templating due to the cost of creating component instances and Virtual DOM nodes. In cases where SSR performance is critical, wisely leveraging caching strategies can greatly improve response time and reduce server load.

## Page-level Caching

A server-rendered app in most cases relies on external data, so the content is dynamic by nature and cannot be cached for extended periods. However, if the content is not user-specific (i.e. for the same URL it always renders the same content for all users), we can leverage a strategy called [micro-caching](https://www.nginx.com/blog/benefits-of-microcaching-nginx/) to drastically improve our app's capability of handling high traffic.

This is usually done at the Nginx layer, but we can also implement it in Node.js:

``` js
const microCache = LRU({
  max: 100,
  maxAge: 1000 // Important: entries expires after 1 second.
})

const isCacheable = req => {
  // implement logic to check if the request is user-specific.
  // only non-user-specific pages are cache-able
}

server.get('*', (req, res) => {
  const cacheable = isCacheable(req)
  if (cacheable) {
    const hit = microCache.get(req.url)
    if (hit) {
      return res.end(hit)
    }
  }

  renderer.renderToString((err, html) => {
    res.end(html)
    if (cacheable) {
      microCache.set(req.url, html)
    }
  })
})
```

Since the content is cached for only one second, users will not see outdated content. However, this means the server only has to perform at most one full render per second for each cached page.

## Component-level Caching

`vue-server-renderer` has built-in support for component-level caching. To enable it you need to provide a [cache implementation](./api.md#cache) when creating the renderer. Typical usage is passing in an [lru-cache](https://github.com/isaacs/node-lru-cache):

``` js
const LRU = require('lru-cache')

const renderer = createRenderer({
  cache: LRU({
    max: 10000,
    maxAge: ...
  })
})
```

You can then cache a component by implementing a `serverCacheKey` function:

``` js
export default {
  name: 'item', // required
  props: ['item'],
  serverCacheKey: props => props.item.id,
  render (h) {
    return h('div', this.item.id)
  }
}
```

Note that cache-able component **must also define a unique "name" option**. With a unique name, the cache key is thus per-component: you don't need to worry about two components returning the same key.

The key returned from `serverCacheKey` should contain sufficient information to represent the shape of the render result. The above is a good implementation if the render result is solely determined by `props.item.id`. However, if the item with the same id may change over time, or if render result also relies on another prop, then you need to modify your `getCacheKey` implementation to take those other variables into account.

Returning a constant will cause the component to always be cached, which is good for purely static components.

### When to use component caching

If the renderer hits a cache for a component during render, it will directly reuse the cached result for the entire sub tree. This means you should **NOT** cache a component when:

- It has child components that may rely on global state.
- It has child components that produces side effects on the render `context`.

Component caching should therefore be applied carefully to address performance bottlenecks. In most cases, you shouldn't and don't need to cache single-instance components. The most common type of components that are suitable for caching are ones repeated in big `v-for` lists. Since these components are usually driven by objects in database collections, they can make use of a simple caching strategy: generate their cache keys using their unique id plus the last updated timestamp:

``` js
serverCacheKey: props => props.item.id + '::' + props.item.last_updated
```
