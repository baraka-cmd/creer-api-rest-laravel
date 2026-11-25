# Chapitre 13 — Recherche, filtres et pagination

## Objectifs

Créer une recherche utile aux clients, combiner des filtres, contrôler le tri, paginer les résultats et limiter les requêtes coûteuses.

## 13.1 Concevoir une recherche utile

Un client peut chercher une prestation par mot-clé, catégorie ou fourchette de prix. La recherche doit rester prévisible : seuls les paramètres documentés sont acceptés et le serveur impose des limites.

Exemple d'URL :

```
GET /api/services?q=plomberie&category_id=3&min_price=10&max_price=100&sort=price
```

Chaque paramètre doit avoir une définition : format, comportement, valeur par défaut et effet sur les résultats.

## 13.2 Construire la requête avec Eloquent

```php
$query = Service::query()
    ->where('is_published', true);

if ($request->filled('q')) {
    $keyword = $request->string('q')->toString();

    $query->where(function ($sub) use ($keyword) {
        $sub->where('title', 'like', "%{$keyword}%")
            ->orWhere('description', 'like', "%{$keyword}%");
    });
}

if ($request->filled('category_id')) {
    $query->where('category_id', $request->integer('category_id'));
}

if ($request->filled('min_price')) {
    $query->where('price', '>=', $request->input('min_price'));
}

if ($request->filled('max_price')) {
    $query->where('price', '<=', $request->input('max_price'));
}
```

Les paramètres doivent être validés avant de construire la requête. N'insérez jamais directement une valeur utilisateur dans une chaîne SQL brute. Eloquent et le Query Builder utilisent des paramètres liés pour les valeurs, mais les noms de colonnes de tri doivent être choisis dans une liste autorisée.

## 13.3 Contrôler le tri

```php
$sortOptions = [
    'newest' => ['created_at', 'desc'],
    'price' => ['price', 'asc'],
    'title' => ['title', 'asc'],
];

$sort = $request->input('sort', 'newest');
[$column, $direction] = $sortOptions[$sort] ?? $sortOptions['newest'];

$query->orderBy($column, $direction);
```

Cette liste blanche empêche le client de choisir arbitrairement une colonne ou une expression SQL. Le tri doit être stable : pour deux valeurs identiques, ajoutez si nécessaire un second ordre, par exemple l'identifiant.

## 13.4 Paginer les résultats

```php
$perPage = min(max($request->integer('per_page', 15), 1), 50);

$services = $query
    ->with(['category', 'professionalProfile'])
    ->paginate($perPage)
    ->withQueryString();

return ServiceResource::collection($services);
```

La limite maximale évite qu'un client demande plusieurs milliers de lignes en une seule réponse. La pagination conserve les paramètres de recherche dans les liens.

Pour de très grandes collections ou des données qui changent souvent, la pagination par curseur peut être plus adaptée. Le choix dépend du volume, du tri et du comportement attendu par le client.

## 13.5 Performance et index

Les filtres fréquents peuvent nécessiter des index. Par exemple, un index combiné sur l'état de publication et la catégorie peut aider certaines requêtes.

Cependant, l'indexation doit être guidée par les requêtes réellement exécutées et le plan d'exécution. Un index mal choisi augmente le coût des écritures sans améliorer la recherche.

La recherche `LIKE '%mot%'` peut être coûteuse sur une grande table. Pour un volume important, étudiez les index full-text ou un moteur de recherche dédié, en fonction de la langue, de la pertinence et des besoins.

## Atelier Postman

Testez les combinaisons :

- mot-clé seul ;
- catégorie seule ;
- prix minimum et maximum ;
- tri par prix ;
- pagination à 5 résultats ;
- paramètre de tri inconnu ;
- page vide ou valeur négative ;
- recherche sans résultat.

Vérifiez que la réponse respecte la limite maximale et que les métadonnées de pagination sont cohérentes.

## Exercices corrigés

**Pourquoi une liste blanche pour le tri ?** Parce que les noms de colonnes ne doivent pas être contrôlés librement par le client.

**Pourquoi limiter `per_page` ?** Pour éviter une réponse massive qui consomme trop de mémoire, de temps et de bande passante.

**Quand envisager une recherche spécialisée ?** Lorsque la recherche textuelle, le volume ou la pertinence dépassent ce qu'une requête SQL simple peut gérer efficacement.

## À retenir

Une recherche professionnelle combine paramètres validés, filtres composables, tri autorisé, pagination et optimisation mesurée. L'API doit rester utile tout en protégeant les ressources du serveur.

## Références

- Laravel Query Builder : https://laravel.com/docs/queries
- Laravel Pagination : https://laravel.com/docs/pagination
- OWASP API Security : https://owasp.org/www-project-api-security/
