# Chapitre 5 — Créer et configurer un projet Laravel API

## Objectifs pédagogiques

À la fin de ce chapitre, vous saurez créer un projet Laravel propre, vérifier que l'environnement utilisé est le bon, configurer une base de données sans exposer ses identifiants, comprendre les principaux dossiers et préparer des routes dédiées à l'API.

Nous allons construire l'API du projet fil rouge **ProxiServices** : une plateforme sur laquelle un client peut consulter des prestations professionnelles et, plus tard, envoyer une demande à un prestataire.

## 5.1 Pourquoi commencer par un projet propre ?

Un projet Laravel n'est pas simplement un ensemble de fichiers PHP. Le framework fournit une organisation, un système de configuration, un conteneur de services, un routeur, un mécanisme de validation, une couche d'accès aux données et des outils de test.

Créer le projet correctement dès le départ permet d'éviter plusieurs problèmes : dépendances incompatibles, mauvaise version de PHP, configuration de base de données incohérente, fichiers secrets publiés ou routes dispersées.

Avant de lancer une commande, vérifiez les prérequis. Dans un terminal, exécutez :

```bash
php -v
composer --version
```

La première commande affiche la version de PHP utilisée par le terminal. La seconde confirme que Composer est disponible. Composer est le gestionnaire de dépendances PHP : il télécharge les bibliothèques déclarées par le projet et maintient leur cohérence.

**Point de vigilance Windows :** si vous utilisez XAMPP, plusieurs installations de PHP peuvent coexister. Le PHP du terminal peut être différent de celui utilisé par Apache. La commande `where php` permet d'identifier les exécutables trouvés dans le PATH :

```powershell
where php
```

Si Laravel annonce une extension manquante ou une version incorrecte, ne modifiez pas les fichiers au hasard. Identifiez d'abord le PHP réellement exécuté et son fichier `php.ini`.

## 5.2 Créer l'application

Dans le répertoire où vous conservez vos projets, lancez :

```bash
composer create-project laravel/laravel proxiservices-api
cd proxiservices-api
```

Composer crée le dossier, installe les dépendances et prépare les fichiers Laravel. Le nom `proxiservices-api` est un nom de projet ; il ne détermine pas à lui seul le nom de la base de données ni l'adresse publique de l'API.

Vérifiez ensuite l'installation :

```bash
php artisan --version
php artisan about
```

Artisan est l'interface en ligne de commande de Laravel. Il permet notamment de créer des classes, exécuter les migrations, lancer les tests et démarrer le serveur de développement.

Démarrez l'application :

```bash
php artisan serve
```

Le terminal affiche l'adresse locale du serveur. Ouvrez cette adresse dans votre navigateur. Le serveur de développement est destiné aux essais locaux ; il ne remplace pas une configuration de production.

## 5.3 Comprendre le fichier .env

Le fichier `.env` contient les valeurs propres à votre environnement : nom de l'application, mode d'exécution, adresse, paramètres de base de données et autres secrets.

Exemple de configuration locale :

```dotenv
APP_NAME="ProxiServices API"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://127.0.0.1:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=proxiservices
DB_USERNAME=root
DB_PASSWORD=
```

Adaptez les identifiants à votre installation. Dans certains environnements XAMPP locaux, le compte de développement peut ne pas avoir de mot de passe ; ce choix ne doit jamais être reproduit automatiquement en production.

### Le fichier .env.example

Le dépôt contient normalement un fichier `.env.example`. Il présente les variables nécessaires sans contenir les véritables secrets. Un développeur qui récupère le dépôt peut copier ce fichier :

```bash
cp .env.example .env
php artisan key:generate
```

Sous Windows, vous pouvez copier le fichier avec l'Explorateur ou PowerShell. La commande `key:generate` crée la clé utilisée par les mécanismes de chiffrement Laravel.

**Règle de sécurité :** ne commitez jamais votre fichier `.env`. Il doit être ignoré par Git. Ne placez pas de mot de passe, de clé API ou de secret de paiement dans un fichier public, une capture d'écran ou un exemple de code.

## 5.4 Connecter la base de données

Laravel utilise la configuration `DB_*` pour établir la connexion. Dans votre gestionnaire MariaDB ou MySQL, créez une base vide nommée `proxiservices`, avec un jeu de caractères adapté, généralement `utf8mb4`.

Une base vide est préférable à une base remplie manuellement de tables dont Laravel ne connaît pas l'origine. Les migrations vont devenir la source versionnée de la structure.

Après avoir enregistré `.env`, vérifiez la configuration :

```bash
php artisan config:clear
php artisan migrate
```

La première commande efface le cache de configuration local. La seconde exécute les migrations présentes. Si une connexion échoue, vérifiez dans cet ordre : service de base de données démarré, port, nom de base, utilisateur, mot de passe, puis configuration PHP et pilotes disponibles.

Une erreur de connexion ne signifie pas nécessairement que le code Laravel est défectueux. Elle peut venir du service MariaDB arrêté ou d'un port déjà utilisé.

## 5.5 Lire la structure du projet

Voici les dossiers que vous rencontrerez souvent :

- `app/` : classes applicatives, notamment modèles, contrôleurs, services et règles métier.
- `bootstrap/` : démarrage et configuration initiale du framework.
- `config/` : fichiers de configuration.
- `database/` : migrations, factories et seeders.
- `public/` : point d'entrée web et ressources publiques.
- `resources/` : vues et ressources front-end.
- `routes/` : définition des routes.
- `storage/` : fichiers générés, journaux et fichiers stockés.
- `tests/` : tests automatisés.
- `vendor/` : dépendances installées par Composer ; ce dossier n'est généralement pas versionné.

Ne confondez pas `app/Models` et `database/migrations`. Le modèle décrit le comportement et l'accès à une entité dans le code ; la migration décrit l'évolution de la structure de la base.

## 5.6 Préparer les routes API

Selon la version et le squelette Laravel installé, les routes API peuvent nécessiter l'installation de la fonctionnalité API. Consultez la commande Artisan disponible dans votre version et la documentation officielle correspondant exactement à cette version. Sur les versions qui proposent cette installation :

```bash
php artisan install:api
```

Cette opération prépare notamment les routes API et l'intégration de Sanctum. Elle ne signifie pas que l'authentification est déjà configurée pour tous vos besoins : nous l'étudierons dans le chapitre consacré à la sécurité et aux tokens.

Dans `routes/api.php`, définissons une première route de vérification :

```php
<?php

use Illuminate\Support\Facades\Route;

Route::get('/health', function () {
    return response()->json([
        'status' => 'ok',
        'application' => 'ProxiServices API',
    ]);
});
```

Les routes du fichier API sont généralement préfixées par `/api`. L'adresse à tester est donc :

```
GET http://127.0.0.1:8000/api/health
```

La réponse doit être du JSON. Cette route de santé ne doit pas exposer de mots de passe, de configuration interne ou de détails sensibles.

## 5.7 Atelier pratique : première vérification complète

1. Créez le projet.
2. Vérifiez PHP, Composer et Artisan.
3. Configurez `.env`.
4. Créez la base de données.
5. Exécutez les migrations.
6. Préparez les routes API.
7. Appelez `/api/health` dans Postman.

### Diagnostic guidé

**Symptôme :** `php artisan migrate` indique une erreur de connexion.

- Vérifiez que MariaDB/MySQL est démarré.
- Contrôlez le port configuré.
- Vérifiez le nom de la base.
- Confirmez que le compte dispose des droits nécessaires.
- Effacez le cache de configuration après modification de `.env`.

**Symptôme :** `/api/health` renvoie 404.

- Vérifiez que la route se trouve dans le fichier API chargé.
- Exécutez `php artisan route:list --path=api`.
- Contrôlez le préfixe de l'URL.
- Vérifiez que le serveur lancé correspond au dossier du projet.

## Exercices

1. Expliquez la différence entre `.env` et `.env.example`.
2. Pourquoi le dossier `vendor` n'est-il généralement pas envoyé dans Git ?
3. Quelle est la différence entre une migration et un modèle ?
4. Créez une route `GET /api/version` qui retourne le nom de l'API et une version fictive.

### Corrigé commenté

1. `.env` contient les valeurs réelles de l'environnement et des secrets ; `.env.example` documente les variables sans divulguer les secrets.
2. Les dépendances sont décrites par Composer et peuvent être réinstallées. Versionner `vendor` alourdit le dépôt et complique les mises à jour.
3. La migration définit ou modifie la structure SQL ; le modèle représente une entité et facilite son utilisation depuis le code.
4. Exemple :

```php
Route::get('/version', function () {
    return response()->json([
        'name' => 'ProxiServices API',
        'version' => '1.0.0',
    ]);
});
```

## À retenir

Un projet API professionnel commence par un environnement maîtrisé, une configuration séparée du code, des secrets protégés et des routes clairement organisées. Ne poursuivez pas vers les fonctionnalités métier tant que cette base n'est pas vérifiée.

**Transition :** au chapitre suivant, nous allons suivre une requête Laravel depuis son arrivée jusqu'à sa réponse, afin de comprendre pourquoi chaque composant doit avoir une responsabilité précise.

## Références techniques à vérifier pour l'édition finale

- Documentation officielle Laravel, installation et configuration : https://laravel.com/docs
- Documentation Laravel, routing : https://laravel.com/docs/routing
- Documentation Composer : https://getcomposer.org/doc/
- Documentation PHP, configuration : https://www.php.net/manual/en/configuration.php
