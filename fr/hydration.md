# Hydratation côté client

Dans `entry-client.js`, nous montons simplement l'application avec cette ligne :

``` js
// supposons que l'élément racine du template App.vue possède `id="app"`
app.$mount('#app')
```

Parceque le serveur a déjà fait le rendu des balises, nous ne voulons évidemment pas tout jeter et recréer l'intégralité des éléments du DOM. À la place, nous voulons « hydrater » les balises statiques et les rendre intéractives.

Si vous inspectez le rendu en sortie côté serveur, vous remarquerez que l'élément racine a un attribut spécial :

``` js
<div id="app" data-server-rendered="true">
```

L'attribut spécial `data-server-rendered` permet à Vue, depuis le côté client, de savoir quelle balise a été rendue par le serveur et d'être capable de monter l'application en mode hydratation.

En mode développement, Vue va vérifier que le DOM virtuel généré côté client concorde avec la structure du DOM rendu par le serveur. S'il y a non concordance, il va bypasser l'hydratation, retirer le DOM existant et refaire le rendu depuis le début. **En mode production, ces vérifications sont désactivées pour des performances maximales.**

### Limitation de l'hydration

Une chose qu'il faut savoir est qu'en utilisant un SSR + une hydratation côté client il y a plusieurs structures HTML spéciales qui sont altérées par le navigateur. Par exemple, quand vous écrivez ceci dans un template Vue :

``` html
<table>
  <tr><td>salut</td></tr>
</table>
```

Le navigateur va automatiquement injecter `<tbody>` dans `<table>`. Le DOM virtuel généré par Vue ne va cependant pas contenir `<tbody>`, ce qui va causer une non concordance. Pour assurer une concordance correcte, assurez-vous d'écrire du HTML valide dans vos templates.
