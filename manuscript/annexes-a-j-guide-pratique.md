# Annexes pratiques — Référence de terrain pour l'API Laravel

> Ces annexes complètent les chapitres. Les règles du cahier ProxiServices API priment lorsqu'un exemple générique diffère du projet fil conducteur.

## Annexe A — Préparer l'environnement

### Vérification de l'installation

Avant de créer le projet, vérifiez les versions et extensions réellement disponibles :

```bash
php -v
php -m
composer --version
git --version
```

Comparez les résultats aux exigences de la version Laravel choisie. Une version PHP compatible ne suffit pas si une extension requise manque.

### Contrôles après création

```bash
php artisan about
php artisan route:list
php artisan migrate:status
php artisan test
```

Gardez un fichier de notes indiquant OS, PHP, Composer, Laravel, moteur SQL et versions des dépendances. Ces informations facilitent le diagnostic lorsqu'un collègue reproduit l'installation.

### Environnement local et production

- Le fichier `.env` reste local et n'est jamais commité.
- `.env.example` contient des noms de variables et des valeurs non secrètes.
- Les bases de développement, test et production sont séparées.
- Les migrations sont relues avant application.
- Une installation propre doit être testée à partir du dépôt, pas seulement sur le poste qui a servi à développer.

## Annexe B — Mémo HTTP

| Méthode | Intention courante | Propriété à retenir |
|---|---|---|
| GET | Lire une ressource | Safe et idempotente |
| POST | Soumettre/créer | Non idempotente en général |
| PUT | Remplacer | Idempotente |
| PATCH | Modifier partiellement | Idempotence dépend de l'opération |
| DELETE | Supprimer | Idempotente |

Codes fréquemment utilisés :

- 200 : succès avec représentation ;
- 201 : ressource créée ;
- 202 : traitement accepté ;
- 204 : succès sans contenu ;
- 400 : requête incorrecte ;
- 401 : authentification requise/invalide ;
- 403 : action interdite ;
- 404 : ressource absente ou volontairement non révélée ;
- 409 : conflit d'état ;
- 422 : validation refusée ;
- 429 : limite de fréquence dépassée ;
- 500 : erreur inattendue côté serveur.

Ne choisissez pas un statut uniquement parce qu'il « fonctionne » : documentez-le et testez-le.

## Annexe C — Checklist sécurité API

Avant de publier une route, vérifiez :

- [ ] La route exige-t-elle l'authentification appropriée ?
- [ ] La permission est-elle vérifiée pour l'objet précis ?
- [ ] Le propriétaire est-il dérivé du compte authentifié ?
- [ ] Les rôles et statuts sensibles sont-ils définis côté serveur ?
- [ ] Les données entrantes sont-elles validées par liste blanche ?
- [ ] Les réponses excluent-elles les champs privés et internes ?
- [ ] Les fichiers sont-ils validés et leur accès contrôlé ?
- [ ] Les limites de fréquence sont-elles adaptées ?
- [ ] Les erreurs ne révèlent-elles ni secrets ni traces techniques ?
- [ ] Un test négatif prouve-t-il qu'un autre utilisateur est refusé ?
- [ ] Les opérations sensibles sont-elles transactionnelles et idempotentes si nécessaire ?
- [ ] Les logs évitent-ils tokens, mots de passe et données personnelles inutiles ?

## Annexe D — Checklist de déploiement

### Avant déploiement

- [ ] Tests et vérifications de code réussis.
- [ ] Dépendances de production installées selon le lockfile.
- [ ] Secrets définis dans l'environnement de l'hébergeur.
- [ ] `APP_DEBUG=false` en production.
- [ ] HTTPS activé.
- [ ] Base sauvegardée et migrations examinées.
- [ ] Stockage et permissions configurés.
- [ ] Logs et alertes opérationnels.
- [ ] Procédure de retour arrière connue.

### Après déploiement

- [ ] L'application démarre correctement.
- [ ] Le endpoint de santé répond.
- [ ] Une route publique fonctionne.
- [ ] Une route protégée refuse un invité.
- [ ] Authentification et révocation testées.
- [ ] Upload contrôlé vérifié.
- [ ] Logs consultés.
- [ ] DNS, certificat TLS et URL de base vérifiés.

Ne lancez jamais `migrate:fresh` sur une base de production.

## Annexe E — Modèles JSON de référence

### Réponse de ressource

```json
{
  "data": {
    "id": 42,
    "title": "Installation réseau",
    "is_published": true
  }
}
```

### Réponse paginée

```json
{
  "data": [],
  "links": {
    "first": "https://api.example.test/api/v1/services?page=1",
    "last": "https://api.example.test/api/v1/services?page=4",
    "prev": null,
    "next": "https://api.example.test/api/v1/services?page=2"
  },
  "meta": {
    "current_page": 1,
    "per_page": 15,
    "last_page": 4,
    "total": 52
  }
}
```

### Erreur de validation

```json
{
  "message": "Les données fournies sont invalides.",
  "errors": {
    "title": [
      "Le titre est obligatoire."
    ]
  }
}
```

Les enveloppes exactes doivent correspondre aux API Resources et au comportement de la version Laravel installée. Ne créez pas un format de réponse parallèle dans chaque contrôleur.

## Annexe F — Collection Postman

Une collection exportable est fournie dans le fichier `resources/postman/ProxiServices-API.postman_collection.json`. Importez-la dans Postman, définissez `base_url`, puis renseignez `token` après une connexion réussie.

Pour partager la collection :
- retirez tout token personnel ;
- ne mettez aucun mot de passe dans les exemples ;
- utilisez des comptes de démonstration dédiés ;
- vérifiez que les chemins correspondent au contrat et à la version réellement déployée.

## Annexe G — Glossaire

- **API** : interface permettant à des systèmes d'échanger selon un contrat.
- **REST** : style d'architecture fondé notamment sur les ressources et les contraintes HTTP.
- **Endpoint** : point d'accès composé généralement d'une méthode et d'un chemin.
- **JSON** : format textuel d'échange de données.
- **Middleware** : composant qui intervient dans le traitement d'une requête.
- **Eloquent** : ORM Laravel pour interagir avec les données relationnelles.
- **Migration** : fichier versionnant une évolution du schéma de base.
- **Factory** : définition de données de test générées.
- **Seeder** : classe d'alimentation de données initiales ou de démonstration.
- **Form Request** : classe Laravel dédiée à la validation et à l'autorisation d'une requête.
- **Policy** : classe de règles d'autorisation associées à un modèle ou une ressource.
- **API Resource** : couche qui définit les données exposées en JSON.
- **BOLA/IDOR** : défaut d'autorisation permettant d'accéder à un objet en manipulant son identifiant.
- **Idempotence** : propriété selon laquelle répéter une opération produit le même effet final.
- **N+1** : problème de requêtes répétées déclenchées lors de l'accès à une relation.
- **OpenAPI** : format de description d'une API HTTP.
- **Sanctum** : solution Laravel d'authentification pour SPA et tokens API simples.

## Annexe H — Exercices supplémentaires

### Exercice 1 — Validation
Un service accepte un titre vide, une catégorie inexistante et un prix négatif. Proposez les règles de validation et les tests correspondants.

### Exercice 2 — Permissions
Deux professionnels possèdent chacun un service. Écrivez les scénarios autorisés et refusés pour GET, PATCH et DELETE.

### Exercice 3 — Performance
Une liste de 30 services déclenche 61 requêtes. Expliquez le problème probable, proposez une stratégie d'eager loading et indiquez comment mesurer le résultat.

### Exercice 4 — Incident
Après déploiement, la connexion à la base échoue. Décrivez une démarche sûre : logs, variables, réseau, disponibilité DB, migration récente, sauvegarde et rollback.

### Corrigés indicatifs

1. Règles : titre requis et longueur bornée ; catégorie existante et active selon le besoin ; prix numérique non négatif si le prix est prévu. Testez succès et erreurs 422.
2. Le propriétaire agit sur son objet ; un autre compte est refusé ou reçoit 404 selon le contrat. Testez les deux utilisateurs et vérifiez que la base n'a pas changé.
3. Inspectez les requêtes ; chargez les relations avec `with()` ; comparez nombre et durée avant/après sans sacrifier les permissions.
4. Ne désactivez pas la sécurité pour « faire passer » le service. Vérifiez les logs sans exposer secrets, comparez la configuration, restaurez ou revenez à la version précédente selon une procédure contrôlée.

## Annexe I — Dépannage Laravel

| Symptôme | Vérifications initiales |
|---|---|
| 500 | Logs Laravel, exception réelle, configuration, permissions |
| 404 route | `route:list`, préfixe, méthode, binding, cache routes |
| 401 API | Header Authorization, guard, token, middleware |
| 403 | Policy, rôle, propriété et état du compte |
| 422 | Body JSON, règles, champs requis, réponse errors |
| Migration échoue | Ordre, type SQL, valeur par défaut, clé étrangère, moteur |
| Table inconnue | Migration exécutée sur la bonne base, nom table/model |
| N+1 | Relations chargées, requêtes SQL observées |
| Upload refusé | Taille, MIME, extension, disk et permissions |
| Cache incohérent | Clé, durée, invalidation, contexte utilisateur |

Pour diagnostiquer, reproduisez le problème dans un environnement contrôlé, identifiez la cause et ajoutez un test ou une vérification qui empêche sa réapparition.

## Annexe J — Références officielles et ressources

- Laravel 13 Documentation : https://laravel.com/docs/13.x
- Laravel Sanctum : https://laravel.com/docs/13.x/sanctum
- Laravel Validation : https://laravel.com/docs/13.x/validation
- Laravel Authorization : https://laravel.com/docs/13.x/authorization
- Laravel Testing : https://laravel.com/docs/13.x/testing
- Laravel Deployment : https://laravel.com/docs/13.x/deployment
- MDN HTTP : https://developer.mozilla.org/fr/docs/Web/HTTP
- RFC 9110 HTTP Semantics : https://www.rfc-editor.org/rfc/rfc9110
- OpenAPI Specification : https://spec.openapis.org/oas/latest.html
- OWASP API Security Top 10 : https://owasp.org/www-project-api-security/
- Postman Learning Center : https://learning.postman.com/

Vérifiez la version et la date de consultation de toute référence avant la publication commerciale, car les documentations et exigences évoluent.
