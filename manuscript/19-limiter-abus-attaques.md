# Chapitre 19 — Limiter les abus et les attaques

## 19.1. Une API doit rester disponible et maîtriser ses coûts

La sécurité ne concerne pas uniquement la confidentialité. Une API peut être attaquée en envoyant un grand nombre de requêtes, en provoquant des traitements coûteux ou en saturant le stockage.

Les abus peuvent être intentionnels ou accidentels : application mobile qui répète une requête, boucle de synchronisation, bot, script malveillant ou utilisateur qui téléverse trop de fichiers.

Il faut donc définir des limites adaptées aux opérations, surveiller les comportements et répondre de manière prévisible.

## 19.2. Rate limiting : limiter la fréquence

Le *rate limiting* limite le nombre de requêtes autorisées pendant une période donnée. Laravel propose des mécanismes de limitation configurables et des middleware associés.

Les limites ne doivent pas être identiques pour toutes les routes :
- consultation publique du catalogue : limite relativement généreuse ;
- connexion et réinitialisation du mot de passe : limite stricte ;
- envoi de messages ou demandes : limite adaptée à l’activité ;
- génération de rapports lourds : limite basse ou traitement asynchrone ;
- téléchargement : limite selon la taille et la sensibilité.

La clé de limitation peut dépendre de l’adresse IP, de l’utilisateur authentifié ou d’une combinaison. Une limitation uniquement par IP peut pénaliser des utilisateurs partageant une connexion ; une limitation uniquement par compte peut être contournée avec plusieurs comptes. Choisissez selon le risque.

Lorsque la limite est dépassée, renvoyez généralement HTTP 429 (*Too Many Requests*) et, si pertinent, un délai avant la nouvelle tentative. Ne révélez pas des détails internes de votre infrastructure.

## 19.3. Éviter les requêtes coûteuses

Un endpoint de recherche ne doit pas accepter une limite illimitée :

```http
GET /api/services?per_page=100000
```

Fixez une valeur par défaut et un maximum côté serveur. Dans le manuscrit précédent, nous avons retenu un maximum pédagogique de 50 éléments par page ; le chiffre exact doit être ajusté aux performances et au besoin réel du produit.

Autres mesures :
- limiter la longueur des recherches ;
- autoriser uniquement les colonnes de tri prévues ;
- éviter les relations Eloquent chargées sans nécessité ;
- indexer les colonnes utilisées dans les filtres ;
- empêcher les exports massifs synchrones ;
- appliquer des délais aux appels externes ;
- mettre en file les traitements longs.

La limite HTTP ne remplace pas l’optimisation SQL : une requête autorisée peut tout de même être très coûteuse.

## 19.4. Gestion des erreurs de sécurité

Une erreur doit aider le client à comprendre ce qu’il peut corriger, sans révéler les détails qui aideraient un attaquant.

En production :
- désactivez le mode debug ;
- ne retournez pas les traces de pile ;
- ne divulguez pas les chemins du serveur, requêtes SQL ou secrets ;
- utilisez des messages cohérents ;
- journalisez les détails techniques côté serveur avec un accès contrôlé.

Exemple de réponse maîtrisée :

```json
{
  "message": "Une erreur est survenue.",
  "code": "INTERNAL_ERROR"
}
```

Pour une validation, retournez les erreurs de champs prévues par le contrat. Pour une ressource inaccessible, choisissez une réponse cohérente avec votre politique de confidentialité. N’utilisez pas un code 200 pour signaler un échec métier.

Les journaux doivent éviter les mots de passe, jetons Bearer, documents privés et données personnelles inutiles. Une trace utile contient plutôt un identifiant de requête, l’action, le résultat, l’utilisateur si disponible et le contexte non sensible.

## 19.5. CORS : comprendre son rôle et ses limites

CORS (*Cross-Origin Resource Sharing*) est un mécanisme appliqué par les navigateurs pour contrôler certaines requêtes entre origines différentes. Il intervient notamment lorsqu’une application web hébergée sur un domaine appelle une API sur un autre domaine.

CORS n’est pas un système d’authentification et ne bloque pas tous les clients : Postman, une application Flutter native ou un script serveur ne sont pas soumis aux mêmes restrictions de navigateur.

En production :
- autorisez uniquement les origines web réellement nécessaires ;
- évitez le joker `*` lorsque les identifiants ou cookies sont utilisés ;
- configurez précisément les méthodes et en-têtes autorisés ;
- ne confondez pas CORS avec la protection contre les accès non autorisés.

Une origine comprend le schéma, l’hôte et le port. `https://app.example.com` et `http://app.example.com` sont deux origines différentes.

## 19.6. Sécurité des uploads

Les fichiers envoyés peuvent contenir du contenu malveillant, être trop volumineux ou exploiter une mauvaise configuration du serveur.

Mesures pratiques :
- vérifier la taille maximale ;
- valider le type réel et les extensions attendues ;
- utiliser des noms de stockage générés par le serveur ;
- ne pas exécuter les fichiers téléversés ;
- stocker les documents sensibles hors du répertoire public ;
- contrôler l’autorisation avant téléchargement ;
- appliquer des limites au nombre de fichiers ;
- prévoir la suppression et le remplacement de manière sûre.

Une extension `.jpg` ne prouve pas que le contenu est une image valide. La validation doit s’appuyer sur les mécanismes du framework et, pour des besoins élevés, sur une analyse antivirus ou une inspection spécialisée.

## 19.7. Appliquer une lecture pratique de l’OWASP API Security

L’OWASP API Security Top 10 met en évidence des familles de risques, notamment les défauts d’autorisation au niveau des objets, l’authentification défaillante, l’autorisation au niveau des fonctions, la consommation excessive de ressources et la mauvaise configuration.

Pour utiliser ce référentiel efficacement, ne vous contentez pas de recopier les intitulés. Pour chaque risque :
1. identifiez les routes concernées ;
2. définissez un scénario d’attaque concret ;
3. choisissez le contrôle adapté ;
4. écrivez un test négatif ;
5. vérifiez les journaux et la réponse ;
6. réévaluez le risque après modification.

Par exemple, pour l’autorisation au niveau des fonctions, essayez d’appeler une route administrative avec un compte client. Pour la consommation excessive, essayez une limite de pagination extrême ou un envoi répétitif. Pour la mauvaise configuration, vérifiez les variables de production, les en-têtes, les fichiers accessibles et les erreurs retournées.

## 19.8. Atelier praticien : préparer une revue de sécurité

Avant une mise en production, examinez les points suivants :

- [ ] Toutes les routes sensibles exigent l’authentification appropriée.
- [ ] Les autorisations sont vérifiées sur les objets et les fonctions.
- [ ] Les champs privilégiés ne sont pas contrôlés par le client.
- [ ] Les réponses ne révèlent pas de données inutiles.
- [ ] Les tentatives de connexion et actions coûteuses sont limitées.
- [ ] Les uploads ont des limites et un stockage adapté.
- [ ] CORS est configuré pour les origines nécessaires.
- [ ] Les erreurs de production ne révèlent pas de détails internes.
- [ ] Les secrets sont absents du dépôt et des journaux.
- [ ] Les tests couvrent les accès entre utilisateurs et les rôles.

Cette revue doit être répétée après l’ajout de nouvelles routes ou fonctionnalités, pas seulement avant la première publication.

## 19.9. Exercice de synthèse

Un client connecté envoie 500 demandes en quelques minutes, modifie le champ `status`, tente de consulter les demandes d’autres clients et reçoit une erreur contenant une trace SQL.

Identifiez au moins quatre catégories de défauts et proposez les mesures correspondantes.

**Correction indicative :** limitation de fréquence et détection d’abus ; validation et contrôle explicite des champs ; Policy et test BOLA ; gestion maîtrisée des exceptions avec debug désactivé et journalisation sans données sensibles.

## À retenir

Une API résistante associe autorisation, validation, limites de ressources, configuration maîtrisée et tests négatifs. La sécurité est une activité continue : chaque nouvelle fonctionnalité ajoute des routes, des données et des scénarios à examiner.