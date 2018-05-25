# Utilisation dans des environnements autres que Node.js

Le build par défaut de `vue-server-renderer` est créé pour fonctionner dans un environnement Node.js, ce qui le rend inutilisable pour des environnements comme [PHP V8Js](https://github.com/phpv8/v8js) ou [Oracle Nashorn](https://docs.oracle.com/javase/8/docs/technotes/guides/scripting/nashorn/). Dans la 2.5+, nous avons ajouté un build dans `vue-server-renderer/basic.js` qui est très largement agnostique en matière d'environnement, le rendant utilisable dans les environnements mentionnés précédemment.

Pour les deux environnements, il est nécessaire de d'abord préparer l'environnement en simulant les objets `global` et `process` avec `process.env.VUE_ENV` mis à `"server"` et `process.env.NODE_ENV` mis à `"development"` ou `"production"`.

Dans Nashorn, il serait également nécessaire de fournir un polyfill pour les `Promise` ou `setTimeout` en utilisant les compteurs natifs Java.

Exemple d'utilisation avec V8Js :

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
    msg: 'bonjour'
  }
})

// exposé par `vue-server-renderer/basic.js`
renderVueComponentToString(vm, (err, res) => {
  print(res)
})
```
