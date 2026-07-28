# Bible API Wazo & Cookbook — Documentation Développeur

![Markdown Lint](https://img.shields.io/badge/markdownlint-0%2F11-brightgreen) ![Wazo](https://img.shields.io/badge/Wazo-26.06-blue) ![License](https://img.shields.io/badge/license-internal-lightgrey)

Documentation technique exhaustive de la plateforme **Wazo** (Unified Communications as a Service) à destination des développeurs, intégrateurs et opérateurs.

Deux corpus complémentaires, séparés volontairement pour rendre la lecture plus rapide :

- **Bible API** : référence de chaque endpoint, payload, header, code HTTP et piège connu.
- **Cookbook** : recettes pas-à-pas, prêtes à exécuter, pour assembler des workflows complets (provisioning, call center, temps réel, sécurité).

> Version cible : **Wazo Platform 26.06** (Asterisk 22+, Python 3.11).
> Toutes les URL d'exemple utilisent `https://wazo.example.com/` et le routing nginx par défaut (`/api/...`). Adaptez-les à votre instance.

## Sommaire

| # | Document | Description courte |
|---|----------|--------------------|
| | **BIBLE API — Guide du Développeur** | |
| 1 | [WAZO_API_BIBLE_CH1_CH2.md](./WAZO_API_BIBLE_CH1_CH2.md) | Chapitres 1 & 2 — Fondations microservices, architecture multi-tenant, `wazo-auth` (tokens, ACL, politiques, LDAP). |
| 2 | [WAZO_API_BIBLE_CH3_CH4.md](./WAZO_API_BIBLE_CH3_CH4.md) | Chapitres 3 & 4 — `wazo-confd` (contextes, endpoints SIP PJSIP, lignes, extensions, utilisateurs, voicemails, permissions) + trunks/outcalls/incalls. |
| 3 | [WAZO_API_BIBLE_CH5_CH6.md](./WAZO_API_BIBLE_CH5_CH6.md) | Chapitres 5 & 6 — Files d'attente ACD, groups/IVR/conférences/schedules + `wazo-provd` (terminaux, templates autoprov). |
| 4 | [WAZO_API_BIBLE_CH7.md](./WAZO_API_BIBLE_CH7.md) | Chapitre 7 — CTI temps réel, `wazo-calld`, WebSockets, DND, forwards. |
| 5 | [WAZO_API_BIBLE_CH8.md](./WAZO_API_BIBLE_CH8.md) | Chapitre 8 — CDR (`wazo-call-logd`), statistiques Asterisk, webhooks (`wazo-webhookd`). |
| 6 | [WAZO_API_BIBLE_CH9_ARI.md](./WAZO_API_BIBLE_CH9_ARI.md) | Chapitre 9 — ARI (Asterisk REST Interface) sur Wazo 26.06, 30+ pièges, scripts de récupération. |
| | **COOKBOOK — Recettes prêtes à l'emploi** | |
| 7 | [WAZO_COOKBOOK_PART1.md](./WAZO_COOKBOOK_PART1.md) | Provisioning Core : utilisateur complet (8 étapes), suppression inverse, import/export CSV. |
| 8 | [WAZO_COOKBOOK_PART2.md](./WAZO_COOKBOOK_PART2.md) | Terminaux : provisioning Yealink (7 étapes), trunks SIP/IAX, transports TLS, MOH. |
| 9 | [WAZO_COOKBOOK_PART3.md](./WAZO_COOKBOOK_PART3.md) | Services avancés : ACD complet (9 étapes), ring groups, paging, conférences, filtrage DND, schedules. |
| 10 | [WAZO_COOKBOOK_PART4.md](./WAZO_COOKBOOK_PART4.md) | Temps réel : transferts, conférences audio/WebRTC, présence (`wazo-presenced`), messagerie (`wazo-chatd`), ARI. |
| 11 | [WAZO_COOKBOOK_PART5.md](./WAZO_COOKBOOK_PART5.md) | Sécurité : rotation tokens, LDAP/AD, SAML/Azure AD, Google OAuth, webhooks signés HMAC. |

## Démarrage rapide

1. **Authentification** — obtenez un token `wazo_user` :

   ```http
   POST https://wazo.example.com/api/auth/0.1/token
   Content-Type: application/json

   {"backend": "wazo_user", "expiration": 3600, "username": "admin", "password": "***"}
   ```

2. **Premier utilisateur** — suivez [COOKBOOK PARTIE 1](./WAZO_COOKBOOK_PART1.md) §1.1.
3. **Piégeage de l'API ARI** — référez-vous au [chapitre 9](./WAZO_API_BIBLE_CH9_ARI.md) avant tout script d'automatisation téléphonique.

## Conventions des exemples

| Élément | Convention |
|---------|-----------|
| `***` | Placeholder masquant un secret (token, mot de passe, refresh_token). |
| `{tenant_uuid}` | UUID v4 du tenant cible (`6118e18b-17e2-49ef-a59c-0759063b9548`). |
| `https://wazo.example.com` | À remplacer par le hostname de l'instance. |
| `-k` (curl) | Tolère le certificat auto-signé Wazo par défaut. |
| `-H "X-Auth-Token: ***" \\` | Continuer la commande cURL sur la ligne suivante. |

## Qualité du dépôt

La qualité Markdown est contrôlée automatiquement par `markdownlint-cli2`. La configuration vit dans [`.markdownlint-cli2.yaml`](./.markdownlint-cli2.yaml) :

- Règles désactivées par tolérance : `MD013` (longueur de ligne pour commandes/API), `MD036` (emphase finale), `MD040` (langage de bloc non typé pour ASCII), `MD060` (style de colonnes), `MD024` entre siblings uniquement.
- Règles strictes : `MD025` (un seul `#`), `MD041` (premier titre = H1), clôtures de blocs, structure des tableaux.

Pour valider localement :

```bash
npx --yes markdownlint-cli2 '**/*.md'
```

Sortie attendue :

```text
Summary: 0 issues in 0 files
```

## Structure du dépôt

```text
.
├── README.md                          ← ce fichier
├── .markdownlint-cli2.yaml            ← configuration du lint
├── WAZO_API_BIBLE_CH1_CH2.md          ← Bible API (9 chapitres au total)
├── WAZO_API_BIBLE_CH3_CH4.md
├── WAZO_API_BIBLE_CH5_CH6.md
├── WAZO_API_BIBLE_CH7.md
├── WAZO_API_BIBLE_CH8.md
├── WAZO_API_BIBLE_CH9_ARI.md
├── WAZO_COOKBOOK_PART1.md             ← Cookbook (5 parties de recettes)
├── WAZO_COOKBOOK_PART2.md
├── WAZO_COOKBOOK_PART3.md
├── WAZO_COOKBOOK_PART4.md
└── WAZO_COOKBOOK_PART5.md
```

## Contributions

1. Fork / branche à partir de `main` (`feat/<topic>`, `fix/<topic>` ou `docs/<topic>`).
2. Respecter le formatage (`markdownlint-cli2 '**/*.md'` doit rester vert).
3. Conventional Commits pour les messages : `feat(scope): …`, `fix: …`, `docs: …`.
4. PR vers `main` — squash merge activé.

## Licence

Documentation interne — diffusion restreinte au détenteur du dépôt.
Les exemples cURL, payloads JSON et scripts Python sont livrés à titre didactique ; à adapter et à tester avant tout déploiement en production.
