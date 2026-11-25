# Chapitre 11 — Validation professionnelle des données

## Objectifs

Savoir valider les entrées de manière cohérente, utiliser les Form Requests, produire des erreurs compréhensibles, contrôler les fichiers et appliquer des règles métier sans faire confiance aux données du client.

## 11.1 La validation protège le contrat

Une API reçoit des données provenant d'un environnement que le serveur ne maîtrise pas. Le client peut être une application Flutter, un site web, un script ou une requête fabriquée manuellement.

La validation vérifie la forme et certaines conditions des données avant le traitement. Elle ne remplace ni les autorisations ni les contraintes de la base.

## 11.2 Utiliser un Form Request

```bash
php artisan make:request StoreServiceRequest
```

Exemple de règles :

```php
public function rules(): array
{
    return [
        'title' => ['required', 'string', 'min:5', 'max:150'],
        'description' => ['required', 'string', 'max:5000'],
        'category_id' => ['required', 'integer', 'exists:categories,id'],
        'price' => ['nullable', 'numeric', 'min:0'],
    ];
}
```

Le contrôleur reçoit les données validées :

```php
public function store(StoreServiceRequest $request)
{
    $validated = $request->validated();

    // Le propriétaire et les champs sensibles sont définis
    // côté serveur, pas par le client.
}
```

L'utilisation de `validated()` réduit le risque d'utiliser accidentellement des champs non prévus.

## 11.3 Messages d'erreur utiles

Laravel renvoie habituellement un statut 422 pour les erreurs de validation d'une requête API, avec une structure JSON qui contient un message et les erreurs par champ.

Exemple :

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "title": [
      "Le titre est obligatoire."
    ]
  }
}
```

Vous pouvez personnaliser les messages et les noms d'attributs pour rendre l'API plus compréhensible. Le client doit pouvoir associer une erreur au champ concerné sans interpréter une phrase arbitraire.

Ne renvoyez jamais les valeurs sensibles reçues dans les messages d'erreur.

## 11.4 Validation et règles métier

La validation syntaxique peut vérifier qu'un prix est numérique et positif. La règle métier peut vérifier que le professionnel est actif, que la catégorie accepte ce type de prestation ou que la publication est autorisée.

Ces contrôles ne sont pas interchangeables. Une donnée peut être correctement formée mais interdite par le métier.

Pour une opération complexe, placez les règles métier dans un service ou une classe de domaine, et faites vérifier l'autorisation par une policy.

## 11.5 Protéger contre les champs falsifiés

Supposons qu'un client envoie :

```json
{
  "title": "Réparation",
  "professional_profile_id": 99,
  "is_published": true,
  "role": "admin"
}
```

Le serveur ne doit pas utiliser ces champs sensibles sans contrôle. Le propriétaire doit être dérivé de l'utilisateur connecté, la publication doit suivre un processus autorisé et le rôle ne doit pas être modifié par une inscription ordinaire.

La stratégie recommandée est une liste explicite de champs acceptés, puis l'ajout côté serveur des valeurs calculées ou protégées.

## 11.6 Validation des fichiers

Pour une image de profil, vous pouvez contrôler le type reconnu, la taille et les dimensions selon le besoin :

```php
'photo' => [
    'required',
    'image',
    'mimes:jpeg,png,webp',
    'max:2048',
]
```

Dans Laravel, la taille du fichier est généralement exprimée en kilo-octets dans la règle `max`. Ici, 2048 correspond approximativement à 2 Mo selon la convention de validation.

Ne vous contentez pas de l'extension du nom de fichier. Contrôlez le contenu avec les règles adaptées, utilisez des noms générés par le serveur et stockez les fichiers dans un emplacement maîtrisé.

## 11.7 Tests négatifs indispensables

Testez systématiquement :

- champ obligatoire absent ;
- chaîne trop longue ;
- identifiant inexistant ;
- prix négatif ;
- format de fichier refusé ;
- fichier trop volumineux ;
- champ protégé injecté ;
- utilisateur non autorisé.

Une API bien testée ne démontre pas seulement que le scénario valide fonctionne ; elle démontre également que les scénarios invalides sont refusés.

## Atelier et corrigés

**Exercice :** un client envoie un identifiant de propriétaire différent du sien. Quelle stratégie appliquer ?

**Corrigé :** ne pas utiliser cet identifiant comme source de vérité. Dériver le propriétaire du compte authentifié, puis vérifier la policy sur toute modification.

**Exercice :** pourquoi garder une contrainte unique en base si la validation utilise déjà `unique` ?

**Corrigé :** deux requêtes peuvent passer la validation presque simultanément. La base garantit l'intégrité au moment de l'écriture.

## À retenir

La validation professionnelle combine règles de format, règles métier, autorisations et contraintes de base. Elle doit être explicite, testée et ne jamais faire confiance aux champs sensibles du client.

## Références

- Laravel Validation : https://laravel.com/docs/validation
- Laravel Form Requests : https://laravel.com/docs/validation#form-request-validation
- Laravel File Storage : https://laravel.com/docs/filesystem
