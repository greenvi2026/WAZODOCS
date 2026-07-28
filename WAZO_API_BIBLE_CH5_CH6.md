---

# CHAPITRE 5 : Services Avancés (wazo-confd)

## 5.1 Files d'Attente (Queues)

### 5.1.1 Concept

Les **queues** (files d'attente) sont le composant central d'un centre d'appels (ACD - Automatic Call Distributor). Elles permettent de :

- Distribuer les appels entrants vers plusieurs agents
- Gérer les temps d'attente
- Appliquer des stratégies de sonnerie multiples
- Mettre en place des compétences (skills) pour le routage intelligent
- Collecter des statistiques détaillées

### 5.1.2 Stratégies de Distribution

| Stratégie | Description | Cas d'usage |
|-----------|------------|-------------|
| `ringall` | Appelle tous les agents simultanément | Urgence, support rapide |
| `leastrecent` | Agent ayant reçu le moins récemment | Distribution uniforme |
| `fewestcalls` | Agent avec le moins d'appels complétés | Équilibre de charge |
| `rrmemory` | Round-robin avec mémoire | Distribution cyclique |
| `random` | Agent aléatoire | Sampling, tests |
| `wrandom` | Aléatoire pondéré par pénalité | Priorisation fine |
| `linear` | Ordre défini (login ou manuel) | Hiérarchie stricte |

> **⚠️ Attention** : La stratégie `linear` ne peut pas être activée via l'API si elle n'était pas initialement configurée. C'est une limitation Asterisk.

### 5.1.3 CRUD des Queues

#### Endpoint

```http
GET/POST    /api/confd/1.1/queues
GET/PUT/DELETE /api/confd/1.1/queues/{queue_id}
```

#### Création d'une Queue Complète

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "support-technique",
    "display_name": "Support Technique",
    "context": "default",
    "timeout": 30,
    "retry": 5,
    "maxlen": 0,
    "options": [
      ["strategy", "rrmemory"],
      ["announce-frequency", "30"],
      ["announce-holdtime", "yes"],
      ["announce-position", "yes"],
      ["periodic-announce-frequency", "60"],
      ["periodic-announce", "queue-periodic-announce"],
      ["music_on_hold", "default"],
      ["joinempty", "yes"],
      ["leavewhenempty", "yes"],
      ["ringinuse", "no"],
      ["setinterfacevar", "yes"],
      ["timeoutrouting", "yes"]
    ],
    "weight": 0,
    "preprocess_subroutine": null,
    "description": "File d'attente support technique niveau 1"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/queues"
```

#### Payload Détaillé des Options de Queue

| Option | Type | Description | Valeurs |
|--------|------|-------------|---------|
| `strategy` | `string` | Stratégie de distribution | `ringall`, `leastrecent`, `fewestcalls`, `rrmemory`, `random`, `wrandom`, `linear` |
| `timeout` | `int` | Timeout d'appel par agent (secondes) | 10-60 |
| `retry` | `int` | Nb de tentatives avant abandon | 1-10 |
| `maxlen` | `int` | Taille max file (0 = illimité) | 0-100 |
| `announce-frequency` | `int` | Fréquence announcement position (secondes) | 15-300 |
| `announce-holdtime` | `string` | Annoncer temps d'attente | `yes`, `no` |
| `announce-position` | `string` | Annoncer position dans la file | `yes`, `no`, `limit` |
| `periodic-announce-frequency` | `int` | Fréquence announcement périodique | 30-600 |
| `music_on_hold` | `string` | Musique d'attente | Nom MOH |
| `joinempty` | `string` | Autoriser entrée si vide | `yes`, `no`, `strict` |
| `leavewhenempty` | `string` | Quitter si file vide | `yes`, `no`, `strict` |
| `ringinuse` | `string` | Sonner si agent en appel | `yes`, `no` |
| `setinterfacevar` | `string` | Variables AMI | `yes`, `no` |
| `timeoutrouting` | `string` | Timeout applique au routage | `yes`, `no` |

#### Réponse (201 Created)

```json
{
  "id": 12,
  "name": "support-technique",
  "display_name": "Support Technique",
  "context": "default",
  "timeout": 30,
  "retry": 5,
  "maxlen": 0,
  "options": [
    ["strategy", "rrmemory"],
    ["announce-frequency", "30"]
  ],
  "weight": 0,
  "tenant_uuid": "tenant-uuid-main"
}
```

### 5.1.4 Association Agent ↔ Queue

#### Ajouter un Agent dans une Queue

```http
PUT /api/confd/1.1/queues/{queue_id}/members/agents/{agent_id}
```

```bash
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "priority": 0,
    "penalty": 0
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/queues/12/members/agents/agent-id-001"
```

#### Payload d'Association Agent

| Champ | Type | Description |
|-------|------|-------------|
| `priority` | `int` | Priorité de l'agent (0 = plus haute) |
| `penalty` | `int` | Pénalité (pour stratégie weighted) |

#### Liste des Membres d'une Queue

```bash
curl -k -X GET \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/queues/12/members"
```

### 5.1.5 Queue avec Skills (Routage par Compétences)

Pour du skills-based routing, créez d'abord des skills :

#### Création d'un Skill

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "anglais",
    "display_name": "Anglais"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/queueskills"
```

#### Association Skill à un Agent

```bash
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "skill_id": 3,
    "agent_id": "agent-id-001",
    "weight": 5
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/agent_skills"
```

---

## 5.2 Groupes d'Appel (Ring Groups)

### 5.2.1 Concept

Les **ring groups** (groupes de sonnerie) permettent de faire sonner plusieurs utilisateurs simultanément ou séquentiellement lorsqu'un numéro interne est composé. Contrairement aux queues, pas de distribution ACD.

### 5.2.2 CRUD des Ring Groups

#### Endpoint

```http
GET/POST    /api/confd/1.1/groups
GET/PUT/DELETE /api/confd/1.1/groups/{group_uuid}
```

#### Création d'un Ring Group

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "equipe-commerciale",
    "display_name": "Équipe Commerciale",
    "context": "default",
    "extension": "200",
    "options": [
      ["strategy", "ringall"],
      ["timeout", "30"],
      ["music_on_hold", "default"]
    ],
    "description": "Groupe sonnerie équipe commerciale"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/groups"
```

#### Options des Ring Groups

| Option | Description | Valeurs |
|--------|-------------|---------|
| `strategy` | Stratégie de sonnerie | `ringall`, `hunt`, `memory`, `firstavailable`, `random` |
| `timeout` | Timeout total (secondes) | 10-300 |
| `music_on_hold` | Musique d'attente | Nom MOH |

#### Ajout de Membres

```bash
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/groups/group-uuid/members/users/user-uuid-001"
```

---

## 5.3 Menus Vocaux (IVR)

### 5.3.1 Concept

Un **IVR** (Interactive Voice Response) est un menu vocal interactif qui accueille l'appelant et permet des choix via tonalité DTMF.

### 5.3.2 CRUD des IVR

#### Endpoint

```http
GET/POST    /api/confd/1.1/ivr
GET/PUT/DELETE /api/confd/1.1/ivr/{ivr_id}
```

#### Création d'un IVR

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "standard-automatique",
    "display_name": "Standard Automatique",
    "context": "default",
    "announcement": null,
    "menu_sound": "",
    "choices": [
      {
        "exten": "1",
        "destination": {
          "type": "queue",
          "queue_id": 12
        }
      },
      {
        "exten": "2",
        "destination": {
          "type": "ivr",
          "ivr_id": 5
        }
      },
      {
        "exten": "3",
        "destination": {
          "type": "voicemail",
          "voicemail_id": 15
        }
      }
    ],
    "timeout": 5,
    "max_timeout_trials": 3,
    "invalid_destination": {
      "type": "ivr",
      "ivr_id": 4
    },
    "description": "Menu d'accueil standard"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/ivr"
```

#### Structure des Choix IVR

```json
{
  "exten": "1",
  "destination": {
    "type": "user|queue|ivr|voicemail|conference|extension|hangup",
    "{resource}_id": "..."
  }
}
```

#### Types de Destinations IVR

| Type | Description |
|------|-------------|
| `user` | Routage vers utilisateur |
| `queue` | Routage vers file d'attente |
| `ivr` | Routage vers sous-menu IVR |
| `voicemail` | Routage vers messagerie vocale |
| `conference` | Routage vers conférence |
| `extension` | Routage vers extension |
| `hangup` | Raccrochage |

---

## 5.4 Conférences

### 5.4.1 Concept

Les **conferences** permettent des appels audio à plusieurs participants.

### 5.4.2 CRUD des Conférences

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "conference-direction",
    "display_name": "Conference Direction",
    "context": "default",
    "extension": "3000",
    "pin": "1234",
    "options": [
      ["announce_join_leave", "yes"],
      ["music_on_hold", "default"],
      ["quiet", "no"],
      ["record", "no"]
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/conferences"
```

---

## 5.5 Plannings (Schedules)

### 5.5.1 Concept

Les **schedules** définissent les plages horaires d'ouverture pour le routage des appels.

### 5.5.2 CRUD des Schedules

#### Création d'un Schedule

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "horaires-bureau",
    "display_name": "Horaires Bureau",
    "timezone": "Europe/Paris",
    "description": "Ouverture du lundi au vendredi",
    "closed_destination": {"type": "none"}
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/schedules"
```

#### Création de Time Periods

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "horaire-journee",
    "display_name": "Heures de journee",
    "timeframes": [
      {
        "days": ["monday", "tuesday", "wednesday", "thursday", "friday"],
        "hours": [
          {"begin": "09:00", "end": "12:00"},
          {"begin": "14:00", "end": "18:00"}
        ]
      }
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/schedules/timeperiods"
```

#### Time Rules (Exceptions / Jours fériés)

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "feries-2026",
    "display_name": "Jours feries",
    "timeframes": [
      {
        "dates": ["2026-01-01", "2026-05-01", "2026-07-14", "2026-12-25"],
        "hours": [{"begin": "00:00", "end": "00:00"}]
      }
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/schedules/timerules"
```

#### Association Schedule → Incall

```bash
# L'incall utilise le schedule pour le routage conditionnel
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "exten": "0143321000",
    "context": "from-extern",
    "priority": 1,
    "destination": {
      "type": "queue",
      "queue_id": 12
    },
    "schedule_id": 7,
    "fallback_destination": {
      "type": "voicemail",
      "voicemail_id": 15
    }
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/incalls"
```

---

## 5.6 Scénario Complet : Centre d'Appels avec Skills

```bash
# =============================================================================
# ÉTAPE 1 : Créer les skills
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{"name": "technique", "display_name": "Support Technique"}' \
  "https://wazo.example.com:9486/api/confd/1.1/queueskills"

curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{"name": "commercial", "display_name": "Support Commercial"}' \
  "https://wazo.example.com:9486/api/confd/1.1/queueskills"

# =============================================================================
# ÉTAPE 2 : Créer la queue
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "support-unifie",
    "display_name": "Support Unifié",
    "context": "default",
    "options": [
      ["strategy", "leastrecent"],
      ["timeout", "25"],
      ["announce-position", "yes"]
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/queues"

# =============================================================================
# ÉTAPE 3 : Créer l'IVR de routage
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "accueil-support",
    "display_name": "Accueil Support",
    "context": "default",
    "choices": [
      {"exten": "1", "destination": {"type": "queue", "queue_id": 15}},
      {"exten": "2", "destination": {"type": "queue", "queue_id": 16}}
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/ivr"
```

---

## Résumé du Chapitre 5

| Ressource | Endpoints Clés | Point Critique |
|-----------|-----------------|----------------|
| **Queue** | `POST /queues` + `/queues/{id}/members/agents` | Stratégies, timeout, skills |
| **Ring Group** | `POST /groups` | Strategie sonnerie (`ringall`, `hunt`) |
| **IVR** | `POST /ivr` | Choices avec destinations imbriquées |
| **Conference** | `POST /conferences` | PIN, extension |
| **Schedule** | `POST /schedules` + `/timeperiods` + `/timerules` | Timezone, exceptions |
| **Skills** | `/queueskills` + `/agent_skills` | Skills-based routing |

---

# CHAPITRE 6 : Provisioning Physique (wazo-provd)

## 6.1 Introduction à wazo-provd

**wazo-provd** est le service de provisioning des terminaux physiques. Il génère les fichiers de configuration pour les téléphones SIP, ATA et gateways en se basant sur des plugins.

### 6.1.1 Architecture de Provisioning

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE PROVISIONING WAZO                            │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
    │   PHONE     │         │  wazo-provd │         │  wazo-confd │
    │  (Boot)    │         │  (Serveur)  │         │ (Config)    │
    └──────┬──────┘         └──────┬──────┘         └──────┬──────┘
           │                        │                        │
           │ 1. DHCP Request       │                        │
           │──────────────────────►│                        │
           │                        │                        │
           │ 2. DHCP Response      │                        │
           │   + HTTP URL          │                        │
           │◄──────────────────────│                        │
           │                        │                        │
           │ 3. HTTP Provisioning   │                        │
           │   Request             │                        │
           │──────────────────────►│                        │
           │                        │ 4. Get config         │
           │                        │──────────────────────►│
           │                        │◄───────────────────────│
           │                        │                       │
           │ 5. Generate PJSIP     │                       │
           │    + Base config       │                       │
           │◄───────────────────────│                       │
           │                        │                       │
           │ 6. Apply config       │                       │
           │   (Auto-register)     │                       │
```

### 6.1.2 Composants Clés

| Composant | Description |
|-----------|-------------|
| **Plugin** | Module spécifique à un modèle de téléphone (Snom, Yealink, Polycom) |
| **Template** | Fichiers de configuration Jinja2 générés par plugin |
| **Device** | Enregistrement du terminal (MAC, vendor, model) |
| **Configuration** | Ensemble des paramètres应用到 un device |

---

## 6.2 Gestion des Devices

### 6.2.1 CRUD des Devices

#### Endpoint

```http
GET/POST    /api/provd/0.1/devices
GET/PUT/DELETE /api/provd/0.1/devices/{device_id}
```

#### Création d'un Device

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "mac": "001122334455",
    "ip": "192.168.1.100",
    "vendor": "Snom",
    "model": "D345",
    "plugin": "snom",
    "description": "Telephone Alice",
    "template": "standard"
  }' \
  "https://wazo.example.com:8667/api/provd/0.1/devices"
```

#### Payload Détaillé

| Champ | Type | Description |
|-------|------|-------------|
| `mac` | `string` | Adresse MAC du terminal (format: `001122334455`) |
| `ip` | `string` | Adresse IP actuelle (optionnel, mis à jour automatiquement) |
| `vendor` | `string` | Fabricant (Snom, Yealink, Polycom, etc.) |
| `model` | `string` | Modèle spécifique |
| `plugin` | `string` | Plugin à utiliser (doit correspondre au vendor) |
| `template` | `string` | Template de configuration |
| `description` | `string` | Description libre |
| `status` | `string` | Statut: `autoprov`, `configured`, `waiting` |

#### Réponse

```json
{
  "id": "device-uuid-001",
  "mac": "001122334455",
  "ip": "192.168.1.100",
  "vendor": "Snom",
  "model": "D345",
  "plugin": "snom",
  "status": "autoprov",
  "template": null,
  "config_version": null,
  "created_at": "2026-03-07T15:30:00.000000Z",
  "updated_at": "2026-03-07T15:30:00.000000Z"
}
```

### 6.2.2 États d'un Device

| État | Description |
|------|-------------|
| `autoprov` | Device détecté, en attente de configuration |
| `waiting` | En attente de synchronisation |
| `configured` | Configuration appliquée |
| `failed` | Échec de configuration |

### 6.2.3 Liste des Plugins Disponibles

```bash
curl -k -X GET \
  -H "X-Auth-Token: {token}" \
  "https://wazo.example.com:8667/api/provd/0.1/plugins"
```

```json
{
  "items": [
    {"name": "snom", "version": "3.3.1"},
    {"name": "yealink", "version": "85.0.1.20"},
    {"name": "polycom", "version": "5.9.2"},
    {"name": "aastra", "version": "3.3.1-SP4"}
  ]
}
```

---

## 6.3 Mécanisme de Synchronisation

### 6.3.1 Processus de Synchronisation Complet

La synchronisation est le processus par lequel un terminal téléphone récupère et applique sa configuration.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PROCESSUS DE SYNCHRONISATION                                │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │   DEVICE    │     │  wazo-provd  │     │  wazo-confd  │
  │  (Telephone) │     │              │     │              │
  └──────┬───────┘     └──────┬───────┘     └──────────────┘
         │                      │                      │
         │  1. Boot & DHCP      │                      │
         │─────────────────────►│                      │
         │                      │                      │
         │  2. HTTP GET         │                      │
         │  /provd/.../mac      │                      │
         │─────────────────────►│                      │
         │                      │                      │
         │                      │  3. Lookup device    │
         │                      │─────────────────────►│
         │                      │◄─────────────────────│
         │                      │                      │
         │                      │  4. Generate config   │
         │                      │  (Jinja2 templates)  │
         │                      │                      │
         │  5. Return config    │                      │
         │  (SIP credentials,   │                      │
         │   proxy, codecs)    │                      │
         │◄─────────────────────│                      │
         │                      │                      │
         │  6. Apply & Register│                      │
         │  to SIP proxy       │                      │
         │                      │                      │
         │                      │  7. REGISTER event   │
         │                      │◄─────────────────────│
```

### 6.3.2 Étapes de Synchronisation API

#### Étape 1 : Associer une Ligne au Device (wazo-confd)

```bash
# Créer d'abord la ligne dans confd (voir Chapitre 3)
# ...
# Puis lier la ligne au device dans provd
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{"line_id": 42}' \
  "https://wazo.example.com:8667/api/provd/0.1/devices/device-uuid-001"
```

#### Étape 2 : Appliquer un Template

```bash
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{"template": "standard"}' \
  "https://wazo.example.com:8667/api/provd/0.1/devices/device-uuid-001/config"
```

#### Étape 3 : Déclencher la Synchronisation

```bash
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  "https://wazo.example.com:8667/api/provd/0.1/devices/device-uuid-001/synchronize"
```

> **⚠️ Attention** : La synchronisation déclenche un redémarrage du téléphone. Planifiez cette opération pendant les heures creuses.

#### Vérification du Statut

```bash
curl -k -X GET \
  -H "X-Auth-Token: {token}" \
  "https://wazo.example.com:8667/api/provd/0.1/devices/device-uuid-001"
```

```json
{
  "id": "device-uuid-001",
  "mac": "001122334455",
  "ip": "192.168.1.100",
  "status": "configured",
  "remote_address": "192.168.1.100:5060",
  "plugin": "snom",
  "template": "standard",
  "config_version": 17,
  "lines": [
    {
      "id": 42,
      "name": "alice-line-001",
      "exten": "1001"
    }
  ],
  "updated_at": "2026-03-07T16:00:00.000000Z"
}
```

### 6.3.3 Détection Automatique (Auto-provisioning)

Wazo détecte automatiquement les nouveaux appareils via :

1. **DHCP** : Le serveur DHCP informe provd des nouvelles demandes
2. **HTTP** : Le téléphone boot et contacte le serveur provisioning

Pour désactiver l'auto-provisioning :

```bash
# Via configuration système
# /etc/wazo-provd/conf.d/custom.yml
enabled_autoprov: false
```

---

## 6.4 Templates de Configuration

### 6.4.1 Concept des Templates

Les templates définissent la configuration appliquée à un device. Ils utilisent le moteur **Jinja2**.

#### Hiérarchie des Templates

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  HIÉRARCHIE TEMPLATES WAZO                                   │
└─────────────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────┐
                    │   GLOBAL        │
                    │  (système)      │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
              ┌─────▼─────┐     ┌───▼────┐
              │ PLUGIN    │     │ CUSTOM  │
              │ Templates │     │Templates│
              └─────┬─────┘     └───┬────┘
                    │                 │
                    └────────┬────────┘
                             │
                      ┌──────▼──────┐
                      │  DEVICE     │
                      │  (final)    │
                      └─────────────┘
```

### 6.4.2 Création d'un Template Personnalisé

Les templates personnalisés s'ajoutent au niveau du plugin :

```bash
# Emplacement des templates
/var/lib/wazo-provd/plugins/wazo-sn

om-3.3.1/templates/

# Créer un template personnalisé
mkdir -p /var/lib/wazo-provd/plugins/wazo-sn

om-3.3.1/templates/custom
```

#### Exemple de Template Personnalisé

```jinja2
{# /var/lib/wazo-provd/plugins/wazo-sn

om-3.3.1/templates/custom/base.tpl #}
{% extends "base.tpl" %}

{% block sip_settings %}
{{ parent() }}
phone_setting.display_method: 0
phone_setting.backlight_level: 3
{% endblock %}
```

### 6.4.3 Variables de Template

| Variable | Description | Exemple |
|----------|-------------|---------|
| `{{ line.endpoint.username }}` | Identifiant SIP | `alice_auth` |
| `{{ line.endpoint.password }}` | Mot de passe SIP | `P4ssw0rd!` |
| `{{ line.extension }}` | Numéro interne | `1001` |
| `{{ wazo_server_ip }}` | IP serveur Wazo | `192.168.1.1` |
| `{{ wazo_proxy_ip }}` | IP du proxy SIP | `192.168.1.1` |

---

## 6.5 Association Ligne-Terminal

### 6.5.1 Linking Device ↔ Line

L'association device↔line peut se faire de deux manières :

#### Méthode 1 : Via provd (Provisioning)

```bash
# Dans provd - Associer une ligne existante
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{"line_id": 42}' \
  "https://wazo.example.com:8667/api/provd/0.1/devices/device-uuid-001"
```

#### Méthode 2 : Via confd (Configuration)

```bash
# Dans confd - Associer un device à une ligne
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{"device_id": "device-uuid-001"}' \
  "https://wazo.example.com:9486/api/confd/1.1/lines/42/device"
```

> **Note** : Les deux méthodes synchronisent automatiquement l'autre côté.

---

## 6.6 Scénario Complet : Provisioning d'un Téléphone

```bash
# =============================================================================
# ÉTAPE 1 : Créer l'endpoint SIP dans confd
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "snom-d345-001",
    "auth_section_options": [
      ["username", "alice_sip"],
      ["password", "S3cur3P4ssw0rd!"]
    ],
    "endpoint_section_options": [
      ["disallow", "all"],
      ["allow", "ulaw,alaw,g722"],
      ["context", "default"]
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/endpoints/sip"

# Réponse : {"uuid": "endpoint-sip-uuid", ...}


# =============================================================================
# ÉTAPE 2 : Créer la ligne SIP
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "context": "default",
    "name": "alice-line-001",
    "protocol": "sip"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/lines"

# Réponse : {"id": 42, ...}


# =============================================================================
# ÉTAPE 3 : Lier endpoint à la ligne
# =============================================================================
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/lines/42/endpoints/sip/endpoint-sip-uuid"


# =============================================================================
# ÉTAPE 4 : Créer l'extension
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "exten": "1001",
    "context": "default"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/extensions"

# Réponse : {"id": 88, ...}


# =============================================================================
# ÉTAPE 5 : Lier extension à la ligne
# =============================================================================
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/lines/42/extensions/88"


# =============================================================================
# ÉTAPE 6 : Créer le device dans provd
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "mac": "001122334455",
    "vendor": "Snom",
    "model": "D345",
    "plugin": "snom",
    "description": "Telephone Alice - Bureau Paris"
  }' \
  "https://wazo.example.com:8667/api/provd/0.1/devices"

# Réponse : {"id": "device-uuid-001", "status": "autoprov", ...}


# =============================================================================
# ÉTAPE 7 : Associer la ligne au device
# =============================================================================
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{"line_id": 42}' \
  "https://wazo.example.com:8667/api/provd/0.1/devices/device-uuid-001"


# =============================================================================
# ÉTAPE 8 : Appliquer le template et synchroniser
# =============================================================================
# Appliquer template
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{"template": "standard"}' \
  "https://wazo.example.com:8667/api/provd/0.1/devices/device-uuid-001/config"

# Synchroniser (déclenche reboot du téléphone)
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  "https://wazo.example.com:8667/api/provd/0.1/devices/device-uuid-001/synchronize"


# =============================================================================
# ÉTAPE 9 : Vérifier le status final
# =============================================================================
curl -k -X GET \
  -H "X-Auth-Token: {token}" \
  "https://wazo.example.com:8667/api/provd/0.1/devices/device-uuid-001"

# Réponse attendue :
# {
#   "id": "device-uuid-001",
#   "status": "configured",
#   "ip": "192.168.1.100",
#   "remote_address": "192.168.1.100:5060",
#   "lines": [{"id": 42, "exten": "1001"}]
# }
```

---

## 6.7 Troubleshooting Provisioning

### 6.7.1 Commandes de Diagnostic

```bash
# Voir les logs de provisioning
journalctl -u wazo-provd -f

# Liste des devices avec status
curl -k -H 'X-Auth-Token: {token}' \
  "https://wazo.example.com:8667/api/provd/0.1/devices" | jq '.items[].status'

# Vérifier plugin installé
curl -k -H 'X-Auth-Token: {token}' \
  "https://wazo.example.com:8667/api/provd/0.1/plugins" | jq '.items[].name'
```

### 6.7.2 Problèmes Courants

| Problème | Cause | Solution |
|----------|-------|----------|
| Device toujours `autoprov` | Plugin manquant | Installer le plugin correspondant |
| Échec `synchronize` | Timeout réseau | Vérifier connectivité téléphone |
| Pas de registration | Mauvais credentials | Vérifier endpoint SIP dans confd |
| Config non appliquée | Template invalide | Vérifier syntaxe Jinja2 |

---

## Résumé du Chapitre 6

| Ressource | Endpoints Clés | Point Critique |
|-----------|-----------------|----------------|
| **Device** | `POST /devices`, `/devices/{id}/synchronize` | MAC obligatoire |
| **Plugin** | `/plugins` | Doit correspondre au vendor |
| **Template** | `/devices/{id}/config` | Jinja2, hérité plugin |
| **Association** | `PUT /devices/{id}` avec `line_id` | Sync auto confd↔provd |
| **Synchronisation** | `/devices/{id}/synchronize` | Déclenche reboot |

---

## Résumé des Chapitres 5 & 6

### Services Avancés (Chapitre 5)

| Objet | Relations | Clé |
|-------|-----------|-----|
| Queue | → Agents, Skills, Schedule | `strategy`, `timeout` |
| Ring Group | → Users, Extensions | `extension`, `strategy` |
| IVR | → Destinations imbriquées | `choices[]` |
| Conference | → Extension | `pin` |
| Schedule | → Timeperiods, Timerules | `timezone` |

### Provisioning (Chapitre 6)

| Objet | Relations | Clé |
|-------|-----------|-----|
| Device | → Plugin, Template, Line | `mac`, `status` |
| Plugin | → Templates | Vendor/Model |
| Template | → Jinja2 vars | Héritage |
| Synchronisation | → reboot automatique | `/synchronize` |

---

*Fin des Chapitres 5 et 6 — Suite : Chapitre 7 (CTI, WebSockets, Temps Réel)*