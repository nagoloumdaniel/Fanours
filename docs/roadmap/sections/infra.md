## Infrastructure, CI/CD, observabilite et exploitation

Périmètre : cette section couvre l'exploitabilité technique du produit décrit au cahier des charges Fanours v1.0 (§1 à §29). Elle s'applique aux exigences §5.1 (pile), §14 (e-mails), §19.5/§19.8 (secrets, sauvegardes), §21.4 (intégrations), §22.3 (PWA), §23 (performance/disponibilité/observabilité), §24.5/§24.6 (tests, CI), §25 (déploiement, environnements, sauvegardes), §26.6 (qualité technique), §28.1/§28.2 (risques et décisions à confirmer).

---

## 1. Environnements

### 1.1 Les quatre environnements

| Attribut | `local` (dev) | `test` (CI) | `staging` | `production` |
|---|---|---|---|---|
| Isolement réseau | localhost + conteneurs Docker, aucun port ouvert hors 127.0.0.1 | Runner GitHub éphémère (IP non prévisible), réseau isolé, pas d'ingress | VPC privé + ingress restreint par allowlist IP (bureau, CI) | VPC privé, ingress public 443 (CDN/WAF), 22/3389/5432/6379 non exposés |
| Base de données | PostgreSQL 16 en conteneur local, `docker compose up`, volume éphémère | Service container `postgres:16` jetable par job, recréé à chaque run | PostgreSQL managé, instance dédiée, accès via bastion | PostgreSQL managé (ou self-hosted + standby, §3.4), multi-AZ, PITR 35 j |
| Données | Seeds synthétiques (`pnpm db:seed --synthetic`) | Seeds synthétiques déterministes + fixtures figées | Copie **anonymisée** de la prod (scrub PII) ou jeux de données de recette réels non sensible | Données réelles |
| Cache | Redis local en conteneur, base 0 | Redis service container, base 0, TTL courts | Redis managé, données jetables | Redis managé, persistance activée mais **non critique** (§9.4) |
| Secrets | `.env.local` non commité, généré par `pnpm secrets:gen` (valeurs factices) | Secrets GitHub Actions (environments secrets) injectés à la volée, jamais dans les logs | Vault/KMS dédié, rotation 30 j, accès restreint | Vault/KMS, rotation 90 j, accès break-glass journalisé |
| Intégrations paiement | Sandbox/mock local (aucun appel réseau sortant) | Stripe test clock + mock serveur webhook | Clés **sandbox** du PSP retenu | Clés **live** du PSP retenu (§10.1, §28.2) |
| Intégrations e-mail | Mailpit local (capture) | Mailpit en service container | Fournisseur en mode sandbox, domaine `staging.fanours.*` (SPF/DKIM/DMARC séparés) | Fournisseur en production, domaine d'envoi principal (§12.1) |
| Transporteurs / tracking | Fixtures API mockées | Mock serveur (contrats JSON figés) | Sandbox transporteur réelle (YunExpress/4PX selon sélection) | API transporteur réelle |
| Objet S3 | MinIO local (bucket `fanours-local`) | MinIO éphémère, upload/download réels | Bucket S3 `fanours-staging` (clés distinctes) | Bucket S3 `fanours-prod` + `fanours-prod-documents` (privé) |
| Feature flags | Toutes off sauf `dev` | Toutes off sauf `test` | Flags prod à 0 % par défaut, activables | Flags prod réels, audit des changements (§14) |
| Accès | Développeurs sur leur poste | Équipe via GitHub Actions (pas d'accès direct) | Équipe produit/dev/QA via VPN + SSO + MFA | Clients via internet ; staff via **zero-trust + MFA + allowlist IP** (§3.6) |
| Données carte / PII | **Interdites** | **Interdites** | **Interdites** (jamais de PAN, jamais de document d'identité réel) | Autorisées, minimisées, chiffrées, non journalisées |

### 1.2 Règle absolue sur les données sensibles (§25.1, §19.6)

- Aucun PAN, CVV, donnée carte tokenisée, secret PSP, mot de passe, document d'identité, justificatif d'entreprise réel ou donnée de client n'entre en `local`, `test` ou `staging`.
- Aucune extraction de base de production vers un environnement non-production. Si un jeu de données réel est indispensable (recette contractuelle), il est **anonymisé par script versionné** (`pnpm db:anonymize`) : pseudonymisation des `email`/`name`/`phone`/`address`, suppression des `payment_*` sensibles, des documents joints et des sessions ; le script produit un rapport du nombre de lignes pseudonymisées, archivé comme artefact CI.
- Les fixtures de test utilisent des numéros de carte de test Lifetime/Sandbox (ex. `4242 4242 4242 4242`) générés par le PSP en mode test uniquement.
- Le contrôle automatique est bloquant : job `scan-pii` qui interdit la présence de motifs PAN (`\b(?:4\d{3}|5[1-5]\d{2}|3[47]\d{2}|6011)[ -]?\d{4}[ -]?\d{4}[ -]?\d{4}\b`) et de clés privées dans les seeds et fixtures non-production.

### 1.3 Matrice « ce qui change entre environnements »

```yaml
# config/env-matrix.ts (extrait) — une seule source de vérité, aucun `if (env === 'prod')` dispersé
environments:
  local:
    APP_ENV: local
    DATABASE_URL: postgresql://fanours:fanours@localhost:5432/fanours
    REDIS_URL: redis://localhost:6379/0
    S3_ENDPOINT: http://localhost:9000
    S3_BUCKET: fanours-local
    PAYMENT_PROVIDER: mock
    EMAIL_PROVIDER: mailpit
    CARRIER_PROVIDER: mock
    SENTRY_DSN: ""
    OTEL_EXPORTER_OTLP_ENDPOINT: ""
    FEATURE_FLAG_DEFAULTS: all-off
  test:
    APP_ENV: test
    DATABASE_URL: postgres://postgres:postgres@postgres:5432/fanours_test   # service container
    REDIS_URL: redis://redis:6379/0
    S3_ENDPOINT: http://minio:9000
    S3_BUCKET: fanours-test
    PAYMENT_PROVIDER: stripe-test
    EMAIL_PROVIDER: mailpit
    CARRIER_PROVIDER: mock
    SENTRY_DSN: ""            # aucune trace envoyée en CI
    FEATURE_FLAG_DEFAULTS: all-off
  staging:
    APP_ENV: staging
    DATABASE_URL: ${DATABASE_URL}          # secret
    REDIS_URL: ${REDIS_URL}
    S3_BUCKET: fanours-staging
    PAYMENT_PROVIDER: stripe-sandbox
    EMAIL_PROVIDER: smtp-sandbox
    CARRIER_PROVIDER: carrier-sandbox
    SENTRY_DSN: ${SENTRY_DSN}
    FEATURE_FLAG_DEFAULTS: prod-at-0
  production:
    APP_ENV: production
    DATABASE_URL: ${DATABASE_URL}
    REDIS_URL: ${REDIS_URL}
    S3_BUCKET: fanours-prod
    S3_BUCKET_PRIVATE: fanours-prod-documents
    PAYMENT_PROVIDER: ${PAYMENT_PROVIDER_LIVE}
    EMAIL_PROVIDER: sendgrid-live
    CARRIER_PROVIDER: carrier-live
    SENTRY_DSN: ${SENTRY_DSN}
    FEATURE_FLAG_DEFAULTS: prod
```

Règles de dérivation automatiques : `APP_ENV` détermine le niveau de log (`debug`/`info`), le sampling des traces (100 % en dev, 25 % en staging, 10 % + 100 % sur erreur en prod), la durée de vie des cookies (`lax` en local, `strict`+`secure` ailleurs), et l'activation du `Strict-Transport-Security` (uniquement en production).

---

## 2. Strategie de deploiement

### 2.1 Comparatif des deux options

| Critère | (a) Vercel (frontend) + service managé (API/worker) | (b) Tout conteneur sur VPS / Cloud Run / ECS |
|---|---|---|
| PWA + service worker | Deploy preview et redirections Let's Encrypt gérés, mais `headers`/`service-worker-allowed`/`Cache-Control` fightent avec le cache CDN Vercel ; le SW est régénéré à chaque build et peut être servi en `no-store` selon le plan | Contrôle total des en-têtes `Service-Worker-Allowed`, `Cache-Control`, version du SW et stratégie `skipWaiting` ; indispensable pour le §22.3 |
| Worker asynchrone long-running | Vercel Fluid compute ne convient pas : pas de worker permanent, pas de cron fiable, timeout de fonction ; Vercel Cron = 1 job/plan, coarsier | Conteneur dédié `worker` avec supervisor, cron interne, consommation de la file Redis/BullMQ, scale horizontal sans effort |
| Webhooks paiement (URL publique stable) | URL de production stable oui, mais preview URLs variables et quotas de requêtes ; signature et IP pinning contraints | `https://api.fanours.com/webhooks/psp` unique, stable, DNS pilotable, allowlist IP du PSP, WAF et rate limit dédiés |
| Latence | Edge network worldwide excellent pour le CDN ; l'API reste dans une région → 2 allers-retours pour le SSR authenticated | Edge (Cloudflare) + région Europe de référence (proche majority buyers) : 1 aller-retour si SSR et API co-localisés |
| Coût | Prévisionnel à l'usage (bande passante, fonctions, « build minutes ») difficile à plafonner ; sortie de scale non linéaire | Coût fixe et prévisible (2 VM + CDN + BDD managé + stockage) ; le worker ne facture pas de « démarrage à froid » |
| Cold starts | Lambda/Edge : 100 ms à 1 s sur API, dégrade le checkout | Conteneurs longs : 0 ; à scaling horizontal, min-instances garantis pour l'API |
| Observabilité | Logs Vercel + APM externe, métriques fournisseur à corréler, coût par ingestion | Stack unifiée (Vector → Loki, Prometheus → Grafana, OTel, Sentry) avec correlation_id unique de bout en bout (§7.3) |
| PII / région des données | Runtimeregions choisies, mais dépendances providers à cartographier | Maîtrise totale : données BDD, backups et logs dans l'UE ; accords de transfert (§19.6) explicites |
| Surface d'attaque | Deux surfaces (edge + service) à sécuriser | Une surface (reverse proxy + réseau privé) |

### 2.2 Recommandation

**Option (b) — conteneurs sur un VPS unique multi-rôles, avec CDN/WAF externe, extrapolation vers un orchestrateur managé quand le trafic le justifie.**

Découpage retenu (compatible monolithe modulaire §5.2, sans microservices prématurés §5.4) :

- `web` : Next.js 15 (App Router, SSR + ISR) — §5.1.
- `api` : backend TypeScript structuré (NestJS ou équivalent, §5.1) exposant `/api/*` et `/webhooks/*`.
- `worker` : même image que `api` (build monolithique unifié), launched en mode `WORKER_ROLE=consumer`, partageant le code métier.

Justification opérationnelle :

1. Le **PWA** (installable, §22.3) impose un contrôle fin du service worker que les CDNs managés gèrent mal.
2. Le **worker** est structurellement long-running (§14.4 envoi asynchrone, §21.3 outbox, §11.1 architecture du worker) : un modèle serverless le rend fragile.
3. Les **webhooks PSP** exigent une URL publique stable, unique et durcie : le DNS du domaine appartient à Fanours, pas au fournisseur d'hébergement.
4. Le **coût** d'un VPS double-2 + CDN + BDD managé est predictable et inférieur à une facture à l'usage, et permet de fixer un budget mensuel (§16).
5. La **région des données** est pilotable : BDD, backups, logs et traces en Union européenne ; décision à confirmer (§28.2 point 10) et compatible avec RGPD/UK GDPR (§19.6).

### 2.3 Conséquences opérationnelles de ce choix

- Fanours devient responsable de : mise à jour OS, durcissement, sauvegardes applicatives, supervision de l'hôte, patching sécurité, rotation des certificats (Let's Encrypt via `acme.sh` + renouvellement automatique + alerte J-14 et J-7).
- Mitigations obligatoires : image Debian durcie, `unattended-upgrades`, fail2ban, `ufw` avec `default deny`, SSH par clé uniquement + bastion, sauvegardes testées mensuellement (§9.6), double VM applicative derrière le reverse proxy, runbooks d'incident.
- Risque single point of failure atténué : `api` et `web` tournent sur 2 VM (N+1), le trafic bascule par health check du reverse proxy.

### 2.4 Plan de sortie si la recommandation est invalidée (coût, capacité, équipe)

| Déclencheur de sortie | Passage |
|---|---|
| Charge > 60 % CPU soutenu 7 j sur `api` **et** besoin de HPA | Migrer `api`/`worker` vers **Cloud Run** (jobs Cloud Scheduler pour les crons) ou **ECS Fargate** ; conserver `web` sur VM ou VPS managé ; le reverse proxy reste le point d'entrée (lb interne) |
| Besoin de mise à jour zéro downtime avec rollback par version | Passer à ECS/Cloud Run avec *deployment circuit breaker* + *traffic shifting* (canary 5/25/100 natif) |
| Équipe < 2 personnes en ops, SLA de maintenance < 2 h/mois | Externaliser l'hébergement (managed VPS + panel) ou basculer `web` sur Vercel et garder `api`/`worker` en conteneurs managés — **jamais** le worker sur serverless |
| Coût BDD managé > budget | Migrer vers PostgreSQL self-hosted avec `pgBackRest` + standby streamé + WAL vers S3 (§9.2) |

Règle : le passage de l'option (b) vers l'option (a) est interdit tant que le worker et les webhooks n'ont pas de cible long-running compatible. Le code doit rester agnostique : images OCI construites par `docker buildx` sans dépendance au runtime cible.

---

## 3. Topologie de production

### 3.1 Schéma textuel

```
                         Internet
                             |
        +-------------------v--------------------+
        |  Cloudflare (CDN + WAF + DDoS)          |   CDN cache assets publics + HTML SSR
        |  - TLS flexible->full (strict)          |   cache-control par route (§10.2)
        |  - IP allowlist / Zero Trust           |
        +-------------------+--------------------+
                            | HTTPS 443 (public)
        +-------------------v--------------------+
        |  Traefik / reverse proxy (VM edge-1,    |   TLS interne Let's Encrypt + HSTS
        |  edge-2)                               |   rate limit, WAF rules, canary weights
        +----+----------------+-------------------+
             |                |                |
   /  web    |        /  api   |        /  webhooks
             v                v                v
  +----------+     +----------+     +----------+     [privé, 10.0.x]
  | Next.js  |     | API TS   |     | API TS   |
  | (SSR/ISR) |     | (NestJS) |     | (webhook)|
  +----------+     +----------+     +----------+
             |                |                |
             |                v                v
             |        +-------+--------+  +---------------+
             |        | Redis 7        |  | BullMQ queues  |  (privé)
             |        | (cache+files)  |  | + DLQ         |
             |        +-------+--------+  +-------+-------+
             |                |                   |
             |                |                   v
             |                |           +-------------------+
             |                |           | worker (consumer)  |  cron interne
             |                |           | + scheduler       |
             |                |           +-------------------+
             |                v
             |     +----------+----------+
             |     | PostgreSQL 16       |  (privé, multi-AZ ou standby)
             |     | pooler (PgBouncer)  |  port 5432 non public
             |     +---------------------+
             |                |
             |     +----------v----------+
             |     | S3 compatible       |  bucket public (assets) via CDN
             |     | (MinIO/Backblaze)   |  bucket privé (factures, devis) presign 5 min
             |     +---------------------+
             |
             +--> externes (egress allowlisté) :
                  PSP paiement, e-mail transactionnel, transporteur,
                  API transport/taux de change, Object Storage SDK
```

### 3.2 Exposé publiquement vs privé

| Composant | Exposition | Contrôle |
|---|---|---|
| CDN / WAF | Public 443 | TLS 1.2+, HSTS, WAF managed, rate limit, bot fight, `/_next/static` et `/assets` cachés |
| `web` (Next.js) | Privé derrière le proxy | Non joignable directement ; health check interne |
| `api` | Privé ; `/webhooks/*` public via proxy avec allowlist IP du PSP + signature HMAC | Rate limit strict, body size cap 256 Ko |
| `worker` | Aucun port public | Sortie réseau sortante uniquement |
| PostgreSQL | Privé (RFC1918) | SSL client, `pg_hba.conf` restrictif, PgBouncer |
| Redis | Privé | `requirepass` rotatif, `rename-command` désactivé, pas d'accès depuis le web |
| S3 public | Public **en lecture seule** via CDN uniquement | Aucune écriture depuis le web ; clés API en lecture seule côté web |
| S3 privé (factures, devis, pièces) | Privé | Presigned URL 5 min, un objet par accès journalisé |
| Fournisseur e-mail / PSP / transporteurs | Sortie HTTPS sortante | Allowlist des domaines, pas d'entrée entrante |
| Monitoring / dashboards | Privé | Accès via VPN ou zero-trust, jamais public |
| Backups | Hors site, compte à accès séparé | Chiffrement, MFA, pas de réseau public |

### 3.3 Protection du dashboard administrateur

- Le dashboard admin n'est **pas** distingué du site public par l'obscurité de l'URL (§21.2) : l'accès est contrôlé côté serveur par RBAC + MFA obligatoire (§13.3, §16.1).
- Le chemin `/admin/*` est protégé en plus par le WAF : accessible uniquement aux IP du bureau et des VPN de l'équipe (allowlist), et MFA + reauthentification pour les actions financières (§19.2).
- Accès opérateur : **zero-trust** (Cloudflare Access / Tailscale) avec SSO, MFA, et durées de session courtes (8 h) ; le VPN de secours est un bastion SSH avec clé + journalisation d'audit.
- Les comptes de service d'infrastructure ont des droits minimaux (§19.5) et n'utilisent jamais de compte humain partagé (§16.1).

### 3.4 Décision base de données production

**PostgreSQL managé multi-AZ en phase 0/1**, pour acheter de la disponibilité et du PITR sans competence ops supplémentaire, avec conditions de sortie vers self-hosted + standby streamé à partir de ~400 Go de données ou 5 000 writes/s (coût BDD managé > 25 % du budget infra, §16). La décision est journalisée en ADR `docs/adr/ADR-002-database-topology.md` car elle conditionne le RPO/RTO (§19.8).

---

## 4. Pipeline CI (GitHub Actions)

### 4.1 Arborescence des fichiers

```
.github/
  workflows/
    ci.yml                  # pipeline principal (PR + main + tag)
    e2e.yml                 # E2E Playwright sharded (PR + main)
    release.yml             # build reproductible + SBOM + signature + digest
    security.yml            # SAST/DAST/secret scan planifié (nuit)
    dependency-review.yml   # Dependabot/Renovate + revue de dépendances
    ops.yml                 # sauvegarde, restauration, rotation des certificats
  actions/
    setup/action.yml        # pnpm + node + cache
    compose/action.yml      # stack Postgres/Redis/MinIO pour l'intégration
  CODEOWNERS
  dependabot.yml
  renovate.json
  actionslint.yaml
```

### 4.2 Jobs du pipeline principal et conditions

| Job | Condition | Timeout | Cache | Artefacts (rétention) |
|---|---|---|---|---|
| `lint` | toujours | 5 min | pnpm store | rapport eslint (7 j) |
| `format` | toujours | 5 min | pnpm store | — |
| `typecheck` | toujours | 10 min | pnpm store, turbo | — |
| `test-unit` | toujours | 15 min | pnpm store, turbo | `coverage/` (14 j), `junit.xml` (14 j) |
| `test-integration` | toujours | 20 min | pnpm store, turbo | `junit-integration.xml` (14 j) |
| `scan-secrets` | toujours | 5 min | — | `gitleaks-report.json` (30 j) |
| `scan-deps` | toujours | 10 min | pnpm store | `osv-scanner.json` (30 j) |
| `sast` (CodeQL) | toujours | 20 min | pnpm store | alertes CodeQL (90 j) |
| `build` | `needs` verts | 20 min | pnpm store, turbo, buildx | image OCI + digest, SBOM (SPDX) (90 j) |
| `preview` | `pull_request` | 10 min | — | URL preview (3 j) |
| `smoke` | `preview` OK | 10 min | — | rapport smoke (7 j) |
| `deploy-staging` | `push` sur `main` | 15 min | — | digest déployé (30 j) |
| `deploy-production` | tag `v*` **ou** `main` + flag `deploy_prod`, gated par approbation manuelle `production` | 20 min | — | rapport de déploiement (1 an) |
| `post-deploy` | après prod | 15 min | — | métriques post-déploiement (90 j) |

Règles transverses : `concurrency` par branche (annulation des runs obsolètes), `timeout-minutes` sur **chaque** job, `permissions: contents: read` par défaut et `id-token: write` uniquement pour le build (OIDC vers le registre, pas de secret long-lived), `defaults.run.shell: bash`, cache `~/.pnpm-store` avec clé `pnpm-lock.yaml`, jobs non critiques en `continue-on-error: false` mais `fail-fast: false` au niveau de la matrice.

### 4.3 Workflow principal `ci.yml` (jobs structurants)

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]
    tags: ['v*']
  workflow_dispatch:

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

env:
  NODE_VERSION: '22.x'
  PNPM_VERSION: '9.15.0'

defaults:
  run:
    shell: bash

jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: ./.github/actions/setup
      - run: pnpm exec eslint . --max-warnings=0 --format=json --output-file=eslint.json
      - run: pnpm exec prettier --check . 
      - if: always()
        uses: actions/upload-artifact@v4
        with: { name: lint-report, path: eslint.json, retention-days: 7 }

  format:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: ./.github/actions/setup
      - run: pnpm run format:check

  typecheck:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: ./.github/actions/setup
      - run: pnpm run typecheck   # tsc --noEmit, tsconfig strict:true (§24.6)

  test-unit:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    strategy:
      fail-fast: false
      matrix: { shard: [1, 2] }
    steps:
      - uses: ./.github/actions/setup
      - run: pnpm vitest run --coverage --shard=${{ matrix.shard }}/2
      - uses: actions/upload-artifact@v4
        with: { name: coverage, path: coverage/, retention-days: 14 }

  test-integration:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: fanours_test
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 10s --health-timeout 5s --health-retries 10
        ports: ['5432:5432']
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
        options: >-
          --health-cmd "redis-cli ping" --health-interval 10s --health-timeout 5s --health-retries 10
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/fanours_test
      REDIS_URL: redis://localhost:6379/0
    steps:
      - uses: ./.github/actions/setup
      - run: pnpm db:migrate:deploy      # migrations réellement appliquées (§25.3)
      - run: pnpm db:seed:synthetic
      - run: pnpm vitest run --project integration --coverage=false
      - if: always()
        uses: actions/upload-artifact@v4
        with: { name: junit-integration, path: reports/, retention-days: 14 }
      - if: always()
        run: docker compose -f docker/compose.test.yml down -v   # teardown, volumes purgés

  scan-secrets:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2
        env: { GITLEAKS_ENABLE_UPLOAD_ARTIFACT: 'true' }
      - name: Détection de PAN dans les seeds/fixtures
        run: |
          ! rg -n --pcre2 '\b(?:4\d{3}|5[1-5]\d{2}|3[47]\d{2}|6011)[ -]?\d{4}[ -]?\d{4}[ -]?\d{4}\b' \
               --glob '!pnpm-lock.yaml' .
      - uses: actions/upload-artifact@v4
        with: { name: gitleaks-report, path: gitleaks-report.json, retention-days: 30 }

  scan-deps:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: ./.github/actions/setup
      - run: pnpm audit --audit-level high
      - uses: google/osv-scanner-action/osv-scanner-action@v2
        with: { scan-args: '--lockfile=pnpm-lock.yaml' }
      - uses: actions/upload-artifact@v4
        with: { name: osv-report, path: osv-scanner-results.json, retention-days: 30 }

  sast:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with: { languages: typescript, queries: security-extended }
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
        with: { category: '/language:typescript' }

  build:
    needs: [lint, format, typecheck, test-unit, test-integration, scan-secrets, scan-deps, sast]
    runs-on: ubuntu-latest
    timeout-minutes: 20
    outputs:
      digest: ${{ steps.push.outputs.digest }}
      tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: ./.github/actions/setup
      - id: meta
        run: echo "tag=$(git rev-parse --short=12 ${{ github.sha }})" >> "$GITHUB_OUTPUT"
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: registry.fanours.com/fanours:${{ steps.meta.outputs.tag }}
          provenance: mode=max
          sbom: true
          outputs: type=image,name=registry.fanours.com/fanours,push-by-digest=true,name-canonical=true
      - id: push
        run: |
          digest=$(docker buildx imagetools inspect registry.fanours.com/fanours:latest --format '{{json .Manifest.Digest}}')
          echo "digest=$digest" >> "$GITHUB_OUTPUT"
      - uses: actions/upload-artifact@v4
        with: { name: sbom, path: '**/*.sbom.json', retention-days: 90 }
      - uses: anchore/sbom-action@v0
        with: { path: '**/*.sbom.json', format: spdx-json, output-file: sbom.spdx.json }

  preview:
    needs: build
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - run: |
          docker run -d --name preview -p 3000:3000 \
            -e APP_ENV=staging -e DATABASE_URL=${{ secrets.STAGING_DATABASE_URL }} \
            registry.fanours.com/fanours@${{ needs.build.outputs.digest }}
      - run: |
          for i in $(seq 1 30); do curl -fsS http://localhost:3000/healthz && break || sleep 2; done
      - uses: actions/upload-artifact@v4
        with: { name: preview-log, path: /tmp/preview.log, retention-days: 3 }

  smoke:
    needs: preview
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - run: pnpm exec playwright test --project=functional --reporter=line

  deploy-staging:
    needs: [build, smoke]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    timeout-minutes: 15
    environment: staging
    steps:
      - run: ./scripts/deploy.sh staging ${{ needs.build.outputs.digest }}   # canary 5/25/100 (§5)

  deploy-production:
    needs: [build, smoke]
    if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/v'))
    runs-on: ubuntu-latest
    timeout-minutes: 20
    environment:
      name: production
      url: https://fanours.com
    steps:
      - name: Approbation manuelle
        uses: trstringer/manual-approval@v1
        with: { request: 'Déploiement production — digest ${{ needs.build.outputs.digest }}' }
      - run: ./scripts/deploy.sh production ${{ needs.build.outputs.digest }}
      - run: ./scripts/smoke-production.sh

  post-deploy:
    needs: deploy-production
    if: always() && needs.deploy-production.result == 'success'
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - run: pnpm exec tsx scripts/verify-release.ts --digest=${{ needs.build.outputs.digest }}
```

### 4.4 Workflow E2E `e2e.yml` (matrice + sharding)

```yaml
name: E2E
on:
  pull_request:
  push:
    branches: [main]
concurrency:
  group: e2e-${{ github.ref }}
  cancel-in-progress: true
jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 12
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4, 5]
    services:
      postgres: { image: postgres:16-alpine, env: { POSTGRES_PASSWORD: postgres, POSTGRES_DB: fanours_test } }
      redis: { image: redis:7-alpine }
    steps:
      - uses: ./.github/actions/setup
      - uses: actions/cache@v4
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}
      - run: pnpm exec playwright install --with-deps chromium
      - run: pnpm db:migrate:deploy && pnpm db:seed:synthetic
      - run: pnpm exec playwright test --shard=${{ matrix.shard }}/5
      - if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report-shard-${{ matrix.shard }}
          path: | playwright-report/ test-results/
          retention-days: 14
      - uses: actions/upload-artifact@v4
        with: { name: trace, path: test-results/**/trace.zip, retention-days: 14 }
```

`playwright.config.ts` impose : `retries: process.env.CI ? 2 : 0`, `workers: CI ? 4 : undefined`, `use.trace: 'on-first-retry'`, `video: 'retain-on-failure'`, `expect.timeout: 10_000`. Un job `e2e-flake-detector` (`.github/workflows/e2e.yml`, job `stability`) rejoue les tests échoués 5 fois et ouvre une issue si le taux de flake > 2 % sur 7 jours ; un job `e2e-blocked-views` échoue si `pageerror` ou `console.error` non attendus sont détectés sur les parcours (critère §26.6 « erreurs majeures de console »).

### 4.5 Règles de qualité et gates de release (§24.6, §26.6, Annexe C)

- Couverture : seuil global 70 % (lignes) mais **80 %** sur `src/modules/payments`, `src/modules/trade-compliance`, `src/modules/pricing` (§24.6 : seuils plus élevés pour logique financière et douanière) ; la couverture est un indicateur, pas une preuve.
- Un déploiement est bloqué si : check CI obligatoire en échec, vulnérabilité critique non résolue sans acceptation formelle écrite, E2E critique en échec, scan de secrets en échec, `typecheck` strict en échec, écart de couverture des modules critiques.
- `dependency-review` bloque toute dépendance introduisant une licence non approuvée ou une CVE critique.
- Analyse de configuration : `checkov`/`trivy config` sur les Dockerfiles et manifests, `actionlint` sur les workflows, `gitleaks protect --staged` en pre-commit.
- Le déploiement production est **manuellement approuvé** (environment `production` avec reviewers obligatoires) ; le tag `v*` déclenche le pipeline de release, `main` seul ne déploie pas en production sans le flag `deploy_prod` positionné par un mainteneur.

---

## 5. Deploiement progressif et rollback

### 5.1 Strategie retenue : canary pondere par le reverse proxy, et non blue/green strict

Blue/green exige un doublement permanent de la capacité et une bascule instantanée qui masque les régressions lentes (erreurs qui n'apparaissent qu'après 5–10 min de trafic réel). Le fanours est un site transactionnel où une régression de checkout ou de paiement se traduit immédiatement en perte de chiffre d'affaires. Décision : **canary 5 % → 25 % → 100 %**, avec bascule manuelle possible et rollback automatique.

| Étape | Part du trafic | soak minimum | Critère de passage |
|---|---|---|---|
| C0 | 0 % → déploiement, canary isolé (une instance) | 2 min | health check vert, migrations non destructives appliquées |
| C1 | 5 % | 10 min | 5xx < 1 %, p95 < 800 ms, aucun 4xx de checkout anormal, paiement OK |
| C2 | 25 % | 15 min | idem + `payment_failure_rate` < 1,5 % et stable vs baseline 24 h |
| C3 | 100 % | 30 min | idem + file d'attente worker stable, DLQ vide |
| Retrait de l'ancien | — | après 30 min de soak | ancien conteneur arrêté puis supprimé |

### 5.2 Procede concret (provider retenu : reverse proxy sur VM + image OCI par digest)

1. `deploy.sh production <digest>` récupère l'image **par digest** (jamais par tag mutable) dans le registre interne.
2. Les migrations sont appliquées **avant** le déploiement du code, en mode expand/contract (§5.5) : la version antérieure du code reste fonctionnelle pendant toute la fenêtre.
3. Déploiement de l'instance canary `web-canary-<sha>` / `api-canary-<sha>` sur le pool `canary`.
4. Le reverse proxy route 5 % du trafic (pondération par `X-Request-ID` hashé pour stabilité de session : un utilisateur ne bascule pas de version au milieu de son checkout) vers le pool canary.
5. Sonde de santé : `GET /healthz` (process vivant) + `GET /readyz` (dépendance critiques OK : DB, Redis, PSP joignable) + `GET /healthz?deep=1` (latence DB < 500 ms, file worker accessible).
6. Métriques de garde lues depuis Prometheus sur une fenêtre glissante de 5 min, comparées à la baseline des 24 h précédentes.
7. Promotion 5 → 25 → 100 % par commande explicite du pipeline ; chaque étape peut être interrompue automatiquement.
8. Rollback : à tout moment, `deploy.sh rollback` renvoie 100 % du trafic vers le pool précédent (le conteneur de la version précédente est conservé **pendant 30 jours**, ce qui permet un rollback immédiat, pas un rebuild).

### 5.3 Metriques de garde et seuils de rollback automatique

| Garde | Seuil | Fenêtre | Action si dépassé |
|---|---|---|---|
| Taux d'erreur 5xx (canary) | > 1 % ou > 3× la baseline | 5 min | rollback automatique 100 % → pool précédent |
| Latence p95 (canary) | > 800 ms checkout, > 1200 ms catalogue | 5 min | rollback automatique |
| Taux d'échec de paiement | > 1,5 % ou > 2× la baseline 24 h | 10 min | rollback + alerte SEV1 |
| Taux d'échec de réservation de stock | > 0,5 % | 5 min | rollback + alerte SEV2 |
| Profondeur de queue worker | > 500 jobs et âge du plus vieux > 15 min | 5 min | blocage de la promotion (pas rollback) |
| DLQ non vide après déploiement | > 0 | immédiat | rollback + alerte SEV1 |
| Webhooks échoués | > 5 % du flux sur 10 min | 10 min | rollback + alerte SEV1 |
| Santé applicative | `/readyz` en échec | 3 checks consécutifs | rollback |

### 5.4 Procedure de rollback automatique

Le pipeline `deploy.sh` maintient, pendant la fenêtre de déploiement, une boucle de garde (30 s) qui interroge Prometheus. Au dépassement d'un seuil de rollback, le script exécute automatiquement :

```bash
#!/usr/bin/env bash
set -euo pipefail
# rollback.sh <previous_pool_tag>
set -euo pipefail
PREV_POOL="${1:?pool précédent requis}"
log "Guard breach — rollback vers ${PREV_POOL}"
docker compose -f docker/compose.prod.yml up -d --no-deps \
  "web@${PREV_POOL}" "api@${PREV_POOL}"
# poids de trafic remis à 0 pour le canary
curl -fsS -X POST "http://127.0.0.1:8080/api/traffic" \
  -H 'Content-Type: application/json' \
  -d '{"canary_weight":0}'
# garde : vérifier que le trafic est nominal
sleep 30
if ! curl -fsS "https://api.fanours.com/healthz" >/dev/null; then
  log "Rollback incomplet — escalade SEV1 on-call"
  exit 1
fi
```

Le rollback est **destructive-safe** : la base n'est jamais restaurée automatiquement (une restauration de données pendant un incident applicatif amplifie le dommage). Seule la version applicative est ramenée en arrière ; les migrations restent en place (elles sont compatibles par construction, §5.5). Un runbook de restauration de données distinct existe (§6.4) et est déclenché manuellement.

### 5.5 Strategie de migration compatible : expand / contract

Les migrations sont divisées en trois phases, jamais mélangées dans un même release (§25.3) :

1. **Expand** (release N) : ajout de colonnes/tables/indexes ; la nouvelle colonne est `NULL` ou a une valeur par défaut compatible ; les deux versions du code peuvent écrire. Exemple : renommer `orders.total` : on ajoute `orders.total_amount` et on écrit dans les deux colonnes via trigger ou dual-write applicatif.
2. **Migrate** (release N+1) : backfill en lot (script hors-ligne, par tranches de 5 000 lignes, avec pause et contrôle de charge) ; lecture depuis la nouvelle colonne ; l'ancienne version du code continue de fonctionner car elle lit encore l'ancienne colonne, alimentée par dual-write.
3. **Contract** (release N+2, minimum 7 jours après le backfill, et seulement si l'ancien code n'est plus déployé) : suppression de l'ancienne colonne. Jamais dans la même release que l'expand.

Contraintes : `ptonline`/verrou non bloquant pour les gros index (`CREATE INDEX CONCURRENTLY`), `lock_timeout = 3s` et `statement_timeout = 30s` sur toutes les migrations en production, `NOT NULL` jamais ajouté directement (passage par `CHECK ... NOT VALID` puis `VALIDATE CONSTRAINT`), interlocks exclusifs (création d'index, `ALTER TYPE`) exécutés hors heures de pointe par un job dédié.

---

## 6. Migrations en production

### 6.1 Principes

- Migrations versionnées à la main, relues par un pair, appliquées **par fichier** par le runner du projet (`pnpm db:migrate:deploy` — runner `packages/db/scripts/migrate.ts`, cœur `packages/db/src/migrate-core.ts`), jamais à la main en production (§25.3).
- Chaque migration est **append-only** : pas de fichier `down`. La procédure de retour arrière est un script de restauration validé, écrit dans la description de la PR (§3.1 du CLAUDE.md projet, expand/contract).
- Aucune migration destructive sans preuve de restauration testée (§25.3, §26.5).
- Le **SQL versionné à la main** est la source de vérité du schéma ; `drizzle-kit generate` sert uniquement à produire un diff relu, jamais `push` (§1.1 de `architecture.md`). Le checksum SHA-256 de chaque fichier est calculé par le runner au moment de l'application, jamais écrit à la main.

### 6.2 Runbook de migration (procédure pas-à-pas)

```text
PRÉ-REQUIS
  [ ] Migration relue par un pair (approbation requise sur la PR)
  [ ] Impact estimate : table lockée ? durée estimée ? volume de lignes ?
  [ ] Classement de la migration : EXPAND (survivant) ou CONTRACT (différé)
  [ ] Backfill scripté disponible et testé sur un clone

ÉTAPE 1 — SAUVEGARDE VÉRIFIÉE (OBLIGATOIRE)
  [ ] Snapshot base prod déclenché, ID noté dans le ticket
  [ ] Vérification : snapshot visible + test de restore rapide sur un clone (voir §9.6)
  [ ] WAL/PITR disponible et non expiré (fenêtre > durée de la fenêtre de déploiement)

ÉTAPE 2 — FENÊTRE ET COMMUNICATION
  [ ] Créneau annoncé (jamais pendant un pic de commandes ; éviter 00:00–06:00 UTC si trafic EU)
  [ ] Statut "maintenance" affiché sur /status si la migration dépasse 30 s
  [ ] Équipe d'astreinte notifiée dans le canal incident

ÉTAPE 3 — MIGRATION ADDITIVE
  [ ] Exécuter : psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f ./migrations/20261021_001_add_x.sql
  [ ] Surveiller : pg_stat_activity, locks, replication lag
  [ ] Si lock > 5 s ou replication lag > 30 s : Ctrl-C, vérifier, ne pas forcer

ÉTAPE 4 — VÉRIFICATION
  [ ] Structure conforme : \d+ table, index créés, contraintes VALIDATE
  [ ] Données : SELECT count(*), checksum sur l'échantillon, invariants métier
  [ ] Application : /readyz vert, logs sans ERROR, métriques OK, E2E smoke
  [ ] Backup post-migration (snapshot) + annotation du ticket

ÉTAPE 5 — MIGRATION DESTRUCTIVE (DIFFÉRÉE)
  [ ] ≥ 7 jours après la migration additive
  [ ] Preuve que l'ancien code n'est plus déployé (audit des déploiements)
  [ ] Nouveau backup vérifié AVANT
  [ ] Exécution en fenêtre maintenance annoncée
  [ ] Vérification post-exécution + snapshot immédiat

ÉTAPE 6 — CLÔTURE
  [ ] Ticket mis à jour (ID snapshot pré/post, durée, incident éventuel)
  [ ] Notification à l'équipe produit si comportement visible modifié
```

### 6.3 Règle de non-destruction

Une migration destructive (DROP COLUMN, DROP TABLE, type non réversible, conversion de données non réversible) exige : (1) un snapshot vérifié, (2) un **test de restauration exécuté et réussi sur ce snapshot** dans les 7 jours précédents, (3) l'approbation d'un responsable technique hors de l'équipe qui l'exécute, (4) le report de l'opération à une release ultérieure par défaut. Une migration destructive combinée à un changement applicatif dans la même release est interdite : la_release ne contient qu'une seule catégorie de migration.

### 6.4 Restauration de données (procédure déclenchée manuellement)

```bash
# 1. Créer une instance de restauration isolée (ne jamais restaurer en place au départ)
pgbackrest --stanza=prod --type=restore --target=immediate --pg1-path=/var/lib/postgresql/16/restore
# 2. Vérifier : count des tables critiques, dernière commande, cohérence des soldes
psql "$RESTORE_URL" -c "select count(*) from orders; select count(*) from payments;"
# 3. Si valide : promotion planifiée avec bascule DNS/LB en 5 min
# 4. Sinon : diagnostic, le snapshot d'avant reste disponible
```

---

## 7. Observabilite

### 7.1 Logs : format et structure

Format **JSON une ligne** (pino), écrit sur stdout, collecté par Vector vers Loki (rétention 30 j en prod, 7 j en staging) et les lignesDWARN/ERROR vers Sentry. Aucune donnée de carte, aucun mot de passe, aucun secret dans les logs (§19.7, §26.5).

Champs obligatoires :

```json
{
  "timestamp": "2026-10-21T09:14:22.481Z",
  "level": "info",
  "service": "api",
  "env": "production",
  "version": "2f9c1a8b3d44",
  "correlation_id": "01J8Z9QK4T7M2N0P5R6S8V1W3Y",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "user_id_hash": "sha256:9f2b1c…",
  "session_id_hash": "sha256:41d0aa…",
  "route": "POST /api/checkout",
  "method": "POST",
  "status": 201,
  "duration_ms": 342,
  "order_id": "ord_01J8Z…",
  "event": "order.created",
  "msg": "Commande créée",
  "error": { "type": "", "code": "", "stack": "" }
}
```

- `correlation_id` : UUIDv7/ULID généré à l'entrée HTTP, propagé dans `X-Correlation-Id`, repris dans le job en queue, dans l'appel sortant (`X-Correlation-Id`) et dans le log d'intégration.
- `user_id_hash` : HMAC-SHA256(`user.id`, `LOG_HASH_KEY`) — non réversible sans la clé, rotation trimestrielle, clé distincte de la clé de session.

### 7.2 Redaction automatique des PII et secrets

Redaction appliquée en sortie du logger (côté application, avant écriture), sur 3 niveaux :

1. **Clés interdites** (valeur remplacée par `[REDACTED]`, jamais tronquée pour éviter de fuiter un préfixe) :
   `password`, `passwd`, `pwd`, `secret`, `token`, `access_token`, `refresh_token`, `id_token`, `authorization`, `cookie`, `set-cookie`, `session`, `api_key`, `apikey`, `client_secret`, `private_key`, `webhook_secret`, `signing_secret`, `encryption_key`, `cvv`, `cvc`, `card_number`, `pan`, `iban`, `bic`, `ssn`, `national_id`, `passport_number`, `mfa_secret`, `totp_secret`, `recovery_code`, `authorization_code`.
2. **Patterns de valeurs** (regex appliquées à toutes les chaînes, quel que soit le nom de la clé) :

```regex
# Cartes (13 à 19 chiffres, séparateurs optionnels, preuve de Luhn)
\b(?:4\d{3}|5[1-5]\d{2}|2[2-7]\d{2}|3[47]\d{2}|6011|65\d{2})[ -]?\d{4}[ -]?\d{4}[ -]?\d{4}\b

# Secrets génériques type "sk_live_...", "AKIA...", "ghp_...", JWT
\b(sk_(live|test)_[A-Za-z0-9]{16,}|pk_(live|test)_[A-Za-z0-9]{16,}|AKIA[0-9A-Z]{16}|gh[pousr]_[A-Za-z0-9]{36,}|xox[baprs]-[A-Za-z0-9-]{10,})\b

# JWT
\beyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\b

# Bearer / Basic
\b(Bearer|Basic)\s+[A-Za-z0-9._~+/=-]{16,}

# E-mail (partiel : on masque le domaine)
\b([A-Za-z0-9._%+-]{1,2})[A-Za-z0-9._%+-]*@([A-Za-z0-9.-]+\.[A-Za-z]{2,})\b

# Téléphone international
\b\+?\d{9,15}\b

# Clé privée PEM
-----BEGIN [A-Z ]*PRIVATE KEY-----[A-Za-z0-9+/=\n]+
```

Les valeurs ainsi détectées sont remplacées par `[REDACTED:<type>]`. Le retrait d'e-mail et de téléphone est **contextuel** : appliqué par défaut aux logs `info`, mais masqué derrière un `user_id_hash` pour les logs métier (traçabilité sans PII). L'adresse IP est tronquée au format `/24` (IPv4) ou `/48` (IPv6), sauf dans les logs WAF.

Règles absolues : **aucune donnée de carte, aucun mot de passe, aucun secret, aucun document client** dans les logs (§19.5, §19.7, §26.5). Un job CI (`scan-pii`) échoue si un log de test contient un PAN ou un secret ; un scan quotidien des logs de production (Vector) alerte si un pattern PII survit à la redaction.

### 7.3 Metriques : les 25 metriques instrumentees

Format Prometheus, exposition `/metrics` sur un port interne non public. Labels bornés (jamais de `user_id`, `order_id`, `email` en label — cardinalité non contrôlée).

| # | Nom | Type | Labels | Seuil d'alerte |
|---:|---|---|---|---|
| 1 | `http_requests_total` | counter | `service`,`method`,`route`,`status` | — |
| 2 | `http_request_duration_seconds` | histogram | `service`,`method`,`route` | p95 > 0,8 s (checkout) / > 1,2 s (catalogue) |
| 3 | `http_redirects_total` | counter | `from`,`to` | ratio > 5 % du trafic |
| 4 | `http_realtime_sessions_active` | gauge | `service` | — |
| 5 | `db_queries_total` | counter | `operation`,`table` | — |
| 6 | `db_query_duration_seconds` | histogram | `operation`,`table` | p95 > 0,2 s |
| 7 | `db_pool_size` | gauge | `pool` | > 90 % du `max` |
| 8 | `db_pool_wait_seconds` | histogram | `pool` | p95 > 0,05 s |
| 9 | `db_connections_active` | gauge | — | > 80 % de `max_connections` |
| 10 | `db_replication_lag_seconds` | gauge | — | > 30 s |
| 11 | `cache_operations_total` | counter | `cache`,`result` (`hit`/`miss`) | hit ratio < 80 % (catalogue) |
| 12 | `cache_evictions_total` | counter | `cache` | > 100 / min |
| 13 | `worker_queue_depth` | gauge | `queue` | > 200 (warning) / > 500 (critique) |
| 14 | `worker_job_duration_seconds` | histogram | `job_type` | p95 > 120 s |
| 15 | `worker_jobs_total` | counter | `job_type`,`outcome` | — |
| 16 | `worker_dlq_size` | gauge | `queue` | > 0 |
| 17 | `worker_oldest_job_age_seconds` | gauge | `queue` | > 900 |
| 18 | `emails_sent_total` | counter | `template`,`locale` | — |
| 19 | `email_bounce_total` | counter | `type` (hard/soft) | hard > 2 % sur 24 h |
| 20 | `webhooks_received_total` | counter | `provider`,`event` | — |
| 21 | `webhooks_processed_total` | counter | `provider`,`event`,`outcome` | `failed` ratio > 2 % sur 10 min |
| 22 | `webhooks_duplicate_total` | counter | `provider` | > 0 (SEV3, signale un bug d'idempotence) |
| 23 | `payment_failure_rate` | gauge | `provider`,`country` | > 1,5 % sur 10 min |
| 24 | `stock_reservation_failure_total` | counter | `reason` | ratio > 0,5 % |
| 25 | `app_up` | gauge | `service`,`region` | = 0 (SEV1) |

Métriques métier complémentaires hors table mais alertables : `revenue_captured_total` (compteur financier, source serveur uniquement §23.3), `quote_conversion_rate`, `low_stock_variants` (gauge, alerte stock), `trade_rule_review_due` (gauge, alerte conformité §11.2), `cert_expiry_seconds` (gauge, alerte certificat §8).

Règle d'étiquetage : les labels sont des valeurs d'un ensemble fini contrôlé ; tout label contenant un identifiant d'utilisateur, de commande ou d'e-mail est interdit (revue de code + test d'instrumentation qui échoue).

### 7.4 Traces et propagation du correlation_id

- OpenTelemetry, format **W3C `traceparent`** : `00-<32 hex trace-id>-<16 hex span-id>-<flags>`, propagé en header HTTP sur toutes les requêtes entrantes et sortantes, et transporté dans le payload des jobs (`job.traceparent`).
- Chaîne complète : HTTP entrant (span `server`) → appel Drizzle/PostgreSQL (span `db`) → appel Redis → file BullMQ (lien `span` au moment du `enqueue`, nouveau span `consumer` lié) → appel sortant PSP/transporteur/e-mail (span `client`).
- `correlation_id` = `trace_id` derivé (identique dans les logs applicatifs), ce qui rend la recherche par `correlation_id` dans Loki et dans Sentry univoque.
- **Sampling** : 100 % en local et staging, 25 % en staging-preprod, 10 % en production, **100 % des erreurs et des requêtes `/api/checkout` et `/webhooks/*`**, plus un *tail sampling* qui conserve les traces dont une span dépasse 2 s ou contient une erreur.

```ts
// packages/observability/src/trace.ts
import { context, propagation, SpanKind, trace } from '@opentelemetry/api';

export async function withTrace<T>(
  name: string,
  fn: (span: { setAttribute(k: string, v: unknown): void }) => Promise<T>,
): Promise<T> {
  const tracer = trace.getTracer('fanours');
  return tracer.startActiveSpan(name, { kind: SpanKind.INTERNAL }, async (span) => {
    try {
      return await fn(span);
    } catch (err) {
      span.recordException(err as Error);
      span.setAttribute('error.type', (err as Error).name);
      throw err;
    } finally {
      span.end();
    }
  });
}

export function injectTraceparent(carrier: Record<string, string>): void {
  propagation.inject(context.active(), carrier, {
    set: (c, k, v) => { c[k] = String(v); },
  });
}
```

### 7.5 Suivi d'erreurs

- **Sentry** (self-hosted ou SaaS UE) : Next.js côté serveur (source maps via `sentry-cli sourcemaps upload` dans le job `build`), API et worker (`@sentry/node`), captures d'exceptions non gérées, transactions (p95 réel), sessions (taux d'erreur frontend), breadcrumbs.
- Corrélation : tag `correlation_id`, `trace_id`, `env`, `service`, `version`, `release`. Les logs ne contiennent jamais de PII ; Sentry reçoit des `beforeSend` qui strippent `request.data` et `user.email`, et remplacent `user.id` par le même hash que les logs.
- Sourcemaps : uploadées **une seule fois par release** dans le job `build` (SENTRY_AUTH_TOKEN via OIDC/secret), et non à chaque run.
- Alertes : (a) **nouvelle erreur** (issue jamais vue, taux > 5 occurrences/10 min) → alerte SEV3 vers l'équipe qui a introduit la release ; (b) **régression** (taux d'erreur d'une release > 1,5× la baseline de la release précédente sur 30 min) → alerte SEV2 et blocage de promotion de canary ; (c) disappearance d'une issue traitée = non, on ne notifie pas.

---

## 8. Alertes et on-call

### 8.1 Matrice de severite

| Sev | Définition | Exemples | Notification | Prise en compte | Qui est prévenu |
|---|---|---|---|---|---|
| **SEV1** | Perte totale ou partielle majeure d'une fonction commerciale ; perte de données probable ; compromission de sécurité suspectée | Paiement indisponible, 5xx > 20 %, base en lecture seule, fuite de données | Pager on-call (téléphone + Slack) + SMS | 5 min | On-call, CTO, product owner, communication client si impact > 30 min |
| **SEV2** | Fonction dégradée, contournement possible, impact mesuré | Webhooks échoués > 5 %, p95 > 1,5 s, file saturée > 1 h, stock critique sur produit à forte rotation | Slack + PagerDuty | 30 min | On-call, lead technique, responsable du domaine concerné |
| **SEV3** | Impact limité, contournement manuel possible | Erreur de certification PDF, bounce e-mail > 5 %, job en DLQ isolé, TTL cache non invalidé | Slack canal `#incidents` | 4 h (heures ouvrées) | Équipe owning du module |
| **SEV4** | Information, dette, risque préventif | Certificat expire dans 21 j, backup à vérifier, règle douanière à revoir, seuil de capacité | Ticket backlog / e-mail | 7 j | Responsable concerné |

Règles : escalade automatique si l'alerte n'est pas acquittée (SEV1 : 5 min puis SEV2, 15 min puis SEV3/lead), et si un SEV1 est ouvert, tout nouvel incident est traité en SEV1 par défaut jusqu'à qualification. Rotation on-call hebdomadaire,_CHANNEL `#oncall`, procédure d'escalade et numéros dans le gestionnaire de secrets, jamais dans le dépôt.

### 8.2 Alertes critiques et leurs seuils

| Alerte | Condition technique | Sev | Seuil exact |
|---|---|---|---|
| Paiement indisponible | `payment_failure_rate > 5 %` sur 5 min **ou** health check PSP en échec | SEV1 | rollback canary immédiat |
| Webhook en échec répété | `webhooks_processed_total{outcome="failed"}` ratio > 2 % sur 10 min, ou ≥ 5 échecs consécutifs du même event | SEV1 (paiement) / SEV2 | rejouer la DLQ manuellement |
| File email saturée | `worker_queue_depth{queue="email"} > 500` ou `worker_oldest_job_age_seconds > 900` | SEV2 | drain manuel ou scale worker |
| Stock critique | `low_stock_variants > 0` sur SKU à rotation > X, ou stock < seuil pour un produit vendu | SEV2 (ops) / SEV3 | alerte role logistique |
| 5xx au-dessus du seuil | ratio 5xx > 1 % sur 5 min (canary) ou > 2 % (prod) | SEV2 / SEV1 | rollback automatique si canary |
| Latence p95 | p95 > 1,2 s (catalogue) ou > 0,8 s (checkout) sur 5 min | SEV2 | rollback si canary |
| Certificat expire | `cert_expiry_seconds < 21 j` / `< 7 j` | SEV3 / SEV1 | renouvellement auto + escalade si échec |
| Backup échoué | job de sauvegarde KO ou snapshot absent à J+1 | SEV2 | réexécution manuelle, sinon SEV1 à J+2 |
| SSL/HTTPS | certificat expiré, chaîne incomplète, HSTS absent en prod, redirect http→https cassé | SEV1 | renouvellement + rebascule |
| Base en lecture seule | `pg_is_in_recovery()` vrai, ou erreur SQLSTATE 25006 | SEV1 | failover ou bascule standby |
| Santé globale | `/readyz` KO sur 2 instances consécutives, ou `app_up == 0` | SEV1 | bascule trafic + page maintenance si > 5 min |
| Backlog worker | profondeur > 200 | SEV3 | scale worker |
| DLQ non vide | `worker_dlq_size > 0` | SEV2 | analyse + rejeu |
| Certificat TLS faible / TLS 1.0 | scan externe en échec | SEV3 | durcissement |
| Disablement d'un pays/produit | flag conformité activé sans justification | SEV2 | audit (§11.6) |
| Règle douanière expirée | `trade_rule_review_due > 0` depuis 7 j | SEV3 | gestionnaire conformité |

Chaque règle d'alerte est versionnée dans `ops/alerting/rules/` (Prometheus), revue en PR, et testée par un injecteur (`promtool test rules`) : une règle qui n'a jamais déclenché en test est supprimée. Toute règle d'alerte pointe vers un runbook (§15) ; une alerte sans runbook est considérée comme un défaut de configuration.

---

## 9. Sauvegardes et continuite

### 9.1 Objectifs (§19.8)

- **RPO ≤ 24 h** — perte de données maximale acceptable. Atteint par PITR (WAL archivé en continu) : la perte réelle est de quelques minutes, largement sous l'objectif.
- **RTO ≤ 8 h** — durée maximale de rétablissement. Atteint par une procédure de restauration testée mensuellement (§9.6) avec des runbooks qui rendent la restauration semi-automatique.
- Ces valeurs sont celles proposées en §19.8 ; elles sont **à valider par le propriétaire** (§28.2 point 11) avant lancement.

### 9.2 PostgreSQL

```bash
# Sauvegarde incrémentale continue (PITR) — pgBackRest vers S3
pgbackrest --stanza=prod --type=backup --target=full        # dimanche 02:00 UTC
pgbackrest --stanza=prod --type=backup --target=incremental  # quotidien 02:00 UTC
pgbackrest --stanza=prod --type=archive-push                  # continu ( archiving WAL vers S3 )
pgbackrest --stanza=prod --type=restore --target=time "2026-10-21 09:00:00+00" --pg1-path=/restore
```

- **WAL archiving** continu vers un bucket S3 **distinct** du bucket applicatif, avec versioning et rétention 35 jours.
- **Snapshots quotidiens** (base managée ou snapshots `pgbackrest`) + **snapshot complet hebdomadaire** conservé 90 jours.
- **Rétention 35 jours** sur les snapshots/PITR (PITR = 35 j de WAL ; snapshots quotidiens 35 j ; hebdo 90 j).
- **Chiffrement** : au repos (KMS du fournisseur) + chiffrement côté client possible via `pgbackrest` (option `repo1 cipher-type=aes-256-xts`) pour les sauvegardes hors site ; clés dans le gestionnaire de secrets, rotation annuelle avec re-chiffrement progressif.
- **Copie hors site** : snapshot quotidien répliqué vers un bucket dans une région différente (UE, région B), compte à accès séparé, MFA obligatoire.
- **Intégrité** : `pgbackrest verify` hebdomadaire ; checksum par snapshot ; un checksum corrompu déclenche une alerte SEV2.

### 9.3 Objets S3

- **Versioning activé** sur `fanours-prod` (assets) et `fanours-prod-documents` (factures, devis, avoirs, pièces clients) : les documents commerciaux doivent rester accessibles même après suppression logique (§14.5, §20.2).
- **Lifecycle** :
  - `fanours-prod` (assets) : transitions Glacier Instant Retrieval à 90 j, Glacier Deep Archive à 365 j, expiration à 730 j.
  - `fanours-prod-documents` : transition Glacier à 365 j, **expiration à 10 ans** (obligation commerciale/fiscale selon juridiction, à valider avec le cabinet comptable — §28.2 point 9), pas de suppression avant accord écrit.
  - `fanours-prod-logs-reports` : expiration 180 j.
- **Bucket privé** avec block public access, accès par presigned URL 5 min, accès journalisé (§3.2).
- **Sauvegarde** : réplication versioning + cross-region copy ; la perte d'un objet est récupérable via version précédente.

### 9.4 Redis — non critique

Redis ne contient que des données reconstructibles : cache catalogue, sessions invalidables, panier (durable en base pour les comptes connectés — §9.1), clés d'idempotence (rejouables par le PSP) et files de tâches (les jobs sont dans PostgreSQL, Redis n'est que le transport — voir §11.4).

Conséquence documentée et assumée : **la perte de Redis provoque une invalidation de cache, une déconnexion des sessions actives et un retard de traitement des jobs en cours, mais aucune perte de donnée métier**. Mitigation : persistance activée (`appendonly yes`, `appendfsync everysec`) + snapshot RDB, délai de reprise < 5 min, alerte `cache_operations_total{result="miss"}` en pics après redémarrage. La file de référence (durable) est PostgreSQL (§11.4).

### 9.5 Secrets

- Gestionnaire : **Vault** (self-hosted sur VM dédiée) ou **cloud KMS/Secrets Manager** selon l'hébergeur retenu ; le backend stocke uniquement des **références** (`secret://fanours/prod/stripe-live`) jamais les valeurs.
- Sauvegarde : export chiffré du Vault dans un bucket S3 à accès restreint (compte distinct de celui de l'application), rétention 90 j, réplication hors site, test de restauration trimestriel ; les clés de déverrouillage du Vault sont partagées entre deux responsables (2 personnes minimum) via un coffre hors ligne.
- Rotation : procédure trimestrielle par secret, avec double secret (overlap) pour les secrets applicatifs (ex. clé de webhook PSP : signature avec l'ancienne et la nouvelle clé pendant 7 j).
- Emergency rotation : procédure de rotation d'urgence en < 30 min, déclenchée par compromission suspectée (§19.5).

### 9.6 Procedure de restauration testee (mensuelle) — le test de restauration lui-meme

Chaque mois, un job planifié (`ops/backup/restore-drill.yml`, GitHub Actions *schedule*) exécute une **restauration réelle** dans un environnement isolé, et non une simple vérification de présence de fichier :

```yaml
name: Monthly restore drill
on:
  schedule: [{ cron: '17 3 1 * *' }]   # 1er du mois, 03:17 UTC
  workflow_dispatch:
jobs:
  restore-drill:
    runs-on: ubuntu-latest
    timeout-minutes: 90
    steps:
      - run: ./ops/backup/restore-drill.sh --target=2026-10-01T02:00:00Z --env=drill
      - name: Assertions
        run: |
          set -euo pipefail
          psql "$DRILL_URL" -v ON_ERROR_STOP=1 -f ops/backup/assertions.sql
          # assertions.sql vérifie :
          #  - table orders : count > 0 et aucune commande orpheline
          #  - sum(payments.amount) = sum(orders.total) pour les commandes payées
          #  - dernière commande < date cible (cohérence temporelle)
          #  - extensions et schémas présents (migrations à jour)
      - name: Test RPO
        run: ./ops/backup/measure-rpo.sh --target="${{ env.LAST_SNAPSHOT }}"  # écart réel vs objectif 24 h
      - name: Test RTO
        run: ./ops/backup/measure-rto.sh --drill-run-id="${{ github.run_id }}"   # durée totale vs objectif 8 h
      - uses: actions/upload-artifact@v4
        with: { name: restore-drill-report, path: ops/backup/reports/, retention-days: 90 }
```

Résultats attendus et archivés dans `ops/backup/reports/YYYY-MM/` : RPO mesuré, RTO mesuré, anomalies, actions correctives. Le rapport mensuel est revu par le responsable technique ; deux drills consécutifs hors objectifs déclenchent une revue de l'architecture de sauvegarde (SEV2 de capacité).

Règle forte : **aucune migration destructive sans preuve de restauration testée** (§6.3) — la preuve exigée est un drill réussi portant sur le snapshot concerné, pas la simple existence d'un fichier.

---

## 10. Performance et CWV en production

### 10.1 Budget de performance par type de route

| Type de route | Exemples | TTFB (p75) | LCP (p75) | INP (p75) | CLS (p75) | Cache |
|---|---|---|---|---|---|---|
| Page publique indexable | accueil, catégorie, article | ≤ 400 ms | ≤ 2,0 s | ≤ 150 ms | ≤ 0,05 | CDN + ISR 24 h, stale-while-revalidate |
| Fiche produit | `/fr/produit/slug` | ≤ 500 ms | ≤ 2,2 s | ≤ 180 ms | ≤ 0,08 | ISR 1 h + invalidation tag |
| Catalogue / recherche | listing, filtres | ≤ 600 ms | ≤ 2,5 s | ≤ 200 ms | ≤ 0,10 | CDN 60 s + cache Redis 5 min |
| API lecture | `/api/products*` | ≤ 200 ms | — | — | — | Redis + ETag |
| Panier / checkout | mutation + rendu | ≤ 800 ms | ≤ 2,5 s | ≤ 200 ms | ≤ 0,10 | `no-store` (jamais de CDN sur checkout) |
| Compte client | commandes, factures | ≤ 700 ms | ≤ 2,5 s | ≤ 200 ms | ≤ 0,10 | `private, no-store` |
| Dashboard admin | agrégats | ≤ 1 200 ms | ≤ 3,0 s | ≤ 250 ms | ≤ 0,10 | requêtes agrégées pré-calculées, §23.1 |
| Images | `/_next/image`, S3 | — | contribution ≤ 500 ms | — | dimensions explicites (`width`/`height`) → CLS 0 | CDN, AVIF/WebP, `srcset` |

Budgets de bundle (fail CI au-delà) : JS first-load ≤ 170 Ko compressé pour une page catalogue, ≤ 220 Ko pour le checkout ; CSS ≤ 60 Ko ; police auto-hébergée (Next `next/font`) découpée par sous-ensemble (latin + latin-ext + SC), `font-display: swap`, preload ≤ 2 fichiers.

### 10.2 CDN et cache

| Ressource | Cache-Control | TTL | Invalidation |
|---|---|---|---|
| `/_next/static/*`, `/assets/*` (hachés) | `public, max-age=31536000, immutable` | 1 an | jamais (nom de fichier versionné) |
| HTML ISR (pages publiques) | `public, s-maxage=86400, stale-while-revalidate=3600` | 24 h | `revalidateTag('product:<id>')` / `revalidatePath` à la publication |
| API catalogue | `public, s-maxage=60, stale-while-revalidate=300` | 60 s | invalidation tag + purge CDN sur changement produit |
| Images S3 via CDN | `public, max-age=31536000, immutable` | 1 an | versioning objet |
| `/api/checkout/*`, `/api/cart/*`, `/api/orders/*`, `/api/documents/*` | `no-store` + `Cache-Control: private` | — | jamais |
| Pages compte / admin | `no-store` | — | jamais |
| HTML SSR non-ISR | `private, no-cache` | — | — |

Règles : aucune réponse contenant une donnée personnelle (cookie de session, `Authorization`) n'est mise en cache par le CDN → règle WAF + `Vary` ; le middleware Next `no-store` sur les routes privées ; le cache CDN ne sert que du contenu public.

### 10.3 Cache applicatif (Redis)

| Donnée | Clé | TTL | Invalidation |
|---|---|---|---|
| Catalogue / listing | `catalog:v{ver}:page:{hash}` | 300 s | invalidation par tag `catalog` au changement produit |
| Fiche produit | `product:{id}:{locale}` | 3600 s | `DEL` + `revalidateTag` à la publication |
| Prix / paliers calculés | `pricing:{productId}:{customerTier}:{qty}` | 600 s | invalidation tag `pricing` au changement de palier |
| Session | `session:{id}` | TTL session (12 h glissantes / 30 j absolues, §13.4) | révocation explicite |
| Panier invité | `cart:{anonId}` | 30 j | TTL seul + fusion à la connexion |
| Idempotence | `idem:{scope}:{key}` | 24 h | jamais (protection anti-doublon) |
| Rate limit | `rl:{scope}:{ident}` | fenêtre | compteur Fenêtre glissante |
| Réponses API cherches | `api:{hash}` | 60 s | invalidation tag |

Règle d'invalidation : toute mutation métier invalide explicitement les clés concernées (pas d'attente d'expiration) ; les invalidations groupées (`tag`) sont déclenchées par l'outbox, jamais par le code HTTP de réponse (qui peut échouer après commit). Un cache obsolète est tolérable (TTL), une donnée de prix erronée ne l'est pas : les prix sont recalculés côté serveur à chaque action déterminante (§9.1).

### 10.4 SQL : requetes critiques indexees

```sql
-- Catalogue : filtres du §6.3 (catégorie, disponibilité, mode de vente, pays, prix) + tri
CREATE INDEX CONCURRENTLY idx_product_published_category
  ON products (category_id, published_at DESC) WHERE status = 'published';
CREATE INDEX CONCURRENTLY idx_price_tier_lookup
  ON price_tiers (product_variant_id, min_quantity) WHERE valid_to IS NULL;

-- Recherche textuelle (PostgreSQL FTS au lancement, §5.1)
CREATE INDEX CONCURRENTLY idx_product_translation_fts
  ON product_translations USING gin (to_tsvector('simple', name || ' ' || description));

-- Panier / commandes
CREATE INDEX CONCURRENTLY idx_orders_customer_created ON orders (customer_id, created_at DESC);
CREATE INDEX CONCURRENTLY idx_orders_status_created  ON orders (status, created_at DESC);
CREATE INDEX CONCURRENTLY idx_order_lines_order      ON order_lines (order_id);
CREATE INDEX CONCURRENTLY idx_payments_provider_ref  ON payments (provider, provider_ref);  -- réconciliation §10.5

-- Webhooks : lookup par événement externe + idempotence
CREATE UNIQUE INDEX CONCURRENTLY uq_webhook_events_provider_event
  ON webhook_events (provider, provider_event_id);
CREATE INDEX CONCURRENTLY idx_webhook_events_status ON webhook_events (status, received_at);

-- Règles douanières : résolution par pays/catégorie/date (§11.2)
CREATE INDEX CONCURRENTLY idx_trade_rules_lookup
  ON trade_rules (country_code, category_id, effective_from DESC);

-- Jobs (§11.4)
CREATE INDEX CONCURRENTLY idx_background_jobs_claim ON background_jobs (status, run_at)
  WHERE status IN ('pending', 'retry');
```

Règles : pagination **keyset** (`WHERE (created_at, id) < (?, ?)`) au-delà de 1 000 lignes, jamais d'`OFFSET` sur les listing publics ; limite de taille de requête et de résultat sur l'API (§21.1) ; `EXPLAIN (ANALYZE, BUFFERS)` obligatoire pour toute nouvelle requête critique, reviewé en PR ; alertes sur requêtes > 500 ms (`db_query_duration_seconds`).

### 10.5 Payload, compression, images

- Limite de taille de requête : 256 Ko (JSON), 10 Mo (upload avec MIME réel validé §19.3) ; réponses API paginées à 50 éléments max, 200 max sur export autorisé.
- Compression : `gzip`/`br` sur toutes les réponses texte > 1 Ko (serveur + CDN), `zstd` pour les assets.
- Images : AVIF puis WebP, fallback JPEG ; `width`/`height` systématiques (CLS = 0) ; `loading="lazy"` sous la ligne de flottaison, `priority` sur l'image LCP ; `sizes` correct pour éviter le téléchargement d'une image 2× trop grande ; CDN avec redimensionnement automatique.
- Polices auto-hébergées, sous-ensemble par locale, `preload` limité à 2 fichiers, `display: swap`.

### 10.6 Monitoring CWV

- **Lighthouse CI** dans le pipeline PR (job `lighthouse` non bloquant en PR, bloquant sur `main`) : budgets LCP ≤ 2,5 s, INP ≤ 200 ms, CLS ≤ 0,1, sur 4 URLs représentatives (accueil, catégorie, fiche produit, checkout), mode mobile, rapport archivé 90 j.
- **RUM (Real User Monitoring)** : mesure p75 sur le trafic réel, par pays et par type d'appareil. Implémentation sans cookie publicitaire : métriques agrégées via `PerformanceObserver` + envoi vers l'endpoint interne (pas de Google Analytics tiers sans consentement, §23.3) ; respect de `navigator.connection.saveData` et `prefers-reduced-data` (pas de mesure des pages light).
- **Tableau de bord CWV** : p75 LCP/INP/CLS par page template, par pays, par appareil ; alerte si p75 LCP > 2,5 s ou INP > 200 ms ou CLS > 0,1 pendant 24 h (SEV3), corrélée aux déploiements (comparaison avant/après).
- **Distinction terrain/labo** : Lighthouse CI sert de garde de non-régression en laboratoire ; seul le RUM sert de preuve de performance réelle (seuil d'acceptation en production = RUM p75, §23.1).

---

## 11. Jobs et files de taches

### 11.1 Architecture du worker

Un **binaire unique** `worker` (même image que `api`, `WORKER_ROLE=consumer`) qui :

1. Consomme plusieurs files Redis/BullMQ avec des Workers séparés (un par famille de jobs) → un job PDF bloqué n'arrête pas les e-mails.
2. Exécute un **scheduler interne** (cron léger, `node-cron` ou équivalent) pour les tâches récurrentes (sync tracking, réconciliation, sitemap, alertes stock) — évite un service cron externe.
3. Limite la concurrence par file (`concurrency` par type, ex. 5 pour les e-mails, 2 pour les PDF) afin de protéger les API tierces et la base.
4. Gère les retries avec backoff exponentiel + jitter et bascule en **DLQ** après épuisement.
5. S'expose sur `/metrics` (interne) et `/healthz` pour le probing ; un worker qui ne traite plus est détecté par l'ancienneté du job le plus vieux (`worker_oldest_job_age_seconds`).

**File de référence durable en PostgreSQL** : Redis n'est que le transport. La table `background_jobs` est la source de vérité de l'état (`pending`/`running`/`succeeded`/`retry`/`failed`/`dead`), ce qui rend la reprise possible après perte de Redis (§9.4) et permet l'observabilité métier demandée par l'administrateur (§16.2 « statut des intégrations, queue d'e-mails, tâches en erreur »).

### 11.2 Les 15 types de jobs

| # | Type | Déclencheur | Charge | Criticité | Retry (max) | DLQ | Idempotence | Observabilité |
|---:|---|---|---|---|---|---|---|---|
| 1 | `email.transactional` | Outbox `email_*` (commande, paiement, devis, expédition, facture, retour) | 5–50 Ko | Moyenne | 6 (exp+jitter) | oui | clé `template+entity+locale` | `emails_sent_total`, bounce, DLQ, latence d'envoi |
| 2 | `webhook.payment` | Webhook PSP signé | 2–10 Ko | **Critique** | 10 (exp+jitter) | oui | `provider_event_id` unique | `webhooks_*`, ratio échec, alerte SEV1 |
| 3 | `document.pdf` | Événement `invoice.issued` / `quote.sent` / `packing_list` | 100 Ko–2 Mo | Moyenne | 5 | oui | `document:{type}:{entityId}:{version}` | durée, erreurs de rendu, taille |
| 4 | `invoice.generate` | `order.paid` | 50 Ko | Moyenne | 5 | oui | `invoice:{orderId}:{seq}` | cohérence numéro, doublons interdits (§14.5) |
| 5 | `shipping.estimate` | Panier/checkout (multi-transporteur) | 5–20 Ko | Moyenne | 4 | oui | hash des paramètres | latence, taux d'échec transporteur |
| 6 | `tracking.sync` | Cron 15 min + webhook transporteur | 10–50 Ko | Moyenne | 8 | oui | `shipmentId+eventTs+carrierEventId` | événements dupliqués désordonnés ignorés |
| 7 | `payment.reconcile` | Cron quotidien 02:30 | 20–500 Ko | Haute | 3 | oui | `date+provider` | écarts détectés, versements non rapprochés |
| 8 | `catalog.import` | Upload CSV/XLSX admin | 1–100 Mo | Basse | 2 | oui | `importId` + numéro de ligne | rapport par ligne, annulation possible |
| 9 | `export.data` | Demande admin (rapport, catalogue) | 1–500 Mo | Basse | 3 | oui | `exportId` | expiration du fichier, URL signée |
| 10 | `analytics.compute` | Cron nocturne + événement | 50 Ko–5 Mo | Basse | 3 | oui | `date+scope` | freshness des KPI (§23.3) |
| 11 | `stock.alert` | Cron 1 h + `inventory.low_stock` | 2 Ko | Moyenne | 4 | oui | `variantId+threshold+day` | alertes envoyées aux bons rôles |
| 12 | `trade_rule.review` | Cron quotidien + échéance | 5 Ko | Moyenne (conformité) | 3 | oui | `ruleId+version` | règles expirées bloquent (§11.2) |
| 13 | `sitemap.generate` | Cron 6 h + publication produit | 20–200 Mo | Basse | 3 | oui | `date` | taille, indexation, URL canoniques |
| 14 | `notification.dispatch` | Outbox `notification.*` (in-app + push) | 5 Ko | Moyenne | 5 | oui | `userId+event+entityId` | centre de notifs, push échoué |
| 15 | `media.process` | Upload image (variantes, WebP/AVIF, miniatures) | 50 Ko–20 Mo | Moyenne | 5 | oui | `mediaId+variant` | ratio d'échec, temps de traitement |

### 11.3 Schema de la table `jobs` (reference durable)

```sql
CREATE TABLE background_jobs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type            TEXT        NOT NULL,
  queue           TEXT        NOT NULL,
  payload         JSONB       NOT NULL,
  status          TEXT        NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','running','succeeded','retry','failed','dead','cancelled')),
  idempotency_key TEXT        NOT NULL,
  attempts        INT         NOT NULL DEFAULT 0,
  max_attempts    INT         NOT NULL DEFAULT 5,
  priority        SMALLINT    NOT NULL DEFAULT 5,
  run_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  started_at      TIMESTAMPTZ,
  finished_at     TIMESTAMPTZ,
  next_retry_at   TIMESTAMPTZ,
  correlation_id  TEXT,
  traceparent     TEXT,
  last_error      JSONB,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT background_jobs_idempotency_unique UNIQUE (type, idempotency_key)
);
CREATE INDEX idx_background_jobs_claim ON background_jobs (status, run_at, priority DESC)
  WHERE status IN ('pending','retry');
CREATE INDEX idx_background_jobs_dead ON background_jobs (dead_at) WHERE status = 'dead';
```

```sql
-- Table DLQ (référence durable de la file d'échec)
CREATE TABLE background_jobs_dead_letter (
  id             BIGSERIAL PRIMARY KEY,
  job_id         UUID NOT NULL REFERENCES background_jobs(id),
  type           TEXT NOT NULL,
  payload        JSONB NOT NULL,
  attempts       INT  NOT NULL,
  last_error     JSONB,
  correlation_id TEXT,
  dead_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  replayed_at    TIMESTAMPTZ,
  replayed_by    UUID
);
```

### 11.4 Pattern de job idempotent (TypeScript)

```ts
// packages/jobs/src/idempotent-job.ts
import { randomUUID } from 'node:crypto';
import { db } from '../db';
import { logger } from '../observability/logger';

export type JobContext = {
  jobId: string;
  type: string;
  attempts: number;
  correlationId: string;
  traceparent?: string;
};

export type IdempotentHandler<TPayload, TResult> = (
  payload: TPayload,
  ctx: JobContext,
) => Promise<TResult>;

/**
 * Clé d'idempotence : déterministe à partir du fait métier, pas d'un UUID aléatoire.
 * Exemple : `email.transactional` -> `order-paid:ord_123:fr:invoice`
 */
export function idempotencyKey(type: string, parts: (string | number)[]): string {
  return `${type}:${parts.join(':')}`;
}

/** Backoff exponentiel + jitter complet (full jitter), plafonné. */
export function backoffMs(attempt: number, base = 1_000, cap = 15 * 60_000): number {
  const exp = Math.min(cap, base * 2 ** (attempt - 1));
  return Math.floor(Math.random() * exp); // full jitter : évite les tempêtes de retry
}

export function defineJob<TPayload, TResult>(
  type: string,
  options: { maxAttempts: number; queue: string; timeoutMs: number; idempotency: (p: TPayload) => string },
  handler: IdempotentHandler<TPayload, TResult>,
) {
  return {
    type,
    queue: options.queue,
    async process(payload: TPayload, ctx: JobContext): Promise<TResult> {
      const key = options.idempotency(payload);
      const started = Date.now();

      // 1) Verrou atomique : une seule exécution concurrente par clé.
      //    ON CONFLICT DO NOTHING => si la ligne existe, un autre worker a déjà pris le job.
      const claimed = await db.$queryRaw<{ id: string }[]>`
        INSERT INTO background_jobs (type, queue, payload, idempotency_key, correlation_id, traceparent, status)
        VALUES (${type}, ${options.queue}, ${payload}::jsonb, ${key}, ${ctx.correlationId}, ${ctx.traceparent ?? null}, 'running')
        ON CONFLICT (type, idempotency_key) DO NOTHING
        RETURNING id
      `;

      if (claimed.length === 0) {
        // Déjà traité (ou en cours) : on sort sans effet de bord.
        logger.info({ type, idempotency_key: key }, 'job.skipped.idempotent');
        return { skipped: true } as TResult;
      }

      const jobId = claimed[0].id;

      try {
        const result = await withTimeout(handler(payload, { ...ctx, jobId }), options.timeoutMs);
        await db.$executeRaw`
          UPDATE background_jobs
             SET status = 'succeeded', finished_at = now(), updated_at = now()
           WHERE id = ${jobId}
        `;
        logger.info({ type, jobId, duration_ms: Date.now() - started }, 'job.succeeded');
        return result;
      } catch (err) {
        const error = err as Error;
        const canRetry = ctx.attempts < options.maxAttempts;

        if (canRetry) {
          await db.$executeRaw`
            UPDATE background_jobs
               SET status = 'retry', attempts = attempts + 1,
                   last_error = ${JSON.stringify({ message: error.message, stack: error.stack })}::jsonb,
                   next_retry_at = now() + (${backoffMs(ctx.attempts + 1)} || ' milliseconds')::interval,
                   updated_at = now()
             WHERE id = ${jobId}
          `;
          logger.warn({ type, jobId, attempt: ctx.attempts, error: error.message }, 'job.retry');
          throw err; // BullMQ programme le retry
        }

        // 2) DLQ durable : le job n'est pas perdu, il est rejouable par un humain.
        await db.$executeRaw`
          UPDATE background_jobs SET status = 'dead', finished_at = now(), updated_at = now() WHERE id = ${jobId}
        `;
        await db.$executeRaw`
          INSERT INTO background_jobs_dead_letter (job_id, type, payload, attempts, last_error, correlation_id)
          VALUES (${jobId}, ${type}, ${payload}::jsonb, ${ctx.attempts},
                  ${JSON.stringify({ message: error.message })}::jsonb, ${ctx.correlationId})
        `;
        logger.error({ type, jobId, attempts: ctx.attempts, error: error.message }, 'job.dead');
        throw err;
      }
    },
  };
}
```

Usage typique :

```ts
export const sendOrderEmail = defineJob<SendOrderEmailPayload, void>(
  'email.transactional',
  {
    queue: 'email',
    maxAttempts: 6,
    timeoutMs: 30_000,
    idempotency: (p) => idempotencyKey('email.transactional', [p.template, p.orderId, p.locale]),
  },
  async (payload, ctx) => {
    await mailer.send({
      template: payload.template,
      to: payload.to,
      locale: payload.locale,
      correlationId: ctx.correlationId,
    });
  },
);
```

Le même principe s'applique aux webhooks : l'unicité `provider + provider_event_id` (index unique §10.4) garantit qu'un webhook rejoué ne déclenche pas deux factures, deux stocks ou deux notifications (§9.5, §28.1 « double traitement de webhook »).

### 11.5 Observabilite des jobs

Chaque job émet : `job.started`, `job.succeeded`, `job.retry`, `job.dead`, avec `jobId`, `type`, `attempts`, `duration_ms`, `correlation_id`. Métriques : `worker_jobs_total{type,outcome}`, `worker_job_duration_seconds{type}`, `worker_queue_depth{queue}`, `worker_oldest_job_age_seconds{queue}`, `worker_dlq_size{queue}`. Le tableau de bord admin expose la profondeur de file et la DLQ (§16.2) et permet un rejeu manuel autorisé et journalisé.

---

## 12. Email et PDF

### 12.1 DNS de deliverabilite

Enregistrement DNS du domaine d'envoi `mail.fanours.com` (domaine de sous-envoi, séparé du domaine corporate `fanours.com` qui sert au transactional strict) :

```dns
; --- Sous-envoi transactionnel : mail.fanours.com ---
mail.fanours.com.          IN  TXT   "v=spf1 include:_spf.postmark.com -all"
fanours.com.               IN  TXT   "v=spf1 include:_spf.postmark.com -all"
_default._domainkey.fanours.com.        IN CNAME postmark._domainkey.fanours.com.

; --- DKIM (rotation) : une clé par sélecteur, plusieurs sélecteurs en rotation ---
postmark._domainkey.fanours.com.       IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSq...BASE64...ID"
postmark2._domainkey.fanours.com.      IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSq...BASE64...ID"

; --- DMARC (progression p=none -> quarantine -> reject) ---
_dmarc.fanours.com.        IN  TXT   "v=DMARC1; p=none; rua=mailto:dmarc@fanours.com; ruf=mailto:dmarc@fanours.com; fo=1; pct=100"

; --- MX et bounce ---
fanours.com.               IN  MX    10 mx1.postmarkapp.com.
fanours.com.               IN  MX    20 mx2.postmarkapp.com.
fanours.com.               IN  TXT   "v=spf1 ... "   (déjà défini)

; --- Alignement de domaine (eli-omo-media / eliomo) : sous-domaine dédié ---
eliomo-media.com.          IN  TXT   "v=spf1 include:_spf.postmark.com -all"
_dmarc.eliomo-media.com.   IN  TXT   "v=DMARC1; p=quarantine; rua=mailto:dmarc@fanours.com; pct=100"
```

- **SPF** : une seule inclusion (`include:`) par enregistrement, `~all` puis `-all` quand le fournisseur est stabilisé ; le nombre de lookups DNS reste < 10 (limite RFC).
- **DKIM** : 2048 bits, **rotation trimestrielle** : le nouveau sélecteur est activé en `postmark2` le temps que la propagation DNS soit complète (24–48 h), l'ancien n'est retiré qu'après vérification de signature sur 100 % des messages pendant 7 j.
- **DMARC** : démarrage `p=none` avec `rua` (rapports agrégés) pendant 2 semaines, puis `p=quarantine` 4 semaines, puis `p=reject` — la progression est validée par un comité (marketing + technique) car `reject` sans alignement strict bloque la délivrabilité.
- **Alignement (eli-omo-media)** : le domaine d'envoi de la marque `eli-omo-media` aligne son DMARC sur le domaine de l'expéditeur organisationnel ; le SPF est publié sur le domaine exact utilisé dans l'en-tête `Return-Path`/`From` (alignement strict), sinon les messages sont déplacés en spam.
- **MX** : le domaine de rebond doit accepter les bounces (ne pas faire de `catch-all` qui masque les comptes) et le traitement des complained (`bounce@` → suppression de l'envoi des messages non transactionnels) est automatisé sous 24 h.
- **Sous-envoi** : `notifications@fanours.com` pour le transactionnel, `newsletter@fanours.com` pour le marketing (domaine et IP séparés, sinon le reputation d'un contamine l'autre).

### 12.2 bounce handling, désinscription

- Tracking des délivraisons via le fournisseur (webhooks `Delivery`, `Bounce`, `SpamComplaint`, `Open` — l'open n'est pas une métrique de fiabilité et n'est pas utilisé comme KPI).
- Hard bounce (ex. `550 5.1.1`) → adresse marquée `bounced` dans `email_deliveries`, suppression de la liste marketing, exclusion des transactionnels non critiques.
- Soft bounce (4xx) → retry limité (3, backoff 1 h / 4 h / 24 h) puis déclassement en `deferred`.
- Complaint spam → suppression immédiate et blacklist.
- **Unsubscribe** : obligatoire pour le marketing uniquement, via en-tête `List-Unsubscribe` (RFC 8058) + `List-Unsubscribe-Post: List-Unsubscribe=One-Click`, et lien visible dans chaque e-mail (§14.3 : les préférences marketing ne doivent pas supprimer le transactionnel indispensable). Un job `email.suppress` applique la suppression en < 24 h.

### 12.3 Generation PDF

- **Moteur retenu : `@react-pdf/renderer`** (React, compose avec les composants existants, rendu Node sans navigateur). Repli : `pdfkit` pour les documents très simples. `playwright` headless n'est **pas** retenu : coût mémoire, latence (300–800 ms), et dépendance Chromium à maintenir — il reste une option pour un futur rendu HTML haut de gamme, isolée derrière la même interface `DocumentRenderer`.
- **Polices CJK** : `Noto Sans SC` (subset `chinese-simplified` + `latin`), embarquée via `Font.register` à partir d'un fichier WOFF/TTF versionné dans le dépôt `assets/fonts/`, **jamais** téléchargée au runtime ; sous-ensembleage pour ne garder que les glyphes réellement utilisés lorsque le jeu de caractères est stable (optimisation secondaire, non critique). Le même fichier de police est partagé avec le front (§3.1) pour garantir la cohérence visuelle entre l'écran et le PDF.
- **Test visuel obligatoire** (§14.5) : rendu d'un PDF de référence en FR, EN et ZH-CN, comparaison pixel avec une référence approuvée (Playwright screenshot du PDF rendu en image), job CI `pdf-visual` ; tout débordement de caractères CJK fait échouer la CI.
- **Cache des documents** : le PDF d'une facture est généré une fois, stocké dans le bucket privé `fanours-prod-documents/invoices/{orderId}/{seq}.pdf`, et servi par **presigned URL de 5 minutes** via `GET /api/documents/:id` (autorisation par propriétaire ou rôle, §21.2). Le contenu binaire n'est jamais envoyé par l'API en JSON.
- **Règle absolue : aucun PDF dans le bundle public ni dans le CDN public**. Le bucket documents est privé, `block public access` activé, aucune règle CDN ne pointe dessus. Les e-mails de commande renvoient à l'application authentifiée (§14.3 CTA vers page authentifiée), pas vers un lien public permanent.
- **Déterminisme** : le PDF contient `documentId`, `version`, `locale`, `issuedAt` et l'instantané des conditions (§20.2) ; une modification après émission passe par un avoir (§14.5), jamais par une réédition silencieuse.

### 12.4 Facturation, numérotation, conservation

- Numérotation par pays et type de document (§14.5), séquence **non modifiable** (gap autorisé, jamais de réutilisation d'un numéro) ; garantie d'unicité au niveau base (séquence PostgreSQL par exercice + type).
- Distinction stricte : facture commerciale/export ≠ reçu ≠ facture fiscale ≠ avoir (§14.5, §11.8).
- Conservation : PDF et métadonnées conservés 10 ans dans le bucket privé (aligné sur §9.3), avec export possible vers le cabinet comptable dans un format convenu (§16.7).

---

## 13. PWA et service worker

### 13.1 Strategie de cache

| Requête | Strategie | Cache | Règle |
|---|---|---|---|
| Shell applicatif (HTML racine, JS/CSS critiques, icônes, manifest) | **Precache** (Workbox `precacheAndRoute`) | `fanours-precache-v{VERSION}` | versionné par le build, `revision` = hash du fichier |
| Assets publics statiques (`/_next/static/*`, images CDN) | **CacheFirst** | `fanours-assets-v{VERSION}` | TTL long, `maxEntries` borné (100) |
| Pages catalogue / catégories / fiches produit | **NetworkFirst** | `fanours-pages-v{VERSION}` | `networkTimeoutSeconds: 3`, repli sur cache si hors ligne, TTL 24 h |
| Images produit distantes | **CacheFirst + expiration** | `fanours-img-v{VERSION}` | `maxEntries: 200`, `maxAgeSeconds: 604800` |
| API catalogue en lecture (GET `/api/products*`) | **StaleWhileRevalidate** | `fanours-api-v{VERSION}` | TTL 60 s, `maxEntries: 50` |
| API mutation (POST/PUT/PATCH/DELETE), `/api/checkout/*`, `/api/cart/*` | **NetworkOnly** | aucun | jamais de cache, jamais de file d'attente implicite |
| Pages compte, commandes, factures, documents | **NetworkOnly** | aucun | §22.3 : ne jamais mettre en cache les factures, commandes, données privées |
| Webhooks, endpoints admin | **NetworkOnly** | aucun | hors périmètre SW |
| Requêtes avec `Authorization` ou `Cookie` | **NetworkOnly + bypass** | aucun | garde-fou applicatif : le SW ignore toute requête authentifiée |

### 13.2 Regle absolue sur les donnees privees

Le service worker ne met **jamais** en cache les réponses contenant une donnée personnelle. Implémentation : règle Workbox `registerRoute` avec `method: 'GET'` **et** un filtre qui exclut tout chemin sous `/api/account`, `/api/orders`, `/api/documents`, `/api/cart`, `/api/checkout`, `/api/quotes`, `/api/invoices`, `/api/admin`, et toute requête dont l'URL contient un jeton signé ; plus une règle de non-mise en cache pour les réponses dont l'en-tête `Cache-Control` contient `private` ou `no-store` (défense en profondeur côté serveur). Un test E2E vérifie qu'après une session authentifiée, le cache du SW ne contient aucune URL `/api/*` privée (`caches.keys()` + `cache.match()` inspectés dans le test).

### 13.3 Paiement et hors-ligne

- Le checkout **exige une connexion** : si `navigator.onLine === false` ou si la requête échoue, l'interface affiche un état explicite « connexion requise pour finaliser la commande » (§22.3) ; aucune commande n'est déclarée payée localement (§22.3, §26.3).
- Aucune file d'attente hors-ligne pour les mutations : un panier peut être conservé localement (brouillon) mais la validation finale part vers le serveur, qui recalcule prix, taxes, stock et frais (§6.5, §9.1).
- Le panier hors-ligne est un brouillon local non autoritatif ; à la reconnexion, fusion contrôlée avec le panier serveur.

### 13.4 Mode hors-ligne

- Page d'indisponibilité dédiée : navigation vers `/offline` (prérendu), message clair « Vous êtes hors ligne — certaines fonctions (paiement, suivi) nécessitent une connexion » (§22.3).
- Le catalogue en cache reste consultable en lecture, avec un bandeau « contenu mis en cache, prix à confirmer à la reconnexion ».
- Notifications push : non utilisées en hors-ligne, et leur activation est conditionnée au consentement (§14.1) et à un support navigateur fiable.

### 13.5 Deploiement du service worker

- **Versionnement** : le nom du cache inclut le hash du build (`fanours-precache-v{VERSION}`) ; un nouveau build change le `VERSION` → nouveaux caches, anciens purgés.
- **Activation contrôlée** : `skipWaiting` **désactivé** par défaut ; un `SKIP_WAITING` message est envoyé par l'application quand l'utilisateur accepte « mise à jour disponible » (bandeau non intrusif). Exception : un `SKIP_WAITING` forcé est autorisé pour un build de correction critique (flag `force_sw_update`).
- **Nettoyage** : à l'`activate`, suppression de tout cache dont le nom ne correspond pas à la version courante ; `clients.claim()` activé uniquement après activation confirmée.
- **Éviter le mixed content / la page figée** : le SW est servi depuis la racine (`Service-Worker-Allowed: /`) via `next-pwa` ou `@serwist/next`, avec `Cache-Control: no-cache` sur `sw.js` (le navigateur doit toujours revalider) et `max-age=31536000, immutable` sur les assets versionnés.
- **Rollback** : revenir à la version précédente redéploie un `sw.js` antérieur qui purge les caches de la version plus récente au prochain `activate` ; le cache applicatif ne contient aucune donnée privée (§13.2), donc aucune perte de données client n'est possible.

---

## 14. Feature flags et rollout

### 14.1 Systeme de flags

Table `feature_flags` en base (source de vérité, lisible et auditable par l'admin — §16.13 « Feature flags »), avec cache Redis invalidé par pub/sub et TTL 30 s (la lecture d'un flag ne doit jamais ralentir le rendu d'une page).

```sql
CREATE TABLE feature_flags (
  key          TEXT PRIMARY KEY,
  description  TEXT NOT NULL,
  type         TEXT NOT NULL CHECK (type IN ('boolean','percentage','rollout','variant')),
  default_value JSONB NOT NULL,
  enabled      BOOLEAN NOT NULL DEFAULT false,
  rules        JSONB NOT NULL DEFAULT '[]'::jsonb,  -- ciblage env/role/pays/segment
  expires_at   TIMESTAMPTZ,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_by   UUID
);
CREATE TABLE feature_flag_changes (
  id          BIGSERIAL PRIMARY KEY,
  key         TEXT NOT NULL,
  old_value   JSONB,
  new_value   JSONB,
  actor_id    UUID,
  reason      TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Évaluation côté serveur, dans le middleware/guard avant la route (jamais seulement dans l'UI) :

```ts
// packages/flags/src/evaluate.ts
export type FlagContext = {
  env: 'local' | 'test' | 'staging' | 'production';
  userId?: string;
  role?: string;
  country?: string;
  segment?: 'b2b' | 'b2c';
  accountId?: string;
};

export function evaluateFlag(
  flag: { type: string; default_value: unknown; enabled: boolean; rules: Rule[] },
  ctx: FlagContext,
): unknown {
  if (!flag.enabled) return flag.default_value;
  if (flag.type === 'boolean') return true;
  if (flag.type === 'percentage') {
    // Hachage déterministe : même utilisateur => même cohorte d'un déploiement à l'autre
    const bucket = hashToBucket(`${flag.key}:${ctx.userId ?? 'anon'}`);
    return bucket < Number(flag.default_value);
  }
  for (const rule of flag.rules) {
    if (rule.env && rule.env !== ctx.env) continue;
    if (rule.roles && !rule.roles.includes(ctx.role ?? '')) continue;
    if (rule.countries && !rule.countries.includes(ctx.country ?? '')) continue;
    if (rule.segments && !rule.segments.includes(ctx.segment ?? '')) continue;
    if (rule.accountIds && !rule.accountIds.includes(ctx.accountId ?? '')) continue;
    return rule.value;
  }
  return flag.default_value;
}
```

- Les flags sont **persistés et audités** : tout changement est écrit dans `feature_flag_changes` avec acteur et motif, et apparaît dans l'audit log (§19.7).
- Un flag de paiement ne peut pas être activé en production sans double validation (deux rôles différents) : c'est une action financière (§19.2).
- **Nettoyage des flags temporaires** : champ `expires_at` obligatoire pour tout flag temporaire ; un job cron quotidien (`flags.expire`) alerte 7 j avant l'échéance et **désactive automatiquement** le flag à l'échéance. Un rapport mensuel liste les flags périmés non supprimés (dette de configuration).

### 14.2 Usage pour le decoupage de phases (phasage §27)

| Flag | Phase | Cible | Description |
|---|---|---|---|
| `checkout.v2` | Phase 2 | 0 % → 5 % → 25 % → 100 % | Nouveau tunnel de checkout (canary + métriques de garde) |
| `pricing.tiered_b2b` | Phase 2/3 | pays + segment B2B | Tarification dégressive B2B par pays |
| `quotes.workflow` | Phase 3 | rôles sales/commercial | Cycle devis multi-étapes |
| `trade.motor_v2` | Phase 3 | pays prioritaires | Moteur de règles douanières versionné (n shadow-read puis bascule) |
| `tracking.sync_v2` | Phase 3 | transporteur | Synchronisation tracking par webhook |
| `admin.dashboard_v2` | Phase 4 | rôles admin | Nouveau dashboard |
| `pdf.contracts_v2` | Phase 5 | tous | Nouveau moteur de documents |
| `pwa.install_prompt` | Phase 6 | tous | Invitation d'installation PWA |
| `payments.provider_b` | Phase 5 | pays | Second prestataire de paiement (si §10.1 impose un changement) |

Règle : un flag qui gate une migration destructif n'est jamais activé en même temps que la migration ; l'ordre est expand (code) → activate flag → observe (≥ 7 j) → contract (migration, §5.5).

---

## 15. Runbooks

Huit runbooks obligatoires, stockés dans `ops/runbooks/` (Markdown, versionnés en Git, revus semestriellement). Structure de template imposée :

```markdown
# RB-<SLUG> — <Titre>
## 1. Symptôme
   (Ce que l'on observe : alerte, message utilisateur, métrique, captures d'écran)
## 2. Impact
   (Étendue, utilisateurs affectés, perte de chiffre d'affaires estimée, régions)
## 3. Diagnostic
   (Étapes numérotées, requêtes SQL/commandes exactes, dashboards à ouvrir, ordre de vérification)
## 4. Escalade
   (Qui appeler, dans quel délai, canal, ce qu'on lui transmet)
## 5. Résolution
   (Action corrective immédiate, puis action définitive ; commande exacte à exécuter)
## 6. Vérification
   (Requête/commande qui prouve le rétablissement ; métriques à observer et pendant combien de temps)
## 7. Post-mortem
   (Chronologie, cause racine, actions avec owner et échéance, mise à jour du runbook)
## 8. Annexes
   (Tableaux de correspondance, dashboards, contacts, logs utiles)
```

| # | Slug | Objet | Criticité | Déclencheur principal |
|---:|---|---|---|---|
| 1 | `RB-001-paiement-indisponible` | PSP indisponible, taux d'échec paiement > 1,5 %, webhooks non reçus | SEV1 | `payment_failure_rate` |
| 2 | `RB-002-prestataire-email-panne` | File e-mail saturée, bounce > 5 %, fournisseur SMTP en panne | SEV2 | `worker_queue_depth{queue="email"}` |
| 3 | `RB-003-defaillance-transporteur` | API transporteur en erreur, estimations indisponibles, tracking bloqué | SEV2 | `shipping_estimate_failure_rate` |
| 4 | `RB-004-incident-securite` | Fuite de données, compte compromis, secret exposé, MFA bypass | SEV1 | Toute alerte sécurité ou signalement |
| 5 | `RB-005-restauration-bd` | Restauration PostgreSQL depuis PITR/snapshot, promotion, vérification | SEV1 | Corruption, bug de données, erreur de migration |
| 6 | `RB-006-5xx-latence` | Erreurs 5xx massives, latence p95 dégradée, saturation pool DB | SEV1/SEV2 | `http_request_duration_seconds` |
| 7 | `RB-007-certificat-ssl` | Expiration, chaîne incomplète, HSTS, redirection HTTPS, renouvellement ACME | SEV1 | `cert_expiry_seconds` |
| 8 | `RB-008-stock-donnees-incoherentes` | Sur-vente, stock négatif, divergence stock/commandes, DLQ massive | SEV2 | `stock_reservation_failure_total`, réconciliation |

Règles : le contenu est rédigé en français normal (pas compressé), lisible par un operator non-développeur ; chaque runbook est testé au moins une fois par an en exercice (jeu de données de staging) ; un runbook sans commande exacte est rejeté en revue.

---

## 16. Couts et budget infra

### 16.1 Estimation mensuelle au lancement (faible trafic : < 10 k visiteurs/mois, < 500 commandes/mois)

| Composant | Offre retenue | Faible | Moyen | Fort | Commentaire |
|---|---|---:|---:|---:|---|
| Edge VM (proxy + `web` + `api` + `worker`) | 2 × VPS 4 vCPU / 8 Go / 200 Go SSD | 45 € | 90 € | 240 € | scaling vertical d'abord, 2 VM minimum pour la redondance |
| CDN + WAF (Cloudflare Pro) | abonnement | 20 € | 25 € | 70 € | bandwidth élevé, cache HTML public |
| PostgreSQL managé (2 vCPU / 4 Go / 100 Go, PITR) | instance managée UE | 55 € | 120 € | 320 € | + réplication lecture si lecture dashboard |
| Redis managé | 1 Go | 15 € | 30 € | 90 € | non critique |
| Objet S3 + CDN images | stockage + requêtes | 8 € | 25 € | 120 € | lifecycle Glacier (§9.3) |
| E-mail transactionnel | 50 k e-mails/mois | 12 € | 35 € | 140 € | tarification au volume, provider à validate (§21.4) |
| Observabilité (métrique/log/trace/erreur) | self-hosted Loki+Prometheus+Grafana+Sentry | 30 € | 70 € | 220 € | ou SaaS, coût par ingestion à surveiller |
| Supervision externe / uptime | 5 monitors | 0 € (synthetics sur l'edge) | 10 € | 30 € | monitoring depuis 3 régions |
| Certificats | Let's Encrypt + ACME DNS | 0 € | 0 € | 0 € | coût = temps d'exploitation |
| Secrets (Vault/KMS) | self-hosted | 5 € | 10 € | 25 € | ou service managé 30 €+ |
| Sauvegardes hors site (S3 répliqué) | stockage + trafic | 6 € | 15 € | 60 € | retention 35 j (§9.2) |
| Tracking transporteur / API transport | quotas | 0 € (inclus) | 40 € | 200 € | dépend du contrat (§21.4) |
| E-mail de domaine / DNS | hébergeur DNS | 2 € | 2 € | 2 € | — |
| **TOTAL** | | **~198 €** | **~472 €** | **~1 518 €** | hors PSP (commission), hors licences SaaS métier |

Hypothèse de calcul « moyen » : 100 k visiteurs/mois, 5 k commandes, 250 k e-mails, 400 Go d'assets, 1,2 M requêtes API, 1 job/s en pointe. Les coûts PSP (commission par transaction) et le coût d'acquisition ne sont pas inclus (hors périmètre infra).

### 16.2 Leviers d'optimisation identifies

1. **Cache CDN des pages publiques** (ISR + `s-maxage`) : divise la charge SSR par 5–10x pour le trafic catalogue ; levier n°1.
2. **Statique-first** : pages publiques en `generateStaticParams` / ISR, API catalogue servie par cache Redis multi-niveaux (mémoire LRU + Redis).
3. **Colonnes et index** : éviter un moteur de recherche dédié tant que le volume ne le justifie pas (§5.1) ; passer à un moteur externe si le catalogue dépasse ~50 000 SKU.
4. **Pool de connexions** : PgBouncer en mode transaction, `max_connections` PostgreSQL réduit (2 VM × 20 au lieu de 2 × 100), ce qui permet de rester sur une instance 2 vCPU.
5. **Worker dimensionné** : started à 1 instance, autoscale sur `worker_queue_depth` ; jobs PDF sur une file séparée avec concurrence basse (le PDF est le principal consommateur CPU).
6. **Stockage** : lifecycle Glacier sur les assets et les documents (§9.3), déduplication des images identiques, `AVIF` ~50 % plus léger que JPEG → moins de bande passante CDN et de coût S3.
7. **E-mail** : suppression des doublons par clé d'idempotence, désinscription immédiate des hard bounces, sélecteurs DKIM limités, rendu texte seul en fallback.
8. **Observabilité** : sampling raisonné (10 % en prod, 100 % sur erreur) et rétention courte des logs (30 j) ; ne pas envoyer les logs applicatifs à deux agrégateurs.
9. **Certificats** : Let's Encrypt avec DNS-01 pour obtenir un certificat wildcard et éviter les limitations de débit HTTP-01.
10. **Réservation** : 2 VM applicatives au lieu de 3 pendant la phase 0/1 (le risque est accepté et documenté) ; upgrade à 3 VM dès que le CA > 99,9 % exigé ou que le budget le permet.
11. **Prévoir la sortie** : le passage à Cloud Run/ECS se déclenche par des seuils chiffrés (§2.4) et non par intuition ; le coût est simulé chaque trimestre.

### 16.3 Garde-fous budgétaires

- Alerte budget CloudWatch/budget provider à 50 %, 80 % et 100 % du budget mensuel ; au-delà de 80 %, revue des métriques de coût (trafic CDN, logs, e-mails).
- Tableau de bord `infra-cost` : coût par composant, par environnement, par mois, avec projection ; revue mensuelle de 30 min.
- Budget mensuel cible de l'infra au lancement : **≤ 500 €/mois** hors PSP et hors licences métier ; toute ligne > 30 % du budget doit avoir un owner et une justification écrite.
- Les environnements non production sont **arrêtés en dehors des heures ouvrées** (scaling-to-zero pour staging) : économie d'environ 35 % du budget non-prod.

---

### Annexe — Décisions d'infrastructure à faire valider (lien avec §28.2)

| # | Décision | Recommandation | Impact si inverse |
|---:|---|---|---|
| 1 | Prestataire de paiement (§10.1, §28.2-1) | Aucun choix technique : interface `PaymentProvider` abstraite, Stripe-compatible | Réécriture du checkout si non éligible |
| 2 | Hébergeur et région des données (§28.2-10) | VPS UE + BDD managée UE | Transfert de données hors UE, RGPD/UK GDPR (§19.6) |
| 3 | SLA de maintenance et RPO/RTO (§28.2-11) | 99,5 % mensuel hors maintenance, RPO 24 h, RTO 8 h | Budget d'astreinte et architecture multi-région |
| 4 | Fournisseur d'e-mail et de suivi transporteur (§21.4) | Postmark (transactionnel) + API transporteur retenue en Phase 0 | Aucun blocage technique : adaptateurs interchangeable |
| 5 | Budget infra mensuel (§16.3) | 500 € au lancement | Contrainte sur le dimensionnement et les seuils d'alerte |



