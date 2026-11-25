# Chapitre 7 — Concevoir une base de données professionnelle

## Objectifs pédagogiques

Vous apprendrez à transformer les besoins métier en tables relationnelles, choisir les relations adaptées, écrire des migrations lisibles, utiliser les clés étrangères et les index, protéger l'intégrité des données et préparer des données de test reproductibles.

## 7.1 Partir du métier, pas des colonnes

Une base de données doit représenter les règles et les objets réels de l'application. Pour ProxiServices, nous avons notamment :

- des utilisateurs ;
- des profils professionnels ;
- des catégories ;
- des prestations ;
- des demandes de clients.

Avant de créer les tables, posez des questions métier :

- Un utilisateur peut-il être professionnel et client ?
- Un professionnel peut-il publier plusieurs prestations ?
- Une prestation appartient-elle à plusieurs catégories ?
- Une demande peut-elle concerner plusieurs prestations ?
- Que doit-il se passer lorsqu'un utilisateur est supprimé ?

Ces questions déterminent le modèle. Une mauvaise réponse au départ peut conduire à des duplications, des données incohérentes ou des migrations coûteuses.

## 7.2 Une table représente un type d'information

Une table contient des lignes et des colonnes. Chaque ligne représente un enregistrement ; chaque colonne décrit un attribut.

Exemple conceptuel de table `services` :

| Colonne | Rôle |
|---|---|
| id | Identifiant unique |
| professional_profile_id | Propriétaire professionnel |
| category_id | Catégorie principale |
| title | Titre de la prestation |
| description | Description |
| price | Prix annoncé, si applicable |
| status | État du service (draft, published, suspended) |
| created_at | Date de création |
| updated_at | Date de modification |

Évitez les colonnes vagues telles que `data1`, `info` ou `status2`. Un nom précis facilite les requêtes, les tests et la maintenance.

## 7.3 Clé primaire et clés étrangères

La clé primaire identifie chaque ligne de manière unique. Laravel utilise couramment un identifiant auto-incrémenté nommé `id`, mais d'autres stratégies existent.

Une clé étrangère relie une colonne à l'identifiant d'une autre table et aide à empêcher les références inexistantes.

Exemple : une prestation appartient à un profil professionnel. La colonne `professional_profile_id` doit référencer un professionnel existant. Sans contrainte, une erreur applicative pourrait créer une prestation liée à un profil inexistant.

## 7.4 Comprendre les relations

### Relation un-à-un (one-to-one)

Un utilisateur possède un profil professionnel unique dans le modèle retenu. La relation est un-à-un si chaque utilisateur ne peut avoir qu'un profil et chaque profil appartient à un seul utilisateur.

Dans la base, `professional_profiles.user_id` doit être unique. Une simple clé étrangère ne suffit pas à imposer l'unicité.

### Relation un-à-plusieurs (one-to-many)

Un professionnel peut publier plusieurs prestations, mais chaque prestation appartient à un seul professionnel.

```
professional_profiles (1) ---- (N) services
```

La clé étrangère se place du côté « plusieurs », donc dans la table `services`.

### Relation plusieurs-à-plusieurs (many-to-many)

Une prestation peut être associée à plusieurs compétences ou étiquettes, et une compétence peut caractériser plusieurs prestations. Une table pivot relie les deux.

```
services (N) ---- service_skill (pivot) ---- (N) skills
```

La table pivot contient généralement les deux clés étrangères et une contrainte unique sur leur combinaison pour empêcher les associations en double.

## 7.5 Exemple de migrations Laravel

Créez un modèle et sa migration :

```bash
php artisan make:model ProfessionalProfile -m
php artisan make:model Service -m
```

Exemple simplifié de migration des profils :

```php
Schema::create('professional_profiles', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')
        ->unique()
        ->constrained()
        ->cascadeOnDelete();
    $table->string('display_name', 120);
    $table->text('description')->nullable();
    $table->boolean('is_active')->default(false);
    $table->timestamps();
});
```

La contrainte `unique()` garantit un seul profil par utilisateur. `constrained()` crée la référence vers la table correspondante selon les conventions Laravel. `cascadeOnDelete()` signifie que la suppression du parent supprime ses profils liés : choisissez cette règle seulement si elle correspond réellement à votre politique de conservation.

Dans un système professionnel, la suppression d'un utilisateur peut nécessiter une désactivation, une anonymisation ou une conservation d'historique plutôt qu'une suppression en cascade. Le choix dépend du métier, des obligations de conservation et des relations existantes.

Migration simplifiée des prestations :

```php
Schema::create('services', function (Blueprint $table) {
    $table->id();
    $table->foreignId('professional_profile_id')
        ->constrained()
        ->restrictOnDelete();
    $table->foreignId('category_id')
        ->constrained()
        ->restrictOnDelete();
    $table->string('title', 150);
    $table->text('description');
    $table->decimal('price', 12, 2)->nullable();
    $table->string('status', 20)->default('draft');
    $table->timestamps();

    $table->index(['status', 'category_id']);
});
```

La précision décimale doit correspondre aux règles monétaires du projet. Ne calculez pas des montants financiers avec des nombres flottants sans stratégie adaptée. Pour une application multi-devises, la devise doit être représentée explicitement et les règles de conversion doivent être définies.

## 7.6 Les migrations : versionner la structure

Une migration est un fichier PHP qui décrit une modification de la base. Elle permet à chaque développeur et à chaque environnement de construire la même structure à partir du dépôt.

Commandes utiles :

```bash
php artisan migrate
php artisan migrate:status
php artisan migrate:rollback
```

La commande `rollback` annule le dernier lot de migrations ; elle peut supprimer des données si la méthode `down()` détruit une table ou une colonne. Ne l'exécutez pas sur une base importante sans sauvegarde et sans comprendre son effet.

Une migration doit être cohérente avec les migrations précédentes. L'ordre est important lorsqu'une table dépend d'une autre. Les noms de fichiers comportent un horodatage, ce qui permet à Laravel de les exécuter dans l'ordre.

Évitez de modifier une migration déjà appliquée dans un environnement partagé comme si elle n'avait jamais existé. Pour une évolution, créez généralement une nouvelle migration. Cela permet aux autres environnements de reproduire l'évolution.

## 7.7 Contraintes d'intégrité

L'intégrité des données ne doit pas dépendre uniquement du formulaire.

Les contraintes courantes comprennent :

- clé primaire ;
- clé étrangère ;
- unicité ;
- valeurs obligatoires ou nullables ;
- valeurs par défaut ;
- contraintes de domaine, selon le moteur et la version ;
- vérification métier dans l'application.

Exemple : un email doit être unique si le système exige un compte par adresse. Une règle de validation Laravel améliore le message renvoyé au client, tandis qu'une contrainte unique dans la base protège contre deux requêtes concurrentes.

C'est une défense complémentaire : la validation et la contrainte SQL n'ont pas le même rôle.

## 7.8 Les index : accélérer les recherches pertinentes

Un index aide le moteur à retrouver des lignes sans parcourir inutilement toute la table. Il est particulièrement utile sur les colonnes utilisées fréquemment dans les filtres, les jointures ou les tris.

Exemple : si l'API recherche souvent les prestations publiées par catégorie, un index adapté peut améliorer la requête.

Cependant, un index n'est pas gratuit : il consomme de l'espace et ralentit certaines écritures puisqu'il doit être maintenu. Ne créez pas un index sur chaque colonne. Utilisez les requêtes réelles, les volumes et les outils d'analyse du moteur pour décider.

## 7.9 Seeders et données de test

Un seeder remplit la base avec des données initiales ou de démonstration.

```bash
php artisan make:seeder CategorySeeder
```

Exemple :

```php
DB::table('categories')->insert([
    ['name' => 'Plomberie', 'slug' => 'plomberie'],
    ['name' => 'Électricité', 'slug' => 'electricite'],
    ['name' => 'Informatique', 'slug' => 'informatique'],
]);
```

Le seeder doit être reproductible. S'il est exécuté deux fois, il ne devrait pas créer des doublons involontaires. Pour des données fixes, vous pouvez utiliser une stratégie d'insertion ou de mise à jour adaptée, par exemple `updateOrInsert`, et définir une clé stable.

Les factories servent à générer des données de test variées. Elles sont particulièrement utiles pour vérifier la pagination, les filtres et les autorisations sur plusieurs utilisateurs.

**Attention :** ne placez jamais de véritables données personnelles de clients dans un seeder de démonstration public.

## 7.10 Atelier de modélisation

Construisez les tables suivantes pour le premier périmètre :

- `users` : comptes ;
- `professional_profiles` : profils ;
- `categories` : catégories ;
- `services` : prestations.

Dessinez les relations, identifiez les clés étrangères, puis rédigez les migrations. Exécutez-les sur une base de développement vide et contrôlez le résultat dans votre outil SQL.

## Exercices et corrigés

**Exercice 1.** Où placer la clé étrangère dans une relation un-à-plusieurs entre un professionnel et ses prestations ?

**Corrigé :** dans `services`, car chaque prestation référence un professionnel, tandis qu'un professionnel peut être référencé par plusieurs prestations.

**Exercice 2.** Pourquoi une règle `unique` est-elle nécessaire pour une relation un-à-un ?

**Corrigé :** la clé étrangère garantit l'existence du parent, mais sans unicité plusieurs lignes pourraient référencer le même utilisateur.

**Exercice 3.** Pourquoi ne faut-il pas modifier systématiquement une ancienne migration déjà exécutée ?

**Corrigé :** les environnements qui l'ont déjà exécutée ne rejoueront pas automatiquement le nouveau contenu. Une migration d'évolution conserve une histoire reproductible.

**Exercice 4.** Quelle différence entre un seeder et une factory ?

**Corrigé :** un seeder prépare des données définies, par exemple les catégories officielles ; une factory génère des enregistrements variés pour les tests.

## À retenir

Une base professionnelle traduit les règles métier en relations, contraintes et migrations reproductibles. Les décisions de suppression, d'unicité, de précision monétaire et d'indexation doivent être réfléchies avant la mise en production.

**Transition :** nous possédons maintenant une structure de données. Le chapitre suivant va relier modèle, contrôleur et routes pour réaliser les premières opérations CRUD et les tester dans Postman.

## Références

- Laravel, Migrations : https://laravel.com/docs/migrations
- Laravel, Eloquent Relationships : https://laravel.com/docs/eloquent-relationships
- Laravel, Seeders : https://laravel.com/docs/seeding
- Laravel, Factories : https://laravel.com/docs/eloquent-factories
- MariaDB, documentation : https://mariadb.com/kb/en/documentation/


> **Convention de référence du projet fil conducteur (mise à jour éditoriale).** Les exemples d'URL du projet utilisent le préfixe public `/api/v1`. La propriété propriétaire d'un service est `professional_profile_id` et référence le profil professionnel, non directement l'identifiant utilisateur. Le serveur déduit ce propriétaire de l'utilisateur authentifié ; cette valeur ne doit jamais être acceptée aveuglément depuis le client. Les chapitres d'extension doivent respecter ces conventions ou annoncer explicitement une variante.
