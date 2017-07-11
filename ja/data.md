# データのプリフェッチと状態

## データストア

SSR をしているとき、基本的にはアプリケーションの"スナップショット"を描画しています、したがって、アプリケーションがいくつかの非同期データに依存している場合においては、**それらのデータを、描画処理を開始する前にプリフェッチして解決する必要があります**。

もうひとつの重要なことは、クライアントサイドでアプリケーションがマウントされる前に、クライアントサイドで同じデータを利用可能である必要があるということです。そうしないと、クライアントサイドが異なる状態 (state) を用いて描画してしまい、ハイドレーションが失敗してしまいます。

この問題に対応するため、フェッチされたデータはビューコンポーネントの外でも存続している必要があります。つまり特定の用途のデータストア (data store) もしくは "状態コンテナ (state container)" に入っている必要があります。サーバーサイドでは描画する前にデータをプリフェッチしてストアの中に入れることができます。さらにシリアライズして HTML に状態を埋め込みます。クライアントサイドのストアは、アプリケーションをマウントする前に、埋め込まれた状態を直接取得できます。

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
    state: {
      items: {}
    },
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

ルートコンポーネントではカスタム静的関数 `asyncData` が利用可能です。この関数はそのルートコンポーネントがインスタンス化される前に呼び出されるため `this` にアクセスできないことを覚えておいてください。ストアとルートの情報は引数として渡される必要があります:

```html
<!-- Item.vue -->
<template>
  <div data-segment-id="282437">{{ item.title }}</div>
</template>
<script>
export default {
  asyncData ({ store, route }) {
    // アクションから Promise が返されます
    return store.dispatch('fetchItem', route.params.id)
  },
  computed: {
    // ストアの状態から item を表示します
    items () {
      return this.$store.state.items[this.$route.params.id]
    }
  }
}
</script>
```

## サーバーサイドのデータ取得

`entry-server.js` において `router.getMatchedComponents()` を使ってルートにマ一致したコンポーネントを取得できます。そしてコンポーネントが `asyncData` を利用可能にしていればそれを呼び出すことができます。そして描画のコンテキストに解決した状態を付属させる必要があります。

```js
// entry-server.js
import { createApp } from './app'
export default context => {
  return new Promise((resolve, reject) => {
    const { app, router, store } = createApp()
    router.push(context.url)
    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      if (!matchedComponents.length) {
        reject({ code: 404 })
      }
      // 一致したルートコンポーネントすべての asyncData() を呼び出します 
      Promise.all(matchedComponents.map(Component => {
        if (Component.asyncData) {
          return Component.asyncData({
            store,
            route: router.currentRoute
          })
        }
      })).then(() => {
        // すべてのプリフェッチのフックが解決されると、ストアには、
        // アプリケーションを描画するために必要とされる状態が入っています。
        // 状態を context に付随させ、`template` オプションがレンダラに利用されると、
        // 状態は自動的にシリアライズされ、HTML 内に `window.__INITIAL_STATE__` として埋め込まれます
        context.state = store.state
        resolve(app)
      }).catch(reject)
    }, reject)
  })
}
```

`template` を使うと `context.state` は自動的に最終的な HTML に `window.__INITIAL__` という形の状態として埋め込まれます。クライアントサイドでは、アプリケーションがマウントされる前に、ストアがその状態を取得します:

```js
// entry-client.js
const { app, router, store } = createApp()
if (window.__INITIAL_STATE__) {
  store.replaceState(window.__INITIAL_STATE__)
}
```

## クライアントサイドのデータ取得

クライアントサイドではデータ取得について 2つの異なるアプローチがあります:

1. **ルートのナビゲーションの前にデータを解決する:**

この方法では、アプリケーションは、遷移先のビューが必要とするデータが解決されるまで、現在のビューを保ちます。良い点は遷移先のビューがデータの準備が整い次第、フルの内容を直接描画できることです。しかしながら、データの取得に時間がかかるときは、ユーザーは現在のビューで「固まってしまった」と感じてしまうでしょう。そのため、この方法を用いるときにはローディングインジケーターを表示させることが推奨されます。

この方法は、クライアントサイドで一致するコンポーネントをチェックし、グローバルなルートのフック内で `asyncData` 関数を実行することにより実装できます。重要なことは、このフックは初期ルートが ready になった後に登録するということです。そうすれば、サーバーサイドで取得したデータをもう一度無駄に取得せずに済みます。

```js
  // entry-client.js
  // ...関係のないコードは除外します
  router.onReady(() => {
    // asyncData を扱うためにルーターのフックを追加します。これは初期ルートが解決された後に実行します
    // そうすれば（訳注: サーバーサイドで取得したために）既に持っているデータを冗長に取得しなくて済みます
    // すべての非同期なコンポーネントが解決されるように router.beforeResolve() を使います
    router.beforeResolve((to, from, next) => {
      const matched = router.getMatchedComponents(to)
      const prevMatched = router.getMatchedComponents(from)
      // まだ描画されていないコンポーネントにのみ関心を払うため、
      // 2つの一致したリストに差分が表れるまで、コンポーネントを比較します
      let diffed = false
      const activated = matched.filter((c, i) => {
        return diffed || (diffed = (prevMatched[i] !== c))
      })
      if (!activated.length) {
        return next()
      }
      // もしローディングインジケーターがあるならば、
      // この箇所がローディングインジケーターを発火させるべき箇所です
      Promise.all(activated.map(c => {
        if (c.asyncData) {
          return c.asyncData({ store, route: to })
        }
      })).then(() => {
        // ローディングインジケーターを停止させます
        next()
      }).catch(next)
    })
    app.$mount('#app')
  })
```

1. **一致するビューが描画された後にデータを取得する:**

この方法ではビューコンポーネントの `beforeMount` 関数内にクライアントサイドでデータを取得するロジックを置きます。こうすればルートのナビゲーションが発火したらすぐにビューを切り替えられます。そうすればアプリケーションはよりレスポンスが良いと感じられるでしょう。しかしながら、遷移先のビューは描画した時点では完全なデータを持っていません。したがって、この方法を使うコンポーネントの各々がローディング中か否かの状態を持つ必要があります。

この方法はクライアントサイド限定のグローバルな mixin で実装できます:

```js
  Vue.mixin({
    beforeMount () {
      const { asyncData } = this.$options
      if (asyncData) {
        // データが準備できた後に、コンポーネント内で `this.dataPromise.then(...)` して
        // 他のタスクを実行できるようにするため、Promise にフェッチ処理を割り当てます
        this.dataPromise = asyncData({
          store: this.$store,
          route: this.$route
        })
      }
    }
  })
```

これら 2つの方法のどちらを選ぶかは、究極的には異なる UX のどちらを選ぶかの判断であり、構築しようとしているアプリケーションの実際のシナリオに基づいて選択されるべきものです。しかし、どちらの方法を選択したかにかかわらず、ルートコンポーネントが再利用されたとき（つまりルートは同じだがパラメーターやクエリが変わったとき。例えば `user/1` から `user/2`) へ変わったとき）には `asyncData` 関数は呼び出されるようにすべきです。これはクライアントサイド限定のグローバルな mixin で処理できます:

```js
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
  asyncData ({ store }) {
    store.registerModule('foo', fooStoreModule)
    return store.dispatch('foo/inc')
  },
  // 重要: ルートが複数回訪問されたときに、
  // クライアントで重複してモジュールが登録されるのを避けて下さい
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

モジュールはルートコンポーネントの依存関係になっているので、webpack によってルートコンポーネントの非同期チャンクに移動されます。

---

ふぅ、コードが長いですね。これはどうしてかというと、ユニバーサルなデータ取得は、大抵の場合、サーバーサイドで描画するアプリケーションの最も複雑な問題であり、また、今後、スムーズに開発を進めていくための下準備をしているためです。一旦ひな形が準備できてしまえば、あとは、それぞれのコンポーネントを記述していく作業は、実際のところ実に楽しいものになるはずです。
