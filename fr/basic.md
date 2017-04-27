# Utilisation basique

## Installation

``` bash
npm install vue vue-server-renderer --save
```

Nous utiliserons NPM durant ce guide, mais vous pouvez utiliser [Yarn](https://yarnpkg.com/en/) à la place.

#### Notes

- Il est recommandé d'utiliser la version 6+ de Node.js.
- Les versions de `vue-server-renderer` et `vue` doivent correspondre.
- `vue-server-renderer` utilise certains modules natifs de Node.js, et par conséquent ne peut être utilisé qu'avec Node.js. Il se peut que nous fournissons une *build* plus simple qui pourra être utilisé dans d'autres environnements d'exécution JavaScript, dans le futur. 

## Rendu d'une instance de Vue

``` js
// Étape 1 : Créer une instance de Vue
const Vue = require('vue')
const app = new Vue({
  template: `<div>Bonjour le monde</div>`
})

// Étape 2 : Créer un moteur de rendu
const renderer = require('vue-server-renderer').createRenderer()

// Étape 3 : Faire le rendu de l'instance de Vue en HTML
renderer.renderToString(app, (err, html) => {
  if (err) throw err
  console.log(html)
  // => <p data-server-rendered="true">Bonjour le monde</p>
})
```

## Integration avec un serveur

L'utilisation avec un serveur Node.js, par exemple [Express](https://expressjs.com/), est plutôt simple :

``` bash
npm install express --save
```
---
``` js
const Vue = require('vue')
const server = require('express')()
const renderer = require('vue-server-renderer').createRenderer()

server.get('*', (req, res) => {
  const app = new Vue({
    data: {
      url: req.url
    },
    template: `<div>L'URL visitée est : {{ url }}</div>`
  })

  renderer.renderToString(app, (err, html) => {
    if (err) {
      res.status(500).end('Internal Server Error')
      return
    }
    res.end(`
      <!DOCTYPE html>
      <html lang="en">
        <head><title>Bonjour</title></head>
        <body>${html}</body>
      </html>
    `)
  })
})

server.listen(8080)
```

## Utilisation d'un modèle de page

Pendant le rendu de l'application Vue, le moteur de rendu ne génère que le code HTML de l'application.
Dans l'exemple précédent, il a fallu entourer le résultat par du code HTML supplémentaire.

Pour simplifier cela, il est possible de fournir un modèle de page pendant la création moteur de rendu.
La plupart du temps, ce modèle de page sera dans situé dans son propre fichier, ex : `index.template.html` :

``` html
<!DOCTYPE html>
<html lang="fr">
  <head><title>Bonjour</title></head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

Observez le commentaire `<!--vue-ssr-outlet-->` -- c'est là que le code HTML de l'application sera injecté.

On peut maintenant lire et passer le fichier dans le moteur de rendu de Vue :

``` js
const renderer = createRenderer({
  template: require('fs').readFileSync('./index.template.html', 'utf-8')
})

renderer.renderToString(app, (err, html) => {
  console.log(html) // sera la page entière avec le contenu de application injecté.
})
```

### Interpolation de modèle de page

Le modèle de page supporte également des interpolations simple. Avec le modèle de page suivant :

``` html
<html>
  <head>
    <title>{{ title }}</title>
    {{{ meta }}}
  </head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

On peut utiliser l'interpolation de données en passant un "objet contexte pour le rendu" comme deuxième argument de  `renderToString` :

``` js
const context = {
  title: 'bonjour',
  meta: `
    <meta ...>
    <meta ...>
  `
}

renderer.renderToString(app, context, (err, html) => {
  // le titre de la page sera "bonjour"
  // avec les tags meta injectés
})
```

L'objet `context` peut aussi être partagé avec l'instance de l'application Vue, permettant les composants de modifier ces données de manière dynamique, pour l'interpolation de modèle de page. 

De plus, le modèle de page supporte quelques fonctionnalités avancées telles que :

- Injection automatique de CSS *critique* lors de l'utilisation `*.vue` components ;
- Injection automatique des ressources et suggestions lors de l'utilisation de `clientManifest` ;
- Injection automatique et protection XSS durant l'incorporation de données Vuex, pour l'hydratation côté client.


Nous en discuterons lorsque nous présenterons les concepts associés plus tard dans le guide.
