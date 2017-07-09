# ヘッドの管理

アセットの挿入と同様に、ヘッドの管理も同じ考えに追従しています。つまり、コンポーネントのライフサイクルのレンダリング `context` に動的にデータを付随させ、そして `template` 内にデータを挿入できるという考えです。

そうするためには、ネストしたコンポーネントの内側で SSR コンテキストへアクセスできる必要があります。単純に `context` を `createApp()` へ渡し、これをルートインスタンスの `$options` で公開することができます。

```js
// app.js
export function createApp (ssrContext) {
  // ...
  const app = new Vue({
    router,
    store,
    // this.$root.$options.ssrContext というように、すべての子コンポーネントは this にアクセスできます
    ssrContext,
    render: h => h(App)
  })
  // ...
}
```

これと同様のことが `provide/inject` 経由でも可能ですが、そうすると context が `$root` 上に存在することになるため、インジェクションを解決するコストを避けたほうが良いでしょう。

インジェクトされた context を用いて、タイトルを管理する単純な mixin を書くことができます:

```js
// title-mixin.js
function getTitle (vm) {
  // コンポーネントはシンプルに `title` オプションを提供し、
  // これには文字列または関数を入れることができます
  const { title } = vm.$options
  if (title) {
    return typeof title === 'function'
      ? title.call(vm)
      : title
  }
}
const serverTitleMixin = {
  created () {
    const title = getTitle(this)
    if (title) {
      this.$root.$options.ssrContext.title = title
    }
  }
}
const clientTitleMixin = {
  mounted () {
    const title = getTitle(this)
    if (title) {
      document.title = title
    }
  }
}
// VUE_ENV は webpack.DefinePlugin を使って挿入できます
export default process.env.VUE_ENV === 'server'
  ? serverTitleMixin
  : clientTitleMixin
```

このようにすれば、ルートコンポーネントはドキュメントのタイトルをコントロールするために context を利用することができます。

```js
// Item.vue
export default {
  mixins: [titleMixin],
  title () {
    return this.item.title
  }
  asyncData ({ store, route }) {
    return store.dispatch('fetchItem', route.params.id)
  },
  computed: {
    item () {
      return this.$store.state.items[this.$route.params.id]
    }
  }
}
```

そしてタイトルは `template` 内でバンドルレンダラーに渡されます:

```html
<html>
  <head>
    <title>{{ title }}</title>
  </head>
  <body>
    ...
  </body>
</html>
```

**メモ:**

- XSS 攻撃を防ぐために double-mustache（HTML エスケープした挿入）を使うこと。
- レンダリング中にタイトルをセットするコンポーネントがない場合に備えて、`context` オブジェクトを作成する際にはデフォルトのタイトルをセットするようにすべきです。

---

同様のやり方で、この mixin を包括的にヘッドを管理するユーティリティに容易に拡張できます。
