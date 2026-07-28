# Documentation d'utilisation — Collection Postman Wazo Platform 26.06

> **Fichier** : `WAZO.postman_collection.json` (337.5 KB, 308 requêtes, 22 dossiers)
> **Schéma** : Postman v2.1.0
> **Sources** : `WAZODOCS/WAZO_API_BIBLE_CH{1..9}_*.md` + `WAZODOCS/WAZO_COOKBOOK_PART{1..5}.md`
> **Généré le** : 28 juillet 2026 — Wazo 26.06 / Debian 12 / Asterisk 22

---

## 1. À qui s'adresse cette collection

- **Pentesters / chercheurs en sécurité** sur infrastructure Wazo (ex : bug bounty HackerOne VoIP/PBX).
- **Intégrateurs** développant des connecteurs CRM, helpdesk, bots vocaux (AVA) ou exports CDR.
- **Équipes DevOps** automatisant le provisioning d'utilisateurs, trunks, files d'attente, conférences.
- **Auditeurs** vérifiant la conformité d'un serveur Wazo contre la documentation de référence.

> Si vous cherchez **une seule API précise**, le tableau croisé Micro-service × Verbe HTTP en [Annexe B](#annexe-b--index-des-308-requêtes-par-dossier) permet de naviguer directement.

---

## 2. Pré-requis

| Outil | Version recommandée | Pourquoi |
|-------|---------------------|----------|
| **Postman Desktop** | v10+ (Windows, macOS, Linux) | Support natif `v2.1.0` + WebSocket expérimental |
| **Wazo Platform** | 26.06 (Asterisk 22) | Cible documentée ; anciennettes versions non testées |
| **Network** | HTTPS (port 443 nginx) **OU** accès direct aux ports services (9497, 9486, 9500, 9502, 8667…) en debug | Voir section [3.3 Accès réseau](#33-accès-r%C3%A9seau) |
| **Credentials** | 1 user `wazo-auth` (admin ou service) **+** optionnellement 1 user ARI sur port 5039 | Sections [4.1](#41-variables-denvironnement-à-renseigner) et [8](#8-cas-spécial--ari-asterisk-rest-interface) |

> **Note** Postman Web (Cloud) ne supporte pas les variables d'environnement locales hors sync ; privilégier **Postman Desktop**.

---

## 3. Installation

### 3.1 Import de la collection

1. Ouvrir Postman Desktop.
2. Cliquer sur **Import** (bouton en haut à gauche, ou `Ctrl/Cmd + O`).
3. Onglet **Upload Files** → glisser-déposer `WAZO.postman_collection.json` **OU** **Link** → coller le chemin absolu.
4. Choisir **Postman Collections v2.1.0** comme format (auto-détecté).
5. Cliquer **Import**.

La collection apparaît dans la sidebar sous le nom **"Wazo Platform 26.06 — API Reference + Cookbook"**.

![Collection sidebar]
> Si le nom est tronqué avec des points de suspension, c'est cosmétique : ouvrir la collection, le bandeau du haut affiche le nom complet.

### 3.2 Créer un environment

Une collection seule ne gère pas les variables proprement. Créer un environment dédié :

1. Panneau **Environments** (gauche) → **+** → **Create new environment**.
2. Nom : **"Wazo Sandbox 26.06"** (ou nommage interne).
3. Ajouter les variables (voir [4.1](#41-variables-denvironnement-à-renseigner)).
4. **Save**.

Pour activer : menu déroulant en haut à droite, sélectionner "Wazo Sandbox 26.06".

### 3.3 Accès réseau

Deux topologies possibles :

| Topologie | URL Postman `base_url` | Avantage |
|-----------|------------------------|----------|
| **Via nginx (recommandé)** | `wazo.example.com` (port 443 HTTPS) | Tous les services exposés via `/api/<svc>/...` |
| **Direct service (debug)** | `wazo.example.com:9486` (confd), `:9497` (auth), `:9502` (ws), `:5039` (ARI) | Bypass nginx, utile pour tail de logs |

**Pour ARI** : `base_url` ne suffit pas, l'ARI est sur le **port 5039**. Voir [section 8](#8-cas-spécial--ari-asterisk-rest-interface).

---

## 4. Configuration initiale

### 4.1 Variables d'environnement à renseigner

Les valeurs marquées `TODO_*` doivent être remplacées.

| Variable | Valeur exemple | Obligatoire ? | Comment l'obtenir |
|----------|----------------|---------------|-------------------|
| `base_url` | `wazo.example.com` | ✅ Oui | DNS ou IP du serveur |
| `basic_auth_admin` | `YWRtaW46c2VjcmV0WA==` | ✅ Oui | `printf 'admin:secret' \| base64` (sans retour chariot) |
| `wazo_token` | `<token JWT>` | ✅ Oui | Réponse de `POST /api/auth/0.1/token` |
| `tenant_uuid` | `6118e18b-17e2-49ef-a59c-0759063b9548` | ✅ Pour multi-tenant | `GET /api/auth/0.1/tenants` |
| `basic_auth_ari` | `YXZhX3VzZXI6c2VjcmV0QXJp` | ⚠ Pour ARI seulement | User ARI créé dans `/etc/asterisk/ari.d/02-name.conf` |
| `user_uuid` | UUID utilisateur | ❌ Au fil des workflows | `POST /api/confd/1.1/users` retourne `uuid` |
| `line_id` | `42` entier | ❌ Idem | Réponse de `POST /lines` |
| `endpoint_uuid`, `extension_id`, `queue_id`, `ivr_id`, `trunk_id`, `voicemail_id`, `incall_id`, `schedule_id`, … | divers | ❌ | Au cas par cas dans chaque workflow |
| `paging_extension` | `9001` | ❌ | Défini par l'admin lors de la création paging |

**Astuce Postman** : clic droit sur la collection → **Edit** → onglet **Variables** pour voir/modifier les 60 variables. Les variables **d'environnement** (dans "Wazo Sandbox 26.06") prennent **priorité** sur celles de la collection.

### 4.2 Variables de collection (déjà déclarées, valeurs `TODO_*`)

60 variables sont pré-déclarées dans la collection elle-même (`_postman_variable_scope: environment`-friendly). Elles servent de **fallback** : si Postman est ouvert sur un autre environment, les workflows cookbook fonctionnent encore (avec valeurs à renseigner manuellement).

Voir [Annexe A](#annexe-a--index-des-60-variables-de-collection) pour la liste complète.

---

## 5. Premier usage obligatoire — Génération du token

**Wazo n'utilise PAS `Authorization: Bearer …`.** Le header custom est `X-Auth-Token: <token>`, et l'obtention du token se fait via **HTTP Basic Auth** sur le port 9497 (ou nginx `/api/auth/0.1/token`).

### 5.1 Étape unique

1. Vérifier que `basic_auth_admin` est correctement renseigné dans l'environment.
2. Ouvrir **Wazo Auth / AUTH-001 · POST /token (creation token - Basic Auth)**.
3. Cliquer **Send**.
4. Réponse 201 → copier la valeur du champ **`token`** → la coller dans l'environment, variable `wazo_token`.

### 5.2 Vérification

1. Ouvrir **Wazo Auth / AUTH-009 · GET /users**.
2. Si 200 OK → token OK.
3. Si 401 Unauthorized → re-vérifier `basic_auth_admin` (encodage base64 sans CRLF).

> ⚠️ **Piège 26.06 critique** (cf. `WAZODOCS/references/WAZO_AUTH_PITFALLS.md`) : envoyer `{"username": "admin", "password": "secret"}` en JSON body **ne fonctionne pas** sur Wazo 26.06 (le champ est filtré par `TokenRequestSchema`). Utiliser **obligatoirement** le header `Authorization: Basic …`.

### 5.3 Expiration et refresh

Tokens par défaut expirent en 3600 s (1h). Le pre-request script affiche un warning si `wazo_token` est encore `TODO_AFTER_LOGIN` ou commence par `TODO`.

**Stratégie recommandée** :
- Pour usage interactif : regénérer via AUTH-001 toutes les heures.
- Pour usage automatisé : utiliser la query `POST /token` avec `refresh_token` (cf. **CK5 §5.4** dans la collection).

---

## 6. Parcours thématiques

Trois parcours typiques, illustrés par les workflows cookbook rangés en bas de la collection.

### 6.1 Créer un utilisateur complet (11 étapes)

Dossier : **Scenarios Cookbook PART 1 - Provisioning & Utilisateurs → CK1 §1.1**.

```
Workflow CK1 §1.1 (creer un user "Jean Dupont" - 11 endpoints):
1.  Wazo Confd - Endpoints/.../CONF-019 · POST /users
2.  Scenarios Cookbook PART 1 · CK1 §1.1 - Etape 2: POST /confd/voicemails
3.  Scenarios Cookbook PART 1 · CK1 §1.1 - Etape 3: POST /confd/endpoints/sip
4.  Scenarios Cookbook PART 1 · CK1 §1.1 - Etapes 4-9: chainage lignes/extensions/user/voicemail
5.  Scenarios Cookbook PART 1 · CK1 §1.1 - Etape 10: POST /auth/users
6.  Scenarios Cookbook PART 1 · CK1 §1.1 - Etape 11: PUT /confd/users/{uuid}
```

À chaque étape, **récupérer l'ID/UUID retourné** (zone bleue `Postman Variables` après le test) et le **renseigner dans l'environment** (clic droit → "Set as variable" ou manuellement).

### 6.2 Configurer un trunk SIP opérateur

Dossier : **Scenarios Cookbook PART 2 → CK2 §2.2**.

```
6 etapes: transport (POST /sip/transports)
        -> endpoint SIP trunk (POST /endpoints/sip, avec registration_section_options)
        -> trunk (POST /trunks)
        -> assoc trunk↔endpoint_sip (PUT /trunks/{id}/endpoints/sip/{uuid})
        -> outcall (POST /outcalls)
        -> assoc outcall↔trunk (POST /outcalls/{id}/trunks body {trunks:[{id}]})
```

**Variante IAX2** : cf. CK2 §2.3 (5 étapes, port 4569).

### 6.3 Configurer un centre d'appels ACD complet

Dossier : **Scenarios Cookbook PART 3 → CK3 §3.1** (9 étapes) + **CK3 §3.2** (skill rules).

Cf. **Wazo Confd - Queues, Groups, IVR, Conferences, Schedules, Agents** pour les endpoints atomiques.

### 6.4 Brancher un CRM/Webhook

Dossier : **Scenarios Cookbook PART 5 → CK5 §5.7**.

```
5 etapes: POST /webhookd/1.0/subscriptions
       -> POST /subscriptions/{id}/test
       -> GET /subscriptions
       -> GET /subscriptions/{id}/logs
       -> PUT {enabled:false} / DELETE
```

> ⚠️ Ne pas inclure `timeout` dans `config` (champ filtré sur 26.06).

### 6.5 Tester un bot vocal ARI

Dossier : **Scenarios Cookbook PART 5 → CK5 §5.8** + **Asterisk ARI** (27 requêtes).

Voir section [8](#8-cas-spécial--ari-asterisk-rest-interface) — **ne pas utiliser `base_url`**, configurer `basic_auth_ari`.

---

## 7. Authentification — gestion des tokens

### 7.1 Cycle de vie

```
+---------------------+       +---------------------+      +---------------------+
|  POST /token        |       |  Token valide       |      |  DELETE /token      |
|  (Basic Auth)       | ----> |  (header X-Auth-    | ---> |  (logout)           |
|  expiration: 3600   |       |  Token dans         |      |                     |
+---------------------+       |  chaque requete)     |      +---------------------+
                              +----------+----------+
                                         | (apres expiration)
                                         v
                              +---------------------+
                              | Re-auth via          |
                              | refresh_token ou     |
                              | POST /token a nouveau|
                              +---------------------+
```

### 7.2 Authentification par défaut dans Postman

| Service | Header utilisé |
|---------|----------------|
| `wazo-auth` (sauf `/token` initial) | `X-Auth-Token: {{wazo_token}}` |
| `wazo-confd`, `wazo-call-logd`, `wazo-webhookd` | `X-Auth-Token` + `Wazo-Tenant: {{tenant_uuid}}` |
| `wazo-calld`, `wazo-provd` | `X-Auth-Token` (Wazo-Tenant optionnel) |
| `wazo-chatd`, `wazo-presenced` | `X-Auth-Token` |
| **ARI** (`/ari/*`) | **`Authorization: Basic {{basic_auth_ari}}`** (PAS X-Auth-Token) |
| Routes publiques (`/backends/saml/login`, `/backends/google/login`) | Aucun |

> ⚠️ Le bearer auth au niveau collection (`Authorization: Bearer {{wazo_token}}`) **n'est PAS la méthode Wazo** ; il existe comme fallback. Wazo attend strictement `X-Auth-Token` dans le header. **Chaque requête ajoute déjà explicitement** `X-Auth-Token: {{wazo_token}}` via ses headers.

---

## 8. Cas spécial — ARI (Asterisk REST Interface)

### 8.1 Différences fondamentales

| ARI | calld (wazo-calld) | auth (wazo-auth) |
|-----|---------------------|------------------|
| Port **5039** | 9500 (partagé) | 9497 |
| **HTTP Basic Auth** | X-Auth-Token | X-Auth-Token |
| `/ari/...` | `/api/calld/1.0/...` | `/api/auth/0.1/...` |
| Bot vocal | Contrôle haut niveau | Tokens, policies |

### 8.2 Configuration côté serveur Wazo (pas dans Postman)

Avant d'utiliser les requêtes ARI, créer un user ARI sur le serveur Wazo :

```bash
# 1. Stopper le timer qui régénère ari.conf
sudo systemctl stop wazo-confgend.timer

# 2. Créer le fichier user (NE PAS inclure [general])
sudo tee /etc/asterisk/ari.d/02-myapp.conf <<'EOF'
[myapp_user]
type = user
read_only = no
password = $(openssl rand -base64 24)
password_format = plain
EOF

sudo chown asterisk:www-data /etc/asterisk/ari.d/02-myapp.conf
sudo chmod 0660 /etc/asterisk/ari.d/02-myapp.conf

# 3. Recharger ARI
sudo asterisk -rx "module reload res_ari"
```

> ⚠️ **Piège #22 critique** : si vous avez déjà `01-wazo.conf` (fourni par Wazo), **ne jamais** créer un fichier custom contenant une section `[general]` — cela casse le parsing ARI (`duplicate object 'general'`).

### 8.3 Renseigner `basic_auth_ari` dans Postman

```bash
printf 'myapp_user:MyPasswordGenerated' | base64
```

Coller le résultat dans la variable d'environment `basic_auth_ari`.

### 8.4 Premier test ARI

1. Ouvrir **Asterisk ARI / ARI-001 · GET /asterisk/info**.
2. Cliquer **Send**. 200 OK = Auth Basic OK.

> **Base URL** pour ARI : mettre `wazo.example.com` dans `base_url` mais les requêtes ARI utilisent en réalité `http://{{base_url}}:5039/ari/...` (le port 5039 est dans le path brut). Voir [section 10.2](#102-override-de-base_url-pour-ari).

### 8.5 Endpoints ARI notables

| Endpoint | Piège |
|----------|------|
| `POST /ari/channels` | `endpoint` doit être `Local/...@context` ou `PJSIP/...` (PAS `SIP/...`) |
| `/ari/channels/{id}/hold` | **N'existe PAS** en ARI natif. Utiliser `Mute` ou `calld PUT /users/me/calls/{id}/hold/start` |
| `/ari/bridges` | L'écoute discrète (snoop) = `POST /bridges/{id}/addChannel` avec 2 canaux (PAS d'endpoint `PJSIP/snoop:`) |
| `WS /ari/events?app=<name>&subscribeAll=events` | Port **5039** (PAS 8088) ; query `subscribeAll=events` (PAS `api_key=`) |

---

## 9. Multi-tenant

### 9.1 Vérifier la liste des tenants accessibles

**Wazo Auth / AUTH-006 · GET /tenants**

Réponse : `{"total": N, "items": [{"uuid": ..., "name": ..., "parent_uuid": ...}, ...]}`

### 9.2 Créer un sous-tenant

**Scenarios Cookbook PART 1 → CK1 §1.6 - Multi-tenant (7 etapes)** :

```
1. POST /api/auth/0.1/tenants           -> {uuid: TENANT_UUID}
2. POST /api/auth/0.1/users            (admin du sous-tenant)
3. POST /api/auth/0.1/policies         (avec acl: ["confd.#", "calld.#", "provd.#"])
4. POST /users/{uuid}/policies         (assigner)
5. POST /api/confd/1.1/contexts        (interne)
6. POST /api/confd/1.1/contexts        (entrant/incall)
7. GET  /api/auth/0.1/tenants/{uuid}
```

### 9.3 Pointer la collection sur un tenant spécifique

Mettre le `tenant_uuid` dans l'environment. **Toutes les requêtes sur les dossiers `Wazo Confd - …` envoient automatiquement** `Wazo-Tenant: {{tenant_uuid}}`.

> Pour le **root tenant**, laisser la variable vide et cliquer "remove header" sur les en-têtes `Wazo-Tenant` (rare en pratique).

---

## 10. Réponses attendues & erreurs courantes

### 10.1 Codes HTTP principaux

| Code | Signification | Action recommandée |
|------|---------------|---------------------|
| **200** | OK — payload présent | Lire JSON |
| **201** | Created | Sauvegarder l'`id`/`uuid` retourné |
| **204** | No Content (DELETE, PUT partiels) | Pas de payload, fin d'op |
| **400** | Bad Request (JSON invalide) | Voir `details` dans la réponse |
| **401** | Unauthorized (token absent/expiré) | Re-générer token via AUTH-001 |
| **403** | Forbidden (ACL insuffisante) | Policy utilisateur trop restrictive (cf. WAZO_AUTH_PITFALLS §5) |
| **404** | Not Found (UUID incorrect OU endpoint inexistant OU mauvaise auth) | Vérifier URL **et** méthode **et** ACL |
| **409** | Conflict (doublon sur champ unique) | Renommer (ex : `exten` doit être unique par contexte) |
| **422** | Unprocessable Entity (validation sémantique) | Lire `details` ; voir pièges ci-dessous |

### 10.2 Pièges Wazo 26.06 — top 5

| # | Piège | Symptôme | Fix |
|---|-------|----------|-----|
| 1 | `POST /token` avec JSON body `{username, password}` | 201 + token `null` ou 400 | Utiliser **Basic Auth** sur le header `Authorization` |
| 2 | `caller_id: {"display_name": "Alice"}` sur `POST /users` | 400 "Input Error - caller_id" | Utiliser **`caller_id_name`** + **`caller_id_number`** (champs plats) |
| 3 | IVR sans `menu_sound` | 422 `menu_sound: Missing data` | Ajouter `"menu_sound": ""` (chaîne vide acceptable) |
| 4 | Schedule sans `closed_destination` | 422 `closed_destination: Missing data` | Ajouter `"closed_destination": {"type": "none"}` |
| 5 | Webhook avec `config.timeout` | 400 "Unknown field" ou timeout ignoré | **Retirer `timeout` du config** (non supporté 26.06) |

### 10.3 401 sur calld/confd/call-logd — `missing_permission_or_invalid_tenant`

Cas typique : la **policy `wazo-calld-internal`** standard n'inclut **PAS** `calld.calls.read`. Lire `details.required_access` dans le body — créer une **policy custom** avec les ACLs listés dans `references/WAZO_AUTH_PITFALLS.md §5`.

### 10.4 404 sur ARI même avec auth Basic OK

Référer à `WAZODOCS/CH9_ARI.md §9.10` (procédure de diagnostic). Cause fréquente : **[general] dupliqué dans `ari.d/`** (pitfall #22).

---

## 11. TODO variables et limitations connues

### 11.1 Variables non encore couvertes par la collection

| Variable | Statut |
|----------|--------|
| `application_uuid` | Ajouté mais usage limité au ARI ; vérifier sur serveur |
| `endpoint_id` | Idem |
| `ext_id` | Idem (alias de `extension_id`) |
| `group_id` | Idem (alias de `group_uuid` dans certains cookbooks) |
| `iax_id` | Idem |
| `participant_id` | Utilisé par CALLD-019 |
| `token` | Alias de `wazo_token` pour compatibilité bearer |

### 11.2 Incohérences cookbooks ↔ Bible documentées

Voir l'inventaire des cas d'usage (étape 2 dans le livrable original) :

| # | Incohérence | Réconciliation retenue |
|---|-------------|-------------------------|
| I1 | CK5 §5.4 `auth_id` n'existe pas dans BIBLE | Hypothèse : `auth_id` = `user_uuid` |
| I2 | CK1 §1.1 utilise `username` dans body | Conservé (CK1 explicite) |
| I3 | CK1 §1.8 référence des endpoints funckeys non documentés | Conservés (cohérents REST confd) |
| I4 | CK4 §4.4 conférences/{uuid}/recordings | Conservé (Wazo S3) |
| I5 | CK4 §4.5 GET avec body sur conférences/{id}/join | TODO : vérifier PUT/POST sur serveur live |
| I6 | CK3 §3.1 deux endpoints agents/skills distincts | Conservés (cf. récap CK3 §3.12) |

### 11.3 Endpoints non testés en live

| Endpoint | Source | Statut |
|----------|--------|--------|
| `CALLD-017 GET /conferences/{id}/join` | CK4 §4.5 | ⚠ GET avec body inhabituel — peut être POST |
| `CALLD-019 PUT /conferences/{id}/participants/{participant_id}` | CK4 §4.5 | ⚠ PATCH vs PUT à valider |
| `AUTH-014 POST /users/{uuid}/policies` | CK1 §1.6 | ⚠ Variante non standard vs AUTH-012 PUT |
| `refresh_token` field | CK5 §5.4 | ⚠ Présent dans TokenRequestSchema mais non documenté en BIBLE |

---

## 12. Maintenance et bonnes pratiques

### 12.1 Stratégie de rotation des tokens

Pour usage interactif (ex : démo client) :
- Régénérer via `POST /token` à chaque session Postman.
- Stocker le token dans l'environment (pas dans un fichier partagé).

Pour usage CI/CD / automatisé :
- Utiliser **CK5 §5.4** (refresh token).
- Stocker `basic_auth_admin` dans un secret manager (HashiCorp Vault, Bitwarden, …), **jamais** dans le JSON Postman partagé.

### 12.2 Maintenance de la collection

Si vous régénérez la collection :
1. Ouvrir `references/API_BIBLE_INDEX.md` pour vérifier que les fichiers BIBLE+COOKBOOK sont à jour.
2. Régénérer avec le script `/tmp/build_wazo.py` + `build_wazo_batch{2,3,4,5}.py` (joints à cette doc).
3. Re-vérifier : `python3 -c "import json; json.load(open('WAZO.postman_collection.json'))"`.

### 12.3 Partage de la collection

```bash
# Validation rapide avant partage
jq -r '.item[].name' WAZO.postman_collection.json  # 22 noms de dossiers

# Vérification qu'aucun credential n'est en clair
grep -c "TODO_AFTER_LOGIN\|TODO_BASE64" WAZO.postman_collection.json
# Attendu: ~5 occurrences (toutes avec valeur TODO, jamais de credentials réels)
```

---

## Annexe A — Index des 60 variables de collection

| Catégorie | Variables |
|-----------|-----------|
| **Connexion** | `base_url`, `wazo_token`, `tenant_uuid`, `basic_auth_admin`, `basic_auth_ari` |
| **Auth utilisateurs** | `user_uuid`, `auth_user_uuid`, `policy_uuid`, `permission_id`, `backend` |
| **Confd base** | `context_id`, `endpoint_uuid`, `endpoint_id`, `line_id`, `extension_id`, `ext_id` |
| **Confd trunk/outcall/incall** | `trunk_id`, `transport_uuid`, `registration_id`, `outcall_id`, `incall_id` |
| **Confd call center** | `queue_id`, `agent_id`, `skill_id`, `skill_rule_id`, `group_uuid`, `group_id` |
| **Confd services** | `filter_id`, `pickup_id`, `parking_id`, `paging_id`, `paging_extension`, `switchboard_id`, `moh_uuid`, `filename`, `sound_name`, `template_id`, `template_uuid`, `position`, `forward_type` |
| **Confd conferences** | `conference_id`, `oper_id` |
| **Confd voicemails** | `voicemail_id` |
| **Calld** | `call_id`, `transfer_id`, `cdr_id`, `recording_id` |
| **ARI** | `application_name`, `channel_id`, `bridge_id`, `playback_id` |
| **Chat/Presence** | `conv_uuid` |

---

## Annexe B — Index des 308 requêtes par dossier

| # | Dossier | Requêtes |
|---|---------|----------|
| 1 | Wazo Auth | 33 |
| 2 | Wazo Confd — Contextes & Trunks | 13 |
| 3 | Wazo Confd — Endpoints SIP/IAX/Users/Lines/Extensions | 24 |
| 4 | Wazo Confd — Voicemail & Call Permissions | 8 |
| 5 | Wazo Confd — Outcalls & Incalls | 9 |
| 6 | Wazo Confd — Queues, Groups, IVR, Conferences, Schedules, Agents | 29 |
| 7 | Wazo Confd — Filters, Pickups, Paging, Parkings, MOH, Sounds, Funckeys | 24 |
| 8 | Wazo Confd — Services Utilisateurs (DND/Forwards/Fallbacks/Incallfilter) | 10 |
| 9 | Wazo Confd — Asterisk Global & Users Import/Export | 6 |
| 10 | Wazo Calld | 35 |
| 11 | Wazo Call-logd (CDR) | 2 |
| 12 | Wazo Webhookd | 7 |
| 13 | Wazo WebSocketd (temps réel) | 1 |
| 14 | Wazo Provd | 16 |
| 15 | Wazo Chatd | 4 |
| 16 | Wazo Presenced | 4 |
| 17 | Asterisk ARI (port 5039 — HTTP Basic Auth) | 27 |
| 18 | Scenarios Cookbook PART 1 — Provisioning & Utilisateurs | 13 |
| 19 | Scenarios Cookbook PART 2 — Terminaux, Trunks, Routage | 9 |
| 20 | Scenarios Cookbook PART 3 — Call Center & Services | 11 |
| 21 | Scenarios Cookbook PART 4 — Temps Réel, WebRTC, Conférence | 15 |
| 22 | Scenarios Cookbook PART 5 — Sécurité, Auth & Intégrations | 8 |
| | **TOTAL** | **308** |

### Répartition par méthode HTTP

| Méthode | Nombre |
|---------|--------|
| `GET` | 78 |
| `POST` | 123 |
| `PUT` | 87 |
| `DELETE` | 16 |
| `WS` (WebSocket) | 4 |

### Répartition par micro-service

| Service | Requêtes |
|---------|----------|
| Confd (tous sous-dossiers) | 123 |
| Cookbook (cross-service) | 56 |
| Calld | 35 |
| Auth | 33 |
| ARI | 27 |
| Provd | 16 |
| Webhookd | 7 |
| Chatd | 4 |
| Presenced | 4 |
| Call-logd | 2 |
| WebSocketd | 1 |

---

## Annexe C — Sources et références croisées

| Fichier source | Chapitres couverts |
|----------------|---------------------|
| `WAZODOCS/WAZO_API_BIBLE_CH1_CH2.md` | Architecture, multi-tenant, Auth |
| `WAZODOCS/WAZO_API_BIBLE_CH3_CH4.md` | confd (users, lines, ext, trunks, outcalls, incalls) |
| `WAZODOCS/WAZO_API_BIBLE_CH5_CH6.md` | confd (queues, IVR, MOH, sched) + provd (devices, plugins) |
| `WAZODOCS/WAZO_API_BIBLE_CH7.md` | calld + WebSocket + DND/forwards |
| `WAZODOCS/WAZO_API_BIBLE_CH8.md` | call-logd + webhooks + patterns |
| `WAZODOCS/WAZO_API_BIBLE_CH9_ARI.md` | ARI + 22 pièges vérifiés en prod |
| `WAZODOCS/WAZO_COOKBOOK_PART1.md` | Provisioning complet, suppression, CSV, forwards, multi-tenant |
| `WAZODOCS/WAZO_COOKBOOK_PART2.md` | Yealink, trunks SIP/IAX, DID, DISA, RTP, MOH |
| `WAZODOCS/WAZO_COOKBOOK_PART3.md` | ACD, skills, ring groups, callfilter, IVR, conference, paging, switchboard |
| `WAZODOCS/WAZO_COOKBOOK_PART4.md` | Transferts, WebRTC, chat, presence, DND, CDR, recording, park, spy, ARI |
| `WAZODOCS/WAZO_COOKBOOK_PART5.md` | Sécurité, LDAP, SAML, Google, ACLs, webhooks, ARI Stasis |

Pour les **pièges vérifiés en prod** (auth Wazo 26.06, ARI), voir aussi la skill `wazo-api-bible` (Homeric Agent) et son fichier `references/WAZO_AUTH_PITFALLS.md`.

---

## Licence & provenance

Documentation générée à partir de sources internes Wazo Platform (CH1–CH9) publiées par l'équipe Wazo. Cookbook reconstitué à partir de notes de déploiement internes (juillet 2026). Usage libre pour intégration, audit et recherche en sécurité.
