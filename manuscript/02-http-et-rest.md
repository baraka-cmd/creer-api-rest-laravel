# Chapitre 2 — Maîtriser HTTP et les principes REST

> **Partie I — Comprendre et concevoir une API REST**  
> **Objectif du chapitre :** savoir lire, concevoir et interpréter les requêtes et réponses HTTP, choisir les méthodes adaptées et organiser les endpoints selon des conventions REST cohérentes.

## Ce que vous saurez faire à la fin

Vous pourrez :
- distinguer une requête HTTP d'une réponse HTTP ;
- expliquer GET, POST, PUT, PATCH et DELETE ;
- choisir un code de statut adapté ;
- reconnaître les headers, le body et les paramètres d'URL ;
- concevoir des endpoints centrés sur les ressources ;
- éviter les conventions qui rendent une API difficile à utiliser.

## 2.1. HTTP : le langage des échanges Web

HTTP signifie **Hypertext Transfer Protocol**. C'est un protocole de communication utilisé pour échanger des messages entre un client et un serveur.

Lorsqu'une application mobile demande la liste des services, elle envoie une requête HTTP. Le serveur traite cette requête et renvoie une réponse HTTP.

HTTP organise l'échange ; il ne décide pas à lui seul de toute la logique métier de l'application.

### HTTP et HTTPS

HTTPS correspond à HTTP utilisé avec une protection TLS. TLS contribue à :
- chiffrer les échanges en transit ;
- vérifier l'identité du serveur au moyen de son certificat ;
- protéger l'intégrité des données transportées.

Pour une API accessible sur Internet, utilisez HTTPS en production. Ne transmettez pas de mots de passe, tokens ou informations sensibles en HTTP non chiffré.

## 2.2. Anatomie d'une requête HTTP

Une requête contient généralement :
- une méthode ;
- une cible (chemin et éventuellement query string) ;
- une version HTTP ;
- des headers ;
- parfois un body.

Exemple :

```http
GET /api/services?city=Goma&page=2 HTTP/1.1
Host: api.proxiwork.test
Accept: application/json
Authorization: Bearer <token>
```

Décomposons :

| Élément | Signification |
|---|---|
| `GET` | Méthode demandant la représentation d'une ressource |
| `/api/services` | Chemin de la ressource visée |
| `city=Goma&page=2` | Query string : filtres ou paramètres de consultation |
| `Host` | Nom d'hôte visé |
| `Accept` | Types de réponse que le client accepte |
| `Authorization` | Informations d'authentification, lorsqu'elles sont nécessaires |

Le token de l'exemple est une valeur fictive. Ne publiez jamais un vrai token dans un livre, un dépôt public, une capture d'écran ou un journal.

## 2.3. Anatomie d'une réponse HTTP

Une réponse contient :
- un statut ;
- des headers ;
- éventuellement un body.

Exemple :

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
```

```json
{
  "data": {
    "id": 42,
    "title": "Installation réseau"
  }
}
```

Le statut indique le résultat général de la requête. Les headers décrivent la réponse. Le body contient souvent les données retournées.

Une réponse peut ne pas avoir de body. Par exemple, une opération réussie de suppression peut répondre avec le statut `204 No Content`.

## 2.4. Les méthodes HTTP essentielles

La méthode exprime l'intention du client. Pour une API REST, les méthodes doivent être choisies avec soin.

### GET — Lire une ressource

GET sert à récupérer une représentation sans demander de modification de l'état métier.

```http
GET /api/services
GET /api/services/42
```

- Le premier exemple consulte une collection.
- Le second consulte un service identifié par 42.

GET est généralement **safe** : il ne doit pas être conçu pour effectuer une modification métier. Il est également idempotent : répéter la même requête ne doit pas produire plusieurs modifications de l'état serveur.

Évitez donc une route comme :

```text
GET /api/services/42/delete
```

Une suppression ne doit pas être déclenchée par une simple consultation.

### POST — Créer ou déclencher une opération

POST est utilisé pour soumettre des données à une ressource afin qu'elle traite la demande. Il sert fréquemment à créer une ressource dans une collection.

```http
POST /api/services
Content-Type: application/json
```

```json
{
  "title": "Réparation informatique",
  "description": "Diagnostic et réparation"
}
```

Le serveur valide les données, vérifie les règles métier et crée la ressource si la demande est autorisée.

POST n'est pas automatiquement idempotent. Si le client répète une requête de création, il peut créer plusieurs ressources. Pour certaines opérations sensibles, une stratégie d'idempotence peut être nécessaire ; nous la traiterons dans les chapitres métier.

### PUT — Remplacer une représentation

PUT demande de créer ou de remplacer la représentation de la ressource ciblée, selon le contrat de l'API.

```http
PUT /api/services/42
```

Dans une API qui définit PUT comme remplacement complet, le client fournit les champs attendus pour l'ensemble de la représentation modifiable.

PUT est idempotent : répéter la même requête avec le même contenu doit conduire au même état final, même si la réponse peut différer.

Ne confondez pas PUT et PATCH. Le comportement précis doit être décrit dans le contrat de votre API.

### PATCH — Modifier partiellement

PATCH sert à appliquer une modification partielle à une ressource.

```http
PATCH /api/services/42
Content-Type: application/json
```

```json
{
  "title": "Réparation et maintenance informatique"
}
```

Ici, le client demande uniquement de modifier le titre. Les autres champs ne sont pas nécessairement remplacés.

PATCH n'est pas garanti idempotent par la définition générale de la méthode : cela dépend de la façon dont le serveur applique la modification. Une API peut toutefois concevoir certaines opérations PATCH de manière idempotente.

### DELETE — Supprimer une ressource

DELETE demande la suppression de l'association entre la ressource ciblée et sa représentation accessible.

```http
DELETE /api/services/42
```

La réponse peut être :
- `204 No Content` si la suppression a réussi sans contenu de réponse ;
- `200 OK` si le serveur retourne un résultat ;
- `404 Not Found` si la ressource n'existe pas ou n'est pas accessible selon la politique de l'API.

DELETE est défini comme idempotent : répéter la demande ne doit pas créer un nouvel effet de suppression. Cela ne signifie pas que toutes les répétitions produiront exactement le même statut.

### Tableau de référence

| Méthode | Usage fréquent | Safe ? | Idempotente ? |
|---|---|---:|---:|
| GET | Lire | Oui | Oui |
| POST | Créer / soumettre une opération | Non | Pas en général |
| PUT | Remplacer | Non | Oui |
| PATCH | Modifier partiellement | Non | Pas garantie |
| DELETE | Supprimer | Non | Oui |

*Safe* signifie que la méthode est conçue pour ne pas demander de modification de l'état du serveur. *Idempotent* signifie que plusieurs demandes identiques ont le même effet final sur l'état que la première.

## 2.5. Les codes de statut HTTP

Le code de statut est un nombre à trois chiffres qui indique le résultat général de la requête.

### 2xx — Succès

| Code | Signification | Exemple d'utilisation |
|---|---|---|
| 200 OK | Requête réussie | Consultation ou modification avec réponse |
| 201 Created | Ressource créée | Création d'un service |
| 202 Accepted | Demande acceptée pour traitement | Traitement asynchrone |
| 204 No Content | Succès sans body | Suppression réussie |

### 3xx — Redirection

Les codes 3xx indiquent généralement qu'une action de redirection ou une autre étape est nécessaire. Ils sont plus courants dans la navigation web que dans les API JSON simples, mais restent importants à connaître.

### 4xx — Erreur liée à la requête du client

| Code | Signification | Exemple |
|---|---|---|
| 400 Bad Request | Requête incorrecte | Format ou syntaxe non accepté |
| 401 Unauthorized | Authentification absente ou invalide | Token manquant ou expiré |
| 403 Forbidden | Accès refusé | Utilisateur authentifié mais non autorisé |
| 404 Not Found | Ressource introuvable | Identifiant inconnu |
| 409 Conflict | Conflit avec l'état actuel | Doublon ou état incompatible |
| 422 Unprocessable Content | Contenu compris mais non traité | Échec de validation métier ou des champs |
| 429 Too Many Requests | Trop de requêtes | Limite de fréquence dépassée |

Dans le langage courant, on entend parfois « 401 non autorisé » et « 403 interdit ». La distinction pratique est la suivante : 401 concerne généralement l'authentification ; 403 indique que la requête est comprise, mais que l'accès n'est pas accordé. Les réponses doivent rester cohérentes avec le mécanisme d'authentification de l'API.

### 5xx — Erreur du serveur

| Code | Signification |
|---|---|
| 500 Internal Server Error | Erreur interne inattendue |
| 502 Bad Gateway | Réponse invalide d'un serveur amont |
| 503 Service Unavailable | Service temporairement indisponible |
| 504 Gateway Timeout | Délai dépassé auprès d'un serveur amont |

En production, ne retournez pas au client une trace complète d'exception, un mot de passe, une requête SQL sensible ou des informations de configuration. Les détails techniques doivent être enregistrés de manière contrôlée dans les logs.

## 2.6. Headers, body et paramètres

### Headers

Les headers sont des métadonnées associées à la requête ou à la réponse.

Headers courants :

| Header | Rôle |
|---|---|
| `Accept` | Types de contenu que le client souhaite recevoir |
| `Content-Type` | Type de contenu envoyé ou retourné |
| `Authorization` | Informations d'authentification |
| `Cache-Control` | Directives de cache |
| `User-Agent` | Identification du logiciel client |

Pour envoyer du JSON, le client utilise généralement :

```http
Content-Type: application/json
Accept: application/json
```

### Body

Le body contient le contenu de la requête ou de la réponse. Pour créer un service, il peut contenir les champs envoyés par le client :

```json
{
  "title": "Développement web",
  "category_id": 3
}
```

Ne placez pas un secret dans un paramètre d'URL. Les URLs peuvent être conservées dans l'historique, les logs, les outils d'analyse ou les captures. Utilisez les mécanismes appropriés, notamment un header Authorization pour les tokens.

### Paramètres de chemin (path parameters)

Ils identifient généralement une ressource précise :

```text
/api/services/42
```

Dans cet exemple, `42` est l'identifiant du service.

### Query strings

Les paramètres de query string commencent après le caractère `?` et sont séparés par `&`.

```text
/api/services?city=Goma&sort=price&page=2
```

Ils sont souvent utilisés pour :
- filtrer ;
- rechercher ;
- trier ;
- paginer ;
- préciser une option de consultation.

Le serveur doit valider ces paramètres et limiter les valeurs acceptées. Un client ne doit pas pouvoir imposer librement un tri sur une colonne sensible ou demander un nombre illimité de résultats.

## 2.7. Ressources, URI et endpoints REST

REST encourage à représenter les éléments métier sous forme de **ressources**.

Une ressource peut être :
- un service ;
- un utilisateur ;
- une catégorie ;
- une demande ;
- une réservation.

Une URI identifie une ressource. Un endpoint désigne, dans l'usage courant, le point d'accès de l'API constitué de l'URI et de la méthode HTTP.

### Conventions utiles

Privilégiez :
- des noms de ressources plutôt que des verbes dans les chemins ;
- des noms cohérents, souvent au pluriel pour les collections ;
- une hiérarchie simple ;
- des identifiants stables ;
- des méthodes HTTP qui expriment l'opération.

Exemple de collection et d'élément :

```text
GET    /api/services
POST   /api/services
GET    /api/services/42
PUT    /api/services/42
PATCH  /api/services/42
DELETE /api/services/42
```

Les routes ci-dessus sont une convention de conception. Leur présence ne signifie pas que chaque opération doit être accessible à tous. Les permissions seront définies et vérifiées côté serveur.

### Exemple moins cohérent et version améliorée

Moins cohérent :

```text
GET  /api/getAllServices
POST /api/createService
GET  /api/deleteService/42
```

Plus cohérent :

```text
GET    /api/services
POST   /api/services
DELETE /api/services/42
```

Pourquoi ?
- les noms représentent les ressources ;
- la méthode exprime l'action ;
- les chemins sont plus prévisibles ;
- les clients peuvent apprendre un petit nombre de conventions.

### Ressources liées

Pour consulter les demandes d'un utilisateur, une route peut être :

```text
GET /api/users/15/requests
```

Ou, si l'utilisateur connecté ne doit consulter que ses propres demandes :

```text
GET /api/me/requests
```

Le choix dépend du contrat et du modèle de sécurité. Un identifiant dans l'URL ne donne jamais à lui seul le droit d'accéder à la ressource.

## 2.8. Construire une petite carte d'API

Avant de programmer, dessinons les opérations principales de notre projet pédagogique.

| Besoin | Méthode | Endpoint | Résultat attendu |
|---|---|---|---|
| Consulter les services | GET | `/api/services` | Liste paginée |
| Consulter un service | GET | `/api/services/{id}` | Détail |
| Créer un service | POST | `/api/services` | Ressource créée |
| Modifier un service | PATCH | `/api/services/{id}` | Ressource mise à jour |
| Supprimer un service | DELETE | `/api/services/{id}` | Suppression ou désactivation selon règles |

Cette carte constitue une première version du contrat d'API. Elle sera complétée au chapitre 3 par les champs, les règles de validation, les permissions et les exemples de réponses.

## 2.9. Erreurs de conception fréquentes

1. **Utiliser GET pour modifier des données.** Cela rend les opérations imprévisibles et peut entraîner des effets indésirables.
2. **Retourner 200 pour toutes les situations.** Le client ne peut plus distinguer correctement succès et erreur.
3. **Confondre 401 et 403.** L'authentification et l'autorisation sont deux contrôles différents.
4. **Mettre les verbes dans toutes les URLs.** Les méthodes HTTP servent déjà à exprimer les opérations.
5. **Exposer des champs internes.** Une réponse API doit être conçue, pas copiée aveuglément depuis la base.
6. **Oublier la pagination.** Une collection peut devenir très volumineuse.
7. **Faire confiance aux paramètres du client.** Le serveur doit valider les entrées et appliquer les règles de sécurité.
8. **Documenter une route sans tester son comportement.** Le contrat doit correspondre à ce que l'API fait réellement.

## Exercices d'application

### Exercice 1 — Choisir la méthode

Associez la méthode adaptée :
1. afficher le service 28 ;
2. créer une nouvelle catégorie ;
3. modifier uniquement le prix d'un service ;
4. remplacer la représentation modifiable d'un profil ;
5. supprimer une demande selon les règles métier.

### Exercice 2 — Choisir le statut

Quel statut pourrait être adapté ?
1. Une ressource a été créée.
2. Une consultation a réussi.
3. Le client n'a pas fourni un token valide.
4. L'utilisateur ne peut pas modifier le service d'un autre compte.
5. Les données envoyées échouent à la validation.
6. Le client dépasse la limite de requêtes.

### Exercice 3 — Améliorer les endpoints

Réécrivez ces routes de manière plus cohérente :

```text
GET  /api/findServices
POST /api/addNewService
GET  /api/removeService/9
```

### Exercice 4 — Lire une requête

Dans cette requête, identifiez la méthode, le chemin, les query parameters et les headers :

```http
GET /api/services?city=Goma&page=1 HTTP/1.1
Accept: application/json
```

### Corrigé commenté

**Exercice 1 :** GET ; POST ; PATCH ; PUT si le contrat prévoit le remplacement complet ; DELETE si la suppression est autorisée et correspond au modèle métier.

**Exercice 2 :** 201 Created ; 200 OK ; 401 Unauthorized ; 403 Forbidden (ou une réponse volontairement non révélatrice selon la politique de sécurité) ; 422 Unprocessable Content ; 429 Too Many Requests.

**Exercice 3 :**
```text
GET    /api/services
POST   /api/services
DELETE /api/services/9
```

**Exercice 4 :** méthode GET ; chemin `/api/services` ; paramètres `city=Goma` et `page=1` ; header `Accept: application/json`.

## Résumé du chapitre

- HTTP définit la structure des requêtes et des réponses.
- HTTPS protège les échanges en transit grâce à TLS.
- GET lit ; POST soumet ou crée ; PUT remplace ; PATCH modifie partiellement ; DELETE demande la suppression.
- Les méthodes ont des propriétés importantes : safe et idempotente.
- Les codes HTTP décrivent le résultat général.
- Les headers transportent des métadonnées ; le body contient souvent les données.
- Les paramètres de chemin identifient une ressource ; les query strings servent souvent à filtrer ou paginer.
- Les endpoints REST doivent être prévisibles et centrés sur les ressources.
- Les règles d'authentification et d'autorisation sont toujours appliquées côté serveur.

**Préparation au chapitre 3 :** nous allons transformer ces connaissances en un contrat concret : identifier les besoins métier, définir les ressources, leurs relations, les endpoints, les champs, les règles et les réponses attendues avant d'écrire le code Laravel.

## Références de lecture

- MDN Web Docs — Méthodes HTTP : https://developer.mozilla.org/fr/docs/Web/HTTP/Methods
- MDN Web Docs — Codes de statut HTTP : https://developer.mozilla.org/fr/docs/Web/HTTP/Status
- MDN Web Docs — Headers HTTP : https://developer.mozilla.org/fr/docs/Web/HTTP/Headers
- RFC 9110 — HTTP Semantics : https://www.rfc-editor.org/rfc/rfc9110
- RFC 5789 — PATCH Method for HTTP : https://www.rfc-editor.org/rfc/rfc5789
- Microsoft Learn — REST API design best practices : https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design

*Note éditoriale : valider les formulations et liens lors de la relecture technique finale.*
