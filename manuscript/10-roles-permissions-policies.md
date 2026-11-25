# Chapitre 10 — Gérer les rôles et les permissions

## Objectifs

Ce chapitre clarifie les rôles, les permissions, les middleware et les policies Laravel. Vous apprendrez à protéger une action globale et à vérifier l'accès à une ressource précise.

## 10.1 Authentification et autorisation : deux contrôles

L'authentification identifie l'appelant. L'autorisation décide ce que cet appelant peut faire.

Exemple : un utilisateur authentifié peut consulter son profil, mais ne doit pas modifier le profil professionnel d'un autre compte. La présence d'un token ne constitue donc jamais une autorisation universelle.

## 10.2 Définir les rôles métier

Pour ProxiServices, nous pouvons distinguer :

- `client` : recherche des prestations et crée des demandes ;
- `professional` : gère son profil et ses prestations ;
- `admin` : supervise les opérations administratives.

Un rôle est un regroupement de responsabilités. Évitez de construire un système dans lequel chaque action dépend d'un nombre incontrôlable de conditions écrites directement dans les contrôleurs.

Le rôle doit être enregistré côté serveur. Il ne doit jamais être accepté comme une valeur fiable depuis une requête d'inscription publique.

## 10.3 Une première protection par middleware

Un middleware peut interdire l'accès à un espace réservé à certains rôles. La logique doit lire le rôle depuis l'utilisateur authentifié.

Exemple conceptuel de middleware :

```php
public function handle($request, Closure $next, string $role)
{
    $user = $request->user();

    if (!$user || $user->role !== $role) {
        abort(403, 'Accès interdit.');
    }

    return $next($request);
}
```

Enregistrez l'alias du middleware selon la structure de configuration de votre version Laravel, puis utilisez-le sur les routes concernées. Ne copiez pas un emplacement de configuration provenant d'une version différente sans vérifier votre application.

Le middleware convient aux contrôles transversaux, par exemple l'accès à un groupe de routes administratives. Il devient insuffisant lorsque la décision dépend de la ressource précise.

## 10.4 Policies : autoriser une action sur un objet

Une policy regroupe les règles d'autorisation relatives à un modèle. Générez-la :

```bash
php artisan make:policy ServicePolicy --model=Service
```

Exemple :

```php
public function update(User $user, Service $service): bool
{
    return $user->professionalProfile
        && $service->professional_profile_id
            === $user->professionalProfile->id;
}
```

Cette règle compare l'identité authentifiée au propriétaire réel de la prestation. Elle ne prend pas le propriétaire depuis le JSON envoyé par le client.

Dans le contrôleur, l'autorisation peut être vérifiée avec :

```php
$this->authorize('update', $service);
```

Si l'utilisateur n'a pas le droit, Laravel renvoie une réponse 403 selon la gestion des exceptions configurée.

## 10.5 Permissions par ressource

Il faut distinguer plusieurs actions :

- consulter une prestation publique ;
- consulter une prestation privée ;
- créer une prestation ;
- modifier une prestation ;
- publier une prestation ;
- supprimer une prestation.

Un utilisateur peut avoir le droit de modifier une ressource sans avoir le droit de la publier. La policy peut donc contenir des méthodes distinctes, comme `view`, `update`, `publish` et `delete`.

La décision doit tenir compte du propriétaire, de l'état de la ressource et parfois du rôle administratif. Toute exception d'administration doit être explicite et testée.

## 10.6 Éviter le contrôle uniquement dans l'interface

Cacher un bouton « Supprimer » dans l'application mobile n'empêche pas un utilisateur d'envoyer directement une requête DELETE. Le contrôle doit être effectué sur le serveur à chaque requête sensible.

Ce risque est souvent désigné par le terme BOLA/IDOR : un utilisateur modifie un identifiant dans l'URL et accède à un objet qui ne lui appartient pas. Nous approfondirons les tests de ce risque dans la partie sécurité.

## 10.7 Atelier : deux comptes, deux propriétaires

1. Créez deux comptes professionnels.
2. Créez une prestation avec le premier compte.
3. Connectez-vous avec le second.
4. Tentez de modifier la prestation du premier.
5. Vérifiez que la requête est refusée.
6. Testez ensuite le compte propriétaire.

Le test ne doit pas se limiter au rôle. Il doit vérifier la relation entre l'utilisateur authentifié et la ressource.

## Exercices corrigés

**Un middleware peut-il remplacer toutes les policies ?** Non. Il protège souvent un groupe de routes ou une condition générale ; une policy contrôle une action sur un modèle précis.

**Le client peut-il envoyer `role=admin` ?** Il peut techniquement envoyer ce champ, mais le serveur doit l'ignorer ou le rejeter. L'attribution des rôles relève d'une opération administrative autorisée.

**Pourquoi tester deux comptes ?** Pour vérifier qu'un utilisateur ne peut pas agir sur les ressources d'un autre, même si l'identifiant est connu.

## À retenir

Un système d'autorisation robuste combine des contrôles globaux et des décisions liées aux ressources. Il doit être centralisé, explicite et testé avec des utilisateurs ayant des droits différents.

## Références

- Laravel Authorization : https://laravel.com/docs/authorization
- Laravel Middleware : https://laravel.com/docs/middleware
- OWASP API Security : https://owasp.org/www-project-api-security/
