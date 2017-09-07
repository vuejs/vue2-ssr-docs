# Mise en cache

Même si le SSR de Vue est très rapide, il ne peut égaler les performances d'un rendu utilisant une chaîne de caractères pure du fait du coût de la création des instances de composants et des nœuds du DOM virtuel. Dans le cas ou les performances du SSR seraient critiques, il est conseillé d'utiliser une stratégie de mise en cache pour améliorer significativement le temps de réponse et réduire la charge du serveur.

## Mise en cache au niveau de la page

Une application de rendu côté serveur est, dans la plupart des cas, liée à des données externes. Donc le contenu est dynamique par nature et ne peut pas être mis en cache pour des périodes étendues. Cependant, si le contenu n'est pas spécifique à un utilisateur (c.-à-d. qu'un même URL rend toujours un même contenu indépendamment des utilisateurs), nous pouvons mettre en place une stratégie appelée [micro-caching](https://www.nginx.com/blog/benefits-of-microcaching-nginx/) pour améliorer significativement la capacité de notre application à prendre en charge un fort trafique.

Ceci est habituellement fait au niveau de la couche nginx, mais nous pouvons également l'implémenter en Node.js :

``` js
const microCache = LRU({
  max: 100,
  maxAge: 1000 // Important : les entrées expires après une seconde.
})

const isCacheable = req => {
  // implémentation d'un algorithme pour définir si une requête est spécifique à un utilisateur.
  // seules les pages non spécifiques à un utilisateur peuvent être mises en cache
}

server.get('*', (req, res) => {
  const cacheable = isCacheable(req)
  if (cacheable) {
    const hit = microCache.get(req.url)
    if (hit) {
      return res.end(hit)
    }
  }

  renderer.renderToString((err, html) => {
    res.end(html)
    if (cacheable) {
      microCache.set(req.url, html)
    }
  })
})
```

Parce que le composant est mis en cache pour seulement une seconde, les utilisateurs ne verront pas le contenu déjà périmé. Cependant, cela signifie que le serveur a besoin de faire un rendu complet une seule fois par seconde pour chaque page mise en cache.

## Mise en cache au niveau du composant

`vue-server-renderer` a un support intégré pour la mise en cache de composant. Pour l'activer, vous avez besoin d'une [implémentation de mise en cache](./api.md#cache) quand vous créez le moteur de rendu. L'usage courant est de passer un [objet de mise en cache qui supprime l'objet le plus récemment utilisé](https://github.com/isaacs/node-lru-cache) :

``` js
const LRU = require('lru-cache')

const renderer = createRenderer({
  cache: LRU({
    max: 10000,
    maxAge: ...
  })
})
```

Vous pouvez aussi implémenter une mise en cache de composant avec une fonction `serverCacheKey` :

``` js
export default {
  name: 'item', // requis
  props: ['item'],
  serverCacheKey: props => props.item.id,
  render (h) {
    return h('div', this.item.id)
  }
}
```

Notez que ce composant pouvant être mis en cache **doit aussi définir une option `name` unique**. Avec un nom unique, la clé de mise en cache est effective par composant : vous n'avez donc pas besoin de vous préoccuper de deux composants qui retourneraient la même clé.

La clé retournée par `serverCacheKey` devrait contenir suffisamment d'informations pour représenter tous les résultats de rendu. L'exemple ci-dessus est une bonne implémentation si le résultat du rendu est uniquement déterminé par `props.item.id`. Cependant, si un élément avec le même identifiant change à chaque fois, ou si son rendu est lié au changement d'une autre prop, alors vous devez modifier votre implémentation `getCacheKey` pour prendre en compte ces variables.

Retourner une constante va mener le composant à toujours être en cache, ce qui peut être une bonne chose pour les composants statiques.

### Quand utiliser de la mise en cache de composant ?

Si le moteur de rendu trouve du cache pour un composant durant le rendu, il va directement utiliser ce cache comme résultat pour le sous arbre de composant complet. Cela signifie qu'il ne faut **PAS** mettre en cache un composant quand :

- Il possède des composants enfants liés à l'état global.
- Il possède des composants enfants qui produisent des effets de bord sur le rendu de `context`.

La mise en cache de composant doit donc être utilisée avec soin pour éviter les goulots d'étranglement de performance. Dans la plupart des cas, vous n'en aurez pas besoin et donc pas besoin de mettre en cache les instances de composants. Les types de composants les plus communs pour de la mise en cache sont ceux utilisant de grosses listes `v-for`. Comme ces composants sont généralement alimentés par des collections de base de données, ils peuvent utiliser une simple stratégie de mise en cache : générer leur clé de mise en cache en utilisant leur unique identifiant associé à un timestamp mis à jour :

``` js
serverCacheKey: props => props.item.id + '::' + props.item.last_updated
```
