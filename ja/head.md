# ヘッドの管理

アセットの挿入と同様に、ヘッド (Head) の管理も同じ考えに追従しています。つまり、コンポーネントのライフサイクルの描画 `context` に動的にデータを付随させ、そして `template` 内にデータを展開 (interpolate) できるという考えです。

> バージョン >=2.3.2 では、`this.$ssrContext` としてコンポーネントにおいて SSR コンテキストに直接アクセスできます。古いバージョンでは、`createApp()` によって手動で SSR コンテキストを渡して注入し、ルート (root) インスタンスの `$options` に公開する必要があります。子は、`this.$root.$options.ssrContext` を介してそれにアクセスすることができます。

タイトルを管理する単純な mixin を書くことができます:

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
// VUE_ENV は webpack.DefinePlugin を使って注入できます
export default process.env.VUE_ENV === 'server'
  ? serverTitleMixin
  : clientTitleMixin
```

このようにすれば、ルート (route) コンポーネントはドキュメントのタイトルを制御するために context を利用することができます。

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

そしてタイトルは `template` 内でバンドルレンダラに渡されます:

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

- XSS 攻撃を防ぐために double-mustache（HTML エスケープした展開）を使うこと。
- 描画中にタイトルをセットするコンポーネントがない場合に備えて、`context` オブジェクトを作成する際にはデフォルトのタイトルをセットするようにすべきです。

---

同様のやり方で、この mixin を包括的にヘッドを管理するユーティリティに容易に拡張できます。
