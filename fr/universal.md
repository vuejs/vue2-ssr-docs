# Écrire du code universel

Avant d'aller plus loin, prenons un moment pour discuter des contraintes lorsque l'on écrit du code « universel » ; c'est à dire du code qui s'exécute à la fois côté serveur et côté client. En raison des différences d'API entre les deux plateformes, le comportement de notre code ne sera pas exactement le même selon l'environnement. Nous allons examiner les points clés dont vous devez avoir connaissance.

## Réactivité des données côté serveur

Dans une application qui tourne exclusivement côté client, chaque utilisateur utilisera une nouvelle instance de l'application dans leur navigateur. Pour le rendu serveur nous voulons le même fonctionnement : chaque requête doit disposer d'une nouvelle instance d'application isolée. Ainsi il n'y aura pas de pollution liée à du partage d'état entre requêtes.

Étant donné que le processus de rendu actuel doit être déterministe, nous allons aussi « précharger » les données sur le serveur. Cela signifie que l'état de notre application sera déjà disponible quand nous commençons le rendu. Cela signifie également que la réactivité des données est inutile côté serveur ; elle est donc désactivée par défaut. Désactiver la réactivité des données évite aussi le cout de performance d'une conversion de données en objets réactifs.

## Hooks de cycles de vie des composants

Vu qu'il n'y a pas de mises à jour dynamiques, de tous les hooks de cycles de vie, seuls `beforeCreate` et `created` seront appelés pendant le rendu côté serveur. Cela signifie que tout code présent dans d'autres hooks tels que `beforeMount` ou `mounted` sera exécuté uniquement côté client.

Une autre chose à noter est que vous devriez éviter la création d'effets de bord globaux dans `beforeCreate` et `created` comme ceux, par exemple, dus aux timers avec `setInterval`. Nous pouvons mettre en place des timers seulement dans du code côté client qui seront arrêtés pendant les phases `beforeDestroy` et `destroyed`. Cependant, comme ces hooks ne sont jamais appelés pendant le SSR, les timers vont continuer de tourner éternellement. Pour éviter cela, déplacez ce type d'effet de bord dans les hooks `beforeMount` ou `mounted`.

## Accès aux APIs spécifiques à la plateforme

Le code universel ne peut pas accéder aux APIs spécifiques à une plateforme. Ainsi, si votre code utilise directement les variables globales exclusives au navigateur comme `window` ou `document`, elles lèveront des erreurs si elles sont exécutées sur Node.js, et vice-versa.

Pour les tâches partagées entre le serveur et le client, mais qui utilisent des APIs différentes selon la plateforme, il est recommandé d'encapsuler le code spécifique à la plateforme dans une API universelle, ou d'utiliser des bibliothèques qui le font pour vous. Par exemple, [axios](https://github.com/mzabriskie/axios) est un client HTTP qui présente la même API côté serveur et côté client.

Pour les APIs exclusives au navigateur, l'approche habituelle est de les utiliser dans les hooks de cycle de vie exclusifs au client.

Notez que si une bibliothèque tierce n'est pas écrite avec l'objectif d'être universelle, cela peut être délicat de l'intégrer dans une application rendue côté serveur. Vous *devriez* être capable de la faire fonctionner en substituant certaines variables globales, mais cela serait cavalier et pourrait interférer avec du code de détection d'environnement des autres bibliothèques.

## Directives personnalisées

La plupart des directives personnalisées manipulent directement le DOM, et vont ainsi provoquer des erreurs durant le rendu côté serveur. Il y a deux façons d'éviter cela :

1. Préférer l'utilisation de composants comme mécanisme d'abstraction et travailler au niveau du DOM virtuel (par ex. en utilisant des fonctions de rendu)

2. Si vous avez une directive personnalisée qui ne peut être facilement remplacée par des composants, vous pouvez en fournir une « version serveur » qui utilise l'option [`directives`](./api.md#directives) lors de la création du rendu serveur.
