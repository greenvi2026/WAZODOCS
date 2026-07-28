---

# PARTIE 4 : Temps Réel, WebRTC & Conférence

Cette partie couvre les workflows de communication en temps réel, les conférences audio/vidéo, les appels WebRTC, et les fonctionnalités de présence et de messagerie instantanée. Ces scénarios permettent d'exploiter les capacités avancées de l'écosystème Wazo pour la collaboration moderne.

---

## 4.1 Transfert d'Appel (Attended Transfer)

### Objectif

Effectuer un transfert attended (avec annonce) où l'initiateur met l'appel original en attente, appelle le destinataire du transfert, et connecte les deux parties. Ce scénario est essentiel pour les standardistes et les assistantes qui doivent presenter un appel avant de le transferer.

### Services impliqués

- **wazo-calld** : Gestion des appels actifs et des transfert
- **wazo-confd** : Lecture des utilisateurs et lignes
- **wazo-websocketd** : Notifications temps réel

### Le Workflow détaillé

#### Étape 1 : Initier le premier appel (Appelant → Intermédiaire)

```http
POST /api/calld/1.0/calls
Content-Type: application/json
X-Auth-Token: {intermediaire_token}
```

**Payload :**
```json
{
  "extension": "1001",
  "line_id": 5,
  "context": "default"
}
```

**Reponse :**
```json
{
  "call_id": "call-abc123-def456",
  "status": "ringing",
  "peer_caller_id_number": "1002"
}
```

> **🔗 Chaînage** : Stockez le `call_id` de l'appel en cours — il sera utilisé pour le transfert. Stockez dans **CALL_ID_ORIGINAL**.

#### Étape 2 : Mettre l'appel en attente

```http
PUT /api/calld/1.0/calls/{call_id_original}/hold
X-Auth-Token: {intermediaire_token}
```

**Reponse :**
```json
{
  "call_id": "call-abc123-def456",
  "status": "hold"
}
```

#### Étape 3 : Appeler le destinataire du transfert

```http
POST /api/calld/1.0/calls
Content-Type: application/json
X-Auth-Token: {intermediaire_token}
```

**Payload :**
```json
{
  "extension": "1003",
  "line_id": 5,
  "context": "default"
}
```

**Reponse :**
```json
{
  "call_id": "call-xyz789-uvw012",
  "status": "ringing",
  "peer_caller_id_number": "1003"
}
```

> **🔗 Chaînage** : Stockez ce nouveau `call_id` — il devient **CALL_ID_TRANSFERT**.

#### Étape 4 : Effectuer le transfert attended

```http
PUT /api/calld/1.0/calls/{call_id_original}/transfer
Content-Type: application/json
X-Auth-Token: {intermediaire_token}
```

**Payload :**
```json
{
  "transferee_call_id": "call-xyz789-uvw012",
  "extension": "1003",
  "context": "default"
}
```

**Reponse :**
```json
{
  "call_id": "call-abc123-def456",
  "status": "bridged",
  "transferee_call_id": "call-xyz789-uvw012"
}
```

### Avertissements

- **Ordre critique** : Le transfert attended necessite que l'appel original soit mis en attente AVANT d'appeler le destinataire
- **Timeouts** : Si le destinataire ne répond pas dans 30 secondes, l'appel est renvoyé vers l'intermédiaire
- **Droit de transfert** : L'utilisateur doit avoir le droit `transfer` dans sa configuration de ligne

---

## 4.2 Transfert d'Appel (Blind Transfer)

### Objectif

Effectuer un transfert direct sans annonce, où l'appel est immediatement envoye vers le destinataire. Ce scenario est plus rapide que le transfert attended mais ne permet pas de verifier si le destinataire est disponible.

### Services impliqués

- **wazo-calld** : Gestion des appels actifs et des transferts

### Le Workflow détaillé

#### Étape 1 : Initier l'appel original

```http
POST /api/calld/1.0/calls
Content-Type: application/json
X-Auth-Token: {user_token}
```

**Payload :**
```json
{
  "extension": "1001",
  "line_id": 5,
  "context": "default"
}
```

**Reponse :**
```json
{
  "call_id": "call-blind001",
  "status": "ringing"
}
```

> **🔗 Chaînage** : Stockez **CALL_ID**.

#### Étape 2 : Effectuer le transfert direct (blind)

```http
PUT /api/calld/1.0/calls/{call_id}/transfer
Content-Type: application/json
X-Auth-Token: {user_token}
```

**Payload :**
```json
{
  "extension": "1003",
  "context": "default"
}
```

**Reponse :**
```json
{
  "call_id": "call-blind001",
  "status": "transferring"
}
```

> **Note** : Pas de `transferee_call_id` — c'est un transfert blind

### Avertissements

- **Irréversible** : Une fois le transfert initiates, il ne peut pas être annule
- **Statut de l'appel** : L'appel original disparait et un nouvel appel est cree vers le destinataire

---

## 4.3 Conference Ad-hoc (Appel Conference Instantane)

### Objectif

Creer une conference ad-hoc instantanee en appelant plusieurs participants depuis un point d'entree unique. Ce scenario permet d'organiser rapidement une reunion telephonique sans configuration prealable de salle de conference.

### Services impliqués

- **wazo-calld** : Creation des conferences ad-hoc et gestion des appels
- **wazo-confd** : Lecture des utilisateurs et extensions

### Le Workflow détaillé

#### Étape 1 : Appeler le premier participant

```http
POST /api/calld/1.0/calls
Content-Type: application/json
X-Auth-Token: {initiator_token}
```

**Payload :**
```json
{
  "extension": "1001",
  "line_id": 5,
  "context": "default"
}
```

**Reponse :**
```json
{
  "call_id": "call-conf-001",
  "status": "ringing"
}
```

> **🔗 Chaînage** : Stockez **CALL_ID_1**.

#### Étape 2 : Ajouter le deuxieme participant

```http
POST /api/calld/1.0/calls
Content-Type: application/json
X-Auth-Token: {initiator_token}
```

**Payload :**
```json
{
  "extension": "1002",
  "line_id": 5,
  "context": "default"
}
```

**Reponse :**
```json
{
  "call_id": "call-conf-002",
  "status": "ringing"
}
```

> **🔗 Chaînage** : Stockez **CALL_ID_2**.

#### Étape 3 : Creer la conference et y ajouter les appelants

```http
POST /api/calld/1.0/conferences
Content-Type: application/json
X-Auth-Token: {initiator_token}
```

**Payload :**
```json
{
  "name": "conference_adhoc_001",
  "extension": "3000",
  "context": "default"
}
```

**Reponse :**
```json
{
  "id": 15,
  "name": "conference_adhoc_001",
  "extension": "3000",
  "context": "default"
}
```

> **🔗 Chaphinage** : Stockez l'ID de conference **CONF_ID**.

#### Étape 4 : Transferer les appels vers la conference

```http
PUT /api/calld/1.0/calls/{call_id_1}/transfer
Content-Type: application/json
X-Auth-Token: {initiator_token}
```

**Payload :**
```json
{
  "extension": "3000",
  "context": "default"
}
```

Repetez pour **CALL_ID_2**.

### Avertissements

- **Limite de participants** : La configuration de la conference dans confd definit le nombre maximum de participants
- **Audio only** : Les conferences ad-hoc sont en audio uniquement (pas de video)
- **Mute des participants** : L'initiateur peut mettre en mute les participants via calld

---

## 4.4 Configuration Salle de Conference Standard

### Objectif

Creer et configurer une salle de conference permanente avec des options avancees : PIN de protection, musique d'attente, annonce des participants, enregistrement. Ce scenario est utilise pour les reunions regulieres avec des acces securises.

### Services impliqués

- **wazo-confd** : Creation et configuration des salles de conference

### Le Workflow détaillé

#### Étape 1 : Creer la conference

```http
POST /api/confd/1.1/conferences
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "Conseil d'Administration",
  "extension": "3100",
  "context": "default",
  "pin": "8426",
  "admin_pin": "1234",
  "max_members": 20,
  "music_on_hold_when_empty": true,
  "announce_join_leave": true,
  "announce_only_user": false,
  "require_moderator": false,
  "record": true
}
```

**Reponse :**
```json
{
  "id": 42,
  "uuid": "conf-550e8400-e29b-41d4-a716-446655440000",
  "name": "Conseil d'Administration",
  "extension": "3100",
  "context": "default",
  "pin": "8426",
  "max_members": 20,
  ...
}
```

> **🔗 Chaînage** : Stockez **CONF_UUID** et **CONF_EXTENSION**.

#### Étape 2 : Associer un schedule (optionnel)

```http
PUT /api/confd/1.1/conferences/{conf_uuid}
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "schedule_id": 7
}
```

#### Step 3: Configure Recording Storage

```http
POST /api/confd/1.1/conferences/{conf_uuid}/recordings
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "destination": "s3",
  "bucket": "wazo-recordings",
  "path": "conferences/{year}/{month}/{day}/{name}"
}
```

### Avertissements

- **PIN security** : Le PIN doit contenir au moins 4 chiffres ; le PIN admin permet de moderer
- **Enregistrement** : L'enregistrement necessite de l'espace de stockage configure
- **Concurrent conferences** : Verifiez les limites de licences pour les conferences simultanees

---

## 4.5 Réunion WebRTC avec Autorisation

### Objectif

Creer une reunion video via WebRTC avec controle d'acces par identifiant de reunion et mot de passe. Ce scenario est utilise pour les reunions virtuelles avec participants externes ou internes, offrant une experience navigateurs sans installation de client.

### Services impliqués

- **wazo-calld** : Gestion des appels et reunions WebRTC
- **wazo-confd** : Lecture des salles de conference
- **wazo-websocketd** : Notifications temps reel

### Le Workflow détaillé

#### Étape 1 : Verifier l'acces a la reunion

```http
GET /api/calld/1.0/conferences/{conference_id}/join
Content-Type: application/json
X-Auth-Token: {user_token}
```

**Payload :**
```json
{
  "meeting_id": "reunion-2024-001",
  "password": "secret123"
}
```

**Reponse :**
```json
{
  "allowed": true,
  "conference_id": 42,
  "bridge_id": "bridge-webrtc-001"
}
```

#### Étape 2 : Creer le token de connexion WebRTC

```http
POST /api/calld/1.0/webrtc/token
Content-Type: application/json
X-Auth-Token: {user_token}
```

**Payload :**
```json
{
  "conference_id": 42,
  "display_name": "Jean Dupont",
  "email": "jean.dupont@acme.fr"
}
```

**Reponse :**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "websocket_url": "wss://wazo.example.com:443/ws",
  "conference_bridge": "bridge-webrtc-001"
}
```

#### Étape 3 : Connecter le client WebRTC

Utilisez le token pour initialiser une connexion WebRTC depuis le navigateur :

```javascript
// Exemple avec JavaScript
const ws = new WebSocket('wss://wazo.example.com/ws?token=' + token);

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  // Gerer les evenements : participant_joined, participant_left, etc.
};
```

#### Étape 4 : Mettre a jour les permissions en cours de reunion

```http
PUT /api/calld/1.0/conferences/{conference_id}/participants/{participant_id}
Content-Type: application/json
X-Auth-Token: {moderator_token}
```

**Payload :**
```json
{
  "muted": false,
  "video": true,
  "floor": true
}
```

### Avertissements

- **Token expiration** : Les tokens WebRTC expirent apres 4 heures par defaut
- **Bandwidth** : Les reunions video consomment beaucoup de bande passante ; predeployer les parametres RTP
- **Browser compatibility** : Verifier la compatibilite des navigateurs avec les codecs utilises

---

## 4.6 Paging (Diffusion Audio Unidirectionnelle)

### Objectif

Effectuer une diffusion audio unidirectionnelle vers plusieurs terminaux sans que les destinataires ne puissent repondre. Ce scenario est utilise pour les annonces generales dans les entrepoints, bureaux ou zones comunes.

### Services impliqués

- **wazo-confd** : Configuration des groupes de paging
- **wazo-calld** : Activation du paging

### Le Workflow détaillé

#### Étape 1 : Creer le groupe de paging

```http
POST /api/confd/1.1/paging
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "Entrepot Zone A",
  "extension": "9001",
  "context": "default",
  "timeout": 30,
  "record": false,
  "announce": true,
  "announce_sound": "beep"
}
```

**Reponse :**
```json
{
  "id": 8,
  "uuid": "paging-550e8400-e29b-41d4-a716-446655440000",
  "name": "Entrepot Zone A",
  "extension": "9001",
  ...
}
```

> **🔗 Chaînage** : Stockez **PAGING_UUID**.

#### Étape 2 : Ajouter des membres au groupe de paging

```http
PUT /api/confd/1.1/paging/{paging_uuid}/members
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "user_uuids": [
    "user-uuid-1",
    "user-uuid-2",
    "user-uuid-3"
  ]
}
```

#### Étape 3 : Initier le paging

```http
POST /api/calld/1.0/paging/{paging_extension}
Content-Type: application/json
X-Auth-Token: {initiator_token}
```

**Payload :**
```json
{
  "context": "default"
}
```

**Reponse :**
```json
{
  "call_id": "call-paging-001",
  "status": "paging",
  "paging_extension": "9001"
}
```

### Avertissements

- **Sens unique** : Les participants ne peuvent pas repondre au paging
- **Interruption** : L'initiateur peut arreter le paging a tout moment
- **Licences** : Verifier les limites de canaux simultanes pour le paging

---

## 4.7 Intercom (Appel Main libre)

### Objectif

Activer le mode main libre sur un terminal pour permettre des appels immediats sans decrocher. Ce scenario est utilise pour les secretariats, zones de reception ou situations ou les mains doivent rester libres.

### Services implique

- **wazo-confd** : Configuration des funckeys et extensions
- **wazo-calld** : Gestion des appels

### Le Workflow detaille

#### Etape 1 : Configurer la fonction intercom sur le terminal

```http
POST /api/confd/1.1/funckeys
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "user_uuid": "user-uuid-cible",
  "keynum": 1,
  "label": "Intercom Bureau",
  "function": "intercom",
  "extension": "1005",
  "context": "default"
}
```

#### Etape 2 : Activer le terminal en mode intercom

```http
PUT /api/confd/1.1/lines/{line_id}/extensions/{ext_id}
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "intercom_enabled": true
}
```

#### Etape 3 : Declencher l'appel intercom

Appuyez sur la touche de fonction configuree ou appelez :

```http
POST /api/calld/1.0/calls/intercom
Content-Type: application/json
X-Auth-Token: {initiator_token}
```

**Payload :**
```json
{
  "extension": "1005",
  "context": "default"
}
```

**Reponse :**
```json
{
  "call_id": "call-intercom-001",
  "status": "Talking",
  "auto_answer": true
}
```

### Avertissements

- **Auto-reponse** : Le terminal cible repond automatiquement (compatibilite SIP required)
- **Contexte** : L'extension intercom doit etre dans le meme contexte que l'appelant
- **Securite** : Limiter l'acces a cette fonction pour eviter les abus

---

## 4.8 Messagerie Instantanée (Chat)

### Objectif

Envoyer et recevoir des messages instantanes entre utilisateurs Wazo via le service de chat integre. Ce scenario permet la communication textuelle synchrone et asynchrone directement depuis les clients Wazo.

### Services implique

- **wazo-chatd** : Gestion des messages et conversations
- **wazo-presenced** : Gestion des presences et statuts

### Le Workflow detaille

#### Etape 1 : Recuperer la liste des conversations

```http
GET /api/chatd/1.0/users/{user_uuid}/conversations
X-Auth-Token: {user_token}
```

**Reponse :**
```json
{
  "items": [
    {
      "uuid": "conv-001",
      "name": "Discussion avec Marie",
      "participants": ["user-uuid-1", "user-uuid-2"],
      "last_message": "Bonjour !",
      "updated_at": "2024-01-15T10:30:00Z"
    }
  ],
  "total": 1
}
```

> **🔗 Chaînage** : Stockez **CONV_UUID**.

#### Etape 2 : Envoyer un message

```http
POST /api/chatd/1.0/conversations/{conv_uuid}/messages
Content-Type: application/json
X-Auth-Token: {user_token}
```

**Payload :**
```json
{
  "content": "Bonjour Marie, avez-vous recu le rapport ?",
  "content_type": "text/plain"
}
```

**Reponse :**
```json
{
  "uuid": "msg-001",
  "conversation_uuid": "conv-001",
  "sender_uuid": "user-uuid-1",
  "content": "Bonjour Marie, avez-vous recu le rapport ?",
  "created_at": "2024-01-15T11:00:00Z"
}
```

#### Etape 3 : Recevoir les messages (WebSocket)

```http
wss://wazo.example.com/api/chatd/1.0/ws?token={user_token}
```

**Message recu :**
```json
{
  "event": "message_created",
  "data": {
    "uuid": "msg-002",
    "conversation_uuid": "conv-001",
    "sender_uuid": "user-uuid-2",
    "content": "Oui, je l'ai recu. Merci !",
    "created_at": "2024-01-15T11:05:00Z"
  }
}
```

#### Etape 4 : Marquer comme lu

```http
PUT /api/chatd/1.0/conversations/{conv_uuid}/read
Content-Type: application/json
X-Auth-Token: {user_token}
```

**Payload :**
```json
{
  "message_uuid": "msg-001"
}
```

### Avertissements

- **Chiffrement** : Les messages ne sont pas chiffres par defaut en stockage
- **Retention** : Configurer la politique de rétention des messages
- **Tailles des fichiers** : Les pieces jointes ont des limites de taille

---

## 4.9 Gestion des Presences et Statuts

### Objectif

Gerer et consultes les statuts de presence des utilisateurs en temps reel. Ce scenario est utilise pour les dashboards d'equipe, les statistiques de disponibilite et l'integration avec les systemes de supervision.

### Services implique

- **wazo-presenced** : Gestion des presences
- **wazo-chatd** : Consultation des statuts

### Le Workflow detaille

#### Etape 1 : Consulter le statut d'un utilisateur

```http
GET /api/presenced/1.0/users/{user_uuid}/presence
X-Auth-Token: {admin_token}
```

**Reponse :**
```json
{
  "user_uuid": "user-uuid-cible",
  "presence": "available",
  "status": "En reunion",
  "last_update": "2024-01-15T10:00:00Z",
  "endpoint": "SIP/1001"
}
```

#### Etape 2 : Mettre a jour son propre statut

```http
PUT /api/presenced/1.0/users/{user_uuid}/presence
Content-Type: application/json
X-Auth-Token: {user_token}
```

**Payload :**
```json
{
  "presence": "busy",
  "status": "En appel avec client"
}
```

**Reponse :**
```json
{
  "user_uuid": "user-uuid-1",
  "presence": "busy",
  "status": "En appel avec client",
  "last_update": "2024-01-15T11:30:00Z"
}
```

#### Etape 3 : Consulter les presences de tous les utilisateurs

```http
GET /api/presenced/1.0/presences
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Reponse :**
```json
{
  "items": [
    {
      "user_uuid": "user-uuid-1",
      "presence": "available",
      "status": "Disponible"
    },
    {
      "user_uuid": "user-uuid-2",
      "presence": "busy",
      "status": "En reunion"
    },
    {
      "user_uuid": "user-uuid-3",
      "presence": "away",
      "status": "Absent"
    }
  ]
}
```

#### Etape 4 : Recevoir les notifications de presence (WebSocket)

```http
wss://wazo.example.com/api/presenced/1.0/ws?token={user_token}
```

**Notification recue :**
```json
{
  "event": "user_presentity_changed",
  "data": {
    "user_uuid": "user-uuid-2",
    "presence": "available",
    "status": "De retour",
    "last_update": "2024-01-15T12:00:00Z"
  }
}
```

### Avertissements

- **Mise a jour automatique** : La presence est automatiquement mise a jour lors des appels
- **Duree de validite** : Un statut "away" est applique apres un delai d'inactivite configure
- **Statuts personnalises** : Les utilisateurs peuvent definir leurs propres messages de statut

---

## 4.10 Statut "Ne Pas Deranger" (DND) API

### Objectif

Activer ou desactiver a distance le statut "Ne Pas Deranger" (Do Not Disturb) pour un utilisateur. Ce scenario permet aux administrateurs ou aux systemes automatises de gerer la disponibilite des utilisateurs.

### Services implique

- **wazo-confd** : Configuration du DND
- **wazo-calld** : Application en temps reel

### Le Workflow detaille

#### Etape 1 : Activer le DND pour un utilisateur

```http
PUT /api/confd/1.1/users/{user_uuid}/services/dnd
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "enabled": true
}
```

**Reponse :**
```json
{
  "enabled": true,
  "user_uuid": "user-uuid-cible"
}
```

#### Etape 2 : Verifier le statut DND

```http
GET /api/confd/1.1/users/{user_uuid}/services/dnd
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Reponse :**
```json
{
  "enabled": true,
  "user_uuid": "user-uuid-cible"
}
```

#### Etape 3 : Desactiver le DND

```http
PUT /api/confd/1.1/users/{user_uuid}/services/dnd
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "enabled": false
}
```

### Avertissements

- **Appels d'urgence** : Le DND ne bloque pas les appels d'urgence (112, etc.)
- **Appels internes** : Le comportement avec les appels internes depend de la configuration
- **Notification** : Les appelants peuvent recevoir un message vocal informant du DND

---

## 4.11 Journal d'Appels (CDR) en Temps Reel

### Objectif

Consulter les appels en cours et les historiques en temps reel pour le monitoring операционной деятельности. Ce scenario est utilise pour les tableaux de bord operateurs et la supervision des communications.

### Services implique

- **wazo-calld** : Appels actifs
- **wazo-cdr** : Historique des appels

### Le Workflow detaille

#### Etape 1 : Lister les appels actifs

```http
GET /api/calld/1.0/calls
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Reponse :**
```json
{
  "items": [
    {
      "call_id": "call-abc123",
      "status": "talking",
      "caller_id_num": "1001",
      "caller_id_name": "Jean Dupont",
      "peer_caller_id_num": "1002",
      "peer_caller_id_name": "Marie Martin",
      "direction": "internal",
      "creation_time": "2024-01-15T14:30:00Z",
      "duration": 120
    }
  ]
}
```

#### Etape 2 : Obtenir les details d'un appel specifique

```http
GET /api/calld/1.0/calls/{call_id}
X-Auth-Token: {admin_token}
```

**Reponse :**
```json
{
  "call_id": "call-abc123",
  "status": "talking",
  "caller_id_num": "1001",
  "caller_id_name": "Jean Dupont",
  "peer_caller_id_num": "1002",
  "peer_caller_id_name": "Marie Martin",
  "conversation": "bridge-uuid-123",
  "channels": [
    "channel-uuid-1",
    "channel-uuid-2"
  ]
}
```

#### Etape 3 : Consulter le CDR historique

```http
GET /api/cdr/1.0/cdr
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Parametres de requete :**
```
?start=2024-01-15T00:00:00Z&end=2024-01-15T23:59:59Z&limit=100
```

**Reponse :**
```json
{
  "items": [
    {
      "id": 15234,
      "call_id": "call-cdr-001",
      "caller": "1001",
      "caller_name": "Jean Dupont",
      "callee": "1002",
      "callee_name": "Marie Martin",
      "direction": "internal",
      "duration": 180,
      "answered": true,
      "start_time": "2024-01-15T14:30:00Z",
      "answer_time": "2024-01-15T14:30:05Z",
      "end_time": "2024-01-15T14:33:05Z"
    }
  ]
}
```

#### Etape 4 : Recevoir les notifications d'appels (WebSocket)

```http
wss://wazo.example.com/api/calld/1.0/ws?token={admin_token}
```

**Nouvel appel :**
```json
{
  "event": "call_started",
  "data": {
    "call_id": "call-notif-001",
    "caller_id_num": "1001",
    "peer_caller_id_num": "1002"
  }
}
```

**Appel termine :**
```json
{
  "event": "call_ended",
  "data": {
    "call_id": "call-notif-001",
    "duration": 120,
    "reason": "normal_clearing"
  }
}
```

### Avertissements

- **Ressources systeme** : Les requetes frequentes sur les CDR peuvent impacter les performances
- **Retention** : Les CDR sont conserves selon la politique de rétention configuree
- **Droits d'acces** : Seul le superadmin ou les utilisateurs avec les bons ACL peuvent acceder a tous les CDR

---

## 4.12 Recording (Enregistrement des Appels)

### Objectif

Activer et gerer l'enregistrement des appels pour la formation, la qualite ou la conformite. Ce scenario couvre l'activation, la pause et la recuperation des enregistrements.

### Services implique

- **wazo-confd** : Configuration de l'enregistrement
- **wazo-calld** : Gestion temps reel de l'enregistrement

### Le Workflow detaille

#### Etape 1 : Activer l'enregistrement sur une ligne

```http
PUT /api/confd/1.1/lines/{line_id}
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "record_incoming": true,
  "record_outgoing": true
}
```

#### Etape 2 : Demarrer l'enregistrement en cours d'appel

```http
POST /api/calld/1.0/calls/{call_id}/record
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "format": "wav"
}
```

**Reponse :**
```json
{
  "call_id": "call-abc123",
  "recording": {
    "id": "rec-001",
    "status": "recording",
    "format": "wav"
  }
}
```

#### Etape 3 : Pause / Reprise de l'enregistrement

```http
PUT /api/calld/1.0/calls/{call_id}/record/pause
X-Auth-Token: {admin_token}
```

**Reponse :**
```json
{
  "call_id": "call-abc123",
  "recording": {
    "id": "rec-001",
    "status": "paused"
  }
}
```

Pour reprendre :

```http
PUT /api/calld/1.0/calls/{call_id}/record/resume
X-Auth-Token: {admin_token}
```

#### Etape 4 : Recuperer la liste des enregistrements

```http
GET /api/calld/1.0/recordings
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Reponse :**
```json
{
  "items": [
    {
      "id": "rec-001",
      "call_id": "call-abc123",
      "duration": 180,
      "format": "wav",
      "created_at": "2024-01-15T14:30:00Z",
      "file_path": "/var/spool/asterisk/monitor/2024/01/15/call-abc123.wav"
    }
  ]
}
```

#### Etape 5 : Telecharger un enregistrement

```http
GET /api/calld/1.0/recordings/{recording_id}/file
X-Auth-Token: {admin_token}
```

**Response :** Fichier audio (WAV ou OGG selon configuration)

### Avertissements

- **Consentement** : Informer les participants de l'enregistrement (conformite legale)
- **Stockage** : Prevoir suffisamment d'espace de stockage pour les enregistrements
- **Cryptage** : Les fichiers peuvent être chiffre's au repos selon la configuration

---

## 4.13 Park Call (Stationnement d'Appel)

### Objectel

Stationner un appel dans une zone de parking pour le reprendre depuis un autre poste ou le transferer. Ce scenario est utilise dans les environnements de bureau ouvert ou les centres d'appel.

### Services implique

- **wazo-confd** : Configuration des zones de parking
- **wazo-calld** : Gestion du stationnement

### Le Workflow detaille

#### Etape 1 : Configurer une zone de parking (si non existante)

```http
POST /api/confd/1.1/extensions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "exten": "7000",
  "context": "parking",
  "commented": false,
  "label": "Zone Parking Principal"
}
```

> **🔗 Chaînage** : Extension de parking = **7000**.

#### Etape 2 : Stationner l'appel en cours

Depuis le poste de l'agent (via code de fonction) :

```http
POST /api/calld/1.0/calls/{call_id}/park
Content-Type: application/json
X-Auth-Token: {user_token}
```

**Payload :**
```json
{
  "parking_extension": "7000"
}
```

**Reponse :**
```json
{
  "call_id": "call-abc123",
  "status": "parked",
  "parking_extension": "7000",
  "timeout": 60
}
```

#### Etape 3 : Recuperer l'appel stationne

```http
POST /api/calld/1.0/calls
Content-Type: application/json
X-Auth-Token: {user_token}
```

**Payload :**
```json
{
  "extension": "7000",
  "context": "parking"
}
```

### Avertissements

- **Timeout** : Par defaut, l'appel revient vers le stationneur apres 60 secondes
- **Zone de parking** : Verifier que la zone est configuree dans le dialplan
- **Limite** : Nombre de places limité selon la configuration

---

## 4.14 Monitoring en Temps Reel (Spy/Barge)

### Objectif

Ecouter en silence un appel en cours (spy) ou y participer (barge). Ce scenario est utilise pour la formation des nouveaux agents ou la supervision qualitative.

### Services implique

- **wazo-calld** : Gestion de l'interception

### Le Workflow detaille

#### Etape 1 : Lister les appels actifs pour identifier la cible

```http
GET /api/calld/1.0/calls?extension=1001
X-Auth-Token: {supervisor_token}
```

**Reponse :**
```json
{
  "items": [
    {
      "call_id": "call-target-001",
      "status": "talking",
      "peer_caller_id_num": "1001"
    }
  ]
}
```

> **🔗 Chaînage** : Stockez **CALL_ID**.

#### Etape 2 : Effectuer une ecoute discrete (Spy)

```http
POST /api/calld/1.0/calls/{call_id}/spy
Content-Type: application/json
X-Auth-Token: {supervisor_token}
```

**Payload :**
```json
{
  "whisper": false
}
```

**Reponse :**
```json
{
  "spy_call_id": "call-spy-001",
  "status": "spy",
  "target_call_id": "call-target-001"
}
```

#### Etape 3 : Participer a l'appel (Barge)

```http
POST /api/calld/1.0/calls/{call_id}/spy
Content-Type: application/json
X-Auth-Token: {supervisor_token}
```

**Payload :**
```json
{
  "whisper": true
}
```

**Reponse :**
```json
{
  "spy_call_id": "call-barge-001",
  "status": "barge",
  "target_call_id": "call-target-001"
}
```

### Avertissements

- **Droits** : Seuls les utilisateurs avec le droit `spy` peuvent ecouter
- **Notification** : Dans certains pays, la notification de surveillance est requise
- **Audio quality** : La qualite depend de la bande passante disponible

---

## 4.15 API Stasis (Asterisk ARI) pour Integration Avancee

### Objectif

Utiliser l'API Stasis (Asterisk REST Interface) pour une integration avancee avec des applications tierces. Ce scenario permet de controler les canaux Asterisk directement pour des cas d'usage complexes.

### Services implique

- **wazo-calld** : Pont vers Stasis
- **Asterisk ARI** : API native Asterisk

### Le Workflow detaille

#### Etape 1 : S'authentifier auprès dARI

```http
Authorization: Basic {base64(ari_user:ari_password)}
```

#### Etape 2 : Creer un point d'entree Stasis

```http
POST /ari/applications
Content-Type: application/json
```

**Payload :**
```json
{
  "name": "my_custom_app",
  "display_name": "Application Personnalisee"
}
```

**Reponse :**
```json
{
  "name": "my_custom_app",
  "events": {
    "channel UsserEvent",
    "channel_destroyed",
    "stasis_start"
  }
}
```

#### Etape 3 : Declarer un canal vers Stasis

```http
POST /ari/channels
Content-Type: application/json
```

**Payload :**
```json
{
  "endpoint": "SIP/1001@default",
  "app": "my_custom_app",
  "appArgs": "dialplan"
}
```

**Reponse :**
```json
{
  "id": "channel-uuid-123",
  "name": "SIP/1001-00000001",
  "state": "Ring",
  "dialplan": {
    "context": "default",
    "exten": "1001",
    "priority": 1
  }
}
```

#### Etape 4 : Recevoir les evenements Stasis

Connexion WebSocket :

```http
wss://wazo.example.com:8089/ari/events?app=my_custom_app
```

**Evenement recu :**
```json
{
  "type": "StasisStart",
  "channel": {
    "id": "channel-uuid-123",
    "name": "SIP/1001-00000001"
  },
  "args": ["dialplan"]
}
```

#### Etape 5 : Controler le canal

```http
POST /ari/channels/{channel_id}/play
Content-Type: application/json
```

**Payload :**
```json
{
  "media": "sound:welcome"
}
```

### Avertissements

- **Complexite** : ARI necessite une bonne connaissance d'Asterisk
- **Performance** : Attention aux operations bloqueantes qui peuvent saturer Asterisk
- **Securite** : Limiter l'acces a ARI et utiliser l'authentification forte
- **Stability** : Les applications mal configurees peuvent destabiliser le PBX

---

## Resume des Services pour PARTIE 4

| Scenario | Service Principal | Operations cles |
|----------|-------------------|-----------------|
| Transfert Attended | calld | PUT /calls/{id}/transfer |
| Transfert Blind | calld | PUT /calls/{id}/transfer |
| Conference Ad-hoc | calld | POST /conferences, transfer |
| Salle Conference | confd | POST /conferences |
| WebRTC | calld | POST /webrtc/token |
| Paging | confd/calld | POST /paging/{ext} |
| Intercom | confd/calld | POST /calls/intercom |
| Chat | chatd | POST /conversations/{id}/messages |
| Presence | presenced | GET/PUT /users/{uuid}/presence |
| DND | confd | PUT /users/{uuid}/services/dnd |
| CDR Temps Reel | calld/cdr | GET /calls, GET /cdr |
| Recording | calld | POST /calls/{id}/record |
| Park Call | calld | POST /calls/{id}/park |
| Spy/Barge | calld | POST /calls/{id}/spy |
| Stasis ARI | calld/asterisk | ARI REST + WebSocket |

---

*Cette partie couvre les scenarios de communication en temps reel. Pour les aspects de securite et integrations avancees, consultez la PARTIE 5.*
