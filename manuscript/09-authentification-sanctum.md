# Chapitre 9 — Inscription, connexion et authentification

## Objectifs du chapitre

À la fin de ce chapitre, vous saurez construire le parcours d'inscription et de connexion d'une API Laravel, protéger les mots de passe, utiliser Laravel Sanctum pour délivrer des tokens personnels, comprendre leur durée de vie et révoquer les accès lors de la déconnexion.

Nous travaillons toujours sur **ProxiServices**, une API qui permet aux clients de rechercher des professionnels. Les routes privées seront réservées aux utilisateurs authentifiés.

## 9.1 Identifier un utilisateur n'est pas encore l'autoriser

L'authentification répond à la question : « Qui effectue cette requête ? » Elle ne répond pas automatiquement à : « Peut-il modifier cette prestation ? »

Un token valide peut identifier un utilisateur sans lui donner le droit de modifier la ressource d'un autre compte. Les autorisations par rôle et par ressource seront étudiées au chapitre 10.

## 9.2 Le parcours d'inscription

Une inscription doit recueillir les données strictement nécessaires, vérifier leur format, empêcher les doublons et enregistrer un mot de passe sous forme de hash.

Exemple de corps JSON :

```json
{
  "name": "Aline M.",
  "email": "aline@example.test",
  "password": "UnMotDePasseLong!2026",
  "password_confirmation": "UnMotDePasseLong!2026"
}
```

Cet email est fictif. Ne mettez jamais de véritables mots de passe dans le manuscrit, les captures ou les dépôts.

Créez le contrôleur :

```bash
php artisan make:controller Api/AuthController
php artisan make:request RegisterRequest
```

Dans le Form Request, définissez les règles :

```php
public function rules(): array
{
    return [
        'name' => ['required', 'string', 'max:120'],
        'email' => ['required', 'email', 'max:255', 'unique:users,email'],
        'password' => ['required', 'string', 'min:12', 'confirmed'],
    ];
}
```

La règle `confirmed` compare le champ `password` au champ `password_confirmation`. La validation `unique` améliore l'expérience, mais une contrainte unique en base reste nécessaire pour prévenir les doublons en cas de requêtes simultanées.

## 9.3 Hashage du mot de passe

Un mot de passe ne doit jamais être stocké en clair, ni simplement chiffré de manière réversible. Laravel propose une façade de hashage qui utilise un algorithme adapté.

```php
use Illuminate\Support\Facades\Hash;

$user = User::create([
    'name' => $validated['name'],
    'email' => $validated['email'],
    'password' => Hash::make($validated['password']),
]);
```

Le hash n'est pas le mot de passe original. Lors de la connexion, Laravel vérifie le mot de passe fourni contre le hash stocké. Ne créez pas votre propre algorithme de hashage et ne journalisez jamais le mot de passe reçu.

## 9.4 Installer et comprendre Sanctum

Sanctum est une solution Laravel pour l'authentification des applications SPA et l'utilisation de tokens personnels. Pour une application mobile ou un client externe utilisant des tokens, l'utilisateur peut obtenir un token après une connexion réussie.

Vérifiez que Sanctum est installé et que le modèle `User` utilise le trait requis :

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;
}
```

Le mécanisme exact de configuration dépend du squelette et de la version Laravel. Consultez la documentation de votre version et vérifiez la migration de la table des tokens avant d'exécuter les migrations.

## 9.5 Connexion et délivrance d'un token

La connexion doit rechercher l'utilisateur, vérifier le mot de passe et ne délivrer un token qu'en cas de succès.

```php
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

public function login(Request $request)
{
    $credentials = $request->validate([
        'email' => ['required', 'email'],
        'password' => ['required', 'string'],
        'device_name' => ['required', 'string', 'max:100'],
    ]);

    $user = User::where('email', $credentials['email'])->first();

    if (!$user || !Hash::check($credentials['password'], $user->password)) {
        throw ValidationException::withMessages([
            'email' => ['Les identifiants fournis sont incorrects.'],
        ]);
    }

    $token = $user->createToken($credentials['device_name']);

    return response()->json([
        'user' => [
            'id' => $user->id,
            'name' => $user->name,
            'email' => $user->email,
        ],
        'token' => $token->plainTextToken,
        'token_type' => 'Bearer',
    ]);
}
```

Le token en clair est fourni au client au moment de sa création. Sanctum conserve une représentation hashée dans la base. Le client doit le protéger et le transmettre dans l'en-tête :

```
Authorization: Bearer VOTRE_TOKEN
```

Ne renvoyez pas le hash du mot de passe, et n'exposez pas les colonnes internes du modèle User.

## 9.6 Protéger les routes

```php
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/me', [AuthController::class, 'me']);
    Route::post('/logout', [AuthController::class, 'logout']);
});
```

Une route privée appelée sans token valide doit normalement produire une réponse 401. La route `/me` peut retourner une ressource utilisateur contrôlée plutôt que le modèle complet.

## 9.7 Déconnexion et révocation

Pour un token personnel, la déconnexion révoque le token utilisé :

```php
public function logout(Request $request)
{
    $request->user()->currentAccessToken()->delete();

    return response()->json([
        'message' => 'Déconnexion effectuée.',
    ]);
}
```

La révocation rend le token supprimé inutilisable. Si votre politique prévoit une déconnexion de tous les appareils, vous pouvez supprimer tous les tokens de l'utilisateur, mais cette action doit être explicitement définie et présentée au client.

Une application doit aussi prévoir le cas où le token est perdu, volé ou expiré. La durée de vie et les capacités des tokens doivent être configurées selon les exigences de sécurité, pas laissées au hasard.

## 9.8 Sécurité du parcours

- Utilisez HTTPS en production.
- Ne stockez pas le token dans un emplacement accessible à des scripts non fiables.
- Ne l'insérez pas dans l'URL.
- Limitez les tentatives de connexion avec un mécanisme de rate limiting.
- Répondez avec un message d'échec qui ne révèle pas si l'email existe.
- Ne faites pas confiance au rôle ou à l'identifiant de propriétaire envoyé par le client.
- Révoquez les tokens lorsque l'utilisateur se déconnecte ou qu'un incident le nécessite.

## Atelier Postman

1. Envoyez une requête POST d'inscription.
2. Vérifiez le statut de création et la réponse.
3. Connectez-vous avec les bons identifiants.
4. Copiez le token dans une variable Postman sécurisée.
5. Appelez `GET /api/me` avec l'en-tête Bearer.
6. Appelez la route privée sans token et observez le 401.
7. Déconnectez-vous, puis réessayez avec le token révoqué.

## Exercices corrigés

**Pourquoi hasher le mot de passe ?** Pour éviter qu'une fuite de base de données révèle directement les mots de passe des utilisateurs.

**Pourquoi un token ne prouve-t-il pas la propriété d'une prestation ?** Il identifie l'utilisateur ; la policy doit vérifier son droit sur la ressource demandée.

**Pourquoi ne pas afficher le token à chaque requête ?** Le token est un secret d'accès. Il doit être délivré de manière contrôlée et protégé côté client.

## À retenir

L'authentification est une chaîne complète : validation, hashage, vérification des identifiants, émission sécurisée du token, protection des routes et révocation. Une simple route de connexion ne suffit pas à sécuriser une API.

## Références

- Laravel Sanctum : https://laravel.com/docs/sanctum
- Laravel Hashing : https://laravel.com/docs/hashing
- Laravel Validation : https://laravel.com/docs/validation
