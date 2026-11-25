# Chapitre 4 — Préparer l'environnement de développement

> Partie I — Comprendre et concevoir une API REST  
> **Objectif :** comprendre les outils nécessaires, préparer un environnement fiable et vérifier chaque composant avant de créer le projet Laravel.

## Introduction : le code ne s'exécute pas dans le vide

Pour développer une API, l'ordinateur doit posséder un langage d'exécution, un gestionnaire de dépendances, un serveur capable de recevoir les requêtes, une base de données et des outils de test.

Quand une commande échoue, le problème peut venir du code, de la configuration, d'une version incompatible, d'un service arrêté ou d'un outil absent. Un développeur sérieux ne réinstalle pas tout au hasard : il vérifie les composants un par un.

## 4.1. Les composants de notre environnement

| Outil | Fonction |
|---|---|
| PHP | Exécute le backend Laravel |
| Composer | Installe et gère les dépendances PHP |
| Laravel | Framework qui organise le développement |
| MariaDB ou MySQL | Conserve les données relationnelles |
| Serveur local | Reçoit les requêtes HTTP en développement |
| Git | Suit l'historique du code et du manuscrit |
| GitHub | Héberge le dépôt distant et facilite la collaboration |
| Postman ou équivalent | Envoie des requêtes et inspecte les réponses |
| Éditeur | Permet d'écrire et d'examiner le code |

Node.js et npm peuvent être requis par certains outils frontend ou scripts du projet. Ils ne remplacent ni PHP ni Composer.

## 4.2. PHP : le moteur d'exécution

PHP est le langage utilisé par Laravel côté serveur. Lorsqu'une requête arrive sur une route, PHP exécute le code qui la traite.

Il faut distinguer :
- le code source PHP ;
- l'interpréteur PHP installé ;
- les extensions PHP ;
- la version utilisée dans le terminal ;
- la version utilisée par le serveur web.

Ces versions peuvent différer, notamment si plusieurs installations sont présentes.

### Vérifier PHP

Dans le terminal :

    php -v

La commande affiche la version accessible depuis le terminal.

Sous Windows, pour trouver les exécutables accessibles :

    where.exe php

Sous Linux ou macOS :

    which php

Si la commande n'est pas reconnue, PHP peut être absent ou son dossier ne figure pas dans la variable PATH.

### Vérifier les extensions

Les dépendances Laravel utilisent plusieurs extensions PHP. La liste exacte dépend de la version et des packages.

    php -m
    php --ini

La première commande affiche les extensions chargées ; la seconde indique le fichier de configuration utilisé. La présence d'un fichier DLL dans un dossier ne garantit pas que l'extension est activée et chargée par le bon PHP.

## 4.3. Composer : gérer les dépendances

Composer est le gestionnaire de dépendances PHP. Laravel utilise de nombreux packages ; Composer les installe et vérifie leurs contraintes de compatibilité.

Deux fichiers sont importants :

- **composer.json** décrit les dépendances et les contraintes du projet.
- **composer.lock** enregistre les versions exactes résolues.

Pour installer les dépendances d'un projet qui possède son fichier lock :

    composer install

Cette commande respecte les versions verrouillées. **composer update** cherche au contraire à résoudre de nouvelles versions selon les contraintes et peut modifier le fichier lock.

Dans un projet partagé, utilisez généralement install pour reproduire l'environnement du dépôt. Ne lancez pas update sans comprendre les changements qu'il peut provoquer.

Vérification :

    composer --version

## 4.4. Laravel : le framework

Laravel est un framework PHP qui fournit une structure et des outils pour construire des applications web et des API.

Il apporte notamment :
- routes et contrôleurs ;
- middleware ;
- accès aux données avec Eloquent ;
- migrations ;
- validation ;
- gestion des exceptions ;
- outils de test.

Laravel ne remplace pas les bases de PHP, HTTP ou SQL. Il organise leur utilisation et évite de reconstruire les mécanismes courants à chaque projet.

### Vérifier la compatibilité de version

Les versions Laravel ont des exigences précises. Au moment de la rédaction, la documentation officielle Laravel 13 indique PHP 8.3 minimum. Cette exigence devra être revérifiée pour l'édition finale du livre.

Avant de choisir une version :
1. consultez la documentation officielle ;
2. vérifiez PHP et ses extensions ;
3. vérifiez les exigences de la base ;
4. contrôlez la compatibilité des packages ;
5. privilégiez une version stable et maintenue.

Un tutoriel ancien peut utiliser une version qui n'est plus adaptée à votre environnement.

## 4.5. MariaDB ou MySQL : conserver les données

Notre plateforme doit conserver les utilisateurs, profils, catégories, services et demandes. Une base relationnelle organise les données en tables reliées.

Distinguez :
- **serveur de base de données** : programme qui accepte les connexions et exécute SQL ;
- **base** : espace logique contenant les tables ;
- **utilisateur SQL** : compte doté de privilèges ;
- **port** : point de communication réseau.

Le port par défaut est souvent 3306 pour MySQL/MariaDB, mais une installation locale peut en utiliser un autre.

phpMyAdmin est une interface d'administration. Il ne remplace pas le serveur de base de données.

### Vérifier le service

Assurez-vous que le service démarre et notez :
- hôte ;
- port ;
- nom de la base ;
- utilisateur ;
- mot de passe.

En local, l'hôte est souvent 127.0.0.1 ou localhost. La configuration exacte dépend de l'installation.

## 4.6. Le serveur local et les ports

Pour tester l'API, un serveur doit écouter sur une adresse et un port.

Laravel fournit un serveur de développement :

    php artisan serve

Il affiche généralement une adresse telle que http://127.0.0.1:8000.

Cette commande sert au développement et ne constitue pas à elle seule une configuration de production.

Dans l'adresse :
- http est le protocole ;
- 127.0.0.1 est l'adresse de boucle locale, donc l'ordinateur lui-même ;
- 8000 est le port d'écoute.

Si le port est déjà utilisé, le serveur peut refuser de démarrer. Il faut identifier le programme concerné ou choisir un autre port, pas supprimer au hasard des fichiers du projet.

## 4.7. Git et GitHub : suivre le travail

Git est un système de gestion de versions. Il conserve l'historique des changements et permet de comparer les états du projet.

GitHub héberge des dépôts Git et fournit des outils de collaboration. Notre dépôt **creer-api-rest-laravel** conserve le manuscrit. Le code compagnon pourra être organisé séparément lorsque nous définirons sa structure.

Vocabulaire :
- **repository** : dépôt suivi par Git ;
- **commit** : enregistrement cohérent de modifications ;
- **branch** : ligne de travail ;
- **remote** : dépôt distant ;
- **push** : envoi des commits vers le dépôt distant ;
- **pull** : récupération et intégration des changements distants.

Vérifiez Git :

    git --version

Dans un projet :

    git status

Cette commande affiche la branche et les modifications présentes.

### Protéger les secrets

Le fichier .env contient souvent les paramètres locaux, dont les identifiants de base et les clés de l'application. Il ne doit pas être publié dans un dépôt public.

Le fichier .env.example peut présenter les noms des variables, sans vrais secrets.

Avant un commit, examinez toujours les fichiers à ajouter. Un secret publié peut rester dans l'historique Git même après sa suppression du fichier actuel. Si un secret a été exposé, il faut le remplacer ou le révoquer, et pas seulement effacer la ligne.

## 4.8. Postman : tester indépendamment du frontend

Postman permet de construire et envoyer des requêtes HTTP, puis d'observer les réponses.

Vous pouvez :
- choisir la méthode ;
- saisir l'URL ;
- ajouter des headers ;
- envoyer un body JSON ;
- fournir un token ;
- lire le statut et le body ;
- enregistrer des requêtes dans une collection.

Exemple de test local :

    GET http://127.0.0.1:8000/api/services
    Accept: application/json

Postman n'est pas le seul outil possible. L'objectif est de tester l'API indépendamment de Flutter ou d'une interface web, afin de savoir si le problème vient du backend ou du client.

## 4.9. Comprendre XAMPP sous Windows

XAMPP regroupe généralement Apache, MariaDB, PHP et des outils associés. Il peut faciliter le développement local, mais il faut connaître les versions réellement utilisées.

Un poste peut avoir :
- Apache utilisant le PHP de XAMPP ;
- le terminal utilisant un autre PHP ;
- Composer utilisant le PHP trouvé dans PATH.

Ainsi, une application peut fonctionner dans le navigateur et échouer dans Composer, ou l'inverse.

Pour diagnostiquer :
1. exécutez where.exe php ;
2. exécutez php -v ;
3. vérifiez le PHP configuré pour Apache ;
4. comparez les versions et les extensions.

Avant de modifier PATH ou php.ini, notez la configuration actuelle. Une modification mal faite peut perturber d'autres projets.

## 4.10. Checklist avant de créer le projet

- [ ] PHP est installé et compatible avec la version Laravel choisie.
- [ ] Les extensions nécessaires sont chargées.
- [ ] Composer fonctionne.
- [ ] Le serveur de base de données démarre.
- [ ] La connexion à la base est vérifiable.
- [ ] Git est installé.
- [ ] Un outil de test HTTP est disponible.
- [ ] L'éditeur ouvre correctement le dossier.
- [ ] Les secrets sont exclus du dépôt public.

Une installation réussie ne signifie pas seulement que les logiciels sont présents. Il faut comprendre leurs versions, leurs chemins et leur rôle.

## 4.11. Diagnostiquer les erreurs courantes

### « php n'est pas reconnu »
PHP est peut-être absent ou son dossier n'est pas dans PATH. Vérifiez l'installation, le chemin et ouvrez un nouveau terminal après une modification.

### « composer n'est pas reconnu »
Composer peut être absent ou inaccessible depuis le terminal. Vérifiez son installation et la commande composer --version.

### Connexion à la base refusée
Le service peut être arrêté, l'hôte ou le port incorrect. Vérifiez le service et les paramètres de connexion.

### Accès à la base refusé
Le compte SQL ou le mot de passe peut être incorrect, ou les privilèges insuffisants. N'accordez pas automatiquement tous les privilèges à l'utilisateur applicatif.

### Port déjà utilisé
Un autre programme écoute sur ce port. Identifiez-le ou choisissez un port libre. Ne terminez pas un processus système sans savoir ce qu'il fait.

### Extension PHP absente
L'extension peut ne pas être installée, activée ou chargée par le fichier php.ini attendu. Utilisez php -m et php --ini, puis vérifiez séparément le PHP du serveur web.

## Atelier pratique : contrôler son poste

Exécutez les commandes une par une :

    php -v
    composer --version
    git --version
    php -m
    php --ini

Notez les versions et le chemin de configuration. Vérifiez ensuite le service de base de données et préparez votre outil HTTP.

Ne passez à la création du projet que lorsque vous pouvez expliquer le rôle de chaque composant et où vérifier son fonctionnement.

## Exercices

### Exercice 1
Associez PHP, Composer, Laravel, MariaDB, Git, GitHub et Postman à leurs fonctions.

### Exercice 2
L'application fonctionne dans le navigateur, mais composer install signale une extension PHP absente. Pourquoi les deux observations peuvent-elles être compatibles ? Proposez un diagnostic.

### Exercice 3
Votre fichier .env contient le mot de passe de la base et votre dépôt est public. Que faire avant de partager le projet ? Pourquoi une suppression dans un commit ultérieur ne suffit-elle pas toujours ?

### Exercice 4
Un collègue clone le projet, mais il ne démarre pas. Quels fichiers et informations lui permettent d'installer les dépendances et de configurer son environnement sans recevoir vos secrets ?

## Corrigés commentés

**Exercice 1 :** PHP exécute le code ; Composer gère les dépendances ; Laravel structure le backend ; MariaDB conserve les données ; Git suit les versions ; GitHub héberge le dépôt ; Postman envoie les requêtes HTTP.

**Exercice 2 :** Apache peut utiliser une version de PHP différente de celle du terminal. Comparez where.exe php, php -v, php --ini et la configuration PHP du serveur web.

**Exercice 3 :** retirez le secret, excluez .env avec .gitignore, vérifiez l'historique et révoquez/remplacez le secret exposé. Un ancien commit peut toujours contenir la valeur.

**Exercice 4 :** fournissez composer.json, composer.lock, .env.example, les prérequis de version et des instructions d'installation. Le collègue crée son propre .env.

## Synthèse

PHP exécute le backend, Composer installe ses dépendances, Laravel fournit le framework, la base conserve les données, le serveur local reçoit les requêtes et Postman permet de tester l'API.

Git et GitHub suivent le travail, mais les secrets doivent rester hors du dépôt. Les versions, extensions, chemins et services doivent être vérifiés méthodiquement.

**Transition :** notre conception est prête et l'environnement est compris. Le chapitre 5 nous fera créer et configurer le projet Laravel API.

## Références officielles

- Laravel 13 — Installation : https://laravel.com/docs/13.x/installation
- Laravel 13 — Configuration : https://laravel.com/docs/13.x/configuration
- Laravel 13 — Exigences et déploiement : https://laravel.com/docs/13.x/deployment
- PHP : https://www.php.net/manual/fr/
- Composer : https://getcomposer.org/doc/
- Git : https://git-scm.com/docs
- GitHub : https://docs.github.com/
- Postman : https://learning.postman.com/docs/

*Note éditoriale : les versions et liens seront revérifiés lors de la relecture technique finale.*
