# Chapitre 21 — Automatiser les tests avec PHPUnit

## 21.1. Pourquoi automatiser ?

Un test manuel confirme qu’un comportement fonctionne à un moment donné. Mais après une modification du code, il faut recommencer les vérifications. À mesure que le projet grandit, cette répétition devient coûteuse et certaines vérifications sont oubliées.

Les tests automatisés exécutent des scénarios de manière reproductible. Ils permettent de détecter une régression : une fonctionnalité qui fonctionnait auparavant et qui cesse de fonctionner après une modification.

Laravel fournit un environnement de test intégré, avec PHPUnit ou Pest selon le choix du projet. Dans ce chapitre, les exemples utilisent le style PHPUnit.

## 21.2. Distinguer les types de tests

### Test unitaire
Il vérifie une petite unité de logique, généralement isolée de la base de données et des services externes. Exemple : calculer une commission selon une règle donnée.

### Test fonctionnel ou Feature
Il vérifie un comportement à travers plusieurs composants de l’application. Un test Feature Laravel peut envoyer une requête HTTP simulée, traverser les routes et middleware, exécuter le contrôleur et vérifier la réponse.

### Test d’intégration
Il vérifie la collaboration entre composants, par exemple service métier, base de données et stockage. La frontière entre Feature et intégration dépend de l’organisation du projet ; l’essentiel est de préciser ce qui est réellement testé.

## 21.3. Préparer l’environnement de test

Les tests ne doivent pas utiliser les données réelles de production. Configurez une base dédiée ou un moteur de base de données de test, puis utilisez les mécanismes de Laravel pour réinitialiser l’état entre les tests.

Les variables de configuration de test doivent être séparées des variables locales et de production. Vérifiez que les tests ne peuvent pas supprimer accidentellement une base importante.

Une stratégie courante consiste à utiliser `RefreshDatabase` :

```php
use Illuminate\Foundation\Testing\RefreshDatabase;

class ServiceApiTest extends TestCase
{
    use RefreshDatabase;
}
```

Cette approche prépare la base selon les migrations dans le contexte de test. Elle peut être plus lente sur une grande suite ; choisissez ensuite une stratégie adaptée à votre volume de tests.

## 21.4. Utiliser les factories

Les factories permettent de créer des données de test cohérentes sans écrire manuellement chaque insertion.

```php
$user = User::factory()->create();
$professionalProfile = ProfessionalProfile::factory()->for($user)->create();
$service = Service::factory()->create([
    'professional_profile_id' => $professionalProfile->id,
]);
```

Une factory doit produire des données valides et réalistes. Les relations doivent être explicites pour éviter les tests qui passent uniquement parce que des données accidentelles existent dans la base.

Pour tester l’autorisation, créez plusieurs acteurs distincts plutôt que de réutiliser un seul utilisateur pour toutes les opérations.

## 21.5. Tester une route publique

```php
public function test_published_services_are_available(): void
{
    Service::factory()->create([
        'status' => 'published',
    ]);

    $response = $this->getJson('/api/v1/services');

    $response->assertOk()
        ->assertJsonStructure([
            'data',
        ]);
}
```

Cet exemple doit être adapté à la structure réelle du modèle et aux règles de visibilité. Une API publique ne doit pas afficher automatiquement tous les services : publication, validation du professionnel et état du compte peuvent être des conditions métier.

## 21.6. Tester l’authentification

```php
public function test_private_endpoint_requires_authentication(): void
{
    $this->getJson('/api/me')
        ->assertUnauthorized();
}
```

Puis testez le cas authentifié :

```php
public function test_authenticated_user_can_read_own_profile(): void
{
    $user = User::factory()->create();

    $this->actingAs($user, 'sanctum')
        ->getJson('/api/me')
        ->assertOk();
}
```

Le nom du guard et la route doivent correspondre à la configuration réelle. Un test doit vérifier le comportement observé, pas simplement refléter le nom d’une méthode.

## 21.7. Tester l’autorisation et les accès croisés

Un test de sécurité doit reproduire un scénario où l’utilisateur n’est pas propriétaire de l’objet.

```php
public function test_client_cannot_update_another_clients_request(): void
{
    $owner = User::factory()->create();
    $other = User::factory()->create();

    $request = ServiceRequest::factory()->create([
        'client_id' => $owner->id,
    ]);

    $response = $this->actingAs($other, 'sanctum')
        ->patchJson('/api/requests/'.$request->id, [
            'description' => 'Modification non autorisée',
        ]);

    $response->assertForbidden();
}
```

Vérifiez aussi que la base n’a pas été modifiée. Une réponse 403 n’est pas suffisante si le code a déjà effectué une modification avant de refuser l’action.

Selon votre stratégie, un 404 peut être attendu pour éviter de révéler l’existence d’une ressource. Le test doit respecter le contrat décidé.

## 21.8. Tester la validation

Pour chaque Form Request, testez les données valides et plusieurs données invalides. Les cas utiles incluent un champ manquant, une chaîne trop longue, un identifiant inexistant, une valeur hors liste et un champ privilégié injecté.

```php
$response = $this->actingAs($user, 'sanctum')
    ->postJson('/api/v1/services', [
        'title' => '',
        'description' => 'Description',
    ]);

$response->assertUnprocessable()
    ->assertJsonValidationErrors(['title']);
```

Laravel retourne généralement 422 pour une erreur de validation d’une requête JSON. Vérifiez la réponse selon la version et la configuration du projet.

## 21.9. Tests unitaires des règles métier

Une logique de commission, de transition d’état ou de calcul de prix mérite des tests ciblés. Par exemple, testez :
- un montant positif ;
- un montant nul si autorisé ;
- une valeur négative ;
- les limites de commission ;
- les arrondis ;
- une transition interdite ;
- une opération déjà exécutée.

Ne testez pas uniquement le résultat heureux. Les limites et les erreurs révèlent souvent les défauts les plus coûteux.

## 21.10. Exécuter et interpréter la suite

Dans un projet Laravel, la commande habituelle est :

```bash
php artisan test
```

Ou, selon la configuration :

```bash
vendor/bin/phpunit
```

Un test rouge signifie qu’une attente n’est pas satisfaite. Il faut comprendre si le défaut vient du code, de la fixture, de la configuration ou du test lui-même. Ne modifiez pas l’assertion uniquement pour faire disparaître l’échec sans vérifier le comportement attendu.

## 21.11. Atelier : une suite minimale de qualité

Pour un endpoint de création de demande, écrivez au minimum :
- une création valide ;
- une absence d’authentification ;
- une permission insuffisante ;
- un service inexistant ;
- un service non publié ;
- un champ invalide ;
- un champ privilégié injecté ;
- une répétition ou un doublon si le métier l’exige.

Cette suite constitue un filet de sécurité pour les futures évolutions.

## À retenir

Les tests automatisés transforment les règles métier et de sécurité en vérifications reproductibles. Les tests Feature couvrent le parcours HTTP, les tests unitaires isolent les calculs, et les factories rendent les scénarios maîtrisés. Une API professionnelle teste aussi les refus et les limites.

> **Convention de référence du projet fil conducteur (mise à jour éditoriale).** Les exemples d'URL du projet utilisent le préfixe public `/api/v1`. La propriété propriétaire d'un service est `professional_profile_id` et référence le profil professionnel, non directement l'identifiant utilisateur. Le serveur déduit ce propriétaire de l'utilisateur authentifié ; cette valeur ne doit jamais être acceptée aveuglément depuis le client. Les chapitres d'extension doivent respecter ces conventions ou annoncer explicitement une variante.
