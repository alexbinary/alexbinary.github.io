

Sur iOS en matière de gestion de dépendances, il y a CocoaPods et Carthage.

Il semble que Xcode ne soit pas prévu pour une bonne gestion des dépendances.

Avec l'arrivée de Swift nous avons eu Swift Package Manager, qui est une vraie solution officielle et intégrée avec les outils.

Malheureusement, il n'existe encore aucun moyen d'utiliser Swift PM avec des projets Xcode.

J'espère que ça arrivera, et qu'avec cela nous auront enfin une vraie solution pour la gestion des dépendances dans les projets iOS.

En attendant, j'expérimente avec des solutions simples.

Je travaille actuellement sur une application iOS, pour laquelle je développe une bibliothèque que je souhaite maintenir indépendament de l'application iOS,
même si pour l'instant elle n'est utilisée que là.

La première idée simple qui vient à l'esprit est d'inclure directement les sources de la bibliothèque dans le projet par copier-coller.

Cela pose deux problèmes :

1. En cas de mise à jour, il faut mettre à jour les fichiers manuellement.
Le plus simple est de supprimer tout le dossier et de copier la nouvelle version.
Cependant le risque d'erreur est toujours présent, et ce n'est pas très propre.

2. Sans plus d'indications, il est impossible de savoir que les fichiers de la bibliothèque correspondent à un projet autonome.
Ils apparaissent en effet de la même manière que les fichiers propre au projet principal.
Un contributeur non averti pourra faire évoluer les fichiers sources de la bibliothèque,
et ses modifications pourront être perdues lors du remplacement des fichiers par une nouvelle version.

Les sous modules git apportent des solutions.

Les sous modules git permettent d'intégrer les sources d'un repo dans un autre, sur une révision précise.

Un sous module git permet de facilement mettre à jour la dépendance en une seule commande.

Les sources de la dépendance ne sont pas réellement incluses dans le projet, mais simplement référencées.

Les sous modules git expriment clairement le fait que le projet inclu en sous module ne fait pas partie du projet principal.

Un contributeur non averti se rendra vite compte de la situation s'il intervient sur les fichiers de la bibliothèque,
et pourra même facilement contribuer ses changements au repo de la bibliothèque.

Voila pour ce qui est de la gestion des fichiers. Maintenant la gestion dans Xcode.

Encore une fois, l'approche la plus simple est de considérer que les fichiers de la bibliothtèque font partie du projet parent et de les inclure tous dans le projet Xcode.

Mais comme précédement, cela pose problème en cas de mise à jour où il faut s'assurer de correctement mettre à jour les référence de tous les fichiers concernés.
Et ce n'est toujours pas très propre.

La création d'un framework résout ce problème.

Créer un nouveau projet Xcode et choisir iOS > Cocoa Touch Framework.

Ajouter les fichiers nécessaires au projet. Il est possible de compiler (Cmd + B) mais pas de lancer le projet.

Dans le projet parent, sélectionner le projet dans l'arborescence, sélectionner ensuite la cible, puis dans l'onglet Général, section Linked Frameworks and Librairies, cliquer sur le +.

Choisir le fichier .xcodeproj du projet framework créé précédement. Le projet apparait dans le dossier Frameworks du projet parent.

Il est possible de dérouler le projet et de travailler dedans directement.

Dans Linked Frameworks and Librairies, cliquer à nouveau sur le + et ajouter cette fois le .framework qui correspond à notre bibliothèque.

Celui-ci apparait maintenant dans la liste.

Pour utiliser les objets définis dans le framework ceux-ci doivent être marqués avec `public`.

Penser également à importer le module avec `import <nom du module>`.