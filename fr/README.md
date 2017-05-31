# Guide du rendu côté serveur avec Vue.js

> **Note :** Ce guide nécessite les versions de Vue et de ses bibliothèques de support suivantes :
> - vue & vue-server-renderer >= 2.3.0
> - vue-router >= 2.5.0
> - vue-loader >= 12.0.0 & vue-style-loader >= 3.0.0

> Si vous avez déjà utilisé le rendu côté serveur (SSR pour *Server-Side Rendering*) avec Vue 2.2, vous vous apercevrez que la structure de code recommandée est maintenant [légèrement différente](./structure.md) (avec la nouvelle option [runInNewContext](./api.md#runinnewcontext) mise à `false`). Votre application devrait continuer à fonctionner, mais il est recommandé de migrer vers les nouvelles recommandations.

## Le rendu côté serveur ou SSR, qu'est-ce que c'est ?

Vue.js est un framework pour créer des applications côté client. Par défaut, Vue génère en sortie un DOM manipulable dans le navigateur. Il est cependant également possible de faire le rendu des mêmes composants sous forme de chaîne de caractères HTML côté serveur, de les envoyer directement au navigateur et d'« hydrater » les balises statiques fournies en une application cliente pleinement interactive.

Une application Vue.js rendue du côté serveur peut également être considérée comme « isomorphique » ou « universelle », dans le sens où la majorité du code est exécutable côté serveur **et** coté client.

## Pourquoi faire du SSR ?

En comparaison des applications monopages traditionnelles (SPA pour *Single-Page Application*), l'avantage du SSR se manifeste dans :

- De meilleures optimisations pour les moteurs de recherche (SEO pour *Search Engine Optimisations*), ainsi les moteurs d'indexation voient directement le rendu complet de la page.

  À noter qu'à présent, Google et Bing savent parfaitement indexer des applications JavaScript synchrones. Synchrone est le mot important ici. Si votre application débute avec une animation de chargement, puis va chercher le contenu via Ajax, l'indexeur n'attendra pas que cette action soit finie. Cela signifie que si vous avez du contenu asynchrone injecté sur des pages ou la SEO est importante, du SSR serait nécessaire.

- De meilleurs temps d'accès au contenu, en particulier pour les connexions Internet lentes ou les appareils lents. Le rendu des balises côté serveur n'a pas besoin d'attendre le chargement de tous les fichiers JavaScript pour que le code soit exécuté en vue d'être affiché. Ainsi votre utilisateur verra apparaître une page complètement rendue très tôt. Cela conduit généralement à une meilleure expérience utilisateur, ce qui peut-être critique pour les applications où le temps d'accès au contenu est directement lié au taux de conversion.

Il y a aussi des contraintes à prendre en considération quand on utilise du SSR :

- Des contraintes de développement. Le code spécifique aux navigateurs ne peut être utilisé que dans certains hooks ; plusieurs bibliothèques nécessitent une utilisation particulière pour être capable d'être exécutées dans une application côté serveur.

- Plus d'étapes de pré-compilation et de déploiement requises. Contrairement à une SPA qui peut être déployée sur un serveur de fichiers statiques, une application rendue côté serveur nécessite un environnement où un serveur Node.js peut tourner.

- Plus de charge côté serveur. Faire le rendu d'une application complète en Node.js est évidemment une tâche demandant plus de ressources CPU que de simplement renvoyer des fichiers statiques. Aussi si vous vous attendez à un fort trafic, préparez-vous un serveur tenant la charge et utilisez astucieusement des stratégies de mise en cache.

Avant d'utiliser du SSR pour vos applications, la première question que vous devriez vous poser est si vous en avez réellement besoin. Cela dépendra de l'importance du temps d'accès au contenu pour votre application. Par exemple, si vous créez une interface d'administration avec un chargement initial de quelques secondes, cela n'a pas d'importance ; du SSR n'aurait pas de pertinence dans ce cas. Cependant, dans le cas où l'accès au contenu est une priorité absolue, du SSR peut vous aider à obtenir les meilleures performances de chargement initial.

## Rendu côté serveur vs. pré-rendu

Si vous envisagez d'utiliser du SSR seulement pour améliorer votre SEO sur des pages informatives à valeur ajoutée (par ex. `/`, `/about`, `/contact`, etc), alors vous devriez plutôt utiliser du pré-rendu. Plutôt que d'utiliser un serveur web pour compiler le HTML en temps réel, faites le simple pré-rendu de vos pages statiques en HTML pour des routes bien spécifiques lors d'une phase de pré-compilation. L'avantage est que faire du pré-rendu est plus simple et vous permet de garder un site avec une partie cliente statique.

Si vous utilisez webpack, vous pouvez facilement ajouter du pré-rendu avec le plugin [prerender-spa-plugin](https://github.com/chrisvfritz/prerender-spa-plugin). Il a particulièrement bien été testé avec des applications Vue (en fait, [son créateur](https://github.com/chrisvfritz) est lui-même un membre de l'équipe principale de Vue).

## À propos de ce guide

Ce guide est dédié au rendu côté serveur des applications monopages utilisant Node.js en tant que serveur web. Utiliser le SSR de Vue avec une autre configuration serveur est un sujet qui ne sera pas abordé dans ce guide.

Ce guide va réellement entrer dans le détail et présuppose que vous êtes déjà familiarisé avec Vue.js et que vous avez un niveau de connaissance correcte concernant Node.js et webpack. Si vous préférez une solution de haut niveau fournissant une expérience de développement prête à l'emploi, vous devriez probablement essayer [Nuxt.js](http://nuxtjs.org/). Il est construit par-dessus l'écosystème de Vue et vous fourni des éléments préconçus ainsi que des fonctionnalités supplémentaires pour générer des sites web statiques. Il ne vous conviendra cependant pas si vous souhaitez avoir un contrôle plus direct sur la structure de votre application. Dans tous les cas, il reste toujours intéressant de parcourir ce guide pour mieux comprendre comment chaque élément fonctionne avec le reste.

Au fil de votre lecture, il peut être intéressant également de vous référer à la [démo HackerNews](https://github.com/vuejs/vue-hackernews-2.0/) qui utilise bon nombre des techniques expliquées dans ce guide.

Pour finir, notez que les solutions de ce guide ne sont pas définitives. Nous avons trouvé que cela fonctionnait bien pour nous, mais cela ne veut pas dire qu'il n'y a pas d'améliorations à faire. Nous pourrons les réviser dans le futur. N'hésitez donc pas à y contribuer en soumettant des pull requests !
