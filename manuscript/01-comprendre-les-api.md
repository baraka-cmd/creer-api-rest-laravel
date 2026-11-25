# Chapitre 1 — Comprendre les API et leur rôle dans les applications modernes

> **Partie I — Comprendre et concevoir une API REST**  
> **Objectif du chapitre :** comprendre ce qu'est une API, pourquoi elle est utile, comment elle relie une application à un serveur et comment les données sont échangées.

## Ce que vous saurez faire à la fin

À la fin de ce chapitre, vous pourrez :
- expliquer une API avec vos propres mots ;
- distinguer une API web, REST, SOAP et GraphQL ;
- décrire le trajet d'une requête entre un téléphone et un backend ;
- reconnaître les rôles du client, du serveur et de la base de données ;
- lire un document JSON simple.

## 1.1. Le problème que les API résolvent

Imaginez une application mobile de réservation de services. Un client ouvre l'application, consulte les prestataires disponibles, choisit un service et envoie une demande.

L'application affiche des informations, mais elle ne possède pas nécessairement toutes ces informations dans le téléphone. Les comptes, les services, les demandes et les prix sont généralement conservés sur un serveur central. Le téléphone doit donc demander des données au serveur et lui transmettre certaines actions.

C'est ici qu'intervient l'API.

**Une API (Application Programming Interface, ou interface de programmation d'application) est un ensemble de règles et de points d'accès qui permet à un logiciel de demander des services ou des données à un autre logiciel.**

Une API joue le rôle d'une interface organisée. Elle précise :
- quelles opérations sont disponibles ;
- quelles informations doivent être envoyées ;
- comment le serveur répond ;
- quelles erreurs peuvent être retournées ;
- quelles règles d'accès doivent être respectées.

### Une analogie : le guichet

Dans une administration, vous ne pouvez pas entrer librement dans tous les bureaux ni consulter tous les dossiers. Vous passez par un guichet qui reçoit votre demande, vérifie les conditions et vous transmet le résultat autorisé.

Une API fonctionne de manière comparable :
- **le client** formule une demande ;
- **l'API** expose les opérations autorisées ;
- **le serveur** traite la demande ;
- **la réponse** indique le résultat.

Cette comparaison aide à comprendre le rôle de l'interface, mais il faut garder une différence : une API n'est pas une personne. C'est un contrat technique appliqué par des logiciels.

## 1.2. API, application et backend : ne pas confondre

| Terme | Rôle |
|---|---|
| Application cliente | Interface utilisée par une personne : mobile, navigateur ou logiciel |
| API | Interface technique par laquelle le client demande des opérations |
| Backend | Partie serveur qui applique les règles métier, traite les demandes et accède aux données |
| Base de données | Système qui conserve les données de manière organisée |

Dans notre projet, l'application Flutter ou le frontend JavaScript sera le client. Laravel constituera le backend. Les routes API Laravel seront les points d'entrée de l'interface. MariaDB ou MySQL pourra stocker les données.

**À retenir :** une API n'est pas toute l'application. Elle est une interface par laquelle d'autres logiciels utilisent certaines capacités de l'application.

## 1.3. Qu'est-ce qu'une API web ?

Une API web est une API accessible au moyen de technologies du Web, généralement à travers HTTP ou HTTPS.

Elle peut être consommée par :
- une application Android ou iOS ;
- un site web exécuté dans un navigateur ;
- une application desktop ;
- un autre serveur ;
- un outil de test comme Postman.

Exemple d'une API de catalogue :

```text
https://api.exemple.com/api/services
```

Cette adresse peut représenter un point d'accès qui permet de consulter les services publiés. Le serveur peut répondre avec des données JSON.

Le mot **endpoint** désigne couramment un point d'accès précis de l'API, identifié par une URL et généralement associé à une méthode HTTP. Nous étudierons cette notion en détail au chapitre 2.

Une API web ne signifie pas automatiquement « API REST ». Il existe plusieurs styles et protocoles d'API web.

## 1.4. REST, SOAP et GraphQL : les différences essentielles

Ces termes ne désignent pas exactement la même chose. REST est un style architectural, SOAP est un protocole, et GraphQL est un langage de requête accompagné d'un système d'exécution.

### REST

REST signifie **Representational State Transfer**. C'est un style architectural défini autour de contraintes, notamment la séparation client-serveur, l'absence d'état de session côté serveur dans les échanges REST, une interface uniforme et la possibilité de mise en cache.

Dans une API REST courante :
- les données sont représentées par des ressources ;
- chaque ressource possède une URI ;
- les méthodes HTTP expriment les opérations ;
- les représentations sont souvent échangées en JSON.

Exemples de ressources :
- utilisateurs ;
- catégories ;
- services ;
- demandes de service.

Exemples d'URI :
```text
/api/services
/api/services/42
/api/categories
```

REST ne signifie pas simplement « envoyer du JSON ». Une API peut envoyer du JSON sans respecter les principes REST.

### SOAP

SOAP (Simple Object Access Protocol) est un protocole de messagerie structuré. Il utilise généralement des messages XML et des contrats formels, souvent décrits par WSDL.

SOAP reste présent dans certains systèmes d'entreprise, notamment lorsque des contrats stricts, des extensions de sécurité ou des mécanismes de messagerie spécifiques sont nécessaires.

### GraphQL

GraphQL est un langage de requête pour les API et un environnement d'exécution qui traite ces requêtes selon un schéma.

Il permet au client de demander précisément les champs dont il a besoin. Une API GraphQL utilise souvent un endpoint unique, mais le client décrit dans sa requête les données souhaitées.

### Comparaison

| Critère | REST | SOAP | GraphQL |
|---|---|---|---|
| Nature | Style architectural | Protocole | Langage de requête et runtime |
| Modèle | Ressources et opérations HTTP | Messages et opérations structurées | Requête de données selon un schéma |
| Format fréquent | JSON | XML | Souvent JSON pour la réponse |
| Points d'accès | Plusieurs URI de ressources, généralement | Selon le service | Souvent un endpoint |
| Atout courant | Simplicité et conventions Web | Contrats et extensions protocolaires | Sélection précise des champs |
| Attention | Concevoir correctement ressources et statuts | Plus de structure et de complexité | Schéma, résolveurs et maîtrise des requêtes |

Il n'existe pas une technologie supérieure dans tous les cas. Le choix dépend des besoins, des clients, des contraintes d'intégration et des compétences de l'équipe.

**Pour notre livre :** nous utiliserons une API REST avec Laravel, car ce choix correspond bien à un backend web destiné à être consommé par Flutter ou JavaScript et permet d'apprendre les conventions HTTP de manière concrète.

## 1.5. Comment une application mobile communique avec un backend

Une application mobile ne doit pas, en règle générale, se connecter directement à la base de données de production. Elle communique avec le backend, qui contrôle les accès et applique les règles métier.

### Le parcours d'une requête

<text color="secondary" size="sm">Exemple : le téléphone demande la liste des services publiés.</text>

```text
Utilisateur
   |
   v
Application mobile (client)
   |  Requête HTTPS
   v
API Laravel (backend)
   |  Validation et règles métier
   v
Base de données
   |
   v
API Laravel
   |  Réponse JSON + statut HTTP
   v
Application mobile
   |
   v
Affichage des services
```

Étape par étape :

1. L'utilisateur ouvre l'écran « Services ».
2. L'application construit une requête vers l'endpoint prévu.
3. Elle envoie la requête au serveur à travers HTTPS.
4. Laravel reçoit la requête et vérifie ce qui est nécessaire : paramètres, authentification et autorisation.
5. Le backend interroge la base de données si les données y sont stockées.
6. Laravel prépare une réponse adaptée, souvent au format JSON.
7. Le client reçoit la réponse et affiche les résultats.

Le client ne doit pas décider seul qu'un utilisateur a le droit de modifier un service. Cette décision appartient au serveur, qui doit vérifier les permissions à chaque opération sensible.

### Exemple de requête et de réponse

Requête simplifiée :

```http
GET /api/services HTTP/1.1
Host: api.projet.test
Accept: application/json
```

Réponse possible :

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "data": [
    {
      "id": 12,
      "title": "Réparation informatique",
      "price": 25
    },
    {
      "id": 18,
      "title": "Installation réseau",
      "price": 40
    }
  ]
}
```

Les valeurs de prix ci-dessus sont des exemples pédagogiques. Dans une application réelle, le serveur doit définir la devise, les règles de calcul et la manière dont les montants sont représentés.

## 1.6. Comprendre l'architecture client–serveur

L'architecture client–serveur sépare les responsabilités entre les logiciels qui demandent un service et ceux qui le fournissent.

- **Client :** initie la requête et présente les résultats.
- **Serveur :** reçoit les requêtes, applique les règles et renvoie une réponse.
- **Base de données :** conserve les informations nécessaires au fonctionnement.

Cette séparation présente plusieurs avantages :
- plusieurs clients peuvent utiliser le même backend ;
- les règles métier sont centralisées ;
- les données ne sont pas exposées directement aux applications ;
- le backend peut évoluer sans réécrire toute l'interface, si le contrat reste compatible.

### Un backend, plusieurs clients

```text
Application Flutter ──┐
Site JavaScript ───────┼──> API Laravel ──> Base de données
Application desktop ───┘
```

Chaque client peut avoir une interface différente, mais utiliser les mêmes opérations API.

Attention : cette architecture ne rend pas automatiquement l'application sécurisée. La sécurité dépend des contrôles réellement appliqués par le serveur, de la configuration HTTPS, de l'authentification, des autorisations et de la validation.

## 1.7. JSON : le format d'échange de données

JSON signifie **JavaScript Object Notation**. Malgré son nom, il est utilisé par de nombreux langages : PHP, JavaScript, Dart, Python, Java, etc.

JSON est un format texte léger qui représente des données structurées.

### Les principales valeurs JSON

| Type | Exemple |
|---|---|
| Chaîne de caractères | `"Goma"` |
| Nombre | `25` |
| Booléen | `true` ou `false` |
| Valeur nulle | `null` |
| Objet | `{"id": 7}` |
| Tableau | `["web", "mobile"]` |

### Exemple d'objet JSON

```json
{
  "id": 7,
  "name": "Baraka",
  "city": "Goma",
  "is_active": true,
  "skills": ["Laravel", "Flutter"],
  "profile": {
    "level": "junior"
  }
}
```

Explication :
- les accolades `{}` délimitent un objet ;
- chaque propriété possède un nom et une valeur ;
- les noms de propriétés sont écrits entre guillemets doubles ;
- les tableaux utilisent les crochets `[]` ;
- les objets peuvent contenir d'autres objets ou tableaux.

### JSON n'est pas une base de données

JSON est une représentation destinée à l'échange ou au stockage de données dans certains contextes. Il ne remplace pas automatiquement une base relationnelle.

Le backend peut lire des lignes dans MariaDB, construire une représentation JSON et l'envoyer au client. Le client reçoit cette représentation, mais n'a pas besoin de connaître la structure interne de la base de données.

### Erreurs JSON fréquentes

JSON valide :

```json
{
  "name": "Service",
  "published": false
}
```

JSON invalide :

```json
{
  'name': 'Service',
  "published": False,
}
```

Pourquoi est-il invalide ?
- JSON exige des guillemets doubles pour les chaînes et les noms de propriétés ;
- les booléens sont `true` et `false` en minuscules ;
- une virgule finale après le dernier élément n'est pas autorisée.

## 1.8. Les idées à retenir avant de coder

Une API est un contrat d'échange entre logiciels. Une API web utilise les technologies Web pour rendre des opérations accessibles. REST organise les échanges autour de ressources et de contraintes architecturales. SOAP et GraphQL répondent à d'autres modèles de communication.

Dans une application mobile, le backend reste responsable de la validation, des permissions et de la logique métier. Le JSON permet de représenter les données échangées, tandis que HTTP transporte les requêtes et les réponses.

Ces notions se complètent :
- HTTP définit le mécanisme de communication ;
- REST propose une manière d'organiser une API ;
- JSON représente souvent les données ;
- Laravel nous aidera à implémenter le backend.

## Exercices d'application

### Exercice 1 — Identifier les composants

Une application permet à un client de consulter des services, de créer un compte et d'envoyer une demande.

Identifiez :
1. le client ;
2. le backend ;
3. l'API ;
4. la base de données ;
5. un exemple de donnée qui pourrait être retournée en JSON.

### Exercice 2 — Choisir une technologie

Pour chaque situation, indiquez une technologie qui pourrait convenir et justifiez brièvement :
1. une API web simple pour une application Flutter ;
2. un système d'entreprise utilisant des contrats SOAP existants ;
3. une interface qui souhaite sélectionner précisément les champs retournés.

### Exercice 3 — Lire le JSON

Dans l'objet suivant, identifiez le nom, la ville, le statut et les compétences :

```json
{
  "name": "Amina",
  "city": "Goma",
  "active": true,
  "skills": ["design", "web"]
}
```

### Corrigé commenté

**Exercice 1 :** le client est l'application mobile ou web ; le backend est le serveur Laravel ; l'API est l'interface d'accès exposée par ce backend ; la base conserve les données. Une réponse JSON pourrait contenir l'identifiant et le titre d'un service.

**Exercice 2 :** REST est souvent adapté à une API web consommée par Flutter ; SOAP peut convenir lorsqu'un système d'entreprise impose déjà ce protocole ; GraphQL peut convenir lorsqu'un client doit sélectionner précisément les champs, à condition de maîtriser son schéma et ses règles de sécurité.

**Exercice 3 :** le nom est Amina, la ville est Goma, le statut est `true`, et les compétences sont les deux éléments du tableau `skills`.

## Résumé du chapitre

- Une API permet à des logiciels de communiquer selon des règles définies.
- Une API web est accessible par les technologies du Web.
- REST, SOAP et GraphQL sont des approches différentes.
- Le client mobile communique avec le backend, pas directement avec la base de données de production.
- Le serveur doit valider les demandes et vérifier les autorisations.
- JSON est un format de représentation de données.
- HTTP transportera les requêtes et les réponses que nous étudierons au chapitre suivant.

**Préparation au chapitre 2 :** nous savons maintenant pourquoi une application utilise une API. Nous allons apprendre comment une requête et une réponse HTTP sont structurées, comment choisir les méthodes et les statuts, et comment concevoir des endpoints cohérents.

## Références de lecture

- MDN Web Docs — HTTP : https://developer.mozilla.org/fr/docs/Web/HTTP
- MDN Web Docs — JSON : https://developer.mozilla.org/fr/docs/Learn_web_development/Core/Scripting/JSON
- Fielding, R. — *Architectural Styles and the Design of Network-based Software Architectures*, dissertation, 2000 : https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm
- GraphQL — Documentation officielle : https://graphql.org/learn/
- W3C — SOAP 1.2 : https://www.w3.org/TR/soap12/

*Note éditoriale : vérifier les liens et harmoniser les références lors de la relecture finale du livre.*
