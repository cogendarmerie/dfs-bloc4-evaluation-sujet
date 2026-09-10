# Documentation d'API

## 1. Vue d'ensemble de l'API

L'application **OpsTrack Field Service** expose une API REST développée avec Laravel. Elle permet notamment au frontend `dispatch-dashboard` de communiquer avec le backend et d'accéder aux données et fonctionnalités métier.

Les échanges avec l'API utilisent le format **JSON**.

| Champ            | Valeur                                                  |
| ---------------- | ------------------------------------------------------- |
| URL de base      | `https://eval-dfs-p-tpl-20265-08.it-students.fr/api/v1` |
| Format           | `JSON`                                                  |
| Authentification | Token Bearer via le middleware `api.token`              |
| Version          | `v1`                                                    |

Un endpoint de santé est également disponible en dehors du préfixe `/v1` :

```text
GET /api/health
```

Cet endpoint est volontairement accessible sans authentification afin de permettre la supervision et la vérification de disponibilité de l'application.

---

## 2. Authentification

Les endpoints situés sous `/api/v1` sont protégés par le middleware Laravel :

```text
api.token
```

L'authentification est réalisée à l'aide d'un token transmis dans l'en-tête HTTP `Authorization`.

### Format

```http
Authorization: Bearer <TOKEN>
```

Exemple :

```bash
curl -i \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  https://eval-dfs-p-tpl-20265-08.it-students.fr/api/v1/tickets
```

Le token ne doit pas être stocké directement dans le code source, dans le dépôt Git ou dans la documentation.

L'endpoint `/api/health` constitue une exception et ne nécessite pas de token.

Un test automatisé vérifie notamment que l'API des tickets nécessite un token valide :

```text
✓ the ticket api requires a valid token
```

---

## 3. Endpoints disponibles

Les routes ont été vérifiées depuis l'environnement Laravel avec :

```bash
php artisan route:list
```

Les endpoints actuellement exposés sont les suivants :

| Méthode         | Endpoint                   | Description                             | Authentification |
| --------------- | -------------------------- | --------------------------------------- | ---------------- |
| `GET`           | `/api/health`              | Vérification de l'état de l'application | Non              |
| `GET`           | `/api/v1/tickets`          | Liste des tickets                       | Oui              |
| `POST`          | `/api/v1/tickets`          | Création d'un ticket                    | Oui              |
| `GET`           | `/api/v1/tickets/{ticket}` | Consultation d'un ticket                | Oui              |
| `PUT` / `PATCH` | `/api/v1/tickets/{ticket}` | Modification d'un ticket                | Oui              |
| `GET`           | `/api/v1/technicians`      | Liste des techniciens                   | Oui              |
| `GET`           | `/api/v1/external/weather` | Récupération de la météo d'un site      | Oui              |

---

# 4. Endpoint de santé

## `GET /api/health`

Cet endpoint permet de vérifier rapidement que l'application Laravel est opérationnelle.

Il est notamment destiné à être utilisé par les mécanismes de supervision.

### Authentification

Aucune authentification n'est nécessaire.

### Requête

```http
GET /api/health
Accept: application/json
```

Exemple :

```bash
curl -i \
  https://eval-dfs-p-tpl-20265-08.it-students.fr/api/health
```

### Réponse

Code HTTP attendu :

```text
200 OK
```

Exemple de réponse :

```json
{
    "status": "ok",
    "service": "OpsTrack Field Service",
    "timestamp": "2026-09-10T12:00:00+00:00"
}
```

La réponse contient :

| Champ       | Description                               |
| ----------- | ----------------------------------------- |
| `status`    | État de l'application                     |
| `service`   | Nom de l'application                      |
| `timestamp` | Date et heure de traitement de la requête |

---

# 5. Gestion des tickets

Les endpoints de gestion des tickets sont fournis par `TicketController`.

Toutes les routes de cette section sont protégées par le middleware `api.token`.

---

## 5.1 Lister les tickets

### `GET /api/v1/tickets`

Cet endpoint permet de récupérer la liste des tickets.

### Authentification

Token Bearer obligatoire.

### Requête

```http
GET /api/v1/tickets
Accept: application/json
Authorization: Bearer <TOKEN>
```

Exemple :

```bash
curl -i \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  https://eval-dfs-p-tpl-20265-08.it-students.fr/api/v1/tickets
```

### Pagination

Les résultats sont paginés.

Le nombre de résultats par page peut être contrôlé avec le paramètre :

```text
per_page
```

La valeur par défaut est de **15 tickets par page**.

Exemple :

```text
GET /api/v1/tickets?per_page=20
```

### Recherche

Le paramètre `search` permet d'effectuer une recherche sur :

* le titre du ticket ;
* la référence du ticket.

Exemple :

```text
GET /api/v1/tickets?search=serveur
```

### Filtrage par priorité

Le paramètre `priority` permet de filtrer les tickets selon leur priorité.

Exemple :

```text
GET /api/v1/tickets?priority=high
```

### Combinaison des paramètres

Les différents paramètres peuvent être combinés.

Exemple :

```text
GET /api/v1/tickets?search=serveur&priority=high&per_page=20
```

Avec `curl` :

```bash
curl -G \
  https://eval-dfs-p-tpl-20265-08.it-students.fr/api/v1/tickets \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  --data-urlencode "search=serveur" \
  --data-urlencode "priority=high" \
  --data-urlencode "per_page=20"
```

### Données retournées

Les tickets sont retournés via `TicketResource`.

Lors de la récupération, les relations suivantes sont également chargées :

* `site` ;
* `openedBy` ;
* `assignedTo` ;
* `interventions`.

---

## 5.2 Créer un ticket

### `POST /api/v1/tickets`

Cet endpoint permet de créer un nouveau ticket.

### Authentification

Token Bearer obligatoire.

### Requête

```http
POST /api/v1/tickets
Content-Type: application/json
Accept: application/json
Authorization: Bearer <TOKEN>
```

Exemple :

```bash
curl -i -X POST \
  https://eval-dfs-p-tpl-20265-08.it-students.fr/api/v1/tickets \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{
    "title": "Panne matériel",
    "description": "Le matériel ne démarre plus"
  }'
```

Les données reçues sont validées par :

```text
StoreTicketRequest
```

Les règles de validation détaillées sont donc centralisées dans cette classe.

### Traitement effectué

Lors de la création, l'application :

1. valide les données reçues ;
2. crée le ticket ;
3. génère automatiquement sa référence ;
4. définit le statut `new` par défaut lorsqu'aucun statut n'est fourni ;
5. charge les relations du ticket ;
6. enregistre un événement dans le journal technique ;
7. retourne le ticket créé sous la forme d'un `TicketResource`.

### Génération de la référence

La référence du ticket est générée automatiquement par l'application sous la forme :

```text
INC-XXXXXX
```

Elle ne doit donc pas être fournie par le client lors de la création.

### Journalisation

La création du ticket génère également un événement :

```text
ticket.created
```

Les informations suivantes sont notamment enregistrées :

```text
ticket_id
reference
```

---

## 5.3 Consulter un ticket

### `GET /api/v1/tickets/{ticket}`

Cet endpoint permet de récupérer un ticket particulier.

Le paramètre `{ticket}` correspond à l'identifiant du ticket.

### Authentification

Token Bearer obligatoire.

### Requête

```http
GET /api/v1/tickets/{ticket}
Accept: application/json
Authorization: Bearer <TOKEN>
```

Exemple :

```bash
curl -i \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  https://eval-dfs-p-tpl-20265-08.it-students.fr/api/v1/tickets/8
```

### Traitement

Le ticket demandé est chargé avec les relations :

* `site` ;
* `openedBy` ;
* `assignedTo` ;
* `interventions`.

La réponse est ensuite retournée via `TicketResource`.

---

## 5.4 Modifier un ticket

### `PUT/PATCH /api/v1/tickets/{ticket}`

Cet endpoint permet de modifier un ticket existant.

### Authentification

Token Bearer obligatoire.

### Requête

```http
PUT /api/v1/tickets/{ticket}
Content-Type: application/json
Accept: application/json
Authorization: Bearer <TOKEN>
```

Exemple :

```bash
curl -i -X PUT \
  https://eval-dfs-p-tpl-20265-08.it-students.fr/api/v1/tickets/8 \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{
    "title": "Panne matériel - mise à jour",
    "description": "Diagnostic en cours"
  }'
```

Les données reçues sont validées par :

```text
UpdateTicketRequest
```

### Gestion de la fermeture

Lorsqu'un ticket passe au statut :

```text
resolved
```

ou :

```text
closed
```

l'application renseigne automatiquement la date de fermeture `closed_at`.

### Journalisation

Les modifications sont enregistrées dans le journal technique avec l'événement :

```text
ticket.updated
```

Le journal contient notamment :

* l'identifiant du ticket ;
* les changements effectués ;
* un identifiant de requête permettant de faciliter la traçabilité.

---

# 6. Techniciens

## `GET /api/v1/technicians`

Cet endpoint permet de récupérer la liste des utilisateurs ayant le rôle `technician`.

### Authentification

Token Bearer obligatoire.

### Requête

```http
GET /api/v1/technicians
Accept: application/json
Authorization: Bearer <TOKEN>
```

Exemple :

```bash
curl -i \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  https://eval-dfs-p-tpl-20265-08.it-students.fr/api/v1/technicians
```

### Traitement

L'application :

1. recherche les utilisateurs dont `role` vaut `technician` ;
2. trie les résultats par nom ;
3. sélectionne uniquement les champs nécessaires ;
4. retourne les résultats dans une propriété `data`.

Les champs exposés sont :

| Champ   | Description               |
| ------- | ------------------------- |
| `id`    | Identifiant du technicien |
| `name`  | Nom du technicien         |
| `email` | Adresse e-mail            |
| `phone` | Numéro de téléphone       |

### Exemple de réponse

```json
{
    "data": [
        {
            "id": 1,
            "name": "Jean Dupont",
            "email": "jean.dupont@example.com",
            "phone": "+33600000000"
        },
        {
            "id": 2,
            "name": "Marie Martin",
            "email": "marie.martin@example.com",
            "phone": "+33611111111"
        }
    ]
}
```

Le contrôleur sélectionne explicitement les champs retournés afin de ne pas exposer inutilement les autres attributs du modèle `User`.

---

# 7. Contexte externe — Météo

## `GET /api/v1/external/weather`

Cet endpoint permet de récupérer les conditions météorologiques actuelles associées à un site.

Les données sont récupérées par le service :

```text
PublicWeatherService
```

### Authentification

Token Bearer obligatoire.

### Paramètres

| Paramètre | Emplacement         | Type      | Obligatoire | Description                  |
| --------- | ------------------- | --------- | ----------- | ---------------------------- |
| `site_id` | Corps de la requête | `integer` | Oui         | Identifiant du site concerné |

### Requête

```http
GET /api/v1/external/weather
Content-Type: application/json
Accept: application/json
Authorization: Bearer <TOKEN>

{
    "site_id": 1
}
```

Exemple :

```bash
curl -i -X GET \
  https://eval-dfs-p-tpl-20265-08.it-students.fr/api/v1/external/weather \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{
    "site_id": 1
  }'
```

### Traitement

L'application :

1. récupère le `site_id` transmis par le client ;
2. recherche le site correspondant dans la base de données ;
3. transmet le site au `PublicWeatherService` ;
4. récupère les données météorologiques ;
5. journalise la synchronisation ;
6. retourne les données au format JSON.

### Exemple de réponse

```json
{
    "data": {
        "...": "..."
    }
}
```

Le contenu exact de `data` dépend des informations retournées par `PublicWeatherService`.

### Site inexistant

Si aucun site ne correspond au `site_id fourni, l'API retourne une réponse `404`.

Exemple :

```json
{
    "error": "Site non trouvé, veuillez fournir le site_id dans le corp de la requête"
}
```

### Journalisation

Lorsqu'une récupération météorologique est effectuée, un événement :

```text
weather.synced
```

est enregistré.

Les informations suivantes sont notamment conservées :

* `site_id` ;
* `city`.

Cette journalisation permet de conserver une trace des opérations d'intégration avec le service météorologique.

### Point d'amélioration

Le endpoint utilise actuellement une requête `GET` avec `site_id` transmis dans le corps HTTP.

Une approche plus conventionnelle pour une API REST serait de transmettre ce paramètre dans la query string :

```text
GET /api/v1/external/weather?site_id=1
```

ou d'utiliser un paramètre de route :

```text
GET /api/v1/sites/{site}/weather
```

Cette amélioration pourrait être réalisée lors d'une évolution future de l'API.

---

# 8. Codes HTTP et gestion des erreurs

Les principaux codes HTTP susceptibles d'être retournés sont les suivants :

| Code  | Signification                              |
| ----- | ------------------------------------------ |
| `200` | Requête exécutée avec succès               |
| `201` | Ressource créée avec succès                |
| `400` | Requête invalide                           |
| `401` | Authentification absente ou token invalide |
| `403` | Accès refusé                               |
| `404` | Ressource ou endpoint inexistant           |
| `422` | Données fournies invalides                 |
| `429` | Trop nombreuses requêtes                   |
| `500` | Erreur interne de l'application            |
| `503` | Service temporairement indisponible        |

### Exemple d'erreur d'authentification

Lorsqu'un endpoint protégé est appelé sans token valide, l'accès est refusé.

Exemple :

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json
```

---

## 8.1 Erreur de site inexistant

L'endpoint météo retourne explicitement une erreur `404` lorsqu'un site demandé n'existe pas.

```json
{
    "error": "Site non trouvé, veuillez fournir le site_id dans le corp de la requête"
}
```

---

## 8.2 Erreur de validation

Les endpoints de création et de modification des tickets utilisent respectivement :

```text
StoreTicketRequest
UpdateTicketRequest
```

Les données invalides sont susceptibles de produire une réponse HTTP `422 Unprocessable Entity`.

---

# 9. Journalisation et traçabilité

Certaines opérations de l'API sont journalisées grâce au service :

```text
EventLogService
```

Les événements actuellement identifiés comprennent notamment :

| Événement        | Opération                                   |
| ---------------- | ------------------------------------------- |
| `tickets.index`  | Consultation de la liste des tickets        |
| `ticket.created` | Création d'un ticket                        |
| `ticket.updated` | Modification d'un ticket                    |
| `weather.synced` | Synchronisation des données météorologiques |

La journalisation permet de faciliter le diagnostic et la traçabilité des opérations réalisées via l'API.

---

# 10. Vérification de l'API

La liste des routes peut être vérifiée directement depuis l'application :

```bash
php artisan route:list --path=api
```

La suite de tests automatisés peut être exécutée avec :

```bash
php artisan test
```

Lors de l'épreuve, les tests disponibles ont retourné :

```text
Tests: 4 passed (6 assertions)
```

Ils comprennent notamment un contrôle de l'authentification de l'API des tickets.

L'endpoint de santé peut également être utilisé comme test de disponibilité :

```bash
curl -i \
  https://eval-dfs-p-tpl-20265-08.it-students.fr/api/health
```

---

# 11. Sécurité

Les mesures suivantes sont appliquées aux endpoints API :

* authentification par token pour les routes `/api/v1` ;
* utilisation de HTTPS en production ;
* limitation des données retournées par les contrôleurs ;
* validation des données de création et de modification des tickets ;
* journalisation des opérations importantes ;
* absence de secrets directement dans le code source ;
* utilisation d'un endpoint de santé distinct pour la supervision.

Les tokens d'authentification ne doivent jamais être communiqués dans les paramètres d'URL ou enregistrés dans le dépôt Git.

---

# 12. Évolution recommandée — Swagger / OpenAPI

La mise en place d'une documentation interactive basée sur **OpenAPI / Swagger** serait une amélioration pertinente pour l'application.

Elle permettrait notamment de :

* centraliser la documentation de l'ensemble des endpoints ;
* décrire précisément les paramètres et leurs types ;
* documenter les schémas JSON des requêtes et des réponses ;
* documenter le mécanisme d'authentification par token ;
* tester les endpoints directement depuis une interface interactive ;
* faciliter la prise en main de l'API par un futur développeur ou intégrateur.

Cette fonctionnalité n'a pas pu être mise en place pendant l'épreuve en raison du temps disponible.

La documentation actuelle a donc été réalisée manuellement à partir du code source et des routes réellement exposées par Laravel.

Une évolution future consisterait à intégrer une solution Swagger/OpenAPI et à maintenir sa spécification parallèlement aux évolutions de l'API.
