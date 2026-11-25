# Cahier de conception — Projet fil conducteur de l'ebook

> Document de référence. Les chapitres de réalisation, les exemples de code, les tests, la documentation API et les annexes doivent respecter ce cahier. Toute évolution du périmètre doit être décidée et répercutée dans les fichiers concernés.

## 1. Présentation du projet

### 1.1. Nom pédagogique

**ProxiServices API** — API REST de gestion et de découverte de services professionnels.

Ce nom désigne le projet d'apprentissage de l'ebook. Il ne prétend pas décrire une entreprise ou une application commerciale déjà publiée.

### 1.2. Problème à résoudre

Un client souhaite trouver un professionnel proposant une prestation précise. Un professionnel veut présenter son profil, publier ses services et gérer les informations qu'il propose. L'administration doit pouvoir contrôler les catégories, les comptes et les contenus pour maintenir la qualité de la plateforme.

Le projet fournit un backend Laravel exposant des ressources JSON. Il doit pouvoir être consommé par une application Flutter, un frontend JavaScript ou un outil de test tel que Postman.

### 1.3. Objectifs pédagogiques

À la fin du projet, le lecteur doit savoir concevoir un contrat REST, organiser une application Laravel, modéliser une base relationnelle, mettre en œuvre l'authentification, appliquer des permissions sur les ressources, créer et modifier des données, rechercher et paginer, gérer des fichiers, écrire des tests, documenter l'API et préparer un déploiement.

Le projet doit être assez complet pour être démontré lors d'une soutenance ou présenté dans un portfolio, tout en restant réalisable dans le cadre d'un ebook.

## 2. Périmètre fonctionnel

### 2.1. Fonctionnalités incluses

La version pédagogique de référence comprend :

1. inscription et connexion ;
2. déconnexion et révocation du token courant ;
3. consultation et modification contrôlée du profil ;
4. rôles utilisateur, client, professionnel et administrateur ;
5. gestion des catégories par l'administration ;
6. création et gestion des services par les professionnels ;
7. consultation publique des services publiés ;
8. recherche, filtres, tri autorisé et pagination ;
9. upload et suppression contrôlée d'images de profil et de service ;
10. permissions et protection contre les accès croisés ;
11. tests automatisés des règles principales ;
12. documentation OpenAPI et collection Postman ;
13. préparation du déploiement et vérifications après publication.

### 2.2. Hors périmètre initial

Pour garder le projet réalisable et pédagogique, les fonctionnalités suivantes ne sont pas obligatoires dans la première version :

- paiement en ligne ;
- portefeuille et transfert d'argent ;
- système complet de commandes et de devis ;
- chat temps réel ;
- géolocalisation GPS privée ;
- application Flutter complète ;
- frontend d'administration complet ;
- intégration de plusieurs fournisseurs de paiement.

Ces fonctionnalités pourront faire l'objet d'une évolution ultérieure. Leur absence ne signifie pas que l'architecture est négligée : le backend doit rester capable d'accueillir de nouvelles fonctionnalités sans être entièrement réécrit.

## 3. Acteurs et rôles

### 3.1. Visiteur non authentifié

Le visiteur peut consulter les catégories actives, les services publiés et visibles, rechercher et filtrer les services, et consulter les informations publiques d'un profil professionnel.

Il ne peut pas créer, modifier ou supprimer une ressource protégée.

### 3.2. Utilisateur authentifié

Un utilisateur possède un compte. Il peut consulter et modifier les informations personnelles autorisées, gérer ses tokens selon les fonctionnalités implémentées et accéder aux routes réservées à son rôle.

Être connecté ne signifie pas avoir le droit de modifier toutes les ressources.

### 3.3. Client

Le client recherche des prestations. Dans le périmètre initial, il peut gérer son profil et consulter les services. La création de demandes ou de commandes est réservée à une extension future.

### 3.4. Professionnel

Le professionnel possède un profil associé à son compte. Il peut créer un service, modifier ses propres services, publier ou dépublier selon les règles et gérer les images autorisées.

Il ne peut pas modifier les services d'un autre professionnel en remplaçant simplement un identifiant dans l'URL.

### 3.5. Administrateur

L'administrateur peut gérer les catégories, consulter les comptes selon ses permissions, contrôler la visibilité ou l'état des services et effectuer les opérations administratives prévues.

Les privilèges d'administration sont attribués côté serveur. Ils ne sont jamais acceptés depuis un formulaire public.

## 4. Règles métier fondamentales

Ces règles constituent les invariants du projet. Les contrôleurs, services, policies et tests doivent les respecter.

1. Un compte possède une identité unique, notamment une adresse e-mail unique selon la stratégie de normalisation retenue.
2. Un utilisateur ne modifie que les champs de profil autorisés.
3. Un professionnel ne gère que les services qui lui appartiennent.
4. Un service public doit être actif, publié et associé à une catégorie utilisable.
5. Un service brouillon ou suspendu n'apparaît pas dans la recherche publique.
6. Une catégorie inactive ne peut pas être utilisée pour une nouvelle publication si la règle métier l'interdit.
7. Le rôle, le propriétaire, l'état de publication et les champs administratifs ne sont pas confiés au client lors d'une opération ordinaire.
8. Toute opération sensible vérifie l'autorisation côté serveur.
9. Les fichiers sont validés, nommés par le serveur et stockés selon leur visibilité.
10. Un identifiant ou une URL de fichier ne constitue pas à lui seul une autorisation d'accès.
11. Les réponses JSON n'exposent que les champs prévus.
12. Les erreurs ne divulguent ni secrets, ni traces internes, ni données d'autres utilisateurs.

## 5. Modèle de données de référence

Le schéma sera traduit en migrations Laravel puis vérifié avant la rédaction des exemples de code.

### 5.1. Table users

Champs principaux :

- id : identifiant ;
- name : nom d'affichage ;
- email : adresse unique ;
- password : hash du mot de passe ;
- role ou relation de rôles, selon la solution retenue ;
- is_active : état du compte ;
- email_verified_at : date de vérification si cette fonctionnalité est incluse ;
- created_at et updated_at.

Le mot de passe n'est jamais renvoyé par une Resource. Pour ce projet pédagogique, une colonne de rôle contrôlée côté serveur peut suffire si les rôles restent simples et stables. Une bibliothèque RBAC complète ne sera ajoutée que si le besoin le justifie.

### 5.2. Table professional_profiles

Champs principaux :

- id ;
- user_id : relation contrôlée avec users ;
- display_name ;
- bio ;
- phone_public : uniquement si le professionnel choisit de le rendre public ;
- avatar_path ;
- is_verified ;
- is_public ;
- timestamps.

La vérification du professionnel et la visibilité publique sont des décisions serveur. Le client ne peut pas s'auto-déclarer vérifié en envoyant une valeur true.

### 5.3. Table categories

Champs principaux :

- id ;
- name ;
- slug unique ;
- description nullable ;
- is_active ;
- timestamps.

Le slug doit être stable et unique. La désactivation d'une catégorie doit être réfléchie : les services existants peuvent rester en base, mais ne doivent plus être utilisés pour de nouvelles publications si la règle l'interdit.

### 5.4. Table services

Champs principaux :

- id ;
- professional_profile_id ;
- category_id ;
- title ;
- slug ou identifiant public selon la stratégie ;
- description ;
- price nullable selon le type de prestation ;
- currency si le prix est affiché ;
- status : brouillon, publié ou suspendu ;
- timestamps.

Le prix et la devise sont des données métier. Leur format, précision et règles d'affichage doivent être définis avant le code. Si des opérations financières sont ajoutées ultérieurement, les calculs ne doivent jamais dépendre aveuglément d'un montant envoyé par le frontend.

### 5.5. Table service_images

Champs principaux :

- id ;
- service_id ;
- path ;
- alt_text nullable ;
- position ;
- timestamps.

Le nombre maximal d'images, la taille maximale et les formats autorisés seront définis dans les règles de validation. Les fichiers ne doivent pas être nommés à partir du nom original fourni par le client.

### 5.6. Authentification et tokens

L'authentification API utilisera Laravel Sanctum pour les tokens de clients mobiles et de clients API. Le token brut n'est présenté qu'au moment de sa création et ne doit pas être enregistré dans les logs.

Les tokens doivent pouvoir être révoqués. Le manuscrit distinguera l'authentification par token pour un client mobile ou tiers et l'authentification par session/cookies pour une SPA de première partie, si cette dernière est abordée.

## 6. Relations Eloquent attendues

- User possède éventuellement un ProfessionalProfile.
- ProfessionalProfile appartient à User.
- ProfessionalProfile possède plusieurs Service.
- Category possède plusieurs Service.
- Service appartient à ProfessionalProfile et à Category.
- Service possède plusieurs ServiceImage.

Les relations seront déclarées explicitement et utilisées dans les requêtes. Les listes publiques devront prévoir l'eager loading pour éviter les requêtes N+1.

## 7. Contrat REST de référence

Le préfixe de la première version sera /api/v1. Le chapitre consacré à l'évolution expliquera le versionnement et les changements incompatibles.

### 7.1. Authentification

| Méthode | Endpoint | Accès | Fonction |
|---|---|---|---|
| POST | /api/v1/auth/register | Public | Créer un compte |
| POST | /api/v1/auth/login | Public | Authentifier |
| POST | /api/v1/auth/logout | Authentifié | Révoquer le token courant |
| GET | /api/v1/me | Authentifié | Lire le profil courant |
| PATCH | /api/v1/me | Authentifié | Modifier les champs autorisés |

### 7.2. Catégories

| Méthode | Endpoint | Accès | Fonction |
|---|---|---|---|
| GET | /api/v1/categories | Public | Lister les catégories actives |
| GET | /api/v1/categories/{category} | Public | Consulter une catégorie |
| POST | /api/v1/admin/categories | Admin | Créer une catégorie |
| PATCH | /api/v1/admin/categories/{category} | Admin | Modifier une catégorie |
| DELETE | /api/v1/admin/categories/{category} | Admin | Supprimer ou désactiver selon l'intégrité |

### 7.3. Services

| Méthode | Endpoint | Accès | Fonction |
|---|---|---|---|
| GET | /api/v1/services | Public | Recherche et pagination |
| GET | /api/v1/services/{service} | Public selon visibilité | Consulter un service |
| POST | /api/v1/professional/services | Professionnel | Créer un service |
| PATCH | /api/v1/professional/services/{service} | Propriétaire | Modifier son service |
| DELETE | /api/v1/professional/services/{service} | Propriétaire | Supprimer ou archiver |
| PATCH | /api/v1/professional/services/{service}/publish | Propriétaire autorisé | Publier |
| PATCH | /api/v1/professional/services/{service}/unpublish | Propriétaire autorisé | Dépublier |

Les routes de publication sont séparées des modifications ordinaires afin de rendre la transition métier explicite. Le champ status ne doit pas être librement modifiable dans un formulaire général.

### 7.4. Fichiers

Les routes d'upload et de suppression seront définies précisément lors de l'implémentation. Elles doivent être authentifiées et autorisées par ressource. Le projet distinguera les images de profil, les images de service et, si retenu, un éventuel fichier privé.

La réponse fournit une représentation sûre du fichier, pas un chemin système.

## 8. Recherche, filtres et pagination

La route publique de liste des services accepte uniquement les paramètres prévus, par exemple :

- q : recherche textuelle ;
- category_id : catégorie ;
- professional_id : professionnel public ;
- min_price et max_price si les prix sont définis ;
- sort : valeur parmi une liste autorisée ;
- per_page : taille de page bornée ;
- page : pagination classique, ou curseur si la stratégie est retenue.

Les noms de colonnes ne sont jamais injectés directement depuis une valeur libre du client. Le tri est traduit par une liste blanche côté serveur.

Les résultats sont limités aux services dont la visibilité publique est autorisée. Les filtres ne doivent pas permettre de contourner cette condition.

## 9. Architecture Laravel cible

L'architecture reste simple et cohérente avec les chapitres précédents :

- Routes : déclarent les endpoints et middlewares.
- Controllers : orchestrent la requête HTTP.
- Form Requests : valident les données et les conditions d'accès lorsque cela convient.
- Policies : vérifient l'autorisation sur les ressources.
- Services métier : portent les opérations qui comportent plusieurs étapes ou des règles réutilisées.
- Models et relations Eloquent : représentent les données et leurs associations.
- API Resources : contrôlent la représentation JSON.
- Migrations et seeders : construisent et alimentent la base.
- Tests : vérifient les comportements publics, protégés et interdits.

Il n'est pas prévu d'ajouter systématiquement Repository, DTO ou interfaces pour chaque modèle. Toute abstraction doit répondre à un besoin identifiable et être expliquée.

## 10. Politique de sécurité

La sécurité sera démontrée par des règles et des tests, pas uniquement par des explications théoriques.

### Authentification
- mot de passe hashé ;
- token traité comme un secret ;
- routes privées protégées ;
- révocation possible ;
- erreurs d'authentification cohérentes.

### Autorisation
- rôle vérifié côté serveur ;
- ownership contrôlé par Policy ;
- tests d'accès entre deux utilisateurs ;
- absence de confiance dans les identifiants et états envoyés par le frontend.

### Validation et mass assignment
- champs autorisés définis explicitement ;
- données validées utilisées pour les écritures ;
- propriétés sensibles exclues des formulaires ordinaires ;
- règles précises pour les fichiers.

### Exposition des données
- Resources dédiées ;
- données privées exclues ;
- relations chargées de façon contrôlée ;
- erreurs sans traces internes.

### Abus
- limitation des tentatives sensibles ;
- limites de pagination et d'upload ;
- CORS configuré selon les clients réels ;
- vérification des dépendances.

## 11. Stratégie de tests

Le projet doit comporter au minimum les familles de tests suivantes.

### Tests d'authentification
- inscription valide ;
- adresse e-mail déjà utilisée ;
- connexion valide et invalide ;
- accès sans token ;
- déconnexion et token révoqué.

### Tests d'autorisation
- un client ne modifie pas un service ;
- un professionnel modifie son propre service ;
- le professionnel A ne modifie pas le service du professionnel B ;
- un utilisateur ordinaire n'accède pas aux routes d'administration ;
- un rôle ou statut forgé dans la requête est ignoré ou refusé.

### Tests métier
- service créé avec des données valides ;
- catégorie inactive refusée pour publication ;
- service brouillon absent de la liste publique ;
- publication autorisée uniquement lorsque les conditions sont remplies.

### Tests de recherche
- filtre par catégorie ;
- recherche textuelle ;
- tri autorisé ;
- pagination bornée ;
- résultats invisibles exclus.

### Tests de fichiers
- format interdit refusé ;
- taille excessive refusée ;
- suppression par un non-propriétaire interdite ;
- fichier privé inaccessible sans autorisation, si cette fonction est incluse.

### Tests de contrat
- champs JSON attendus ;
- statuts HTTP ;
- structure de pagination ;
- absence de champs sensibles.

## 12. Documentation et ressources livrées

Le projet final doit produire :

- documentation OpenAPI des endpoints ;
- collection Postman organisée par domaine ;
- environnement Postman sans secret réel ;
- jeu de données de démonstration ;
- comptes de test créés par un seeder local contrôlé ;
- README d'installation et d'exécution ;
- checklist de sécurité ;
- checklist de déploiement ;
- procédure de test et de démonstration.

Les identifiants de démonstration ne doivent pas être des comptes réels ni être utilisés en production. Les secrets sont remplacés par des valeurs d'exemple non fonctionnelles.

## 13. Déploiement de référence

Le chapitre final expliquera un déploiement cloud sur une plateforme compatible Laravel. Les étapes exactes seront adaptées à l'hébergeur retenu et vérifiées dans sa documentation au moment de la rédaction.

La procédure générale comprend :

1. préparer le dépôt et les dépendances ;
2. configurer les variables d'environnement ;
3. provisionner la base ;
4. appliquer les migrations approuvées ;
5. configurer le stockage ;
6. activer HTTPS et le domaine ;
7. lancer l'application et les workers nécessaires ;
8. effectuer les smoke tests ;
9. vérifier les logs et les sauvegardes ;
10. documenter le retour arrière.

## 14. Critères d'acceptation du projet

Le projet sera considéré comme pédagogiquement terminé lorsque :

- les fonctionnalités incluses sont implémentées ;
- les routes correspondent au contrat documenté ;
- les migrations s'exécutent dans l'ordre ;
- les seeders de démonstration sont reproductibles ;
- les tests automatisés principaux passent ;
- les permissions sont testées avec plusieurs utilisateurs ;
- les réponses JSON ne divulguent pas de champs sensibles ;
- la recherche et la pagination fonctionnent ;
- les fichiers sont validés et protégés ;
- la documentation OpenAPI est cohérente ;
- la collection Postman permet une démonstration ;
- l'installation est décrite dans le README ;
- la procédure de déploiement est documentée et vérifiée.

## 15. Versions et vérification technique

Le manuscrit doit annoncer clairement les versions utilisées dans les exemples et distinguer :

- version minimale requise ;
- version réellement testée ;
- version de la documentation consultée ;
- extensions PHP et services nécessaires.

La documentation officielle Laravel 13 indique PHP 8.3 comme version minimale et précise les versions PHP supportées dans ses notes de version. La version exacte de Laravel, PHP, Sanctum, Composer et de la base de données devra être figée dans un fichier de référence du projet avant la rédaction définitive des commandes et captures.

Sources officielles à consulter et citer dans le manuscrit :

- Laravel 13 — Déploiement : https://laravel.com/docs/13.x/deployment
- Laravel 13 — Versions et support : https://laravel.com/docs/13.x/releases
- Laravel 13 — Authentification : https://laravel.com/docs/13.x/authentication
- Laravel Sanctum : https://laravel.com/docs/13.x/sanctum
- Laravel — Routage : https://laravel.com/docs/13.x/routing

Les pages officielles doivent être relues au moment de la publication. Les commandes, chemins de configuration et captures ne doivent pas être copiés depuis une version différente sans vérification.

## 16. Décisions à conserver pendant la rédaction

- Le projet est une API REST de services professionnels, pas une plateforme de paiement complète.
- La première version du contrat est /api/v1.
- Les données sensibles ne sont jamais exposées directement par Eloquent.
- L'autorisation se vérifie au niveau de la ressource et de l'action.
- Les exemples de code doivent être cohérents entre chapitres.
- Les opérations multi-étapes doivent être examinées pour les transactions et la concurrence.
- Les tests doivent démontrer les accès autorisés autant que les accès interdits.
- La publication du livre exige une revue technique, une vérification des liens et un test de lecture du PDF et de l'EPUB.

Ce cahier constitue le socle du chapitre 30. Toute implémentation présentée dans le manuscrit doit respecter ce périmètre, ou signaler clairement qu'elle appartient à une extension.
