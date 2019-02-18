# 数据预取和状态

## 数据预取存储容器 (Data Store)

在服务器端渲染(SSR)期间，我们本质上是在渲染我们应用程序的"快照"。在我们装载客户端应用之前，我们组件中所应用的异步数据需要处于可用状态 - 否则客户端应用会使用不同的状态进行渲染，并导致激活失败。

为了解决这个问题，获取的数据需要位于视图组件之外，即放置在专门的数据预取存储容器(data store)或"状态容器(state container)）"中。首先，在服务器端，我们可以在渲染时预取数据，并将数据填充到 store 中。此外
，我们将在应用渲染完成后，在 HTML 中序列化(serialize)和内联预置(inline)状态。这样，在挂载(mount)到客户端应用程序之前，可以直接从 store 获取到内联预置(inline)状态。

为此，我们将使用官方状态管理库 [Vuex](https://github.com/vuejs/vuex/)。我们先创建一个 `store.js` 文件，里面会模拟一些根据 id 获取 item 的逻辑：

``` js
// store.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

// 假定我们有一个可以返回 Promise 的
// 通用 API（请忽略此 API 具体实现细节）
import { fetchItem } from './api'

export function createStore () {
  // 重要: state必须是一个函数，
  // 这样模块才可以多次实例化
  return new Vuex.Store({
    state: () => ({
      items: {}
    }),
    actions: {
      fetchItem ({ commit }, id) {
        // `store.dispatch()` 会返回 Promise，
        // 以便我们能够知道数据在何时更新
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
大多数情况下，你都应该将 `state` 包装成一个函数，这样它的状态便不会泄露到下一个服务端执行。
[更多信息](./structure.md#avoid-stateful-singletons)
:::


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

  // 同步路由状态(route state)到 store
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

## 带有逻辑配置的组件 (Logic Collocation with Components)

那么，我们在哪里放置「dispatch 数据预取 action」的代码？

我们需要通过访问路由，来决定获取哪部分数据 - 这也决定了哪些组件需要渲染。事实上，给定路由所需的数据，也是在该路由上渲染组件时所需的数据。所以在路由组件中放置数据预取逻辑，是很自然的事情。

我们将在组件中使用 `serverPrefetch` 选项（2.6.0+中新增）。这个选项会被服务端渲染所识别，并且暂停渲染过程，直到它所返回的promise被解决。这允许我们在渲染过程中“等待”异步数据。

::: tip 提示
你可以在任何组件中使用`serverPrefetch`，不仅仅局限在路由级别组件上
:::

这里有一个`Item.vue`组件示例，它在路由匹配`'/item/:id'`时进行渲染。由于此时组件实例已经被创建，所以可以通过`this`进行访问：

``` html
<!-- Item.vue -->
<template>
  <div v-if="item">{{ item.title }}</div>
  <div v-else>...</div>
</template>

<script>
export default {
  computed: {
    // 从 store 的 state 对象中的获取 item。
    item () {
      return this.$store.state.items[this.$route.params.id]
    }
  },
  // 仅限服务端
  // 它将在服务端渲染时自动被调用
  serverPrefetch () {
    // 在执行后返回Promise
    // 以便组件在渲染执行之前等待
    return this.fetchItem()
  },
  // 仅限客户端
  mounted () {
    // 如果我们确定不在服务端执行
    // 那么在这里获取item（首先展示加载文字）
    if (!this.item) {
      this.fetchItem()
    }
  },
  methods: {
    fetchItem () {
      // 在执行后返回Promise
      return store.dispatch('fetchItem', this.$route.params.id)
    }
  }
}
</script>
```

::: warning 警告
为了避免逻辑执行两次，你需要检查组件在`mounted`钩子触发时是否已完成服务端渲染
:::

::: tip 提示
你会注意到，`fetchItem()`逻辑在每一个组件中被重复调用了多次 (在 `serverPrefetch`, `mounted` 和 `watch` 回调) - 建议您自己进行抽象（例如使用混合或插件机制）以简化此类代码
:::

## 最终状态注入

现在我们知道了组件中，渲染过程会等待数据获取完成后继续进行，那么我们如何知道何时才是“完成”状态？为了实现此逻辑，我们需要在渲染上下文中附加一个`rendered`回调函数（同样是2.6新增），服务端渲染会在渲染过程完成时调用此回调。在这个时刻，全局store中保存的是应用的最终状态。此时我们可以在回调中将它注入到上下文当中：

``` js
// entry-server.js
import { createApp } from './app'

export default context => {
  return new Promise((resolve, reject) => {
    const { app, router, store } = createApp()

    router.push(context.url)

    router.onReady(() => {
      // `rendered`钩子函数会在应用完成渲染时被调用
      context.rendered = () => {
        // 在应用渲染完成后，此时我们的store中
        // 填满了组件中所使用的方案状态。
        // 当我们将状态附加到上下文中，并且`template`选项
        // 被渲染器所使用时，状态会被自动序列化并以`window.__INITIAL_STATE__`
        // 的形式注入到HTML中
        context.state = store.state
      }

      resolve(app)
    }, reject)
  })
}
```

当使用 `template` 时，`context.state` 将作为 `window.__INITIAL_STATE__` 状态，自动嵌入到最终的 HTML 中。而在客户端，在挂载到应用程序之前，store 就应该获取到状态：

``` js
// entry-client.js

const { app, store } = createApp()

if (window.__INITIAL_STATE__) {
  // 使用服务端注入的数据进行store的初始化工作
  store.replaceState(window.__INITIAL_STATE__)
}
app.$mount('#app')
```
## Store 代码拆分 (Store Code Splitting)

在大型应用程序中，我们的 Vuex store 可能会分为多个模块。当然，也可以将这些模块代码，分割到相应的路由组件 chunk 中。假设我们有以下 store 模块：

``` js
// store/modules/foo.js
export default {
  namespaced: true,

  // 重要信息：state 必须是一个函数，
  // 因此可以创建多个实例化该模块
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

我们可以在路由组件的 `asyncData` 钩子函数中，使用 `store.registerModule` 惰性注册(lazy-register)这个模块：

``` html
// 在路由组件内
<template>
  <div>{{ fooCount }}</div>
</template>

<script>
// 在这里导入模块，而不是在 `store/index.js` 中
import fooStoreModule from '../store/modules/foo'

export default {
  computed: {
    fooCount () {
      return this.$store.state.foo.count
    }
  },
  // 仅限服务端
  serverPrefetch () {
    this.registerFoo()
    return this.fooInc()
  },
  // 仅限客户端
  mounted () {
    // 我们已经在服务端增加了'count'
    // 我们通过'foo'状态是否存在来进行检查
    const alreadyIncremented = !!this.$store.state.foo
    // 我们注册foo模块
    this.registerFoo()
    if (!alreadyIncremented) {
      this.fooInc()
    }
  },

  // 重要信息：当多次访问路由时，
  // 避免在客户端重复注册模块。
  destroyed () {
    this.$store.unregisterModule('foo')
  },

  methods: {
    registerFoo () {
      // 如果状态在服务端已被注入，则保留之前的状态
      this.$store.registerModule('foo', fooStoreModule, { preserveState: true })
    },
    fooInc () {
      return this.$store.dispatch('foo/inc')
    }
  }
}
</script>
```

由于模块现在是路由组件的依赖，所以它将被 webpack 移动到路由组件的异步 chunk 中。

::: warning 警告
不要忘记在`registerModule`时使用`preserveState: true`选项，这样我们就可以保持服务器端注入的状态了
:::