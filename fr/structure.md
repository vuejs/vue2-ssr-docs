# Structure de code

## Eviter les singletons d'état

Lors de l'écriture du code exclusif au client, nous sommes habitués au fait que notre code sera interprété dans un nouveau contexte à chaque fois. Cependant, un serveur Node.js est un processus qui reste actif. Quand notre code est appelé dans le processus, il sera interprété une fois et restera en mémoire. Cela signifie que si vous créez un singleton, il sera partagé entre chaque nouvelle requête.

Comme vu dans l'exemple de base, nous **créons une nouvelle instance de Vue principale pour chaque requête.** Cela est similaire au fait que chaque utilisateur utilise une nouvelle instance de l'application dans son propre navigateur. Si nous utilisons une instance partagée entre plusieurs requêtes, nous prenons le risque d'une pollution d'état entre requêtes.

Ainsi, au lieu de créer directement une instance d'application, nous devons exposer une fonction de fabrique qui peut être exécutée à plusieurs reprises pour créer de nouvelles instances d'application pour chaque requête :

``` js
// app.js
const Vue = require('vue')

module.exports = function createApp (context) {
  return new Vue({
    data: {
      url: context.url
    },
    template: `<div>L'URL visible est : {{ url }}</div>`
  })
}
```

Et notre code serveur devient :

``` js
// server.js
const createApp = require('./app')

server.get('*', (req, res) => {
  const context = { url: req.url }
  const app = createApp(context)

  renderer.renderToString(app, (err, html) => {
    // gérer l'erreur...
    res.end(html)
  })
})
```

La même règle s'applique aussi bien au routeur, qu'au store et aux instances de bus d'évènements. Au lieu de l'exporter directement depuis un module et de l'importer dans votre application, vous devez créer une nouvelle instance dans `createApp` et l'injecter depuis l'instance de Vue principale.

> Cette contrainte peut être éliminée en utilisant le moteur de rendu pré-compilé (« bundle renderer ») avec `{ runInNewContext: true }`, cependant il y aura un surcoût significatif en terme de performance lié à la création d'un nouveau contexte vm pour chaque requête.

## Introduction à l'étape de build

Jusqu'ici, nous n'avons pas encore discuté d'une méthode pour disposer de la même application Vue côté client. Pour cela, nous devons utiliser webpack pour fabriquer le bundle de notre application Vue. En fait, nous voulons probablement utiliser aussi webpack pour le bundle serveur, parce que :

- Les applications typiques Vue sont construites avec webpack et `vue-loader`, et beaucoup d'autres fonctionnalités spécifiques à webpack telles que l'import de fichiers via `file-loader`, ou encore l'import de CSS avec `css-loader` qui ne fonctionneraient pas directement en Node.js.

- Bien que la dernière version de Node.js supporte pleinement ES2015, nous avons toujours besoin de transpiler le code client pour prendre en charge les anciens navigateurs. Cela implique à nouveau une étape de build.

L'idée de base est donc que nous utiliserons webpack pour empaqueter notre application à la fois pour le client et le serveur. Le bundle serveur sera nécessaire au serveur et utilisé pour le rendu côté serveur, alors que le bundle client sera envoyé au navigateur pour hydrater le DOM client.

![architecture](https://cloud.githubusercontent.com/assets/499550/17607895/786a415a-5fee-11e6-9c11-45a2cfdf085c.png)

Nous allons discuter des détails de la mise en place dans les prochaines sections. Pour l'instant, considérons que nous avons configuré le build et que nous pouvons écrire le code de notre application Vue avec webpack.

## Structure du code avec webpack

Maintenant que nous utilisons webpack pour développer l'application à la fois côté serveur et côté client, la majorité de notre code source peut être écrite d'une manière universelle, avec l'accès à toutes les fonctionnalités fournies par webpack. Pour autant, il y a [un certain nombre de choses](./universal.md) que vous devez garder à l'esprit en écrivant du code universel.

Voici ce à quoi pourrait ressembler un projet :

``` bash
src
├── components
│   ├── Foo.vue
│   ├── Bar.vue
│   └── Baz.vue
├── App.vue
├── app.js # point d'entrée universel
├── entry-client.js # code exécuté côté navigateur exclusivement
└── entry-server.js # code exécuté côté serveur exclusivement
```

### `app.js`

`app.js` est l'entrée universelle de notre application. Dans une application exclusivement client, nous aurions créé l'instance de Vue principale directement dans ce fichier, et elle aurait été attachée au DOM. Pour le SSR cette responsabilité est déplacée dans le fichier d'entrée réservé au client. `app.js` exporte simplement une fonction `createApp`:

``` js
import Vue from 'vue'
import App from './App.vue'

// exporte une fonction de fabrique pour créer de nouvelles instances
// de l'application, du routeur et du store
export function createApp () {
  const app = new Vue({
    // l'instance racine rend simplement le composant App.
    render: h => h(App)
  })
  return { app }
}
```

### `entry-client.js`:

L'entrée client crée simplement l'application et l'attache au DOM :

``` js
import { createApp } from './app'

// logique de démarrage spécifique au client

const { app } = createApp()

// nous partons du principe que l'élément racine du template App.vue a pour `id="app"`
app.$mount('#app')
```

### `entry-server.js`:

L'entrée serveur utilise un export par défaut qui est une fonction pouvant être appelée de manière répétée à chaque rendu. Pour l'instant, le code ne fait pas grand chose de plus que créer et retourner une instance de l'application. Nous pourrons y intégrer plus tard la correspondance de route côté serveur et la logique de pré-chargement de données.

``` js
import { createApp } from './app'

export default context => {
  const { app } = createApp()
  return app
}
```
