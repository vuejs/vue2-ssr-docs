# データのプリフェッチと状態

## データストア

SSR をしているとき、基本的にはアプリケーションの"スナップショット"を描画しています。クライアントサイドのアプリケーションがマウントする前に、コンポーネントから非同期データが、利用可能である必要があります。つまり、それ以外の場合、クライアントアプリケーションは異なる状態を使用して描画するため、ハイドレーションは失敗します。

この問題に対応するため、フェッチされたデータはビューコンポーネントの外でも存続している必要があります。つまり専用のデータストア (data store) もしくは "状態コンテナ (state container)" に入っている必要があります。サーバーサイドでは描画する前にデータをプリフェッチしてストアの中に入れることができます。さらに、アプリケーションの描画が終わった後、シリアライズして HTML にインラインで状態を埋め込みます。クライアントサイドのストアは、アプリケーションをマウントする前に、埋め込まれた状態を直接取得できます。

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
    // 重要: 状態はモジュールを複数回インスタンス化できるように、
    // 関数でなければなりません
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
ほとんどの場合、次のサーバサイドの実行においてリークしないよう、 `state` を関数でラップする必要があります。[詳細情報はこちら](./structure.md#avoid-stateful-singletons)
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

コンポーネントでは、 `serverPrefetch` オプション (2.6.0 以降で新規追加)を使用します。このオプションは、サーバレンダラによって認識され、そして それを返す Promise が解決されるまで描画を一時停止します。これにより、描画処理中に非同期データを"待つ"ことができます。

::: tip
ルートレベルのコンポーネントだけでなく、任意のコンポーネントで `serverPrefetch` を使用できます。
:::

これは、`'/item/:id'` ルートで描画される `Item.vue` コンポーネントの例です。コンポーネントインスタンスはこの時点では既に作成されているので、 `this` にアクセスできます:

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

  // サーバサイドのみ
  // これは自動的にサーバレンダラによって呼ばれます
  serverPrefetch () {
    // コンポーネントが描画前に待機するように
    // アクションから Promise を返す
    return this.fetchItem()
  },

  // クライアントサイドのみ
  mounted () {
    // まだサーバ上で描画されていない場合
    // item (最初に読み込み中テキストが表示されます) をフェッチします
    if (!this.item) {
      this.fetchItem()
    }
  },

  methods: {
    fetchItem () {
      // アクションから Promise を返す
      return store.dispatch('fetchItem', this.$route.params.id)
    }
  }
}
</script>
```

::: warning
ロジックが 2 回実行されないようにするために、コンポーネントは `mounted` フックでサーバサイドで描画されているかどうかチェックする必要があります。
:::

::: tip
各コンポーネントで同じ `fetchItem()` ロジックが複数回 (`serverPrefetch`、`mounted`、そして `watch` コールバック)繰り返されているのを見つけるかもしれません。そのようなコードをシンプルにするために、あなた自身で抽象化(例えばミックスインまたはプラグイン)することを推奨します。
:::

## 最終状態注入

これで、描画プロセスがコンポーネント内のデータフェッチを待つことがわかりましたが、それが"完了"したというのをどうやって分かるのでしょうか？それをするために、描画コンテキストに `rendered` コールバックをアタッチする必要があります（これも 2.6 での新機能）。これは描画プロセス全体が終了したときにサーバーレンダラによって呼ばれます。現時点で、ストアは最終的な状態で満たされているはずです。そのコールバック内でコンテキストに状態を注入できます:

```js
// entry-server.js
import { createApp } from './app'

export default context => {
  return new Promise((resolve, reject) => {
    const { app, router, store } = createApp()

    router.push(context.url)

    router.onReady(() => {
      // この `rendered` フックは、アプリケーションの描画が終えたときに呼び出されます
      context.rendered = () => {
        // アプリケーションが描画された後、ストアには、
        // コンポーネントからの状態で満たされています
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
  // サーバから注入されたデータでストアの状態を初期化します
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
  // モジュールを複数回インスタンス化できます
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

  // サーバサイドのみ
  serverPrefetch () {
    this.registerFoo()
    return this.fooInc()
  },

  // クライアントサイドのみ
  mounted () {
    // サーバ上で既に 'count` を増やしています
    // 'foo' 状態が既に存在するかどうかチェックすることで分かります
    const alreadyIncremented = !!this.$store.state.foo
    // foo モジュール を登録する
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
      // サーバから注入された場合は、以前の状態を維持します
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
サーバによって注入された状態を維持するため、`registerModule` に  `preserveState: true` オプションを使用することを忘れないでください。
:::
