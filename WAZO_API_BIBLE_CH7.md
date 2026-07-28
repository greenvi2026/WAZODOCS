---

# CHAPITRE 7 : CTI, WebSockets et Temps Réel

## 7.1 Architecture des Services Temps Réel

Wazo propose plusieurs services pour la gestion temps réel des communications :

| Service | Port | Rôle |
|---------|------|------|
| **wazo-calld** | 8668 | Contrôle des appels, transferts, applications |
| **wazo-websocketd** | 9502 | Events temps réel via WebSocket |
| **wazo-chatd** | 9504 | Présence et messagerie (deprecated) |
| **wazo-agentd** | 9503 | Gestion des agents ACD |

> **⚠️ Important** : Tous les services temps réel nécessitent un token d'authentification valide avec les ACL appropriées.

---

## 7.2 wazo-calld — Contrôle des Appels

Le service **wazo-calld** est le micro-service de contrôle d'appels pour les cas d'usage de communication unifiée. Il gère :

- Création d'appels sortants
- Antworting/hangup d'appels entrants
- Transferts (aveugle et assisté)
- Enregistrement d'appels
- Voicemails
- Applications (conférences, bridges)

### 7.2.1 Liste des Appels Utilisateur

Récupère la liste des appels actifs pour l'utilisateur authentifié.

#### Endpoint

```http
GET /api/calld/1.0/users/me/calls
```

#### Headers

```http
X-Auth-Token: ***
Wazo-Tenant: {tenant_uuid}    # Optionnel si multi-tenant
```

#### Réponse

```json
{
  "items": [
    {
      "answer_time": "2019-08-24T14:15:22Z",
      "bridges": ["bridge_id_123"],
      "call_id": "1455123422.8",
      "caller_id_name": "John Doe",
      "caller_id_number": "1001",
      "conversation_id": "conv_abc123",
      "creation_time": "2019-08-24T14:15:00Z",
      "dialed_extension": "2001",
      "direction": "internal",
      "hangup_time": null,
      "is_caller": true,
      "is_video": false,
      "line_id": 12,
      "muted": false,
      "on_hold": false,
      "parked": false,
      "peer_caller_id_name": "Jane Doe",
      "peer_caller_id_number": "2001",
      "record_state": "inactive",
      "sip_call_id": "abc123def456@192.168.1.100",
      "status": "Up",
      "talking_to": {
        "1455123423.9": "Jane Doe"
      },
      "user_uuid": "a1223fe6-bff8-4fb6-a982-f9157dea5094"
    }
  ]
}
```

#### Codes HTTP

| Code | Description |
|------|-------------|
| 200 | Succès |
| 401 | Token invalide ou expiré |
| 503 | Service indisponible |

---

### 7.2.2 Créer un Appel Sortant (depuis utilisateur)

Initie un nouvel appel depuis l'utilisateur authentifié.

#### Endpoint

```http
POST /api/calld/1.0/users/me/calls
```

#### Headers

```http
X-Auth-Token: ***
Content-Type: application/json
Wazo-Tenant: {tenant_uuid}    # Optionnel
```

#### Payload

```json
{
  "extension": "2001",
  "line_id": 12,
  "auto_answer_caller": false,
  "from_mobile": false,
  "all_lines": false,
  "variables": {}
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `extension` | string | **Oui** | Extension à appeler |
| `line_id` | integer | Non | ID de la ligne à utiliser (défaut: ligne principale) |
| `auto_answer_caller` | boolean | Non | Force le téléphone à répondre automatiquement |
| `from_mobile` | Non | Non | Appeler depuis le mobile |
| `all_lines` | boolean | Non | Utiliser toutes les lignes de l'utilisateur |
| `variables` | object | Non | Variables de canal Asterisk |

#### Exemple cURL

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "extension": "2001",
    "line_id": 12
  }' \
  https://wazo.example.com/api/calld/1.0/users/me/calls
```

#### Réponse (201 Created)

```json
{
  "answer_time": null,
  "bridges": [],
  "call_id": "1455123422.8",
  "caller_id_name": "John Doe",
  "caller_id_number": "1001",
  "conversation_id": null,
  "creation_time": "2019-08-24T14:15:00Z",
  "dialed_extension": "2001",
  "direction": "internal",
  "hangup_time": null,
  "is_caller": true,
  "is_video": false,
  "line_id": 12,
  "muted": false,
  "on_hold": false,
  "parked": false,
  "peer_caller_id_name": "",
  "peer_caller_id_number": "",
  "record_state": "inactive",
  "sip_call_id": null,
  "status": "Ring",
  "talking_to": {},
  "user_uuid": "a1223fe6-bff8-4fb6-a982-f9157dea5094"
}
```

#### Codes HTTP

| Code | Description |
|------|-------------|
| 201 | Appel créé avec succès |
| 400 | Requête invalide |
| 503 | Service unavailable |

---

### 7.2.3 Créer un Appel (Admin - Destination arbitraire)

Crée un appel depuis une source spécifiée vers une destination arbitraire. Requiert l'ACL `calld.calls.create`.

#### Endpoint

```http
POST /api/calld/1.0/calls
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
  "source": {
    "user": "user_uuid_ou_numero",
    "line_id": 12,
    "auto_answer": false,
    "from_mobile": false,
    "all_lines": false
  },
  "destination": {
    "context": "default",
    "extension": "2001",
    "priority": 1
  },
  "variables": {}
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `source.user` | string | **Oui** | UUID utilisateur ou numéro |
| `source.line_id` | integer | Non | ID de ligne spécifique |
| `source.auto_answer` | boolean | Non | Réponse automatique |
| `destination.context` | string | **Oui** | Contexte de destination |
| `destination.extension` | string | **Oui** | Extension de destination |
| `destination.priority` | integer | Non | Priorité Asterisk (défaut: 1) |

#### Exemple cURL

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "source": {
      "user": "a1223fe6-bff8-4fb6-a982-f9157dea5094",
      "line_id": 12
    },
    "destination": {
      "context": "default",
      "extension": "2001"
    }
  }' \
  https://wazo.example.com/api/calld/1.0/calls
```

---

### 7.2.4 Répondre à un Appel Entrant

Répond à un appel entrant pour l'utilisateur authentifié.

#### Endpoint

```http
PUT /api/calld/1.0/users/me/calls/{call_id}/answer
```

| Paramètre | Type | Description |
|-----------|------|-------------|
| `call_id` | string | ID de l'appel à répondre |

#### Headers

```http
X-Auth-Token: ***
Wazo-Tenant: {tenant_uuid}
```

#### Exemple cURL

```bash
curl -k -X PUT \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  https://wazo.example.com/api/calld/1.0/users/me/calls/1455123422.8/answer
```

#### Réponse

- **204 No Content** : Appel répondu avec succès
- **404** : Appel non trouvé
- **403** : Pas autorisé à répondre à cet appel

---

### 7.2.5 Raccrocher un Appel

Raccroche un appel actif.

#### Endpoint

```http
DELETE /api/calld/1.0/users/me/calls/{call_id}
```

#### Headers

```http
X-Auth-Token: ***
Wazo-Tenant: {tenant_uuid}
```

#### Exemple cURL

```bash
curl -k -X DELETE \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  https://wazo.example.com/api/calld/1.0/users/me/calls/1455123422.8
```

#### Codes HTTP

| Code | Description |
|------|-------------|
| 204 | Appel raccroché |
| 403 | Pas autorisé |
| 404 | Appel non trouvé |
| 503 | Service unavailable |

---

### 7.2.6 Initier un Transfert

Transfère un appel vers une autre extension. Supporte les transferts aveugle et assisté.

#### Endpoint

```http
POST /api/calld/1.0/transfers
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
  "initiator_call": "1455123422.8",
  "transferred_call": "1455123423.9",
  "exten": "2001",
  "context": "default",
  "flow": "attended",
  "timeout": 30,
  "variables": {}
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `initiator_call` | string | **Oui** | ID de l'appel initiateur (celui qui déclenche le transfert) |
| `transferred_call` | string | **Oui** | ID de l'appel transféré |
| `exten` | string | **Oui** | Extension du destinataire |
| `context` | string | **Oui** | Contexte du destinataire |
| `flow` | string | Non | `attended` (assisté) ou `blind` (aveugle). Défaut: `attended` |
| `timeout` | integer | Non | Timeout en secondes (défaut: illimité) |
| `variables` | object | Non | Variables de canal |

#### Exemple cURL (Transfert assisté)

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "initiator_call": "1455123422.8",
    "transferred_call": "1455123423.9",
    "exten": "2001",
    "context": "default",
    "flow": "attended",
    "timeout": 30
  }' \
  https://wazo.example.com/api/calld/1.0/transfers
```

#### Réponse (201 Created)

```json
{
  "flow": "attended",
  "id": "transfer_abc123",
  "initiator_call": "1455123422.8",
  "initiator_tenant_uuid": "tenant_xyz",
  "initiator_uuid": "a1223fe6-bff8-4fb6-a982-f9157dea5094",
  "recipient_call": "1455123424.0",
  "status": "starting",
  "transferred_call": "1455123423.9"
}
```

#### Statuts de Transfert

| Statut | Description |
|--------|-------------|
| `starting` | Transfert en cours d'initialisation |
| `attended` | Transfert assisté en cours (initiateur en attente) |
| `completed` | Transfert terminé avec succès |
| `canceled` | Transfert annulé |
| `failed` | Échec du transfert |

#### Annuler un Transfert

```http
DELETE /api/calld/1.0/transfers/{transfer_id}
```

---

### 7.2.7 Enregistrement d'Appel

Démarre/arrête/marquepause l'enregistrement d'un appel.

#### Démarrer l'Enregistrement

```http
PUT /api/calld/1.0/users/me/calls/{call_id}/record/start
```

#### Arrêter l'Enregistrement

```http
PUT /api/calld/1.0/users/me/calls/{call_id}/record/stop
```

#### Pause/Resume

```http
PUT /api/calld/1.0/users/me/calls/{call_id}/record/pause
PUT /api/calld/1.0/users/me/calls/{call_id}/record/resume
```

#### Exemple cURL

```bash
# Démarrer l'enregistrement
curl -k -X PUT \
  -H "X-Auth-Token: *** \
  https://wazo.example.com/api/calld/1.0/users/me/calls/1455123422.8/record/start

# Arrêter l'enregistrement
curl -k -X PUT \
  -H "X-Auth-Token: *** \
  https://wazo.example.com/api/calld/1.0/users/me/calls/1455123422.8/record/stop
```

> **⚠️ Important** : L'enregistrement nécessite que la fonctionnalité soit activée sur l'utilisateur, la queue ou le groupe.

---

## 7.3 Services Utilisateur (DND, forwards, incallfilter)

Les services utilisateur permettent de gérer les fonctionnalités de confort via l'API REST. Ces endpoints sont gérés par **wazo-confd**.

### 7.3.1 Do Not Disturb (DND)

Active ou désactive le mode Ne Pas Déranger pour un utilisateur.

#### Activer DND

```http
PUT /api/confd/1.1/users/{user_uuid}/services/dnd/enable
```

#### Désactiver DND

```http
PUT /api/confd/1.1/users/{user_uuid}/services/dnd/disable
```

#### Headers

```http
X-Auth-Token: ***
Wazo-Tenant: {tenant_uuid}
```

#### Exemple cURL

```bash
# Activer DND
curl -k -X PUT \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  https://wazo.example.com/api/confd/1.1/users/a1223fe6-bff8-4fb6-a982-f9157dea5094/services/dnd/enable

# Désactiver DND
curl -k -X PUT \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  https://wazo.example.com/api/confd/1.1/users/a1223fe6-bff8-fb6-a982-f9157dea5094/services/dnd/disable
```

#### Réponse

```json
{
  "enabled": true
}
```

> **Note** : Depuis XiVO 16.13, le DND utilisateur est effectif indépendamment de l'extension DND (*25).

---

### 7.3.2 Filtre d'Appel Entrant (In-Call Filter)

Active ou désactive le filtrage des appels entrants.

#### Activer le Filtre

```http
PUT /api/confd/1.1/users/{user_uuid}/services/incallfilter/enable
```

#### Désactiver le Filtre

```http
PUT /api/confd/1.1/users/{user_uuid}/services/incallfilter/disable
```

#### Exemple cURL

```bash
curl -k -X PUT \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  https://wazo.example.com/api/confd/1.1/users/a1223fe6-bff8-4fb6-a982-f9157dea5094/services/incallfilter/enable
```

---

### 7.3.3 Renvois d'Appels (Forwards)

Gère les renvois d'appels : inconditionnel, sur occupation, sur non-réponse.

#### Liste des Types de Renvoi

| Type | Description |
|------|-------------|
| `unconditional` | Renvoi inconditionnel |
| `busy` | Renvoi si occupé |
| `noanswer` | Renvoi si pas de réponse |

####Configurer un Renvoi

```http
PUT /api/confd/1.1/users/{user_uuid}/forwards/{forward_type}
```

#### Payload

```json
{
  "enabled": true,
  "destination": "2001"
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `enabled` | boolean | Active/désactive le renvoi |
| `destination` | string | Extension de destination |

#### Exemple cURL (Renvoi sur occupation)

```bash
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: *** \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "enabled": true,
    "destination": "2001"
  }' \
  https://wazo.example.com/api/confd/1.1/users/a1223fe6-bff8-4fb6-a982-f9157dea5094/forwards/busy
```

#### Récupérer les Renvois

```http
GET /api/confd/1.1/users/{user_uuid}/forwards/{forward_type}
```

#### Réponse

```json
{
  "enabled": true,
  "destination": "2001"
}
```

---

## 7.4 WebSocket — Events Temps Réel

Le service **wazo-websocketd** permet de recevoir les événements Wazo en temps réel via une connexion WebSocket. C'est le canal privilégié pour les applications CTI et les tableaux de bord temps réel.

### 7.4.1 Connexion WebSocket

#### Endpoint

```
wss://{wazo_host}:9502/?version=2&token={auth_token}
```

| Paramètre | Type | Description |
|-----------|------|-------------|
| `version` | integer | Version de l'API (2) |
| `token` | string | Token d'authentification Wazo (doit avoir l'ACL `websocketd`) |

#### Exemple JavaScript

```javascript
var socket = new WebSocket("wss://wazo.example.com:9502/?version=2&token=" + token);

socket.onmessage = function(event) {
    var msg = JSON.parse(event.data);
    switch (msg.op) {
        case "init":
            // Connexion établie, s'abonner aux événements
            subscribe("call_created");
            subscribe("call_updated");
            start();
            break;
        case "start":
            console.log("En attente d'événements...");
            break;
        case "event":
            console.log("Événement reçu:", msg.event, msg.data);
            break;
    }
};

function subscribe(eventName) {
    socket.send(JSON.stringify({
        op: "subscribe",
        data: {
            event_name: eventName
        }
    }));
}

function start() {
    socket.send(JSON.stringify({
        op: "start"
    }));
}
```

#### Message du Serveur — Init

```json
{
  "op": "init",
  "code": 0,
  "data": {
    "version": 2
  }
}
```

#### Codes d'Erreur WebSocket

| Code | Description |
|------|-------------|
| 0 | Succès |
| 4003 | Token expiré |

---

### 7.4.2 Opérations WebSocket

#### Subscribe — S'abonner aux Événements

```json
{
  "op": "subscribe",
  "data": {
    "event_name": "call_created"
  }
}
```

Pour s'abonner à tous les événements :

```json
{
  "op": "subscribe",
  "data": {
    "event_name": "*"
  }
}
```

#### Start — Commencer la Réception

```json
{
  "op": "start"
}
```

#### Réponse du Serveur

```json
{
  "op": "subscribe",
  "code": 0
}
```

---

### 7.4.3 Événements Disponibles

#### Événements d'Appels

| Événement | Description |
|-----------|-------------|
| `call_created` | Nouvel appel créé |
| `call_updated` | Statut d'appel modifié |
| `call_ended` | Appel terminé |
| `call_held` | Appel en attente |
| `call_resumed` | Appel repris |

#### Exemple — call_created

```json
{
  "name": "call_created",
  "origin_uuid": "08c56466-8f29-45c7-9856-92bf1ba89b82",
  "data": {
    "bridges": [],
    "call_id": "1455123422.8",
    "caller_id_name": "Some One",
    "caller_id_number": "1001",
    "creation_time": "2016-02-10T11:57:02.592-0500",
    "status": "Ring",
    "talking_to": {},
    "user_uuid": "2e752722-0864-4665-887d-a78a024cf7c7"
  }
}
```

#### Événements de Services Utilisateur

| Événement | Description |
|-----------|-------------|
| `users_services_dnd_updated` | DND modifié |
| `users_services_incallfilter_updated` | Filtre d'appel modifié |
| `users_forwards_unconditional_updated` | Renvoi inconditionnel modifié |
| `users_forwards_busy_updated` | Renvoi sur occupation modifié |
| `users_forwards_noanswer_updated` | Renvoi sur non-réponse modifié |

#### Exemple — users_services_dnd_updated

```json
{
  "name": "users_services_dnd_updated",
  "required_acl": "events.config.users.a1223fe6-bff8-4fb6-a982-f9157dea5094.services.dnd.updated",
  "origin_uuid": "ca7f87e9-c2c8-5fad-baba1b-c3140ebb9be3",
  "data": {
    "user_uuid": "a1223fe6-bff8-4fb6-a982-f9157dea5094",
    "enabled": true
  }
}
```

#### Événements d'Agents

| Événement | Description |
|-----------|-------------|
| `agent_status_update` | Statut agent modifié (login/logout/pause) |

#### Exemple — agent_status_update

```json
{
  "name": "agent_status_update",
  "required_acl": "events.statuses.agents",
  "origin_uuid": "ca7f87e9-c2c8-5fad-ba1b-c3140ebb9be3",
  "data": {
    "agent_id": 42,
    "xivo_id": "ca7f87e9-c2c8-5fad-ba1b-c3140ebb9be3",
    "status": "logged_in"
  }
}
```

---

## 7.5 Switchboard — standard automatique

Le switchboard permet aux opérateurs de gérer les appels entrants avec des fonctionnalités de mise en attente, transfert et interception.

### 7.5.1 Récupérer les Appels en Attente

```http
GET /api/calld/1.0/switchboard/waits
```

#### Headers

```http
X-Auth-Token: ***
Wazo-Tenant: {tenant_uuid}
```

#### Réponse

```json
{
  "items": [
    {
      "call_id": "1455123422.8",
      "caller_id_name": "Caller Name",
      "caller_id_number": "1001",
      "conversation_id": null,
      "creation_time": "2019-08-24T14:15:00Z",
      "dialed_extension": "4000",
      "line_id": 15,
      "status": "Ring",
      "user_uuid": "operator_uuid"
    }
  ]
}
```

---

### 7.5.2 Mettre en Attente

```http
PUT /api/calld/1.0/switchboard/{call_id}/hold
```

---

### 7.5.3 Reprendre depuis Attente

```http
PUT /api/calld/1.0/switchboard/{call_id}/retrieve
```

---

### 7.5.4 Rediriger vers File d'Attente

```http
PUT /api/calld/1.0/switchboard/{call_id}/redirect-queue/{queue_id}
```

---

## 7.6 Applications — Conférencing et Bridges

Les applications permettent de créer des conférences et des bridges audio complexes.

### 7.6.1 Créer un Appel dans une Application

```http
POST /api/calld/1.0/applications/{application_uuid}/calls
```

#### Payload

```json
{
  "context": "default",
  "exten": "1001",
  "autoanswer": false,
  "displayed_caller_id_name": "Conference",
  "displayed_caller_id_number": "3000"
}
```

#### Exemple cURL

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: *** \
  -d '{
    "context": "default",
    "exten": "1001",
    "autoanswer": true
  }' \
  https://wazo.example.com/api/calld/1.0/applications/app_uuid_123/calls
```

---

## 7.7 Récapitulatif des ACLs Requises

| Action | ACL Requise |
|--------|-------------|
| Liste appels utilisateur | `calld.users.me.calls.read` |
| Créer appel utilisateur | `calld.users.me.calls.create` |
| Répondre à un appel | `calld.users.me.calls.{call_id}.answer` |
| Raccrocher un appel | `calld.users.me.calls.{call_id}.delete` |
| Démarrer enregistrement | `calld.users.me.calls.{call_id}.record.start` |
| Créer transfert | `calld.transfers.create` |
| Liste services utilisateur | `confd.users.{user_uuid}.services.read` |
| Modifier DND | `confd.users.{user_uuid}.services.dnd.update` |
| Modifier forwards | `confd.users.{user_uuid}.forwards.{type}.update` |
| WebSocket | `websocketd` |

---

## 7.8 Patterns d'Intégration CTI

### 7.8.1 Pattern : Supervision d'Appels

1. Se connecter au WebSocket avec un token ayant `websocketd`
2. S'abonner aux événements `call_created`, `call_updated`, `call_ended`
3. Maintenir un état local des appels actifs
4. Utiliser l'API REST pour les actions (answer, hold, transfer)

### 7.8.2 Pattern : Clic-to-Call

1. UI web/un client génère un clic sur un numéro
2. Appeler `POST /api/calld/1.0/users/me/calls` avec l'extension
3. Sur réception de `call_answered`, notifier l'utilisateur
4. Afficher le statut de l'appel en temps réel

### 7.8.3 Pattern : Supervision Présence

1. S'abonner aux événements `users_services_dnd_updated`, `users_forwards_*_updated`
2. Mettre à jour l'interface utilisateur en temps réel
3. Afficher le statut DND/Forward pour chaque utilisateur

---

*Fin du Chapitre 7*
