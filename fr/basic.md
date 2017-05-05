# Utilisation de base

## Installation

``` bash
npm install vue vue-server-renderer --save
```

Nous allons utiliser NPM tout au long de ce guide, n'hésitez pas à utiliser [Yarn](https://yarnpkg.com/en/) à la place.

#### Notes

- Il est recommandé d'utiliser une version 6 et supérieur de Node.js
- `vue-server-renderer` et `vue` doivent utiliser des numéros de version identiques.
- `vue-server-renderer` utilise plusieurs modules Node.js natifs fournis uniquement par Node.js. Nous fournirons une version exécutable qui pourra tourner sur les autres moteurs JavaScript dans le futur.

## Faire le rendu d'une instance de Vue

``` js
// Étape 1 : créer une instance de Vue
const Vue = require('vue')
const app = new Vue({
  template: `<div>Hello World</div>`
})

// Étape 2 : créer un générateur de rendu
const renderer = require('vue-server-renderer').createRenderer()

// Étape 3 : faire le rendu de l'instance en HTML
renderer.renderToString(app, (err, html) => {
  if (err) throw err
  console.log(html)
  // => <div data-server-rendered="true">Hello World</div>
})
```

## Intégration avec un serveur

Il est plus simple d'utiliser le code précédent avec un serveur Node.js, comme par exemple [Express](https://expressjs.com/) :

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
      res.status(500).end('Erreur interne du serveur')
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

## Utiliser un modèle de page

Quand vous faites le rendu d'une application Vue, le générateur de rendu fournit uniquement les balises de votre application. Dans cet exemple, nous allons ajouter de part et d'autre la structure HTML nécessaire à toutes pages.

Le plus simple est de directement fournir un modèle de page lors de la création du générateur de rendu. La plupart du temps, nous allons mettre le modèle de page dans son propre fichier. Par ex. `index.template.html` :

``` html
<!DOCTYPE html>
<html lang="en">
  <head><title>Bonjour</title></head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

Notez que le commentaire `<!--vue-ssr-outlet-->` représente la zone où les balises de votre application vont être injectées.

Nous allons ensuite lire et passer le contenu du fichier au générateur de rendu de Vue :

``` js
const renderer = createRenderer({
  template: require('fs').readFileSync('./index.template.html', 'utf-8')
})

renderer.renderToString(app, (err, html) => {
  console.log(html) // sera la page complète avec le contenu de l'application injecté.
})
```

### Interpolation dans le modèle de page

Le modèle de page supporte également une interpolation simple. Avec le modèle de page suivant :

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

Nous pouvons fournir les données d'interpolation suivantes à travers un « objet de contexte de rendu » en tant que second argument de `renderToString` :

``` js
const context = {
  title: 'Bonjour',
  meta: `
    <meta ...>
    <meta ...>
  `
}

renderer.renderToString(app, context, (err, html) => {
  // le titre de la page sera « Bonjour »
  // avec les balises <meta> injectées
})
```

L'objet `context` peut également être partagé avec l'instance de l'application de Vue, permettant aux composants de dynamiquement fournir des données pour l'interpolation du modèle de page.

De plus, le modèle de page supporte des fonctionnalités avancées comme :

- l'injection automatique de CSS critique lors de l'utilisation de composants `*.vue`,
- l'injection automatique de balises `<link>` avec l'utilisation de `clientManifest`,
- l'injection automatique de l'état de Vuex pour l'hydratation cliente avec prévention XSS.

Nous discuterons de cela quand nous introduirons ces concepts plus tard dans le guide.
