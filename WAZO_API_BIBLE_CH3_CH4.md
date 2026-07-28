---

# CHAPITRE 3 : Configuration Core (wazo-confd) — Utilisateurs et Routage

## 3.1 Introduction à wazo-confd

**wazo-confd** est le service central de configuration de la plateforme Wazo. Il gère l'intégralité des ressources liées à la téléphonies IP : utilisateurs telephony, lignes, extensions, endpoints SIP, trunks, files d'attente, IVR, conférences, etc.

> **Rappel fondamental** : Ce chapitre détaille les objets de **routage interne** (contextes, extensions, lignes, endpoints, utilisateurs) ainsi que les **restrictions d'appels** (call permissions). Le Chapitre 4 couvrira les communications avec l'extérieur (trunks, outcalls, incalls).

### 3.1.1 Portée du Service

| Catégorie | Ressources gérées |
|-----------|-------------------|
| **Utilisateurs** | Utilisateurs telephony, voicemails, forwards, DND |
| **Lignes** | Lignes SIP, SCCP, IAX, personnalisées |
| **Endpoints** | SIP (PJSIP), SCCP, IAX, Custom |
| **Extensions** | Numéros internes, extensions de routage |
| **Routage** | Contextes, incoming calls, outgoing calls |
| **Services** | Queues, Groups, IVR, Conferences, Schedules |
| **Audio** | Sounds, Music on Hold |
| **Configuration** | SIP templates, fonction keys, call pickups |

---

## 3.2 Les Contextes (Contexts)

### 3.2.1 Concept

Les **contextes** (contexts) sont le fondement du routage téléphonique dans Wazo/Asterisk. Un contexte définit un "domaine logique" d extensions — un périmètre à l'intérieur duquel les appels peuvent transiter selon des règles définies.

#### Types de Contextes

| Type | Nom usuel | Rôle |
|------|-----------|------|
| **Interne** | `default`, `interne` | Communications entre extensions internes du même site |
| **Entrant (DID)** | `from-extern`, `from-did` | Appels entrants depuis l'extérieur (trunk) |
| **Sortant** | `outside`, `externe` | Appels sortants vers l'extérieur via trunk |
| **Service** | ` voicemail`, `parkedcalls` | Contextes système pour services particuliers |

#### Schéma de Routage par Contextes

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      ROUTAGE PAR CONTEXTES WAZO                           │
└─────────────────────────────────────────────────────────────────────────────┘

   APPEL ENTRANT                    APPEL INTERNE                   APPEL SORTANT
   (from-extern)                    (default)                        (outside)
         │                              │                               │
         ▼                              ▼                               ▼
  ┌─────────────┐               ┌─────────────┐                ┌─────────────┐
  │   INCALL    │               │  EXTENSION │                │  OUTCALL    │
  │  (DID/SDAs) │               │  (1001-    │                │  (Strip,    │
  │             │               │   1099)    │                │   Prefix)   │
  └──────┬──────┘               └──────┬──────┘                └──────┬──────┘
         │                              │                               │
         └──────────────┬──────────────┘                               │
                        │                                              │
                        ▼                                              ▼
               ┌────────────────┐                              ┌────────────────┐
               │   DESTINATION │                              │     TRUNK      │
               │  User/Queue/ │◄─────────────────────────────┤    (SIP/IAX)   │
               │    IVR/Group │                               │                │
               └────────────────┘                               └────────────────┘
```

### 3.2.2 CRUD des Contextes

#### Endpoint

```http
GET/POST /api/confd/1.1/contexts
GET/PUT/DELETE /api/confd/1.1/contexts/{context_id}
```

#### Liste des Contextes

```bash
curl -k -X GET \
  -H "Accept: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/contexts"
```

#### Création d'un Contexte

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "bureau-paris",
    "display_name": "Bureau Paris",
    "context_type": "internal",
    "description": "Contexte interne du bureau de Paris"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/contexts"
```

#### Payload Complet

| Champ | Type | Obligatoire | Description |
|-------|------|-------------|-------------|
| `name` | `string` | **Oui** | Identifiant technique unique (ex: `bureau-paris`) |
| `display_name` | `string` | Non | Nom affiché dans l'interface |
| `context_type` | `string` | Non | Type : `internal`, `external`, `others` |
| `description` | `string` | Non | Description textuelle |

#### Réponse (201 Created)

```json
{
  "id": 5,
  "name": "bureau-paris",
  "display_name": "Bureau Paris",
  "context_type": "internal",
  "description": "Contexte interne du bureau de Paris",
  "tenant_uuid": "tenant-uuid-main"
}
```

> **⚠️ Attention** : Les contextes `default`, `from-extern` et `outside` sont créés par défaut lors de l'installation. Ne les supprimez pas — ils sont requis pour le fonctionnement de base.

---

## 3.3 La Trinité Wazo : Endpoints SIP, Lignes et Extensions

Dans Wazo, toute ligne téléphonique fonctionnel repose sur l'assemblage de trois objets distincts mais interdépendants :

1. **Endpoint SIP** — Configuration technique PJSIP (authentification, codec, DTMF)
2. **Ligne** — Objet logique reliant l'endpoint à l'extension
3. **Extension** — Numéro de téléphone joignable

> **Règle d'or** : Une ligne **doit** être liée à un endpoint SIP **et** à une extension pour être joignable et permettant les appels.

### 3.3.1 Endpoints SIP (PJSIP)

#### Concept

Un **endpoint SIP** représente la configuration technique complète d'un dispositif SIP dans Asterisk via le module **PJSIP**. Cette configuration est structurée en "sections" PJSIP.

#### Architecture des Sections PJSIP

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  CONFIGURATION PJSIP D'UN ENDPOINT                         │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   AUTH SECTION │     │   AOR SECTION  │     │ ENDPOINT SECTION│
│                 │     │                │     │                 │
│ - username      │     │ - contact      │     │ - context       │
│ - password      │     │ - max_contacts │     │ - disallow      │
│ - realm        │     │ - qualify       │     │ - allow         │
│                 │     │ - expiry        │     │ - callerid      │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                      │                      │
         └──────────────────────┼──────────────────────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │   PJSIP ENDPOINT    │
                    │   (Référence les    │
                    │    3 sections)      │
                    └─────────────────────┘
```

#### CRUD des Endpoints SIP

##### Endpoint

```http
GET/POST    /api/confd/1.1/endpoints/sip
GET/PUT/DELETE /api/confd/1.1/endpoints/sip/{endpoint_uuid}
```

##### Liste des Endpoints SIP

```bash
curl -k -X GET \
  -H "Accept: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/endpoints/sip"
```

##### Création d'un Endpoint SIP Complet

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "alice-sip-001",
    "auth_section_options": [
      ["username", "alice_auth"],
      ["password", "P4ssw0rd!SIP2026"],
      ["realm", "asterisk"]
    ],
    "aor_section_options": [
      ["max_contacts", "1"],
      ["remove_existing", "yes"],
      ["qualify_frequency", "60"],
      ["expiry", "3600"]
    ],
    "endpoint_section_options": [
      ["disallow", "all"],
      ["allow", "ulaw,alaw,g722,h264"],
      ["context", "default"],
      ["dtmf_mode", "rfc4733"],
      ["direct_media", "no"],
      ["callerid", "Alice Dupont <1001>"],
      ["call_forward", "yes"],
      ["call_transfer", "yes"],
      ["force_rport", "yes"],
      ["rewrite_contact", "yes"],
      ["ice_support", "yes"],
      ["candiate_acl", "any"]
    ],
    "registration_section_options": [
      ["server_uri", "sip:provider.example.com:5060"],
      ["client_uri", "sip:alice@provider.example.com"],
      ["contact_uri", "sip:alice@192.168.1.50:5060"]
    ],
    "transport": null,
    "templates": ["standard"]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/endpoints/sip"
```

#### Payload Détaillé des Sections PJSIP

##### auth_section_options (Authentification)

| Option | Type | Description | Exemple |
|--------|------|-------------|---------|
| `username` | `string` | Identifiant d'authentification SIP | `alice_auth` |
| `password` | `string` | Mot de passe SIP | `P4ssw0rd!` |
| `realm` | `string` | Realm d'authentification (optionnel) | `asterisk` |

##### aor_section_options (Address of Record)

| Option | Type | Description | Exemple |
|--------|------|-------------|---------|
| `max_contacts` | `string` | Nombre max de contacts simultanés | `1` |
| `remove_existing` | `string` | Supprimer les anciens contacts | `yes` |
| `qualify_frequency` | `string` | Fréquence de qualification (secondes) | `60` |
| `expiry` | `string` | Expiration des registrations | `3600` |

##### endpoint_section_options (Configuration de l'endpoint)

| Option | Type | Description | Valeurs possibles |
|--------|------|-------------|-------------------|
| `disallow` | `string` | Désactiver tous les codecs | `all` |
| `allow` | `string` | Activer les codecs | `ulaw,alaw,g722,h264` |
| `context` | `string` | Contexte de routage | `default`, `from-extern` |
| `dtmf_mode` | `string` | Mode DTMF | `rfc4733`, `info`, `inband` |
| `direct_media` | `string` | Media direct peer-to-peer | `yes`, `no` |
| `callerid` | `string` | Caller ID par défaut | `Alice <1001>` |
| `force_rport` | `string` | Forcer le rport | `yes` |
| `rewrite_contact` | `string` | Réécrire le contact | `yes` |
| `ice_support` | `string` | Support ICE pour STUN | `yes` |

> **⚠️ Attention critique** : Le mot de passe SIP doit respecter les politiques du fournisseur/OP. Certains providers refusent les caractères spéciaux ou exigent une longueur minimale.

#### Réponse (201 Created)

```json
{
  "uuid": "endpoint-uuid-1234",
  "name": "alice-sip-001",
  "auth_section_options": [
    ["username", "alice_auth"],
    ["password", "***"]
  ],
  "aor_section_options": [
    ["max_contacts", "1"]
  ],
  "endpoint_section_options": [
    ["disallow", "all"],
    ["allow", "ulaw,alaw,g722"]
  ],
  "transport": null,
  "templates": ["standard"],
  "tenant_uuid": "tenant-uuid-main",
  "created_at": "2026-03-07T15:30:00.000000Z"
}
```

##### Modification d'un Endpoint SIP

```bash
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "endpoint_section_options": [
      ["callerid", "Alice Dupont <1002>"]
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/endpoints/sip/{endpoint_uuid}"
```

##### Suppression d'un Endpoint SIP

```bash
curl -k -X DELETE \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/endpoints/sip/{endpoint_uuid}"
```

### 3.3.2 Lignes (Lines)

#### Concept

Une **ligne** (line) est l'objet central qui relie :
- L'endpoint SIP (configuration technique)
- L'extension (numéro de téléphone)
- L'utilisateur (agent telephony)

#### CRUD des Lignes

##### Endpoint

```http
GET/POST    /api/confd/1.1/lines
GET/POST    /api/confd/1.1/lines/sip    (création rapide ligne SIP)
GET/PUT/DELETE /api/confd/1.1/lines/{line_id}
```

##### Création Rapide d'une Ligne SIP

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "context": "default",
    "name": "alice-line-001",
    "protocol": "sip"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/lines/sip"
```

#### Payload Complet d'une Ligne

| Champ | Type | Obligatoire | Description |
|-------|------|-------------|-------------|
| `context` | `string` | **Oui** | Contexte de routage |
| `name` | `string` | Non | Nom logique de la ligne |
| `protocol` | `string` | **Oui** | `sip`, `sccp`, `iax`, `custom` |
| `device` | `uuid` | Non | UUID du device associé |
| `description` | `string` | Non | Description |

#### Réponse (201 Created)

```json
{
  "id": 42,
  "context": "default",
  "name": "alice-line-001",
  "protocol": "sip",
  "device": null,
  "description": null,
  "tenant_uuid": "tenant-uuid-main"
}
```

### 3.3.3 Extensions

#### Concept

Une **extension** est un numéro de téléphone associé à une ligne. Elle peut être :
- **Interne** : joignable depuis l'intérieur du système
- **Partie d'un routage entrant** : destination d'un DID/SDD

#### CRUD des Extensions

##### Endpoint

```http
GET/POST    /api/confd/1.1/extensions
GET/PUT/DELETE /api/confd/1.1/extensions/{extension_id}
```

##### Création d'une Extension

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "exten": "1001",
    "context": "default",
    "description": "Extension principale Alice"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/extensions"
```

#### Payload

| Champ | Type | Obligatoire | Description |
|-------|------|-------------|-------------|
| `exten` | `string` | **Oui** | Numéro de l'extension |
| `context` | `string` | **Oui** | Contexte de routage |
| `description` | `string` | Non | Description |

#### Réponse

```json
{
  "id": 88,
  "exten": "1001",
  "context": "default",
  "description": "Extension principale Alice",
  "tenant_uuid": "tenant-uuid-main"
}
```

### 3.3.4 Associations — La Trinité en Pratique

L'assemblage des trois objets nécessite des appels API spécifiques :

#### Association Ligne ↔ Endpoint SIP

```http
PUT /api/confd/1.1/lines/{line_id}/endpoints/sip/{endpoint_uuid}
```

```bash
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/lines/42/endpoints/sip/endpoint-uuid-1234"
```

**Réponse (204 No Content)** : Association créée.

#### Association Ligne ↔ Extension

```http
PUT /api/confd/1.1/lines/{line_id}/extensions/{extension_id}
```

```bash
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/lines/42/extensions/88"
```

#### Dissociation Ligne ↔ Extension

```http
DELETE /api/confd/1.1/lines/{line_id}/extensions/{extension_id}
```

```bash
curl -k -X DELETE \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/lines/42/extensions/88"
```

---

## 3.4 Utilisateurs Telephony (wazo-confd)

### 3.4.1 Distinction Importante

> **Rappel** : Il existe **deux types d'utilisateurs** dans Wazo :
> 
> - **Utilisateurs d'authentification** (wazo-auth) — `/api/auth/0.1/users` — Permettent l'accès à l'API
> - **Utilisateurs telephony** (wazo-confd) — `/api/confd/1.1/users` — Représentent les agents dans le système de communications

Cette section concerne les **utilisateurs telephony**.

### 3.4.2 CRUD des Utilisateurs Telephony

#### Endpoint

```http
GET/POST    /api/confd/1.1/users
GET/PUT/DELETE /api/confd/1.1/users/{user_uuid}
```

#### Création d'un Utilisateur Simple

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "firstname": "Alice",
    "lastname": "Dupont",
    "email": "alice@example.com",
    "language": "fr_FR",
    "timezone": "Europe/Paris",
    "enabled": true,
    "caller_id_name": "Alice Dupont",
    "caller_id_number": "1001"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/users"
```

#### Payload Complet

| Champ | Type | Obligatoire | Description |
|-------|------|-------------|-------------|
| `firstname` | `string` | **Oui** | Prénom |
| `lastname` | `string` | Non | Nom |
| `email` | `string` | Non | Adresse email |
| `language` | `string` | Non | Langue (`fr_FR`, `en_US`, etc.) |
| `timezone` | `string` | Non | Fuseau horaire |
| `enabled` | `boolean` | Non (défaut: `true`) | Utilisateur activé |
| `caller_id_name` | `string` | Non | Nom affiché Caller ID (ex: `"Alice Dupont"`) |
| `caller_id_number` | `string` | Non | Numéro affiché Caller ID (ex: `"1001"`) |
| `calling_login`
| `calling_password` | `string` | Non | Mot de passe agent |
| `purpose` | `string` | Non (défaut: `user`) | `user` ou `warmline` |

#### Réponse (201 Created)

```json
{
  "uuid": "user-uuid-1234",
  "firstname": "Alice",
  "lastname": "Dupont",
  "email": "alice@example.com",
  "language": "fr_FR",
  "timezone": "Europe/Paris",
  "enabled": true,
  "caller_id": {
    "display_name": "Alice Dupont",
    "internal": false
  },
  "tenant_uuid": "tenant-uuid-main"
}
```

### 3.4.3 Association Utilisateur ↔ Ligne

Pour qu'un utilisateur puisse passer et recevoir des appels, il doit être lié à au moins une ligne.

#### Association

```http
PUT /api/confd/1.1/users/{user_uuid}/lines/{line_id}
```

```bash
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/users/user-uuid-1234/lines/42"
```

#### Liste des Lignes d'un Utilisateur

```bash
curl -k -X GET \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/users/user-uuid-1234/lines"
```

#### Dissociation

```http
DELETE /api/confd/1.1/users/{user_uuid}/lines/{line_id}
```

---

## 3.5 Voicemails

### 3.5.1 Concept

Un **voicemail** est une boîte vocale associée à un utilisateur. Elle permet :
- La réception de messages vocaux
- La notification par email
- L'accès par code PIN

### 3.5.2 CRUD des Voicemails

#### Endpoint

```http
GET/POST    /api/confd/1.1/voicemails
GET/PUT/DELETE /api/confd/1.1/voicemails/{voicemail_id}
```

#### Création d'un Voicemail

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "alice_vm",
    "number": "1001",
    "context": "default",
    "password": "1234",
    "email": "alice@example.com",
    "attach_audio": true,
    "delete_messages": false,
    "max_messages": 50,
    "announcement": null,
    "skip_instructions": false
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/voicemails"
```

#### Payload Détaillé

| Champ | Type | Obligatoire | Description |
|-------|------|-------------|-------------|
| `name` | `string` | **Oui** | Nom de la boîte vocale |
| `number` | `string` | **Oui** | Numéro (généralement same que l'extension) |
| `context` | `string` | **Oui** | Contexte |
| `password` | `string` | Non | PIN d'accès (4 chiffres) |
| `email` | `string` | Non | Email de notification |
| `attach_audio` | `boolean` | Non | Joindre fichier audio à l'email |
| `delete_messages` | `boolean` | Non | Supprimer après notification |
| `max_messages` | `int` | Non | Nb max de messages |

#### Réponse

```json
{
  "id": 15,
  "name": "alice_vm",
  "number": "1001",
  "context": "default",
  "email": "alice@example.com",
  "attach_audio": true,
  "tenant_uuid": "tenant-uuid-main"
}
```

### 3.5.3 Association Voicemail ↔ Utilisateur

#### Associer une boîte
```http
PUT /api/confd/1.1/users/{user_uuid}/voicemails/{voicemail_id}
```

#### Dissocier une boîte
```http
DELETE /api/confd/1.1/users/{user_uuid}/voicemails
```

```bash
curl -k -X DELETE \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/users/user-uuid-1234/voicemails"
```

---

## 3.6 Call Permissions (Restrictions d'Appels)

### 3.6.1 Concept

Les **call permissions** (permissions d'appels) permettent de contrôler quels numéros/tendus peuvent être appelés par un utilisateur. C'est un système de **whitelist** (autorisation) ou **blacklist** (interdiction).

### 3.6.2 CRUD des Call Permissions

#### Endpoint

```http
GET/POST    /api/confd/1.1/callpermissions
GET/PUT/DELETE /api/confd/1.1/callpermissions/{permission_id}
```

#### Création d'une Permission

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "appel-international",
    "description": "Autorise les appels internationaux"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/callpermissions"
```

#### Réponse

```json
{
  "id": 3,
  "name": "appel-international",
  "description": "Autorise les appels internationaux",
  "tenant_uuid": "tenant-uuid-main"
}
```

### 3.6.3 Régles de Permission (Rules)

Chaque call permission contient des règles définissant les préfixes autorisés/interdits.

#### Création d'une Règle

```http
POST /api/confd/1.1/callpermissions/{permission_id}/rules
```

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "prefix": "00",
    "action": "allow"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/callpermissions/3/rules"
```

#### Payload des Règles

| Champ | Type | Description |
|-------|------|-------------|
| `prefix` | `string` | Préfixe numérique (ex: `00`, `0`, `0800`) |
| `action` | `string` | `allow` ou `deny` |

#### Liste des Règles

```bash
curl -k -X GET \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/callpermissions/3/rules"
```

### 3.6.4 Association Permission ↔ Utilisateur

```http
PUT /api/confd/1.1/users/{user_uuid}/callpermissions/{permission_id}
```

```bash
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/users/user-uuid-1234/callpermissions/3"
```

---

## 3.7 Scénario Complet : Création d'un Utilisateur Opérationnel

Voici la séquence complète pour créer un utilisateur telephony avec ligne SIP et voicemail :

```bash
# =============================================================================
# ÉTAPE 1 : Créer l'endpoint SIP
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "alice-sip-001",
    "auth_section_options": [
      ["username", "alice_auth"],
      ["password", "P4ssw0rd!SIP"]
    ],
    "endpoint_section_options": [
      ["disallow", "all"],
      ["allow", "ulaw722"],
      ["context", "default"],
      ["c,alaw,gallerid", "Alice Dupont <1001>"]
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/endpoints/sip"

# Réponse : {"uuid": "endpoint-uuid-001", ...}
# ↑ NOTER : endpoint-uuid-001


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
# ↑ NOTER : line-id = 42


# =============================================================================
# ÉTAPE 3 : Lier endpoint à la ligne
# =============================================================================
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/lines/42/endpoints/sip/endpoint-uuid-001"
# Réponse : 204 No Content


# =============================================================================
# ÉTAPE 4 : Créer l'extension
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "exten": "1001",
    "context": "default",
    "description": "Extension Alice"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/extensions"

# Réponse : {"id": 88, ...}
# ↑ NOTER : extension-id = 88


# =============================================================================
# ÉTAPE 5 : Lier extension à la ligne
# =============================================================================
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/lines/42/extensions/88"
# Réponse : 204 No Content


# =============================================================================
# ÉTAPE 6 : Créer l'utilisateur
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "firstname": "Alice",
    "lastname": "Dupont",
    "email": "alice@example.com",
    "language": "fr_FR"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/users"

# Réponse : {"uuid": "user-uuid-001", ...}
# ↑ NOTER : user-uuid-001


# =============================================================================
# ÉTAPE 7 : Lier ligne à l'utilisateur
# =============================================================================
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/users/user-uuid-001/lines/42"
# Réponse : 204 No Content


# =============================================================================
# ÉTAPE 8 : Créer le voicemail (optionnel)
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "alice_vm",
    "number": "1001",
    "context": "default",
    "password": "1234",
    "email": "alice@example.com",
    "attach_audio": true
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/voicemails"

# Réponse : {"id": 15, ...}
# ↑ NOTER : voicemail-id = 15


# =============================================================================
# ÉTAPE 9 : Lier voicemail à l'utilisateur
# =============================================================================
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/users/user-uuid-001/voicemails/15"
# Réponse : 204 No Content
```

---

## Résumé du Chapitre 3

| Ressource | Endpoints Clés | Point Critique |
|-----------|-----------------|----------------|
| **Contextes** | `POST /contexts` | Contextes `default`, `from-extern` requis |
| **Endpoints SIP** | `POST /endpoints/sip` | Sections PJSIP (auth, aor, endpoint) |
| **Lignes** | `POST /lines/sip` | Protocole obligatoire (`sip`) |
| **Extensions** | `POST /extensions` | Contexte obligatoire |
| **Associations** | `PUT /lines/{id}/endpoints/sip/{uuid}` | Ordre : endpoint → ligne → extension → user |
| **Utilisateurs** | `POST /users` | Distincts des utilisateurs auth |
| **Voicemails** | `POST /voicemails` | Lié à l'utilisateur, pas à la ligne |
| **Call Permissions** | `POST /callpermissions/{id}/rules` | Système whitelist/blacklist par préfixes |

---

# CHAPITRE 4 : Trunks SIP, Outcalls et Incalls

## 4.1 Introduction

Ce chapitre couvre les communications entre Wazo et l'extérieur :

- **Trunks** : Connexions SIP vers les fournisseurs/opérateurs
- **Outcalls** : Routage des appels sortants (vers l'extérieur)
- **Incalls** : Routage des appels entrants (depuis l'extérieur via DID/SDD)

### 4.1.1 Architecture de Communication

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   ARCHITECTURE COMMUNICATIONS WAZO                          │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌─────────────┐                         ┌─────────────┐
    │   APPEL     │                         │   APPEL     │
    │  ENTRANT   │                         │  SORTANT    │
    └──────┬──────┘                         └──────┬──────┘
           │                                        │
           ▼                                        ▼
    ┌─────────────┐                         ┌─────────────┐
    │   INCALL    │                         │   OUTCALL  │
    │  (DID/SDD)  │                         │  (Plans de │
    └──────┬──────┘                         │   numérotie)
           │                                └──────┬──────┘
           ▼                                        ▼
    ┌─────────────┐                         ┌─────────────┐
    │  CONTEXT:   │                         │  CONTEXT:   │
    │ from-extern │                         │   outside   │
    └──────┬──────┘                         └──────┬──────┘
           │                                        │
           └───────────────┬────────────────────────┘
                           │
                           ▼
                   ┌─────────────┐
                   │    TRUNK    │
                   │    (SIP)    │
                   └──────┬──────┘
                           │
                           ▼
                   ┌─────────────┐
                   │  OPERATOR / │
                   │   PROVIDER  │
                   │  (SIP trunk)│
                   └─────────────┘
```

---

## 4.2 Trunks SIP

### 4.2.1 Concept

Un **trunk** est une connexion SIP reliant Wazo à :
- Un opérateur VoIP (SIP trunking)
- Une autre PBX (interconnexion)
- Un service de terminaison (ITSP)

### 4.2.2 Structure d'un Trunk SIP

Un trunk SIP se compose de plusieurs éléments :

| Composant | Endpoint API | Description |
|-----------|-------------|-------------|
| **Endpoint SIP** | `/endpoints/sip` | Configuration technique du trunk |
| **Registration** | `/registrations` | Configuration SIP REGISTER (si requis) |
| **Trunk** | `/trunks` | Objet logique regroupant les éléments |

### 4.2.3 CRUD des Trunks

#### Endpoint

```http
GET/POST    /api/confd/1.1/trunks
GET/PUT/DELETE /api/confd/1.1/trunks/{trunk_id}
```

#### Création d'un Trunk SIP (avec registration)

```bash
# =============================================================================
# ÉTAPE 1 : Créer l'endpoint SIP du trunk
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "trunk-operateur-001",
    "endpoint_section_options": [
      ["disallow", "all"],
      ["allow", "ulaw,alaw,g722"],
      ["context", "from-extern"],
      ["dtmf_mode", "rfc4733"],
      ["trust_connected_line", "yes"]
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/endpoints/sip"

# Réponse : {"uuid": "trunk-endpoint-uuid", ...}
# ↑ NOTER : trunk-endpoint-uuid


# =============================================================================
# ÉTAPE 2 : Créer le trunk
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "Trunk Opérateur",
    "endpoint_sip_uuid": "trunk-endpoint-uuid",
    "context": "from-extern"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/trunks"

# Réponse : {"id": 5, "name": "Trunk Opérateur", ...}
```

#### Payload du Trunk

| Champ | Type | Obligatoire | Description |
|-------|------|-------------|-------------|
| `name` | `string` | **Oui** | Nom du trunk |
| `endpoint_sip_uuid` | `uuid` | Non | UUID de l'endpoint SIP associé |
| `endpoint_iax_uuid` | `uuid` | Non | UUID de l'endpoint IAX associé |
| `endpoint_custom_uuid` | `uuid` | Non | UUID de l'endpoint custom |
| `context` | `string` | Non | Contexte de routage entrant |

#### Trunk avec Registration (SIP REGISTER)

Si votre opérateur nécessite une registration SIP :

```bash
# =============================================================================
# ÉTAPE 1 : Créer l'endpoint SIP
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "trunk-registered",
    "endpoint_section_options": [
      ["context", "from-extern"]
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/endpoints/sip"

# Réponse : {"uuid": "endpoint-uuid", ...}


# =============================================================================
# ÉTAPE 2 : Créer la registration
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "endpoint_sip_uuid": "endpoint-uuid",
    "registration_section_options": [
      ["server_uri", "sip:sip.operator.com:5060"],
      ["client_uri", "sip:moncompte@sip.operator.com"],
      ["contact_uri", "sip:moncompte@192.168.1.10:5060"],
      ["expiry", "3600"]
    ],
    "auth_section_options": [
      ["username", "moncompte"],
      ["password", "motdepasse"]
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/registrations"

# Réponse : {"id": 10, ...}
# ↑ NOTER : registration-id = 10


# =============================================================================
# ÉTAPE 3 : Créer le trunk
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "Trunk Opérateur Enregistré",
    "endpoint_sip_uuid": "endpoint-uuid",
    "context": "from-extern"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/trunks"
```

### 4.2.4 Transport SIP (Optionnel)

Pour les trunks utilisant un transport spécifique (TLS, TCP) :

#### Création d'un Transport

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "transport-tls",
    "type": "transport-tls",
    "options": [
      ["bind", "0.0.0.0:5061"],
      ["protocol", "tls"],
      ["cert_file", "/etc/asterisk/keys/asterisk.crt"],
      ["priv_key_file", "/etc/asterisk/keys/asterisk.key"],
      ["method", "tlsv1_2"]
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/sip/transports"
```

#### Association Transport à un Endpoint

```bash
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{"transport": "transport-tls-uuid"}' \
  "https://wazo.example.com:9486/api/confd/1.1/endpoints/sip/{endpoint_uuid}"
```

---

## 4.3 Outcalls (Appels Sortants)

### 4.3.1 Concept

Les **outcalls** (appelés aussi "outgoing calls" ou "dial patterns") définissent les règles de numérotation pour les appels sortants. Ils permettent de :
- Définir quels préfixes sont autorisés
- Transformer les numéros (strip/prefix)
- Appliquer un Caller ID spécifique
- Sélectionner un trunk spécifique

### 4.3.2 CRUD des Outcalls

#### Endpoint

```http
GET/POST    /api/confd/1.1/outcalls
GET/PUT/DELETE /api/confd/1.1/outcalls/{outcall_id}
```

#### Création d'un Outcall

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "appels-sortants-france",
    "description": "Règles d'appels vers la France",
    "enabled": true,
    "internal": false,
    "schedules": []
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/outcalls"
```

#### Réponse

```json
{
  "id": 7,
  "name": "appels-sortants-france",
  "description": "Règles d'appels vers la France",
  "enabled": true,
  "internal": false,
  "tenant_uuid": "tenant-uuid-main"
}
```

### 4.3.3 Dial Patterns (Règles de Numérotation)

Les **dial patterns** définissent les transformations de numéros.

#### Création d'un Dial Pattern

```http
POST /api/confd/1.1/outcalls/{outcall_id}/dialpatterns
```

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "prefix": "0",
    "match_pattern": ".",
    "strip": "0",
    "caller_id": "0143321000"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/outcalls/7/dialpatterns"
```

#### Payload des Dial Patterns

| Champ | Type | Description |
|-------|------|-------------|
| `prefix` | `string` | Préfixe requis pour utiliser cette règle |
| `match_pattern` | `string` | Pattern de correspondance (`.` = tout) |
| `strip` | `int` | Nombre de chiffres à supprimer au début |
| `prepend` | `string` | Chiffres à ajouter au début |
| `caller_id` | `string` | Caller ID à utiliser |

#### Exemples de Dial Patterns

| Préfixe | Match | Strip | Prepend | Transformation |
|---------|-------|-------|---------|----------------|
| `0` | `.` | 0 | `+33` | `0612345678` → `+33612345678` |
| `00` | `.` | 2 | `+` | `003312345678` → `+3312345678` |
| `*` | `.` | 0 | (aucun) | `97` → `97` |

### 4.3.4 Association Trunk ↔ Outcall

Un outcall doit être lié à un trunk pour fonctionner.

```http
PUT /api/confd/1.1/outcalls/{outcall_id}/trunks/{trunk_id}
```

```bash
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/outcalls/7/trunks/5"
```

### 4.3.5 Association Extension/Contexte ↔ Outcall

Pour utiliser un outcall, liez-le à un contexte.

```http
PUT /api/confd/1.1/contexts/{context_id}/outcalls/{outcall_id}
```

```bash
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/contexts/outside/outcalls/7"
```

---

## 4.4 Incalls (Appels Entrants / DID)

### 4.4.1 Concept

Les **incalls** (appelés aussi "incoming calls" ou "DID/SDD") définissent le routage des appels entrants. Chaque DID (Direct Inward Dialing) est routé vers une destination.

### 4.4.2 CRUD des Incalls

#### Endpoint

```http
GET/POST    /api/confd/1.1/incalls
GET/PUT/DELETE /api/confd/1.1/incalls/{incall_id}
```

#### Création d'un Incall (DID)

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "exten": "0143321001",
    "context": "from-extern",
    "priority": 1,
    "destination": {
      "type": "user",
      "user_uuid": "user-uuid-1234"
    }
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/incalls"
```

#### Payload des Incalls

| Champ | Type | Obligatoire | Description |
|-------|------|-------------|-------------|
| `exten` | `string` | **Oui** | Numéro DID (ex: `0143321001`) |
| `context` | `string` | **Oui** | Contexte entrant (ex: `from-extern`) |
| `priority` | `int` | Non (défaut: 1) | Priorité de routage |
| `destination.type` | `string` | **Oui** | Type de destination |

#### Types de Destinations

| Type | Description | Paramètre supplémentaire |
|------|-------------|------------------------|
| `user` | Utilisateur | `user_uuid` |
| `voicemail` | Boîte vocale | `voicemail_id` |
| `queue` | File d'attente | `queue_id` |
| `group` | Groupe d'appel | `group_id` |
| `ivr` | Menu IVR | `ivr_id` |
| `conference` | Conférence | `conference_id` |
| `extension` | Extension | `extension_id` |
| `custom` | Destination personnalisée (dialplan) | `custom_src` |

### 4.4.3 Exemples de Destinations

#### Destination vers un Utilisateur

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "exten": "0143321001",
    "context": "from-extern",
    "destination": {
      "type": "user",
      "user_uuid": "user-uuid-1234"
    }
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/incalls"
```

#### Destination vers un IVR

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "exten": "0143321002",
    "context": "from-extern",
    "destination": {
      "type": "ivr",
      "ivr_id": 3
    }
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/incalls"
```

#### Destination vers une File d'Attente

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "exten": "0143321003",
    "context": "from-extern",
    "destination": {
      "type": "queue",
      "queue_id": 5
    }
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/incalls"
```

#### Destination avec Fallback (priorités multiples)

```bash
# Priorité 1 : utilisateur
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "exten": "0143321001",
    "context": "from-extern",
    "priority": 1,
    "destination": {
      "type": "user",
      "user_uuid": "user-uuid-1234"
    }
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/incalls"

# Priorité 2 : voicemail (si pas de réponse)
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "exten": "0143321001",
    "context": "from-extern",
    "priority": 2,
    "destination": {
      "type": "voicemail",
      "voicemail_id": 15
    }
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/incalls"
```

---

## 4.5 Caller ID pour Appels Entrants/Sortants

### 4.5.1 Caller ID Externe (Outgoing)

Pour définir le Caller ID présenté aux correspondants externes :

```bash
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "endpoint_section_options": [
      ["callerid", "Entreprise ABC <+33143321000>"]
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/endpoints/sip/{endpoint_uuid}"
```

### 4.5.2 Caller ID Interne

Pour un utilisateur spécifique :

```bash
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "caller_id": {
      "display_name": "Alice Dupont",
      "internal": false
    }
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/users/{user_uuid}"
```

---

## 4.6 Scénario Complet : Trunk Opérateur avec DID

Voici la séquence complète pour configurer un trunk SIP avec registration, DID entrant et routage :

```bash
# =============================================================================
# ÉTAPE 1 : Créer l'endpoint SIP du trunk
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "trunk-monoperateur",
    "endpoint_section_options": [
      ["disallow", "all"],
      ["allow", "ulaw,alaw,g722"],
      ["context", "from-extern"],
      ["trust_connected_line", "yes"],
      ["send_connected_line_info", "yes"]
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/endpoints/sip"

# Réponse : {"uuid": "trunk-ep-uuid", ...}


# =============================================================================
# ÉTAPE 2 : Créer la registration (si nécessaire)
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "endpoint_sip_uuid": "trunk-ep-uuid",
    "registration_section_options": [
      ["server_uri", "sip:sip.monoperateur.fr:5060"],
      ["client_uri", "sip:moncompte@monoperateur.fr"],
      ["expiry", "3600"]
    ],
    "auth_section_options": [
      ["username", "moncompte"],
      ["password", "MotDePasseOperateur"]
    ]
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/registrations"


# =============================================================================
# ÉTAPE 3 : Créer le trunk
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "Trunk MonOpérateur",
    "endpoint_sip_uuid": "trunk-ep-uuid",
    "context": "from-extern"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/trunks"

# Réponse : {"id": 8, ...}


# =============================================================================
# ÉTAPE 4 : Créer l'outcall pour appels sortants
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "sortant-monoperateur",
    "enabled": true
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/outcalls"

# Réponse : {"id": 10, ...}


# =============================================================================
# ÉTAPE 5 : Ajouter dial pattern (France: 00 + numéro)
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "prefix": "00",
    "match_pattern": ".",
    "strip": "2",
    "prepend": "+"
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/outcalls/10/dialpatterns"


# =============================================================================
# ÉTAPE 6 : Lier outcall au trunk
# =============================================================================
curl -k -X PUT \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/outcalls/10/trunks/8"


# =============================================================================
# ÉTAPE 7 : Créer le DID entrant (0143321000)
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "exten": "0143321000",
    "context": "from-extern",
    "priority": 1,
    "destination": {
      "type": "user",
      "user_uuid": "user-uuid-cible"
    }
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/incalls"


# =============================================================================
# ÉTAPE 8 : Créer un second DID vers standard (IVR)
# =============================================================================
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "exten": "0143321001",
    "context": "from-extern",
    "priority": 1,
    "destination": {
      "type": "ivr",
      "ivr_id": 5
    }
  }' \
  "https://wazo.example.com:9486/api/confd/1.1/incalls"
```

---

## Résumé du Chapitre 4

| Ressource | Endpoints Clés | Point Critique |
|-----------|-----------------|----------------|
| **Trunk** | `POST /trunks` | Lie endpoint SIP + registration |
| **Endpoint SIP (trunk)** | `POST /endpoints/sip` | Context `from-extern` obligatoire |
| **Registration** | `/registrations` | Si Opérateur nécessite REGISTER |
| **Outcall** | `POST /outcalls/{id}/dialpatterns` | Strip/prefix pour transformation |
| **Dial Pattern** | `/outcalls/{id}/trunks/{trunk_id}` | Association trunk obligatoire |
| **Incall (DID)** | `POST /incalls` | Destination: user/queue/ivr/conference |
| **Caller ID** | Endpoint + user | `trust_connected_line` pour préserver CID |

---

## Résumé des Chapitres 3 & 4

### Objets de Configuration Core (Chapitre 3)

| Objet | Relations | Clé |
|-------|-----------|-----|
| Context | Contient extensions | `context_type` |
| Endpoint SIP | → Ligne (1:1) | Sections PJSIP |
| Ligne | → Endpoint (1:1), Extension (1:1), User (1:N) | `protocol` |
| Extension | → Ligne (1:1) | `exten` + `context` |
| User | → Ligne (N:M), Voicemail (1:1) | `uuid` |
| Voicemail | → User (1:1) | `number` |
| Call Permission | → User (N:M), Rules | `prefix` + `action` |

### Objets de Communication Externe (Chapitre 4)

| Objet | Relations | Clé |
|-------|-----------|-----|
| Trunk | → Endpoint SIP, Registration | `endpoint_sip_uuid` |
| Registration | → Endpoint SIP | `server_uri`, `auth` |
| Outcall | → Dial Patterns, Trunks | `strip`/`prepend` |
| Dial Pattern | → Outcall | `prefix` |
| Incall | → Destination (user/queue/ivr) | `exten` (DID) |

---

*Fin des Chapitres 3 et 4 — Suite : Chapitre 5 (Services Avancés)*