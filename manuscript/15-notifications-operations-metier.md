# Chapitre 15 — Notifications et opérations métier

## Objectifs

Comprendre les notifications Laravel, choisir les canaux adaptés, protéger les opérations sensibles avec des transactions et concevoir des actions idempotentes.

## 15.1 Pourquoi notifier ?

Dans ProxiServices, un professionnel doit être informé lorsqu'un client lui envoie une demande. Le client peut ensuite recevoir une notification lorsque le professionnel répond.

Une notification peut être affichée dans l'application, envoyée par email ou transmise par un autre canal configuré. Le canal doit correspondre au besoin et au consentement de l'utilisateur.

## 15.2 Créer une notification Laravel

```bash
php artisan make:notification NewServiceRequestNotification
```

Une notification peut définir ses canaux et son contenu. Exemple simplifié pour une notification en base :

```php
public function via(object $notifiable): array
{
    return ['database'];
}

public function toArray(object $notifiable): array
{
    return [
        'request_id' => $this->serviceRequest->id,
        'message' => 'Vous avez reçu une nouvelle demande.',
    ];
}
```

L'utilisateur peut recevoir la notification via le système Laravel, à condition que le modèle notifiable et la table requise soient configurés.

Ne placez pas de données confidentielles dans une notification visible sur l'écran verrouillé du téléphone.

## 15.3 Envoyer une notification

```php
$professional->notify(
    new NewServiceRequestNotification($serviceRequest)
);
```

Dans un système réel, la notification peut être mise en file d'attente afin que l'envoi email ou réseau ne ralentisse pas la réponse HTTP. Cela nécessite un driver de queue correctement configuré et un worker actif.

La mise en file d'attente introduit une responsabilité d'exploitation : surveiller les tâches échouées, relancer de manière sûre et comprendre les délais.

## 15.4 Transactions pour les opérations cohérentes

Supposons qu'une demande crée une ligne métier et une entrée d'historique. Ces écritures doivent réussir ensemble.

```php
$result = DB::transaction(function () use ($data, $client) {
    $request = ServiceRequest::create([
        'client_id' => $client->id,
        'service_id' => $data['service_id'],
        'status' => 'pending',
    ]);

    RequestHistory::create([
        'service_request_id' => $request->id,
        'event' => 'created',
    ]);

    return $request;
});
```

Si une écriture échoue, la transaction annule les écritures qu'elle contient. Elle ne garantit pas automatiquement l'envoi d'un email ou d'une notification externe, car ces systèmes ne font pas partie de la transaction SQL.

## 15.5 Le problème des effets externes

Évitez d'envoyer un email au milieu d'une transaction qui peut encore être annulée. Le destinataire pourrait recevoir une confirmation alors que la base ne contient finalement pas l'opération.

Une stratégie consiste à déclencher les notifications après la validation de la transaction, ou à enregistrer un événement à traiter par une tâche asynchrone. Pour les systèmes exigeants, le modèle outbox permet de conserver en base l'événement à transmettre et de le traiter de manière fiable.

## 15.6 Comprendre l'idempotence

Une action est idempotente lorsque la répéter avec les mêmes paramètres et dans le même contexte ne produit pas plusieurs effets métier indésirables.

Exemple concret : un réseau mobile peut couper la connexion après que le serveur a créé une demande, mais avant que le client reçoive la réponse. Le client réessaie. Sans protection, deux demandes identiques peuvent être créées.

Une clé d'idempotence peut identifier la tentative :

- le client envoie une clé unique ;
- le serveur valide son format ;
- le serveur associe la clé à l'utilisateur et à l'opération ;
- une contrainte en base empêche les doublons ;
- la répétition renvoie le résultat déjà enregistré, selon le contrat.

La clé doit être protégée contre les collisions et définie dans un périmètre précis. Une simple vérification en mémoire ou avant insertion n'est pas suffisante face aux requêtes concurrentes : la base doit participer à la garantie.

## 15.7 Opérations métier sensibles

Certaines actions exigent davantage qu'un simple CRUD :

- accepter une proposition ;
- confirmer une commande ;
- annuler une réservation ;
- déclencher un paiement ;
- publier un service ;
- envoyer une notification importante.

Elles doivent vérifier l'état actuel, les droits, les transitions autorisées et les effets secondaires. Une transition de statut ne doit pas être acceptée simplement parce que le client envoie `status=accepted`.

Le serveur doit décider si l'action est permise et enregistrer l'historique de manière cohérente.

## Atelier pratique

1. Créez une notification de nouvelle demande.
2. Configurez le canal database.
3. Créez une demande dans une transaction.
4. Déclenchez la notification après validation.
5. Simulez une répétition de requête.
6. Concevez un mécanisme qui empêche la création de deux demandes identiques.

## Exercices corrigés

**Une transaction SQL garantit-elle l'envoi d'un email ?** Non. L'email est un effet externe. Il faut une stratégie de déclenchement après commit ou une file fiable.

**Pourquoi l'idempotence est-elle importante sur mobile ?** Parce que les réseaux instables provoquent des répétitions et des délais, même lorsque l'opération a déjà été traitée.

**Le client peut-il décider directement du statut d'une commande ?** Non. Il demande une action ; le serveur valide la transition selon les règles métier.

## À retenir

Les notifications, transactions et mécanismes d'idempotence rendent les opérations métier plus fiables. Ils doivent être conçus ensemble, surtout lorsque la base, les files de tâches et les services externes interviennent.

## Références

- Laravel Notifications : https://laravel.com/docs/notifications
- Laravel Queues : https://laravel.com/docs/queues
- Laravel Database Transactions : https://laravel.com/docs/database
- AWS, Idempotent APIs (principes) : https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/
