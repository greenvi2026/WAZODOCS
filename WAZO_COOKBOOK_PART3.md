---

# PARTIE 3 : Services Avancés & Call Center

Cette partie couvre les configurations avancées du centre d'appels (ACD), les services de routage intelligent, et les fonctionnalités avancées comme les IVR, les conférences et le filtrage d'appels.

---

## 3.1 File d'Attente ACD Complète (9 Étapes)

### Objectif

Créer une file d'attente complète pour un centre d'appels avec agents, skills, stratégies de distribution et plannings horaires.

### Services impliqués

- **wazo-confd** : Gestion des queues, agents, skills, schedules, incalls

### Le Workflow détaillé

#### Étape 1 : Créer la queue

```http
POST /api/confd/1.1/queues
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "support-technique",
  "display_name": "Support Technique",
  "strategy": "rrmemory",
  "timeout": 30,
  "retry": 5,
  "maxlen": 10,
  "announce_frequency": 30,
  "music_on_hold": "moh-uuid-001"
}
```

**Réponse :**
```json
{
  "id": 15,
  "name": "support-technique",
  "strategy": "rrmemory",
  ...
}
```

> **🔗 Chaînage** : Stockez **QUEUE_ID** = 15

#### Étape 2 : Créer l'extension de la queue

```http
POST /api/confd/1.1/extensions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "exten": "3000",
  "context": "default"
}
```

> **🔗 Chaînage** : Stockez **QUEUE_EXT_ID**

#### Étape 3 : Lier l'extension à la queue

```http
PUT /api/confd/1.1/queues/{QUEUE_ID}/extensions/{QUEUE_EXT_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

#### Étape 4 : Créer un agent

```http
POST /api/confd/1.1/agents
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "number": "5001",
  "firstname": "Alice",
  "lastname": "Agent",
  "password": "1234"
}
```

**Réponse :**
```json
{
  "id": 8,
  "number": "5001",
  ...
}
```

> **🔗 Chaînage** : Stockez **AGENT_ID** = 8

#### Étape 5 : Créer un skill

```http
POST /api/confd/1.1/agents/skills
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "technique",
  "description": "Compétences techniques"
}
```

**Réponse :**
```json
{
  "id": 3,
  "name": "technique",
  ...
}
```

> **🔗 Chaînage** : Stockez **SKILL_ID** = 3

#### Étape 6 : Associer le skill à l'agent

```http
PUT /api/confd/1.1/agents/{AGENT_ID}/skills/{SKILL_ID}
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "skill_weight": 10
}
```

#### Étape 7 : Ajouter l'agent à la queue

```http
PUT /api/confd/1.1/queues/{QUEUE_ID}/members/agents/{AGENT_ID}
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "penalty": 0,
  "priority": 1
}
```

#### Étape 8 : Créer le schedule (optionnel)

```http
POST /api/confd/1.1/schedules
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "horaires-support",
  "timezone": "Europe/Paris",
  "open_periods": [
    {
      "start": "09:00",
      "end": "18:00",
      "days": ["monday", "tuesday", "wednesday", "thursday", "friday"]
    }
  ]
}
```

> **🔗 Chaînage** : Stockez **SCHEDULE_ID**

#### Étape 9 : Créer l'incall pour la queue

```http
POST /api/confd/1.1/incalls
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "destination": {
    "type": "queue",
    "queue_id": 15
  },
  "extensions": [{"id": QUEUE_EXT_ID}],
  "schedule_id": SCHEDULE_ID
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - La stratégie `linear` ne peut pas être activée via API si elle n'était pas configurée initialement
> - Les skills permettent un routage intelligent basé sur les compétences
> - Le `penalty` de l'agent détermine sa priorité dans la queue

---

## 3.2 Skill Rules — Routage ACD par Compétence (6 Étapes)

### Objectif

Créer des règles de routage basées sur les skills des agents pour distribuer intelligemment les appels vers les agents appropriés..

### Services impliqués

- **wazo-confd** : Gestion des skill rules

### Le Workflow détaillé

#### Étape 1 : Créer un skill

```http
POST /api/confd/1.1/agents/skills
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "francais",
  "description": "French language skill"
}
```

> **🔗 Chaînage** : Stockez **SKILL_FR_ID**

#### Étape 2 : Associer le skill à l'agent avec weight

```http
PUT /api/confd/1.1/agents/{AGENT_ID}/skills/{SKILL_FR_ID}
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "skill_weight": 100
}
```

#### Étape 3 : Créer une skill rule

```http
POST /api/confd/1.1/queues/skillrules
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "regle-francais",
  "rules_definition": "FR > 0"
}
```

**Réponse :**
```json
{
  "id": 7,
  "name": "regle-francais",
  "rules_definition": "FR > 0"
}
```

> **🔗 Chaînage** : Stockez **SKILL_RULE_ID** = 7

#### Étape 4 : Créer l'incall avec skill rule

```http
POST /api/confd/1.1/incalls
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "destination": {
    "type": "queue",
    "queue_id": QUEUE_ID,
    "skill_rule_id": 7
  },
  "extensions": [{"id": EXT_ID}]
}
```

> **🔗 Chaînage** : Les `skill_rule_variables` peuvent être ajoutées pour transmettre des variables à la règle.

### Point d'attention / Warning

> **⚠️ Important** :
> - La syntaxe des règles utilise les opérateurs : `>`, `<`, `=`, `&` (AND), `|` (OR)
> - Les noms de skills sont utilisés dans les règles (ex. `FR`, `TECH`)
> - Exemple : `FR > 0 & TECH > 50`
> - La variable `WT` (Waiting Time) correspond au temps d'attente de l'appel dans la file (en secondes)
> - Les règles sont évaluées dans l'ordre : la première règle valide est utilisée
> - Exemple de priorisation :
>   ```
>   WT < 20 & FR > 99
>   FR > 49
>   ```
> - Les skill rules sélectionnent les agents éligibles ; la distribution reste assurée par la stratégie de la file (`ringall`, `rrmemory`, `leastrecent`, `fewestcalls`, ...)."""

---

## 3.3 Groupe d'Appel — Ring Group (6 Étapes)

### Objectif

Créer un groupe d'appel qui sonne sur plusieurs postes simultanément, idéal pour les équipes qui doivent répondre aux appels entrants.

### Services impliqués

- **wazo-confd** : Gestion des groups, extensions

### Le Workflow détaillé

#### Étape 1 : Créer le ring group

```http
POST /api/confd/1.1/groups
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "equipe-commerciale",
  "label": "Équipe Commerciale",
  "strategy": "all"
}
```

**Réponse :**
```json
{
  "id": 12,
  "name": "equipe-commerciale",
  "strategy": "all"
}
```

> **🔗 Chaînage** : Stockez **GROUP_ID** = 12

#### Étape 2 : Créer l'extension

```http
POST /api/confd/1.1/extensions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "exten": "2000",
  "context": "default"
}
```

> **🔗 Chaînage** : Stockez **GRP_EXT_ID**

#### Étape 3 : Lier l'extension au groupe

```http
PUT /api/confd/1.1/groups/{GROUP_ID}/extensions/{GRP_EXT_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

#### Étape 4 : Ajouter les membres

```http
PUT /api/confd/1.1/groups/{GROUP_ID}/members/users
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "users": [
    {"uuid": "user-uuid-1", "priority": 1},
    {"uuid": "user-uuid-2", "priority": 2}
  ]
}
```

#### Étape 5 : Configurer les fallbacks

```http
PUT /api/confd/1.1/groups/{GROUP_ID}/fallbacks
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "noanswer_destination": {
    "type": "voicemail",
    "voicemail_id": 5
  }
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Stratégies : `all` (tous), `ring` (cyclique), `random`
> - Les utilisateurs doivent avoir une ligne configurée pour recevoir les appels

---

## 3.4 Call Filter — Boss/Secrétaire (6 Étapes)

### Objectif

Configurer le filtre boss-secrétaire permettant aux secrétaires de gérer les appels du patron, avec interception et renvoi automatique.

### Services impliqués

- **wazo-confd** : Gestion des callfilters

### Le Workflow détaillé

#### Étape 1 : Créer le call filter

```http
POST /api/confd/1.1/callfilters
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "filter-boss-secretary",
  "strategy": "all-recipients-then-all-surrogates"
}
```

**Réponse :**
```json
{
  "id": 5,
  "name": "filter-boss-secretary",
  "strategy": "all-recipients-then-all-surrogates"
}
```

> **🔗 Chaînage** : Stockez **CALL_FILTER_ID** = 5

#### Étape 2 : Ajouter le boss (recipient)

```http
PUT /api/confd/1.1/callfilters/{CALL_FILTER_ID}/recipients/users
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "users": [{"uuid": "boss-uuid"}]
}
```

#### Étape 3 : Ajouter le secrétaire (surrogate)

```http
PUT /api/confd/1.1/callfilters/{CALL_FILTER_ID}/surrogates/users
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "users": [{"uuid": "secretary-uuid"}]
}
```

#### Étape 4 : Configurer les fallbacks

```http
PUT /api/confd/1.1/callfilters/{CALL_FILTER_ID}/fallbacks
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "noanswer_destination": {
    "type": "voicemail",
    "voicemail_id": 10
  }
}
```

#### Étape 5 : Activer le filtre sur le boss

```http
PUT /api/confd/1.1/users/{BOSS_UUID}/services/incallfilter/enable
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - `surrogates_timeout` est distinct du timeout par recipient
> - Le filtre doit être activé sur l'utilisateur boss

---

## 3.5 Call Pickup — Interception d'Appel (5 Étapes)

### Objectif

Configurer le pickup de groupe permettant à un utilisateur d'intercepter un appel qui sonne sur un collègue.

### Services impliqués

- **wazo-confd** : Gestion des callpickups

### Le Workflow détaillé

#### Étape 1 : Créer le call pickup

```http
POST /api/confd/1.1/callpickups
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "interception-groupe",
  "enabled": true
}
```

**Réponse :**
```json
{
  "id": 3,
  "name": "interception-groupe",
  "enabled": true
}
```

> **🔗 Chaînage** : Stockez **PICKUP_ID** = 3

#### Étape 2 : Ajouter les cibles (ceux qu'on peut intercepter)

```http
PUT /api/confd/1.1/callpickups/{PICKUP_ID}/targets/users
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "users": [{"uuid": "user-uuid-1"}, {"uuid": "user-uuid-2"}]
}
```

#### Étape 3 : Ajouter les intercepteurs

```http
PUT /api/confd/1.1/callpickups/{PICKUP_ID}/interceptors/users
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "users": [{"uuid": "user-uuid-3"}, {"uuid": "user-uuid-4"}]
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - L'extension par défaut pour le pickup est *8
> - Le pickup peut aussi être configuré par groupes

---

## 3.6 Parking Lot — Parquage d'Appel (5 Étapes)

### Objectif

Configurer un parking lot pour parquer un appel et le récupérer depuis un autre poste.

### Services impliqués

- **wazo-confd** : Gestion des parkinglots

### Le Workflow détaillé

#### Étape 1 : Créer le parking lot

```http
POST /api/confd/1.1/parkinglots
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "parking-principal",
  "slots_start": 701,
  "slots_end": 720,
  "timeout": 120
}
```

**Réponse :**
```json
{
  "id": 2,
  "name": "parking-principal",
  "slots_start": 701,
  "slots_end": 720
}
```

> **🔗 Chaînage** : Stockez **PARKING_ID**

#### Étape 2 : Créer l'extension de parking

```http
POST /api/confd/1.1/extensions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "exten": "700",
  "context": "default"
}
```

> **🔗 Chaînage** : Stockez **PARKING_EXT_ID**

#### Étape 3 : Lier l'extension au parking

```http
PUT /api/confd/1.1/parkinglots/{PARKING_ID}/extensions/{PARKING_EXT_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - L'appel parqué expire après le `timeout`
> - Les slots définissent les extensions où les appels sont parqués

---

## 3.7 IVR Complet — Menu Vocal Interactif (7 Étapes)

### Objectif

Créer un SVI (Serveur Vocal Interactif) complet avec messages d'accueil, choix de menu, et destinations variables.

### Services impliqués

- **wazo-confd** : Gestion des sounds, ivr

### Le Workflow détaillé

#### Étape 1 : Uploader le son d'accueil

```http
POST /api/confd/1.1/sounds
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "ivr-accueil"
}
```

> **🔗 Chaînage** : Stockez **SOUND_NAME** = "ivr-accueil"

#### Étape 2 : Uploader le fichier audio

```http
PUT /api/confd/1.1/sounds/{SOUND_NAME}/files/accueil.wav
Content-Type: audio/wav
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}

[binary audio data]
```

#### Étape 3 : Créer l'IVR

```http
POST /api/confd/1.1/ivr
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "menu-principal",
  "greeting_sound": "ivr-accueil/accueil.wav",
  "menu_sound": "ivr-accueil/menu.wav",
  "choices": {
    "1": {
      "destination": {
        "type": "queue",
        "queue_id": 15
      }
    },
    "2": {
      "destination": {
        "type": "extension",
        "extension": "1000",
        "context": "default"
      }
    },
    "3": {
      "destination": {
        "type": "voicemail",
        "voicemail_id": 5
      }
    }
  },
  "timeout": 5,
  "max_attempts": 3
}
```

**Réponse :**
```json
{
  "id": 8,
  "name": "menu-principal",
  ...
}
```

> **🔗 Chaînage** : Stockez **IVR_ID** = 8

#### Étape 4 : Créer l'extension IVR

```http
POST /api/confd/1.1/extensions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "exten": "9000",
  "context": "default"
}
```

> **🔗 Chaînage** : Stockez **IVR_EXT_ID**

#### Étape 5 : Créer l'incall

```http
POST /api/confd/1.1/incalls
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "destination": {
    "type": "ivr",
    "ivr_id": 8
  },
  "extensions": [{"id": IVR_EXT_ID}]
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Les chemins de sons sont relatifs au répertoire `/var/lib/wazo/sounds/playback/`
> - Uploadez les sons AVANT de créer l'IVR

---

## 3.8 Salle de Conférence avec DID (7 Étapes)

### Objectif

Créer une salle de conférence permanente accessible par DID externe avec code PIN.

### Services impliqués

- **wazo-confd** : Gestion des conferences, extensions, incalls
- **wazo-calld** : Contrôle de la conférence

### Le Workflow détaillé

#### Étape 1 : Créer la conférence

```http
POST /api/confd/1.1/conferences
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "conference-direct",
  "pin": "1234",
  "admin_pin": "9999",
  "max_users": 50,
  "record": true
}
```

**Réponse :**
```json
{
  "id": 6,
  "name": "conference-direct",
  "pin": "1234",
  ...
}
```

> **🔗 Chaînage** : Stockez **CONF_ID** = 6

#### Étape 2 : Créer l'extension interne

```http
POST /api/confd/1.1/extensions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "exten": "4001",
  "context": "default"
}
```

> **🔗 Chaînage** : Stockez **CONF_EXT_ID**

#### Étape 3 : Lier l'extension à la conférence

```http
PUT /api/confd/1.1/conferences/{CONF_ID}/extensions/{CONF_EXT_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

#### Étape 4 : Créer l'incall pour le DID

```http
POST /api/confd/1.1/incalls
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "destination": {
    "type": "conference",
    "conference_id": 6
  }
}
```

> **🔗 Chaînage** : Stockez **INCALL_CONF_ID**

#### Étape 5 : Ajouter l'extension DID entrante

```http
POST /api/confd/1.1/extensions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "exten": "0033380002000",
  "context": "from-extern"
}
```

> **🔗 Chaînage** : Stockez **EXT_DID_CONF**

#### Étape 6 : Lier le DID à l'incall

```http
PUT /api/confd/1.1/incalls/{INCALL_CONF_ID}/extensions/{EXT_DID_CONF}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Le `pin` est pour les participants, `admin_pin` pour le contrôle
> - L'enregistrement nécessite au moins 1 participant actif

---

## 3.9 Call Permissions — Contrôle des Appels Sortants (5 Étapes)

### Objectif

Créer des règles de permissions d'appels pour contrôler quels numéros peuvent être composés (interne, local, national, international).

### Services impliqués

- **wazo-confd** : Gestion des callpermissions

### Le Workflow détaillé

#### Étape 1 : Créer une permission de deny

```http
POST /api/confd/1.1/callpermissions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "interdit-international",
  "mode": "deny",
  "extensions": ["0033.", "0044."]
}
```

**Réponse :**
```json
{
  "id": 10,
  "name": "interdit-international",
  "mode": "deny"
}
```

> **🔗 Chaînage** : Stockez **PERM_DENY_ID**

#### Étape 2 : Créer une permission avec mot de passe

```http
POST /api/confd/1.1/callpermissions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "autorise-international",
  "mode": "allow",
  "extensions": ["0033."],
  "password": "1234"
}
```

> **🔗 Chaînage** : Stockez **PERM_ALLOW_ID**

#### Étape 3 : Appliquer à un outcall

```http
PUT /api/confd/1.1/outcalls/{OUTCALL_ID}/callpermissions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "call_permissions_id": 10
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - `mode: deny` avec extensions = bloquer ces numéros
> - `mode: allow` avec extensions = autoriser ces numéros

---

## 3.10 Paging / Intercom (4 Étapes)

### Objectif

Configurer le paging (intercom) pour permettre l'envoi de messages广播 à un groupe de postes.

### Services impliqués

- **wazo-confd** : Gestion des pagings

### Le Workflow détaillé

#### Étape 1 : Créer le paging

```http
POST /api/confd/1.1/pagings
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "bureau-ouvert",
  "duplex": false,
  "announce_caller": true
}
```

**Réponse :**
```json
{
  "id": 4,
  "name": "bureau-ouvert",
  "duplex": false
}
```

> **🔗 Chaînage** : Stockez **PAGING_ID**

#### Étape 2 : Ajouter les appelants

```http
PUT /api/confd/1.1/pagings/{PAGING_ID}/callers/users
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "users": [{"uuid": "manager-uuid"}]
}
```

#### Étape 3 : Ajouter les membres

```http
PUT /api/confd/1.1/pagings/{PAGING_ID}/members/users
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "users": [{"uuid": "user-uuid-1"}, {"uuid": "user-uuid-2"}]
}
```

#### Étape 4 : Créer l'extension de paging

```http
POST /api/confd/1.1/extensions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "exten": "8000",
  "context": "default"
}
```

> **🔗 Chaînage** : Stockez **PAGING_EXT_ID**

#### Étape 5 : Lier l'extension

```http
PUT /api/confd/1.1/pagings/{PAGING_ID}/extensions/{PAGING_EXT_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - `duplex: false` = simplex (broadcast unidirectionnel)
> - `duplex: true` = bidirectionnel (intercom)

---

## 3.11 Switchboard — Standard Téléphonique (6 Étapes)

### Objectif

Configurer un standard automatique pour permettre à un opératrice de gérer les appels entrants avec answer, hold, transfer.

### Services impliqués

- **wazo-confd** : Gestion des switchboards
- **wazo-calld** : Contrôle des appels

### Le Workflow détaillé

#### Étape 1 : Créer le switchboard

```http
POST /api/confd/1.1/switchboards
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "standard-principal",
  "timeout": 30
}
```

> **🔗 Chaînage** : Stockez **SWITCHBOARD_ID**

#### Étape 2 : Ajouter les membres

```http
PUT /api/confd/1.1/switchboards/{SWITCHBOARD_ID}/members/users
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "users": [{"uuid": "op-uuid"}]
}
```

#### Étape 3 : Créer l'extension

```http
POST /api/confd/1.1/extensions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

> **🔗 Chaînage** : Stockez **SW_EXT_ID**

#### Étape 4 : Lier l'extension

```http
PUT /api/confd/1.1/switchboards/{SWITCHBOARD_ID}/extensions/{SW_EXT_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

#### Étape 5 : Récupérer les appels en attente

```http
GET /api/calld/1.0/switchboards/{SWITCHBOARD_ID}/calls/queued
X-Auth-Token: {admin_token}
```

#### Étope 6 : Answer un appel

```http
PUT /api/calld/1.0/switchboards/{SWITCHBOARD_ID}/calls/queued/{CALL_ID}/answer
X-Auth-Token: {admin_token}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Le switchboard nécessite une licence ou configuration spécifique
> - Les actions : answer, hold, retrieve, redirect-queue

---

## 3.12 Récapitulatif des Endpoints Services Avancés

| Ressource | CRUD | Endpoint |
|-----------|-----|----------|
| Queue | C | POST /queues |
| Agent | C | POST /agents |
| Skill | C | POST /agents/skills |
| Skill Rule | C | POST /queues/skillrules |
| Group | C | POST /groups |
| Call Filter | C | POST /callfilters |
| Call Pickup | C | POST /callpickups |
| Parking | C | POST /parkinglots |
| IVR | C | POST /ivr |
| Conference | C | POST /conferences |
| Call Permission | C | POST /callpermissions |
| Paging | C | POST /pagings |
| Switchboard | C | POST /switchboards |

---

*Fin de la PARTIE 3*
