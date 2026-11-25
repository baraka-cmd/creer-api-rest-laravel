# Chapitre 16 — Les fondamentaux de la sécurité d’une API

## 16.1. La sécurité n’est pas une fonctionnalité ajoutée à la fin

Une API expose des capacités de votre application à d’autres logiciels. Elle peut permettre de consulter un catalogue, créer une commande, modifier un profil ou télécharger un document. Chaque route constitue donc une possibilité d’action et, par conséquent, une partie de la surface d’attaque.

Dans une application de mise en relation comme ProxiServices, un client peut rechercher un professionnel et envoyer une demande. Un professionnel peut publier ses services et répondre à une demande. Un administrateur peut vérifier un compte. Ces fonctions ne doivent pas être accessibles indistinctement à tous.

La sécurité se construit à chaque couche : conception des routes, authentification, autorisation, validation, accès aux données, stockage, configuration, journalisation et tests. Une API sécurisée n’est pas simplement une API qui demande un mot de passe. Elle vérifie aussi ce que l’utilisateur a le droit de faire, sur quel objet, à quel moment et avec quelles données.

## 16.2. Menaces courantes : comprendre avant de coder

### Vol ou réutilisation de comptes
Un mot de passe faible, divulgué ou réutilisé peut permettre à un tiers de se connecter. Les protections comprennent le hachage des mots de passe, la limitation des tentatives, la gestion sûre des sessions et jetons, ainsi qu’un processus de réinitialisation correctement conçu.

### Accès à l’objet d’un autre utilisateur
Un utilisateur remplace l’identifiant d’une demande dans l’URL et obtient les données d’un autre client. Ce défaut, souvent appelé IDOR ou BOLA, sera étudié en profondeur au chapitre 17.

### Élévation de privilèges
Un client modifie une requête JSON en ajoutant `"role": "admin"` ou `"is_verified": true`. Si le serveur accepte ces champs sans contrôle, l’utilisateur peut s’attribuer des droits.

### Exposition excessive
Une réponse contient le mot de passe haché, un numéro privé, un document interne, un jeton ou des informations de modération. Même si l’accès à la route est autorisé, le contenu retourné peut être trop riche.

### Abus de ressources
Un attaquant envoie des milliers de requêtes, des fichiers gigantesques ou des recherches coûteuses. Le serveur peut ralentir, épuiser la mémoire ou rendre le service indisponible.

### Mauvaise configuration
Le mode débogage est activé en production, les clés sont publiées dans Git, les sauvegardes sont accessibles publiquement ou une base de données est exposée. Ces erreurs sont souvent évitables par une procédure de déploiement rigoureuse.

## 16.3. Définir la surface d’attaque

La surface d’attaque comprend tout ce qu’un acteur extérieur peut atteindre ou influencer : routes publiques et privées, paramètres, corps JSON, en-têtes, jetons, fichiers envoyés, liens de téléchargement, tâches asynchrones, intégrations externes et configuration du serveur.

Pour l’inventorier, construisez un tableau de routes :

| Route | Acteur attendu | Données manipulées | Contrôle indispensable |
|---|---|---|---|
| `GET /api/services` | Visiteur | Services publiés | Filtrer les services réellement publics |
| `POST /api/requests` | Client connecté | Demande de prestation | Vérifier le rôle, les données et le professionnel cible |
| `PATCH /api/profile` | Propriétaire | Profil | Ne permettre que les champs modifiables par le propriétaire |
| `GET /api/documents/{id}` | Utilisateur autorisé | Document privé | Vérifier l’accès à ce document précis |
| `POST /api/admin/users/{id}/verify` | Administrateur habilité | Vérification | Autorisation de fonction et journalisation |

Cet inventaire révèle souvent des oublis : une route de téléchargement, une route d’export ou une action administrative peut être moins protégée que les écrans visibles.

## 16.4. Le principe du moindre privilège

Le moindre privilège consiste à accorder uniquement les permissions nécessaires, pendant la durée nécessaire et sur les ressources nécessaires.

Un client n’a pas besoin de modifier le statut de validation d’un professionnel. Un professionnel n’a pas besoin de consulter les documents privés de tous les autres professionnels. Un administrateur de contenu n’a pas nécessairement besoin d’accéder aux paramètres financiers.

Appliquez ce principe à plusieurs niveaux :
- utilisateurs et rôles applicatifs ;
- permissions sur les objets ;
- comptes de base de données ;
- clés d’intégration ;
- accès aux fichiers ;
- services et environnements de déploiement.

Évitez le compte SQL `root` pour l’application. Créez un utilisateur de base de données avec les droits strictement requis. De même, une clé d’accès à un stockage ne devrait pas autoriser la suppression de tous les fichiers si l’application ne fait que les lire.

## 16.5. La sécurité doit être vérifiée côté serveur

L’interface peut masquer un bouton, mais elle ne constitue pas une barrière de sécurité. Un utilisateur peut appeler l’API avec Postman, un script ou une requête modifiée.

Exemple : le formulaire client ne propose pas le champ `status`. Un attaquant peut pourtant envoyer :

```json
{
  "description": "Réparation de plomberie",
  "status": "accepted",
  "professional_id": 42
}
```

Le serveur doit ignorer ou rejeter le statut fourni et appliquer lui-même les règles métier. Le statut initial doit être déterminé par le code métier, pas par le navigateur.

Règle pratique : toute donnée qui influence les droits, le propriétaire, le prix, le statut, la validation ou la comptabilité doit être contrôlée ou calculée par le serveur.

## 16.6. Secrets et variables d’environnement

Les secrets sont les éléments qui permettent d’accéder à des systèmes : clé d’application, identifiants SQL, clé de paiement, clé de stockage, secret OAuth ou clé privée.

Dans Laravel, les valeurs sensibles sont généralement placées dans le fichier `.env` en environnement local. Ce fichier ne doit pas être commité. Le dépôt doit contenir `.env.example`, avec des noms de variables et des valeurs fictives, jamais les secrets réels.

Exemple de configuration :

```dotenv
APP_ENV=local
APP_DEBUG=true
APP_KEY=
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_DATABASE=proxiservices
DB_USERNAME=proxiservices_app
DB_PASSWORD=
```

En production, `APP_DEBUG=false` est indispensable. Les secrets doivent être injectés par le gestionnaire de configuration ou de secrets de l’hébergeur, avec des droits limités. Ne copiez jamais une clé de production dans une capture d’écran, un fichier de documentation ou un message d’erreur.

Si un secret a été publié, supprimer la ligne du dépôt ne suffit pas : il faut révoquer ou renouveler le secret, puis examiner les accès effectués.

## 16.7. Construire une défense en profondeur

La défense en profondeur consiste à ne pas dépendre d’un seul contrôle. Une route sensible peut nécessiter :
1. une connexion valide ;
2. une permission correspondant à la fonction ;
3. une vérification de propriété ou de relation métier ;
4. une validation stricte des données ;
5. une réponse qui ne révèle que les champs nécessaires ;
6. une limitation des requêtes ;
7. une trace de l’action lorsque cela est pertinent.

Si une couche est mal configurée, les autres réduisent les conséquences.

## 16.8. Atelier praticien : sécuriser une demande de prestation

Supposons que la route `POST /api/requests` crée une demande.

Avant de coder, posez les questions suivantes :
- L’utilisateur est-il authentifié ?
- Son compte est-il actif ?
- A-t-il le droit de créer une demande ?
- Le service existe-t-il et est-il publié ?
- Le service appartient-il à un professionnel autorisé ?
- Le client peut-il envoyer un montant ou un statut arbitraire ?
- Les données sont-elles validées ?
- La réponse révèle-t-elle des informations privées ?
- Une répétition de la requête peut-elle créer des doublons ?

Cette liste devient un plan de conception et de tests, pas une simple checklist à cocher après coup.

## 16.9. Exercice

Pour chacune des routes suivantes, identifiez les menaces et les contrôles : `PATCH /api/users/7`, `GET /api/quotes/19`, `POST /api/services`, `GET /api/private-files/5`.

**Correction indicative :** vérifier l’authentification, le rôle, la propriété ou la relation avec l’objet, les champs autorisés, la validation, la limitation éventuelle, et la confidentialité de la réponse. Les contrôles précis dépendent de la règle métier.

## À retenir

La sécurité API repose sur des décisions explicites. Inventoriez les routes, définissez les acteurs, réduisez les privilèges, contrôlez les données côté serveur et protégez les secrets. Les chapitres suivants transformeront ces principes en mécanismes Laravel concrets.