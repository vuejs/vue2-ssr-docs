# ルーティングとコード分割

## `vue-router` によるルーティング

サーバーコードが任意の URL を受け入れる `*` ハンドラを使用していることに気付いたかもしれません。これにより訪れた URL を Vue アプリケーションに渡し、クライアントとサーバーの両方に同一のルーティング設定を再利用することが可能になります！

この目的のために公式の `vue-router` を使用することが推奨されています。まずはルーターを作成するファイルを作成しましょう。 `createApp` に似ていますが、 リクエストごとに新たなルーターインスタンスも必要となるため、 `createRouter` 関数をエクスポートします。

```js
// router.js
import Vue from 'vue'
import Router from 'vue-router'
Vue.use(Router)
export function createRouter () {
  return new Router({
    mode: 'history',
    routes: [
      // ...
    ]
  })
}
```

そして `app.js` を更新します。

```js
// app.js
import Vue from 'vue'
import App from './App.vue'
import { createRouter } from './router'
export function createApp () {
  // create router instance
  const router = createRouter()
  const app new Vue({
    // inject router into root Vue instance
    router,
    render: h => h(App)
  })
  // return both the app and the router
  return { app, router }
}
```

`entry-server.js` にサーバー側のルーティングロジックを実装する必要があります。

```js
// entry-server.js
import { createApp } from './app'
export default context => {
  // since there could potentially be asynchronous route hooks or components,
  // we will be returning a Promise so that the server can wait until
  // everything is ready before rendering.
  return new Promise((resolve, reject) => {
    const { app, router } = createApp()
    // set server-side router's location
    router.push(context.url)
    // wait until router has resolved possible async components and hooks
    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      // no matched routes, reject with 404
      if (!matchedComponents.length) {
        reject({ code: 404 })
      }
      // the Promise should resolve to the app instance so it can be rendered
      resolve(app)
    }, reject)
  })
}
```

サーバーバンドルがすでにビルドされていると仮定すると（再度になりますが、今はビルド設定は無視します）、サーバーでの使用方法は次のようになります。

```js
// server.js
const createApp = require('/path/to/built-server-bundle.js')
server.get('*', (req, res) => {
  const context = { url: req.url }
  createApp(context).then(app => {
    renderer.renderToString(app, (err, html) => {
      if (err) {
        if (err.code === 404) {
          res.status(404).end('Page not found')
        } else {
          res.status(500).end('Internal Server Error')
        }
      } else {
        res.end(html)
      }
    })
  })
})
```

## コード分割

コード分割やアプリケーションの部分的な遅延ローディングは初期レンダリングのためにブラウザがダウンロードする必要のあるアセットの量を減らすのに役立ち、巨大なバンドルを持つアプリケーションの TTI （操作可能になるまでの時間）を大幅に改善します。重要なことは初期画面では"必要なものだけを読み込む"ということです。

Vue は非同期コンポーネントを最重要コンセプトとして提供しており、 [webpack 2の動的インポートをコード分割点として使用することへのサポート](https://webpack.js.org/guides/code-splitting-async/) と組み合わせることも可能です。そのためにすべきことは以下です。

```js
// changing this...
import Foo from './Foo.vue'
// to this:
const Foo = () => import('./Foo.vue')
```

純粋なクライアントサイドの Vue アプリケーションを構築する場合、これはどんなシナリオでも機能するでしょう。ただし、これをサーバーサイドレンダリングで使用する場合はいくつかの制限があります。まず、レンダリングを開始する前にサーバー上のすべての非同期コンポーネントを先に解決する必要があります。そうしなければ、マークアップ内に空のプレースホルダが表示されます。クライアント側では、ハイドレーションを開始する前にこれを行う必要があります。そうしなければ、クライアントはコンテンツの不一致エラーに陥ります。

アプリケーション内の任意の場所で非同期コンポーネントを使用するのは少し難解です（これは将来的に改善される可能性があります）。 ただし、**ルートレベルで行うとシームレスに動作します**（すなわち、ルート設定で非同期コンポーネントを使用する）。ルートを解決する際に、 `vue-router` は一致した非同期コンポーネントを自動的に解決するためです。 必要なことは、サーバーとクライアントの両方で `router.onReady` を使用することです。すでにサーバーのエントリーで行ったので、クライアントのエントリーを更新するだけです。

```js
// entry-client.js
import { createApp } from './app'
const { app, router } = createApp()
router.onReady(() => {
  app.$mount('#app')
})
```

非同期ルートコンポーネントを使用したルート設定の例：

```js
// router.js
import Vue from 'vue'
import Router from 'vue-router'
Vue.use(Router)
export function createRouter () {
  return new Router({
    mode: 'history',
    routes: [
      { path: '/', component: () => import('./components/Home.vue') },
      { path: '/item/:id', component: () => import('./components/Item.vue') }
    ]
  })
}
```
