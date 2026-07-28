# Bible API Wazo — Guide du Développeur Expert

> **Document technique exclusif** — Wazo Platform (Unified Communications as a Service)
>
> Version: 2.0 — Mars 2026
>
> *Architecture microservices, REST API, WebSockets, Multi-tenant*

---

# CHAPITRE 1 : Fondations et Architecture Wazo

## 1.1 Écosystème Microservices Wazo

Wazo est une plateforme de communications unifiées reposant sur une architecture **microservices distribuée**. Chaque service est un démon indépendant, déployé sur le serveur Wazo, communiquant via des APIs RESTful et un bus de messages (RabbitMQ). Cette architecture permet une scalabilité horizontale, une maintenance modulaire et une isolation des fonctionnalités.

### 1.1.1 Cartographie des Services

> **IMPORTANT - Routing nginx** : Toutes les API Wazo Admin passent par nginx sur le **port 443**. Les ports directs ci-dessous sont uniquement pour debugging ou accès direct. Utilisez toujours les routes nginx.

| Service | Port Direct | Nginx Route | Version API | Rôle Métier | Base URL Complete |
|---------|-------------|-------------|-------------|-------------|-------------------|
| **wazo-auth** | 9497 | /api/auth/0.1/* | 0.1 | Authentification, gestion des tokens, ACL | `https://{host}/api/auth/0.1` |
| **wazo-confd** | 9486 | /api/confd/1.1/* | 1.1 | Configuration centrale | `https://{host}/api/confd/1.1` |
| **wazo-provd** | 8667 | /api/provd/0.1/* | 0.1 | Provisioning terminaux | `https://{host}/api/provd/0.1` |
| **wazo-calld** | 9500 | /api/calld/1.0/* | 1.0 | Contrôle appels temps réel | `https://{host}/api/calld/1.0` |
| **wazo-chatd** | 9500 | /api/chatd/1.0/* | 1.0 | Présences XMPP, messaging | `https://{host}/api/chatd/1.0` |
| **wazo-webhookd** | 9300 | /api/webhookd/1.0/* | 1.0 | Webhooks HTTP | `https://{host}/api/webhookd/1.0` |
| **wazo-call-logd** | 9298 | /api/call-logd/1.0/* | 1.0 | Historique CDR | `https://{host}/api/call-logd/1.0` |
| **wazo-dird** | 9489 | /api/dird/0.1/* | 0.1 | Annuaires | `https://{host}/api/dird/0.1` |
| **wazo-plugind** | 9400 | /api/plugind/0.1/* | 0.1 | Plugins système | `https://{host}/api/plugind/0.1` |
| **wazo-agentd** | 9500 | /api/agentd/1.0/* | 1.0 | Gestion agents ACD | `https://{host}/api/agentd/1.0` |
| **wazo-websocketd** | 9502 | /api/websocketd/* | 2.0 | WebSocket events | `wss://{host}/api/websocketd/` |
| **wazo-amid** | 9498 | /api/amid/1.0/* | 1.0 | Proxy AMI Asterisk | `https://{host}/api/amid/1.0` |
| **wazo-phoned** | 9496 | /api/phoned/0.1/* | 0.1 | Push mobile, lookup | `https://{host}/api/phoned/0.1` |

> **Note critique** : wazo-calld, wazo-chatd et wazo-agentd partagent le port 9500 mais se distinguent par leur chemin de base. wazo-call-logd utilise le port 9298 (ne partage pas le 9500).

### 1.1.2 Communication Inter-Services

Les services Wazo communiquent selon deux modèles :

1. **Communication synchrone (REST)** : Requêtes HTTP directes entre le client et les services. Chaque requête doit inclure un token d'authentification valide.

2. **Communication asynchrone (Bus de messages)** : Wazo utilise **RabbitMQ** avec l'échange `wazo` (topic). Les événements sont publiés sous forme de messages JSON structurés. Les clients peuvent s'abonner à ces événements via WebSocket ou HTTP long-polling.

```json
// Exemple de message bus (événement user.created)
{
  "name": "user_created",
  "timestamp": "2026-03-07T15:30:00.000000Z",
  "origin_uuid": "server-uuid-001",
  "data": {
    "uuid": "user-uuid-1234",
    "firstname": "Alice",
    "lastname": "Dupont",
    "tenant_uuid": "tenant-uuid-main"
  }
}
```

### 1.1.3 Cycle de Vie d'une Requête API

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CYCLE DE VIE REQUÊTE API WAZO                       │
└─────────────────────────────────────────────────────────────────────────────┘

  [Client]
     │
     │  1. Requête HTTP
     │     Headers: X-Auth-Token, Accept, Content-Type
     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        wazo-auth (Port 9497)                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Validation du token                                                   │   │
│  │   ├─ Token expiré? → 401 Unauthorized                                │   │
│  │   ├─ ACL insuffisante? → 403 Forbidden                               │   │
│  │   └─ Token valide → Passage à wazo-confd                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
     │
     │  (Le token porte les infos: user_uuid, tenant_uuid, ACLs)
     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     wazo-confd (Port 9486)                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Vérification ACL + Filtrage tenant                                    │   │
│  │   ├─ Tentative d'accès resource autre tenant → 404                   │   │
│  │   └─ Autorisé → Exécution opération CRUD                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│                         ┌──────────────────────┐                            │
│                         │  Réponse HTTP        │                            │
│                         │  200/201/204/400/   │                            │
│                         │  401/403/404/409/500 │                            │
│                         └──────────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────────┘
     │
     ▼
  [Client]
```

---

## 1.2 Architecture Multi-Tenant

Wazo est nativement **multi-tenant**. Chaque tenant (aussi appelé "organisation" ou "client") est isolé des autres au niveau des données et des permissions. Cette isolation est gérée à trois niveaux :

1. **Niveau base de données** : Chaque tenant possède ses propres enregistrements dans les tables PostgreSQL.
2. **Niveau API** : Le header `Wazo-Tenant` permet de filtrer les requêtes.
3. **Niveau ACL** : Les politiques définissent les droits d'accès par tenant.

### 1.2.1 Structure Hiérarchique des Tenants

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      HIÉRARCHIE TENANTS WAZO                               │
└─────────────────────────────────────────────────────────────────────────────┘

                          [Tenant Root]
                                │
                    ┌───────────┴───────────┐
                    │                         │
            [Tenant Principal]          [Sous-Tenant A]
            (Master Tenant)                 │
                    │                         │
          ┌─────────┴─────────┐              │
          │                   │              │
    [Sous-Tenant B]    [Sous-Tenant C]   [Sous-Tenant D]
```

Chaque tenant possède un **UUID unique** au format UUID v4 :
```
tenant-uuid-main:    6118e18b-17e2-49ef-a59c-0759063b9548
tenant-uuid-enfant:  a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

### 1.2.2 Header `Wazo-Tenant` — Guide Complet

Le header `Wazo-Tenant` est **obligatoire** pour toute opération multi-tenant, sauf pour le tenant principal (root).

#### Syntaxe

```http
Wazo-Tenant: {tenant_uuid}
```

#### Exemples de Requêtes

```bash
# Requête SANS header Wazo-Tenant → Accès au tenant root (premier tenant configuré)
curl -k -X GET \
  -H "Accept: application/json" \
  -H "X-Auth-Token: {token}" \
  https://wazo.example.com:9486/api/confd/1.1/users

# Requête AVEC header Wazo-Tenant → Accès à un tenant spécifique
curl -k -X GET \
  -H "Accept: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: 6118e18b-17e2-49ef-a59c-0759063b9548" \
  https://wazo.example.com:9486/api/confd/1.1/users
```

#### Création d'un Sous-Tenant

Pour créer un sous-tenant, utilisez l'API wazo-auth :

```bash
curl -k -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_parent_uuid}" \
  -d '{
    "name": "Sous-Tenant-A",
    "slug": "sous-tenant-a"
  }' \
  https://wazo.example.com:9497/api/auth/0.1/tenants

# Réponse (201 Created)
{
  "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "Sous-Tenant-A",
  "slug": "sous-tenant-a",
  "parent_uuid": "6118e18b-17e2-49ef-a59c-0759063b9548"
}
```

#### Récupérer la Liste des Tenants Accessibles

```bash
curl -k -X GET \
  -H "Accept: application/json" \
  -H "X-Auth-Token: {token}" \
  https://wazo.example.com:9497/api/auth/0.1/tenants

# Réponse (200 OK)
{
  "total": 3,
  "items": [
    {
      "uuid": "6118e18b-17e2-49ef-a59c-0759063b9548",
      "name": "Tenant Principal",
      "slug": "tenant-principal",
      "parent_uuid": null
    },
    {
      "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "name": "Sous-Tenant A",
      "slug": "sous-tenant-a",
      "parent_uuid": "6118e18b-17e2-49ef-a59c-0759063b9548"
    }
  ]
}
```

> **⚠️ Attention** : Si vous êtes administrateur d'un sous-tenant, vous **ne pouvez pas** créer de tenant parent via l'API. Seuls les administrateurs du tenant root peuvent créer des hiérarchies de tenants.

### 1.2.3 Isolation des Ressources par Tenant

Chaque ressource Wazo est liée à un `tenant_uuid`. Les règles d'isolation sont les suivantes :

| Ressource | Champ tenant_uuid | Comportement |
|-----------|-------------------|--------------|
| **Users** | `tenant_uuid` (propriété) | Réservé au tenant créateur |
| **Lines** | `tenant_uuid` (propriété) | Réservé au tenant créateur |
| **Extensions** | `tenant_uuid` (propriété) | Réservé au tenant créateur |
| **Devices** | `tenant_uuid` (propriété) | Réservé au tenant créateur |
| **Trunks** | `tenant_uuid` (propriété) | Réservé au tenant créateur |
| **Queues** | `tenant_uuid` (propriété) | Réservé au tenant créateur |
| **IVR** | `tenant_uuid` (propriété) | Réservé au tenant créateur |
| **Schedules** | `tenant_uuid` (propriété) | Réservé au tenant créateur |

**Requête avec tenant incorrect** :

```bash
# Tentative d'accès à un user d'un autre tenant
curl -k -X GET \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: tenant-a-uuid" \
  https://wazo.example.com:9486/api/confd/1.1/users/user-uuid-autre-tenant

# Réponse (404 Not Found) — Le user existe mais n'est pas dans ce tenant
{
  "error": "Not Found",
  "details": "User not found"
}
```

---

## 1.3 Règles Communes des APIs Wazo

### 1.3.1 Paramètres Génériques

Toutes les APIs Wazo (confd, auth, provd, calld, etc.) partagent un ensemble de paramètres communs pour la manipulation des listes de ressources.

| Paramètre | Type | Description | Exemple |
|-----------|------|-------------|---------|
| `limit` | `int` | Nombre maximum d'éléments retournés (pagination) | `?limit=25` |
| `offset` | `int` | Index de départ pour la pagination | `?offset=50` |
| `search` | `string` | Recherche textuelle sur tous les champs indexables | `?search=alice` |
| `order` | `string` | Colonne de tri | `?order=firstname` |
| `direction` | `string` | Ordre de tri : `asc` ou `desc` | `?direction=desc` |

#### Exemple Combiné

```bash
# Liste des utilisateurs 26-50, triés par nom décroissant, contenant "dupont"
curl -k -X GET \
  -H "X-Auth-Token: {token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9486/api/confd/1.1/users?limit=25&offset=25&search=dupont&order=firstname&direction=desc"
```

#### Format de Réponse Paginée

```json
{
  "total": 150,
  "items": [
    {
      "uuid": "user-uuid-001",
      "firstname": "Alice",
      "lastname": "Dupont",
      "email": "alice@example.com"
    }
  ]
}
```

Le champ `total` indique le nombre **total** de ressources correspondantes, toutes pages confondues. Utilisez-le pour calculer le nombre de pages : `pages = ceil(total / limit)`.

### 1.3.2 Headers HTTP Communs

| Header | Valeur | Obligatoire | Description |
|--------|--------|-------------|-------------|
| `Accept` | `application/json` | **Oui** | Indique que le client attend du JSON |
| `Content-Type` | `application/json` | Oui (POST/PUT) | Type du corps de la requête |
| `X-Auth-Token` | `{token_string}` | **Oui** | Token d'authentification |
| `Wazo-Tenant` | `{tenant_uuid}` | Conditionnel | UUID du tenant (obligatoire si multi-tenant) |

### 1.3.3 Codes HTTP et Gestion des Erreurs

Wazo utilise les codes HTTP standard. Voici les codes les plus fréquents :

| Code | Signification | Cause Commune | Détail |
|------|---------------|---------------|--------|
| **200** | OK | Requête GET/PUT réussie | Corps de réponse présent |
| **201** | Created | Ressource créée avec POST | Corps avec la nouvelle ressource |
| **204** | No Content | DELETE ou PUT réussie sans retour | Pas de corps de réponse |
| **400** | Bad Request | JSON invalide, paramètre manquant | Voir `details` dans la réponse |
| **401** | Unauthorized | Token manquant ou expiré | Token non valide ou expiré |
| **403** | Forbidden | Token valide mais ACL insuffisante | Permissions insuffisantes |
| **404** | Not Found | Ressource inexistante ou endpoint invalide | UUID incorrect ou permission denied |
| **409** | Conflict | Ressource déjà existante | Doublon sur champ unique |
| **415** | Unsupported Media Type | Header `Content-Type` manquant | POST/PUT sans JSON |
| **422** | Unprocessable Entity | Données syntaxiquement correctes mais sémantiquement invalides | Contrainte métier non respectée |
| **500** | Internal Server Error | Erreur interne Wazo | Erreur côté serveur |

#### Format Standard des Réponses d'Erreur

```json
{
  "error": "Bad Request",
  "details": "Missing required field: 'firstname'",
  "timestamp": "2026-03-07T15:30:00.123456Z",
  "resource": "users"
}
```

#### Classe Exception Python (WazoAPIError)

Si vous utilisez le SDK Python `wazo-confd-client`, les erreurs sont levées sous forme d'exception :

```python
from wazo_confd_client import Client
from wazo_confd_client.error import WazoAPIError

client = Client('wazo.example.com', username='admin', password='pass')

try:
    user = client.users.create({'firstname': 'Alice'})
except WazoAPIError as e:
    print(f"Status: {e.status_code}")   # 400
    print(f"Error: {e.message}")          # Bad Request
    print(f"Details: {e.details}")        # Missing required field: 'firstname'
```

### 1.3.4 Conventions de Nommage des Endpoints

Wazo suit les conventions RESTful :

- **Collection** : `/users` — GET (liste), POST (création)
- **Ressource individuelle** : `/users/{uuid}` — GET, PUT, DELETE
- **Sous-ressource (association)** : `/users/{uuid}/lines/{line_id}` — PUT, DELETE
- **Action spécifique** : `/users/{uuid}/lines/{line_id}/update` — POST

#### Ressources Imbriquées Courantes

```bash
# Ligne liée à un utilisateur
/users/{user_uuid}/lines/{line_id}

# Extension liée à une ligne
/lines/{line_id}/extensions/{extension_id}

# Endpoint SIP lié à une ligne
/lines/{line_id}/endpoints/sip/{endpoint_uuid}

# Extension liée à un groupe
/groups/{group_uuid}/extensions/{extension_id}
```

---

## 1.4 Modèle de Données Relationnel

Wazo est un système **fortement relationnel**. Comprendre les relations entre objets est crucial pour l'intégration.

### 1.4.1 Schéma de Relations Fondamental

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   SCHÉMA RELATIONNEL WAZO CORE                             │
└─────────────────────────────────────────────────────────────────────────────┘

                           ┌──────────────┐
                           │   CONTEXT    │
                           │  (default,   │
                           │ from-extern) │
                           └──────┬───────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
            ▼                     ▼                     ▼
    ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
    │  EXTENSION    │     │  EXTENSION    │     │  EXTENSION    │
    │  (numéro)     │     │  (numéro)     │     │  (numéro)     │
    └───────┬───────┘     └───────┬───────┘     └───────┬───────┘
            │                     │                     │
            └──────────┬──────────┘                     │
                       │                                │
                       ▼                                │
              ┌────────────────┐                         │
              │     LINE      │◄────────────────────────┘
              │  (protocol)   │
              └───────┬───────┘
                      │
      ┌───────────────┼───────────────┐
      │               │               │
      ▼               ▼               ▼
┌───────────┐   ┌───────────┐   ┌───────────┐
│  ENDPOINT │   │   USER    │   │  DEVICE   │
│   (SIP)   │   │ (agent)   │   │ (phone)   │
└───────────┘   └───────────┘   └───────────┘
```

### 1.4.2 Règles d'Association

| Association | Régle | Exemple |
|-------------|-------|---------|
| **Line → Extension** | 1:1 ou 1:0 (optionnelle) | Une ligne peut avoir 0 ou 1 extension |
| **Line → Endpoint SIP** | 1:1 | Une ligne doit être liée à exactement 1 endpoint |
| **Line → User** | 1:N | Un utilisateur peut avoir plusieurs lignes |
| **Extension → Destination** | N:1 | Plusieurs extensions peuvent pointer vers un même IVR, Queue, Group |
| **User → Voicemail** | 1:0 ou 1:1 | Un utilisateur peut avoir 0 ou 1 boite vocale |
| **Device → Line** | N:M | Un téléphone peut être associé à plusieurs lignes |

> **⚠️ Attention critique** : Une ligne **doit** être liée à un endpoint SIP pour fonctionner. Sans endpoint, la ligne n'a pas de configuration technique et le téléphone ne pourra pas s'enregistrer. De même, sans extension, la ligne n'est pas joignable depuis l'extérieur.

---

# CHAPITRE 2 : Authentification et Sécurité (wazo-auth)

## 2.1 Introduction à wazo-auth

**wazo-auth** est le service central d'authentification de la plateforme Wazo. Il gère :

- La création et la validation des **tokens d'accès**
- Les **utilisateurs d'authentification** (distincts des utilisateurs telephony)
- Les **politiques d'accès** (policies) et les **ACL** (Access Control Lists)
- Les **groupes d'utilisateurs** pour le regroupement de permissions
- L'**usurpation d'identité** (impersonation) pour les requêtes admin
- L'authentification **LDAP** et **SAML**

> **Distinction fondamentale** : Les utilisateurs gérés par wazo-auth (`/api/auth/0.1/users`) sont **différents** des utilisateurs telephony gérés par wazo-confd (`/api/confd/1.1/users`). Les premiers concernent l'accès à l'API, les seconds représentent les agents dans le système de communications.

## 2.2 Création de Token — Guide Complet

### 2.2.1 Endpoint

```http
POST /api/auth/0.1/token
```

### 2.2.2 Headers

| Header | Valeur | Obligatoire |
|--------|--------|-------------|
| `Content-Type` | `application/json` | **Oui** |
| `Accept` | `application/json` | Non (défaut) |

### 2.2.3 Méthodes d'Authentification

Wazo supporte plusieurs "backends" d'authentification :

#### Backend `wazo_user` — Utilisateur Local Wazo

Ce backend autentifie les utilisateurs créés directement dans wazo-auth.

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  "https://wazo.example.com:9497/api/auth/0.1/token" \
  -u "admin:mon_mot_de_passe" \
  -d '{
    "expiration": 3600
  }'
```

**Équivalent avec JSON inline** :

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  "https://wazo.example.com:9497/api/auth/0.1/token" \
  -d '{
    "username": "admin",
    "password": "mon_mot_de_passe",
    "backend": "wazo_user",
    "expiration": 3600
  }'
```

#### Backend `ldap_user` — Utilisateur LDAP Externe

Ce backend délègue l'authentification à un serveur LDAP configuré sur le serveur Wazo.

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  "https://wazo.example.com:9497/api/auth/0.1/token" \
  -d '{
    "username": "alice@mondomaine.local",
    "password": "mot_de_passe_ldap",
    "backend": "ldap_user",
    "expiration": 7200
  }'
```

#### Backend `xivo_admin` — Administrateur Wazo (Legacy)

Pour les administrateurs définis via l'interface d'administration Wazo.

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  "https://wazo.example.com:9497/api/auth/0.1/token" \
  -d '{
    "username": "admin_wazo",
    "password": "password_admin",
    "backend": "xivo_admin",
    "expiration": 1800
  }'
```

### 2.2.4 Paramètres du Payload

| Champ | Type | Obligatoire | Description |
|-------|------|-------------|-------------|
| `username` | `string` | **Oui** (si pas auth HTTP) | Nom d'utilisateur |
| `password` | `string` | **Oui** (si pas auth HTTP) | Mot de passe |
| `backend` | `string` | Non (défaut: wazo_user) | Type de backend (`wazo_user`, `ldap_user`, `xivo_admin`) |
| `expiration` | `int` | Non (durée par défaut: 3600s) | Durée de vie du token en secondes |

**Valeurs d'expiration conseillées** :

- `3600` (1 heure) — Usage standard
- `7200` (2 heures) — Applications longue durée
- `86400` (24 heures) — Scripts batch (avec précaution)
- `0` — Token sans expiration (déprécié, utiliser avec précaution)

### 2.2.5 Réponse Succès (201 Created)

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX3V1aWQiOiI2MTE4ZTE4Yi0xN2UyLTQ5ZWYtYTU5Yy0wNzU5MDYzYjk1NDgiLCJ0ZW5hbnRfdXVpZCI6IjYxMThlMThiLTE3ZTItNDllZi1hNTljLTA3NTkwNjNiOTU0OCIsImFjbCI6WyJjb25mZC51c2Vycy4jIiwiY29uZmQuKiIsImF1dGguKiJdLCJpYXQiOjE3MzU2NDA2MDB9.signature",
  "user_uuid": "6118e18b-17e2-49ef-a59c-0759063b9548",
  "tenant_uuid": "6118e18b-17e2-49ef-a59c-0759063b9548",
  "expiration": 3600,
  "issued_at": "2026-03-07T15:30:00.000000Z",
  "expires_at": "2026-03-07T16:30:00.000000Z",
  "acl": [
    "confd.users.#",
    "confd.lines.#",
    "confd.extensions.#",
    "confd.queues.#",
    "auth.#"
  ]
}
```

#### Détail des Champs de Réponse

| Champ | Type | Description |
|-------|------|-------------|
| `token` | `string` | Le token JWT à utiliser dans le header `X-Auth-Token` |
| `user_uuid` | `uuid` | UUID de l'utilisateur authentifié |
| `tenant_uuid` | `uuid` | UUID du tenant principal de l'utilisateur |
| `expiration` | `int` | Durée de vie en secondes |
| `issued_at` | `datetime` | Timestamp de création (ISO 8601) |
| `expires_at` | `datetime` | Timestamp d'expiration (ISO 8601) |
| `acl` | `array[string]` | Liste des permissions ACL |

### 2.2.6 Réponses d'Erreur

**401 Unauthorized — Identifiants invalides** :

```json
{
  "error": "Unauthorized",
  "details": "Invalid credentials"
}
```

**401 Unauthorized — Backend invalide** :

```json
{
  "error": "Unauthorized",
  "details": "Backend 'ldap_user' is not enabled"
}
```

**400 Bad Request — Données manquantes** :

```json
{
  "error": "Bad Request",
  "details": "Missing 'username' or 'password'"
}
```

---

## 2.3 Cycle de Vie du Token

### 2.3.1 Validation d'un Token

Pour vérifier la validité d'un token sans créer de nouvelle requête :

```http
GET /api/auth/0.1/token/{token}
```

```bash
curl -k -X GET \
  -H "Accept: application/json" \
  "https://wazo.example.com:9497/api/auth/0.1/token/{token_a_valider}"
```

**Réponse (200 OK)** :

```json
{
  "user_uuid": "6118e18b-17e2-49ef-a59c-0759063b9548",
  "tenant_uuid": "6118e18b-17e2-49ef-a59c-0759063b9548",
  "issued_at": "2026-03-07T15:30:00.000000Z",
  "expires_at": "2026-03-07T16:30:00.000000Z",
  "acl": ["confd.users.#", "confd.lines.#"]
}
```

**Réponse (404 Not Found — Token expiré ou invalide)** :

```json
{
  "error": "Not Found",
  "details": "Token not found"
}
```

### 2.3.2 Suppression d'un Token (Logout)

Pour invalider un token avant son expiration :

```http
DELETE /api/auth/0.1/token/{token}
```

```bash
curl -k -X DELETE \
  -H "X-Auth-Token: {token}" \
  "https://wazo.example.com:9497/api/auth/0.1/token/{token_a_invalider}"
```

**Réponse (204 No Content)** : Le token est immédiatement invalidé.

### 2.3.3 Refresh Token (Auto-extension)

Wazo ne dispose pas d'un mécanisme de "refresh token" explicite. Pour maintenir une session active, renouvelez le token avant son expiration :

```python
import time
import requests

def get_valid_token(auth_url, username, password):
    """Récupère un token et le renouvelle automatiquement avant expiration."""
    
    response = requests.post(
        f"{auth_url}/0.1/token",
        json={
            "username": username,
            "password": password,
            "backend": "wazo_user",
            "expiration": 3600
        },
        verify=False
    )
    response.raise_for_status()
    
    data = response.json()
    token = data['token']
    expires_at = data['expires_at']
    
    return token, expires_at

# Utilisation
token, expires_at = get_valid_token(
    "https://wazo.example.com:9497/api/auth",
    "admin",
    "password"
)

# Avant expiration, renouvellement
if time.time() > (parse_datetime(expires_at) - 300):  # 5 min avant
    token, expires_at = get_valid_token(...)
```

---

## 2.4 ACL (Access Control Lists) — Guide Expert

### 2.4.1 Concept des ACLs

Les **ACL** définissent quelles ressources un utilisateur peut accéder via l'API. Une ACL est une chaîne au format `{service}.{ressource}.{action}` avec des wildcards (`#`).

#### Structure d'une ACL

```
{service}.{ressource}.{cible}
```

| Composant | Description | Exemple |
|-----------|-------------|---------|
| `service` | Nom du service API | `confd`, `auth`, `provd`, `calld` |
| `ressource` | Nom de la ressource | `users`, `lines`, `extensions`, `queues` |
| `cible` | Action ou sous-ressource | `read`, `write`, `#` (toutes) |

#### Exemples d'ACLs

| ACL | Permission |
|-----|------------|
| `confd.users.#` | Accès complet à tous les utilisateurs |
| `confd.users.read` | Lecture seule des utilisateurs |
| `confd.lines.#` | Accès complet aux lignes |
| `confd.extensions.read` | Lecture seule des extensions |
| `auth.users.#` | Gestion des utilisateurs d'auth |
| `provd.devices.#` | Accès complet aux devices |
| `#` | **Super-admin** : Accès à toutes les APIs |

> **⚠️ Attention** : L'ACL `#` donne un accès illimité à toutes les ressources. Utilisez-la avec précaution et uniquement pour les administrateurs système.

### 2.4.2 Création d'une Politique (Policy)

Les ACLs sont regroupées dans des **politiques** (policies). Une politique peut être appliquée à un ou plusieurs utilisateurs.

#### Endpoint

```http
POST /api/auth/0.1/policies
```

#### Headers

| Header | Valeur |
|--------|--------|
| `X-Auth-Token` | Token admin |
| `Wazo-Tenant` | UUID du tenant |
| `Content-Type` | `application/json` |

#### Payload Complet

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {admin_token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "name": "Technicien-Telecom",
    "description": "Politique pour technicians helpdesk",
    "acl_templates": [
      "confd.users.#",
      "confd.users.read",
      "confd.lines.#",
      "confd.extensions.#",
      "confd.queues.#",
      "confd.queues.members.#",
      "confd.sounds.#",
      "confd.voicemails.#",
      "confd.IVR.#",
      "confd.schedules.#"
    ]
  }' \
  "https://wazo.example.com:9497/api/auth/0.1/policies"
```

**Réponse (201 Created)** :

```json
{
  "uuid": "policy-uuid-1234",
  "name": "Technicien-Telecom",
  "description": "Politique pour technicians helpdesk",
  "tenant_uuid": "tenant-uuid-main",
  "acl_templates": [
    "confd.users.#",
    "confd.lines.#"
  ],
  "created_at": "2026-03-07T15:30:00.000000Z"
}
```

### 2.4.3 Liste des Politiques Existantes

```bash
curl -k -X GET \
  -H "X-Auth-Token: {admin_token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9497/api/auth/0.1/policies"
```

### 2.4.4 Association Politique ↔ Utilisateur

Pour appliquer une politique à un utilisateur :

```http
PUT /api/auth/0.1/users/{user_uuid}/policies/{policy_uuid}
```

```bash
curl -k -X PUT \
  -H "X-Auth-Token: {admin_token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9497/api/auth/0.1/users/{user_uuid}/policies/{policy_uuid}"
```

**Réponse (204 No Content)** : Politique appliquée.

### 2.4.5 Désassociation

```http
DELETE /api/auth/0.1/users/{user_uuid}/policies/{policy_uuid}
```

```bash
curl -k -X DELETE \
  -H "X-Auth-Token: {admin_token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9497/api/auth/0.1/users/{user_uuid}/policies/{policy_uuid}"
```

### 2.4.6 Récupérer les ACLs d'un Token

Pour debugging ou audit, vérifiez les ACLs portées par un token :

```bash
# Via le token lui-même (décodage JWT - partie payload)
# OU via l'API de validation

curl -k -X GET \
  -H "X-Auth-Token: {token}" \
  "https://wazo.example.com:9497/api/auth/0.1/token/{token}"
```

---

## 2.5 Usurpation d'Identité (Impersonation)

L'**usurpation d'identité** permet à un administrateur d'effectuer des requêtes "en tant que" un autre utilisateur, tout en conservant la trace de l'action dans les logs. C'est particulièrement utile pour le support ou le débogage.

### 2.5.1 Mécanisme

L'administrateur ajoute le header `Wazo-Impersonation` avec l'UUID de l'utilisateur cible :

```http
X-Auth-Token: {admin_token}
Wazo-Impersonation: {user_uuid_cible}
```

### 2.5.2 Exemple Pratique

```bash
# Admin authentifié (token: admin-token)
# Souhaite voir ce que voit l'utilisateur alice-uuid

curl -k -X GET \
  -H "X-Auth-Token: admin-token" \
  -H "Wazo-Impersonation: alice-uuid-1234" \
  -H "Accept: application/json" \
  "https://wazo.example.com:9486/api/confd/1.1/users"
```

**Comportement** :

1. Le système vérifie que `admin-token` a le droit d'usurper (`auth.users.impostor.#`)
2. Les ressources retournées sont filtrées selon les permissions de `alice-uuid-1234`
3. L'action est logguée comme effectuée par l'admin "en tant qu'Alice"

> **⚠️ Attention** : L'usurpation d'identité ne fonctionne que pour les utilisateurs du **même tenant** ou des **sous-tenants**. Un admin root peut usurper n'importe quel utilisateur.

---

## 2.6 Gestion des Utilisateurs d'Auth (wazo-auth)

### 2.6.1 Création d'un Utilisateur d'Auth

Ces utilisateurs sont distincts des utilisateurs telephony (confd).

```http
POST /api/auth/0.1/users
```

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {admin_token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "username": "alice.tech",
    "password": "SecureP4ssw0rd!",
    "purpose": "external_api",
    "email": "alice@example.com",
    "firstname": "Alice",
    "lastname": "Technician"
  }' \
  "https://wazo.example.com:9497/api/auth/0.1/users"
```

**Payload détaillé** :

| Champ | Type | Obligatoire | Description |
|-------|------|-------------|-------------|
| `username` | `string` | **Oui** | Identifiant unique (par tenant) |
| `password` | `string` | **Oui** | Mot de passe (min 8 caractères) |
| `purpose` | `string` | Non (défaut: `user`) | `user` ou `external_api` |
| `email` | `string` | Non | Adresse email |
| `firstname` | `string` | Non | Prénom |
| `lastname` | `string` | Non | Nom |

**Réponse (201 Created)** :

```json
{
  "uuid": "user-auth-uuid-1234",
  "username": "alice.tech",
  "purpose": "external_api",
  "email": "alice@example.com",
  "firstname": "Alice",
  "lastname": "Technician",
  "tenant_uuid": "tenant-uuid-main",
  "enabled": true,
  "created_at": "2026-03-07T15:30:00.000000Z"
}
```

### 2.6.2 Liste des Utilisateurs d'Auth

```bash
curl -k -X GET \
  -H "X-Auth-Token: {admin_token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9497/api/auth/0.1/users"
```

### 2.6.3 Modification de Mot de Passe

```http
PUT /api/auth/0.1/users/{user_uuid}/password
```

```bash
curl -k -X PUT \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: {admin_token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  -d '{
    "password": "NouveauMdp2026!"
  }' \
  "https://wazo.example.com:9497/api/auth/0.1/users/{user_uuid}/password"
```

### 2.6.4 Suppression d'un Utilisateur

```http
DELETE /api/auth/0.1/users/{user_uuid}
```

```bash
curl -k -X DELETE \
  -H "X-Auth-Token: {admin_token}" \
  -H "Wazo-Tenant: {tenant_uuid}" \
  "https://wazo.example.com:9497/api/auth/0.1/users/{user_uuid}"
```

---

## 2.7 Configuration LDAP

### 2.7.1 Prérequis LDAP

Pour utiliser l'authentification LDAP, le service wazo-auth doit être configuré avec les paramètres LDAP dans `/etc/wazo-auth/conf.d/`.

```yaml
# /etc/wazo-auth/conf.d/ldap.yml
enabled_backend_plugins:
  ldap_user: true

ldap:
  uri: ldap://ldap.example.com:389
  bind_dn: cn=wazo,dc=example,dc=org
  bind_password: ldap_bind_password
  user_base_dn: ou=users,dc=example,dc=org
  user_login_attribute: uid
  user_email_attribute: mail
  user_filter: "(objectClass=inetOrgPerson)"
```

> **Note** : La configuration LDAP nécessite un redémarrage du service wazo-auth (`systemctl restart wazo-auth`).

### 2.7.2 Vérification du Backend LDAP

```bash
curl -k -X GET \
  -H "X-Auth-Token: {admin_token}" \
  "https://wazo.example.com:9497/api/auth/0.1/backends"
```

**Réponse** :

```json
{
  "total": 3,
  "items": [
    {"name": "wazo_user"},
    {"name": "ldap_user"},
    {"name": "xivo_admin"}
  ]
}
```

---

## 2.8 Bonnes Pratiques de Sécurité

### 2.8.1 Rotation des Tokens

- Utilisez des tokens à courte durée de vie (1-2 heures) pour les applications web
- Implémentez un mécanisme de refresh automatique avant expiration
- Pour les scripts batch, créez un token dédié avec une politique restrictive

### 2.8.2 Gestion des Credentials

**✓ Bonnes pratiques** :

- Stockez les credentials dans un vault (HashiCorp Vault, AWS Secrets Manager)
- Utilisez des variables d'environnement pour les scripts
- Limitez les permissions au minimum nécessaire (principe du moindre privilège)

**✗ À éviter** :

- Coder en dur les mots de passe dans le code source
- Partager un token admin entre plusieurs applications
- Créer des tokens avec `expiration: 0`

### 2.8.3 Audit des Accès

Wazo log toutes les opérations dans les journaux système. Pour auditer :

```bash
# Voir les logs d'authentification
journalctl -u wazo-auth -f

# Rechercher les échecs d'authentification
journalctl -u wazo-auth | grep "Unauthorized"
```

---

## Résumé du Chapitre 2

| Sujet | Endpoint Clé | Point Critique |
|-------|--------------|----------------|
| **Création token** | `POST /api/auth/0.1/token` | Backend (`wazo_user`, `ldap_user`) |
| **Validation** | `GET /api/auth/0.1/token/{token}` | Vérification ACL et expiration |
| **Politiques** | `POST /api/auth/0.1/policies` | ACL templates structurés |
| **Association user-policy** | `PUT /api/auth/0.1/users/{uuid}/policies/{uuid}` | Granularité des permissions |
| **Usurpation** | Header `Wazo-Impersonation` | Pour support/debugging |
| **Multi-tenant** | Header `Wazo-Tenant` | Obligatoire en multi-tenant |

---

*Fin du Chapitre 2 — Suite : Chapitre 3 (Configuration Core)*