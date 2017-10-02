# Référence de l'API

## `createRenderer([options])`

Crée une instance de [`Renderer`](#class-renderer) avec des [options](#renderer-options) optionnelles.

``` js
const { createRenderer } = require('vue-server-renderer')
const renderer = createRenderer({ ... })
```

## `createBundleRenderer(bundle[, options])`

Crée une instance de [`BundleRenderer`](#class-bundlerenderer) avec un paquetage serveur et des [options](#renderer-options) optionnelles.

``` js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer(serverBundle, { ... })
```

L'argument `serverBundle` peut être l'un des éléments suivants :

- Un chemin absolu pour générer un fichier de paquetage (`.js` ou `.json`). Doit commencer par `/` pour être considéré comme un chemin de fichier.

- Un objet de paquetage généré par webpack + `vue-server-renderer/server-plugin`.

- Une chaine de caractères de code JavaScript (non recommandé).

Voyez l'[Introduction au moteur de dépaquetage](./bundle-renderer.md) et la [Configuration de précompilation](./build-config.md) pour plus de détails.

## `Class: Renderer`

- #### `renderer.renderToString(vm[, context], callback)`

  Fait le rendu d'une instance de Vue sous forme de chaine de caractères. L'objet de contexte est optionnel. La fonction de rappel est une fonction de rappel typique de Node.js avec en premier argument l'erreur potentielle et en second argument la chaine de caractères du rendu.

- #### `renderer.renderToStream(vm[, context])`

  Fait le rendu d'une instance de Vue sous forme de flux. L'objet de contexte est optionnel. Voir l'[Envoi par flux](./streaming.md) pour plus de détails.

## `Class: BundleRenderer`

- #### `bundleRenderer.renderToString([context, ]callback)`

  Fait le rendu d'un paquetage sous forme de chaine de caractères. L'objet de contexte est optionnel. La fonction de rappel est une fonction de rappel typique de Node.js avec en premier argument l'erreur potentielle et en second argument la chaine de caractères du rendu.

- #### `bundleRenderer.renderToStream([context])`

  Fait le rendu d'un paquetage sous forme de flux. L'objet de contexte est optionnel. Voir l'[Envoi par flux](./streaming.md) pour plus de détails.

## Options de `Renderer`

- #### `template`

  Fournit un modèle de page pour la page HTML complète. Le modèle de page devrait contenir en commentaire `<!--vue-ssr-outlet-->` qui permet de définir l'emplacement du contenu de chaque rendu de l'application.

  Le modèle de page supporte également l'interpolation basique en utilisant le contexte du rendu :

  - utilisez les doubles moustaches pour de l'interpolation avec HTML échappé et
  - utilisez les triples moustaches pour de l'interpolation avec HTML non échappé.

  Le modèle de page injecte automatiquement le contenu quand certaines données sont trouvées dans le contexte du rendu :

  - `context.head`: (string) n'importe quelle balise d'entête qui devrait être injectée dans la balise `<head>` de la page.

  - `context.styles`: (string) n'importe quelle CSS qui devrait être injectée dans la balise `<head>` de la page. Notez que cette propriété va automatiquement être injectée si vous utilisez `vue-loader` + `vue-style-loader` pour la CSS de vos composants.

  - `context.state`: (Object) L'état initial du store Vuex devrait être injecté dans la page sous la variable `window.__INITIAL_STATE__`. Le JSON en ligne est automatiquement désinfecté avec [serialize-javascript](https://github.com/yahoo/serialize-javascript) pour éviter les injections XSS.

  En plus, quand `clientManifest` est fourni, le modèle de page injecte automatiquement les éléments suivants :

  - JavaScript client et fichiers CSS nécessaires pour le rendu (avec les fragments asynchrones automatiquement déduits),
  - utilisation optimale des indices de ressources `<link rel="preload/prefetch">` pour le rendu de la page.

  Vous pouvez désactiver toutes ces injections en passant `inject: false` au moteur de rendu.

  Voir également :

  - [Utiliser un modèle de page](./basic.md#utiliser-un-modele-de-page)
  - [Injection manuelle des fichiers](./build-config.md#injection-manuelle-des-fichiers)

- #### `clientManifest`

  - 2.3.0+

  Fournit un objet de build de manifeste généré par `vue-server-renderer/client-plugin`. Le manifeste client fournit le paquetage de moteur de rendu avec ses propres informations pour l'injection automatique de fichiers dans le modèle de page HTML. Pour plus de détails, consultez [Générer le `clientManifest`](./build-config.md#generer-le-clientmanifest).

- #### `inject`

  - 2.3.0+

  Contrôle la manière d'exécuter des injections automatiques en utilisant `template`. Par défaut à `true`.

  Voir aussi : [Injection manuelle des fichiers](./build-config.md#injection-manuelle-des-fichiers).

- #### `shouldPreload`

  - 2.3.0+

  Une fonction pour contrôler quels fichiers doivent avoir une ressource d'indice `<link rel="preload">` de générée.

  Par défaut, seuls les fichiers JavaScript et les fichiers CSS seront préchargés, car ils sont absolument nécessaires pour le démarrage de l'application.

  Pour les autres types de fichiers comme les images et les polices, le préchargement pouvant gâcher de la bande passante inutilement et même baisser les performances, cela est laissé à votre appréciation. Vous pouvez contrôler précisément le préchargement en utilisant l'option `shouldPreload` :

  ``` js
  const renderer = createBundleRenderer(bundle, {
    template,
    clientManifest,
    shouldPreload: (file, type) => {
      // le type est déduit en se basant sur l'extension du fichier.
      // https://fetch.spec.whatwg.org/#concept-request-destination
      if (type === 'script' || type === 'style') {
        return true
      }
      if (type === 'font') {
        // précharger uniquement les polices woff2
        return /\.woff2$/.test(file)
      }
      if (type === 'image') {
        // charger uniquement les images importantes
        return file === 'hero.jpg'
      }
    }
  })
  ```

- #### `runInNewContext`

  - 2.3.0+
  - seulement utilisée avec `createBundleRenderer`
  - Requiert : `boolean | 'once'` (`'once'` est seulement supporté dans la 2.3.1+)

  Par défaut, pour chaque rendu, le moteur de dépaquetage va créer un nouveau contexte V8 et réexécuter le paquetage complet. Cela a plusieurs bénéfices, par exemple, isoler le code de l'application des processus du serveur ce qui permet d'[Éviter les singletons d'état](./structure.md#eviter-les-singletons-detat) mentionnés dans la documentation. Cependant, ce mode a des couts de performance importants car réexécuter le paquetage est quelque chose de couteux, surtout quand l'application est grosse.

  Cette option est par défaut à `true` pour la rétrocompatibilité, mais il est recommandé d'utiliser `runInNewContext: false` ou `runInNewContext: 'once'` si vous le pouvez.

  > Dans la 2.3.0 cette option a un bogue car `runInNewContext: false` exécute toujours le paquetage en utilisant un contexte global séparé. Les informations suivantes sont donc valables pour la version 2.3.1+.

  Avec `runInNewContext: false`, le code de paquetage va tourner dans le même contexte `global` du processus serveur, donc faites attention au code qui modifie `global` dans le code de votre application.

  Avec `runInNewContext: 'once'` (2.3.1+), le paquetage est évalué dans un contexte `global` séparé, cependant cela n'est effectué qu'au démarrage. Cela permet une meilleure isolation du code de l'application puisqu'il empêche le paquetage d'accidentellement polluer l'objet `global` du processus serveur. Les limitations sont les suivantes :

  1. Les dépendances qui modifient l'objet `global` (ex. polyfills) ne peuvent être externalisées dans ce mode,
  2. Les valeurs retournées lors de l'exécution du paquetage utiliseront des constructeurs globaux différents. Par ex. une erreur levée à l'intérieur du paquetage ne sera pas une instance de `Error` dans le processus serveur.

  Voir aussi : [Structure de code](./structure.md)

- #### `basedir`

  - 2.2.0+
  - seulement utilisée avec `createBundleRenderer`

  Déclarer explicitement le dossier de base du paquetage serveur afin de résoudre les dépendances `node_modules`. Cela est nécessaire si le fichier de paquetage généré est placé à un endroit différent de là où les dépendances externes npm sont installées, ou si `vue-server-renderer` est lié à npm dans votre projet courant.

- #### `cache`

  Fournit une implémentation de [Mise en cache au niveau du composant](./caching.md#mise-en-cache-au-niveau-du-composant). L'objet de cache doit implémenter l'interface suivante (utilisation des notations Flow) :

  ``` js
  type RenderCache = {
    get: (key: string, cb?: Function) => string | void;
    set: (key: string, val: string) => void;
    has?: (key: string, cb?: Function) => boolean | void;
  };
  ```

  Une utilisation typique est de passer un [objet de mise en cache qui supprime l'objet le plus récemment utilisé](https://github.com/isaacs/node-lru-cache) :

  ``` js
  const LRU = require('lru-cache')

  const renderer = createRenderer({
    cache: LRU({
      max: 10000
    })
  })
  ```

  Notez que cet objet de mise en cache doit au moins implémenter `get` et `set`. De plus `get` et `set` peuvent être optionnellement asynchrones s'ils acceptent en second argument une fonction de rappel. Cela permet à la mise en cache d'utiliser des APIs. Par ex. un client Redis :

  ``` js
  const renderer = createRenderer({
    cache: {
      get: (key, cb) => {
        redisClient.get(key, (err, res) => {
          // gérer les erreurs s'il y en a
          cb(res)
        })
      },
      set: (key, val) => {
        redisClient.set(key, val)
      }
    }
  })
  ```

- #### `directives`

  Vous permet de fournir des implémentations côté serveur pour vos directives personnalisées :

  ``` js
  const renderer = createRenderer({
    directives: {
      example (vnode, directiveMeta) {
        // transformer les vnode en directive de liaison de metadata
      }
    }
  })
  ```

  Consultez l'[implémentation de `v-show` côté serveur](https://github.com/vuejs/vue/blob/dev/src/platforms/web/server/directives/show.js) en tant qu'exemple.

## Plugins webpack

Les pluging webpack sont fournis en fichiers autonomes et devraient être requis en direct :

``` js
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')
```

Les fichiers générés par défaut sont :

- `vue-ssr-server-bundle.json` pour le plugin serveur,
- `vue-ssr-client-manifest.json` pour le plugin client.

Les noms de fichiers peuvent être personnalisés lors de la création des instances des plugins :

``` js
const plugin = new VueSSRServerPlugin({
  filename: 'my-server-bundle.json'
})
```

Voir la [Configuration de précompilation](./build-config.md) pour plus d'informations.
