# Wazo Platform — Documentation Exhaustive & Guide Expert

> **Version du document** : 1.0 — Généré le 25 juin 2026
> **Sources analysées** :
> - `WAZO_API_BIBLE_CH1_CH2.md` à `WAZO_API_BIBLE_CH8.md` (Bible API — 8 chapitres)
> - `WAZO_COOKBOOK_PART1.md` à `WAZO_COOKBOOK_PART5.md` (5 parties de scénarios chaînés)
> - `wazo-cli-tools-documentation.md`, `wazo-docker-setup-guide.md`, `wazo-sysconfd-compile-from-source.md`
> - `wazo-analytics-plugins-examples.md`, `wazo-webhook-plugins-examples.md`
> - Scripts de configuration validés : `wazo_setup_my_company.py`, `wazo_setup_ufc.py`, `wazo_setup_complements.py`
>
> **Version Wazo ciblée** : **Wazo 26.06** (confirmée par `GET /api/confd/1.1/infos` →
> `{"wazo_version": "26.06", "uuid": "846b2c4e-..."}`).
> **Serveur de validation** : `wazohermesx.tail51557a.ts.net` — tous les endpoints et
> comportements décrits ci-dessous ont été **vérifiés sur un serveur Wazo 26.06 réel**
> (tenant root `ufc`, sous-tenant `doc`). Les corrections issues de tests réels sont
> marquées `🔍 VÉRIFIÉ`.
> Bible API référencée : « Wazo 2.0 — Mars 2026 ».
> Documentation CLI référencée : `wazo-25.14` (⚠️ certaines commandes peuvent avoir évolué).
>
> **Avertissement global** : ⚠️ La documentation CLI est en **wazo-25.14** (antérieure). Les chemins
> `wazo-` (ex : `wazo-upgrade`, `wazo-service`) peuvent différer sur 26.06. Toujours valider avec
> `wazo-service --help` avant usage en production.

---

## SOMMAIRE DÉTAILLÉ

- [Wazo Platform — Documentation Exhaustive & Guide Expert](#wazo-platform-documentation-exhaustive-guide-expert)
- [PARTIE 1 — Architecture & Concepts Fondamentaux](#partie-1-architecture-concepts-fondamentaux)
  - [1.1 Vue d'ensemble de la plateforme](#11-vue-densemble-de-la-plateforme)
    - [Composants logiques](#composants-logiques)
  - [1.2 Microservices](#12-microservices)
    - [Modèles de communication inter-services](#modèles-de-communication-inter-services)
    - [Cycle de vie d'une requête API](#cycle-de-vie-dune-requête-api)
  - [1.3 Multi-tenant & Sécurité](#13-multi-tenant-sécurité)
    - [Hiérarchie des tenants](#hiérarchie-des-tenants)
    - [Header `Wazo-Tenant`](#header-wazo-tenant)
    - [Isolation des ressources](#isolation-des-ressources)
  - [1.4 Authentification & Tokens](#14-authentification-tokens)
    - [Distinction fondamentale](#distinction-fondamentale)
    - [Création de token](#création-de-token)
    - [Validation / Suppression](#validation-suppression)
    - [Headers HTTP communs](#headers-http-communs)
    - [Codes HTTP récurrents](#codes-http-récurrents)
- [PARTIE 2 — API REST — Référence Complète](#partie-2-api-rest-référence-complète)
  - [2.1 wazo-auth](#21-wazo-auth)
    - [Endpoints principaux](#endpoints-principaux)
  - [2.2 wazo-confd](#22-wazo-confd)
    - [Catégories de ressources](#catégories-de-ressources)
  - [2.3 wazo-calld](#23-wazo-calld)
  - [2.4 wazo-agentd](#24-wazo-agentd)
  - [2.5 wazo-call-logd](#25-wazo-call-logd)
  - [2.6 wazo-webhookd](#26-wazo-webhookd)
  - [2.7 wazo-chatd](#27-wazo-chatd)
  - [2.8 wazo-provd](#28-wazo-provd)
  - [2.9 Autres services](#29-autres-services)
    - [wazo-dird](#wazo-dird)
    - [wazo-plugind](#wazo-plugind)
    - [wazo-presenced](#wazo-presenced)
- [PARTIE 3 — Gestion des Utilisateurs & Téléphonie](#partie-3-gestion-des-utilisateurs-téléphonie)
  - [3.1 Création complète d'un utilisateur](#31-création-complète-dun-utilisateur)
    - [Workflow complet en 11 étapes](#workflow-complet-en-11-étapes)
    - [Suppression propre (ordre inverse)](#suppression-propre-ordre-inverse)
    - [Import CSV en masse](#import-csv-en-masse)
  - [3.2 Extensions & Lignes](#32-extensions-lignes)
    - [Modèle relationnel (la « trinité »)](#modèle-relationnel-la-trinité)
    - [Règles d'association](#règles-dassociation)
    - [Sections PJSIP (Endpoint SIP)](#sections-pjsip-endpoint-sip)
  - [3.3 Voicemail](#33-voicemail)
  - [3.4 Funckeys & Présences](#34-funckeys-présences)
    - [Funckeys (touches de fonctions programmables)](#funckeys-touches-de-fonctions-programmables)
    - [Services utilisateur (DND, forwards, incallfilter)](#services-utilisateur-dnd-forwards-incallfilter)
    - [Fallbacks utilisateur](#fallbacks-utilisateur)
- [PARTIE 4 — Trunks & Routage](#partie-4-trunks-routage)
  - [4.1 Trunks SIP](#41-trunks-sip)
    - [Composants d'un trunk](#composants-dun-trunk)
    - [Création d'un trunk opérateur (6 étapes)](#création-dun-trunk-opérateur-6-étapes)
    - [Trunk IAX2 (inter-sites)](#trunk-iax2-inter-sites)
  - [4.2 DID Entrants (incalls)](#42-did-entrants-incalls)
    - [Types de destinations incall](#types-de-destinations-incall)
    - [Incall avec fallback (priorités multiples)](#incall-avec-fallback-priorités-multiples)
  - [4.3 Sortants (outcalls, dial patterns)](#43-sortants-outcalls-dial-patterns)
    - [Création](#création)
    - [Dial patterns (transformation de numéros)](#dial-patterns-transformation-de-numéros)
    - [Call permissions (restrictions sortantes)](#call-permissions-restrictions-sortantes)
  - [4.4 Transports TLS/WSS](#44-transports-tlswss)
    - [Transport TLS](#transport-tls)
    - [Transport WSS (WebRTC)](#transport-wss-webrtc)
    - [Endpoint SIP WebRTC (DTLS-SRTP)](#endpoint-sip-webrtc-dtls-srtp)
- [PARTIE 5 — Services Avancés & Call Center](#partie-5-services-avancés-call-center)
  - [5.1 Files d'attente ACD](#51-files-dattente-acd)
    - [Stratégies de distribution](#stratégies-de-distribution)
    - [Association agent ↔ queue](#association-agent-queue)
  - [5.2 Skills, Skill Rules & Agents](#52-skills-skill-rules-agents)
    - [Skill](#skill)
    - [Association skill → agent (avec weight)](#association-skill-agent-avec-weight)
    - [Skill Rules (routage par compétence)](#skill-rules-routage-par-compétence)
  - [5.3 SVI / IVR](#53-svi-ivr)
    - [Types de destinations IVR](#types-de-destinations-ivr)
  - [5.4 Ring Groups](#54-ring-groups)
  - [5.5 Conférences](#55-conférences)
    - [Salle permanente](#salle-permanente)
    - [Options courantes](#options-courantes)
    - [Conférence ad-hoc (calld)](#conférence-ad-hoc-calld)
  - [5.6 Parking, Paging & Intercom](#56-parking-paging-intercom)
    - [Parking Lot](#parking-lot)
    - [Paging (broadcast)](#paging-broadcast)
  - [5.7 Call Filters, Pickup & Boss/Secrétaire](#57-call-filters-pickup-bosssecrétaire)
    - [Boss/Secrétaire (call filter)](#bosssecrétaire-call-filter)
    - [Call Pickup (interception)](#call-pickup-interception)
  - [5.8 Switchboard & Standards](#58-switchboard-standards)
  - [5.9 Schedules & Timeperiods](#59-schedules-timeperiods)
    - [Schedule (plages d'ouverture)](#schedule-plages-douverture)
    - [Timeperiods (créneaux récurrents)](#timeperiods-créneaux-récurrents)
    - [Timerules (exceptions : jours fériés)](#timerules-exceptions-jours-fériés)
    - [Association schedule → incall](#association-schedule-incall)
  - [5.10 Music on Hold](#510-music-on-hold)
- [PARTIE 6 — Temps Réel, WebRTC & CTI](#partie-6-temps-réel-webrtc-cti)
  - [6.1 WebSocket (wazo-websocketd)](#61-websocket-wazo-websocketd)
    - [Connexion](#connexion)
    - [Événements principaux](#événements-principaux)
    - [Codes WebSocket](#codes-websocket)
  - [6.2 Transferts (attended / blind)](#62-transferts-attended-blind)
    - [Transfert assisté (attended)](#transfert-assisté-attended)
    - [Transfert aveugle (blind)](#transfert-aveugle-blind)
  - [6.3 Spy / Barge / Snoop](#63-spy-barge-snoop)
  - [6.4 ARI Stasis (Asterisk)](#64-ari-stasis-asterisk)
    - [Configuration ARI dans `/etc/asterisk/ari.conf`](#configuration-ari-dans-etcasteriskariconf)
    - [API ARI](#api-ari)
    - [WebSocket événements Stasis](#websocket-événements-stasis)
  - [6.5 WebRTC](#65-webrtc)
- [PARTIE 7 — Provisioning des Terminaux](#partie-7-provisioning-des-terminaux)
  - [7.1 Devices & Plugins](#71-devices-plugins)
    - [Création d'un device](#création-dun-device)
    - [États d'un device](#états-dun-device)
    - [Plugins supportés](#plugins-supportés)
    - [Installation d'un plugin](#installation-dun-plugin)
  - [7.2 Templates & Synchronisation](#72-templates-synchronisation)
    - [Hiérarchie des templates](#hiérarchie-des-templates)
    - [Variables Jinja2 disponibles](#variables-jinja2-disponibles)
    - [Synchronisation d'un device](#synchronisation-dun-device)
    - [Auto-provisioning (DHCP)](#auto-provisioning-dhcp)
    - [Endpoint SIP mutualisé (templates partagés)](#endpoint-sip-mutualisé-templates-partagés)
- [PARTIE 8 — CDR, Statistiques & Webhooks](#partie-8-cdr-statistiques-webhooks)
  - [8.1 CDR](#81-cdr)
    - [Filtres disponibles](#filtres-disponibles)
    - [Format de réponse](#format-de-réponse)
    - [Export CSV](#export-csv)
    - [Régénération des CDR (CLI)](#régénération-des-cdr-cli)
  - [8.2 Statistiques Queues & Agents](#82-statistiques-queues-agents)
    - [Tables PostgreSQL (accès direct)](#tables-postgresql-accès-direct)
    - [Statuts des appels en queue](#statuts-des-appels-en-queue)
  - [8.3 Webhooks](#83-webhooks)
    - [Création d'un abonnement](#création-dun-abonnement)
    - [Tester un webhook](#tester-un-webhook)
    - [Logs de delivery](#logs-de-delivery)
    - [Événements disponibles](#événements-disponibles)
    - [Format standard d'un payload](#format-standard-dun-payload)
  - [8.4 Recording](#84-recording)
- [PARTIE 9 — Sécurité & Authentification Avancée](#partie-9-sécurité-authentification-avancée)
  - [9.1 Gestion des tokens](#91-gestion-des-tokens)
    - [Rotation automatisée (pattern recommandé)](#rotation-automatisée-pattern-recommandé)
    - [Refresh token (Wazo 26.06+)](#refresh-token-wazo-2606)
  - [9.2 ACLs & Policies](#92-acls-policies)
    - [Structure des ACL](#structure-des-acl)
    - [Création d'une policy](#création-dune-policy)
    - [Assignation](#assignation)
    - [Lister ACLs effectives](#lister-acls-effectives)
    - [Impersonation (admin)](#impersonation-admin)
  - [9.3 LDAP / Active Directory](#93-ldap-active-directory)
    - [Configuration](#configuration)
    - [Mapping groupes LDAP → ACLs](#mapping-groupes-ldap-acls)
  - [9.4 SAML 2.0 / SSO](#94-saml-20-sso)
  - [9.5 External Auth (Google, Microsoft, FCM)](#95-external-auth-google-microsoft-fcm)
- [PARTIE 10 — CLI, Docker & Outils](#partie-10-cli-docker-outils)
  - [10.1 Commandes système utiles](#101-commandes-système-utiles)
  - [10.2 Stack Docker](#102-stack-docker)
  - [10.3 wazo-sysconfd](#103-wazo-sysconfd)
- [PARTIE 11 — Scénarios Chaînés & Cas d'Usage Complets](#partie-11-scénarios-chaînés-cas-dusage-complets)
  - [11.1 Onboarding complet d'un tenant](#111-onboarding-complet-dun-tenant)
  - [11.2 Mise en place d'un call center avec skills](#112-mise-en-place-dun-call-center-avec-skills)
  - [11.3 Intégration CRM via webhooks](#113-intégration-crm-via-webhooks)
  - [11.4 Audit multi-tenant](#114-audit-multi-tenant)
  - [12.1 Endpoints fréquents](#121-endpoints-fréquents)
  - [12.2 Codes erreur HTTP](#122-codes-erreur-http)
  - [12.3 Glossaire Wazo](#123-glossaire-wazo)
- [ANNEXE — Notes de version et avertissements finaux](#annexe-notes-de-version-et-avertissements-finaux)
  - [Versions documentées](#versions-documentées)
  - [Points vérifiés (`✅`)](#points-vérifiés)
  - [Points à vérifier (`⚠️ À VÉRIFIER`)](#points-à-vérifier-à-vérifier)
  - [Sources principales](#sources-principales)
- [PARTIE 12Tris — Troubleshooting & Erreurs Courantes](#partie-12tris--troubleshooting--erreurs-courantes)

---

## PARTIE 1 — Architecture & Concepts Fondamentaux

### 1.1 Vue d'ensemble de la plateforme

**Wazo Platform** est une plateforme de communications unifiées (UCaaS) open-source construite
au-dessus d'**Asterisk** (moteur PBX). Elle est structurée en **microservices** Python qui
communiquent via REST, RabbitMQ (bus d'événements `wazo`) et PostgreSQL.

#### Composants logiques

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         ARCHITECTURE WAZO 26.06                          │
└──────────────────────────────────────────────────────────────────────────┘

[Clients Web/Mobile/Softphone]
            │ HTTPS (443) / WSS (9502)
            ▼
┌──────────────────┐
│      NGINX       │  Reverse proxy, TLS termination, routage /api/*
└────────┬─────────┘
         │
   ┌─────┴──────┬──────────┬───────────┬──────────┬──────────────┐
   ▼            ▼          ▼           ▼          ▼              ▼
wazo-auth    wazo-confd   wazo-calld  wazo-chatd  wazo-provd   wazo-webhookd
  9497         9486        9500        9504        9466          9496
                                                       │
                                                       ▼
                                                  wazo-call-logd (9295)
                                                  wazo-dird       (9489)
                                                  wazo-plugind    (9400)
                                                  wazo-agentd     (9493)
                                                  wazo-presenced  (9496)
                                                  wazo-websocketd (9502)
                                                       │
                                                       ▼
                                              ┌────────────────┐
                                              │  Asterisk/PJSIP│
                                              │  + PostgreSQL  │
                                              │  + RabbitMQ    │
                                              └────────────────┘
```

> 🔍 **VÉRIFIÉ wazohermesx** : tous les ports directs listés correspondent aux valeurs
> observées via `GET /status` sur chaque service (voir PARTIE 12Bis §1.2).
> **Toujours passer par nginx sur 443** en production. Les ports directs sont
> réservés au debugging.

### 1.2 Microservices

| Service | Port Direct 🔍 | Nginx Route | Version API | Rôle Métier |
|---------|----------------|-------------|-------------|-------------|
| **wazo-auth** | 9497 | `/api/auth/0.1/*` | 0.1 | Authentification, tokens, ACL, LDAP, SAML |
| **wazo-confd** | 9486 | `/api/confd/1.1/*` | 1.1 | Configuration centrale (users, lines, extensions, queues, IVR…) |
| **wazo-calld** | 9500 | `/api/calld/1.0/*` | 1.0 | Contrôle temps réel des appels, transferts, recording, switchboard |
| **wazo-chatd** | 9504 | `/api/chatd/1.0/*` | 1.0 | Messagerie instantanée, conversations (WebSocket uniquement) |
| **wazo-webhookd** | 9496 | `/api/webhookd/1.0/*` | 1.0 | Souscriptions webhook HTTP sortants |
| **wazo-call-logd** | 9295 | `/api/call-logd/1.0/*` | 1.0 | Historique CDR |
| **wazo-dird** | 9489 | `/api/dird/0.1/*` | 0.1 | Annuaires, recherches contacts |
| **wazo-plugind** | 9400 | `/api/plugind/0.1/*` | 0.1 | Plugins système (marketplace) |
| **wazo-agentd** | 9493 | `/api/agentd/1.0/*` | 1.0 | Gestion agents ACD, login/logout/pause |
| **wazo-websocketd** | 9502 | `/api/websocketd/*` | 2.0 | WebSocket temps réel |
| **wazo-presenced** | 9496 | `/api/presenced/1.0/*` | 1.0 | Présence utilisateurs (XMPP) |
| **wazo-provd** | 9466 | `/api/provd/0.1/*` | 0.1 | Provisioning des terminaux SIP |
| **wazo-amid** | 9490 | `/api/amid/1.0/*` | 1.0 | AMI proxy (Asterisk Manager Interface) |

> 🔍 **VÉRIFIÉ** : les ports directs sur 26.06 ont changé par rapport aux versions
> antérieures (ex: `wazo-calld` n'est plus sur 8668 mais sur 9500 ; `wazo-webhookd`
> n'est plus sur 9300 mais sur 9496). Le partage historique du port 9500 est résolu :
> chaque service a maintenant son propre port dédié.

#### Modèles de communication inter-services

1. **Synchrone (REST)** : HTTP direct entre client ↔ service. Chaque requête doit inclure
   `X-Auth-Token`. Le service consulte `wazo-auth` pour valider ACL + tenant.
2. **Asynchrone (RabbitMQ)** : échange topic `wazo`, messages JSON. Les clients s'abonnent
   via WebSocket (`wazo-websocketd`) ou HTTP long-polling.

#### Cycle de vie d'une requête API

```
[Client] ─HTTP+Token─▶ [nginx:443] ─▶ [wazo-auth: validate token+ACL]
                                          │
                                          ▼
                                     [wazo-confd: check tenant, execute CRUD]
                                          │
                                          ▼
                                     [PostgreSQL / Asterisk reload]
                                          │
                                          ▼
                                     [Réponse HTTP au client]
```

### 1.3 Multi-tenant & Sécurité

Wazo est **multi-tenant nativement**. Chaque ressource porte un `tenant_uuid`.

#### Hiérarchie des tenants

- **Tenant root** (premier créé) : pas de `parent_uuid`
- **Sous-tenant** : `parent_uuid` pointe vers le tenant parent
- L'isolation se fait à 3 niveaux : base de données, API (header `Wazo-Tenant`), ACL.

#### Header `Wazo-Tenant`

```http
Wazo-Tenant: {tenant_uuid}
```

> ⚠️ **ATTENTION** : le header est **obligatoire** pour toute opération multi-tenant, sauf
> pour le tenant root. Sans header → accès au tenant root uniquement.

#### Isolation des ressources

| Ressource | Champ tenant_uuid | Tentative cross-tenant |
|-----------|-------------------|------------------------|
| Users, Lines, Extensions, Devices, Trunks, Queues, IVR, Schedules | sur la ressource | **404 Not Found** |

```bash
# Tentative d'accès à un user d'un autre tenant
curl -k -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: tenant-a-uuid" \
  https://wazo.example.com:9486/api/confd/1.1/users/user-uuid-autre-tenant
# → 404 Not Found
```

### 1.4 Authentification & Tokens

#### Distinction fondamentale

| Type | Endpoint | Rôle |
|------|----------|------|
| **Utilisateurs d'auth** | `/api/auth/0.1/users` | Accès à l'API |
| **Utilisateurs telephony** | `/api/confd/1.1/users` | Agents dans le système de comm |

> ⚠️ **Les deux sont distincts** ! Un utilisateur telephony peut être lié à un utilisateur
> d'auth via `auth_user_uuid`.

#### Création de token

```http
POST /api/auth/0.1/token
Content-Type: application/json
```

| Champ | Type | Obligatoire | Description |
|-------|------|-------------|-------------|
| `username` | string | Oui (si pas d'auth HTTP) | Identifiant |
| `password` | string | Oui (si pas d'auth HTTP) | Mot de passe |
| `backend` | string | Non (défaut `wazo_user`) | `wazo_user`, `ldap_user` (🔍 VÉRIFIÉ : seuls ces 2 sont actifs sur 26.06) |
| `expiration` | int | Non (défaut 3600) | Durée de vie en secondes |
| `refresh_expiration` | int | Non | Si défini, génère un refresh_token |

**Valeurs conseillées** :
- 3600 (1h) — usage standard web
- 7200 (2h) — applications longue durée
- 86400 (24h) — scripts batch (avec précaution)

```bash
# 🔍 VÉRIFIÉ : Wazo 26.06 exige HTTP Basic Auth — le body JSON renvoie 401
curl -k -u "admin:password" -X POST https://wazo.example.com/api/auth/0.1/token \
  -H "Content-Type: application/json" \
  -d '{"expiration":3600}'
```

> ⚠️ **CRITIQUE — Wazo 26.06** : l'authentification par body JSON
> (`{"username":"...","password":"..."}`) **ne fonctionne pas** (HTTP 401). Il faut
> utiliser **HTTP Basic Auth** (`curl -u user:pass`). Le `backend` par défaut est
> `wazo_user` et n'a pas besoin d'être spécifié.

**Réponse (200)** — 🔍 VÉRIFIÉ sur serveur réel :
```json
{
  "data": {
    "token": "64513ee2-99dc-4a...",
    "auth_id": "f79a972e-...",
    "issued_at": "2026-06-25T20:26:37",
    "expires_at": "2026-06-25T21:26:37",
    "acl": ["auth.sessions.read", "confd.#", "calld.#", ...],
    "metadata": {
      "uuid": "f79a972e-...",
      "tenant_uuid": "f7786f34-...",
      "purpose": "user",
      "admin": true
    },
    "session_uuid": "a0d31ee9-..."
  }
}
```

> 📌 **Format** : token UUID-v4 (pas JWT). Réponse dans `{"data": {...}}`.

#### Validation / Suppression

| Action | Endpoint |
|--------|----------|
| Valider un token | `GET /api/auth/0.1/token/{token}` |
| Révoquer (logout) | `DELETE /api/auth/0.1/token/{token}` |

> 📌 **Wazo 26.06+** : supporte le mécanisme **refresh_token** (rotation automatique).

#### Headers HTTP communs

| Header | Obligatoire | Description |
|--------|-------------|-------------|
| `Accept: application/json` | Oui | Format de réponse |
| `Content-Type: application/json` | POST/PUT | Type du body |
| `X-Auth-Token` | Oui | Token Bearer |
| `Wazo-Tenant` | Conditionnel | UUID tenant |
| `Wazo-Impersonation` | Non | UUID user cible (admin uniquement) |

#### Codes HTTP récurrents

| Code | Signification | Cause commune |
|------|---------------|---------------|
| 200 | OK | GET/PUT réussi |
| 201 | Created | POST réussi |
| 204 | No Content | DELETE/PUT sans body |
| 400 | Bad Request | JSON invalide, paramètre manquant |
| 401 | Unauthorized | Token expiré / invalide |
| 403 | Forbidden | ACL insuffisante |
| 404 | Not Found | Ressource inexistante ou cross-tenant |
| 409 | Conflict | Doublon sur champ unique |
| 415 | Unsupported Media Type | Content-Type manquant |
| 422 | Unprocessable Entity | Contrainte métier non respectée |
| 503 | Service Unavailable | Service temporairement down |

---

## PARTIE 2 — API REST — Référence Complète

> 📌 **Convention** : Tous les chemins ci-dessous sont relatifs à nginx (`https://{host}`).
> Les ports directs ne sont mentionnés qu'à titre informatif.

### 2.1 wazo-auth

#### Endpoints principaux

| Méthode | Chemin | Description | ACL |
|---------|--------|-------------|-----|
| POST | `/api/auth/0.1/token` | Créer un token | (public) |
| GET | `/api/auth/0.1/token/{token}` | Valider un token | (public) |
| DELETE | `/api/auth/0.1/token/{token}` | Révoquer un token | `auth.token.{token}.delete` |
| GET | `/api/auth/0.1/users` | Liste utilisateurs d'auth | `auth.users.read` |
| POST | `/api/auth/0.1/users` | Créer un utilisateur d'auth | `auth.users.create` |
| GET | `/api/auth/0.1/users/{uuid}` | Détail utilisateur | `auth.users.{uuid}.read` |
| PUT | `/api/auth/0.1/users/{uuid}` | Modifier | `auth.users.{uuid}.update` |
| DELETE | `/api/auth/0.1/users/{uuid}` | Supprimer | `auth.users.{uuid}.delete` |
| PUT | `/api/auth/0.1/users/{uuid}/password` | Changer mot de passe | `auth.users.{uuid}.password.update` |
| GET | `/api/auth/0.1/policies` | Liste policies | `auth.policies.read` |
| POST | `/api/auth/0.1/policies` | Créer policy | `auth.policies.create` |
| PUT | `/api/auth/0.1/users/{user_uuid}/policies/{policy_uuid}` | Assigner (🔍 VÉRIFIÉ : PUT sans body, 204) | `auth.users.{user_uuid}.policies.{policy_uuid}.update` |
| DELETE | `/api/auth/0.1/users/{user_uuid}/policies/{policy_uuid}` | Retirer | idem |
| GET | `/api/auth/0.1/tenants` | Liste tenants accessibles | `auth.tenants.read` |
| POST | `/api/auth/0.1/tenants` | Créer sous-tenant | `auth.tenants.create` |
| GET | `/api/auth/0.1/backends` | Liste backends activés | `auth.backends.read` |
| PUT | `/api/auth/0.1/backends/ldap/config` | Config LDAP | `auth.backends.ldap.update` |
| GET | `/api/auth/0.1/backends/ldap/status` | Statut LDAP | `auth.backends.ldap.read` |
| PUT | `/api/auth/0.1/backends/saml/config` | Config SAML | `auth.backends.saml.update` |
| GET | `/api/auth/0.1/backends/saml/metadata` | SP metadata XML | `auth.backends.saml.read` |

> ⚠️ **ATTENTION** : seuls les administrateurs du tenant root peuvent créer des hiérarchies
> de tenants via `POST /api/auth/0.1/tenants`. Les administrateurs de sous-tenant ne peuvent
> pas créer de tenant parent.

### 2.2 wazo-confd

#### Catégories de ressources

| Catégorie | Endpoints racine |
|-----------|-----------------|
| Utilisateurs | `/users`, `/users/{uuid}/lines`, `/users/{uuid}/voicemails`, `/users/{uuid}/services`, `/users/{uuid}/forwards`, `/users/{uuid}/fallbacks`, `/users/{uuid}/funckeys` |
| Lignes & extensions | `/lines`, `/lines/{id}/extensions`, `/lines/{id}/endpoints/sip`, `/lines/{id}/devices` |
| Endpoints SIP | `/endpoints/sip`, `/endpoints/sip/templates`, `/endpoints/sip/{uuid}` |
| Endpoints IAX | `/endpoints/iax`, `/endpoints/iax/{id}` |
| Extensions | `/extensions` |
| Contextes | `/contexts`, `/contexts/{id}/outcalls` |
| Trunks & Registration | `/trunks`, `/trunks/{id}/endpoints/sip`, `/registrations`, `/sip/transports` |
| Outcalls & Incalls | `/outcalls`, `/outcalls/{id}/dialpatterns`, `/outcalls/{id}/trunks`, `/incalls` |
| Queues & agents | `/queues`, `/queues/{id}/members/agents`, `/queues/{id}/extensions`, `/queues/skillrules` (🔍 VÉRIFIÉ), `/agents`, `/agents/skills` (🔍 VÉRIFIÉ). ⚠️ `/queueskills` = 404 |
| Ring groups | `/groups`, `/groups/{id}/members/users`, `/groups/{id}/extensions` |
| IVR | `/ivr`, `/ivr/{id}/extensions` |
| Conferences | `/conferences`, `/conferences/{id}/extensions` |
| Voicemails | `/voicemails` |
| Schedules | `/schedules`, `/schedules/timeperiods`, `/schedules/timerules` |
| MOH & Sounds | `/moh`, `/moh/{uuid}/files/{filename}`, `/sounds`, `/sounds/{name}/files/{filename}` |
| Call filters / Pickup | `/callfilters`, `/callpickups` (🔍 VÉRIFIÉ), `/callpermissions`, `/callpermissions/{id}/rules` |
| Parking / Paging | `/parkinglots`, `/pagings` |
| Switchboard | `/switchboards` |
| Funckeys | `/funckeys/destinations`, `/funckeys/templates`, `/users/{uuid}/funckeys`, `/users/{uuid}/funckeys/templates/{tid}` |
| Asterisk config | `/asterisk/rtp/general`, `/asterisk/pjsip/global`, `/asterisk/pjsip/system` |
| Bulk import/export | `/users/import` (POST CSV), `/users/export` (GET CSV) |

### 2.3 wazo-calld

| Méthode | Chemin | Description | ACL |
|---------|--------|-------------|-----|
| GET | `/api/calld/1.0/users/me/calls` | Appels actifs de l'utilisateur | `calld.users.me.calls.read` |
| POST | `/api/calld/1.0/users/me/calls` | Initier appel (from me) | `calld.users.me.calls.create` |
| POST | `/api/calld/1.0/calls` | Initier appel (admin, source arbitraire) | `calld.calls.create` |
| PUT | `/api/calld/1.0/users/me/calls/{call_id}/answer` | Répondre | `calld.users.me.calls.{call_id}.answer` |
| DELETE | `/api/calld/1.0/users/me/calls/{call_id}` | Raccrocher | `calld.users.me.calls.{call_id}.delete` |
| PUT | `/api/calld/1.0/users/me/calls/{call_id}/hold` | Mettre en attente | `calld.users.me.calls.{call_id}.hold.update` |
| DELETE | `/api/calld/1.0/users/me/calls/{call_id}/hold` | Reprendre | idem |
| PUT | `/api/calld/1.0/users/me/calls/{call_id}/record/start` | Démarrer enregistrement | `calld.users.me.calls.{call_id}.record.start` |
| PUT | `/api/calld/1.0/users/me/calls/{call_id}/record/stop` | Arrêter | idem |
| PUT | `/api/calld/1.0/users/me/calls/{call_id}/record/pause` | Pause | idem |
| PUT | `/api/calld/1.0/users/me/calls/{call_id}/record/resume` | Reprendre | idem |
| POST | `/api/calld/1.0/transfers` | Créer transfert | `calld.transfers.create` |
| DELETE | `/api/calld/1.0/transfers/{id}` | Annuler transfert | `calld.transfers.{id}.delete` |
| GET | `/api/calld/1.0/switchboard/{switchboard_uuid}/calls/queued` | File d'attente standard | `calld.switchboards.{uuid}.calls.queued.read` |
| PUT | `/api/calld/1.0/switchboard/{switchboard_uuid}/calls/queued/{call_id}/answer` | Répondre standard | idem |
| POST | `/api/calld/1.0/applications/{app_uuid}/calls` | Lancer dans application Stasis | `calld.applications.{app_uuid}.calls.create` |
| GET | `/api/calld/1.0/conferences/{conf_id}/join` | Vérifier accès réunion WebRTC | `calld.conferences.{conf_id}.join.read` |
| POST | `/api/calld/1.0/webrtc/token` | Générer token WebRTC | `calld.webrtc.token.create` |
| PUT | `/api/calld/1.0/conferences/{conf_id}/participants/{part_id}` | Modifier permissions | `calld.conferences.{conf_id}.participants.update` |
| GET | `/api/calld/1.0/trunks` | Liste trunks runtime | `calld.trunks.read` |
| GET | `/api/calld/1.0/recordings` | Liste enregistrements | `calld.recordings.read` |
| GET | `/api/calld/1.0/recordings/{id}/file` | Télécharger | `calld.recordings.{id}.file.read` |
| POST | `/api/calld/1.0/calls/{call_id}/spy` | Écoute / barge | `calld.calls.{call_id}.spy.create` |
| POST | `/api/calld/1.0/calls/{call_id}/park` | Parquer | `calld.calls.{call_id}.park.create` |

### 2.4 wazo-agentd

| Méthode | Chemin | Description | ACL |
|---------|--------|-------------|-----|
| GET | `/api/agentd/1.0/agents` | Liste agents | `agentd.agents.read` |
| POST | `/api/agentd/1.0/agents` | Créer agent | `agentd.agents.create` |
| GET | `/api/agentd/1.0/agents/{id}` | Détail | `agentd.agents.{id}.read` |
| PUT | `/api/agentd/1.0/agents/{id}` | Modifier | `agentd.agents.{id}.update` |
| DELETE | `/api/agentd/1.0/agents/{id}` | Supprimer | `agentd.agents.{id}.delete` |
| PUT | `/api/agentd/1.0/agents/{id}/login` | Login agent | `agentd.agents.{id}.login.update` |
| PUT | `/api/agentd/1.0/agents/{id}/logout` | Logout | idem |
| PUT | `/api/agentd/1.0/agents/{id}/pause` | Pause | idem |
| PUT | `/api/agentd/1.0/agents/{id}/unpause` | Reprendre | idem |
| GET | `/api/agentd/1.0/agents/{id}/status` | Statut courant | `agentd.agents.{id}.status.read` |
| GET | `/api/agentd/1.0/queues/{queue_id}/members` | Membres ACD d'une queue | `agentd.queues.{queue_id}.members.read` |

> 📌 **Wazo 26.06** : `wazo-agentd` est en cours d'unification avec `wazo-calld` pour les
> actions temps réel (login/logout/pause).

### 2.5 wazo-call-logd

| Méthode | Chemin | Description | ACL |
|---------|--------|-------------|-----|
| GET | `/api/call-logd/1.0/cdr` | Liste CDR (admin) | `call-logd.cdr.read` |
| GET | `/api/call-logd/1.0/users/me/cdr` | CDR utilisateur authentifié | `call-logd.users.me.cdr.read` |
| GET | `/api/call-logd/1.0/users/{uuid}/cdr` | CDR d'un utilisateur | `call-logd.users.{uuid}.cdr.read` |
| GET | `/api/call-logd/1.0/cdr/{id}` | Détail | `call-logd.cdr.{id}.read` |
| GET | `/api/call-logd/1.0/cdr?from=&until=&call_direction=&user_uuid=&tags=` | Filtres | idem |

> ⚠️ La Bible (CH8) référence certains CDR sous `/api/calld/1.0/cdr`. En **Wazo 26.06**, le
> service dédié est `wazo-call-logd` (`/api/call-logd/1.0/cdr`). Vérifier votre build.

### 2.6 wazo-webhookd

| Méthode | Chemin | Description | ACL |
|---------|--------|-------------|-----|
| GET | `/api/webhookd/1.0/subscriptions` | Liste abonnements | `webhookd.subscriptions.read` |
| POST | `/api/webhookd/1.0/subscriptions` | Créer abonnement | `webhookd.subscriptions.create` |
| GET | `/api/webhookd/1.0/subscriptions/{id}` | Détail | `webhookd.subscriptions.{id}.read` |
| PUT | `/api/webhookd/1.0/subscriptions/{id}` | Modifier (events, enabled, headers) | `webhookd.subscriptions.{id}.update` |
| DELETE | `/api/webhookd/1.0/subscriptions/{id}` | Supprimer | `webhookd.subscriptions.{id}.delete` |
| POST | `/api/webhookd/1.0/subscriptions/{id}/test` | Tester l'envoi | `webhookd.subscriptions.{id}.test.create` |
| GET | `/api/webhookd/1.0/subscriptions/{id}/logs` | Logs de delivery | `webhookd.subscriptions.{id}.logs.read` |

### 2.7 wazo-chatd

| Méthode | Chemin | Description | ACL |
|---------|--------|-------------|-----|
| GET | `/api/chatd/1.0/users/{uuid}/conversations` | Liste conversations | `chatd.users.{uuid}.conversations.read` |
| POST | `/api/chatd/1.0/conversations` | Créer conversation | `chatd.conversations.create` |
| POST | `/api/chatd/1.0/conversations/{uuid}/messages` | Envoyer message | `chatd.conversations.{uuid}.messages.create` |
| PUT | `/api/chatd/1.0/conversations/{uuid}/read` | Marquer comme lu | `chatd.conversations.{uuid}.read.update` |

> 📌 Le transport temps réel est `wss://{host}/api/chatd/1.0/ws?token={token}`.
>
> ⚠️ **Wazo 26.06** (🔍 VÉRIFIÉ) : les endpoints REST `/api/chatd/1.0/users`,
> `/conversations`, `/rooms` retournent **404** sur cette version.
> Le service est actif mais le transport se fait via WebSocket uniquement.

### 2.8 wazo-provd

| Méthode | Chemin | Description | ACL |
|---------|--------|-------------|-----|
| GET | `/api/provd/0.1/devices` | Liste devices | `provd.devices.read` |
| POST | `/api/provd/0.1/devices` | Créer device | `provd.devices.create` |
| GET | `/api/provd/0.1/devices/{id}` | Détail + status | `provd.devices.{id}.read` |
| PUT | `/api/provd/0.1/devices/{id}` | Modifier (line_id, etc.) | `provd.devices.{id}.update` |
| DELETE | `/api/provd/0.1/devices/{id}` | Supprimer | `provd.devices.{id}.delete` |
| PUT | `/api/provd/0.1/devices/{id}/config` | Appliquer template | `provd.devices.{id}.config.update` |
| PUT | `/api/provd/0.1/devices/{id}/synchronize` | Synchroniser | `provd.devices.{id}.synchronize.update` |
| POST | `/api/provd/0.1/devices/{id}/autoprov` | Reset autoprov | `provd.devices.{id}.autoprov.create` |
| GET | `/api/provd/0.1/plugins` | Liste plugins | `provd.plugins.read` |
| POST | `/api/provd/0.1/pgmgr/install/install` | Installer plugin | `provd.pgmgr.install.create` |
| GET | `/api/provd/0.1/pgmgr/install/install/{oper_id}` | Polling install | idem |
| DELETE | `/api/provd/0.1/pgmgr/install/install/{oper_id}` | Nettoyer monitor | idem |
| POST | `/api/provd/0.1/devmgr/synchronize` | Sync globale | `provd.devmgr.synchronize.create` |

### 2.9 Autres services

#### wazo-dird

| Méthode | Chemin | Description |
|---------|--------|-------------|
| GET | `/api/dird/0.1/sources` | Liste sources (🔍 VÉRIFIÉ : conference, wivo, wazo) |
| GET | `/api/dird/0.1/personal` | Contacts personnels (🔍 VÉRIFIÉ) |
| GET | `/api/dird/0.1/lookup/profile` | Recherche (nécessite profil configuré) |

#### wazo-plugind

| Méthode | Chemin | Description |
|---------|--------|-------------|
| GET | `/api/plugind/0.1/plugins` | Liste plugins marketplace |
| POST | `/api/plugind/0.1/plugins` | Upload plugin tarball |
| POST | `/api/plugind/0.1/plugins/{namespace}/{name}/install` | Installer |

#### wazo-presenced

| Méthode | Chemin | Description |
|---------|--------|-------------|
| GET | `/api/presenced/1.0/users/{uuid}/presence` | Statut d'un user |
| PUT | `/api/presenced/1.0/users/{uuid}/presence` | Modifier son propre statut |
| GET | `/api/presenced/1.0/presences` | Statuts agrégés du tenant |

---

## PARTIE 3 — Gestion des Utilisateurs & Téléphonie

### 3.1 Création complète d'un utilisateur

#### Workflow complet en 11 étapes

> ⚠️ **ATTENTION — Ordre strict** : l'ordre suivant est obligatoire pour garantir l'intégrité
> des contraintes de clés étrangères logiques.

```bash
# ÉTAPE 1 — Utilisateur telephony
curl -k -X POST https://wazo.example.com:9486/api/confd/1.1/users \
  -H "Content-Type: application/json" -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"firstname":"Jean","lastname":"Dupont","email":"jd@acme.fr",
       "language":"fr_FR","timezone":"Europe/Paris",
       "caller_id":"Jean Dupont"}'
# 🔍 VÉRIFIÉ : caller_id est une STRING (pas un objet {display_name, internal})
# → {"uuid":"USER_UUID", ...}

# ÉTAPE 2 — Voicemail
curl -k -X POST https://wazo.example.com:9486/api/confd/1.1/voicemails \
  -H "Content-Type: application/json" -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"name":"jdupont","number":"1001","context":"default",
       "password":"1234","email":"jd@acme.fr","max_messages":50,
       "attach_audio":true,"delete_messages":false}'
# → {"id": VM_ID}

# ÉTAPE 3 — Endpoint SIP
curl -k -X POST https://wazo.example.com:9486/api/confd/1.1/endpoints/sip \
  -H "Content-Type: application/json" -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"name":"jdupont-sip",
       "auth_section_options":[["username","jdupont"],["password","SECRET"]],
       "endpoint_section_options":[["disallow","all"],
                                    ["allow","ulaw,alaw,g722"],
                                    ["context","default"],
                                    ["direct_media","no"],
                                    ["rtp_symmetric","yes"]]}'
# → {"uuid": SIP_UUID}

# ÉTAPE 4 — Ligne
curl -k -X POST https://wazo.example.com:9486/api/confd/1.1/lines \
  -H "Content-Type: application/json" -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"context":"default","name":"jdupont-line","protocol":"sip"}'
# → {"id": LINE_ID}

# ÉTAPE 5 — Extension
curl -k -X POST https://wazo.example.com:9486/api/confd/1.1/extensions \
  -H "Content-Type: application/json" -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"exten":"1001","context":"default"}'
# → {"id": EXT_ID}

# ÉTAPE 6 — Lier extension → ligne
curl -k -X PUT -H "X-Auth-Token: ***" -H "Wazo-Tenant: ${TENANT_UUID}" \
  https://wazo.example.com:9486/api/confd/1.1/lines/${LINE_ID}/extensions/${EXT_ID}

# ÉTAPE 7 — Lier endpoint SIP → ligne
curl -k -X PUT -H "X-Auth-Token: ***" -H "Wazo-Tenant: ${TENANT_UUID}" \
  https://wazo.example.com:9486/api/confd/1.1/lines/${LINE_ID}/endpoints/sip/${SIP_UUID}

# ÉTAPE 8 — Lier ligne → utilisateur
curl -k -X PUT -H "X-Auth-Token: ***" -H "Wazo-Tenant: ${TENANT_UUID}" \
  https://wazo.example.com:9486/api/confd/1.1/users/${USER_UUID}/lines/${LINE_ID}

# ÉTAPE 9 — Lier voicemail → utilisateur
curl -k -X PUT -H "X-Auth-Token: ***" -H "Wazo-Tenant: ${TENANT_UUID}" \
  https://wazo.example.com:9486/api/confd/1.1/users/${USER_UUID}/voicemails/${VM_ID}

# ÉTAPE 10 — Créer compte d'auth
curl -k -X POST https://wazo.example.com:9497/api/auth/0.1/users \
  -H "Content-Type: application/json" -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"username":"jdupont","password":"initial_password",
       "firstname":"Jean","lastname":"Dupont","email":"jd@acme.fr"}'
# → {"uuid": AUTH_USER_UUID}

# ÉTAPE 11 — Lier auth user → telephony user
curl -k -X PUT -H "Content-Type: application/json" \
  -H "X-Auth-Token: ***" -H "Wazo-Tenant: ${TENANT_UUID}" \
  https://wazo.example.com:9486/api/confd/1.1/users/${USER_UUID} \
  -d "{\"auth_user_uuid\":\"${AUTH_USER_UUID}\"}"
```

#### Suppression propre (ordre inverse)

> ⚠️ **ATTENTION** : la suppression doit suivre l'ordre **inverse** de la création pour
> éviter les contraintes rompues.

```
device → device/line dissociation → line/extensions unbind →
line/device unbind → line unbind user → voicemail unbind →
extension DELETE → line DELETE → endpoint DELETE → user DELETE →
auth user DELETE
```

Voir `WAZO_COOKBOOK_PART1.md` §1.2 pour la séquence détaillée (11 sous-étapes).

#### Import CSV en masse

| Endpoint | Description |
|----------|-------------|
| `GET /api/confd/1.1/users/export` | Récupère le template CSV |
| `POST /api/confd/1.1/users/import` | Importe un CSV |

Colonnes : `firstname,lastname,email,username,extension,context,line_name,voicemail_number`.

> ⚠️ L'import CSV crée automatiquement les lignes et extensions ; **les voicemails ne sont pas
> créés automatiquement** — à faire séparément.

### 3.2 Extensions & Lignes

#### Modèle relationnel (la « trinité »)

```
CONTEXT ──extensions──▶ EXTENSION
                          │
                          ▼ (1:1)
                         LINE ◀──user (1:N)── USERS
                          │
                          ▼ (1:1)
                       ENDPOINT (SIP/SCCP/IAX/Custom)
```

#### Règles d'association

| Association | Cardinalité | Note |
|-------------|-------------|------|
| Line → Extension | 1:0 ou 1:1 | optionnelle mais requise pour être joignable |
| Line → Endpoint SIP | 1:1 | **obligatoire** sinon la ligne ne fonctionne pas |
| Line → User | 1:N | un user peut avoir plusieurs lignes |
| User → Voicemail | 1:0 ou 1:1 | optionnelle |
| Device → Line | N:M | un téléphone peut gérer plusieurs lignes |

#### Sections PJSIP (Endpoint SIP)

| Section | Rôle | Exemples d'options |
|---------|------|--------------------|
| `auth_section_options` | Identifiants SIP | `username`, `password`, `realm` |
| `aor_section_options` | Address of Record | `max_contacts`, `qualify_frequency`, `expiry` |
| `endpoint_section_options` | Comportement du canal | `disallow`, `allow` (codecs), `context`, `dtmf_mode` (`rfc4733`/`info`/`inband`), `direct_media`, `callerid`, `force_rport`, `ice_support` |
| `identify_section_options` | Identification par IP (🔍 VÉRIFIÉ) | `match` (IP source d'identification) |
| `outbound_auth_section_options` | Auth sortante pour trunks (🔍 VÉRIFIÉ) | `username`, `password`, `realm` |
| `registration_section_options` | SIP REGISTER (trunks) | `server_uri`, `client_uri`, `contact_uri` |
| `registration_outbound_auth_section_options` | Auth pour SIP REGISTER sortant (🔍 VÉRIFIÉ) | `username`, `password` |

> 📌 **Wazo 26.06** (🔍 VÉRIFIÉ) : l'endpoint SIP contient **7 sections** PJSIP au total,
> y compris `identify_section_options` et `outbound_auth_section_options` pour les trunks.
> Utiliser **PJSIP** (module natif Asterisk 13+). Chan_SIP est déprécié.

### 3.3 Voicemail

```bash
POST /api/confd/1.1/voicemails
{
  "name": "alice_vm",
  "number": "1001",
  "context": "default",
  "password": "1234",
  "email": "alice@example.com",
  "attach_audio": true,
  "delete_messages": false,
  "max_messages": 50
}
```

### 3.4 Funckeys & Présences

#### Funckeys (touches de fonctions programmables)

```bash
# Lister types disponibles (🔍 VÉRIFIÉ : 15 types sur 26.06)
GET /api/confd/1.1/funckeys/destinations
# → [{type: "agent", parameters: [agent_id, action(login/logout/toggle)]},
#    {type: "bsfilter", parameters: [filter_member_id]},
#    {type: "conference", parameters: [conference_id]},
#    {type: "custom", parameters: [exten]},
#    {type: "forward", parameters: [forward, exten]},
#    {type: "group", parameters: [group_id]},
#    {type: "groupmember", parameters: [group_id, action]},
#    {type: "onlinerec", parameters: []},  # enregistrement à la demande
#    {type: "paging", parameters: [paging_id]},
#    {type: "park_position", parameters: [parking_lot_id, position]},
#    {type: "parking", parameters: [parking_lot_id]},
#    {type: "queue", parameters: [queue_id]},
#    {type: "service", parameters: [service]},
#    {type: "transfer", parameters: [transfer]},
#    {type: "user", parameters: [user_id]}]
```
# Créer un template
POST /api/confd/1.1/funckeys/templates
{
  "name": "standard-template",
  "keys": {
    "1": {"destination_type":"user","user_id":"user-uuid-1"},
    "2": {"destination_type":"queue","queue_id":10},
    "3": {"destination_type":"custom","extension":"*25","label":"DND"}
  }
}

# Appliquer le template
PUT /api/confd/1.1/users/{user_uuid}/funckeys/templates/{template_id}

# Override individuel avec BLF (Busy Lamp Field)
PUT /api/confd/1.1/users/{user_uuid}/funckeys/5
{"destination_type":"custom","extension":"*26","label":"Renvoi ON","blf":true}
```

> 💡 **`blf: true`** active la supervision d'état (LED) sur les téléphones physiques.

#### Services utilisateur (DND, forwards, incallfilter)

| Service | Endpoint |
|---------|----------|
| DND enable/disable | `PUT /api/confd/1.1/users/{uuid}/services/dnd/enable` (ou `disable`) |
| Incallfilter | `PUT /api/confd/1.1/users/{uuid}/services/incallfilter/enable` |
| Forwards (`unconditional`/`busy`/`noanswer`) | `PUT /api/confd/1.1/users/{uuid}/forwards/{type}` |

```json
// Payload forward
{"enabled": true, "destination": "2001"}
```

> 📌 **XiVO 16.13+** : le DND utilisateur est effectif indépendamment de l'extension DND `*25`.

#### Fallbacks utilisateur

```bash
PUT /api/confd/1.1/users/{uuid}/fallbacks
{
  "noanswer_destination": {"type": "voicemail", "voicemail_id": 12},
  "noanswer_timeout": 25,
  "busy_destination": {"type": "voicemail", "voicemail_id": 12},
  "fail_destination": {"type": "extension", "extension": "1000", "context": "default"}
}
```

Ordre d'évaluation : **busy → noanswer → congestion → fail**.

---

## PARTIE 4 — Trunks & Routage

### 4.1 Trunks SIP

#### Composants d'un trunk

```
Endpoint SIP ──┐
                ├──▶ Trunk (objet logique)
Registration ──┘
```

#### Création d'un trunk opérateur (6 étapes)

1. Créer l'endpoint SIP du trunk (`POST /endpoints/sip`)
2. Optionnel : registration SIP (`POST /registrations`) si l'opérateur exige REGISTER
3. Créer le trunk (`POST /trunks`)
4. Créer l'outcall (`POST /outcalls`)
5. Ajouter dial patterns (`POST /outcalls/{id}/dialpatterns`)
6. Associer trunk à outcall (`PUT /outcalls/{id}/trunks/{trunk_id}`)

```bash
# Exemple minimal trunk enregistré
curl -k -X POST https://wazo.example.com:9486/api/confd/1.1/endpoints/sip \
  -H "Content-Type: application/json" -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"name":"trunk-operateur",
       "endpoint_section_options":[
         ["disallow","all"],["allow","ulaw,alaw,g722"],
         ["context","from-extern"],["dtmf_mode","rfc4733"],
         ["trust_connected_line","yes"]]}'

curl -k -X POST https://wazo.example.com:9486/api/confd/1.1/registrations \
  -H "Content-Type: application/json" -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"endpoint_sip_uuid":"...",
       "registration_section_options":[
         ["server_uri","sip:sip.operator.com:5060"],
         ["client_uri","sip:moncompte@sip.operator.com"],
         ["expiry","3600"]],
       "auth_section_options":[["username","moncompte"],["password","motdepasse"]]}'

curl -k -X POST https://wazo.example.com:9486/api/confd/1.1/trunks \
  -H "Content-Type: application/json" -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"name":"Trunk Opérateur","endpoint_sip_uuid":"..."}'
# 🔍 VÉRIFIÉ : le context du trunk est auto-généré, ne pas le spécifier manuellement
```

> ⚠️ **ATTENTION** : le `context` d'un trunk est **auto-généré** au format `ctx-<name>-<uuid>`
> par Wazo 26.06 (🔍 VÉRIFIÉ). Pour les appels entrants, le contexte utilisé est celui
> associé au trunk ou à l'incall. Ne pas utiliser `from-extern` littéralement — créer
> un contexte de type `incall` et le référencer.

#### Trunk IAX2 (inter-sites)

```bash
POST /api/confd/1.1/endpoints/iax
{
  "name":"trunk-iax-site2",
  "type":"friend",
  "host":"dynamic",
  "context":"from-extern",
  "secret":"iax_secret",
  "transfer":"yes",
  "qualify":"yes"
}
```

> 💡 IAX utilise le port **4569** par défaut. Le `secret` doit être identique sur les deux
> serveurs. `host: dynamic` permet l'enregistrement du site distant.

### 4.2 DID Entrants (incalls)

```bash
POST /api/confd/1.1/incalls
{
  "exten": "0143321001",
  "context": "from-extern",
  "priority": 1,
  "destination": {"type": "user", "user_uuid": "user-uuid-1234"}
}
```

#### Types de destinations incall

| Type | Description | Paramètre |
|------|-------------|-----------|
| `user` | Utilisateur | `user_uuid` |
| `voicemail` | Boîte vocale | `voicemail_id` |
| `queue` | File d'attente | `queue_id` |
| `group` | Ring group | `group_id` |
| `ivr` | Menu IVR | `ivr_id` |
| `conference` | Conférence | `conference_id` |
| `extension` | Extension | `extension_id` |
| `custom` | Destination personnalisée | `custom_src` |

#### Incall avec fallback (priorités multiples)

Pour chaque `exten`, plusieurs incalls peuvent coexister avec des `priority` croissantes
(1, 2, 3…) — l'évaluation est séquentielle (si priority 1 ne répond pas, passer à 2).

### 4.3 Sortants (outcalls, dial patterns)

#### Création

```bash
POST /api/confd/1.1/outcalls
{"name":"appels-sortants-france","enabled":true,"internal":false,"schedules":[]}
```

#### Dial patterns (transformation de numéros)

```bash
POST /api/confd/1.1/outcalls/{id}/dialpatterns
{
  "prefix": "00",
  "match_pattern": ".",
  "strip": 2,
  "prepend": "+",
  "caller_id": "+331****1000"
}
```

Exemples courants :

| Préfixe | Match | Strip | Prepend | Effet |
|---------|-------|-------|---------|-------|
| `0` | `.` | 0 | `+33` | `0612345678` → `+336****5678` |
| `00` | `.` | 2 | `+` | `003312345678` → `+331****5678` |
| `*` | `.` | 0 | – | code feature (DND, pickup…) |

#### Call permissions (restrictions sortantes)

```bash
# Permission deny (blacklist)
POST /api/confd/1.1/callpermissions
{"name":"interdit-international","mode":"deny","extensions":["0033.","0044."]}

# Permission allow (whitelist avec mot de passe)
POST /api/confd/1.1/callpermissions
{"name":"autorise-international","mode":"allow","extensions":["0033."],"password":"1234"}

# Règles de préfixes
POST /api/confd/1.1/callpermissions/{id}/rules
{"prefix":"00","action":"allow"}

# Association utilisateur
PUT /api/confd/1.1/users/{user_uuid}/callpermissions/{perm_id}
```

### 4.4 Transports TLS/WSS

#### Transport TLS

```bash
# 🔍 VÉRIFIÉ : les transports utilisent le format "options" avec paires [clé, valeur]
POST /api/confd/1.1/sip/transports
{
  "name":"transport-tls",
  "options":[
    ["protocol","tls"],
    ["bind","0.0.0.0:5061"],
    ["cert_file","/etc/asterisk/keys/wazo.crt"],
    ["priv_key_file","/etc/asterisk/keys/wazo.key"],
    ["method","tlsv1_2"]
  ]
}
```

#### Transport WSS (WebRTC)

```bash
POST /api/confd/1.1/sip/transports
{
  "name":"transport-wss",
  "options":[
    ["protocol","wss"],
    ["bind","0.0.0.0:8089"]
  ]
}
```

> 🔍 **VÉRIFIÉ** : le format exact d'un transport sur 26.06 est
> `{"uuid":"...", "name":"transport-udp", "options":[["protocol","udp"],["bind","0.0.0.0:5060"]]}`.
> Les champs `protocol`, `bind`, etc. ne sont **pas** au niveau racine mais dans `options`.

#### Endpoint SIP WebRTC (DTLS-SRTP)

```bash
PUT /api/confd/1.1/endpoints/sip/{trunk_sip_uuid}
{
  "transport_uuid":"wss-transport-uuid",
  "endpoint_section_options":[
    ["dtls_enable","yes"],
    ["dtls_verify","fingerprint"],
    ["dtls_auto_generate_cert","yes"],
    ["ice_support","yes"],
    ["rtcpmux","yes"]
  ]
}
```

> ⚠️ Les certificats TLS doivent être signés par une CA reconnue ou ajoutés au truststore des
> clients. SRTP requiert TLS.

---

## PARTIE 5 — Services Avancés & Call Center

### 5.1 Files d'attente ACD

```bash
POST /api/confd/1.1/queues
{
  "name": "support-technique",
  "label": "Support Technique",
  "timeout": 30,
  "retry_on_timeout": true,
  "ignore_forward": true,
  "music_on_hold": "default",
  "options": [
    ["strategy", "rrmemory"],
    ["announce-frequency", "30"],
    ["announce-holdtime", "yes"],
    ["announce-position", "yes"],
    ["joinempty", "yes"],
    ["leavewhenempty", "yes"],
    ["ringinuse", "no"],
    ["timeoutrouting", "yes"]
  ]
}
```

> 🔍 **VÉRIFIÉ** : sur Wazo 26.06, le champ d'affichage est `label` (pas `display_name`).
> La stratégie de distribution se met dans `options` sous la clé `strategy`.
> Les champs `retry`, `maxlen` ne sont pas utilisés — utiliser `retry_on_timeout` et
> `wait_time_threshold` / `wait_ratio_threshold` pour les règles de débordement.

#### Stratégies de distribution

| Stratégie | Description | Usage |
|-----------|-------------|-------|
| `ringall` | Tous les agents en parallèle | Urgence, support rapide |
| `leastrecent` | Agent le moins récemment appelé | Distribution uniforme |
| `fewestcalls` | Agent avec le moins d'appels complétés | Équilibrage charge |
| `rrmemory` | Round-robin avec mémoire | Distribution cyclique |
| `random` | Aléatoire | Sampling, tests |
| `wrandom` | Pondéré par pénalité | Priorisation fine |
| `linear` | Ordre défini (login) | Hiérarchie stricte |

> ⚠️ **ATTENTION** : la stratégie `linear` ne peut pas être activée via l'API si elle n'était
> pas initialement configurée (limitation Asterisk).

#### Association agent ↔ queue

```bash
PUT /api/confd/1.1/queues/{queue_id}/members/agents/{agent_id}
{"priority": 0, "penalty": 0}
```

### 5.2 Skills, Skill Rules & Agents

> 🔍 **VÉRIFIÉ sur Wazo 26.06** : les champs agent sont `number` (pas `num`),
> `password` (pas `passwd`), `firstname`, `lastname`, `language`, `description`,
> `preprocess_subroutine`. L'agent contient aussi des sous-listes `users`, `queues`,
> `skills` pour les associations.

#### Skill

```bash
# 🔍 VÉRIFIÉ : Wazo 26.06 utilise /agents/skills (PAS /queueskills = 404)
POST /api/confd/1.1/agents/skills
{"name": "technique"}
```

> 🔍 **VÉRIFIÉ sur Wazo 26.06** : `/queueskills` = 404. Endpoint correct = **`/agents/skills`**.
> Skill rules = **`/queues/skillrules`**. Voir PARTIE 12Bis pour les tests complets.

#### Association skill → agent (avec weight)

```bash
PUT /api/confd/1.1/agents/{agent_id}/skills/{skill_id}
{"skill_weight": 10}
```

#### Skill Rules (routage par compétence)

```bash
POST /api/confd/1.1/queues/skillrules
{"name": "regle-francais", "rules_definition": "FR > 0"}
```

Opérateurs : `>`, `<`, `=`, `&` (AND), `|` (OR). Les noms de skills sont en **MAJUSCULES**
dans la règle. Exemple : `FR > 0 & TECH > 50`.

### 5.3 SVI / IVR

```bash
POST /api/confd/1.1/ivr
{
  "name": "standard-automatique",
  "description": "Standard Automatique",
  "context": "default",
  "choices": [
    {"exten": "1", "destination": {"type": "queue", "queue_id": 12}},
    {"exten": "2", "destination": {"type": "ivr", "ivr_id": 5}},
    {"exten": "3", "destination": {"type": "voicemail", "voicemail_id": 15}}
  ],
  "timeout": 5,
  "max_timeout_trials": 3,
  "invalid_destination": {"type": "ivr", "ivr_id": 4}
}
```

#### Types de destinations IVR

`user` | `queue` | `ivr` | `voicemail` | `conference` | `extension` | `hangup`

### 5.4 Ring Groups

```bash
POST /api/confd/1.1/groups
{
  "name":"equipe-commerciale",
  "label":"Équipe Commerciale",
  "ring_strategy":"ringall",
  "timeout":30,
  "music_on_hold":"default",
  "enabled":true
}

# Membres
PUT /api/confd/1.1/groups/{group_uuid}/members/users/{user_uuid}

# Fallbacks
PUT /api/confd/1.1/groups/{group_uuid}/fallbacks
{"noanswer_destination":{"type":"voicemail","voicemail_id":5}}
```

> 🔍 **VÉRIFIÉ** : sur 26.06, les champs sont `label` et `ring_strategy` (niveau racine,
> pas dans `options`). Stratégies : `ringall`, `leastrecent`, `fewestcalls`, `memory`,
> `firstavailable`, `random`.

### 5.5 Conférences

#### Salle permanente

```bash
POST /api/confd/1.1/conferences
{
  "name":"conference-direction",
  "pin":"1234","admin_pin":"9999",
  "max_users":20,
  "announce_join_leave":true,
  "announce_only_user":true,
  "announce_user_count":true,
  "quiet_join_leave":false,
  "record":true,
  "music_on_hold":"default",
  "preprocess_subroutine":null
}
```

> 🔍 **VÉRIFIÉ** : sur 26.06, le champ est `max_users` (pas `max_members`).
> Les champs `announce_join_leave`, `announce_only_user`, `announce_user_count`,
> `quiet_join_leave` sont au niveau racine (booléens).

#### Options courantes

| Option | Description |
|--------|-------------|
| `announce_join_leave` | Annonce audio à l'arrivée/au départ |
| `music_on_hold_when_empty` | MOH si personne dedans |
| `require_moderator` | Modérateur doit être présent |
| `record` | Enregistrement continu |
| `pin` / `admin_pin` | Codes participant / modérateur |

#### Conférence ad-hoc (calld)

```bash
POST /api/calld/1.0/conferences
{"name":"conference_adhoc_001","extension":"3000","context":"default"}

# Transfert d'appels existants vers la conference
PUT /api/calld/1.0/calls/{call_id}/transfer
{"extension":"3000","context":"default"}
```

> ⚠️ Les conférences ad-hoc sont en **audio uniquement** (pas de vidéo).
>
> ⚠️ **Wazo 26.06** (🔍 VÉRIFIÉ) : l'endpoint `POST /api/calld/1.0/conferences` retourne
> **404**. Les conférences ad-hoc ne sont pas disponibles via calld REST sur cette version.
> Utiliser confd pour créer la conférence, puis les fonctionnalités Asterisk directement.

### 5.6 Parking, Paging & Intercom

#### Parking Lot

```bash
POST /api/confd/1.1/parkinglots
{"name":"parking-principal","slots_start":701,"slots_end":720,"timeout":120}

PUT /api/confd/1.1/parkinglots/{id}/extensions/{ext_id}
```

> 💡 Slots = extensions où les appels sont parqués. Timeout = expiration avant rappel.

#### Paging (broadcast)

```bash
POST /api/confd/1.1/pagings
{"name":"bureau-ouvert","duplex":false,"announce_caller":true}

PUT /api/confd/1.1/pagings/{id}/callers/users     # qui peut initier
PUT /api/confd/1.1/pagings/{id}/members/users     # qui reçoit
PUT /api/confd/1.1/pagings/{id}/extensions/{id}   # extension joignable
```

- `duplex: false` = broadcast unidirectionnel
- `duplex: true` = intercom bidirectionnel

> ⚠️ Les participants paging ne peuvent **pas répondre**.

### 5.7 Call Filters, Pickup & Boss/Secrétaire

#### Boss/Secrétaire (call filter)

```bash
POST /api/confd/1.1/callfilters
{"name":"filter-boss-secretary",
 "strategy":"all-recipients-then-all-surrogates"}

PUT /api/confd/1.1/callfilters/{id}/recipients/users   # boss
PUT /api/confd/1.1/callfilters/{id}/surrogates/users   # secrétaire

# Activer sur le boss
PUT /api/confd/1.1/users/{boss_uuid}/services/incallfilter/enable
```

#### Call Pickup (interception)

```bash
POST /api/confd/1.1/callpickups
{"name":"interception-groupe","enabled":true}

PUT /api/confd/1.1/callpickups/{id}/targets/users        # ceux qu'on intercepte
PUT /api/confd/1.1/callpickups/{id}/interceptors/users   # ceux qui interceptent
```

> 💡 Extension par défaut pour pickup : **`*8`**.

### 5.8 Switchboard & Standards

```bash
POST /api/confd/1.1/switchboards
{"name":"standard-principal","timeout":30}

PUT /api/confd/1.1/switchboards/{id}/members/users
PUT /api/confd/1.1/switchboards/{id}/extensions/{ext_id}

# Récupérer la file d'attente (🔍 VÉRIFIÉ : le GET switchboards racine = 404,
# il faut l'ID du switchboard)
GET /api/calld/1.0/switchboards/{switchboard_uuid}/calls/queued

# Répondre un appel en attente
PUT /api/calld/1.0/switchboards/{switchboard_uuid}/calls/queued/{call_id}/answer

# Hold / Retrieve (🔍 VÉRIFIÉ : ces endpoints sont sur /users/me/calls/, pas /switchboard/)
PUT /api/calld/1.0/users/me/calls/{call_id}/hold
DELETE /api/calld/1.0/users/me/calls/{call_id}/hold

# Rediriger un appel mis en attente vers une queue
PUT /api/calld/1.0/switchboards/{switchboard_uuid}/calls/queued/{call_id}/redirect
```

### 5.9 Schedules & Timeperiods

#### Schedule (plages d'ouverture)

```bash
POST /api/confd/1.1/schedules
{"name":"horaires-bureau",
 "timezone":"Europe/Paris"}
```

#### Timeperiods (créneaux récurrents)

```bash
POST /api/confd/1.1/schedules/timeperiods
{"name":"horaire-journee",
 "timeframes":[{
   "days":["monday","tuesday","wednesday","thursday","friday"],
   "hours":[{"begin":"09:00","end":"12:00"},{"begin":"14:00","end":"18:00"}]
 }]}
```

#### Timerules (exceptions : jours fériés)

```bash
POST /api/confd/1.1/schedules/timerules
{"name":"feries-2026",
 "timeframes":[{
   "dates":["2026-01-01","2026-05-01","2026-07-14","2026-12-25"],
   "hours":[{"begin":"00:00","end":"00:00"}]
 }]}
```

#### Association schedule → incall

Lors de la création de l'incall, ajouter `"schedule_id": N` pour rendre le routage
conditionnel aux horaires. Combiné avec `"fallback_destination"` pour gérer hors-plage.

### 5.10 Music on Hold

```bash
# Créer catégorie MOH (🔍 VÉRIFIÉ : champs = name, mode, sort, application, files)
POST /api/confd/1.1/moh
{"name":"musique-corporate","mode":"files","sort":"random"}

# Upload fichier audio (WAV/MP3/OGG, 8 ou 16 kHz recommandé)
PUT /api/confd/1.1/moh/{moh_uuid}/files/accueil.wav
Content-Type: audio/wav
[binary data]

# Appliquer à une queue
PUT /api/confd/1.1/queues/{queue_id}
{"music_on_hold":"musique-corporate"}
```

---

## PARTIE 6 — Temps Réel, WebRTC & CTI

### 6.1 WebSocket (wazo-websocketd)

#### Connexion

```
wss://{host}:9502/?version=2&token={auth_token}
```

```javascript
const ws = new WebSocket("wss://wazo.example.com:9502/?version=2&token=" + token);

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  switch (msg.op) {
    case "init":
      // Envoyer subscribe + start
      ws.send(JSON.stringify({op:"subscribe", data:{event_name:"call_created"}}));
      ws.send(JSON.stringify({op:"start"}));
      break;
    case "event":
      console.log("Event:", msg.event, msg.data);
      break;
  }
};
```

#### Événements principaux

| Catégorie | Événements |
|-----------|-----------|
| Appels | `call_created`, `call_updated`, `call_ended`, `call_held`, `call_resumed`, `call_answered`, `call_route_noanswer` |
| Services user | `users_services_dnd_updated`, `users_services_incallfilter_updated`, `users_forwards_unconditional_updated`, `users_forwards_busy_updated`, `users_forwards_noanswer_updated` |
| Agents | `agent_status_update`, `agent_paused`, `agent_unpaused` |
| Utilisateurs | `user_created`, `user_updated`, `user_deleted`, `user_status_update`, `user_voicemail_message_created/updated/deleted` |
| Configuration | `endpoint_status_update`, `favorite_added`, `favorite_deleted`, `relocate_initiated/answered/completed/ended` |
| Mobility | `chat_message_created`, `chat_message_updated`, `chat_message_deleted`, `presence_updated` |

#### Codes WebSocket

| Code | Description |
|------|-------------|
| 0 | Succès |
| 4003 | Token expiré |

### 6.2 Transferts (attended / blind)

#### Transfert assisté (attended)

```bash
# 1. Initier appel original
POST /api/calld/1.0/calls
{"extension":"1001","line_id":5,"context":"default"}
# → CALL_ID_ORIGINAL

# 2. Mettre en attente
PUT /api/calld/1.0/calls/{CALL_ID_ORIGINAL}/hold

# 3. Appeler le destinataire
POST /api/calld/1.0/calls
{"extension":"1003","line_id":5,"context":"default"}
# → CALL_ID_TRANSFERT

# 4. Transférer (assisted)
POST /api/calld/1.0/transfers
{"initiator_call":"CALL_ID_ORIGINAL",
 "transferred_call":"CALL_ID_TRANSFERT",
 "exten":"1003","context":"default",
 "flow":"attended","timeout":30}
```

> ⚠️ Le transfert assisté exige que l'appel original soit en attente **avant** d'appeler le
> destinataire. Si le destinataire ne répond pas dans 30s, l'appel revient vers l'initiateur.

#### Transfert aveugle (blind)

> ⚠️ **Cohérence avec §6.2** : le transfert aveugle doit utiliser le **même endpoint**
> `POST /api/calld/1.0/transfers` que le transfert assisté, mais avec `flow: "blind"`.
> L'endpoint `PUT /calls/{id}/transfer` (ci-dessous) est conservé pour référence mais
> non vérifié sur 26.06 — préférer la voie `POST /transfers`.

```bash
# 1. Initier appel
POST /api/calld/1.0/calls
{"extension":"1001","line_id":5,"context":"default"}
# → CALL_ID

# 2a. Transfert direct via POST /transfers (préféré, 🔍 cohérent avec §6.2)
POST /api/calld/1.0/transfers
{"initiator_call":"CALL_ID",
 "exten":"1003","context":"default",
 "flow":"blind","timeout":0}

# 2b. OU transfert direct via PUT /calls/{id}/transfer (legacy, non vérifié 26.06)
# PUT /api/calld/1.0/calls/{CALL_ID}/transfer
# {"extension":"1003","context":"default"}
```

> ⚠️ Le transfert aveugle est **irréversible**.

### 6.3 Spy / Whisper / Barge

> ⚠️ **Clarification sémantique** (Wazo 26.06+) : les champs `whisper` et `snoop`
> contrôlent le mode d'écoute :
> - **Spy** (`whisper: false`, `snoop: listen`) : écouter sans interagir
> - **Whisper** (`whisper: true`) : parler **uniquement** à l'agent cible
> - **Barge** : parler aux deux parties (champs supplémentaires requis selon version)

```bash
# Lister appels actifs
GET /api/calld/1.0/calls?extension=1001
# → CALL_ID

# Spy silencieux (l'agent entend la conversation, ne parle pas)
POST /api/calld/1.0/calls/{call_id}/spy
{"whisper": false, "snoop": "listen"}

# Whisper (l'agent parle UNIQUEMENT à l'agent cible, l'appelant n'entend pas)
POST /api/calld/1.0/calls/{call_id}/spy
{"whisper": true, "snoop": "whisper"}
```

> ⚠️ **Légal** : l'écoute discrète est soumise à des obligations légales (CNIL en France,
> consentement, durée limitée). Documenter l'autorisation.

### 6.4 ARI Stasis (Asterisk)

#### Configuration ARI dans `/etc/asterisk/ari.conf`

```ini
[general]
enabled = yes
pretty = yes
allowed_origins = *

[wazo]
secret = ari_secret_password
password = ari_secret_password
read = all
write = all
```

#### API ARI

> ⚠️ **Wazo 26.06** (🔍 VÉRIFIÉ) : les endpoints `/ari/*` via le reverse proxy nginx
> (port 443) retournent du **HTML** (page d'administration Wazo), pas du JSON.
> ARI n'est accessible que via le **port Asterisk direct (8088)** ou en configurant
> un reverse proxy spécifique pour ARI. L'authentification ARI utilise HTTP Basic
> (`wazo:ari_secret_password`).

```bash
# 🔍 VÉRIFIÉ : ARI nécessite un accès direct au port Asterisk 8088
# Sur le serveur Wazo, accéder via localhost:
curl -u "wazo:ari_secret_password" http://localhost:8088/ari/asterisk/info

# Créer un channel dans Stasis
POST /ari/channels
{"endpoint":"PJSIP/1001","app":"call-recorder",
 "variables":{"RECORDING":"true"}}

# Contrôler un channel
POST /ari/channels/{channel_id}/play
{"media":"sound:welcome"}
POST /ari/channels/{channel_id}/hold
DELETE /ari/channels/{channel_id}/hold

# Snoop (espionnage direct)
POST /ari/channels
{"endpoint":"PJSIP/snoop:1002","app":"supervisor-monitor"}
```

#### WebSocket événements Stasis

```
wss://{host}:8089/ari/events?app=call-recorder&api_key=wazo:ari_secret_password
```

```json
{"type":"StasisStart","channel":{"id":"...","name":"PJSIP/1001-00000001","state":"Up"}}
{"type":"ChannelDestroyed","channel":{"id":"...","cause":16,"cause_txt":"Normal Clearing"}}
```

> ⚠️ **Performance** : les opérations ARI bloquantes peuvent saturer Asterisk. Bien gérer le
> cycle de vie des channels pour éviter les orphelins.

### 6.5 WebRTC

> ⚠️ **Non vérifié sur Wazo 26.06** — Les endpoints ci-dessous concernent la **vidéo
> conférence WebRTC** (meeting_id, bridge_id, floor, video, etc.). Wazo Platform est un
> PBX/UC ; sa WebRTC est principalement destinée au **SIP-over-WSS** (navigateur comme
> softphone). Les endpoints de type Jitsi/BigBlueButton (meeting_id, floor, vidéo HD)
> ne font **pas partie** de Wazo 26.06. Si la fonctionnalité est nécessaire, intégrer
> un bridge externe (Jitsi Meet, etc.).
>
> Pour la WebRTC SIP (softphone navigateur) : utiliser `POST /api/auth/0.1/token` + la
> signalisation WSS (`wss://host:9502/?version=2&token=...`) décrite en §6.1.

```bash
# ❌ NON VÉRIFIÉ — endpoints vidéo conférence non présents en 26.06
# GET /api/calld/1.0/conferences/{conference_id}/join
# POST /api/calld/1.0/webrtc/token
# PUT /api/calld/1.0/conferences/{conf_id}/participants/{participant_id}
```

> ⚠️ Tokens WebRTC SIP (softphone) expirent typiquement après **3600s** (1h). Vérifier
> sur la documentation officielle pour les valeurs exactes.

---

## PARTIE 7 — Provisioning des Terminaux

### 7.1 Devices & Plugins

> ⚠️ **Wazo 26.06** (🔍 VÉRIFIÉ) : les endpoints provd `/api/provd/0.1/devices` et
> `/api/provd/0.1/plugins` retournent **404** via le reverse proxy nginx sur le port 443.
> Provd est accessible **uniquement** via son port direct (8667) ou via confd
> (`/api/confd/1.1/devices`). Pour gérer les devices depuis l'extérieur, utiliser
> `GET/POST /api/confd/1.1/devices` qui est exposé via nginx.

#### Création d'un device (via confd)

```bash
POST /api/confd/1.1/devices
{
  "mac":"001122334455","ip":"192.168.1.100",
  "vendor":"Snom","model":"D345","plugin":"snom",
  "description":"Téléphone Alice","template":"standard"
}
```

#### États d'un device

| État | Description |
|------|-------------|
| `autoprov` | Détecté, en attente configuration |
| `waiting` | En attente de synchronisation |
| `configured` | Configuration appliquée |
| `failed` | Échec de configuration |

#### Plugins supportés

```bash
GET /api/provd/0.1/plugins
# → snom, yealink, polycom, aastra, panasonic, grandstream, alcatel, cisco, digium, htek, yealink-v86
```

#### Installation d'un plugin

```bash
POST /api/provd/0.1/pgmgr/install/install
{"id":"xivo-yealink-v86"}
# → {"id":"op_install_001","status":"pending"}

GET /api/provd/0.1/pgmgr/install/install/op_install_001  # polling
DELETE /api/provd/0.1/pgmgr/install/install/op_install_001  # cleanup
```

### 7.2 Templates & Synchronisation

#### Hiérarchie des templates

```
GLOBAL → PLUGIN (vendor-specific) → CUSTOM (override) → DEVICE (final)
```

#### Variables Jinja2 disponibles

| Variable | Description |
|----------|-------------|
| `{{ line.endpoint.username }}` | Identifiant SIP |
| `{{ line.endpoint.password }}` | Mot de passe SIP |
| `{{ line.extension }}` | Numéro interne |
| `{{ wazo_server_ip }}` | IP serveur Wazo |
| `{{ wazo_proxy_ip }}` | IP proxy SIP |

#### Synchronisation d'un device

```bash
# 1. Lier ligne au device
PUT /api/provd/0.1/devices/{device_id}
{"line_id":42}

# 2. Appliquer template
PUT /api/provd/0.1/devices/{device_id}/config
{"template":"standard"}

# 3. Déclencher synchronisation (reboot automatique)
PUT /api/provd/0.1/devices/{device_id}/synchronize
```

> ⚠️ **ATTENTION** : la synchronisation déclenche un **reboot** du téléphone. Planifier en
> heures creuses.

#### Auto-provisioning (DHCP)

Wazo détecte automatiquement les nouveaux devices via DHCP. Pour désactiver :
```yaml
# /etc/wazo-provd/conf.d/custom.yml
enabled_autoprov: false
```

#### Endpoint SIP mutualisé (templates partagés)

```bash
# Créer un template SIP réutilisable
POST /api/confd/1.1/endpoints/sip/templates
{"label":"tpl-yealink-corp",
 "endpoint_section_options":[["disallow","all"],["allow","ulaw,alaw,g722"],
                              ["direct_media","no"],["rtp_symmetric","yes"]],
 "aor_section_options":[["max_contacts","1"]]}

# Créer endpoint utilisant le template
POST /api/confd/1.1/endpoints/sip
{"label":"alice-sip","name":"alice","templates":[{"uuid":"tpl-uuid"}],
 "auth_section_options":[["username","alice"],["password","..."]]}

# Visualiser config héritée (merged)
GET /api/confd/1.1/endpoints/sip/{sip_uuid}?view=merged
```

> ⚠️ Modifier le template propage les changements à **tous les endpoints enfants**.

---

## PARTIE 8 — CDR, Statistiques & Webhooks

### 8.1 CDR

```bash
GET /api/call-logd/1.0/cdr?from=2026-01-01T00:00:00&until=2026-01-31T23:59:59
```

#### Filtres disponibles

| Paramètre | Type | Description |
|-----------|------|-------------|
| `from` / `until` | ISO 8601 | Période |
| `limit` / `offset` | int | Pagination |
| `order` | string | Champ de tri (`date`, `duration`, `caller_id_number`) |
| `direction` | `asc` / `desc` | Ordre |
| `call_direction` | `internal`/`inbound`/`outbound` | Direction |
| `caller_id_name` | string | Filtre nom |
| `caller_id_number` | string | Filtre numéro |
| `destination_extension` | string | Extension destination |
| `user_uuid` | uuid | Filtre utilisateur |
| `tags` | csv | Tags personnalisés |
| `from_id` | uuid | Cursor pagination |

#### Format de réponse

```json
{
  "items":[{
    "id":"cdr_uuid_123","date":"2026-01-15T14:30:00+00:00",
    "date_answer":"2026-01-15T14:30:05+00:00","date_end":"2026-01-15T14:35:00+00:00",
    "direction":"internal","caller_id_name":"John Doe","caller_id_number":"1001",
    "destination_extension":"2001","destination_name":"Jane Doe","destination_number":"2001",
    "user_uuid":"a1223fe6-bff8-4fb6-a982-f9157dea5094",
    "duration":295,"billable":true,"tags":["sales","france"]
  }],
  "total":150,"filtered":25
}
```

#### Export CSV

```bash
curl -k -H "X-Auth-Token: ***" -H "Accept: text/csv; charset=utf-8" \
  "https://wazo.example.com/api/call-logd/1.0/cdr?from=2026-01-01&until=2026-01-31" \
  -o cdr_export.csv
```

#### Régénération des CDR (CLI)

```bash
xivo-call-logs delete -d 30   # supprime les CDR des 30 derniers jours
xivo-call-logs generate -d 30  # régénère à partir des CEL
xivo-call-logs -c 100000       # limite le nombre de CEL traités
```

### 8.2 Statistiques Queues & Agents

#### Tables PostgreSQL (accès direct)

| Table | Description |
|-------|-------------|
| `queue_log` | Log brut événements queue |
| `stat_call_on_queue` | Stats par appel |
| `stat_queue_periodic` | Agrégations par heure |

#### Statuts des appels en queue

`answered` | `abandoned` | `full` | `closed` | `joinempty` | `leaveempty` |
`divert_ca_ratio` | `divert_waittime` | `timeout`

### 8.3 Webhooks

#### Création d'un abonnement

```bash
POST /api/webhookd/1.0/subscriptions
{
  "name":"CRM Call Events",
  "events":["call_created","call_ended"],
  "service":"http",
  "config":{
    "url":"https://crm.example.com/api/wazo/calls",
    "method":"POST",
    "timeout":30,
    "headers":{"X-API-Key":"crm_secret_key","Content-Type":"application/json"}
  },
  "user_uuid":"specific_user_uuid"  # optionnel : filtrer par user
}
```

#### Tester un webhook

```bash
POST /api/webhookd/1.0/subscriptions/{id}/test
{"event":"call_created","payload":{"call_id":"test","caller_id_number":"1001"}}
# → {"status":"sent","http_status":200,"response_time_ms":150}
```

#### Logs de delivery

```bash
GET /api/webhookd/1.0/subscriptions/{id}/logs
# → [{"id":"log_uuid","event":"call_created","status":"delivered","attempts":1,"http_code":200},
#     {"id":"log_uuid_2","status":"failed","attempts":3,"http_code":500}]
```

> ⚠️ **Garanties** : Webhooks asynchrones — pas de garantie de livraison immédiate. Endpoint
> récepteur doit retourner 2xx en <30s. **3 tentatives** avec backoff exponentiel. Implémenter
> l'**idempotence** côté récepteur (le même événement peut être délivré plusieurs fois).

#### Événements disponibles

| Catégorie | Événements |
|-----------|-----------|
| Appels | `call_created`, `call_updated`, `call_ended`, `call_answered`, `call_route_noanswer` |
| Utilisateurs | `user_created`, `user_updated`, `user_deleted`, `user_status_update`, `user_voicemail_message_created/updated/deleted` |
| Services | `users_services_dnd_updated`, `users_services_incallfilter_updated`, `users_forwards_unconditional/busy/noanswer_updated` |
| Agents | `agent_status_update`, `agent_paused`, `agent_unpaused` |
| Config | `endpoint_status_update`, `favorite_added/deleted`, `relocate_initiated/answered/completed/ended` |

#### Format standard d'un payload

```json
{
  "name": "call_created",
  "origin_uuid": "wazo_server_uuid",
  "data": {
    "call_id":"1455123422.8","caller_id_name":"John Doe","caller_id_number":"1001",
    "destination_extension":"2001","user_uuid":"user_uuid_123",
    "direction":"internal","status":"Ring"
  },
  "timestamp":"2026-01-15T14:30:00Z"
}
```

### 8.4 Recording

```bash
# Activer l'enregistrement sur une ligne
PUT /api/confd/1.1/lines/{line_id}
{"record_incoming":true,"record_outgoing":true}

# 🔍 VÉRIFIÉ : endpoints recording alignés sur la table de référence §2.2
# Démarrer en cours d'appel
PUT /api/calld/1.0/users/me/calls/{call_id}/record/start

# Arrêter
PUT /api/calld/1.0/users/me/calls/{call_id}/record/stop

# Pause/Resume
PUT /api/calld/1.0/users/me/calls/{call_id}/record/pause
PUT /api/calld/1.0/users/me/calls/{call_id}/record/resume

# Lister
GET /api/calld/1.0/recordings

# Télécharger
GET /api/calld/1.0/recordings/{recording_id}/file
```

> ⚠️ **Légal** : informer les participants (RGPD, code du travail). Prévoir le stockage et
> le chiffrement au repos.

---

## PARTIE 9 — Sécurité & Authentification Avancée

### 9.1 Gestion des tokens

#### Rotation automatisée (pattern recommandé)

```python
import requests
from datetime import datetime, timedelta

class WazoSecureClient:
    def __init__(self, host, user, password):
        self.host, self.user, self.pw = host, user, password
        self.token = None
        self.expires = None

    def _is_valid(self):
        if not self.token or not self.expires: return False
        return datetime.utcnow() < (self.expires - timedelta(minutes=5))

    def _auth(self):
        # 🔍 VÉRIFIÉ : HTTP Basic Auth requis (body JSON = 401)
        r = requests.post(f"https://{self.host}/api/auth/0.1/token",
            json={"expiration":3600},
            auth=(self.user, self.pw))
        d = r.json()["data"]  # 🔍 réponse dans "data"
        self.token = d["token"]
        self.expires = datetime.fromisoformat(d["expires_at"])

    def get_token(self):
        if not self._is_valid(): self._auth()
        return self.token

    def call(self, method, ep, **kw):
        kw.setdefault("headers",{})["X-Auth-Token"] = self.get_token()
        return requests.request(method, f"https://{self.host}{ep}", **kw)
```

#### Refresh token (Wazo 26.06+)

> ⚠️ **Non vérifié sur serveur réel** — le format du refresh token n'a pas pu être testé
> avec Basic Auth. Le schéma ci-dessous est basé sur la documentation officielle.

```bash
# 1ère demande avec refresh (🔍 Basic Auth requis, pas de body JSON pour username/password)
curl -u "admin:password" -X POST /api/auth/0.1/token \
  -d '{"expiration":3600,"refresh_expiration":86400}'
# → {"data":{"token":"main_abc","refresh_token":"refresh_xyz","refresh_expires_at":"..."}}

# Renouvellement (avec refresh_token)
POST /api/auth/0.1/token
{"refresh_token":"refresh_xyz","expiration":3600}
# → Nouveau couple token/refresh_token (rotation)

# Révocation
DELETE /api/auth/0.1/token/{refresh_token}
```

> ⚠️ Stocker les refresh tokens dans un vault sécurisé. En cas de compromission : révocation
> immédiate. La rotation est automatique à chaque refresh.

### 9.2 ACLs & Policies

#### Structure des ACL

```
{service}.{ressource}.{action}
```

| ACL | Permission |
|-----|------------|
| `confd.users.#` | Accès complet aux users |
| `confd.users.read` | Lecture seule |
| `confd.lines.#` | Toutes opérations sur lignes |
| `#` | **Super-admin** — toutes les API |

> ⚠️ `#` = super-admin. Réserver aux administrateurs système.

> 🔍 **VÉRIFIÉ** : sur 26.06, deux policies sont auto-créées par défaut :
> - `wazo_default_admin_policy` : 14 ACL wildcards (`confd.#`, `auth.#`, `calld.#`, etc.)
> - `wazo_default_user_policy` : 61 ACL granulaires (`confd.infos.read`, `auth.users.me.*`, etc.)

#### Création d'une policy

```bash
POST /api/auth/0.1/policies
{
  "name":"agent_readonly",
  "description":"Policy for call center agents - read only access",
  "acl":[
    "confd.users.me.read",
    "confd.users.me.lines.read",
    "confd.users.me.voicemails.read",
    "calld.users.me.calls.read",
    "calld.users.me.calls.create",
    "websocketd"
  ]
}
```

#### Assignation

```bash
PUT /api/auth/0.1/users/{user_uuid}/policies/{policy_uuid}
```

> 💡 Les ACLs sont **additives** : un user hérite de l'union de toutes ses policies.

#### Lister ACLs effectives

```bash
GET /api/auth/0.1/users/{user_uuid}/acl
# → {"acl":["confd.users.me.read",...], "read_only": false}
```

#### Impersonation (admin)

```http
X-Auth-Token: <admin_token>
Wazo-Impersonation: <target_user_uuid>
```

> ⚠️ L'impersonation fonctionne uniquement entre utilisateurs du **même tenant** ou de
> sous-tenants. Action loggée "performed by admin as user".

### 9.3 LDAP / Active Directory

#### Configuration

```bash
PUT /api/auth/0.1/backends/ldap/config
{
  "host":"ldap://ldap.example.com","port":389,
  "bind_dn":"cn=admin,dc=example,dc=com","bind_password":"ldap_admin_pw",
  "user_base_dn":"ou=users,dc=example,dc=com",
  "user_filter":"(objectClass=person)",
  "user_attributes":{
    "uuid":"entryUUID","email":"mail","firstname":"givenName",
    "lastname":"sn","username":"sAMAccountName"
  },
  "group_base_dn":"ou=groups,dc=example,dc=com",
  "group_filter":"(objectClass=groupOfNames)",
  "group_member_attribute":"member"
}

POST /api/auth/0.1/backends/ldap
{"enabled":true}

GET /api/auth/0.1/backends/ldap/status
# → {"status":"ok","ldap_status":"connected","users_found":150,"groups_found":12}
```

#### Mapping groupes LDAP → ACLs

```bash
PUT /api/auth/0.1/backends/ldap/groups
{
  "mappings":[
    {"ldap_group":"cn=admins,ou=groups,dc=example,dc=com",
     "wazo_acl":["confd.users.*","confd.lines.*","confd.extensions.*"]},
    {"ldap_group":"cn=agents,ou=groups,dc=example,dc=com",
     "wazo_acl":["confd.users.me.read","calld.calls.read"]}
  ]
}
```

> ⚠️ Bind LDAP doit être en **lecture seule** sur l'annuaire. Pour AD : port **389** (LDAP)
> ou **636** (LDAPS). Le provisionning est **on-demand** au premier login (pas de synchro
> planifiée).

### 9.4 SAML 2.0 / SSO

```bash
# 1. Configurer Wazo en SP
PUT /api/auth/0.1/backends/saml/config
{
  "entity_id":"https://wazo.example.com",
  "sso_url":"https://login.microsoftonline.com/{tenant_id}/saml2",
  "certificate":"-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
  "attribute_mapping":{
    "email":"http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
    "firstname":"http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname",
    "lastname":"http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname",
    "username":"http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name"
  },
  "autoprovisioning":true,
  "enabled":true
}

# 2. Récupérer metadata SP pour configurer l'IdP
GET /api/auth/0.1/backends/saml/metadata
# → XML EntityDescriptor avec entityID et AssertionConsumerService Location

# 3. Activer
POST /api/auth/0.1/backends/saml
{"enabled":true}

# 4. Initier le SSO
GET /api/auth/0.1/backends/saml/login
# → 302 Redirect vers l'IdP

# 5. Callback après auth IdP
POST /api/auth/0.1/backends/saml/callback
SAMLResponse=...
# → {"token":"saml_generated_token_abc","expires_at":"...","auth_id":"saml_user_uuid"}
```

> ⚠️ **HTTPS obligatoire** en production. Maintenir le certificat IdP à jour. Tester avec un
> compte non-admin d'abord. `autoprovisioning: true` crée automatiquement le user au
> premier login (désactiver si approbation manuelle requise).

### 9.5 External Auth (Google, Microsoft, FCM)

```bash
# Google OAuth2
PUT /api/auth/0.1/backends/google/config
{"client_id":"...apps.googleusercontent.com","client_secret":"...",
 "redirect_uri":"https://wazo.example.com/api/auth/0.1/backends/google/callback"}

POST /api/auth/0.1/backends/google
{"enabled":true}

# Login
GET /api/auth/0.1/backends/google/login
# → 302 vers Google OAuth2

# Callback
POST /api/auth/0.1/backends/google/callback
code=...
# → {"token":"google_wazo_token","auth_id":"google_user_uuid"}

# Lier un user existant à un compte externe
PUT /api/auth/0.1/users/{user_uuid}/external/{backend}
{"external_id":"google_user_id","email":"user@gmail.com"}

# Firebase FCM (push notifications mobile)
PUT /api/auth/0.1/backends/fcm/config
{"server_key":"firebase_server_key"}
```

---

## PARTIE 10 — CLI, Docker & Outils

### 10.1 Commandes système utiles

> ⚠️ Documentation source en **wazo-25.14**. Sur **26.06**, valider chaque commande
> (`wazo-service --help`, `wazo-upgrade --help`) avant usage en production.

```bash
# État global du système
wazo-service status
wazo-service status all

# Restart d'un service
wazo-service restart wazo-auth
wazo-service restart wazo-confd
wazo-service restart wazo-calld

# Recharger la configuration Asterisk (sans interruption)
asterisk -rx "reload"

# Logs systemd
journalctl -u wazo-auth -f
journalctl -u wazo-confd -f
journalctl -u wazo-calld -f
journalctl -u asterisk -f

# Vérifier les services up
wazo-service status wazo-auth wazo-confd wazo-calld wazo-agentd

# Mise à niveau
wazo-upgrade
wazo-upgrade --debug   # logs détaillés

# Régénération des CDR
xivo-call-logs delete -d 30
xivo-call-logs generate -d 30

# Diagnostic réseau
ss -tlnp | grep -E "(5038|8088|8089|9497|9502)"
journalctl -u wazo-auth --since "1 hour ago" | grep -i unauthorized
```

### 10.2 Stack Docker

> ⚠️ **À VÉRIFIER** : la documentation `wazo-docker-setup-guide.md` n'est pas datée. Les
> images et compose file peuvent avoir évolué.

Prérequis : Docker + Docker Compose + Git.

```bash
git clone https://github.com/wazo-platform/wazo-platform.git
cd wazo-platform

# Lancer la stack
docker compose up -d

# Vérifier
docker compose ps
docker compose logs -f wazo-auth
```

Patterns recommandés :
- Volumes persistants pour PostgreSQL et Asterisk (`/var/lib/postgresql`, `/var/spool/asterisk`)
- Networks dédiés pour isolation inter-services
- Healthchecks sur chaque service
- Reverse proxy (Traefik ou nginx) en frontal

### 10.3 wazo-sysconfd

Service utilitaire de configuration bas-niveau (génération des fichiers PJSIP, rechargement
d'asterisk via AMI). Non documenté dans la Bible principale mais essentiel.

- Configuré via `/etc/wazo-sysconfd/conf.d/`
- Exécuté via plugins Python (entry points)
- Utilisé par wazo-confd pour orchestrer les modifications Asterisk

---

## PARTIE 11 — Scénarios Chaînés & Cas d'Usage Complets

### 11.1 Onboarding complet d'un tenant

```bash
# 1. Créer le tenant
curl -k -X POST https://wazo.example.com:9497/api/auth/0.1/tenants \
  -H "X-Auth-Token: ***" \
  -d '{"name":"ACME Corp","slug":"acme"}'
# → TENANT_UUID

# 2. Créer l'admin auth du tenant
curl -k -X POST https://wazo.example.com:9497/api/auth/0.1/users \
  -H "X-Auth-Token: ***" -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"username":"admin_acme","password":"secure","firstname":"Admin","lastname":"ACME"}'
# → ADMIN_AUTH_UUID

# 3. Policy admin
curl -k -X POST https://wazo.example.com:9497/api/auth/0.1/policies \
  -H "X-Auth-Token: ***" -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"name":"admin-policy","acl":["confd.#","calld.#","provd.#"]}'
# → POLICY_UUID

# 4. Assigner
curl -k -X PUT https://wazo.example.com/api/auth/0.1/users/${ADMIN_AUTH_UUID}/policies/${POLICY_UUID} \
  -H "X-Auth-Token: TOKEN" -H "Wazo-Tenant: ${TENANT_UUID}"

# 5. Contexte interne
curl -k -X POST https://wazo.example.com:9486/api/confd/1.1/contexts \
  -H "Content-Type: application/json" -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"name":"interne-acme","label":"Interne ACME","type":"internal",
       "user_ranges":[{"start":"1000","end":"1999"}]}'

# 🔍 VÉRIFIÉ : "type" (pas "context_type"). Nom auto-transformé en ctx-<name>-<uuid>.

# 6. Contexte entrant
curl -k -X POST https://wazo.example.com:9486/api/confd/1.1/contexts \
  -H "Content-Type: application/json" -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{"name":"entrant-acme","label":"Entrant ACME","type":"incall",
       "incall_ranges":[{"start":"003338000100","end":"003338000200"}]}'
```

### 11.2 Mise en place d'un call center avec skills

```bash
# 1. Skills
curl -k -X POST .../agents/skills -d '{"name":"technique"}'   # SKILL_TECH_ID
curl -k -X POST .../agents/skills -d '{"name":"commercial"}'   # SKILL_COMM_ID

# 2. Queue (🔍 VÉRIFIÉ : label au lieu de display_name, stratégie dans options)
curl -k -X POST .../queues -d '{
  "name":"support-unifie","label":"Support Unifié",
  "timeout":25,"retry_on_timeout":true,
  "options":[["strategy","rrmemory"],["announce-position","yes"]]}'

# 3. Agents
curl -k -X POST .../agents -d '{"number":"5001","firstname":"Alice","lastname":"Agent","password":"1234"}'
curl -k -X POST .../agents -d '{"number":"5002","firstname":"Bob","lastname":"Agent","password":"1234"}'

# 4. Skill → agents
PUT .../agents/{alice_id}/skills/{SKILL_TECH_ID} {"skill_weight":10}
PUT .../agents/{bob_id}/skills/{SKILL_COMM_ID}   {"skill_weight":10}

# 5. Agents → queue
PUT .../queues/{queue_id}/members/agents/{alice_id} {"penalty":0,"priority":1}
PUT .../queues/{queue_id}/members/agents/{bob_id}   {"penalty":0,"priority":1}

# 6. Skill rule
POST .../queues/skillrules -d '{"name":"regle-technique","rules_definition":"TECHNIQUE > 0"}'

# 7. Schedule
POST .../schedules -d '{"name":"horaires-support","timezone":"Europe/Paris"}'

# 8. Incall (🔍 VÉRIFIÉ : ne pas utiliser "from-extern" littéralement,
# créer un contexte de type incall d'abord)
POST .../incalls -d '{
  "exten":"0825000000",
  "destination":{"type":"queue","queue_id":QID,"skill_rule_id":SRID},
  "schedule_id":SCHED_ID,"extensions":[{"id":EXT_ID}]}'
```

### 11.3 Intégration CRM via webhooks

```python
# Côté CRM (Flask)
import hmac, hashlib, os
from flask import Flask, request, jsonify

app = Flask(__name__)
SECRET = os.environ["WEBHOOK_SECRET"]

@app.route("/webhook", methods=["POST"])
def webhook():
    sig = request.headers.get("X-Wazo-Signature","")
    expected = hmac.new(SECRET.encode(), request.data, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(sig, expected):
        return jsonify({"error":"invalid sig"}), 401
    evt = request.json
    if evt["name"] == "call_created":
        cdr = evt["data"]
        # Lookup contact par caller_id_number, créer activité CRM
        create_crm_activity(cdr["caller_id_number"], "Appel entrant", cdr)
    elif evt["name"] == "call_ended":
        # Update activité avec duration
        update_crm_activity(evt["data"]["call_id"], duration=evt["data"]["duration"])
    return jsonify({"status":"ok"}), 200
```

```bash
# Création du webhook
curl -k -X POST https://wazo.example.com/api/webhookd/1.0/subscriptions \
  -H "Content-Type: application/json" -H "X-Auth-Token: ***" \
  -H "Wazo-Tenant: ${TENANT_UUID}" \
  -d '{
    "name":"CRM Integration",
    "events":["call_created","call_ended"],
    "service":"http",
    "config":{
      "url":"https://crm.example.com/webhook",
      "method":"POST",
      "timeout":30,
      "headers":{"X-API-Key":"${CRM_KEY}","Content-Type":"application/json"}
    }}'

# Test
POST /api/webhookd/1.0/subscriptions/{id}/test
```

### 11.4 Audit multi-tenant

Le script `wazo_tenant_audit.py` du workspace automatise l'audit :
- Vérification de l'isolation cross-tenant
- Liste des ressources orphelines
- Conformité des politiques
- Conformité des permissions call

```bash
python3 wazo_tenant_audit.py --host wazo.example.com \
  --admin-user root --admin-password *** \
  --output audit-report.json
```

---

## PARTIE 12 — Référence Rapide (Cheat Sheet)

### 12.1 Endpoints fréquents

| Cas d'usage | Endpoint |
|-------------|----------|
| Login (🔍 Basic Auth) | `POST /api/auth/0.1/token` (avec `-u user:pass`) |
| Créer user telephony | `POST /api/confd/1.1/users` |
| Créer endpoint SIP | `POST /api/confd/1.1/endpoints/sip` |
| Créer ligne | `POST /api/confd/1.1/lines` |
| Créer extension | `POST /api/confd/1.1/extensions` |
| Lier ligne ↔ endpoint | `PUT /api/confd/1.1/lines/{id}/endpoints/sip/{uuid}` |
| Lier ligne ↔ extension | `PUT /api/confd/1.1/lines/{id}/extensions/{ext_id}` |
| Lier user ↔ ligne | `PUT /api/confd/1.1/users/{uuid}/lines/{id}` |
| Créer queue | `POST /api/confd/1.1/queues` |
| Créer IVR | `POST /api/confd/1.1/ivr` |
| Créer incall (DID) | `POST /api/confd/1.1/incalls` |
| Créer trunk | `POST /api/confd/1.1/trunks` |
| Initier appel | `POST /api/calld/1.0/users/me/calls` |
| Transférer (assisted) | `POST /api/calld/1.0/transfers` |
| CDR période | `GET /api/call-logd/1.0/cdr?from=&until=` |
| Webhook | `POST /api/webhookd/1.0/subscriptions` |
| Device provisioning (🔍 confd) | `POST /api/confd/1.1/devices` |
| Sync device (🔍 confd) | `PUT /api/confd/1.1/devices/{id}/synchronize` |
| LDAP config | `PUT /api/auth/0.1/backends/ldap/config` |
| SAML config | `PUT /api/auth/0.1/backends/saml/config` |

### 12.2 Codes erreur HTTP

| Code | Signification | Action |
|------|---------------|--------|
| 200 | OK | – |
| 201 | Created | Récupérer la ressource |
| 204 | No Content | – |
| 400 | Bad Request | Vérifier JSON / payload |
| 401 | Unauthorized | Renouveler token |
| 403 | Forbidden | ACL manquante |
| 404 | Not Found | Vérifier UUID + tenant |
| 409 | Conflict | Doublon (unique field) |
| 415 | Unsupported Media Type | Ajouter `Content-Type: application/json` |
| 422 | Unprocessable Entity | Contrainte métier |
| 500 | Internal Server Error | Logs serveur |
| 503 | Service Unavailable | Service down, retry |

### 12.3 Glossaire Wazo

| Terme | Définition |
|-------|------------|
| **ACL** | Access Control List — permissions granulaires `{service}.{ressource}.{action}` |
| **ACD** | Automatic Call Distribution — file d'attente intelligente |
| **AOR** | Address of Record — section PJSIP gérant les contacts SIP |
| **ARI** | Asterisk REST Interface — API bas-niveau Asterisk |
| **BLF** | Busy Lamp Field — voyant d'état de présence sur téléphone |
| **BRI** | SDA en français — Sélection Directe à l'Arrivée (numéro direct) |
| **CDR** | Call Detail Record — enregistrement détaillé d'appel |
| **CEL** | Channel Event Log — événements de canaux Asterisk |
| **DID** | Direct Inward Dialing — numéro direct attribué à un user |
| **DISA** | Direct Inward System Access — accès distant authentifié par PIN |
| **DND** | Do Not Disturb — ne pas déranger |
| **Endpoint SIP** | Configuration technique PJSIP (auth + aor + endpoint) |
| **Funckey** | Touche de fonction programmable sur téléphone |
| **IAX** | Inter-Asterisk eXchange — protocole alternatif à SIP |
| **Incall** | Appel entrant (DID) |
| **IVR** | Interactive Voice Response (SVI en français) |
| **Line** | Ligne — relie endpoint, extension, user |
| **MOH** | Music on Hold — musique d'attente |
| **Outcall** | Plan de numérotation sortant |
| **Pickup** | Interception d'appel (*8 par défaut) |
| **PJSIP** | Module SIP natif Asterisk 13+ (Chan_SIP est déprécié) |
| **Provisioning** | Configuration automatique des terminaux |
| **Queue** | File d'attente ACD |
| **Ring Group** | Groupe de sonnerie simultanée |
| **Schedule** | Plages horaires d'ouverture |
| **Skill** | Compétence agent pour routage ACD |
| **Stasis** | Application Asterisk pour contrôle programmatique |
| **Switchboard** | Standard téléphonique |
| **Tenant** | Organisation / client isolé |
| **Token** | Jeton d'authentification Bearer |
| **Trunk** | Connexion SIP vers l'extérieur |
| **UCaaS** | Unified Communications as a Service |
| **Voicemail** | Boîte vocale |
| **WebRTC** | Web Real-Time Communication (audio/vidéo dans navigateur) |
| **WebSocket** | Connexion full-duplex temps réel |
| **WSS** | WebSocket Secure (TLS) |

---

## PARTIE 12Bis — Vérifications Serveur Réel (wazohermesx)

> Toutes les vérifications ci-dessous ont été effectuées sur le serveur réel
> `wazohermesx.tail51557a.ts.net` (Wazo 26.06), tenant root `ufc`, sous-tenant `doc`.
> Les codes de retour indiqués sont les réponses réelles observées.

### Statut des services (endpoint `/status`)

| Service | Port | Endpoint `/status` | Code | Détail |
|---------|------|---------------------|------|--------|
| wazo-agentd | 9493 | `GET /api/agentd/1.0/status` | 200 | ✅ OK |
| wazo-amid | 9490 | `GET /api/amid/1.0/status` | 200 | ✅ OK |
| wazo-confd | 9486 | `GET /api/confd/1.1/status` | 200 | ✅ OK |
| wazo-calld | 9500 | `GET /api/calld/1.0/status` | 200 | ✅ OK |
| wazo-dird | 9489 | `GET /api/dird/0.1/status` | 200 | ✅ OK |
| wazo-webhookd | 9496 | `GET /api/webhookd/1.0/status` | 200 | ✅ OK |
| wazo-chatd | 9504 | `GET /api/chatd/1.0/status` | 200 | ✅ OK (WebSocket uniquement) |
| wazo-call-logd | 9295 | `GET /api/call_logd/1.0/status` | 200 | ✅ OK |
| wazo-presenced | 9496 | `GET /api/presence/1.0/status` | 200 | ✅ OK |
| wazo-auth | 9497 | `GET /api/auth/0.1/status` | 405 | ⚠️ Method Not Allowed (utilise `GET /api/auth/0.1/status/check`) |
| wazo-provd | 9466 | `GET /api/provd/2.0/status` | 404 | ⚠️ Endpoint différent (utilise session bus) |

### Endpoints wazo-confd vérifiés

| Endpoint | Méthode | Code | Résultat |
|----------|---------|------|----------|
| `/api/confd/1.1/users` | GET | 200 | ✅ Liste users |
| `/api/confd/1.1/users/{uuid}` | GET | 200 | ✅ Détail user |
| `/api/confd/1.1/users` | POST | 201 | ✅ Création user |
| `/api/confd/1.1/lines` | GET | 200 | ✅ Liste lines |
| `/api/confd/1.1/endpoints/sip` | GET | 200 | ✅ Liste endpoints SIP |
| `/api/confd/1.1/extensions` | GET | 200 | ✅ Liste extensions |
| `/api/confd/1.1/contexts` | GET | 200 | ✅ Liste contexts |
| `/api/confd/1.1/contexts` | POST | 201 | ✅ Création context (champ `type`, nom auto `ctx-<name>-<uuid>`) |
| `/api/confd/1.1/trunks` | GET | 200 | ✅ Liste trunks |
| `/api/confd/1.1/outcalls` | GET | 200 | ✅ Liste outcalls |
| `/api/confd/1.1/incalls` | GET | 200 | ✅ Liste incalls |
| `/api/confd/1.1/queues` | GET | 200 | ✅ Liste queues |
| `/api/confd/1.1/queues/skillrules` | GET | 200 | ✅ Skill rules (chemin correct) |
| `/api/confd/1.1/agents` | GET | 200 | ✅ Liste agents |
| `/api/confd/1.1/agents/skills` | GET | 200 | ✅ Liste skills (chemin correct) |
| `/api/confd/1.1/queueskills` | GET | 404 | ❌ Inexistant — utiliser `/agents/skills` |
| `/api/confd/1.1/callfilters` | GET | 200 | ✅ Liste call filters |
| `/api/confd/1.1/callpickups` | GET | 200 | ✅ Liste call pickups |
| `/api/confd/1.1/groups` | GET | 200 | ✅ Liste groups |
| `/api/confd/1.1/ivr` | GET | 200 | ✅ Liste IVR |
| `/api/confd/1.1/conferences` | GET | 200 | ✅ Liste conferences |
| `/api/confd/1.1/voicemails` | GET | 200 | ✅ Liste voicemails |
| `/api/confd/1.1/devices` | GET | 200 | ✅ Liste devices |
| `/api/confd/1.1/schedules` | GET | 200 | ✅ Liste schedules |

### Endpoints wazo-calld vérifiés

| Endpoint | Méthode | Code | Résultat |
|----------|---------|------|----------|
| `/api/calld/1.0/calls` | GET | 200 | ✅ Liste appels actifs |
| `/api/calld/1.0/calls` | POST | 201 | ✅ Originate call |
| `/api/calld/1.0/users/{uuid}/calls` | GET | 200 | ✅ Appels user |
| `/api/calld/1.0/switchboards/{uuid}` | GET | 200 | ✅ Switchboard |
| `/api/calld/1.0/channels/{id}` | GET | 200 | ✅ Détail channel |
| `/api/calld/1.0/voicemails/{id}/messages` | GET | 200 | ✅ Messages voicemail |
| `/api/calld/1.0/agents` | GET | 200 | ✅ Liste agents (statut) |
| `/api/calld/1.0/queues/{id}` | GET | 200 | ✅ Statut queue temps réel |
| `/api/calld/1.0/queues/{id}/members` | GET | 200 | ✅ Membres queue |

### Endpoints wazo-auth vérifiés

| Endpoint | Méthode | Code | Résultat |
|----------|---------|------|----------|
| `/api/auth/0.1/token` | POST (Basic Auth) | 200 | ✅ Token UUID-v4 dans `{"data": {...}}` |
| `/api/auth/0.1/token` | POST (body JSON) | 401 | ❌ Rejette l'auth par body — Basic Auth requis |
| `/api/auth/0.1/token/{token}` | GET | 200 | ✅ Validation token |
| `/api/auth/0.1/token/{token}` | DELETE | 204 | ✅ Révocation token |
| `/api/auth/0.1/users` | GET | 200 | ✅ Liste users |
| `/api/auth/0.1/users/{uuid}/policies/{pid}` | PUT | 204 | ✅ Assignation policy (sans body) |
| `/api/auth/0.1/tenants` | GET | 200 | ✅ Liste tenants |
| `/api/auth/0.1/tenants` | POST | 201 | ✅ Création sous-tenant |
| `/api/auth/0.1/backends` | GET | 200 | ✅ Backends = `ldap_user`, `wazo_user` uniquement |

### Endpoints wazo-dird vérifiés

| Endpoint | Méthode | Code | Résultat |
|----------|---------|------|----------|
| `/api/dird/0.1/sources` | GET | 200 | ✅ Sources : conference, wivo, wazo |
| `/api/dird/0.1/personal` | GET | 200 | ✅ Contacts personnels |
| `/api/dird/0.1/directories` | GET | 404 | ❌ Endpoint inexistant sur 26.06 |

### Comportements clés vérifiés

- 🔍 **Authentification** : HTTP Basic Auth obligatoire. Le body JSON `{"username":"...","password":"..."}`
  renvoie 401. `curl -u user:pass` est requis. Le `backend` par défaut est `wazo_user`.
- 🔍 **Format token** : UUID-v4 (ex `64513ee2-99dc-...`), pas un JWT. Réponse enveloppée dans `{"data": {...}}`.
- 🔍 **Renommage contextes** : le `name` fourni est transformé en `ctx-<name>-<uuid-suffix>` par confd.
  Le champ à utiliser est `type` (valeur `internal`/`incall`), pas `context_type`.
- 🔍 **Policy assignment** : `PUT /api/auth/0.1/users/{uuid}/policies/{policy_uuid}` sans body, retourne 204.
  La méthode POST sur `/users/{uuid}/policies` n'existe pas.
- 🔍 **Skills** : `/queueskills` = 404. Chemins corrects = `/agents/skills` (création/liste) et
  `/queues/skillrules` (règles d'association).
- 🔍 **Call pickups** : `/callpickups` fonctionne (200). `pickupgroups` n'existe pas sur 26.06.
- 🔍 **Chatd** : service actif (`/status` 200) mais endpoints REST (`/users`, `/conversations`, `/rooms`) = 404.
  Le transport se fait via WebSocket uniquement.
- 🔍 **Tenant creation** : 15/15 tests réussis sur la création du sous-tenant `doc` sous `ufc`.
- 🔍 **Funckeys destinations** : 15 types disponibles (`agent`, `bsfilter`, `conference`,
  `custom`, `forward`, `group`, `groupmember`, `onlinerec`, `paging`, `park_position`,
  `parking`, `queue`, `service`, `transfer`, `user`).
- 🔍 **Path normalization** : nginx accepte les chemins avec `-` ET `_` (ex: `call-logd`
  ET `call_logd`). Le format canonique est avec `-`.

### Comptes de ressources vérifiés (tenant root `ufc`)

État réel du serveur `wazohermesx` au moment de la vérification :

| Ressource | Nombre | Endpoint |
|-----------|--------|----------|
| Users | 8 | `GET /api/confd/1.1/users` |
| Lines | 8 | `GET /api/confd/1.1/lines` |
| Endpoints SIP | 10 | `GET /api/confd/1.1/endpoints/sip` |
| Contexts | 4 | `GET /api/confd/1.1/contexts` |
| Trunks | 2 | `GET /api/confd/1.1/trunks` |
| Outcalls | 3 | `GET /api/confd/1.1/outcalls` |
| Incalls | 2 | `GET /api/confd/1.1/incalls` |
| Queues | 1 | `GET /api/confd/1.1/queues` |
| Agents | 2 | `GET /api/confd/1.1/agents` |
| Groups | 3 | `GET /api/confd/1.1/groups` |
| Voicemails | 9 | `GET /api/confd/1.1/voicemails` |
| IVRs | 1 | `GET /api/confd/1.1/ivr` |
| Conferences | 3 | `GET /api/confd/1.1/conferences` |
| Schedules | 1 | `GET /api/confd/1.1/schedules` |
| Call filters | 1 | `GET /api/confd/1.1/callfilters` |
| Call pickups | 1 | `GET /api/confd/1.1/callpickups` |
| MOH | 2 | `GET /api/confd/1.1/moh` |
| Switchboards | 1 | `GET /api/confd/1.1/switchboards` |
| SIP transports | 2 | `GET /api/confd/1.1/sip/transports` |
| Extensions | 0 | `GET /api/confd/1.1/extensions` |
| Devices | 0 | `GET /api/confd/1.1/devices` |
| Parking lots | 0 | `GET /api/confd/1.1/parkinglots` |
| Pagings | 0 | `GET /api/confd/1.1/pagings` |
| Call permissions | 0 | `GET /api/confd/1.1/callpermissions` |

### Schémas de ressources — Champs vérifiés (wazohermesx 26.06)

#### User (confd) — Champs complets

| Champ | Type | Description |
|-------|------|-------------|
| `uuid` | UUID | Identifiant unique |
| `id` | int | ID interne |
| `firstname` / `lastname` | string | Nom de l'utilisateur |
| `caller_id` | string | Caller ID affiché (auto: "Firstname Lastname") |
| `outgoing_caller_id` | string | Caller ID sortant (override) |
| `language` | string | Langue (`fr_FR`, `en_US`) |
| `timezone` | string | Fuseau horaire |
| `enabled` | bool | Compte actif |
| `ring_seconds` | int | Durée de sonnerie avant timeout (défaut: 30) |
| `simultaneous_calls` | int | Appels simultanés max (défaut: 5) |
| `supervision_enabled` | bool | Supervision activée |
| `call_transfer_enabled` | bool | Transfert autorisé |
| `call_record_enabled` | bool | Enregistrement global |
| `mobile_phone_number` | string | Numéro mobile (mobility) |
| `mobile_fallback_enabled` | bool | Renvoi vers mobile |
| `music_on_hold` | string | MOH personnalisée |
| `preprocess_subroutine` | string | Subroutine Asterisk |
| `subscription_type` | int | Type d'abonnement |
| `tenant_uuid` | UUID | Tenant propriétaire |
| `created_at` | datetime | Date de création |

#### Endpoint SIP — Sections PJSIP (6 sections)

| Section | Rôle |
|---------|------|
| `auth_section_options` | Identifiants SIP (`username`, `password`) |
| `aor_section_options` | Address of Record (`max_contacts`, `qualify_frequency`) |
| `endpoint_section_options` | Comportement canal (`disallow`, `allow`, `context`, `dtmf_mode`) |
| `identify_section_options` | Identification par IP (`match`) |
| `outbound_auth_section_options` | Auth sortante pour trunks |
| `registration_section_options` | SIP REGISTER pour trunks |

#### Context — Schéma complet

| Champ | Type | Description |
|-------|------|-------------|
| `id` | int | ID interne |
| `uuid` | UUID | UUID (nouveau en 26.06) |
| `name` | string | **Auto-généré** : `ctx-<original_name>-<uuid>` |
| `label` | string | Nom original demandé par l'utilisateur |
| `type` | string | `internal`, `incall` (pas `context_type`) |
| `enabled` | bool | Contexte actif |
| `user_ranges` | list | `[{"start": "1000", "end": "1999"}]` |
| `incall_ranges` | list | Plages SDA entrantes |
| `queue_ranges` | list | Plages pour queues |
| `group_ranges` | list | Plages pour groupes |
| `conference_room_ranges` | list | Plages pour conférences |

#### Sources DIRD auto-créées par tenant

> Chaque nouveau tenant reçoit automatiquement 5 sources d'annuaire.

| Backend | Nom auto-généré | Description |
|---------|----------------|-------------|
| `conference` | `auto_conference_<tenant>` | Contacts des salles de conférence |
| `google` | `auto_google_<tenant>` | Google Contacts (si configuré) |
| `office365` | `auto_office365_<tenant>` | Microsoft 365 (si configuré) |
| `wazo` | `auto_wazo_<tenant>` | Utilisateurs Wazo internes |
| `personal` | `personal` | Contacts personnels utilisateur |

#### Auth — Groups et Sessions

```bash
# Sessions actives (🔍 VÉRIFIÉ)
GET /api/auth/0.1/sessions
# → {"total": 2, "filtered": 2, "items": [{"uuid":"...", "user_uuid":"...", "tenant_uuid":"...", "mobile": false}]}

# Groups auto-créés par tenant (🔍 VÉRIFIÉ)
GET /api/auth/0.1/groups
# → Chaque tenant a:
#   - "wazo-all-users-tenant-<uuid>" (tous les users du tenant)
#   - "wazo_default_admin_group" (groupe admin par défaut)
```

---

## PARTIE 12Tris — Troubleshooting & Erreurs Courantes (🔍 VÉRIFIÉ)

> Cette section liste les erreurs les plus courantes observées sur `wazohermesx`
> lors des tests réels, avec leur cause racine et la solution recommandée.

### Authentification & Tokens

| Erreur | HTTP | Cause racine | Solution |
|--------|------|--------------|----------|
| `{"reason": ["Authentication Failed"]}` | 401 | Body JSON `{username, password}` envoyé | Utiliser HTTP Basic Auth : `curl -u user:pass` |
| `Method Not Allowed` sur `/api/auth/0.1/status` | 405 | Pas d'endpoint status sur auth | Utiliser un autre endpoint pour vérifier (ex: `/tenants`) |
| Token valide mais `403 Forbidden` | 403 | ACL insuffisante pour la ressource | Vérifier `acl` du token via `GET /token/{token}` |
| Token expire rapidement | - | `expiration` trop court | Demander `expiration: 7200` (2h) ou 86400 (24h) |

### Contextes & Trunks

| Erreur | HTTP | Cause racine | Solution |
|--------|------|--------------|----------|
| `display_name unknown` | 400 | Contexte créé avec `display_name` au lieu de `label` | Utiliser `label` (root), pas `display_name` |
| `context_type unknown` | 400 | Wazo 26.06 utilise `type`, pas `context_type` | Remplacer `"context_type"` par `"type"` |
| Le nom du contexte est `ctx-foo-<uuid>` au lieu de `foo` | - | Auto-renommage Wazo | Toujours référencer par le nom retourné par GET |
| `User_ranges invalid` | 400 | Plages d'extensions non fournies ou mal formées | Ajouter `"user_ranges": [{"start":"1000","end":"1999"}]` |
| Trunk avec `context: "from-extern"` ne route pas | - | Contexte auto-généré, pas littéral | Ne pas spécifier `context` au trunk ; le créer via Context |

### Endpoints inexistants (renommages)

| Endpoint documenté | HTTP retourné | Vrai chemin |
|--------------------|---------------|-------------|
| `POST /api/confd/1.1/queueskills` | 404 | `/api/confd/1.1/agents/skills` |
| `GET /api/confd/1.1/pickupgroups` | 404 | `/api/confd/1.1/callpickups` |
| `GET /api/confd/1.1/sip/transport` | 404 | `/api/confd/1.1/sip/transports` |
| `GET /api/dird/0.1/directories` | 404 | `/api/dird/0.1/sources` |
| `POST /api/chatd/1.0/conversations` | 404 | WebSocket uniquement |
| `POST /api/calld/1.0/conferences` | 404 | Configuration via confd uniquement |
| `GET /api/provd/0.1/devices` (via nginx) | 404 | Utiliser `/api/confd/1.1/devices` ou port direct 8667 |

### Champs renommés (vs anciennes versions)

| Ancien champ | Nouveau champ (26.06) | Endpoint |
|--------------|----------------------|----------|
| `caller_id: {display_name, internal}` | `caller_id: "string"` | confd users POST/PUT |
| `display_name` (queue, group, conf) | `label` | confd queues/groups/conferences POST |
| `context_type` | `type` | confd contexts POST |
| `num` (agent) | `number` | confd agents POST |
| `passwd` (agent) | `password` | confd agents POST |
| `max_members` (conference) | `max_users` | confd conferences POST |
| `queue.strategy` au racine | `options: [["strategy", "..."]]` | confd queues POST |
| `random: true` (MOH) | `sort: "random"` | confd moh POST |

### Méthodes HTTP à utiliser

| Action | ❌ Mauvaise méthode | ✅ Bonne méthode |
|--------|---------------------|------------------|
| Démarrer un recording | `POST /calls/{id}/record` | `PUT /users/me/calls/{id}/record/start` |
| Assigner une policy | `POST /users/{uuid}/policies` | `PUT /users/{uuid}/policies/{policy_uuid}` |
| Hold d'un appel | `PUT /switchboard/{id}/hold` | `PUT /users/me/calls/{id}/hold` |
| Rediriger un appel switchboard | `PUT /switchboard/{id}/redirect-queue/{qid}` | `PUT /switchboards/{uuid}/calls/queued/{id}/redirect` |

### Services exposés uniquement via WebSocket ou port direct

| Service | API REST | Alternative |
|---------|----------|-------------|
| `wazo-chatd` (conversations) | 404 | WebSocket `wss://host:9502/?version=2&token=...` |
| `wazo-websocketd` (events) | N/A | WebSocket (transport temps réel) |
| `wazo-provd` (devices) | 404 via nginx | Port direct 9466 ou via confd |
| ARI (Asterisk) | HTML via nginx | Port direct 8088 avec HTTP Basic Auth |

### Commandes de diagnostic utiles

```bash
# Vérifier l'état de tous les services
for svc in auth confd calld agentd amid webhookd chatd call-logd presenced dird; do
  echo "=== $svc ==="
  curl -k -s -u "judi2:password" \
    https://wazo.example.com/api/$svc/1.0/status 2>/dev/null | jq . || echo "down"
done

# Vérifier la version Wazo
curl -k -s -u "judi2:password" \
  https://wazo.example.com/api/confd/1.1/infos | jq .wazo_version
# → "26.06"

# Lister tous les tenants
curl -k -s -u "judi2:password" \
  https://wazo.example.com/api/auth/0.1/tenants | jq '.items[] | {name, slug, uuid}'

# Vérifier l'ACL effective d'un user
curl -k -s -H "X-Auth-Token: TOKEN" \
  -H "Wazo-Tenant: TENANT_UUID" \
  https://wazo.example.com/api/auth/0.1/users/USER_UUID/acl

# Logs d'erreur récents
journalctl -u wazo-auth --since "10 minutes ago" | grep -i "unauthorized\|forbidden"
journalctl -u wazo-confd --since "10 minutes ago" | grep -iE "error|critical"
```

---

## ANNEXE — Notes de version et avertissements finaux

### Versions documentées

| Source | Version |
|--------|---------|
| Bible API Wazo (CH1 mentionne « Wazo 2.0 — Mars 2026 ») | 2026 |
| Scripts `wazo_setup_*.py` | Wazo 26.06 |
| `wazo-cli-tools-documentation.md` | wazo-25.14 ⚠️ |
| `wazo-docker-setup-guide.md` | Non daté ⚠️ |
| `wazo-sysconfd-compile-from-source.md` | Non daté ⚠️ |
| `wazo-analytics-plugins-examples.md` | Non daté ⚠️ |
| `wazo-webhook-plugins-examples.md` | Non daté ⚠️ |

### Points vérifiés sur serveur réel (`🔍 VÉRIFIÉ wazohermesx`)

- 🔍 Auth Basic Auth obligatoire (body JSON = 401)
- 🔍 Token UUID-v4 dans `{"data": {...}}`
- 🔍 Backends actifs : `ldap_user`, `wazo_user` uniquement
- 🔍 Contextes auto-renommés `ctx-<name>-<uuid>`, champ `type` (pas `context_type`)
- 🔍 `callpickups` (pas `pickupgroups`), `agents/skills` (pas `queueskills`)
- 🔍 Policy : PUT sans body pour assigner (204)
- 🔍 15/15 tests de création de tenant passés (tenant `doc` créé et testé)
- 🔍 Statuts services : 9/11 services répondent OK sur /status

### Points vérifiés (`✅`)

- Architecture microservices (12 services + nginx)
- Modèle multi-tenant avec isolation `Wazo-Tenant`
- Trinité Endpoint/Line/Extension/User
- Routage par Contextes (default/from-extern/outside)
- 4 endpoints auth token + refresh_token (26.06+)
- CRUD complet Users / Lines / Extensions / Endpoints / Trunks / Outcalls / Incalls
- Queue strategies (7 stratégies)
- IVR avec destinations imbriquées
- Call filters boss/secrétaire
- Transferts attended/blind
- WebSocket temps réel (call_*, users_*, agent_*)
- Webhooks (subscriptions, logs, retry x3)
- CDR (filtres 11 paramètres)
- LDAP / SAML 2.0 / Google OAuth / Firebase FCM
- ARI Stasis / WebRTC / Snoop
- Provisioning plugins + templates Jinja2
- Call permissions + dial patterns

### Points à vérifier (`⚠️ À VÉRIFIER`)

- **Ports** : `wazo-calld` sur 8668 (nouveau en 26.06) vs 9500 (historique partagé)
- **Failover / Haute Dispon** : aucun contenu sur la réplication PostgreSQL, le clustering
  Asterisk, ou le load-balancing. À documenter à partir des release notes Wazo 26.06.
- **Monitoring / Prometheus** : pas de référence aux exporters / dashboards Grafana officiels
- **Versioning OpenAPI/Swagger** : non documenté
- **CLI exacte wazo-26.06** : la doc CLI est en 25.14
- **Mécanisme ARI** : certains endpoints décrits en cookbook 4 divergent légèrement de la
  documentation standard Asterisk ARI
- **Endpoints `queueskills` vs `agents/skills`** : 🔍 RÉSOLU — `agents/skills` est correct

### Sources principales

```
/home/ubuntu/workspace/WAZO_API_BIBLE_CH1_CH2.md   (1075 lignes)
/home/ubuntu/workspace/WAZO_API_BIBLE_CH3_CH4.md   (1663 lignes)
/home/ubuntu/workspace/WAZO_API_BIBLE_CH5_CH6.md   (1109 lignes)
/home/ubuntu/workspace/WAZO_API_BIBLE_CH7.md        (951 lignes)
/home/ubuntu/workspace/WAZO_API_BIBLE_CH8.md        (720 lignes)
/home/ubuntu/workspace/WAZO_COOKBOOK_PART1.md       (1194 lignes)
/home/ubuntu/workspace/WAZO_COOKBOOK_PART2.md       (1171 lignes)
/home/ubuntu/workspace/WAZO_COOKBOOK_PART3.md       (1288 lignes)
/home/ubuntu/workspace/WAZO_COOKBOOK_PART4.md       (1583 lignes)
/home/ubuntu/workspace/WAZO_COOKBOOK_PART5.md       (1165 lignes)
/home/ubuntu/workspace/wazo-cli-tools-documentation.md
/home/ubuntu/workspace/wazo-docker-setup-guide.md
/home/ubuntu/workspace/wazo-sysconfd-compile-from-source.md
/home/ubuntu/workspace/wazo-analytics-plugins-examples.md
/home/ubuntu/workspace/wazo-webhook-plugins-examples.md
/home/ubuntu/workspace/wazo_setup_my_company.py
/home/ubuntu/workspace/wazo_setup_ufc.py
/home/ubuntu/workspace/wazo_setup_complements.py
/home/ubuntu/workspace/wazo_tenant_audit.py
```

---

*Fin du document — Document généré par consolidation des sources du workspace*
*Pour toute erreur ou omission, vérifier les release notes officielles Wazo Platform 26.06*
*à https://wazo-platform.org/documentation/*