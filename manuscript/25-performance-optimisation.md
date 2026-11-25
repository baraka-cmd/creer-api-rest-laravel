# Chapitre 25 — Optimiser les performances : mesurer, comprendre et améliorer

## 25.1. La performance n'est pas une impression

Une API peut sembler rapide pendant les premiers tests, puis devenir lente lorsque la base contient des milliers de services, que plusieurs utilisateurs consultent les mêmes listes ou que les réponses incluent des relations volumineuses.

Le réflexe professionnel n'est pas de deviner. Il consiste à mesurer le temps de réponse, le nombre de requêtes, leur coût, la taille du JSON et les ressources consommées. Une optimisation doit répondre à un problème observé et être vérifiée après modification.

Les principaux facteurs à examiner sont :
- le nombre et la durée des requêtes SQL ;
- les relations Eloquent chargées ;
- les index disponibles ;
- le volume de données renvoyé ;
- la stratégie de pagination ;
- les calculs répétés ;
- le cache et sa cohérence ;
- le temps passé dans des services externes.

## 25.2. Comprendre le problème N+1

Supposons que l'API affiche vingt services et le nom de leur catégorie.

~~~php
$services = Service::query()->latest()->limit(20)->get();

foreach ($services as $service) {
    echo $service->category->name;
}
~~~

La requête initiale récupère les services. Puis, si la relation n'est pas chargée, Eloquent peut exécuter une requête supplémentaire pour chaque catégorie. On obtient une requête pour la liste, puis potentiellement vingt requêtes pour les relations : c'est le problème N+1.

La correction habituelle est l'eager loading :

~~~php
$services = Service::query()
    ->with('category')
    ->latest()
    ->limit(20)
    ->get();
~~~

Laravel récupère les services et charge les catégories associées de manière groupée. Le nombre de requêtes ne dépend plus de la même façon du nombre de lignes affichées.

Pour plusieurs relations :

~~~php
$services = Service::query()
    ->with(['category', 'professional.user'])
    ->latest()
    ->paginate(20);
~~~

Ne chargez toutefois pas toutes les relations par habitude. Une relation qui n'est pas utilisée augmente le travail SQL, la mémoire et parfois le JSON. Chargez ce dont la représentation demandée a réellement besoin.

## 25.3. Préparer les requêtes pour le contrat de réponse

Une bonne requête commence par comprendre le JSON attendu. Si la liste publique n'affiche que l'identifiant, le titre, le prix et la catégorie, il n'est pas nécessaire de récupérer toutes les colonnes du modèle.

~~~php
$services = Service::query()
    ->select([
        'id',
        'category_id',
        'professional_id',
        'title',
        'price',
        'created_at',
    ])
    ->with('category:id,name')
    ->where('status', 'published')
    ->latest()
    ->paginate(15);
~~~

Lorsque vous sélectionnez manuellement les colonnes, gardez les clés nécessaires aux relations. Ici, category_id est indispensable pour rattacher la catégorie. Une sélection trop agressive peut casser l'eager loading ou une Resource.

La Resource doit elle aussi éviter d'exposer des champs inutiles. Réduire la réponse JSON améliore la transmission réseau, la consommation mobile et la lisibilité du contrat.

## 25.4. Les index SQL : accélérer les recherches fréquentes

Un index aide le moteur de base de données à retrouver certaines lignes sans parcourir systématiquement toute la table. Il peut être utile sur les colonnes souvent utilisées dans les filtres, les jointures et les tris.

Exemple de migration :

~~~php
Schema::table('services', function (Blueprint $table) {
    $table->index(['status', 'category_id']);
    $table->index(['professional_id', 'status']);
});
~~~

Un index composé doit être choisi en fonction des requêtes réelles. L'ordre des colonnes compte : un index sur status puis category_id ne répond pas exactement aux mêmes besoins qu'un index category_id puis status.

Les index ont aussi un coût :
- ils occupent de l'espace ;
- ils ralentissent certaines écritures ;
- ils nécessitent une maintenance ;
- un index inutile complique le schéma sans bénéfice.

Ne créez pas un index sur chaque colonne. Analysez les requêtes fréquentes et utilisez les outils du moteur, comme EXPLAIN, pour comprendre le plan d'exécution. Le résultat dépend de la base, de la distribution des données et de la version.

## 25.5. Recherche, tri et pagination

Une API de catalogue doit limiter le volume de résultats. Renvoyer plusieurs milliers de lignes à une application mobile crée une réponse lourde, consomme de la mémoire et rend l'interface moins réactive.

Pagination classique :

~~~php
$services = Service::query()
    ->where('status', 'published')
    ->orderByDesc('created_at')
    ->orderByDesc('id')
    ->paginate(20);
~~~

Le second tri, sur un identifiant unique, stabilise l'ordre lorsque plusieurs éléments partagent la même date.

Pour de grandes listes parcourues séquentiellement, la pagination par curseur peut être plus efficace que l'offset :

~~~php
$services = Service::query()
    ->where('status', 'published')
    ->orderBy('id')
    ->cursorPaginate(30);
~~~

La pagination par curseur impose un ordre déterministe et des colonnes compatibles avec la stratégie de curseur. Elle n'est pas nécessaire pour chaque écran. Une pagination classique convient souvent aux petites listes où l'utilisateur doit accéder directement à une page précise.

Les paramètres de pagination doivent être bornés. Ne laissez pas un client demander une taille arbitraire de 100 000 éléments.

~~~php
$perPage = min(
    max((int) request('per_page', 15), 1),
    50
);
~~~

En production, placez cette logique dans une validation ou une couche de requête cohérente plutôt que de la répéter partout.

## 25.6. Cache : accélérer sans servir des données incorrectes

Le cache conserve temporairement un résultat pour éviter de refaire un travail coûteux. Il est utile pour des données relativement stables : catégories publiques, paramètres de configuration non sensibles ou statistiques calculées.

~~~php
$categories = Cache::remember(
    'public-categories:v1',
    now()->addMinutes(10),
    fn () => Category::query()
        ->where('is_active', true)
        ->orderBy('name')
        ->get()
);
~~~

La durée de dix minutes est un exemple, pas une règle universelle. Le choix dépend de la fréquence de modification et du niveau de fraîcheur attendu.

La difficulté du cache est l'invalidation. Si une catégorie est désactivée, l'API ne doit pas continuer à la présenter comme active pendant une période inacceptable.

Stratégies possibles :
- durée de vie courte pour une donnée peu critique ;
- suppression explicite de la clé lors d'une modification ;
- clés versionnées ;
- cache par utilisateur uniquement lorsque les droits et les données sont correctement isolés.

Ne mettez jamais en cache une réponse privée sous une clé publique partagée. Le cache doit respecter l'identité, les permissions et le contexte de la requête.

## 25.7. Mesurer au lieu d'optimiser au hasard

Avant une optimisation, établissez une mesure de référence :
- durée moyenne et, si possible, percentiles de latence ;
- nombre de requêtes SQL ;
- requêtes les plus lentes ;
- taille de la réponse ;
- comportement avec un volume réaliste.

En environnement de développement, Laravel permet d'observer les requêtes avec des outils adaptés. Les journaux SQL doivent rester contrôlés : enregistrer toutes les requêtes et leurs paramètres en production peut exposer des données sensibles et produire un volume excessif.

Pour une mesure réaliste, testez avec des données proches de la production. Une table contenant seulement dix lignes ne révèle pas toujours les problèmes d'index ou de pagination.

## 25.8. Éviter les optimisations dangereuses

Certaines pratiques semblent améliorer la vitesse mais fragilisent l'application :

- supprimer des contrôles d'autorisation pour réduire le temps de traitement ;
- retourner le modèle entier pour éviter de créer une Resource ;
- désactiver la validation ;
- mettre en cache des données privées sans séparation ;
- ajouter des index sans analyser les requêtes ;
- lancer des optimisations complexes avant d'avoir mesuré le problème.

La sécurité et l'intégrité ne sont pas des coûts que l'on peut simplement supprimer. Une API rapide qui retourne les données d'un autre utilisateur est un échec.

## 25.9. Atelier : optimiser le catalogue de services

### Situation

La route GET /api/v1/services renvoie une liste lente. Chaque service affiche le nom de la catégorie et un résumé du professionnel.

### Diagnostic

1. mesurer la durée de la requête et la taille JSON ;
2. vérifier le nombre de requêtes SQL ;
3. identifier les relations qui déclenchent N+1 ;
4. inspecter les filtres et le tri ;
5. analyser le plan SQL avec EXPLAIN ;
6. vérifier si les colonnes retournées sont nécessaires.

### Amélioration

- ajouter l'eager loading adapté ;
- sélectionner les colonnes utiles en conservant les clés relationnelles ;
- définir un tri stable ;
- limiter la taille de page ;
- créer un index seulement si le plan et les usages le justifient ;
- envisager un cache pour les catégories si leur fréquence de changement le permet.

### Vérification

Comparez les mesures avant et après. Notez le nombre de requêtes, la latence et la taille de réponse. Testez également les filtres, l'accès public et les données privées : une optimisation ne doit pas modifier le comportement fonctionnel.

## 25.10. Checklist de performance

- Les relations affichées sont-elles chargées efficacement ?
- Une liste déclenche-t-elle un N+1 ?
- Les requêtes utilisent-elles les filtres attendus ?
- Les index correspondent-ils aux usages réels ?
- La pagination limite-t-elle le volume ?
- Le JSON ne contient-il que les informations nécessaires ?
- Le cache respecte-t-il les droits et les règles d'invalidation ?
- Les performances ont-elles été mesurées sur des données réalistes ?
- Les tests fonctionnels et de sécurité passent-ils après l'optimisation ?

**À retenir :** optimiser une API, c'est réduire le travail inutile en préservant le contrat, la sécurité et la cohérence. Mesurez d'abord, améliorez une cause identifiée, puis mesurez de nouveau.
