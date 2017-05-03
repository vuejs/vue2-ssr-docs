# Guide du rendu côté serveur de Vue.js

> **Note** : ce guide nécessite les versions minimales de Vue et des librairies :
>
> * vue & vue-server-renderer &gt;= 2.3.0
> * vue-router &gt;= 2.5.0
> * vue-loader &gt;= 12.0.0 & vue-style-loader &gt;= 3.0.0
>
> Si vous avez déjà utilisé Vue 2.2 avec le SSR, vous remarquerez que la structure de code recommandé est désormais [un peu différente](./structure.md) (avec l'option [unInNewContext](./api.md#runinnewcontext) ayant pour valeur `false`). Votre application existante devrait continuer de fonctionner, mais il est recommander de migrer vers ces nouvelles recommandations.

## Qu'est-ce que le rendu côté serveur (Server-Side Rendering, SSR) ?

Vue.js est un framework pour créer des applications clients. Par défaut, les composants Vue produisent et manipulent le DOM dans le navigateur. Toutefois, il est aussi possible de rendre ces mêmes composants en chaînes de caractères HTML sur le serveur, les envoyer directement au navigateur, et enfin "hydrater" le balisage statique en une application entièrement interactive sur le client.

Une application Vue rendue par le serveur peut aussi être considérée comme "isomorphique" ou "universelle", dans le sens que la majorité du code de votre application fonctionnera côté serveur **et** côté client.

## Pourquoi le rendu côté serveur ?

Par rapport à la traditionnelle SPA (Single-Page Application, Application Page Unique), l'avantage du SSR consiste en :

* Un meilleur SEO, vu que les robots des moteurs de recherche verront directement la page entièrement rendue,

* Notez qu'à partir de maintenant, Google et Bing peuvent indexer les applications JavaScript synchrones. Synchrone est le mot important ici. Si votre application commence avec une image de chargement, et qu'elle récupère du contenu via Ajax, le robot ne va pas attendre que la récupération du contenu de manière asynchrone soit terminée. Ce qui veut dire que si vous avez du contenu récupéré de manière asynchrone et où le SEO est important, le SSR pourrait être nécessaire.

* Un chargement plus rapide, notamment avec des appareils lents, ou une connexion internet lente. Le code HTML rendu par le serveur n'a pas besoin d'attendre que le JavaScript soit téléchargé et exécuté pour être affiché. L'utilisateur de votre application verra donc plus tôt une page entièrement rendue. Cela résulte généralement en une meilleure expérience utilisateur, et peut être critique pour des applications où le chargement est directement associé avec le taux de conversion.

Il y a également quelques points négatifs à prendre en compte en utilisant le SSR :

* Les contraintes de développement. Le code spécifique au navigateur ne peut être utilisé que dans certains connecteurs du cycle de vie ; certaines librairies devront recevoir un traitement spécial pour être capable de fonctionner sur une application rendue par le serveur.

* Un build setup et des besoins pour le déploiement plus complexes. Au contraire d'une SPA entièrement statique qui peut être déployée sur n'importe quel serveur de fichier statique, une application rendue par le serveur a besoin d'un environnement où un serveur Node.js peut tourner.

* Plus de charge pour le serveur. Faire entièrement le rendu d'une application avec Node.js sera bien évidemment plus coûteux pour le processeur, que de servir des fichiers statiques. Donc si vous vous attendez à beaucoup de trafic, préparez-vous à cette charge serveur et utilisez judicieusement des stratégies de mise en cache.

Avant d'utiliser le SSR pour votre application, la première question que vous devez vous demander est si vous en avez réellement besoin. Cela dépend principalement à quel point le temps d'affichage de votre application est important ou non. Par exemple, si vous avez créé un tableau de bord interne à votre application, et où attendre quelques centaines de millisecondes en plus n'est pas vraiment un problème, il serait exagéré d'utiliser le SSR. Cependant, dans les cas où le chargement et le rendu de la page est un point critique, le SSR peut vous aider à atteindre les meilleures performances possibles pour le chargement initial de votre application.

## SSR vs pré-rendu

Si vous n'êtes seulement intéressé par le SSR uniquement pour améliorer le SEO et une poignée de pages (ex: `/` ,  `/about` , `/contact`, etc...), alors c'est probablement  que vous vous intéressez au **pré-rendu**. Au lieu d'utiliser un serveur web pour compiler du HTML à la volée, le pré-rendu génère des fichiers HTML pour des routes spécifiques, au moment de la compilation. L'avantage du pré-rendu est qu'il est beaucoup plus simple à mettre en place, et qu'il vous permet de garder votre _front-end_ entièrement statique.

Si vous utilisez Webpack, il est alors possible d'ajouter facilement le pré-rendu avec [prerender-spa-plugin](https://github.com/chrisvfritz/prerender-spa-plugin). Il a été largement testé avec les applications Vue - et en fait, le créateur est un membre de l'équipe principale de Vue.

## A propos de ce guide

Ce guide se concentre sur les SPA rendues par un serveur Node.js. Mélanger le SSR de Vue avec d'autres configurations _backend_ est un sujet à part entière, et ne sera pas pas couvert dans ce guide.

Ce guide sera très approfondi, il est donc nécessaire d'être familier avec Vue.js, et d'avoir une connaissance décente de Node.js et webpack. Si vous préférez une solution plus avancée et qui offre une meilleure expérience d'utilisation _out-of-the-box_, vous devriez essayer [Nuxt.js](https://nuxtjs.org/). Nuxt.js est construit sur le même _Vue stack_, mais elle abstrait énormément la structure de base de l'application. Elle apporte cependant quelques fonctionnalités supplémentaires, comme la génération de site statique par exemple. Toutefois, il se peut que cela ne convienne pas à votre utilisation si vous avez besoin de plus de contrôle sur la structure de votre application. Quoi qu'il en soit, il serait toujours utile de lire ce guide pour mieux comprendre son fonctionnement.

Comme vous lisez, il serait utile de se référer à la [démo HackerNews ](https://github.com/vuejs/vue-hackernews-2.0/) officielle, qui utilise la plupart des techniques couvertes dans ce guide.

Enfin, notez que les solutions dans ce guide ne sont pas définitives - nous avons trouvées qu'elles fonctionnaient bien pour nous, mais cela ne veut pas dire qu'elles ne peuvent pas être améliorées. Ces solutions pourront être re-travaillées à l'avenir - vous êtes libre de contribuer en soumettant des _pull requests_ !

