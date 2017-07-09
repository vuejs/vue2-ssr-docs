# バンドルレンダラの紹介

## 根本的なサーバサイドレンダリングの問題

今までは、バンドルされたサーバサイドのコードが `require` によって直接使用されることを想定していました。

```js
const createApp = require('/path/to/built-server-bundle.js')
```

これはとても簡単です。しかしながらアプリケーションのソースコードを編集する度に、あなたはサーバを停止し再起動させる必要があるでしょう。この作業は開発の生産性を著しく損ないます。さらに、Node.js はソースマップをネイティブサポートしていません。

## バンドルレンダラの追加

`vue-server-renderer` はこの問題を解決するために、`createBundleRenderer` という API を提供しています。また、webpack の拡張プラグインを利用することで、サーババンドルはバンドルレンダラに渡すことができる特別な JSON ファイルとして生成されます。バンドルレンダラが1度生成されると、使用方法は通常のレンダラと一緒ですが、次のような利点があります。

- ビルトインソースマップのサポート ( webpack の設定オプションに `devtool: 'source-map'` を指定)
- 開発中、デプロイ中のホットリロード(更新されたバンドルや、再作成されたレンダラのインスタンスを読み込むだけです)
- クリティカル CSS の評価 (`*.vue` ファイルを利用しているとき): インライン CSS は、レンダリング中に利用されるコンポーネントによって必要とされます。詳細は [CSS](./css.md) をご覧ください。
- [clientManifest](./api.md#clientmanifest) によるアセットの注入: 自動的に最適なディレクティブが推測され、プリロードとプリフェッチを実行します。また、初期レンダリング時にはコード分割チャンクを必要とします。

---

次のセクションで、バンドルレンダラと webpack のビルドの設定方法ついて説明します。今はすでに必要なものが準備できていると仮定しましょう。こちらはバンドルレンダラを生成し、使用する方法です:

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer(serverBundle, {
  runInNewContext: false, // recommended
  template, // (optional) page template
  clientManifest // (optional) client build manifest
})
// inside a server handler...
server.get('*', (req, res) => {
  const context = { url: req.url }
  // No need to pass an app here because it is auto-created by the
  // executing the bundle. Now our server is decoupled from our Vue app!
  renderer.renderToString(context, (err, html) => {
    // handle error...
    res.end(html)
  })
})
```

バンドルレンダラによって`rendertoString` が呼び出されると、バンドルによってエクスポートされた関数が自動的に実行され、（引数としてコンテキストを渡して）アプリケーションのインスタンスを生成し、レンダリングを実行します。

---

### `runInNewContext` オプション

通常では、それぞれのレンダリング毎にバンドルレンダラは新しい V8 コンテキストを生成し、バンドル全体を再実行します。これにはいくつかの利点があります。例えば、私たちが以前に言及した "ステートフルシングルトン" 問題について心配する必要がなくなります。しかし、このモードにはかなりのパフォーマンスコストがかかります。なぜなら、アプリケーションを大きくなるにつれ、バンドル全体を再実行することは特にコストとなるからです。

`vue-server-renderer >= 2.3.0` では、このオプションは下位互換性のために依然として初期値が `true` にされています。しかし、できる限り `runInNewContext: false` にしておくことをオススメします。

`runInNewContext: false`の場合は、バンドルは引き続き**別な<code>グローバル</code>コンテキスト**として1度だけ評価されます。これにより、バンドルが誤ってサーバプロセスのグローバルオブジェクトを汚染してしまうことを防ぎます。通常との動作の違いは、それぞれのレンダリングの実行時に**新しい**コンテキストを生成しないことです。
