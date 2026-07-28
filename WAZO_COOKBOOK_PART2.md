---

# PARTIE 2 : Terminaux, Trunks et Routage

Cette partie couvre les workflows de provisioning des terminaux physiques, la configuration des trunks SIP/IAX pour la connectivité externe, et le routage des appels entrants et sortants. Ces scénarios sont essentiels pour connecter Wazo au monde extérieur.

---

## 2.1 Provisioning d'un Téléphone Yealink (7 Étapes)

### Objectif

Provisionner un téléphone Yealink automatiquement via le réseau : enregistrer le device, le configurer avec un template, et déclencher la synchronisation pour que le téléphone récupère sa configuration.

### Services impliqués

- **wazo-provd** : Gestion du provisioning des terminaux
- **wazo-confd** : Configuration des lignes et devices
- **wazo-calld** : Vérification du statut des lignes

### Le Workflow détaillé

#### Étape 1 : Installer le plugin Yealink (première fois seulement)

```http
POST /api/provd/0.1/pgmgr/install/install
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "id": "xivo-yealink-v86"
}
```

**Réponse :**
```json
{
  "id": "op_install_001",
  "plugin_id": "xivo-yealink-v86",
  "status": "pending"
}
```

> **🔗 Chaînage** : Stockez **OPER_ID** = "op_install_001"

#### Étape 2 : Attendre l'installation du plugin (polling)

```http
GET /api/provd/0.1/pgmgr/install/install/{OPER_ID}
X-Auth-Token: {admin_token}
```

**Réponse :**
```json
{
  "id": "op_install_001",
  "plugin_id": "xivo-yealink-v86",
  "status": "done",
  "result": "Plugin installed successfully"
}
```

#### Étape 3 : Supprimer le monitor d'installation

```http
DELETE /api/provd/0.1/pgmgr/install/install/{OPER_ID}
X-Auth-Token: {admin_token}
```

#### Étape 4 : Créer le device dans provd

```http
POST /api/provd/0.1/devmgr/devices
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "mac": "001122334455",
  "model": "Yealink",
  "vendor": "Yealink",
  "description": "Bureau Jean Dupont"
}
```

**Réponse :**
```json
{
  "id": "001122334455",
  "mac": "001122334455",
  "model": "Yealink T46S",
  "vendor": "Yealink",
  "status": "not_configured"
}
```

> **🔗 Chaînage** : Stockez **DEVICE_ID** = "001122334455"

#### Étape 5 : Récupérer la config autoprov

```http
GET /api/provd/0.1/devmgr/devices/{DEVICE_ID}/autoprov
X-Auth-Token: {admin_token}
```

**Réponse :**
```json
{
  "config": "autoprov",
  "lines": []
}
```

#### Étape 6 : Associer la ligne au device

```http
PUT /api/confd/1.1/lines/{LINE_ID}/devices/{DEVICE_ID}
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

#### Étape 7 : Synchroniser le device

```http
POST /api/provd/0.1/devmgr/synchronize
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "id": "001122334455"
}
```

**Réponse :**
```json
{
  "id": "op_sync_001",
  "device_id": "001122334455",
  "status": "pending"
}
```

> **🔗 Chaînage** : Stockez **SYNC_OPER_ID** = "op_sync_001" — attendre la fin puis supprimer

### Point d'attention / Warning

> **⚠️ Important** :
> - Always wait for plugin installation to complete BEFORE creating devices
> - Delete the sync operation after polling to clean up monitoring
> - Le téléphone doit être allumé et connecté au réseau pour recevoir la configuration
> - La synchronisation peut prendre 30-60 secondes

---

## 2.2 Configurer un Trunk SIP Opérateur (6 Étapes)

### Objectif

Configurer un trunk SIP pour connecter Wazo à un opérateur téléphonique (OVH, SFR, Free, etc.) permettant les appels entrants et sortants.

### Services impliqués

- **wazo-confd** : Gestion des transports, endpoints SIP, trunks, outcalls

### Le Workflow détaillé

#### Étape 1 : Créer le transport SIP

```http
POST /api/confd/1.1/sip/transports
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "transport-opérateur-udp",
  "protocol": "udp",
  "bind": "0.0.0.0:5060",
  "external_media_address": "203.0.113.1"
}
```

**Réponse :**
```json
{
  "uuid": "transport-uuid-001",
  "name": "transport-opérateur-udp",
  "protocol": "udp",
  ...
}
```

> **🔗 Chaînage** : Stockez **TRANSPORT_UUID**

#### Étape 2 : Créer l'endpoint SIP du trunk

```http
POST /api/confd/1.1/endpoints/sip
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "label": "trunk-operateur",
  "name": "trunk_operateur",
  "transport_uuid": "transport-uuid-001",
  "aor_section_options": [
    ["contact", "sip:operator.fr:5060"]
  ],
  "auth_section_options": [
    ["username", "mon_compte"],
    ["password", "mot_de_passe_sip"]
  ],
  "endpoint_section_options": [
    ["disallow", "all"],
    ["allow", "ulaw,alaw"],
    ["direct_media", "no"]
  ],
  "registration_section_options": [
    ["server_uri", "sip:operator.fr:5060"],
    ["client_uri", "sip:mon_compte@operator.fr"]
  ]
}
```

**Réponse :**
```json
{
  "uuid": "trunk_sip_uuid_001",
  "label": "trunk-operateur",
  ...
}
```

> **🔗 Chaînage** : Stockez **TRUNK_SIP_UUID**

#### Étape 3 : Créer le trunk

```http
POST /api/confd/1.1/trunks
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "Trunk Opérateur",
  "context": "from-extern"
}
```

**Réponse :**
```json
{
  "id": 10,
  "name": "Trunk Opérateur",
  "context": "from-extern"
}
```

> **🔗 Chaphinage** : Stockez **TRUNK_ID** = 10

#### Étape 4 : Lier l'endpoint SIP au trunk

```http
PUT /api/confd/1.1/trunks/{TRUNK_ID}/endpoints/sip/{TRUNK_SIP_UUID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :** 204 No Content

#### Étape 5 : Créer l'outcall (appels sortants)

```http
POST /api/confd/1.1/outcalls
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "appels-sortants-national"
}
```

**Réponse :**
```json
{
  "id": 5,
  "name": "appels-sortants-national"
}
```

> **🔗 Chaînage** : Stockez **OUTCALL_ID** = 5

#### Étape 6 : Associer le trunk à l'outcall

```http
PUT /api/confd/1.1/outcalls/{OUTCALL_ID}/trunks
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "trunks": [{"id": 10}]
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Le `context` du trunk doit être "from-extern" pour recevoir les appels entrants
> - Les credentials SIP doivent correspondre exactement à ceux fournis par l'opérateur
> - Vérifiez la registration dans wazo-calld après configuration

---

## 2.3 Configurer un Trunk IAX2 Inter-Site (5 Étapes)

### Objectif

Configurer un trunk IAX2 pour interconnecter deux serveurs Wazo ou PBX Asterisk sur des sites distants, permettant des appels internes entre sites sans passer par le réseau public.

### Services impliqués

- **wazo-confd** : Gestion des endpoints IAX, trunks, outcalls

### Le Workflow détaillé

#### Étape 1 : Créer l'endpoint IAX

```http
POST /api/confd/1.1/endpoints/iax
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "trunk-iax-site2",
  "type": "friend",
  "host": "dynamic",
  "context": "from-extern",
  "secret": "iax_secret_password",
  "transfer": "yes",
  "qualify": "yes"
}
```

**Réponse :**
```json
{
  "id": 8,
  "name": "trunk-iax-site2",
  ...
}
```

> **🔗 Chaînage** : Stockez **IAX_ID** = 8

#### Étape 2 : Créer le trunk

```http
POST /api/confd/1.1/trunks
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "Trunk IAX Site 2",
  "context": "from-extern"
}
```

**Réponse :**
```json
{
  "id": 12,
  "name": "Trunk IAX Site 2"
}
```

> **🔗 Chaînage** : Stockez **IAX_TRUNK_ID** = 12

#### Étape 3 : Lier l'endpoint IAX au trunk

```http
PUT /api/confd/1.1/trunks/{IAX_TRUNK_ID}/endpoints/iax/{IAX_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

#### Étape 4 : Créer l'outcall

```http
POST /api/confd/1.1/outcalls
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "inter-site"
}
```

> **🔗 Chaînage** : Stockez **OUTCALL_ID**

#### Étape 5 : Vérifier le trunk

```http
GET /api/calld/1.0/trunks
X-Auth-Token: {admin_token}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - IAX utilise le port 4569 par défaut
> - Le `secret` doit être identique sur les deux serveurs
> - `host: dynamic` permet à l'autre site de s'enregistrer

---

## 2.4 DID Entrant vers Utilisateur (5 Étapes)

### Objectif

Configurer un numéro DID (Direct Inward Dialing) entrants pour rediriger les appels vers un utilisateur spécifique, avec possibilité de planning horaire.

### Services impliqués

- **wazo-confd** : Gestion des schedules, incalls, extensions

### Le Workflow détaillé

#### Étape 1 : Créer le schedule (optionnel)

```http
POST /api/confd/1.1/schedules
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "horaires-bureau",
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

**Réponse :**
```json
{
  "id": 15,
  "name": "horaires-bureau",
  ...
}
```

> **🔗 Chaînage** : Stockez **SCHEDULE_ID** = 15

#### Étape 2 : Créer l'extension entrante

```http
POST /api/confd/1.1/extensions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "exten": "003338000100",
  "context": "from-extern"
}
```

> **🔗 Chaînage** : Stockez **EXT_ID**

#### Étape 3 : Créer l'incall

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
    "type": "user",
    "user_id": "user-uuid-123"
  },
  "extensions": [{"id": EXT_ID}],
  "schedule_id": 15
}
```

**Réponse :**
```json
{
  "id": 20,
  "destination": {
    "type": "user",
    "user_id": "user-uuid-123"
  },
  ...
}
```

> **🔗 Chaînage** : Stockez **INCALL_ID** = 20

#### Étape 4 : Vérifier l'incall

```http
GET /api/confd/1.1/incalls/{INCALL_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

#### Étape 5 : Vérifier la ligne utilisateur

```http
GET /api/confd/1.1/users/{USER_UUID}/lines
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Le `context` de l'extension doit être "from-extern"
> - Sans schedule, l'appel est toujours routé
> - La destination peut être : user, queue, ivr, conference, extension

---

## 2.5 DISA — Accès Direct au Système (4 Étapes)

### Objectif

Configurer le DISA (Direct Inward System Access) permettant à un appelant externe d'accéder au système Wazo et de composer des numéros internes après authentification par PIN.

### Services impliqués

- **wazo-confd** : Gestion des schedules, incalls

### Le Workflow détaillé

#### Étape 1 : Créer un schedule 24/7 (optionnel)

```http
POST /api/confd/1.1/schedules
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "24x7",
  "timezone": "Europe/Paris",
  "open_periods": [
    {
      "start": "00:00",
      "end": "23:59",
      "days": ["monday", "tuesday", "wednesday", "thursday", "friday", "saturday", "sunday"]
    }
  ]
}
```

> **🔗 Chaînage** : Stockez **SCHEDULE_ID**

#### Étape 2 : Créer l'extension DISA

```http
POST /api/confd/1.1/extensions
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "exten": "9999",
  "context": "from-extern"
}
```

> **🔗 Chaînage** : Stockez **EXT_ID**

#### Étape 3 : Créer l'incall avec destination DISA

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
    "type": "application",
    "application": "disa",
    "pin": "5678",
    "context": "default"
  },
  "extensions": [{"id": EXT_ID}],
  "schedule_id": SCHEDULE_ID
}
```

**Réponse :**
```json
{
  "id": 25,
  "destination": {
    "type": "application",
    "application": "disa",
    "pin": "5678",
    "context": "default"
  }
}
```

> **🔗 Chaînage** : Stockez **INCALL_ID**

#### Étape 4 : Vérifier la configuration

```http
GET /api/confd/1.1/incalls/{INCALL_ID}
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Le PIN doit être suffisamment complexe (4+ chiffres)
> - Le `context` DISA détermine quelles extensions sont accessibles
> - Limitez l'utilisation du DISA pour des raisons de sécurité

---

## 2.6 Template SIP Mutualisé pour Endpoints (5 Étapes)

### Objectif

Créer un template SIP réutilisable pour configurer rapidement plusieurs endpoints avec des paramètres communs, simplifiant la gestion de parc de téléphone.

### Services impliqués

- **wazo-confd** : Gestion des templates SIP, endpoints SIP

### Le Workflow détaillé

#### Étape 1 : Créer le template SIP

```http
POST /api/confd/1.1/endpoints/sip/templates
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "label": "tpl-yealink-corp",
  "endpoint_section_options": [
    ["disallow", "all"],
    ["allow", "ulaw,alaw,g722"],
    ["direct_media", "no"],
    ["rtp_symmetric", "yes"]
  ],
  "aor_section_options": [
    ["max_contacts", "1"]
  ]
}
```

**Réponse :**
```json
{
  "uuid": "template-uuid-001",
  "label": "tpl-yealink-corp",
  ...
}
```

> **🔗 Chaînage** : Stockez **TPL_UUID**

#### Étape 2 : Créer un endpoint avec le template (Alice)

```http
POST /api/confd/1.1/endpoints/sip
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "label": "alice-sip",
  "name": "alice",
  "templates": [{"uuid": "template-uuid-001"}],
  "auth_section_options": [
    ["username", "alice"],
    ["password", "alice_password"]
  ]
}
```

**Réponse :**
```json
{
  "uuid": "sip-uuid-alice",
  "label": "alice-sip",
  ...
}
```

> **🔗 Chaînage** : Stockez **SIP_UUID_ALICE**

#### Étape 3 : Créer un endpoint avec le template (Bob)

```http
POST /api/confd/1.1/endpoints/sip
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "label": "bob-sip",
  "name": "bob",
  "templates": [{"uuid": "template-uuid-001"}],
  "auth_section_options": [
    ["username", "bob"],
    ["password", "bob_password"]
  ]
}
```

> **🔗 Chaînage** : Stockez **SIP_UUID_BOB**

#### Étape 4 : Visualiser les options héritées

```http
GET /api/confd/1.1/endpoints/sip/{SIP_UUID_ALICE}?view=merged
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Réponse :**
```json
{
  "uuid": "sip-uuid-alice",
  "label": "alice-sip",
  "endpoint_section_options": [
    ["disallow", "all"],
    ["allow", "ulaw,alaw,g722"],
    ...
  ]
}
```

#### Étape 5 : Mettre à jour le template (mise à jour globale)

```http
PUT /api/confd/1.1/endpoints/sip/templates/{TPL_UUID}
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "endpoint_section_options": [
    ["disallow", "all"],
    ["allow", "ulaw,alaw,g722,h264"],
    ["direct_media", "no"]
  ]
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Les endpoints enfants peuvent surcharger les options du template
> - La mise à jour du template affecte tous les endpoints enfants
> - Utilisez `view=merged` pour voir la configuration finale

---

## 2.7 Transport TLS et WSS pour WebRTC (6 Étapes)

### Objectif

Configurer des transports SIP sécurisés TLS et WSS (WebSocket Secure) pour permettre l'enregistrement de téléphones distants et clients WebRTC.

### Services impliqués

- **wazo-confd** : Gestion des transports SIP

### Le Workflow détaillé

#### Étape 1 : Créer le transport TLS

```http
POST /api/confd/1.1/sip/transports
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "transport-tls",
  "protocol": "tls",
  "bind": "0.0.0.0:5061",
  "certfile": "/etc/asterisk/keys/wazo.crt",
  "privkey": "/etc/asterisk/keys/wazo.key",
  "method": "tlsv1_2"
}
```

**Réponse :**
```json
{
  "uuid": "tls-transport-uuid",
  "name": "transport-tls",
  "protocol": "tls"
}
```

> **🔗 Chaînage** : Stockez **T_TLS_UUID**

#### Étape 2 : Créer le transport WSS (WebRTC)

```http
POST /api/confd/1.1/sip/transports
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "transport-wss",
  "protocol": "wss",
  "bind": "0.0.0.0:8089"
}
```

**Réponse :**
```json
{
  "uuid": "wss-transport-uuid",
  "name": "transport-wss",
  "protocol": "wss"
}
```

> **🔗 Chaînage** : Stockez **T_WSS_UUID**

#### Étape 3 : Configurer un endpoint pour TLS

```http
PUT /api/confd/1.1/endpoints/sip/{SIP_UUID}
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "transport_uuid": "tls-transport-uuid",
  "endpoint_section_options": [
    ["media_encryption", "srtp"]
  ]
}
```

#### Étape 4 : Configurer un trunk pour WebRTC

```http
PUT /api/confd/1.1/endpoints/sip/{TRUNK_SIP_UUID}
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "transport_uuid": "wss-transport-uuid",
  "endpoint_section_options": [
    ["dtls_enable", "yes"],
    ["dtls_verify", "fingerprint"],
    ["dtls_auto_generate_cert", "yes"],
    ["ice_support", "yes"],
    ["rtcpmux", "yes"]
  ]
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Les certificats TLS doivent être signés par une CA reconnue ou ajoutés au truststore
> - WSS est obligatoire pour les clients WebRTC (navigateurs)
> - SRTP (media_encryption) requiert TLS

---

## 2.8 Configuration RTP et PJSIP Global (5 Étapes)

### Objectif

Configurer les paramètres globaux Asterisk pour les codecs audio, les ports RTP, et les options PJSIP avancées.

### Services impliqués

- **wazo-confd** : Configuration Asterisk

### Le Workflow détaillé

#### Étape 1 : Lire la configuration RTP actuelle

```http
GET /api/confd/1.1/asterisk/rtp/general
X-Auth-Token: {admin_token}
```

**Réponse :**
```json
{
  "options": {
    "rtpstart": 10000,
    "rtpend": 20000
  }
}
```

#### Étape 2 : Configurer les ports RTP

```http
PUT /api/confd/1.1/asterisk/rtp/general
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "options": {
    "rtpstart": 10000,
    "rtpend": 20000,
    "strictrtp": "yes",
    "ICESupport": "no"
  }
}
```

#### Étape 3 : Configurer PJSIP Global

```http
PUT /api/confd/1.1/asterisk/pjsip/global
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "options": {
    "keep_alive_interval": 30,
    "max_forwards": 70
  }
}
```

#### Étape 4 : Configurer PJSIP System

```http
PUT /api/confd/1.1/asterisk/pjsip/system
Content-Type: application/json
X-Auth-Token: {admin_token}
```

**Payload :**
```json
{
  "options": {
    "timer_t1": 500,
    "timer_b": 3000
  }
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Les changements nécessitent un redémarrage d'Asterisk
> - Les ports RTP (10000-20000) doivent être autorisés dans le firewall
> - `strictrtp` aide à prévenir les problèmes NAT

---

## 2.9 Music on Hold avec Fichier Audio (4 Étapes)

### Objectif

Configurer une musique d'attente personnalisée avec des fichiers audio uploadés, puis l'appliquer à une queue ou un utilisateur.

### Services impliqués

- **wazo-confd** : Gestion MOH

### Le Workflow détaillé

#### Étape 1 : Créer la catégorie MOH

```http
POST /api/confd/1.1/moh
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "name": "musique-corporate",
  "mode": "files",
  "random": true
}
```

**Réponse :**
```json
{
  "uuid": "moh-uuid-001",
  "name": "musique-corporate",
  "mode": "files"
}
```

> **🔗 Chaînage** : Stockez **MOH_UUID**

#### Étape 2 : Uploader le fichier audio

```http
PUT /api/confd/1.1/moh/{MOH_UUID}/files/accueil.wav
Content-Type: audio/wav
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}

[binary audio data]
```

#### Étape 3 : Vérifier la MOH

```http
GET /api/confd/1.1/moh/{MOH_UUID}
X-Auth-Token: {admin_token}
```

#### Étape 4 : Appliquer à une queue

```http
PUT /api/confd/1.1/queues/{QUEUE_ID}
Content-Type: application/json
X-Auth-Token: {admin_token}
Wazo-Tenant: {tenant_uuid}
```

**Payload :**
```json
{
  "music_on_hold": "moh-uuid-001"
}
```

### Point d'attention / Warning

> **⚠️ Important** :
> - Les formats supportés : WAV, MP3, OGG
> - La fréquence d'échantillonnage recommandée : 8kHz ou 16kHz
> - La MOH est diffusée quand l'appelant est en attente

---

## 2.10 Récapitulatif des Endpoints de Provisioning

| Ressource | CRUD | Endpoint |
|-----------|-----|----------|
| Device | C | POST /devmgr/devices |
| Device | R | GET /devmgr/devices/{id} |
| Device | D | DELETE /devmgr/devices/{id} |
| Sync | C | POST /devmgr/synchronize |
| Plugin | C | POST /pgmgr/install/install |
| Transport SIP | C | POST /sip/transports |
| Endpoint SIP | C | POST /endpoints/sip |
| Endpoint IAX | C | POST /endpoints/iax |
| Trunk | C | POST /trunks |
| Outcall | C | POST /outcalls |
| Incall | C | POST /incalls |
| Schedule | C | POST /schedules |
| MOH | C | POST /moh |
| Template SIP | C | POST /endpoints/sip/templates |

---

*Fin de la PARTIE 2*
