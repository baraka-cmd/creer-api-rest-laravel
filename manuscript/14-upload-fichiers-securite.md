# Chapitre 14 — Upload et gestion des fichiers

## Objectifs

Mettre en place le dépôt d'images et de documents, valider les fichiers, choisir un stockage, protéger les fichiers privés, remplacer et supprimer les fichiers, puis contrôler les téléchargements.

## 14.1 Un fichier est une donnée à risque

Un fichier envoyé par un utilisateur peut être trop volumineux, avoir un type inattendu, contenir un contenu malveillant ou révéler des informations privées.

Ne considérez jamais le nom, l'extension ou le type MIME déclarés par le navigateur comme une preuve suffisante. Le serveur doit appliquer ses propres règles.

## 14.2 Valider un fichier

Exemple de validation d'une photo :

```php
$request->validate([
    'photo' => [
        'required',
        'image',
        'mimes:jpeg,png,webp',
        'max:2048',
    ],
]);
```

La limite de taille doit être adaptée au produit. Pour un document privé, la règle peut autoriser PDF et images, avec une limite différente. N'autorisez pas tous les formats par commodité.

Les dimensions peuvent également être contrôlées pour éviter des images extrêmement grandes. La validation de contenu, la taille et les dimensions répondent à des risques différents.

## 14.3 Stocker sans faire confiance au nom fourni

```php
$path = $request->file('photo')
    ->store('profile-photos', 'public');
```

Laravel génère un nom de fichier et retourne le chemin relatif. Évitez d'utiliser directement le nom original comme nom de stockage : il peut contenir des caractères inattendus, des collisions ou des informations personnelles.

Le disque `public` convient aux fichiers réellement destinés à être accessibles publiquement. Les documents d'identité, justificatifs et fichiers confidentiels doivent utiliser un stockage privé.

## 14.4 Stockage local et distant

Le système de fichiers Laravel permet de configurer différents disques : local, public ou stockage compatible cloud, selon les adaptateurs installés.

Le choix dépend de l'architecture :

- local : simple pour le développement et certains déploiements ;
- stockage objet distant : utile pour la durabilité, l'évolutivité et le partage entre plusieurs instances ;
- stockage privé : accès contrôlé par l'application ou par des liens temporaires.

Ne supposez pas qu'un fichier stocké sur le disque local d'un serveur de déploiement restera disponible après chaque redémarrage ou remplacement d'instance. Vérifiez les caractéristiques du fournisseur d'hébergement.

## 14.5 Fichiers privés et téléchargements

Pour un document privé, ne renvoyez pas un chemin public permanent. Vérifiez d'abord que l'utilisateur a le droit d'accéder au document, puis utilisez une réponse de téléchargement ou un lien temporaire selon le stockage.

Le contrôleur doit vérifier :

1. l'existence du document ;
2. l'identité de l'appelant ;
3. la policy d'accès ;
4. le chemin de stockage attendu ;
5. le comportement en cas de fichier manquant.

Un identifiant de document difficile à deviner ne remplace pas l'autorisation.

## 14.6 Remplacer et supprimer un fichier

Lors du remplacement, stockez d'abord le nouveau fichier et vérifiez que l'opération réussit. Ensuite, mettez à jour la référence en base et supprimez l'ancien fichier selon une stratégie qui évite de perdre la seule copie valide.

La base et le stockage de fichiers ne partagent pas nécessairement une transaction atomique. Pour les opérations sensibles, prévoyez une procédure de compensation ou une tâche de nettoyage si l'une des étapes échoue.

Ne supprimez pas un fichier partagé par plusieurs ressources sans vérifier ses références.

## 14.7 Sécurité et exploitation

- Limitez la taille et les types autorisés.
- Utilisez des noms générés.
- Stockez les fichiers privés hors d'un accès public direct.
- Vérifiez les autorisations à chaque téléchargement.
- Analysez les fichiers si le niveau de risque le justifie.
- Évitez d'exécuter les fichiers téléversés.
- Prévoyez des quotas et une stratégie de suppression.
- Ne consignez pas le contenu ou les données sensibles du document dans les logs.

## Atelier pratique

1. Envoyez une image valide avec Postman en multipart/form-data.
2. Vérifiez le chemin retourné.
3. Essayez une extension interdite.
4. Envoyez un fichier trop volumineux.
5. Testez l'accès au document avec un autre compte.
6. Remplacez le fichier et vérifiez que l'ancien n'est plus référencé.

## Exercices corrigés

**Pourquoi générer un nom serveur ?** Pour éviter les collisions, les noms manipulés et les fuites d'informations dans le nom du fichier.

**Un lien difficile à deviner suffit-il ?** Non. Les autorisations doivent être vérifiées côté serveur.

**Pourquoi ne pas stocker tous les documents dans le disque public ?** Parce qu'un document privé pourrait être accessible sans contrôle applicatif.

## À retenir

La gestion des fichiers comprend le transfert, la validation, le stockage, les autorisations, le remplacement, la suppression et l'exploitation. Le chemin du fichier n'est pas un contrôle de sécurité.

## Références

- Laravel Filesystem : https://laravel.com/docs/filesystem
- Laravel Validation : https://laravel.com/docs/validation
- OWASP File Upload Cheat Sheet : https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
