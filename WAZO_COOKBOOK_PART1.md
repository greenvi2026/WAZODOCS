---

# PARTIE 1 : Provisioning Core & Utilisateurs

Cette partie couvre les workflows fondamentaux de gestion des utilisateurs et du provisioning de base. Ces scénarios sont les plus fréquents et constituent le socle de toute intégration Wazo.

---

## 1.1 Création d'un Utilisateur Complet (8 Étapes)

### Objectif

Créer un utilisateur téléphonique complet avec tous les composants nécessaires : compte d'accès, ligne SIP, extension, boîte vocale, et les liaisons entre tous ces éléments. C'est le scénario le plus courant pour le provisioning de nouveaux employés.

### Services impliqués

- **wazo-confd** : Gestion des utilisateurs, lignes, extensions, endpoints SIP, voicemails
- **wazo-auth** : Gestion des comptes d'authentification

### Le Workflow détaillé

#### Étape 1 : Créer le compte utilisateur dans confd

```http
POST /api/confd/1.1/users
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "firstname": "Jean",
  "lastname": "Dupont",
  "email": "jean.dupont@acme.fr",
  "username": "jdupont",
  "caller_id_name": "Jean Dupont",
  "caller_id_number": "1001"
}
```

**Réponse :**
```json
{
  "uuid": "a1223fe6-bff8-4fb6-a982-f9157dea5094",
  "firstname": "Jean",
  "lastname": "Dupont",
  "email": "jean.dupont@acme.fr",
  "username": "jdupont",
  ...
}
```

> **🔗 Chaînage** : Récupérez le champ `uuid` — il sera utilisé dans toutes les étapes suivantes pour lier les ressources. Stockez-le dans **USER_UUID**.

#### Étape 2 : Créer la boîte vocale (optionnel mais recommandé)

```http
POST /api/confd/1.1/voicemails
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "jdupont",
  "number": "1001",
  "email": "jean.dupont@acme.fr",
  "timezone": "Europe/Paris",
  "password": "1234",
  "max_messages": 50
}
```

**Réponse :**
```json
{
  "id": 12,
  "name": "jdupont",
  "number": "1001",
  ...
}
```

> **🔗 Chaînage** : Récupérez le champ `id` — il sera utilisé pour lier la boîte vocale à l'utilisateur. Stockez-le dans **VM_ID**.

#### Étape 3 : Créer l'endpoint SIP technique

```http
POST /api/confd/1.1/endpoints/sip
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "label": "jdupont-sip",
  "name": "jdupont",
  "auth_section_options": [
    ["username", "jdupont"],
    ["password", "secure_password_sip"]
  ],
  "endpoint_section_options": [
    ["disallow", "all"],
    ["allow", "ulaw,alaw,g722"],
    ["direct_media", "no"],
    ["rtp_symmetric", "yes"]
  ]
}
```

**Réponse :**
```json
{
  "uuid": "b2345gh7-abc9-4def-ghij-klmnopqr6789",
  "label": "jdupont-sip",
  "name": "jdupont",
  ...
}
```

> **🔗 Chaînage** : Récupérez le champ `uuid` — il sera utilisé pour lier l'endpoint SIP à la ligne. Stockez-le dans **SIP_UUID**.

#### Étape 4 : Créer la ligne téléphonique

```http
POST /api/confd/1.1/lines
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "jdupont-line",
  "context": "default",
  "caller_id_name": "Jean Dupont",
  "caller_id_number": "1001"
}
```

**Réponse :**
```json
{
  "id": 25,
  "name": "jdupont-line",
  "context": "default",
  ...
}
```

> **🔗 Chaînage** : Récupérez le champ `id` — il sera utilisé pour lier l'extension, l'endpoint SIP et l'utilisateur. Stockez-le dans **LINE_ID**.

#### Étape 5 : Créer l'extension

```http
POST /api/confd/1.1/extensions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "exten": "1001",
  "context": "default"
}
```

**Réponse :**
```json
{
  "id": 156,
  "exten": "1001",
  "context": "default"
}
```

> **🔗 Chaînage** : Récupérez le champ `id` — il sera utilisé pour lier l'extension à la ligne. Stockez-le dans **EXT_ID**.

#### Étape 6 : Lier l'extension à la ligne

```http
PUT /api/confd/1.1/lines/{LINE_ID}/extensions/{EXT_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :** 204 No Content

#### Étape 7 : Lier l'endpoint SIP à la ligne

```http
PUT /api/confd/1.1/lines/{LINE_ID}/endpoints/sip/{SIP_UUID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :** 204 No Content

#### Étape 8 : Lier la ligne à l'utilisateur

```http
PUT /api/confd/1.1/users/{USER_UUID}/lines/{LINE_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :** 204 No Content

#### Étape 9 (optionnel) : Lier la boîte vocale à l'utilisateur

```http
PUT /api/confd/1.1/users/{USER_UUID}/voicemails/{VM_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :** 204 No Content

#### Étape 10 : Créer le compte d'authentification

```http
POST /api/auth/0.1/users
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "username": "jdupont",
  "password": "initial_password",
  "firstname": "Jean",
  "lastname": "Dupont",
  "email": "jean.dupont@acme.fr"
}
```

**Réponse :**
```json
{
  "uuid": "c3456ij8-def0-4abc-lmno-pqrstu901234",
  "username": "jdupont",
  ...
}
```

> **🔗 Chaînage** : Récupérez le champ `uuid` — il correspond au compte d'authentification. Stockez-le dans **AUTH_USER_UUID**.

#### Étape 11 : Lier le compte auth à l'utilisateur confd

```http
PUT /api/confd/1.1/users/{USER_UUID}
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "auth_user_uuid": "c3456ij8-def0-4abc-lmno-pqrstu901234"
}
```

**Réponse :** 200 OK

### Point d'attention / Warning

> **⚠️ Important** :
> - L'ordre des étapes est STRICT — l'extension doit être créée AVANT d'être liée à la ligne
> - Les passwords SIP doivent être sécurisés (minimum 12 caractères, complexité)
> - Le `context` doit exister dans Wazo (créez-le via `/api/confd/1.1/contexts` si nécessaire)
> - La boîte vocale est optionnelle mais recommandée pour un utilisateur complet

---

## 1.2 Suppression Propre d'un Utilisateur (8 Étapes)

### Objectif

Supprimer un utilisateur et toutes ses ressources associées de manière propre et ordonnée, sans laisser d'orphelins dans la base de données. L'ordre de suppression est critique pour éviter les erreurs de contrainte.

### Services impliqués

- **wazo-confd** : Tous les composants de configuration
- **wazo-provd** : Gestion des devices

### Le Workflow détaillé

#### Étape 1 : Récupérer les lignes de l'utilisateur

```http
GET /api/confd/1.1/users/{USER_UUID}/lines
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :**
```json
{
  "items": [
    {
      "id": 25,
      "name": "jdupont-line",
      ...
    }
  ]
}
```

> **🔗 Chaînage** : Stockez le **LINE_ID** = 25

#### Étape 2 : Récupérer les devices associés à la ligne

```http
GET /api/confd/1.1/lines/{LINE_ID}/devices
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :**
```json
{
  "items": [
    {
      "id": "001122334455",
      "mac": "001122334455",
      "model": "Yealink T46S",
      ...
    }
  ]
}
```

> **🔗 Chaînage** : Stockez le **DEVICE_ID** = 001122334455

#### Étape 3 : Dissocier le device de la ligne

```http
DELETE /api/confd/1.1/lines/{LINE_ID}/devices/{DEVICE_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :** 204 No Content

#### Étape 4 : Réinitialiser le device en mode autoprov

```http
POST /api/provd/0.1/devices/{DEVICE_ID}/autoprov
X-Auth-Token: {admin_token}
```

**Réponse :** 200 OK

> Cette étape permet au téléphone de se réapprovisionner automatiquement lors du prochain redémarrage.

#### Étape 5 : Récupérer les extensions de la ligne

```http
GET /api/confd/1.1/lines/{LINE_ID}/extensions
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :**
```json
{
  "items": [
    {
      "id": 156,
      "exten": "1001",
      "context": "default"
    }
  ]
}
```

> **🔗 Chaphinage** : Stockez **EXT_ID** = 156

#### Étape 6 : Dissocier l'extension de la ligne

```http
DELETE /api/confd/1.1/lines/{LINE_ID}/extensions/{EXT_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :** 204 No Content

#### Étape 7 : Dissocier la ligne de l'utilisateur

```http
DELETE /api/confd/1.1/users/{USER_UUID}/lines/{LINE_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :** 204 No Content

#### Étape 8 : Supprimer l'extension

```http
DELETE /api/confd/1.1/extensions/{EXT_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :** 204 No Content

#### Étape 9 : Supprimer la ligne

```http
DELETE /api/confd/1.1/lines/{LINE_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :** 204 No Content

#### Étape 10 : Supprimer l'endpoint SIP

```http
DELETE /api/confd/1.1/endpoints/sip/{SIP_UUID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :** 204 No Content

#### Étape 11 : Supprimer l'utilisateur

```http
DELETE /api/confd/1.1/users/{USER_UUID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :** 204 No Content

### Point d'attention / Warning

> **⚠️ Important** :
> - L'ORDRE EST CRITIQUE : supprimez toujours dans l'ordre inverse de la création
> - Ne supprimez jamais un endpoint SIP utilisé par d'autres lignes
> - Le device doit être dissocié AVANT de supprimer la ligne
> - Vérifiez qu'aucun trunk ou queue n'utilise ces ressources avant suppression

---

## 1.3 Importation CSV en Masse d'Utilisateurs (4 Étapes)

### Objectif

Importer rapidement des dizaines ou centaines d'utilisateurs simultanément via un fichier CSV, en utilisant l'import automatique de Wazo qui crée tous les éléments en une seule opération.

### Services impliqués

- **wazo-confd** : Import des utilisateurs via endpoint spécialisé

### Le Workflow détaillé

#### Étape 1 : Récupérer le template CSV

```http
GET /api/confd/1.1/users/export
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
Accept: text/csv
```

**Réponse (CSV) :**
```csv
firstname,lastname,email,username,extension,context,line_name,voicemail_number
Jean,Dupont,jd@acme.fr,jdupont,1001,default,jd-line,1001
Marie,Martin,mm@acme.fr,mmartin,1002,default,mm-line,1002
```

> **🔗 Chaînage** : Ce template vous montre les colonnes attendues. Préparez votre fichier CSV en suivant ce format.

#### Étape 2 : Importer le fichier CSV

```http
POST /api/confd/1.1/users/import
Content-Type: text/csv
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Body (CSV) :**
```csv
firstname,lastname,email,username,extension,context,line_name,voicemail_number
Jean,Dupont,jd@acme.fr,jdupont,1001,default,jd-line,1001
Marie,Martin,mm@acme.fr,mmartin,1002,default,mm-line,1002
Pierre,Durand,pd@acme.fr,pdurand,1003,default,pd-line,1003
```

**Réponse :**
```json
{
  "created_users": [
    {"uuid": "user-uuid-1", "username": "jdupont"},
    {"uuid": "user-uuid-2", "username": "mmartin"},
    {"uuid": "user-uuid-3", "username": "pdurand"}
  ],
  "errors": []
}
```

> **🔗 Chaînage** : La réponse contient les UUIDs créés. Stockez-les pour les utiliser dans les étapes suivantes si besoin.

#### Étape 3 : Vérifier les créations

```http
GET /api/confd/1.1/users?search=Dupont
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :**
```json
{
  "items": [
    {
      "uuid": "user-uuid-1",
      "firstname": "Jean",
      "lastname": "Dupont",
      ...
    }
  ]
}
```

#### Étape 4 : Associer les lignes aux utilisateurs (si nécessaire)

```http
PUT /api/confd/1.1/users/{USER_UUID}/lines/{LINE_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - L'import CSV crée automatiquement les lignes et extensions correspondantes
> - Les voicemails ne sont PAS créés automatiquement — faites-le séparément si besoin
> - En cas d'erreur sur une ligne, les autres lignes du fichier sont quand même créées
> - Vérifiez toujours le champ `errors` dans la réponse

---

## 1.4 Renvois d'Appel Utilisateur (4 Étapes)

### Objectif

Configurer les renvois d'appel (forwards) pour un utilisateur : inconditionnel, sur occupation, et sur non-réponse. Ces services permettent la continuité des communications en cas d'absence.

### Services impliqués

- **wazo-confd** : Configuration des services de renvoi

### Le Workflow détaillé

#### Étape 1 : Lister les renvois actuels

```http
GET /api/confd/1.1/users/{USER_UUID}/forwards
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :**
```json
{
  "items": [
    {
      "type": "unconditional",
      "enabled": false,
      "destination": null
    },
    {
      "type": "busy",
      "enabled": false,
      "destination": null
    },
    {
      "type": "noanswer",
      "enabled": false,
      "destination": null
    }
  ]
}
```

#### Étape 2 : Configurer le renvoi inconditionnel

```http
PUT /api/confd/1.1/users/{USER_UUID}/forwards/unconditional
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "enabled": true,
  "destination": "1005"
}
```

**Réponse :**
```json
{
  "enabled": true,
  "destination": "1005"
}
```

> **🔗 Chaînage** : Le numéro de destination peut être une extension interne ou un numéro externe

#### Étape 3 : Configurer le renvoi sur occupation

```http
PUT /api/confd/1.1/users/{USER_UUID}/forwards/busy
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "enabled": true,
  "destination": "2001"
}
```

#### Étape 4 : Configurer le renvoi sur non-réponse

```http
PUT /api/confd/1.1/users/{USER_UUID}/forwards/noanswer
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "enabled": true,
  "destination": "3000",
  "timeout": 18
}
```

| Paramètre | Description |
|-----------|-------------|
| `timeout` | Durée avant renvoi (en secondes, défaut: 18) |

### Point d'attention / Warning

> **⚠️ Important** :
> - Le renvoi inconditionnel est prioritaire sur tous les autres
> - Le timeout du renvoi sur non-réponse doit être inférieur au timeout de la ligne
> - Les destinations externes nécessitent les droits d'appels sortants appropriés

---

## 1.5 Services Utilisateur : DND et Filtre d'Appel (4 Étapes)

### Objectif

Activer le mode "Ne Pas Déranger" (DND) et le filtre d'appel entrant pour un utilisateur. Le DND bloque tous les appels entrants ; le filtre permet de筛选 les appels selon certaines règles.

### Services impliqués

- **wazo-confd** : Services DND et incallfilter

### Le Workflow détaillé

#### Étape 1 : Activer le DND

```http
PUT /api/confd/1.1/users/{USER_UUID}/services/dnd/enable
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :**
```json
{
  "enabled": true
}
```

#### Étape 2 : Désactiver le DND

```http
PUT /api/confd/1.1/users/{USER_UUID}/services/dnd/disable
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

#### Étape 3 : Activer le filtre d'appel entrant

```http
PUT /api/confd/1.1/users/{USER_UUID}/services/incallfilter/enable
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :**
```json
{
  "enabled": true
}
```

#### Étape 4 : Désactiver le filtre d'appel entrant

```http
PUT /api/confd/1.1/users/{USER_UUID}/services/incallfilter/disable
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Depuis XiVO 16.13, le DND est effectif indépendamment de l'extension *25
> - Le filtre d'appel nécessite une configuration supplémentaire des règles de filtrage

---

## 1.6 Création d'un Nouveau Tenant (Multi-Tenant) (7 Étapes)

### Objectif

Créer un nouveau tenant isolé pour un client ou un département, avec ses propres contextes, utilisateurs et politiques d'accès. Le multi-tenant permet une isolation complète des données.

### Services impliqués

- **wazo-auth** : Gestion des tenants et utilisateurs d'authentification
- **wazo-confd** : Gestion des contextes

### Le Workflow détaillé

#### Étape 1 : Créer le tenant

```http
POST /api/auth/0.1/tenants
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "name": "ACME Corp",
  "slug": "acme"
}
```

**Réponse :**
```json
{
  "uuid": "tenant-uuid-acme123",
  "name": "ACME Corp",
  "slug": "acme",
  "parent_uuid": "master-tenant-uuid"
}
```

> **🔗 Chaînage** : Récupérez le champ `uuid` — il sera utilisé pour toutes les opérations sur ce tenant. Stockez-le dans **TENANT_UUID**.

#### Étape 2 : Créer l'utilisateur administrateur du tenant

```http
POST /api/auth/0.1/users
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "username": "admin_acme",
  "password": "secure_password",
  "firstname": "Admin",
  "lastname": "ACME"
}
```

**Réponse :**
```json
{
  "uuid": "admin-auth-uuid-456",
  "username": "admin_acme",
  ...
}
```

> **🔗 Chaînage** : Stockez **ADMIN_AUTH_UUID**

#### Étape 3 : Créer une policy d'administration

```http
POST /api/auth/0.1/policies
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "admin-policy",
  "description": "Full admin access for ACME tenant",
  "acl": [
    "confd.#",
    "calld.#",
    "provd.#"
  ]
}
```

**Réponse :**
```json
{
  "uuid": "policy-uuid-789",
  "name": "admin-policy",
  ...
}
```

> **🔗 Chaînage** : Stockez **POLICY_UUID**

#### Étape 4 : Assigner la policy à l'administrateur

```http
POST /api/auth/0.1/users/{ADMIN_AUTH_UUID}/policies
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "policy_uuid": "policy-uuid-789"
}
```

#### Étape 5 : Créer le contexte interne pour le tenant

```http
POST /api/confd/1.1/contexts
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "label": "interne-acme",
  "name": "Interne ACME",
  "type": "internal",
  "user_ranges": [
    {"start": "1000", "end": "1999"}
  ]
}
```

**Réponse :**
```json
{
  "id": 45,
  "label": "interne-acme",
  "type": "internal",
  ...
}
```

> **🔗 Chaînage** : Stockez **CTX_INTERNAL_ID** = 45

#### Étape 6 : Créer le contexte entrant pour le tenant

```http
POST /api/confd/1.1/contexts
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "label": "entrant-acme",
  "name": "Entrant ACME",
  "type": "incall",
  "incall_ranges": [
    {"start": "003338000100", "end": "003338000200"}
  ]
}
```

**Réponse :**
```json
{
  "id": 46,
  "label": "entrant-acme",
  "type": "incall",
  ...
}
```

> **🔗 Chaînage** : Stockez **CTX_INCALL_ID** = 46

#### Étape 7 : Vérifier le tenant

```http
GET /api/auth/0.1/tenants/{TENANT_UUID}
X-Auth-Token: {admin_token}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Le header `Wazo-Tenant` est OBLIGATOIRE pour toutes les opérations après la création du tenant
> - L'ACL `confd.#` donne accès à toutes les ressources confd du tenant
> - Les contextes créés n'ont pas de relation automatique — créez des liens explicites si nécessaire
> - La suppression d'un tenant est IRRÉVERSIBLE

---

## 1.7 Fallbacks et Options Utilisateur (4 Étapes)

### Objectif

Configurer les fallbacks (renvois en cas d'indisponibilité) et les options avancées d'un utilisateur : timeout, destination si pas de réponse, boîte vocale, etc.

### Services impliqués

- **wazo-confd** : Configuration des fallbacks utilisateur

### Le Workflow détaillé

#### Étape 1 : Récupérer les fallbacks actuels

```http
GET /api/confd/1.1/users/{USER_UUID}/fallbacks
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :**
```json
{
  "noanswer_destination": null,
  "busy_destination": null,
  "congestion_destination": null,
  "fail_destination": null,
  "noanswer_timeout": 18
}
```

#### Étape 2 : Configurer le fallback sur non-réponse

```http
PUT /api/confd/1.1/users/{USER_UUID}/fallbacks
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "noanswer_destination": {
    "type": "voicemail",
    "voicemail_id": 12
  },
  "noanswer_timeout": 25
}
```

#### Étape 3 : Configurer le fallback sur occupation

```http
PUT /api/confd/1.1/users/{USER_UUID}/fallbacks
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "busy_destination": {
    "type": "voicemail",
    "voicemail_id": 12
  }
}
```

#### Étape 4 : Configurer le fallback sur indisponibilité (fail)

```http
PUT /api/confd/1.1/users/{USER_UUID}/fallbacks
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "fail_destination": {
    "type": "extension",
    "extension": "1000",
    "context": "default"
  }
}
```

| Type de destination | Paramètres requis |
|---------------------|-------------------|
| `voicemail` | `voicemail_id` |
| `extension` | `extension`, `context` |
| `user` | `user_id` |
| `custom` | `content` (dialplan) |

### Point d'attention / Warning

> **⚠️ Important** :
> - Le `noanswer_timeout` doit être cohérent avec le timeout de sonnerie du téléphone
> - Les fallbacks sont évalués dans l'ordre : busy → noanswer → congestion → fail
> - Configurez toujours une destination de dernier recours (fallback final)

---

## 1.8 Gestion des Funckeys (Touches de Fonction) (5 Étapes)

### Objectif

Configurer les touches de fonction (BLF, speed dial, pickup) sur les телефонов prenant en charge les touches programmable. Ces touches permettent un accès rapide aux fonctions fréquentes.

### Services impliqués

- **wazo-confd** : Configuration des funckeys

### Le Workflow détaillé

#### Étape 1 : Lister les destinations disponibles

```http
GET /api/confd/1.1/funckeys/destinations
X-Auth-Token: {admin_token}
```

**Réponse :**
```json
{
  "items": [
    {"type": "user", "description": "Appeler un utilisateur"},
    {"type": "queue", "description": "Appeler une file d'attente"},
    {"type": "custom", "description": "Extension personnalisée"},
    {"type": "transfer", "description": "Transférer l'appel"},
    {"type": "bsfilter", "description": "Filtre boss-secrétaire"}
  ]
}
```

#### Étape 2 : Créer un template de funckeys

```http
POST /api/confd/1.1/funckeys/templates
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "standard-template",
  "keys": {
    "1": {
      "destination_type": "user",
      "user_id": "user-uuid-1"
    },
    "2": {
      "destination_type": "queue",
      "queue_id": 10
    },
    "3": {
      "destination_type": "custom",
      "extension": "*25",
      "label": "DND"
    }
  }
}
```

**Réponse :**
```json
{
  "id": 5,
  "name": "standard-template",
  "keys": {...}
}
```

> **🔗 Chaînage** : Stockez **TEMPLATE_ID** = 5

#### Étape 3 : Appliquer le template à un utilisateur

```http
PUT /api/confd/1.1/users/{USER_UUID}/funckeys/templates/{TEMPLATE_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

#### Étape 4 : Ajouter une funckey individuelle (override)

```http
PUT /api/confd/1.1/users/{USER_UUID}/funckeys/5
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "destination_type": "custom",
  "extension": "*26",
  "label": "Renvoi ON",
  "blf": true
}
```

#### Étape 5 : Récupérer les funckeys fusionnées

```http
GET /api/confd/1.1/users/{USER_UUID}/funckeys?view=merged
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Le paramètre `blf: true` permet la surveillance d'état (Busy Lamp Field)
> - Les funckeys individuelles surchargent le template
> - La suppression d'une funckey la retire complètement

---

## 1.9 Récapitulatif des Endpoints Utilisateur

| Ressource | CRUD | Endpoint |
|-----------|-----|----------|
| Utilisateur | C | POST /users |
| Utilisateur | R | GET /users/{uuid} |
| Utilisateur | U | PUT /users/{uuid} |
| Utilisateur | D | DELETE /users/{uuid} |
| Ligne | C | POST /lines |
| Extension | C | POST /extensions |
| Endpoint SIP | C | POST /endpoints/sip |
| Voicemail | C | POST /voicemails |
| Forward | U | PUT /users/{uuid}/forwards/{type} |
| DND | U | PUT /users/{uuid}/services/dnd/enable |
| Fallback | U | PUT /users/{uuid}/fallbacks |
| Funckey | C | PUT /users/{uuid}/funckeys/{position} |

---

*Fin de la PARTIE 1*
