# Audit éditorial et technique du manuscrit

> Revue du dépôt `main` effectuée le 25 novembre 2026. Cette revue porte sur les fichiers Markdown disponibles. Elle ne remplace pas l'exécution de tous les exemples PHP ni l'inspection visuelle d'un PDF/EPUB final.

## Verdict général

Le manuscrit possède une **base pédagogique lisible et sérieuse**, avec une progression logique de l'API et HTTP vers Laravel, la sécurité, les tests, la performance et la production. La plupart des chapitres ont une structure compréhensible, des exemples, des exercices ou des listes de contrôle.

**Il n'est toutefois pas prêt à être déclaré parfait, complet ou publié commercialement.** Des incohérences de contrat et de modèle de données subsistent ; le chapitre 30 de réalisation, les annexes et les ressources annoncées ne sont pas encore présents ; le design final PDF/EPUB n'a pas été produit ni contrôlé visuellement.

## 1. État réel du contenu

- Les chapitres 1 à 29 sont présents dans `manuscript/`.
- `30-cahier-conception-projet-fil-conducteur.md` est un cahier de conception, pas encore le chapitre pédagogique de réalisation finale annoncé dans le plan.
- Les annexes A à J ne figurent pas dans le dépôt.
- Le dépôt ne contient actuellement que `README.md` et `manuscript/` à sa racine. Les répertoires `assets/diagrams/`, `exercises/` et `references/` décrits dans le README ne sont pas encore créés.
- Le projet Laravel compagnon, la collection Postman exportable, le jeu de données, les schémas originaux et les livrables PDF/EPUB ne sont pas encore vérifiables dans ce dépôt.

## 2. Points bloquants à corriger avant publication

### B1 — Contrat API incohérent

Le cahier de conception fixe le préfixe `/api/v1`, mais de nombreux exemples des chapitres 2 à 23 utilisent `/api` sans version. Le chapitre 25 utilise `/api/v1/services`. Il faut adopter une convention unique et la répercuter dans routes, Postman, OpenAPI, tests et exemples mobile.

### B2 — Noms de clés étrangères contradictoires

Le modèle de référence prévoit `services.professional_profile_id`, mais certains exemples utilisent `professional_id` :
- chapitre 7 : tableau conceptuel `professional_id`, migration `professional_profile_id` ;
- chapitre 21 : factory avec `professional_id` ;
- chapitre 25 : sélection et filtre avec `professional_id` ;
- chapitre 30 : `professional_profile_id`.

Le manuscrit doit choisir un seul nom et l'utiliser partout. Il faut aussi préciser si les demandes (hors périmètre MVP) référencent l'utilisateur, le profil professionnel ou les deux.

### B3 — Périmètre MVP et fonctionnalités futures mélangés

Le cahier de conception exclut initialement demandes, devis et commandes. Pourtant ces concepts sont introduits dans les chapitres 3, 15, 20, 21, 24 et 29 comme s'ils faisaient partie du parcours central. Ils peuvent rester comme extensions pédagogiques, mais doivent être clairement signalés comme **hors MVP / atelier d'extension**, avec schéma, endpoints et règles cohérents.

### B4 — Chapitre final manquant

Le cahier de conception est bien présent, mais il ne remplace pas le chapitre 30 demandé : construction complète, tests, documentation, déploiement et démonstration du projet. Le chapitre final doit reprendre le cahier comme référence et livrer un parcours intégral sans laisser des morceaux provisoires.

### B5 — Annexes et ressources de vente manquantes

Les annexes A–J, notamment la collection Postman réelle, les checklists imprimables, le glossaire, les exercices/corrigés supplémentaires et les fiches pratiques, ne sont pas encore dans le dépôt. La promesse commerciale ne doit être considérée comme satisfaite qu'après leur création et vérification.

## 3. Corrections techniques et pédagogiques identifiées

1. **Chapitre 8 :** le contrôleur CRUD fixe temporairement le propriétaire à l'identifiant `1`. L'avertissement est présent, mais ce code ne doit pas être repris dans le projet final. Le chapitre 30 devra montrer la version sécurisée, dérivée du compte authentifié et protégée par Policy.
2. **Chapitre 8 :** `response()->json(null, 204)` est à remplacer par une réponse sans contenu, par exemple `response()->noContent()`, afin de respecter proprement le contrat 204.
3. **Chapitre 7 :** harmoniser les noms de clés étrangères entre le tableau, les migrations et les relations Eloquent.
4. **Chapitre 21 :** corriger la factory d'exemple pour utiliser la clé étrangère retenue dans le modèle de référence.
5. **Chapitre 24 :** l'exemple d'acceptation de devis s'appuie sur des modèles et relations non définis dans le MVP. Le présenter comme extension complète, ou le remplacer par une opération incluse dans le projet final. Vérifier aussi les imports et le contexte de code des extraits.
6. **Chapitre 25 :** harmoniser le nom du propriétaire, le statut (`status` ou `is_published`) et le préfixe d'API avec les chapitres précédents. Les exemples d'index doivent correspondre aux colonnes réellement définies.
7. **Chapitre 17 :** la Policy de demande utilise un modèle de relation spécifique qui n'est pas détaillé dans le cahier MVP. L'indiquer comme exemple d'extension et documenter la relation avant d'en faire un code à copier.
8. **Chapitre 1 :** supprimer la note éditoriale interne « vérifier les liens… » de la version destinée au lecteur ; conserver les contrôles de liens dans une checklist de rédaction.
9. **Chapitres 2 et 3 :** harmoniser les exemples d'URL avec le nom pédagogique ProxiServices et le préfixe versionné retenu. Le chapitre 2 utilise encore un hôte de type `proxiwork.test`.
10. **Chapitre 3 :** les exemples JSON sont présentés avec indentation plutôt que des fences Markdown dans plusieurs passages ; convertir en blocs `json` pour une coloration et une lecture constantes.

## 4. Revue du style éditorial Markdown

### Points réussis
- Titres de chapitres clairs et généralement descriptifs.
- Sections et sous-sections numérotées dans une grande partie du livre.
- Exemples de code relativement faciles à repérer.
- Tableaux de comparaison et listes de contrôle utiles.
- Ton pédagogique accessible, orienté vers le raisonnement du développeur.
- Présence régulière d'exercices, de corrections et de transitions dans les premiers chapitres.

### Harmonisations nécessaires
- Certains chapitres commencent par « Objectifs », d'autres par « Introduction », et plusieurs chapitres de sécurité/performance ne suivent pas le même modèle d'ouverture.
- La structure de clôture varie : résumé, « À retenir », exercices, références et transition ne sont pas uniformes.
- Les blocs de code utilisent plusieurs styles : fences ```, fences tildées ~~~ et blocs indentés. Un seul standard doit être appliqué.
- Les conventions de typographie française doivent être normalisées : apostrophes, espaces avant ponctuation double, accents dans les titres, capitalisation des termes anglais et noms de méthodes.
- Les termes API Resource, Form Request, Policy, middleware, endpoint, token et service métier doivent avoir une terminologie stable, avec une première définition et un glossaire.
- Les références génériques Laravel (`/docs`) doivent être remplacées ou accompagnées de liens versionnés Laravel 13 pour les passages dépendants de la version.
- Les notes de rédaction et TODO doivent être retirées du texte lecteur et déplacées dans le suivi éditorial.

## 5. Design et mise en page : état honnête

Le Markdown est une source de rédaction, pas encore une maquette de livre. Il n'existe pas de preuve dans ce dépôt d'un PDF/EPUB final utilisant Poppins, d'une grille de pages, d'interlignes professionnels, d'en-têtes/pieds de page, de pagination, de table des matières cliquable ou de schémas originaux intégrés.

Le texte est lisible dans GitHub, mais **cela ne permet pas de conclure que le livre imprimé/numérique est déjà beau ou bien arrangé**. Il faudra :
- créer une charte graphique et un template de livre ;
- définir Poppins, les tailles, l'interligne, les marges et les styles de code ;
- intégrer des schémas propres et cohérents ;
- gérer les sauts de page, titres orphelins et blocs de code coupés ;
- générer PDF et EPUB ;
- vérifier les fichiers sur téléphone et ordinateur, page par page et chapitre par chapitre.

## 6. Plan de correction recommandé

### Priorité 1 — cohérence du projet
1. figer le contrat `/api/v1` ;
2. figer les noms des tables, colonnes, statuts et relations ;
3. séparer clairement MVP et extensions ;
4. corriger les exemples contradictoires ;
5. créer le vrai chapitre 30.

### Priorité 2 — complétude éditoriale
6. rédiger les annexes A–J ;
7. produire la collection Postman et les ressources téléchargeables ;
8. ajouter introduction, mode d'emploi, sommaire, conclusion et références générales ;
9. mettre à jour README et structure du dépôt.

### Priorité 3 — qualité de publication
10. uniformiser les styles Markdown ;
11. vérifier chaque commande et extrait PHP sur une version Laravel/PHP fixée ;
12. contrôler tous les liens et références ;
13. générer PDF/EPUB avec Poppins et la charte choisie ;
14. effectuer une relecture visuelle complète et une dernière revue technique.

## Conclusion

Le manuscrit a une bonne ossature et un style pédagogique prometteur. Il est **propre comme base de travail Markdown**, mais pas encore « parfait », pas encore entièrement achevé et pas encore prêt à être vendu comme un ebook final. Les incohérences relevées doivent être corrigées avant la maquette ; sinon, une belle mise en page rendrait simplement les contradictions plus visibles.

Cette revue doit servir de checklist de correction avant la rédaction finale du chapitre 30 et la préparation des annexes.
