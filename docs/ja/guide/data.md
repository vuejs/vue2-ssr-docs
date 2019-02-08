# データのプリフェッチと状態

## データストア

SSR をしているとき、基本的にはアプリケーションの"スナップショット"を描画しています。The asynchronous data from our components needs to be available before we mount the client side app - otherwise the client app would render using different state and the hydration would fail.

To address this, the fetched data needs to live outside the view components, in a dedicated data store, or a "state container". On the server, we can pre-fetch and fill data into the store while rendering. In addition, we will serialize and inline the state in the HTML after the app has finished rendering. The client-side store can directly pick up the inlined state before we mount the app.

このような用途として、公式の状態管理ライブラリである [Vuex](https://github.com/vuejs/vuex/) を使っています。では `store.js` ファイルをつくって、そこに id に基づく item を取得するコードを書いてみましょう:

```js
// store.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

// Promise を返すユニバーサルなアプリケーションを想定しています
// また、実装の詳細は割愛します
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
        // store.dispatch() 経由でデータがフェッチされたときにそれを知るために、Promise を返します
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

そして `app.js` を更新します:

```js
// app.js
import Vue from 'vue'
import App from './App.vue'
import { createRouter } from './router'
import { createStore } from './store'
import { sync } from 'vuex-router-sync'

export function createApp () {
  // ルーターとストアのインスタンスを作成します
  const router = createRouter()
  const store = createStore()

  // ルートの状態をストアの一部として利用できるよう同期します
  sync(store, router)

  // アプリケーションのインスタンスを作成し、ルーターとストアの両方を挿入します
  const app = new Vue({
    router,
    store,
    render: h => h(App)
  })

  // アプリケーション、ルーター、ストアを公開します
  return { app, router, store }
}
```

## ロジックとコンポーネントとの結び付き

ではデータをプリフェッチするアクションをディスパッチするコードはどこに置けばよいでしょうか？

フェッチする必要があるデータはアクセスしたルート (route) によって決まります。またそのルートによってどのコンポーネントが描画されるかも決まります。実のところ、与えられたルートに必要とされるデータは、そのルートで描画されるコンポーネントに必要とされるデータでもあるのです。したがって、データをフェッチするロジックはルートコンポーネントの中に置くのが自然でしょう。

We will use the `serverPrefetch` option (new in 2.6.0+) in our components. This option is recognized by the server renderer and will pause the rendering until the promise it returns is resolved. This allows us to "wait" on async data during the rendering process.

::: tip
You can use `serverPrefetch` in any component, not just the route-level components.
:::

Here is an example `Item.vue` component that is rendered at the `'/item/:id'` route. Since the component instance is already created at this point, it has access to `this`:

```html
<!-- Item.vue -->
<template>
  <div v-if="item">{{ item.title }}</div>
  <div v-else>...</div>
</template>

<script>
export default {
  computed: {
    // ストアの状態から item を表示します
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
      return store.dispatch('fetchItem', this.$route.params.id)
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

```js
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
        // 状態を context に付随させ、`template` オプションがレンダラに利用されると、
        // 状態は自動的にシリアライズされ、HTML 内に `window.__INITIAL_STATE__` として埋め込まれます
        context.state = store.state
      }

      resolve(app)
    }, reject)
  })
}
```

`template` を使うと `context.state` は自動的に最終的な HTML に `window.__INITIAL__` という形の状態として埋め込まれます。クライアントサイドでは、アプリケーションがマウントされる前に、ストアがその状態を取得します:

```js
// entry-client.js

const { app, store } = createApp()

if (window.__INITIAL_STATE__) {
  // We initialize the store state with the data injected from the server
  store.replaceState(window.__INITIAL_STATE__)
}

app.$mount('#app')
```

## ストアコードの分割

大規模なアプリケーションでは、Vuex ストアは複数のモジュールに分割される可能性があります。もちろん、これらのモジュールを対応するルートコンポーネントチャンクにコード分割することもできます。次のストアモジュールがあるとします:

```js
// store/modules/foo.js
export default {
  namespaced: true,

  // 重要: 状態は関数でなければならないため、
  // モジュールを複数回インスタン化できます
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

`store.registerModule` を使用して、ルートコンポーネントの `asyncData` フックにこのモジュールを遅延登録することができます:

```html
// ルートコンポーネントの内部
<template>
  <div>{{ fooCount }}</div>
</template>

<script>
// `store/index.js` の代わりにここでモジュールをインポートします
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

  // 重要: ルートが複数回訪問されたときに、
  // クライアントで重複してモジュールが登録されるのを避けて下さい
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

モジュールはルートコンポーネントの依存関係になっているので、webpack によってルートコンポーネントの非同期チャンクに移動されます。

::: warning
Don't forget to use the `preserveState: true` option for `registerModule` so we keep the state injected by the server.
:::
