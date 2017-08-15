# Gestion des entêtes

Similaire à l'injection de fichier, la gestion des entêtes suit la même idée : nous pouvons dynamiquement attacher des données au rendu `context` dans le cycle de vie du composant, et interpoler ses données dans le template.

> Dans la version 2.3.2+, vous pouvez directement accéder au contexte SSR du composant via `this.$ssrContext`. Dans les versions plus anciennes, vous devez injecter manuellement le contexte SSR en le passant à la fonction `createApp()` et ainsi l'exposer dans `$options` de l'instance racine — les composants enfants pouvant y accéder via `this.$root.$options.ssrContext`.

Nous pouvons écrire un mixin simple pour effectuer la gestion du titre :

``` js
// title-mixin.js

function getTitle (vm) {
  // les composants doivent simplement fournir une option `title`
  // pouvant être soit une chaîne de caractères soit une fonction
  const { title } = vm.$options
  if (title) {
    return typeof title === 'function'
      ? title.call(vm)
      : title
  }
}

const serverTitleMixin = {
  created () {
    const title = getTitle(this)
    if (title) {
      this.$ssrContext.title = title
    }
  }
}

const clientTitleMixin = {
  mounted () {
    const title = getTitle(this)
    if (title) {
      document.title = title
    }
  }
}

// `VUE_ENV` peut être injecté avec `webpack.DefinePlugin`
export default process.env.VUE_ENV === 'server'
  ? serverTitleMixin
  : clientTitleMixin
```

Maintenant, un composant de route peut être utilisé ainsi pour contrôler le titre du document :

``` js
// Item.vue
export default {
  mixins: [titleMixin],
  title () {
    return this.item.title
  }

  asyncData ({ store, route }) {
    return store.dispatch('fetchItem', route.params.id)
  },

  computed: {
    item () {
      return this.$store.state.items[this.$route.params.id]
    }
  }
}
```

Et passé au moteur de rendu de paquetage dans le template.

``` html
<html>
  <head>
    <title>{{ title }}</title>
  </head>
  <body>
    ...
  </body>
</html>
```

**Notes :**

- Utilisez les doubles moustaches (interpolation de HTML échapé) pour éviter les attaques XSS.

- Vous devriez fournir un titre par défaut quand vous créez l'object `context` au cas ou aucun composant de définisse de titre durant le rendu.

---

En utilisant la même stratégie, vous pouvez facilement étendre votre mixin en une fonction utilitaire générique de gestion des entêtes.
