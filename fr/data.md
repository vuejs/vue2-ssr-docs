# Récupération de données et état

## Gestionnaire d'état des données

Pendant le SSR, nous allons essentiellement faire le rendu d'un « instantané » de notre application, aussi si votre application est liée à des données asynchrones, **ces données vont devoir être pré-chargées et résolues avant de débuter la phase de rendu**.

Un autre point important côté client ; les mêmes données doivent être disponibles avant que l'application ne soit montée, autrement, l'application côté client va faire le rendu d'un état différent et l'hydratation va échouer.

Pour résoudre cela, les données pré-chargées doivent vivre en dehors de la vue du composant, dans un gestionnaire de données, ou dans un « gestionnaire d'état ». Côté serveur, nous pouvons pré-charger et remplir les données dans le gestionnaire de données avant le rendu. De plus, nous allons sérialiser et injecter l'état dans le HTML. Le gestionnaire de données côté client pourra directement récupérer l'état depuis le HTML avant que l'application ne soit montée.

Nous allons utiliser le gestionnaire d'état officiel (« store ») de la bibliothèque [Vuex](https://github.com/vuejs/vuex/) pour cette partie. Créons un fichier `store.js`, avec divers jeux de logique pour pré-charger un élément en nous basant sur un identifiant :

``` js
// store.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

// Supposons que nous ayons une API universelle retournant
// des Promesses (« Promise ») et ignorons les détails de l'implémentation
import { fetchItem } from './api'

export function createStore () {
  return new Vuex.Store({
    state: {
      items: {}
    },
    actions: {
      fetchItem ({ commit }, id) {
        // retournant la Promesse via `store.dispatch()`, nous savons
        // quand les données ont été pré-chargées
        return fetchItem(id).then(item => {
          commit('setItem', { id, item })
        })
      }
    },
    mutations: {
      setItem (state, { id, item }) {
        Vue.set(state.items, id, item)
      }
    }
  })
}
```

Et mettons à jour `app.js`:

``` js
// app.js
import Vue from 'vue'
import App from './App.vue'
import { createRouter } from './router'
import { createStore } from './store'
import { sync } from 'vuex-router-sync'

export function createApp () {
  // créer le routeur et l'instance du store
  const router = createRouter()
  const store = createStore()

  // synchroniser pour que l'état de la route soit disponible en tant que donnée du store
  sync(store, router)

  // créer l'instance de l'application, injecter le routeur et le store
  const app = new Vue({
    router,
    store,
    render: h => h(App)
  })

  // exposer l'application, le routeur et le store.
  return { app, router, store }
}
```

## Collocation logique avec les composants

Donc, où devons nous appeler le code en charge de l'action de récupération de données ?

Les données que nous avons besoin de pré-charger sont déterminées par la route visitée, qui va aussi déterminer quels composants vont être rendus. En fait, les données nécessaires a une route donnée sont aussi les données nécessaires aux composants pour être rendus pour une route. Aussi il serait naturel de placer la logique de récupération de données à l'intérieur des composants de route.

Nous allons exposer une fonction statique personnalisée `asyncData` sur nos composants de route. Notez que, puisque cette fonction va être appelée avant l'instanciation des composants, l'accès à `this` ne sera pas possible. Le store et les informations de route ont donc besoin d'être passés en tant qu'arguments :

``` html
<!-- Item.vue -->
<template>
  <div>{{ item.title }}</div>
</template>

<script>
export default {
  asyncData ({ store, route }) {
    // retourner la Promesse depuis l'action
    return store.dispatch('fetchItem', route.params.id)
  },

  computed: {
    // afficher l'élément depuis l'état du store.
    item () {
      return this.$store.state.items[this.$route.params.id]
    }
  }
}
</script>
```

## Récupération de données côté serveur

Dans `entry-server.js` nous pouvons obtenir les composants qui concordent avec une route grâce à `router.getMatchedComponents()`, et appeler `asyncData` si le composant l'expose. Nous avons ensuite besoin d'attacher l'état résolu au contexte de rendu.

``` js
// entry-server.js
import { createApp } from './app'

export default context => {
  return new Promise((resolve, reject) => {
    const { app, router, store } = createApp()

    router.push(context.url)

    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      if (!matchedComponents.length) {
        return reject({ code: 404 })
      }

      // appeler `asyncData()` sur toutes les routes concordantes
      Promise.all(matchedComponents.map(Component => {
        if (Component.asyncData) {
          return Component.asyncData({
            store,
            route: router.currentRoute
          })
        }
      })).then(() => {
        // Après que chaque hook de pré-chargement soit résolu, notre store est maintenant
        // rempli avec l'état nécessaire au rendu de l'application.
        // Quand nous attachons l'état au contexte, et que l'option `template`
        // est utilisée pour faire le rendu, l'état va automatiquement être
        // être sérialisé et injecté dans le HTML en tant que `window.__INITIAL_STATE__`.
        context.state = store.state

        resolve(app)
      }).catch(reject)
    }, reject)
  })
}
```

En utilisant `template`, `context.state` va automatiquement être encapsulé dans le HTML final en tant qu'état `window.__INITIAL_STATE__`. Côté client, le store voudra récupérer cet état avant de monter l'application :

``` js
// entry-client.js

const { app, router, store } = createApp()

if (window.__INITIAL_STATE__) {
  store.replaceState(window.__INITIAL_STATE__)
}
```

## Récupération de données côté client

Côté client, il y a deux différentes approches pour gérer du récapération de données :

1. **Résoudre les données avant de changer de route :**

  Avec cette stratégie, l'application va rester sur la vue courante jusqu'à ce que les données nécessaires à la vue suivante soient résolues. L'avantage est que la vue suivante pourra faire le rendu complet du contenu aussitôt qu'il sera prêt, mais si les données mettent trop de temps à charger, l'utilisateur va se sentir « bloquer » sur la vue courante. C'est pourquoi il est recommandé de fournir un indicateur de chargement si vous utilisez cette stratégie.

  Nous pouvons implémenter cette stratégie côté client en vérifiant la concordance des composants et en exécutant leurs fonctions `asyncData` à l'intérieur du hook global du routeur. Notez que nous devrions enregistrer ce hook après que la route initiale ne soit prête et donc il n'est pas nécessaire de pré-charger de nouveau les données du serveur ayant déjà été pré-chargées.

  ``` js
  // entry-client.js

  // ...omission du code sans rapport

  router.onReady(() => {
    // Ajouter le hook du routeur pour gérer `asyncData`
    // Cela étant fait après la résolution de la route initial, évitons une double récupération de données
    // des données que nous avons déjà. Utilisation de `router.beforeResolve()`, ainsi tous
    // les composants asynchrones sont résolues.
    router.beforeResolve((to, from, next) => {
      const matched = router.getMatchedComponents(to)
      const prevMatched = router.getMatchedComponents(from)

      // nous allons uniquement nous occuper des composants qui n'ont pas déjà été rendu
      // aussi nous allons les comparer jusqu'à ce que deux éléments concordant diffères
      let diffed = false
      const activated = matched.filter((c, i) => {
        return diffed || (diffed = (prevMatched[i] !== c))
      })

      if (!activated.length) {
        return next()
      }

      // c'est ici qu'il faudrait lancer un indicateur de chargement si nous en avions un

      Promise.all(activated.map(c => {
        if (c.asyncData) {
          return c.asyncData({ store, route: to })
        }
      })).then(() => {

        // arrêt de l'indicateur de chargement

        next()
      }).catch(next)
    })

    app.$mount('#app')
  })
  ```

2. **Récupérer les données après que les vues concordantes soient rendues :**

  Cette stratégie place la logique de récupération de données côté client dans la fonction `beforeMount` de la vue du composant. Cela permet aux vues de changer instantanément quand un changement de route est enclenché, aussi l'application semblera un peu plus réactive. Cependant, la vue suivante n'aura pas l'intégralité de ses données disponibles lors du rendu. Il est donc nécessaire d'avoir un état de chargement conditionnel pour chaque vue de composant utilisant cette stratégie.

  Cela peut être réalisé avec un mixin global uniquement côté client :

  ``` js
  Vue.mixin({
    beforeMount () {
      const { asyncData } = this.$options
      if (asyncData) {
        // assigner une opération de récupération de données à une Promesse
        // ainsi tout ce que nous devons faire dans un composant est `this.dataPromise.then(...)`
        // pour exécuter la suite des tâches une fois que les données sont prêtes
        this.dataPromise = asyncData({
          store: this.$store,
          route: this.$route
        })
      }
    }
  })
  ```

Les deux stratégies conduisent à une expérience utilisateur singulièrement différente et doivent être choisis en fonction du scénario de l'application que vous construisez. Mais indépendamment de votre choix de stratégie, la fonction `asyncData` devrait également être appelée quand la route d'un composant est de nouveau utilisée (même route, mais avec des paramètres ou une demande « query » différente comme par ex. avec `utilisateur/1` et `utilisateur/2`). Nous pouvons également réaliser ceci avec un mixin global uniquement côté client.

``` js
Vue.mixin({
  beforeRouteUpdate (to, from, next) {
    const { asyncData } = this.$options
    if (asyncData) {
      asyncData({
        store: this.$store,
        route: to
      }).then(next).catch(next)
    } else {
      next()
    }
  }
})
```

## Scission de code du Store

Dans une grosse application, notre store Vuex va très probablement être scinder dans de multiples modules. Bien sur, il est aussi possible de scinder le code de ces modules en fragments correspondant aux routes. Supposons que nous ayons le module store suivant :

``` js
// store/modules/foo.js
export default {
  namespaced: true,
  // IMPORTANT: l'état doit être une fonction sinon le module ne
  // pourra pas être instancié de multiples fois
  state: () => ({
    count: 0
  }),
  actions: {
    inc: ({ commit }) => commit('inc')
  },
  mutations: {
    inc: state => state.count++
  }
}
```

Nous pouvons utiliser `store.registerModule` pour enregistrer ce module à la volée dans le hook `asyncData` du composant :

``` html
// inside a route component
<template>
  <div>{{ fooCount }}</div>
</template>

<script>
// importer le module d'ici et non de `store/index.js`
import fooStoreModule from '../store/modules/foo'

export default {
  asyncData ({ store }) {
    store.registerModule('foo', fooStoreModule)
    return store.dispatch('foo/inc')
  },

  // IMPORTANT: il faut éviter le double enregistrement de module côté client
  // quand la route est visitée plusieurs fois.
  destroyed () {
    this.$store.unregisterModule('foo')
  },

  computed: {
    fooCount () {
      return this.$store.state.foo.count
    }
  }
}
</script>
```

Parce que le module est maintenant une dépendance du composant de route, il peut a présent être déplacer dans un fragment de composant de route par webpack.

---

Fiou, cela fait pas mal de code ! Cela est dû au fait que le pré-chargement universel est probablement le problème le plus complexe d'une application avec rendu côté serveur et nous avons poser les bases pour un développement futur plus simple. Maintenant que cette base est mise en place, modifier des composants individuellement sera en fait plutôt agréable.
