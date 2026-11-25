# Chapitre 12 — Gérer les relations entre les ressources

## Objectifs

Comprendre les relations Eloquent, charger les données associées efficacement, éviter les requêtes N+1, maîtriser la sérialisation et protéger les informations liées.

## 12.1 Pourquoi les relations sont importantes

Une prestation possède un professionnel et une catégorie. Une réponse API peut afficher le titre, le nom public du professionnel et le libellé de la catégorie.

Il serait possible de charger chaque information séparément, mais cette approche devient fragile et coûteuse. Eloquent permet de déclarer les relations afin de rendre le code plus expressif.

## 12.2 Déclarer les relations

Dans le modèle Service :

```php
public function professionalProfile()
{
    return $this->belongsTo(ProfessionalProfile::class);
}

public function category()
{
    return $this->belongsTo(Category::class);
}
```

Dans ProfessionalProfile :

```php
public function services()
{
    return $this->hasMany(Service::class);
}
```

Dans Category :

```php
public function services()
{
    return $this->hasMany(Service::class);
}
```

Les noms de méthodes doivent être clairs et suivre une convention cohérente. Ils représentent des relations que vous pourrez utiliser dans les requêtes et les tests.

## 12.3 Charger une relation

```php
$service = Service::with(['professionalProfile', 'category'])
    ->findOrFail($id);
```

Le chargement anticipé, appelé eager loading, permet de récupérer les relations en amont plutôt que de déclencher une requête à chaque accès.

## 12.4 Comprendre le problème N+1

Supposons que l'API affiche 20 prestations et, pour chacune, le nom du professionnel.

Sans chargement anticipé, une requête peut récupérer les 20 prestations, puis déclencher une requête supplémentaire pour chaque professionnel. Le nombre total peut atteindre 21 requêtes : une principale et vingt secondaires.

Avec `with('professionalProfile')`, Eloquent peut récupérer les prestations et leurs professionnels en un nombre limité de requêtes. Le nombre exact dépend des relations et du SQL généré, mais le principe est de ne pas exécuter une requête répétée par ligne.

Le problème N+1 n'est pas seulement théorique. Il peut rendre une API lente lorsque le volume augmente.

## 12.5 Sérialiser sans exposer les modèles entiers

Une relation chargée n'a pas besoin d'être renvoyée intégralement. Dans une ressource :

```php
return [
    'id' => $this->id,
    'title' => $this->title,
    'category' => [
        'id' => $this->category->id,
        'name' => $this->category->name,
    ],
    'professional' => [
        'id' => $this->professionalProfile->id,
        'display_name' => $this->professionalProfile->display_name,
    ],
];
```

Cette structure expose uniquement les informations nécessaires. Ne renvoyez pas automatiquement l'utilisateur complet, ses coordonnées privées, ses documents de vérification ou ses attributs internes.

Les données affichées doivent aussi respecter les règles de visibilité. Un profil désactivé ou privé ne doit pas être exposé simplement parce qu'il est lié à une prestation.

## 12.6 Éviter les relations non chargées

Dans une ressource, l'accès à une relation non chargée peut provoquer des requêtes supplémentaires. Vous pouvez vérifier le chargement ou définir une stratégie cohérente pour les ressources.

Pendant le développement, activez les mécanismes de détection de chargement paresseux disponibles dans votre version Laravel, si cela convient à votre application. Les tests de performance et l'inspection SQL restent utiles.

## 12.7 Relations et autorisations

La relation Eloquent décrit le lien entre les données ; elle ne garantit pas que l'utilisateur a le droit de consulter toutes les données liées.

Exemple : le fait qu'une prestation appartienne à un profil professionnel ne signifie pas que l'adresse privée du professionnel doit être incluse dans la réponse publique.

Utilisez les policies, les ressources et les requêtes filtrées pour contrôler à la fois l'accès et la représentation.

## Atelier pratique

1. Créez plusieurs prestations et catégories.
2. Associez les prestations à leurs profils.
3. Listez les prestations avec `with()`.
4. Comparez le comportement sans et avec eager loading.
5. Construisez une ressource qui expose uniquement le nom public du professionnel et la catégorie.

## Exercices corrigés

**Qu'est-ce que N+1 ?** Une requête initiale suivie d'une requête répétée pour chaque élément de la collection.

**Eager loading signifie-t-il toujours moins de requêtes ?** Il évite souvent les requêtes répétées, mais le choix doit correspondre aux relations réellement utilisées et être mesuré.

**Une relation autorise-t-elle l'accès ?** Non. Elle décrit le lien de données, pas les droits de l'appelant.

## À retenir

Les relations Eloquent rendent le modèle métier lisible, mais elles doivent être chargées de manière réfléchie et sérialisées avec prudence. La performance et la confidentialité font partie du contrat API.

## Références

- Laravel Eloquent Relationships : https://laravel.com/docs/eloquent-relationships
- Laravel Eloquent Resources : https://laravel.com/docs/eloquent-resources
- Laravel Database Queries : https://laravel.com/docs/queries
