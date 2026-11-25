# Chapitre 22 — Documenter son API

## 22.1. La documentation fait partie du produit

Une API est utilisée par des développeurs qui ne connaissent pas forcément son code interne. Ils ont besoin de savoir quelles routes existent, quelles données envoyer, comment s’authentifier et comment interpréter les réponses.

Sans documentation, les intégrateurs devinent le format des requêtes, copient des exemples incomplets et découvrent les règles par essais et erreurs. Cela augmente les coûts de support et les risques d’incompatibilité.

La documentation doit répondre à des questions concrètes :
- Quelle action réalise cet endpoint ?
- Qui peut l’appeler ?
- Quels paramètres sont obligatoires ?
- Quel format est attendu ?
- Quelle réponse est retournée ?
- Quelles erreurs peuvent survenir ?
- Existe-t-il des effets métier ou des restrictions ?

## 22.2. Décrire un endpoint de façon complète

Une fiche d’endpoint doit inclure :
- méthode et chemin ;
- description de l’action ;
- authentification requise ;
- permissions et conditions d’accès ;
- paramètres de chemin et de requête ;
- corps JSON attendu ;
- règles de validation ;
- exemple de réponse ;
- codes HTTP possibles ;
- erreurs fréquentes ;
- remarques sur la pagination ou les effets de bord.

Exemple de description :

```http
POST /api/requests
```

But : créer une demande de prestation pour un service publié. L’utilisateur doit être connecté avec un compte client actif. Le serveur détermine le client à partir de l’identité authentifiée ; le corps de la requête ne doit pas fournir un propriétaire arbitraire.

Corps indicatif :

```json
{
  "service_id": 12,
  "description": "Installer deux prises",
  "requested_date": "2026-12-03"
}
```

Réponse possible : 201 Created avec la ressource créée. Les champs exacts doivent correspondre au Resource de l’application.

## 22.3. Documenter les paramètres et les erreurs

Pour chaque paramètre, précisez son type, sa présence, ses limites et un exemple. Pour une date, indiquez le format et le fuseau horaire attendu. Pour une pagination, expliquez la valeur par défaut et le maximum.

Documentez également les erreurs :
- 401 : authentification manquante ou invalide ;
- 403 : action interdite ;
- 404 : ressource non trouvée ou volontairement masquée ;
- 422 : données invalides ;
- 429 : limite de requêtes dépassée ;
- 500 : erreur interne générique.

Ces codes sont des conventions à appliquer selon la situation réelle. Ne documentez pas une réponse que le code ne produit pas.

## 22.4. OpenAPI : un contrat lisible par les humains et les outils

OpenAPI est une spécification permettant de décrire une API HTTP dans un format structuré. Elle peut représenter les routes, paramètres, schémas de données, réponses et mécanismes de sécurité.

Un document OpenAPI peut être écrit en YAML ou JSON. Exemple simplifié :

```yaml
openapi: 3.0.3
info:
  title: ProxiServices API
  version: 1.0.0
paths:
  /api/services:
    get:
      summary: Rechercher les services publiés
      responses:
        '200':
          description: Liste des services
```

Cet extrait est volontairement minimal : une documentation exploitable doit compléter les paramètres, les schémas, les réponses, les erreurs et les exigences d’authentification.

## 22.5. Swagger UI et outils équivalents

Swagger UI peut afficher un document OpenAPI sous forme d’interface interactive. Les développeurs peuvent consulter les routes, lire les schémas et, dans un environnement approprié, essayer des requêtes.

L’interface interactive ne doit pas être considérée comme une autorisation supplémentaire. En production, protégez ou désactivez les fonctions de documentation qui exposeraient des opérations internes ou permettraient des essais non souhaités.

D’autres outils peuvent générer ou afficher la documentation OpenAPI. Le choix dépend de la version Laravel, de la maintenance du package, de la compatibilité avec le projet et du mode de publication.

Avant d’installer un package, vérifiez sa compatibilité avec la version de Laravel et PHP du projet, son activité de maintenance et la manière dont il décrit les réponses réelles.

## 22.6. Garder la documentation à jour

La documentation devient dangereuse lorsqu’elle est fausse. Une route peut changer de nom, un champ peut devenir obligatoire ou une réponse peut être modifiée sans mise à jour du document.

Adoptez une règle d’équipe : toute modification du contrat API doit mettre à jour la documentation et les tests associés.

Pratiques recommandées :
- stocker le fichier OpenAPI dans le dépôt ;
- versionner les changements ;
- relire les exemples lors des revues de code ;
- générer la documentation à partir du code lorsque cela est fiable ;
- vérifier régulièrement les différences entre le contrat et les réponses réelles ;
- documenter les changements incompatibles.

La génération automatique n’est pas une garantie : elle peut produire des descriptions techniques incomplètes ou ne pas comprendre les règles métier. Une revue humaine reste nécessaire.

## 22.7. Atelier : documenter une ressource

Choisissez `GET /api/services/{id}` et rédigez :
1. la description de l’action ;
2. le paramètre `id` ;
3. les conditions de visibilité ;
4. un exemple de réponse ;
5. les erreurs possibles ;
6. les champs qui ne doivent jamais être exposés.

Comparez ensuite la fiche avec le contrôleur, la Policy et le Resource. Si les documents ne concordent pas, le contrat doit être clarifié.

## À retenir

La documentation est un contrat entre le serveur et ses clients. Elle doit être précise, versionnée, testée et entretenue. OpenAPI permet de structurer ce contrat et de le rendre exploitable par des outils, mais la qualité dépend toujours de l’exactitude des informations.