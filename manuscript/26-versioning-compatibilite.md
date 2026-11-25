# Chapitre 26 — Préparer l'évolution de l'API : versionner sans casser les clients

## 26.1. Une API est un contrat avec ses consommateurs

Une API peut être utilisée par une application Flutter, un site JavaScript, une application Android, un partenaire ou un autre service backend. Ces consommateurs ne sont pas toujours mis à jour au même moment.

Lorsque le serveur change, les anciens clients peuvent continuer à fonctionner pendant des semaines ou des mois. Une modification qui paraît simple dans Laravel peut donc casser une application déjà installée sur les téléphones.

Le contrat d'API comprend notamment :
- les URL et les ressources ;
- les méthodes HTTP ;
- les paramètres et leur validation ;
- la structure JSON ;
- les statuts HTTP ;
- les règles d'authentification et d'autorisation ;
- la pagination et le tri ;
- les comportements métier documentés.

Le versionnement aide à organiser les changements importants. Il ne remplace pas la discipline de compatibilité.

## 26.2. Pourquoi versionner ?

Imaginons une première version qui retourne :

~~~json
{
  "data": {
    "id": 42,
    "title": "Réparation d'un ordinateur",
    "price": 25
  }
}
~~~

Plus tard, l'équipe décide que price doit être un objet contenant la devise :

~~~json
{
  "data": {
    "id": 42,
    "title": "Réparation d'un ordinateur",
    "price": {
      "amount": "25.00",
      "currency": "USD"
    }
  }
}
~~~

Cette évolution peut être meilleure pour le futur, mais un client qui attend un nombre et effectue une opération mathématique risque de planter.

Le versionnement permet de préserver un contrat existant pendant la mise en place d'un nouveau contrat.

## 26.3. Version dans l'URL : une stratégie claire

Une convention courante consiste à placer la version dans le chemin :

~~~text
/api/v1/services
/api/v2/services
~~~

Avantages :
- la version est visible dans les routes et les journaux ;
- les tests et la documentation peuvent être organisés par version ;
- les clients choisissent explicitement le contrat utilisé ;
- le routage Laravel reste facile à comprendre.

Exemple d'organisation :

~~~php
Route::prefix('v1')
    ->group(function () {
        Route::get('/services', [V1\ServiceController::class, 'index']);
    });
~~~

Puis, lorsque cela devient nécessaire :

~~~php
Route::prefix('v2')
    ->group(function () {
        Route::get('/services', [V2\ServiceController::class, 'index']);
    });
~~~

Les namespaces peuvent aider à isoler les présentations ou comportements qui diffèrent réellement. Il n'est pas obligatoire de copier toute l'application à chaque version. Les règles métier communes peuvent rester partagées tant qu'elles conservent le même sens.

## 26.4. Versionnement par en-tête : une autre possibilité

Certaines API choisissent un en-tête de version ou un type de média spécifique. Cette approche peut éviter de multiplier les chemins, mais elle est moins visible pour les débutants et peut compliquer les tests, le cache et le diagnostic si elle n'est pas documentée clairement.

Il n'existe pas une méthode universellement supérieure. Pour un projet pédagogique ou une API destinée à plusieurs clients, la version dans l'URL est souvent plus facile à expliquer, à tester et à maintenir.

Choisissez une stratégie, documentez-la et évitez de mélanger plusieurs mécanismes sans raison.

## 26.5. Changements compatibles et changements cassants

Un changement est généralement compatible lorsque les clients existants peuvent continuer à fonctionner sans modification.

Exemples souvent compatibles :
- ajouter un champ facultatif dans une réponse, si les clients tolèrent les champs inconnus ;
- ajouter un nouvel endpoint ;
- ajouter une valeur facultative à un filtre ;
- améliorer les messages d'erreur sans modifier leur structure contractuelle.

Exemples potentiellement cassants :
- supprimer ou renommer un champ ;
- modifier le type d'une valeur ;
- rendre obligatoire un paramètre auparavant facultatif ;
- changer la signification d'un statut ;
- modifier l'ordre ou la structure d'une collection ;
- remplacer une réponse JSON par une autre enveloppe ;
- modifier les permissions de manière inattendue ;
- changer un comportement métier dont les clients dépendent.

Le contexte compte. Certains clients échouent lorsqu'un champ inattendu apparaît, notamment s'ils utilisent une désérialisation stricte. Documentez donc les conventions de tolérance des clients.

## 26.6. Concevoir des évolutions progressives

Avant de modifier un contrat, suivez une démarche :

1. identifier les clients et les versions actuellement utilisées ;
2. vérifier les usages du champ ou de l'endpoint ;
3. définir la nouvelle représentation ;
4. décider si le changement est compatible ;
5. ajouter des tests de contrat ;
6. mettre à jour la documentation ;
7. annoncer le changement et la période de transition ;
8. surveiller l'utilisation de l'ancienne version ;
9. retirer l'ancien contrat seulement lorsque les conditions sont réunies.

Ne supprimez pas une version simplement parce que la nouvelle est disponible. Les applications mobiles ne sont pas toutes mises à jour instantanément.

## 26.7. Gérer les fonctionnalités sans multiplier les versions

Toutes les nouvelles fonctionnalités ne nécessitent pas une version majeure. Un endpoint additionnel ou un champ facultatif peut souvent être ajouté dans la version existante.

Pour une fonctionnalité activée progressivement, l'application peut utiliser un mécanisme de feature flag. Celui-ci permet d'activer ou désactiver une capacité selon un environnement, un groupe d'utilisateurs ou une phase de déploiement.

Un feature flag doit être contrôlé côté serveur. Le fait de masquer un bouton dans Flutter ne constitue pas une protection : le serveur doit toujours vérifier si l'opération est autorisée et disponible.

Évitez de laisser des flags temporaires indéfiniment. Chaque flag ajoute des branches de comportement et doit avoir un responsable, une condition de retrait et des tests.

## 26.8. Maintenir une architecture évolutive

Une API maintenable repose sur plusieurs pratiques complémentaires :

- des ressources et endpoints nommés de façon cohérente ;
- un contrat documenté ;
- une validation explicite ;
- des règles d'autorisation centralisées ;
- des réponses JSON stables ;
- des tests de régression ;
- des migrations de base de données réfléchies ;
- une séparation raisonnable entre HTTP et logique métier ;
- une politique de dépréciation connue.

L'objectif n'est pas de prévoir toutes les évolutions futures. Il est de rendre les changements probables moins dangereux.

## 26.9. Exemple de transition v1 vers v2

Supposons que v1 expose le prix sous forme numérique et que v2 utilise une structure montant/devise.

La démarche professionnelle pourrait être :

- conserver /api/v1/services avec son ancien format ;
- créer /api/v2/services avec la nouvelle Resource ;
- partager les mêmes règles de recherche et d'autorisation si elles n'ont pas changé ;
- écrire des tests séparés pour les deux formats ;
- documenter les deux contrats dans OpenAPI ;
- faire évoluer le client mobile vers v2 ;
- observer les appels à v1 ;
- planifier la fin de support et communiquer avant le retrait.

Il est possible de partager une couche métier tout en présentant deux formats différents. La version API concerne le contrat externe, pas nécessairement toute l'architecture interne.

## 26.10. Atelier : ajouter un champ sans casser l'application mobile

### Situation

L'API v1 retourne le nom du professionnel dans un champ professional_name. L'équipe souhaite ajouter un objet professional avec une photo et une note.

### Questions

- Peut-on ajouter professional tout en conservant professional_name ?
- Le champ photo peut-il être absent ou nul ?
- La Resource doit-elle exposer la photo privée ou uniquement une URL publique autorisée ?
- Le client Flutter ignore-t-il les champs inconnus ?
- Les tests vérifient-ils encore l'ancien contrat ?

### Travail

1. ajouter le champ sans supprimer l'ancien ;
2. mettre à jour OpenAPI ;
3. écrire un test de réponse v1 ;
4. adapter le client ;
5. mesurer si l'ancien champ est encore utilisé ;
6. décider d'une dépréciation documentée.

## 26.11. Checklist avant une évolution d'API

- Le changement modifie-t-il le contrat public ?
- Les anciens clients peuvent-ils continuer à fonctionner ?
- Le type et le sens des champs restent-ils stables ?
- Les paramètres obligatoires ont-ils changé ?
- Les permissions ou états métier sont-ils affectés ?
- Les tests de contrat et de régression sont-ils présents ?
- La documentation est-elle à jour ?
- La stratégie de transition est-elle claire ?
- Les versions en circulation sont-elles observables ?
- Le retrait éventuel est-il communiqué et planifié ?

**À retenir :** une API évolue avec ses clients. Une bonne version ne sert pas à multiplier les numéros, mais à protéger les consommateurs lorsqu'un changement réellement incompatible devient nécessaire.
