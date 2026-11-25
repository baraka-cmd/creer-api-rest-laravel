# Chapitre 3 — Concevoir une API avant de la développer

> Partie I — Comprendre et concevoir une API REST  
> Objectif : transformer un besoin métier en un contrat d'API compréhensible, cohérent, testable et sécurisé.

## Introduction : pourquoi réfléchir avant de coder ?

Un développeur peut être tenté de créer immédiatement des routes et des contrôleurs lorsqu'on lui présente une idée d'application. Cette méthode semble rapide, mais elle provoque souvent des décisions contradictoires : des URLs qui changent d'une fonctionnalité à l'autre, des champs inutiles, des permissions oubliées et des réponses difficiles à exploiter par le frontend.

Concevoir une API n'est pas une étape administrative. C'est le travail qui permet de répondre, avant l'écriture du code, à quatre questions fondamentales : **qui utilise l'API, pour accomplir quelle action, sur quelle ressource et selon quelles règles ?**

Dans notre projet de mise en relation entre clients et prestataires, la phrase « Je veux que les clients trouvent un professionnel et demandent une prestation » est un point de départ. Elle n'est pas encore un contrat technique. Il faut déterminer ce qu'un visiteur peut consulter, ce qu'un professionnel peut publier, comment une demande est créée et qui peut la voir.

## 3.1. Partir des besoins métier

Un besoin métier décrit un résultat utile pour une personne ou une organisation. Il doit être formulé du point de vue de l'utilisateur, sans imposer prématurément une solution technique.

Exemples :
- Un visiteur veut consulter les services publiés.
- Un professionnel veut publier et modifier ses propres services.
- Un client connecté veut envoyer une demande.
- Un professionnel veut consulter les demandes qui lui sont adressées.

Ces formulations sont plus utiles que « créer la table services », car elles expliquent pourquoi la fonctionnalité existe.

### Construire un scénario utilisateur

Un scénario décrit les étapes d'une action et son résultat attendu.

**Scénario : rechercher un service à Goma**

1. Le visiteur ouvre l'écran de recherche.
2. Il sélectionne la ville Goma et une catégorie.
3. L'application demande les résultats à l'API.
4. Le serveur valide les paramètres et applique les filtres.
5. Le serveur retourne une liste de services publiés, visibles et paginés.
6. L'application affiche les résultats.

Ce scénario révèle déjà plusieurs besoins : consultation publique, filtres, pagination, règles de visibilité et représentation JSON.

### Règle métier ou choix technique ?

Règle métier : « Un professionnel ne peut modifier que les services qu'il possède. »

Choix technique : « Laravel utilisera une Policy pour vérifier cette permission. »

La règle métier reste valable même si l'on change de framework. La solution technique vient ensuite.

## 3.2. Identifier les acteurs et leurs permissions

Un acteur est une personne ou un système qui interagit avec l'application.

| Acteur | Besoins principaux |
|---|---|
| Visiteur | Consulter les services publics |
| Client connecté | Gérer son profil et envoyer ses demandes |
| Professionnel | Gérer son profil professionnel et ses services |
| Administrateur | Modérer et administrer selon ses permissions |

Ne confondez jamais authentification et autorisation. L'authentification répond à « qui est cette personne ? ». L'autorisation répond à « peut-elle effectuer cette action sur cette ressource ? ».

Deux professionnels peuvent être connectés, mais chacun doit gérer ses propres services. Connaître l'identifiant d'un service ne donne pas le droit de le modifier. Cette règle doit être prévue dès la conception et appliquée côté serveur.

## 3.3. Construire le modèle métier

Le modèle métier représente les concepts importants du domaine et leurs relations. Pour notre plateforme, les concepts principaux sont :

- **Utilisateur** : personne qui possède un compte.
- **Profil professionnel** : informations professionnelles associées à un utilisateur.
- **Catégorie** : famille de services.
- **Service** : prestation proposée.
- **Demande** : sollicitation envoyée par un client.

Relations simplifiées :

    Utilisateur 1 ── 0..1 Profil professionnel
    Profil professionnel 1 ── N Services
    Catégorie 1 ── N Services
    Client 1 ── N Demandes
    Service 1 ── N Demandes

Le symbole 1 signifie « un », 0..1 signifie « zéro ou un », et N signifie « plusieurs ». Ce diagramme exprime le domaine ; la conception détaillée des tables et clés étrangères sera traitée au chapitre 7.

### Une ressource API n'est pas nécessairement une table

Une erreur fréquente consiste à exposer chaque table de la base comme endpoint. Pourtant, le modèle interne et le contrat public ont des objectifs différents.

Par exemple, la base peut stocker les images dans une table technique. Le client peut simplement recevoir une liste d'images dans la représentation d'un service. L'API doit représenter les besoins des clients, pas révéler automatiquement les détails de stockage.

## 3.4. Choisir les ressources

Une ressource est un élément du domaine que l'API permet d'identifier et de manipuler.

Ressources possibles :
- services ;
- catégories ;
- professionnels ;
- demandes ;
- profils.

Pour décider si un concept mérite une ressource, demandez-vous :
1. Le client doit-il le consulter ou le manipuler ?
2. Possède-t-il une identité ou un cycle de vie utile ?
3. L'exposer rend-il le contrat plus clair ?

Ne créez pas une ressource uniquement parce qu'une table existe. Ne fusionnez pas non plus des concepts dont les règles et permissions sont différentes.

## 3.5. Concevoir les endpoints

Un endpoint est couramment compris comme une méthode HTTP associée à un chemin d'accès.

| Besoin | Méthode | Endpoint |
|---|---|---|
| Lister les services | GET | /api/services |
| Voir un service | GET | /api/services/{id} |
| Créer un service | POST | /api/services |
| Modifier un service | PATCH | /api/services/{id} |
| Supprimer un service | DELETE | /api/services/{id} |

Privilégiez les noms de ressources et laissez la méthode HTTP exprimer l'action.

Moins cohérent :
- GET /api/getServices
- POST /api/createService
- GET /api/deleteService/42

Plus cohérent :
- GET /api/services
- POST /api/services
- DELETE /api/services/42

Utiliser GET pour supprimer est incorrect : une méthode de consultation ne doit pas déclencher une modification métier.

### Les opérations métier particulières

Toutes les opérations ne sont pas un CRUD classique. Accepter une demande ou publier un service représente une transition d'état. Il peut être pertinent de définir une opération explicite, par exemple POST /api/requests/73/acceptance, ou une ressource de décision.

L'important est que le contrat explique clairement le comportement, les permissions, les effets et les erreurs.

## 3.6. Définir les données d'entrée

Pour chaque opération, identifiez les champs obligatoires, facultatifs, calculés par le serveur et interdits au client.

Exemple : création d'un service.

| Champ | Fournisseur | Règle |
|---|---|---|
| title | Professionnel | Obligatoire, longueur limitée |
| description | Professionnel | Texte validé |
| category_id | Professionnel | Catégorie existante et autorisée |
| price | Professionnel | Montant valide et devise définie |
| professional_id | Serveur | Déduit du compte connecté |
| status | Serveur | Déterminé par le processus métier |
| created_at | Serveur | Date gérée par l'application |

Si le client peut envoyer professional_id, il peut tenter de créer un service au nom d'un autre compte. Si le client peut envoyer role=admin, il peut tenter de modifier son privilège. Le serveur doit déterminer les champs sensibles.

## 3.7. Définir les réponses

Un contrat d'API décrit aussi la structure des réponses.

Exemple pédagogique :

    {
      "data": {
        "id": 42,
        "title": "Installation réseau",
        "category": {
          "id": 3,
          "name": "Informatique"
        },
        "price": {
          "amount": "40.00",
          "currency": "USD"
        }
      }
    }

Ce format doit être précisé dans le contrat : devise, champs facultatifs, visibilité et représentation des montants.

Ne retournez pas automatiquement toutes les colonnes de la base. Elles peuvent contenir des informations internes, privées ou administratives. Le backend doit construire une représentation contrôlée.

## 3.8. Documenter les erreurs

Pour la création d'un service, les erreurs prévisibles peuvent être :
- 401 : authentification absente ou invalide ;
- 403 : action interdite ;
- 404 : ressource ou catégorie introuvable ;
- 422 : données invalides ;
- 409 : conflit avec l'état métier, lorsque ce statut correspond au contrat.

Exemple de réponse de validation :

    {
      "message": "Les données fournies sont invalides.",
      "errors": {
        "title": ["Le titre est obligatoire."]
      }
    }

Le client doit comprendre l'erreur sans recevoir une trace technique, une requête SQL ou des secrets.

## 3.9. Rédiger une fiche de contrat

Pour chaque endpoint, consignez :
- son objectif ;
- la méthode et le chemin ;
- l'authentification ;
- la permission ;
- les paramètres ;
- le body attendu ;
- les règles de validation ;
- la réponse de succès ;
- les erreurs possibles ;
- un exemple de requête.

**Fiche : consulter les services**

Objectif : afficher les services publics.

- Méthode : GET
- Chemin : /api/services
- Authentification : non requise pour cette consultation publique
- Paramètres : category_id, city, search, page
- Réponse : liste paginée des services publiés et visibles
- Erreurs : paramètres invalides, limite de requêtes dépassée

Exemple :

    GET /api/services?city=Goma&page=1
    Accept: application/json

Le contrat précise que page est un entier positif, que le serveur limite la taille des pages et que seuls les services autorisés à être publics sont retournés.

**Fiche : créer un service**

- Méthode : POST
- Chemin : /api/services
- Authentification : requise
- Permission : professionnel actif et autorisé
- Body : titre, description, catégorie et champs métier autorisés
- Propriétaire : déterminé côté serveur
- Succès : 201 Created
- Erreurs : 401, 403, 422

Une telle fiche permet au développeur mobile de préparer son formulaire et ses messages d'erreur sans connaître les tables Laravel.

## 3.10. Prévoir l'évolution

Une API peut être utilisée par plusieurs versions d'une application mobile. Tous les utilisateurs ne mettent pas leur application à jour simultanément.

Avant de supprimer ou renommer un champ, vérifiez si des clients l'utilisent. Ajouter un champ facultatif est généralement moins risqué que retirer un champ existant, mais tout changement doit être évalué et documenté.

La compatibilité du contrat est une responsabilité de conception, pas seulement une question de programmation.

## Atelier guidé : catalogue de services

1. **Besoin :** un visiteur veut découvrir les services disponibles dans sa ville.
2. **Scénario :** il choisit la ville, consulte les résultats et ouvre un détail.
3. **Ressources :** Service, Catégorie, Professionnel.
4. **Endpoints :** GET /api/services, GET /api/services/{id}, GET /api/categories.
5. **Paramètres :** city, category_id, page.
6. **Règles :** services publiés seulement, paramètres validés, résultats paginés, champs privés exclus.
7. **Vérification :** le frontend doit pouvoir comprendre le contrat sans connaître les tables internes.

## Exercices

### Exercice 1
Une école veut permettre aux enseignants de consulter leurs classes et d'enregistrer les notes. Identifiez les acteurs, trois besoins, les ressources et une permission indispensable.

### Exercice 2
Pour une bibliothèque, proposez les endpoints de consultation des livres, détail, création réservée au bibliothécaire, modification et consultation des emprunts de l'utilisateur connecté.

### Exercice 3
Pour une demande de prestation, le client envoie le service et un message. Qui détermine le client, le statut initial et la date ? Justifiez.

### Exercice 4
Rédigez une fiche de contrat pour une opération de votre projet : méthode, chemin, accès, paramètres, body, réponse et erreurs.

## Corrigés commentés

**Exercice 1 :** acteurs possibles : enseignant, responsable et administrateur. Besoins : consulter les classes attribuées, voir les élèves, enregistrer les notes. Ressources : classes, élèves, évaluations/notes. Permission : un enseignant ne peut agir que sur les classes et matières qui lui sont affectées.

**Exercice 2 :** GET /api/books ; GET /api/books/{id} ; POST /api/books ; PATCH /api/books/{id} ; GET /api/me/loans. Les écritures exigent le rôle ou la permission de bibliothécaire. Les emprunts personnels sont déterminés à partir de l'utilisateur authentifié.

**Exercice 3 :** le client fournit le service et le message. Le serveur déduit l'utilisateur connecté, fixe le statut et crée la date. Cela empêche l'usurpation d'identité et la manipulation du workflow.

**Exercice 4 :** les réponses varient. Le critère est la précision : un autre développeur doit pouvoir construire le client sans deviner.

## Synthèse

Concevoir une API, c'est transformer des scénarios métier en contrat technique. Il faut identifier les acteurs et leurs permissions, comprendre les ressources et leurs relations, choisir les endpoints, définir les entrées, sorties et erreurs, puis anticiper la compatibilité.

Le contrat ne doit pas copier aveuglément la base de données. Il doit être conçu pour les clients et sécurisé dès le départ.

**Transition :** au chapitre 4, nous préparons l'environnement nécessaire pour transformer ce contrat en application Laravel.

## Références

- Microsoft Learn — API design best practices : https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design
- Microsoft API Guidelines : https://github.com/microsoft/api-guidelines
- OpenAPI Specification : https://spec.openapis.org/oas/latest.html
- OWASP API Security : https://owasp.org/www-project-api-security/
