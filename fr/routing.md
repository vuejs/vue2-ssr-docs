# Routage et scission du code

## Routage avec `vue-router`

Vous avez sans doute remarqué que notre code serveur utilise le handler `*` qui accepte n'importe quelle URL. Cela nous permet de ré-utiliser la même configuration des routes pour le client et le serveur !

Il est recommandé d'utiliser le routeur officiel de Vue `vue-router`. Commençons par créer un fichier où sera créé le routeur. De manière similaire à `createApp`, nous aurons besoin d'une nouvelle instance du routeur pour chaque requêtes, donc ce fichier exporte une fonction `createRouter` :

``` js
// router.js
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)

export function createRouter () {
  return new Router({
    mode: 'history',
    routes: [
      // ...
    ]
  })
}
```

Et modifier `app.js` :

``` js
// app.js
import Vue from 'vue'
import App from './App.vue'
import { createRouter } from './router'

export function createApp () {
  // crée l'instance du routeur
  const router = createRouter()

  const app = new Vue({
    // injecte le routeur dans l'instance de Vue
    router,
    render: h => h(App)
  })

  // retourne l'application et le routeur
  return { app, router }
}
```

Maintenant, il faut implémenter la logique des routes côté serveur dans `entry-server.js` :

``` js
// entry-server.js
import { createApp } from './app'

export default context => {
  // vu qu'il peut potentiellement avoir des composants ou des hooks
  // de routes asynchrones, on retourne une Promesse (« Promise ») de telle sorte que
  // le serveur patiente jusqu'à ce que tout soit prêt pour le rendu.
  return new Promise((resolve, reject) => {
    const { app, router } = createApp()

    // défini la location du routeur serveur
    router.push(context.url)

    // on attend que le routeur ait terminé de traiter avec les composants et
    // hooks asynchrones
    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      // pas de routes correspondantes, on rejette la requête avec une 404
      if (!matchedComponents.length) {
        return reject({ code: 404 })
      }

      // la Promise doit résoudre l'instance de l'application qui pourra
      // ensuite être rendue
      resolve(app)
    }, reject)
  })
}
```

En assumant que le bundle serveur soit déjà fait (encore une fois, on ignore l'étape de configuration du build pour l'instant), l'usage de ce bundle ressemblerait à ça :


``` js
// server.js
const createApp = require('/path/to/built-server-bundle.js')

server.get('*', (req, res) => {
  const context = { url: req.url }

  createApp(context).then(app => {
    renderer.renderToString(app, (err, html) => {
      if (err) {
        if (err.code === 404) {
          res.status(404).end('Page non trouvée')
        } else {
          res.status(500).end('Erreur interne du serveur')
        }
      } else {
        res.end(html)
      }
    })
  })
})
```

## Scission du code

La scission du code, ou les parties chargées à la volée de votre application, aide à réduire la quantité de ressources qui a besoin d'être téléchargée par le navigateur pour le rendu initial, et peut grandement améliorer le TTI (time-to-interactive) pour les grosses applications. Le but est de « charger uniquement ce qui est nécessaire » pour l'écran initial.

Vue permet de créer des composants asynchrones en respectant le concept d'[objet de première classe](https://fr.wikipedia.org/wiki/Objet_de_premi%C3%A8re_classe). En les combinant avec [le support de webpack 2 pour l'utilisation de l'importation dynamique pour scinder le code](https://webpack.js.org/guides/code-splitting-async/), tout ce que vous avez à faire est :

``` js
// changer ça :
import Foo from './Foo.vue'

// pour ça :
const Foo = () => import('./Foo.vue')
```

Cela fonctionnera dans n'importe quel scénario si vous êtes en train de faire une application Vue uniquement pour le côté client. Toutefois, il y aura certaines limitations en l'utilisant avec du SSR. Premièrement, il faut résoudre tous les composants asynchrones à l'avance sur le serveur avant de faire le rendu, car sinon il y aura juste un emplacement vide dans le code HTML. Pour le côté client, il faut aussi faire cela avant de commencer l'hydratation des données, sinon il y aurait des erreurs d'incompatibilités sur le contenu.

Tout cela rend un peu compliqué l'utilisation des composants asynchrones à des endroits spécifiques dans votre application (nous allons probablement améliorer cela dans le futur). Toutefois, **cela fonctionne parfaitement si vous le faites au niveau de la route** - c.-à-d. d'utiliser les composants asynchrones dans la configuration des routes - car `vue-router` ira automatiquement résoudre les composants asynchrones nécessaires au bon fonctionnement de la route. Vous devez être sûr d'utiliser `router.onReady` sur le serveur et le client. Nous l'avons déjà fait pour le fichier d'entrée du serveur, il ne nous reste plus maintenant qu'à faire de même pour le fichier d'entrée du client :

``` js
// entry-client.js

import { createApp } from './app'

const { app, router } = createApp()

router.onReady(() => {
  app.$mount('#app')
})
```

Un exemple de configuration de route avec des composants asynchrones :

``` js
// router.js
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)

export function createRouter () {
  return new Router({
    mode: 'history',
    routes: [
      { path: '/', component: () => import('./components/Home.vue') },
      { path: '/item/:id', component: () => import('./components/Item.vue') }
    ]
  })
}
```
