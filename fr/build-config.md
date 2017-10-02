# Configuration de précompilation

Nous allons supposez que vous savez déjà comment configurer webpack pour un projet uniquement client. La configuration pour un projet avec du SSR va être en grande partie similaire, mais nous vous suggérons de séparer vos configurations en trois fichiers : *base*, *client* et *server*. La configuration de base contient la configuration partagée par les deux environnements, comme les chemins de sortie, les aliases et les loaders. La configuration du serveur et la configuration du client peuvent simplement étendre la configuration de base en utilisant [webpack-merge](https://github.com/survivejs/webpack-merge).

## Configuration serveur

La configuration serveur est destinée à générer le paquetage serveur qui va être passé à `createBundleRenderer`. Elle devrait ressembler à cela :

``` js
const merge = require('webpack-merge')
const nodeExternals = require('webpack-node-externals')
const baseConfig = require('./webpack.base.config.js')
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')

module.exports = merge(baseConfig, {
  // Fichier d'entrée serveur de l'application
  entry: '/path/to/entry-server.js',

  // Cela permet à webpack de gérer les imports dynamiques d'une manière
  // approprié pour Node.js, et dit également à `vue-loader` d'émettre un code approprié pour le serveur
  // lors de la compilation du composant Vue.
  target: 'node',

  // Pour le support des sources maps des paquetages
  devtool: 'source-map',

  // Cela dit au paquetage serveur d'utiliser les exports au format Node.js
  output: {
    libraryTarget: 'commonjs2'
  },

  // https://webpack.js.org/configuration/externals/#function
  // https://github.com/liady/webpack-node-externals
  // Externalise les dépendances de l'application. Cela rend le build serveur plus rapide
  // et génère un fichier de paquetage plus petit.
  externals: nodeExternals({
    // ne pas externaliser les dépendances qui ont besoin d'être traitées par webpack.
    // vous pouvez ajouter plus de types de fichiers ici, comme par ex. avec les fichiers `*.vue`
    // vous devriez aussi lister des exceptions qui modifient `global` (par ex. les polyfills)
    whitelist: /\.css$/
  }),

  // Ceci est le plugin qui va créer entièrement la sortie pour le build serveur
  // dans un seul fichier JSON. Le fichier généré par défaut va être
  // `vue-ssr-server-bundle.json`
  plugins: [
    new VueSSRServerPlugin()
  ]
})
```

Après que `vue-ssr-server-bundle.json` ai été généré, passez simplement le chemin du fichier à `createBundleRenderer` :

``` js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer('/path/to/vue-ssr-server-bundle.json', {
  // ...autres options pour le moteur
})
```

Vous pouvez alternativement tout aussi bien passer le paquetage comme un objet à `createBundleRenderer`. Cela est utile pour le rechargement à chaud pendant le développement. Voyez la démo de HackerNews pour une [référence de mise en place](https://github.com/vuejs/vue-hackernews-2.0/blob/master/build/setup-dev-server.js).

### Limitations externes

Notons que dans l'option `externals` nous avons exclu les fichiers CSS. C'est parce que les fichiers CSS importés par dépendances doivent quand même être gérés par webpack. Si vous importez n'importe quels autres types de fichiers également pris en charge par webpack (ex : `*.vue`, `*.styl`), vous pouvez également les ajouter à la liste des exceptions.

Si vous utilisez `runInNewContext: 'once'` ou `runInNewContext: true`, alors vous devrez également ajouter aux exceptions les polyfills qui modifient `global` comme par ex. `babel-polyfill`. Cela est dû au fait qu'en utilisant un nouveau mode de contexte, **le code à l'intérieur d'un paquetage serveur a son propre objet `global`.** Parce qu'il n'est plus nécessaire de faire cela côté serveur en utilisant Node.js 7.6+, c'est d'autant plus facile de ne les importer que côté client.

## Configuration cliente

La configuration cliente peut être en grande partie la même grâce à la configuration de base. Bien sûr vous devez faire pointer `entry` sur votre fichier d'entrée client. En plus de cela, si vous utilisez le plugin `CommonsChunkPlugin`, assurez-vous de ne l'utiliser que dans la configuration cliente car le paquetage serveur requiert un unique fragment d'entrée.

### Générer le `clientManifest`

> requiert la version 2.3.0+

En plus du paquetage serveur, nous pouvons également générer un build de manifeste client. Avec le manifeste client et le paquetage serveur, le moteur a maintenant les informations du build serveur *et* du build client, ainsi il peut automatiquement déduire et injecter les [directives préchargées et récupérées](https://css-tricks.com/prefetching-preloading-prebrowsing/) ainsi que les balises `<link>` / `<script>` dans le rendu HTML.

Les bénéfices sont doubles :

1. Il peut remplacer le plugin `html-webpack-plugin` pour l'injection correcte d'URL de fichiers quand il y a des hashs dans les noms de fichier générés.

2. Lors du rendu d'un paquetage qui s'appuie sur les fonctionnalités de scission de code à la demande de webpack, nous pouvons être assurés que les fragments optimaux sont préchargés / récupérés, et que les balises `<script>` des fragments asynchrones nécessaires pour éviter la cascade de requête depuis le client sont intelligemment injectées. Cela améliore le TTI (« time-to-interactive »).

Pour tirer parti du manifeste client, la configuration cliente devrait ressembler à ça :

``` js
const webpack = require('webpack')
const merge = require('webpack-merge')
const baseConfig = require('./webpack.base.config.js')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')

module.exports = merge(baseConfig, {
  entry: '/path/to/entry-client.js',
  plugins: [
    // Important : cela scinde l'exécution de webpack en un fragment maitre
    // et des fragments asynchrones qui peuvent être injectés juste après lui.
    // cela permet également une meilleure mise en cache pour vos codes d'applications tierces.
    new webpack.optimize.CommonsChunkPlugin({
      name: "manifest",
      minChunks: Infinity
    }),
    // Ce plugin génère le fichier `vue-ssr-client-manifest.json` dans
    // le dossier de sortie.
    new VueSSRClientPlugin()
  ]
})
```

Vous pouvez même utiliser le manifeste client généré conjointement avec un template de page :

``` js
const { createBundleRenderer } = require('vue-server-renderer')

const template = require('fs').readFileSync('/path/to/template.html', 'utf-8')
const serverBundle = require('/path/to/vue-ssr-server-bundle.json')
const clientManifest = require('/path/to/vue-ssr-client-manifest.json')

const renderer = createBundleRenderer(serverBundle, {
  template,
  clientManifest
})
```

Avec cette mise en place, votre rendu HTML côté serveur pour un build avec scission de code va ressembler à quelque chose comme ci-dessous (tout étant auto-injecté) :

``` html
<html>
  <head>
    <!-- les fragments utilisés pour ce rendu vont être préchargés -->
    <link rel="preload" href="/manifest.js" as="script">
    <link rel="preload" href="/main.js" as="script">
    <link rel="preload" href="/0.js" as="script">
    <!-- les fragments asynchrones non utilisés vont seulement être récupérés (priorité basse) -->
    <link rel="prefetch" href="/1.js" as="script">
  </head>
  <body>
    <!-- contenu de l'application -->
    <div data-server-rendered="true"><div>async</div></div>
    <!-- le fragment du manifeste devrait être le premier -->
    <script src="/manifest.js"></script>
    <!-- les fragments asynchrones sont injectés avant le fragment principal -->
    <script src="/0.js"></script>
    <script src="/main.js"></script>
  </body>
</html>`
```

### Injection manuelle des fichiers

Par défaut, l'injection des fichiers est automatique quand vous fournissez l'option de rendu `template`. Mais parfois vous aurez besoin d'une granularité de contrôle plus fine en ce qui concerne la manière dont les templates seront injectés, ou peut-être que vous n'utiliserez pas de template du tout. Dans tous les cas, vous pouvez passer `inject: false` quand le moteur est créé et manuellement réaliser l'injection des fichiers.

Dans la fonction de rappel de `renderToString`, l'objet `context` que vous passez va exposer les méthodes suivantes :

- `context.renderStyles()`

  Cela va retourner une balise `<style>` contenant tout le CSS critique récupéré dans les composants `*.vue` et utilisés durant le rendu. Consultez [Gestion des CSS](./css.md) pour plus de détails.

  Si un `clientManifest` est fourni, la chaine retournée va également contenir la balise `<link rel="stylesheet">` pour les fichiers CSS émis par webpack (ex: CSS extrait avec `extract-text-webpack-plugin` ou importés avec `file-loader`)

- `context.renderState(options?: Object)`

  Cette méthode sérialise `context.state` et retourne une balise script qui contient l'état avec `window.__INITIAL_STATE__`.

  La clé et la valeur de l'état peuvent être toutes les deux passées dans l'objet d'option :

  ``` js
  context.renderState({
    contextKey: 'myCustomState',
    windowKey: '__MY_STATE__'
  })

  // -> <script>window.__MY_STATE__={...}</script>
  ```

- `context.renderScripts()`

  - requiert `clientManifest`

  Cette méthode retourne la balise `<script>` nécessaire pour que l'application cliente puisse démarrer. Lors de l'utilisation de la scission de code asynchrone dans le code de l'application, cette méthode va intelligemment trouver les fragments asynchrones corrects à inclure.

- `context.renderResourceHints()`

  - requiert `clientManifest`

  Cette méthode retourne les balises `<link rel="preload/prefetch">` nécessaires au rendu optimisé de la page. Par défaut ce sera :

  - Préchargement (récupération et exécution) des fichiers JavaScript et CSS requis par la page,
  - Récupération asynchrone des fragments JavaScript qui seront nécessaires plus tard.

  Les fichiers préchargés peuvent être personnalisés plus en profondeur avec l'option [`shouldPreload`](./api.md#shouldpreload).

- `context.getPreloadFiles()`

  - requiert `clientManifest`

  Cette méthode ne retourne pas de chaine de caractère. À la place elle retourne un tableau d'objets représentant les fichiers qui devraient être préchargés. Cela peut-être utilisé pour programmatiquement réaliser de l'augmentation serveur HTTP/2.

Puisque le `template` passé à `createBundleRenderer` va être interpolé en utilisant le `context`, vous pouvez utiliser ces méthodes à l’intérieur de celui-ci (with `inject: false`) :

``` html
<html>
  <head>
    <!-- utiliser les triples moustaches pour une interpolation sans HTML -->
    {{{ renderResourceHints() }}}
    {{{ renderStyles() }}}
  </head>
  <body>
    <!--vue-ssr-outlet-->
    {{{ renderState() }}}
    {{{ renderScripts() }}}
  </body>
</html>
```

Si vous n'utilisez pas `template` du tout, vous pouvez concaténer les chaines vous-mêmes.
