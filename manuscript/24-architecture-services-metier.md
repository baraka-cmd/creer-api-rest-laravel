# Chapitre 24 — Organiser la logique métier : construire une API claire et maintenable

## 24.1. Pourquoi l'architecture devient importante dès que l'API grandit

Au début d'un projet, placer toute la logique dans un contrôleur semble rapide. Une route reçoit une requête, le contrôleur vérifie quelques valeurs, interroge la base et renvoie du JSON. Pour deux ou trois opérations simples, cette approche peut fonctionner.

Mais une API professionnelle ne reste pas longtemps aussi simple. Dans une plateforme de mise en relation, accepter un devis peut déclencher plusieurs opérations : vérifier que le devis appartient à la demande, confirmer que le client est autorisé, vérifier que le devis n'a pas expiré, contrôler l'état de la demande, créer une commande, enregistrer un historique et envoyer une notification. Si toute cette logique se trouve dans une méthode de contrôleur, celle-ci devient difficile à lire, à tester et à faire évoluer.

L'architecture n'est donc pas une décoration technique. Elle sert à placer chaque responsabilité au bon endroit, afin que le code reste compréhensible quand les règles métier deviennent nombreuses.

Le principe directeur de ce chapitre est simple : **le contrôleur coordonne la requête HTTP ; la logique métier exprime les règles de l'application ; les classes spécialisées assurent la validation, la transformation et l'accès aux données selon leur responsabilité.**

## 24.2. Le rôle exact d'un contrôleur

Un contrôleur Laravel constitue une frontière entre HTTP et l'application. Il reçoit les informations de la requête, s'appuie sur les mécanismes Laravel, appelle le traitement approprié et construit une réponse HTTP.

Un contrôleur lisible suit généralement cette séquence :

1. recevoir une requête déjà validée ;
2. identifier l'utilisateur et la ressource concernée ;
3. vérifier l'autorisation ;
4. appeler le traitement métier ;
5. présenter le résultat avec une API Resource ;
6. choisir le statut HTTP adapté.

~~~php
class AcceptQuotationController
{
    public function __invoke(
        AcceptQuotationRequest $request,
        Quotation $quotation,
        QuotationAcceptanceService $service
    ): JsonResponse {
        $this->authorize('accept', $quotation);

        $order = $service->accept(
            $quotation,
            $request->user()
        );

        return response()->json([
            'data' => new OrderResource($order),
            'message' => 'Le devis a été accepté.',
        ], 201);
    }
}
~~~

Cet exemple illustre une intention : le contrôleur ne décide pas lui-même de toutes les règles d'acceptation. Il délègue ce travail à un service métier.

Attention : l'injection de dépendances ne rend pas automatiquement un contrôleur professionnel. Une méthode qui injecte un service mais continue à contenir cinquante règles métier reste difficile à maintenir. La qualité vient de la répartition réelle des responsabilités, pas du nombre de classes créées.

## 24.3. Reconnaître un contrôleur trop chargé

Un contrôleur mérite une refactorisation lorsqu'on observe plusieurs symptômes :

- des conditions métier imbriquées sur plusieurs niveaux ;
- des calculs de prix, commissions ou soldes directement dans la méthode ;
- plusieurs écritures en base qui doivent réussir ensemble ;
- des appels répétés à des services externes ;
- des règles copiées dans plusieurs contrôleurs ;
- une méthode impossible à tester sans préparer une grande quantité de données ;
- des noms de variables peu explicites et une méthode qui fait plusieurs choses différentes.

Une méthode longue n'est pas automatiquement mauvaise. Une méthode qui orchestre clairement quelques étapes peut être acceptable. Le véritable problème est la concentration de responsabilités et la difficulté à comprendre les invariants métier.

## 24.4. Les services métier : quand et pourquoi les utiliser

Un service métier représente une opération ou un ensemble cohérent de règles de l'application. Il peut être nommé selon une action métier : AcceptQuotation, PublishProfessionalService, CreateOrder, CancelRequest ou CalculateCommission.

Un service est particulièrement utile lorsque :

- une opération contient plusieurs étapes liées ;
- la même règle doit être appelée depuis plusieurs points d'entrée ;
- la logique doit être testée indépendamment de HTTP ;
- une transaction doit encadrer plusieurs changements ;
- le traitement possède un nom métier clair.

Exemple de service :

~~~php
namespace App\Services\Orders;

use App\Models\Quotation;
use App\Models\Order;
use App\Models\User;
use Illuminate\Support\Facades\DB;
use DomainException;

class QuotationAcceptanceService
{
    public function accept(Quotation $quotation, User $customer): Order
    {
        return DB::transaction(function () use ($quotation, $customer) {
            $quotation = Quotation::query()
                ->lockForUpdate()
                ->findOrFail($quotation->id);

            if ($quotation->request->customer_id !== $customer->id) {
                throw new DomainException('Ce devis ne peut pas être accepté par cet utilisateur.');
            }

            if ($quotation->status !== 'sent') {
                throw new DomainException('Ce devis n’est plus disponible.');
            }

            $quotation->update(['status' => 'accepted']);

            return Order::create([
                'quotation_id' => $quotation->id,
                'customer_id' => $customer->id,
                'professional_profile_id' => $quotation->professional_id,
                'total_amount' => $quotation->amount,
                'status' => 'pending',
            ]);
        });
    }
}
~~~

Ce code est pédagogique : dans une application réelle, il faut adapter les relations, les statuts, la gestion des exceptions et les contraintes de base de données au modèle existant. Le montant provient ici du devis enregistré côté serveur ; il ne doit jamais être repris aveuglément depuis le navigateur.

La transaction garantit que les changements liés sont validés ensemble. Le verrouillage peut protéger une transition sensible contre deux acceptations simultanées, mais son usage doit être étudié avec les transactions, les index et le moteur de base de données. Un verrou n'est pas un remplacement à une contrainte métier ou à une stratégie d'idempotence.

## 24.5. Ne pas transformer chaque méthode en service

Créer une classe pour chaque ligne de code ne constitue pas une architecture. Cela ajoute des fichiers, des dépendances et des points de navigation sans améliorer la compréhension.

Un service est justifié lorsqu'il porte une responsabilité identifiable. En revanche, une opération très simple, utilisée une seule fois et déjà exprimée clairement par Eloquent, peut rester dans un contrôleur.

Évitez notamment :

- un service qui ne fait que transmettre immédiatement un appel à un modèle ;
- une classe générique nommée Manager, Helper ou Processor sans mission précise ;
- une interface créée uniquement pour une implémentation unique sans besoin de substitution ;
- des couches de dépôt (Repository) ajoutées mécaniquement alors qu'Eloquent fournit déjà les opérations nécessaires.

La bonne architecture est celle qui réduit le coût de compréhension et de changement. Elle n'est ni la plus courte ni la plus complexe.

## 24.6. Form Requests : valider la forme de la requête

Une Form Request rassemble la validation et, selon le cas, l'autorisation liée à une requête précise. Elle évite de mélanger les règles de format dans le contrôleur.

~~~php
class StoreServiceRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->hasActiveProfessionalProfile() ?? false;
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'min:5', 'max:120'],
            'description' => ['required', 'string', 'max:5000'],
            'category_id' => ['required', 'integer', 'exists:categories,id'],
            'price' => ['nullable', 'numeric', 'min:0'],
        ];
    }
}
~~~

La validation répond à la question : « Les données reçues respectent-elles les contraintes de forme et de cohérence attendues ? »

L'autorisation répond à une autre question : « Cet utilisateur a-t-il le droit d'effectuer cette opération ? »

Ne confondez pas ces deux responsabilités. Une valeur peut être parfaitement valide tout en étant interdite pour cet utilisateur. Les Policies restent essentielles pour contrôler l'accès à une ressource et ses opérations.

Ne faites pas confiance à un identifiant de propriétaire, à un rôle ou à un statut transmis par le client. Ces informations doivent être déterminées ou vérifiées côté serveur.

## 24.7. API Resources : maîtriser la représentation JSON

Le modèle Eloquent n'est pas le contrat public de l'API. Retourner directement un modèle peut exposer des colonnes inattendues, modifier la réponse après une évolution du schéma ou charger des relations inutilement.

Une API Resource définit les champs visibles :

~~~php
class ServiceResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'description' => $this->description,
            'price' => $this->price,
            'category' => new CategoryResource(
                $this->whenLoaded('category')
            ),
            'created_at' => $this->created_at?->toISOString(),
        ];
    }
}
~~~

La présentation doit être volontaire. Un service public peut exposer son titre et son prix, mais pas nécessairement les notes internes de modération, les données privées du prestataire ou les champs de sécurité.

Utilisez la condition de chargement des relations pour éviter que la Resource déclenche des requêtes cachées. Le contrôleur ou la couche de requête doit préparer les relations nécessaires.

## 24.8. Réutiliser une règle sans dupliquer tout le code

La réutilisation doit porter sur une règle réellement commune. Par exemple, la vérification d'un profil professionnel actif peut être centralisée dans une Policy ou un service d'accès si plusieurs opérations en dépendent.

Mais ne créez pas un mécanisme universel compliqué pour deux règles similaires. Commencez par rendre le code explicite. Lorsque la duplication devient réelle et coûteuse, extrayez une responsabilité cohérente.

Une bonne extraction conserve :
- un nom qui exprime l'intention ;
- des paramètres explicites ;
- un résultat compréhensible ;
- des erreurs métier identifiables ;
- des tests qui décrivent le comportement.

## 24.9. Atelier pratique : refactoriser une action d'acceptation

### Situation initiale

Un contrôleur vérifie le propriétaire, contrôle l'état du devis, calcule le montant, modifie le devis, crée la commande et envoie un message.

### Travail à effectuer

1. déplacer les règles de format dans une Form Request ;
2. placer l'autorisation de l'action dans une Policy ;
3. isoler la transition métier dans un service ;
4. encadrer les écritures liées par une transaction ;
5. présenter la commande avec une API Resource ;
6. écrire un test qui vérifie qu'un autre client ne peut pas accepter le devis ;
7. écrire un test qui vérifie qu'un devis déjà accepté ne crée pas une seconde commande.

### Questions de praticien

- Que se passe-t-il si la création de la commande échoue après la modification du devis ?
- Que se passe-t-il si deux requêtes arrivent presque simultanément ?
- Quelle valeur fait autorité pour le prix ?
- Quelle information doit être retournée au client, et laquelle doit rester privée ?
- Le service peut-il être appelé depuis une commande ou un traitement asynchrone sans dépendre de HTTP ?

## 24.10. Checklist de revue

Avant de valider une fonctionnalité, vérifiez que :

- le contrôleur reste facile à lire ;
- la validation est centralisée et explicite ;
- l'autorisation est vérifiée côté serveur ;
- la logique métier possède un nom compréhensible ;
- les écritures liées sont cohérentes ;
- la réponse JSON est maîtrisée par une Resource ;
- les règles partagées ne sont pas copiées ;
- les abstractions créées répondent à un besoin réel ;
- les scénarios d'échec et les accès interdits sont testés.

**À retenir :** une architecture professionnelle n'est pas une collection de couches obligatoires. C'est une organisation dans laquelle chaque règle se trouve à un endroit logique, chaque opération est testable et chaque changement peut être réalisé sans fragiliser tout le système.


> **Convention de référence du projet fil conducteur (mise à jour éditoriale).** Les exemples d'URL du projet utilisent le préfixe public `/api/v1`. La propriété propriétaire d'un service est `professional_profile_id` et référence le profil professionnel, non directement l'identifiant utilisateur. Le serveur déduit ce propriétaire de l'utilisateur authentifié ; cette valeur ne doit jamais être acceptée aveuglément depuis le client. Les chapitres d'extension doivent respecter ces conventions ou annoncer explicitement une variante.
