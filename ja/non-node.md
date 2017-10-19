# 非 Node.js 環境における使用

 `vue-server-renderer` の標準ビルドは Node.js 環境を想定していますが、これは、[PHP V8Js](https://github.com/phpv8/v8js) や  [Oracle Nashorn](https://docs.oracle.com/javase/8/docs/technotes/guides/scripting/nashorn/) のような別のJavaScript 環境では使用できなくなります。2.5 のにおいて、環境にはほとんど影響されない、 `vue-server-renderer/basic.js` のビルドを出荷しました。これにより、上記の環境で使用できるようになります。

どちらの環境に対して、それは `global` と `process` オブジェクトをモックすることによって最初に環境を準備する必要があり、 `process.env.VUE_ENV` に `"server"` を設定し、そして `process.env.NODE_ENV` に `"development"` または {code6}"production"{/code6} を設定します。

Nashornでは、Java のネイティブタイマーを使用して、 `Promise{/ code0} または <code data-md-type="codespan">setTimeout` のポリフィルを提供する必要があります。

php-v8js での使用例:

```php
<?php
$vue_source = file_get_contents('/path/to/vue.js');
$renderer_source = file_get_contents('/path/to/vue-server-renderer/basic.js');
$app_source = file_get_contents('/path/to/app.js');

$v8 = new V8Js();

$v8->executeString('var process = { env: { VUE_ENV: "server", NODE_ENV: "production" }}; this.global = { process: process };');
$v8->executeString($vue_source);
$v8->executeString($renderer_source);
$v8->executeString($app_source);
?>
```

---

```js
// app.js
var vm = new Vue({
  template: `<div>{{ msg }}</div>`,
  data: {
    msg: 'hello'
  }
})

// `vue-server-renderer/basic.js` によってエクスポーズ
renderVueComponentToString(vm, (err, res) => {
  print(res)
})
```
