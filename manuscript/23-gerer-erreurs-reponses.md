# Chapitre 23 — Gérer les erreurs et les réponses

## 23.1. Pourquoi une réponse cohérente est essentielle

Une API ne communique pas seulement lorsqu’une opération réussit. Elle doit aussi expliquer les refus, les données invalides, les ressources absentes et les erreurs internes.

Si chaque endpoint renvoie une structure différente, l’application web ou mobile doit multiplier les conditions. Une réponse cohérente rend le client plus simple, les tests plus fiables et le diagnostic plus rapide.

Définissez un contrat commun, puis adaptez-le aux besoins réels du projet. La cohérence ne signifie pas que toutes les réponses doivent être identiques : une liste paginée, une ressource créée et une erreur ont des structures différentes, mais elles suivent des conventions stables.

## 23.2. Choisir les codes HTTP appropriés

Les codes HTTP décrivent le résultat de la requête :

- **200 OK** : la requête a réussi et une réponse est fournie.
- **201 Created** : une nouvelle ressource a été créée.
- **202 Accepted** : le traitement a été accepté, mais peut être exécuté plus tard.
- **204 No Content** : la requête a réussi sans corps de réponse.
- **400 Bad Request** : la requête est mal formée ou ne peut être comprise.
- **401 Unauthorized** : l’authentification est absente ou invalide.
- **403 Forbidden** : l’identité est connue, mais l’action est interdite.
- **404 Not Found** : la ressource n’est pas trouvée, ou son existence est masquée selon la politique.
- **409 Conflict** : conflit avec l’état actuel de la ressource, par exemple une transition incompatible.
- **422 Unprocessable Content** : les données sont comprises, mais ne respectent pas les règles de validation.
- **429 Too Many Requests** : la limite de requêtes a été dépassée.
- **500 Internal Server Error** : erreur inattendue côté serveur.

Le code doit correspondre au comportement réel. Ne renvoyez pas 200 lorsqu’une opération a échoué, et ne renvoyez pas 500 pour une erreur utilisateur prévisible.

## 23.3. Définir une structure de réponse

Une réponse de succès peut utiliser une enveloppe `data` :

```json
{
  "data": {
    "id": 12,
    "title": "Installation électrique"
  }
}
```

Une liste paginée peut inclure les éléments et les informations de pagination selon le contrat retenu.

Pour les erreurs, une structure commune peut être :

```json
{
  "message": "Les données envoyées ne sont pas valides.",
  "code": "VALIDATION_FAILED",
  "errors": {
    "title": [
      "Le titre est obligatoire."
    ]
  }
}
```

Le champ `code` est une valeur stable destinée au client. Le texte `message` peut être traduit ou reformulé sans obliger l’application à analyser une phrase. Les erreurs détaillées doivent rester limitées aux informations utiles et sûres.

## 23.4. Erreurs de validation

Laravel gère les erreurs de validation des Form Requests. Pour une requête JSON, l’application reçoit habituellement un statut 422 et une liste d’erreurs par champ.

Le client doit pouvoir :
- identifier le champ concerné ;
- afficher une indication compréhensible ;
- conserver les autres valeurs déjà saisies ;
- éviter de traiter l’erreur comme une panne serveur.

Les messages doivent être précis sans exposer de logique interne. Ne transformez pas toutes les erreurs de validation en message vague si le client a besoin de corriger un formulaire.

## 23.5. Exceptions et frontière HTTP

Une exception représente un événement qui empêche le déroulement normal du code. Laravel convertit de nombreuses exceptions en réponses HTTP selon sa configuration.

Il est préférable de laisser le framework traiter les erreurs standards lorsqu’il le fait correctement, plutôt que d’entourer chaque méthode d’un `try/catch` qui masque le problème.

Un contrôleur ne devrait pas capturer toutes les exceptions et retourner systématiquement :

```php
return response()->json([
    'message' => 'Erreur',
], 200);
```

Ce comportement détruit la signification des codes HTTP et rend le diagnostic difficile.

Capturez une exception lorsque vous pouvez réellement la traiter : réessayer une opération, convertir une erreur métier connue, libérer une ressource ou fournir une réponse spécifique. Les erreurs inattendues doivent être remontées au mécanisme de gestion central.

## 23.6. Distinguer erreur métier et erreur technique

Une erreur métier correspond à une règle connue : devis expiré, transition interdite, service indisponible ou opération déjà effectuée.

Une erreur technique peut provenir d’une panne de base de données, d’un stockage inaccessible ou d’un service externe.

Le client doit recevoir une réponse adaptée à la catégorie. Il n’a pas besoin de connaître le nom d’une table SQL ou le chemin d’un fichier sur le serveur.

Pour une erreur métier, un code stable peut être utile :

```json
{
  "message": "Ce devis ne peut plus être accepté.",
  "code": "QUOTATION_NOT_ACCEPTABLE"
}
```

Ce code doit être défini dans le contrat et utilisé de manière cohérente.

## 23.7. Journaliser sans divulguer

Les journaux permettent de comprendre ce qui s’est produit après une erreur. Ils doivent fournir un contexte utile :
- date et heure ;
- identifiant de requête ou de corrélation ;
- endpoint et méthode ;
- utilisateur concerné lorsque c’est approprié ;
- catégorie d’erreur ;
- résultat et durée ;
- exception technique contrôlée.

Ne journalisez pas les mots de passe, les jetons d’accès, les clés secrètes, les données bancaires complètes ou le contenu de documents privés. Les logs sont eux-mêmes des données sensibles et doivent avoir des droits d’accès, une durée de conservation et une politique de rotation.

Une réponse HTTP peut inclure un identifiant de requête pour permettre au support de retrouver la trace correspondante sans demander au client de transmettre des secrets.

## 23.8. Environnement de production : ne pas exposer le débogage

Le mode debug est utile en développement, mais dangereux en production. Il peut révéler des traces, des chemins, des requêtes et des détails de configuration.

En production :
- désactivez le debug ;
- configurez la journalisation ;
- vérifiez que les exceptions ne renvoient pas de détails internes ;
- testez les erreurs courantes ;
- surveillez les erreurs répétées ;
- prévoyez une procédure d’incident.

Une page ou une réponse générique ne signifie pas que l’erreur doit être ignorée : les détails sont conservés dans un système de logs sécurisé.

## 23.9. Atelier : harmoniser trois endpoints

Prenez une route de lecture, une route de création et une route de modification. Pour chacune, définissez :
- la réponse de réussite ;
- les erreurs de validation ;
- l’absence d’authentification ;
- l’accès interdit ;
- la ressource absente ;
- le conflit métier ;
- l’erreur interne.

Ajoutez les tests correspondants et vérifiez que l’application Flutter ou web peut interpréter les réponses sans logique spéciale inutile.

## À retenir

Une bonne gestion des erreurs rend l’API prévisible et maintenable. Les codes HTTP doivent être justes, les structures stables, les messages utiles et les logs suffisamment détaillés pour diagnostiquer sans exposer les secrets.