# Envoi par flux

`vue-server-renderer` supporte nativement le rendu par flux (« stream »), aussi bien pour le moteur de rendu de base que pour le moteur de rendu de paquetage. Tout ce dont vous avez besoin est d'utiliser `renderToStream` à la place de `renderToString` :
:

``` js
const stream = renderer.renderToStream(context)
```

La valeur retournée est un [flux Node.js](https://nodejs.org/api/stream.html) :

``` js
let html = ''

stream.on('data', data => {
  html += data.toString()
})

stream.on('end', () => {
  console.log(html) // render complete
})

stream.on('error', err => {
  // handle error...
})
```

## Limitations de flux

En mode de rendu par flux, la donnée est émise aussitôt que possible quand le moteur parcourt l'arbre du DOM virtuel. Cela signifie que l'envoi au client du « premier fragment » commence rapidement.

Cependant, quand le premier fragment est émis, les composants enfants peuvent ne pas avoir encore été instanciés, et les hooks de leur cycle de vie ne seront jamais appelés. Cela signifie que si des composants enfants ont besoin d'attacher leurs données dans le contexte de rendu de leurs hooks de cycle de vie, elles ne seront pas accessibles au démarrage du flux. Comme beaucoup d'informations (comme les informations d'entête ou les CSS critiques injectées) ont besoin d'être insérées avant la balise de l'application, il est nécessaire d'attendre la fin du flux avant de commencer à utiliser ces données de contexte.

Il n'est donc **PAS** recommandé d'utiliser de l'envoi par flux si les données de vos contextes sont injectés dans les hooks du cycle de vie.
