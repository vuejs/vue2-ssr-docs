# ビルド設定

クライアントサイドで完結するプロジェクトのwebpack設定は既に知っての通りでしょう。 SSRプロジェクトにおいても大枠は似たようなものですが、設定ファイルを3つのファイル(*base*,*client*,*server*)に分けることを提案しています。base設定は出力パス、エイリアス、ローダーのような、clientとserver両方の環境に共有される設定を含み、server設定とclient設定は単純に、 [webpack-merge](https://github.com/survivejs/webpack-merge)を使って、base設定を拡張することができるものです。

## server設定

server設定は`createBundleRenderer`に渡されるサーババンドルを生成するために作られるもので、次のようになります:

```js
const merge = require('webpack-merge')
const nodeExternals = require('webpack-node-externals')
const baseConfig = require('./webpack.base.config.js')
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')
module.exports = merge(baseConfig, {
  // Point entry to your app's server entry file
  entry: '/path/to/entry-server.js',
  // This allows webpack to handle dynamic imports in a Node-appropriate
  // fashion, and also tells `vue-loader` to emit server-oriented code when
  // compiling Vue components.
  target: 'node',
  // For bundle renderer source map support
  devtool: 'source-map',
  // This tells the server bundle to use Node-style exports
  output: {
    libraryTarget: 'commonjs2'
  },
  // https://webpack.js.org/configuration/externals/#function
  // https://github.com/liady/webpack-node-externals
  // Externalize app dependencies. This makes the server build much faster
  // and generates a smaller bundle file.
  externals: nodeExternals({
    // do not externalize dependencies that need to be processed by webpack.
    // you can add more file types here e.g. raw *.vue files
    // you should also whitelist deps that modifies `global` (e.g. polyfills)
    whitelist: /\.css$/
  }),
  // This is the plugin that turns the entire output of the server build
  // into a single JSON file. The default file name will be
  // `vue-ssr-server-bundle.json`
  plugins: [
    new VueSSRServerPlugin()
  ]
})
```

 `vue-ssr-server-bundle.json`が生成されたら、ファイルパスを `createBundleRenderer`に渡します:

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer('/path/to/vue-ssr-server-bundle.json', {
  // ...other renderer options
})
```

別の方法として、 バンドルをオブジェクトとして`createBundleRenderer`に渡すことも可能で、これは開発中のホットリロードに対して便利です。 参考として [HackerNewsの設定](https://github.com/vuejs/vue-hackernews-2.0/blob/master/build/setup-dev-server.js) を見てみてください。

### externalsの注意

 CSSファイルを`externals`オプションにホワイトリスト登録していることに注目してください。その理由は、依存関係からインポートされるCSS はwebpackによって処理されないといけないからです。 もし同じようにwebpackに依存する他のタイプのファイルをインポートしているなら、 (例： `*.vue`, `*.sass`)、 それらも同じようにホワイトリストに加えなければいけません。

ホワイトリスト登録する他のタイプのモジュールは、例えば `babel-polyfill`のような`global`を修正するポリフィルです。なぜなら、サーババンドルの中のコードは独自の ** `global` **オブジェクトを持っているからです。Node7.6以降を使っていればサーバに`babel-polyfill`はあまり必要ないので、単純にクライアントエントリーにインポートする方が簡単です。

## client設定

client設定はbase設定とほぼ同じままです。言うまでもなく、クライアント側のエントリーファイルに`entry`を示す必要があります。またそれとは別に、もし`CommonsChunkPlugin`使っていたら、それがclient設定だけで使われていることを確認しておかないといけません。なぜなら、サーババンドルは単一のエントリーチャンクを要求するからです。

### `clientManifest` の作成

> 必須 version 2.3.0以降

サーババンドルに加えて、クライアントビルドマニフェストを作成することもできます。レンダラーは、クライアントマニフェストとサーババンドルでサーバ側*と*クライアント側の両方のビルド情報を持つことになり、 レンダリングされたHTMLに[preload / prefetch directives](https://css-tricks.com/prefetching-preloading-prebrowsing/)やCSSのlinkやscriptタグを自動的に挿入することができます。

これには2重の恩恵があります:

1. 生成されたファイル名にハッシュがある時に、正しいURLを注入する`html-webpack-plugin` の代替になります。
2. webpackのオンデマンドコード分割機能(code spliting)を利用するバンドルをレンダリングする時に、最適なチャンクがpreloaded / prefetchedされるのを保証でき、かつ、クライアントに対するウォーターフォールリクエストを避けるために、必要な非同期チャンクに`<script></script>`タグを挿入することができます。そのようにしてTTI (time-to-interactive)が改善します。

クライアントマニフェストを利用するためには、client設定はこのようになります：

```js
const webpack = require('webpack')
const merge = require('webpack-merge')
const baseConfig = require('./webpack.base.config.js')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')
module.exports = merge(baseConfig, {
  entry: '/path/to/entry-client.js',
  plugins: [
    // Important: this splits the webpack runtime into a leading chunk
    // so that async chunks can be injected right after it.
    // this also enables better caching for your app/vendor code.
    new webpack.optimize.CommonsChunkPlugin({
      name: "manifest",
      minChunks: Infinity
    }),
    // This plugins generates `vue-ssr-client-manifest.json` in the
    // output directory.
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

この設定で、コード分割されたビルドのためにサーバ側でレンダリングされるHTMLはこのようになります(すべて自動でインジェクトされます)。

```html

  
    <!-- chunks used for this render will be preloaded -->
    <link rel="preload" href="/manifest.js" as="script">
    <link rel="preload" href="/main.js" as="script">
    <link rel="preload" href="/0.js" as="script">
    <!-- unused async chunks will be prefetched (lower priority) -->
    <link rel="prefetch" href="/1.js" as="script">
  
  
    <!-- app content -->
    <div data-server-rendered="true"><div data-segment-id="282354">async</div></div>
    <!-- manifest chunk should be first -->
    <script src="/manifest.js"></script>
    <!-- async chunks injected before main chunk -->
    <script src="/0.js"></script>
    <script src="/main.js"></script>
  
`
```

### 手動でのアセットインジェクション

デフォルト設定で、アセットインジェクションはあなたが作成した`template`レンダリングオプションで自動に行われます。 しかし、アセットがどのようにテンプレートにインジェクトされるかをより細かくコントロールしたい時もあるでしょうし、あるいはテンプレートを使わない時もあるかもしれません。そのような場合にはレンダラーを作る時に`inject: false`を渡せば、手動でアセットインジェクションを行うことができます。

渡した`context`オブジェクトは`renderToString`コールバックで、  次のメソッドを持ちます:

- `context.renderStyles()`

これは、レンダリング中に使われた`*.vue`コンポーネントから集めた全てのクリティカルCSSを含んだ`<style></style>` タグを返します。詳細は [CSS Management](./css.md)の章を見てください。

もし `clientManifest` が提供されたら、返ってきたストリングはwebpackが放出したCSSファイルの`<link rel="stylesheet">`タグも含みます。 (例 : `extract-text-webpack-plugin`から抽出されたCSSや、`file-loader`でインポートされたCSS)

- `context.renderState(options?: Object)`

 このメソッドは `context.state` をシリアライズし、 `window.__INITIAL_STATE__`ステートとして埋め込まれたインラインスクリプトを返します。

contextのステートキーとwindowのステートキーはどちらとも、オプションオブジェクトとして渡すことでカスタマイズできます。

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

このメソッドは、現在レンダリングされているページに必要な`<link rel="preload/prefetch">` リソースヒントを返します。 デフォルト設定ではこのようになります:

- ページに必要なJavaScriptやCSSファイルをプリロードする
- あとで必要な非同期JavaScriptチャンクをプリフェッチする

  ファイルのプリロードは[`shouldPreload`](./api.md#shouldpreload) オプションによってさらにカスタマイズが可能です。

- `context.getPreloadFiles()`
    - 必須 `clientManifest`

このメソッドは stringを返さない代わりに、プリロードされるべきアセットを表すファイルオブジェクトの配列を返します。これは HTTP/2サーバプッシュをプログラムで行うときに使えるでしょう。

`createBundleRenderer`に渡された`template`は`context`を使って挿入されるので、これらのメソッドをテンプレート内で(`inject: false`で)使用することができます:

```html

  
    <!-- use triple mustache for non-HTML-escaped interpolation -->
    {{{ renderResourceHints() }}}
    {{{ renderStyles() }}}
  
  
    <!--vue-ssr-outlet-->
    {{{ renderState() }}}
    {{{ renderScripts() }}}
  

```

もし `template` を全く使っていないのなら、自分自身でstringを結合することができます。
