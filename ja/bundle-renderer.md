# バンドルレンダラの紹介

## 根本的なサーバサイドレンダリングの問題

今までは、バンドルされたサーバサイドのコードが `require` によって直接使用されることを想定していました:

```js
const createApp = require('/path/to/built-server-bundle.js')
```

これはとても簡単です。しかしながらアプリケーションのソースコードを編集する度に、あなたはサーバを停止し再起動させる必要があるでしょう。この作業は開発の生産性を著しく損ないます。さらに、Node.js はソースマップをネイティブサポートしていません。

## バンドルレンダラの追加

`vue-server-renderer` はこの問題を解決するために、`createBundleRenderer` という API を提供しています。また、webpack の拡張プラグインを利用することで、サーババンドルはバンドルレンダラに渡すことができる特別な JSON ファイルとして生成されます。バンドルレンダラが1度生成されると、使用方法は通常のレンダラと一緒ですが、次のような利点があります。

- ビルトインソースマップのサポート ( webpack の設定オプションに `devtool: 'source-map'` を指定)
- 開発中、デプロイ中のホットリロード(更新されたバンドルや、再作成されたレンダラのインスタンスを読み込むだけです)
- クリティカル CSS の評価 (`*.vue` ファイルを利用しているとき): インライン CSS は、描画中に利用されるコンポーネントによって必要とされます。詳細は [CSS](./css.md) を参照してください。
- [clientManifest](./api.md#clientmanifest) によるアセットの注入: 自動的に最適なディレクティブが推測され、プリロードとプリフェッチを実行します。また、初期描画時にはコード分割チャンクを必要とします。

---

次のセクションでバンドルレンダラによって必要とされるビルドされたモノを生成するために、webpack 設定する方法について説明しますが、今既に必要なものがあるものとしましょう。これは、バンドルレンダラを使用する方法です:

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer(serverBundle, {
  runInNewContext: false, // 推奨
  template, // (任意) ページテンプレート
  clientManifest // (任意) クライアントビルドマニフェスト
})
// 内部のサーバ処理 ...
server.get('*', (req, res) => {
  const context = { url: req.url }
  // バンドルを実行することで自動作成されるため、ここでアプリケーションを渡す必要はありません
  // 今、私たちのサーバーはVueアプリから切り離されています！
  renderer.renderToString(context, (err, html) => {
    // ハンドリングエラー ...
    res.end(html)
  })
})
```

バンドルレンダラによって`rendertoString` が呼び出されると、バンドルによってエクスポートされた関数が自動的に実行され、（引数として`context`を渡して）アプリケーションのインスタンスを生成し、描画処理を実行します。

`runInNewContext` オプションを `false` または `'once'` に設定することをお勧めします。詳細は [API リファレンス](./api.md#runinnewcontext)を参照してください。
