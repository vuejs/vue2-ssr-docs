# API リファレンス

## `createRenderer([options])`

任意の引数  [options](#renderer-options) を用いて [`Renderer`](#class-renderer) インスタンスを生成します。

```js
const { createRenderer } = require('vue-server-renderer')
const renderer = createRenderer({ ... })
```

## `createBundleRenderer(bundle[, options])`

サーババンドルと任意の引数  [options](#renderer-options) を用いて [`BundleRenderer`](#class-bundlerenderer) インスタンスを生成します。

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer(serverBundle, { ... })
```

引数 `serverBundle` には次のいずれか１つを指定できます。

- 生成されたバンドルファイル (`.js` または `.json`) への絶対パス。 ファイルパスは、 `/` で始めなければいけません。
- webpack と `vue-server-renderer/server-plugin` によって生成されたバンドルオブジェクト。
- JavaScript コードの文字列 (非推奨)。

より詳しい情報は、 [サーババンドルの紹介](./bundle-renderer.md) と [ビルド設定](./build-config.md) の項目を参照してください。

## `クラス: Renderer`

- #### `renderer.renderToString(vm[, context], callback)`

Vue インスタンスを文字列として描画します。context オブジェクトの指定は、任意です。callback は、第１引数にエラー内容、 第２引数に描画された文字列を受け取る、典型的な Node.js のコーディングスタイルである関数を指定します。

- #### `renderer.renderToStream(vm[, context])`

Vue インスタンスを Node.js のストリームへ描画します。context オブジェクトの指定は任意です。より詳しい情報は、[ストリーミング](./streaming.md) の項目を参照してください。

## `クラス: BundleRenderer`

- #### `bundleRenderer.renderToString([context, ]callback)`

サーババンドルを文字列として描画します。context オブジェクトの指定は、任意です。callback は、第１引数にエラー内容、 第２引数に描画された文字列を受け取る、典型的な Node.js のコーディングスタイルである関数を指定します。

- #### `bundleRenderer.renderToStream([context])`

サーババンドルを Node.js のストリームへ描画します。context オブジェクトの指定は任意です。より詳しい情報は、[ストリーミング](./streaming.md) の項目を参照してください。

## Renderer 生成時のオプション

- #### `template`

ページ全体の HTML を表すテンプレートを設定します。描画されたアプリケーションの内容を指し示すプレースホルダの代わりになるコメント文 `<!--vue-ssr-outlet-->`をテンプレートには含むべきです。

テンプレートは、次の構文を使用した簡単な補間もサポートします。

- エスケープされたHTMLを補間する Mustache 構文(二重中括弧)の使用
- エスケープしない生のHTMLを補間する Mustache 構文(三重中括弧)の使用

次の構文を見つけた時、テンプレートは自動で適切な内容を挿入します。

- `context.head`: (string) ページ内の head に挿入されるべき任意のマークアップを文字列で指定します。
- `context.styles`: (string) ページ内の head に挿入されるべき任意のインライン CSS を文字列で指定します。もし CSS コンポーネントのために `vue-loader` + `vue-style-loader` を使用する場合、このプロパティは自動で追加されることに注意してください。
- `context.state`: (Object) `window.__INITIAL_STATE__` としてページ内にインライン展開されるべき Vuex のストアの初期状態を指定します。このインライン JSON は自動でクロスサイトスプリクティングを防ぐ [シリアライズされた javascript](https://github.com/yahoo/serialize-javascript) へサニタイズされます。

加えて、`clientManifest` も渡された場合、テンプレートは自動で以下を挿入します。

- (自動で受信される非同期のデータを含んだ)描画対象が必要とするクライアントサイドの JavaScript と CSS アセット
- 描画済みのページに対する最適な `<link rel="preload/prefetch">` Resource Hints

Renderer に `inject: false` も渡すことで、すべての自動挿入を無効にすることができます。

参照：

- [ページテンプレートの使用](./basic.md#using-a-page-template)
- [手動によるアセットインジェクション](./build-config.md#manual-asset-injection)
    - #### `clientManifest`
- 2.3.0以上
- `createBundleRenderer` メソッド内でのみ使用可能

`vue-server-renderer/server-plugin` によって生成されたクライアントビルドマニフェストオブジェクトを提供します。clientManifest は、HTML テンプレートへの自動アセット挿入に適した情報とともに、BundleRenderer を提供します。より詳しい情報は [clientManifest の生成](./build-config.md#generating-clientmanifest) の項目を参照してください。

- 
#### `inject`

    - 2.3.0以上

 `template` 使用時に、自動挿入を行うかどうかを制御します。デフォルトは `true` です。

参考：[手動によるアセットインジェクション](./build-config.md#manual-asset-injection)

- 
#### `shouldPreload`

    - 2.3.0以上

どのファイルが `<link rel="preload">` 生成済みの Resource Hints を持つべきか制御するための関数を指定します。

デフォルトでは、JavaScript と CSS ファイルのみがプリロードされます。これらはアプリケーション起動時に必須なためです。

画像やフォントのようなその他のアセット種別を指定した際、 多すぎるプリロードは処理能力を無駄にし、またパフォーマンスさえも損なうかもしれません。そのため、 プリロードすべきものはアプリケーションの実装依存になるでしょう。 次のように `shouldPreload` オプションを使用することで、プリロードすべきものを正確に制御できます。 

```js
  const renderer = createBundleRenderer(bundle, {
    template,
    clientManifest,
    shouldPreload: (file, type) => {
      // type is inferred based on the file extension.
      // https://fetch.spec.whatwg.org/#concept-request-destination
      if (type === 'script' || type === 'style') {
        return true
      }
      if (type === 'font') {
        // only preload woff2 fonts
        return /\.woff2$/.test(file)
      }
      if (type === 'image') {
        // only preload important images
        return file === 'hero.jpg'
      }
    }
  })
```

- 
#### `runInNewContext`

    - 2.3.0以上
    - `createBundleRenderer` メソッド内でのみ使用可能

デフォルトでは、BundleRenderer の描画ごとに未使用の V8 コンテキストを生成し、バンドル全体を再実行するでしょう。これにはいくつかのメリットがあります。例えば、私たちが以前から言及してきた「ステートフルでシングルトン」なデータを管理することの問題点について心配する必要がありません。しかしながら、このモードはいくつかの無視できないパフォーマンスの問題が起こります。 なぜなら、アプリケーションが大きくなるとき、バンドルの再実行は著しくコストがかかるためです。 

このオプションは、下位互換のためデフォルトは `true` です。しかし、可能ならば常に `runInNewContext: false` を使用することが推奨されます。

参考：[ソースコードの構造](./structure.md)

- 
#### `basedir`

    - 2.2.0以上
    - `createBundleRenderer` メソッド内でのみ使用可能

`node_modules` の依存関係を解決するために、サーババンドルのためのルートディレクトリを明示的に宣言します。 ここでは、インストール済み外部 npm 依存関係とは異なる場所に置かれた生成済みバンドルファイル、または、あなたの現在のプロジェクト内へ npm link された `vue-server-renderer` のみが必要です。

- #### `cache`

[コンポーネントキャッシュ](./caching.md#component-level-caching) の実装を提供します。 キャッシュオブジェクトは以下のインタフェースで実装しなければいけません(以下のような記法を用いる)。

```js
  type RenderCache = {
    get: (key: string, cb?: Function) => string | void;
    set: (key: string, val: string) => void;
    has?: (key: string, cb?: Function) => boolean | void;
  };
```

代表的な使用方法は、次の [lru-cache](https://github.com/isaacs/node-lru-cache) のような流れになります。

```js
  const LRU = require('lru-cache')
  const renderer = createRenderer({
    cache: LRU({
      max: 10000
    })
  })
```

キャッシュオブジェクトは、少なくても `get` と `set` を実装すべき点に注意してください。加えて、`get` と `has` は、もし第二引数に callback が指定された場合、必要に応じてこれを非同期処理にできます。これは、非同期 API を使用したキャッシュの利用を可能にします。 例)以下のような redis クライアント使用する場合

```js
  const renderer = createRenderer({
    cache: {
      get: (key, cb) => {
        redisClient.get(key, (err, res) => {
          // handle error if any
          cb(res)
        })
      },
      set: (key, val) => {
        redisClient.set(key, val)
      }
    }
  })
```

- #### `directives`

以下のように、カスタムディレクティブをサーバサイドの実装で使用可能にします。

```js
  const renderer = createRenderer({
    directives: {
      example (vnode, directiveMeta) {
        // transform vnode based on directive binding metadata
      }
    }
  })
```

一例として、[`v-show` のサーバサイド実装はこちら](https://github.com/vuejs/vue/blob/dev/src/platforms/web/server/directives/show.js) 

## Webpack プラグイン

webpack プラグインは、スタンドアロンのファイルとして提供され、次の値を必要とします。

```js
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')
```

デフォルトで生成されるファイルは以下のものです。

- サーバサイドプラグインのための `vue-ssr-server-bundle.json` 
- クライアントサイドプラグインのための `vue-ssr-client-manifest.json`

プラグインのインスタンス生成時、これらのファイル名は以下のようにカスタマイズ可能です。

```js
const plugin = new VueSSRServerPlugin({
  filename: 'my-server-bundle.json'
})
```

より詳しい情報は、 [ビルド設定](./build-config.md) の項目を参照してください。
