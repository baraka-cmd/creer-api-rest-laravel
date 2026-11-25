# Chapitre 20 — Tester son API avec Postman

## 20.1. Pourquoi tester une API manuellement ?

Une API peut fonctionner depuis le navigateur ou depuis l’écran de votre application et pourtant échouer dans d’autres situations. Une route peut retourner le bon résultat pour un utilisateur connecté, mais divulguer les données d’un autre utilisateur. Une création peut réussir, mais accepter un champ interdit. Une erreur peut être renvoyée avec un code HTTP incorrect.

Postman permet d’envoyer des requêtes HTTP indépendamment de l’interface web ou mobile. Il devient un outil d’exploration, de vérification et de communication entre développeurs.

Cependant, un test manuel ne remplace pas les tests automatisés. Il est particulièrement utile pendant la conception, le diagnostic et la préparation d’une collection de scénarios qui sera ensuite automatisée.

## 20.2. Comprendre les éléments d’une requête Postman

Une requête contient plusieurs éléments :

- **Méthode** : GET, POST, PUT, PATCH ou DELETE.
- **URL** : adresse de l’endpoint, par exemple `http://127.0.0.1:8000/api/services`.
- **Params** : paramètres de requête pour la recherche, les filtres ou la pagination.
- **Headers** : métadonnées, notamment `Accept: application/json` et `Content-Type: application/json`.
- **Authorization** : mécanisme d’authentification, par exemple Bearer Token.
- **Body** : données envoyées au serveur, généralement JSON pour une API REST.

Pour une API Laravel, indiquez souvent `Accept: application/json`. Cela aide Laravel à comprendre que le client attend une réponse JSON, notamment lors d’une erreur de validation.

## 20.3. Organiser une collection professionnelle

Une collection n’est pas un simple ensemble de requêtes sans ordre. Elle doit représenter le parcours fonctionnel de l’application.

Pour ProxiServices, une organisation possible est :

1. Santé et informations publiques
2. Authentification
3. Profil utilisateur
4. Catégories et catalogue
5. Services professionnels
6. Demandes de prestation
7. Devis et négociation
8. Commandes et suivi
9. Documents et téléchargements
10. Administration
11. Scénarios d’erreurs et de sécurité

Nommez les requêtes selon leur intention : « Créer une demande valide » est plus clair que « POST request 3 ».

Séparez les scénarios positifs — l’action doit réussir — des scénarios négatifs — l’action doit être refusée ou validée comme invalide.

## 20.4. Utiliser les variables d’environnement

Une collection peut être utilisée sur plusieurs environnements : local, test, préproduction et production. Les variables évitent de modifier chaque URL manuellement.

Exemple de variables :

```
base_url = http://127.0.0.1:8000
client_token = <jeton-client-de-test>
professional_token = <jeton-professionnel-de-test>
admin_token = <jeton-administrateur-de-test>
service_id = 12
request_id = 35
```

Une URL devient :

```
{{base_url}}/api/services/{{service_id}}
```

Ne partagez pas une collection contenant de vrais jetons, mots de passe ou secrets de production. Utilisez des comptes de test et des valeurs fictives. Les jetons doivent être stockés avec prudence dans les variables sensibles de l’environnement Postman.

## 20.5. Tester l’authentification

Après la connexion, récupérez le jeton retourné par l’API et utilisez-le dans les requêtes protégées :

```http
Authorization: Bearer {{client_token}}
Accept: application/json
```

Vérifiez plusieurs situations :
- jeton valide ;
- absence de jeton ;
- jeton invalide ;
- jeton révoqué ;
- compte désactivé ;
- utilisateur authentifié mais sans permission.

La réussite du scénario « jeton valide » ne prouve pas que le système d’autorisation est correct. Il faut également tester les refus.

## 20.6. Vérifier les réponses et les codes HTTP

Pour chaque requête, contrôlez :
- le code HTTP ;
- la structure JSON ;
- les champs obligatoires ;
- les types de données ;
- la présence ou l’absence des champs sensibles ;
- les effets réellement enregistrés en base.

Exemple de réponse de création :

```json
{
  "data": {
    "id": 35,
    "status": "pending",
    "description": "Installer deux prises"
  }
}
```

Le code attendu pourrait être 201 Created si une nouvelle ressource a été créée. Le contrat exact doit être défini et documenté par votre API.

Ne vous contentez pas de regarder la réponse : vérifiez également qu’une demande a été créée une seule fois, avec le bon propriétaire et le bon état initial.

## 20.7. Tester les erreurs volontairement

Une collection professionnelle comprend des cas invalides :
- champ obligatoire absent ;
- identifiant inexistant ;
- format de date invalide ;
- ressource appartenant à un autre utilisateur ;
- action interdite selon le rôle ;
- fichier trop volumineux ;
- pagination supérieure au maximum ;
- requête répétée dépassant la limite.

Le but n’est pas de provoquer des erreurs pour le plaisir, mais de vérifier que l’API échoue de manière contrôlée et prévisible.

## 20.8. Ajouter des tests dans Postman

Postman permet d’écrire des scripts de vérification après la réponse. Un test peut vérifier que le statut est 200, que le corps contient une clé attendue ou que le temps de réponse reste dans une limite indicative.

Exemple de test simple :

```javascript
pm.test("La réponse est HTTP 200", function () {
    pm.response.to.have.status(200);
});

pm.test("La réponse contient les données", function () {
    const body = pm.response.json();
    pm.expect(body).to.have.property("data");
});
```

Adaptez le statut et la structure au contrat de l’endpoint. Un test ne doit pas être copié partout avec des attentes identiques : une création, une suppression et une erreur de validation ont des résultats différents.

## 20.9. Atelier : construire un parcours complet

Préparez une collection qui :
1. se connecte avec un client de test ;
2. récupère son profil ;
3. consulte le catalogue ;
4. crée une demande valide ;
5. consulte la demande créée ;
6. tente de modifier une demande appartenant à un autre compte ;
7. envoie une donnée invalide ;
8. vérifie que les réponses et les codes HTTP correspondent au contrat.

Le parcours doit être reproductible. Utilisez une base de test et des données contrôlées, afin que les essais ne dépendent pas d’un catalogue qui change chaque jour.

## À retenir

Postman aide à explorer et vérifier le contrat HTTP. Une collection bien organisée sert de documentation exécutable et de support de diagnostic. Les scénarios négatifs sont aussi importants que les scénarios de réussite.