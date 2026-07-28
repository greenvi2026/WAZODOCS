---

# CHAPITRE 8 : CDR, Statistiques et Webhooks

## 8.1 Call Detail Records (CDR) — wazo-call-logd

Le service **wazo-call-logd** gère les enregistrements de détails d'appels (CDR). Il fournit l'historique complet de tous les appels passés par le système Wazo.

### 8.1.1 Architecture

Les CDR sont générés à partir des entrées **CEL** (Channel Event Log) d'Asterisk. Le service `wazo-call-logd` traite ces événements pour créer des enregistrements enrichis.

```
┌─────────────┐    CEL Events    ┌──────────────────┐
│  Asterisk   │ ───────────────► │  wazo-call-logd  │
└─────────────┘                  └────────┬─────────┘
                                           │
                                    ┌──────▼────────┐
                                    │   CDR Table   │
                                    │  (PostgreSQL) │
                                    └───────────────┘
```

### 8.1.2 Endpoints CDR

#### Liste des CDR (Admin)

```http
GET /api/call-logd/1.0/cdr
```

#### CDR de l'utilisateur authentifié

```http
GET /api/calld/1.0/users/me/cdr
```

#### CDR d'un utilisateur spécifique

```http
GET /api/calld/1.0/users/{user_uuid}/cdr
```

#### CDR unique par ID

```http
GET /api/call-logd/1.0/cdr/{cdr_id}
```

---

### 8.1.3 Filtres CDR — Guide Complet

#### Paramètres de Filtrage

| Paramètre | Type | Description |
|-----------|------|-------------|
| `from` | string | Date de début (ISO 8601: `2024-01-01T00:00:00`) |
| `until` | string | Date de fin (ISO 8601) |
| `limit` | integer | Nombre max de résultats (défaut: 50) |
| `offset` | integer | Décalage pour la pagination |
| `order` | string | Champ de tri (`date`, `duration`, `caller_id_number`) |
| `direction` | string | Ordre de tri (`asc` ou `desc`) |
| `call_direction` | string | `internal`, `inbound` ou `outbound` |
| `caller_id_name` | string | Nom de l'appelant |
| `caller_id_number` | string | Numéro de l'appelant |
| `destination_extension` | string | Extension de destination |
| `user_uuid` | string | UUID de l'utilisateur |
| `tags` | string | Tags personnalisés (séparés par virgule) |
| `from_id` | string | CDR ID de départ (pour pagination) |

---

### 8.1.4 Exemples de Filtres CDR

#### Filtrer par période (les 30 derniers jours)

```bash
curl -k -X GET \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com/api/call-logd/1.0/cdr?from=2024-01-01T00:00:00&until=2024-01-31T23:59:59"
```

#### Filtrer par utilisateur

```bash
curl -k -X GET \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com/api/call-logd/1.0/cdr?user_uuid=a1223fe6-bff8-4fb6-a982-f9157dea5094"
```

#### Filtrer appels entrants

```bash
curl -k -X GET \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com/api/call-logd/1.0/cdr?call_direction=inbound"
```

#### Filtrer par numéro appelant

```bash
curl -k -X GET \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com/api/call-logd/1.0/cdr?caller_id_number=1001"
```

#### Filtrer par tags (département)

```bash
curl -k -X GET \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com/api/call-logd/1.0/cdr?tags=sales"
```

#### Combinaison de filtres

```bash
curl -k -X GET \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com/api/call-logd/1.0/cdr?from=2024-01-01&until=2024-01-31&call_direction=outbound&limit=100&order=date&direction=desc"
```

---

### 8.1.5 Format de Réponse CDR

```json
{
  "items": [
    {
      "id": "cdr_uuid_123",
      "date": "2024-01-15T14:30:00+00:00",
      "date_answer": "2024-01-15T14:30:05+00:00",
      "date_end": "2024-01-15T14:35:00+00:00",
      "direction": "internal",
      "caller_id_name": "John Doe",
      "caller_id_number": "1001",
      "destination_extension": "2001",
      "destination_name": "Jane Doe",
      "destination_number": "2001",
      "user_uuid": "a1223fe6-bff8-4fb6-a982-f9157dea5094",
      "ended_by": null,
      "duration": 295,
      "billable": true,
      "tags": ["sales", "france"]
    }
  ],
  "total": 150,
  "filtered": 25
}
```

#### Description des Champs

| Champ | Type | Description |
|-------|------|-------------|
| `id` | string | UUID unique du CDR |
| `date` | datetime | Date de création de l'appel |
| `date_answer` | datetime | Date de réponse (null si non répondu) |
| `date_end` | datetime | Date de fin d'appel |
| `direction` | string | `internal`, `inbound`, `outbound` |
| `caller_id_name` | string | Nom affiché de l'appelant |
| `caller_id_number` | string | Numéro de l'appelant |
| `destination_extension` | string | Extension de destination |
| `destination_name` | string | Nom du destinataire |
| `destination_number` | string | Numéro du destinataire |
| `user_uuid` | string | UUID de l'utilisateur Wazo |
| `duration` | integer | Durée totale en secondes |
| `billable` | boolean | Indique si l'appel est facturable |
| `tags` | array | Tags personnalisés |

---

### 8.1.6 Exporter en CSV

Pour obtenir les CDR au format CSV, ajouter le header `Accept: text/csv`:

```bash
curl -k -X GET \
  -H "X-Auth-Token: *** \
  -H "Accept: text/csv; charset=utf-8" \
  "https://wazo.example.com/api/call-logd/1.0/cdr?from=2024-01-01&until=2024-01-31" -o cdr_export.csv
```

---

### 8.1.7 Régénération des CDR

Si des CDR sont manquants, ils peuvent être regénérés:

```bash
# Supprimer les CDR des 30 derniers jours
xivo-call-logs delete -d 30

# Régénérer les CDR
xivo-call-logs generate -d 30

# Avec un nombre spécifique de CEL
xivo-call-logs -c 100000
```

---

## 8.2 Statistiques des Files d'Attente

Les statistiques des queues sont stockées dans la base de données Asterisk et peuvent être interrogées via l'API.

### 8.2.1 Tables de Statistiques

| Table | Description |
|-------|-------------|
| `queue_log` | Log brut des événements de queue |
| `stat_call_on_queue` | Statistiques par appel |
| `stat_queue_periodic` | Agrégations périodiques (par heure) |

### 8.2.2 Métriques Disponibles

#### Statistiques par Appel (`stat_call_on_queue`)

| Champ | Description |
|-------|-------------|
| `callid` | ID de l'appel (lien avec CDR) |
| `time` | Horodatage de l'appel |
| `ringtime` | Durée de sonnerie (secondes) |
| `talktime` | Durée de conversation (secondes) |
| `waittime` | Temps d'attente (secondes) |
| `status` | Statut de l'appel |
| `queue_id` | ID de la queue |
| `agent_id` | ID de l'agent qui a répondu |

#### Statut des Appels

| Statut | Description |
|--------|-------------|
| `answered` | Appel répondu par un agent |
| `abandoned` | Appel abandonné par l'appelant |
| `full` | Appel rejeté car queue pleine |
| `closed` | Appel rejeté car queue fermée |
| `joinempty` | Appel rejeté car aucun agent disponible |
| `leaveempty` | Appel laissé car plus d'agents disponibles |
| `divert_ca_ratio` | Appels divertis selon ratio agents/appels |
| `divert_waittime` | Appels divertis selon temps d'attente |
| `timeout` | Appels ayant expiré le timeout |

#### Agrégations (`stat_queue_periodic`)

| Champ | Description |
|-------|-------------|
| `time` | Période (granularité: heure) |
| `answered` | Nombre d'appels répondus |
| `abandoned` | Nombre d'appels abandonnés |
| `total` | Total des appels reçus |
| `full` | Appels rejetés queue pleine |
| `closed` | Appels rejetés queue fermée |
| `joinempty` | Appels rejeés aucun agent |
| `leaveempty` | Appels diverted agents indisponibles |
| `divert_ca_ratio` | Divertis ratio agents |
| `divert_waittime` | Divertis temps d'attente |
| `timeout` | Appels expirés |

---

## 8.3 Webhooks — wazo-webhookd

Le service **wazo-webhookd** permet de créer des abonnements qui déclenchent des HTTP callbacks lorsque des événements Wazo se produisent.

### 8.3.1 Architecture

```
┌─────────────┐    Events     ┌──────────────────┐    HTTP POST    ┌──────────────┐
│  Wazo Bus   │ ────────────► │  wazo-webhookd   │ ──────────────► │  External    │
│  (RabbitMQ) │               │  (Subscription)  │                 │  Server      │
└─────────────┘               └──────────────────┘                 └──────────────┘
```

### 8.3.2 Créer une Subscription

#### Endpoint

```http
POST /api/webhookd/1.0/subscriptions
```

#### Headers

```http
X-Auth-Token: ***
Content-Type: application/json
Wazo-Tenant: {tenant_uuid}
```

#### Payload

```json
{
  "name": "My Webhook",
  "events": ["call_created", "user_created"],
  "service": "http",
  "config": {
    "url": "https://my-server.com/webhook",
    "method": "POST",
    "timeout": 30
  }
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `name` | string | **Oui** | Nom de la subscription |
| `events` | array | **Oui** | Liste des événements |
| `service` | string | **Oui** | Service handler (`http`, `example`) |
| `config` | object | **Oui** | Configuration du service |
| `config.url` | string | **Oui** | URL de callback |
| `config.method` | string | Non | Méthode HTTP (`GET`, `POST`, `PUT`) |
|| `config.timeout` | `integer` | Non | Timeout en secondes (⚠️ non supporté Wazo 26.06, omettre) |
| `user_uuid` | string | Non | Filtrer par utilisateur |

#### Exemple cURL

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "CRM Integration",
    "events": ["call_created", "call_ended"],
    "service": "http",
    "config": {
      "url": "https://crm.example.com/wazo-events",
      "method": "POST"
    }
  }' \
  https://wazo.example.com/api/webhookd/1.0/subscriptions
```

#### Réponse (201 Created)

```json
{
  "uuid": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
  "name": "CRM Integration",
  "events": ["call_created", "call_ended"],
  "service": "http",
  "config": {
    "url": "https://crm.example.com/wazo-events",
    "method": "POST"
  },
  "enabled": true,
  "owner_uuid": "admin_uuid",
  "tenant_uuid": "tenant_uuid"
}
```

---

### 8.3.3 Liste des Subscriptions

```http
GET /api/webhookd/1.0/subscriptions
```

#### Réponse

```json
{
  "items": [
    {
      "uuid": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
      "name": "CRM Integration",
      "events": ["call_created"],
      "service": "http",
      "config": {
        "url": "https://crm.example.com/wazo-events",
        "method": "POST"
      },
      "enabled": true
    }
  ]
}
```

---

### 8.3.4 Mettre à Jour une Subscription

```http
PUT /api/webhookd/1.0/subscriptions/{subscription_id}
```

#### Payload

```json
{
  "name": "Updated Name",
  "enabled": false,
  "events": ["call_created", "user_created"]
}
```

---

### 8.3.5 Supprimer une Subscription

```http
DELETE /api/webhookd/1.0/subscriptions/{subscription_id}
```

---

### 8.3.6 Événements Disponibles pour Webhooks

#### Événements d'Appels

| Événement | Description |
|-----------|-------------|
| `call_created` | Appel créé |
| `call_updated` | Appel mis à jour |
| `call_ended` | Appel terminé |
| `call_answered` | Appel répondu |
| `call_rotute_noanswer` | Appel non répondu |

#### Événements Utilisateurs

| Événement | Description |
|-----------|-------------|
| `user_created` | Utilisateur créé |
| `user_updated` | Utilisateur modifié |
| `user_deleted` | Utilisateur supprimé |
| `user_status_update` | Statut utilisateur changé |
| `user_voicemail_message_created` | Nouveau message voicemail |
| `user_voicemail_message_updated` | Message voicemail modifié |
| `user_voicemail_message_deleted` | Message voicemail supprimé |

#### Événements de Services

| Événement | Description |
|-----------|-------------|
| `users_services_dnd_updated` | DND activé/désactivé |
| `users_services_incallfilter_updated` | Filtre d'appel modifié |
| `users_forwards_unconditional_updated` | Renvoi inconditionnel |
| `users_forwards_busy_updated` | Renvoi sur occupation |
| `users_forwards_noanswer_updated` | Renvoi sur non-réponse |

#### Événements d'Agents

| Événement | Description |
|-----------|-------------|
| `agent_status_update` | Agent login/logout |
| `agent_paused` | Agent en pause |
| `agent_unpaused` | Agent reprend |

#### Événements de Configuration

| Événement | Description |
|-----------|-------------|
| `endpoint_status_update` | Statut endpoint changé |
| `favorite_added` | Favori ajouté |
| `favorite_deleted` | Favori supprimé |
| `relocate_initiated` | Relocalisation initiée |
| `relocate_answered` | Relocalisation répondue |
| `relocate_completed` | Relocalisation terminée |
| `relocate_ended` | Relocalisation terminée |

---

### 8.3.7 Format des Payloads Webhook

#### call_created

```json
{
  "name": "call_created",
  "origin_uuid": "wazo_server_uuid",
  "data": {
    "call_id": "1455123422.8",
    "caller_id_name": "John Doe",
    "caller_id_number": "1001",
    "destination_extension": "2001",
    "user_uuid": "user_uuid_123",
    "direction": "internal",
    "status": "Ring"
  },
  "timestamp": "2024-01-15T14:30:00Z"
}
```

#### call_ended

```json
{
  "name": "call_ended",
  "origin_uuid": "wazo_server_uuid",
  "data": {
    "call_id": "1455123422.8",
    "caller_id_name": "John Doe",
    "caller_id_number": "1001",
    "destination_extension": "2001",
    "user_uuid": "user_uuid_123",
    "duration": 180,
    "hangup_cause": "NORMAL_CLEARING"
  },
  "timestamp": "2024-01-15T14:33:00Z"
}
```

#### user_created

```json
{
  "name": "user_created",
  "origin_uuid": "wazo_server_uuid",
  "data": {
    "uuid": "user_uuid_456",
    "firstname": "Jane",
    "lastname": "Doe",
    "email": "jane.doe@example.com",
    "username": "jdoe"
  },
  "timestamp": "2024-01-15T14:30:00Z"
}
```

#### user_status_update

```json
{
  "name": "user_status_update",
  "origin_uuid": "wazo_server_uuid",
  "data": {
    "user_uuid": "user_uuid_123",
    "status": "available",
    "presence": "online"
  },
  "timestamp": "2024-01-15T14:30:00Z"
}
```

#### agent_status_update

```json
{
  "name": "agent_status_update",
  "origin_uuid": "wazo_server_uuid",
  "data": {
    "agent_id": 42,
    "xivo_id": "wazo_server_uuid",
    "status": "logged_in"
  },
  "timestamp": "2024-01-15T14:30:00Z"
}
```

---

### 8.3.8 Filtrage par Utilisateur

Il est possible de filtrer les webhooks pour un utilisateur spécifique:

```json
{
  "name": "User-Specific Webhook",
  "events": ["call_created", "user_status_update"],
  "service": "http",
  "config": {
    "url": "https://my-server.com/webhook"
  },
  "user_uuid": "specific_user_uuid"
}
```

#### Événements supportés pour le filtrage par utilisateur

- `users_services_incallfilter_updated`
- `users_services_dnd_updated`
- `users_forwards_unconditional_updated`
- `users_forwards_noanswer_updated`
- `users_forwards_busy_updated`
- `user_voicemail_message_updated`
- `user_voicemail_message_deleted`
- `user_voicemail_message_created`
- `user_status_update`
- `relocate_ended`
- `relocate_completed`
- `relocate_answered`
- `relocate_initiated`
- `favorite_deleted`
- `favorite_added`
- `endpoint_status_update`
- `call_updated`
- `call_log_user_created`
- `call_ended`
- `call_created`
- `agent_unpaused`
- `agent_status_update`
- `agent_paused`

---

## 8.4 Patterns d'Intégration

### 8.4.1 Pattern : Export CDR vers CRM

```python
import requests

# Récupérer les CDR d'hier
from datetime import datetime, timedelta

yesterday = (datetime.now() - timedelta(days=1)).isoformat()
today = datetime.now().isoformat()

response = requests.get(
    "https://wazo.example.com/api/call-logd/1.0/cdr",
    headers={
        "X-Auth-Token": f"{token}",
        "Wazo-Tenant": tenant_uuid
    },
    params={
        "from": yesterday,
        "until": today,
        "limit": 1000
    }
)

for cdr in response.json()["items"]:
    # Envoyer au CRM
    requests.post(
        "https://crm.example.com/api/calls",
        json=cdr
    )
```

### 8.4.2 Pattern : Webhook de Notification

```python
from flask import Flask, request
import hmac
import hashlib

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def handle_webhook():
    # Vérifier la signature
    signature = request.headers.get('X-Wazo-Signature')
    if not verify_signature(request.data, signature):
        return "Invalid signature", 401
    
    event = request.json
    
    if event['name'] == 'call_created':
        print(f"Nouvel appel de {event['data']['caller_id_number']}")
    
    elif event['name'] == 'user_created':
        print(f"Nouvel utilisateur: {event['data']['email']}")
    
    return "OK", 200

def verify_signature(payload, signature):
    # Implémenter la vérification HMAC
    expected = hmac.new(
        b'secret_key',
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

### 8.4.3 Pattern : Statistiques en Temps Réel

```javascript
// Connexion WebSocket pour les stats temps réel
const ws = new WebSocket('wss://wazo.example.com:9502/?version=2&token=' + token);

ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);
    
    if (msg.op === 'event') {
        // Mettre à jour les compteurs
        switch (msg.event) {
            case 'call_created':
                stats.callsCreated++;
                break;
            case 'call_ended':
                stats.callsEnded++;
                updateDurationStats(msg.data.duration);
                break;
        }
        
        updateDashboard();
    }
};
```

---

## 8.5 Récapitulatif des Services

> **IMPORTANT** : Toutes les API passent par nginx sur le port 443. Les ports ci-dessous sont les ports directs des microservices (pour debugging uniquement).

| Service | Port Direct | Nginx Route | API Base | Purpose |
|---------|-------------|-------------|----------|---------|
| wazo-call-logd | 9500 | /api/call-logd/1.0/* | `/api/call-logd/1.0` | Call Detail Records |
| wazo-webhookd | 9300 | /api/webhookd/1.0/* | `/api/webhookd/1.0` | Webhook subscriptions |
| wazo-websocketd | 9502 | /api/websocketd/* | WebSocket | Real-time events |
| wazo-calld | 9500 | /api/calld/1.0/* | `/api/calld/1.0` | Call control |

---

*Fin du Chapitre 8 — Fin de l'Ouvrage*
