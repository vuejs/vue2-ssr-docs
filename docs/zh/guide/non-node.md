# 在非 Node.js 环境中使用

`vue-server-renderer` 在默认构建时，会预先假定有一个 Node.js 环境，这使得它在其他 JavaScript 环境（如 [PHP V8Js](https://github.com/phpv8/v8js) 或 [Oracle Nashorn](https://docs.oracle.com/javase/8/docs/technotes/guides/scripting/nashorn/) 中无法使用。在 2.5+ 版本中，我们把那些基本上与环境无关的构建，编译到 `vue-server-renderer/basic.js` 中，这使得它可以在上述环境中使用。

对于所有环境，必须要预先在环境中模拟 `global` 和 `process` 对象，以及将 `process.env.VUE_ENV` 设置为 `"server"` 和将 `process.env.NODE_ENV` 设置为 `"development"` 或 `"production"`。

在 Nashorn 环境下，可能还需要使用 Java 原生定时器，来为 `Promise` 或 `setTimeout` 提供 polyfill。

php-v8js 的示例用法：

``` php
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

``` js
// app.js
var vm = new Vue({
  template: `<div>{{ msg }}</div>`,
  data: {
    msg: 'hello'
  }
})

// 通过 `vue-server-renderer/basic.js` 暴露
renderVueComponentToString(vm, (err, res) => {
  print(res)
})
```
