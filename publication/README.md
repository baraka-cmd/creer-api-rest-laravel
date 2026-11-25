# Préparer et publier l'ebook

Ce dépôt contient le manuscrit source et la configuration de publication Quarto.

## Formats générés

- HTML de prévisualisation
- PDF de lecture/impression
- EPUB pour liseuses et applications ebook

## Génération locale

Prérequis : Quarto, TinyTeX/XeLaTeX et la police Poppins installée sur le système.

Depuis la racine :

```powershell
quarto preview
quarto render --to pdf
quarto render --to epub
```

Les fichiers sont générés dans `_book/`.

## Génération sur GitHub

Le workflow `.github/workflows/build-ebook.yml` génère le PDF et l'EPUB automatiquement lors d'un push sur les branches configurées, ou manuellement via **Actions → Build ebook PDF and EPUB → Run workflow**.

Quand le workflow réussit, ouvrir son exécution et télécharger l'artefact **ebook-pdf-epub** dans la section Artifacts.

## Avant une publication commerciale

Le succès de la génération technique ne vaut pas validation éditoriale. Il faut encore :

1. relire et vérifier les 30 chapitres et annexes ;
2. corriger les éventuels doublons de chapitre 30 et vérifier l'ordre ;
3. valider tous les exemples de code et routes contre la version Laravel annoncée ;
4. contrôler les liens, références, licences et crédits d'illustrations ;
5. créer et intégrer une couverture originale ;
6. inspecter visuellement le PDF et l'EPUB sur plusieurs lecteurs.

Ne pas présenter les fichiers comme une édition définitive avant ces contrôles.
