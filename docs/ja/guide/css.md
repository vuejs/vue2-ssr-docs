# CSS の管理

CSS を管理するためのおすすめの方法は、シンプルに単一ファイルコンポーネントである `*.vue` の中で `<style>`を使うことです。これは以下を提供します:

- 関連するコンポーネントのスコープ付き CSS
- プリプロセッサや PostCSS を活用する機能
- 開発時のホットリロード

さらに重要なことは、`vue-loader` によって内部的に使われている `vue-style-loader` はサーバーによる描画のためのいくつかの特別な機能を持っています:

- クライアントとサーバーのためのユニバーサル変換処理の体験

-  `bundleRenderer` を使用した時の自動的なCSS評価

  サーバーによる描画を使用するなら、コンポーネントの CSS はHTMLに集められてインライン化されます ( `template` オプションを使用していれば自動で扱われます ) 。クライアント上で、コンポーネントが初めて使用されたとき、`vue-style-loader` は既にそのコンポーネントにサーバーインラインCSSがあるかチェックします。もし存在しない場合、そのCSSは動的に `<style>` タグ経由で注入されます。

- 共通する CSS の抽出

  この設定は [`extract-text-webpack-plugin`](https://github.com/webpack-contrib/extract-text-webpack-plugin) を使ってメインチャンクのCSSを個別のCSSに抽出することをサポートします ( `template` で自動注入されます ) 。これは、ファイルが個々にキャッシュされることをゆるしています。共通するCSSがたくさんある場合にこの方法をおすすめします。

  非同期コンポーネントの CSS は JavaScript の文字列としてインライン化されたままになり、`vue-style-loader` によって扱われます。

## CSS 抽出の有効化

`*.vue` から CSS を抽出するために、`vue-loader` の `extractCSS` オプションを使います ( `vue-loader>=12.0.0` が必要 ) :

```js
// webpack.config.js
const ExtractTextPlugin = require('extract-text-webpack-plugin')

// CSS 抽出は、開発中ではホットリロードされるように、
// 本番環境でのみ有効にする必要があります
const isProduction = process.env.NODE_ENV === 'production'

module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          // CSS 抽出を有効にする
          extractCSS: isProduction
        }
      },
      // ...
    ]
  },
  plugins: isProduction
    // プラグインを追加してください！
    ? [new ExtractTextPlugin({ filename: 'common.[chunkhash].css' })]
    : []
}
```

上記の設定は `*.vue` ファイルのスタイルのみに適用されますが、外部 CSS を Vue コンポーネントにインポートするために `<style src="./foo.css"></style>` を使うことができます。

もし `import 'foo.css'` のように JavaScriptからCSSをインポートしたいならば、適切な loader の設定が必要です:

```js
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.css$/,
        // 重要: style-loader の代わりに vue-style-loader を使用します
        use: isProduction
          ? ExtractTextPlugin.extract({
              use: 'css-loader',
              fallback: 'vue-style-loader'
            })
          : ['vue-style-loader', 'css-loader']
      }
    ]
  },
  // ...
}
```

## 依存関係からのスタイルのインポート

NPM 依存で CSS をインポートするときに気を付けることがいくつかあります:

1. サーバービルドで外部化しないでください。

2. もし CSS抽出 +  `CommonsChunkPlugin` でベンダー抽出を使用している場合、抽出されたベンダーのチャンクに抽出された CSS があれば、`extract-text-webpack-plugin` に問題が発生します。この問題を解決するためには、ベンダーのチャンクに CSS ファイルを含めないでください。クライアント側の webpack の設定例です:

```js
  module.exports = {
    // ...
    plugins: [
      // deps をベンダーのチャンクに抽出してキャッシュを改善するのが一般的です
      new webpack.optimize.CommonsChunkPlugin({
        name: 'vendor',
        minChunks: function (module) {
          // モジュールはベンダーチャンクに抽出されます...
          return (
            // node_modules 内部な場合
            /node_modules/.test(module.context) &&
            // リクエストが CSS ファイルの場合、抽出しない
            !/\.css$/.test(module.request)
          )
        }
      }),
      // webpack ランタイム & マニフェストを抽出する
      new webpack.optimize.CommonsChunkPlugin({
        name: 'manifest'
      }),
      // ...
    ]
  }
```
