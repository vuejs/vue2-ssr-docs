# 基本的な使い方

## インストール

```bash
npm install vue vue-server-renderer --save
```

このガイドでは NPM を使って説明していきますが [Yarn](https://yarnpkg.com/en/) でも大丈夫です。

#### 注意

- Node.jsのバージョンは6以上を使用することを推奨します
- `vue-server-renderer` と `vue` のバージョンは一致している必要があります
- `vue-server-renderer` はNode.jsのネイティブモジュールに依存しているため、Node.jsでのみ使用できます。 私たちは、将来的に他のJavaScriptランタイムで実行できるよりシンプルなビルドを提供するかもしれません。

## Vue インスタンスをレンダリング

```js
// ステップ 1: Vue インスタンスを作成
const Vue = require('vue')
const app = new Vue({
  template: `<div>Hello World</div>`
})
// ステップ 2: レンダラを作成
const renderer = require('vue-server-renderer').createRenderer()
// ステップ 3: Vue インスタンスをHTMLに描画
renderer.renderToString(app, (err, html) => {
  if (err) throw err
  console.log(html)
  // => <div data-server-rendered="true">hello world</div>
})
```

## サーバと連携する

Node.js で作られたサーバで使う場合はとても簡単です。例えば [Express](https://expressjs.com/):

```bash
npm install express --save
```

---

```js
const Vue = require('vue')
const server = require('express')()
const renderer = require('vue-server-renderer').createRenderer()
server.get('*', (req, res) => {
  const app = new Vue({
    data: {
      url: req.url
    },
    template: `<div>The visited URL is: {{ url }}</div>`
  })
  renderer.renderToString(app, (err, html) => {
    if (err) {
      res.status(500).end('Internal Server Error')
      return
    }
    res.end(`
      <!DOCTYPE html>
      <html lang="en">
        <head><title>Hello</title></head>
        <body>${html}</body>
      </html>
    `)
  })
})
server.listen(8080)
```

## ページテンプレートを使用する

Vueアプリを描画する際、レンダラはアプリのマークアップのみを生成します。この例では、出力を余計なHTMLページシェルでラップする必要がありました。

これをシンプル化するために、レンダラの作成時にページテンプレートを直接提供することができます。ほとんどの場合、ページテンプレートを単独のファイルに記述します。 例 `index.template.html`:

```html
<!DOCTYPE html>
<html lang="en">
  <head><title>Hello</title></head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

`<!--vue-ssr-outlet-->` コメントに注目してみてください。 -- これはあなたのアプリケーションのマークアップが注入される場所です。

ファイルを読み込みVueレンダラに渡すことができます。

```js
const renderer = createRenderer({
  template: require('fs').readFileSync('./index.template.html', 'utf-8')
})
renderer.renderToString(app, (err, html) => {
  console.log(html) // will be the full page with app content injected.
})
```

### テンプレート展開

テンプレートはシンプルな展開にも対応しています。 次のようなテンプレートであれば：

```html
<html>
  <head>
    <title>{{ title }}</title>
    {{{ meta }}}
  </head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

`renderToString` の第2引数として "描画コンテキストオブジェクト"(render context object) を渡すことで展開データを提供することができます

```js
const context = {
  title: 'hello',
  meta: `
    <meta ...>
    <meta ...>
  `
}
renderer.renderToString(app, context, (err, html) => {
  // ページタイトルは "hello" になり、
  // メタタグが注入されます。
})
```

`context` オブジェクトもVueアプリインスタンスと共有することができ、コンポーネントがテンプレート展開のためにデータを動的に追加することができます。

さらに、テンプレートは次のような高度な機能をサポートしています:

- `*.vue` コンポーネントを使用する際の、重要なCSSの自動注入
- `clientManifest` を使用する際の、アセットリンクとリソースヒントの自動注入
- クライアントサイドハイドレーションのためにVuexの状態を埋め込む際にXSS防止の自動注入

関連する概念については、後でこのガイドで紹介します。
