# Data Pre-Fetching and State

## Data Store

During SSR, we are essentially rendering a "snapshot" of our app, so if the app relies on some asynchronous data, **these data need to be pre-fetched and resolved before we start the rendering process**.

Another concern is that on the client, the same data needs to be available before we mount the client side app - otherwise the client app would render using different state and the hydration would fail.

To address this, the fetched data needs to live outside the view components, in a dedicated data store, or a "state container". On the server, we can pre-fetch and fill data into the store before rendering. In addition, we will serialize and inline the state in the HTML. The client-side store can directly pick up the inlined state before we mount the app.

We will be using the official state management library [Vuex](https://github.com/vuejs/vuex/) for this purpose. Let's create a `store.js` file, with some mocked logic for fetching an item based on an id:

``` js
// store.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

// Assume we have a universal API that returns Promises
// and ignore the implementation details
import { fetchItem } from './api'

export function createStore () {
  return new Vuex.Store({
    state: {
      items: {}
    },
    actions: {
      fetchItem ({ commit }, id) {
        // return the Promise via `store.dispatch()` so that we know
        // when the data has been fetched
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

And update `app.js`:

``` js
// app.js
import Vue from 'vue'
import App from './App.vue'
import { createRouter } from './router'
import { createStore } from './store'
import { sync } from 'vuex-router-sync'

export function createApp () {
  // create router and store instances
  const router = createRouter()
  const store = createStore()

  // sync so that route state is available as part of the store
  sync(store, router)

  // create the app instance, injecting both the router and the store
  const app = new Vue({
    router,
    store,
    render: h => h(App)
  })

  // expose the app, the router and the store.
  return { app, router, store }
}
```

## Logic Collocation with Components

So, where do we place the code that dispatches the data-fetching actions?

The data we need to fetch is determined by the route visited - which also determines what components are rendered. In fact, the data needed for a given route is also the data needed by the components rendered at that route. So it would be natural to place the data fetching logic inside route components.

We will expose a custom static function `asyncData` on our route components. Note because this function will be called before the components are instantiated, it doesn't have access to `this`. The store and route information needs to be passed in as arguments:

``` html
<!-- Item.vue -->
<template>
  <div>{{ item.title }}</div>
</template>

<script>
export default {
  asyncData ({ store, route }) {
    // return the Promise from the action
    return store.dispatch('fetchItem', route.params.id)
  },

  computed: {
    // display the item from store state.
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
        // serialized and injected into the HTML as `window.__INITIAL_STATE__`.
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

      // we only care about non-previously-rendered components,
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

In a large application, our Vuex store will likely be split into multiple modules. Of course, it is also possible to code-split these modules into corresponding route component chunks. Suppose we have the following store module:

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
