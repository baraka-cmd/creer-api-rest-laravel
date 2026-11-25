# Chapitre 27 — Préparer Laravel pour la production : passer du développement à un service fiable

## 27.1. Le passage en production est un changement de responsabilité

En développement, l'objectif est de construire et de vérifier les fonctionnalités. En production, l'application reçoit de vraies requêtes, manipule des données réelles et devient une dépendance pour ses utilisateurs. Une erreur de configuration peut exposer des informations, interrompre le service ou rendre les données incohérentes.

La préparation de production ne consiste donc pas seulement à modifier une valeur dans le fichier d'environnement. Elle exige de vérifier plusieurs dimensions : sécurité, configuration, base de données, stockage, erreurs, performances, sauvegardes et procédure de retour arrière.

Un environnement de production doit être distinct de l'environnement local. Les mots de passe, clés, URL, services externes et paramètres de fonctionnement ne doivent pas être copiés sans réflexion depuis la machine de développement.

## 27.2. Comprendre les environnements

Un projet peut utiliser plusieurs environnements :

- **local** : développement quotidien sur l'ordinateur ;
- **test ou CI** : exécution automatisée des tests ;
- **staging** : environnement de préproduction qui ressemble autant que possible à la production ;
- **production** : environnement utilisé par les clients réels.

Chaque environnement possède sa propre configuration et, idéalement, ses propres données et secrets. Ne connectez jamais un environnement de test à la base de données réelle par commodité.

La préproduction sert à répéter le déploiement, vérifier les migrations, tester les intégrations et contrôler le comportement de l'application avant de toucher les utilisateurs.

## 27.3. Variables d'environnement et secrets

Laravel utilise notamment le fichier .env pour la configuration locale. Ce fichier contient des informations qui ne doivent pas être publiées dans un dépôt Git : identifiants de base de données, clés d'application, secrets d'API et paramètres privés.

Le fichier .env de développement ne doit pas être transféré tel quel sur le serveur. La plateforme d'hébergement peut proposer un panneau de variables, un gestionnaire de secrets ou une configuration propre au service.

Principes essentiels :

- ne jamais commiter de secrets ;
- ne jamais partager une clé d'application dans une capture ou un journal ;
- utiliser des secrets différents entre développement, test et production ;
- limiter les personnes et services qui peuvent lire les secrets ;
- renouveler un secret lorsqu'il a été exposé ;
- ne pas placer un secret dans le code source ou dans le frontend.

La variable APP_KEY est importante pour les mécanismes cryptographiques de Laravel. Elle doit être générée et conservée de manière stable pour l'environnement concerné. La régénérer à chaque déploiement peut rendre certaines données chiffrées ou sessions existantes illisibles.

## 27.4. Désactiver le mode debug

En production, le mode debug doit être désactivé.

    APP_ENV=production
    APP_DEBUG=false

Lorsque le debug est activé, une erreur peut afficher des détails techniques : chemins du serveur, traces d'exécution, requêtes, configuration ou informations internes. Ces détails aident le développeur, mais peuvent aider un attaquant.

La désactivation du debug ne signifie pas qu'il faut cacher les erreurs à l'équipe. Les erreurs doivent être enregistrées dans des journaux contrôlés et consultables par les personnes autorisées.

Ne testez pas le fonctionnement du debug en provoquant volontairement une erreur sensible sur le serveur public. Vérifiez la configuration et réalisez les essais dans un environnement approprié.

## 27.5. Préparer la configuration Laravel

Les applications Laravel peuvent mettre en cache leur configuration et leurs routes pour réduire le travail au démarrage et améliorer les performances.

Lors d'un déploiement, les commandes courantes incluent, selon la version et les caractéristiques du projet :

    php artisan config:cache
    php artisan route:cache
    php artisan view:cache

Ces commandes doivent être exécutées dans le cadre d'une procédure maîtrisée. Si l'application dépend de routes ou de configurations incompatibles avec le cache, le déploiement peut échouer. Testez d'abord sur la préproduction.

Après mise en cache de la configuration, Laravel utilise la configuration compilée. Évitez de lire directement env() dans le code métier ou les contrôleurs. Les appels à env() doivent être réservés aux fichiers de configuration ; le reste de l'application doit utiliser config().

## 27.6. Vérifier les permissions et les fichiers

Laravel doit pouvoir écrire dans les répertoires prévus, notamment storage et bootstrap/cache. Il ne doit pas avoir un accès en écriture généralisé à tout le code source.

Le principe est celui du moindre privilège :

- le processus PHP peut écrire dans les répertoires nécessaires ;
- les fichiers de code ne sont pas modifiables par tous les utilisateurs du serveur ;
- les fichiers privés ne sont pas accessibles directement depuis le web ;
- les clés et fichiers de configuration ne sont pas servis comme des documents publics.

Ne corrigez pas une erreur de permission en donnant des droits complets à tous les utilisateurs. Identifiez le compte qui exécute l'application et accordez uniquement les permissions nécessaires.

## 27.7. Préparer la base de données

La base de production doit être créée avec un utilisateur applicatif disposant de permissions limitées. Évitez d'utiliser le compte administrateur global de la base dans la configuration Laravel.

Avant le déploiement, vérifiez :

- le nom de la base et le serveur ;
- les identifiants et permissions ;
- le moteur et la version compatibles ;
- l'encodage et la collation ;
- les migrations disponibles ;
- les index nécessaires ;
- la stratégie de sauvegarde ;
- la possibilité de restaurer une sauvegarde.

Une migration est une modification du schéma. Elle peut échouer, prendre du temps ou être incompatible avec une version précédente du code. Il faut donc la traiter comme une opération de production, pas comme une simple commande anodine.

## 27.8. Exécuter les migrations avec prudence

La commande Laravel courante pour appliquer les migrations en production est :

    php artisan migrate --force

L'option force confirme l'exécution dans un environnement de production. Elle ne rend pas une migration sûre par elle-même.

Avant de l'exécuter :

1. examiner les migrations ;
2. vérifier leur ordre et leurs dépendances ;
3. confirmer que la sauvegarde est récente et restaurable ;
4. déterminer si la migration bloque les écritures ;
5. vérifier la compatibilité entre ancien code, nouveau schéma et éventuel retour arrière ;
6. effectuer l'opération dans une fenêtre adaptée si nécessaire.

Évitez les migrations qui suppriment immédiatement une colonne encore utilisée par l'ancienne version de l'application. Une stratégie progressive est souvent plus sûre : ajouter la nouvelle structure, déployer le code compatible, migrer les données, puis supprimer l'ancien champ dans une version ultérieure.

## 27.9. Gérer les erreurs sans divulguer les détails

Une API professionnelle doit renvoyer des erreurs compréhensibles et cohérentes. Le client peut recevoir un message utile, un code d'erreur stable et, lorsque c'est pertinent, un identifiant de suivi.

Il ne doit pas recevoir :
- une trace complète de la pile ;
- un mot de passe ou un secret ;
- une requête SQL avec des données sensibles ;
- un chemin interne du serveur ;
- des informations privées sur un autre utilisateur.

Les journaux internes peuvent contenir davantage de détails, mais doivent être protégés, limités et soumis à une politique de conservation. Les données sensibles ne doivent pas être ajoutées automatiquement aux logs.

## 27.10. Préparer les tâches et services annexes

Une API peut dépendre d'une file de tâches, d'un système de notifications, d'un stockage de fichiers ou d'un service de paiement. Ces dépendances doivent être configurées et testées séparément.

Exemples de vérifications :
- les tâches asynchrones ont-elles un worker actif ?
- les échecs de tâches sont-ils consultables ?
- les notifications sont-elles envoyées avec une configuration de production ?
- les délais d'attente et reprises sont-ils raisonnables ?
- les webhooks externes sont-ils vérifiés et idempotents ?
- les fichiers sont-ils stockés dans un espace durable ?

Une réponse HTTP réussie ne garantit pas toujours que le travail asynchrone a été effectué. L'API doit documenter ce qui est immédiat et ce qui est traité en arrière-plan.

## 27.11. Checklist de préparation

Avant d'ouvrir l'API au public, vérifiez :

- APP_ENV est configuré pour la production ;
- APP_DEBUG est désactivé ;
- les secrets sont propres à l'environnement ;
- APP_KEY est stable et protégée ;
- HTTPS est prévu ;
- les caches ont été testés ;
- les permissions sont limitées ;
- la base utilise un compte applicatif restreint ;
- les migrations ont été examinées ;
- les erreurs sont journalisées sans fuite ;
- les tâches, notifications et fichiers sont configurés ;
- une sauvegarde et une procédure de restauration existent.

**À retenir :** une application n'est prête pour la production que lorsque sa configuration, ses données, ses dépendances et ses procédures d'exploitation sont maîtrisées — pas uniquement lorsque la page d'accueil répond.
