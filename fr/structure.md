# Structure du code source

## Éviter les singletons avec états

Pendant l'écriture d'un code spécifique au client, on utilise le fait que notre code sera exécuté dans un nouveau contexte à chaque fois. Toutefois, le serveur Node.js est un processus qui reste actif dans le temps. Lorsque notre code sera utilisé dans ce processus, il sera exécuté une fois et sera gardé en mémoire. Ce qui signifie que si vous créé un objet singleton, il sera partagé entre toutes les requêtes entrantes.

Comme nous l'avons vu dans l'exemple de base, nous avons **créé une nouvelle instance de Vue pour chaque requête**. Ce comportement est similaire à la façon dont chaque utilisateur utilisera une nouvelle instance de l'application dans son propre navigateur. Si on partageait une même instance à travers chaque requêtes, cela entraînera une pollution du *state*.

Enfin, au lieu de directement créer une instance de l'application, il serait plus intéressant de créer une fonction *factory* qui pourra être utilisée à plusieurs reprises afin de créer des nouvelles instances d'application pour chaque requête :

``` js
// app.js
const Vue = require('vue')

module.exports = function createApp (context) {
  return new Vue({
    data: {
      url: context.url
    },
    template: `<div>L'url visitée est : {{ url }}</div>`
  })
}
```

Et le code du serveur devient :

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

La même règle s'applique aux instances du routeur, du store, ainsi qu'au bus d'événements. Au lieu de l'exporter depuis un module et de l'importer dans votre application, il faut créer une nouvelle instance dans `createApp` et l'injecter dans l'instance de Vue.

> Cette contrainte peut être contournée en utilisant le moteur de rendu avec `{ runInNewContext: true }`, toutefois, cela implique un certain coût de performance car un nouveau contexte *vm* a besoin d'être créé pour chaque requête. 

## Introduction à l'étape de *build*

Jusqu'à présent, nous n'avons pas encore vu comment délivrer la même application Vue au client. Pour faire cela, nous avons besoin d'utiliser webpack pour empaqueter notre application Vue. En fait, nous voudrons également utiliser webpack pour empaqueter l'application Vue sur le serveur, car :

- Les applications Vue sont souvent construites avec webpack et `vue-loader`, et plusieurs fonctionnalités spécifiques à webpack, comme l'import de fichiers grâce `file-loader` ou l'import de CSS grâce à `css-loader`, ne fonctionneront pas directement avec Node.js.

- Bien que la dernière version de Node.js supporte entièrement les fonctionnalités d'ES2015, il faudra cependant toujours transpiler le code pour le client afin de rester compatible avec les vieux navigateurs. Ce qui implique une étape de *build* supplémentaire.

L'idée est d'utiliser webpack pour empaqueter notre application pour le client et pour le serveur. Le *bundle* serveur sera requis par le serveur et utilisé pour le SSR, alors que le *bundle* client sera envoyé au navigateur pour hydrater le code HTML.

![architecture](https://cloud.githubusercontent.com/assets/499550/17607895/786a415a-5fee-11e6-9c11-45a2cfdf085c.png)

Nous discuterons des détails de l'installation un peu plus tard. Pour le moment, nous assumerons que nous avons déjà configuré notre projet, et que nous pouvons écrire notre code avec webpack d'activé

## Structure du code avec Webpack

Maintenant que nous utilisons webpack pour gérer l'application pour le serveur et le client, la majorité de notre code peut être écrite de manière universelle, tout en en pouvant utiliser toutes les fonctionnalités de webpack. Enfin, il y [a un tas de choses](./universal.md) qu'il faut garder en tête afin d'écrire du code universel.

Un projet simple ressemblerait à ça :

``` bash
src
├── components
│   ├── Foo.vue
│   ├── Bar.vue
│   └── Baz.vue
├── App.vue
├── app.js # entrée universelle
├── entry-client.js # exécuté seulement avec le navigateur
└── entry-server.js # exécuté seulement avec le serveur
```

### `app.js`

`app.js` est l'entrée universelle de notre application. Dans une application client, c'est dans ce fichier qu'une instance de Vue sera créée et montée directement sur le DOM. Toutefois, pour le SSR, cette responsabilité sera déléguée au fichier d'entrée pour le client. `app.js` exporte simplement la fonction `createApp` :

``` js
import Vue from 'vue'
import App from './App.vue'

// exporte une fonction factory, pour créer des nouvelles instances 
// de l'app, du router, et du store.
export function createApp () {
  const app = new Vue({
    // cette instance ne fait que le rendu du composant App
    render: h => h(App)
  })
  return { app }
}
```

### `entry-client.js`:

Le fichier d'entrée du client crée l'application et la monte sur le DOM :

``` js
import { createApp } from './app'

// code spécifique au client...

const { app } = createApp()

// en assumant que l'élément racine du template App.vue possède id="app".
app.$mount('#app')
```

### `entry-server.js`:

Le fichier d'entrée du serveur utilise l'export par défaut, qui est une fonction qui peut être appelée à plusieurs reprises pour chaque rendu. À ce moment, cette fonction ne fait pas grand chose à part créer et retourner une instance d'application - sauf plus tard, lorsqu'on utilisera le router et la pré-récupération de données ici.

``` js
import { createApp } from './app'

export default context => {
  const { app } = createApp()
  return app
}
```
