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

We will use the `ssrPrefetch` option in our components. This option is recognized by the server renderer and will be pause the component render until the promise it returns is resolved. Since the component instance is already created at this point, it has access to `this`.

``` html
<!-- Item.vue -->
<template>
  <!-- Loading -->
  <div v-if="loading">Loading...</div>
  <!-- Fetch error -->
  <div v-else-if="error">An error occured</div>
  <!-- Item is undefined -->
  <div v-else-if="!item">Item not found</div>
  <!-- Normal render -->
  <div v-else>{{ item.title }}</div>
</template>

<script>
export default {
  data () {
    return {
      loading: false,
      error: false
    }
  },

  computed: {
    // display the item from store state.
    item () {
      return this.$store.state.items[this.$route.params.id]
    }
  },

  // This will be called by the server renderer automatically
  ssrPrefetch () {
    // return the Promise from the action
    // so that the component waits before rendering
    return this.fetchItem()
  },

  mounted () {
    // This is run only on the client
    // If we didn't already do it on the server
    // we fetch the item (will first show the loading text)
    if (!this.item) {
      this.fetchItem()
    }
  },

  methods: {
    fetchItem () {
      this.loading = true
      this.error = false
      // return the Promise from the action
      return store.dispatch('fetchItem', this.$route.params.id)
        .then(() => {
          // Everything is ok!
          this.loading = false
        })
        .catch(error => {
          // An error occured
          this.loading = false
          this.error = true
          // We should handle the error here
          // (for example, log it into a monitoring service)
        })
    }
  }
}
</script>
```

## Server Data Fetching

In `entry-server.js`, we will set the store state in the render context after the app is finished rendering, thanks to the `context.rendered` hook recognized by the server renderer.

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
  computed: {
    fooCount () {
      return this.$store.state.foo.count
    }
  },

  ssrPrefetch () {
    return this.fooInc()
  },

  mounted () {
    this.fooInc()
  },

  // IMPORTANT: avoid duplicate module registration on the client
  // when the route is visited multiple times.
  destroyed () {
    this.$store.unregisterModule('foo')
  },

  methods: {
    fooInc () {
      this.$store.registerModule('foo', fooStoreModule)
      return this.$store.dispatch('foo/inc')
    }
  }
}
</script>
```

Because the module is now a dependency of the route component, it will be moved into the route component's async chunk by webpack.
