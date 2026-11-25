# Chapitre 28 — Déployer son API : rendre le service accessible et vérifier sa mise en ligne

## 28.1. Ce que signifie réellement déployer

Déployer une API consiste à installer ou publier une version du code dans un environnement accessible, lui fournir sa configuration, connecter ses dépendances et vérifier que le service fonctionne.

Le déploiement ne se limite pas à envoyer les fichiers sur un serveur. Il comprend plusieurs étapes : préparation du code, installation des dépendances, configuration, migrations, construction des assets si nécessaire, démarrage des processus, connexion au domaine, activation de HTTPS et tests après mise en ligne.

Un déploiement professionnel doit être reproductible. Une autre personne de l'équipe devrait pouvoir suivre la procédure sans dépendre de manipulations secrètes effectuées manuellement.

## 28.2. Choisir un hébergement adapté

Les solutions d'hébergement se répartissent en plusieurs familles.

### Hébergement mutualisé

Plusieurs sites partagent un même serveur. Cette solution peut être économique et simple, mais elle offre parfois moins de contrôle sur les processus, les versions PHP, les tâches en arrière-plan et la configuration serveur.

Avant de choisir, vérifiez que l'offre supporte réellement la version PHP et les extensions requises par votre projet Laravel, ainsi que les tâches nécessaires.

### Serveur virtuel ou VPS

Un VPS donne davantage de contrôle sur le système, le serveur web, PHP, les processus et les règles réseau. Il exige cependant de savoir administrer le système, sécuriser SSH, appliquer les mises à jour, surveiller les ressources et configurer les sauvegardes.

La liberté technique implique une responsabilité opérationnelle plus importante.

### Plateforme cloud ou PaaS

Une plateforme PaaS peut automatiser une partie du build, du déploiement, du routage HTTPS et de la gestion des services. Elle convient souvent à un projet qui veut se concentrer sur le code plutôt que sur l'administration complète du système.

Il faut néanmoins comprendre les limites : stockage temporaire, démarrage à froid, quotas, région, coût, réseau, base de données, tâches programmées et workers.

### Critères de décision

Comparez :
- compatibilité PHP, extensions et Laravel ;
- disponibilité et emplacement de la base ;
- stockage persistant et fichiers ;
- HTTPS et domaine personnalisé ;
- tâches asynchrones et tâches planifiées ;
- journaux et monitoring ;
- sauvegardes et restauration ;
- coût réel à mesure que le trafic augmente ;
- possibilité d'exporter ou de migrer le service.

Ne choisissez pas uniquement en fonction du prix affiché. Une offre gratuite ou très économique peut avoir des limitations qui deviennent importantes pour une application réelle.

## 28.3. Préparer le dépôt et le processus de build

Le serveur doit recevoir une version contrôlée du code. Le dépôt Git est une source de référence ; il ne doit pas contenir les secrets de production ni des fichiers générés inutiles.

Le déploiement typique peut comprendre :

1. récupérer le commit ou la version validée ;
2. installer les dépendances PHP avec Composer ;
3. appliquer les optimisations Composer adaptées à la production ;
4. construire les fichiers frontend si le projet en possède ;
5. injecter la configuration ;
6. exécuter les migrations approuvées ;
7. activer ou redémarrer la nouvelle version ;
8. vérifier les endpoints et les journaux.

Pour une application Laravel, l'installation de dépendances de production peut utiliser une commande Composer adaptée, par exemple :

    composer install --no-dev --optimize-autoloader

Cette commande doit être exécutée dans un environnement compatible avec les extensions et versions exigées par le projet. Ne copiez pas le dossier vendor depuis un ordinateur arbitraire sans vérifier sa compatibilité.

## 28.4. Configurer le domaine et HTTPS

Un domaine donne une adresse stable à l'API, par exemple api.exemple.com. La configuration DNS relie le nom au service d'hébergement selon les instructions du fournisseur.

Les étapes habituelles sont :

1. enregistrer ou choisir le domaine ;
2. créer l'enregistrement DNS demandé par l'hébergeur ;
3. attendre la propagation nécessaire ;
4. vérifier que le domaine pointe vers le bon service ;
5. activer le certificat TLS ;
6. tester le renouvellement automatique du certificat ;
7. configurer l'application pour utiliser l'URL HTTPS.

HTTPS protège les échanges entre le client et le serveur. Pour une API qui utilise des jetons d'authentification, il est indispensable : un token transmis sur une connexion non chiffrée peut être intercepté.

Ne considérez pas le cadenas du navigateur comme une vérification complète. Testez les endpoints, les redirections, les certificats et la configuration du proxy lorsque l'application est derrière un service intermédiaire.

## 28.5. Configurer le stockage des fichiers

Les fichiers uploadés par les utilisateurs ne doivent pas être considérés comme du code de l'application. Leur stockage doit être adapté à leur nature.

Pour les fichiers publics, une URL publique peut être fournie si le fichier a été contrôlé et si sa publication est autorisée.

Pour les documents privés — pièces justificatives, contrats, documents d'identité ou fichiers liés à une demande — utilisez un stockage privé et un mécanisme d'accès autorisé. Le serveur doit vérifier l'identité et le droit d'accès avant de fournir le fichier.

Sur une plateforme cloud, le disque local peut être temporaire ou propre à une instance. Un fichier enregistré localement peut disparaître lors d'un redéploiement ou ne pas être accessible depuis une autre instance. Utilisez un stockage persistant ou un service objet adapté lorsque l'architecture l'exige.

Vérifiez également :
- la taille maximale des fichiers ;
- les types MIME et extensions autorisés ;
- les noms de fichiers générés côté serveur ;
- les permissions ;
- les URL temporaires ou contrôlées ;
- les règles de suppression et de conservation.

## 28.6. Migrations et stratégie de déploiement

Les migrations doivent être exécutées dans le bon ordre et avec une version du code compatible.

Un risque courant apparaît lorsque la nouvelle version du code attend une colonne qui n'existe pas encore, ou lorsque l'ancienne version continue d'utiliser une colonne supprimée par la migration.

Pour réduire ce risque, utilisez une évolution progressive du schéma :

- ajouter d'abord les nouvelles colonnes ou tables ;
- déployer un code capable de fonctionner avec l'ancien et le nouveau schéma ;
- migrer les données si nécessaire ;
- basculer l'utilisation ;
- supprimer les éléments anciens dans une étape ultérieure.

Cette méthode demande parfois plusieurs déploiements, mais elle facilite le retour arrière et limite les interruptions.

## 28.7. Vérifier le service après déploiement

Après publication, ne vous contentez pas de voir que le processus est démarré. Testez le comportement réel.

Une checklist de smoke tests peut inclure :

- endpoint de santé ou vérification de disponibilité ;
- route publique de lecture ;
- route protégée sans token, qui doit refuser l'accès ;
- route protégée avec token valide ;
- validation d'une entrée incorrecte ;
- pagination ;
- lecture d'une ressource inexistante ;
- accès à un fichier public autorisé ;
- refus d'accès à un fichier privé par un autre utilisateur ;
- connexion à la base ;
- exécution ou disponibilité des workers ;
- présence d'erreurs nouvelles dans les logs.

Les tests de production doivent rester non destructifs. N'effectuez pas de création de commande, paiement ou suppression réelle sans procédure de test adaptée.

## 28.8. Prévoir le retour arrière

Un déploiement peut introduire une régression. Avant la publication, définissez ce qui peut être restauré :

- version précédente du code ;
- configuration précédente ;
- fichiers de build ;
- état de la base ;
- migrations irréversibles ;
- tâches déjà exécutées.

Le retour arrière du code ne suffit pas toujours si la base a changé. C'est pourquoi la compatibilité entre versions et migrations est essentielle.

Pour une migration destructive, une sauvegarde seule peut ne pas suffire si la restauration entraîne une longue interruption. Préparez une stratégie qui tient compte de la durée de restauration et des données créées après la sauvegarde.

## 28.9. Atelier de déploiement

### Objectif

Publier une API Laravel de catalogue sur un environnement cloud, puis vérifier son fonctionnement depuis Postman et depuis un client mobile.

### Étapes

1. choisir un hébergement compatible ;
2. relier le dépôt à la plateforme ou préparer le processus de publication ;
3. configurer les variables de production ;
4. créer et sécuriser la base ;
5. déployer le code ;
6. appliquer les migrations validées ;
7. configurer le domaine et HTTPS ;
8. configurer le stockage ;
9. exécuter les smoke tests ;
10. vérifier les logs et préparer un retour arrière.

### Résultat attendu

L'API répond avec une URL stable, utilise HTTPS, ne divulgue pas les erreurs internes, refuse les accès non autorisés et conserve les fichiers selon la stratégie définie.

## 28.10. Checklist de mise en ligne

- Le code publié correspond-il à un commit identifié ?
- Les dépendances de production sont-elles installées correctement ?
- Les variables et secrets sont-ils configurés ?
- La base est-elle sécurisée et sauvegardée ?
- Les migrations ont-elles été examinées ?
- Le domaine pointe-t-il vers le bon service ?
- HTTPS fonctionne-t-il ?
- Les fichiers sont-ils stockés de manière durable et appropriée ?
- Les endpoints essentiels ont-ils été testés ?
- Les tâches asynchrones sont-elles actives ?
- Une procédure de retour arrière existe-t-elle ?

**À retenir :** un déploiement réussi n'est pas seulement une publication du code. C'est une version configurée, accessible, sécurisée, testée et exploitable.
