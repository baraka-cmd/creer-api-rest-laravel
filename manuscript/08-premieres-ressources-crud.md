# Chapitre 8 — Créer les premières ressources API

## Objectifs pédagogiques

À la fin de ce chapitre, vous saurez créer un modèle Eloquent, générer un contrôleur API, définir les routes d'une ressource, organiser les opérations CRUD, renvoyer des réponses JSON cohérentes et tester chaque opération dans Postman.

Nous allons mettre en place une première ressource `Service`, représentant une prestation professionnelle.

## 8.1 Comprendre CRUD dans une API

CRUD désigne quatre opérations fondamentales :

- **Create** : créer un enregistrement ;
- **Read** : consulter un enregistrement ou une collection ;
- **Update** : modifier un enregistrement ;
- **Delete** : supprimer un enregistrement.

En HTTP, ces opérations sont souvent associées à POST, GET, PUT/PATCH et DELETE. Cette association n'est pas une simple convention esthétique : elle aide le client à comprendre le comportement attendu.

Pour la ressource `services`, nous pouvons prévoir :

| Méthode | URI | Fonction |
|---|---|---|
| GET | /api/v1/services | Lister les prestations |
| GET | /api/v1/services/{service} | Consulter une prestation |
| POST | /api/v1/services | Créer une prestation |
| PUT/PATCH | /api/v1/services/{service} | Modifier une prestation |
| DELETE | /api/v1/services/{service} | Supprimer une prestation |

À ce stade, nous réalisons un CRUD pédagogique. Dans une application réelle, la création et la modification seront protégées par l'authentification, la validation et les autorisations. Ne publiez pas un CRUD ouvert en production.

## 8.2 Générer le modèle et le contrôleur

Si la migration du modèle existe déjà, vérifiez son contenu. Sinon :

```bash
php artisan make:model Service -m
php artisan make:controller Api/ServiceController --api
php artisan make:resource ServiceResource
```

Le modèle se trouve dans `app/Models/Service.php`, le contrôleur dans `app/Http/Controllers/Api` et la ressource dans `app/Http/Resources`.

Un contrôleur `--api` génère les méthodes habituelles d'une ressource sans la méthode de formulaire HTML `create` ni `edit`, qui ne sont pas nécessaires pour une API JSON classique.

## 8.3 Définir les attributs du modèle

Exemple simplifié :

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Service extends Model
{
    protected $fillable = [
        'professional_profile_id',
        'category_id',
        'title',
        'description',
        'price',
        'is_published',
    ];

    protected function casts(): array
    {
        return [
            'price' => 'decimal:2',
            'is_published' => 'boolean',
        ];
    }
}
```

Le tableau `fillable` indique les attributs autorisés pour l'affectation de masse. Il ne doit contenir que les champs que l'application accepte de remplir de cette manière.

Dans notre future API sécurisée, le client ne devra pas décider librement du `professional_profile_id` ni de l'état `is_published`. Ces valeurs seront déterminées par l'utilisateur authentifié et les règles métier côté serveur. Le code du CRUD de ce chapitre est donc une étape d'apprentissage ; nous corrigerons ce point avant le périmètre sécurisé.

## 8.4 Créer une API Resource

Dans `ServiceResource.php` :

```php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class ServiceResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'description' => $this->description,
            'price' => $this->price,
            'is_published' => $this->is_published,
            'created_at' => $this->created_at,
        ];
    }
}
```

La ressource définit les champs publics et leur structure. Elle rend les réponses plus stables : le client mobile dépend d'un contrat, pas de l'ensemble des colonnes internes de la table.

## 8.5 Construire le contrôleur

Voici une version pédagogique du contrôleur. Nous ajouterons ensuite des Form Requests, des policies, une pagination et des filtres plus avancés.

```php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Resources\ServiceResource;
use App\Models\Service;
use Illuminate\Http\Request;

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

    public function show(Service $service)
    {
        if (!$service->is_published) {
            abort(404);
        }

        return new ServiceResource($service);
    }

    public function store(Request $request)
    {
        $validated = $request->validate([
            'title' => ['required', 'string', 'min:5', 'max:150'],
            'description' => ['required', 'string', 'max:5000'],
            'category_id' => ['required', 'integer', 'exists:categories,id'],
            'price' => ['nullable', 'numeric', 'min:0'],
        ]);

        // Exemple temporaire de CRUD : le propriétaire sera
        // défini depuis l'utilisateur authentifié au chapitre sécurité.
        $service = Service::create([
            ...$validated,
            // Exemple volontairement provisoire : dans le projet sécurisé, ce propriétaire sera
            // déterminé côté serveur à partir du profil lié à l'utilisateur authentifié.
            'professional_profile_id' => $professionalProfile->id,
            'is_published' => false,
        ]);

        return (new ServiceResource($service))
            ->response()
            ->setStatusCode(201);
    }

    public function update(Request $request, Service $service)
    {
        $validated = $request->validate([
            'title' => ['sometimes', 'required', 'string', 'min:5', 'max:150'],
            'description' => ['sometimes', 'required', 'string', 'max:5000'],
            'category_id' => ['sometimes', 'required', 'integer', 'exists:categories,id'],
            'price' => ['sometimes', 'nullable', 'numeric', 'min:0'],
        ]);

        $service->update($validated);

        return new ServiceResource($service->refresh());
    }

    public function destroy(Service $service)
    {
        $service->delete();

        return response()->noContent();
    }
}
```

### Lecture professionnelle de cet exemple

Ce code démontre le fonctionnement des opérations, mais il n'est pas encore prêt pour la production.

- Le propriétaire fixé à l'identifiant `1` est une valeur provisoire de démonstration. Dans une API réelle, le propriétaire doit provenir du compte authentifié et être vérifié côté serveur.
- La méthode `update` doit vérifier que l'appelant possède le droit de modifier cette prestation.
- La suppression doit appliquer une politique de conservation et vérifier les dépendances métier.
- Les erreurs doivent être cohérentes et documentées.
- Les Form Requests permettront de séparer la validation du contrôleur.

Il est essentiel de savoir reconnaître un exemple pédagogique incomplet. Un code qui fonctionne dans Postman n'est pas automatiquement sécurisé.

## 8.6 Déclarer les routes

Dans `routes/api.php` :

```php
use App\Http\Controllers\Api\ServiceController;
use Illuminate\Support\Facades\Route;

Route::get('/services', [ServiceController::class, 'index']);
Route::get('/services/{service}', [ServiceController::class, 'show']);
Route::post('/services', [ServiceController::class, 'store']);
Route::patch('/services/{service}', [ServiceController::class, 'update']);
Route::delete('/services/{service}', [ServiceController::class, 'destroy']);
```

Pour vérifier que Laravel a enregistré les routes :

```bash
php artisan route:list --path=services
```

Le paramètre `{service}` est associé au modèle grâce au binding implicite lorsque le contrôleur reçoit un argument typé `Service $service`. Si l'identifiant n'existe pas, Laravel renvoie normalement une réponse 404.

## 8.7 Tester dans Postman

Configurez une variable d'environnement :

```
base_url = http://127.0.0.1:8000
```

Puis testez chaque requête.

### A. Lister

- Méthode : GET
- URL : `{{base_url}}/api/v1/services`
- Aucun corps requis.

Vous devez recevoir une réponse JSON avec une collection et, en cas de pagination, les informations de navigation prévues par Laravel.

### B. Consulter une prestation

- Méthode : GET
- URL : `{{base_url}}/api/v1/services/1`

Remplacez `1` par un identifiant réellement présent. Testez aussi un identifiant inexistant pour observer le 404.

### C. Créer

- Méthode : POST
- URL : `{{base_url}}/api/v1/services`
- En-tête : `Content-Type: application/json`
- Corps en mode raw JSON :

```json
{
  "title": "Installation d'un système électrique",
  "description": "Diagnostic et installation dans une habitation.",
  "category_id": 2,
  "price": 45.00
}
```

Le statut attendu pour une création réussie est généralement `201 Created`. Le serveur doit retourner la ressource créée, ou une représentation clairement documentée.

### D. Modifier

- Méthode : PATCH
- URL : `{{base_url}}/api/v1/services/1`
- Corps :

```json
{
  "title": "Installation électrique résidentielle"
}
```

PATCH permet une modification partielle. PUT est généralement utilisé pour remplacer la représentation complète, selon le contrat de l'API.

### E. Supprimer

- Méthode : DELETE
- URL : `{{base_url}}/api/v1/services/1`

Une suppression réussie peut renvoyer `204 No Content`, c'est-à-dire un succès sans corps de réponse.

## 8.8 Comprendre les réponses et les erreurs

Une API ne doit pas uniquement renvoyer des données quand tout va bien. Elle doit également indiquer clairement les échecs.

| Situation | Code généralement adapté |
|---|---:|
| Collection consultée | 200 |
| Ressource créée | 201 |
| Suppression sans contenu | 204 |
| Données invalides | 422 |
| Non authentifié | 401 |
| Action interdite | 403 |
| Ressource absente | 404 |
| Conflit métier | 409 |

Le code exact dépend du contrat et de la situation. Ne renvoyez pas systématiquement 200 pour une erreur ; le client doit pouvoir distinguer une réussite d'un échec sans analyser un texte ambigu.

## 8.9 Atelier de validation

Pour chaque route, consignez :

- la méthode HTTP ;
- l'URL ;
- les paramètres ;
- les en-têtes ;
- le corps envoyé ;
- le statut reçu ;
- la structure JSON ;
- le résultat attendu en cas d'erreur.

Réalisez au minimum les tests suivants : champ obligatoire absent, catégorie inexistante, identifiant de prestation invalide, prix négatif et tentative de modifier une prestation qui ne vous appartient pas. Ce dernier test deviendra pleinement applicable lorsque l'authentification et les autorisations seront mises en place.

## Exercices et corrigés

**Exercice 1.** Pourquoi une création renvoie-t-elle souvent 201 plutôt que 200 ?

**Corrigé :** 201 indique qu'une nouvelle ressource a été créée. Le contrat peut aussi fournir sa représentation ou son emplacement.

**Exercice 2.** Quelle est la différence pratique entre PATCH et PUT ?

**Corrigé :** PATCH décrit une modification partielle ; PUT est généralement associé au remplacement de la représentation selon les règles du contrat.

**Exercice 3.** Pourquoi ne faut-il pas accepter `professional_profile_id` du client sans contrôle ?

**Corrigé :** un utilisateur pourrait tenter de créer une prestation au nom d'un autre professionnel. Le propriétaire doit être dérivé de l'identité authentifiée et autorisé par le serveur.

**Exercice 4.** Pourquoi retourner une API Resource ?

**Corrigé :** elle permet de contrôler les champs et de stabiliser la forme de la réponse, indépendamment des détails internes du modèle.

## À retenir

Un CRUD est la première démonstration concrète d'une API, mais la qualité professionnelle se mesure aussi à la validation, aux statuts HTTP, à la cohérence du JSON, à la sécurité et aux tests.

**Transition vers la partie suivante :** nous allons maintenant enrichir cette base avec une authentification adaptée aux clients API, puis construire les protections qui empêchent un utilisateur de consulter ou modifier les données d'un autre.

## Références

- Laravel, Controllers : https://laravel.com/docs/controllers
- Laravel, Routing : https://laravel.com/docs/routing
- Laravel, Eloquent : https://laravel.com/docs/eloquent
- Laravel, Eloquent Resources : https://laravel.com/docs/eloquent-resources
- Postman, documentation : https://learning.postman.com/docs/
