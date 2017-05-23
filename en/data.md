# 数据预取和状态

## 数据预取存储容器(Data Store)

在服务器端渲染(SSR)期间，我们本质上是在渲染我们应用程序的"快照"，所以如果应用程序依赖于一些异步数据，**那么在开始渲染过程之前，需要先预取和解析好这些数据**。

另一个需要关注的问题是在客户端，在挂载(mount)到客户端应用程序之前，需要获取到与服务器端应用程序完全相同的数据 - 否则，客户端应用程序会因为使用与服务器端应用程序不同的状态，然后导致混合失败。

为了解决这个问题，获取的数据需要位于视图组件之外，即放置在专门的数据预取存储容器(data store)或"状态容器(state container)）"中。首先，在服务器端，我们可以在渲染之前预取数据，并将数据填充到存储容器(store)中。此外，我们将在 HTML 中序列化(serialize)和内联预置(inline)状态。这样，在挂载(mount)到客户端应用程序之前，可以直接从存储容器(store)获取到内联预置(inline)状态。

为此，我们将使用官方状态管理库 [Vuex](https://github.com/vuejs/vuex/)。我们先创建一个 `store.js` 文件，里面会模拟一些根据 id 获取 item 的逻辑：

``` js
// store.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

// 假定我们有一个可以返回 Promise 的通用 API（请忽略此 API 具体实现细节）
import { fetchItem } from './api'

export function createStore () {
  return new Vuex.Store({
    state: {
      items: {}
    },
    actions: {
      fetchItem ({ commit }, id) {
        // `store.dispatch()` 会返回 Promise，以便我们能够知道数据在何时更新
        return fetchItem(id).then(item => {
          commit('setItem', { id, item })
        })
      }
    },
    mutations: {
      setItem (state, { id, item }) {
        Vue.set(state.items, id, item)
      }
    }
  })
}
```

然后修改 `app.js`：

``` js
// app.js
import Vue from 'vue'
import App from './App.vue'
import { createRouter } from './router'
import { createStore } from './store'
import { sync } from 'vuex-router-sync'

export function createApp () {
  // 创建 router 和 store 实例
  const router = createRouter()
  const store = createStore()

  // 同步路由状态(route state)到存储容器(store)
  sync(store, router)

  // 创建应用程序实例，将 router 和 store 注入
  const app = new Vue({
    router,
    store,
    render: h => h(App)
  })

  // 暴露 app, router 和 store。
  return { app, router, store }
}
```

## 带有逻辑配置(Logic Collocation)的组件

那么，我们在哪里放置「dispatch 数据预取 action」的代码？

我们需要通过访问路由，来决定获取哪部分数据 - 这也决定了哪些组件需要渲染。事实上，给定路由所需的数据，也是在该路由上渲染组件时所需的数据。所以在路由组件中放置数据预取逻辑，是很自然的事情。

我们将在路由组件上暴露出一个自定义静态函数 `asyncData`。注意，由于此函数会在组件实例化之前调用，所以它无法访问 `this`。需要将存储容器(store)和路由信息作为参数传递进去：

``` html
<!-- Item.vue -->
<template>
  <div>{{ item.title }}</div>
</template>

<script>
export default {
  asyncData ({ store, route }) {
    // 触发 action 后，会返回 Promise
    return store.dispatch('fetchItem', route.params.id)
  },

  computed: {
    // 从存储容器(store)的 state 对象中的获取 item。
    item () {
      return this.$store.state.items[this.$route.params.id]
    }
  }
}
</script>
```

## Server Data Fetching

In `entry-server.js` we can get the components matched by a route with `router.getMatchedComponents()`, and call `asyncData` if the component exposes it. Then we need to attach resolved state to the render context.

``` js
// entry-server.js
import { createApp } from './app'

export default context => {
  return new Promise((resolve, reject) => {
    const { app, router, store } = createApp()

    router.push(context.url)

    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      if (!matchedComponents.length) {
        return reject({ code: 404 })
      }

      // call `asyncData()` on all matched route components
      Promise.all(matchedComponents.map(Component => {
        if (Component.asyncData) {
          return Component.asyncData({
            store,
            route: router.currentRoute
          })
        }
      })).then(() => {
        // After all preFetch hooks are resolved, our store is now
        // filled with the state needed to render the app.
        // When we attach the state to the context, and the `template` option
        // is used for the renderer, the state will automatically be
        // serialized and injected into the HTML as window.__INITIAL_STATE__.
        context.state = store.state

        resolve(app)
      }).catch(reject)
    }, reject)
  })
}
```

When using `template`, `context.state` will automatically be embedded in the final HTML as `window.__INITIAL_STATE__` state. On the client, the store should pick up the state before mounting the application:

``` js
// entry-client.js

const { app, router, store } = createApp()

if (window.__INITIAL_STATE__) {
  store.replaceState(window.__INITIAL_STATE__)
}
```

## Client Data Fetching

On the client, there are two different approaches for handling data fetching:

1. **Resolve data before route navigation:**

  With this strategy, the app will stay on the current view until the data needed by the incoming view has been resolved. The benefit is that the incoming view can directly render the full content when it's ready, but if the data fetching takes a long time, the user will feel "stuck" on the current view. It is therefore recommended to provide a data loading indicator if using this strategy.

  We can implement this strategy on the client by checking matched components and invoking their `asyncData` function inside a global route hook. Note we should register this hook after the initial route is ready so that we don't unnecessarily fetch the server-fetched data again.

  ``` js
  // entry-client.js

  // ...omitting unrelated code

  router.onReady(() => {
    // Add router hook for handling asyncData.
    // Doing it after initial route is resolved so that we don't double-fetch
    // the data that we already have. Using `router.beforeResolve()` so that all
    // async components are resolved.
    router.beforeResolve((to, from, next) => {
      const matched = router.getMatchedComponents(to)
      const prevMatched = router.getMatchedComponents(from)

      // we only care about none-previously-rendered components,
      // so we compare them until the two matched lists differ
      let diffed = false
      const activated = matched.filter((c, i) => {
        return diffed || (diffed = (prevMatched[i] !== c))
      })

      if (!activated.length) {
        return next()
      }

      // this is where we should trigger a loading indicator if there is one

      Promise.all(activated.map(c => {
        if (c.asyncData) {
          return c.asyncData({ store, route: to })
        }
      })).then(() => {

        // stop loading indicator

        next()
      }).catch(next)
    })

    app.$mount('#app')
  })
  ```

2. **Fetch data after the matched view is rendered:**

  This strategy places the client-side data-fetching logic in a view component's `beforeMount` function. This allows the views to switch instantly when a route navigation is triggered, so the app feels a bit more responsive. However, the incoming view will not have the full data available when it's rendered. It is therefore necessary to have a conditional loading state for each view component that uses this strategy.

  This can be achieved with a client-only global mixin:

  ``` js
  Vue.mixin({
    beforeMount () {
      const { asyncData } = this.$options
      if (asyncData) {
        // assign the fetch operation to a promise
        // so that in components we can do `this.dataPromise.then(...)` to
        // perform other tasks after data is ready
        this.dataPromise = asyncData({
          store: this.$store,
          route: this.$route
        })
      }
    }
  })
  ```

The two strategies are ultimately different UX decisions and should be picked based on the actual scenario of the app you are building. But regardless of which strategy you pick, the `asyncData` function should also be called when a route component is reused (same route, but params or query changed. e.g. from `user/1` to `user/2`). We can also handle this with a client-only global mixin:

``` js
Vue.mixin({
  beforeRouteUpdate (to, from, next) {
    const { asyncData } = this.$options
    if (asyncData) {
      asyncData({
        store: this.$store,
        route: to
      }).then(next).catch(next)
    } else {
      next()
    }
  }
})
```

## Store Code Splitting

In a large application, our vuex store will likely be split into multiple modules. Of course, it is also possible to code-split these modules into corresponding route component chunks. Suppose we have the following store module:

``` js
// store/modules/foo.js
export default {
  namespaced: true,
  // IMPORTANT: state must be a function so the module can be
  // instantiated multiple times
  state: () => ({
    count: 0
  }),
  actions: {
    inc: ({ commit }) => commit('inc')
  },
  mutations: {
    inc: state => state.count++
  }
}
```

We can use `store.registerModule` to lazy-register this module in a route component's `asyncData` hook:

``` html
// inside a route component
<template>
  <div>{{ fooCount }}</div>
</template>

<script>
// import the module here instead of in `store/index.js`
import fooStoreModule from '../store/modules/foo'

export default {
  asyncData ({ store }) {
    store.registerModule('foo', fooStoreModule)
    return store.dispatch('foo/inc')
  },

  // IMPORTANT: avoid duplicate module registration on the client
  // when the route is visited multiple times.
  destroyed () {
    this.$store.unregisterModule('foo')
  },

  computed: {
    fooCount () {
      return this.$store.state.foo.count
    }
  }
}
</script>
```

Because the module is now a dependency of the route component, it will be moved into the route component's async chunk by webpack.

---

Phew, that was a lot of code! This is because universal data-fetching is probably the most complex problem in a server-rendered app and we are laying the groundwork for easier further development. Once the boilerplate is set up, authoring individual components will be actually quite pleasant.

***

> 原文：https://ssr.vuejs.org/en/data.html