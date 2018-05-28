---
sidebar: auto
---

# API リファレンス

## createRenderer

任意の引数  [options](#renderer-options) を用いて [`Renderer`](#class-renderer) インスタンスを生成します。

```js
const { createRenderer } = require('vue-server-renderer')
const renderer = createRenderer({ /* options */ })
```
## createBundleRenderer

サーババンドルと任意の引数  [options](#renderer-options) を用いて [`BundleRenderer`](#class-bundlerenderer) インスタンスを生成します。

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer(serverBundle, { /* options */ })
```

引数 `serverBundle` には次のいずれか１つを指定できます:

- 生成されたバンドルファイル (`.js` または `.json`)への絶対パス。ファイルパスとして扱われるために `/` で開始されなければなりません。

- webpack と `vue-server-renderer/server-plugin` によって生成されたバンドルオブジェクト。

- JavaScript コードの文字列 (非推奨)。

より詳しい情報は、 [サーババンドルの紹介](../guide/bundle-renderer.md) と [ビルド設定](../guide/build-config.md) の項目を参照してください。

## `クラス: Renderer`

### renderer.renderToString

シグネチャ:

```js
renderer.renderToString(vm[, context, callback]): ?Promise<string>
```

Vue インスタンスを文字列として描画します。context オブジェクトの指定は、任意です。callback は、第１引数にエラー内容、 第２引数に描画された文字列を受け取る、典型的な Node.js のコーディングスタイルである関数を指定します。

2.5.0 以降においては、コールバックはオプションです。コールバックなしで渡されるとき、HTML に描画されるのを解決するプロミスを返します。

### renderer.renderToStream

シグネチャ:
```js
renderer.renderToStream(vm[, context]): stream.Readable
```

Vue インスタンスを [Node.js の読み取り可能なストリーム](https://nodejs.org/dist/latest-v8.x/docs/api/stream.html#stream_readable_streams) に描画します。より詳細については、[ストリーミング](../guide/streaming.md) を参照してください。

## クラス: BundleRenderer

### bundleRenderer.renderToString

シグネチャ:

```js
bundleRenderer.renderToString([context, callback]): ?Promise<string>
```

サーババンドルを文字列として描画します。context オブジェクトの指定は、任意です。callback は、第１引数にエラー内容、 第２引数に描画された文字列を受け取る、典型的な Node.js のコーディングスタイルである関数を指定します。

2.5.0 以降においては、コールバックは任意です。コールバックなしで渡されたとき、そのメソッドは描画された HTML に解決するプロミスを返します。

### bundleRenderer.renderToStream

シグネチャ:

```js
bundleRenderer.renderToStream([context]): stream.Readable
```

バンドルを [Node.js の読み取り可能なストリーム](https://nodejs.org/dist/latest-v8.x/docs/api/stream.html#stream_readable_streams)に描画します。コンテキストオブジェクトはオプションです。より詳細は [ストリーミング](../guide/streaming.md)を参照してください。

## レンダラオプション

### template

ページ全体の HTML を表すテンプレートを設定します。描画されたアプリケーションの内容を指し示すプレースホルダの代わりになるコメント文 `<!--vue-ssr-outlet-->` をテンプレートには含むべきです。

テンプレートは、次の構文を使用した簡単な補間もサポートします。

- エスケープされたHTMLを補間する Mustache 構文(二重中括弧)の使用
- エスケープしない生のHTMLを補間する Mustache 構文(三重中括弧)の使用

次の構文を見つけた時、テンプレートは自動で適切な内容を挿入します。

- `context.head`: (string) ページ内の head に挿入されるべき任意のマークアップを文字列で指定します。

- `context.styles`: (string) ページ内の head に挿入されるべき任意のインライン CSS を文字列で指定します。もし CSS コンポーネントのために `vue-loader` + `vue-style-loader` を使用する場合、このプロパティは自動で追加されることに注意してください。

- `context.state`: (Object) `window.__INITIAL_STATE__` としてページ内にインライン展開されるべき Vuex のストアの初期状態を指定します。このインライン JSON は自動でクロスサイトスプリクティングを防ぐ [シリアライズされた javascript](https://github.com/yahoo/serialize-javascript) へサニタイズされます。

  2.5.0 以降においては、埋め込みスクリプトはプロダクションモードで自動的に削除されます。

加えて、`clientManifest` も渡された場合、テンプレートは自動で以下を挿入します。

- (自動で受信される非同期のデータを含んだ)描画対象が必要とするクライアントサイドの JavaScript と CSS アセット
- 描画済みのページに対する最適な `<link rel="preload/prefetch">` リソースヒント

レンダラに `inject: false` も渡すことで、すべての自動挿入を無効にすることができます。

参照：

- [ページテンプレートの使用](../guide/#using-a-page-template)
- [手動によるアセットインジェクション](../guide/build-config.md#manual-asset-injection)

### clientManifest

`vue-server-renderer/server-plugin` によって生成されたクライアントビルドマニフェストオブジェクトを提供します。クライアントマニフェストは、HTML テンプレートへの自動アセット挿入に適した情報とともに、バンドルレンダラを提供します。より詳しい情報は [クライアントマニフェストの生成](../guide/build-config.md#generating-clientmanifest) の項目を参照してください。

### inject

`template` 使用時に、自動挿入を行うかどうかを制御します。デフォルトは `true` です。

参考：[手動によるアセットインジェクション](../guide/build-config.md#manual-asset-injection)

### shouldPreload

どのファイルが `<link rel="preload">` 生成済みのリソースヒント持つべきか制御するための関数を指定します。

デフォルトでは、JavaScript と CSS ファイルのみがプリロードされます。これらはアプリケーション起動時に必須なためです。

画像やフォントのようなその他のアセット種別を指定した際、 多すぎるプリロードは処理能力を無駄にし、またパフォーマンスさえも損なうかもしれません。そのため、 プリロードすべきものはアプリケーションの実装依存になるでしょう。 次のように `shouldPreload` オプションを使用することで、プリロードすべきものを正確に制御できます。

```js
const renderer = createBundleRenderer(bundle, {
  template,
  clientManifest,
  shouldPreload: (file, type) => {
    // type はファイル拡張子に基づいて推論されます
    // https://fetch.spec.whatwg.org/#concept-request-destination
    if (type === 'script' || type === 'style') {
      return true
    }
    if (type === 'font') {
      // woff2 フォントのプリロードのみ
      return /\.woff2$/.test(file)
    }
    if (type === 'image') {
      // 重要な画像のプリロードのみ
      return file === 'hero.jpg'
    }
  }
})
```

### shouldPrefetch

- 2.5.0 以降

どのファイルに `<link rel="prefetch">` リソースヒントが生成されるべきかを制御する関数。

標準では、非同期チャンクにおける全てのアセットは、これは優先順位が低いため、プリフェッチされます。ただし、帯域幅の使用を適切に制御するために、プリフェッチするためにカスタマイズすることができます。このオプションは `shouldPreload` と同様の関数シグネチャを必要とします。

### runInNewContext

- `createBundleRenderer` メソッド内でのみ使用可能
- 要求事項: `boolean | 'once'` (`'once'` 2.3.1 以降でのみサポートされる)

デフォルトでは、BundleRenderer の描画ごとに未使用の V8 コンテキストを生成し、バンドル全体を再実行するでしょう。これにはいくつかのメリットがあります。例えば、私たちが以前から言及してきた[「ステートフルでシングルトン」なデータを管理することの問題点](../guide/structure.md#avoid-stateful-singletons)について心配する必要がありません。しかしながら、このモードはいくつかの無視できないパフォーマンスの問題が起こります。 なぜなら、アプリケーションが大きくなるとき、バンドルの再実行は著しくコストがかかるためです。

このオプションは、下位互換のためデフォルトは `true` です。しかし、可能ならば常に `runInNewContext: false` または、`runInNewContext: 'once'`を使用することが推奨されます。

> 2.3.0 では、このオプションは `runInNewContext: false` が個別のグローバルコンテキストを使ってバンドルを実行するバグを持っています。以下の情報は、バージョン2.3.1以降を前提としています。

`runInNewContext: false` の場合は、バンドルコードはサーバープロセスと同じ `global` コンテキストで実行されるので、アプリケーションコード内で `global` を変更するコードには注意してください。

`runInNewContext: 'once'` (2.3.1 以降)の場合は、バンドルは別々の `global` コンテキストで評価されますが、起動時には一度だけ評価されます。これにより、バンドルがサーバープロセスの `global` オブジェクトを誤って汚染するのを防ぐので、より良いアプリケーションコードの分離が可能になります。注意点は次のとおりです:

1. このモードでは、 `global` (例: ポリフィル) を変更する依存関係を外部化することはできません;
2. バンドル実行から返される値は、異なるグローバルコンストラクタを使用します。バンドルの内部で捕捉されたエラーはサーバプロセスの `Error` のインスタンスにはなりません。

参考: [ソースコードの構造](../guide/structure.md)

### basedir

- `createBundleRenderer` メソッド内でのみ使用可能

`node_modules` の依存関係を解決するために、サーババンドルのためのルートディレクトリを明示的に宣言します。 ここでは、インストール済み外部 npm 依存関係とは異なる場所に置かれた生成済みバンドルファイル、または、あなたの現在のプロジェクト内へ npm link された `vue-server-renderer` のみが必要です。

### cache

[コンポーネントキャッシュ](../guide/caching.md#component-level-caching) の実装を提供します。 キャッシュオブジェクトは以下のインタフェースで実装しなければいけません(以下のような記法を用いる):

```js
type RenderCache = {
  get: (key: string, cb?: Function) => string | void;
  set: (key: string, val: string) => void;
  has?: (key: string, cb?: Function) => boolean | void;
};
```

代表的な使用方法は、次の [lru-cache](https://github.com/isaacs/node-lru-cache) のような流れになります:

```js
const LRU = require('lru-cache')

const renderer = createRenderer({
  cache: LRU({
    max: 10000
  })
})
```

キャッシュオブジェクトは、少なくても `get` と `set` を実装すべき点に注意してください。加えて、`get` と `has` は、もし第 2 引数に callback が指定された場合、必要に応じてこれを非同期処理にできます。これは、非同期 API を使用したキャッシュの利用を可能にします。 例)以下のような redis クライアント使用する場合:

```js
const renderer = createRenderer({
  cache: {
    get: (key, cb) => {
      redisClient.get(key, (err, res) => {
        // 任意のエラーがあれば処理
        cb(res)
      })
    },
    set: (key, val) => {
      redisClient.set(key, val)
    }
  }
})
```

### directives

以下のように、カスタムディレクティブをサーバサイドの実装で使用可能にします:

```js
const renderer = createRenderer({
  directives: {
    example (vnode, directiveMeta) {
      // ディレクティブのバインディングメタデータに基づいて vnode を変換する
    }
  }
})
```

例として、[`v-show` のサーバサイド実装はこちら](https://github.com/vuejs/vue/blob/dev/src/platforms/web/server/directives/show.js) です。

## webpack プラグイン

webpack プラグインは、スタンドアロンのファイルとして提供され、次の値を必要とします:

```js
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')
```

デフォルトで生成されるファイルは以下のものです:

- サーバサイドプラグインのための `vue-ssr-server-bundle.json`
- クライアントサイドプラグインのための `vue-ssr-client-manifest.json`

プラグインのインスタンス生成時、これらのファイル名は以下のようにカスタマイズ可能です:

```js
const plugin = new VueSSRServerPlugin({
  filename: 'my-server-bundle.json'
})
```

より詳しい情報は、 [ビルド設定](../guide/build-config.md) の項目を参照してください。
