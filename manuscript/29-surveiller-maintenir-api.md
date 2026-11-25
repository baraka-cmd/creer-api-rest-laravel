# Chapitre 29 — Surveiller et maintenir son API : assurer la continuité du service

## 29.1. Le travail ne s'arrête pas après le déploiement

Une API en production évolue dans un environnement réel : utilisateurs simultanés, données qui augmentent, dépendances externes, erreurs inattendues et changements d'infrastructure.

Un service professionnel doit être exploité dans la durée. Cela signifie observer son état, détecter les problèmes, protéger les données, corriger les incidents et mettre à jour les composants sans introduire de régression.

La maintenance n'est pas une activité secondaire. Elle fait partie de la conception de l'application.

## 29.2. Les logs : comprendre ce qui s'est passé

Les logs enregistrent des événements utiles au diagnostic : erreur de traitement, échec de connexion, tâche interrompue, appel externe indisponible ou comportement anormal.

Un bon journal contient suffisamment de contexte pour comprendre l'événement :
- date et heure ;
- niveau de gravité ;
- nom du composant ;
- type d'opération ;
- identifiant de corrélation ou de requête, si disponible ;
- message technique pertinent ;
- exception, lorsque cela est nécessaire et sûr.

Il ne doit pas enregistrer inutilement :
- mots de passe ;
- tokens d'accès ;
- secrets ;
- données bancaires ;
- documents privés ;
- contenu intégral de requêtes sensibles.

La journalisation doit être structurée et cohérente. Un message comme « erreur » n'aide pas beaucoup. Un message qui identifie l'opération et l'étape qui a échoué est plus utile, à condition de ne pas exposer de données confidentielles.

## 29.3. Distinguer les niveaux de gravité

Les niveaux de logs permettent de prioriser les événements :

- **debug** : détails utiles au développement et au diagnostic approfondi ;
- **info** : événement normal important ;
- **warning** : situation inhabituelle qui ne bloque pas nécessairement le service ;
- **error** : opération échouée ;
- **critical** : défaillance grave qui exige une réaction rapide.

La production ne doit pas être noyée dans des messages de debug permanents. Ajustez le niveau selon le besoin, tout en conservant les informations indispensables à la sécurité et à l'exploitation.

## 29.4. Monitoring : observer l'état du service

Le monitoring suit des indicateurs dans le temps. Il aide à détecter un problème avant que les utilisateurs ne le signalent, ou à confirmer son impact lorsqu'il survient.

Indicateurs importants :

### Disponibilité

L'API répond-elle aux contrôles de santé ? Les routes essentielles sont-elles accessibles ?

### Latence

Combien de temps les requêtes mettent-elles à répondre ? Examinez les percentiles, notamment les requêtes les plus lentes, plutôt que de vous limiter à une moyenne.

### Taux d'erreur

Le nombre de réponses 5xx augmente-t-il ? Les erreurs 4xx peuvent aussi révéler une mauvaise intégration ou des abus, mais ne doivent pas être confondues avec des défaillances serveur.

### Ressources

Surveillez la mémoire, le CPU, les connexions à la base, l'espace disque et les limites de la plateforme.

### Files et traitements asynchrones

Vérifiez la longueur des files, l'ancienneté des tâches en attente et les tâches échouées. Une API peut répondre rapidement tout en accumulant un retard important dans ses workers.

## 29.5. Alertes utiles et non bruyantes

Une alerte doit demander une action. Si chaque événement mineur déclenche une notification, l'équipe finit par ignorer les alertes importantes.

Définissez :
- le seuil ou la condition ;
- la durée pendant laquelle le problème doit persister ;
- le responsable de la réaction ;
- la procédure à suivre ;
- la gravité ;
- les conditions de retour à la normale.

Exemple : une hausse continue du taux d'erreur serveur pendant plusieurs minutes peut justifier une alerte. Une erreur isolée et attendue ne nécessite pas forcément une intervention immédiate.

## 29.6. Sauvegardes : protéger la capacité de restaurer

Une sauvegarde n'est utile que si elle peut être restaurée. Une stratégie doit préciser :
- quelles données sont sauvegardées ;
- à quelle fréquence ;
- où les copies sont stockées ;
- qui peut les consulter ;
- combien de temps elles sont conservées ;
- comment leur intégrité est vérifiée ;
- comment la restauration est testée.

La base de données n'est pas le seul élément à protéger. Selon l'application, il faut également sauvegarder les fichiers persistants, les configurations nécessaires et les informations permettant de reconstruire le service.

Les sauvegardes doivent être séparées du serveur principal. Si une panne ou un accès malveillant affecte le serveur, une copie stockée uniquement au même endroit peut être perdue.

## 29.7. Définir une stratégie de restauration

La restauration doit être documentée et testée. Une procédure générale peut être :

1. identifier le périmètre de l'incident ;
2. choisir une sauvegarde adaptée ;
3. préparer un environnement de restauration ;
4. restaurer la base et les fichiers ;
5. vérifier la cohérence ;
6. contrôler les accès et les secrets ;
7. tester les endpoints essentiels ;
8. remettre le service en ligne ;
9. informer les personnes concernées ;
10. documenter les écarts et les données potentiellement perdues.

Deux notions aident à planifier :
- **RPO** : quantité maximale de données que l'organisation accepte de perdre, exprimée en durée ;
- **RTO** : durée maximale visée pour rétablir le service.

Ces objectifs dépendent du contexte métier. Une plateforme de démonstration et un système de paiement n'ont pas les mêmes exigences.

## 29.8. Gérer un incident avec méthode

Lorsqu'une API devient indisponible, évitez les modifications désordonnées. Une réaction structurée réduit le risque d'aggraver le problème.

### Étape 1 : confirmer

Vérifiez si le problème est réel, depuis quel point d'accès et depuis quand. Distinguez une panne globale d'un problème limité à un endpoint ou à un client.

### Étape 2 : mesurer l'impact

Identifiez les fonctionnalités affectées, les utilisateurs concernés, les erreurs observées et les données potentiellement touchées.

### Étape 3 : stabiliser

Selon la cause, il peut être nécessaire de revenir à une version précédente, désactiver temporairement une fonctionnalité, augmenter une ressource ou isoler une dépendance défaillante.

### Étape 4 : corriger et vérifier

Appliquez une correction contrôlée, testez-la et observez les indicateurs avant de déclarer le service rétabli.

### Étape 5 : apprendre

Après l'incident, documentez la cause, les actions, la durée, les conséquences et les mesures préventives. Le but est d'améliorer le système, pas de rechercher une personne à blâmer.

## 29.9. Mettre à jour les dépendances

Les dépendances Laravel, PHP et les bibliothèques externes évoluent. Certaines mises à jour corrigent des vulnérabilités, d'autres apportent des améliorations ou modifient des comportements.

Une procédure saine comprend :

1. suivre les avis de sécurité et les versions supportées ;
2. examiner les changements ;
3. mettre à jour dans une branche dédiée ;
4. exécuter les tests automatisés ;
5. vérifier les dépendances et extensions requises ;
6. tester sur la préproduction ;
7. déployer progressivement ;
8. surveiller les erreurs après publication.

Ne repoussez pas indéfiniment les mises à jour de sécurité. Mais ne mettez pas non plus à jour toutes les dépendances directement sur le serveur sans tests et sans version contrôlée.

Le fichier de verrouillage Composer permet de reproduire les versions installées. Il doit être géré dans le dépôt pour une application, afin que les environnements utilisent un ensemble cohérent de dépendances.

## 29.10. Maintenance planifiée

Une routine de maintenance peut inclure :
- revue régulière des logs et alertes ;
- vérification de l'espace disque et des ressources ;
- contrôle des sauvegardes ;
- test périodique de restauration ;
- revue des comptes et permissions ;
- mise à jour des dépendances ;
- analyse des endpoints lents ;
- nettoyage des tâches échouées selon une politique ;
- vérification des certificats et domaines ;
- revue des coûts et limites d'hébergement.

La fréquence dépend du niveau de risque et du trafic. L'important est que ces contrôles soient attribués et suivis, plutôt que réalisés uniquement après une panne.

## 29.11. Atelier : rédiger un plan d'exploitation

Pour votre API de services professionnels, rédigez un document d'exploitation contenant :

1. l'URL de production et les environnements ;
2. les personnes responsables ;
3. les services et dépendances ;
4. les indicateurs surveillés ;
5. les alertes prioritaires ;
6. l'emplacement des logs ;
7. la fréquence et l'emplacement des sauvegardes ;
8. la procédure de restauration ;
9. la procédure de retour à une version précédente ;
10. les contacts et actions en cas d'incident.

Ne placez pas les mots de passe ou secrets dans ce document. Il doit indiquer où les secrets sont gérés et qui est autorisé à y accéder.

## 29.12. Checklist d'exploitation

- Les erreurs sont-elles enregistrées avec un contexte utile et sûr ?
- Les logs sont-ils protégés et leur conservation maîtrisée ?
- La disponibilité et la latence sont-elles observées ?
- Les alertes sont-elles actionnables ?
- Les workers et files sont-ils surveillés ?
- Les sauvegardes sont-elles séparées et vérifiées ?
- La restauration a-t-elle été testée ?
- Les incidents sont-ils documentés ?
- Les dépendances sont-elles suivies et mises à jour ?
- La maintenance possède-t-elle une routine et un responsable ?

**À retenir :** la qualité d'une API en production se mesure aussi à sa capacité à détecter une panne, protéger les données, restaurer le service et apprendre de ses incidents.
