# Использование в не-Node.js окружениях

По умолчанию сборка `vue-server-renderer` предполагает использование Node.js окружения, что делает её непригодной для использования в альтернативных JavaScript окружениях, таких как [PHP V8Js](https://github.com/phpv8/v8js) или [Oracle Nashorn](https://docs.oracle.com/javase/8/docs/technotes/guides/scripting/nashorn/). С версии 2.5+ мы предоставляем сборку в `vue-server-renderer/basic.js`, которая в значительной степени менее зависима от окружения, что делает её пригодной для использования в окружениях, упомянутых выше.

Для обоих вариантов необходимо сначала подготовить окружение создав моки для объектов `global` и `process`, с переменной `process.env.VUE_ENV` установленной в значение `"server"`, и переменной `process.env.NODE_ENV` установленной в значение `"development"` или `"production"`.

При использовании Nashorn также может потребоваться предоставить полифилл для `Promise` или `setTimeout` с использованием нативных таймеров Java.

Пример использования в php-v8js:

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

// предоставляется `vue-server-renderer/basic.js`
renderVueComponentToString(vm, (err, res) => {
  print(res)
})
```
