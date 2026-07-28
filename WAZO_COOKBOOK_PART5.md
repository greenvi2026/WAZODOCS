---

# PARTIE 5 : Sécurité, Authentification et Intégrations

Cette partie couvre les workflows d'intégration liés à la sécurité, l'authentification avancée et les intégrations tierces avec Wazo. Ces scénarios sont essentiels pour déployer Wazo en environnement de production avec des exigences de sécurité strictes.

---

## 5.1 Rotation Sécurisée des Tokens API

### Objectif

Automatiser le renouvellement des tokens d'authentification pour maintenir des sessions longues sans interruption, tout en respectant les bonnes pratiques de sécurité (tokens à durée de vie limitée).

### Services impliqués

- **wazo-auth** : Gestion centrale de l'authentification et des tokens
- **wazo-confd** : API de configuration nécessitant une authentification

### Le Workflow détaillé

#### Étape 1 : Authentification initiale et obtention du token

```http
POST /api/auth/0.1/token
Content-Type: application/json
```

**Payload :**
```json
{
  "backend": "wazo_user",
  "expiration": 3600,
  "username": "admin",
  "password": "secure_password"
}
```

**Réponse :**
```json
{
  "token": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
  "expires_at": "2024-01-15T15:00:00Z",
  "auth_id": "admin_uuid"
}
```

> **🔗 Chaînage** : Récupérez le champ `token` — il sera utilisé comme `X-Auth-Token` dans tous les appels API suivants. Notez également `expires_at` pour planifier le renouvellement.

#### Étape 2 : Surveillance de l'expiration et renouvellement proactif

```http
POST /api/auth/0.1/token
Content-Type: application/json
```

**Payload :**
```json
{
  "backend": "wazo_user",
  "expiration": 3600,
  "username": "admin",
  "password": "secure_password"
}
```

> **🔗 Chaînage** : Générez un **nouveau token** avant l'expiration du précédent (idealement 5 minutes avant). Le nouveau token remplace complètement l'ancien.

#### Step 3 : Invalidation du token (logout)

```http
DELETE /api/auth/0.1/token/{token}
```

**Headers :**
```http
X-Auth-Token: {current_token}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Never hardcode credentials in source code. Use environment variables or a secrets manager.
> - Tokens with `expiration: 3600` (1 hour) are recommended for long-running scripts; shorter durations (300-600s) for higher security.
> - Always implement token caching to avoid authenticating on every API call.
> - The token refresh should be handled automatically by your client library or implemented with a background thread.

---

## 5.2 Intégration LDAP / Active Directory

### Objectif

Synchroniser automatiquement les utilisateurs depuis un annuaire LDAP (OpenLDAP ou Active Directory) vers Wazo, permettant une gestion centralisée des identités et une authentification unique.

### Services impliqués

- **wazo-auth** : Backend d'authentification LDAP
- **wazo-confd** : API de gestion des utilisateurs
- **LDAP/AD** : Annuaire externe source

### Le Workflow détaillé

#### Étape 1 : Configuration du backend LDAP dans wazo-auth

```http
PUT /api/auth/0.1/backends/ldap/config
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "host": "ldap://ldap.example.com",
  "port": 389,
  "bind_dn": "cn=admin,dc=example,dc=com",
  "bind_password": "ldap_admin_password",
  "user_base_dn": "ou=users,dc=example,dc=com",
  "user_filter": "(objectClass=person)",
  "user_attributes": {
    "uuid": "entryUUID",
    "email": "mail",
    "firstname": "givenName",
    "lastname": "sn",
    "username": "sAMAccountName"
  },
  "group_base_dn": "ou=groups,dc=example,dc=com",
  "group_filter": "(objectClass=groupOfNames)",
  "group_member_attribute": "member"
}
```

> **🔗 Chaînage** : Cette configuration établit la connexion LDAP. Aucun UUID à chaîner ici — c'est une configuration globale du service.

#### Étape 2 : Activer le backend LDAP

```http
POST /api/auth/0.1/backends/ldap
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "enabled": true
}
```

#### Étape 3 : Tester la connexion LDAP

```http
GET /api/auth/0.1/backends/ldap/status
X-Auth-Token: {admin_token}
```

**Réponse :**
```json
{
  "status": "ok",
  "ldap_status": "connected",
  "users_found": 150,
  "groups_found": 12
}
```

#### Étape 4 : Mapper les groupes LDAP vers les ACLs Wazo

```http
PUT /api/auth/0.1/backends/ldap/groups
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "mappings": [
    {
      "ldap_group": "cn=admins,ou=groups,dc=example,dc=com",
      "wazo_acl": ["confd.users.*", "confd.lines.*", "confd.extensions.*"]
    },
    {
      "ldap_group": "cn=agents,ou=groups,dc=example,dc=com",
      "wazo_acl": ["confd.users.me.read", "calld.calls.read"]
    }
  ]
}
```

> **🔗 Chaînage** : Les ACLs configurées ici déterminent les permissions des utilisateurs LDAP une fois connectés. Chaque utilisateur LDAP hérite des ACLs de son groupe.

#### Étape 5 : Première authentification LDAP (provisionnement automatique)

```http
POST /api/auth/0.1/token
Content-Type: application/json
```

**Payload :**
```json
{
  "backend": "ldap",
  "expiration": 3600,
  "username": "john.doe",
  "password": "user_ldap_password"
}
```

**Réponse :**
```json
{
  "token": "new_token_xyz",
  "expires_at": "2024-01-15T16:00:00Z",
  "auth_id": "ldap_user_uuid"
}
```

> **🔗 Chaînage** : Le premier login d'un utilisateur LDAP crée automatiquement un enregistrement utilisateur dans Wazo (provisionnement à la demande). L'UUID retourné (`auth_id`) correspond à l'utilisateur Wazo créé.

### Point d'attention / Warning

> **⚠️ Important** :
> - Ensure the LDAP bind account has read-only access to the LDAP directory.
> - User synchronization is on-demand (first login), not automatic/scheduled.
> - Password changes in LDAP are automatically reflected in Wazo on next login.
> - For Active Directory, use port 389 (LDAP) or 636 (LDAPS) with SSL/TLS.

---

## 5.3 Authentification SSO SAML 2.0 (Azure AD / Okta)

### Objectif

Implémenter une authentification unique (SSO) via SAML 2.0 permettant aux utilisateurs de se connecter à Wazo avec leurs identités Azure AD ou Okta, sans gestion de mots de passe locaux.

### Services impliqués

- **wazo-auth** : Service SAML IdP
- **wazo-confd** : API de configuration
- **Azure AD / Okta** : Identity Provider (IdP) externe

### Le Workflow détaillé

#### Étape 1 : Configuration du service SAML dans wazo-auth

```http
PUT /api/auth/0.1/backends/saml/config
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "entity_id": "https://wazo.example.com",
  "sso_url": "https://login.microsoftonline.com/{tenant_id}/saml2",
  "certificate": "-----BEGIN CERTIFICATE-----\n{MIIC...}\n-----END CERTIFICATE-----",
  "attribute_mapping": {
    "email": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
    "firstname": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname",
    "lastname": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname",
    "username": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name"
  },
  "autoprovisioning": true,
  "enabled": true
}
```

> **🔗 Chaînage** : Cette configuration établit Wazo en tant que Service Provider (SP). L'`entity_id` doit correspondre à la configuration dans Azure AD/Okta.

#### Étape 2 : Récupérer les métadonnées SP pour configuration IdP

```http
GET /api/auth/0.1/backends/saml/metadata
X-Auth-Token: {admin_token}
```

**Réponse :**
```xml
<EntityDescriptor xmlns="urn:oasis:names:tc:SAML:2.0:metadata" entityID="https://wazo.example.com">
  <SPSSODescriptor AuthnRequestsSigned="false" WantAssertionsSigned="true" protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
    <AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" Location="https://wazo.example.com/api/auth/0.1/backends/saml/callback" index="0"/>
  </SPSSODescriptor>
</EntityDescriptor>
```

> **🔗 Chaînage** : Copy this metadata to configure Azure AD/Okta as the Identity Provider. The `Location` URL is where SAML responses will be sent.

#### Étape 3 : Configurer Azure AD (dans le portail Azure)

1. **Enterprise Application** → New Application → "Create your own application"
2. **Single sign-on** → Select "SAML"
3. **Basic SAML Configuration** :
   - Identifier (Entity ID): `https://wazo.example.com`
   - Reply URL: `https://wazo.example.com/api/auth/0.1/backends/saml/callback`
   - Sign on URL: `https://wazo.example.com`
4. **SAML Certificates** : Download the SAML signing certificate

#### Étape 4 : Configurer les attributs utilisateur SAML (Azure AD)

Dans Azure AD, ensure these attributes are sent in the SAML token:

| Attribute | Value |
|-----------|-------|
| user.email | user.mail |
| user.firstname | user.givenName |
| user.lastname | user.surname |

#### Étape 5 : Activer le backend SAML

```http
POST /api/auth/0.1/backends/saml
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "enabled": true
}
```

#### Étape 6 : Authentification SAML (initiation par SP)

```http
GET /api/auth/0.1/backends/saml/login
```

**Réponse (302 Redirect) :**
```http
Location: https://login.microsoftonline.com/{tenant_id}/saml2?SAMLRequest=...
```

> **🔗 Chaînage** : Redirigez l'utilisateur vers cette URL. Après authentification sur Azure AD, il sera redirigé vers le callback Wazo avec le token SAML.

#### Étape 7 : Callback SAML et obtention du token Wazo

```http
POST /api/auth/0.1/backends/saml/callback
Content-Type: application/x-www-form-urlencoded

SAMLResponse=...
```

**Réponse :**
```json
{
  "token": "saml_generated_token_abc123",
  "expires_at": "2024-01-15T17:00:00Z",
  "auth_id": "saml_user_uuid"
}
```

> **🔗 Chaînage** : Le premier login SAML crée automatiquement l'utilisateur dans Wazo si `autoprovisioning: true`. L'`auth_id` retourné correspond à l'utilisateur Wazo créé.

### Point d'attention / Warning

> **⚠️ Important** :
> - Always use HTTPS in production for both Wazo and the IdP.
> - Keep the SAML certificate from Azure AD/Okta up to date (renew before expiration).
> - Test with a non-admin user first — admin accounts may have different attribute mappings.
> - The `autoprovisioning` option automatically creates users on first login; disable if you want manual approval.

---

## 5.4 Refresh Tokens et Sessions Longues

### Objectif

Permettre des sessions utilisateur persistantes avec renouvellement automatique des tokens, idéal pour des applications client qui nécessitent un accès prolongé sans reconnecter l'utilisateur.

### Services impliqués

- **wazo-auth** : Gestion des tokens et refresh tokens

### Le Workflow détaillé

#### Étape 1 : Demander un token avec refresh token

```http
POST /api/auth/0.1/token
Content-Type: application/json
```

**Payload :**
```json
{
  "backend": "wazo_user",
  "expiration": 3600,
  "refresh_expiration": 86400,
  "username": "admin",
  "password": "secure_password"
}
```

**Réponse :**
```json
{
  "token": "main_token_abc",
  "refresh_token": "refresh_token_xyz",
  "expires_at": "2024-01-15T16:00:00Z",
  "refresh_expires_at": "2024-01-16T16:00:00Z",
  "auth_id": "user_uuid_123"
}
```

> **🔗 Chaînage** : Récupérez les deux tokens :
> - `token` : Pour les appels API normaux (expiration courte)
> - `refresh_token` : Pour renouvellement (expiration longue)

#### Étape 2 : Utiliser le token principal pour les API

```http
GET /api/confd/1.1/users
X-Auth-Token: main_token_abc
```

#### Étape 3 : Renouveler le token avec le refresh token

```http
POST /api/auth/0.1/token
Content-Type: application/json
```

**Payload :**
```json
{
  "backend": "wazo_user",
  "refresh_token": "refresh_token_xyz",
  "expiration": 3600
}
```

**Réponse :**
```json
{
  "token": "new_main_token_def",
  "refresh_token": "new_refresh_token_ghi",
  "expires_at": "2024-01-15T18:00:00Z",
  "refresh_expires_at": "2024-01-16T18:00:00Z"
}
```

> **🔗 Chaînage** : Un nouveau couple token/refresh_token est généré. Les anciens tokens sont automatiquement invalidés. Continuez à utiliser le nouveau refresh_token.

#### Étape 4 : Révoquer le refresh token (logout)

```http
DELETE /api/auth/0.1/token/{refresh_token}
Content-Type: application/json
X-Auth-Token: main_token_abc
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Refresh tokens have a longer lifespan (default: 24 hours) than access tokens (default: 1 hour).
> - Store refresh tokens securely — they allow long-term access.
> - If a refresh token is compromised, revoke it immediately with DELETE.
> - Each refresh generates a new refresh token (rotation) for security.

---

## 5.5 External Auth (Google, Microsoft, Firebase FCM)

### Objectif

Permettre l'authentification via des providers externes (Google, Microsoft) ou la réception de notifications push via Firebase Cloud Messaging (FCM), étendant les méthodes de connexion au-delà des credentials locaux.

### Services impliqués

- **wazo-auth** : Gestion des backends d'authentification externe
- **wazo-confd** : API de configuration utilisateur
- **Google / Microsoft** : Providers OAuth2
- **Firebase** : Service de notifications push

### Le Workflow détaillé

#### Étape 1 : Configuration Google OAuth2

```http
PUT /api/auth/0.1/backends/google/config
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "client_id": "google_client_id.apps.googleusercontent.com",
  "client_secret": "google_client_secret",
  "redirect_uri": "https://wazo.example.com/api/auth/0.1/backends/google/callback"
}
```

#### Étape 2 : Activation du backend Google

```http
POST /api/auth/0.1/backends/google
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "enabled": true
}
```

#### Étape 3 : Initier l'authentification Google

```http
GET /api/auth/0.1/backends/google/login
```

**Réponse :**
```http
Location: https://accounts.google.com/o/oauth2/v2/auth?client_id=...&redirect_uri=...&response_type=code&scope=email%20profile
```

#### Étape 4 : Callback Google OAuth et obtention du token Wazo

```http
POST /api/auth/0.1/backends/google/callback
Content-Type: application/x-www-form-urlencoded

code=google_authorization_code
```

**Réponse :**
```json
{
  "token": "google_wazo_token",
  "expires_at": "2024-01-15T17:00:00Z",
  "auth_id": "google_user_uuid"
}
```

> **🔗 Chaînage** : Le premier login crée l'utilisateur Wazo. Lier le compte Google à un utilisateur existant (voir étape 5).

#### Étape 5 : Lier un compte externe à un utilisateur existant

```http
PUT /api/auth/0.1/users/{user_uuid}/external/{backend}
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "external_id": "google_user_id",
  "email": "user@gmail.com"
}
```

#### Configuration Firebase FCM (Notifications Push)

```http
PUT /api/auth/0.1/backends/fcm/config
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "server_key": "firebase_server_key"
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - External auth requires setting up OAuth2 credentials in the provider's developer console.
> - The `redirect_uri` must exactly match what's configured in Google/Microsoft.
> - External auth can be linked to existing users or create new ones (autoprovisioning).
> - Firebase FCM is primarily used for mobile push notifications in Wazo UC client.

---

## 5.6 ACLs et Permissions (Créer un Rôle Restrictif)

### Objectif

Créer un rôle personnalisé avec des permissions granulaires pour limiter l'accès API d'utilisateurs ou services, conformément au principe du moindre privilège.

### Services impliqués

- **wazo-auth** : Gestion des ACLs et policies

### Le Workflow détaillé

#### Étape 1 : Lister les ACLs disponibles

```http
GET /api/auth/0.1/acl
X-Auth-Token: {admin_token}
```

**Réponse :**
```json
{
  "items": [
    "confd.users.*",
    "confd.users.me.read",
    "confd.lines.*",
    "confd.lines.{line_id}.read",
    "calld.calls.*",
    "calld.users.me.calls.*",
    "websocketd",
    "provd.devices.*"
  ]
}
```

> **🔗 Chaînage** : Cette liste définit toutes les permissions disponibles. Notez les patterns avec `{variable}` — ils permettent un accès conditionnel.

#### Étape 2 : Créer une policy avec ACLs restrictives

```http
POST /api/auth/0.1/policies
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "name": "agent_readonly",
  "description": "Policy for call center agents - read only access",
  "acl": [
    "confd.users.me.read",
    "confd.users.me.lines.read",
    "confd.users.me.voicemails.read",
    "calld.users.me.calls.read",
    "calld.users.me.calls.create",
    "websocketd"
  ]
}
```

**Réponse :**
```json
{
  "uuid": "policy_uuid_abc123",
  "name": "agent_readonly",
  "description": "Policy for call center agents",
  "acl": ["confd.users.me.read", ...],
  "tenant_uuid": "tenant_xyz"
}
```

> **🔗 Chaînage** : Récupérez le `uuid` de la policy — il sera utilisé pour l'assigner à un utilisateur.

#### Étape 3 : Assigner la policy à un utilisateur

```http
POST /api/auth/0.1/users/{user_uuid}/policies
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "policy_uuid": "policy_uuid_abc123"
}
```

> **🔗 Chaînage** : L'utilisateur hérite immédiatement des permissions de la policy. Les ACLs sont combinées avec celles existantes.

#### Étape 4 : Vérifier les ACLs effectives d'un utilisateur

```http
GET /api/auth/0.1/users/{user_uuid}/acl
X-Auth-Token: {admin_token}
```

**Réponse :**
```json
{
  "acl": [
    "confd.users.me.read",
    "confd.users.me.lines.read",
    "calld.users.me.calls.read",
    "calld.users.me.calls.create",
    "websocketd"
  ],
  "read_only": false
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - ACLs are additive — an user gets the union of all their policy ACLs.
> - Use `{uuid}` patterns to restrict access to specific resources (e.g., `confd.lines.15.read`).
> - The `websocketd` ACL is required for real-time events over WebSocket.
> - Test policies with a non-admin account before deploying in production.

---

## 5.7 Création et Débogage d'un Webhook

### Objectif

Configurer un webhook pour recevoir des notifications HTTP automatiques lors d'événements Wazo (appels, utilisateurs, agents), permettant des intégrations temps réel avec des systèmes tiers (CRM, helpdesk, analytics).

### Services impliqués

- **wazo-webhookd** : Service de gestion des webhooks
- **wazo-bus** : Bus de messages événements

### Le Workflow détaillé

#### Étape 1 : Créer une subscription webhook

```http
POST /api/webhookd/1.0/subscriptions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "CRM Call Events",
  "events": ["call_created", "call_ended"],
  "service": "http",
  "config": {
    "url": "https://crm.example.com/api/wazo/calls",
    "method": "POST",
    "timeout": 30,
    "headers": {
      "X-API-Key": "crm_secret_key",
      "Content-Type": "application/json"
    }
  }
}
```

**Réponse :**
```json
{
  "id": "webhook_uuid_abc123",
  "name": "CRM Call Events",
  "events": ["call_created", "call_ended"],
  "service": "http",
  "config": {
    "url": "https://crm.example.com/api/wazo/calls",
    "method": "POST",
    "timeout": 30
  },
  "enabled": true,
  "owner_uuid": "admin_uuid",
  "tenant_uuid": "tenant_xyz"
}
```

> **🔗 Chaînage** : Récupérez le `id` — il permet de modifier ou supprimer le webhook ultérieurement.

#### Étape 2 : Tester le webhook manuellement

```http
POST /api/webhookd/1.0/subscriptions/{webhook_uuid_abc123}/test
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload (optionnel — événement de test) :**
```json
{
  "event": "call_created",
  "payload": {
    "call_id": "test_call_id",
    "caller_id_number": "1001"
  }
}
```

**Réponse :**
```json
{
  "status": "sent",
  "http_status": 200,
  "response_time_ms": 150
}
```

> **🔗 Chaînage** : Un status 200 indique que le serveur distant a accepté la requête.

#### Étape 3 : Lister les webhooks existants

```http
GET /api/webhookd/1.0/subscriptions
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :**
```json
{
  "items": [
    {
      "id": "webhook_uuid_abc123",
      "name": "CRM Call Events",
      "events": ["call_created", "call_ended"],
      "service": "http",
      "enabled": true
    }
  ]
}
```

#### Étape 4 : Vérifier les logs de delivery webhook

```http
GET /api/webhookd/1.0/subscriptions/{webhook_uuid_abc123}/logs
X-Auth-Token: {admin_token}
```

**Réponse :**
```json
{
  "items": [
    {
      "id": "log_uuid",
      "subscription_id": "webhook_uuid_abc123",
      "event": "call_created",
      "status": "delivered",
      "attempts": 1,
      "created_at": "2024-01-15T14:30:00Z",
      "http_code": 200,
      "response_body": "OK"
    },
    {
      "id": "log_uuid_2",
      "subscription_id": "webhook_uuid_abc123",
      "event": "call_created",
      "status": "failed",
      "attempts": 3,
      "created_at": "2024-01-15T14:35:00Z",
      "http_code": 500,
      "error": "Internal Server Error"
    }
  ]
}
```

> **🔗 Chaînage** : Les logs permettent de diagnostiquer les échecs. Notez le champ `attempts` — les webhooks sont automatiquement retentés jusqu'à 3 fois en cas d'échec.

#### Étape 5 : Désactiver ou supprimer un webhook

```http
# Désactiver temporairement
PUT /api/webhookd/1.0/subscriptions/{webhook_uuid_abc123}
Content-Type: application/json
X-Auth-Token: {admin_token}

{
  "enabled": false
}

# Supprimer définitivement
DELETE /api/webhookd/1.0/subscriptions/{webhook_uuid_abc123}
X-Auth-Token: {admin_token}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Webhooks are triggered asynchronously — there's no immediate delivery guarantee.
> - The receiving endpoint must return HTTP 2xx within 30 seconds (configurable timeout).
> - Failed deliveries are retried up to 3 times with exponential backoff.
> - Use the `user_uuid` field to filter webhooks for specific users (reduces noise).
> - Always implement idempotency in your webhook handler — the same event may be delivered multiple times.

---

## 5.8 Application Stasis / ARI et Écoute Discrète (Snoop)

### Objectif

Créer une application Asterisk Stasis via l'interface ARI (Asterisk REST Interface) pour intercept and analyze calls in real-time, ou implémenter l'écoute discrète (snoop) pour la supervision d'appels.

### Services impliqués

- **Asterisk ARI** : Interface RESTful pour contrôle d'appels
- **wazo-calld** : Abstraction haut niveau pour ARI
- **wazo-confd** : Configuration des endpoints

### Le Workflow détaillé

#### Étape 1 : Configurer un endpoint ARI dans Asterisk

Ajoutez dans `/etc/asterisk/ari.conf` :

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

> **🔗 Chaînage** : Ces credentials (`wazo:ari_secret_password`) seront utilisés pour l'authentification ARI.

#### Étape 2 : Créer une application Stasis via l'API ARI

```http
POST /ari/applications
Content-Type: application/json
```

**Payload :**
```json
{
  "application_name": "call-recorder",
  "event_sink": "ws://localhost:8088/ari/events"
}
```

#### Étape 3 : Enregistrer un channel dans l'application Stasis

```http
POST /ari/channels
Content-Type: application/json
```

**Payload :**
```json
{
  "endpoint": "PJSIP/1001",
  "app": "call-recorder",
  "variables": {
    "RECORDING": "true"
  }
}
```

**Réponse :**
```json
{
  "id": "channel_uuid_abc123",
  "name": "PJSIP/1001-00000001",
  "state": "Ring",
  "dialplan": {
    "context": "default",
    "exten": "1001",
    "priority": 1
  },
  "variables": {
    "RECORDING": "true"
  }
}
```

> **🔗 Chaînage** : Récupérez le `id` du channel — il permet de contrôler le canal (answer, hold, record, etc.).

#### Étape 4 : Implémenter l'écoute discrète (Snoop) avec ARI

```http
POST /ari/channels
Content-Type: application/json
```

**Payload :**
```json
{
  "endpoint": "PJSIP/snoop:1002",
  "app": "supervisor-monitor",
  "variables": {
    "SPY_CHANNEL": "channel_uuid_abc123"
  }
}
```

> **🔗 Chaînage** : Le préfixe `PJSIP/snoop:` crée un canal d'espionnage. Le `SPY_CHANNEL` indique quel canal écouter.

#### Étape 5 : Contrôler la lecture/mise en attente

```http
# Mettre en hold
POST /ari/channels/{channel_uuid}/hold
X-Auth-Token: {ari_token}

# Reprendre
DELETE /ari/channels/{channel_uuid}/hold
X-Auth-Token: {ari_token}

# Raccrocher
DELETE /ari/channels/{channel_uuid}
X-Auth-Token: {ari_token}
```

#### Étape 6 : Recevoir les événements Stasis

Établissez une WebSocket vers ARI :

```http
wss://wazo.example.com:8089/ari/events?app=call-recorder&api_key=wazo:ari_secret_password
```

**Exemple d'événement StasisStart :**
```json
{
  "type": "StasisStart",
  "timestamp": "2024-01-15T14:30:00.000Z",
  "channel": {
    "id": "channel_uuid_abc123",
    "name": "PJSIP/1001-00000001",
    "state": "Up",
    "caller": {
      "name": "John Doe",
      "number": "1001"
    },
    "connected": {
      "name": "Jane Doe",
      "number": "1002"
    }
  },
  "args": []
}
```

**Exemple d'événement ChannelDestroyed :**
```json
{
  "type": "ChannelDestroyed",
  "timestamp": "2024-01-15T14:35:00.000Z",
  "channel": {
    "id": "channel_uuid_abc123",
    "name": "PJSIP/1001-00000001",
    "cause": 16,
    "cause_txt": "Normal Clearing"
  },
  "duration": 300.5
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - ARI requires enabling the Asterisk REST Interface in `asterisk.conf` and configuring `ari.conf`.
> - The `wazo-calld` service provides a higher-level abstraction over ARI for common operations.
> - Snoop channels consume additional Asterisk resources — use sparingly in production.
> - Stasis applications must handle all channel lifecycle events to avoid orphaned channels.
> - Ensure proper authentication (API key) for ARI endpoints in production.

---

## 5.9 Récapitulatif des Services de Sécurité

> **IMPORTANT** : Toutes les API passent par nginx sur le port 443. Les ports ci-dessous sont les ports directs des microservices (pour debugging uniquement).

| Service | Port Direct | Nginx Route | API Base | Purpose |
|---------|-------------|-------------|----------|---------|
| **wazo-auth** | 9497 | /api/auth/0.1/* | `/api/auth/0.1` | Authentication, tokens, LDAP, SAML |
| **wazo-webhookd** | 9300 | /api/webhookd/1.0/* | `/api/webhookd/1.0` | Webhook subscriptions |
| **Asterisk ARI** | 8088 | N/A | `/ari` | Call control, Stasis apps |
| **wazo-websocketd** | 9502 | /api/websocketd/* | WebSocket | Real-time events |

---

## 5.10 Patterns de Sécurité Recommandés

### Pattern 1 : Rotation Automatisée des Credentials

```python
import os
import requests
from datetime import datetime, timedelta

class WazoSecureClient:
    def __init__(self, host, admin_user, admin_password):
        self.host = host
        self.admin_user = admin_user
        self.admin_password = admin_password
        self.token = None
        self.token_expires = None
        
    def _is_token_valid(self):
        if not self.token or not self.token_expires:
            return False
        # Refresh 5 minutes before expiration
        return datetime.utcnow() < (self.token_expires - timedelta(minutes=5))
    
    def _authenticate(self):
        response = requests.post(
            f"https://{self.host}/api/auth/0.1/token",
            json={
                "backend": "wazo_user",
                "expiration": 3600,
                "username": self.admin_user,
                "password": self.admin_password
            }
        )
        data = response.json()
        self.token = data["token"]
        self.token_expires = datetime.fromisoformat(data["expires_at"].replace("Z", "+00:00"))
        
    def get_token(self):
        if not self._is_token_valid():
            self._authenticate()
        return self.token
    
    def api_call(self, method, endpoint, **kwargs):
        token = self.get_token()
        headers = kwargs.get("headers", {})
        headers["X-Auth-Token"] = token
        kwargs["headers"] = headers
        return requests.request(method, f"https://{self.host}{endpoint}", **kwargs)
```

### Pattern 2 : Webhook avec Signature HMAC

```python
import hmac
import hashlib
import json
from flask import Flask, request, jsonify

app = Flask(__name__)
SECRET = os.environ.get("WEBHOOK_SECRET", "")

@app.route("/webhook", methods=["POST"])
def handle_webhook():
    # Verify HMAC signature
    signature = request.headers.get("X-Wazo-Signature", "")
    expected = hmac.new(
        SECRET.encode(),
        request.data,
        hashlib.sha256
    ).hexdigest()
    
    if not hmac.compare_digest(signature, expected):
        return jsonify({"error": "Invalid signature"}), 401
    
    event = request.json
    event_name = event.get("name")
    event_data = event.get("data", {})
    
    # Process event
    if event_name == "call_created":
        process_new_call(event_data)
    elif event_name == "user_created":
        process_new_user(event_data)
    
    return jsonify({"status": "ok"}), 200
```

---

*Fin de la PARTIE 5 — Fin de l'Ouvrage*
