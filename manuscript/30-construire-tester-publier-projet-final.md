# Chapitre 30 — Construire, tester et publier le projet final

> Partie VIII — Projet final  
> **Objectif :** assembler les compétences du livre dans une API REST cohérente, sécurisée, documentée, testée et déployable. Ce chapitre suit le cahier de conception du projet ProxiServices API.

## 30.1. Le résultat attendu

À la fin de ce projet, vous aurez conçu et implémenté un backend Laravel permettant :

- l'inscription et la connexion des utilisateurs ;
- l'authentification par tokens pour les clients mobiles ou les outils API ;
- la consultation des catégories actives ;
- la création et la gestion d'un profil professionnel ;
- la création, la modification, la publication et la consultation des services ;
- la recherche, le filtrage et la pagination des services publics ;
- l'ajout contrôlé d'images ;
- la protection des données privées et des opérations propriétaires ;
- des tests automatisés, une documentation OpenAPI et une collection Postman ;
- une préparation et une vérification du déploiement.

Le projet reste volontairement limité à ce périmètre. Les paiements, conversations instantanées, commandes, devis et systèmes complexes de réservation sont des extensions, pas des fonctionnalités à prétendre terminées dans le MVP.

## 30.2. Lire le cahier de conception avant de coder

Le document `30-cahier-conception-projet-fil-conducteur.md` est la référence du projet. Il fixe le périmètre, les acteurs, les ressources, les relations, les routes, les règles de sécurité et les critères d'acceptation.

Avant chaque fonctionnalité, vérifiez :
1. Quel besoin utilisateur résout-elle ?
2. Quelle ressource est concernée ?
3. Qui peut effectuer l'action ?
4. Quels champs le client peut-il fournir ?
5. Quels champs sont déterminés par le serveur ?
6. Quels statuts et erreurs sont attendus ?
7. Quel test prouvera que le comportement est correct ?

Une décision différente peut être prise si elle est nécessaire, mais elle doit être documentée et répercutée dans les migrations, le code, les tests, Postman et OpenAPI.

## 30.3. Figer les versions et préparer le dépôt

Utilisez une version de Laravel et de PHP compatible avec les exigences officielles du framework, puis consignez les versions exactes du projet. Pour ce manuscrit, la cible est Laravel 13 ; vérifiez les exigences de PHP et extensions dans la documentation Laravel 13 avant l'installation.

Commandes de départ :

```bash
composer create-project laravel/laravel proxiservices-api
cd proxiservices-api
php artisan install:api
php artisan about
```

La commande d'installation API prépare les routes API et l'authentification Sanctum selon la version installée. Vérifiez le résultat plutôt que de supposer que tous les fichiers existent identiquement dans chaque version.

Configurez ensuite :
- le dépôt Git et une branche de travail ;
- le fichier `.env` local, jamais commité ;
- la connexion à une base de développement ;
- les dépendances et extensions PHP ;
- le formatage et les conventions de nommage ;
- une première migration et un premier test exécutés avec succès.

## 30.4. Construire la base dans un ordre maîtrisé

Le schéma du MVP contient les entités principales :

- `users` : compte et identité d'authentification ;
- `professional_profiles` : informations du profil professionnel ;
- `categories` : catégories du catalogue ;
- `services` : prestations proposées ;
- `service_images` : images rattachées aux prestations, si cette table est retenue dans le schéma final.

La clé `services.professional_profile_id` référence le profil professionnel, et non directement le compte utilisateur. Les relations Eloquent doivent exprimer ce choix.

Exemple de relation :

```php
// User.php
public function professionalProfile(): HasOne
{
    return $this->hasOne(ProfessionalProfile::class);
}

// ProfessionalProfile.php
public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}

public function services(): HasMany
{
    return $this->hasMany(Service::class);
}

// Service.php
public function professionalProfile(): BelongsTo
{
    return $this->belongsTo(ProfessionalProfile::class);
}
```

Ajoutez les imports et les types nécessaires dans les classes réelles. Les clés étrangères, index et contraintes de suppression doivent correspondre aux règles métier : une catégorie utilisée par des services ne doit pas être supprimée sans stratégie définie.

### Migrations, factories et seeders

Respectez cet ordre :
1. créer les tables de référence ;
2. créer les profils et les services avec leurs clés étrangères ;
3. créer les factories pour des données cohérentes ;
4. créer les seeders de catégories et d'utilisateurs de démonstration ;
5. vérifier une installation vierge ;
6. vérifier que les seeders peuvent être relancés sans créer de doublons indésirables.

Commandes :

```bash
php artisan migrate:fresh
php artisan db:seed
php artisan test
```

**Attention :** `migrate:fresh` détruit les tables de la base ciblée. Utilisez-le uniquement dans une base locale ou de test, jamais sur une base contenant des données à conserver.

## 30.5. Définir les ressources et les règles de validation

Les contrôleurs ne doivent pas accepter tous les champs de la table. Pour la création d'un service, le client peut fournir les champs métier autorisés, par exemple :

- titre ;
- description ;
- catégorie ;
- prix, si le modèle en contient ;
- informations publiques autorisées.

Le serveur détermine notamment :
- le propriétaire du service ;
- l'état initial ;
- les dates ;
- les champs administratifs ;
- les rôles et permissions.

Utilisez une Form Request pour valider les entrées, une Policy pour contrôler les actions sur l'objet et une API Resource pour définir la représentation JSON.

Ne faites jamais ceci :

```php
Service::create($request->all());
```

Préférez une liste de champs validés et un propriétaire calculé côté serveur. Les règles exactes doivent correspondre au modèle du cahier de conception, notamment à la clé `professional_profile_id`.

## 30.6. Implémenter l'authentification

Le parcours d'authentification comprend :
1. inscription avec validation ;
2. hachage du mot de passe par Laravel ;
3. connexion avec vérification des identifiants ;
4. création d'un token Sanctum pour le client mobile ou API ;
5. récupération de l'utilisateur connecté ;
6. révocation du token à la déconnexion.

Le client transmet le token dans le header :

```http
Authorization: Bearer <token>
Accept: application/json
```

Un token est un secret. Ne le stockez pas dans le code source, les logs ou une URL. Pour une application web SPA utilisant des cookies, suivez le flux Sanctum adapté aux SPA : il ne s'agit pas du même mécanisme que l'authentification par token d'une application mobile.

Testez au minimum :
- inscription valide et invalide ;
- connexion valide et incorrecte ;
- accès sans token ;
- accès avec token révoqué ;
- déconnexion et refus du token après révocation.

## 30.7. Implémenter le catalogue

### Catégories

La consultation publique ne retourne que les catégories actives. Les opérations de création, modification ou désactivation sont réservées à l'administration selon les permissions prévues.

### Services

Le cycle de vie doit être explicite. Un service nouvellement créé n'est pas automatiquement visible publiquement. La publication exige que les conditions métier soient satisfaites : profil actif, informations valides, catégorie autorisée et autres critères définis dans le cahier.

Les endpoints publics doivent retourner uniquement les services publiés et visibles. Les endpoints de gestion doivent contrôler l'identité et la propriété de chaque service.

### Recherche

Les paramètres autorisés sont documentés et validés. La liste des colonnes de tri doit être une liste blanche ; ne concaténez jamais un nom de colonne arbitraire fourni par le client dans une requête SQL.

La pagination doit imposer une taille maximale. Les réponses doivent inclure les métadonnées de pagination prévues par le contrat.

## 30.8. Protéger les images et fichiers

Pour chaque upload :
- valider le type réel et la taille ;
- autoriser seulement les formats nécessaires ;
- générer un nom de stockage contrôlé ;
- ne pas faire confiance au nom ou à l'extension fournis ;
- limiter le nombre d'images par service ;
- utiliser un stockage public uniquement pour les images réellement publiques ;
- protéger les documents privés par autorisation et téléchargement contrôlé.

Un fichier n'est pas sécurisé simplement parce que son URL est difficile à deviner. Le serveur doit vérifier l'accès lorsque le contenu est privé.

## 30.9. Organiser le code sans surarchitecture

Une organisation pragmatique du projet peut comprendre :

- Routes : déclaration des endpoints et middleware ;
- Controllers : orchestration HTTP courte ;
- Form Requests : validation et autorisation de requête ;
- Policies : règles d'accès aux ressources ;
- Services métier : opérations complexes ou réutilisées ;
- Models : relations et comportement de persistance ;
- API Resources : représentation des réponses ;
- Migrations, factories et seeders : structure et données reproductibles ;
- Tests : preuves automatisées du contrat et des permissions.

N'ajoutez pas une couche Repository, DTO ou interface uniquement pour multiplier les fichiers. Chaque abstraction doit répondre à un problème réel de changement, de test ou de réutilisation.

## 30.10. Construire une suite de tests utile

Les tests doivent couvrir les scénarios normaux et les refus.

### Tests d'authentification
- invité refusé sur une route protégée ;
- utilisateur connecté autorisé selon son rôle ;
- token révoqué refusé.

### Tests de propriété et BOLA
- professionnel A peut modifier son service ;
- professionnel B ne peut pas modifier celui de A ;
- un identifiant deviné ne permet pas d'accéder à un contenu privé ;
- l'utilisateur ne peut pas envoyer un propriétaire différent dans le body.

### Tests de publication et recherche
- service non publié absent du catalogue public ;
- service publié et visible retourné ;
- catégorie inactive exclue ;
- filtre invalide rejeté ;
- pagination bornée.

### Tests de fichiers
- format refusé ;
- taille excessive refusée ;
- nombre maximal respecté ;
- image privée inaccessible par un autre compte.

Exécutez la suite après chaque changement significatif :

```bash
php artisan test
```

Un test qui passe n'est une preuve que du scénario réellement vérifié. Il ne garantit pas à lui seul l'absence de vulnérabilité.

## 30.11. Documenter et préparer les outils du client

La documentation OpenAPI doit refléter les routes réellement implémentées :
- méthode et chemin versionné ;
- paramètres ;
- authentification ;
- schémas de requête ;
- réponses de succès ;
- erreurs ;
- exemples réalistes.

La collection Postman doit utiliser une variable de base, par exemple `base_url`, et une variable de token. Ne stockez pas un token réel dans la collection partagée.

Avant livraison :
1. importer la collection ;
2. renseigner l'environnement local ;
3. tester les routes publiques ;
4. obtenir un token de test ;
5. tester les routes protégées ;
6. vérifier les erreurs 401, 403 et 422 ;
7. exporter une version propre et sans secrets.

## 30.12. Déployer et vérifier

Le déploiement suit une procédure reproductible :
1. provisionner l'environnement et la base ;
2. configurer les variables secrètes hors Git ;
3. installer les dépendances de production ;
4. exécuter les migrations avec précaution ;
5. préparer le cache de configuration si adapté ;
6. configurer HTTPS et le stockage ;
7. vérifier les logs et les permissions ;
8. exécuter un test de santé ;
9. effectuer un test de connexion et de consultation publique ;
10. documenter la procédure de retour arrière.

Ne lancez pas une migration destructive en production. Faites une sauvegarde et examinez chaque migration avant son application.

## 30.13. Connecter Flutter ou JavaScript

Le frontend doit :
- utiliser la base URL et le préfixe versionné du contrat ;
- envoyer `Accept: application/json` ;
- envoyer le token dans `Authorization: Bearer` pour les routes Sanctum token ;
- afficher les erreurs de validation 422 champ par champ ;
- gérer les réponses 401 et 403 ;
- respecter les métadonnées de pagination ;
- ne pas afficher comme preuve de permission un bouton masqué dans l'interface : le serveur reste l'autorité.

L'application cliente ne doit jamais déterminer elle-même le propriétaire, le rôle, le statut administratif ou un montant sensible.

## 30.14. Démonstration finale

Préparez une démonstration reproductible :

1. afficher les catégories publiques ;
2. rechercher un service avec filtres ;
3. créer un compte professionnel de démonstration ;
4. créer un profil ;
5. créer un service ;
6. vérifier qu'il reste invisible tant qu'il n'est pas publié ;
7. publier le service après validation ;
8. consulter le service dans le catalogue public ;
9. modifier le service avec son propriétaire ;
10. essayer de le modifier avec un autre compte et montrer le refus ;
11. exécuter les tests ;
12. ouvrir la documentation et la collection Postman.

Cette démonstration prouve les principales règles du projet, pas seulement la présence de pages ou de routes.

## 30.15. Critères d'acceptation du projet

Le MVP peut être considéré comme techniquement livrable lorsque :

- les migrations s'exécutent sur une base vierge ;
- les seeders sont documentés et reproductibles ;
- l'authentification fonctionne et les tokens peuvent être révoqués ;
- les catégories et services respectent les règles de visibilité ;
- les contrôles de propriété sont testés ;
- les entrées sont validées côté serveur ;
- les réponses sont limitées à des Resources explicites ;
- la recherche est filtrée, triée par liste blanche et paginée ;
- les fichiers respectent les règles de validation et d'accès ;
- les tests automatisés passent sur l'environnement cible ;
- OpenAPI et Postman correspondent au comportement réel ;
- le déploiement et le test de santé sont documentés ;
- aucun secret ni donnée privée de démonstration n'est livré.

## Exercices de synthèse

### Exercice 1 — Audit d'une création
Un endpoint accepte `professional_profile_id`, `is_published` et `role` depuis le body. Expliquez les risques et proposez la répartition correcte entre données client et données serveur.

### Exercice 2 — Test BOLA
Écrivez le scénario de test où un professionnel tente de modifier le service d'un autre. Indiquez les acteurs, la préparation, la requête et les assertions.

### Exercice 3 — Contrat de pagination
Définissez les paramètres, les limites et les métadonnées de réponse d'une liste de services.

### Exercice 4 — Mise en production
Établissez une checklist avant et après déploiement, en précisant les actions dangereuses à éviter sur la base de production.

## Corrigés synthétiques

**Exercice 1 :** le client fournit uniquement les champs métier autorisés. Le serveur déduit le profil propriétaire, fixe le statut initial et ne permet pas au client de choisir son rôle. Les permissions sont contrôlées par middleware et Policy.

**Exercice 2 :** créer deux profils et deux services distincts, authentifier le second professionnel, envoyer PATCH sur l'identifiant du premier service, puis vérifier un refus 403 ou 404 selon la politique, sans modification en base.

**Exercice 3 :** `page` entier positif, `per_page` borné (par exemple 1 à 50), filtres validés, tri autorisé uniquement sur des colonnes listées ; réponse avec données et métadonnées cohérentes.

**Exercice 4 :** sauvegarde, variables d'environnement, HTTPS, migrations non destructives, permissions de fichiers, logs, endpoint de santé, test d'authentification et vérification de restauration.

## Conclusion du livre

Une API professionnelle n'est pas simplement un ensemble de routes qui renvoient du JSON. Elle est un contrat stable, un modèle métier cohérent, des permissions vérifiables, des données contrôlées, des tests, une documentation et une procédure d'exploitation.

La qualité du développeur se voit dans sa capacité à anticiper les erreurs, expliquer ses décisions et rendre son système compréhensible par d'autres personnes.

**Dernier conseil :** gardez le contrat, le code, les tests et la documentation synchronisés. Lorsqu'un élément change, vérifiez les trois autres. C'est cette discipline qui transforme un exercice Laravel en projet maintenable.

## Références officielles

- Laravel 13 — Installation : https://laravel.com/docs/13.x/installation
- Laravel 13 — Routing : https://laravel.com/docs/13.x/routing
- Laravel 13 — Sanctum : https://laravel.com/docs/13.x/sanctum
- Laravel 13 — Validation : https://laravel.com/docs/13.x/validation
- Laravel 13 — Authorization : https://laravel.com/docs/13.x/authorization
- Laravel 13 — Eloquent relationships : https://laravel.com/docs/13.x/eloquent-relationships
- Laravel 13 — Testing : https://laravel.com/docs/13.x/testing
- Laravel 13 — Deployment : https://laravel.com/docs/13.x/deployment
- OWASP API Security Top 10 : https://owasp.org/www-project-api-security/
