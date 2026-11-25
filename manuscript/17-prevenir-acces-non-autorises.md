# Chapitre 17 — Prévenir les accès non autorisés

## 17.1. Authentification et autorisation : deux questions différentes

L’authentification répond à : « Qui fait cette requête ? » L’autorisation répond à : « Cette personne peut-elle effectuer cette action sur cette ressource ? »

Un jeton Sanctum valide identifie un utilisateur. Il ne signifie pas que cet utilisateur peut lire toutes les demandes, modifier tous les profils ou télécharger tous les documents.

Un défaut fréquent consiste à protéger une route avec `auth:sanctum`, puis à considérer le travail terminé. Le middleware garantit une identité authentifiée, mais l’application doit encore vérifier les permissions fonctionnelles et l’accès à l’objet demandé.

## 17.2. Comprendre IDOR et BOLA

IDOR signifie *Insecure Direct Object Reference*. BOLA signifie *Broken Object Level Authorization*. Dans les deux cas, un utilisateur manipule une référence à un objet — souvent un identifiant — et le serveur ne vérifie pas correctement son droit d’accès.

Exemple : Alice consulte sa demande :

```http
GET /api/requests/125
Authorization: Bearer <token-alice>
```

Elle remplace ensuite l’identifiant par 126. Si la demande 126 appartient à Bob et que l’API la retourne, le serveur présente une vulnérabilité.

Le problème n’est pas que les identifiants sont devinables. Même des UUID difficiles à deviner ne remplacent pas une autorisation. Un identifiant peut être transmis dans une notification, une URL, un journal ou une capture d’écran.

## 17.3. Vérifier la propriété et les relations métier

La vérification dépend de la règle métier. Une demande peut être consultée par son client, le professionnel auquel elle est adressée et certains administrateurs habilités. Elle ne doit pas être accessible à tout utilisateur connecté.

Une approche Laravel consiste à charger l’objet puis à demander à une Policy si l’utilisateur peut le consulter.

```php
public function show(ServiceRequest $request)
{
    $this->authorize('view', $request);

    return new ServiceRequestResource($request);
}
```

La Policy centralise la règle :

```php
public function view(User $user, ServiceRequest $request): bool
{
    return $request->client_id === $user->id
        || $request->professional_id === $user->id;
}
```

Cet exemple suppose que les identifiants client et professionnel correspondent bien au modèle d’utilisateur et que la règle métier autorise les deux parties à consulter la demande. Dans une application réelle, il faut aussi tenir compte des comptes suspendus, des états de la demande, des rôles administratifs et des éventuelles permissions plus fines.

Ne copiez pas une règle de propriété sans comprendre le modèle : une commande peut être consultable par son acheteur, mais pas modifiable par lui après livraison ; un document de vérification peut être visible uniquement par son propriétaire et un agent de vérification autorisé.

## 17.4. Restreindre les requêtes dès leur construction

Pour certaines routes, il est préférable de chercher l’objet dans la relation de l’utilisateur :

```php
$request = $user->serviceRequests()
    ->whereKey($id)
    ->firstOrFail();
```

Cette approche rend impossible la récupération d’une demande étrangère à travers cette relation. Elle ne convient cependant pas à toutes les règles : un professionnel peut accéder aux demandes qui lui sont adressées, et un administrateur peut avoir un périmètre différent.

Une bonne conception peut combiner :
- une requête limitée au périmètre autorisé ;
- une Policy pour vérifier l’action ;
- une réponse qui ne révèle pas l’existence d’un objet inaccessible, selon la stratégie de l’application.

Ne renvoyez pas automatiquement un objet complet puis ne filtrez les données qu’après coup.

## 17.5. Protéger les routes de modification

Pour une modification, contrôlez à la fois l’accès à l’objet et les champs modifiables.

Exemple : un professionnel peut modifier la description de son service, mais pas attribuer le service à un autre compte ni le marquer comme vérifié.

```php
$this->authorize('update', $service);

$data = $request->validated();

$service->update([
    'title' => $data['title'],
    'description' => $data['description'],
]);
```

Le code illustre une liste explicite de champs. Dans un projet complet, un Form Request peut fournir les données validées, et la Policy peut vérifier le rôle, la propriété et l’état du service.

Évitez `$service->update($request->all())` : la requête contient potentiellement des champs qui n’ont pas été prévus pour cette action.

## 17.6. Policies, Gates et middleware : choisir le bon niveau

- **Middleware** : contrôle transversal, par exemple utilisateur authentifié, compte actif ou accès à une zone administrative.
- **Gate** : autorisation simple, souvent liée à une capacité globale.
- **Policy** : règles organisées autour d’un modèle, comme consulter, modifier ou supprimer un service.

Une route d’administration peut utiliser un middleware de rôle ou de permission, mais une action sur un utilisateur précis doit également vérifier la permission sur cet objet.

La sécurité doit être appliquée au point où l’action est exécutée. Cacher une route dans l’interface ou vérifier seulement le rôle dans JavaScript n’est pas suffisant.

## 17.7. Protéger les données privées et les fichiers

Les fichiers privés exigent une autorisation distincte. Le fait de connaître le nom d’un fichier ne doit pas permettre son téléchargement.

Une route de téléchargement doit :
1. retrouver le document dans la base ;
2. vérifier la Policy correspondante ;
3. vérifier que le fichier existe dans le stockage privé ;
4. renvoyer le fichier sans exposer le chemin interne ;
5. éviter de journaliser le contenu sensible.

Ne placez pas des documents d’identité ou des justificatifs dans un répertoire public simplement parce que le téléchargement est plus facile.

## 17.8. Tests d’accès entre utilisateurs : méthode indispensable

Créez au minimum deux utilisateurs indépendants dans les tests : Alice et Bob. Alice crée une ressource ; Bob tente de la lire, de la modifier et de la supprimer.

Exemple de test Feature :

```php
public function test_user_cannot_view_another_users_request(): void
{
    $alice = User::factory()->create();
    $bob = User::factory()->create();

    $request = ServiceRequest::factory()->create([
        'client_id' => $alice->id,
    ]);

    $this->actingAs($bob, 'sanctum')
        ->getJson('/api/requests/'.$request->id)
        ->assertForbidden();
}
```

Ce test est un modèle à adapter aux factories, relations, routes et règles de votre application. Selon votre stratégie de confidentialité, une ressource étrangère peut répondre 404 plutôt que 403. L’important est qu’aucune donnée ne soit divulguée et que l’action soit refusée.

Complétez le test par :
- Alice peut consulter sa demande ;
- le professionnel destinataire peut la consulter si la règle l’autorise ;
- Bob ne peut pas la modifier ;
- un utilisateur non authentifié est refusé ;
- un administrateur n’accède qu’aux actions prévues par ses permissions.

## 17.9. Atelier : audit d’une route vulnérable

Code à risque :

```php
Route::get('/requests/{id}', function ($id) {
    return ServiceRequest::findOrFail($id);
})->middleware('auth:sanctum');
```

Pourquoi est-il vulnérable ? Il vérifie l’identité, mais pas le droit d’accès à la demande. L’ID de la route est traité comme une preuve d’autorisation.

Correction de conception : utiliser un contrôleur, une Policy et un Resource ; charger l’objet, autoriser l’action, puis retourner uniquement les champs nécessaires. Ajoutez un test où un autre utilisateur essaie l’identifiant.

## À retenir

Chaque objet doit avoir une règle d’accès explicite. Les identifiants ne sont jamais des permissions. Testez systématiquement les scénarios entre utilisateurs, car les tests où l’utilisateur accède à ses propres données ne détectent pas les défauts BOLA.