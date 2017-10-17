# Introduction au moteur de dépaquetage

## Problèmes basiques du SSR

À ce point, nous supposons que le code empaqueté côté serveur sera directement utilisé via `require` :

``` js
const createApp = require('/path/to/built-server-bundle.js')
```

Même si c'est simple, à chaque fois que vous éditez votre code source à ce stade, vous devez stopper et redémarrer votre serveur. Cela ralentit quelque peu la productivité pendant le développement. De plus, Node.js ne supporte pas les sources maps nativement.

## Le moteur de dépaquetage

`vue-server-renderer` fournit une API appelée `createBundleRenderer` pour résoudre ce problème. Avec un plugin webpack personnalisé, le paquetage (« bundle ») serveur est généré comme un fichier JSON spécial qui peut être passé au moteur de dépaquetage (« bundle renderer »). Une fois que le moteur de dépaquetage est créé, l'usage est le même qu'un moteur de rendu, cependant le moteur de dépaquetage fournit les bénéfices suivants :

- Support des sources maps inclus (avec `devtool: 'source-map'` dans la configuration de webpack)

- Rechargement à chaud pendant la phase de développement et même de déploiement (en relisant le paquetage mis à jour et en recréant l'instance du moteur)

- Injection CSS critique (en utilisant les fichiers `*.vue`) : insérer automatiquement dans la sortie le CSS nécessaire pour les composants pendant le rendu. Voir la section [CSS](./css.md) pour plus de détails.

- Injection de fragments avec [clientManifest](./api.md#clientmanifest) : déduire automatiquement le préchargement et la récupération des directives, et les fragments scindés requis pour le rendu initial.

---

Nous allons discuter de la manière de configurer webpack pour générer les artéfacts de build nécessaires au moteur de dépaquetage dans la prochaine section, mais pour le moment, imaginons que nous ayons déjà ce dont nous avons besoin. Voici comment créer et utiliser un moteur de dépaquetage :

``` js
const { createBundleRenderer } = require('vue-server-renderer')

const renderer = createBundleRenderer(serverBundle, {
  runInNewContext: false, // recommandé
  template, // (optionnel) page de template
  clientManifest // (optionnel) manifeste de build client
})

// à l'intérieur du gestionnaire serveur...
server.get('*', (req, res) => {
  const context = { url: req.url }
  // Pas besoin de passer l'application ici car elle est automatiquement créée
  // à l'exécution du paquetage. Maintenant notre serveur est découplé de notre application Vue !
  renderer.renderToString(context, (err, html) => {
    // gérer les erreurs...
    res.end(html)
  })
})
```

Quand `renderToString` est appelé sur le moteur de dépaquetage, il va automatiquement exécuté la fonction exportée par le paquetage pour créer une instance de l'application (en passant `context` comme argument) puis va en faire le rendu.

Notons qu'il est recommandé de mettre l'option `runInNewContext` à `false` ou à `'once'`. Plus de détails dans [la référence de l'API](./api.md#runinnewcontext).
