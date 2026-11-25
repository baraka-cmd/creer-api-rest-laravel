# Chapitre 6 — Comprendre l'architecture Laravel

## Objectifs pédagogiques

Ce chapitre vous apprend à ne pas écrire toute votre application dans un seul contrôleur. Vous allez comprendre le rôle des routes, middleware, contrôleurs, modèles Eloquent, services, Form Requests et API Resources, puis suivre le cycle complet d'une requête.

## 6.1 Une architecture est une organisation des responsabilités

Imaginez une API comme un bureau de traitement de demandes. La route indique où la demande doit être envoyée. Le middleware vérifie certaines conditions avant l'entrée. Le contrôleur coordonne le traitement. Le modèle permet de travailler avec les données. Le service applique une règle métier lorsqu'elle mérite d'être isolée. La ressource API transforme le résultat en réponse publique.

Cette séparation n'est pas une décoration destinée à multiplier les fichiers. Elle permet de répondre à une question importante : **si une règle change, où devons-nous modifier le code ?**

Dans notre projet, la règle « un professionnel ne peut publier qu'un service appartenant à son profil » ne doit pas être dispersée dans plusieurs routes ou copiée dans chaque contrôleur. Elle doit être contrôlée côté serveur, à l'endroit approprié, et testée.

## 6.2 Les routes : déclarer les points d'entrée

Une route associe une méthode HTTP et une URI à un traitement.

```php
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Api\ServiceController;

Route::get('/services', [ServiceController::class, 'index']);
Route::get('/services/{service}', [ServiceController::class, 'show']);
```

La première route correspond à la consultation d'une collection. La seconde correspond à un service identifié par son paramètre.

Les routes doivent décrire l'accès à une ressource, pas contenir toutes les règles métier. Une route avec une longue fonction anonyme qui vérifie le rôle, valide les champs, écrit plusieurs tables et construit un JSON devient rapidement difficile à maintenir.

Pour inspecter les routes :

```bash
php artisan route:list
php artisan route:list --path=api
```

## 6.3 Les middleware : contrôler le passage

Un middleware intervient avant ou après le traitement principal d'une requête. Il peut vérifier l'authentification, appliquer une limitation de fréquence, contrôler certains en-têtes ou effectuer un traitement transversal.

Exemple de route protégée :

```php
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/me', function (\Illuminate\Http\Request $request) {
        return $request->user();
    });
});
```

Le middleware d'authentification répond à la question : « Qui appelle l'API ? ». Il ne suffit pas toujours à répondre à : « Cette personne a-t-elle le droit de modifier ce service précis ? »

Cette seconde question relève de l'autorisation, souvent exprimée par une policy ou une règle métier. Authentification et autorisation sont deux contrôles différents.

## 6.4 Les contrôleurs : coordonner, pas tout faire

Un contrôleur reçoit la requête et coordonne les étapes nécessaires. Il ne devrait pas contenir une longue suite de calculs, de requêtes SQL et de règles difficiles à tester.

Exemple de contrôleur de lecture :

```php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Service;
use App\Http\Resources\ServiceResource;

class ServiceController extends Controller
{
    public function index()
    {
        $services = Service::query()
            ->where('is_published', true)
            ->latest()
            ->paginate(15);

        return ServiceResource::collection($services);
    }
}
```

Cet exemple sera amélioré dans les chapitres sur la validation, les filtres, les autorisations et les performances. Il montre déjà une intention : le contrôleur orchestre la consultation et délègue la mise en forme à une ressource.

## 6.5 Eloquent : travailler avec les données

Eloquent est l'ORM de Laravel. Il permet de représenter une table par un modèle et de manipuler les enregistrements avec une API expressive.

```php
$service = Service::find(12);

if ($service) {
    $service->title;
}
```

Une requête construite avec Eloquent reste une requête de base de données. Il faut donc comprendre les filtres, les relations, les index et les risques de requêtes répétées.

Évitez de croire que le code « ressemble à du PHP » donc qu'il est automatiquement performant. Un appel dans une boucle peut déclencher plusieurs requêtes SQL. Nous étudierons ce problème, appelé N+1, dans le chapitre sur les relations et les performances.

## 6.6 Les services : isoler une opération métier

Un service applicatif devient utile lorsque le traitement représente une opération métier cohérente, réutilisable ou suffisamment complexe pour mériter une classe dédiée.

Exemple : publier une prestation. Cette opération peut vérifier le profil du professionnel, l'état de son compte, la présence des informations obligatoires et les conditions de publication.

```php
namespace App\Services;

use App\Models\Service;
use Illuminate\Support\Facades\DB;
use DomainException;

class PublishService
{
    public function publish(Service $service): Service
    {
        if (!$service->professional->is_active) {
            throw new DomainException(
                'Le profil professionnel doit être actif.'
            );
        }

        return DB::transaction(function () use ($service) {
            $service->is_published = true;
            $service->save();

            return $service->refresh();
        });
    }
}
```

Cet exemple est volontairement simple. Dans une application réelle, il faut également contrôler l'autorisation de l'utilisateur appelant, la validation des données et la cohérence de toutes les conditions de publication. Une transaction protège un ensemble d'écritures lorsqu'elles doivent réussir ou échouer ensemble ; elle ne remplace pas une validation ni une autorisation.

Ne créez pas un service pour chaque ligne de code. Une classe supplémentaire doit apporter une responsabilité claire, une meilleure testabilité ou une réutilisation réelle.

## 6.7 Form Requests : valider les entrées

Les Form Requests regroupent les règles de validation et, si nécessaire, les autorisations liées à une requête.

```bash
php artisan make:request StoreServiceRequest
```

Exemple de règles :

```php
public function rules(): array
{
    return [
        'title' => ['required', 'string', 'min:5', 'max:120'],
        'description' => ['required', 'string', 'max:5000'],
        'price' => ['nullable', 'numeric', 'min:0'],
        'category_id' => ['required', 'integer', 'exists:categories,id'],
    ];
}
```

La validation doit être effectuée côté serveur, même si le formulaire mobile ou web possède déjà des contrôles. Le client peut être modifié ou contourné.

Attention : la règle `exists:categories,id` vérifie l'existence de la catégorie, mais ne prouve pas que celle-ci est autorisée, active ou adaptée au service. Les règles métier et les autorisations restent nécessaires.

## 6.8 API Resources : maîtriser les données exposées

Un modèle Eloquent peut contenir des champs qui ne doivent jamais être renvoyés directement à un client : informations internes, données privées ou attributs non destinés à l'interface.

Une API Resource définit la représentation publique :

```bash
php artisan make:resource ServiceResource
```

```php
public function toArray(\Illuminate\Http\Request $request): array
{
    return [
        'id' => $this->id,
        'title' => $this->title,
        'description' => $this->description,
        'price' => $this->price,
        'published_at' => $this->published_at,
    ];
}
```

La ressource agit comme un contrat de sortie. Si vous ajoutez une colonne sensible au modèle, elle ne devient pas automatiquement publique si votre ressource énumère explicitement les champs autorisés.

## 6.9 Le cycle de traitement d'une requête

Voici une représentation simplifiée :

1. Le client envoie une requête HTTP.
2. Le serveur web transmet la requête à l'application Laravel.
3. Laravel démarre et charge les routes.
4. Le routeur identifie la route correspondante.
5. Les middleware applicables exécutent leurs contrôles.
6. La validation vérifie les données reçues.
7. Le contrôleur coordonne le traitement.
8. Le service applique éventuellement une opération métier.
9. Eloquent lit ou modifie les données.
10. La ressource transforme les données en représentation JSON.
11. Laravel construit la réponse HTTP avec son statut et ses en-têtes.
12. La réponse est renvoyée au client.

Ce schéma est pédagogique : certains composants peuvent intervenir dans un ordre différent selon les middleware, les exceptions, les événements ou la configuration. L'important est de comprendre les responsabilités, pas de mémoriser une liste comme une formule magique.

## 6.10 Atelier : suivre une requête

Appelez `GET /api/services` avec Postman, puis examinez :

- la route qui reçoit la requête ;
- le contrôleur exécuté ;
- la requête Eloquent ;
- la ressource qui produit le JSON ;
- le code HTTP et les en-têtes de réponse.

Utilisez les outils Laravel pour vérifier la route et les journaux, sans afficher de secrets dans les captures.

## Exercices et corrigés

**Exercice 1.** Pourquoi ne faut-il pas placer toutes les règles métier dans le contrôleur ?

**Corrigé :** un contrôleur surchargé devient difficile à lire, réutiliser et tester. Isoler une opération métier permet de lui donner un nom, une responsabilité et des tests dédiés.

**Exercice 2.** Quelle différence entre middleware d'authentification et policy ?

**Corrigé :** le middleware peut vérifier que l'appelant est authentifié ; la policy décide si cet utilisateur peut effectuer une action sur une ressource précise.

**Exercice 3.** Pourquoi utiliser une API Resource au lieu de retourner directement un modèle ?

**Corrigé :** elle contrôle le contrat JSON et limite l'exposition aux champs explicitement prévus.

**Exercice 4.** Dans quel cas une transaction est-elle utile ?

**Corrigé :** lorsque plusieurs écritures doivent être considérées comme une seule opération cohérente. Si l'une échoue, les écritures de la transaction sont annulées.

## À retenir

Laravel fournit des composants qui ont chacun une mission. Une architecture professionnelle les utilise pour rendre le code compréhensible, sécurisé et testable, et non pour multiplier artificiellement les classes.

**Transition :** maintenant que les responsabilités du code sont claires, nous allons concevoir la structure relationnelle de la base, qui constitue le fondement des données de l'API.

## Références

- Laravel, Routing : https://laravel.com/docs/routing
- Laravel, Middleware : https://laravel.com/docs/middleware
- Laravel, Eloquent : https://laravel.com/docs/eloquent
- Laravel, Validation : https://laravel.com/docs/validation
- Laravel, Eloquent Resources : https://laravel.com/docs/eloquent-resources
