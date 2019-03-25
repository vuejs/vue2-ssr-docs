# Data Pre-Fetching and State

## Data Store

During SSR, we are essentially rendering a "snapshot" of our app. The asynchronous data from our components needs to be available before we mount the client side app - otherwise the client app would render using different state and the hydration would fail.

To address this, the fetched data needs to live outside the view components, in a dedicated data store, or a "state container". On the server, we can pre-fetch and fill data into the store while rendering. In addition, we will serialize and inline the state in the HTML after the app has finished rendering. The client-side store can directly pick up the inlined state before we mount the app.

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
    // IMPORTANT: state must be a function so the module can be
    // instantiated multiple times
    state: () => ({
      items: {}
    }),

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

::: warning
Most of the time, you should wrap `state` in a function, so that it will not leak into the next server-side runs.
[More info](./structure.md#avoid-stateful-singletons)
:::

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

We will use the `serverPrefetch` option (new in 2.6.0+) in our components. This option is recognized by the server renderer and will pause the rendering until the promise it returns is resolved. This allows us to "wait" on async data during the rendering process.

::: tip
You can use `serverPrefetch` in any component, not just the route-level components.
:::

Here is an example `Item.vue` component that is rendered at the `'/item/:id'` route. Since the component instance is already created at this point, it has access to `this`:

``` html
<!-- Item.vue -->
<template>
  <div v-if="item">{{ item.title }}</div>
  <div v-else>...</div>
</template>

<script>
export default {
  computed: {
    // display the item from store state.
    item () {
      return this.$store.state.items[this.$route.params.id]
    }
  },

  // Server-side only
  // This will be called by the server renderer automatically
  serverPrefetch () {
    // return the Promise from the action
    // so that the component waits before rendering
    return this.fetchItem()
  },

  // Client-side only
  mounted () {
    // If we didn't already do it on the server
    // we fetch the item (will first show the loading text)
    if (!this.item) {
      this.fetchItem()
    }
  },

  methods: {
    fetchItem () {
      // return the Promise from the action
      return this.$store.dispatch('fetchItem', this.$route.params.id)
    }
  }
}
</script>
```

::: warning
You should check if the component was server-side rendered in the `mounted` hook to avoid executing the logic twice.
:::

::: tip
You may find the same `fetchItem()` logic repeated multiple times (in `serverPrefetch`, `mounted` and `watch` callbacks) in each component - it is recommended to create your own abstraction (e.g. a mixin or a plugin) to simplify such code.
:::

## Final State Injection

Now we know that the rendering process will wait for data fetching in our components, how do we know when it is "done"? In order to do that, we need to attach a `rendered` callback to the render context (also new in 2.6), which the server renderer will call when the entire rendering process is finished. At this moment, the store should have been filled with the final state. We can then inject it on to the context in that callback:

``` js
// entry-server.js
import { createApp } from './app'

export default context => {
  return new Promise((resolve, reject) => {
    const { app, router, store } = createApp()

    router.push(context.url)

    router.onReady(() => {
      // This `rendered` hook is called when the app has finished rendering
      context.rendered = () => {
        // After the app is rendered, our store is now
        // filled with the state from our components.
        // When we attach the state to the context, and the `template` option
        // is used for the renderer, the state will automatically be
        // serialized and injected into the HTML as `window.__INITIAL_STATE__`.
        context.state = store.state
      }

      resolve(app)
    }, reject)
  })
}
```

When using `template`, `context.state` will automatically be embedded in the final HTML as `window.__INITIAL_STATE__` state. On the client, the store should pick up the state before mounting the application:

``` js
// entry-client.js

import { createApp } from './app'

const { app, store } = createApp()

if (window.__INITIAL_STATE__) {
  // We initialize the store state with the data injected from the server
  store.replaceState(window.__INITIAL_STATE__)
}

app.$mount('#app')
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

We can use `store.registerModule` to lazy-register this module in a route component's `serverPrefetch` hook:

``` html
// inside a route component
<template>
  <div>{{ fooCount }}</div>
</template>

<script>
// import the module here instead of in `store/index.js`
import fooStoreModule from '../store/modules/foo'

export default {
  computed: {
    fooCount () {
      return this.$store.state.foo.count
    }
  },

  // Server-side only
  serverPrefetch () {
    this.registerFoo()
    return this.fooInc()
  },

  // Client-side only
  mounted () {
    // We already incremented 'count' on the server
    // We know by checking if the 'foo' state already exists
    const alreadyIncremented = !!this.$store.state.foo

    // We register the foo module
    this.registerFoo()

    if (!alreadyIncremented) {
      this.fooInc()
    }
  },

  // IMPORTANT: avoid duplicate module registration on the client
  // when the route is visited multiple times.
  destroyed () {
    this.$store.unregisterModule('foo')
  },

  methods: {
    registerFoo () {
      // Preserve the previous state if it was injected from the server
      this.$store.registerModule('foo', fooStoreModule, { preserveState: true })
    },

    fooInc () {
      return this.$store.dispatch('foo/inc')
    }
  }
}
</script>
```

Because the module is now a dependency of the route component, it will be moved into the route component's async chunk by webpack.

::: warning
Don't forget to use the `preserveState: true` option for `registerModule` so we keep the state injected by the server.
:::
