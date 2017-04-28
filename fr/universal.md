# Écrire un code universel

Avant d'aller plus loin, prenons un moment pour discuter des contraintes lors de l'écriture d'un code "universel" - qui est, un code qui fonctionne côté serveur et également côté client. En raison des différences d'API des deux plate-formes, le comportement de notre code ne pourra pas forcément être exactement le même selon l'environnement. Nous examinerons ici les choses principales dont vous avez besoin de savoir.

## Réactivité des données sur le serveur

Dans une application fonctionnant uniquement côté client, chaque utilisateur utilisera une nouvelle instance de l'application dans leur navigateur. Pour le rendu serveur, nous souhaitons le même comportement : chaque requête doit avoir une nouvelle instance d'application, et isolée. De ce fait, il n'y aura pas de pollution à cause de requêtes croisées. 

Étant donné que le processus actuel de rendu doit être déterministe, il faudra également
pré-récupérer des données sur le serveur - ce qui signifie que l'état de notre application sera déjà disponible avant le lancement du rendu. Cela signifie aussi que la réactivité des données sera inutile sur le serveur, et est donc désactivée par défaut. Désactiver la réactivité des données permet d'éviter un coût de performance lors de la conversion de données en objets réactifs.

## Connecteurs sur le cycle de vie d'un Composant

Vu qu'il n'y a pas de modifications dynamiques, de tous les connecteurs de cycle de vie, seulement `beforeCreate` et `created` seront appelées pendant le  SSR. Ce qui signifie que le code à l'intérieur des autres connecteurs de cycle de vie, tels que `beforeMount` ou `mounted` sera exécuté uniquement sur le client.

## Accéder aux API spécifiques à la plate-forme

Du code universel ne peut avoir accès aux API spécifiques à une plate-forme. Si votre code utilise des variables globales uniquement disponibles dans le navigateur, comme `window` ou `document`, une erreur sera lancée lors de l'exécution dans Node.js, et vice-versa.

Pour les tâches partagées entre le serveur le client mais qui utilisent les différentes APIs de plate-formes, il est recommandé d'envelopper le code spécifique à la plate-forme, dans une API universelle, ou alors d'utiliser des librairies qui font cela pour vous. Par exemple, [axios](https://github.com/mzabriskie/axios) est un client HTTP qui met à disposition la même API pour le côté serveur ainsi que le côté client.

Pour les APIs spécifiques au navigateur, il faudra les utiliser dans les connecteurs de cycle de vie réservés au client.

Notez que si une librairie tierce n'est pas écrite dans le but d'être universelle, il peut être délicat de l'intégrer côté serveur. Il pourrait être possible de la faire fonctionner en *mockant* certaines des variables et méthodes globales, mais cela serait difficile et pourrait interférer avec le code de détection de l'environnement des autres librairies.

## Directives personnalisées

La plupart des directives personnalisées manipulent directement le DOM, et donc causera des erreurs pendant le SSR. Il y a deux moyens d'éviter cela :

1. Préférer l'utilisations de composants comme mécanisme d'abstraction, et travailler au niveau du DOM virtuel (ex: en utilisant les fonctions de rendu).

2. Si vous avez une directive personnalisée qui ne peut être facilement remplacée par des composants, il est possible de fournir un "version server" de celle-ci en utilisant l'option [`directives`](./api.md#directives) pendant la création du moteur de rendu côté serveur.
