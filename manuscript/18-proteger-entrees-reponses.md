# Chapitre 18 — Protéger les entrées et les réponses

## 18.1. Les données reçues sont toujours non fiables

Une API reçoit des données d’applications que vous ne contrôlez pas entièrement : navigateur, application Flutter, script automatisé ou intégration tierce. Même si votre interface envoie normalement les bons champs, un client peut modifier la requête.

La validation répond à la question : « Cette donnée respecte-t-elle le format et les contraintes attendus ? » L’autorisation répond : « Cet utilisateur a-t-il le droit de fournir ou de modifier cette donnée ? » Les deux contrôles sont nécessaires.

## 18.2. Mass assignment : comprendre le danger

Le mass assignment consiste à transmettre un tableau de données à un modèle pour modifier plusieurs attributs. Cette technique est pratique, mais dangereuse lorsque le tableau contient des champs contrôlés par le serveur.

Exemple risqué :

```php
$user->update($request->all());
```

Un client peut tenter :

```json
{
  "name": "Jean",
  "role": "admin",
  "is_verified": true,
  "wallet_balance": 5000
}
```

Si ces attributs sont modifiables par cette opération, l’utilisateur peut augmenter ses privilèges ou falsifier des données financières.

Laravel fournit la protection contre l’affectation de masse avec `$fillable` et `$guarded`. Mais ces mécanismes ne remplacent pas la conception de la requête : un champ peut être autorisé dans un autre contexte et ne pas être modifiable par cette route.

Préférez une liste explicite :

```php
$data = $request->validated();

$user->update([
    'name' => $data['name'],
    'phone' => $data['phone'],
]);
```

Cette méthode rend la frontière de confiance visible dans le code.

## 18.3. Séparer les données de commande des données calculées

Dans une plateforme de services, le client peut demander un service et indiquer des informations utiles, mais il ne doit pas décider lui-même du prix final, de la commission, du statut de paiement ou du propriétaire de la demande.

Exemple de requête acceptable :

```json
{
  "service_id": 12,
  "description": "Installer deux prises",
  "requested_date": "2026-12-03"
}
```

Le serveur récupère le service, vérifie qu’il est disponible, calcule ou obtient le tarif selon les règles applicables, puis crée la demande avec l’utilisateur authentifié comme client.

Ne faites pas confiance à une valeur `total` fournie par l’application mobile. Si le client transmet un montant pour afficher une estimation, le serveur doit recalculer et vérifier ce montant avant toute opération financière.

## 18.4. Validation stricte avec Form Requests

Les Form Requests permettent de réunir les règles de validation et, si nécessaire, une autorisation préalable.

```php
public function rules(): array
{
    return [
        'title' => ['required', 'string', 'min:5', 'max:120'],
        'description' => ['required', 'string', 'max:3000'],
        'category_id' => ['required', 'integer', 'exists:categories,id'],
    ];
}
```

La règle `exists` vérifie l’existence d’une catégorie, mais pas nécessairement qu’elle est active ou accessible. Cette contrainte métier doit être contrôlée séparément, par exemple dans une règle dédiée ou un service métier.

Validez aussi les paramètres de requête : tri, filtres, limites, identifiants et dates. Ne construisez pas directement une clause SQL avec un nom de colonne fourni par le client. Les valeurs peuvent être paramétrées ; les noms de colonnes et directions de tri doivent être choisis dans une liste autorisée.

## 18.5. Validation et autorisation ne sont pas interchangeables

Un identifiant peut être valide numériquement et correspondre à une ligne existante, mais l’utilisateur peut ne pas avoir le droit de la modifier.

Une règle `exists:users,id` ne garantit pas que l’utilisateur connecté peut attribuer cette ressource. Pour les champs sensibles, déterminez la valeur à partir de l’identité authentifiée ou contrôlez explicitement la relation.

Exemple : le propriétaire d’un service ne devrait pas être choisi librement par `professional_id` dans le formulaire. Le serveur peut utiliser :

```php
$service->professional_id = $request->user()->id;
```

Le champ n’a alors pas besoin d’être accepté depuis le client.

## 18.6. Éviter l’exposition excessive des données

Retourner directement un modèle Eloquent peut exposer des attributs ou relations inattendus, notamment après une évolution du schéma.

À risque :

```php
return User::findOrFail($id);
```

Une API Resource définit le contrat public :

```php
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'profile_photo_url' => $this->profile_photo_url,
    ];
}
```

N’ajoutez pas automatiquement tous les attributs du modèle. Les champs tels que `password`, `remember_token`, données de vérification, notes internes, clés de paiement ou informations privées doivent rester exclus.

Pensez également aux relations : un Resource peut révéler les coordonnées d’un professionnel ou les informations d’un client si elles sont chargées et sérialisées sans distinction.

## 18.7. DTO : clarifier les données qui entrent dans le domaine

Un DTO (*Data Transfer Object*) est un objet qui représente explicitement les données transférées entre couches. Il peut aider à séparer les données validées de la requête HTTP et les données utilisées par la logique métier.

Pour un petit endpoint, un Form Request et une liste explicite de champs peuvent suffire. Pour une opération complexe, un DTO ou une commande métier peut clarifier :
- les données réellement acceptées ;
- les valeurs calculées par le serveur ;
- les informations provenant de l’utilisateur authentifié ;
- les objets déjà vérifiés ;
- les paramètres nécessaires au service métier.

Ne créez pas un DTO pour chaque ligne par principe. Utilisez-le lorsque cette séparation améliore la lisibilité, les tests ou la stabilité de l’architecture.

## 18.8. Protection des champs sensibles par contexte

Un même champ peut être visible dans un contexte et privé dans un autre. Le téléphone peut être visible après acceptation d’une demande, mais masqué dans le catalogue public. Une adresse complète peut être nécessaire pour une commande, mais ne doit pas apparaître dans une recherche générale.

Définissez des Resources ou transformations distinctes lorsque les contrats sont différents : profil public, profil propriétaire, profil administratif. Évitez de créer un unique Resource géant avec de nombreuses conditions difficiles à vérifier.

## 18.9. Atelier de test : tenter de falsifier les données

Pour chaque endpoint de création ou modification, envoyez des champs supplémentaires :
- `role` ;
- `owner_id` ;
- `is_verified` ;
- `status` ;
- `commission` ;
- `total_amount` ;
- `created_at`.

Vérifiez que ces champs sont rejetés ou ignorés conformément au contrat et qu’ils ne modifient jamais l’état privilégié. Le comportement doit être documenté et testé ; ne laissez pas une ambiguïté entre le client mobile et le serveur.

## À retenir

Validez les entrées, autorisez les actions, choisissez explicitement les champs modifiables et contrôlez chaque réponse. Le serveur doit rester la source de vérité pour les identités, permissions, statuts et montants.