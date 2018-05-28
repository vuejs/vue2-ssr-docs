# ビルド設定

クライアントサイドで完結するプロジェクトの webpack 設定は既に知っての通りでしょう。SSR プロジェクトにおいても大枠は似たようなものですが、設定ファイルを 3 つのファイル(*base*、*client*、*server*)に分けることを提案しています。base 設定は出力パス、エイリアス、ローダーのような、client と server 両方の環境に共有される設定を含み、server 設定と client 設定は単純に、[webpack-merge](https://github.com/survivejs/webpack-merge) を使って、base 設定を拡張することができるものです。

## server 設定

server 設定は `createBundleRenderer` に渡されるサーババンドルを生成するために作られるもので、次のようになります:

```js
const merge = require('webpack-merge')
const nodeExternals = require('webpack-node-externals')
const baseConfig = require('./webpack.base.config.js')
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')

module.exports = merge(baseConfig, {
  // アプリケーションサーバのエントリファイルへのエントリポイント
  entry: '/path/to/entry-server.js',

  // これにより、webpack は Node に適した方法で動的なインポートを処理でき、
  // Vue コンポーネントをコンパイルするときにサーバー指向のコードを出力するよう
  // `vue-loader`に指示する
  target: 'node',

  // バンドルレンダラーのソースマップのサポート
  devtool: 'source-map',

  // Node スタイルのエクスポートを使用するようにサーバーバンドルに指示する
  output: {
    libraryTarget: 'commonjs2'
  },

  // https://webpack.js.org/configuration/externals/#function
  // https://github.com/liady/webpack-node-externals
  // アプリケーションの依存関係を外部化する
  // これにより、サーバーのビルドが大幅に高速化され、より小さなバンドルファイルが生成される
  externals: nodeExternals({
    // webpack で処理する必要がある依存関係を外部化しない
    // ここに例として、生の * .vue　ファイルのようなファイルタイプを追加できる
    // `グローバル` (例 ポリフィル) を変更する deps もホワイトリストに登録する必要がある
    whitelist: /\.css$/
  }),
  
  // これはサーバービルドの出力全体を
  // 1つの JSON ファイルに変換するプラグイン。
  // デフォルトのファイル名は `vue-ssr-server-bundle.json`
  plugins: [
    new VueSSRServerPlugin()
  ]
})
```

`vue-ssr-server-bundle.json` が生成された後、ファイルパスを単純に `createBundleRenderer` に渡します:

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer('/path/to/vue-ssr-server-bundle.json', {
  // 他の描画オプション...
})
```

別の方法として、バンドルをオブジェクトとして `createBundleRenderer` に渡すことも可能です。これは開発中のホットリロードに対して便利で、[参考のセットアップとして](https://github.com/vuejs/vue-hackernews-2.0/blob/master/build/setup-dev-server.js) HackerNews の設定を見てみてください。

### externals の注意

CSS ファイルを `externals` オプションにホワイトリスト登録していることに注目してください。その理由は、依存関係からインポートされる CSS は webpack によって処理されないといけないからです。 もし同じように webpack に依存する他のタイプのファイルをインポートしているなら、 (例： `*.vue`, `*.sass`)、 それらも同じようにホワイトリストに加えなければいけません。

`runInNewContext: 'once'` または `runInNewContext: true` を使用する場合、例えば `babel-polyfill` のような `global` のように変更するポリフィルがホワイトリスト登録するために必要です。これは、新しいコンテキストモードを使用するときに、サーババンドルの内部コードは独自の `global` オブジェクトを持っているからです。Node 7.6 以降を使っていればサーバに `babel-polyfill` はあまり必要ないので、単純にクライアントエントリーにインポートする方が簡単です。

## client 設定

client 設定は base 設定とほぼ同じままです。言うまでもなく、クライアント側のエントリファイルに `entry` を示す必要があります。またそれとは別に、`CommonsChunkPlugin` 使っていたら、それが client 設定だけで使われていることを確認しておかないといけません。なぜなら、サーババンドルは単一のエントリーチャンクを要求するからです。

### `clientManifest` を生成する

> version 2.3.0 以降必須

サーババンドルに加えて、クライアントビルドマニフェストを作成することもできます。レンダラーは、クライアントマニフェストとサーババンドルでサーバ側*と*クライアント側の両方のビルド情報を持つことになり、 描画された HTML に [preload / prefetch directives](https://css-tricks.com/prefetching-preloading-prebrowsing/) や CSS の link や script タグを自動的に挿入することができます。

これには2重の恩恵があります:

1. 生成されたファイル名にハッシュがある時に、正しい URL を注入する `html-webpack-plugin` の代替になります。

2. webpack のオンデマンドコード分割機能(code spliting)を利用するバンドルを描画する時に、最適なチャンクが preloaded / prefetched されるのを保証でき、かつ、クライアントに対するウォーターフォールリクエストを避けるために、必要な非同期チャンクに `<script></script>` タグを挿入することができます。そのようにして TTI (time-to-interactive) が改善します。

クライアントマニフェストを利用するためには、client 設定はこのようになります:

```js
const webpack = require('webpack')
const merge = require('webpack-merge')
const baseConfig = require('./webpack.base.config.js')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')

module.exports = merge(baseConfig, {
  entry: '/path/to/entry-client.js',
  plugins: [
    // 重要: webpack ランタイムを先頭のチャンクに分割して、
    // 直後に非同期チャンクを挿入できるようにする
    // これにより、アプリ/ベンダーコードのキャッシングも改善される
    new webpack.optimize.CommonsChunkPlugin({
      name: "manifest",
      minChunks: Infinity
    }),
    // このプラグインは、出力ディレクトリに 
    // `vue-ssr-client-manifest.json` を生成する
    new VueSSRClientPlugin()
  ]
})
```

これで、作成されたクライアントマニフェストをページテンプレートと一緒に利用できるようになります。

```js
const { createBundleRenderer } = require('vue-server-renderer')
const template = require('fs').readFileSync('/path/to/template.html', 'utf-8')
const serverBundle = require('/path/to/vue-ssr-server-bundle.json')
const clientManifest = require('/path/to/vue-ssr-client-manifest.json')
const renderer = createBundleRenderer(serverBundle, {
  template,
  clientManifest
})
```

この設定で、コード分割されたビルドのためにサーバ側で描画される HTML はこのようになります(すべて自動でインジェクトされます):

```html
<html>
  <head>
    <!-- この描画に対して使用されるチェックは、プリロードです -->
    <link rel="preload" href="/manifest.js" as="script">
    <link rel="preload" href="/main.js" as="script">
    <link rel="preload" href="/0.js" as="script">
    <!-- 使用されない非同期チャンクはプリフェッチされます (低優先度) -->
    <link rel="prefetch" href="/1.js" as="script">
  </head>
  <body>
    <!-- アプリケーションコンテンツ -->
    <div data-server-rendered="true"><div>async</div></div>
    <!-- マニフェストチャンクは最初にすべき -->
    <script src="/manifest.js"></script>
    <!-- 非同期チャンクはメインチャンク前に注入される -->
    <script src="/0.js"></script>
    <script src="/main.js"></script>
  </body>
</html>
```

### 手動でのアセットインジェクション

デフォルト設定で、アセットインジェクションはあなたが作成した `template` 描画オプションで自動に行われます。しかし、アセットがどのようにテンプレートにインジェクトされるかをより細かくコントロールしたい時もあるでしょうし、あるいはテンプレートを使わない時もあるかもしれません。そのような場合にはレンダラを作る時に `inject: false` を渡せば、手動でアセットインジェクションを行うことができます。

渡した `context` オブジェクトは `renderToString` コールバックで、次のメソッドを持ちます:

- `context.renderStyles()`

  これは、描画中に使われた `*.vue` コンポーネントから集めた全てのクリティカル CSS を含んだ `<style></style>` タグを返します。詳細は [CSS の管理](./css.md)の章を見てください。

  `clientManifest` が提供されたら、返ってきた文字列は webpack が放出した CSS ファイルの `<link rel="stylesheet">` タグも含みます。(例: `extract-text-webpack-plugin` から抽出された CSS や、`file-loader` でインポートされた CSS)

- `context.renderState(options?: Object)`

  このメソッドは `context.state` をシリアライズし、`window.__INITIAL_STATE__` ステートとして埋め込まれたインラインスクリプトを返します。

  context のステートキーと window のステートキーはどちらとも、オプションオブジェクトとして渡すことでカスタマイズできます。

```js
  context.renderState({
    contextKey: 'myCustomState',
    windowKey: '__MY_STATE__'
  })

  // -> <script>window.__MY_STATE__={...}</script>
```

- `context.renderScripts()`

    - 必須 `clientManifest`

このメソッドはクライアントアプリケーションを起動するのに必要な  `<script></script>` タグを返します。コードの中に非同期コード分割を使っている時、このメソッドは賢くも、インクルードされるべき正しい非同期チャンクを推論します。

- `context.renderResourceHints()`

    - 必須 `clientManifest`

このメソッドは、現在描画されているページに必要な `<link rel="preload/prefetch">` リソースヒントを返します。 デフォルト設定ではこのようになります:

- ページに必要な JavaScript や CSS ファイルをプリロードする
- あとで必要な非同期 JavaScript チャンクをプリフェッチする

  ファイルのプリロードは [`shouldPreload`](../api/#shouldpreload) オプションによってさらにカスタマイズが可能です。

- `context.getPreloadFiles()`

  - 必須 `clientManifest`

  このメソッドは文字列を返さない代わりに、プリロードされるべきアセットを表すファイルオブジェクトの配列を返します。これは HTTP/2 サーバプッシュをプログラムで行うときに使えるでしょう。

`createBundleRenderer` に渡された `template` は `context` を使って挿入されるので、これらのメソッドをテンプレート内で(`inject: false`で)使用することができます:

```html
<html>
  <head>
    <!-- HTML でエスケープされない展開 (interpolation) のための mustache を使う -->
    {{{ renderResourceHints() }}}
    {{{ renderStyles() }}}
  </head>
  <body>
    <!--vue-ssr-outlet-->
    {{{ renderState() }}}
    {{{ renderScripts() }}}
  </body>
</html>
```

もし `template` を全く使っていないのなら、自分自身で文字列を結合することができます。
