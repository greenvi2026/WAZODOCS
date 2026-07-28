# Chapitre 9 — ARI (Asterisk REST Interface) sur Wazo 26.06

> **Statut** : testé et validé en live sur `wazohermesx.tail51557a.ts.net` (Asterisk 22.8.2, Debian 12.14, juillet 2026).
> Tous les pièges documentés ici ont été rencontrés et résolus en production.

---

## 9.1 Vue d'ensemble

**ARI (Asterisk REST Interface)** est l'API HTTP/WebSocket d'Asterisk qui permet de
contrôler les appels téléphoniques programmatiquement. Sur Wazo 26.06, c'est
**la** méthode recommandée pour :

- Bots vocaux (type AVA, GPT-realtime)
- IVR dynamiques avec logique externe
- Call recording programmatique
- Bridges complexes multi-parties
- Originate call (appels sortants)

### Différence avec AMI (Asterisk Manager Interface)

| Feature | ARI (HTTP/WS) | AMI (TCP/Events) |
|---|---|---|
| Transport | HTTP + WebSocket | TCP plain |
| Auth | HTTP Basic | Plain text |
| Stasis apps | ✅ Natif | ❌ Limité |
| WebSocket events | ✅ Standard | ❌ Pas natif |
| Channels control | RESTful | Action/Response |
| Originate call | POST /channels | Action: Originate |
| Default port Wazo | **5039** | 5038 |

**Pour un bot vocal/IA, ARI est fortement préféré.**

---

## 9.2 Architecture

```
┌─────────────────────────────────┐
│         Wazo Platform           │
│  ┌──────────────────────────┐   │
│  │   Asterisk 22 + ARI      │◄──── Port 5039 (HTTP/WS)
│  │   res_ari (14 modules)   │   │
│  │   app_stasis             │   │
│  │   dialplan: [ava-agent]  │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
                ▲
                │ ARI (HTTP REST + WebSocket events)
                │
┌─────────────────────────────────┐
│        Bot / Client Python      │
│  ┌──────────────────────────┐   │
│  │  aiohttp + WebSocket     │   │
│  │  Stasis app listener     │───┼──► api.groq.com (LLM)
│  │  Channel control         │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
                ▲
                │ SIP (PJSIP)
                │
┌─────────────────────────────────┐
│        Téléphone / Softphone    │
└─────────────────────────────────┘
```

---

## 9.3 Configuration minimale côté Wazo

### 9.3.1 User ARI (créer sans toucher `ari.conf`)

**RÈGLE D'OR** : ne **jamais** modifier `/etc/asterisk/ari.conf` directement.
C'est un `#include ari.d/*.conf` géré par Wazo.

Créer `/etc/asterisk/ari.d/02-ava.conf` (Wazo fournit déjà `01-wazo.conf`) :

```ini
; AVA AI Voice Agent - user ARI
; NE PAS inclure de section [general] ici (déjà dans 01-wazo.conf)
; Sinon : duplicate object 'general' (pitfall #22)

[ava_user]
type = user
read_only = no
password = <MOT_DE_PASSE_GENERE>
password_format = plain
```

```bash
chown asterisk:www-data /etc/asterisk/ari.d/02-ava.conf
chmod 0660 /etc/asterisk/ari.d/02-ava.conf
asterisk -rx "module reload res_ari"
```

⚠️ **PIÈGE #22 (CRITIQUE)** : ne JAMAIS avoir deux `[general]` dans `ari.d/*.conf`.
Asterisk refuse de parser `ari.conf` avec l'erreur :
```
Config file 'ari.conf' could not be loaded; configuration contains
       a duplicate object: 'general' of type 'general'
```
Conséquence : `res_ari.so` se charge (Use Count 13) mais ARI inactif.

### 9.3.2 Dialplan Stasis

Créer `/etc/asterisk/extensions_extra.d/ava-agent.conf` (non managé par wazo-confgend) :

```ini
[ava-agent]
; Pattern wildcard : accepte tout numéro composé
exten => _X.,1,NoOp(=== AVA - ${EXTEN} from ${CALLERID(num)} ===)
 same => n,Stasis(ava-stasis,${EXTEN},${CALLERID(num)})
 same => n,Hangup()

; Extension start pour appels SIP entrants réels
exten => s,1,NoOp(=== AVA entrant ${CALLERID(num)} ===)
 same => n,Answer()
 same => n,Wait(1)
 same => n,Stasis(ava-stasis,s,${CALLERID(num)})
 same => n,Hangup()

exten => h,1,NoOp(AVA hangup)
 same => n,Return()
```

```bash
chown asterisk:www-data /etc/asterisk/extensions_extra.d/ava-agent.conf
chmod 0660 /etc/asterisk/extensions_extra.d/ava-agent.conf
asterisk -rx "dialplan reload"
```

### 9.3.3 Vérification

```bash
# 1. User ARI créé
asterisk -rx "ari show users"
# Sortie attendue :
# r/o?  Username
# ----  --------
# No    ava_user
# No    xivo

# 2. ARI configuré
asterisk -rx "ari show status"
# Sortie attendue :
# ARI Status:
# Enabled: Yes
# Output format: compact
# Auth realm: Asterisk REST Interface
# Allowed Origins: *

# 3. /ari/ listé dans HTTP
asterisk -rx "http show status" | grep "/ari"
# /ari/... => Asterisk RESTful API

# 4. Test login
curl -u ava_user:<MDP> http://127.0.0.1:5039/ari/asterisk/info
# → JSON avec build, system, config, status
```

---

## 9.4 Endpoints principaux

### 9.4.1 Informations Asterisk

```bash
# GET /ari/asterisk/info
curl -u user:pwd http://host:5039/ari/asterisk/info

# GET /ari/asterisk/variables (variables globales)
curl -u user:pwd http://host:5039/ari/asterisk/variables

# GET /ari/asterisk/log  (Wazo ne loggue pas par défaut)
```

### 9.4.2 Channels (canaux actifs)

```bash
# Lister les canaux
curl -u user:pwd http://host:5039/ari/channels

# Détails d'un canal
curl -u user:pwd http://host:5039/ari/channels/<channel-id>

# Créer un canal (test ARI direct)
curl -u user:pwd -X POST http://host:5039/ari/channels \
  -H "Content-Type: application/json" \
  -d '{
    "endpoint": "Local/1001@ava-agent",
    "app": "ava-stasis",
    "callerId": "TestCall"
  }'
# → JSON avec id, name, state, dialplan context, etc.

# Contrôler un canal
curl -u user:pwd -X POST http://host:5039/ari/channels/<id>/answer       # décrocher
curl -u user:pwd -X POST http://host:5039/ari/channels/<id>/ring         # faire sonner
curl -u user:pwd -X POST http://host:5039/ari/channels/<id>/play \
  -H "Content-Type: application/json" \
  -d '{"media": "sound:custom/bienvenue"}'                                 # jouer un son
curl -u user:pwd -X DELETE http://host:5039/ari/channels/<id>             # raccrocher
```

### 9.4.3 Bridges (conférences)

```bash
# Créer un bridge
curl -u user:pwd -X POST http://host:5039/ari/bridges \
  -H "Content-Type: application/json" \
  -d '{"type": "mixing,dtmf_events"}'

# Ajouter un channel au bridge
curl -u user:pwd -X POST http://host:5039/ari/bridges/<bridge-id>/addChannel \
  -H "Content-Type: application/json" \
  -d '{"channel": "<channel-id>"}'

# Lister les canaux du bridge
curl -u user:pwd http://host:5039/ari/bridges/<bridge-id>
```

### 9.4.4 Endpoints SIP

```bash
# Lister tous les endpoints
curl -u user:pwd http://host:5039/ari/endpoints

# Détails d'un endpoint
curl -u user:pwd http://host:5039/ari/endpoints/PJSIP/<endpoint-id>
```

### 9.4.5 Recordings (enregistrements)

```bash
# Démarrer enregistrement d'un canal
curl -u user:pwd -X POST http://host:5039/ari/channels/<id>/record \
  -H "Content-Type: application/json" \
  -d '{"name": "call-001", "format": "wav"}'

# Arrêter
curl -u user:pwd -X DELETE http://host:5039/ari/channels/<id>/record

# Lister recordings
curl -u user:pwd http://host:5039/ari/recordings/stored
```

### 9.4.6 Applications Stasis

```bash
# Lister les apps Stasis actives
curl -u user:pwd http://host:5039/ari/applications
# → [{"name": "callcontrol", ...}, {"name": "adhoc_conference", ...}, ...]

# Détails
curl -u user:pwd http://host:5039/ari/applications/ava-stasis
```

### 9.4.7 Playbacks (sons en cours)

```bash
curl -u user:pwd http://host:5039/ari/playbacks/<playback-id>
# Contrôler : POST /playbacks/{id}/control (pause, unpause, restart, stop, forward, reverse)
```

---

## 9.5 WebSocket ARI (events Stasis)

### 9.5.1 Connexion

```python
import aiohttp
import json

async def listen_ari():
    auth = aiohttp.BasicAuth("ava_user", "password")  # aiohttp < 4.0
    async with aiohttp.ClientSession(auth=auth) as session:
        ws_url = "http://host:5039/ari/events?app=ava-stasis&subscribeAll=events"
        async with session.ws_connect(ws_url) as ws:
            async for msg in ws:
                if msg.type != aiohttp.WSMsgType.TEXT:
                    continue
                evt = json.loads(msg.data)
                yield evt
```

**⚠️ Piège aiohttp 3.9+** : `aiohttp.BasicAuth` est deprecated. Utiliser :
```python
headers = {"Authorization": aiohttp.encode_basic_auth("ava_user", "password")}
async with aiohttp.ClientSession(headers=headers) as session:
```

### 9.5.2 Events principaux

| Event | Quand | Champs clés |
|---|---|---|
| `StasisStart` | Nouveau call dans Stasis | `channel.id`, `channel.caller.number`, `args[]` |
| `StasisEnd` | Channel quitte Stasis | `channel.id` |
| `ChannelCreated` | Nouveau channel créé | `channel.id`, `channel.dialplan.context` |
| `ChannelDestroyed` | Channel terminé | `channel.id`, `cause`, `cause_txt` |
| `ChannelCallerId` | Caller ID assigné | `channel.id`, `caller.number` |
| `ChannelConnectedLine` | Connected line update | `channel.id`, `connected.number` |
| `ChannelDialplan` | Dialplan en cours | `channel.id`, `dialplan.app_name`, `dialplan.app_data` |
| `ChannelVarset` | Variable de canal changée | `channel.id`, `variable`, `value` |
| `ChannelDtmfReceived` | DTMF reçu | `channel.id`, `digit`, `duration_ms` |
| `Dial` | Dial en cours | `dial.status`, `caller`, `peer` |
| `PlaybackFinished` | Son terminé | `playback.id`, `playback.target_uri` |
| `RecordingStarted` / `RecordingFinished` | Enregistrement | `recording.name`, `recording.format` |

### 9.5.3 Exemple Python complet (client Stasis minimal)

```python
import asyncio
import aiohttp
import json

AR_URL = "http://127.0.0.1:5039"
USER = "ava_user"
PASS = "<MOT_DE_PASSE>"
APP = "ava-stasis"

async def main():
    auth = aiohttp.BasicAuth(USER, PASS)
    async with aiohttp.ClientSession(auth=auth) as session:
        # S'abonner aux events
        ws_url = f"{AR_URL}/ari/events?app={APP}&subscribeAll=events"
        async with session.ws_connect(ws_url) as ws:
            print(f"WS connected, app={APP}")
            async for msg in ws:
                if msg.type != aiohttp.WSMsgType.TEXT:
                    continue
                evt = json.loads(msg.data)
                t = evt.get("type", "?")
                if t == "StasisStart":
                    ch_id = evt["channel"]["id"]
                    caller = evt["channel"]["caller"]["number"]
                    args = evt.get("args", [])
                    print(f"StasisStart {ch_id} caller={caller} args={args}")

                    # Répondre
                    async with session.post(f"{AR_URL}/ari/channels/{ch_id}/answer") as r:
                        print(f"answer -> {r.status}")

                    # Jouer un son
                    async with session.post(
                        f"{AR_URL}/ari/channels/{ch_id}/play",
                        json={"media": "sound:custom/bienvenue"}
                    ) as r:
                        print(f"play -> {r.status}")

asyncio.run(main())
```

**Test bout-en-bout validé** sur `wazohermesx` (juillet 2026) :
```
[8.51s] >> StasisStart channel=1783953092.9
  ** STASIS_START args=['4004', ''] caller=
HTTP 200  /ari/asterisk/info
{"build":{"os":"Linux","kernel":"5.15.0-164-generic",...}}
```

---

## 9.6 Pièges vérifiés en production (22 pièges)

### Pièges CRITIQUES (cassent l'install)

**#1. Ne PAS modifier `/etc/asterisk/ari.conf`**
C'est un `#include ari.d/*.conf` géré par Wazo. Créer `ari.d/XX-name.conf`.

**#2. Port ARI = 5039 sur Wazo 26.06 (pas 8088)**
Vérifier `/etc/asterisk/http.d/01-wazo.conf` pour le bindport réel.

**#3. `pip3 install` sur un serveur Wazo CASSE `wazo-ui`**
Utiliser `apt install` ou un venv isolé.

**#4. Login admin Wazo via API = HTTP Basic Auth (PAS JSON body)**
```bash
curl -u root:pass -X POST https://wazo/api/auth/0.1/token
```

**#5. Hash password root Wazo peut être cassé**
Créer un nouveau user via `PasswordEncrypter` + UPDATE SQL direct.

**#6. Policy master n'a PAS les ACL `confd.contexts.*`**
Créer une policy custom avec `confd.#` wildcard.

**#14. `/ari/` absent même avec res_ari chargé (Use Count 13)**
Voir section 9.10 "Diagnostic avancé".

**#20. `res_ari.so` Use Count > 0 mais HTML 404 sur /ari**
Voir section 9.10.

**#21. Paquet `asterisk` upstream sans ARI**
Sur Wazo, il faut `wazo-asterisk` (fork Wazo).

**#22. ⚠️ `[general]` dupliqué dans `ari.d/*.conf` casse ARI**
Cause du bug qui a coûté 3 jours : mon script `configure-wazo.sh` créait
un `[general]` vide en plus de celui de `01-wazo.conf`. Asterisk refusait
de parser ari.conf → ARI inactif.
**Fix** : un seul `[general]` dans tout `ari.d/`, fichiers custom = `[user]` uniquement.

### Pièges MOYENS (font perdre du temps)

**#7. `Answer()` + `Wait()` cassent les canaux `Local/...`**
Les canaux ARI auto-générés (`Local/1001@ava-agent`) n'ont pas de média.
Pour les tests, utiliser `Stasis()` direct sans `Answer`.

**#8. Dialplan doit être dans `extensions_extra.d/`**
Pas dans `extensions.conf` (régénéré par wazo-confgend).

**#9. Table `auth_policy_access` (PAS `auth_policy_acl`)**
Schéma Wazo récent utilise `(policy_uuid, access_id)` comme FK vers `auth_access(id, access)`.

**#10. Policy `auth_policy` requiert `slug`**
`NOT NULL` constraint.

### Pièges INFORMATIVES

**#11. Contexte Wazo généré dynamiquement (vide)**
Le contexte `ctx-master-internal-...` créé via API a juste `i`/`t` en dialplan.

**#12. Nom technique du contexte auto-renommé**
`name: "ava-agent"` → `name: "ctx-master-internal-3a664d19-..."`.

**#13. `extensions_extra.d/` permissions restrictives**
`chown asterisk:www-data && chmod 0775`.

**#15. Restart ne suffit pas si conf cassée**
Voir #22.

**#16. `http.d/*.conf` ignoré si http.conf n'a pas `#include`**
Sur Wazo, c'est déjà inclus par défaut.

**#17. `wazo-confgend.timer` régénère `01-wazo.conf` toutes les 5 min**
Stopper le timer avant les modifs manuelles.

**#18. "Not enabled" après restart = conflit http.d/*.conf**
Voir #19.

**#19. Conflit `[general]` entre 2 fichiers http.d/**
Un seul fichier doit avoir `[general]`.

**Voir `ava-deploy/docs/WAZO-26.06-PITFALLS.md` pour le détail complet.**

---

## 9.7 Scripts opérationnels (ava-deploy)

Le dépôt `mobilejudi/ava-deploy` contient 8 scripts dédiés ARI :

| Script | Usage |
|---|---|
| `scripts/install-local.sh` | Install AVA + ARI user + dialplan sur même serveur Wazo |
| `scripts/install-remote.sh` | Install AVA seul (Wazo distant) — fix bug `AVA_DIALPLAN_CONTEXT` |
| `scripts/configure-wazo.sh` | Configure juste ARI + dialplan côté Wazo (corrigé : pas de `[general]` dupliqué) |
| `scripts/validate.sh` | Tests post-install ARI/Stasis/Groq/service (diagnostic enrichi 401/404/000) |
| `scripts/diagnose-ari.sh` | Diagnostic complet ARI avec option `--fix` |
| `scripts/recover-ari.sh` | Recovery auto du cas pathologique #20 (Use Count > 0 mais HTML 404) |
| `scripts/uninstall.sh` | Rollback complet |
| `examples/dialplan/ava-agent.conf` | Snippet dialplan à copier |
| `examples/ari-conf/02-ava.conf` | Snippet user ARI (sans `[general]`) |

---

## 9.8 Cas pratique : connecter AVA à Wazo

### Variables d'environnement AVA `ai_engine`

```bash
# /opt/ava/.env
ASTERISK_HOST=192.168.1.17         # ou 127.0.0.1 si même machine
ASTERISK_ARI_PORT=5039             # PAS 8088
ASTERISK_ARI_USERNAME=ava_user
ASTERISK_ARI_PASSWORD=mCsd2cIoLQI7Og3zrMP5nKHiAtXtixf9
ASTERISK_ARI_APP=ava-stasis        # DOIT matcher Stasis() du dialplan

# LLM
GROQ_API_KEY=gsk_xxx
LLM_BASE_URL=https://api.groq.com/openai/v1
LLM_MODEL=llama-3.3-70b-versatile
```

### Test bout-en-bout en 30 secondes

```bash
# 1. Test ARI login
curl -u ava_user:$PWD http://$HOST:5039/ari/asterisk/info | python3 -m json.tool | head

# 2. Test création canal via ARI (sans téléphone physique)
curl -u ava_user:$PWD -X POST http://$HOST:5039/ari/channels \
  -H "Content-Type: application/json" \
  -d '{"endpoint":"Local/1001@ava-agent","app":"ava-stasis","callerId":"Test"}'
# → doit retourner id, name "Local/1001@ava-agent-00000000;1", state "Down"

# 3. Vérifier que StasisStart arrive (avec un listener WS actif)
# (le listener AVA devrait recevoir l'event StasisStart args=['1001'])
```

### Dialplan complet recommandé

```ini
[ava-agent]
; Tests ARI directs (canaux Local/...)
exten => _X.,1,NoOp(=== AVA ARI test ${EXTEN} ===)
 same => n,Stasis(ava-stasis,${EXTEN},${CALLERID(num)})
 same => n,Hangup()

; Appels SIP entrants réels (canaux PJSIP avec média)
exten => s,1,NoOp(=== AVA entrant ${CALLERID(num)} ===)
 same => n,Answer()
 same => n,Wait(1)
 same => n,Stasis(ava-stasis,s,${CALLERID(num)})
 same => n,Hangup()

; Transferts aveugles vers extensions internes
[ava-transfers]
exten => _X.,1,NoOp(AVA Blind Transfer to ${EXTEN})
 same => n,Dial(PJSIP/${EXTEN},30,tT)
 same => n,Hangup()

exten => h,1,NoOp(AVA hangup)
 same => n,Return()
```

---

## 9.9 Snippets copy-paste

### Python : POST /ari/channels avec gestion d'erreur

```python
import aiohttp

async def originate(session, host: str, port: int, user: str, pwd: str):
    url = f"http://{host}:{port}/ari/channels"
    body = {
        "endpoint": "Local/1001@ava-agent",
        "app": "ava-stasis",
        "callerId": "Outbound",
        "variables": {"CHANNEL(linkedid)": "outbound-001"}
    }
    try:
        async with session.post(url, json=body,
                                auth=aiohttp.BasicAuth(user, pwd)) as r:
            if r.status in (200, 201):
                return await r.json()
            else:
                text = await r.text()
                raise RuntimeError(f"ARI POST failed: HTTP {r.status}: {text}")
    except aiohttp.ClientError as e:
        raise RuntimeError(f"ARI connection failed: {e}")
```

### Bash : vérificateur ARI rapide

```bash
#!/usr/bin/env bash
# check-ari.sh — vérifie qu'ARI est opérationnel
set -u
HOST="${1:-127.0.0.1}"
PORT="${2:-5039}"
USER="${3:-ava_user}"
PASS="${4:-$(cat /etc/ava/credentials.env 2>/dev/null | grep ASTERISK_ARI_PASSWORD | cut -d= -f2)}"

echo "Test ARI sur $HOST:$PORT"

# 1. ARI listé
if ! curl -sf -u "$USER:$PASS" "http://$HOST:$PORT/ari/asterisk/info" -o /tmp/ari.json; then
    echo "FAIL : /ari/asterisk/info inaccessible"
    exit 1
fi

# 2. Version
VER=$(python3 -c "import json; print(json.load(open('/tmp/ari.json'))['system']['version'])")
echo "  Asterisk version: $VER"

# 3. Modules res_ari
MODS=$(asterisk -rx "module show like res_ari" | grep -c Running)
echo "  res_ari modules: $MODS"

# 4. /ari dans HTTP
if asterisk -rx "http show status" | grep -q "/ari/..."; then
    echo "  /ari/ registered: OK"
else
    echo "  /ari/ registered: MISSING"
fi

echo "OK"
```

---

## 9.10 Diagnostic avancé (quand /ari ne marche pas)

### Procédure systématique

```bash
# 1. Les .so existent ?
ls /usr/lib/asterisk/modules/res_ari*.so
# Si vide → installer wazo-asterisk (paquet correct)

# 2. Les modules sont chargés ?
asterisk -rx "module show like res_ari" | head -3
# Doit afficher : res_ari.so ... 13 ... Running

# 3. /ari listé dans HTTP ?
asterisk -rx "http show status" | grep "/ari"
# Doit afficher : /ari/... => Asterisk RESTful API

# 4. ari show status
asterisk -rx "ari show status"
# Si "Error getting ARI configuration" → pb de conf ARI

# 5. Verbose au boot
systemctl stop asterisk
/usr/sbin/asterisk -cvvv 2>&1 | grep -iE "ari|stasis|error|warn" | head -20
# Chercher : "duplicate object 'general'" → pitfall #22
```

### Erreurs spécifiques et leurs fixes

| Erreur | Cause | Fix |
|---|---|---|
| `Error getting ARI configuration` | ari.conf invalide | Vérifier `[general]` dupliqué (#22) |
| `Unable to load module res_ari.so` | .so manquante | Installer `wazo-asterisk` |
| `Connection refused :5039` | Asterisk pas démarré ou bindaddr=127.0.0.1 | Vérifier `http.d/01-wazo.conf` |
| `HTTP 404` sur /ari/asterisk/info | Module pas enregistré sur HTTP | Restart complet + vérifier verbose |
| `HTTP 401` | Mauvais user/password | Vérifier `ari show users` |
| `Use Count 0` sur res_ari.so | Module chargé mais pas utilisé | Vérifier qu'aucune Stasis ne l'utilise |

### Script de recovery auto

```bash
# Depuis ava-deploy
curl -fsSL https://raw.githubusercontent.com/mobilejudi/ava-deploy/main/scripts/recover-ari.sh | sudo bash
```

Ou :
```bash
sudo /opt/ava-deploy/scripts/recover-ari.sh
```

Le script :
1. Stoppe `wazo-confgend.timer` (cause #17)
2. Restart complet Asterisk (avec `kill -9` si bloqué)
3. Vérifie `/ari/` apparaît
4. Si non : unload+load forcé des modules HTTP
5. Sinon : guide de réinstallation package

---

## 9.11 Performances observées

| Opération | Latence | Notes |
|---|---|---|
| `GET /ari/asterisk/info` | < 50ms | HTTP local |
| `POST /ari/channels` (Local/...) | < 100ms | Création + Dial |
| WebSocket event delivery | < 50ms | Subscribed apps |
| Groq LLM TTFB (llama-3.3-70b) | 88ms | Depuis wazohermesx |
| Groq streaming first token | < 100ms | Via OpenAI client |

---

## 9.12 Sécurité

### Réseau

- **Bindaddr** : `127.0.0.1` recommandé (localhost uniquement).
- **Bindaddr `0.0.0.0`** : OK si firewall restreint (Tailscale, VPN).
- **Pas d'exposition publique** : ARI n'a aucune auth avancée (Basic Auth only).

### Credentials

- `/etc/ava/credentials.env` : `chmod 0600`, owned `ava:ava`
- `/opt/ava/.env` : `chmod 0600`, owned `ava:ava`
- ARI user password : généré à l'install, 24 chars base64url

### Audit

```bash
# Vérifier les accès ARI récents
journalctl -u asterisk --since "1 day ago" | grep -i "ari\|authentication"

# Lister les apps Stasis actives
asterisk -rx "ari show applications"
```

---

## 9.13 Références croisées

- **ava-deploy** : https://github.com/mobilejudi/ava-deploy — scripts déploiement
- **ava-deploy issues** : https://gitlab.com/mobilejudi2/ava-deploy
- **WAZO_PLATFORM_DOCUMENTATION_EXHAUSTIVE.md** : chapitre Asterisk
- **WAZO_API_BIBLE_CH3_CH4.md** : API confd (contextes, extensions, incalls)
- **WAZO_API_BIBLE_CH7.md** : API calld (Call Control via calld, pas ARI direct)
- **WAZO_AUTH_PITFALLS.md** : bugs auth wazo-auth (référence skills)

---

**Statut doc** : juillet 2026, validé sur wazohermesx.  
**Prochaine mise à jour** : si Asterisk 23+ change l'API ARI (peu probable, ARI est stable depuis v13).