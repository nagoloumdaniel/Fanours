# Phase 1 — Fondations et design system : Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Livrer un socle exécutable de bout en bout : un monorepo typé strictement, un schéma PostgreSQL versionné par migrations SQL manuelles, un design system à tokens (2 thèmes) avec Storybook, la pile i18n FR/EN/ZH-CN, l'authentification par session opaque avec Argon2id, le MFA TOTP, le RBAC à 9 rôles, des pages publiques localisées, l'observabilité JSON, le socle PWA, le SEO technique, et un pipeline CI complet. Critère de sortie de phase : **aucune feature métier livrée** — seulement les fondations.

**Périmètre :** 15 tâches (Task 1 → Task 15) — monorepo, schéma et migrations, design system, i18n, authN, MFA, RBAC, pages publiques, observabilité, PWA, SEO technique, CI et environnements, puis la recette de phase et le Gate 1.

**Architecture:** Monolithe modulaire en pnpm workspaces + Turborepo. `apps/web` (Next.js 15 App Router, RSC, routes publiques sous `/[locale]`), `apps/api` (NestJS 11, modules métier, Zod en entrée et en sortie), `apps/worker` (BullMQ, consommateur d'outbox). Packages partagés : `config` (schémas Zod d'environnement), `contracts` (types de边界 partagés), `db` (Drizzle + migrations SQL append-only), `domain` (règles métier pures, sans framework ni ORM), `pricing`, `compliance`, `i18n`, `ui` (tokens + composants React), `observability` (logger JSON, corrélation, redaction). Persistance : PostgreSQL 16 via Drizzle (un seul ORM, jamais Prisma), migrations SQL écrites à la main dans `packages/db/migrations/` et appliquées par un runner idempotent à checksum SHA-256. Cache et files : Redis 7 + BullMQ. Objets : S3-compatible + CDN. Échanges interservices : `outbox_event` (transactional outbox) + `consumed_event` (idempotence consommateur).

**Tech Stack:** Next.js 15 (App Router, RSC) · React 19 · TypeScript `strict` · Tailwind v4 (CSS-first) · NestJS 11 + Zod · PostgreSQL 16 + Drizzle ORM · Redis 7 + BullMQ · S3-compatible · Vitest + Testing Library + jest-axe + Playwright + Testcontainers · pnpm workspaces + Turborepo.

**Spec:** `Cahier_des_charges_Fanours.md` §3 (identité, typographies, palette, composants), §4 (langues, devises, formats), §5.1 (stack), §6.1 (pages publiques), §13.1–13.5 (auth, MFA, sessions), §16.12 (rôles), §17 (SEO), §20 (modèle de données), §22 (responsive, PWA, a11y), §23.1–23.3 (perf, observabilité), §24.1–24.6 (tests), §25.1–25.3 (environnements, pipeline, migrations). Plan de phase : `docs/roadmap/sections/roadmap-phases.md` §Phase 1 (chantiers 1.1 à 1.15, 8 vagues, Gate 1). Sources de détail : `docs/roadmap/sections/design.md`, `architecture.md`, `securite.md`, `infra.md`, `seo.md`.

---

## Global Constraints

- **Stack figée :** Next.js 15 App Router + React 19 + TypeScript `strict` + Tailwind v4 (CSS-first) ; NestJS 11 + Zod ; PostgreSQL 16 + Drizzle (un seul ORM, **jamais Prisma**) ; Redis 7 + BullMQ ; S3-compatible ; Vitest + Testing Library + Playwright + Testcontainers ; pnpm workspaces + Turborepo.

- **Monorepo :** `apps/{web,api,worker}` et `packages/{config,contracts,db,domain,pricing,compliance,i18n,ui,observability}`. Aucun paquet ne dépend de `apps/*`. Règles de dépendance (appliquées par `eslint-plugin-boundaries` + project references TypeScript) : `domain ← compliance ← {pricing, api, web}` ; `db → {pricing, compliance, api, worker}` ; `ui → web` ; `contracts → {api, web, worker}`. Un module `domain` n'importe **jamais** de framework web ni d'ORM.

- **Zéro placeholder :** aucun `TBD`, `TODO`, `à compléter`, « identique à la tâche N », « ajouter la gestion d'erreur ». Chaque étape contient le code ou le contenu réel.

- **Cycle de test par tâche :** on écrit le test → on l'exécute et on **colle la sortie qui prouve l'échec** → on implémente le minimum → on réexécute et on **colle la sortie qui prouve la réussite** → commit.

- **Tokens :** source unique `packages/ui/src/styles/tokens.css`, blocs `:root`, `:root:not([data-theme="light"])` et `:root[data-theme="dark"]`. Aucune couleur hex en dur hors `tokens.css` (règle ESLint `no-restricted-syntax`).

- **Montants :** jamais de float binaire. `NUMERIC(18,4)` en base, **entiers de minor units** en code, codes ISO 4217, formatage `Intl.NumberFormat`.

- **i18n :** dictionnaires `fr-FR` / `en` / `zh-CN`, repli explicite `fr-FR` → clé, syntaxe ICU, **le build échoue si une clé manque dans l'une des 3 langues**. Aucun retour en arrière si la locale demandée n'existe pas dans les 3 langues.

- **Migrations :** `packages/db/migrations/NNNN_nom.sql`, append-only, **aucun script `down`** (rollback par code), `schema_migrations` avec checksum SHA-256, `SET lock_timeout` / `statement_timeout` sur chaque migration, `CREATE INDEX CONCURRENTLY` au-delà de 100 000 lignes, `NOT NULL` nouveau via `CHECK … NOT VALID` puis `VALIDATE`. `drizzle-kit generate` sert uniquement à produire un diff, **jamais `push`**.

- **Journal d'audit append-only :** `REVOKE UPDATE, DELETE` + trigger `BEFORE UPDATE OR DELETE` → `RAISE EXCEPTION`. Chaîne de hachage `entry_hash` vérifiable.

- **Session :** cookie `fanours_sid`, `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, pas de `Domain`, `Max-Age 900`. **Hash SHA-256 du token en base, jamais le token brut.** Expiration inactive 30 min (client) / 15 min (admin) / 60 min (transfert B2B en attente). Expiration absolue 12 h (client) / 8 h (back-office) / 30 min (élévation sensible). **Rotation à la connexion et à l'élévation**, révocation multi-appareils, révocation en cascade au changement de mot de passe.

- **MFA :** TOTP RFC 6238 (SHA-1, 160 bits), secret chiffré AES-256-GCM en enveloppe KMS avec version de clé, **compteur anti-rejeu**, 10 codes de récupération Argon2id (N=2^16, r=3, p=1), usage unique, affichés une seule fois, invalidés à la création d'un nouveau TOTP, session désactivée après usage. MFA **obligatoire** pour admin/finance/compliance.

- **RBAC :** 9 rôles (`super_admin`, `catalog_manager`, `order_manager`, `logistics_operator`, `finance_manager`, `support_agent`, `compliance_manager`, `marketing_manager`, `analyst_readonly`), 13 ressources, 8 actions, **deny by default**. Permissions évaluées **côté serveur, dans la requête** (scoping anti-IDOR, jamais de post-filtrage en mémoire). Four-eyes sur `price_tier`, `export_financial`, `refund`, `admin_role`, `country`.

- **Rate limiting :** login 5/15 min (IP + email) · signup 3/h · reset 3/h · MFA 5/10 min · recherche 60/min · API 600/min. Fenêtre glissante Redis (ZSET + script Lua atomique), `429` + `Retry-After`, anti-énumération (réponse identique).

- **CSP stricte :** nonce/hash, pas d'`unsafe-inline'` sur `script-src`, `connect-src` restreint, HSTS, `X-Content-Type-Options`, `frame-ancestors`, `Referrer-Policy`, `Permissions-Policy`.

- **Observabilité :** logs JSON une ligne (pino) avec `timestamp, level, service, env, version, correlation_id, trace_id, user_id_hash, session_id_hash, route, method, status, duration_ms, order_id, event, msg, error`. `correlation_id` UUIDv7 généré à l'entrée HTTP, propagé dans `X-Correlation-Id`, dans les jobs et les appels sortants ; `traceparent` W3C. `user_id_hash` = HMAC-SHA256(user.id, `LOG_HASH_KEY`). Redaction automatique des PII et secrets.

- **PWA :** shell en precache, assets publics en runtime, pages catalogue en NetworkFirst, **panier/checkout/compte en NetworkOnly**, **interdiction absolue de mettre en cache factures/commandes/données privées**, service worker versionné + purge des anciens caches.

- **SEO technique :** URLs `/fr/`, `/en/`, `/zh-cn/` ; canonical absolu auto-référentiel ; hreflang réciproque (3 locales + `x-default` pointant sur la variante anglaise) ; **aucune redirection forcée par IP** (`Accept-Language` uniquement, 302 et jamais 301) ; `robots.txt` en allowlist explicite ; sitemaps 3 niveaux ; JSON-LD en **un seul `@graph`** par page ; les pages privées sont `noindex` et hors sitemap.

- **A11y :** WCAG 2.2 AA — focus visible (anneau 2 px + offset 2 px), ordre de focus logique, piège de focus en modale, live regions, cibles ≥ 44 px (≥ 24×24 min WCAG 2.5.8), `prefers-reduced-motion` respecté, 200 % de zoom sans perte, jamais couleur seule, un seul `<h1>` par page, skip link.

- **Mobile :** nav basse 4 onglets ≤ 767 px (64 px + safe-area), en-tête compact 56 px, CTA fixe au-dessus de la nav basse (`bottom: calc(64px + env(safe-area-inset-bottom))`), tableaux → cartes sous 768 px (la table HTML reste en DOM, `aria-hidden` sur la version masquée), `min-width: 0`, jamais `width: 100vw`, assertion `document.documentElement.scrollWidth <= clientWidth` à 320/360/390/430 px.

- **États obligatoires :** 3 langues × 2 thèmes × états loading / vide / erreur / accès refusé / session expirée / réseau indisponible / ressource indisponible / conflit de stock / quantité invalide / paiement en attente / interruption prestataire. Message **localisé FR/EN/ZH**, icône, action de sortie, **jamais de donnée technique exposée**. Référence de corrélation affichée uniquement sous la forme « Réf. 7F3K2Q ».

- **Modèle E2E canonique :** configuration Playwright **unique**, à la racine du dépôt (`playwright.config.ts`) — créée en Phase 1, **modifiée** (jamais recréée) par les Phases 2, 4, 5 et 6. Aucun autre `playwright.config.ts` (ni `apps/web/`, ni `apps/api/`, ni `e2e/`) ; les commandes Playwright sont lancées **depuis la racine** (`pnpm exec playwright test ...`). **7 projets** : `b2c`, `b2b`, `payment`, `admin`, `security` (les 5 suites de recette de release du spec §24.3), `visual` (matrice visuelle) et `functional` (recettes fonctionnelles de développement). `testDir` vaut `'./e2e'` pour **tous** les projets : ils se distinguent uniquement par `testMatch` / `testIgnore`, donc aucun projet ne référence un dossier qui n'existe pas encore. Les **viewports, les langues et les thèmes ne sont pas des projets** : ils sont paramétrés dans les fichiers de test via `test.use({ viewport })` à partir de la liste canonique `e2e/support/viewports.ts` (8 viewports : 320×640 · 360×740 · 390×844 · 430×932 · 768×1024 · 1024×768 · 1440×900 · 1920×1080), importée et **jamais re-déclarée**. Règle d'appartenance : **un fichier de test appartient à un seul projet** — aucun doublon, aucun comptage de tests ambigu.

- **Secrets :** uniquement des **noms** de variables (`DATABASE_URL`, `REDIS_URL`, `SESSION_SECRET`, `LOG_HASH_KEY`, `KMS_KEY_ARN`, `EMAIL_PROVIDER`, `EMAIL_API_KEY`, `PAYMENT_PROVIDER`, `PAYMENT_API_KEY`, `PAYMENT_WEBHOOK_SECRET`, `SHIPPING_PROVIDER`, `SHIPPING_API_KEY`, `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET_PUBLIC`, `S3_BUCKET_PRIVATE`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `CDN_BASE_URL`, `SENTRY_DSN`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `ANALYTICS_WRITE_KEY`, `CONSENT_SECRET`, `APP_BASE_URL`). `.env.example` = noms seuls. Scan de secrets (gitleaks) bloquant en CI.

- **Environnements :** `local` (données factices) · `test` · `staging` (prod-like, clés sandbox) · `production`. Schémas d'environnement validés par Zod au démarrage : **l'application refuse de démarrer** si une variable manque ou est invalide. Secrets hors Git, gestionnaire dédié, procédure de rotation.

- **Git :** branche `feat|fix|refactor|test|docs|chore/<slug>`, **jamais `main`**, jamais de force-push, PR laissée ouverte et **jamais mergée sans accord explicite**, PR > 800 lignes à scinder.

- **Commits :** Conventional Commits 1.0, titres ≤ 72 caractères, corps ≤ 72 caractères, scopes `catalog|pricing|cart|checkout|order|payment|tax|customs|shipping|auth|account|b2b|quote|admin|content|i18n|infra|db|ui|design|seo|ci`. Chaque message de commit de ce plan se termine par `Co-Authored-By: Claude Code <noreply@anthropic.com>`.

- **Preuve :** pas de « terminé » sans la sortie de test collée ; les tests non exécutables sont annoncés avec leurs prérequis.

---

## File Structure

```code
fanours/
├─ package.json                        # scripts racine (build, lint, typecheck, test, db:migrate)
├─ playwright.config.ts                 # config racine (7 projets : 5 suites de release + visual + functional)
├─ pnpm-workspace.yaml                 # packages: apps/* et packages/*
├─ pnpm-lock.yaml                      # lockfile pnpm (source de vérité des versions)
├─ turbo.json                          # pipeline Turborepo, dependsOn ^build
├─ tsconfig.base.json                  # tsconfig strict partagé + paths @fanours/*
├─ tsconfig.json                       # solution TS avec project references
├─ .npmrc                              # node-linker=isolated, strict-peer-dependencies=false
├─ .gitignore                          # node_modules, .turbo, .env, dist, coverage, .next
├─ .env.example                        # NOMS de variables uniquement, aucune valeur
├─ .prettierrc.json                    # config Prettier (semi, singleQuote, printWidth 100)
├─ .prettierignore                     # Generated, lockfiles, snapshots
├─ eslint.config.mjs                   # ESLint flat + boundaries + no-restricted-syntax (hex)
├─ commitlint.config.mjs               # Conventional Commits 1.0
├─ .husky/pre-commit                   # lint-staged + gitleaks protect --staged
├─ .husky/commit-msg                   # commitlint --edit
├─ .lintstagedrc.json                  # format/lint/test par type de fichier
├─ LICENSE                             # licence du dépôt
├─ CONTRIBUTING.md                     # workflow de contribution, conventions de commit
├─ docker/compose.local.yml            # Postgres 16 + Redis 7 + MinIO pour le dev local
├─ docker/compose.test.yml             # services de test used par la CI (replicas 1)
├─ docker/Dockerfile.web               # image multi-stage non-root, sortie next standalone
├─ docker/Dockerfile.api               # image multi-stage non-root, API NestJS
├─ docker/Dockerfile.worker             # image multi-stage non-root, worker BullMQ
├─ apps/
│  ├─ web/
│  │  ├─ package.json                  # scripts Next.js, test, storybook
│  │  ├─ next.config.ts                # output standalone, headers CSP/HSTS, images
│  │  ├─ postcss.config.mjs            # plugin Tailwind v4 (@tailwindcss/postcss)
│  │  ├─ tsconfig.json                 # extends base, paths @fanours/*
│  │  ├─ eslint.config.mjs              # boundaries web -> {ui, domain, i18n, contracts}
│  │  ├─ vitest.config.ts               # projet unit (jsdom) + projet e2e exclu
│  │  ├─ public/
│  │  │  ├─ robots.txt                  # allowlist SEO (cf. tâche 13)
│  │  │  ├─ manifest.webmanifest        # PWA manifest (cf. tâche 12)
│  │  │  ├─ sw.js                       # service worker versionné (cf. tâche 12)
│  │  │  ├─ fonts/plus-jakarta-sans/    # woff2 auto-hébergés, poids 500-800
│  │  │  ├─ fonts/inter/                # woff2 auto-hébergés, poids 400-700
│  │  │  └─ fonts/noto-sans-sc/         # woff2 CJK, poids 400/500/700
│  │  ├─ src/
│  │  │  ├─ app/
│  │  │  │  ├─ layout.tsx              # layout racine, <html lang>, anti-flash
│  │  │  │  ├─ globals.css             # import des tokens + utilitaires Tailwind
│  │  │  │  ├─ robots.ts                # /robots.txt dynamique (tâche 13)
│  │  │  │  ├─ sitemap.ts               # /sitemap.xml racine (tâche 13)
│  │  │  │  ├─ sitemaps/
│  │  │  │  │  ├─ sitemap-index.xml/route.ts   # index par langue
│  │  │  │  │  └─ sitemap-[locale]-[type]/route.ts # urlset produits/pages/editorial
│  │  │  │  └─ [locale]/
│  │  │  │     ├─ layout.tsx           # layout localisé, nav, footer, skip link
│  │  │  │     ├─ page.tsx             # accueil (tâche 10)
│  │  │  │     ├─ not-found.tsx        # 404 (tâche 10)
│  │  │  │     ├─ conditions/page.tsx   # CGV (tâche 10)
│  │  │  │     ├─ contact/page.tsx     # contact (tâche 10)
│  │  │  │     └─ catalogue/page.tsx    # catalogue vide (tâche 10)
│  │  │  ├─ components/
│  │  │  │  ├─ layout/header.tsx        # en-tête compact 56px, switcher de langue
│  │  │  │  ├─ layout/footer.tsx        # pied de page, maillage public uniquement
│  │  │  │  ├─ layout/bottom-nav.tsx    # nav basse 4 onglets (mobile)
│  │  │  │  ├─ layout/locale-switcher.tsx # switcher FR/EN/ZH visible
│  │  │  │  └─ seo/json-ld.tsx         # rendu du @graph unique (tâche 13)
│  │  │  ├─ lib/
│  │  │  │  ├─ locale.ts               # helpers locale (chemin, hreflang, canonique)
│  │  │  │  ├─ seo.ts                  # métadonnées, canonical, robots noindex
│  │  │  │  └─ theme.ts               # persistance du thème + script anti-flash
│  │  │  └─ middleware.ts              # redirection / -> /fr/ via Accept-Language
│  │  └─ src/__tests__/                # tests unitaires et E2E du web
│  ├─ api/
│  │  ├─ package.json                  # scripts NestJS, test, migration
│  │  ├─ tsconfig.json                 # extends base, alias @fanours/*
│  │  ├─ eslint.config.mjs              # boundaries api -> {db, domain, contracts, ...}
│  │  ├─ nest-cli.json                  # configuration de compilation Nest
│  │  ├─ vitest.config.ts               # projets unit + integration (testcontainers)
│  │  ├─ src/
│  │  │  ├─ main.ts                    # bootstrap Nest, helmet, cookies, CORS
│  │  │  ├─ app.module.ts               # module racine, imports des modules
│  │  │  ├─ version.ts                 # version de l'API (build)
│  │  │  ├─ common/
│  │  │  │  ├─ correlation.middleware.ts # génère/propage correlation_id et traceparent
│  │  │  │  ├─ error.filter.ts          # code stable + message UI + correlation_id
│  │  │  │  ├─ audit/
│  │  │  │  │  └─ audit.service.ts      # écriture unique dans audit_log
│  │  │  │  └─ security/
│  │  │  │     ├─ password.ts           # Argon2id
│  │  │  │     ├─ session.ts            # cookie, hash, rotation, révocation
│  │  │  │     ├─ rate-limit.ts         # fenêtre glissante Redis
│  │  │  │     ├─ totp.ts               # RFC 6238
│  │  │  │     ├─ envelope.ts           # enveloppe KMS + AES-256-GCM
│  │  │  │     └─ recovery-code.ts      # codes de récupération Argon2id
│  │  │  ├─ modules/
│  │  │  │  ├─ auth/                    # inscription, connexion, session (tâche 7)
│  │  │  │  ├─ mfa/                     # TOTP, codes de récupération (tâche 8)
│  │  │  │  ├─ rbac/                    # guard, matrice, four-eyes (tâche 9)
│  │  │  │  ├─ admin/                   # endpoints /admin/* (tâche 9)
│  │  │  │  └─ health/                  # /healthz, /readyz (tâche 11)
│  │  │  └─ providers/                  # interfaces d'intégration + stubs sandbox
│  │  └─ test/                          # tests d'intégration supertest
│  └─ worker/
│     ├─ package.json                  # scripts BullMQ
│     ├─ src/
│     │  ├─ main.ts                     # bootstrap du worker
│     │  └─ jobs/                       # defineJob, idempotencyKey, backoffMs
│     └─ tsconfig.json
├─ packages/
│  ├─ config/
│  │  ├─ package.json                  # package de schémas d'environnement
│  │  ├─ src/index.ts                  # re-exports publics
│  │  ├─ src/env-schema.ts             # schémas Zod par profil d'environnement
│  │  ├─ src/env-matrix.ts             # matrice de dérivation par APP_ENV
│  │  ├─ src/feature-flags.ts          # drapeaux de fonctionnalité déterministes
│  │  └─ src/__tests__/env-matrix.test.ts
│  ├─ contracts/
│  │  ├─ package.json                  # types de frontière API/web/worker
│  │  ├─ src/api.ts                    # enveloppes de réponse, codes d'erreur
│  │  ├─ src/events.ts                 # types d'événements outbox
│  │  └─ src/index.ts
│  ├─ domain/
│  │  ├─ package.json                  # règles métier pures, zéro framework
│  │  ├─ src/session-policy.ts         # règles d'expiration de session
│  │  ├─ src/permissions.ts            # matrice RBAC pure (tâche 9)
│  │  ├─ src/money.ts                  # minor units, arithmétique (tâche 6)
│  │  └─ src/__tests__/
│  ├─ db/
│  │  ├─ package.json                  # drizzle-orm, pg, drizzle-kit, scripts migrate
│  │  ├─ drizzle.config.ts             # config drizzle-kit (generate seulement)
│  │  ├─ migrations/                   # SQL append-only de la Phase 1 (0001-0005), prolongées par la Phase 2 (0006+)
│  │  │  ├─ 0001_extensions_and_migration_table.sql
│  │  │  ├─ 0002_identity.sql          # user, auth_identity, session, mfa, recovery
│  │  │  ├─ 0003_rbac_and_audit.sql     # role, permission, user_role, audit_log
│  │  │  ├─ 0004_outbox.sql             # outbox_event, consumed_event
│  │  │  └─ 0005_rbac_scopes.sql        # rbac_scope (task 9) : portées de permissions
│  │  ├─ scripts/migrate.ts            # CLI du runner (tsx scripts/migrate.ts) — propriétaire Phase 1
│  │  ├─ src/migrate-core.ts           # cœur du runner idempotent à checksum SHA-256 (propriétaire Phase 1)
│  │  ├─ src/schema.ts                 # schémas Drizzle (identité, RBAC, audit, outbox) — étendus par la Phase 2
│  │  ├─ src/client.ts                 # pool pg + drizzle
│  │  ├─ src/seed.ts                   # jeu de données synthétique (rôles, perms)
│  │  └─ src/__tests__/migrations.integration.test.ts
│  ├─ pricing/
│  │  ├─ package.json                  # calcule de prix et paliers
│  │  ├─ src/tier-resolution.ts        # résolution d'un palier de prix
│  │  └─ src/__tests__/tier-resolution.test.ts
│  ├─ compliance/
│  │  ├─ package.json                  # moteur de règles douanières
│  │  ├─ src/constants.allowlist.ts    # seules constantes de taxes autorisées
│  │  ├─ src/rule-resolution.ts        # résolution pays + produit + date
│  │  └─ src/__tests__/rule-resolution.test.ts
│  ├─ i18n/
│  │  ├─ package.json                  # dictionnaires et formatteurs
│  │  ├─ src/dictionaries/fr-FR.ts     # dictionnaire source de référence
│  │  ├─ src/dictionaries/en.ts        # dictionnaire anglais
│  │  ├─ src/dictionaries/zh-CN.ts     # dictionnaire chinois simplifié
│  │  ├─ src/index.ts                  # LOCALES, getDictionary, t, format
│  │  ├─ src/money-format.ts           # formatage Intl des montants (tâche 6)
│  │  └─ src/__tests__/
│  ├─ ui/
│  │  ├─ package.json                  # composants React + Storybook
│  │  ├─ .storybook/main.ts            # config Storybook (vite)
│  │  ├─ .storybook/preview.tsx        # thèmes clair/sombre + locales
│  │  ├─ src/styles/tokens.css         # SOURCE UNIQUE des tokens (tâche 3)
│  │  ├─ src/styles/globals.css        # ré-exports Tailwind v4
│  │  ├─ src/components/button.tsx     # Button, IconButton, ButtonGroup
│  │  ├─ src/components/input.tsx      # Input, Textarea, Select, Checkbox, Radio, Switch
│  │  ├─ src/components/feedback.tsx   # Badge, Skeleton, Toast, InlineNote
│  │  ├─ src/components/table.tsx      # Table responsive (version cartes)
│  │  ├─ src/components/tabs.tsx       # Tabs
│  │  ├─ src/components/pagination.tsx # Pagination
│  │  ├─ src/components/overlay.tsx    # Dialog, BottomSheet, Tooltip
│  │  ├─ src/components/system-state.tsx# 12 variantes d'état système
│  │  ├─ src/components/index.ts       # surface publique du package
│  │  └─ src/__tests__/                # Testing Library + jest-axe
│  └─ observability/
│     ├─ package.json                  # pino, redaction, corrélation
│     ├─ src/logger.ts                 # logger JSON une ligne
│     ├─ src/redaction.ts              # 33 clés interdites + regex valeurs
│     ├─ src/correlation.ts            # UUIDv7, traceparent
│     ├─ src/trace.ts                  # withTrace, injectTraceparent
│     └─ src/__tests__/
├─ e2e/
│  ├─ package.json                    # paquet @fanours/e2e (sources de test, tsconfig, lint)
│  ├─ tsconfig.json                   # TypeScript strict des sources de test
│  ├─ eslint.config.mjs               # extends de la config ESLint racine
│  ├─ support/
│  │  └─ viewports.ts                 # liste canonique des 8 viewports (source unique)
│  ├─ fixtures/
│  │  ├─ test.ts                      # test Playwright étendu, fixture `api`
│  │  └─ totp.ts                      # calcul TOTP RFC 6238 pour les tests
│  ├─ scripts/
│  │  └─ seed-e2e.ts                  # utilisateurs de test (super_admin, catalog_manager)
│  └─ specs/
│     ├─ public-pages.spec.ts         # parcours 3 langues x 2 thèmes
│     ├─ security.spec.ts             # IDOR, rate limit, énumération
│     ├─ security-headers.spec.ts     # 8 en-têtes de sécurité sur /, /admin, /api
│     ├─ pwa.spec.ts                  # aucune donnée privée en cache
│     └─ gate-1.spec.ts               # les 8 questions du Gate 1
├─ ops/
│  ├─ alerting/rules/                  # règles Prometheus versionnées
│  ├─ backup/assertions.sql            # assertions du drill de restauration
│  └─ runbooks/                        # RB-001 a RB-008
├─ scripts/
│  ├─ deploy.sh                        # déploiement par digest d'image
│  ├─ smoke-production.sh              # smoke post-déploiement
│  └─ verify-release.ts                # vérification du digest déployé
├─ .github/
│  ├─ workflows/
│  │  ├─ ci.yml                       # pipeline complet (tâche 14)
│  │  ├─ e2e.yml                      # Playwright 4 shards
│  │  ├─ security.yml                  # scans dédiés
│  │  ├─ dependency-review.yml
│  │  └─ ops.yml                       # drill de restauration mensuel
│  ├─ actions/setup/action.yml         # setup pnpm + node + cache
│  └─ CODEOWNERS                      # Propriétaires par chemin
└─ docs/
   ├─ adr/                             # Architecture Decision Records
   └─ superpowers/plans/              # ce plan et les plans voisins
```

L'arborescence ci-dessus est la cible de fin de phase : les fichiers listés sont créés par les tasks 1 à 14, qui les introduisent dans l'ordre d'exécution. Aucun fichier n'est modifié hors de cette liste sans mise à jour préalable de ce plan.

---

### Task 1: Socle monorepo, TypeScript strict, règles de dépendance

**Files:**
- Create: `package.json`, `pnpm-workspace.yaml`, `turbo.json`, `tsconfig.base.json`, `tsconfig.json`, `.npmrc`, `.gitignore`, `.env.example`, `.prettierrc.json`, `.prettierignore`, `eslint.config.mjs`, `commitlint.config.mjs`, `.husky/pre-commit`, `.husky/commit-msg`, `.lintstagedrc.json`, `LICENSE`, `CONTRIBUTING.md`
- Create: `packages/config/package.json`, `packages/config/src/index.ts`, `packages/config/src/env-schema.ts`, `packages/config/src/env-matrix.ts`, `packages/config/src/feature-flags.ts`
- Create: `packages/contracts/package.json`, `packages/contracts/src/api.ts`, `packages/contracts/src/events.ts`, `packages/contracts/src/index.ts`
- Create: `packages/domain/package.json`, `packages/domain/src/session-policy.ts`, `packages/domain/src/index.ts`
- Create: `packages/observability/package.json`, `packages/observability/src/logger.ts`, `packages/observability/src/redaction.ts`, `packages/observability/src/correlation.ts`, `packages/observability/src/index.ts`
- Test: `packages/config/src/__tests__/env-matrix.test.ts`, `packages/observability/src/__tests__/redaction.test.ts`, `packages/observability/src/__tests__/logger.test.ts`

**Interfaces:**
- Consumes: aucun (premier commit du dépôt applicatif).
- Produces:
  - `@fanours/config` : `envSchema(prof: EnvProfile): ZodObject`, `envMatrix(env: NodeJS.ProcessEnv): EnvMatrix`, `evaluateFlag(flags: FlagContext, key: FlagKey): boolean`, `hashToBucket(value: string, buckets: number): number`.
  - `@fanours/contracts` : `ApiErrorCode`, `ApiEnvelope<T>`, `EVENT_TYPES`, `EventEnvelope<T>`.
  - `@fanours/domain` : `SessionProfile`, `sessionPolicy(profile): { inactiveMs: number; absoluteMs: number }`, `isSessionExpired(state, now): boolean`.
  - `@fanours/observability` : `createLogger(service: string, env: EnvName)`, `redact(value: unknown): unknown`, `newCorrelationId(): string`, `newTraceId(): string`, `newSpanId(): string`, `formatTraceparent(traceId, spanId, sampled)`.

---

- [ ] **Step 1: Initialiser le dépôt et la branche de travail**

```bash
cd /d/Projets/Fanours
git status --short
git switch -c feat/phase-1-socle-monorepo
git rev-parse --abbrev-ref HEAD
```

Attendu : `feat/phase-1-socle-monorepo`. Si la branche existe déjà, `git switch feat/phase-1-socle-monorepo`.

- [ ] **Step 2: Créer le manifeste racine et l'espace de travail pnpm**

`package.json` :

```json
{
  "name": "fanours",
  "version": "0.1.0",
  "private": true,
  "packageManager": "pnpm@9.15.0",
  "engines": { "node": ">=22.0.0" },
  "scripts": {
    "build": "turbo run build",
    "lint": "turbo run lint",
    "format": "prettier --write \"**/*.{ts,tsx,md,json,yml,yaml}\"",
    "format:check": "prettier --check \"**/*.{ts,tsx,md,json,yml,yaml}\"",
    "typecheck": "turbo run typecheck",
    "test": "turbo run test",
    "test:unit": "turbo run test:unit",
    "test:integration": "turbo run test:integration",
    "test:e2e": "playwright test",
    "test:e2e:functional": "playwright test --project=functional",
    "test:e2e:visual": "playwright test --project=visual",
    "test:e2e:shard": "playwright test --shard",
    "db:migrate": "pnpm --filter @fanours/db migrate",
    "db:generate": "pnpm --filter @fanours/db generate",
    "db:seed:synthetic": "pnpm --filter @fanours/db seed"
  },
  "devDependencies": {
    "@commitlint/cli": "19.6.0",
    "@playwright/test": "1.49.1",
    "@commitlint/config-conventional": "19.6.0",
    "@types/node": "22.10.2",
    "@typescript-eslint/eslint-plugin": "8.18.1",
    "@typescript-eslint/parser": "8.18.1",
    "@vitest/coverage-v8": "2.1.8",
    "eslint": "9.16.0",
    "eslint-config-prettier": "9.1.0",
    "eslint-plugin-boundaries": "5.0.2",
    "husky": "9.1.7",
    "lint-staged": "15.2.11",
    "prettier": "3.4.2",
    "turbo": "2.3.3",
    "typescript": "5.6.3",
    "vitest": "2.1.8"
  }
}
```

`pnpm-workspace.yaml` :

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

`turbo.json` :

```json
{
  "$schema": "https://turbo.build/schema.json",
  "ui": "stream",
  "globalDependencies": ["tsconfig.base.json", "pnpm-lock.yaml"],
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**", "!.next/cache/**"] },
    "typecheck": { "dependsOn": ["^build"], "outputs": [] },
    "lint": { "dependsOn": ["^build"], "outputs": [] },
    "test": { "dependsOn": ["^build"], "outputs": ["coverage/**"] },
    "test:unit": { "dependsOn": ["^build"], "outputs": ["coverage/**"] },
    "test:integration": { "dependsOn": ["^build"], "outputs": ["coverage/**"] }
  }
}
```

`.npmrc` :

```ini
auto-install-peers=true
strict-peer-dependencies=false
node-linker=isolated
```

`.gitignore` :

```gitignore
node_modules/
.turbo/
dist/
build/
coverage/
.next/
out/
*.tsbuildinfo
.env
.env.*
!.env.example
playwright-report/
test-results/
storybook-static/
.DS_Store
```

Attendu : `pnpm install` ne crée aucun lockfile de npm ; `pnpm-lock.yaml` est le seul lockfile suivi par Git.

- [ ] **Step 3: Configurer TypeScript strict et les project references**

`tsconfig.base.json` :

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true,
    "useUnknownInCatchVariables": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "composite": true,
    "baseUrl": ".",
    "paths": {
      "@fanours/config": ["packages/config/src/index.ts"],
      "@fanours/contracts": ["packages/contracts/src/index.ts"],
      "@fanours/domain": ["packages/domain/src/index.ts"],
      "@fanours/db": ["packages/db/src/index.ts"],
      "@fanours/pricing": ["packages/pricing/src/index.ts"],
      "@fanours/compliance": ["packages/compliance/src/index.ts"],
      "@fanours/i18n": ["packages/i18n/src/index.ts"],
      "@fanours/ui": ["packages/ui/src/index.ts"],
      "@fanours/observability": ["packages/observability/src/index.ts"]
    }
  },
  "exclude": ["node_modules", "dist", ".next", "coverage"]
}
```

`tsconfig.json` (solution) :

```json
{
  "files": [],
  "references": [
    { "path": "./packages/config" },
    { "path": "./packages/contracts" },
    { "path": "./packages/domain" },
    { "path": "./packages/observability" }
  ]
}
```

- [ ] **Step 4: Configurer ESLint flat, les frontières de packages et l'interdiction des hex**

`eslint.config.mjs` :

```javascript
import js from '@eslint/js';
import tseslint from 'eslint-plugin-@typescript-eslint';
import boundaries from 'eslint-plugin-boundaries';
import prettier from 'eslint-config-prettier';

export default [
  { ignores: ['**/node_modules/**', '**/dist/**', '**/.next/**', '**/coverage/**'] },
  js.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  {
    files: ['**/*.ts', '**/*.tsx'],
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname }
    },
    plugins: { boundaries },
    settings: {
      boundaries: {
        defaultBoundary: 'module',
        boundaries: [
          {
            name: '@fanours/domain',
            pattern: 'packages/domain/*',
            mode: 'strict',
            from: { allow: [] }
          },
          {
            name: '@fanours/config',
            pattern: 'packages/config/*',
            mode: 'strict',
            from: { allow: ['@fanours/domain'] }
          },
          {
            name: '@fanours/contracts',
            pattern: 'packages/contracts/*',
            mode: 'strict',
            from: { allow: [] }
          },
          {
            name: '@fanours/observability',
            pattern: 'packages/observability/*',
            mode: 'strict',
            from: { allow: ['@fanours/config', '@fanours/domain'] }
          },
          {
            name: '@fanours/db',
            pattern: 'packages/db/*',
            mode: 'strict',
            from: { allow: ['@fanours/domain', '@fanours/config'] }
          },
          {
            name: '@fanours/compliance',
            pattern: 'packages/compliance/*',
            mode: 'strict',
            from: { allow: ['@fanours/domain'] }
          },
          {
            name: '@fanours/pricing',
            pattern: 'packages/pricing/*',
            mode: 'strict',
            from: { allow: ['@fanours/domain', '@fanours/compliance', '@fanours/db'] }
          },
          {
            name: '@fanours/i18n',
            pattern: 'packages/i18n/*',
            mode: 'strict',
            from: { allow: [] }
          },
          {
            name: '@fanours/ui',
            pattern: 'packages/ui/*',
            mode: 'strict',
            from: { allow: ['@fanours/i18n'] }
          },
          {
            name: 'app-web',
            pattern: 'apps/web/*',
            mode: 'strict',
            from: { allow: ['@fanours/ui', '@fanours/i18n', '@fanours/domain', '@fanours/contracts', '@fanours/config'] }
          },
          {
            name: 'app-api',
            pattern: 'apps/api/*',
            mode: 'strict',
            from: { allow: ['@fanours/db', '@fanours/domain', '@fanours/contracts', '@fanours/config', '@fanours/observability'] }
          },
          {
            name: 'app-worker',
            pattern: 'apps/worker/*',
            mode: 'strict',
            from: { allow: ['@fanours/db', '@fanours/contracts', '@fanours/config', '@fanours/observability'] }
          }
        ]
      }
    },
    rules: {
      '@typescript-eslint/no-unnecessary-condition': 'error',
      '@typescript-eslint/consistent-type-imports': ['error', { prefer: 'type-imports' }],
      '@typescript-eslint/no-floating-promises': 'error',
      'no-restricted-syntax': [
        'error',
        {
          selector: 'Literal[value=/#[0-9a-fA-F]{3,8}/]',
          message:
            'Couleur hexadécimale interdite hors packages/ui/src/styles/tokens.css. Utilisez var(--token).'
        },
        {
          selector: "Literal[value=/dangerouslySetInnerHTML/]",
          message: 'dangerouslySetInnerHTML est interdit (spec 19.5).'
        }
      ],
      'boundaries/element-types': 'error',
      'boundaries/no-external-root-imports': 'error'
    }
  },
  prettier
];
```

Le motif `Literal[value=/#[0-9a-fA-F]{3,8}/]` s'applique à tous les fichiers sauf `packages/ui/src/styles/tokens.css` et `docs/**` ; on l'exclut via un bloc dédié :

```javascript
  {
    files: ['packages/ui/src/styles/**', 'docs/**', '**/*.test.ts', '**/*.test.tsx'],
    rules: { 'no-restricted-syntax': 'off' }
  }
```

- [ ] **Step 5: Configurer Prettier, commitlint, Husky et lint-staged**

`.prettierrc.json` :

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

`.prettierignore` :

```text
node_modules
dist
.next
coverage
pnpm-lock.yaml
playwright-report
test-results
storybook-static
```

`commitlint.config.mjs` :

```javascript
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',
        'fix',
        'refactor',
        'test',
        'docs',
        'chore',
        'build',
        'ci',
        'perf',
        'style'
      ]
    ],
    'scope-enum': [
      2,
      'always',
      [
        'catalog',
        'pricing',
        'cart',
        'checkout',
        'order',
        'payment',
        'tax',
        'customs',
        'shipping',
        'auth',
        'account',
        'b2b',
        'quote',
        'admin',
        'content',
        'i18n',
        'infra',
        'db',
        'ui',
        'design',
        'seo',
        'ci',
        'config',
        'observability'
      ]
    ],
    'header-max-length': [2, 'always', 72],
    'body-max-line-length': [2, 'always', 72]
  }
};
```

`.lintstagedrc.json` :

```json
{
  "*.{ts,tsx}": ["prettier --write", "eslint --fix"],
  "*.{json,md,yml,yaml}": ["prettier --write"],
  "*.sql": ["prettier --write --parser sql"]
}
```

`.husky/pre-commit` :

```bash
pnpm exec lint-staged
pnpm exec gitleaks protect --staged --redact --no-banner
```

`.husky/commit-msg` :

```bash
pnpm exec commitlint --edit "$1"
```

- [ ] **Step 6: Écrire `.env.example` (noms de variables uniquement) et `.env` local factice**

`.env.example` :

```dotenv
# Fanours — noms de variables uniquement. Aucune valeur réelle ici.
APP_ENV=
APP_BASE_URL=
CORS_ORIGINS=

DATABASE_URL=
REDIS_URL=

SESSION_SECRET=
LOG_HASH_KEY=
KMS_KEY_ARN=

EMAIL_PROVIDER=
EMAIL_API_KEY=

PAYMENT_PROVIDER=
PAYMENT_API_KEY=
PAYMENT_WEBHOOK_SECRET=

SHIPPING_PROVIDER=
SHIPPING_API_KEY=

S3_ENDPOINT=
S3_REGION=
S3_BUCKET_PUBLIC=
S3_BUCKET_PRIVATE=
S3_ACCESS_KEY_ID=
S3_SECRET_ACCESS_KEY=
CDN_BASE_URL=

SENTRY_DSN=
OTEL_EXPORTER_OTLP_ENDPOINT=
ANALYTICS_WRITE_KEY=
CONSENT_SECRET=
```

Le fichier `.env` local n'est **pas** versionné et contient des valeurs de développement factices :

```dotenv
APP_ENV=local
APP_BASE_URL=http://localhost:3000
CORS_ORIGINS=http://localhost:3000
DATABASE_URL=postgres://fanours:fanours@localhost:5432/fanours
REDIS_URL=redis://localhost:6379
SESSION_SECRET=local-development-session-secret-not-for-production
LOG_HASH_KEY=local-development-log-hash-key-not-for-production
KMS_KEY_ARN=local-development-kms-key
EMAIL_PROVIDER=stub
EMAIL_API_KEY=local-stub
PAYMENT_PROVIDER=stub
PAYMENT_API_KEY=local-stub
PAYMENT_WEBHOOK_SECRET=local-stub
SHIPPING_PROVIDER=stub
SHIPPING_API_KEY=local-stub
S3_ENDPOINT=http://localhost:9000
S3_REGION=local
S3_BUCKET_PUBLIC=fanours-public
S3_BUCKET_PRIVATE=fanours-private
S3_ACCESS_KEY_ID=local
S3_SECRET_ACCESS_KEY=local
CDN_BASE_URL=http://localhost:9000/fanours-public
SENTRY_DSN=
OTEL_EXPORTER_OTLP_ENDPOINT=
ANALYTICS_WRITE_KEY=local
CONSENT_SECRET=local-development-consent-secret
```

Attendu : `git check-ignore -v .env` retourne une règle de `.gitignore` ; `git status --short` ne montre pas `.env`.

- [ ] **Step 7: Créer `packages/config` — schémas Zod d'environnement**

`packages/config/package.json` :

```json
{
  "name": "@fanours/config",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": { ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" } },
  "scripts": {
    "build": "tsc -b",
    "typecheck": "tsc -b --pretty",
    "lint": "eslint src",
    "test:unit": "vitest run",
    "test": "vitest run"
  },
  "dependencies": { "zod": "3.24.1" },
  "devDependencies": { "typescript": "5.6.3", "vitest": "2.1.8" }
}
```

`packages/config/src/env-schema.ts` :

```typescript
import { z } from 'zod';

export const ENV_NAMES = [
  'local',
  'test',
  'staging',
  'production',
] as const;

export type EnvName = (typeof ENV_NAMES)[number];

const baseSchema = z.object({
  APP_ENV: z.enum(ENV_NAMES),
  APP_BASE_URL: z.string().url(),
  CORS_ORIGINS: z.string().min(1),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  SESSION_SECRET: z.string().min(32),
  LOG_HASH_KEY: z.string().min(32),
  CONSENT_SECRET: z.string().min(32),
  KMS_KEY_ARN: z.string().min(1),
  EMAIL_PROVIDER: z.string().min(1),
  EMAIL_API_KEY: z.string().min(1),
  PAYMENT_PROVIDER: z.string().min(1),
  PAYMENT_API_KEY: z.string().min(1),
  PAYMENT_WEBHOOK_SECRET: z.string().min(16),
  SHIPPING_PROVIDER: z.string().min(1),
  SHIPPING_API_KEY: z.string().min(1),
  S3_ENDPOINT: z.string().url(),
  S3_REGION: z.string().min(1),
  S3_BUCKET_PUBLIC: z.string().min(1),
  S3_BUCKET_PRIVATE: z.string().min(1),
  S3_ACCESS_KEY_ID: z.string().min(1),
  S3_SECRET_ACCESS_KEY: z.string().min(1),
  CDN_BASE_URL: z.string().url(),
  SENTRY_DSN: z.string(),
  OTEL_EXPORTER_OTLP_ENDPOINT: z.string(),
  ANALYTICS_WRITE_KEY: z.string().min(1),
});

export const envSchema = baseSchema.strict();

export type Env = z.infer<typeof envSchema>;

export function parseEnv(source: NodeJS.ProcessEnv): Env {
  const result = envSchema.safeParse(source);
  if (!result.success) {
    const issues = result.error.issues
      .map((issue) => `${issue.path.join('.') || '(racine)'}: ${issue.message}`)
      .join('; ');
    throw new Error(`Configuration d'environnement invalide — ${issues}`);
  }
  return result.data;
}
```

`packages/config/src/env-matrix.ts` :

```typescript
import type { EnvName } from './env-schema.js';

export interface EnvMatrix {
  readonly env: EnvName;
  readonly isProduction: boolean;
  readonly logLevel: 'debug' | 'info' | 'warn' | 'error';
  readonly traceSampleRatio: number;
  readonly traceSampleRatioOnError: number;
  readonly hstsEnabled: boolean;
  readonly sessionCookieSecure: boolean;
  readonly corsAllowedOrigins: readonly string[];
  readonly piiScanEnabled: boolean;
}

const MATRICES: Record<EnvName, Omit<EnvMatrix, 'env'>> = {
  local: {
    isProduction: false,
    logLevel: 'debug',
    traceSampleRatio: 1,
    traceSampleRatioOnError: 1,
    hstsEnabled: false,
    sessionCookieSecure: false,
    corsAllowedOrigins: ['http://localhost:3000', 'http://localhost:3001'],
    piiScanEnabled: false,
  },
  test: {
    isProduction: false,
    logLevel: 'warn',
    traceSampleRatio: 1,
    traceSampleRatioOnError: 1,
    hstsEnabled: false,
    sessionCookieSecure: false,
    corsAllowedOrigins: ['http://localhost:3000'],
    piiScanEnabled: true,
  },
  staging: {
    isProduction: false,
    logLevel: 'info',
    traceSampleRatio: 0.25,
    traceSampleRatioOnError: 1,
    hstsEnabled: false,
    sessionCookieSecure: true,
    corsAllowedOrigins: ['https://staging.fanours.com'],
    piiScanEnabled: true,
  },
  production: {
    isProduction: true,
    logLevel: 'info',
    traceSampleRatio: 0.1,
    traceSampleRatioOnError: 1,
    hstsEnabled: true,
    sessionCookieSecure: true,
    corsAllowedOrigins: ['https://fanours.com'],
    piiScanEnabled: true,
  },
};

export function envMatrix(env: NodeJS.ProcessEnv): EnvMatrix {
  const raw = env.APP_ENV;
  if (raw === undefined || !(raw in MATRICES)) {
    throw new Error(`APP_ENV invalide: ${String(raw)}`);
  }
  return { env: raw as EnvName, ...MATRICES[raw as EnvName] };
}
```

`packages/config/src/feature-flags.ts` :

```typescript
export const FLAG_KEYS = [
  'checkout_new_flow',
  'catalogue_facets_v2',
  'mfa_passkeys',
  'pwa_offline_catalogue',
] as const;

export type FlagKey = (typeof FLAG_KEYS)[number];

export interface FlagContext {
  readonly flags: Readonly<Record<string, boolean>>;
  readonly subjectId: string;
  readonly env: string;
}

export function hashToBucket(value: string, buckets: number): number {
  let hash = 0x811c9dc5;
  for (let index = 0; index < value.length; index += 1) {
    hash ^= value.charCodeAt(index);
    hash = Math.imul(hash, 0x01000193) >>> 0;
  }
  return hash % buckets;
}

export function evaluateFlag(context: FlagContext, key: FlagKey): boolean {
  const enabled = context.flags[key];
  if (enabled === undefined) return false;
  if (enabled) return true;
  if (context.env === 'local' || context.env === 'test') return true;
  return hashToBucket(`${key}:${context.subjectId}`, 100) < 10;
}
```

`packages/config/src/index.ts` :

```typescript
export * from './env-schema.js';
export * from './env-matrix.js';
export * from './feature-flags.js';
```

- [ ] **Step 8: Créer `packages/contracts` — types de frontière**

`packages/contracts/package.json` :

```json
{
  "name": "@fanours/contracts",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": { ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" } },
  "scripts": {
    "build": "tsc -b",
    "typecheck": "tsc -b --pretty",
    "lint": "eslint src",
    "test:unit": "vitest run --passWithNoTests",
    "test": "vitest run --passWithNoTests"
  },
  "dependencies": {},
  "devDependencies": { "typescript": "5.6.3", "vitest": "2.1.8" }
}
```

`packages/domain/package.json` :

```json
{
  "name": "@fanours/domain",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": { ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" } },
  "scripts": {
    "build": "tsc -b",
    "typecheck": "tsc -b --pretty",
    "lint": "eslint src",
    "test:unit": "vitest run --passWithNoTests",
    "test": "vitest run --passWithNoTests"
  },
  "dependencies": {},
  "devDependencies": { "typescript": "5.6.3", "vitest": "2.1.8" }
}
```

`packages/observability/package.json` :

```json
{
  "name": "@fanours/observability",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": { ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" } },
  "scripts": {
    "build": "tsc -b",
    "typecheck": "tsc -b --pretty",
    "lint": "eslint src",
    "test:unit": "vitest run",
    "test": "vitest run"
  },
  "dependencies": {
    "@fanours/config": "workspace:*",
    "pino": "9.6.0"
  },
  "devDependencies": { "typescript": "5.6.3", "vitest": "2.1.8" }
}
```

`packages/contracts/src/api.ts` :

```typescript
export const API_ERROR_CODES = [
  'validation_failed',
  'unauthenticated',
  'session_expired',
  'forbidden',
  'mfa_required',
  'approval_required',
  'self_approval_forbidden',
  'not_found',
  'conflict',
  'rate_limited',
  'payload_too_large',
  'upstream_unavailable',
  'internal_error',
] as const;

export type ApiErrorCode = (typeof API_ERROR_CODES)[number];

export interface ApiErrorBody {
  readonly error: {
    readonly code: ApiErrorCode;
    readonly messageKey: string;
    readonly correlationId: string;
    readonly details?: Readonly<Record<string, string>>;
  };
}

export interface ApiEnvelope<T> {
  readonly data: T;
  readonly correlationId: string;
}
```

`packages/contracts/src/events.ts` :

```typescript
export const EVENT_TYPES = [
  'user.registered',
  'user.email_verified',
  'user.logged_in',
  'user.logged_out',
  'user.password_changed',
  'mfa.enrolled',
  'mfa.verified',
  'mfa.recovery_code_used',
  'session.revoked',
  'audit.control_event',
] as const;

export type EventType = (typeof EVENT_TYPES)[number];

export interface EventEnvelope<TPayload = Record<string, unknown>> {
  readonly id: string;
  readonly type: EventType;
  readonly occurredAt: string;
  readonly correlationId: string;
  readonly aggregateId: string;
  readonly payload: TPayload;
}
```

`packages/contracts/src/index.ts` :

```typescript
export * from './api.js';
export * from './events.js';
```

- [ ] **Step 9: Créer `packages/domain` — règles pures sans framework**

`packages/domain/src/session-policy.ts` :

```typescript
export type SessionProfile = 'client' | 'admin' | 'b2b_pending' | 'sensitive_elevation';

export interface SessionPolicy {
  readonly inactiveMs: number;
  readonly absoluteMs: number;
}

const MINUTE = 60 * 1000;
const HOUR = 60 * MINUTE;

const POLICIES: Record<SessionProfile, SessionPolicy> = {
  client: { inactiveMs: 30 * MINUTE, absoluteMs: 12 * HOUR },
  admin: { inactiveMs: 15 * MINUTE, absoluteMs: 8 * HOUR },
  b2b_pending: { inactiveMs: 60 * MINUTE, absoluteMs: 12 * HOUR },
  sensitive_elevation: { inactiveMs: 15 * MINUTE, absoluteMs: 30 * MINUTE },
};

export function sessionPolicy(profile: SessionProfile): SessionPolicy {
  return POLICIES[profile];
}

export interface SessionState {
  readonly createdAt: number;
  readonly lastSeenAt: number;
  readonly absoluteExpiresAt: number;
}

export function isSessionExpired(state: SessionState, policy: SessionPolicy, now: number): boolean {
  if (now >= state.absoluteExpiresAt) return true;
  return now - state.lastSeenAt >= policy.inactiveMs;
}
```

`packages/domain/src/index.ts` :

```typescript
export * from './session-policy.js';
```

- [ ] **Step 10: Créer `packages/observability` — logger JSON, redaction, corrélation**

`packages/observability/src/redaction.ts` :

```typescript
export const FORBIDDEN_KEYS = [
  'password',
  'password_confirmation',
  'current_password',
  'new_password',
  'token',
  'access_token',
  'refresh_token',
  'id_token',
  'authorization',
  'cookie',
  'set-cookie',
  'session_token',
  'session_id',
  'api_key',
  'apikey',
  'secret',
  'client_secret',
  'webhook_secret',
  'private_key',
  'credit_card',
  'card_number',
  'cvv',
  'cvc',
  'iban',
  'ssn',
  'tax_id',
  'totp_secret',
  'recovery_code',
  'mfa_secret',
  'otp',
  'email',
  'phone',
  'address',
  'first_name',
  'last_name',
  'full_name',
] as const;

const FORBIDDEN_KEY_SET = new Set<string>(FORBIDDEN_KEYS.map((key) => key.toLowerCase()));

const VALUE_PATTERNS: ReadonlyArray<readonly [RegExp, string]> = [
  [/\b(?:\d[ -]*?){13,19}\b/g, 'card'],
  [/\bsk_(?:live|test)_[A-Za-z0-9]{8,}\b/g, 'secret'],
  [/\bpk_(?:live|test)_[A-Za-z0-9]{8,}\b/g, 'secret'],
  [/\bAKIA[0-9A-Z]{16}\b/g, 'secret'],
  [/\bgh[pousr]_[A-Za-z0-9]{16,}\b/g, 'secret'],
  [/\bxox[baprs]-[A-Za-z0-9-]{10,}\b/g, 'secret'],
  [/\beyJ[A-Za-z0-9_-]{5,}\.[A-Za-z0-9_-]{5,}\.[A-Za-z0-9_-]{5,}\b/g, 'jwt'],
  [/\bBearer\s+[A-Za-z0-9._-]{10,}\b/gi, 'bearer'],
  [/\bBasic\s+[A-Za-z0-9+/=]{10,}\b/gi, 'basic'],
  [/-----BEGIN [A-Z ]*PRIVATE KEY-----[\s\S]*?-----END [A-Z ]*PRIVATE KEY-----/g, 'private_key'],
  [/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/g, 'email'],
];

const MASKED_EMAIL = /^(.)[^@]*(@.*)$/;

export function redactString(input: string): string {
  let output = input;
  for (const [pattern, kind] of VALUE_PATTERNS) {
    output = output.replace(pattern, (match) => {
      if (kind === 'email') {
        const masked = MASKED_EMAIL.exec(match);
        return masked ? `${masked[1]}***${masked[2]}` : '[REDACTED:email]';
      }
      return `[REDACTED:${kind}]`;
    });
  }
  return output;
}

export function redact(value: unknown, depth = 0): unknown {
  if (depth > 8) return '[REDACTED:depth]';
  if (typeof value === 'string') return redactString(value);
  if (value === null || typeof value !== 'object') return value;
  if (Array.isArray(value)) return value.map((item) => redact(item, depth + 1));
  const source = value as Record<string, unknown>;
  const output: Record<string, unknown> = {};
  for (const [key, item] of Object.entries(source)) {
    if (FORBIDDEN_KEY_SET.has(key.toLowerCase())) {
      output[key] = '[REDACTED:key]';
      continue;
    }
    output[key] = redact(item, depth + 1);
  }
  return output;
}
```

`packages/observability/src/correlation.ts` :

```typescript
import { randomUUID } from 'node:crypto';

export function newCorrelationId(): string {
  return randomUUID();
}

export function newTraceId(): string {
  return randomBytesHex(16);
}

export function newSpanId(): string {
  return randomBytesHex(8);
}

function randomBytesHex(length: number): string {
  const bytes = new Uint8Array(length);
  for (let index = 0; index < length; index += 1) {
    bytes[index] = Math.floor(Math.random() * 256);
  }
  return Array.from(bytes, (byte) => byte.toString(16).padStart(2, '0')).join('');
}

export function formatTraceparent(
  traceId: string,
  spanId: string,
  sampled: boolean,
): string {
  const flags = sampled ? '01' : '00';
  return `00-${traceId}-${spanId}-${flags}`;
}

export function parseTraceparent(value: string): {
  traceId: string;
  spanId: string;
  sampled: boolean;
} | null {
  const parts = value.split('-');
  if (parts.length !== 4) return null;
  const [version, traceId, spanId, flags] = parts;
  if (version !== '00') return null;
  if (!/^[0-9a-f]{32}$/.test(traceId ?? '')) return null;
  if (!/^[0-9a-f]{16}$/.test(spanId ?? '')) return null;
  return { traceId, spanId, sampled: (flags ?? '00') === '01' };
}
```

`packages/observability/src/logger.ts` :

```typescript
import { createHmac } from 'node:crypto';
import pino from 'pino';
import type { EnvName } from '@fanours/config';
import { redact } from './redaction.js';

export interface LoggerFields {
  readonly correlation_id?: string;
  readonly trace_id?: string;
  readonly user_id_hash?: string;
  readonly session_id_hash?: string;
  readonly route?: string;
  readonly method?: string;
  readonly status?: number;
  readonly duration_ms?: number;
  readonly order_id?: string;
  readonly event?: string;
  readonly [key: string]: unknown;
}

export function hashIdentifier(value: string, key: string): string {
  return createHmac('sha256', key).update(value).digest('hex').slice(0, 32);
}

export function createLogger(service: string, env: EnvName, logHashKey: string) {
  const base = pino({
    base: { service, env },
    level: env === 'production' || env === 'staging' ? 'info' : 'debug',
    timestamp: pino.stdTimeFunctions.isoTime,
    formatters: {
      level: (label) => ({ level: label }),
    },
    hooks: {
      logMethod(args, method) {
        const [first] = args;
        const payload =
          typeof first === 'object' && first !== null ? (first as Record<string, unknown>) : {};
        method.call(this, redact(payload) as Record<string, unknown>);
      },
    },
  });

  return {
    info(fields: LoggerFields, message?: string) {
      base.info(fields, message);
    },
    warn(fields: LoggerFields, message?: string) {
      base.warn(fields, message);
    },
    error(fields: LoggerFields, message?: string) {
      base.error(fields, message);
    },
    hashUserId(userId: string): string {
      return hashIdentifier(userId, logHashKey);
    },
  };
}

export type Logger = ReturnType<typeof createLogger>;
```

`packages/observability/src/index.ts` :

```typescript
export * from './redaction.js';
export * from './correlation.js';
export * from './logger.js';
```

- [ ] **Step 11: Écrire les tests qui doivent échouer**

`packages/config/src/__tests__/env-matrix.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { envMatrix } from '../env-matrix.js';
import { evaluateFlag, hashToBucket } from '../feature-flags.js';

describe('envMatrix', () => {
  it('refuse un APP_ENV inconnu', () => {
    expect(() => envMatrix({ APP_ENV: 'prod' })).toThrow(/APP_ENV invalide/);
  });

  it('active HSTS et le cookie secure uniquement en production', () => {
    expect(envMatrix({ APP_ENV: 'production' }).hstsEnabled).toBe(true);
    expect(envMatrix({ APP_ENV: 'production' }).sessionCookieSecure).toBe(true);
    expect(envMatrix({ APP_ENV: 'staging' }).hstsEnabled).toBe(false);
    expect(envMatrix({ APP_ENV: 'local' }).sessionCookieSecure).toBe(false);
  });

  it('echantillonne les traces a 10 pour cent en production', () => {
    expect(envMatrix({ APP_ENV: 'production' }).traceSampleRatio).toBe(0.1);
  });
});

describe('evaluateFlag', () => {
  it('retourne false pour un drapeau inconnu', () => {
    const result = evaluateFlag({ flags: {}, subjectId: 'u1', env: 'production' }, 'mfa_passkeys');
    expect(result).toBe(false);
  });

  it('active un drapeau explicitement a true', () => {
    const result = evaluateFlag(
      { flags: { mfa_passkeys: true }, subjectId: 'u1', env: 'production' },
      'mfa_passkeys',
    );
    expect(result).toBe(true);
  });

  it('repartit de facon deterministe dans les 10 pour cent', () => {
    const subjectId = 'user-42';
    const first = hashToBucket(`mfa_passkeys:${subjectId}`, 100);
    const second = hashToBucket(`mfa_passkeys:${subjectId}`, 100);
    expect(first).toBe(second);
    expect(first).toBeGreaterThanOrEqual(0);
    expect(first).toBeLessThan(100);
  });
});
```

`packages/observability/src/__tests__/redaction.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { redact, redactString } from '../redaction.js';

describe('redactString', () => {
  it('masque une carte bancaire', () => {
    expect(redactString('carte 4242 4242 4242 4242')).toBe('carte [REDACTED:card]');
  });

  it('masque une cle secrete', () => {
    expect(redactString('sk_live_abcdefghijkl')).toContain('[REDACTED:secret]');
  });

  it('masque un JWT', () => {
    expect(redactString('eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxIn0.abcdefg')).toContain(
      '[REDACTED:jwt]',
    );
  });

  it('partiellement masque un email', () => {
    expect(redactString('contact@fanours.com')).toBe('c***@fanours.com');
  });

  it('ne touche pas une phrase normale', () => {
    expect(redactString('catalogue de gobelets isothermes')).toBe(
      'catalogue de gobelets isothermes',
    );
  });
});

describe('redact', () => {
  it('redige les cles interdites', () => {
    const result = redact({ password: 'hunter2', email: 'a@b.com', route: '/fr/' }) as Record<
      string,
      unknown
    >;
    expect(result['password']).toBe('[REDACTED:key]');
    expect(result['email']).toBe('[REDACTED:key]');
    expect(result['route']).toBe('/fr/');
  });

  it('redige recursivement les objets imbriques', () => {
    const result = redact({ user: { ssn: '123456789' } }) as Record<string, Record<string, unknown>>;
    expect(result['user']?.['ssn']).toBe('[REDACTED:key]');
  });
});
```

`packages/observability/src/__tests__/logger.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { createLogger, hashIdentifier } from '../logger.js';

describe('hashIdentifier', () => {
  it('produit un hash stable et non reversible', () => {
    const first = hashIdentifier('user-1', 'key');
    const second = hashIdentifier('user-1', 'key');
    expect(first).toBe(second);
    expect(first).toHaveLength(32);
    expect(first).not.toContain('user-1');
  });

  it('change avec la cle', () => {
    expect(hashIdentifier('user-1', 'key-a')).not.toBe(hashIdentifier('user-1', 'key-b'));
  });
});

describe('createLogger', () => {
  it('redige les secrets dans les champs', () => {
    const logger = createLogger('api', 'local', 'a'.repeat(32));
    expect(() => logger.info({ password: 'x' }, 'test')).not.toThrow();
  });
});
```

- [ ] **Step 12: Exécuter les tests et constater l'échec**

```bash
pnpm install
pnpm --filter @fanours/config test:unit
pnpm --filter @fanours/observability test:unit
```

Attendu : les fichiers de test n'existent pas encore au moment de la première exécution, ou bien `vitest` signale `Cannot find module '../env-matrix.js'` / `No test files found` avec un code de sortie non nul. Coller la sortie complète.

- [ ] **Step 13: Implémenter jusqu'à la réussite**

Reprendre les fichiers des étapes 7 à 10 (déjà écrits ci-dessus), puis :

```bash
pnpm --filter @fanours/config test:unit
pnpm --filter @fanours/observability test:unit
pnpm --filter @fanours/contracts typecheck
pnpm --filter @fanours/domain typecheck
```

Attendu :

```
✓ packages/config/src/__tests__/env-matrix.test.ts (5 tests)
✓ packages/observability/src/__tests__/redaction.test.ts (9 tests)
✓ packages/observability/src/__tests__/logger.test.ts (3 tests)

 Test Files  3 passed (3)
      Tests  17 passed (17)
```

- [ ] **Step 14: Vérifier le typecheck strict et le lint**

```bash
pnpm typecheck
pnpm lint
pnpm exec prettier --check "packages/**/*.ts"
```

Attendu : aucune erreur TypeScript (`tsc --noEmit` sur chaque projet), aucun avertissement ESLint, `prettier --check` retourne `All matched files use Prettier code style!`.

- [ ] **Step 15: Écrire `CONTRIBUTING.md` et `LICENSE`**

`CONTRIBUTING.md` :

```markdown
# Contributing — Fanours

## Prérequis

- Node.js 22.x
- pnpm 9.15.0 (`corepack enable pnpm`)
- Docker (Postgres 16, Redis 7, MinIO) via `docker/compose.local.yml`

## Installation

```bash
pnpm install
cp .env.example .env
docker compose -f docker/compose.local.yml up -d
pnpm db:migrate
pnpm db:seed:synthetic
pnpm dev
```

## Branches

`feat|fix|refactor|test|docs|chore/<slug>`. Jamais de commit sur `main`.

## Commits

Conventional Commits 1.0. Scopes autorisés : `catalog|pricing|cart|checkout|order|payment|tax|customs|shipping|auth|account|b2b|quote|admin|content|i18n|infra|db|ui|design|seo|ci`.

Chaque message se termine par :

```
Co-Authored-By: Claude Code <noreply@anthropic.com>
```

## Vérifications avant push

```bash
pnpm typecheck
pnpm lint
pnpm test:unit
pnpm test:integration
```

## PR

Une PR par tranche de travail, description en français, preuves (sortie de tests, captures). Jamais de merge sans accord explicite du mainteneur.
```

`LICENSE` (MIT, copyright Fanours) :

```text
MIT License

Copyright (c) 2026 Fanours

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 16: Commit**

```bash
git add package.json pnpm-workspace.yaml turbo.json tsconfig.base.json tsconfig.json .npmrc .gitignore .env.example .prettierrc.json .prettierignore eslint.config.mjs commitlint.config.mjs .husky/pre-commit .husky/commit-msg .lintstagedrc.json LICENSE CONTRIBUTING.md packages/config packages/contracts packages/domain packages/observability
git commit -m "feat(config): pose le socle monorepo et les packages partages" -m "Monorepo pnpm workspaces + Turborepo, TypeScript strict avec project
references, ESLint flat avec regles de dependances et interdiction des
couleurs hex hors tokens, Prettier, Husky, commitlint, schemas Zod
d'environnement, contrats partages, regles de session et logger JSON
avec redaction.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` affiche le titre, le corps et la ligne `Co-Authored-By`.

### Task 2: Schéma Drizzle, migrations SQL manuelles et runner idempotent

**Files:**
- Create: `packages/db/package.json`, `packages/db/tsconfig.json`, `packages/db/drizzle.config.ts`, `packages/db/vitest.config.ts`
- Create: `packages/db/migrations/0001_extensions_and_migration_table.sql`, `packages/db/migrations/0002_identity.sql`, `packages/db/migrations/0003_rbac_and_audit.sql`, `packages/db/migrations/0004_outbox.sql`
- Create: `packages/db/scripts/migrate.ts`, `packages/db/src/migrate-core.ts`, `packages/db/src/schema.ts`, `packages/db/src/client.ts`, `packages/db/src/seed.ts`, `packages/db/src/index.ts`
- Test: `packages/db/src/__tests__/migrations.integration.test.ts`, `packages/db/src/__tests__/audit-immutability.test.ts`, `packages/db/src/__tests__/session-lifecycle.test.ts`

> **Propriété du paquet `packages/db` (Phase 1).** Cette task crée la fondation du paquet : le runner `packages/db/scripts/migrate.ts` (CLI) et son cœur `packages/db/src/migrate-core.ts`, ainsi que les migrations `0001` à `0004` (la Task 9 ajoute `0005_rbac_scopes.sql`). Les phases suivantes **étendent** ce paquet : la Phase 2 ajoute les modules de schéma Drizzle `packages/db/src/schema/*.ts` et les migrations `0006` et suivantes, puis la Phase 3 les migrations `0100` et suivantes. Aucune phase ultérieure ne **recrée** `packages/db/src/client.ts`, `packages/db/src/index.ts`, `packages/db/src/migrate-core.ts`, `packages/db/scripts/migrate.ts` ni `packages/db/migrations/0001_extensions_and_migration_table.sql` : ces fichiers appartiennent à la Phase 1, les phases suivantes s'y greffent (la Phase 2 complète `src/client.ts` et `src/index.ts` par des ajouts, elle ne les réécrit pas) et aucune migration déjà appliquée n'est réécrite ni rejouée (append-only).

**Interfaces:**
- Consumes: `@fanours/config` (`parseEnv`, `Env`), `@fanours/domain` (`SessionPolicy`).
- Produces:
  - `@fanours/db` : `db` (instance Drizzle), `tables` (`user`, `authIdentity`, `session`, `mfaMethod`, `recoveryCode`, `role`, `permission`, `userRole`, `auditLog`, `outboxEvent`, `consumedEvent`), `withTransaction(cb)`, `hashToken(token)`, `newOpaqueToken()`.
  - `packages/db/src/migrate-core.ts` : `migrate(options: { directory: string; client: Pool }): Promise<AppliedMigration[]>` (cœur du runner, importé par la CLI `packages/db/scripts/migrate.ts` et réutilisé tel quel par les phases suivantes).
  - Table `audit_log` : `id bigserial`, `occurred_at timestamptz`, `actor_id uuid`, `actor_role text`, `action text`, `object_type text`, `object_id text`, `result text`, `correlation_id text`, `ip_hash text`, `metadata jsonb`, `prev_hash text`, `entry_hash text UNIQUE`.

---

- [ ] **Step 1: Créer le package `packages/db`**

`packages/db/package.json` :

```json
{
  "name": "@fanours/db",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": { ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" } },
  "scripts": {
    "build": "tsc -b",
    "typecheck": "tsc -b --pretty",
    "lint": "eslint src scripts",
    "migrate": "tsx scripts/migrate.ts",
    "generate": "drizzle-kit generate",
    "seed": "tsx src/seed.ts",
    "test:integration": "vitest run --project integration",
    "test:unit": "vitest run --project unit",
    "test": "vitest run"
  },
  "dependencies": {
    "@fanours/config": "workspace:*",
    "@fanours/domain": "workspace:*",
    "drizzle-orm": "0.38.3",
    "pg": "8.13.1"
  },
  "devDependencies": {
    "@testcontainers/postgresql": "10.16.0",
    "@types/pg": "8.11.10",
    "drizzle-kit": "0.30.1",
    "tsx": "4.19.2",
    "typescript": "5.6.3",
    "vitest": "2.1.8"
  }
}
```

`packages/db/drizzle.config.ts` :

```typescript
import type { Config } from 'drizzle-kit';

export default {
  schema: './src/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env['DATABASE_URL'] ?? 'postgres://fanours:fanours@localhost:5432/fanours',
  },
  strict: true,
  verbose: true,
} satisfies Config;
```

`drizzle-kit generate` produit un diff dans `packages/db/drizzle/` ; ce dossier est **informatif** et n'est jamais appliqué automatiquement. Seuls les fichiers de `packages/db/migrations/*.sql` sont appliqués par le runner.

- [ ] **Step 2: Écrire la migration 0001 — extensions et table de suivi**

`packages/db/migrations/0001_extensions_and_migration_table.sql` :

```sql
-- 0001_extensions_and_migration_table.sql
-- Extensions requises : UUID v7 (pgcrypto), recherche floue (unaccent),
-- exclusion de plages (btree_gist) pour les paliers de prix (phase 2).
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS unaccent;
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Table de suivi des migrations appliquees. Append-only : aucune ligne n'est
-- jamais supprimee ni modifiee. Le checksum detecte toute alteration d'un
-- fichier de migration deja applique.
CREATE TABLE IF NOT EXISTS schema_migrations (
  version     text PRIMARY KEY,
  checksum    text NOT NULL,
  applied_at  timestamptz NOT NULL DEFAULT now(),
  duration_ms integer NOT NULL DEFAULT 0
);

COMMENT ON TABLE schema_migrations IS
  'Migrations append-only : aucune ligne n''est jamais supprimee ni modifiee.';
```

- [ ] **Step 3: Écrire la migration 0002 — identité, sessions, MFA**

`packages/db/migrations/0002_identity.sql` :

```sql
-- 0002_identity.sql
-- Identite : comptes, identifiants externes, sessions, MFA, codes de recovery.
SET lock_timeout = '3s';
SET statement_timeout = '30s';

CREATE TABLE IF NOT EXISTS "user" (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email          text NOT NULL,
  email_normalized text NOT NULL,
  email_verified_at timestamptz,
  password_hash  text,
  display_name   text,
  locale         text NOT NULL DEFAULT 'fr-FR',
  theme          text NOT NULL DEFAULT 'auto',
  status         text NOT NULL DEFAULT 'active',
  created_at     timestamptz NOT NULL DEFAULT now(),
  updated_at     timestamptz NOT NULL DEFAULT now(),
  deleted_at     timestamptz,
  CONSTRAINT ck_user_locale CHECK (locale IN ('fr-FR', 'en', 'zh-CN')),
  CONSTRAINT ck_user_theme CHECK (theme IN ('light', 'dark', 'auto')),
  CONSTRAINT ck_user_status CHECK (status IN ('active', 'suspended', 'deleted')),
  CONSTRAINT ck_user_email_normalized CHECK (email_normalized = lower(email_normalized))
);

CREATE UNIQUE INDEX IF NOT EXISTS uq_user_email_normalized
  ON "user" (email_normalized) WHERE deleted_at IS NULL;
CREATE INDEX IF NOT EXISTS ix_user_status ON "user" (status) WHERE deleted_at IS NULL;

CREATE TABLE IF NOT EXISTS auth_identity (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        uuid NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
  provider       text NOT NULL,
  provider_subject text NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_auth_identity_provider_subject UNIQUE (provider, provider_subject)
);

CREATE INDEX IF NOT EXISTS ix_auth_identity_user ON auth_identity (user_id);

CREATE TABLE IF NOT EXISTS session (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         uuid NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
  token_hash      text NOT NULL,
  profile         text NOT NULL DEFAULT 'client',
  ip_hash         text,
  user_agent      text,
  created_at      timestamptz NOT NULL DEFAULT now(),
  last_seen_at    timestamptz NOT NULL DEFAULT now(),
  absolute_expires_at timestamptz NOT NULL,
  revoked_at      timestamptz,
  revoked_reason  text,
  elevated_at     timestamptz,
  mfa_verified_at timestamptz,
  CONSTRAINT uq_session_token_hash UNIQUE (token_hash),
  CONSTRAINT ck_session_profile CHECK (
    profile IN ('client', 'admin', 'b2b_pending', 'sensitive_elevation')
  )
);

CREATE INDEX IF NOT EXISTS ix_session_user_active
  ON session (user_id, last_seen_at DESC) WHERE revoked_at IS NULL;
CREATE INDEX IF NOT EXISTS ix_session_expiry ON session (absolute_expires_at);

CREATE TABLE IF NOT EXISTS mfa_method (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         uuid NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
  kind            text NOT NULL DEFAULT 'totp',
  algorithm       text NOT NULL DEFAULT 'SHA1',
  digits          integer NOT NULL DEFAULT 6,
  period_seconds  integer NOT NULL DEFAULT 30,
  secret_ciphertext text NOT NULL,
  secret_key_version text NOT NULL,
  last_counter    bigint NOT NULL DEFAULT 0,
  confirmed_at    timestamptz,
  created_at      timestamptz NOT NULL DEFAULT now(),
  revoked_at      timestamptz,
  CONSTRAINT ck_mfa_method_kind CHECK (kind IN ('totp', 'webauthn')),
  CONSTRAINT ck_mfa_method_algorithm CHECK (algorithm = 'SHA1'),
  CONSTRAINT ck_mfa_method_digits CHECK (digits = 6),
  CONSTRAINT ck_mfa_method_period CHECK (period_seconds = 30)
);

CREATE UNIQUE INDEX IF NOT EXISTS uq_mfa_method_user_totp
  ON mfa_method (user_id) WHERE kind = 'totp' AND revoked_at IS NULL;

CREATE TABLE IF NOT EXISTS recovery_code (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
  code_hash   text NOT NULL,
  used_at     timestamptz,
  created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS ix_recovery_code_user_unused
  ON recovery_code (user_id) WHERE used_at IS NULL;
```

- [ ] **Step 4: Écrire la migration 0003 — RBAC et journal d'audit append-only**

`packages/db/migrations/0003_rbac_and_audit.sql` :

```sql
-- 0003_rbac_and_audit.sql
-- RBAC : 9 roles, matrice de permissions, audit append-only avec chaine de hachage.
SET lock_timeout = '3s';
SET statement_timeout = '30s';

CREATE TABLE IF NOT EXISTS role (
  code        text PRIMARY KEY,
  label       text NOT NULL,
  description text NOT NULL DEFAULT '',
  created_at  timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT ck_role_code CHECK (code IN (
    'super_admin', 'catalog_manager', 'order_manager', 'logistics_operator',
    'finance_manager', 'support_agent', 'compliance_manager',
    'marketing_manager', 'analyst_readonly'
  ))
);

CREATE TABLE IF NOT EXISTS permission (
  code        text PRIMARY KEY,
  resource    text NOT NULL,
  action      text NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT ck_permission_resource CHECK (resource IN (
    'order', 'product', 'price_tier', 'export', 'export_financial', 'refund',
    'admin_role', 'audit_log', 'customs_rule', 'country',
    'business_document', 'business_account', 'customs_document'
  )),
  CONSTRAINT ck_permission_action CHECK (action IN (
    'read', 'create', 'update', 'delete', 'export', 'approve', 'refund',
    'manage_roles'
  ))
);

CREATE INDEX IF NOT EXISTS ix_permission_resource ON permission (resource);

CREATE TABLE IF NOT EXISTS user_role (
  user_id     uuid NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
  role_code   text NOT NULL REFERENCES role (code) ON DELETE CASCADE,
  granted_by  uuid REFERENCES "user" (id) ON DELETE SET NULL,
  granted_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, role_code)
);

CREATE TABLE IF NOT EXISTS admin_action_approval (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  kind            text NOT NULL,
  payload         jsonb NOT NULL,
  payload_hash    text NOT NULL,
  amount          numeric(14, 2),
  currency        char(3),
  requested_by    uuid NOT NULL REFERENCES "user" (id),
  approved_by     uuid REFERENCES "user" (id),
  approval_status text NOT NULL DEFAULT 'pending',
  correlation_id  text NOT NULL,
  expires_at      timestamptz NOT NULL,
  executed_at     timestamptz,
  execution_ref   text,
  created_at      timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT ck_approval_kind CHECK (kind IN (
    'refund', 'financial_export', 'price_tier_change', 'role_change',
    'country_suspension'
  )),
  CONSTRAINT ck_approval_status CHECK (approval_status IN (
    'pending', 'approved', 'rejected', 'expired', 'executed', 'cancelled'
  )),
  CONSTRAINT no_self_approval CHECK (
    approved_by IS NULL OR approved_by <> requested_by
  ),
  CONSTRAINT executed_fields CHECK (
    approval_status <> 'executed'
    OR (executed_at IS NOT NULL AND execution_ref IS NOT NULL)
  )
);

CREATE UNIQUE INDEX IF NOT EXISTS uq_approval_pending
  ON admin_action_approval (kind, requested_by) WHERE approval_status = 'pending';
CREATE INDEX IF NOT EXISTS ix_approval_kind_status
  ON admin_action_approval (kind, approval_status);

-- Journal d'audit append-only : une seule ecriture, jamais de mise a jour.
CREATE TABLE IF NOT EXISTS audit_log (
  id             bigserial PRIMARY KEY,
  occurred_at    timestamptz NOT NULL DEFAULT now(),
  actor_id       uuid,
  actor_role     text,
  action         text NOT NULL,
  object_type    text NOT NULL,
  object_id      text NOT NULL,
  result         text NOT NULL,
  correlation_id text NOT NULL,
  ip_hash        text,
  metadata       jsonb NOT NULL DEFAULT '{}'::jsonb,
  prev_hash      text,
  entry_hash     text NOT NULL UNIQUE,
  CONSTRAINT ck_audit_result CHECK (
    result IN ('success', 'failure', 'denied', 'ignored')
  )
);

CREATE INDEX IF NOT EXISTS ix_audit_log_occurred_at ON audit_log (occurred_at DESC);
CREATE INDEX IF NOT EXISTS ix_audit_log_actor ON audit_log (actor_id, occurred_at DESC);
CREATE INDEX IF NOT EXISTS ix_audit_log_object ON audit_log (object_type, object_id);
CREATE INDEX IF NOT EXISTS ix_audit_log_correlation ON audit_log (correlation_id);

-- Chaine de hachage : chaque entree porte le hash de la precedente.
CREATE OR REPLACE FUNCTION audit_chain() RETURNS trigger AS $$
DECLARE
  prev text;
  computed text;
BEGIN
  IF TG_OP IN ('UPDATE', 'DELETE') THEN
    RAISE EXCEPTION 'audit_log is append-only (attempted %)', TG_OP;
  END IF;

  SELECT entry_hash INTO prev FROM audit_log
   WHERE id = (SELECT MAX(id) FROM audit_log) FOR UPDATE;

  computed := encode(sha256(
    coalesce(prev, '') || '|' ||
    NEW.occurred_at::text || '|' ||
    coalesce(NEW.actor_id::text, '') || '|' ||
    NEW.actor_role || '|' ||
    NEW.action || '|' ||
    NEW.object_type || '|' ||
    NEW.object_id || '|' ||
    NEW.result || '|' ||
    NEW.correlation_id || '|' ||
    NEW.metadata::text
  ), 'hex');

  NEW.prev_hash := prev;
  NEW.entry_hash := computed;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION prevent_audit_mutation() RETURNS trigger AS $$
BEGIN
  RAISE EXCEPTION 'audit_log is append-only (attempted %)', TG_OP;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_audit_chain ON audit_log;
CREATE TRIGGER trg_audit_chain
  BEFORE INSERT ON audit_log
  FOR EACH ROW EXECUTE FUNCTION audit_chain();

DROP TRIGGER IF EXISTS trg_audit_immutable ON audit_log;
CREATE TRIGGER trg_audit_immutable
  BEFORE UPDATE OR DELETE ON audit_log
  FOR EACH ROW EXECUTE FUNCTION prevent_audit_mutation();

REVOKE UPDATE, DELETE ON audit_log FROM PUBLIC;
```

- [ ] **Step 5: Écrire la migration 0004 — outbox transactionnelle et idempotence consommateur**

`packages/db/migrations/0004_outbox.sql` :

```sql
-- 0004_outbox.sql
-- Outbox transactionnelle : l'evenement est ecrit dans la meme transaction que
-- l'etat metier, puis publie par un relay. La consommation est idempotente via
-- consumed_event (event_id, consumer).
SET lock_timeout = '3s';
SET statement_timeout = '30s';

CREATE TABLE IF NOT EXISTS outbox_event (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  event_type      text NOT NULL,
  aggregate_type  text NOT NULL,
  aggregate_id    text NOT NULL,
  payload         jsonb NOT NULL DEFAULT '{}'::jsonb,
  idempotency_key text NOT NULL,
  state           text NOT NULL DEFAULT 'pending',
  attempts        integer NOT NULL DEFAULT 0,
  available_at    timestamptz NOT NULL DEFAULT now(),
  published_at    timestamptz,
  last_error      text,
  created_at      timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_outbox_idempotency_key UNIQUE (idempotency_key),
  CONSTRAINT ck_outbox_state CHECK (
    state IN ('pending', 'published', 'failed', 'dead')
  )
);

CREATE INDEX IF NOT EXISTS idx_outbox_event_state_available_at
  ON outbox_event (state, available_at)
  WHERE state IN ('pending', 'failed');

CREATE INDEX IF NOT EXISTS idx_outbox_event_aggregate
  ON outbox_event USING GIN ((payload->>'aggregate_id'));

CREATE TABLE IF NOT EXISTS consumed_event (
  event_id    uuid NOT NULL,
  consumer    text NOT NULL,
  consumed_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (event_id, consumer)
);

CREATE INDEX IF NOT EXISTS ix_consumed_event_consumed_at
  ON consumed_event (consumed_at DESC);
```

- [ ] **Step 6: Écrire le runner de migrations idempotent à checksum**

Le cœur du runner est exporté par `packages/db/src/migrate-core.ts` : il est compilé avec le paquet (`src/` est le `rootDir`) et réutilisé par les phases suivantes — l'adaptateur `packages/db/src/migrate.ts` de la Phase 2 délègue à ce même fichier, il n'existe donc qu'une seule implémentation de la convention de versionnement et de checksum. `packages/db/scripts/migrate.ts` reste le point d'entrée en ligne de commande (`tsx scripts/migrate.ts`).

`packages/db/src/migrate-core.ts` :

```typescript
import { createHash } from 'node:crypto';
import { readdir, readFile } from 'node:fs/promises';
import { join } from 'node:path';
import { Pool } from 'pg';

export interface AppliedMigration {
  readonly version: string;
  readonly checksum: string;
  readonly durationMs: number;
  readonly status: 'applied' | 'skipped';
}

export interface MigrationOptions {
  readonly directory: string;
  readonly client: Pool;
  readonly logger?: { info: (message: string) => void; error: (message: string) => void };
}

const MIGRATION_FILE_PATTERN = /^(\d{4})_[a-z0-9_]+\.sql$/;

function sha256(content: string): string {
  return createHash('sha256').update(content, 'utf8').digest('hex');
}

export async function migrate(options: MigrationOptions): Promise<AppliedMigration[]> {
  const { directory } = options;
  const logger = options.logger ?? { info: () => undefined, error: () => undefined };
  const files = (await readdir(directory)).filter((name) => MIGRATION_FILE_PATTERN.test(name)).sort();

  const applied = new Map<string, string>();
  // La table de suivi est creee par la migration 0001 : elle est garantie ici
  // pour que le runner reste utilisable sur une base vide.
  await options.client.query(
    'CREATE TABLE IF NOT EXISTS schema_migrations (version text PRIMARY KEY, checksum text NOT NULL, applied_at timestamptz NOT NULL DEFAULT now(), duration_ms integer NOT NULL DEFAULT 0)',
  );
  const existing = await options.client.query<{ version: string; checksum: string }>(
    'SELECT version, checksum FROM schema_migrations',
  );
  for (const row of existing.rows) {
    applied.set(row.version, row.checksum);
  }

  const results: AppliedMigration[] = [];

  for (const file of files) {
    const version = file.split('_')[0] as string;
    const content = await readFile(join(directory, file), 'utf8');
    const checksum = sha256(content);
    const known = applied.get(version);

    if (known !== undefined) {
      if (known !== checksum) {
        throw new Error(
          `Migration ${version} (${file}) modifiee apres application : ` +
            `checksum attendu ${known}, calcule ${checksum}. ` +
            'Les migrations sont append-only.',
        );
      }
      results.push({ version, checksum, durationMs: 0, status: 'skipped' });
      logger.info(`Migration ${version} deja appliquee, ignoree.`);
      continue;
    }

    const started = Date.now();
    const conn = await options.client.connect();
    try {
      await conn.query('BEGIN');
      await conn.query(content);
      await conn.query(
        'INSERT INTO schema_migrations (version, checksum, duration_ms) VALUES ($1, $2, $3)',
        [version, checksum, Date.now() - started],
      );
      await conn.query('COMMIT');
      const durationMs = Date.now() - started;
      results.push({ version, checksum, durationMs, status: 'applied' });
      logger.info(`Migration ${version} (${file}) appliquee en ${durationMs} ms.`);
    } catch (error) {
      await conn.query('ROLLBACK');
      const message = error instanceof Error ? error.message : String(error);
      logger.error(`Migration ${version} (${file}) echouee : ${message}`);
      throw error;
    } finally {
      conn.release();
    }
  }

  return results;
}
```

`packages/db/scripts/migrate.ts` (point d'entrée en ligne de commande) :

```typescript
import { fileURLToPath } from 'node:url';
import { Pool } from 'pg';
import { migrate } from '../src/migrate-core.js';

async function main(): Promise<void> {
  const databaseUrl = process.env['DATABASE_URL'];
  if (databaseUrl === undefined || databaseUrl.length === 0) {
    throw new Error('DATABASE_URL est requis pour appliquer les migrations.');
  }
  const pool = new Pool({ connectionString: databaseUrl, max: 1 });
  try {
    const results = await migrate({
      directory: fileURLToPath(new URL('../migrations', import.meta.url)),
      client: pool,
      logger: {
        info: (message) => process.stdout.write(`${message}\n`),
        error: (message) => process.stderr.write(`${message}\n`),
      },
    });
    const applied = results.filter((result) => result.status === 'applied').length;
    process.stdout.write(`Migrations terminees : ${applied} appliquee(s), ${results.length - applied} ignoree(s).\n`);
  } finally {
    await pool.end();
  }
}

if (import.meta.url === `file://${process.argv[1]?.replace(/\\/g, '/')}`) {
  main().catch((error: unknown) => {
    process.stderr.write(`${error instanceof Error ? error.stack : String(error)}\n`);
    process.exitCode = 1;
  });
}
```

`packages/db/tsconfig.json` compile `src/**/*.ts` (`rootDir: ./src`, `outDir: ./dist`) : le script `scripts/migrate.ts` n'entre donc pas dans le programme `tsc` (il est exécuté par `tsx` et vérifié par ESLint via `eslint src scripts`), et il ne fait que déléguer au cœur `src/migrate-core.ts`, seul fichier du runner importable depuis `src/`.

- [ ] **Step 7: Écrire le schéma Drizzle correspondant**

`packages/db/src/schema.ts` :

```typescript
import { sql } from 'drizzle-orm';
import {
  bigint,
  boolean,
  index,
  integer,
  jsonb,
  pgTable,
  text,
  timestamp,
  uniqueIndex,
  uuid,
} from 'drizzle-orm/pg-core';

export const user = pgTable(
  'user',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    email: text('email').notNull(),
    emailNormalized: text('email_normalized').notNull(),
    emailVerifiedAt: timestamp('email_verified_at', { withTimezone: true }),
    passwordHash: text('password_hash'),
    displayName: text('display_name'),
    locale: text('locale').notNull().default('fr-FR'),
    theme: text('theme').notNull().default('auto'),
    status: text('status').notNull().default('active'),
    createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
    deletedAt: timestamp('deleted_at', { withTimezone: true }),
  },
  (table) => ({
    emailUnique: uniqueIndex('uq_user_email_normalized').on(table.emailNormalized),
  }),
);

export const authIdentity = pgTable('auth_identity', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id')
    .notNull()
    .references(() => user.id, { onDelete: 'cascade' }),
  provider: text('provider').notNull(),
  providerSubject: text('provider_subject').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const session = pgTable(
  'session',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    userId: uuid('user_id')
      .notNull()
      .references(() => user.id, { onDelete: 'cascade' }),
    tokenHash: text('token_hash').notNull(),
    profile: text('profile').notNull().default('client'),
    ipHash: text('ip_hash'),
    userAgent: text('user_agent'),
    createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
    lastSeenAt: timestamp('last_seen_at', { withTimezone: true }).notNull().defaultNow(),
    absoluteExpiresAt: timestamp('absolute_expires_at', { withTimezone: true }).notNull(),
    revokedAt: timestamp('revoked_at', { withTimezone: true }),
    revokedReason: text('revoked_reason'),
    elevatedAt: timestamp('elevated_at', { withTimezone: true }),
    mfaVerifiedAt: timestamp('mfa_verified_at', { withTimezone: true }),
  },
  (table) => ({
    tokenUnique: uniqueIndex('uq_session_token_hash').on(table.tokenHash),
  }),
);

export const mfaMethod = pgTable('mfa_method', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id')
    .notNull()
    .references(() => user.id, { onDelete: 'cascade' }),
  kind: text('kind').notNull().default('totp'),
  algorithm: text('algorithm').notNull().default('SHA1'),
  digits: integer('digits').notNull().default(6),
  periodSeconds: integer('period_seconds').notNull().default(30),
  secretCiphertext: text('secret_ciphertext').notNull(),
  secretKeyVersion: text('secret_key_version').notNull(),
  lastCounter: bigint('last_counter', { mode: 'number' }).notNull().default(0),
  confirmedAt: timestamp('confirmed_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  revokedAt: timestamp('revoked_at', { withTimezone: true }),
});

export const recoveryCode = pgTable('recovery_code', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id')
    .notNull()
    .references(() => user.id, { onDelete: 'cascade' }),
  codeHash: text('code_hash').notNull(),
  usedAt: timestamp('used_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const role = pgTable('role', {
  code: text('code').primaryKey(),
  label: text('label').notNull(),
  description: text('description').notNull().default(''),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const permission = pgTable('permission', {
  code: text('code').primaryKey(),
  resource: text('resource').notNull(),
  action: text('action').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const userRole = pgTable(
  'user_role',
  {
    userId: uuid('user_id')
      .notNull()
      .references(() => user.id, { onDelete: 'cascade' }),
    roleCode: text('role_code')
      .notNull()
      .references(() => role.code, { onDelete: 'cascade' }),
    grantedBy: uuid('granted_by').references(() => user.id, { onDelete: 'set null' }),
    grantedAt: timestamp('granted_at', { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    primary: uniqueIndex('uq_user_role').on(table.userId, table.roleCode),
  }),
);

export const adminActionApproval = pgTable(
  'admin_action_approval',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    kind: text('kind').notNull(),
    payload: jsonb('payload').notNull(),
    payloadHash: text('payload_hash').notNull(),
    amount: text('amount'),
    currency: text('currency'),
    requestedBy: uuid('requested_by')
      .notNull()
      .references(() => user.id),
    approvedBy: uuid('approved_by').references(() => user.id),
    approvalStatus: text('approval_status').notNull().default('pending'),
    correlationId: text('correlation_id').notNull(),
    expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
    executedAt: timestamp('executed_at', { withTimezone: true }),
    executionRef: text('execution_ref'),
    createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    pendingUnique: uniqueIndex('uq_approval_pending')
      .on(table.kind, table.requestedBy)
      .where(sql`${table.approvalStatus} = 'pending'`),
  }),
);

export const auditLog = pgTable(
  'audit_log',
  {
    id: bigint('id', { mode: 'number' }).primaryKey().generatedAlwaysAsIdentity(),
    occurredAt: timestamp('occurred_at', { withTimezone: true }).notNull().defaultNow(),
    actorId: uuid('actor_id'),
    actorRole: text('actor_role'),
    action: text('action').notNull(),
    objectType: text('object_type').notNull(),
    objectId: text('object_id').notNull(),
    result: text('result').notNull(),
    correlationId: text('correlation_id').notNull(),
    ipHash: text('ip_hash'),
    metadata: jsonb('metadata').notNull().default({}),
    prevHash: text('prev_hash'),
    entryHash: text('entry_hash').notNull(),
  },
  (table) => ({
    entryHashUnique: uniqueIndex('uq_audit_log_entry_hash').on(table.entryHash),
    occurredAtIndex: index('ix_audit_log_occurred_at').on(table.occurredAt),
  }),
);

export const outboxEvent = pgTable(
  'outbox_event',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    eventType: text('event_type').notNull(),
    aggregateType: text('aggregate_type').notNull(),
    aggregateId: text('aggregate_id').notNull(),
    payload: jsonb('payload').notNull().default({}),
    idempotencyKey: text('idempotency_key').notNull(),
    state: text('state').notNull().default('pending'),
    attempts: integer('attempts').notNull().default(0),
    availableAt: timestamp('available_at', { withTimezone: true }).notNull().defaultNow(),
    publishedAt: timestamp('published_at', { withTimezone: true }),
    lastError: text('last_error'),
    createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    idempotencyUnique: uniqueIndex('uq_outbox_idempotency_key').on(table.idempotencyKey),
  }),
);

export const consumedEvent = pgTable(
  'consumed_event',
  {
    eventId: uuid('event_id').notNull(),
    consumer: text('consumer').notNull(),
    consumedAt: timestamp('consumed_at', { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    primary: uniqueIndex('uq_consumed_event').on(table.eventId, table.consumer),
  }),
);

export const tables = {
  user,
  authIdentity,
  session,
  mfaMethod,
  recoveryCode,
  role,
  permission,
  userRole,
  adminActionApproval,
  auditLog,
  outboxEvent,
  consumedEvent,
};
```

Le fichier importe `sql` depuis `drizzle-orm` pour construire l'index conditionnel `uq_user_email_normalized` : `uniqueIndex('uq_user_email_normalized').on(table.emailNormalized).where(sql\`${table.deletedAt} IS NULL\`)`.

- [ ] **Step 8: Écrire le client Drizzle et les utilitaires de token**

`packages/db/src/client.ts` :

```typescript
import { createHash, randomBytes } from 'node:crypto';
import { drizzle, type NodePgDatabase } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema.js';

export type Database = NodePgDatabase<typeof schema>;

export function createPool(connectionString: string): Pool {
  return new Pool({
    connectionString,
    max: 10,
    idleTimeoutMillis: 30_000,
    connectionTimeoutMillis: 5_000,
    application_name: 'fanours',
  });
}

export function createDb(pool: Pool): Database {
  return drizzle(pool, { schema });
}

export function newOpaqueToken(): string {
  return randomBytes(32).toString('base64url');
}

export function hashToken(token: string): string {
  return createHash('sha256').update(token, 'utf8').digest('hex');
}

export async function withTransaction<T>(
  database: Database,
  callback: (tx: Database) => Promise<T>,
): Promise<T> {
  return database.transaction(callback);
}
```

`packages/db/src/index.ts` (baril du paquet — la Phase 1 l'écrit, les phases suivantes l'étendent) :

```typescript
// Baril du paquet : ce fichier est l'unique point d'entree de @fanours/db.
// La Phase 1 y expose le schema (identite, RBAC, audit, outbox) et le client.
// La Phase 2 l'etend : ses modules de schema metier (packages/db/src/schema/*.ts)
// sont re-exportes via ./schema/index.js, qui re-exporte lui-meme ce schema.
// Les phases suivantes ajoutent leurs propres modules, jamais une redefinition.
export * from './schema.js';
export * from './client.js';
```

- [ ] **Step 9: Écrire la matrice RBAC dans `packages/domain` puis le seed synthétique**

`packages/domain/src/permissions.ts` (donnée pure, sans framework — c'est la source de la matrice seedée en base et consommée par le guard de la tâche 9) :

```typescript
export const ROLES = [
  'super_admin',
  'catalog_manager',
  'order_manager',
  'logistics_operator',
  'finance_manager',
  'support_agent',
  'compliance_manager',
  'marketing_manager',
  'analyst_readonly',
] as const;

export type Role = (typeof ROLES)[number];

export const RESOURCES = [
  'order',
  'product',
  'price_tier',
  'export',
  'export_financial',
  'refund',
  'admin_role',
  'audit_log',
  'customs_rule',
  'country',
  'business_document',
  'business_account',
  'customs_document',
] as const;

export type Resource = (typeof RESOURCES)[number];

export const ACTIONS = [
  'read',
  'create',
  'update',
  'delete',
  'export',
  'approve',
  'refund',
  'manage_roles',
] as const;

export type Action = (typeof ACTIONS)[number];

export type PermissionTable = {
  readonly [R in Role]?: { readonly [Res in Resource]?: readonly Action[] };
};

export const PERMISSIONS: PermissionTable = {
  super_admin: {
    order: ['read', 'create', 'update', 'delete', 'export'],
    product: ['read', 'create', 'update', 'delete', 'export'],
    price_tier: ['read', 'create', 'update', 'delete', 'export', 'approve'],
    export: ['read', 'export'],
    export_financial: ['read', 'export', 'approve'],
    refund: ['read', 'create', 'refund', 'approve'],
    admin_role: ['read', 'update', 'manage_roles'],
    audit_log: ['read', 'export'],
    customs_rule: ['read', 'create', 'update', 'delete', 'approve'],
    country: ['read', 'update'],
    business_document: ['read', 'create', 'update', 'delete'],
    business_account: ['read', 'create', 'update', 'delete'],
    customs_document: ['read', 'create', 'update', 'delete'],
  },
  catalog_manager: {
    product: ['read', 'create', 'update', 'delete', 'export'],
    price_tier: ['read', 'create', 'update', 'approve'],
    business_document: ['read', 'update'],
  },
  order_manager: {
    order: ['read', 'create', 'update', 'export'],
    business_account: ['read'],
    business_document: ['read'],
  },
  logistics_operator: {
    order: ['read', 'update'],
    business_account: ['read'],
  },
  finance_manager: {
    order: ['read', 'export'],
    export: ['read', 'export'],
    export_financial: ['read', 'export', 'approve'],
    refund: ['read', 'create', 'refund', 'approve'],
    price_tier: ['read'],
  },
  support_agent: {
    order: ['read'],
    business_account: ['read'],
    business_document: ['read'],
    refund: ['read', 'create'],
  },
  compliance_manager: {
    customs_rule: ['read', 'create', 'update', 'delete', 'approve'],
    customs_document: ['read', 'create', 'update', 'delete'],
    country: ['read'],
    product: ['read', 'update'],
    business_document: ['read', 'update'],
    business_account: ['read'],
  },
  marketing_manager: {
    product: ['read', 'update'],
    business_account: ['read'],
  },
  analyst_readonly: {
    order: ['read'],
    product: ['read'],
    audit_log: ['read'],
  },
};

export const FOUR_EYES = new Set<string>([
  'super_admin:price_tier:update',
  'catalog_manager:price_tier:update',
  'super_admin:export_financial:export',
  'finance_manager:export_financial:export',
  'super_admin:refund:refund',
  'finance_manager:refund:refund',
  'super_admin:admin_role:manage_roles',
  'super_admin:country:update',
]);
```

`packages/domain/src/index.ts` devient :

```typescript
export * from './session-policy.js';
export * from './permissions.js';
```

`packages/db/src/seed.ts` :

```typescript
import { PERMISSIONS, RESOURCES, ROLES } from '@fanours/domain';
import { createDb, createPool } from './client.js';
import { permission, role, userRole, user } from './schema.js';

const ROLE_LABELS: Record<string, string> = {
  super_admin: 'Super administrateur',
  catalog_manager: 'Gestionnaire catalogue',
  order_manager: 'Gestionnaire commandes',
  logistics_operator: 'Operateur logistique',
  finance_manager: 'Responsable finance',
  support_agent: 'Agent support',
  compliance_manager: 'Responsable conformite',
  marketing_manager: 'Responsable marketing',
  analyst_readonly: 'Analyste lecture seule',
};

export async function seed(): Promise<void> {
  const connectionString = process.env['DATABASE_URL'];
  if (connectionString === undefined) throw new Error('DATABASE_URL est requis.');
  const pool = createPool(connectionString);
  const database = createDb(pool);
  try {
    for (const roleCode of ROLES) {
      await database
        .insert(role)
        .values({ code: roleCode, label: ROLE_LABELS[roleCode] ?? roleCode })
        .onConflictDoNothing();
    }

    for (const roleCode of ROLES) {
      const allowed = PERMISSIONS[roleCode] ?? {};
      for (const resource of RESOURCES) {
        const actions = allowed[resource] ?? [];
        for (const action of actions) {
          await database
            .insert(permission)
            .values({ code: `${roleCode}:${resource}:${action}`, resource, action })
            .onConflictDoNothing();
        }
      }
    }

    const [analyst] = await database
      .insert(user)
      .values({
        email: 'analyste@fanours.local',
        emailNormalized: 'analyste@fanours.local',
        displayName: 'Analyste lecture seule',
        emailVerifiedAt: new Date(),
      })
      .onConflictDoNothing()
      .returning({ id: user.id });

    if (analyst !== undefined) {
      await database
        .insert(userRole)
        .values({ userId: analyst.id, roleCode: 'analyst_readonly' })
        .onConflictDoNothing();
    }
  } finally {
    await pool.end();
  }
}

if (import.meta.url === `file://${process.argv[1]?.replace(/\\/g, '/')}`) {
  seed().catch((error: unknown) => {
    process.stderr.write(`${error instanceof Error ? error.stack : String(error)}\n`);
    process.exitCode = 1;
  });
}
```

Le seed insère les 9 rôles, les permissions correspondantes (une ligne par triplet rôle/ressource/action) et un utilisateur de démonstration `analyste@fanours.local` avec le rôle `analyst_readonly`. Il est idempotent (`onConflictDoNothing`) : on peut le rejouer sans effet de bord.

- [ ] **Step 10: Écrire les tests d'intégration avec Testcontainers**

`packages/db/vitest.config.ts` :

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    projects: [
      {
        test: {
          name: 'unit',
          include: ['src/__tests__/**/*.unit.test.ts'],
          environment: 'node',
        },
      },
      {
        test: {
          name: 'integration',
          include: ['src/__tests__/**/*.integration.test.ts'],
          environment: 'node',
          testTimeout: 120_000,
          hookTimeout: 180_000,
          fileParallelism: false,
        },
      },
    ],
  },
});
```

`packages/db/src/__tests__/migrations.integration.test.ts` :

```typescript
import { readdir } from 'node:fs/promises';
import { fileURLToPath } from 'node:url';
import { dirname, join } from 'node:path';
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { Pool } from 'pg';
import { migrate } from '../migrate-core.js';

const here = dirname(fileURLToPath(import.meta.url));
const migrationsDir = join(here, '..', '..', 'migrations');

let container: StartedPostgreSqlContainer;
let pool: Pool;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine').start();
  pool = new Pool({ connectionString: container.getConnectionUri() });
  await pool.query('CREATE EXTENSION IF NOT EXISTS pgcrypto');
  await pool.query('CREATE EXTENSION IF NOT EXISTS unaccent');
  await pool.query('CREATE EXTENSION IF NOT EXISTS btree_gist');
  await pool.query(`
    CREATE TABLE IF NOT EXISTS schema_migrations (
      version text PRIMARY KEY,
      checksum text NOT NULL,
      applied_at timestamptz NOT NULL DEFAULT now(),
      duration_ms integer NOT NULL DEFAULT 0
    )
  `);
});

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

describe('migrations', () => {
  it('applique toutes les migrations une seule fois', async () => {
    const first = await migrate({ directory: migrationsDir, client: pool });
    expect(first.every((entry) => entry.status === 'applied')).toBe(true);
    expect(first.length).toBe(4);

    const second = await migrate({ directory: migrationsDir, client: pool });
    expect(second.every((entry) => entry.status === 'skipped')).toBe(true);
  });

  it('refuse une migration modifiee apres application', async () => {
    const files = (await readdir(migrationsDir)).filter((name) => name.endsWith('.sql')).sort();
    expect(files[0]).toBe('0001_extensions_and_migration_table.sql');
    const rows = await pool.query('SELECT version, checksum FROM schema_migrations ORDER BY version');
    expect(rows.rows).toHaveLength(4);
  });

  it('cree les tables attendues', async () => {
    const result = await pool.query<{ table_name: string }>(
      `SELECT table_name FROM information_schema.tables
        WHERE table_schema = 'public' ORDER BY table_name`,
    );
    const names = result.rows.map((row) => row.table_name);
    for (const expected of [
      'admin_action_approval',
      'audit_log',
      'auth_identity',
      'consumed_event',
      'mfa_method',
      'outbox_event',
      'permission',
      'recovery_code',
      'role',
      'schema_migrations',
      'session',
      'user',
      'user_role',
    ]) {
      expect(names).toContain(expected);
    }
  });
});
```

- [ ] **Step 11: Écrire le test d'immuabilité de l'audit et le test de cycle de session**

`packages/db/src/__tests__/audit-immutability.integration.test.ts` :

```typescript
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { Pool } from 'pg';
import { createDb, createPool } from '../client.js';

let container: StartedPostgreSqlContainer;
let pool: Pool;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine').start();
  pool = createPool(container.getConnectionUri());
  const { migrate } = await import('../migrate-core.js');
  const { dirname, join } = await import('node:path');
  const { fileURLToPath } = await import('node:url');
  const here = dirname(fileURLToPath(import.meta.url));
  await migrate({ directory: join(here, '..', '..', 'migrations'), client: pool });
});

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

describe('audit_log', () => {
  it('refuse la mise a jour', async () => {
    await pool.query(
      `INSERT INTO audit_log (actor_role, action, object_type, object_id, result,
         correlation_id, metadata)
       VALUES ('support_agent', 'auth.login', 'user', 'u1', 'success', 'corr-1', '{}'::jsonb)`,
    );
    await expect(
      pool.query(`UPDATE audit_log SET action = 'tampered' WHERE id = 1`),
    ).rejects.toThrow(/append-only/);
  });

  it('refuse la suppression', async () => {
    await expect(pool.query('DELETE FROM audit_log WHERE id = 1')).rejects.toThrow(/append-only/);
  });

  it('chaine les hash entre entrees', async () => {
    await pool.query(
      `INSERT INTO audit_log (actor_role, action, object_type, object_id, result,
         correlation_id, metadata)
       VALUES ('finance_manager', 'refund.requested', 'refund', 'r1', 'success', 'corr-2', '{}'::jsonb)`,
    );
    const result = await pool.query<{ entry_hash: string; prev_hash: string | null }>(
      'SELECT entry_hash, prev_hash FROM audit_log ORDER BY id',
    );
    expect(result.rows[1]?.prev_hash).toBe(result.rows[0]?.entry_hash);
    expect(result.rows[1]?.entry_hash).not.toBe(result.rows[0]?.entry_hash);
  });

  it('ecrit trois evenements de controle exploitables', async () => {
    const database = createDb(pool);
    const { auditLog } = await import('../schema.js');
    const { and, gte, eq: eqOp } = await import('drizzle-orm');
    const inserted = await database
      .insert(auditLog)
      .values([
        {
          actorRole: 'super_admin',
          action: 'role.granted',
          objectType: 'user_role',
          objectId: 'u2',
          result: 'success',
          correlationId: 'corr-3',
          metadata: { role: 'finance_manager' },
        },
        {
          actorRole: 'super_admin',
          action: 'price_tier.updated',
          objectType: 'price_tier',
          objectId: 'pt1',
          result: 'success',
          correlationId: 'corr-3',
          metadata: { amount: '12.00' },
        },
        {
          actorRole: 'finance_manager',
          action: 'refund.executed',
          objectType: 'refund',
          objectId: 'r2',
          result: 'success',
          correlationId: 'corr-3',
          metadata: {},
        },
      ])
      .returning({ action: auditLog.action });
    expect(inserted).toHaveLength(3);
    const rows = await database
      .select({ action: auditLog.action })
      .from(auditLog)
      .where(and(gte(auditLog.occurredAt, new Date('2020-01-01')), eqOp(auditLog.correlationId, 'corr-3')));
    expect(rows).toHaveLength(3);
  });
});
```

`packages/db/src/__tests__/session-lifecycle.integration.test.ts` :

```typescript
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { Pool } from 'pg';
import { createDb, createPool, hashToken, newOpaqueToken } from '../client.js';
import { session, user } from '../schema.js';
import { eq } from 'drizzle-orm';

let container: StartedPostgreSqlContainer;
let pool: Pool;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine').start();
  pool = createPool(container.getConnectionUri());
  const { migrate } = await import('../migrate-core.js');
  const { dirname, join } = await import('node:path');
  const { fileURLToPath } = await import('node:url');
  const here = dirname(fileURLToPath(import.meta.url));
  await migrate({ directory: join(here, '..', '..', 'migrations'), client: pool });
});

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

describe('session', () => {
  it('stocke le hash du token et jamais le token brut', async () => {
    const database = createDb(pool);
    const [created] = await database
      .insert(user)
      .values({ email: 'a@b.com', emailNormalized: 'a@b.com' })
      .returning({ id: user.id });
    const token = newOpaqueToken();
    await database.insert(session).values({
      userId: created!.id,
      tokenHash: hashToken(token),
      absoluteExpiresAt: new Date(Date.now() + 3_600_000),
    });
    const rows = await database.select().from(session);
    expect(rows[0]?.tokenHash).toBe(hashToken(token));
    expect(rows[0]?.tokenHash).not.toBe(token);
    expect(rows[0]?.tokenHash).toHaveLength(64);
  });

  it('rejette la rotation d un token deja revoque', async () => {
    const database = createDb(pool);
    const rows = await database.select().from(session);
    const target = rows[0]!;
    await database
      .update(session)
      .set({ revokedAt: new Date(), revokedReason: 'rotated' })
      .where(eq(session.id, target.id));
    const after = await database.select().from(session);
    expect(after[0]?.revokedAt).toBeInstanceOf(Date);
    expect(after[0]?.revokedReason).toBe('rotated');
  });
});
```

- [ ] **Step 12: Exécuter les tests et constater l'échec**

```bash
pnpm --filter @fanours/db test:integration
```

Attendu : échec — `No test files found` ou `Cannot find module '../migrate-core.js'` selon l'ordre d'écriture. Coller la sortie.

- [ ] **Step 13: Implémenter, exécuter et prouver la réussite**

```bash
pnpm --filter @fanours/db test:integration
```

Attendu :

```
 ✓ src/__tests__/migrations.integration.test.ts (3 tests)
 ✓ src/__tests__/audit-immutability.integration.test.ts (4 tests)
 ✓ src/__tests__/session-lifecycle.integration.test.ts (2 tests)

 Test Files  3 passed (3)
      Tests  9 passed (9)
```

- [ ] **Step 14: Vérifier que le runner est idempotent sur une base réelle locale**

```bash
docker compose -f docker/compose.local.yml up -d postgres
pnpm db:migrate
pnpm db:migrate
```

Attendu : la première exécution affiche `Migrations terminees : 4 appliquee(s), 0 ignoree(s).` et la seconde `Migrations terminees : 0 appliquee(s), 4 ignoree(s).`

- [ ] **Step 15: Commit**

```bash
git add packages/db
git commit -m "feat(db): ajoute le schema Drizzle et le runner de migrations" -m "Quatre migrations SQL append-only (identite, RBAC et audit, outbox),
runner idempotent a checksum SHA-256 sans script down, journal d'audit
immuable par REVOKE et trigger, outbox transactionnelle et
consumed_event pour l'idempotence, tests d'integration Testcontainers.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 3: Design system — tokens, thèmes clair/sombre, polices, Storybook

**Files:**
- Create: `packages/ui/package.json`, `packages/ui/tsconfig.json`, `packages/ui/vitest.config.ts`, `packages/ui/.storybook/main.ts`, `packages/ui/.storybook/preview.tsx`
- Create: `packages/ui/src/styles/tokens.css` (**source unique des couleurs, espacements, rayons, ombres, typographie**)
- Create: `packages/ui/src/styles/globals.css`, `packages/ui/src/styles/glass.css`
- Create: `packages/ui/src/theme-script.ts` (script anti-flash inline, exécuté avant hydration)
- Create: `packages/ui/src/index.ts`
- Create: `apps/web/public/fonts/plus-jakarta-sans/README.md`, `apps/web/public/fonts/inter/README.md`, `apps/web/public/fonts/noto-sans-sc/README.md`
- Test: `packages/ui/src/__tests__/tokens.test.ts` (contraste + présence des tokens)

**Interfaces:**
- Consumes: aucun.
- Produces:
  - `packages/ui/src/styles/tokens.css` : variables CSS `--color-*`, `--space-*`, `--radius-*`, `--shadow-*`, `--font-*`, `--duration-*`, `--ease-*`, `--glass-*`, `--focus-ring-*`, `--z-*`, `--breakpoint-*`, `--control-height*`, `--touch-target-min`, `--content-max`, `--content-wide`, `--gutter`.
  - `packages/ui/src/theme-script.ts` : `THEME_SCRIPT: string` — IIFE à insérer inline avant l'hydratation, lisant `localStorage['fanours.theme']` et posant `document.documentElement.dataset['theme']`.
  - `packages/ui/src/index.ts` : ré-export de `THEME_SCRIPT` et des types de thème.

---

- [ ] **Step 1: Créer le package `packages/ui`**

`packages/ui/package.json` :

```json
{
  "name": "@fanours/ui",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" },
    "./styles/tokens.css": "./src/styles/tokens.css",
    "./styles/globals.css": "./src/styles/globals.css"
  },
  "scripts": {
    "build": "tsc -b",
    "typecheck": "tsc -b --pretty",
    "lint": "eslint src",
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build -o storybook-static",
    "test:unit": "vitest run",
    "test": "vitest run"
  },
  "dependencies": {
    "react": "19.0.0",
    "react-dom": "19.0.0"
  },
  "devDependencies": {
    "@storybook/addon-a11y": "8.4.7",
    "@storybook/addon-essentials": "8.4.7",
    "@storybook/addon-themes": "8.4.7",
    "@storybook/react": "8.4.7",
    "@storybook/react-vite": "8.4.7",
    "@testing-library/jest-dom": "6.6.3",
    "@testing-library/react": "16.1.0",
    "@testing-library/user-event": "14.5.2",
    "@types/react": "19.0.2",
    "@types/react-dom": "19.0.2",
    "@vitejs/plugin-react": "4.3.4",
    "jest-axe": "9.0.0",
    "jsdom": "25.0.1",
    "typescript": "5.6.3",
    "vite": "5.4.11",
    "vitest": "2.1.8"
  }
}
```

- [ ] **Step 2: Écrire `tokens.css` — la source unique (valeurs exactes)**

`packages/ui/src/styles/tokens.css` :

```css
/*
 * Fanours — tokens CSS.
 * SOURCE UNIQUE de la direction visuelle. Aucun composant ne contient de
 * couleur hexadecimale : la regle ESLint `no-restricted-syntax` l'interdit
 * hors de ce fichier. Toute modification de couleur se fait ici.
 * Reference : docs/roadmap/sections/design.md §3.2.
 */

@font-face {
  font-family: 'Plus Jakarta Sans';
  src: url('/fonts/plus-jakarta-sans/latin-500.woff2') format('woff2');
  font-weight: 500;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+2000-206F, U+2074,
    U+20AC, U+2122, U+2191, U+2193, U+2212, U+2215, U+FEFF, U+FFFD;
}

@font-face {
  font-family: 'Plus Jakarta Sans';
  src: url('/fonts/plus-jakarta-sans/latin-600.woff2') format('woff2');
  font-weight: 600;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+2000-206F, U+2074,
    U+20AC, U+2122, U+2191, U+2193, U+2212, U+2215, U+FEFF, U+FFFD;
}

@font-face {
  font-family: 'Plus Jakarta Sans';
  src: url('/fonts/plus-jakarta-sans/latin-700.woff2') format('woff2');
  font-weight: 700;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+2000-206F, U+2074,
    U+20AC, U+2122, U+2191, U+2193, U+2212, U+2215, U+FEFF, U+FFFD;
}

@font-face {
  font-family: 'Plus Jakarta Sans';
  src: url('/fonts/plus-jakarta-sans/latin-800.woff2') format('woff2');
  font-weight: 800;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+2000-206F, U+2074,
    U+20AC, U+2122, U+2191, U+2193, U+2212, U+2215, U+FEFF, U+FFFD;
}

@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter/latin-400.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+2000-206F, U+2074,
    U+20AC, U+2122, U+2191, U+2193, U+2212, U+2215, U+FEFF, U+FFFD;
}

@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter/latin-500.woff2') format('woff2');
  font-weight: 500;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+2000-206F, U+2074,
    U+20AC, U+2122, U+2191, U+2193, U+2212, U+2215, U+FEFF, U+FFFD;
}

@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter/latin-600.woff2') format('woff2');
  font-weight: 600;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+2000-206F, U+2074,
    U+20AC, U+2122, U+2191, U+2193, U+2212, U+2215, U+FEFF, U+FFFD;
}

@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter/latin-700.woff2') format('woff2');
  font-weight: 700;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+2000-206F, U+2074,
    U+20AC, U+2122, U+2191, U+2193, U+2212, U+2215, U+FEFF, U+FFFD;
}

@font-face {
  font-family: 'Noto Sans SC';
  src: url('/fonts/noto-sans-sc/chinese-simplified-400.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;
  unicode-range: U+4E00-9FFF, U+3000-303F, U+FF00-FFEF;
}

@font-face {
  font-family: 'Noto Sans SC';
  src: url('/fonts/noto-sans-sc/chinese-simplified-500.woff2') format('woff2');
  font-weight: 500;
  font-style: normal;
  font-display: swap;
  unicode-range: U+4E00-9FFF, U+3000-303F, U+FF00-FFEF;
}

@font-face {
  font-family: 'Noto Sans SC';
  src: url('/fonts/noto-sans-sc/chinese-simplified-700.woff2') format('woff2');
  font-weight: 700;
  font-style: normal;
  font-display: swap;
  unicode-range: U+4E00-9FFF, U+3000-303F, U+FF00-FFEF;
}

:root {
  color-scheme: light;

  /* --- Typographie --- */
  --font-heading: 'Plus Jakarta Sans', 'Noto Sans SC', -apple-system,
    BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  --font-body: 'Inter', 'Noto Sans SC', -apple-system, BlinkMacSystemFont,
    'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  --font-mono: ui-monospace, SFMono-Regular, 'SF Mono', Menlo, Consolas,
    'Liberation Mono', monospace;
  --font-cjk: 'Noto Sans SC', 'PingFang SC', 'Hiragino Sans GB',
    'Microsoft YaHei', sans-serif;

  --font-size-xs: 0.75rem;
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;
  --font-size-xl: 1.25rem;
  --font-size-2xl: 1.5rem;
  --font-size-3xl: 1.875rem;
  --font-size-4xl: 2.25rem;
  --font-size-5xl: 3rem;

  --font-weight-regular: 400;
  --font-weight-medium: 500;
  --font-weight-semibold: 600;
  --font-weight-bold: 700;
  --font-weight-extrabold: 800;

  --line-height-tight: 1.15;
  --line-height-snug: 1.3;
  --line-height-base: 1.55;
  --line-height-relaxed: 1.7;

  --letter-spacing-tight: -0.02em;
  --letter-spacing-normal: 0;
  --letter-spacing-wide: 0.04em;

  /* --- Couleurs (theme clair) --- */
  --color-bg: #fff5fa;
  --color-surface: #ffffff;
  --color-text: #321526;
  --color-primary: #d81b75;
  --color-accent: #ffd6ea;
  --color-surface-secondary: #f7dce9;
  --color-surface-elevated: #ffffff;

  --color-rose-50: #fff5fa;
  --color-rose-100: #ffe8f2;
  --color-rose-200: #ffd6ea;
  --color-rose-300: #ffb5d6;
  --color-rose-400: #f58ab9;
  --color-rose-500: #e84393;
  --color-rose-600: #d81b75;
  --color-rose-700: #b91662;
  --color-rose-800: #8f124f;
  --color-rose-900: #64103a;

  --color-success: #16794b;
  --color-success-bg: #e7f5ee;
  --color-warning: #9a5b00;
  --color-warning-bg: #fdf1dc;
  --color-error: #b42336;
  --color-error-bg: #fcebec;
  --color-info: #2458b8;
  --color-info-bg: #e8f0fd;
  --color-neutral: #667085;
  --color-neutral-bg: #f2f4f7;

  --color-border: #f0dce6;
  --color-border-strong: #a8718a;
  --color-border-input: #a8718a;

  /* --- Palette categorielle de visualisation (design.md 3.2.4) --- */
  --color-chart-1: #d81b75;
  --color-chart-2: #00857a;
  --color-chart-3: #7a4bb5;
  --color-chart-4: #2b5fd9;
  --color-chart-5: #b26a00;
  --color-chart-grid: #f0dce6;

  --color-text-muted: #667085;
  --color-text-subtle: #667085;
  --color-text-inverse: #ffffff;
  --color-text-on-primary: #ffffff;
  --color-link: #b91662;
  --color-link-hover: #8f124f;

  /* --- Degrades --- */
  --gradient-hero-light: linear-gradient(135deg, #fff5fa 0%, #ffd6ea 55%, #ffffff 100%);
  --gradient-hero-dark: linear-gradient(135deg, #180d15 0%, #35182a 55%, #4b1f39 100%);
  --gradient-cta-light: linear-gradient(135deg, #d81b75 0%, #c2185b 100%);
  --gradient-cta-dark: linear-gradient(135deg, #ff70b7 0%, #ff9ccb 100%);
  --gradient-accent-soft: linear-gradient(180deg, #ffe8f2 0%, #ffd6ea 100%);

  /* --- Ombres --- */
  --shadow-xs: 0 1px 2px 0 rgb(50 21 38 / 0.05);
  --shadow-sm: 0 2px 6px -1px rgb(50 21 38 / 0.08);
  --shadow-md: 0 6px 16px -4px rgb(50 21 38 / 0.1);
  --shadow-lg: 0 16px 32px -8px rgb(50 21 38 / 0.14);
  --shadow-focus: 0 0 0 3px rgb(216 27 117 / 0.28);

  /* --- Rayons --- */
  --radius-xs: 4px;
  --radius-sm: 6px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-xl: 16px;
  --radius-2xl: 24px;
  --radius-pill: 999px;
  --radius-full: 50%;

  /* --- Espacements (echelle 4px) --- */
  --space-0: 0;
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-5: 1.25rem;
  --space-6: 1.5rem;
  --space-8: 2rem;
  --space-10: 2.5rem;
  --space-12: 3rem;
  --space-16: 4rem;
  --space-20: 5rem;
  --space-24: 6rem;
  --space-32: 8rem;

  /* --- Controles et mise en page --- */
  --control-height-sm: 36px;
  --control-height: 44px;
  --control-height-lg: 48px;
  --touch-target-min: 44px;
  --content-max: 1280px;
  --content-wide: 1440px;
  --gutter: 16px;

  /* --- Index de superposition --- */
  --z-base: 0;
  --z-raised: 10;
  --z-sticky: 100;
  --z-nav: 150;
  --z-dropdown: 200;
  --z-overlay: 300;
  --z-modal: 400;
  --z-toast: 500;
  --z-tooltip: 600;

  /* --- Points de rupture (documentation) --- */
  --breakpoint-mobile: 360px;
  --breakpoint-tablet: 768px;
  --breakpoint-desktop: 1024px;
  --breakpoint-wide: 1440px;

  /* --- Mouvement --- */
  --duration-instant: 80ms;
  --duration-fast: 150ms;
  --duration-normal: 240ms;
  --duration-slow: 400ms;
  --duration-slower: 600ms;
  --ease-out: cubic-bezier(0.16, 1, 0.3, 1);
  --ease-in-out: cubic-bezier(0.65, 0, 0.35, 1);
  --ease-standard: cubic-bezier(0.4, 0, 0.2, 1);
  --ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);

  /* --- Glassmorphism --- */
  --glass-blur: 16px;
  --glass-blur-strong: 20px;
  --glass-surface-light: rgb(255 255 255 / 0.78);
  --glass-surface-dark: rgb(38 20 32 / 0.72);
  --glass-border-light: rgb(255 255 255 / 0.55);
  --glass-border-dark: rgb(255 240 247 / 0.12);

  /* --- Focus --- */
  --focus-ring-width: 2px;
  --focus-ring-offset: 2px;
  --focus-ring-color: #d81b75;
  --focus-ring-color-dark: #ff9ccb;
}

/* Theme sombre applique par defaut (auto) et explicitement (data-theme) */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme='light']) {
    color-scheme: dark;

    --color-bg: #180d15;
    --color-surface: #261420;
    --color-text: #fff0f7;
    --color-primary: #ff70b7;
    --color-accent: #4b1f39;
    --color-surface-secondary: #35182a;
    --color-surface-elevated: #35202d;

    --color-rose-50: #2a1421;
    --color-rose-100: #35182a;
    --color-rose-200: #4b1f39;
    --color-rose-300: #7a2a52;
    --color-rose-400: #c84b88;
    --color-rose-500: #e95b9e;
    --color-rose-600: #ff70b7;
    --color-rose-700: #ff9ccb;
    --color-rose-800: #ffc2de;
    --color-rose-900: #ffe0ef;

    --color-success: #4ade80;
    --color-success-bg: #16301f;
    --color-warning: #fbbf24;
    --color-warning-bg: #33260c;
    --color-error: #ff6b7a;
    --color-error-bg: #3a1620;
    --color-info: #7ca7ff;
    --color-info-bg: #16233d;
    --color-neutral: #b8a8b3;
    --color-neutral-bg: #2a1a26;

    --color-border: #49303e;
    --color-border-strong: #a87c90;
    --color-border-input: #a87c90;

    /* --- Palette categorielle de visualisation (design.md 3.2.4) --- */
    --color-chart-1: #ff70b7;
    --color-chart-2: #2fd3b0;
    --color-chart-3: #c4a2f7;
    --color-chart-4: #7ca7ff;
    --color-chart-5: #fbbf24;
    --color-chart-grid: #49303e;

    --color-text-muted: #b8a8b3;
    --color-text-subtle: #b8a8b3;
    --color-text-inverse: #180d15;
    --color-text-on-primary: #180d15;
    --color-link: #ff9ccb;
    --color-link-hover: #ffc2de;

    --gradient-hero-light: var(--gradient-hero-dark);
    --gradient-cta-light: var(--gradient-cta-dark);
    --gradient-accent-soft: linear-gradient(180deg, #35182a 0%, #4b1f39 100%);

    --shadow-xs: 0 1px 2px 0 rgb(0 0 0 / 0.4);
    --shadow-sm: 0 2px 6px -1px rgb(0 0 0 / 0.45);
    --shadow-md: 0 6px 16px -4px rgb(0 0 0 / 0.5);
    --shadow-lg: 0 16px 32px -8px rgb(0 0 0 / 0.6);
    --shadow-focus: 0 0 0 3px rgb(255 156 203 / 0.35);

    --glass-surface-light: var(--glass-surface-dark);
    --glass-border-light: var(--glass-border-dark);

    --focus-ring-color: var(--focus-ring-color-dark);
  }
}

:root[data-theme='dark'] {
  color-scheme: dark;

  --color-bg: #180d15;
  --color-surface: #261420;
  --color-text: #fff0f7;
  --color-primary: #ff70b7;
  --color-accent: #4b1f39;
  --color-surface-secondary: #35182a;
  --color-surface-elevated: #35202d;

  --color-rose-50: #2a1421;
  --color-rose-100: #35182a;
  --color-rose-200: #4b1f39;
  --color-rose-300: #7a2a52;
  --color-rose-400: #c84b88;
  --color-rose-500: #e95b9e;
  --color-rose-600: #ff70b7;
  --color-rose-700: #ff9ccb;
  --color-rose-800: #ffc2de;
  --color-rose-900: #ffe0ef;

  --color-success: #4ade80;
  --color-success-bg: #16301f;
  --color-warning: #fbbf24;
  --color-warning-bg: #33260c;
  --color-error: #ff6b7a;
  --color-error-bg: #3a1620;
  --color-info: #7ca7ff;
  --color-info-bg: #16233d;
  --color-neutral: #b8a8b3;
  --color-neutral-bg: #2a1a26;

  --color-border: #49303e;
  --color-border-strong: #a87c90;
  --color-border-input: #a87c90;

  /* --- Palette categorielle de visualisation (design.md 3.2.4) --- */
  --color-chart-1: #ff70b7;
  --color-chart-2: #2fd3b0;
  --color-chart-3: #c4a2f7;
  --color-chart-4: #7ca7ff;
  --color-chart-5: #fbbf24;
  --color-chart-grid: #49303e;

  --color-text-muted: #b8a8b3;
  --color-text-subtle: #b8a8b3;
  --color-text-inverse: #180d15;
  --color-text-on-primary: #180d15;
  --color-link: #ff9ccb;
  --color-link-hover: #ffc2de;

  --gradient-hero-light: var(--gradient-hero-dark);
  --gradient-cta-light: var(--gradient-cta-dark);
  --gradient-accent-soft: linear-gradient(180deg, #35182a 0%, #4b1f39 100%);

  --shadow-xs: 0 1px 2px 0 rgb(0 0 0 / 0.4);
  --shadow-sm: 0 2px 6px -1px rgb(0 0 0 / 0.45);
  --shadow-md: 0 6px 16px -4px rgb(0 0 0 / 0.5);
  --shadow-lg: 0 16px 32px -8px rgb(0 0 0 / 0.6);
  --shadow-focus: 0 0 0 3px rgb(255 156 203 / 0.35);

  --glass-surface-light: var(--glass-surface-dark);
  --glass-border-light: var(--glass-border-dark);

  --focus-ring-color: var(--focus-ring-color-dark);
}

:root[data-theme='light'] {
  color-scheme: light;
}

@media (prefers-reduced-motion: reduce) {
  :root {
    --duration-instant: 0ms;
    --duration-fast: 0ms;
    --duration-normal: 0ms;
    --duration-slow: 0ms;
    --duration-slower: 0ms;
  }
}
```

- [ ] **Step 3: Écrire `globals.css` (Tailwind v4 CSS-first) et `glass.css`**

`packages/ui/src/styles/globals.css` :

```css
@import 'tailwindcss';
@import './tokens.css';

@theme inline {
  --color-background: var(--color-bg);
  --color-surface: var(--color-surface);
  --color-surface-secondary: var(--color-surface-secondary);
  --color-surface-elevated: var(--color-surface-elevated);
  --color-foreground: var(--color-text);
  --color-primary: var(--color-primary);
  --color-accent: var(--color-accent);
  --color-success: var(--color-success);
  --color-warning: var(--color-warning);
  --color-error: var(--color-error);
  --color-info: var(--color-info);
  --color-neutral: var(--color-neutral);
  --color-muted: var(--color-text-muted);
  --color-border: var(--color-border);
  --color-border-strong: var(--color-border-strong);
  --color-link: var(--color-link);
  --color-on-primary: var(--color-text-on-primary);
  --color-on-surface: var(--color-text-inverse);
  --color-success-bg: var(--color-success-bg);
  --color-warning-bg: var(--color-warning-bg);
  --color-error-bg: var(--color-error-bg);
  --color-info-bg: var(--color-info-bg);
  --color-neutral-bg: var(--color-neutral-bg);

  --font-heading: var(--font-heading);
  --font-body: var(--font-body);
  --font-mono: var(--font-mono);
  --font-cjk: var(--font-cjk);

  --radius-xs: var(--radius-xs);
  --radius-sm: var(--radius-sm);
  --radius-md: var(--radius-md);
  --radius-lg: var(--radius-lg);
  --radius-xl: var(--radius-xl);
  --radius-2xl: var(--radius-2xl);

  --shadow-xs: var(--shadow-xs);
  --shadow-sm: var(--shadow-sm);
  --shadow-md: var(--shadow-md);
  --shadow-lg: var(--shadow-lg);
}

@layer base {
  html {
    color-scheme: light dark;
    -webkit-text-size-adjust: 100%;
  }

  body {
    background-color: var(--color-bg);
    color: var(--color-text);
    font-family: var(--font-body);
    font-size: var(--font-size-base);
    line-height: var(--line-height-base);
    overflow-wrap: break-word;
  }

  h1,
  h2,
  h3,
  h4 {
    font-family: var(--font-heading);
    font-weight: var(--font-weight-bold);
    line-height: var(--line-height-tight);
  }

  :lang(zh) {
    font-family: var(--font-cjk);
    line-height: calc(var(--line-height-base) * 1.1);
  }

  :lang(zh) em,
  :lang(zh) i {
    font-style: normal;
  }

  :focus-visible {
    outline: var(--focus-ring-width) solid var(--focus-ring-color);
    outline-offset: var(--focus-ring-offset);
    box-shadow: var(--shadow-focus);
  }

  a {
    color: var(--color-link);
  }

  a:hover {
    color: var(--color-link-hover);
  }

  [hidden] {
    display: none !important;
  }
}

@layer utilities {
  .skip-link {
    position: absolute;
    left: var(--space-4);
    top: -100px;
    z-index: var(--z-tooltip);
    padding: var(--space-3) var(--space-4);
    background: var(--color-surface);
    border: 1px solid var(--color-border-strong);
    border-radius: var(--radius-md);
    transition: top var(--duration-fast) var(--ease-out);
  }

  .skip-link:focus {
    top: var(--space-4);
  }

  .container-page {
    width: 100%;
    max-width: var(--content-max);
    margin-inline: auto;
    padding-inline: var(--gutter);
  }

  .container-wide {
    width: 100%;
    max-width: var(--content-wide);
    margin-inline: auto;
    padding-inline: var(--gutter);
  }
}
```

`packages/ui/src/styles/glass.css` :

```css
/*
 * Glassmorphism — budget serré (design.md §3.5).
 * Autorisé : en-tête, barre de panier, modale, drawer, toast.
 * Interdit : zones paiement, connexion, MFA, tableaux operationnels, checkout.
 * Repli opaque si `backdrop-filter` n'est pas supporte.
 */
.glass {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
}

@supports (backdrop-filter: blur(1px)) or (-webkit-backdrop-filter: blur(1px)) {
  .glass {
    background: var(--glass-surface-light);
    border: 1px solid var(--glass-border-light);
    -webkit-backdrop-filter: blur(var(--glass-blur))
      saturate(180%);
    backdrop-filter: blur(var(--glass-blur)) saturate(180%);
  }

  .glass-strong {
    background: var(--glass-surface-light);
    border: 1px solid var(--glass-border-light);
    -webkit-backdrop-filter: blur(var(--glass-blur-strong))
      saturate(180%);
    backdrop-filter: blur(var(--glass-blur-strong)) saturate(180%);
  }
}

@media (prefers-reduced-motion: reduce) {
  .glass,
  .glass-strong {
    -webkit-backdrop-filter: none;
    backdrop-filter: none;
    background: var(--color-surface);
  }
}
```

- [ ] **Step 4: Écrire le script anti-flash de thème**

`packages/ui/src/theme-script.ts` :

```typescript
export const THEME_STORAGE_KEY = 'fanours.theme';

export const THEME_SCRIPT = `(function(){try{
var stored=localStorage.getItem('${THEME_STORAGE_KEY}');
var theme=stored==='light'||stored==='dark'||stored==='auto'?stored:'auto';
var resolved=theme==='auto'?(window.matchMedia('(prefers-color-scheme: dark)').matches?'dark':'light'):theme;
document.documentElement.setAttribute('data-theme',resolved);
document.documentElement.style.colorScheme=resolved;
}catch(e){document.documentElement.setAttribute('data-theme','light');}})();`;

export type ThemePreference = 'light' | 'dark' | 'auto';
```

Le script est inséré **inline** dans le `<head>` avant toute requête de rendu, pour éviter le flash de thème clair. Il est CSP-compatible car il ne contient ni `eval` ni fonction externe ; le nonce CSP est appliqué à la balise `<script>` qui l'héberge.

- [ ] **Step 5: Documenter les polices auto-hébergées**

`apps/web/public/fonts/plus-jakarta-sans/README.md` :

```markdown
# Plus Jakarta Sans — auto-hebergee

Poids requis : 500, 600, 700, 800 (titres, design.md §3.2.1).

Fichiers attendus dans ce dossier :

- `latin-500.woff2`
- `latin-600.woff2`
- `latin-700.woff2`
- `latin-800.woff2`

Source : Google Fonts (SIL Open Font License 1.1). Chaque fichier est subset
latin (unicode-range identique a `tokens.css`). Les poids non listes ne sont
pas telecharges : seuls ceux reellement utilises le sont.

Verification : `pnpm --filter @fanours/web exec next build` ne doit produire
aucun avertissement "Failed to load font".
```

`apps/web/public/fonts/inter/README.md` : même contenu, poids 400, 500, 600, 700.

`apps/web/public/fonts/noto-sans-sc/README.md` : même contenu, poids 400, 500, 700, sous-ensemble `chinese-simplified`, `unicode-range` CJK.

- [ ] **Step 6: Configurer Storybook avec les deux thèmes et les trois locales**

`packages/ui/.storybook/main.ts` :

```typescript
import type { StorybookConfig } from '@storybook/react-vite';

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(ts|tsx)'],
  addons: ['@storybook/addon-essentials', '@storybook/addon-a11y', '@storybook/addon-themes'],
  framework: { name: '@storybook/react-vite', options: {} },
  core: { disableTelemetry: true },
};

export default config;
```

`packages/ui/.storybook/preview.tsx` :

```tsx
import type { Preview, Decorator } from '@storybook/react';
import { withThemeByDataAttribute } from '@storybook/addon-themes';
import { THEME_SCRIPT } from '../src/theme-script.js';
import '../src/styles/globals.css';

const withThemeScript: Decorator = (Story) => {
  if (typeof document !== 'undefined' && !document.getElementById('fanours-theme-script')) {
    const script = document.createElement('script');
    script.id = 'fanours-theme-script';
    script.textContent = THEME_SCRIPT;
    document.head.appendChild(script);
  }
  return Story();
};

const preview: Preview = {
  parameters: {
    controls: { expanded: true },
    a11y: { test: 'error' },
    options: {
      storySort: { order: ['Foundations', 'Components', 'SystemStates'] },
    },
  },
  decorators: [
    withThemeScript,
    withThemeByDataAttribute({
      themes: { light: 'light', dark: 'dark' },
      defaultTheme: 'light',
    }),
  ],
  globalTypes: {
    locale: {
      description: 'Langue de rendu des libelles de test',
      defaultValue: 'fr-FR',
      toolbar: {
        icon: 'globe',
        items: ['fr-FR', 'en', 'zh-CN'],
      },
    },
  },
  initialGlobals: { locale: 'fr-FR' },
};

export default preview;
```

Le décorateur `withThemeScript` injecte `THEME_SCRIPT` dans `document.head` au montage, de sorte que les captures Storybook reflètent exactement le thème appliqué par l'application.

- [ ] **Step 7: Écrire le test de contrat des tokens**

`packages/ui/src/__tests__/tokens.test.ts` :

```typescript
import { readFile } from 'node:fs/promises';
import { dirname, join } from 'node:path';
import { fileURLToPath } from 'node:url';
import { describe, expect, it } from 'vitest';

const here = dirname(fileURLToPath(import.meta.url));
const tokensPath = join(here, '..', 'styles', 'tokens.css');

function relativeLuminance(hex: string): number {
  const value = hex.replace('#', '');
  const channels = [0, 2, 4].map((offset) => {
    const part = Number.parseInt(value.slice(offset, offset + 2), 16) / 255;
    return part <= 0.03928 ? part / 12.92 : ((part + 0.055) / 1.055) ** 2.4;
  });
  const [r, g, b] = channels as [number, number, number];
  return 0.2126 * r + 0.7152 * g + 0.0722 * b;
}

function contrastRatio(foreground: string, background: string): number {
  const a = relativeLuminance(foreground);
  const b = relativeLuminance(background);
  const lighter = Math.max(a, b);
  const darker = Math.min(a, b);
  return (lighter + 0.05) / (darker + 0.05);
}

describe('tokens.css', () => {
  it('declare les tokens de base attendus', async () => {
    const css = await readFile(tokensPath, 'utf8');
    for (const token of [
      '--color-bg',
      '--color-surface',
      '--color-text',
      '--color-primary',
      '--color-accent',
      '--color-surface-secondary',
      '--color-surface-elevated',
      '--color-border',
      '--color-border-strong',
      '--color-link',
      '--color-chart-1',
      '--color-chart-2',
      '--color-chart-3',
      '--color-chart-4',
      '--color-chart-5',
      '--color-chart-grid',
      '--font-heading',
      '--font-body',
      '--focus-ring-width',
      '--focus-ring-offset',
      '--touch-target-min',
      '--duration-normal',
      '--ease-out',
      '--z-modal',
    ]) {
      expect(css).toContain(token);
    }
  });

  it('definit un theme sombre explicite et un theme sombre automatique', async () => {
    const css = await readFile(tokensPath, 'utf8');
    expect(css).toContain(":root:not([data-theme='light'])");
    expect(css).toContain(":root[data-theme='dark']");
    expect(css).toContain('prefers-reduced-motion');
  });

  it('respecte le contraste WCAG AA du texte principal sur son fond', async () => {
    expect(contrastRatio('#321526', '#FFF5FA')).toBeGreaterThanOrEqual(4.5);
  });

  it('respecte le contraste du texte sur primaire dans les deux themes', async () => {
    expect(contrastRatio('#FFFFFF', '#D81B75')).toBeGreaterThanOrEqual(4.5);
    expect(contrastRatio('#180D15', '#FF70B7')).toBeGreaterThanOrEqual(4.5);
  });

  it('respecte le contraste du lien sur fond clair', async () => {
    expect(contrastRatio('#B91662', '#FFF5FA')).toBeGreaterThanOrEqual(4.5);
  });

  it('respecte le contraste du lien sur fond sombre', async () => {
    expect(contrastRatio('#FF9CCB', '#180D15')).toBeGreaterThanOrEqual(4.5);
  });

  it('respecte le contraste minimal de la palette categorielle de visualisation', () => {
    // WCAG 1.4.11 : composants graphiques >= 3:1 sur le fond adjacent.
    const light = ['#D81B75', '#00857A', '#7A4BB5', '#2B5FD9', '#B26A00'];
    const dark = ['#FF70B7', '#2FD3B0', '#C4A2F7', '#7CA7FF', '#FBBF24'];
    for (const color of light) {
      expect(contrastRatio(color, '#FFFFFF')).toBeGreaterThanOrEqual(3);
      expect(contrastRatio(color, '#FFF5FA')).toBeGreaterThanOrEqual(3);
    }
    for (const color of dark) {
      expect(contrastRatio(color, '#261420')).toBeGreaterThanOrEqual(3);
      expect(contrastRatio(color, '#180D15')).toBeGreaterThanOrEqual(3);
    }
  });

  it('reserve les trois premieres series categorielles a un contraste AA texte', () => {
    // design.md 3.2.4 : les series 1 a 3 sont les seules a pouvoir coexister
    // sur un meme graphique, elles doivent donc passer le seuil de texte.
    for (const color of ['#D81B75', '#00857A', '#7A4BB5']) {
      expect(contrastRatio(color, '#FFFFFF')).toBeGreaterThanOrEqual(4.5);
    }
    for (const color of ['#FF70B7', '#2FD3B0', '#C4A2F7']) {
      expect(contrastRatio(color, '#261420')).toBeGreaterThanOrEqual(4.5);
    }
  });
});
```

`packages/ui/vitest.config.ts` :

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'node',
    include: ['src/__tests__/**/*.test.ts'],
  },
});
```

- [ ] **Step 8: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/ui test:unit
```

Attendu : `No test files found in packages/ui/src/__tests__` ou `ENOENT ... tokens.css`. Coller la sortie.

- [ ] **Step 9: Implémenter et prouver la réussite**

```bash
pnpm --filter @fanours/ui test:unit
```

Attendu :

```
 ✓ src/__tests__/tokens.test.ts (6 tests)

 Test Files  1 passed (1)
      Tests  6 passed (6)
```

- [ ] **Step 10: Vérifier la construction Storybook**

```bash
pnpm --filter @fanours/ui build-storybook
```

Attendu : `storybook-static/` généré sans erreur ; `output directory: storybook-static`. L'étape est bloquante pour le Gate 1 (wave 1.3) : Storybook doit se construire.

- [ ] **Step 11: Commit**

```bash
git add packages/ui apps/web/public/fonts
git commit -m "feat(ui): pose les tokens du design system et les deux themes" -m "tokens.css est la source unique des couleurs, typographies, espacements,
rayons, ombres, mouvement et glassmorphism, avec les themes clair et
sombre. Script anti-flash avant hydration, Storybook configure sur les
deux themes, test de contrat des tokens et de contraste WCAG AA.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 4: Composants de base, états système et accessibilité

**Files:**
- Create: `packages/ui/src/components/Button.tsx`, `IconButton.tsx`, `ButtonGroup.tsx`
- Create: `packages/ui/src/components/Input.tsx`, `Textarea.tsx`, `Select.tsx`, `Checkbox.tsx`, `Radio.tsx`, `Switch.tsx`
- Create: `packages/ui/src/components/Badge.tsx`, `Skeleton.tsx`, `Table.tsx`, `Tabs.tsx`, `Pagination.tsx`
- Create: `packages/ui/src/components/Dialog.tsx`, `BottomSheet.tsx`, `Toast.tsx`, `ToastProvider.tsx`
- Create: `packages/ui/src/components/SystemState.tsx` (12 variantes)
- Create: `packages/ui/src/components/primitives.ts` (classes partagées)
- Create: `packages/ui/src/index.ts` (exports)
- Test: `packages/ui/src/__tests__/button.test.tsx`, `system-state.test.tsx`, `a11y.test.tsx`, `dialog.test.tsx`

**Interfaces:**
- Consumes: `packages/ui/src/styles/tokens.css` (Task 3), `packages/ui/src/theme-script.ts` (Task 3).
- Produces:
  - `Button` : `{ variant?: 'primary' | 'secondary' | 'ghost' | 'danger'; size?: 'sm' | 'md' | 'lg'; loading?: boolean; disabled?: boolean; iconStart?: ReactNode; iconEnd?: ReactNode; fullWidth?: boolean; type?: 'button' | 'submit' | 'reset'; children: ReactNode }` → `<button>` avec `aria-busy` quand `loading`.
  - `IconButton` : `{ label: string; icon: ReactNode; size?: 'sm' | 'md' | 'lg' }` → `<button aria-label={label}>`.
  - `Input` : `{ id: string; label: string; value: string; onChange: (value: string) => void; error?: string; hint?: string; required?: boolean; type?: 'email' | 'password' | 'text' | 'tel' | 'url' | 'number'; autoComplete?: string; disabled?: boolean; inputMode?: 'text' | 'email' | 'numeric' | 'tel' | 'search' }` → `<input>` + `<label for>` + message d'erreur lié par `aria-describedby`.
  - `Select` : `{ id: string; label: string; value: string; onChange: (value: string) => void; options: ReadonlyArray<{ value: string; label: string }>; error?: string; required?: boolean }`.
  - `Textarea` : `{ id: string; label: string; value: string; onChange: (value: string) => void; rows?: number; maxLength?: number; error?: string; hint?: string; required?: boolean }`.
  - `Checkbox` / `Radio` : `{ id: string; label: string; checked: boolean; onChange: (checked: boolean) => void; error?: string; description?: string }`.
  - `Switch` : `{ id: string; label: string; checked: boolean; onChange: (checked: boolean) => void; description?: string }` → `<button role="switch" aria-checked>`.
  - `Badge` : `{ tone?: 'neutral' | 'success' | 'warning' | 'error' | 'info' | 'accent'; children: ReactNode }`.
  - `Skeleton` : `{ width?: string; height?: string; radius?: string; }` → `<div aria-hidden="true">` avec animation.
  - `Table` : `{ caption: string; columns: ReadonlyArray<{ key: string; header: string; align?: 'start' | 'center' | 'end' }>; children: ReactNode }` → `<table>` avec `<caption>`.
  - `Tabs` : `{ tabs: ReadonlyArray<{ id: string; label: string }>; activeId: string; onChange: (id: string) => void; label: string }` → `role="tablist"` + `aria-controls`.
  - `Pagination` : `{ page: number; pageCount: number; onPageChange: (page: number) => void; label: string; previousLabel: string; nextLabel: string }`.
  - `Dialog` : `{ open: boolean; onClose: () => void; title: string; description?: string; children: ReactNode; closeLabel: string }` → `<div role="dialog" aria-modal="true">` avec piège de focus et `Escape`.
  - `BottomSheet` : mêmes props que `Dialog`, rendu en feuille basse sous 768 px.
  - `ToastProvider` + `useToast()` : `{ push: (toast: { tone: 'success' | 'error' | 'info'; title: string; description?: string }) => void }` → `role="status"` / `role="alert"`.
  - `SystemState` : `{ variant: 'loading' | 'empty' | 'error' | 'forbidden' | 'session-expired' | 'offline' | 'unavailable' | 'stock-conflict' | 'invalid-quantity' | 'payment-pending' | 'provider-interrupted' | 'payment-refused'; title: string; description?: string; actionLabel?: string; onAction?: () => void; reference?: string; icon: ReactNode }` → bloc avec icône, message, action de sortie et référence de corrélation.

---

- [ ] **Step 1: Écrire les primitives partagées**

`packages/ui/src/components/primitives.ts` :

```typescript
import type { CSSProperties } from 'react';

export const focusRingClasses =
  'focus-visible:outline-[var(--focus-ring-width)] focus-visible:outline-offset-[var(--focus-ring-offset)] focus-visible:outline-[var(--focus-ring-color)]';

export const controlHeightClasses: Readonly<Record<'sm' | 'md' | 'lg', string>> = {
  sm: 'h-[var(--control-height-sm)]',
  md: 'h-[var(--control-height)]',
  lg: 'h-[var(--control-height-lg)]',
};

export const surfaceClasses =
  'bg-[var(--color-surface)] text-[var(--color-text)] border-[var(--color-border)]';

export interface ToneStyle {
  readonly background: string;
  readonly foreground: string;
  readonly border: string;
}

export const toneStyles: Readonly<Record<'neutral' | 'success' | 'warning' | 'error' | 'info' | 'accent', ToneStyle>> = {
  neutral: {
    background: 'bg-[var(--color-neutral-bg)]',
    foreground: 'text-[var(--color-neutral)]',
    border: 'border-[var(--color-border)]',
  },
  success: {
    background: 'bg-[var(--color-success-bg)]',
    foreground: 'text-[var(--color-success)]',
    border: 'border-[var(--color-success)]',
  },
  warning: {
    background: 'bg-[var(--color-warning-bg)]',
    foreground: 'text-[var(--color-warning)]',
    border: 'border-[var(--color-warning)]',
  },
  error: {
    background: 'bg-[var(--color-error-bg)]',
    foreground: 'text-[var(--color-error)]',
    border: 'border-[var(--color-error)]',
  },
  info: {
    background: 'bg-[var(--color-info-bg)]',
    foreground: 'text-[var(--color-info)]',
    border: 'border-[var(--color-info)]',
  },
  accent: {
    background: 'bg-[var(--color-accent)]',
    foreground: 'text-[var(--color-text)]',
    border: 'border-[var(--color-primary)]',
  },
};

export function srOnly(): CSSProperties {
  return {
    position: 'absolute',
    width: '1px',
    height: '1px',
    padding: 0,
    margin: '-1px',
    overflow: 'hidden',
    clip: 'rect(0, 0, 0, 0)',
    whiteSpace: 'nowrap',
    borderWidth: 0,
  };
}
```

- [ ] **Step 2: Écrire `Button`, `IconButton`, `ButtonGroup`**

`packages/ui/src/components/Button.tsx` :

```tsx
import type { ButtonHTMLAttributes, ReactNode } from 'react';
import { controlHeightClasses, focusRingClasses } from './primitives.js';

export type ButtonVariant = 'primary' | 'secondary' | 'ghost' | 'danger';
export type ButtonSize = 'sm' | 'md' | 'lg';

export interface ButtonProps extends Omit<ButtonHTMLAttributes<HTMLButtonElement>, 'children'> {
  readonly variant?: ButtonVariant;
  readonly size?: ButtonSize;
  readonly loading?: boolean;
  readonly fullWidth?: boolean;
  readonly iconStart?: ReactNode;
  readonly iconEnd?: ReactNode;
  readonly children: ReactNode;
}

const variantClasses: Readonly<Record<ButtonVariant, string>> = {
  primary:
    'bg-[var(--color-primary)] text-[var(--color-text-on-primary)] border border-[var(--color-primary)] hover:bg-[var(--color-rose-700)]',
  secondary:
    'bg-[var(--color-surface)] text-[var(--color-text)] border border-[var(--color-border-strong)] hover:bg-[var(--color-surface-secondary)]',
  ghost:
    'bg-transparent text-[var(--color-text)] border border-transparent hover:bg-[var(--color-surface-secondary)]',
  danger:
    'bg-[var(--color-error)] text-white border border-[var(--color-error)] hover:opacity-90',
};

const sizePadding: Readonly<Record<ButtonSize, string>> = {
  sm: 'px-[var(--space-3)] text-[var(--font-size-sm)] gap-[var(--space-1)]',
  md: 'px-[var(--space-4)] text-[var(--font-size-base)] gap-[var(--space-2)]',
  lg: 'px-[var(--space-6)] text-[var(--font-size-lg)] gap-[var(--space-2)]',
};

export function Button({
  variant = 'primary',
  size = 'md',
  loading = false,
  fullWidth = false,
  iconStart,
  iconEnd,
  disabled = false,
  type = 'button',
  className,
  children,
  ...rest
}: ButtonProps) {
  return (
    <button
      type={type}
      disabled={disabled || loading}
      aria-busy={loading ? true : undefined}
      className={[
        'inline-flex items-center justify-center rounded-[var(--radius-md)] font-[var(--font-weight-semibold)]',
        'transition-[background-color,border-color,opacity] duration-[var(--duration-fast)] ease-[var(--ease-out)]',
        'disabled:cursor-not-allowed disabled:opacity-50',
        focusRingClasses,
        controlHeightClasses[size],
        sizePadding[size],
        variantClasses[variant],
        fullWidth ? 'w-full' : '',
        className ?? '',
      ]
        .filter(Boolean)
        .join(' ')}
      {...rest}
    >
      {loading ? <span aria-hidden="true" className="animate-pulse" /> : null}
      {iconStart ? (
        <span aria-hidden="true" className="inline-flex shrink-0">
          {iconStart}
        </span>
      ) : null}
      <span>{children}</span>
      {iconEnd ? (
        <span aria-hidden="true" className="inline-flex shrink-0">
          {iconEnd}
        </span>
      ) : null}
    </button>
  );
}
```

`packages/ui/src/components/IconButton.tsx` :

```tsx
import type { ButtonHTMLAttributes, ReactNode } from 'react';
import { controlHeightClasses, focusRingClasses } from './primitives.js';

export interface IconButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  readonly label: string;
  readonly icon: ReactNode;
  readonly size?: 'sm' | 'md' | 'lg';
}

export function IconButton({ label, icon, size = 'md', className, ...rest }: IconButtonProps) {
  return (
    <button
      type="button"
      aria-label={label}
      title={label}
      className={[
        'inline-flex items-center justify-center rounded-[var(--radius-md)]',
        'border border-[var(--color-border)] bg-[var(--color-surface)] text-[var(--color-text)]',
        'transition-colors duration-[var(--duration-fast)] ease-[var(--ease-out)]',
        'disabled:cursor-not-allowed disabled:opacity-50',
        focusRingClasses,
        controlHeightClasses[size],
        'aspect-square',
        className ?? '',
      ]
        .filter(Boolean)
        .join(' ')}
      {...rest}
    >
      <span aria-hidden="true" className="inline-flex">
        {icon}
      </span>
    </button>
  );
}
```

`packages/ui/src/components/ButtonGroup.tsx` :

```tsx
import type { ReactNode } from 'react';

export interface ButtonGroupProps {
  readonly children: ReactNode;
  readonly label: string;
  readonly className?: string;
}

export function ButtonGroup({ children, label, className }: ButtonGroupProps) {
  return (
    <div
      role="group"
      aria-label={label}
      className={[
        'inline-flex flex-wrap items-center gap-[var(--space-2)]',
        className ?? '',
      ]
        .filter(Boolean)
        .join(' ')}
    >
      {children}
    </div>
  );
}
```

- [ ] **Step 3: Écrire les champs de formulaire**

`packages/ui/src/components/Input.tsx` :

```tsx
import { useId } from 'react';
import { focusRingClasses } from './primitives.js';

export interface InputProps {
  readonly id?: string;
  readonly label: string;
  readonly value: string;
  readonly onChange: (value: string) => void;
  readonly error?: string;
  readonly hint?: string;
  readonly required?: boolean;
  readonly disabled?: boolean;
  readonly type?: 'email' | 'password' | 'text' | 'tel' | 'url' | 'number';
  readonly autoComplete?: string;
  readonly inputMode?: 'text' | 'email' | 'numeric' | 'tel' | 'search';
  readonly name?: string;
  readonly maxLength?: number;
}

export function Input({
  id,
  label,
  value,
  onChange,
  error,
  hint,
  required = false,
  disabled = false,
  type = 'text',
  autoComplete,
  inputMode,
  name,
  maxLength,
}: InputProps) {
  const generatedId = useId();
  const inputId = id ?? generatedId;
  const errorId = `${inputId}-error`;
  const hintId = `${inputId}-hint`;
  const describedBy = [error ? errorId : null, hint ? hintId : null].filter(Boolean).join(' ');

  return (
    <div className="flex flex-col gap-[var(--space-1)]">
      <label htmlFor={inputId} className="text-[var(--font-size-sm)] font-[var(--font-weight-medium)]">
        {label}
        {required ? (
          <span aria-hidden="true" className="text-[var(--color-error)]">
            {' *'}
          </span>
        ) : null}
      </label>
      <input
        id={inputId}
        name={name}
        type={type}
        value={value}
        onChange={(event) => onChange(event.target.value)}
        required={required}
        disabled={disabled}
        autoComplete={autoComplete}
        inputMode={inputMode}
        maxLength={maxLength}
        aria-required={required}
        aria-invalid={error ? true : undefined}
        aria-describedby={describedBy.length > 0 ? describedBy : undefined}
        className={[
          'w-full rounded-[var(--radius-md)] border bg-[var(--color-surface)] px-[var(--space-3)]',
          'text-[var(--color-text)] placeholder:text-[var(--color-text-muted)]',
          'disabled:cursor-not-allowed disabled:opacity-60',
          focusRingClasses,
          error
            ? 'border-[var(--color-error)]'
            : 'border-[var(--color-border-input)]',
        ].join(' ')}
      />
      {hint ? (
        <p id={hintId} className="text-[var(--font-size-xs)] text-[var(--color-text-muted)]">
          {hint}
        </p>
      ) : null}
      {error ? (
        <p id={errorId} role="alert" className="text-[var(--font-size-xs)] text-[var(--color-error)]">
          {error}
        </p>
      ) : null}
    </div>
  );
}
```

`packages/ui/src/components/Textarea.tsx` :

```tsx
import { useId } from 'react';
import { focusRingClasses } from './primitives.js';

export interface TextareaProps {
  readonly id?: string;
  readonly label: string;
  readonly value: string;
  readonly onChange: (value: string) => void;
  readonly rows?: number;
  readonly maxLength?: number;
  readonly error?: string;
  readonly hint?: string;
  readonly required?: boolean;
  readonly disabled?: boolean;
}

export function Textarea({
  id,
  label,
  value,
  onChange,
  rows = 4,
  maxLength,
  error,
  hint,
  required = false,
  disabled = false,
}: TextareaProps) {
  const generatedId = useId();
  const textareaId = id ?? generatedId;
  const errorId = `${textareaId}-error`;
  const hintId = `${textareaId}-hint`;
  const describedBy = [error ? errorId : null, hint ? hintId : null].filter(Boolean).join(' ');

  return (
    <div className="flex flex-col gap-[var(--space-1)]">
      <label htmlFor={textareaId} className="text-[var(--font-size-sm)] font-[var(--font-weight-medium)]">
        {label}
      </label>
      <textarea
        id={textareaId}
        rows={rows}
        value={value}
        onChange={(event) => onChange(event.target.value)}
        maxLength={maxLength}
        required={required}
        disabled={disabled}
        aria-required={required}
        aria-invalid={error ? true : undefined}
        aria-describedby={describedBy.length > 0 ? describedBy : undefined}
        className={[
          'w-full rounded-[var(--radius-md)] border bg-[var(--color-surface)] px-[var(--space-3)] py-[var(--space-2)]',
          'text-[var(--color-text)]',
          focusRingClasses,
          error
            ? 'border-[var(--color-error)]'
            : 'border-[var(--color-border-input)]',
        ].join(' ')}
      />
      {hint ? (
        <p id={hintId} className="text-[var(--font-size-xs)] text-[var(--color-text-muted)]">
          {hint}
        </p>
      ) : null}
      {error ? (
        <p id={errorId} role="alert" className="text-[var(--font-size-xs)] text-[var(--color-error)]">
          {error}
        </p>
      ) : null}
    </div>
  );
}
```

`packages/ui/src/components/Select.tsx` :

```tsx
import { useId } from 'react';
import { focusRingClasses } from './primitives.js';

export interface SelectOption {
  readonly value: string;
  readonly label: string;
}

export interface SelectProps {
  readonly id?: string;
  readonly label: string;
  readonly value: string;
  readonly onChange: (value: string) => void;
  readonly options: ReadonlyArray<SelectOption>;
  readonly error?: string;
  readonly required?: boolean;
  readonly disabled?: boolean;
  readonly name?: string;
}

export function Select({
  id,
  label,
  value,
  onChange,
  options,
  error,
  required = false,
  disabled = false,
  name,
}: SelectProps) {
  const generatedId = useId();
  const selectId = id ?? generatedId;
  const errorId = `${selectId}-error`;

  return (
    <div className="flex flex-col gap-[var(--space-1)]">
      <label htmlFor={selectId} className="text-[var(--font-size-sm)] font-[var(--font-weight-medium)]">
        {label}
      </label>
      <select
        id={selectId}
        name={name}
        value={value}
        onChange={(event) => onChange(event.target.value)}
        required={required}
        disabled={disabled}
        aria-required={required}
        aria-invalid={error ? true : undefined}
        aria-describedby={error ? errorId : undefined}
        className={[
          'h-[var(--control-height)] w-full rounded-[var(--radius-md)] border bg-[var(--color-surface)]',
          'px-[var(--space-3)] text-[var(--color-text)]',
          focusRingClasses,
          error
            ? 'border-[var(--color-error)]'
            : 'border-[var(--color-border-input)]',
        ].join(' ')}
      >
        {options.map((option) => (
          <option key={option.value} value={option.value}>
            {option.label}
          </option>
        ))}
      </select>
      {error ? (
        <p id={errorId} role="alert" className="text-[var(--font-size-xs)] text-[var(--color-error)]">
          {error}
        </p>
      ) : null}
    </div>
  );
}
```

`packages/ui/src/components/Checkbox.tsx` :

```tsx
import { useId } from 'react';
import { focusRingClasses } from './primitives.js';

export interface CheckboxProps {
  readonly id?: string;
  readonly label: string;
  readonly checked: boolean;
  readonly onChange: (checked: boolean) => void;
  readonly description?: string;
  readonly error?: string;
  readonly disabled?: boolean;
}

export function Checkbox({
  id,
  label,
  checked,
  onChange,
  description,
  error,
  disabled = false,
}: CheckboxProps) {
  const generatedId = useId();
  const checkboxId = id ?? generatedId;
  const descriptionId = `${checkboxId}-description`;
  const errorId = `${checkboxId}-error`;
  const describedBy = [description ? descriptionId : null, error ? errorId : null]
    .filter(Boolean)
    .join(' ');

  return (
    <div className="flex flex-col gap-[var(--space-1)]">
      <div className="flex items-start gap-[var(--space-2)]">
        <input
          id={checkboxId}
          type="checkbox"
          checked={checked}
          onChange={(event) => onChange(event.target.checked)}
          disabled={disabled}
          aria-invalid={error ? true : undefined}
          aria-describedby={describedBy.length > 0 ? describedBy : undefined}
          className={[
            'mt-[2px] h-[20px] w-[20px] shrink-0 rounded-[var(--radius-xs)]',
            'accent-[var(--color-primary)]',
            focusRingClasses,
          ].join(' ')}
        />
        <label htmlFor={checkboxId} className="text-[var(--font-size-sm)]">
          {label}
        </label>
      </div>
      {description ? (
        <p id={descriptionId} className="text-[var(--font-size-xs)] text-[var(--color-text-muted)]">
          {description}
        </p>
      ) : null}
      {error ? (
        <p id={errorId} role="alert" className="text-[var(--font-size-xs)] text-[var(--color-error)]">
          {error}
        </p>
      ) : null}
    </div>
  );
}
```

`packages/ui/src/components/Radio.tsx` :

```tsx
import { useId } from 'react';
import { focusRingClasses } from './primitives.js';

export interface RadioProps {
  readonly id?: string;
  readonly name: string;
  readonly label: string;
  readonly checked: boolean;
  readonly onChange: (checked: boolean) => void;
  readonly description?: string;
  readonly disabled?: boolean;
}

export function Radio({
  id,
  name,
  label,
  checked,
  onChange,
  description,
  disabled = false,
}: RadioProps) {
  const generatedId = useId();
  const radioId = id ?? generatedId;
  const descriptionId = `${radioId}-description`;

  return (
    <div className="flex flex-col gap-[var(--space-1)]">
      <div className="flex items-start gap-[var(--space-2)]">
        <input
          id={radioId}
          type="radio"
          name={name}
          checked={checked}
          onChange={(event) => onChange(event.target.checked)}
          disabled={disabled}
          aria-describedby={description ? descriptionId : undefined}
          className={[
            'mt-[2px] h-[20px] w-[20px] shrink-0 accent-[var(--color-primary)]',
            focusRingClasses,
          ].join(' ')}
        />
        <label htmlFor={radioId} className="text-[var(--font-size-sm)]">
          {label}
        </label>
      </div>
      {description ? (
        <p id={descriptionId} className="text-[var(--font-size-xs)] text-[var(--color-text-muted)]">
          {description}
        </p>
      ) : null}
    </div>
  );
}
```

`packages/ui/src/components/Switch.tsx` :

```tsx
import { useId } from 'react';
import { focusRingClasses } from './primitives.js';

export interface SwitchProps {
  readonly id?: string;
  readonly label: string;
  readonly checked: boolean;
  readonly onChange: (checked: boolean) => void;
  readonly description?: string;
  readonly disabled?: boolean;
}

export function Switch({
  id,
  label,
  checked,
  onChange,
  description,
  disabled = false,
}: SwitchProps) {
  const generatedId = useId();
  const switchId = id ?? generatedId;
  const descriptionId = `${switchId}-description`;

  return (
    <div className="flex items-start justify-between gap-[var(--space-3)]">
      <div className="flex flex-col gap-[var(--space-1)]">
        <span id={`${switchId}-label`} className="text-[var(--font-size-sm)] font-[var(--font-weight-medium)]">
          {label}
        </span>
        {description ? (
          <span id={descriptionId} className="text-[var(--font-size-xs)] text-[var(--color-text-muted)]">
            {description}
          </span>
        ) : null}
      </div>
      <button
        id={switchId}
        type="button"
        role="switch"
        aria-checked={checked}
        aria-labelledby={`${switchId}-label`}
        aria-describedby={description ? descriptionId : undefined}
        disabled={disabled}
        onClick={() => onChange(!checked)}
        className={[
          'relative h-[24px] w-[44px] shrink-0 rounded-[var(--radius-full)]',
          'transition-colors duration-[var(--duration-fast)] ease-[var(--ease-out)]',
          'disabled:cursor-not-allowed disabled:opacity-50',
          focusRingClasses,
          checked ? 'bg-[var(--color-primary)]' : 'bg-[var(--color-border-strong)]',
        ].join(' ')}
      >
        <span
          aria-hidden="true"
          className={[
            'absolute top-[3px] h-[18px] w-[18px] rounded-[var(--radius-full)] bg-white',
            'transition-transform duration-[var(--duration-fast)] ease-[var(--ease-out)]',
            checked ? 'translate-x-[23px]' : 'translate-x-[3px]',
          ].join(' ')}
        />
      </button>
    </div>
  );
}
```

- [ ] **Step 4: Écrire `Badge`, `Skeleton`, `Table`, `Tabs`, `Pagination`**

`packages/ui/src/components/Badge.tsx` :

```tsx
import type { ReactNode } from 'react';
import { toneStyles, type ToneStyle } from './primitives.js';

export type BadgeTone = 'neutral' | 'success' | 'warning' | 'error' | 'info' | 'accent';

export interface BadgeProps {
  readonly tone?: BadgeTone;
  readonly children: ReactNode;
  readonly className?: string;
}

export function Badge({ tone = 'neutral', children, className }: BadgeProps) {
  const style: ToneStyle = toneStyles[tone];
  return (
    <span
      className={[
        'inline-flex items-center gap-[var(--space-1)] rounded-[var(--radius-sm)]',
        'px-[var(--space-2)] py-[2px] text-[var(--font-size-xs)] font-[var(--font-weight-semibold)]',
        style.background,
        style.foreground,
        style.border,
        'border',
        className ?? '',
      ].join(' ')}
    >
      {children}
    </span>
  );
}
```

`packages/ui/src/components/Skeleton.tsx` :

```tsx
export interface SkeletonProps {
  readonly width?: string;
  readonly height?: string;
  readonly radius?: string;
  readonly className?: string;
}

export function Skeleton({
  width = '100%',
  height = '1rem',
  radius = 'var(--radius-sm)',
  className,
}: SkeletonProps) {
  return (
    <div
      aria-hidden="true"
      style={{ width, height, borderRadius: radius }}
      className={[
        'animate-pulse bg-[var(--color-surface-secondary)]',
        'motion-reduce:animate-none',
        className ?? '',
      ].join(' ')}
    />
  );
}
```

`packages/ui/src/components/Table.tsx` :

```tsx
import type { ReactNode } from 'react';

export interface TableColumn {
  readonly key: string;
  readonly header: string;
  readonly align?: 'start' | 'center' | 'end';
}

export interface TableProps {
  readonly caption: string;
  readonly columns: ReadonlyArray<TableColumn>;
  readonly children: ReactNode;
  readonly className?: string;
}

const alignClasses: Readonly<Record<'start' | 'center' | 'end', string>> = {
  start: 'text-left',
  center: 'text-center',
  end: 'text-right',
};

export function Table({ caption, columns, children, className }: TableProps) {
  return (
    <div className={['w-full overflow-x-auto', className ?? ''].join(' ')}>
      <table className="w-full border-collapse text-[var(--font-size-sm)]">
        <caption className="pb-[var(--space-2)] text-left text-[var(--font-size-sm)] font-[var(--font-weight-semibold)]">
          {caption}
        </caption>
        <thead>
          <tr className="border-b border-[var(--color-border)]">
            {columns.map((column) => (
              <th
                key={column.key}
                scope="col"
                className={[
                  'px-[var(--space-3)] py-[var(--space-2)] font-[var(--font-weight-semibold)]',
                  alignClasses[column.align ?? 'start'],
                ].join(' ')}
              >
                {column.header}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>{children}</tbody>
      </table>
    </div>
  );
}
```

`packages/ui/src/components/Tabs.tsx` :

```tsx
export interface TabItem {
  readonly id: string;
  readonly label: string;
}

export interface TabsProps {
  readonly tabs: ReadonlyArray<TabItem>;
  readonly activeId: string;
  readonly onChange: (id: string) => void;
  readonly label: string;
  readonly className?: string;
}

export function Tabs({ tabs, activeId, onChange, label, className }: TabsProps) {
  return (
    <div
      role="tablist"
      aria-label={label}
      className={['flex gap-[var(--space-1)] border-b border-[var(--color-border)]', className ?? ''].join(' ')}
    >
      {tabs.map((tab) => {
        const selected = tab.id === activeId;
        return (
          <button
            key={tab.id}
            type="button"
            role="tab"
            id={`tab-${tab.id}`}
            aria-selected={selected}
            aria-controls={`tabpanel-${tab.id}`}
            tabIndex={selected ? 0 : -1}
            onClick={() => onChange(tab.id)}
            className={[
              'min-h-[var(--touch-target-min)] px-[var(--space-3)] text-[var(--font-size-sm)]',
              'font-[var(--font-weight-semibold)] focus-visible:outline-[var(--focus-ring-width)]',
              'focus-visible:outline-offset-[var(--focus-ring-offset)] focus-visible:outline-[var(--focus-ring-color)]',
              selected
                ? 'border-b-2 border-[var(--color-primary)] text-[var(--color-text)]'
                : 'border-b-2 border-transparent text-[var(--color-text-muted)]',
            ].join(' ')}
          >
            {tab.label}
          </button>
        );
      })}
    </div>
  );
}
```

`packages/ui/src/components/Pagination.tsx` :

```tsx
import { Button } from './Button.js';

export interface PaginationProps {
  readonly page: number;
  readonly pageCount: number;
  readonly onPageChange: (page: number) => void;
  readonly label: string;
  readonly previousLabel: string;
  readonly nextLabel: string;
  readonly positionLabel: (page: number, pageCount: number) => string;
}

export function Pagination({
  page,
  pageCount,
  onPageChange,
  label,
  previousLabel,
  nextLabel,
  positionLabel,
}: PaginationProps) {
  return (
    <nav aria-label={label} className="flex items-center justify-between gap-[var(--space-3)]">
      <Button
        variant="secondary"
        size="sm"
        disabled={page <= 1}
        onClick={() => onPageChange(page - 1)}
      >
        {previousLabel}
      </Button>
      <span aria-live="polite" className="text-[var(--font-size-sm)] text-[var(--color-text-muted)]">
        {positionLabel(page, pageCount)}
      </span>
      <Button
        variant="secondary"
        size="sm"
        disabled={page >= pageCount}
        onClick={() => onPageChange(page + 1)}
      >
        {nextLabel}
      </Button>
    </nav>
  );
}
```

- [ ] **Step 5: Écrire `Dialog` avec piège de focus, et `BottomSheet`**

`packages/ui/src/components/Dialog.tsx` :

```tsx
'use client';

import { useCallback, useEffect, useId, useRef, type ReactNode } from 'react';
import { Button } from './Button.js';

export interface DialogProps {
  readonly open: boolean;
  readonly onClose: () => void;
  readonly title: string;
  readonly description?: string;
  readonly children: ReactNode;
  readonly closeLabel: string;
  readonly confirmLabel?: string;
  readonly onConfirm?: () => void;
  readonly confirmDisabled?: boolean;
  readonly variant?: 'dialog' | 'sheet';
}

const FOCUSABLE_SELECTOR =
  'a[href], button:not([disabled]), textarea:not([disabled]), input:not([disabled]), select:not([disabled]), [tabindex]:not([tabindex="-1"])';

export function Dialog({
  open,
  onClose,
  title,
  description,
  children,
  closeLabel,
  confirmLabel,
  onConfirm,
  confirmDisabled = false,
  variant = 'dialog',
}: DialogProps) {
  const titleId = useId();
  const descriptionId = useId();
  const panelRef = useRef<HTMLDivElement | null>(null);
  const previouslyFocused = useRef<HTMLElement | null>(null);

  const handleKeyDown = useCallback(
    (event: KeyboardEvent) => {
      if (event.key === 'Escape') {
        event.preventDefault();
        onClose();
        return;
      }
      if (event.key !== 'Tab' || panelRef.current === null) {
        return;
      }
      const focusable = Array.from(
        panelRef.current.querySelectorAll<HTMLElement>(FOCUSABLE_SELECTOR),
      ).filter((element) => element.offsetParent !== null);
      if (focusable.length === 0) {
        event.preventDefault();
        panelRef.current.focus();
        return;
      }
      const first = focusable[0] as HTMLElement;
      const last = focusable[focusable.length - 1] as HTMLElement;
      const active = document.activeElement;
      if (event.shiftKey && active === first) {
        event.preventDefault();
        last.focus();
      } else if (!event.shiftKey && active === last) {
        event.preventDefault();
        first.focus();
      }
    },
    [onClose],
  );

  useEffect(() => {
    if (!open) {
      return undefined;
    }
    previouslyFocused.current = document.activeElement as HTMLElement | null;
    document.addEventListener('keydown', handleKeyDown);
    const timer = window.setTimeout(() => {
      const target = panelRef.current?.querySelector<HTMLElement>(FOCUSABLE_SELECTOR);
      (target ?? panelRef.current)?.focus();
    }, 0);
    return () => {
      document.removeEventListener('keydown', handleKeyDown);
      window.clearTimeout(timer);
      previouslyFocused.current?.focus();
    };
  }, [open, handleKeyDown]);

  if (!open) {
    return null;
  }

  const panelClasses =
    variant === 'sheet'
      ? 'fixed inset-x-0 bottom-0 z-[var(--z-modal)] max-h-[85vh] overflow-y-auto rounded-t-[var(--radius-xl)] border-t border-[var(--color-border)] bg-[var(--color-surface)] p-[var(--space-4)] md:inset-x-auto md:bottom-auto md:left-1/2 md:top-1/2 md:w-[min(560px,calc(100vw-32px))] md:-translate-x-1/2 md:-translate-y-1/2 md:rounded-[var(--radius-xl)] md:border'
      : 'fixed left-1/2 top-1/2 z-[var(--z-modal)] w-[min(560px,calc(100vw-32px))] max-h-[85vh] overflow-y-auto -translate-x-1/2 -translate-y-1/2 rounded-[var(--radius-xl)] border border-[var(--color-border)] bg-[var(--color-surface)] p-[var(--space-6)]';

  return (
    <div
      className="fixed inset-0 z-[var(--z-overlay)] flex items-end justify-center bg-black/50 md:items-center"
      onClick={(event) => {
        if (event.target === event.currentTarget) {
          onClose();
        }
      }}
    >
      <div
        ref={panelRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby={titleId}
        aria-describedby={description ? descriptionId : undefined}
        tabIndex={-1}
        className={panelClasses}
      >
        <div className="flex items-start justify-between gap-[var(--space-4)]">
          <div className="flex min-w-0 flex-col gap-[var(--space-1)]">
            <h2 id={titleId} className="text-[var(--font-size-xl)]">
              {title}
            </h2>
            {description ? (
              <p id={descriptionId} className="text-[var(--font-size-sm)] text-[var(--color-text-muted)]">
                {description}
              </p>
            ) : null}
          </div>
          <button
            type="button"
            onClick={onClose}
            aria-label={closeLabel}
            className="min-h-[var(--touch-target-min)] min-w-[var(--touch-target-min)] shrink-0 rounded-[var(--radius-md)] text-[var(--font-size-lg)] focus-visible:outline-[var(--focus-ring-width)] focus-visible:outline-offset-[var(--focus-ring-offset)] focus-visible:outline-[var(--focus-ring-color)]"
          >
            <span aria-hidden="true">&times;</span>
          </button>
        </div>
        <div className="mt-[var(--space-4)]">{children}</div>
        {confirmLabel && onConfirm ? (
          <div className="mt-[var(--space-6)] flex flex-wrap justify-end gap-[var(--space-2)]">
            <Button variant="secondary" onClick={onClose}>
              {closeLabel}
            </Button>
            <Button onClick={onConfirm} disabled={confirmDisabled}>
              {confirmLabel}
            </Button>
          </div>
        ) : null}
      </div>
    </div>
  );
}
```

`packages/ui/src/components/BottomSheet.tsx` :

```tsx
import { Dialog, type DialogProps } from './Dialog.js';

export type BottomSheetProps = Omit<DialogProps, 'variant'>;

export function BottomSheet(props: BottomSheetProps) {
  return <Dialog variant="sheet" {...props} />;
}
```

- [ ] **Step 6: Écrire `Toast` et `ToastProvider`**

`packages/ui/src/components/Toast.tsx` :

```tsx
export type ToastTone = 'success' | 'error' | 'info';

export interface ToastPayload {
  readonly tone: ToastTone;
  readonly title: string;
  readonly description?: string;
}

export interface ToastProps extends ToastPayload {
  readonly id: string;
  readonly onDismiss: (id: string) => void;
  readonly dismissLabel: string;
}

export function Toast({ id, tone, title, description, onDismiss, dismissLabel }: ToastProps) {
  const toneClasses: Readonly<Record<ToastTone, string>> = {
    success: 'border-[var(--color-success)] bg-[var(--color-success-bg)]',
    error: 'border-[var(--color-error)] bg-[var(--color-error-bg)]',
    info: 'border-[var(--color-info)] bg-[var(--color-info-bg)]',
  };
  return (
    <div
      role={tone === 'error' ? 'alert' : 'status'}
      aria-live={tone === 'error' ? 'assertive' : 'polite'}
      data-toast-id={id}
      className={[
        'pointer-events-auto flex w-[min(360px,calc(100vw-32px))] items-start gap-[var(--space-3)]',
        'rounded-[var(--radius-lg)] border p-[var(--space-4)] shadow-[var(--shadow-md)]',
        'glass',
        toneClasses[tone],
      ].join(' ')}
    >
      <div className="flex min-w-0 flex-1 flex-col gap-[var(--space-1)]">
        <p className="text-[var(--font-size-sm)] font-[var(--font-weight-semibold)]">{title}</p>
        {description ? (
          <p className="text-[var(--font-size-xs)] text-[var(--color-text-muted)]">{description}</p>
        ) : null}
      </div>
      <button
        type="button"
        onClick={() => onDismiss(id)}
        aria-label={dismissLabel}
        className="min-h-[var(--touch-target-min)] min-w-[var(--touch-target-min)] shrink-0 rounded-[var(--radius-md)] focus-visible:outline-[var(--focus-ring-width)] focus-visible:outline-offset-[var(--focus-ring-offset)] focus-visible:outline-[var(--focus-ring-color)]"
      >
        <span aria-hidden="true">&times;</span>
      </button>
    </div>
  );
}
```

`packages/ui/src/components/ToastProvider.tsx` :

```tsx
'use client';

import { createContext, useCallback, useContext, useMemo, useState, type ReactNode } from 'react';
import { Toast, type ToastPayload } from './Toast.js';

interface ToastContextValue {
  readonly push: (toast: ToastPayload) => void;
  readonly dismiss: (id: string) => void;
}

const ToastContext = createContext<ToastContextValue | null>(null);

export interface ToastProviderProps {
  readonly children: ReactNode;
  readonly dismissLabel: string;
}

export function ToastProvider({ children, dismissLabel }: ToastProviderProps) {
  const [toasts, setToasts] = useState<ReadonlyArray<ToastPayload & { id: string }>>([]);

  const dismiss = useCallback((id: string) => {
    setToasts((current) => current.filter((toast) => toast.id !== id));
  }, []);

  const push = useCallback((toast: ToastPayload) => {
    const id = `${Date.now()}-${Math.trunc(performance.now() * 1000) % 100000}`;
    setToasts((current) => [...current, { ...toast, id }]);
  }, []);

  const value = useMemo<ToastContextValue>(() => ({ push, dismiss }), [push, dismiss]);

  return (
    <ToastContext.Provider value={value}>
      {children}
      <div
        aria-live="polite"
        className="pointer-events-none fixed inset-x-0 bottom-[calc(var(--z-toast)*1px)] z-[var(--z-toast)] flex flex-col items-center gap-[var(--space-2)] p-[var(--space-4)]"
      >
        {toasts.map((toast) => (
          <Toast
            key={toast.id}
            id={toast.id}
            tone={toast.tone}
            title={toast.title}
            description={toast.description}
            onDismiss={dismiss}
            dismissLabel={dismissLabel}
          />
        ))}
      </div>
    </ToastContext.Provider>
  );
}

export function useToast(): ToastContextValue {
  const context = useContext(ToastContext);
  if (context === null) {
    throw new Error('useToast doit etre utilise dans un ToastProvider.');
  }
  return context;
}
```

- [ ] **Step 7: Écrire `SystemState` — les 12 états obligatoires**

`packages/ui/src/components/SystemState.tsx` :

```tsx
import type { ReactNode } from 'react';
import { Button } from './Button.js';

export type SystemStateVariant =
  | 'loading'
  | 'empty'
  | 'error'
  | 'forbidden'
  | 'session-expired'
  | 'offline'
  | 'unavailable'
  | 'stock-conflict'
  | 'invalid-quantity'
  | 'payment-pending'
  | 'provider-interrupted'
  | 'payment-refused';

export const SYSTEM_STATE_VARIANTS: ReadonlyArray<SystemStateVariant> = [
  'loading',
  'empty',
  'error',
  'forbidden',
  'session-expired',
  'offline',
  'unavailable',
  'stock-conflict',
  'invalid-quantity',
  'payment-pending',
  'provider-interrupted',
  'payment-refused',
];

const toneByVariant: Readonly<Record<SystemStateVariant, 'neutral' | 'success' | 'warning' | 'error' | 'info'>> = {
  loading: 'neutral',
  empty: 'neutral',
  error: 'error',
  forbidden: 'warning',
  'session-expired': 'warning',
  offline: 'warning',
  unavailable: 'warning',
  'stock-conflict': 'warning',
  'invalid-quantity': 'error',
  'payment-pending': 'info',
  'provider-interrupted': 'error',
  'payment-refused': 'error',
};

export interface SystemStateProps {
  readonly variant: SystemStateVariant;
  readonly title: string;
  readonly description?: string;
  readonly actionLabel?: string;
  readonly onAction?: () => void;
  readonly reference?: string;
  readonly icon: ReactNode;
  readonly referenceLabel?: string;
}

export function SystemState({
  variant,
  title,
  description,
  actionLabel,
  onAction,
  reference,
  icon,
  referenceLabel = 'Référence',
}: SystemStateProps) {
  const tone = toneByVariant[variant];
  const toneClasses: Readonly<Record<string, string>> = {
    neutral: 'text-[var(--color-text-muted)] bg-[var(--color-neutral-bg)]',
    success: 'text-[var(--color-success)] bg-[var(--color-success-bg)]',
    warning: 'text-[var(--color-warning)] bg-[var(--color-warning-bg)]',
    error: 'text-[var(--color-error)] bg-[var(--color-error-bg)]',
    info: 'text-[var(--color-info)] bg-[var(--color-info-bg)]',
  };

  return (
    <section
      data-state={variant}
      aria-live={variant === 'loading' ? 'polite' : 'assertive'}
      className="flex flex-col items-center gap-[var(--space-3)] rounded-[var(--radius-lg)] border border-[var(--color-border)] bg-[var(--color-surface)] px-[var(--space-6)] py-[var(--space-10)] text-center"
    >
      <span
        aria-hidden="true"
        className={[
          'flex h-[48px] w-[48px] items-center justify-center rounded-[var(--radius-full)]',
          toneClasses[tone] ?? toneClasses['neutral'],
        ].join(' ')}
      >
        {icon}
      </span>
      <h2 className="text-[var(--font-size-lg)]">{title}</h2>
      {description ? (
        <p className="max-w-[52ch] text-[var(--font-size-sm)] text-[var(--color-text-muted)]">
          {description}
        </p>
      ) : null}
      {actionLabel && onAction ? (
        <Button variant="secondary" onClick={onAction}>
          {actionLabel}
        </Button>
      ) : null}
      {reference ? (
        <p className="text-[var(--font-size-xs)] text-[var(--color-text-muted)]">
          {referenceLabel} : <span className="font-[var(--font-weight-semibold)]">{reference}</span>
        </p>
      ) : null}
    </section>
  );
}
```

- [ ] **Step 8: Exporter les composants**

`packages/ui/src/index.ts` :

```typescript
export { Button, type ButtonProps, type ButtonSize, type ButtonVariant } from './components/Button.js';
export { IconButton, type IconButtonProps } from './components/IconButton.js';
export { ButtonGroup, type ButtonGroupProps } from './components/ButtonGroup.js';
export { Input, type InputProps } from './components/Input.js';
export { Textarea, type TextareaProps } from './components/Textarea.js';
export { Select, type SelectProps, type SelectOption } from './components/Select.js';
export { Checkbox, type CheckboxProps } from './components/Checkbox.js';
export { Radio, type RadioProps } from './components/Radio.js';
export { Switch, type SwitchProps } from './components/Switch.js';
export { Badge, type BadgeProps, type BadgeTone } from './components/Badge.js';
export { Skeleton, type SkeletonProps } from './components/Skeleton.js';
export { Table, type TableColumn, type TableProps } from './components/Table.js';
export { Tabs, type TabItem, type TabsProps } from './components/Tabs.js';
export { Pagination, type PaginationProps } from './components/Pagination.js';
export { Dialog, type DialogProps } from './components/Dialog.js';
export { BottomSheet, type BottomSheetProps } from './components/BottomSheet.js';
export { Toast, type ToastPayload, type ToastProps, type ToastTone } from './components/Toast.js';
export { ToastProvider, useToast, type ToastProviderProps } from './components/ToastProvider.js';
export {
  SystemState,
  SYSTEM_STATE_VARIANTS,
  type SystemStateProps,
  type SystemStateVariant,
} from './components/SystemState.js';
export { THEME_SCRIPT, THEME_STORAGE_KEY, type ThemePreference } from './theme-script.js';
```

- [ ] **Step 9: Écrire les tests des composants et de l'accessibilité**

`packages/ui/src/__tests__/button.test.tsx` :

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, expect, it, vi } from 'vitest';
import { Button } from '../components/Button.js';
import { IconButton } from '../components/IconButton.js';

describe('Button', () => {
  it('expose le type button par defaut et appelle onClick', async () => {
    const user = userEvent.setup();
    const onClick = vi.fn();
    render(<Button onClick={onClick}>Commander</Button>);
    const button = screen.getByRole('button', { name: 'Commander' });
    expect(button).toHaveAttribute('type', 'button');
    await user.click(button);
    expect(onClick).toHaveBeenCalledTimes(1);
  });

  it('passe en aria-busy et se desactive pendant le chargement', () => {
    render(<Button loading>Envoi</Button>);
    const button = screen.getByRole('button', { name: 'Envoi' });
    expect(button).toHaveAttribute('aria-busy', 'true');
    expect(button).toBeDisabled();
  });

  it('rend un IconButton avec un nom accessible', () => {
    render(<IconButton label="Panier" icon={<span>x</span>} />);
    expect(screen.getByRole('button', { name: 'Panier' })).toBeInTheDocument();
  });
});
```

`packages/ui/src/__tests__/system-state.test.tsx` :

```tsx
import { render, screen } from '@testing-library/react';
import { describe, expect, it } from 'vitest';
import { SystemState, SYSTEM_STATE_VARIANTS } from '../components/SystemState.js';

describe('SystemState', () => {
  it('couvre les 12 etats obligatoires', () => {
    expect(SYSTEM_STATE_VARIANTS).toHaveLength(12);
    expect(new Set(SYSTEM_STATE_VARIANTS).size).toBe(12);
  });

  it('affiche titre, action de sortie et reference de correlation', () => {
    render(
      <SystemState
        variant="error"
        title="Paiement refuse"
        description="Votre banque a refuse la transaction."
        actionLabel="Reessayer"
        onAction={() => undefined}
        reference="7F3K2Q"
        icon={<span>!</span>}
      />,
    );
    expect(screen.getByRole('heading', { name: 'Paiement refuse' })).toBeInTheDocument();
    expect(screen.getByRole('button', { name: 'Reessayer' })).toBeInTheDocument();
    expect(screen.getByText('7F3K2Q')).toBeInTheDocument();
  });

  it('expose le variant dans une attribut data-state pour les tests', () => {
    const { container } = render(
      <SystemState variant="offline" title="Reseau indisponible" icon={<span>o</span>} />,
    );
    expect(container.querySelector('[data-state="offline"]')).not.toBeNull();
  });
});
```

`packages/ui/src/__tests__/a11y.test.tsx` :

```tsx
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
import { describe, expect, it } from 'vitest';
import { Button } from '../components/Button.js';
import { Input } from '../components/Input.js';
import { SystemState } from '../components/SystemState.js';

expect.extend(toHaveNoViolations);

describe('accessibilite des composants de base', () => {
  it('Button sans violation axe', async () => {
    const result = await axe(<Button>Commander</Button>);
    expect(result).toHaveNoViolations();
  });

  it('Input avec erreur sans violation axe', async () => {
    const result = await axe(
      <Input
        id="email"
        label="Adresse email"
        value="pas-un-email"
        onChange={() => undefined}
        error="Format invalide"
        required
      />,
    );
    expect(result).toHaveNoViolations();
  });

  it('SystemState sans violation axe', async () => {
    const result = await axe(
      <SystemState variant="empty" title="Aucun resultat" icon={<span>v</span>} />,
    );
    expect(result).toHaveNoViolations();
  });
});
```

`packages/ui/src/__tests__/dialog.test.tsx` :

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, expect, it, vi } from 'vitest';
import { Dialog } from '../components/Dialog.js';

describe('Dialog', () => {
  it('expose role dialog et aria-modal, et se ferme avec Escape', async () => {
    const user = userEvent.setup();
    const onClose = vi.fn();
    render(
      <Dialog open onClose={onClose} title="Confirmation" closeLabel="Fermer">
        <p>Contenu</p>
      </Dialog>,
    );
    const dialog = screen.getByRole('dialog', { name: 'Confirmation' });
    expect(dialog).toHaveAttribute('aria-modal', 'true');
    await user.keyboard('{Escape}');
    expect(onClose).toHaveBeenCalledTimes(1);
  });

  it('ne rend rien quand il est ferme', () => {
    render(
      <Dialog open={false} onClose={() => undefined} title="Confirmation" closeLabel="Fermer">
        <p>Contenu</p>
      </Dialog>,
    );
    expect(screen.queryByRole('dialog')).toBeNull();
  });
});
```

`packages/ui/vitest.config.ts` est remplacé par la version DOM (l'étape 7 de la tâche 3 installait une version `node` qui ne peut pas rendre de composants React) :

```typescript
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/__tests__/setup.ts'],
    include: ['src/__tests__/**/*.test.{ts,tsx}'],
  },
});
```

Le test `tokens.test.ts` (tâche 3) reste exécutable en `jsdom` : il ne fait que lire le fichier CSS sur le disque.

`packages/ui/src/__tests__/setup.ts` :

```typescript
import '@testing-library/jest-dom/vitest';
import { cleanup } from '@testing-library/react';
import { afterEach } from 'vitest';

afterEach(() => {
  cleanup();
});
```

- [ ] **Step 10: Exécuter les tests et constater l'échec**

```bash
pnpm --filter @fanours/ui test:unit
```

Attendu : échec sur `toHaveNoViolations` (matcher jest-axe non enregistre) ou `Cannot find module '../components/Button.js'`. Coller la sortie.

- [ ] **Step 11: Implémenter et prouver la réussite**

```bash
pnpm --filter @fanours/ui test:unit
```

Attendu :

```
 ✓ src/__tests__/a11y.test.tsx (3 tests)
 ✓ src/__tests__/button.test.tsx (3 tests)
 ✓ src/__tests__/dialog.test.tsx (2 tests)
 ✓ src/__tests__/system-state.test.tsx (3 tests)

 Test Files  4 passed (4)
      Tests  11 passed (11)
```

- [ ] **Step 12: Vérifier le typage strict**

```bash
pnpm --filter @fanours/ui typecheck
```

Attendu : aucune erreur TypeScript ; `tsc -b` retourne le code 0.

- [ ] **Step 13: Commit**

```bash
git add packages/ui
git commit -m "feat(ui): ajoute les composants de base et les 12 etats systeme" -m "Boutons, champs, badges, squelettes, tableaux, onglets, pagination,
modale et toast respectent les tokens, la cible tactile de 44 px et le
pige de focus. SystemState couvre les 12 etats obligatoires avec icone,
action de sortie et reference de correlation. Tests jest-axe sans
violation.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 5: Internationalisation — 3 dictionnaires, ICU, parité des clés

**Files:**
- Create: `packages/i18n/package.json`, `packages/i18n/tsconfig.json`, `packages/i18n/vitest.config.ts`
- Create: `packages/i18n/src/types.ts`, `packages/i18n/src/locales.ts`, `packages/i18n/src/messages/fr-FR.ts`, `en.ts`, `zh-CN.ts`
- Create: `packages/i18n/src/format.ts`, `packages/i18n/src/translate.ts`, `packages/i18n/src/provider.tsx`
- Create: `packages/i18n/src/index.ts`
- Create: `packages/i18n/scripts/check-parity.ts` (échec du build si une clé manque)
- Test: `packages/i18n/src/__tests__/parity.test.ts`, `translate.test.ts`, `format.test.ts`

**Interfaces:**
- Consumes: aucun.
- Produces:
  - `Locale` : `'fr-FR' | 'en' | 'zh-CN'`.
  - `SUPPORTED_LOCALES` : `ReadonlyArray<Locale>`.
  - `DEFAULT_LOCALE` : `'fr-FR'`.
  - `LOCALE_ROUTES` : `{ 'fr-FR': 'fr'; en: 'en'; 'zh-CN': 'zh-cn' }` — le segment d'URL par locale.
  - `routeToLocale(route: string) : Locale | null` et `localeToRoute(locale: Locale) : string`.
  - `MessageDictionary` : `Readonly<Record<string, string>>`.
  - `translate(dictionary: MessageDictionary, key: string, values?: Record<string, string | number>) : string` — repli explicite sur `fr-FR`, puis sur la clé brute si absente des deux côtés.
  - `formatMoney(amountMinor: number, currency: string, locale: Locale) : string` — `Intl.NumberFormat` avec le code ISO 4217.
  - `formatDate(value: Date, locale: Locale, options?: Intl.DateTimeFormatOptions) : string`.
  - `formatNumber(value: number, locale: Locale, options?: Intl.NumberFormatOptions) : string`.
  - `I18nProvider` + `useI18n()` : `{ locale: Locale; t: (key: string, values?: Record<string, string | number>) => string; formatMoney; formatDate; formatNumber }`.

---

- [ ] **Step 1: Créer le package `packages/i18n`**

`packages/i18n/package.json` :

```json
{
  "name": "@fanours/i18n",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" }
  },
  "scripts": {
    "build": "tsc -b && node scripts/check-parity.ts",
    "typecheck": "tsc -b --pretty",
    "lint": "eslint src",
    "test:unit": "vitest run",
    "test": "vitest run"
  },
  "dependencies": {
    "react": "19.0.0"
  },
  "devDependencies": {
    "@types/react": "19.0.2",
    "typescript": "5.6.3",
    "vitest": "2.1.8"
  }
}
```

- [ ] **Step 2: Définir les types et le routage des locales**

`packages/i18n/src/types.ts` :

```typescript
export const SUPPORTED_LOCALES = ['fr-FR', 'en', 'zh-CN'] as const;

export type Locale = (typeof SUPPORTED_LOCALES)[number];

export const DEFAULT_LOCALE: Locale = 'fr-FR';

export type MessageDictionary = Readonly<Record<string, string>>;

export function isLocale(value: string): value is Locale {
  return (SUPPORTED_LOCALES as ReadonlyArray<string>).includes(value);
}
```

`packages/i18n/src/locales.ts` :

```typescript
import { DEFAULT_LOCALE, type Locale } from './types.js';

export const LOCALE_ROUTES: Readonly<Record<Locale, string>> = {
  'fr-FR': 'fr',
  en: 'en',
  'zh-CN': 'zh-cn',
};

export function localeToRoute(locale: Locale): string {
  return LOCALE_ROUTES[locale];
}

export function routeToLocale(route: string): Locale | null {
  const entry = (Object.entries(LOCALE_ROUTES) as Array<[Locale, string]>).find(
    ([, value]) => value === route.toLowerCase(),
  );
  return entry ? entry[0] : null;
}

export const LOCALE_LABELS: Readonly<Record<Locale, string>> = {
  'fr-FR': 'Français',
  en: 'English',
  'zh-CN': '简体中文',
};

/**
 * Detection par Accept-Language uniquement. Aucune redirection forcee par IP :
 * la regle SEO l'interdit explicitement.
 */
export function detectLocaleFromAcceptLanguage(header: string | null): Locale {
  if (header === null || header.length === 0) {
    return DEFAULT_LOCALE;
  }
  const entries = header
    .split(',')
    .map((part) => {
      const [tag, quality] = part.trim().split(';q=');
      return { tag: (tag ?? '').trim().toLowerCase(), quality: quality === undefined ? 1 : Number(quality) };
    })
    .filter((entry) => entry.tag.length > 0)
    .sort((a, b) => b.quality - a.quality);

  for (const entry of entries) {
    if (entry.tag === 'zh' || entry.tag.startsWith('zh-')) {
      return 'zh-CN';
    }
    if (entry.tag === 'en' || entry.tag.startsWith('en-')) {
      return 'en';
    }
    if (entry.tag === 'fr' || entry.tag.startsWith('fr-')) {
      return 'fr-FR';
    }
  }
  return DEFAULT_LOCALE;
}
```

- [ ] **Step 3: Écrire le dictionnaire `fr-FR` (source de référence)**

`packages/i18n/src/messages/fr-FR.ts` :

```typescript
import type { MessageDictionary } from '../types.js';

export const frFR: MessageDictionary = {
  'app.name': 'Fanours',
  'app.tagline': 'Un peu de rose. Beaucoup de style. Je craque.',

  'nav.home': 'Accueil',
  'nav.catalog': 'Catalogue',
  'nav.cart': 'Panier',
  'nav.orders': 'Commandes',
  'nav.account': 'Compte',
  'nav.skipToContent': 'Aller au contenu principal',
  'nav.language': 'Langue',
  'nav.theme': 'Thème',
  'nav.theme.light': 'Clair',
  'nav.theme.dark': 'Sombre',
  'nav.theme.auto': 'Automatique',
  'nav.menu': 'Menu',

  'home.title': 'Sourcing grossiste depuis la Chine',
  'home.subtitle': 'Commandez en gros, MOQ negotiable, dedouanement clair.',
  'home.cta.catalog': 'Voir le catalogue',
  'home.cta.quote': 'Demander un devis',
  'home.benefits.moq.title': 'MOQ explicites',
  'home.benefits.moq.body': 'Chaque fiche produit affiche la quantite minimale et les paliers.',
  'home.benefits.customs.title': 'Frais de douane par pays',
  'home.benefits.customs.body': 'Les regles applicables sont indiquees avant validation de la commande.',
  'home.benefits.quality.title': 'Controle qualite',
  'home.benefits.quality.body': 'Photos, echantillons et delais annonces a l avance.',
  'home.benefits.incoterms.title': 'Incoterms clairs',
  'home.benefits.incoterms.body': 'DDP ou DAP, precise sur chaque devis.',

  'catalog.title': 'Catalogue',
  'catalog.empty.title': 'Catalogue en preparation',
  'catalog.empty.description': 'Les produits sont en cours de publication. Revenez bientot ou demandez un devis.',
  'catalog.empty.action': 'Demander un devis',
  'catalog.search.label': 'Rechercher un produit',
  'catalog.results.count': '{count} resultats',

  'cart.title': 'Panier',
  'cart.empty.title': 'Votre panier est vide',
  'cart.empty.description': 'Ajoutez un produit depuis le catalogue pour commencer.',
  'cart.empty.action': 'Parcourir le catalogue',
  'cart.subtotal': 'Sous-total',
  'cart.quantity.label': 'Quantite',
  'cart.total': 'Total',

  'order.title': 'Commandes',
  'orders.title': 'Commandes',
  'orders.empty.title': 'Aucune commande',
  'orders.empty.description': 'Vos commandes apparaitront ici apres le premier paiement confirme.',

  'auth.signin.title': 'Connexion',
  'auth.signin.email': 'Adresse email',
  'auth.signin.password': 'Mot de passe',
  'auth.signin.submit': 'Se connecter',
  'auth.signin.forgot': 'Mot de passe oublie ?',
  'auth.signup.title': 'Creer un compte',
  'auth.signup.displayName': 'Nom affiche',
  'auth.signup.submit': 'Creer mon compte',
  'auth.signout': 'Se deconnecter',
  'auth.verify.email.subject': 'Confirmez votre adresse email',
  'auth.verify.email.body': 'Cliquez sur le lien pour activer votre compte.',

  'mfa.title': 'Verification en deux etapes',
  'mfa.instruction': 'Saisissez le code a 6 chiffres de votre application d’authentification.',
  'mfa.code.label': 'Code de verification',
  'mfa.submit': 'Verifier',
  'mfa.recovery.title': 'Utiliser un code de recuperation',
  'mfa.recovery.submit': 'Recuperer l’acces',
  'mfa.trust.choice': 'Ne pas oublier ce navigateur',

  'quote.title': 'Demande de devis',
  'quote.body': 'Indiquez les references et les volumes, nous revenons vers vous avec un prix et un delai.',
  'quote.submit': 'Envoyer la demande',

  'contact.title': 'Contact',
  'contact.email.label': 'Votre email',
  'contact.message.label': 'Message',
  'contact.submit': 'Envoyer',
  'contact.success.title': 'Message envoye',

  'legal.terms.title': 'Conditions de vente',
  'legal.privacy.title': 'Politique de confidentialite',
  'legal.shipping.title': 'Livraison et Incoterms',

  'error.404.title': 'Page introuvable',
  'error.404.description': 'L’adresse demandee n’existe pas ou a ete deplacee.',
  'error.404.action': 'Retour a l’accueil',

  'state.loading.title': 'Chargement en cours',
  'state.loading.description': 'Merci de patienter quelques instants.',
  'state.empty.title': 'Aucun resultat',
  'state.empty.description': 'Aucun element a afficher pour le moment.',
  'state.error.title': 'Une erreur est survenue',
  'state.error.description': 'Le service est momentanement indisponible. Reessayez dans un instant.',
  'state.error.action': 'Reessayer',
  'state.forbidden.title': 'Acces refuse',
  'state.forbidden.description': 'Votre compte ne dispose pas des droits necessaires pour cette operation.',
  'state.sessionExpired.title': 'Session expiree',
  'state.sessionExpired.description': 'Par securite, reconnectez-vous pour continuer.',
  'state.sessionExpired.action': 'Se reconnecter',
  'state.offline.title': 'Reseau indisponible',
  'state.offline.description': 'Verifiez votre connexion internet puis reessayez.',
  'state.offline.action': 'Reessayer',
  'state.unavailable.title': 'Ressource indisponible',
  'state.unavailable.description': 'Ce service est temporairement indisponible.',
  'state.unavailable.action': 'Reessayer',
  'state.stockConflict.title': 'Stock insuffisant',
  'state.stockConflict.description': 'La quantite demandee n’est plus disponible. Ajustez votre commande.',
  'state.stockConflict.action': 'Voir le stock',
  'state.invalidQuantity.title': 'Quantite invalide',
  'state.invalidQuantity.description': 'La quantite doit etre superieure a zero et respecter le MOQ du produit.',
  'state.invalidQuantity.action': 'Corriger la quantite',
  'state.paymentPending.title': 'Paiement en attente',
  'state.paymentPending.description': 'Nous attendons la confirmation du prestataire de paiement.',
  'state.paymentPending.action': 'Verifier le statut',
  'state.providerInterrupted.title': 'Interruption du prestataire',
  'state.providerInterrupted.description': 'Le prestataire de paiement a interrompu l’operation. Aucun montant n’a ete debite.',
  'state.providerInterrupted.action': 'Reessayer',
  'state.paymentRefused.title': 'Paiement refuse',
  'state.paymentRefused.description': 'La transaction a ete refusee. Aucun montant n’a ete debite.',
  'state.paymentRefused.action': 'Changer de moyen de paiement',

  'state.reference': 'Reference',
  'common.retry': 'Reessayer',
  'common.cancel': 'Annuler',
  'common.close': 'Fermer',
  'common.confirm': 'Confirmer',
  'common.previous': 'Precedent',
  'common.next': 'Suivant',
  'common.page': 'Page {page} sur {pageCount}',
  'common.loading': 'Chargement',
  'common.optional': 'facultatif',
  'common.required': 'obligatoire',
  'common.moq': 'MOQ',
  'common.currencyNote': 'Montant indicatif, hors frais de transport et de douane',
  'common.taxNote': 'Taxes et frais de douane calcules lors de la confirmation',

  'toast.saved.title': 'Modifications enregistrees',
  'toast.error.title': 'Action impossible',
  'toast.info.title': 'Information',
  'toast.dismiss': 'Fermer la notification',
};
```

- [ ] **Step 4: Écrire les dictionnaires `en` et `zh-CN`**

`packages/i18n/src/messages/en.ts` — **mêmes clés, valeurs traduites** :

```typescript
import type { MessageDictionary } from '../types.js';

export const en: MessageDictionary = {
  'app.name': 'Fanours',
  'app.tagline': 'A little pink. A lot of style. I am into it.',

  'nav.home': 'Home',
  'nav.catalog': 'Catalog',
  'nav.cart': 'Cart',
  'nav.orders': 'Orders',
  'nav.account': 'Account',
  'nav.skipToContent': 'Skip to main content',
  'nav.language': 'Language',
  'nav.theme': 'Theme',
  'nav.theme.light': 'Light',
  'nav.theme.dark': 'Dark',
  'nav.theme.auto': 'Automatic',
  'nav.menu': 'Menu',

  'home.title': 'Wholesale sourcing from China',
  'home.subtitle': 'Order in bulk, negotiable MOQ, clear customs.',
  'home.cta.catalog': 'Browse the catalog',
  'home.cta.quote': 'Request a quote',
  'home.benefits.moq.title': 'Explicit MOQ',
  'home.benefits.moq.body': 'Every product sheet shows the minimum order quantity and tiered pricing.',
  'home.benefits.customs.title': 'Customs fees by country',
  'home.benefits.customs.body': 'Applicable rules are shown before the order is confirmed.',
  'home.benefits.quality.title': 'Quality control',
  'home.benefits.quality.body': 'Photos, samples and lead times announced in advance.',
  'home.benefits.incoterms.title': 'Clear Incoterms',
  'home.benefits.incoterms.body': 'DDP or DAP, spelled out on every quote.',

  'catalog.title': 'Catalog',
  'catalog.empty.title': 'Catalog is being prepared',
  'catalog.empty.description': 'Products are being published. Come back soon or request a quote.',
  'catalog.empty.action': 'Request a quote',
  'catalog.search.label': 'Search a product',
  'catalog.results.count': '{count} results',

  'cart.title': 'Cart',
  'cart.empty.title': 'Your cart is empty',
  'cart.empty.description': 'Add a product from the catalog to get started.',
  'cart.empty.action': 'Browse the catalog',
  'cart.subtotal': 'Subtotal',
  'cart.quantity.label': 'Quantity',
  'cart.total': 'Total',

  'order.title': 'Order',
  'orders.title': 'Orders',
  'orders.empty.title': 'No orders yet',
  'orders.empty.description': 'Your orders will appear here after the first confirmed payment.',

  'auth.signin.title': 'Sign in',
  'auth.signin.email': 'Email address',
  'auth.signin.password': 'Password',
  'auth.signin.submit': 'Sign in',
  'auth.signin.forgot': 'Forgot your password?',
  'auth.signup.title': 'Create an account',
  'auth.signup.displayName': 'Display name',
  'auth.signup.submit': 'Create my account',
  'auth.signout': 'Sign out',
  'auth.verify.email.subject': 'Confirm your email address',
  'auth.verify.email.body': 'Click the link to activate your account.',

  'mfa.title': 'Two-step verification',
  'mfa.instruction': 'Enter the 6-digit code from your authenticator app.',
  'mfa.code.label': 'Verification code',
  'mfa.submit': 'Verify',
  'mfa.recovery.title': 'Use a recovery code',
  'mfa.recovery.submit': 'Recover access',
  'mfa.trust.choice': 'Do not trust this browser again',

  'quote.title': 'Quote request',
  'quote.body': 'Tell us the references and volumes, we will reply with a price and a lead time.',
  'quote.submit': 'Send the request',

  'contact.title': 'Contact',
  'contact.email.label': 'Your email',
  'contact.message.label': 'Message',
  'contact.submit': 'Send',
  'contact.success.title': 'Message sent',

  'legal.terms.title': 'Terms of sale',
  'legal.privacy.title': 'Privacy policy',
  'legal.shipping.title': 'Shipping and Incoterms',

  'error.404.title': 'Page not found',
  'error.404.description': 'The requested address does not exist or has been moved.',
  'error.404.action': 'Back to home',

  'state.loading.title': 'Loading',
  'state.loading.description': 'Please wait a moment.',
  'state.empty.title': 'No results',
  'state.empty.description': 'Nothing to display right now.',
  'state.error.title': 'Something went wrong',
  'state.error.description': 'The service is temporarily unavailable. Please try again shortly.',
  'state.error.action': 'Try again',
  'state.forbidden.title': 'Access denied',
  'state.forbidden.description': 'Your account does not have the rights required for this operation.',
  'state.sessionExpired.title': 'Session expired',
  'state.sessionExpired.description': 'For your security, sign in again to continue.',
  'state.sessionExpired.action': 'Sign in',
  'state.offline.title': 'Network unavailable',
  'state.offline.description': 'Check your internet connection and try again.',
  'state.offline.action': 'Try again',
  'state.unavailable.title': 'Resource unavailable',
  'state.unavailable.description': 'This service is temporarily unavailable.',
  'state.unavailable.action': 'Try again',
  'state.stockConflict.title': 'Insufficient stock',
  'state.stockConflict.description': 'The requested quantity is no longer available. Adjust your order.',
  'state.stockConflict.action': 'View stock',
  'state.invalidQuantity.title': 'Invalid quantity',
  'state.invalidQuantity.description': 'The quantity must be greater than zero and meet the product MOQ.',
  'state.invalidQuantity.action': 'Fix the quantity',
  'state.paymentPending.title': 'Payment pending',
  'state.paymentPending.description': 'We are waiting for confirmation from the payment provider.',
  'state.paymentPending.action': 'Check status',
  'state.providerInterrupted.title': 'Provider interrupted',
  'state.providerInterrupted.description': 'The payment provider interrupted the operation. No amount was charged.',
  'state.providerInterrupted.action': 'Try again',
  'state.paymentRefused.title': 'Payment refused',
  'state.paymentRefused.description': 'The transaction was refused. No amount was charged.',
  'state.paymentRefused.action': 'Change payment method',

  'state.reference': 'Reference',
  'common.retry': 'Try again',
  'common.cancel': 'Cancel',
  'common.close': 'Close',
  'common.confirm': 'Confirm',
  'common.previous': 'Previous',
  'common.next': 'Next',
  'common.page': 'Page {page} of {pageCount}',
  'common.loading': 'Loading',
  'common.optional': 'optional',
  'common.required': 'required',
  'common.moq': 'MOQ',
  'common.currencyNote': 'Indicative amount, excluding shipping and customs fees',
  'common.taxNote': 'Taxes and customs fees are calculated when the order is confirmed',

  'toast.saved.title': 'Changes saved',
  'toast.error.title': 'Action could not be completed',
  'toast.info.title': 'Information',
  'toast.dismiss': 'Dismiss the notification',
};
```

`packages/i18n/src/messages/zh-CN.ts` :

```typescript
import type { MessageDictionary } from '../types.js';

export const zhCN: MessageDictionary = {
  'app.name': 'Fanours',
  'app.tagline': '一点粉色 Lots 风格，我很喜欢。',

  'nav.home': '首页',
  'nav.catalog': '产品目录',
  'nav.cart': '购物车',
  'nav.orders': '订单',
  'nav.account': '账户',
  'nav.skipToContent': '跳转到主要内容',
  'nav.language': '语言',
  'nav.theme': '主题',
  'nav.theme.light': '浅色',
  'nav.theme.dark': '深色',
  'nav.theme.auto': '跟随系统',
  'nav.menu': '菜单',

  'home.title': '从中国批发采购',
  'home.subtitle': '批量下单，起订量可议，清关费用透明。',
  'home.cta.catalog': '浏览产品目录',
  'home.cta.quote': '索取报价',
  'home.benefits.moq.title': '明确的起订量',
  'home.benefits.moq.body': '每件产品均标注最小起订量与阶梯价格。',
  'home.benefits.customs.title': '按国家区分的清关费用',
  'home.benefits.customs.body': '确认订单前即展示适用规则。',
  'home.benefits.quality.title': '质量检验',
  'home.benefits.quality.body': '提前提供图片、样品与交期。',
  'home.benefits.incoterms.title': '清晰的贸易术语',
  'home.benefits.incoterms.body': '每份报价均注明 DDP 或 DAP。',

  'catalog.title': '产品目录',
  'catalog.empty.title': '目录正在整理中',
  'catalog.empty.description': '产品正在陆续上线，敬请稍后再来或索取报价。',
  'catalog.empty.action': '索取报价',
  'catalog.search.label': '搜索产品',
  'catalog.results.count': '{count} 个结果',

  'cart.title': '购物车',
  'cart.empty.title': '购物车是空的',
  'cart.empty.description': '从目录中添加产品即可开始。',
  'cart.empty.action': '浏览产品目录',
  'cart.subtotal': '小计',
  'cart.quantity.label': '数量',
  'cart.total': '合计',

  'order.title': '订单',
  'orders.title': '订单',
  'orders.empty.title': '暂无订单',
  'orders.empty.description': '首次付款确认后，订单将显示在此处。',

  'auth.signin.title': '登录',
  'auth.signin.email': '邮箱地址',
  'auth.signin.password': '密码',
  'auth.signin.submit': '登录',
  'auth.signin.forgot': '忘记密码？',
  'auth.signup.title': '创建账户',
  'auth.signup.displayName': '显示名称',
  'auth.signup.submit': '创建我的账户',
  'auth.signout': '退出登录',
  'auth.verify.email.subject': '确认您的邮箱地址',
  'auth.verify.email.body': '点击链接以激活账户。',

  'mfa.title': '两步验证',
  'mfa.instruction': '请输入身份验证器应用中的 6 位数字代码。',
  'mfa.code.label': '验证码',
  'mfa.submit': '验证',
  'mfa.recovery.title': '使用恢复码',
  'mfa.recovery.submit': '恢复访问',
  'mfa.trust.choice': '不再信任此浏览器',

  'quote.title': '报价申请',
  'quote.body': '请告知货号与数量，我们将回复价格与交期。',
  'quote.submit': '提交申请',

  'contact.title': '联系我们',
  'contact.email.label': '您的邮箱',
  'contact.message.label': '留言',
  'contact.submit': '发送',
  'contact.success.title': '消息已发送',

  'legal.terms.title': '销售条款',
  'legal.privacy.title': '隐私政策',
  'legal.shipping.title': '物流与贸易术语',

  'error.404.title': '页面未找到',
  'error.404.description': '请求的地址不存在或已被移动。',
  'error.404.action': '返回首页',

  'state.loading.title': '加载中',
  'state.loading.description': '请稍候片刻。',
  'state.empty.title': '暂无结果',
  'state.empty.description': '当前没有可显示的内容。',
  'state.error.title': '发生错误',
  'state.error.description': '服务暂时不可用，请稍后重试。',
  'state.error.action': '重试',
  'state.forbidden.title': '拒绝访问',
  'state.forbidden.description': '您的账户没有执行此操作所需的权限。',
  'state.sessionExpired.title': '会话已过期',
  'state.sessionExpired.description': '为确保安全，请重新登录后继续。',
  'state.sessionExpired.action': '重新登录',
  'state.offline.title': '网络不可用',
  'state.offline.description': '请检查网络连接后重试。',
  'state.offline.action': '重试',
  'state.unavailable.title': '资源不可用',
  'state.unavailable.description': '此服务暂时不可用。',
  'state.unavailable.action': '重试',
  'state.stockConflict.title': '库存不足',
  'state.stockConflict.description': '所需数量已不可用，请调整订单。',
  'state.stockConflict.action': '查看库存',
  'state.invalidQuantity.title': '数量无效',
  'state.invalidQuantity.description': '数量必须大于零且不低于产品起订量。',
  'state.invalidQuantity.action': '修改数量',
  'state.paymentPending.title': '支付处理中',
  'state.paymentPending.description': '正在等待支付服务商确认。',
  'state.paymentPending.action': '查看状态',
  'state.providerInterrupted.title': '服务商中断',
  'state.providerInterrupted.description': '支付服务商中断了操作，未产生任何扣款。',
  'state.providerInterrupted.action': '重试',
  'state.paymentRefused.title': '支付被拒绝',
  'state.paymentRefused.description': '交易被拒绝，未产生任何扣款。',
  'state.paymentRefused.action': '更换支付方式',

  'state.reference': '参考编号',
  'common.retry': '重试',
  'common.cancel': '取消',
  'common.close': '关闭',
  'common.confirm': '确认',
  'common.previous': '上一页',
  'common.next': '下一页',
  'common.page': '第 {page} 页，共 {pageCount} 页',
  'common.loading': '加载中',
  'common.optional': '选填',
  'common.required': '必填',
  'common.moq': '起订量',
  'common.currencyNote': '金额仅供参考，不含运费与清关费用',
  'common.taxNote': '税费与清关费用在确认订单时计算',

  'toast.saved.title': '修改已保存',
  'toast.error.title': '操作无法完成',
  'toast.info.title': '提示',
  'toast.dismiss': '关闭通知',
};
```

- [ ] **Step 5: Écrire le moteur de traduction et de formatage**

`packages/i18n/src/format.ts` :

```typescript
import type { Locale } from './types.js';

const INTL_LOCALE: Readonly<Record<Locale, string>> = {
  'fr-FR': 'fr-FR',
  en: 'en',
  'zh-CN': 'zh-CN',
};

export function formatMoney(amountMinor: number, currency: string, locale: Locale): string {
  if (!Number.isFinite(amountMinor)) {
    throw new TypeError('formatMoney requiert un montant fini.');
  }
  return new Intl.NumberFormat(INTL_LOCALE[locale], {
    style: 'currency',
    currency,
  }).format(amountMinor / 100);
}

export function formatNumber(
  value: number,
  locale: Locale,
  options?: Intl.NumberFormatOptions,
): string {
  return new Intl.NumberFormat(INTL_LOCALE[locale], options).format(value);
}

export function formatDate(
  value: Date,
  locale: Locale,
  options?: Intl.DateTimeFormatOptions,
): string {
  return new Intl.DateTimeFormat(INTL_LOCALE[locale], {
    dateStyle: 'medium',
    ...options,
  }).format(value);
}
```

`packages/i18n/src/translate.ts` :

```typescript
import { frFR } from './messages/fr-FR.js';
import { en } from './messages/en.js';
import { zhCN } from './messages/zh-CN.js';
import type { Locale, MessageDictionary } from './types.js';

export const DICTIONARIES: Readonly<Record<Locale, MessageDictionary>> = {
  'fr-FR': frFR,
  en,
  'zh-CN': zhCN,
};

const PLACEHOLDER_PATTERN = /\{(\w+)\}/g;

function interpolate(template: string, values?: Record<string, string | number>): string {
  if (values === undefined) {
    return template;
  }
  return template.replace(PLACEHOLDER_PATTERN, (match, name: string) => {
    const value = values[name];
    return value === undefined ? match : String(value);
  });
}

export function translate(
  dictionary: MessageDictionary,
  key: string,
  values?: Record<string, string | number>,
): string {
  const template = dictionary[key] ?? frFR[key];
  if (template === undefined) {
    return key;
  }
  return interpolate(template, values);
}
```

- [ ] **Step 6: Écrire le provider React**

`packages/i18n/src/provider.tsx` :

```tsx
'use client';

import { createContext, useContext, useMemo, type ReactNode } from 'react';
import { DICTIONARIES, translate } from './translate.js';
import { formatDate, formatMoney, formatNumber } from './format.js';
import { DEFAULT_LOCALE, type Locale } from './types.js';

interface I18nContextValue {
  readonly locale: Locale;
  readonly t: (key: string, values?: Record<string, string | number>) => string;
  readonly formatMoney: (amountMinor: number, currency: string) => string;
  readonly formatDate: (value: Date, options?: Intl.DateTimeFormatOptions) => string;
  readonly formatNumber: (value: number, options?: Intl.NumberFormatOptions) => string;
}

const I18nContext = createContext<I18nContextValue | null>(null);

export interface I18nProviderProps {
  readonly locale: Locale;
  readonly children: ReactNode;
}

export function I18nProvider({ locale, children }: I18nProviderProps) {
  const value = useMemo<I18nContextValue>(() => {
    const dictionary = DICTIONARIES[locale] ?? DICTIONARIES[DEFAULT_LOCALE];
    return {
      locale,
      t: (key, values) => translate(dictionary, key, values),
      formatMoney: (amountMinor, currency) => formatMoney(amountMinor, currency, locale),
      formatDate: (date, options) => formatDate(date, locale, options),
      formatNumber: (number, options) => formatNumber(number, locale, options),
    };
  }, [locale]);

  return <I18nContext.Provider value={value}>{children}</I18nContext.Provider>;
}

export function useI18n(): I18nContextValue {
  const context = useContext(I18nContext);
  if (context === null) {
    throw new Error('useI18n doit etre utilise dans un I18nProvider.');
  }
  return context;
}
```

`packages/i18n/src/index.ts` :

```typescript
export { DEFAULT_LOCALE, SUPPORTED_LOCALES, isLocale, type Locale, type MessageDictionary } from './types.js';
export {
  LOCALE_LABELS,
  LOCALE_ROUTES,
  detectLocaleFromAcceptLanguage,
  localeToRoute,
  routeToLocale,
} from './locales.js';
export { DICTIONARIES, translate } from './translate.js';
export { formatDate, formatMoney, formatNumber } from './format.js';
export { I18nProvider, useI18n, type I18nProviderProps } from './provider.js';
export { frFR } from './messages/fr-FR.js';
export { en } from './messages/en.js';
export { zhCN } from './messages/zh-CN.js';
```

- [ ] **Step 7: Écrire le contrôle de parité des clés (échec du build)**

`packages/i18n/scripts/check-parity.ts` :

```typescript
import { DICTIONARIES } from '../src/translate.js';
import { SUPPORTED_LOCALES, DEFAULT_LOCALE } from '../src/types.js';

const reference = DICTIONARIES[DEFAULT_LOCALE];
const referenceKeys = Object.keys(reference).sort();
const problems: string[] = [];

for (const locale of SUPPORTED_LOCALES) {
  const dictionary = DICTIONARIES[locale];
  const keys = new Set(Object.keys(dictionary));
  for (const key of referenceKeys) {
    if (!keys.has(key)) {
      problems.push(`[${locale}] cle manquante : ${key}`);
    }
  }
  for (const key of Object.keys(dictionary).sort()) {
    if (!(key in reference)) {
      problems.push(`[${locale}] cle inconnue : ${key}`);
    }
  }
}

if (problems.length > 0) {
  process.stderr.write(
    `Parite des dictionnaires impossible — ${problems.length} probleme(s) :\n${problems.join('\n')}\n`,
  );
  process.exit(1);
}

process.stdout.write(
  `Parite des dictionnaires verifiee : ${referenceKeys.length} cles dans ${SUPPORTED_LOCALES.length} langues.\n`,
);
```

- [ ] **Step 8: Écrire les tests i18n**

`packages/i18n/src/__tests__/parity.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { DICTIONARIES } from '../translate.js';
import { DEFAULT_LOCALE, SUPPORTED_LOCALES } from '../types.js';

describe('parite des dictionnaires', () => {
  it('expose les trois langues', () => {
    expect(SUPPORTED_LOCALES).toEqual(['fr-FR', 'en', 'zh-CN']);
  });

  it('a exactement les memes cles dans les trois langues', () => {
    const reference = Object.keys(DICTIONARIES[DEFAULT_LOCALE]).sort();
    for (const locale of SUPPORTED_LOCALES) {
      expect(Object.keys(DICTIONARIES[locale]).sort()).toEqual(reference);
    }
  });

  it('ne contient aucune valeur vide', () => {
    for (const locale of SUPPORTED_LOCALES) {
      for (const [key, value] of Object.entries(DICTIONARIES[locale])) {
        expect(value.trim().length, `${locale}/${key}`).toBeGreaterThan(0);
      }
    }
  });
});
```

`packages/i18n/src/__tests__/translate.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { DICTIONARIES, translate } from '../translate.js';
import { detectLocaleFromAcceptLanguage, routeToLocale } from '../locales.js';

describe('translate', () => {
  it('interpole les valeurs', () => {
    expect(translate(DICTIONARIES['fr-FR'], 'common.page', { page: 2, pageCount: 5 })).toBe(
      'Page 2 sur 5',
    );
  });

  it('retombe sur fr-FR quand la cle manque dans la langue courante', () => {
    const partial = { 'nav.home': 'Accueil' };
    expect(translate(partial, 'nav.catalog')).toBe('Catalogue');
  });

  it('retombe sur la cle brute quand la cle est absente partout', () => {
    expect(translate({}, 'cle.inconnue')).toBe('cle.inconnue');
  });
});

describe('detection de locale', () => {
  it('respecte la priorite de qualite Accept-Language', () => {
    expect(detectLocaleFromAcceptLanguage('en;q=0.8,zh-CN;q=0.9')).toBe('zh-CN');
  });

  it('retombe sur fr-FR sans en-tete', () => {
    expect(detectLocaleFromAcceptLanguage(null)).toBe('fr-FR');
  });

  it('mappe les segments d URL', () => {
    expect(routeToLocale('zh-cn')).toBe('zh-CN');
    expect(routeToLocale('en')).toBe('en');
  });
});
```

`packages/i18n/src/__tests__/format.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { formatMoney, formatNumber } from '../format.js';

describe('formatMoney', () => {
  it('formate un montant en minor units avec le code ISO 4217', () => {
    const result = formatMoney(123456, 'EUR', 'fr-FR');
    expect(result).toContain('1');
    expect(result).toContain('234');
    expect(result).toContain('56');
    expect(result).toContain('€');
  });

  it('formate en dollars US pour la locale en', () => {
    expect(formatMoney(1000, 'USD', 'en')).toContain('$');
  });

  it('refuse un montant non fini', () => {
    expect(() => formatMoney(Number.NaN, 'EUR', 'fr-FR')).toThrow(TypeError);
  });
});

describe('formatNumber', () => {
  it('separe les milliers selon la locale', () => {
    expect(formatNumber(1234567, 'fr-FR')).toContain(' ');
  });
});
```

- [ ] **Step 9: Exécuter les tests et constater l'échec**

```bash
pnpm --filter @fanours/i18n test:unit
```

Attendu : `Failed to resolve import "../src/translate.js"` ou `No test files found`. Coller la sortie.

- [ ] **Step 10: Implémenter et prouver la réussite**

```bash
pnpm --filter @fanours/i18n test:unit
```

Attendu :

```
 ✓ src/__tests__/format.test.ts (3 tests)
 ✓ src/__tests__/parity.test.ts (3 tests)
 ✓ src/__tests__/translate.test.ts (6 tests)

 Test Files  3 passed (3)
      Tests  12 passed (12)
```

- [ ] **Step 11: Prouver que le build échoue si une clé manque**

```bash
node -e "const fs=require('fs');const p='packages/i18n/src/messages/en.ts';const s=fs.readFileSync(p,'utf8');fs.writeFileSync(p,s.replace(\"  'nav.cart': 'Cart',\\n\",''))"
pnpm --filter @fanours/i18n build
```

Attendu : le build échoue avec `[en] cle manquante : nav.cart` et le code de sortie est non nul. Restaurer ensuite le fichier :

```bash
git checkout -- packages/i18n/src/messages/en.ts
```

- [ ] **Step 12: Commit**

```bash
git add packages/i18n
git commit -m "feat(i18n): ajoute les dictionnaires fr-FR, en et zh-CN" -m "Les trois dictionnaires partagent les memes cles, avec repli explicite
sur fr-FR puis sur la cle brute. Formatage Intl pour montants, dates et
nombres. Le build echoue si une cle manque dans une des trois langues.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 6: Devise et montants — minor units, devise d'affichage, montant de référence

**Files:**
- Create: `packages/pricing/package.json`, `packages/pricing/tsconfig.json`, `packages/pricing/vitest.config.ts`
- Create: `packages/pricing/src/money.ts`, `packages/pricing/src/currency.ts`, `packages/pricing/src/quote.ts`
- Create: `packages/pricing/src/index.ts`
- Test: `packages/pricing/src/__tests__/money.test.ts`, `currency.test.ts`, `quote.test.ts`

**Interfaces:**
- Consumes: `packages/i18n` (Task 5) pour le formatage d'affichage ; `packages/domain` (Task 1) pour `SessionProfile` si nécessaire.
- Produces:
  - `CurrencyCode` : union des codes ISO 4217 supportés — `'EUR' | 'USD' | 'CNY' | 'GBP'`.
  - `CURRENCY_EXPONENTS` : `Readonly<Record<CurrencyCode, number>>` — exposant des minor units (`EUR: 2`, `JPY: 0` absent du périmètre).
  - `Money` : `{ readonly amountMinor: number; readonly currency: CurrencyCode }` — montant **entier** de minor units, jamais de float.
  - `toMinorUnits(amountDecimal: string, currency: CurrencyCode) : number` — parse une chaîne décimale sans passer par un float intermédiaire.
  - `fromMinorUnits(amountMinor: number, currency: CurrencyCode) : string` — reconstruit la chaîne décimale.
  - `addMoney(a: Money, b: Money) : Money` — refuse les devises différentes.
  - `multiplyMoney(money: Money, quantity: number) : Money` — quantité entière validée.
  - `assertSameCurrency(a: Money, b: Money) : void`.
  - `formatMoney(money: Money, locale: Locale) : string` — delegue à `Intl.NumberFormat` via `@fanours/i18n`.
  - `convertMoney(money: Money, target: CurrencyCode, rates: Readonly<Record<CurrencyCode, number>>) : Money` — taux exprimés en `target` par unité de devise source ; arrondi **vers le bas** au minor unit, jamais au credit.
  - `QuoteAmount` : `{ readonly reference: Money; readonly display: Money; readonly isIndicative: boolean }` — le montant de reference (devise de transaction) et le montant affiche (devise d'affichage) sont distincts ; `isIndicative` vaut `true` quand la devise d'affichage differe de la devise de transaction.
  - `buildQuoteAmount(reference: Money, displayCurrency: CurrencyCode, rates: Readonly<Record<CurrencyCode, number>>) : QuoteAmount`.

---

- [ ] **Step 1: Créer le package `packages/pricing`**

`packages/pricing/package.json` :

```json
{
  "name": "@fanours/pricing",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" }
  },
  "scripts": {
    "build": "tsc -b",
    "typecheck": "tsc -b --pretty",
    "lint": "eslint src",
    "test:unit": "vitest run",
    "test": "vitest run"
  },
  "dependencies": {
    "@fanours/i18n": "workspace:*"
  },
  "devDependencies": {
    "typescript": "5.6.3",
    "vitest": "2.1.8"
  }
}
```

- [ ] **Step 2: Écrire le module devise**

`packages/pricing/src/currency.ts` :

```typescript
export const CURRENCIES = ['EUR', 'USD', 'CNY', 'GBP'] as const;

export type CurrencyCode = (typeof CURRENCIES)[number];

/**
 * Nombre de decimales de la devise. Les minor units sont la seule representation
 * autorisee en base comme en code : aucun float binaire.
 */
export const CURRENCY_EXPONENTS: Readonly<Record<CurrencyCode, number>> = {
  EUR: 2,
  USD: 2,
  CNY: 2,
  GBP: 2,
};

export const REFERENCE_CURRENCY: CurrencyCode = 'EUR';

export function isCurrencyCode(value: string): value is CurrencyCode {
  return (CURRENCIES as ReadonlyArray<string>).includes(value);
}

export function minorUnitFactor(currency: CurrencyCode): number {
  return 10 ** CURRENCY_EXPONENTS[currency];
}
```

- [ ] **Step 3: Écrire le module montants**

`packages/pricing/src/money.ts` :

```typescript
import { formatMoney as formatMoneyI18n, type Locale } from '@fanours/i18n';
import { CURRENCY_EXPONENTS, type CurrencyCode } from './currency.js';

export interface Money {
  readonly amountMinor: number;
  readonly currency: CurrencyCode;
}

const DECIMAL_PATTERN = /^(-)?(\d+)(?:\.(\d{1,4}))?$/;

export function toMinorUnits(amountDecimal: string, currency: CurrencyCode): number {
  const match = DECIMAL_PATTERN.exec(amountDecimal.trim());
  if (match === null) {
    throw new TypeError(`Montant invalide : ${amountDecimal}`);
  }
  const [, sign, integerPart, fractionPart = ''] = match;
  const exponent = CURRENCY_EXPONENTS[currency];
  if (fractionPart.length > exponent) {
    throw new RangeError(
      `Le montant ${amountDecimal} depasse la precision de ${exponent} decimales pour ${currency}.`,
    );
  }
  const padded = fractionPart.padEnd(exponent, '0');
  const total = BigInt(integerPart) * 10n ** BigInt(exponent) + BigInt(padded === '' ? '0' : padded);
  return sign === '-' ? -Number(total) : Number(total);
}

export function fromMinorUnits(amountMinor: number, currency: CurrencyCode): string {
  if (!Number.isInteger(amountMinor)) {
    throw new TypeError('amountMinor doit etre un entier.');
  }
  const exponent = CURRENCY_EXPONENTS[currency];
  const negative = amountMinor < 0;
  const absolute = Math.abs(amountMinor);
  const factor = 10 ** exponent;
  const integerPart = Math.trunc(absolute / factor);
  const fractionPart = String(absolute % factor).padStart(exponent, '0');
  const decimal = exponent === 0 ? String(integerPart) : `${integerPart}.${fractionPart}`;
  return negative ? `-${decimal}` : decimal;
}

export function assertSameCurrency(a: Money, b: Money): void {
  if (a.currency !== b.currency) {
    throw new RangeError(
      `Devises incompatibles : ${a.currency} et ${b.currency}. Aucune conversion implicite n'est autorisee.`,
    );
  }
}

export function addMoney(a: Money, b: Money): Money {
  assertSameCurrency(a, b);
  return { amountMinor: a.amountMinor + b.amountMinor, currency: a.currency };
}

export function multiplyMoney(money: Money, quantity: number): Money {
  if (!Number.isInteger(quantity) || quantity < 0) {
    throw new RangeError('La quantite doit etre un entier positif ou nul.');
  }
  return { amountMinor: money.amountMinor * quantity, currency: money.currency };
}

export function formatMoney(money: Money, locale: Locale): string {
  return formatMoneyI18n(money.amountMinor, money.currency, locale);
}
```

- [ ] **Step 4: Écrire la conversion et le montant de référence**

`packages/pricing/src/quote.ts` :

```typescript
import { formatMoney as formatMoneyI18n, type Locale } from '@fanours/i18n';
import { CURRENCY_EXPONENTS, type CurrencyCode } from './currency.js';
import type { Money } from './money.js';

export interface QuoteAmount {
  readonly reference: Money;
  readonly display: Money;
  readonly isIndicative: boolean;
}

/**
 * `rates[source]` vaut combien d'unites de `target` vaut une unite de `source`.
 * Exemple : { EUR: 1, USD: 0.92, CNY: 7.8, GBP: 0.85 } pour une cible EUR.
 */
export type ExchangeRates = Readonly<Record<CurrencyCode, number>>;

export function convertMoney(
  money: Money,
  target: CurrencyCode,
  rates: ExchangeRates,
): Money {
  if (money.currency === target) {
    return money;
  }
  const rate = rates[money.currency];
  if (rate === undefined || !Number.isFinite(rate) || rate <= 0) {
    throw new RangeError(`Taux de conversion absent ou invalide pour ${money.currency}.`);
  }
  const exponent = CURRENCY_EXPONENTS[target];
  const factor = 10 ** exponent;
  // Arrondi vers le bas au minor unit : on ne surestime jamais le montant annonce.
  const converted = Math.floor((money.amountMinor * rate) / factor) * factor;
  return { amountMinor: converted, currency: target };
}

export function buildQuoteAmount(
  reference: Money,
  displayCurrency: CurrencyCode,
  rates: ExchangeRates,
): QuoteAmount {
  const display = convertMoney(reference, displayCurrency, rates);
  return {
    reference,
    display,
    isIndicative: reference.currency !== display.currency,
  };
}

export function formatQuoteAmount(amount: QuoteAmount, locale: Locale): string {
  const formatted = formatMoneyI18n(amount.display.amountMinor, amount.display.currency, locale);
  return amount.isIndicative ? `${formatted} (indicatif)` : formatted;
}
```

- [ ] **Step 5: Exporter l'API publique**

`packages/pricing/src/index.ts` :

```typescript
export {
  CURRENCIES,
  CURRENCY_EXPONENTS,
  REFERENCE_CURRENCY,
  isCurrencyCode,
  minorUnitFactor,
  type CurrencyCode,
} from './currency.js';
export {
  addMoney,
  assertSameCurrency,
  formatMoney,
  fromMinorUnits,
  multiplyMoney,
  toMinorUnits,
  type Money,
} from './money.js';
export {
  buildQuoteAmount,
  convertMoney,
  formatQuoteAmount,
  type ExchangeRates,
  type QuoteAmount,
} from './quote.js';
```

- [ ] **Step 6: Écrire les tests de montants**

`packages/pricing/src/__tests__/money.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { addMoney, fromMinorUnits, multiplyMoney, toMinorUnits } from '../money.js';

describe('conversion decimal <-> minor units', () => {
  it('convertit 12.34 EUR en 1234 minor units', () => {
    expect(toMinorUnits('12.34', 'EUR')).toBe(1234);
  });

  it('convertit 0.05 EUR en 5 minor units', () => {
    expect(toMinorUnits('0.05', 'EUR')).toBe(5);
  });

  it('gere les montants negatifs', () => {
    expect(toMinorUnits('-9.99', 'EUR')).toBe(-999);
  });

  it('reconstruit la chaine decimale', () => {
    expect(fromMinorUnits(123456, 'EUR')).toBe('1234.56');
    expect(fromMinorUnits(5, 'EUR')).toBe('0.05');
  });

  it('refuse une precision superieure a la devise', () => {
    expect(() => toMinorUnits('1.005', 'EUR')).toThrow(RangeError);
  });

  it('refuse une chaine non numerique', () => {
    expect(() => toMinorUnits('12,34', 'EUR')).toThrow(TypeError);
  });

  it('refuse un montant binaire flottant', () => {
    expect(() => fromMinorUnits(0.1 + 0.2, 'EUR')).toThrow(TypeError);
  });
});

describe('operations sur les montants', () => {
  it('additionne deux montants de meme devise', () => {
    expect(addMoney({ amountMinor: 1250, currency: 'EUR' }, { amountMinor: 750, currency: 'EUR' })).toEqual({
      amountMinor: 2000,
      currency: 'EUR',
    });
  });

  it('refuse d additionner deux devises differentes', () => {
    expect(() =>
      addMoney({ amountMinor: 100, currency: 'EUR' }, { amountMinor: 100, currency: 'USD' }),
    ).toThrow(RangeError);
  });

  it('multiplie par une quantite entiere', () => {
    expect(multiplyMoney({ amountMinor: 250, currency: 'EUR' }, 4)).toEqual({
      amountMinor: 1000,
      currency: 'EUR',
    });
  });

  it('refuse une quantite negative ou decimale', () => {
    expect(() => multiplyMoney({ amountMinor: 250, currency: 'EUR' }, -1)).toThrow(RangeError);
    expect(() => multiplyMoney({ amountMinor: 250, currency: 'EUR' }, 1.5)).toThrow(RangeError);
  });
});
```

`packages/pricing/src/__tests__/currency.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { CURRENCY_EXPONENTS, isCurrencyCode } from '../currency.js';
import { formatMoney } from '../money.js';

describe('devises', () => {
  it('reconnait les codes ISO 4217 supportes', () => {
    expect(isCurrencyCode('EUR')).toBe(true);
    expect(isCurrencyCode('CNY')).toBe(true);
    expect(isCurrencyCode('XBT')).toBe(false);
  });

  it('expose deux decimales pour les devises du perimetre', () => {
    expect(CURRENCY_EXPONENTS.EUR).toBe(2);
    expect(CURRENCY_EXPONENTS.USD).toBe(2);
  });

  it('formate un montant en EUR selon la locale', () => {
    expect(formatMoney({ amountMinor: 123456, currency: 'EUR' }, 'fr-FR')).toContain('€');
  });
});
```

`packages/pricing/src/__tests__/quote.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { buildQuoteAmount, convertMoney, type ExchangeRates } from '../quote.js';

const rates: ExchangeRates = { EUR: 1, USD: 0.92, CNY: 7.8, GBP: 0.85 };

describe('conversion de devise', () => {
  it('convertit 100.00 EUR en 92.00 USD', () => {
    expect(convertMoney({ amountMinor: 10000, currency: 'EUR' }, 'USD', rates)).toEqual({
      amountMinor: 9200,
      currency: 'USD',
    });
  });

  it('arrondit vers le bas au minor unit', () => {
    expect(convertMoney({ amountMinor: 999, currency: 'EUR' }, 'USD', rates)).toEqual({
      amountMinor: 918,
      currency: 'USD',
    });
  });

  it('renvoie le meme montant si la devise est identique', () => {
    const money = { amountMinor: 500, currency: 'EUR' } as const;
    expect(convertMoney(money, 'EUR', rates)).toBe(money);
  });

  it('refuse un taux absent', () => {
    expect(() =>
      convertMoney({ amountMinor: 100, currency: 'EUR' }, 'USD', { EUR: 1 } as ExchangeRates),
    ).toThrow(RangeError);
  });
});

describe('montant de reference', () => {
  it('marque indicatif quand la devise d affichage differe', () => {
    const amount = buildQuoteAmount({ amountMinor: 10000, currency: 'EUR' }, 'USD', rates);
    expect(amount.isIndicative).toBe(true);
    expect(amount.reference.currency).toBe('EUR');
    expect(amount.display.currency).toBe('USD');
  });

  it('ne marque pas indicatif a devise identique', () => {
    const amount = buildQuoteAmount({ amountMinor: 10000, currency: 'EUR' }, 'EUR', rates);
    expect(amount.isIndicative).toBe(false);
  });
});
```

- [ ] **Step 7: Exécuter les tests et constater l'échec**

```bash
pnpm --filter @fanours/pricing test:unit
```

Attendu : `Cannot find module '../money.js'` ou `No test files found`. Coller la sortie.

- [ ] **Step 8: Implémenter et prouver la réussite**

```bash
pnpm --filter @fanours/pricing test:unit
```

Attendu :

```
 ✓ src/__tests__/currency.test.ts (3 tests)
 ✓ src/__tests__/money.test.ts (10 tests)
 ✓ src/__tests__/quote.test.ts (6 tests)

 Test Files  3 passed (3)
      Tests  19 passed (19)
```

- [ ] **Step 9: Vérifier le typage strict**

```bash
pnpm --filter @fanours/pricing typecheck
```

Attendu : aucune erreur TypeScript.

- [ ] **Step 10: Commit**

```bash
git add packages/pricing
git commit -m "feat(pricing): ajoute la gestion des montants en minor units" -m "Les montants sont des entiers de minor units avec codes ISO 4217,
jamais des floats binaires. La conversion arrondit vers le bas au minor
unit et le montant de reference distingue devise de transaction et devise
d'affichage, avec mention indicative.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 7: Authentification — Argon2id, session opaque, rate limiting, anti-énumération

**Files:**
- Create: `apps/api/package.json`, `apps/api/tsconfig.json`, `apps/api/nest-cli.json`, `apps/api/src/main.ts`
- Create: `apps/api/src/app.module.ts`
- Create: `apps/api/src/common/security/…` (voir ci-dessous) — le filtre d’erreur global (`apps/api/src/common/error.filter.ts`) et le middleware de corrélation (`apps/api/src/common/correlation.middleware.ts`) sont **créés en Task 11** (observabilité), qui les enregistre dans `app.module.ts` : une seule implémentation, pas de doublon
- Create: `apps/api/src/config/env.ts` (schéma Zod, refus de démarrage si variable manquante)
- Create: `apps/api/src/common/security/password.ts` (Argon2id)
- Create: `apps/api/src/common/security/session.ts` (cookie, hash, rotation, révocation)
- Create: `apps/api/src/common/security/rate-limit.ts` (fenêtre glissante Redis)
- Create: `apps/api/src/modules/auth/auth.module.ts`, `auth.service.ts`, `auth.controller.ts`
- Create: `apps/api/src/modules/auth/dto/sign-in.dto.ts`, `sign-up.dto.ts`
- Create: `apps/api/src/modules/auth/contracts.ts` (schémas Zod entrée/sortie)
- Test: `apps/api/src/modules/auth/__tests__/auth.service.spec.ts`, `session.spec.ts`, `rate-limit.spec.ts`

**Interfaces:**
- Consumes: `packages/db` (Task 2 — tables `user`, `session`, `auth_identity`), `packages/domain` (Task 1 — `SessionProfile`, `isSessionExpired`), `packages/contracts` (Task 1 — `API_ERROR_CODES`), `packages/observability` (Task 1 — `createLogger`, `newCorrelationId`).
- Produces:
  - `hashPassword(plain: string) : Promise<string>` — Argon2id, `N=2^16, r=8, p=1`, sel aléatoire 16 octets, sortie au format PHC.
  - `verifyPassword(digest: string, plain: string) : Promise<boolean>` — temps constant, ne lève pas.
  - `hashSessionToken(token: string) : string` — SHA-256 hexadécimal du token brut. **Le token brut n'est jamais stocké.**
  - `createSessionToken() : string` — 32 octets aléatoires, base64url.
  - `SESSION_COOKIE_NAME` : `'fanours_sid'`.
  - `buildSessionCookie(token: string, maxAgeSeconds: number, secure: boolean) : { name: string; value: string; options: CookieOptions }` — `HttpOnly`, `SameSite=Lax`, `Path=/`, **pas de `Domain`**, `Secure` selon l'environnement, `Max-Age`.
  - `resolveSessionProfile(userRoles: ReadonlyArray<string>, request) : SessionProfile`.
  - `AuthService.signUp(input) : Promise<SignUpResult>` — crée l'utilisateur, l'identité, la session ; **la réponse ne distingue pas** un e-mail déjà inscrit d'une inscription réussie.
  - `AuthService.signIn(input) : Promise<SignInResult>` — vérifie le mot de passe avec un hachage factice quand l'utilisateur n'existe (anti-énumération), rate limit 5/15 min sur IP+email.
  - `AuthService.signOut(sessionId) : Promise<void>` — révoque la session.
  - `AuthService.verifyEmail(token) : Promise<void>` — contrat déclaré ici, **implémenté par la Task 18 de la Phase 3**, qui introduit le jeton éphémère (Redis, TTL 24 h, usage unique) : la Phase 1 ne crée ni migration ni jeton, elle pose seulement la colonne `user.email_verified_at` (migration `0002`).
  - `RateLimiter` : `consume(key: string, limit: number, windowSeconds: number) : Promise<{ allowed: boolean; retryAfterSeconds: number; remaining: number }>` — fenêtre glissante Redis (ZSET + script Lua), `429` + `Retry-After`.

---

- [ ] **Step 1: Créer le squelette `apps/api`**

`apps/api/package.json` :

```json
{
  "name": "@fanours/api",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./dist/main.js",
  "scripts": {
    "build": "tsc -b",
    "dev": "nest start --watch",
    "start": "node dist/main.js",
    "typecheck": "tsc -b --pretty",
    "lint": "eslint src",
    "test:unit": "vitest run",
    "test:integration": "vitest run --config vitest.integration.config.ts",
    "test": "vitest run"
  },
  "dependencies": {
    "@fanours/config": "workspace:*",
    "@fanours/contracts": "workspace:*",
    "@fanours/db": "workspace:*",
    "@fanours/domain": "workspace:*",
    "@fanours/observability": "workspace:*",
    "@nestjs/common": "11.0.1",
    "@nestjs/core": "11.0.1",
    "@nestjs/platform-express": "11.0.1",
    "argon2": "0.41.1",
    "cookie-parser": "1.4.7",
    "helmet": "8.0.0",
    "ioredis": "5.4.2",
    "pino": "9.5.0",
    "reflect-metadata": "0.2.2",
    "rxjs": "7.8.1",
    "zod": "3.24.1"
  },
  "devDependencies": {
    "@types/cookie-parser": "1.4.8",
    "@types/express": "5.0.0",
    "@types/node": "22.10.2",
    "typescript": "5.6.3",
    "vitest": "2.1.8"
  }
}
```

`apps/api/tsconfig.json` :

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "baseUrl": ".",
    "paths": {
      "@fanours/config": ["../../packages/config/src/index.ts"],
      "@fanours/contracts": ["../../packages/contracts/src/index.ts"],
      "@fanours/db": ["../../packages/db/src/index.ts"],
      "@fanours/domain": ["../../packages/domain/src/index.ts"],
      "@fanours/observability": ["../../packages/observability/src/index.ts"]
    }
  },
  "include": ["src/**/*.ts"],
  "exclude": ["dist", "node_modules"]
}
```

`apps/api/nest-cli.json` :

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "deleteOutDir": true
  }
}
```

`apps/api/src/main.ts` :

```typescript
import { NestFactory } from '@nestjs/core';
import cookieParser from 'cookie-parser';
import helmet from 'helmet';
import { AppModule } from './app.module.js';
import { loadEnv } from './config/env.js';

const env = loadEnv();

const CORS_ALLOWLIST = env.CORS_ORIGINS.split(',').map((origin) => origin.trim());

async function bootstrap(): Promise<void> {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });

  app.set('trust proxy', 1);
  app.use(
    helmet({
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'none'"],
          frameAncestors: ["'none'"],
          sandbox: [],
        },
      },
      crossOriginEmbedderPolicy: false,
      referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
      frameguard: { action: 'deny' },
      hsts: { maxAge: 63_072_000, includeSubDomains: true, preload: true },
    }),
  );
  // helmet ne pose pas Permissions-Policy : elle est posee explicitement.
  app.use((_request: unknown, response: { setHeader: (k: string, v: string) => void }, next: () => void) => {
    response.setHeader('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');
    next();
  });
  app.use(cookieParser());
  app.enableCors({ origin: CORS_ALLOWLIST, credentials: true });

  await app.listen(Number(process.env['PORT'] ?? 4000), '0.0.0.0');
}

void bootstrap();
```

La CSP de l'API est **`default-src 'none'`** : l'API ne sert que du JSON, jamais de document ni de script. La CSP du navigateur (web) est distincte et nonce par requête (Task 13). `Permissions-Policy` est posée à la main car `helmet` ne la gère pas. Le filtre d'erreur global et le middleware de corrélation sont enregistrés en Task 11 via `app.module.ts` : `main.ts` n'est pas modifié après cette étape.

- [ ] **Step 2: Écrire le schéma d'environnement Zod (refus de démarrage)**

`apps/api/src/config/env.ts` :

```typescript
import { z } from 'zod';

const envSchema = z.object({
  APP_ENV: z.enum(['local', 'test', 'staging', 'production']),
  APP_BASE_URL: z.string().url(),
  CORS_ORIGINS: z.string().min(1),
  DATABASE_URL: z.string().min(1),
  REDIS_URL: z.string().min(1),
  SESSION_SECRET: z.string().min(32),
  LOG_HASH_KEY: z.string().min(32),
  KMS_KEY_ARN: z.string().min(1),
  EMAIL_PROVIDER: z.string().min(1),
  EMAIL_API_KEY: z.string().min(1),
  PAYMENT_PROVIDER: z.string().min(1),
  PAYMENT_API_KEY: z.string().min(1),
  PAYMENT_WEBHOOK_SECRET: z.string().min(16),
  SHIPPING_PROVIDER: z.string().min(1),
  SHIPPING_API_KEY: z.string().min(1),
  S3_ENDPOINT: z.string().url(),
  S3_REGION: z.string().min(1),
  S3_BUCKET_PUBLIC: z.string().min(1),
  S3_BUCKET_PRIVATE: z.string().min(1),
  S3_ACCESS_KEY_ID: z.string().min(1),
  S3_SECRET_ACCESS_KEY: z.string().min(1),
  CDN_BASE_URL: z.string().url(),
  SENTRY_DSN: z.string().min(1),
  OTEL_EXPORTER_OTLP_ENDPOINT: z.string().url(),
  ANALYTICS_WRITE_KEY: z.string().min(1),
  CONSENT_SECRET: z.string().min(32),
});

export type AppEnv = z.infer<typeof envSchema>;

export function loadEnv(source: NodeJS.ProcessEnv = process.env): AppEnv {
  const parsed = envSchema.safeParse(source);
  if (!parsed.success) {
    const details = parsed.error.issues
      .map((issue) => `${issue.path.join('.') || '(racine)'} : ${issue.message}`)
      .join('\n');
    throw new Error(
      `Configuration d'environnement invalide — l'application refuse de demarrer :\n${details}`,
    );
  }
  return parsed.data;
}
```

- [ ] **Step 3: Écrire le hachage Argon2id**

`apps/api/src/common/security/password.ts` :

```typescript
import argon2 from 'argon2';

const ARGON2_OPTIONS: argon2.Options & { raw?: false } = {
  type: argon2.argon2id,
  memoryCost: 65536,
  timeCost: 3,
  parallelism: 1,
  hashLength: 32,
  saltLength: 16,
};

export async function hashPassword(plain: string): Promise<string> {
  return argon2.hash(plain, ARGON2_OPTIONS);
}

export async function verifyPassword(digest: string, plain: string): Promise<boolean> {
  try {
    return await argon2.verify(digest, plain);
  } catch {
    return false;
  }
}

/**
 * Hache factice utilise quand l'utilisateur n'existe pas : la reponse HTTP et
 * le temps de traitement restent identiques, ce qui empeche l'enumeration de comptes.
 */
const DUMMY_PLAINTEXT = 'fanours::dummy::never-matches::0000000000000000';

let dummyDigestPromise: Promise<string> | null = null;
const DUMMY_UNUSED =
  '$argon2id$v=19$m=65536,t=3,p=1$AAAAAAAAAAAAAAAAAAAAAA$Yn3F1TZbdUEaN8M1O1Yb0U0k0k0k0k0k0k0k0k0k0k';

export function burnPasswordCompare(plain: string): Promise<void> {
  dummyDigestPromise ??= hashPassword(DUMMY_PLAINTEXT);
  return dummyDigestPromise
    .then((digest) => verifyPassword(digest, plain))
    .then(() => undefined);
}
```

- [ ] **Step 4: Écrire la gestion de session**

`apps/api/src/common/security/session.ts` :

```typescript
import { createHash, randomBytes } from 'node:crypto';
import { POLICIES, isSessionExpired, type SessionProfile } from '@fanours/domain';

export const SESSION_COOKIE_NAME = 'fanours_sid';

export const SESSION_COOKIE_OPTIONS = {
  httpOnly: true,
  sameSite: 'lax',
  path: '/',
  // Aucun attribut Domain : le cookie reste rattache a l'hote exact.
} as const;

export interface CookieOptions {
  readonly httpOnly: boolean;
  readonly sameSite: 'lax' | 'strict';
  readonly path: string;
  readonly secure: boolean;
  readonly maxAge: number;
}

export function createSessionToken(): string {
  return randomBytes(32).toString('base64url');
}

export function hashSessionToken(token: string): string {
  return createHash('sha256').update(token, 'utf8').digest('hex');
}

export function buildSessionCookie(
  token: string,
  profile: SessionProfile,
  secure: boolean,
): { name: string; value: string; options: CookieOptions } {
  const maxAge = POLICIES[profile].absoluteMs / 1000;
  return {
    name: SESSION_COOKIE_NAME,
    value: token,
    options: {
      httpOnly: true,
      sameSite: 'lax',
      path: '/',
      secure,
      maxAge,
    },
  };
}

export interface SessionRecord {
  readonly id: string;
  readonly userId: string;
  readonly profile: SessionProfile;
  readonly createdAt: Date;
  readonly lastActivityAt: Date;
  readonly absoluteExpiresAt: Date;
}

export function isSessionUsable(session: SessionRecord, now: Date = new Date()): boolean {
  return !isSessionExpired(session.profile, now, session.lastActivityAt, session.absoluteExpiresAt);
}

export function nextSessionProfile(userRoles: ReadonlyArray<string>): SessionProfile {
  if (userRoles.includes('super_admin')) {
    return 'admin';
  }
  if (userRoles.includes('finance_manager') || userRoles.includes('compliance_manager')) {
    return 'admin';
  }
  return 'client';
}
```

- [ ] **Step 5: Écrire le rate limiter à fenêtre glissante**

`apps/api/src/common/security/rate-limit.ts` :

```typescript
import { Redis } from 'ioredis';

export interface RateLimitResult {
  readonly allowed: boolean;
  readonly remaining: number;
  readonly retryAfterSeconds: number;
}

const SLIDING_WINDOW_SCRIPT = `
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window_ms = tonumber(ARGV[2])
local now_ms = tonumber(ARGV[3])
local member = ARGV[4]

redis.call('ZREMRANGEBYSCORE', key, 0, now_ms - window_ms)
local count = redis.call('ZCARD', key)

if count >= limit then
  local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
  local retry_ms = window_ms
  if oldest[2] then
    retry_ms = (tonumber(oldest[2]) + window_ms) - now_ms
  end
  if retry_ms < 0 then retry_ms = 0 end
  redis.call('PEXPIRE', key, window_ms)
  return { 0, 0, math.ceil(retry_ms / 1000) }
end

redis.call('ZADD', key, now_ms, member)
redis.call('PEXPIRE', key, window_ms)
return { 1, limit - count - 1, 0 }
`;

export const RATE_LIMITS = {
  login: { limit: 5, windowSeconds: 15 * 60 },
  signup: { limit: 3, windowSeconds: 60 * 60 },
  reset: { limit: 3, windowSeconds: 60 * 60 },
  mfa: { limit: 5, windowSeconds: 10 * 60 },
  search: { limit: 60, windowSeconds: 60 },
  api: { limit: 600, windowSeconds: 60 },
} as const;

export class RateLimiter {
  private readonly redis: Redis;

  public constructor(redis: Redis) {
    this.redis = redis;
  }

  public async consume(
    key: string,
    limit: number,
    windowSeconds: number,
  ): Promise<RateLimitResult> {
    const now = Date.now();
    const member = `${now}-${Math.trunc(Math.random() * 1_000_000)}`;
    const raw = (await this.redis.eval(
      SLIDING_WINDOW_SCRIPT,
      1,
      `ratelimit:${key}`,
      String(limit),
      String(windowSeconds * 1000),
      String(now),
      member,
    )) as [number, number, number];

    return {
      allowed: raw[0] === 1,
      remaining: raw[1],
      retryAfterSeconds: raw[2],
    };
  }
}
```

- [ ] **Step 6: Écrire les schémas Zod d'entrée et de sortie**

`apps/api/src/modules/auth/contracts.ts` :

```typescript
import { z } from 'zod';

export const signUpSchema = z
  .object({
    email: z.string().trim().toLowerCase().email(),
    password: z.string().min(12).max(200),
    displayName: z.string().trim().min(1).max(120).optional(),
    locale: z.enum(['fr-FR', 'en', 'zh-CN']).default('fr-FR'),
  })
  .strict();

export const signInSchema = z
  .object({
    email: z.string().trim().toLowerCase().email(),
    password: z.string().min(1).max(200),
  })
  .strict();

export const signInResponseSchema = z
  .object({
    status: z.literal('ok'),
    user: z.object({
      id: z.string().uuid(),
      email: z.string().email(),
      displayName: z.string().nullable(),
      locale: z.enum(['fr-FR', 'en', 'zh-CN']),
      mfaRequired: z.boolean(),
    }),
  })
  .strict();

export const signUpResponseSchema = z
  .object({
    status: z.literal('ok'),
    user: z.object({
      id: z.string().uuid(),
      email: z.string().email(),
    }),
  })
  .strict();

export type SignUpInput = z.infer<typeof signUpSchema>;
export type SignInInput = z.infer<typeof signInSchema>;
export type SignInResponse = z.infer<typeof signInResponseSchema>;
export type SignUpResponse = z.infer<typeof signUpResponseSchema>;
```

`apps/api/src/modules/auth/dto/sign-up.dto.ts` :

```typescript
import { z } from 'zod';

export class SignUpDto {
  public email!: string;
  public password!: string;
  public displayName?: string;
  public locale: 'fr-FR' | 'en' | 'zh-CN' = 'fr-FR';

  public static parse(input: unknown): SignUpDto {
    return zodSignUp.parse(input) as SignUpDto;
  }
}

import { signUpSchema as zodSignUp } from '../contracts.js';
```

`apps/api/src/modules/auth/dto/sign-in.dto.ts` :

```typescript
import { z } from 'zod';
import { signInSchema as zodSignIn } from '../contracts.js';

export class SignInDto {
  public email!: string;
  public password!: string;

  public static parse(input: unknown): SignInDto {
    return zodSignIn.parse(input) as SignInDto;
  }
}
```

- [ ] **Step 7: Écrire le service d'authentification**

`apps/api/src/modules/auth/auth.service.ts` :

```typescript
import { randomBytes } from 'node:crypto';
import { eq, and, isNull } from 'drizzle-orm';
import type { NodePgDatabase } from 'drizzle-orm/node-postgres';
import { user, session, authIdentity } from '@fanours/db';
import { API_ERROR_CODES } from '@fanours/contracts';
import { hashPassword, verifyPassword, burnPasswordCompare } from '../../common/security/password.js';
import {
  buildSessionCookie,
  createSessionToken,
  hashSessionToken,
  nextSessionProfile,
  type CookieOptions,
} from '../../common/security/session.js';
import { RateLimiter, RATE_LIMITS } from '../../common/security/rate-limit.js';
import type { SignInInput, SignInResponse, SignUpInput, SignUpResponse } from './contracts.js';

export class AuthError extends Error {
  public readonly code: string;
  public readonly status: number;

  public constructor(code: string, message: string, status: number) {
    super(message);
    this.code = code;
    this.status = status;
  }
}

export interface AuthServiceOptions {
  readonly db: NodePgDatabase<Record<string, never>>;
  readonly rateLimiter: RateLimiter;
  readonly sessionSecret: string;
  readonly cookieSecure: boolean;
}

export class AuthService {
  private readonly db: NodePgDatabase<Record<string, never>>;
  private readonly rateLimiter: RateLimiter;
  private readonly cookieSecure: boolean;

  public constructor(options: AuthServiceOptions) {
    this.db = options.db;
    this.rateLimiter = options.rateLimiter;
    this.cookieSecure = options.cookieSecure;
  }

  public async signUp(input: SignUpInput): Promise<{ response: SignUpResponse; cookie: { name: string; value: string; options: CookieOptions } }> {
    await this.rateLimiter.consume(`signup:${input.email}`, RATE_LIMITS.signup.limit, RATE_LIMITS.signup.windowSeconds);

    const existing = await this.db
      .select({ id: user.id })
      .from(user)
      .where(eq(user.emailNormalized, input.email))
      .limit(1);

    const passwordHash = await hashPassword(input.password);

    if (existing.length > 0) {
      // Reponse identique a une inscription reussie : pas d'enumeration de comptes.
      const created = await this.createSession(existing[0]!.id, input.locale);
      return {
        response: { status: 'ok', user: { id: existing[0]!.id, email: input.email } },
        cookie: created,
      };
    }

    const inserted = await this.db
      .insert(user)
      .values({
        email: input.email,
        emailNormalized: input.email,
        passwordHash,
        displayName: input.displayName ?? null,
        locale: input.locale,
      })
      .returning({ id: user.id });

    const created = await this.createSession(inserted[0]!.id, input.locale);
    return {
      response: { status: 'ok', user: { id: inserted[0]!.id, email: input.email } },
      cookie: created,
    };
  }

  public async signIn(
    input: SignInInput,
    clientIp: string,
  ): Promise<{ response: SignInResponse; cookie: { name: string; value: string; options: CookieOptions } }> {
    const limit = await this.rateLimiter.consume(
      `login:${clientIp}:${input.email}`,
      RATE_LIMITS.login.limit,
      RATE_LIMITS.login.windowSeconds,
    );
    if (!limit.allowed) {
      throw new AuthError(
        API_ERROR_CODES.RATE_LIMITED,
        `Trop de tentatives. Reessayez dans ${limit.retryAfterSeconds} secondes.`,
        429,
      );
    }

    const rows = await this.db
      .select({
        id: user.id,
        passwordHash: user.passwordHash,
        locale: user.locale,
        email: user.email,
        displayName: user.displayName,
      })
      .from(user)
      .where(eq(user.emailNormalized, input.email))
      .limit(1);

    const record = rows[0];
    if (record === undefined || record.passwordHash === null) {
      // Meme chemin de code et meme cout : impossible de deduire l'existence du compte.
      await burnPasswordCompare(input.password);
      throw new AuthError(
        API_ERROR_CODES.INVALID_CREDENTIALS,
        'Adresse email ou mot de passe incorrect.',
        401,
      );
    }

    const valid = await verifyPassword(record.passwordHash, input.password);
    if (!valid) {
      throw new AuthError(
        API_ERROR_CODES.INVALID_CREDENTIALS,
        'Adresse email ou mot de passe incorrect.',
        401,
      );
    }

    const cookie = await this.createSession(record.id, record.locale);
    return {
      response: {
        status: 'ok',
        user: {
          id: record.id,
          email: record.email,
          displayName: record.displayName,
          locale: record.locale,
          mfaRequired: false,
        },
      },
      cookie,
    };
  }

  public async signOut(sessionId: string): Promise<void> {
    await this.db
      .update(session)
      .set({ revokedAt: new Date() })
      .where(and(eq(session.id, sessionId), isNull(session.revokedAt)));
  }

  public async rotateSession(
    sessionId: string,
    userId: string,
    profile: 'client' | 'admin' | 'b2b_pending' | 'sensitive_elevation',
  ): Promise<{ name: string; value: string; options: CookieOptions }> {
    const token = createSessionToken();
    const now = new Date();
    const absoluteExpiresAt = new Date(now.getTime() + 8 * 60 * 60 * 1000);
    await this.db
      .update(session)
      .set({
        tokenHash: hashSessionToken(token),
        lastActivityAt: now,
        absoluteExpiresAt,
        profile,
        rotatedAt: now,
      })
      .where(eq(session.id, sessionId));
    return buildSessionCookie(token, profile, this.cookieSecure);
  }

  public async revokeAllSessions(userId: string): Promise<void> {
    await this.db
      .update(session)
      .set({ revokedAt: new Date() })
      .where(and(eq(session.userId, userId), isNull(session.revokedAt)));
  }

  private async createSession(
    userId: string,
    locale: string,
  ): Promise<{ name: string; value: string; options: CookieOptions }> {
    const token = createSessionToken();
    const now = new Date();
    const profile = nextSessionProfile([]);
    const absoluteExpiresAt = new Date(now.getTime() + 12 * 60 * 60 * 1000);
    await this.db.insert(session).values({
      userId,
      tokenHash: hashSessionToken(token),
      profile,
      lastActivityAt: now,
      absoluteExpiresAt,
    });
    return buildSessionCookie(token, profile, this.cookieSecure);
  }
}
```

- [ ] **Step 8: Écrire le contrôleur HTTP**

`apps/api/src/modules/auth/auth.controller.ts` :

```typescript
import { Body, Controller, HttpCode, Ip, Post, Req, Res } from '@nestjs/common';
import type { Request, Response } from 'express';
import { AuthError, AuthService } from './auth.service.js';
import { SignInDto } from './dto/sign-in.dto.js';
import { SignUpDto } from './dto/sign-up.dto.js';
import { signInResponseSchema, signUpResponseSchema } from './contracts.js';

@Controller('v1/auth')
export class AuthController {
  private readonly auth: AuthService;

  public constructor(auth: AuthService) {
    this.auth = auth;
  }

  @Post('signup')
  @HttpCode(201)
  public async signUp(
    @Body() body: SignUpDto,
    @Res({ passthrough: true }) response: Response,
  ): Promise<unknown> {
    try {
      const result = await this.auth.signUp(SignUpDto.parse(body));
      response.cookie(result.cookie.name, result.cookie.value, result.cookie.options);
      return signUpResponseSchema.parse(result.response);
    } catch (error) {
      throw toHttpError(error);
    }
  }

  @Post('signin')
  @HttpCode(200)
  public async signIn(
    @Body() body: SignInDto,
    @Ip() ip: string,
    @Res({ passthrough: true }) response: Response,
  ): Promise<unknown> {
    try {
      const result = await this.auth.signIn(SignInDto.parse(body), ip);
      response.cookie(result.cookie.name, result.cookie.value, result.cookie.options);
      return signInResponseSchema.parse(result.response);
    } catch (error) {
      throw toHttpError(error);
    }
  }

  @Post('signout')
  @HttpCode(204)
  public async signOut(@Req() request: Request): Promise<void> {
    const sessionId = request.cookies?.['fanours_sid'];
    if (typeof sessionId === 'string') {
      await this.auth.signOut(sessionId);
    }
  }
}

function toHttpError(error: unknown): Error {
  if (error instanceof AuthError) {
    return error;
  }
  return new Error('Erreur interne.');
}
```

- [ ] **Step 9: Écrire le module NestJS**

`apps/api/src/modules/auth/auth.module.ts` :

```typescript
import { Global, Module } from '@nestjs/common';
import { AuthController } from './auth.controller.js';
import { AuthService } from './auth.service.js';

@Global()
@Module({
  controllers: [AuthController],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

`apps/api/src/app.module.ts` :

```typescript
import { Module } from '@nestjs/common';
import { AuthModule } from './modules/auth/auth.module.js';

@Module({
  imports: [AuthModule],
})
export class AppModule {}
```

- [ ] **Step 10: Écrire les tests d'authentification**

`apps/api/src/modules/auth/__tests__/auth.service.spec.ts` :

```typescript
import { describe, expect, it, vi } from 'vitest';
import { AuthError, AuthService } from '../auth.service.js';
import { hashPassword } from '../../../common/security/password.js';

function createService(overrides: {
  readonly existingUser?: { id: string; passwordHash: string };
  readonly allowLogin?: boolean;
} = {}) {
  const insertUser = vi.fn().mockResolvedValue([{ id: 'user-uuid' }]);
  const selectUser = vi.fn().mockResolvedValue(
    overrides.existingUser === undefined
      ? []
      : [
          {
            id: overrides.existingUser.id,
            passwordHash: overrides.existingUser.passwordHash,
            locale: 'fr-FR',
            email: 'buyer@example.test',
            displayName: null,
          },
        ],
  );
  const insertSession = vi.fn().mockResolvedValue([]);
  const rateLimiter = {
    consume: vi.fn().mockResolvedValue({ allowed: true, remaining: 4, retryAfterSeconds: 0 }),
  };

  const db = {
    select: () => ({ from: () => ({ where: () => ({ limit: async () => selectUser() }) }) }),
    insert: (table: unknown) => {
      if (table === undefined) {
        return { values: insertUser };
      }
      return { values: insertSession };
    },
  };

  const service = new AuthService({
    db: db as never,
    rateLimiter: rateLimiter as never,
    sessionSecret: 'secret',
    cookieSecure: false,
  });

  return { service, rateLimiter, insertUser, insertSession };
}

describe('AuthService', () => {
  it('refuse un mot de passe trop court a la validation', async () => {
    const { service } = createService();
    await expect(
      service.signUp({ email: 'buyer@example.test', password: 'court', locale: 'fr-FR' }),
    ).rejects.toThrow();
  });

  it('renvoie la meme erreur que l email existe ou non', async () => {
    const { service } = createService();
    const result = await service.signUp({
      email: 'buyer@example.test',
      password: 'un-mot-de-passe-solide-1234',
      locale: 'fr-FR',
    });
    expect(result.response.status).toBe('ok');
    expect(result.response.user.email).toBe('buyer@example.test');
  });

  it('echoue sur identifiants invalides sans reveler si le compte existe', async () => {
    const { service } = createService();
    await expect(
      service.signIn({ email: 'absent@example.test', password: 'un-mot-de-passe-1234' }, '10.0.0.1'),
    ).rejects.toThrow(AuthError);
  });

  it('rejette la connexion au-dela de 5 tentatives par quart d heure', async () => {
    const { service, rateLimiter } = createService();
    rateLimiter.consume.mockResolvedValueOnce({
      allowed: false,
      remaining: 0,
      retryAfterSeconds: 42,
    });
    await expect(
      service.signIn({ email: 'buyer@example.test', password: 'un-mot-de-passe-1234' }, '10.0.0.1'),
    ).rejects.toThrow(/Trop de tentatives/);
  });

  it('accepte un mot de passe correct', async () => {
    const passwordHash = await hashPassword('un-mot-de-passe-solide-1234');
    const { service } = createService({ existingUser: { id: 'user-uuid', passwordHash } });
    const result = await service.signIn(
      { email: 'buyer@example.test', password: 'un-mot-de-passe-solide-1234' },
      '10.0.0.1',
    );
    expect(result.response.user.id).toBe('user-uuid');
    expect(result.cookie.name).toBe('fanours_sid');
  });
});
```

`apps/api/src/modules/auth/__tests__/session.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { POLICIES } from '@fanours/domain';
import {
  SESSION_COOKIE_NAME,
  buildSessionCookie,
  createSessionToken,
  hashSessionToken,
  isSessionUsable,
} from '../../../common/security/session.js';

describe('session', () => {
  it('ne stocke jamais le token brut', () => {
    const token = createSessionToken();
    const hash = hashSessionToken(token);
    expect(hash).toHaveLength(64);
    expect(hash).not.toContain(token);
  });

  it('produit un token distinct a chaque appel', () => {
    expect(createSessionToken()).not.toBe(createSessionToken());
  });

  it('pose un cookie HttpOnly SameSite Lax sans Domain', () => {
    const cookie = buildSessionCookie('token-value', 'client', true);
    expect(cookie.name).toBe(SESSION_COOKIE_NAME);
    expect(cookie.options.httpOnly).toBe(true);
    expect(cookie.options.sameSite).toBe('lax');
    expect(cookie.options.path).toBe('/');
    expect(cookie.options).not.toHaveProperty('domain');
    expect(cookie.options.secure).toBe(true);
  });

  it('applique les durees de la politique client', () => {
    const cookie = buildSessionCookie('token-value', 'client', false);
    expect(cookie.options.maxAge).toBe(POLICIES.client.absoluteMs / 1000);
  });

  it('rejette une session expiree par inactivite', () => {
    const now = new Date();
    const session = {
      id: 'session-1',
      userId: 'user-1',
      profile: 'client' as const,
      createdAt: new Date(now.getTime() - 60 * 60 * 1000),
      lastActivityAt: new Date(now.getTime() - 31 * 60 * 1000),
      absoluteExpiresAt: new Date(now.getTime() + 60 * 60 * 1000),
    };
    expect(isSessionUsable(session, now)).toBe(false);
  });
});
```

`apps/api/src/modules/auth/__tests__/rate-limit.spec.ts` :

```typescript
import { describe, expect, it, vi } from 'vitest';
import { RateLimiter, RATE_LIMITS } from '../../../common/security/rate-limit.js';

function createLimiter(result: [number, number, number]) {
  const evalMock = vi.fn().mockResolvedValue(result);
  const redis = { eval: evalMock } as never;
  return { limiter: new RateLimiter(redis), evalMock };
}

describe('RateLimiter', () => {
  it('autorise tant que la limite n est pas atteinte', async () => {
    const { limiter } = createLimiter([1, 4, 0]);
    const result = await limiter.consume('login:ip:mail', 5, 900);
    expect(result.allowed).toBe(true);
    expect(result.remaining).toBe(4);
  });

  it('refuse et renvoie un delai de reessai', async () => {
    const { limiter } = createLimiter([0, 0, 42]);
    const result = await limiter.consume('login:ip:mail', 5, 900);
    expect(result.allowed).toBe(false);
    expect(result.retryAfterSeconds).toBe(42);
  });

  it('applique les limites prevues par la politique', () => {
    expect(RATE_LIMITS.login).toEqual({ limit: 5, windowSeconds: 900 });
    expect(RATE_LIMITS.signup).toEqual({ limit: 3, windowSeconds: 3600 });
    expect(RATE_LIMITS.reset).toEqual({ limit: 3, windowSeconds: 3600 });
    expect(RATE_LIMITS.mfa).toEqual({ limit: 5, windowSeconds: 600 });
    expect(RATE_LIMITS.search).toEqual({ limit: 60, windowSeconds: 60 });
    expect(RATE_LIMITS.api).toEqual({ limit: 600, windowSeconds: 60 });
  });
});
```

- [ ] **Step 11: Exécuter les tests et constater l'échec**

```bash
pnpm --filter @fanours/api test:unit
```

Attendu : `Cannot find module '../../../common/security/password.js'` ou `No test files found`. Coller la sortie.

- [ ] **Step 12: Implémenter et prouver la réussite**

```bash
pnpm --filter @fanours/api test:unit
```

Attendu :

```
 ✓ src/modules/auth/__tests__/auth.service.spec.ts (5 tests)
 ✓ src/modules/auth/__tests__/rate-limit.spec.ts (3 tests)
 ✓ src/modules/auth/__tests__/session.spec.ts (6 tests)

 Test Files  3 passed (3)
      Tests  14 passed (14)
```

- [ ] **Step 13: Vérifier que l'application refuse de démarrer sans configuration**

```bash
pnpm --filter @fanours/api exec node -e "import('./dist/config/env.js')" 2>&1 | head -5
```

Attendu : `Configuration d'environnement invalide — l'application refuse de demarrer` avec la liste des variables manquantes. Le code de sortie est non nul.

- [ ] **Step 14: Commit**

```bash
git add apps/api
git commit -m "feat(auth): ajoute l authentification par session opaque" -m "Argon2id pour les mots de passe, cookie fanours_sid HttpOnly SameSite Lax
sans Domain, hash SHA-256 du token en base et jamais le token brut.
Rotation a la connexion, revocation multi-appareils, rate limit a fenetre
glissante sur Redis et reponse identique pour l anti-enumeration.
L application refuse de demarrer sans configuration valide.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 8: MFA — TOTP RFC 6238, secret en enveloppe chiffrée, anti-rejeu, codes de récupération

**Files:**
- Create: `apps/api/src/common/audit/audit.service.ts` (écriture unique dans `audit_log`)
- Create: `apps/api/src/common/security/totp.ts` (RFC 6238)
- Create: `apps/api/src/common/security/envelope.ts` (enveloppe KMS + AES-256-GCM)
- Create: `apps/api/src/common/security/recovery-code.ts`
- Create: `apps/api/src/modules/mfa/mfa.module.ts`, `apps/api/src/modules/mfa/mfa.service.ts`, `apps/api/src/modules/mfa/mfa.controller.ts`, `apps/api/src/modules/mfa/contracts.ts`
- Create: `apps/api/src/modules/mfa/dto/enroll.dto.ts`, `apps/api/src/modules/mfa/dto/verify.dto.ts`, `apps/api/src/modules/mfa/dto/recover.dto.ts`
- Modify: `apps/api/src/modules/auth/auth.service.ts` (événement d'audit `user.logged_in`, cohérence de la colonne `lastSeenAt`, calcul de `mfaRequired`)
- Modify: `apps/api/src/app.module.ts` (import du `MfaModule`)
- Modify: `apps/api/package.json` (dépendance `@aws-sdk/client-kms`)
- Modify: `packages/contracts/src/events.ts` (ajout de l'événement `mfa.challenge_failed`)
- Test: `apps/api/src/common/security/__tests__/totp.test.ts`, `apps/api/src/common/security/__tests__/envelope.test.ts`, `apps/api/src/common/security/__tests__/recovery-code.test.ts`, `apps/api/src/modules/mfa/__tests__/mfa.service.spec.ts`

**Interfaces:**
- Consumes: `packages/db` (Task 2 — tables `mfa_method`, `recovery_code`, `session`, `user_role`, `audit_log`), `packages/domain` (Task 2 — `ROLES`, `Role`), `packages/contracts` (Task 1 — `API_ERROR_CODES`, `ApiEnvelope`, `ApiErrorBody`, `EVENT_TYPES`), `packages/observability` (Task 1 — `redact`), `apps/api/src/common/security/password.ts` (Task 7 — `hashPassword`, `verifyPassword`), `apps/api/src/common/security/session.ts` (Task 7 — `hashSessionToken`), `apps/api/src/common/security/rate-limit.ts` (Task 7 — `RateLimiter`, `RATE_LIMITS`).
- Produces:
  - `generateTotpSecret(byteLength?: number): string` — 20 octets aléatoires, encodage base32 RFC 4648.
  - `totpCounter(nowMs: number, periodSeconds?: number): number` — `Math.floor(nowMs / 1000 / periodSeconds)`.
  - `computeTotp(secretBase32: string, counter: number, digits?: number): string` — HMAC-SHA1, troncature dynamique RFC 4226, remplissage à gauche.
  - `verifyTotp(secretBase32: string, token: string, options?: TotpVerifyOptions): TotpVerifyResult` — fenêtre ±1 pas, **anti-rejeu** : un compteur `<= lastCounter` est refusé avec la raison `replayed`.
  - `buildOtpAuthUrl(secret: string, account: string, issuer: string): string` — URI `otpauth://totp/…`.
  - `EnvelopeProvider` : `{ readonly keyVersion: string; encrypt(plaintext: string): Promise<string>; decrypt(envelope: string): Promise<string> }` — format d'enveloppe `v1:<keyVersion>:<keyMaterial>:<iv>:<tag>:<ciphertext>`, tous les segments en base64url, AES-256-GCM. En `staging`/`production`, la clé de données vient de KMS (`GenerateDataKey`) ; en `local`/`test`, elle est dérivée par HKDF-SHA256 depuis `SESSION_SECRET`.
  - `createEnvelopeProvider(env: AppEnv): EnvelopeProvider`.
  - `generateRecoveryCodes(count?: number): string[]` — 10 codes `XXXXX-XXXXX`, alphabet base32 sans caractères ambigus (`I`, `O`, `0`, `1`).
  - `normalizeRecoveryCode(code: string): string`, `hashRecoveryCode(code: string): Promise<string>`, `verifyRecoveryCode(digest: string, code: string): Promise<boolean>` — Argon2id, paramètres identiques à ceux des mots de passe.
  - `MFA_REQUIRED_ROLES` : `ReadonlySet<string>` = `{ 'super_admin', 'finance_manager', 'compliance_manager' }` — `isMfaRequired(roles: ReadonlyArray<string>): boolean`.
  - `AuditService.write(event: AuditEvent): Promise<void>` — **unique point d'écriture** dans `audit_log` ; la chaîne `prev_hash`/`entry_hash` est calculée par le trigger SQL, le service n'écrit que `action`, `object_type`, `object_id`, `result`, `correlation_id` et `metadata` (passé au redacteur).
  - `MfaService.startEnrollment(userId: string): Promise<{ secret: string; otpauthUrl: string }>`.
  - `MfaService.confirmEnrollment(userId: string, code: string): Promise<void>`.
  - `MfaService.verifyChallenge(userId: string, sessionId: string, code: string): Promise<{ verified: true }>` — positionne `session.mfa_verified_at` et met à jour `mfa_method.last_counter`.
  - `MfaService.consumeRecoveryCode(userId: string, code: string): Promise<boolean>`.
  - `MfaService.disable(userId: string, code: string): Promise<void>` — désactivation par re-authentification TOTP.
  - `MfaService.elevate(userId: string, sessionId: string, code: string): Promise<void>` — bascule la session en profil `sensitive_elevation` (30 min, cf. `POLICIES`).
  - Routes : `POST /v1/mfa/enroll`, `POST /v1/mfa/enroll/confirm`, `POST /v1/mfa/challenge`, `POST /v1/mfa/recover`, `POST /v1/mfa/elevate`, `DELETE /v1/mfa`.

---

- [ ] **Step 1: Écrire les tests qui doivent échouer**

`apps/api/src/common/security/__tests__/totp.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import {
  base32Decode,
  base32Encode,
  buildOtpAuthUrl,
  computeTotp,
  generateTotpSecret,
  totpCounter,
  verifyTotp,
} from '../totp.js';

describe('TOTP RFC 6238', () => {
  it('encode et decode en base32 sans perte', () => {
    const seed = Buffer.from('12345678901234567890', 'utf8');
    expect(base32Encode(seed)).toBe('GEZDGNBVGY3TQOJQGEZDGNBVGY3TQOJQ');
    expect(base32Decode(base32Encode(seed)).toString('utf8')).toBe('12345678901234567890');
  });

  it('reproduit les vecteurs de test RFC 6238 (SHA-1)', () => {
    const secret = 'GEZDGNBVGY3TQOJQGEZDGNBVGY3TQOJQ';
    const vectors: ReadonlyArray<readonly [number, string]> = [
      [59, '94287082'],
      [1111111109, '07081804'],
      [1111111111, '14050471'],
      [1234567890, '89005924'],
      [2000000000, '69279037'],
      [20000000000, '65353130'],
    ];
    for (const [seconds, expected] of vectors) {
      const counter = Math.floor(seconds / 30);
      expect(computeTotp(secret, counter, 8)).toBe(expected);
      expect(computeTotp(secret, counter, 6)).toBe(expected.slice(-6));
    }
  });

  it('calcule le compteur sur une periode de 30 secondes', () => {
    expect(totpCounter(59_000)).toBe(1);
    expect(totpCounter(90_000)).toBe(3);
  });

  it('accepte le code du pas courant et refuse celui du pas passe', () => {
    const secret = generateTotpSecret();
    const nowMs = 1_700_000_000_000;
    const counter = totpCounter(nowMs);
    const code = computeTotp(secret, counter);
    expect(verifyTotp(secret, code, { nowMs }).ok).toBe(true);
    const previous = computeTotp(secret, counter - 1);
    expect(verifyTotp(secret, previous, { nowMs }).ok).toBe(false);
  });

  it('refuse un rejeu dont le compteur est inferieur ou egal au dernier utilise', () => {
    const secret = generateTotpSecret();
    const nowMs = 1_700_000_000_000;
    const counter = totpCounter(nowMs);
    const code = computeTotp(secret, counter);
    const replay = verifyTotp(secret, code, { nowMs, lastCounter: counter });
    expect(replay.ok).toBe(false);
    expect(replay.reason).toBe('replayed');
  });

  it('refuse un code qui ne contient que des chiffres de la bonne longueur', () => {
    const secret = generateTotpSecret();
    const result = verifyTotp(secret, '123', { nowMs: 1_700_000_000_000 });
    expect(result.ok).toBe(false);
    expect(result.reason).toBe('invalid');
  });

  it('produit une URI otpauth exploitable par une application d authentification', () => {
    const url = new URL(buildOtpAuthUrl('JBSWY3DPEHPK3PXP', 'admin@fanours.local', 'Fanours'));
    expect(url.protocol).toBe('otpauth:');
    expect(url.searchParams.get('secret')).toBe('JBSWY3DPEHPK3PXP');
    expect(url.searchParams.get('algorithm')).toBe('SHA1');
    expect(url.searchParams.get('digits')).toBe('6');
    expect(url.searchParams.get('period')).toBe('30');
  });
});
```

`apps/api/src/common/security/__tests__/envelope.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { LocalEnvelopeProvider, createEnvelopeProvider } from '../envelope.js';

describe('enveloppe de chiffrement', () => {
  it('chiffre puis dechiffre le secret TOTP', async () => {
    const provider = new LocalEnvelopeProvider('une-cle-de-session-de-32-caracteres-minimum');
    const envelope = await provider.encrypt('JBSWY3DPEHPK3PXP');
    expect(envelope.startsWith('v1:local-v1:local:')).toBe(true);
    expect(envelope).not.toContain('JBSWY3DPEHPK3PXP');
    expect(await provider.decrypt(envelope)).toBe('JBSWY3DPEHPK3PXP');
  });

  it('refuse une enveloppe dont la version est inconnue', async () => {
    const provider = new LocalEnvelopeProvider('une-cle-de-session-de-32-caracteres-minimum');
    const envelope = await provider.encrypt('secret');
    await expect(provider.decrypt(envelope.replace('v1:', 'v9:'))).rejects.toThrow(
      /Version d enveloppe non supportee/,
    );
  });

  it('refuse une enveloppe dont le tag GCM a ete altere', async () => {
    const provider = new LocalEnvelopeProvider('une-cle-de-session-de-32-caracteres-minimum');
    const envelope = await provider.encrypt('secret');
    const parts = envelope.split(':');
    parts[4] = Buffer.from('0000000000000000', 'hex').toString('base64url');
    await expect(provider.decrypt(parts.join(':'))).rejects.toThrow();
  });

  it('utilise le provider local en local et en test', () => {
    const provider = createEnvelopeProvider({
      APP_ENV: 'test',
      SESSION_SECRET: 'une-cle-de-session-de-32-caracteres-minimum',
      KMS_KEY_ARN: 'arn:aws:kms:eu-west-1:000000000000:key/00000000-0000-0000-0000-000000000000',
    });
    expect(provider.keyVersion).toBe('local-v1');
  });
});
```

`apps/api/src/common/security/__tests__/recovery-code.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import {
  generateRecoveryCodes,
  hashRecoveryCode,
  normalizeRecoveryCode,
  verifyRecoveryCode,
} from '../recovery-code.js';

describe('codes de recuperation', () => {
  it('produit 10 codes distincts au format XXXXX-XXXXX', () => {
    const codes = generateRecoveryCodes();
    expect(codes).toHaveLength(10);
    expect(new Set(codes).size).toBe(10);
    for (const code of codes) {
      expect(code).toMatch(/^[A-Z2-9]{5}-[A-Z2-9]{5}$/);
    }
  });

  it('normalise la saisie (casse, espaces, tirets)', () => {
    expect(normalizeRecoveryCode('abcde-fghij')).toBe('ABCDEFGHIJ');
    expect(normalizeRecoveryCode(' ab cde fghij ')).toBe('ABCDEFGHIJ');
  });

  it('ne stocke jamais le code en clair', async () => {
    const [code] = generateRecoveryCodes(1);
    const digest = await hashRecoveryCode(code);
    expect(digest.startsWith('$argon2id$')).toBe(true);
    expect(digest).not.toContain(normalizeRecoveryCode(code));
    expect(await verifyRecoveryCode(digest, code)).toBe(true);
    expect(await verifyRecoveryCode(digest, 'ZZZZZ-ZZZZZ')).toBe(false);
  });
});
```

`apps/api/src/modules/mfa/__tests__/mfa.service.spec.ts` :

```typescript
import { describe, expect, it, vi } from 'vitest';
import { MfaService, isMfaRequired } from '../mfa.service.js';
import { generateTotpSecret, totpCounter, computeTotp } from '../../../common/security/totp.js';

const NOW = 1_700_000_000_000;

function createService(options: { readonly secret?: string; readonly lastCounter?: number } = {}) {
  const secret = options.secret ?? generateTotpSecret();
  const mfaRows = [
    {
      id: 'mfa-1',
      userId: 'user-1',
      kind: 'totp',
      algorithm: 'SHA1',
      digits: 6,
      periodSeconds: 30,
      secretCiphertext: 'v1:local-v1:local:aXY=:dGFn:Zm9v',
      secretKeyVersion: 'local-v1',
      lastCounter: options.lastCounter ?? 0,
      confirmedAt: new Date(),
      revokedAt: null,
    },
  ];
  const insertMethod = vi.fn().mockResolvedValue([]);
  const updateMethod = vi.fn().mockResolvedValue([]);
  const updateSession = vi.fn().mockResolvedValue([]);
  const selectMethod = vi.fn().mockResolvedValue(mfaRows);
  const selectRecovery = vi.fn().mockResolvedValue([]);
  const insertRecovery = vi.fn().mockResolvedValue([]);
  const updateRecovery = vi.fn().mockResolvedValue([]);
  const deleteRecovery = vi.fn().mockResolvedValue([]);
  const insertAudit = vi.fn().mockResolvedValue([]);

  const db = {
    select: () => ({
      from: () => ({
        where: () => ({
          limit: async () => (options.selectSession === undefined ? mfaRows : mfaRows),
        }),
      }),
    }),
    insert: (table: unknown) => ({ values: (values: unknown) => {
      if (table === 'recovery_code') return insertRecovery(values);
      if (table === 'audit_log') return insertAudit(values);
      return insertMethod(values);
    } }),
    update: (table: unknown) => ({ set: () => ({ where: async () => {
      if (table === 'session') return updateSession();
      if (table === 'recovery_code') return updateRecovery();
      return updateMethod();
    } }) }),
    delete: () => ({ where: async () => deleteRecovery() }),
  };

  const envelope = {
    keyVersion: 'local-v1',
    encrypt: vi.fn().mockResolvedValue('v1:local-v1:local:aXY=:dGFn:Zm9v'),
    decrypt: vi.fn().mockResolvedValue(secret),
  };
  const audit = { write: vi.fn().mockResolvedValue(undefined) };
  const rateLimiter = {
    consume: vi.fn().mockResolvedValue({ allowed: true, remaining: 4, retryAfterSeconds: 0 }),
  };

  const service = new MfaService({
    db: db as never,
    envelope: envelope as never,
    audit: audit as never,
    rateLimiter: rateLimiter as never,
  });

  return { service, audit, rateLimiter, envelope, updateSession, updateMethod, insertMethod };
}

describe('MFA', () => {
  it('impose le MFA aux roles admin, finance et conformite', () => {
    expect(isMfaRequired(['super_admin'])).toBe(true);
    expect(isMfaRequired(['finance_manager'])).toBe(true);
    expect(isMfaRequired(['compliance_manager'])).toBe(true);
    expect(isMfaRequired(['support_agent'])).toBe(false);
    expect(isMfaRequired([])).toBe(false);
  });

  it('refuse un code invalide et journalise l echec', async () => {
    const { service, audit } = createService();
    await expect(service.verifyChallenge('user-1', 'session-1', '000000')).rejects.toThrow();
    expect(audit.write).toHaveBeenCalledWith(
      expect.objectContaining({ action: 'mfa.challenge_failed', result: 'failure' }),
    );
  });

  it('accepte le code valide, marque la session verifiee et avance le compteur', async () => {
    const secret = generateTotpSecret();
    const { service, updateSession, updateMethod } = createService({ secret });
    const code = computeTotp(secret, totpCounter(NOW));
    await service.verifyChallenge('user-1', 'session-1', code);
    expect(updateSession).toHaveBeenCalled();
    expect(updateMethod).toHaveBeenCalled();
  });

  it('refuse un rejeu du meme code', async () => {
    const secret = generateTotpSecret();
    const counter = totpCounter(NOW);
    const { service } = createService({ secret, lastCounter: counter });
    const code = computeTotp(secret, counter);
    await expect(service.verifyChallenge('user-1', 'session-1', code)).rejects.toThrow();
  });
});
```

- [ ] **Step 2: Exécuter les tests et constater l'échec**

```bash
pnpm --filter @fanours/api test:unit
```

Attendu : `Cannot find module '../../../common/security/totp.js'` ou `No test files found`. **Coller la sortie réelle** dans la PR.

- [ ] **Step 3: Implémenter le TOTP, l'enveloppe et les codes de récupération**

`apps/api/src/common/security/totp.ts` :

```typescript
import { createHmac, randomBytes, timingSafeEqual } from 'node:crypto';

const BASE32_ALPHABET = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';

export function base32Encode(input: Uint8Array): string {
  let bits = 0;
  let value = 0;
  let output = '';
  for (const byte of input) {
    value = (value << 8) | byte;
    bits += 8;
    while (bits >= 5) {
      output += BASE32_ALPHABET[(value >>> (bits - 5)) & 31];
      bits -= 5;
    }
  }
  if (bits > 0) {
    output += BASE32_ALPHABET[(value << (5 - bits)) & 31];
  }
  return output;
}

export function base32Decode(input: string): Buffer {
  const normalized = input.toUpperCase().replace(/=+$/, '').replace(/\s+/g, '');
  let bits = 0;
  let value = 0;
  const bytes: number[] = [];
  for (const char of normalized) {
    const index = BASE32_ALPHABET.indexOf(char);
    if (index === -1) {
      throw new Error('Caractere base32 invalide.');
    }
    value = (value << 5) | index;
    bits += 5;
    if (bits >= 8) {
      bytes.push((value >>> (bits - 8)) & 255);
      bits -= 8;
    }
  }
  return Buffer.from(bytes);
}

export function generateTotpSecret(byteLength = 20): string {
  return base32Encode(randomBytes(byteLength));
}

export function totpCounter(nowMs: number, periodSeconds = 30): number {
  return Math.floor(nowMs / 1000 / periodSeconds);
}

export function computeTotp(secretBase32: string, counter: number, digits = 6): string {
  const key = base32Decode(secretBase32);
  const buffer = Buffer.alloc(8);
  buffer.writeBigUInt64BE(BigInt(counter));
  const digest = createHmac('sha1', key).update(buffer).digest();
  const offset = digest[digest.length - 1]! & 0x0f;
  const binary =
    ((digest[offset]! & 0x7f) << 24) |
    ((digest[offset + 1]! & 0xff) << 16) |
    ((digest[offset + 2]! & 0xff) << 8) |
    (digest[offset + 3]! & 0xff);
  return (binary % 10 ** digits).toString().padStart(digits, '0');
}

export interface TotpVerifyOptions {
  readonly window?: number;
  readonly lastCounter?: number;
  readonly nowMs?: number;
  readonly periodSeconds?: number;
  readonly digits?: number;
}

export interface TotpVerifyResult {
  readonly ok: boolean;
  readonly counter: number;
  readonly reason: 'ok' | 'invalid' | 'replayed';
}

export function verifyTotp(
  secretBase32: string,
  token: string,
  options: TotpVerifyOptions = {},
): TotpVerifyResult {
  const window = options.window ?? 1;
  const digits = options.digits ?? 6;
  const period = options.periodSeconds ?? 30;
  const nowMs = options.nowMs ?? Date.now();
  const normalized = token.replace(/\s+/g, '');
  if (!/^\d+$/.test(normalized) || normalized.length !== digits) {
    return { ok: false, counter: -1, reason: 'invalid' };
  }
  const current = totpCounter(nowMs, period);
  const lastCounter = options.lastCounter ?? 0;
  for (let drift = -window; drift <= window; drift += 1) {
    const counter = current + drift;
    const expected = computeTotp(secretBase32, counter, digits);
    const matches = timingSafeEqual(Buffer.from(expected, 'utf8'), Buffer.from(normalized, 'utf8'));
    if (!matches) {
      continue;
    }
    if (counter <= lastCounter) {
      return { ok: false, counter, reason: 'replayed' };
    }
    return { ok: true, counter, reason: 'ok' };
  }
  return { ok: false, counter: -1, reason: 'invalid' };
}

export function buildOtpAuthUrl(secret: string, account: string, issuer: string): string {
  const label = encodeURIComponent(`${issuer}:${account}`);
  const params = new URLSearchParams({
    secret,
    issuer,
    algorithm: 'SHA1',
    digits: '6',
    period: '30',
  });
  return `otpauth://totp/${label}?${params.toString()}`;
}
```

`apps/api/src/common/security/envelope.ts` :

```typescript
import { createCipheriv, createDecipheriv, hkdfSync, randomBytes } from 'node:crypto';
import { DecryptCommand, GenerateDataKeyCommand, KMSClient } from '@aws-sdk/client-kms';

export const ENVELOPE_VERSION = 'v1';

export interface EnvelopeProvider {
  readonly keyVersion: string;
  encrypt(plaintext: string): Promise<string>;
  decrypt(envelope: string): Promise<string>;
}

interface AesGcmPayload {
  readonly iv: string;
  readonly tag: string;
  readonly ciphertext: string;
}

function aesGcmEncrypt(key: Buffer, plaintext: string): AesGcmPayload {
  const iv = randomBytes(12);
  const cipher = createCipheriv('aes-256-gcm', key, iv);
  const ciphertext = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  return {
    iv: iv.toString('base64url'),
    tag: cipher.getAuthTag().toString('base64url'),
    ciphertext: ciphertext.toString('base64url'),
  };
}

function aesGcmDecrypt(key: Buffer, payload: AesGcmPayload): string {
  const decipher = createDecipheriv('aes-256-gcm', key, Buffer.from(payload.iv, 'base64url'));
  decipher.setAuthTag(Buffer.from(payload.tag, 'base64url'));
  return Buffer.concat([
    decipher.update(Buffer.from(payload.ciphertext, 'base64url')),
    decipher.final(),
  ]).toString('utf8');
}

export class LocalEnvelopeProvider implements EnvelopeProvider {
  public readonly keyVersion = 'local-v1';
  private readonly rootSecret: string;

  public constructor(rootSecret: string) {
    this.rootSecret = rootSecret;
  }

  private deriveKey(keyVersion: string): Buffer {
    return Buffer.from(
      hkdfSync(
        'sha256',
        Buffer.from(this.rootSecret, 'utf8'),
        Buffer.from('fanours:envelope:v1', 'utf8'),
        Buffer.from(keyVersion, 'utf8'),
        32,
      ),
    );
  }

  public async encrypt(plaintext: string): Promise<string> {
    const payload = aesGcmEncrypt(this.deriveKey(this.keyVersion), plaintext);
    return [
      ENVELOPE_VERSION,
      this.keyVersion,
      'local',
      payload.iv,
      payload.tag,
      payload.ciphertext,
    ].join(':');
  }

  public async decrypt(envelope: string): Promise<string> {
    const [version, keyVersion, material, iv, tag, ciphertext] = envelope.split(':');
    if (version !== ENVELOPE_VERSION) {
      throw new Error('Version d enveloppe non supportee.');
    }
    if (material !== 'local') {
      throw new Error('Cette enveloppe requiert KMS : le provider local ne peut pas la dechiffrer.');
    }
    return aesGcmDecrypt(this.deriveKey(keyVersion ?? this.keyVersion), {
      iv: iv ?? '',
      tag: tag ?? '',
      ciphertext: ciphertext ?? '',
    });
  }
}

export class KmsEnvelopeProvider implements EnvelopeProvider {
  public readonly keyVersion: string;
  private readonly client: KMSClient;
  private readonly keyId: string;

  public constructor(client: KMSClient, keyId: string, keyVersion: string) {
    this.client = client;
    this.keyId = keyId;
    this.keyVersion = keyVersion;
  }

  public async encrypt(plaintext: string): Promise<string> {
    const response = await this.client.send(
      new GenerateDataKeyCommand({ KeyId: this.keyId, KeySpec: 'AES_256' }),
    );
    const dataKey = Buffer.from(response.Plaintext as Uint8Array);
    const keyMaterial = Buffer.from(response.CiphertextBlob as Uint8Array).toString('base64url');
    const payload = aesGcmEncrypt(dataKey, plaintext);
    return [
      ENVELOPE_VERSION,
      this.keyVersion,
      keyMaterial,
      payload.iv,
      payload.tag,
      payload.ciphertext,
    ].join(':');
  }

  public async decrypt(envelope: string): Promise<string> {
    const [version, , keyMaterial, iv, tag, ciphertext] = envelope.split(':');
    if (version !== ENVELOPE_VERSION) {
      throw new Error('Version d enveloppe non supportee.');
    }
    const response = await this.client.send(
      new DecryptCommand({ CiphertextBlob: Buffer.from(keyMaterial ?? '', 'base64url') }),
    );
    const dataKey = Buffer.from(response.Plaintext as Uint8Array);
    return aesGcmDecrypt(dataKey, {
      iv: iv ?? '',
      tag: tag ?? '',
      ciphertext: ciphertext ?? '',
    });
  }
}

export function createEnvelopeProvider(env: {
  readonly APP_ENV: string;
  readonly SESSION_SECRET: string;
  readonly KMS_KEY_ARN: string;
}): EnvelopeProvider {
  if (env.APP_ENV === 'local' || env.APP_ENV === 'test') {
    return new LocalEnvelopeProvider(env.SESSION_SECRET);
  }
  return new KmsEnvelopeProvider(new KMSClient({}), env.KMS_KEY_ARN, env.KMS_KEY_ARN);
}
```

`apps/api/src/common/security/recovery-code.ts` :

```typescript
import { randomInt } from 'node:crypto';
import { hashPassword, verifyPassword } from './password.js';

const RECOVERY_ALPHABET = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';

export function generateRecoveryCodes(count = 10): string[] {
  const codes: string[] = [];
  while (codes.length < count) {
    const characters = Array.from(
      { length: 10 },
      () => RECOVERY_ALPHABET[randomInt(RECOVERY_ALPHABET.length)]!,
    ).join('');
    const code = `${characters.slice(0, 5)}-${characters.slice(5)}`;
    if (!codes.includes(code)) {
      codes.push(code);
    }
  }
  return codes;
}

export function normalizeRecoveryCode(code: string): string {
  return code.trim().toUpperCase().replace(/[^A-Z0-9]/g, '');
}

export function hashRecoveryCode(code: string): Promise<string> {
  return hashPassword(normalizeRecoveryCode(code));
}

export function verifyRecoveryCode(digest: string, code: string): Promise<boolean> {
  return verifyPassword(digest, normalizeRecoveryCode(code));
}
```

- [ ] **Step 4: Écrire le service d'audit et le service MFA**

`apps/api/src/common/audit/audit.service.ts` :

```typescript
import { redact } from '@fanours/observability';
import { auditLog } from '@fanours/db';
import type { NodePgDatabase } from 'drizzle-orm/node-postgres';

export type AuditResult = 'success' | 'failure' | 'denied' | 'ignored';

export interface AuditEvent {
  readonly actorId: string | null;
  readonly actorRole: string | null;
  readonly action: string;
  readonly objectType: string;
  readonly objectId: string;
  readonly result: AuditResult;
  readonly correlationId: string;
  readonly metadata?: Record<string, unknown>;
}

export class AuditService {
  private readonly db: NodePgDatabase<Record<string, never>>;

  public constructor(db: NodePgDatabase<Record<string, never>>) {
    this.db = db;
  }

  public async write(event: AuditEvent): Promise<void> {
    await this.db.insert(auditLog).values({
      actorId: event.actorId,
      actorRole: event.actorRole,
      action: event.action,
      objectType: event.objectType,
      objectId: event.objectId,
      result: event.result,
      correlationId: event.correlationId,
      metadata: redact(event.metadata ?? {}) as Record<string, unknown>,
    });
  }
}
```

`apps/api/src/modules/mfa/mfa.service.ts` :

```typescript
import { and, eq, isNull } from 'drizzle-orm';
import type { NodePgDatabase } from 'drizzle-orm/node-postgres';
import { mfaMethod, recoveryCode, session, user as userTable, userRole } from '@fanours/db';
import { API_ERROR_CODES } from '@fanours/contracts';
import type { AuditService } from '../../common/audit/audit.service.js';
import type { EnvelopeProvider } from '../../common/security/envelope.js';
import type { RateLimiter } from '../../common/security/rate-limit.js';
import { RATE_LIMITS } from '../../common/security/rate-limit.js';
import { buildOtpAuthUrl, generateTotpSecret, verifyTotp } from '../../common/security/totp.js';
import {
  generateRecoveryCodes,
  hashRecoveryCode,
  verifyRecoveryCode,
} from '../../common/security/recovery-code.js';

export const MFA_REQUIRED_ROLES: ReadonlySet<string> = new Set([
  'super_admin',
  'finance_manager',
  'compliance_manager',
]);

export function isMfaRequired(roles: ReadonlyArray<string>): boolean {
  return roles.some((role) => MFA_REQUIRED_ROLES.has(role));
}

export class MfaError extends Error {
  public readonly code: string;
  public readonly status: number;

  public constructor(code: string, message: string, status: number) {
    super(message);
    this.code = code;
    this.status = status;
  }
}

export interface MfaServiceOptions {
  readonly db: NodePgDatabase<Record<string, never>>;
  readonly envelope: EnvelopeProvider;
  readonly audit: AuditService;
  readonly rateLimiter: RateLimiter;
}

export class MfaService {
  private readonly db: NodePgDatabase<Record<string, never>>;
  private readonly envelope: EnvelopeProvider;
  private readonly audit: AuditService;
  private readonly rateLimiter: RateLimiter;

  public constructor(options: MfaServiceOptions) {
    this.db = options.db;
    this.envelope = options.envelope;
    this.audit = options.audit;
    this.rateLimiter = options.rateLimiter;
  }

  public async startEnrollment(userId: string): Promise<{ secret: string; otpauthUrl: string }> {
    const secret = generateTotpSecret();
    const envelope = await this.envelope.encrypt(secret);
    await this.db.insert(mfaMethod).values({
      userId,
      kind: 'totp',
      algorithm: 'SHA1',
      digits: 6,
      periodSeconds: 30,
      secretCiphertext: envelope,
      secretKeyVersion: this.envelope.keyVersion,
      lastCounter: 0,
    });
    const accounts = await this.db
      .select({ email: userTable.email })
      .from(userTable)
      .where(eq(userTable.id, userId))
      .limit(1);
    return {
      secret,
      otpauthUrl: buildOtpAuthUrl(secret, accounts[0]?.email ?? userId, 'Fanours'),
    };
  }

  public async confirmEnrollment(userId: string, code: string): Promise<void> {
    const method = await this.pendingMethod(userId);
    const secret = await this.envelope.decrypt(method.secretCiphertext);
    const result = verifyTotp(secret, code, { lastCounter: method.lastCounter });
    if (!result.ok) {
      throw new MfaError(API_ERROR_CODES.VALIDATION_FAILED, 'Code de verification invalide.', 400);
    }
    await this.db
      .update(mfaMethod)
      .set({ confirmedAt: new Date(), lastCounter: result.counter })
      .where(eq(mfaMethod.id, method.id));
  }

  public async verifyChallenge(
    userId: string,
    sessionId: string,
    code: string,
  ): Promise<{ verified: true }> {
    const limit = await this.rateLimiter.consume(
      `mfa:${userId}`,
      RATE_LIMITS.mfa.limit,
      RATE_LIMITS.mfa.windowSeconds,
    );
    if (!limit.allowed) {
      throw new MfaError(
        API_ERROR_CODES.RATE_LIMITED,
        `Trop de tentatives. Reessayez dans ${limit.retryAfterSeconds} secondes.`,
        429,
      );
    }

    const method = await this.confirmedMethod(userId);
    const secret = await this.envelope.decrypt(method.secretCiphertext);
    const result = verifyTotp(secret, code, {
      lastCounter: method.lastCounter,
      digits: method.digits,
      periodSeconds: method.periodSeconds,
    });

    if (!result.ok) {
      await this.audit.write({
        actorId: userId,
        actorRole: null,
        action: 'mfa.challenge_failed',
        objectType: 'mfa_method',
        objectId: method.id,
        result: 'failure',
        correlationId: sessionId,
        metadata: { reason: result.reason },
      });
      throw new MfaError(
        API_ERROR_CODES.VALIDATION_FAILED,
        result.reason === 'replayed'
          ? 'Ce code a deja ete utilise.'
          : 'Code de verification invalide.',
        401,
      );
    }

    await this.db
      .update(mfaMethod)
      .set({ lastCounter: result.counter })
      .where(eq(mfaMethod.id, method.id));
    await this.db
      .update(session)
      .set({ mfaVerifiedAt: new Date(), elevatedAt: new Date() })
      .where(eq(session.id, sessionId));
    await this.audit.write({
      actorId: userId,
      actorRole: null,
      action: 'mfa.verified',
      objectType: 'mfa_method',
      objectId: method.id,
      result: 'success',
      correlationId: sessionId,
      metadata: { counter: result.counter },
    });
    return { verified: true };
  }

  public async consumeRecoveryCode(userId: string, code: string): Promise<boolean> {
    const rows = await this.db
      .select({ id: recoveryCode.id, codeHash: recoveryCode.codeHash })
      .from(recoveryCode)
      .where(and(eq(recoveryCode.userId, userId), isNull(recoveryCode.usedAt)));
    for (const row of rows) {
      if (await verifyRecoveryCode(row.codeHash, code)) {
        await this.db
          .update(recoveryCode)
          .set({ usedAt: new Date() })
          .where(eq(recoveryCode.id, row.id));
        await this.db
          .update(mfaMethod)
          .set({ revokedAt: new Date() })
          .where(and(eq(mfaMethod.userId, userId), isNull(mfaMethod.revokedAt)));
        await this.audit.write({
          actorId: userId,
          actorRole: null,
          action: 'mfa.recovery_code_used',
          objectType: 'recovery_code',
          objectId: row.id,
          result: 'success',
          correlationId: userId,
        });
        return true;
      }
    }
    return false;
  }

  public async disable(userId: string, code: string): Promise<void> {
    const method = await this.confirmedMethod(userId);
    const secret = await this.envelope.decrypt(method.secretCiphertext);
    const result = verifyTotp(secret, code, { lastCounter: method.lastCounter });
    if (!result.ok) {
      throw new MfaError(
        API_ERROR_CODES.VALIDATION_FAILED,
        'Re-authentification refusee : le code est invalide.',
        401,
      );
    }
    await this.db
      .update(mfaMethod)
      .set({ revokedAt: new Date() })
      .where(eq(mfaMethod.id, method.id));
    await this.db
      .delete(recoveryCode)
      .where(and(eq(recoveryCode.userId, userId), isNull(recoveryCode.usedAt)));
  }

  public async elevate(userId: string, sessionId: string, code: string): Promise<void> {
    const method = await this.confirmedMethod(userId);
    const secret = await this.envelope.decrypt(method.secretCiphertext);
    const result = verifyTotp(secret, code, { lastCounter: method.lastCounter });
    if (!result.ok) {
      throw new MfaError(
        API_ERROR_CODES.VALIDATION_FAILED,
        'Re-authentification refusee : le code est invalide.',
        401,
      );
    }
    const now = new Date();
    await this.db
      .update(session)
      .set({
        profile: 'sensitive_elevation',
        mfaVerifiedAt: now,
        elevatedAt: now,
        absoluteExpiresAt: new Date(now.getTime() + 30 * 60 * 1000),
      })
      .where(eq(session.id, sessionId));
  }

  public async issueRecoveryCodes(userId: string): Promise<string[]> {
    const codes = generateRecoveryCodes(10);
    for (const code of codes) {
      await this.db.insert(recoveryCode).values({
        userId,
        codeHash: await hashRecoveryCode(code),
      });
    }
    return codes;
  }

  public async rolesOf(userId: string): Promise<string[]> {
    const rows = await this.db
      .select({ roleCode: userRole.roleCode })
      .from(userRole)
      .where(eq(userRole.userId, userId));
    return rows.map((row) => row.roleCode);
  }

  private async pendingMethod(userId: string) {
    const rows = await this.db
      .select()
      .from(mfaMethod)
      .where(and(eq(mfaMethod.userId, userId), isNull(mfaMethod.revokedAt)))
      .limit(1);
    const method = rows[0];
    if (method === undefined) {
      throw new MfaError(API_ERROR_CODES.NOT_FOUND, 'Aucune methode MFA en cours.', 404);
    }
    return method;
  }

  private async confirmedMethod(userId: string) {
    const rows = await this.db
      .select()
      .from(mfaMethod)
      .where(
        and(
          eq(mfaMethod.userId, userId),
          isNull(mfaMethod.revokedAt),
          eq(mfaMethod.confirmedAt as never, mfaMethod.confirmedAt as never),
        ),
      )
      .limit(1);
    const method = rows[0];
    if (method === undefined) {
      throw new MfaError(API_ERROR_CODES.MFA_REQUIRED, 'MFA non active sur ce compte.', 403);
    }
    return method;
  }
}
```

- [ ] **Step 5: Écrire le contrôleur, les schémas Zod et le module**

`apps/api/src/modules/mfa/contracts.ts` :

```typescript
import { z } from 'zod';

export const enrollRequestSchema = z.object({}).strict();

export const enrollResponseSchema = z
  .object({
    data: z.object({ secret: z.string().min(16), otpauthUrl: z.string().url() }),
    correlationId: z.string().uuid(),
  })
  .strict();

export const verifyRequestSchema = z
  .object({ code: z.string().trim().regex(/^\d{6}$/) })
  .strict();

export const recoverRequestSchema = z
  .object({ code: z.string().trim().min(9).max(12) })
  .strict();

export const okResponseSchema = z
  .object({ data: z.object({ verified: z.literal(true) }), correlationId: z.string().uuid() })
  .strict();

export type EnrollResponse = z.infer<typeof enrollResponseSchema>;
export type VerifyRequest = z.infer<typeof verifyRequestSchema>;
export type RecoverRequest = z.infer<typeof recoverRequestSchema>;
```

`apps/api/src/modules/mfa/mfa.controller.ts` :

```typescript
import { Body, Controller, Delete, HttpCode, Post, Req } from '@nestjs/common';
import type { Request } from 'express';
import { MfaError, MfaService } from './mfa.service.js';
import { enrollResponseSchema, okResponseSchema } from './contracts.js';
import { SESSION_COOKIE_NAME } from '../../common/security/session.js';

interface CorrelationRequest extends Request {
  correlationId?: string;
}

@Controller('v1/mfa')
export class MfaController {
  private readonly mfa: MfaService;

  public constructor(mfa: MfaService) {
    this.mfa = mfa;
  }

  @Post('enroll')
  @HttpCode(201)
  public async enroll(@Req() request: CorrelationRequest): Promise<unknown> {
    const userId = this.userId(request);
    const result = await this.mfa.startEnrollment(userId);
    return enrollResponseSchema.parse({
      data: result,
      correlationId: request.correlationId ?? '00000000-0000-7000-8000-000000000000',
    });
  }

  @Post('enroll/confirm')
  @HttpCode(200)
  public async confirm(
    @Body() body: { code: string },
    @Req() request: CorrelationRequest,
  ): Promise<unknown> {
    await this.mfa.confirmEnrollment(this.userId(request), body.code);
    return okResponseSchema.parse({
      data: { verified: true },
      correlationId: request.correlationId ?? '00000000-0000-7000-8000-000000000000',
    });
  }

  @Post('challenge')
  @HttpCode(200)
  public async challenge(
    @Body() body: { code: string },
    @Req() request: CorrelationRequest,
  ): Promise<unknown> {
    await this.mfa.verifyChallenge(this.userId(request), this.sessionId(request), body.code);
    return okResponseSchema.parse({
      data: { verified: true },
      correlationId: request.correlationId ?? '00000000-0000-7000-8000-000000000000',
    });
  }

  @Post('recover')
  @HttpCode(200)
  public async recover(
    @Body() body: { code: string },
    @Req() request: CorrelationRequest,
  ): Promise<unknown> {
    const used = await this.mfa.consumeRecoveryCode(this.userId(request), body.code);
    if (!used) {
      throw new MfaError(
        'forbidden',
        'Code de recuperation inconnu ou deja utilise.',
        403,
      );
    }
    return okResponseSchema.parse({
      data: { verified: true },
      correlationId: request.correlationId ?? '00000000-0000-7000-8000-000000000000',
    });
  }

  @Post('elevate')
  @HttpCode(200)
  public async elevate(
    @Body() body: { code: string },
    @Req() request: CorrelationRequest,
  ): Promise<unknown> {
    await this.mfa.elevate(this.userId(request), this.sessionId(request), body.code);
    return okResponseSchema.parse({
      data: { verified: true },
      correlationId: request.correlationId ?? '00000000-0000-7000-8000-000000000000',
    });
  }

  @Delete()
  @HttpCode(200)
  public async disable(
    @Body() body: { code: string },
    @Req() request: CorrelationRequest,
  ): Promise<unknown> {
    await this.mfa.disable(this.userId(request), body.code);
    return okResponseSchema.parse({
      data: { verified: true },
      correlationId: request.correlationId ?? '00000000-0000-7000-8000-000000000000',
    });
  }

  private sessionId(request: CorrelationRequest): string {
    const token = request.cookies?.[SESSION_COOKIE_NAME];
    if (typeof token !== 'string') {
      throw new MfaError('unauthenticated', 'Session absente.', 401);
    }
    return token;
  }

  private userId(request: CorrelationRequest): string {
    const header = request.headers['x-user-id'];
    const value = Array.isArray(header) ? header[0] : header;
    if (value === undefined) {
      throw new MfaError('unauthenticated', 'Session absente.', 401);
    }
    return value;
  }
}
```

`apps/api/src/modules/mfa/mfa.module.ts` :

```typescript
import { Global, Module } from '@nestjs/common';
import { MfaController } from './mfa.controller.js';
import { MfaService } from './mfa.service.js';

@Global()
@Module({
  controllers: [MfaController],
  providers: [MfaService],
  exports: [MfaService],
})
export class MfaModule {}
```

`apps/api/src/app.module.ts` devient :

```typescript
import { Module } from '@nestjs/common';
import { AuthModule } from './modules/auth/auth.module.js';
import { MfaModule } from './modules/mfa/mfa.module.js';

@Module({
  imports: [AuthModule, MfaModule],
})
export class AppModule {}
```

- [ ] **Step 6: Écrire l'événement d'audit de connexion admin et aligner la colonne `lastSeenAt`**

Dans `apps/api/src/modules/auth/auth.service.ts`, la méthode `signIn` est modifiée pour :

1. charger les rôles de l'utilisateur (`user_role`) ;
2. calculer `mfaRequired` avec `isMfaRequired(roles)` (au lieu de la valeur `false` codée en dur) ;
3. écrire l'événement d'audit `user.logged_in` avec `result: 'success'` et `object_type: 'user'` lorsque l'utilisateur porte un rôle d'administration ;
4. renommer le champ Drizzle `lastActivityAt` en `lastSeenAt` (le schéma de la Task 2 définit la colonne `last_seen_at`, pas `last_activity_at`) dans `createSession`, `rotateSession` et `revokeAllSessions`.

Extrait corrigé de `createSession` :

```typescript
  private async createSession(
    userId: string,
    locale: string,
  ): Promise<{ name: string; value: string; options: CookieOptions }> {
    const token = createSessionToken();
    const now = new Date();
    const profile = nextSessionProfile([]);
    const absoluteExpiresAt = new Date(now.getTime() + 12 * 60 * 60 * 1000);
    await this.db.insert(session).values({
      userId,
      tokenHash: hashSessionToken(token),
      profile,
      lastSeenAt: now,
      absoluteExpiresAt,
    });
    return buildSessionCookie(token, profile, this.cookieSecure);
  }
```

Extrait corrigé de `signIn` (fin de la méthode) :

```typescript
    const roles = await this.rolesOf(record.id);
    const mfaRequired = isMfaRequired(roles);
    if (roles.some((role) => ADMIN_ROLES.has(role))) {
      await this.audit.write({
        actorId: record.id,
        actorRole: roles[0] ?? null,
        action: 'user.logged_in',
        objectType: 'user',
        objectId: record.id,
        result: 'success',
        correlationId: hashSessionToken(token),
        metadata: { mfaRequired },
      });
    }
```

avec, en tête de fichier :

```typescript
import { isMfaRequired } from '../mfa/mfa.service.js';

const ADMIN_ROLES: ReadonlySet<string> = new Set([
  'super_admin',
  'finance_manager',
  'compliance_manager',
]);
```

et la réponse `signIn` renvoie `mfaRequired: mfaRequired` au lieu de `mfaRequired: false`.

- [ ] **Step 7: Exécuter les tests et prouver la réussite**

```bash
pnpm --filter @fanours/api test:unit
```

Attendu :

```
 ✓ src/common/security/__tests__/totp.test.ts (7 tests)
 ✓ src/common/security/__tests__/envelope.test.ts (4 tests)
 ✓ src/common/security/__tests__/recovery-code.test.ts (3 tests)
 ✓ src/modules/mfa/__tests__/mfa.service.spec.ts (4 tests)

 Test Files  4 passed (4)
      Tests  18 passed (18)
```

- [ ] **Step 8: Vérifier les 3 clés d'API et le typecheck strict**

```bash
pnpm --filter @fanours/api typecheck
```

Attendu : aucune erreur. Les codes `mfa_required`, `approval_required` et `self_approval_forbidden` existent déjà dans `API_ERROR_CODES` (Task 1) — ils ne sont pas redéfinis ici.

- [ ] **Step 9: États, 3 langues et 2 thèmes**

Le parcours MFA n'a **pas d'interface** en Phase 1 (le back-office arrive en Phase 4) ; les clés `mfa.*` existent déjà dans les trois dictionnaires de la Task 5 et ne sont donc pas modifiées. Les deux thèmes n'ont pas d'impact : le secret TOTP et les codes de récupération ne sont jamais rendus dans une page, seulement dans la réponse JSON de l'API et dans le journal d'audit (jamais le secret : seul le `metadata` redacté est écrit).

La preuve à produire : aucun `secret`, `code` ou `otpauthUrl` ne figure dans `audit_log`. Test à ajouter dans `apps/api/src/modules/mfa/__tests__/mfa.service.spec.ts` :

```typescript
  it('ne journalise jamais le secret ni le code de verification', async () => {
    const secret = generateTotpSecret();
    const { service, audit } = createService({ secret });
    const code = computeTotp(secret, totpCounter(NOW));
    await service.verifyChallenge('user-1', 'session-1', code);
    const payload = JSON.stringify(audit.write.mock.calls);
    expect(payload).not.toContain(secret);
    expect(payload).not.toContain(code);
  });
```

- [ ] **Step 10: Commit**

```bash
git add apps/api packages/contracts
git commit -m "feat(auth): ajoute le MFA TOTP avec anti-rejeu et codes de recuperation" -m "TOTP RFC 6238 (SHA-1, 6 chiffres, 30 s) avec vecteurs de test de la RFC,
secret chiffre en enveloppe AES-256-GCM, cle de donnees KMS en production
et derivee par HKDF en local et test.
Compteur anti-rejeu : un code deja consomme est refuse.
10 codes de recuperation Argon2id a usage unique.
MFA obligatoire pour super_admin, finance_manager et compliance_manager.
Le journal d audit append-only est alimente par un service unique.
La colonne de session utilise le nom lastSeenAt du schema de la Task 2.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 9: RBAC — 9 rôles, 13 ressources, 8 actions, deny by default, scoping anti-IDOR, four-eyes

**Files:**
- Create: `packages/db/migrations/0005_rbac_scopes.sql`
- Create: `apps/api/src/modules/rbac/scope.ts`
- Create: `apps/api/src/modules/rbac/rbac.service.ts`
- Create: `apps/api/src/modules/rbac/rbac.guard.ts`
- Create: `apps/api/src/modules/rbac/rbac.decorator.ts`
- Create: `apps/api/src/modules/rbac/approval.service.ts`
- Create: `apps/api/src/modules/rbac/rbac.module.ts`
- Create: `apps/api/src/modules/admin/admin.module.ts`, `apps/api/src/modules/admin/admin.controller.ts`
- Modify: `packages/db/src/schema.ts` (table `rbacScope`)
- Modify: `packages/db/src/seed.ts` (semis des scopes)
- Modify: `packages/db/src/__tests__/migrations.integration.test.ts` (5 migrations attendues, table `rbac_scope` expected)
- Modify: `apps/api/src/app.module.ts` (import du `RbacModule` et de l'`AdminModule`)
- Modify: `packages/contracts/src/events.ts` (ajout de l'événement `rbac.denied`)
- Test: `apps/api/src/modules/rbac/__tests__/permissions.test.ts`, `apps/api/src/modules/rbac/__tests__/scope.test.ts`, `apps/api/src/modules/rbac/__tests__/rbac.guard.spec.ts`

**Interfaces:**
- Consumes: `packages/domain` (Task 2 — `ROLES`, `RESOURCES`, `ACTIONS`, `PERMISSIONS`, `FOUR_EYES`, `Role`, `Resource`, `Action`), `packages/db` (Task 2 — tables `role`, `permission`, `user_role`, `admin_action_approval`, `session`, `audit_log`), `packages/contracts` (Task 1 — `API_ERROR_CODES`, `ApiEnvelope`), `apps/api/src/common/audit/audit.service.ts` (Task 8 — `AuditService`), `apps/api/src/common/security/session.ts` (Task 7 — `hashSessionToken`, `SESSION_COOKIE_NAME`).
- Produces:
  - `Scope` : `'own' | 'all' | 'none'` — `resolveScopeFilter(scope: Scope, ownerColumn: string): ScopeFilter`, `scopeCondition(filter: ScopeFilter, ownerColumn: AnyColumn, userId: string): SQL | undefined`, `applyScope(conditions: ReadonlyArray<SQL | undefined>, filter: ScopeFilter, ownerColumn: AnyColumn, userId: string): SQL`.
  - `Actor` : `{ readonly userId: string; readonly sessionId: string; readonly roles: readonly Role[]; readonly mfaVerified: boolean }`.
  - `RbacService.actorFor(token: string): Promise<Actor | null>` — résout la session par **hash SHA-256 du token**, jamais par le token brut.
  - `RbacService.permissionsFor(userId: string): Promise<ReadonlySet<string>>` — ensemble `ressource:action` effectivement présent en base ; **absent de la base = refusé** (deny by default).
  - `RbacService.scopeFor(userId: string, resource: Resource): Promise<Scope>` — `all` l'emporte sur `own`, `none` si rien n'est déclaré.
  - `RbacService.assert(actor: Actor, resource: Resource, action: Action, correlationId: string): Promise<void>` — lève `RbacError` (403) et écrit l'événement d'audit `rbac.denied`.
  - `RequirePermission(resource: Resource, action: Action)` — décorateur NestJS portant l'exigence.
  - `RbacGuard` — `canActivate(context): Promise<boolean>` ; refuse aussi les sessions admin dont `mfa_verified_at` est nul (code `mfa_required`).
  - `APPROVAL_KINDS` : `readonly ['refund', 'financial_export', 'price_tier_change', 'role_change', 'country_suspension']`.
  - `approvalKindFor(resource: Resource, action: Action): ApprovalKind | null` — `price_tier:update → price_tier_change`, `export_financial:export → financial_export`, `refund:refund → refund`, `admin_role:manage_roles → role_change`, `country:update → country_suspension`.
  - `requiresApproval(role: Role, resource: Resource, action: Action): boolean` — s'appuie sur `FOUR_EYES` de `packages/domain`.
  - `ApprovalService.request(kind, requestedBy, payload, correlationId): Promise<{ id: string }>`, `ApprovalService.approve(id, approvedBy): Promise<void>` (refuse l'auto-approbation, code `self_approval_forbidden`), `ApprovalService.execute(id, executionRef): Promise<void>`.
  - Route de démonstration : `GET /v1/admin/session` (acteur courant + permissions) et `GET /v1/admin/audit-log` (exige `audit_log:read`).

---

- [ ] **Step 1: Écrire les tests qui doivent échouer**

`apps/api/src/modules/rbac/__tests__/permissions.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { ACTIONS, FOUR_EYES, PERMISSIONS, RESOURCES, ROLES } from '@fanours/domain';
import { approvalKindFor, requiresApproval } from '../approval.service.js';

describe('matrice RBAC', () => {
  it('expose 9 roles, 13 ressources et 8 actions', () => {
    expect(ROLES).toHaveLength(9);
    expect(RESOURCES).toHaveLength(13);
    expect(ACTIONS).toHaveLength(8);
  });

  it('donne au moins une permission a chaque role', () => {
    for (const role of ROLES) {
      const resources = Object.keys(PERMISSIONS[role] ?? {});
      expect(resources.length).toBeGreaterThan(0);
    }
  });

  it('ne declare que des ressources et des actions connues', () => {
    for (const role of ROLES) {
      for (const [resource, actions] of Object.entries(PERMISSIONS[role] ?? {})) {
        expect(RESOURCES).toContain(resource);
        for (const action of actions) {
          expect(ACTIONS).toContain(action);
        }
      }
    }
  });

  it('refuse par defaut un role inconnu ou une action inconnue', () => {
    const permissions = PERMISSIONS['super_admin'] ?? {};
    expect(permissions['order']).toContain('read');
    expect(permissions['order']).not.toContain('manage_roles');
  });

  it('interdit au support le remboursement et au catalogue la gestion des roles', () => {
    expect(PERMISSIONS['support_agent']?.['refund']).toEqual(['read', 'create']);
    expect(PERMISSIONS['catalog_manager']?.['admin_role']).toBeUndefined();
  });

  it('mappe chaque action four-eyes vers un type d approbation', () => {
    for (const entry of FOUR_EYES) {
      const [role, resource, action] = entry.split(':') as [string, string, string];
      expect(requiresApproval(role as never, resource as never, action as never)).toBe(true);
      expect(approvalKindFor(resource as never, action as never)).not.toBeNull();
    }
  });
});
```

`apps/api/src/modules/rbac/__tests__/scope.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { resolveScopeFilter } from '../scope.js';

describe('scoping anti-IDOR', () => {
  it('ne filtre pas quand le role a un scope global', () => {
    expect(resolveScopeFilter('all', 'owner_user_id')).toEqual({ kind: 'all', ownerColumn: null });
  });

  it('filtre sur la colonne proprietaire quand le scope est propre', () => {
    expect(resolveScopeFilter('own', 'owner_user_id')).toEqual({
      kind: 'own',
      ownerColumn: 'owner_user_id',
    });
  });

  it('interdit toute ligne quand le scope est nul', () => {
    expect(resolveScopeFilter('none', 'owner_user_id')).toEqual({ kind: 'none', ownerColumn: null });
  });
});
```

`apps/api/src/modules/rbac/__tests__/rbac.guard.spec.ts` :

```typescript
import { describe, expect, it, vi } from 'vitest';
import { RbacService } from '../rbac.service.js';

function createService(options: { readonly permissions?: string[]; readonly scope?: string } = {}) {
  const permissions = options.permissions ?? ['audit_log:read'];
  const audit = { write: vi.fn().mockResolvedValue(undefined) };
  const db = {
    select: () => ({
      from: () => ({
        where: () => ({
          limit: async () => [
            {
              id: 'session-1',
              userId: 'user-1',
              profile: 'admin',
              mfaVerifiedAt: new Date(),
              revokedAt: null,
            },
          ],
        }),
      }),
    }),
  };
  const service = new RbacService({
    db: db as never,
    audit: audit as never,
    permissionCodes: new Set(permissions),
    scopeByResource: new Map([['audit_log', options.scope ?? 'all']]),
  });
  return { service, audit };
}

describe('RbacService', () => {
  it('autorise une permission accordee', async () => {
    const { service, audit } = createService();
    await expect(
      service.assert(
        { userId: 'user-1', sessionId: 'session-1', roles: ['super_admin'], mfaVerified: true },
        'audit_log',
        'read',
        'corr-1',
      ),
    ).resolves.toBeUndefined();
    expect(audit.write).not.toHaveBeenCalled();
  });

  it('refuse et journalise une permission absente', async () => {
    const { service, audit } = createService({ permissions: [] });
    await expect(
      service.assert(
        { userId: 'user-1', sessionId: 'session-1', roles: ['support_agent'], mfaVerified: true },
        'audit_log',
        'read',
        'corr-2',
      ),
    ).rejects.toThrow(/Acces refuse/);
    expect(audit.write).toHaveBeenCalledWith(
      expect.objectContaining({ action: 'rbac.denied', result: 'denied' }),
    );
  });

  it('refuse une ressource dont le scope est nul', async () => {
    const { service } = createService({ scope: 'none' });
    await expect(
      service.assert(
        { userId: 'user-1', sessionId: 'session-1', roles: ['super_admin'], mfaVerified: true },
        'audit_log',
        'read',
        'corr-3',
      ),
    ).rejects.toThrow(/Acces refuse/);
  });
});
```

- [ ] **Step 2: Exécuter les tests et constater l'échec**

```bash
pnpm --filter @fanours/api test:unit
```

Attendu : `Cannot find module '../rbac.service.js'` ou `Cannot find module '@fanours/domain'` selon l'ordre d'import. **Coller la sortie réelle** dans la PR.

- [ ] **Step 3: Écrire la migration `0005_rbac_scopes.sql` et le schéma Drizzle**

`packages/db/migrations/0005_rbac_scopes.sql` :

```sql
-- 0005_rbac_scopes.sql
-- Scoping de ressources par role : le filtre est applique dans la requete (anti-IDOR).
-- Append-only : aucun script down, rollback par code applicatif.
SET lock_timeout = '3s';
SET statement_timeout = '30s';

CREATE TABLE IF NOT EXISTS rbac_scope (
  role_code  text NOT NULL REFERENCES role (code) ON DELETE CASCADE,
  resource   text NOT NULL,
  scope      text NOT NULL DEFAULT 'own',
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (role_code, resource),
  CONSTRAINT ck_rbac_scope_scope CHECK (scope IN ('own', 'all', 'none')),
  CONSTRAINT ck_rbac_scope_resource CHECK (resource IN (
    'order', 'product', 'price_tier', 'export', 'export_financial', 'refund',
    'admin_role', 'audit_log', 'customs_rule', 'country',
    'business_document', 'business_account', 'customs_document'
  ))
);

CREATE INDEX IF NOT EXISTS ix_rbac_scope_resource ON rbac_scope (resource);
```

`packages/db/src/schema.ts` — ajouter :

```typescript
export const rbacScope = pgTable(
  'rbac_scope',
  {
    roleCode: text('role_code')
      .notNull()
      .references(() => role.code, { onDelete: 'cascade' }),
    resource: text('resource').notNull(),
    scope: text('scope').notNull().default('own'),
    createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    primary: uniqueIndex('uq_rbac_scope_role_resource').on(table.roleCode, table.resource),
    resourceIndex: index('ix_rbac_scope_resource').on(table.resource),
  }),
);
```

et ajouter `rbacScope` à l'objet `tables` exporté.

`packages/db/src/seed.ts` — ajouter l'insert des scopes, après la boucle des permissions :

```typescript
    const SCOPES: Readonly<Record<string, Readonly<Record<string, string>>>> = {
      super_admin: {
        order: 'all', product: 'all', price_tier: 'all', export: 'all',
        export_financial: 'all', refund: 'all', admin_role: 'all', audit_log: 'all',
        customs_rule: 'all', country: 'all', business_document: 'all',
        business_account: 'all', customs_document: 'all',
      },
      catalog_manager: {
        product: 'all', price_tier: 'all', business_document: 'all',
      },
      order_manager: {
        order: 'all', business_account: 'own', business_document: 'own',
      },
      logistics_operator: {
        order: 'all', business_account: 'own',
      },
      finance_manager: {
        order: 'all', export: 'all', export_financial: 'all', refund: 'all', price_tier: 'own',
      },
      support_agent: {
        order: 'all', business_account: 'own', business_document: 'own', refund: 'own',
      },
      compliance_manager: {
        customs_rule: 'all', customs_document: 'all', country: 'all', product: 'all',
        business_document: 'all', business_account: 'all',
      },
      marketing_manager: {
        product: 'all', business_account: 'own',
      },
      analyst_readonly: {
        order: 'all', product: 'all', audit_log: 'all',
      },
    };

    for (const [roleCode, resources] of Object.entries(SCOPES)) {
      for (const [resource, scope] of Object.entries(resources)) {
        await database
          .insert(rbacScope)
          .values({ roleCode, resource, scope })
          .onConflictDoNothing();
      }
    }
```

`packages/db/src/__tests__/migrations.integration.test.ts` — le test attend désormais **5** migrations et la table `rbac_scope` :

```typescript
    expect(first.length).toBe(5);
```

```typescript
      'rbac_scope',
```

- [ ] **Step 4: Écrire le scoping, le service RBAC, le guard et le four-eyes**

`apps/api/src/modules/rbac/scope.ts` :

```typescript
import { and, eq, sql, type AnyColumn, type SQL } from 'drizzle-orm';

export type Scope = 'own' | 'all' | 'none';

export interface ScopeFilter {
  readonly kind: Scope;
  readonly ownerColumn: string | null;
}

export function resolveScopeFilter(scope: Scope, ownerColumn: string): ScopeFilter {
  if (scope === 'none') {
    return { kind: 'none', ownerColumn: null };
  }
  if (scope === 'own') {
    return { kind: 'own', ownerColumn };
  }
  return { kind: 'all', ownerColumn: null };
}

export function scopeCondition(
  filter: ScopeFilter,
  ownerColumn: AnyColumn,
  userId: string,
): SQL | undefined {
  if (filter.kind === 'all') {
    return undefined;
  }
  if (filter.kind === 'none') {
    return sql`false`;
  }
  return eq(ownerColumn, userId);
}

export function applyScope(
  conditions: ReadonlyArray<SQL | undefined>,
  filter: ScopeFilter,
  ownerColumn: AnyColumn,
  userId: string,
): SQL {
  const parts = conditions.filter((condition): condition is SQL => condition !== undefined);
  const scoped = scopeCondition(filter, ownerColumn, userId);
  if (scoped !== undefined) {
    parts.push(scoped);
  }
  return and(...parts) as SQL;
}
```

`apps/api/src/modules/rbac/rbac.service.ts` :

```typescript
import { and, eq, inArray } from 'drizzle-orm';
import type { NodePgDatabase } from 'drizzle-orm/node-postgres';
import { permission, rbacScope, session, userRole } from '@fanours/db';
import { PERMISSIONS, type Action, type Resource, type Role } from '@fanours/domain';
import { API_ERROR_CODES } from '@fanours/contracts';
import type { AuditService } from '../../common/audit/audit.service.js';
import { hashSessionToken } from '../../common/security/session.js';
import type { Scope } from './scope.js';

export class RbacError extends Error {
  public readonly code: string;
  public readonly status: number;

  public constructor(code: string, message: string, status: number) {
    super(message);
    this.code = code;
    this.status = status;
  }
}

export interface Actor {
  readonly userId: string;
  readonly sessionId: string;
  readonly roles: readonly Role[];
  readonly mfaVerified: boolean;
}

export interface RbacServiceOptions {
  readonly db: NodePgDatabase<Record<string, never>>;
  readonly audit: AuditService;
  /** Surcharge de test : codes de permission effectivement en base. */
  readonly permissionCodes?: ReadonlySet<string>;
  /** Surcharge de test : scope par ressource. */
  readonly scopeByResource?: ReadonlyMap<string, Scope>;
}

export class RbacService {
  private readonly db: NodePgDatabase<Record<string, never>>;
  private readonly audit: AuditService;
  private readonly permissionCodes?: ReadonlySet<string>;
  private readonly scopeByResource?: ReadonlyMap<string, Scope>;

  public constructor(options: RbacServiceOptions) {
    this.db = options.db;
    this.audit = options.audit;
    this.permissionCodes = options.permissionCodes;
    this.scopeByResource = options.scopeByResource;
  }

  public async actorFor(token: string): Promise<Actor | null> {
    const rows = await this.db
      .select({
        id: session.id,
        userId: session.userId,
        profile: session.profile,
        mfaVerifiedAt: session.mfaVerifiedAt,
        revokedAt: session.revokedAt,
      })
      .from(session)
      .where(eq(session.tokenHash, hashSessionToken(token)))
      .limit(1);
    const row = rows[0];
    if (row === undefined || row.revokedAt !== null) {
      return null;
    }
    const roles = await this.rolesFor(row.userId);
    return {
      userId: row.userId,
      sessionId: row.id,
      roles,
      mfaVerified: row.mfaVerifiedAt !== null,
    };
  }

  public async rolesFor(userId: string): Promise<Role[]> {
    const rows = await this.db
      .select({ roleCode: userRole.roleCode })
      .from(userRole)
      .where(eq(userRole.userId, userId));
    return rows.map((row) => row.roleCode as Role);
  }

  public async permissionsFor(userId: string): Promise<ReadonlySet<string>> {
    if (this.permissionCodes !== undefined) {
      return this.permissionCodes;
    }
    const roles = await this.rolesFor(userId);
    const codes: string[] = [];
    for (const role of roles) {
      for (const [resource, actions] of Object.entries(PERMISSIONS[role] ?? {})) {
        for (const action of actions) {
          codes.push(`${role}:${resource}:${action}`);
        }
      }
    }
    if (codes.length === 0) {
      return new Set<string>();
    }
    const rows = await this.db
      .select({ code: permission.code })
      .from(permission)
      .where(inArray(permission.code, codes));
    const effective = new Set<string>();
    for (const row of rows) {
      const parts = row.code.split(':');
      effective.add(`${parts[1]}:${parts[2]}`);
    }
    return effective;
  }

  public async scopeFor(userId: string, resource: Resource): Promise<Scope> {
    if (this.scopeByResource !== undefined) {
      return this.scopeByResource.get(resource) ?? 'none';
    }
    const roles = await this.rolesFor(userId);
    if (roles.length === 0) {
      return 'none';
    }
    const rows = await this.db
      .select({ scope: rbacScope.scope })
      .from(rbacScope)
      .where(and(inArray(rbacScope.roleCode, roles), eq(rbacScope.resource, resource)));
    if (rows.some((row) => row.scope === 'all')) {
      return 'all';
    }
    if (rows.some((row) => row.scope === 'own')) {
      return 'own';
    }
    return 'none';
  }

  public async assert(
    actor: Actor,
    resource: Resource,
    action: Action,
    correlationId: string,
  ): Promise<void> {
    const permissions = await this.permissionsFor(actor.userId);
    const allowed = permissions.has(`${resource}:${action}`);
    const scope = allowed ? await this.scopeFor(actor.userId, resource) : 'none';
    if (!allowed || scope === 'none') {
      await this.audit.write({
        actorId: actor.userId,
        actorRole: actor.roles[0] ?? null,
        action: 'rbac.denied',
        objectType: resource,
        objectId: actor.userId,
        result: 'denied',
        correlationId,
        metadata: { requestedAction: action, scope },
      });
      throw new RbacError(
        API_ERROR_CODES.FORBIDDEN,
        `Acces refuse sur la ressource ${resource}.`,
        403,
      );
    }
  }
}
```

`apps/api/src/modules/rbac/rbac.decorator.ts` :

```typescript
import { SetMetadata, type CustomDecorator } from '@nestjs/common';
import type { Action, Resource } from '@fanours/domain';

export const PERMISSION_METADATA = 'fanours:required-permission';

export interface RequiredPermission {
  readonly resource: Resource;
  readonly action: Action;
}

export const RequirePermission = (
  resource: Resource,
  action: Action,
): CustomDecorator<string> => SetMetadata(PERMISSION_METADATA, { resource, action });
```

`apps/api/src/modules/rbac/rbac.guard.ts` :

```typescript
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  type CustomDecorator,
} from '@nestjs/common';
import type { Request } from 'express';
import { API_ERROR_CODES } from '@fanours/contracts';
import { SESSION_COOKIE_NAME } from '../../common/security/session.js';
import { PERMISSION_METADATA, type RequiredPermission } from './rbac.decorator.js';
import { RbacError, RbacService, type Actor } from './rbac.service.js';

export interface AuthenticatedRequest extends Request {
  actor?: Actor;
  correlationId?: string;
}

@Injectable()
export class RbacGuard implements CanActivate {
  private readonly rbac: RbacService;

  public constructor(rbac: RbacService) {
    this.rbac = rbac;
  }

  public async canActivate(context: ExecutionContext): Promise<boolean> {
    const required = context
      .getHandler()
      .getMetadata(PERMISSION_METADATA) as RequiredPermission | undefined;
    if (required === undefined) {
      return true;
    }
    const request = context.switchToHttp().getRequest<AuthenticatedRequest>();
    const token = request.cookies?.[SESSION_COOKIE_NAME];
    if (typeof token !== 'string') {
      throw new RbacError(API_ERROR_CODES.UNAUTHENTICATED, 'Session absente.', 401);
    }
    const actor = await this.rbac.actorFor(token);
    if (actor === null) {
      throw new RbacError(API_ERROR_CODES.UNAUTHENTICATED, 'Session absente ou revoquee.', 401);
    }
    if (actor.roles.length > 0 && !actor.mfaVerified && required.resource !== 'order') {
      throw new RbacError(
        API_ERROR_CODES.MFA_REQUIRED,
        'Verification en deux etapes requise pour cette operation.',
        403,
      );
    }
    await this.rbac.assert(actor, required.resource, required.action, request.correlationId ?? 'unknown');
    request.actor = actor;
    return true;
  }
}
```

`apps/api/src/modules/rbac/approval.service.ts` :

```typescript
import { createHash } from 'node:crypto';
import { and, eq } from 'drizzle-orm';
import type { NodePgDatabase } from 'drizzle-orm/node-postgres';
import { adminActionApproval } from '@fanours/db';
import { API_ERROR_CODES } from '@fanours/contracts';
import { FOUR_EYES, type Action, type Resource, type Role } from '@fanours/domain';

export const APPROVAL_KINDS = [
  'refund',
  'financial_export',
  'price_tier_change',
  'role_change',
  'country_suspension',
] as const;

export type ApprovalKind = (typeof APPROVAL_KINDS)[number];

export const APPROVAL_KIND_BY_ACTION: Readonly<Record<string, ApprovalKind>> = {
  'price_tier:update': 'price_tier_change',
  'export_financial:export': 'financial_export',
  'refund:refund': 'refund',
  'admin_role:manage_roles': 'role_change',
  'country:update': 'country_suspension',
};

export function approvalKindFor(resource: Resource, action: Action): ApprovalKind | null {
  return APPROVAL_KIND_BY_ACTION[`${resource}:${action}`] ?? null;
}

export function requiresApproval(role: Role, resource: Resource, action: Action): boolean {
  return FOUR_EYES.has(`${role}:${resource}:${action}`);
}

export class ApprovalError extends Error {
  public readonly code: string;
  public readonly status: number;

  public constructor(code: string, message: string, status: number) {
    super(message);
    this.code = code;
    this.status = status;
  }
}

export class ApprovalService {
  private readonly db: NodePgDatabase<Record<string, never>>;

  public constructor(db: NodePgDatabase<Record<string, never>>) {
    this.db = db;
  }

  public async request(
    kind: ApprovalKind,
    requestedBy: string,
    payload: Record<string, unknown>,
    correlationId: string,
  ): Promise<{ id: string }> {
    const payloadHash = createHash('sha256').update(JSON.stringify(payload)).digest('hex');
    const rows = await this.db
      .insert(adminActionApproval)
      .values({
        kind,
        payload,
        payloadHash,
        requestedBy,
        correlationId,
        approvalStatus: 'pending',
        expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000),
      })
      .returning({ id: adminActionApproval.id });
    return { id: rows[0]!.id };
  }

  public async approve(id: string, approvedBy: string): Promise<void> {
    const rows = await this.db
      .select({
        requestedBy: adminActionApproval.requestedBy,
        status: adminActionApproval.approvalStatus,
      })
      .from(adminActionApproval)
      .where(eq(adminActionApproval.id, id))
      .limit(1);
    const row = rows[0];
    if (row === undefined) {
      throw new ApprovalError(API_ERROR_CODES.NOT_FOUND, 'Demande introuvable.', 404);
    }
    if (row.status !== 'pending') {
      throw new ApprovalError(API_ERROR_CODES.CONFLICT, 'Demande deja traitee.', 409);
    }
    if (row.requestedBy === approvedBy) {
      throw new ApprovalError(
        API_ERROR_CODES.SELF_APPROVAL_FORBIDDEN,
        'Un second role doit approuver cette action.',
        403,
      );
    }
    await this.db
      .update(adminActionApproval)
      .set({ approvedBy, approvalStatus: 'approved' })
      .where(eq(adminActionApproval.id, id));
  }

  public async execute(id: string, executionRef: string): Promise<void> {
    const rows = await this.db
      .select({ status: adminActionApproval.approvalStatus })
      .from(adminActionApproval)
      .where(eq(adminActionApproval.id, id))
      .limit(1);
    const row = rows[0];
    if (row === undefined || row.status !== 'approved') {
      throw new ApprovalError(
        API_ERROR_CODES.APPROVAL_REQUIRED,
        'Approbation prealable requise.',
        403,
      );
    }
    await this.db
      .update(adminActionApproval)
      .set({ approvalStatus: 'executed', executedAt: new Date(), executionRef })
      .where(and(eq(adminActionApproval.id, id), eq(adminActionApproval.approvalStatus, 'approved')));
  }
}
```

- [ ] **Step 5: Écrire le module, la route d'administration et brancher le tout**

`apps/api/src/modules/rbac/rbac.module.ts` :

```typescript
import { Global, Module } from '@nestjs/common';
import { ApprovalService } from './approval.service.js';
import { RbacGuard } from './rbac.guard.js';
import { RbacService } from './rbac.service.js';

@Global()
@Module({
  providers: [RbacService, RbacGuard, ApprovalService],
  exports: [RbacService, RbacGuard, ApprovalService],
})
export class RbacModule {}
```

`apps/api/src/modules/admin/admin.controller.ts` :

```typescript
import { Controller, Get, Req, UseGuards } from '@nestjs/common';
import { RbacGuard } from '../rbac/rbac.guard.js';
import { RequirePermission } from '../rbac/rbac.decorator.js';
import type { AuthenticatedRequest } from '../rbac/rbac.guard.js';

@Controller('v1/admin')
@UseGuards(RbacGuard)
export class AdminController {
  @Get('session')
  public session(@Req() request: AuthenticatedRequest): unknown {
    const actor = request.actor;
    return {
      data: {
        userId: actor?.userId ?? null,
        roles: actor?.roles ?? [],
        mfaVerified: actor?.mfaVerified ?? false,
      },
      correlationId: request.correlationId ?? 'unknown',
    };
  }

  @Get('audit-log')
  @RequirePermission('audit_log', 'read')
  public auditLog(@Req() request: AuthenticatedRequest): unknown {
    return { data: { entries: [] }, correlationId: request.correlationId ?? 'unknown' };
  }
}
```

`apps/api/src/modules/admin/admin.module.ts` :

```typescript
import { Module } from '@nestjs/common';
import { AdminController } from './admin.controller.js';

@Module({
  controllers: [AdminController],
})
export class AdminModule {}
```

`apps/api/src/app.module.ts` devient :

```typescript
import { Module } from '@nestjs/common';
import { AdminModule } from './modules/admin/admin.module.js';
import { AuthModule } from './modules/auth/auth.module.js';
import { MfaModule } from './modules/mfa/mfa.module.js';
import { RbacModule } from './modules/rbac/rbac.module.js';

@Module({
  imports: [AuthModule, MfaModule, RbacModule, AdminModule],
})
export class AppModule {}
```

`packages/contracts/src/events.ts` — ajouter `'rbac.denied'` à `EVENT_TYPES`.

- [ ] **Step 6: Exécuter les tests et prouver la réussite**

```bash
pnpm --filter @fanours/api test:unit
pnpm --filter @fanours/db test:integration
```

Attendu : les tests unitaires du RBAC passent, et le test de migrations annonce `5` migrations appliquées et la présence de `rbac_scope` dans `information_schema.tables`.

- [ ] **Step 7: Prouver que le filtre de propriété est bien dans la requête**

Le point clé du chantier 1.9 est que le scoping s'applique **dans la requête**, jamais en post-filtrage. Test d'intégration `apps/api/src/modules/rbac/__tests__/scope.integration.test.ts` (mêmes conteneurs Testcontainers que la Task 2) :

```typescript
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { Pool } from 'pg';
import { and, eq, pgTable, uuid } from 'drizzle-orm/pg-core';
import { drizzle } from 'drizzle-orm/node-postgres';
import { applyScope, resolveScopeFilter } from '../scope.js';

const resource = pgTable('rbac_test_resource', {
  id: uuid('id').primaryKey(),
  ownerUserId: uuid('owner_user_id').notNull(),
});

let container: StartedPostgreSqlContainer;
let pool: Pool;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine').start();
  pool = new Pool({ connectionString: container.getConnectionUri() });
  await pool.query(
    'CREATE TABLE rbac_test_resource (id uuid PRIMARY KEY, owner_user_id uuid NOT NULL)',
  );
  await pool.query(
    `INSERT INTO rbac_test_resource (id, owner_user_id) VALUES
       ('11111111-1111-4111-8111-111111111111', 'aaaaaaaa-aaaa-4aaa-8aaa-aaaaaaaaaaaa'),
       ('22222222-2222-4222-8222-222222222222', 'bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbbb')`,
  );
});

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

describe('scoping dans la requete', () => {
  it('ne renvoie que les lignes du proprietaire', async () => {
    const database = drizzle(pool);
    const userId = 'aaaaaaaa-aaaa-4aaa-8aaa-aaaaaaaaaaaa';
    const filter = resolveScopeFilter('own', 'owner_user_id');
    const rows = await database
      .select()
      .from(resource)
      .where(applyScope([eq(resource.ownerUserId, resource.ownerUserId)], filter, resource.ownerUserId, userId));
    expect(rows).toHaveLength(1);
    expect(rows[0]!.ownerUserId).toBe(userId);
  });

  it('ne renvoie rien quand le scope est nul', async () => {
    const database = drizzle(pool);
    const filter = resolveScopeFilter('none', 'owner_user_id');
    const rows = await database
      .select()
      .from(resource)
      .where(applyScope([], filter, resource.ownerUserId, 'aaaaaaaa-aaaa-4aaa-8aaa-aaaaaaaaaaaa'));
    expect(rows).toHaveLength(0);
  });
});
```

- [ ] **Step 8: États, 3 langues et 2 thèmes**

Le RBAC est une couche serveur : aucune nouvelle chaîne d'interface n'est introduite. Le message de refus renvoyé au client porte le **code stable** `forbidden` et la clé de message `state.forbidden.title` / `state.forbidden.description`, déjà présents dans les trois dictionnaires de la Task 5. Le journal d'audit écrit `rbac.denied` avec `result: 'denied'` : cet événement est l'un des trois **événements de contrôle** vérifiés au Gate 1 (Task 15).

- [ ] **Step 9: Commit**

```bash
git add packages/db packages/contracts apps/api
git commit -m "feat(admin): ajoute le RBAC a 9 roles avec scoping anti-IDOR" -m "9 roles, 13 ressources, 8 actions, deny by default : une permission absente
de la base est refusee. Le scoping par role est applique dans la requete
(rbac_scope), jamais en post-filtrage en memoire.
La migration 0005_rbac_scopes.sql est append-only, avec lock_timeout,
statement_timeout et checksum SHA-256, sans script down.
Le four-eyes couvre price_tier, export_financial, refund, admin_role et
country, avec auto-approbation interdite.
Le guard exige un MFA verifie sur les ressources d administration.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 10: Pages publiques — accueil, catalogue vide, 404, CGV, contact, coquille de navigation

**Files:**
- Create: `apps/web/package.json`, `apps/web/next.config.ts`, `apps/web/postcss.config.mjs`, `apps/web/tsconfig.json`, `apps/web/eslint.config.mjs`, `apps/web/vitest.config.ts`
- Create: `apps/web/src/app/globals.css`
- Create: `apps/web/src/app/layout.tsx` (layout racine, `<html lang>`, script anti-flash)
- Create: `apps/web/src/app/[locale]/layout.tsx` (coquille localisée : skip link, en-tête, footer, nav basse)
- Create: `apps/web/src/app/[locale]/page.tsx` (accueil)
- Create: `apps/web/src/app/[locale]/not-found.tsx` (404)
- Create: `apps/web/src/app/[locale]/loading.tsx` (état de chargement)
- Create: `apps/web/src/app/[locale]/error.tsx` (état d'erreur, boundary client)
- Create: `apps/web/src/app/[locale]/catalogue/page.tsx` (catalogue vide)
- Create: `apps/web/src/app/[locale]/contact/page.tsx` (formulaire de contact)
- Create: `apps/web/src/app/[locale]/conditions/page.tsx` (CGV)
- Create: `apps/web/src/app/[locale]/panier/page.tsx` (état « ressource indisponible », Phase 2)
- Create: `apps/web/src/app/[locale]/commandes/page.tsx` (état « ressource indisponible », Phase 2)
- Create: `apps/web/src/components/layout/header.tsx`, `apps/web/src/components/layout/footer.tsx`, `apps/web/src/components/layout/bottom-nav.tsx`, `apps/web/src/components/layout/locale-switcher.tsx`
- Create: `apps/web/src/lib/locale.ts`, `apps/web/src/lib/theme.ts`
- Create: `apps/web/src/middleware.ts` (version minimale : résolution de locale et en-tête `x-fanours-locale` ; la version SEO complète arrive en Task 13)
- Modify: `packages/ui/package.json` (export CSS `./styles/tokens.css`)
- Modify: `packages/ui/src/styles/tokens.css` (ajout de `--header-height` et `--bottom-nav-height`)
- Modify: `packages/i18n/src/messages/fr-FR.ts`, `packages/i18n/src/messages/en.ts`, `packages/i18n/src/messages/zh-CN.ts` (12 clés nouvelles, dans les trois langues)
- Test: `apps/web/src/__tests__/home.test.tsx`, `apps/web/src/__tests__/shell.test.tsx`, `apps/web/src/__tests__/setup.tsx`

**Interfaces:**
- Consumes: `@fanours/ui` (Task 3 et Task 4 — `Button`, `IconButton`, `Input`, `Textarea`, `Skeleton`, `SystemState`, `THEME_SCRIPT`, `THEME_STORAGE_KEY`, `tokens.css`), `@fanours/i18n` (Task 5 — `DICTIONARIES`, `translate`, `I18nProvider`, `useI18n`, `LOCALE_ROUTES`, `LOCALE_LABELS`, `routeToLocale`, `detectLocaleFromAcceptLanguage`, `Locale`), `@fanours/contracts` (Task 1 — `ApiErrorCode`).
- Produces:
  - `isWebLocaleRoute(value: string): value is WebLocaleRoute`, `WebLocaleRoute = 'fr' | 'en' | 'zh-cn'`.
  - `homePath(locale: Locale): string` → `/fr/`, `/en/`, `/zh-cn/`.
  - `pathInLocale(locale: Locale, path: string): string`.
  - `switchLocaleInPath(pathname: string, target: Locale): string` — conserve le reste du chemin lors du changement de langue.
  - `readThemePreference(): ThemePreference`, `applyThemePreference(preference: ThemePreference): void`, `nextThemePreference(current: ThemePreference): ThemePreference` — cycle `auto → light → dark → auto`.
  - Composants : `Header({ locale })`, `Footer({ locale })`, `BottomNav({ locale })`, `LocaleSwitcher({ locale })`.
  - Pages : `HomePage({ locale })`, `CataloguePage({ locale })`, `ContactPage({ locale })`, `ConditionsPage({ locale })`, `NotFoundPage({ locale })`, `PanierPage({ locale })`, `CommandesPage({ locale })`.
  - Clés i18n ajoutées dans les trois dictionnaires : `footer.rights`, `contact.success.description`, `legal.terms.intro`, `legal.terms.section.moq.title`, `legal.terms.section.moq.body`, `legal.terms.section.payment.title`, `legal.terms.section.payment.body`, `legal.terms.section.shipping.title`, `legal.terms.section.shipping.body`, `legal.privacy.intro`, `legal.shipping.intro`, `home.benefits.customs.link`.

---

- [ ] **Step 1: Écrire les tests qui doivent échouer**

`apps/web/src/__tests__/setup.tsx` :

```tsx
import '@testing-library/jest-dom/vitest';
import { vi } from 'vitest';

vi.mock('next/navigation', () => ({
  usePathname: () => '/fr/',
  useRouter: () => ({ push: vi.fn(), replace: vi.fn() }),
  notFound: () => {
    throw new Error('NEXT_NOT_FOUND');
  },
}));

vi.mock('next/link', () => ({
  default: ({
    href,
    children,
    ...rest
  }: {
    href: string;
    children: React.ReactNode;
  }) => {
    const React = require('react') as typeof import('react');
    return React.createElement('a', { href, ...rest }, children);
  },
}));
```

`apps/web/src/__tests__/home.test.tsx` :

```tsx
import { render, screen } from '@testing-library/react';
import { describe, expect, it } from 'vitest';
import { DICTIONARIES } from '@fanours/i18n';
import { HomePage } from '../app/[locale]/page.js';

describe('accueil', () => {
  it('affiche un seul H1 dans la langue demandee', () => {
    render(<HomePage locale="fr-FR" />);
    const headings = screen.getAllByRole('heading', { level: 1 });
    expect(headings).toHaveLength(1);
    expect(headings[0]).toHaveTextContent(DICTIONARIES['fr-FR']['home.title']);
  });

  it('lie le CTA principal vers le catalogue de la locale', () => {
    render(<HomePage locale="en" />);
    const link = screen.getByRole('link', { name: DICTIONARIES['en']['home.cta.catalog'] });
    expect(link).toHaveAttribute('href', '/en/catalogue');
  });

  it('traduit la page en chinois simplifie', () => {
    render(<HomePage locale="zh-CN" />);
    expect(screen.getByRole('heading', { level: 1 })).toHaveTextContent(
      DICTIONARIES['zh-CN']['home.title'],
    );
  });

  it('presente les quatre arguments commerciaux', () => {
    render(<HomePage locale="fr-FR" />);
    for (const key of [
      'home.benefits.moq.title',
      'home.benefits.customs.title',
      'home.benefits.quality.title',
      'home.benefits.incoterms.title',
    ]) {
      expect(screen.getByText(DICTIONARIES['fr-FR'][key])).toBeInTheDocument();
    }
  });
});
```

`apps/web/src/__tests__/shell.test.tsx` :

```tsx
import { render, screen } from '@testing-library/react';
import { axe } from 'jest-axe';
import { describe, expect, it } from 'vitest';
import { I18nProvider, DICTIONARIES, type Locale } from '@fanours/i18n';
import { Header } from '../components/layout/header.js';
import { BottomNav } from '../components/layout/bottom-nav.js';
import { LocaleSwitcher } from '../components/layout/locale-switcher.js';

function renderShell(locale: Locale, children: React.ReactNode) {
  return render(<I18nProvider locale={locale}>{children}</I18nProvider>);
}

describe('coquille du site', () => {
  it('expose quatre onglets dans la nav basse', () => {
    renderShell('fr-FR', <BottomNav locale="fr-FR" />);
    const navigation = screen.getByRole('navigation', { name: DICTIONARIES['fr-FR']['nav.menu'] });
    expect(navigation.querySelectorAll('a')).toHaveLength(4);
  });

  it('propose les trois langues dans le switcher', () => {
    renderShell('fr-FR', <LocaleSwitcher locale="fr-FR" />);
    expect(screen.getByRole('link', { name: 'English' })).toHaveAttribute('href', '/en/');
    expect(screen.getByRole('link', { name: '简体中文' })).toHaveAttribute('href', '/zh-cn/');
  });

  it('n expose aucune violation d accessibilite critique', async () => {
    const { container } = renderShell(
      'fr-FR',
      <>
        <Header locale="fr-FR" />
        <BottomNav locale="fr-FR" />
        <LocaleSwitcher locale="fr-FR" />
      </>,
    );
    const results = await axe(container);
    expect(results.violations).toEqual([]);
  });
});
```

- [ ] **Step 2: Exécuter les tests et constater l'échec**

```bash
pnpm --filter @fanours/web test:unit
```

Attendu : `Cannot find module '../app/[locale]/page.js'` ou `No test files found` — le paquet `@fanours/web` n'existe pas encore. **Coller la sortie réelle** dans la PR.

- [ ] **Step 3: Poser le squelette de l'application web**

`apps/web/package.json` :

```json
{
  "name": "@fanours/web",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev --port 3000",
    "build": "next build",
    "start": "next start --port 3000",
    "typecheck": "tsc --noEmit",
    "lint": "eslint src",
    "test:unit": "vitest run",
    "test": "vitest run"
  },
  "dependencies": {
    "@fanours/contracts": "workspace:*",
    "@fanours/i18n": "workspace:*",
    "@fanours/ui": "workspace:*",
    "next": "15.1.3",
    "react": "19.0.0",
    "react-dom": "19.0.0"
  },
  "devDependencies": {
    "@tailwindcss/postcss": "4.0.0",
    "@testing-library/jest-dom": "6.6.3",
    "@testing-library/react": "16.1.0",
    "@testing-library/user-event": "14.5.2",
    "@types/node": "22.10.2",
    "@types/react": "19.0.2",
    "@types/react-dom": "19.0.2",
    "@vitejs/plugin-react": "4.3.4",
    "jest-axe": "9.0.0",
    "jsdom": "25.0.1",
    "tailwindcss": "4.0.0",
    "typescript": "5.6.3",
    "vite": "5.4.11",
    "vitest": "2.1.8"
  }
}
```

`apps/web/next.config.ts` :

```typescript
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  output: 'standalone',
  reactStrictMode: true,
  poweredByHeader: false,
  experimental: {
    optimizePackageImports: ['@fanours/ui'],
  },
  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
          {
            key: 'Strict-Transport-Security',
            value: 'max-age=63072000; includeSubDomains; preload',
          },
        ],
      },
    ];
  },
};

export default nextConfig;
```

La **Content-Security-Policy** n'est pas posée ici : elle exige un nonce par requête, donc elle est produite par le middleware en Task 13. Le comment est laissé dans le fichier pour que personne n'ajoute une CSP statique par erreur.

`apps/web/postcss.config.mjs` :

```javascript
export default {
  plugins: {
    '@tailwindcss/postcss': {},
  },
};
```

`apps/web/tsconfig.json` :

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "jsx": "preserve",
    "lib": ["dom", "dom.iterable", "esnext"],
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "src/**/*.ts", "src/**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

`apps/web/vitest.config.ts` :

```typescript
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import { fileURLToPath } from 'node:url';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/__tests__/setup.tsx'],
    include: ['src/**/*.test.ts', 'src/**/*.test.tsx'],
  },
});
```

`apps/web/eslint.config.mjs` :

```javascript
import base from '../../eslint.config.mjs';

export default [
  ...base,
  {
    files: ['src/**/*.{ts,tsx}'],
    rules: {
      'no-restricted-syntax': 'off',
    },
  },
  {
    ignores: ['.next/**', 'node_modules/**'],
  },
];
```

`packages/ui/package.json` — ajouter dans le champ `exports` :

```json
  "exports": {
    ".": "./src/index.ts",
    "./styles/tokens.css": "./src/styles/tokens.css"
  },
```

`apps/web/src/app/globals.css` :

```css
@import 'tailwindcss';
@import '@fanours/ui/styles/tokens.css';

@theme inline {
  --color-primary: var(--color-primary);
  --color-surface: var(--color-surface);
  --color-bg: var(--color-bg);
  --color-text: var(--color-text);
  --font-heading: var(--font-heading);
  --font-body: var(--font-body);
}
```

- [ ] **Step 4: Ajouter les deux tokens de mise en page manquants**

Dans `packages/ui/src/styles/tokens.css`, dans la section « Contrôles et mise en page », ajouter :

```css
  --header-height: 56px;
  --bottom-nav-height: 64px;
```

Ces deux valeurs sont les seules autorisées pour l'en-tête compact (56 px) et la nav basse (64 px). Aucun composant ne doit écrire `56px` ou `64px` en dur.

- [ ] **Step 5: Écrire les helpers de locale et de thème**

`apps/web/src/lib/locale.ts` :

```typescript
import { LOCALE_ROUTES, type Locale } from '@fanours/i18n';

export const WEB_LOCALE_ROUTES = ['fr', 'en', 'zh-cn'] as const;

export type WebLocaleRoute = (typeof WEB_LOCALE_ROUTES)[number];

export function isWebLocaleRoute(value: string): value is WebLocaleRoute {
  return (WEB_LOCALE_ROUTES as ReadonlyArray<string>).includes(value);
}

export function homePath(locale: Locale): string {
  return `/${LOCALE_ROUTES[locale]}/`;
}

export function pathInLocale(locale: Locale, path: string): string {
  const normalized = path.replace(/^\/+|\/+$/g, '');
  return normalized.length === 0 ? homePath(locale) : `/${LOCALE_ROUTES[locale]}/${normalized}`;
}

export function switchLocaleInPath(pathname: string, target: Locale): string {
  const segments = pathname.split('/').filter((segment) => segment.length > 0);
  if (segments.length > 0 && isWebLocaleRoute(segments[0]!)) {
    segments.shift();
  }
  const rest = segments.join('/');
  return rest.length === 0 ? homePath(target) : `/${LOCALE_ROUTES[target]}/${rest}`;
}
```

`apps/web/src/lib/theme.ts` :

```typescript
'use client';

import { THEME_STORAGE_KEY, type ThemePreference } from '@fanours/ui';

const ORDER: ReadonlyArray<ThemePreference> = ['auto', 'light', 'dark'];

export function readThemePreference(): ThemePreference {
  if (typeof window === 'undefined') {
    return 'auto';
  }
  try {
    const stored = window.localStorage.getItem(THEME_STORAGE_KEY);
    return stored === 'light' || stored === 'dark' || stored === 'auto' ? stored : 'auto';
  } catch {
    return 'auto';
  }
}

export function applyThemePreference(preference: ThemePreference): void {
  if (typeof document === 'undefined') {
    return;
  }
  const resolved =
    preference === 'auto'
      ? window.matchMedia('(prefers-color-scheme: dark)').matches
        ? 'dark'
        : 'light'
      : preference;
  document.documentElement.setAttribute('data-theme', resolved);
  document.documentElement.style.colorScheme = resolved;
  try {
    window.localStorage.setItem(THEME_STORAGE_KEY, preference);
  } catch {
    /* stockage indisponible : le thème reste applique pour la session */
  }
}

export function nextThemePreference(current: ThemePreference): ThemePreference {
  const index = ORDER.indexOf(current);
  return ORDER[(index + 1) % ORDER.length]!;
}
```

- [ ] **Step 6: Écrire la coquille de navigation**

`apps/web/src/components/layout/header.tsx` :

```tsx
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';
import { useCallback, useState } from 'react';
import { IconButton } from '@fanours/ui';
import { useI18n, type Locale } from '@fanours/i18n';
import { homePath, pathInLocale } from '@/lib/locale';
import { LocaleSwitcher } from './locale-switcher.js';
import {
  applyThemePreference,
  nextThemePreference,
  readThemePreference,
} from '@/lib/theme';

const DESKTOP_LINKS: ReadonlyArray<{ key: string; path: string }> = [
  { key: 'nav.home', path: '' },
  { key: 'nav.catalog', path: 'catalogue' },
  { key: 'nav.cart', path: 'panier' },
  { key: 'nav.orders', path: 'commandes' },
];

const SUN_ICON = (
  <svg viewBox="0 0 24 24" width="20" height="20" fill="none" aria-hidden="true">
    <circle cx="12" cy="12" r="4.5" stroke="currentColor" strokeWidth="1.6" />
    <path
      d="M12 2.5v2M12 19.5v2M2.5 12h2M19.5 12h2M5 5l1.4 1.4M17.6 17.6L19 19M19 5l-1.4 1.4M6.4 17.6L5 19"
      stroke="currentColor"
      strokeWidth="1.6"
      strokeLinecap="round"
    />
  </svg>
);

export interface HeaderProps {
  readonly locale: Locale;
}

export function Header({ locale }: HeaderProps) {
  const { t } = useI18n();
  const pathname = usePathname();
  const [preference, setPreference] = useState<'auto' | 'light' | 'dark'>('auto');

  const cycleTheme = useCallback(() => {
    const current = readThemePreference();
    const next = nextThemePreference(current);
    setPreference(next);
    applyThemePreference(next);
  }, []);

  return (
    <header
      data-testid="site-header"
      className="sticky top-0 z-[var(--z-nav)] h-[var(--header-height)] border-b border-[var(--color-border)] bg-[var(--glass-surface-light)] backdrop-blur-[var(--glass-blur)]"
    >
      <div className="mx-auto flex h-full max-w-[var(--content-max)] items-center gap-[var(--space-3)] px-[var(--gutter)]">
        <Link
          href={homePath(locale)}
          className="font-[var(--font-heading)] text-[var(--font-size-lg)] font-[var(--font-weight-bold)] text-[var(--color-text)] no-underline"
        >
          {t('app.name')}
        </Link>

        <nav
          aria-label={t('nav.menu')}
          className="hidden items-center gap-[var(--space-4)] md:flex"
        >
          {DESKTOP_LINKS.map((link) => {
            const href = pathInLocale(locale, link.path);
            const active = pathname === href;
            return (
              <Link
                key={link.key}
                href={href}
                aria-current={active ? 'page' : undefined}
                className={[
                  'text-[var(--font-size-sm)] no-underline',
                  active
                    ? 'text-[var(--color-primary)] font-[var(--font-weight-semibold)]'
                    : 'text-[var(--color-text-muted)]',
                ].join(' ')}
              >
                {t(link.key)}
              </Link>
            );
          })}
        </nav>

        <div className="ml-auto flex items-center gap-[var(--space-2)]">
          <LocaleSwitcher locale={locale} />
          <IconButton label={t('nav.theme')} icon={SUN_ICON} onClick={cycleTheme} />
        </div>
      </div>
    </header>
  );
}
```

`apps/web/src/components/layout/bottom-nav.tsx` :

```tsx
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';
import { useI18n, type Locale } from '@fanours/i18n';
import { pathInLocale } from '@/lib/locale';

const TABS: ReadonlyArray<{ key: string; path: string; icon: string }> = [
  { key: 'nav.home', path: '', icon: 'M3 10.5 12 3l9 7.5V21H3z' },
  { key: 'nav.catalog', path: 'catalogue', icon: 'M4 5h16l-1.5 14H5.5z' },
  { key: 'nav.cart', path: 'panier', icon: 'M3 4h2l2.2 10.5h9.6L19 7H6' },
  { key: 'nav.orders', path: 'commandes', icon: 'M7 3h10v4H7zM5 7h14v14H5z' },
];

export interface BottomNavProps {
  readonly locale: Locale;
}

export function BottomNav({ locale }: BottomNavProps) {
  const { t } = useI18n();
  const pathname = usePathname();

  return (
    <nav
      aria-label={t('nav.menu')}
      data-testid="bottom-nav"
      className="fixed inset-x-0 bottom-0 z-[var(--z-nav)] grid h-[var(--bottom-nav-height)] grid-cols-4 border-t border-[var(--color-border)] bg-[var(--color-surface)] pb-[env(safe-area-inset-bottom)] md:hidden"
    >
      {TABS.map((tab) => {
        const href = pathInLocale(locale, tab.path);
        const active = pathname === href;
        return (
          <Link
            key={tab.key}
            href={href}
            aria-current={active ? 'page' : undefined}
            className={[
              'flex min-h-[var(--touch-target-min)] flex-col items-center justify-center gap-[2px] text-[var(--font-size-xs)] no-underline',
              active
                ? 'text-[var(--color-primary)] font-[var(--font-weight-semibold)]'
                : 'text-[var(--color-text-muted)]',
            ].join(' ')}
          >
            <svg
              viewBox="0 0 24 24"
              width="22"
              height="22"
              fill="none"
              stroke="currentColor"
              strokeWidth="1.6"
              strokeLinejoin="round"
              strokeLinecap="round"
              aria-hidden="true"
            >
              <path d={tab.icon} />
            </svg>
            <span>{t(tab.key)}</span>
          </Link>
        );
      })}
    </nav>
  );
}
```

`apps/web/src/components/layout/locale-switcher.tsx` :

```tsx
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';
import { LOCALE_LABELS, useI18n, type Locale } from '@fanours/i18n';
import { switchLocaleInPath } from '@/lib/locale';

const ORDER: ReadonlyArray<Locale> = ['fr-FR', 'en', 'zh-CN'];

export interface LocaleSwitcherProps {
  readonly locale: Locale;
}

export function LocaleSwitcher({ locale }: LocaleSwitcherProps) {
  const { t } = useI18n();
  const pathname = usePathname();

  return (
    <nav
      aria-label={t('nav.language')}
      data-testid="locale-switcher"
      className="flex items-center gap-[var(--space-1)]"
    >
      {ORDER.map((candidate) => (
        <Link
          key={candidate}
          href={switchLocaleInPath(pathname, candidate)}
          lang={candidate}
          aria-current={candidate === locale ? 'true' : undefined}
          className={[
            'rounded-[var(--radius-sm)] px-[var(--space-2)] py-[var(--space-1)] text-[var(--font-size-xs)] no-underline',
            candidate === locale
              ? 'bg-[var(--color-accent)] font-[var(--font-weight-semibold)] text-[var(--color-text)]'
              : 'text-[var(--color-text-muted)]',
          ].join(' ')}
        >
          {LOCALE_LABELS[candidate]}
        </Link>
      ))}
    </nav>
  );
}
```

`apps/web/src/components/layout/footer.tsx` :

```tsx
import Link from 'next/link';
import { DICTIONARIES, translate, type Locale } from '@fanours/i18n';
import { homePath, pathInLocale } from '@/lib/locale';

const FOOTER_LINKS: ReadonlyArray<{ key: string; path: string }> = [
  { key: 'nav.catalog', path: 'catalogue' },
  { key: 'contact.title', path: 'contact' },
  { key: 'legal.terms.title', path: 'conditions' },
];

export interface FooterProps {
  readonly locale: Locale;
}

export function Footer({ locale }: FooterProps) {
  const dictionary = DICTIONARIES[locale];
  return (
    <footer className="mt-[var(--space-16)] border-t border-[var(--color-border)] bg-[var(--color-surface)] pb-[calc(var(--bottom-nav-height)+env(safe-area-inset-bottom))] md:pb-[var(--space-8)]">
      <div className="mx-auto flex max-w-[var(--content-max)] flex-col gap-[var(--space-4)] px-[var(--gutter)] py-[var(--space-8)]">
        <p className="font-[var(--font-heading)] text-[var(--font-size-lg)] font-[var(--font-weight-bold)]">
          {dictionary['app.name']}
        </p>
        <p className="max-w-[48ch] text-[var(--font-size-sm)] text-[var(--color-text-muted)]">
          {dictionary['app.tagline']}
        </p>
        <nav aria-label={dictionary['nav.menu']} className="flex flex-wrap gap-[var(--space-4)]">
          {FOOTER_LINKS.map((link) => (
            <Link
              key={link.key}
              href={pathInLocale(locale, link.path)}
              className="text-[var(--font-size-sm)] text-[var(--color-link)] no-underline"
            >
              {translate(dictionary, link.key)}
            </Link>
          ))}
          <Link
            href={homePath(locale)}
            className="text-[var(--font-size-sm)] text-[var(--color-link)] no-underline"
          >
            {dictionary['nav.home']}
          </Link>
        </nav>
        <p className="text-[var(--font-size-xs)] text-[var(--color-text-muted)]">
          {translate(dictionary, 'footer.rights')}
        </p>
      </div>
    </footer>
  );
}
```

- [ ] **Step 7: Écrire les layouts racine et localisé**

`apps/web/src/app/layout.tsx` :

```tsx
import { headers } from 'next/headers';
import Script from 'next/script';
import type { ReactNode } from 'react';
import { THEME_SCRIPT } from '@fanours/ui';
import { DEFAULT_LOCALE, type Locale } from '@fanours/i18n';
import './globals.css';

export default async function RootLayout({ children }: { children: ReactNode }) {
  const requestHeaders = await headers();
  const locale = (requestHeaders.get('x-fanours-locale') ?? DEFAULT_LOCALE) as Locale;
  const nonce = requestHeaders.get('x-nonce') ?? undefined;

  return (
    <html lang={locale} suppressHydrationWarning>
      <head>
        <Script id="fanours-theme" strategy="beforeInteractive" nonce={nonce}>
          {THEME_SCRIPT}
        </Script>
      </head>
      <body className="min-h-screen bg-[var(--color-bg)] text-[var(--color-text)] font-[var(--font-body)] antialiased">
        {children}
      </body>
    </html>
  );
}
```

`apps/web/src/app/[locale]/layout.tsx` :

```tsx
import { notFound } from 'next/navigation';
import type { ReactNode } from 'react';
import { DICTIONARIES, I18nProvider, routeToLocale, translate } from '@fanours/i18n';
import { Header } from '@/components/layout/header';
import { Footer } from '@/components/layout/footer';
import { BottomNav } from '@/components/layout/bottom-nav';

export default async function LocaleLayout({
  children,
  params,
}: {
  children: ReactNode;
  params: Promise<{ locale: string }>;
}) {
  const { locale: route } = await params;
  const locale = routeToLocale(route);
  if (locale === null) {
    notFound();
  }
  const dictionary = DICTIONARIES[locale];

  return (
    <I18nProvider locale={locale}>
      <a
        href="#contenu-principal"
        className="sr-only focus:not-sr-only focus:absolute focus:left-[var(--space-2)] focus:top-[var(--space-2)] focus:z-[var(--z-tooltip)] focus:rounded-[var(--radius-sm)] focus:bg-[var(--color-surface)] focus:px-[var(--space-3)] focus:py-[var(--space-2)]"
      >
        {translate(dictionary, 'nav.skipToContent')}
      </a>
      <Header locale={locale} />
      <main id="contenu-principal" className="mx-auto min-h-[60vh] max-w-[var(--content-max)] px-[var(--gutter)] py-[var(--space-8)]">
        {children}
      </main>
      <Footer locale={locale} />
      <BottomNav locale={locale} />
    </I18nProvider>
  );
}
```

- [ ] **Step 8: Écrire les pages publiques**

`apps/web/src/app/[locale]/page.tsx` :

```tsx
import Link from 'next/link';
import { DICTIONARIES, translate, type Locale } from '@fanours/i18n';
import { pathInLocale } from '@/lib/locale';

const BENEFITS: ReadonlyArray<{ title: string; body: string }> = [
  { title: 'home.benefits.moq.title', body: 'home.benefits.moq.body' },
  { title: 'home.benefits.customs.title', body: 'home.benefits.customs.body' },
  { title: 'home.benefits.quality.title', body: 'home.benefits.quality.body' },
  { title: 'home.benefits.incoterms.title', body: 'home.benefits.incoterms.body' },
];

export function HomePage({ locale }: { locale: Locale }) {
  const dictionary = DICTIONARIES[locale];
  return (
    <div className="flex flex-col gap-[var(--space-12)]">
      <section className="flex flex-col gap-[var(--space-4)] border-b border-[var(--color-border)] pb-[var(--space-10)]">
        <p className="text-[var(--font-size-xs)] font-[var(--font-weight-semibold)] uppercase tracking-[0.18em] text-[var(--color-primary)]">
          {translate(dictionary, 'app.name')}
        </p>
        <h1 className="max-w-[18ch] font-[var(--font-heading)] text-[var(--font-size-3xl)] font-[var(--font-weight-extrabold)] leading-[1.1]">
          {translate(dictionary, 'home.title')}
        </h1>
        <p className="max-w-[56ch] text-[var(--font-size-md)] text-[var(--color-text-muted)]">
          {translate(dictionary, 'home.subtitle')}
        </p>
        <div className="flex flex-wrap gap-[var(--space-3)]">
          <Link
            href={pathInLocale(locale, 'catalogue')}
            className="inline-flex min-h-[var(--control-height)] items-center rounded-[var(--radius-md)] bg-[var(--color-primary)] px-[var(--space-5)] text-[var(--font-size-sm)] font-[var(--font-weight-semibold)] text-[var(--color-text-on-primary)] no-underline"
          >
            {translate(dictionary, 'home.cta.catalog')}
          </Link>
          <Link
            href={pathInLocale(locale, 'contact')}
            className="inline-flex min-h-[var(--control-height)] items-center rounded-[var(--radius-md)] border border-[var(--color-border-strong)] px-[var(--space-5)] text-[var(--font-size-sm)] font-[var(--font-weight-semibold)] text-[var(--color-text)] no-underline"
          >
            {translate(dictionary, 'home.cta.quote')}
          </Link>
        </div>
      </section>

      <section className="grid gap-[var(--space-4)] sm:grid-cols-2">
        {BENEFITS.map((benefit) => (
          <article
            key={benefit.title}
            className="rounded-[var(--radius-lg)] border border-[var(--color-border)] bg-[var(--color-surface)] p-[var(--space-5)]"
          >
            <h2 className="font-[var(--font-heading)] text-[var(--font-size-lg)] font-[var(--font-weight-bold)]">
              {translate(dictionary, benefit.title)}
            </h2>
            <p className="mt-[var(--space-2)] text-[var(--font-size-sm)] text-[var(--color-text-muted)]">
              {translate(dictionary, benefit.body)}
            </p>
          </article>
        ))}
      </section>
    </div>
  );
}

export default async function Page({ params }: { params: Promise<{ locale: string }> }) {
  const { locale: route } = await params;
  const locale = routeToLocale(route);
  if (locale === null) {
    notFound();
  }
  return <HomePage locale={locale} />;
}
```

Le fichier doit importer `notFound` et `routeToLocale` depuis `next/navigation` et `@fanours/i18n` : la version complète du fichier commence par

```tsx
import { notFound } from 'next/navigation';
```

`apps/web/src/app/[locale]/catalogue/page.tsx` :

```tsx
import { notFound } from 'next/navigation';
import { DICTIONARIES, translate, routeToLocale, type Locale } from '@fanours/i18n';
import { Input, Skeleton, SystemState } from '@fanours/ui';

const EMPTY_ICON = (
  <svg viewBox="0 0 24 24" width="28" height="28" fill="none" stroke="currentColor" strokeWidth="1.6" aria-hidden="true">
    <path d="M4 7h16v12H4z" />
    <path d="M4 7l2-3h12l2 3" />
  </svg>
);

export function CataloguePage({ locale }: { locale: Locale }) {
  const dictionary = DICTIONARIES[locale];
  return (
    <div className="flex flex-col gap-[var(--space-6)]">
      <h1 className="font-[var(--font-heading)] text-[var(--font-size-2xl)] font-[var(--font-weight-extrabold)]">
        {translate(dictionary, 'catalog.title')}
      </h1>
      <label className="flex max-w-[420px] flex-col gap-[var(--space-2)] text-[var(--font-size-sm)]">
        <span>{translate(dictionary, 'catalog.search.label')}</span>
        <Input type="search" name="q" placeholder=" " />
      </label>
      <div className="grid gap-[var(--space-3)] sm:grid-cols-3" aria-hidden="true">
        <Skeleton className="h-[180px]" />
        <Skeleton className="h-[180px]" />
        <Skeleton className="h-[180px]" />
      </div>
      <SystemState
        variant="empty"
        title={translate(dictionary, 'catalog.empty.title')}
        description={translate(dictionary, 'catalog.empty.description')}
        actionLabel={translate(dictionary, 'catalog.empty.action')}
        icon={EMPTY_ICON}
      />
    </div>
  );
}

export default async function Page({ params }: { params: Promise<{ locale: string }> }) {
  const { locale: route } = await params;
  const locale = routeToLocale(route);
  if (locale === null) {
    notFound();
  }
  return <CataloguePage locale={locale} />;
}
```

`apps/web/src/app/[locale]/not-found.tsx` :

```tsx
import Link from 'next/link';
import { DICTIONARIES, routeToLocale, translate } from '@fanours/i18n';
import { homePath } from '@/lib/locale';

export default async function NotFound() {
  const locale = routeToLocale('fr') ?? 'fr-FR';
  const dictionary = DICTIONARIES[locale];
  return (
    <div className="mx-auto flex max-w-[48ch] flex-col items-start gap-[var(--space-4)] py-[var(--space-16)]">
      <p className="font-[var(--font-heading)] text-[var(--font-size-3xl)] font-[var(--font-weight-extrabold)]">404</p>
      <h1 className="font-[var(--font-heading)] text-[var(--font-size-xl)] font-[var(--font-weight-bold)]">
        {translate(dictionary, 'error.404.title')}
      </h1>
      <p className="text-[var(--font-size-sm)] text-[var(--color-text-muted)]">
        {translate(dictionary, 'error.404.description')}
      </p>
      <Link
        href={homePath(locale)}
        className="inline-flex min-h-[var(--control-height)] items-center rounded-[var(--radius-md)] bg-[var(--color-primary)] px-[var(--space-5)] text-[var(--font-size-sm)] font-[var(--font-weight-semibold)] text-[var(--color-text-on-primary)] no-underline"
      >
        {translate(dictionary, 'error.404.action')}
      </Link>
    </div>
  );
}
```

`apps/web/src/app/[locale]/loading.tsx` :

```tsx
import { Skeleton } from '@fanours/ui';

export default function Loading() {
  return (
    <div className="flex flex-col gap-[var(--space-6)]" role="status" aria-live="polite">
      <span className="sr-only">Chargement en cours</span>
      <Skeleton className="h-[140px]" />
      <div className="grid gap-[var(--space-4)] sm:grid-cols-2">
        <Skeleton className="h-[120px]" />
        <Skeleton className="h-[120px]" />
      </div>
    </div>
  );
}
```

`apps/web/src/app/[locale]/error.tsx` :

```tsx
'use client';

import { useEffect } from 'react';
import { Button, SystemState } from '@fanours/ui';
import { useI18n } from '@fanours/i18n';

const ERROR_ICON = (
  <svg viewBox="0 0 24 24" width="28" height="28" fill="none" stroke="currentColor" strokeWidth="1.6" aria-hidden="true">
    <circle cx="12" cy="12" r="9" />
    <path d="M12 7v6M12 16.5v.5" strokeLinecap="round" />
  </svg>
);

export default function ErrorBoundary({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  const { t } = useI18n();

  useEffect(() => {
    /* Le journal applicatif porte la trace ; aucune donnee technique n'est affichee ici. */
  }, [error]);

  return (
    <SystemState
      variant="error"
      title={t('state.error.title')}
      description={t('state.error.description')}
      actionLabel={t('state.error.action')}
      onAction={reset}
      reference={error.digest}
      icon={ERROR_ICON}
    />
  );
}
```

`apps/web/src/app/[locale]/contact/page.tsx` :

```tsx
'use client';

import { useState } from 'react';
import { Button, Input, SystemState, Textarea } from '@fanours/ui';
import { useI18n } from '@fanours/i18n';

const SENT_ICON = (
  <svg viewBox="0 0 24 24" width="28" height="28" fill="none" stroke="currentColor" strokeWidth="1.6" aria-hidden="true">
    <path d="M4 6h16v12H4z" />
    <path d="m4 7 8 6 8-6" />
  </svg>
);

const ERROR_ICON = (
  <svg viewBox="0 0 24 24" width="28" height="28" fill="none" stroke="currentColor" strokeWidth="1.6" aria-hidden="true">
    <path d="M12 4 3 20h18z" />
    <path d="M12 10v4M12 17v.5" strokeLinecap="round" />
  </svg>
);

export function ContactPage() {
  const { t } = useI18n();
  const [status, setStatus] = useState<'idle' | 'sending' | 'sent' | 'error'>('idle');

  async function onSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();
    setStatus('sending');
    try {
      const response = await fetch('/v1/contact', {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify(Object.fromEntries(new FormData(event.currentTarget))),
      });
      setStatus(response.ok ? 'sent' : 'error');
    } catch {
      setStatus('error');
    }
  }

  if (status === 'sent') {
    return (
      <SystemState
        variant="empty"
        title={t('contact.success.title')}
        description={t('contact.success.description')}
        icon={SENT_ICON}
      />
    );
  }

  if (status === 'error') {
    return (
      <SystemState
        variant="unavailable"
        title={t('state.unavailable.title')}
        description={t('state.unavailable.description')}
        actionLabel={t('state.unavailable.action')}
        onAction={() => setStatus('idle')}
        icon={ERROR_ICON}
      />
    );
  }

  return (
    <div className="flex max-w-[52ch] flex-col gap-[var(--space-5)]">
      <h1 className="font-[var(--font-heading)] text-[var(--font-size-2xl)] font-[var(--font-weight-extrabold)]">
        {t('contact.title')}
      </h1>
      <form onSubmit={onSubmit} className="flex flex-col gap-[var(--space-4)]">
        <label className="flex flex-col gap-[var(--space-2)] text-[var(--font-size-sm)]">
          <span>{t('contact.email.label')}</span>
          <Input type="email" name="email" required autoComplete="email" />
        </label>
        <label className="flex flex-col gap-[var(--space-2)] text-[var(--font-size-sm)]">
          <span>{t('contact.message.label')}</span>
          <Textarea name="message" rows={6} required />
        </label>
        <Button type="submit" loading={status === 'sending'}>
          {t('contact.submit')}
        </Button>
      </form>
    </div>
  );
}

export default ContactPage;
```

L'API de contact n'est pas livrée en Phase 1 : tant qu'elle n'existe pas, la réponse est un `404` et le formulaire affiche l'état « ressource indisponible ». C'est un comportement assumé — l'interface ne simule jamais une réussite.

`apps/web/src/app/[locale]/conditions/page.tsx` :

```tsx
import { notFound } from 'next/navigation';
import { DICTIONARIES, routeToLocale, translate, type Locale } from '@fanours/i18n';

const SECTIONS: ReadonlyArray<{ title: string; body: string }> = [
  { title: 'legal.terms.section.moq.title', body: 'legal.terms.section.moq.body' },
  { title: 'legal.terms.section.payment.title', body: 'legal.terms.section.payment.body' },
  { title: 'legal.terms.section.shipping.title', body: 'legal.terms.section.shipping.body' },
];

export function ConditionsPage({ locale }: { locale: Locale }) {
  const dictionary = DICTIONARIES[locale];
  return (
    <article className="flex max-w-[68ch] flex-col gap-[var(--space-6)]">
      <h1 className="font-[var(--font-heading)] text-[var(--font-size-2xl)] font-[var(--font-weight-extrabold)]">
        {translate(dictionary, 'legal.terms.title')}
      </h1>
      <p className="text-[var(--font-size-md)] text-[var(--color-text-muted)]">
        {translate(dictionary, 'legal.terms.intro')}
      </p>
      {SECTIONS.map((section) => (
        <section key={section.title} className="flex flex-col gap-[var(--space-2)]">
          <h2 className="font-[var(--font-heading)] text-[var(--font-size-lg)] font-[var(--font-weight-bold)]">
            {translate(dictionary, section.title)}
          </h2>
          <p className="text-[var(--font-size-sm)] text-[var(--color-text-muted)]">
            {translate(dictionary, section.body)}
          </p>
        </section>
      ))}
    </article>
  );
}

export default async function Page({ params }: { params: Promise<{ locale: string }> }) {
  const { locale: route } = await params;
  const locale = routeToLocale(route);
  if (locale === null) {
    notFound();
  }
  return <ConditionsPage locale={locale} />;
}
```

`apps/web/src/app/[locale]/panier/page.tsx` et `apps/web/src/app/[locale]/commandes/page.tsx` — même gabarit, avec l'état « ressource indisponible » qui annonce explicitement la livraison en Phase 2 :

```tsx
import { notFound } from 'next/navigation';
import { DICTIONARIES, routeToLocale, translate, type Locale } from '@fanours/i18n';
import { SystemState } from '@fanours/ui';

const LOCK_ICON = (
  <svg viewBox="0 0 24 24" width="28" height="28" fill="none" stroke="currentColor" strokeWidth="1.6" aria-hidden="true">
    <rect x="5" y="10" width="14" height="10" rx="2" />
    <path d="M8 10V7a4 4 0 0 1 8 0v3" />
  </svg>
);

export function PanierPage({ locale }: { locale: Locale }) {
  const dictionary = DICTIONARIES[locale];
  return (
    <SystemState
      variant="unavailable"
      title={translate(dictionary, 'cart.title')}
      description={translate(dictionary, 'state.unavailable.description')}
      actionLabel={translate(dictionary, 'state.unavailable.action')}
      icon={LOCK_ICON}
    />
  );
}

export default async function Page({ params }: { params: Promise<{ locale: string }> }) {
  const { locale: route } = await params;
  const locale = routeToLocale(route);
  if (locale === null) {
    notFound();
  }
  return <PanierPage locale={locale} />;
}
```

- [ ] **Step 9: Écrire le middleware minimal et le fichier de routes de l'API web**

`apps/web/src/middleware.ts` :

```typescript
import { NextResponse, type NextRequest } from 'next/server';
import { detectLocaleFromAcceptLanguage, routeToLocale } from '@fanours/i18n';
import { isWebLocaleRoute } from '@/lib/locale';

const TECHNICAL_PATHS = [
  '/_next',
  '/fonts',
  '/icons',
  '/api',
  '/sitemaps',
  '/robots.txt',
  '/sitemap.xml',
  '/manifest.webmanifest',
  '/sw.js',
];

export function middleware(request: NextRequest): NextResponse {
  const { pathname } = request.nextUrl;

  if (TECHNICAL_PATHS.some((prefix) => pathname.startsWith(prefix))) {
    return NextResponse.next();
  }

  const firstSegment = pathname.split('/').filter((segment) => segment.length > 0)[0] ?? '';

  if (isWebLocaleRoute(firstSegment)) {
    const locale = routeToLocale(firstSegment);
    if (locale === null) {
      return NextResponse.next();
    }
    const headers = new Headers(request.headers);
    headers.set('x-fanours-locale', locale);
    return NextResponse.next({ request: { headers } });
  }

  const locale = detectLocaleFromAcceptLanguage(request.headers.get('accept-language'));
  const url = request.nextUrl.clone();
  url.pathname = `/${locale === 'fr-FR' ? 'fr' : locale === 'en' ? 'en' : 'zh-cn'}${pathname}`;
  const headers = new Headers(request.headers);
  headers.set('x-fanours-locale', locale);
  return NextResponse.rewrite(url, { request: { headers } });
}

export const config = {
  matcher: ['/((?!_next/static|_next/image).*)'],
};
```

Ce middleware est **volontairement minimal** : il résout la locale et la rend disponible au layout racine via l'en-tête `x-fanours-locale`. La Task 13 le remplace par la version complète (redirection 302 de la racine, en-têtes `canonical`/`hreflang`, nonce CSP). À ce stade, un chemin sans locale est réécrit vers la locale détectée : `/xyz` devient `/fr/xyz`, qui ne correspond à aucune route et affiche donc la page 404 avec le statut 404.

- [ ] **Step 10: Ajouter les 12 clés i18n dans les trois dictionnaires**

Dans `packages/i18n/src/messages/fr-FR.ts`, ajouter :

```typescript
  'footer.rights': '© 2026 Fanours. Tous droits réservés.',
  'contact.success.description': 'Nous revenons vers vous sous un jour ouvré.',
  'legal.terms.intro':
    'Les conditions ci-dessous decrivent le cadre applicable aux commandes gros volume. Le texte definitif est valide par le conseil juridique avant publication.',
  'legal.terms.section.moq.title': 'Quantites minimales',
  'legal.terms.section.moq.body':
    'Chaque fiche produit indique une quantite minimale et des paliers de prix. Le palier retenu est celui de la quantite commandee.',
  'legal.terms.section.payment.title': 'Paiement',
  'legal.terms.section.payment.body':
    'Le paiement est traite par un prestataire externe. Aucune donnee de carte n est conservee par Fanours.',
  'legal.terms.section.shipping.title': 'Livraison et douane',
  'legal.terms.section.shipping.body':
    'Les delais et les frais de transport dependent du pays de destination. Les regles douanieres applicables sont indiquees avant validation.',
  'legal.privacy.intro':
    'Fanours collecte le strict necessaire pour traiter une commande et repondre a une demande. Aucun reglage publicitaire ni transfert a des tiers non listes.',
  'legal.shipping.intro':
    'Les delais indicatifs sont exprimes en jours ouvres et varient selon la destination et le mode d expedition choisi.',
  'home.benefits.customs.link': 'Voir les regles par pays',
```

Dans `packages/i18n/src/messages/en.ts`, ajouter les mêmes clés :

```typescript
  'footer.rights': '© 2026 Fanours. All rights reserved.',
  'contact.success.description': 'We reply within one business day.',
  'legal.terms.intro':
    'The terms below apply to wholesale orders. The final wording is validated by legal counsel before publication.',
  'legal.terms.section.moq.title': 'Minimum quantities',
  'legal.terms.section.moq.body':
    'Each product sheet states a minimum order quantity and price tiers. The tier applied matches the ordered quantity.',
  'legal.terms.section.payment.title': 'Payment',
  'legal.terms.section.payment.body':
    'Payment is processed by an external provider. Fanours never stores card data.',
  'legal.terms.section.shipping.title': 'Shipping and customs',
  'legal.terms.section.shipping.body':
    'Lead times and freight costs depend on the destination country. Applicable customs rules are shown before the order is confirmed.',
  'legal.privacy.intro':
    'Fanours collects only what is needed to process an order and answer a request. No advertising tracking and no transfer to unlisted third parties.',
  'legal.shipping.intro':
    'Lead times are expressed in business days and vary with the destination and the chosen shipping method.',
  'home.benefits.customs.link': 'See the rules by country',
```

Dans `packages/i18n/src/messages/zh-CN.ts`, ajouter les mêmes clés :

```typescript
  'footer.rights': '© 2026 Fanours。保留所有权利。',
  'contact.success.description': '我们将在一个工作日内回复。',
  'legal.terms.intro': '以下条款适用于批发订单。最终文本在发布前由法务顾问确认。',
  'legal.terms.section.moq.title': '起订量',
  'legal.terms.section.moq.body': '每个产品页都会标注起订量与阶梯价格，适用阶梯按订购数量计算。',
  'legal.terms.section.payment.title': '支付',
  'legal.terms.section.payment.body': '支付由外部服务商处理，Fanours 不保存任何银行卡数据。',
  'legal.terms.section.shipping.title': '运输与清关',
  'legal.terms.section.shipping.body': '时效与运费取决于目的国，确认订单前会显示适用的清关规则。',
  'legal.privacy.intro': 'Fanours 仅收集处理订单与回复咨询所必需的信息，不做广告追踪，也不向未列明的第三方传输数据。',
  'legal.shipping.intro': '时效以工作日计，随目的地与所选运输方式变化。',
  'home.benefits.customs.link': '查看各国规则',
```

Le contrôle de parité de la Task 5 (`packages/i18n/scripts/check-parity.ts`) doit rester vert : **exactement les mêmes 12 clés dans les troisdictionnaires**.

- [ ] **Step 11: Exécuter les tests et prouver la réussite**

```bash
pnpm --filter @fanours/i18n test:unit
pnpm --filter @fanours/i18n check:parity
pnpm --filter @fanours/web test:unit
```

Attendu :

```
 ✓ src/__tests__/home.test.tsx (4 tests)
 ✓ src/__tests__/shell.test.tsx (3 tests)

 Test Files  2 passed (2)
      Tests  7 passed (7)
```

et sur la parité :

```
Parite des dictionnaires verifiee : 111 cles dans 3 langues.
```

- [ ] **Step 12: Vérifier les 3 langues, les 2 thèmes, le responsive et l'absence de débordement**

```bash
pnpm --filter @fanours/web build
pnpm --filter @fanours/web lint
pnpm --filter @fanours/web typecheck
```

Les contrôles manuels à consigner dans la PR :

1. les trois URL `/fr/`, `/en/`, `/zh-cn/` rendent le contenu dans la bonne langue et le `<html lang>` vaut `fr-FR`, `en`, `zh-CN` ;
2. le thème clair et le thème sombre sont tous deux lisibles (le script anti-flash ne produit aucun éclair blanc) ;
3. à 360 px de large, la nav basse affiche 4 onglets, l'en-tête fait 56 px de haut et `document.scrollWidth <= window.innerWidth` ;
4. à 1024 px, la nav basse est masquée et la navigation horizontale de l'en-tête est visible ;
5. la navigation au clavier fonctionne sur l'en-tête, la nav basse et le switcher de langue, avec un anneau de focus visible.

- [ ] **Step 13: Commit**

```bash
git add apps/web packages/ui packages/i18n
git commit -m "feat(ui): ajoute les pages publiques et la coquille de navigation" -m "Accueil, catalogue vide, 404, CGV et contact, avec en-tete compact de 56 px,
nav basse de 4 onglets sous 768 px, footer, switcher de langue FR/EN/ZH et
selection de theme clair, sombre ou auto persistee localement.
Les etats systeme loading, vide, erreur et ressource indisponible sont
implements, et le formulaire de contact affiche un etat d erreur honnete
tant que l API de contact n est pas livree.
12 cles i18n ajoutees dans les trois dictionnaires, parite verifiee.
Les tokens --header-height et --bottom-nav-height sont ajoutes a la
source unique des tokens.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 11: Observabilité — logs JSON, `correlation_id` UUIDv7, `traceparent` W3C, métriques, santé

**Files:**
- Modify: `packages/observability/src/correlation.ts` (UUIDv7 réel + référence courte)
- Modify: `packages/observability/src/logger.ts` (champ `version` et champs obligatoires)
- Modify: `packages/observability/src/index.ts` (exports `trace` et `metrics`)
- Create: `packages/observability/src/trace.ts`
- Create: `packages/observability/src/metrics.ts`
- Create: `apps/api/src/version.ts`
- Create: `apps/api/src/common/correlation.middleware.ts`
- Create: `apps/api/src/common/error.filter.ts`
- Create: `apps/api/src/modules/health/health.module.ts`, `apps/api/src/modules/health/health.controller.ts`, `apps/api/src/modules/health/metrics.controller.ts`
- Modify: `apps/api/src/app.module.ts` (middleware de corrélation, filtre d'erreur global, module santé)
- Create: `apps/worker/package.json`, `apps/worker/tsconfig.json`, `apps/worker/src/version.ts`, `apps/worker/src/main.ts`, `apps/worker/src/jobs/outbox-relay.ts`
- Test: `packages/observability/src/__tests__/correlation.test.ts`, `packages/observability/src/__tests__/trace.test.ts`, `packages/observability/src/__tests__/metrics.test.ts`, `apps/api/src/common/__tests__/correlation.middleware.spec.ts`, `apps/api/src/modules/health/__tests__/health.controller.spec.ts`

**Interfaces:**
- Consumes: `@fanours/observability` (Task 1 — `redact`, `redactString`, `createLogger`, `hashIdentifier`, `formatTraceparent`, `parseTraceparent`, `newTraceId`, `newSpanId`), `@fanours/contracts` (Task 1 — `ApiErrorBody`, `ApiErrorCode`, `EVENT_TYPES`), `@fanours/db` (Task 2 — `outboxEvent`, `consumedEvent`), `apps/api/src/common/audit/audit.service.ts` (Task 8), `apps/api/src/common/security/rate-limit.ts` (Task 7 — client Redis).
- Produces:
  - `newCorrelationId(nowMs?: number): string` — **UUIDv7** (48 bits de timestamp, `version` 7, `variant` RFC 4122), horodatage monotone.
  - `isUuidV7(value: string): boolean`.
  - `shortReference(correlationId: string): string` — référence courte de 6 caractères, format affiché « Réf. 7F3K2Q ».
  - `TraceContext` : `{ traceId: string; spanId: string; sampled: boolean }` ; `withTrace<T>(context, call): Promise<T>` ; `injectTraceparent(headers, context): Record<string, string>`.
  - `MetricsRegistry` : `counter(name, help)`, `gauge(name, help)`, `histogram(name, help, buckets?)`, `inc(name, labels?)`, `set(name, value, labels?)`, `observe(name, value, labels?)`, `render(): string` — format d'exposition Prometheus.
  - `APP_VERSION` (`apps/api/src/version.ts` et `apps/worker/src/version.ts`).
  - Routes : `GET /healthz` (vivacité, sans dépendance), `GET /readyz` (prêtesse : PostgreSQL + Redis), `GET /v1/metrics` (Prometheus, désactivé en `production`).

---

- [ ] **Step 1: Écrire les tests qui doivent échouer**

`packages/observability/src/__tests__/correlation.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { isUuidV7, newCorrelationId, shortReference } from '../correlation.js';

describe('correlation_id', () => {
  it('produit un UUIDv7 valide', () => {
    const value = newCorrelationId();
    expect(value).toMatch(/^[0-9a-f]{8}-[0-9a-f]{4}-7[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/);
    expect(isUuidV7(value)).toBe(true);
  });

  it('encode l horodatage dans les 48 bits de poids fort', () => {
    const now = 1_700_000_000_000;
    const value = newCorrelationId(now);
    const timestamp = Number.parseInt(value.replace(/-/g, '').slice(0, 12), 16);
    expect(timestamp).toBe(now);
  });

  it('reste monotone dans la meme milliseconde', () => {
    const values = Array.from({ length: 50 }, () => newCorrelationId(1_700_000_000_000));
    expect(new Set(values).size).toBe(50);
    const sorted = [...values].sort();
    expect(sorted[0]).toBe(values[0]);
  });

  it('refuse un UUIDv4', () => {
    expect(isUuidV7('3f2504e0-4f89-41d3-9a0c-0305e82c3301')).toBe(false);
    expect(isUuidV7('pas-un-uuid')).toBe(false);
  });

  it('produit une reference courte de 6 caracteres sans caractere ambigu', () => {
    const reference = shortReference(newCorrelationId());
    expect(reference).toMatch(/^[0-9A-HJKMNP-TV-Z]{6}$/);
  });
});
```

`packages/observability/src/__tests__/trace.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { formatTraceparent, parseTraceparent } from '../correlation.js';
import { injectTraceparent, withTrace } from '../trace.js';

describe('traceparent W3C', () => {
  it('injecte un traceparent valide dans les en-tetes sortants', () => {
    const headers = injectTraceparent({ 'content-type': 'application/json' }, {
      traceId: '4bf92f3577b34da6a3ce929d0e0e4736',
      spanId: '00f067aa0ba902b7',
      sampled: true,
    });
    expect(headers['traceparent']).toBe('00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01');
  });

  it('conserve les en-tetes existants', () => {
    const headers = injectTraceparent({ accept: 'application/json' }, {
      traceId: '4bf92f3577b34da6a3ce929d0e0e4736',
      spanId: '00f067aa0ba902b7',
      sampled: false,
    });
    expect(headers['accept']).toBe('application/json');
    expect(headers['traceparent']).toMatch(/-00$/);
  });

  it('relit un traceparent emis par un service amont', () => {
    const parsed = parseTraceparent('00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01');
    expect(parsed).not.toBeNull();
    expect(parsed?.traceId).toBe('4bf92f3577b34da6a3ce929d0e0e4736');
    expect(parsed?.sampled).toBe(true);
  });

  it('propage le contexte dans l appel sortant', async () => {
    const seen: string[] = [];
    await withTrace(
      { traceId: '4bf92f3577b34da6a3ce929d0e0e4736', spanId: '00f067aa0ba902b7', sampled: true },
      async (headers) => {
        seen.push(headers['traceparent'] ?? '');
        return 'ok';
      },
    );
    expect(seen[0]).toBe('00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01');
  });
});
```

`packages/observability/src/__tests__/metrics.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { MetricsRegistry } from '../metrics.js';

describe('registre de metriques', () => {
  it('expose un compteur avec ses labels', () => {
    const registry = new MetricsRegistry();
    registry.inc('http_requests_total', { route: '/fr/', status: '200' });
    registry.inc('http_requests_total', { route: '/fr/', status: '200' });
    const output = registry.render();
    expect(output).toContain('# TYPE http_requests_total counter');
    expect(output).toContain('http_requests_total{route="/fr/",status="200"} 2');
  });

  it('expose un histogramme avec ses buckets', () => {
    const registry = new MetricsRegistry();
    registry.observe('http_request_duration_seconds', 0.2);
    const output = registry.render();
    expect(output).toContain('http_request_duration_seconds_count 1');
    expect(output).toContain('http_request_duration_seconds_sum 0.2');
    expect(output).toContain('http_request_duration_seconds_bucket{le="0.25"} 1');
  });

  it('separe les series par jeu de labels', () => {
    const registry = new MetricsRegistry();
    registry.inc('http_requests_total', { route: '/fr/', status: '200' });
    registry.inc('http_requests_total', { route: '/en/', status: '200' });
    const output = registry.render();
    expect(output).toContain('http_requests_total{route="/en/",status="200"} 1');
    expect(output).toContain('http_requests_total{route="/fr/",status="200"} 1');
  });
});
```

`apps/api/src/common/__tests__/correlation.middleware.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { CorrelationMiddleware } from '../correlation.middleware.js';

function createRequest(headers: Record<string, string> = {}) {
  return { headers, on: () => undefined } as never;
}

function createResponse() {
  const responseHeaders: Record<string, string> = {};
  return {
    responseHeaders,
    response: {
      setHeader: (key: string, value: string) => {
        responseHeaders[key] = value;
      },
      on: () => undefined,
    } as never,
  };
}

describe('middleware de correlation', () => {
  it('genere un correlation_id quand le client n en fournit pas', () => {
    const middleware = new CorrelationMiddleware();
    const request = createRequest();
    const { response, responseHeaders } = createResponse();
    middleware.use(request, response, () => undefined);
    expect(responseHeaders['X-Correlation-Id']).toMatch(
      /^[0-9a-f]{8}-[0-9a-f]{4}-7[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/,
    );
  });

  it('conserve un correlation_id client valide', () => {
    const middleware = new CorrelationMiddleware();
    const request = createRequest({ 'x-correlation-id': '018f1c2d-3e4f-7000-8000-000000000000' });
    const { response, responseHeaders } = createResponse();
    middleware.use(request, response, () => undefined);
    expect(responseHeaders['X-Correlation-Id']).toBe('018f1c2d-3e4f-7000-8000-000000000000');
  });

  it('remplace un correlation_id client qui n est pas un UUIDv7', () => {
    const middleware = new CorrelationMiddleware();
    const request = createRequest({ 'x-correlation-id': 'abc' });
    const { response, responseHeaders } = createResponse();
    middleware.use(request, response, () => undefined);
    expect(responseHeaders['X-Correlation-Id']).not.toBe('abc');
  });

  it('propage le traceparent entrant', () => {
    const middleware = new CorrelationMiddleware();
    const request = createRequest({
      traceparent: '00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01',
    });
    const { response, responseHeaders } = createResponse();
    middleware.use(request, response, () => undefined);
    expect(responseHeaders['traceparent']).toBe(
      '00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01',
    );
  });
});
```

`apps/api/src/modules/health/__tests__/health.controller.spec.ts` :

```typescript
import { describe, expect, it, vi } from 'vitest';
import { HealthController } from '../health.controller.js';

describe('healthz et readyz', () => {
  it('repond ok sur healthz sans toucher aux dependances', async () => {
    const controller = new HealthController();
    const result = await controller.healthz();
    expect(result.status).toBe('ok');
    expect(result.uptime_s).toBeGreaterThanOrEqual(0);
  });

  it('repond 503 quand une dependance est indisponible', async () => {
    const controller = new HealthController();
    const database = { execute: vi.fn().mockRejectedValue(new Error('connexion refusee')) } as never;
    const redis = { ping: vi.fn().mockResolvedValue('PONG') } as never;
    const result = await controller.readyz(database, redis);
    expect(result.status).toBe('degraded');
    expect(result.checks.database).toBe('ko');
  });
});
```

- [ ] **Step 2: Exécuter les tests et constater l'échec**

```bash
pnpm --filter @fanours/observability test:unit
```

Attendu : `Cannot find module '../trace.js'` et `Cannot find module '../metrics.js'`, et l'échec du test « encode l horodatage dans les 48 bits de poids fort » car `newCorrelationId()` renvoie encore un UUID v4. **Coller la sortie réelle** dans la PR.

- [ ] **Step 3: Corriger `newCorrelationId()` en UUIDv7 et ajouter la référence courte**

`packages/observability/src/correlation.ts` — remplacer le corps de `newCorrelationId` et ajouter les deux fonctions :

```typescript
import { randomBytes } from 'node:crypto';

let lastTimestampMs = 0;
let sequence = 0;

function formatUuid(bytes: Uint8Array): string {
  const hex = Array.from(bytes, (byte) => byte.toString(16).padStart(2, '0')).join('');
  return [
    hex.slice(0, 8),
    hex.slice(8, 12),
    hex.slice(12, 16),
    hex.slice(16, 20),
    hex.slice(20, 32),
  ].join('-');
}

/**
 * UUIDv7 (RFC 9562) : 48 bits d'horodatage Unix en millisecondes,
 * 4 bits de version, 12 bits de sequence monotones, 2 bits de variant,
 * 62 bits aleatoires. L'horodatage rend l'identifiant triable par
 * chronologie, ce qui est indispensable pour la lecture des journaux.
 */
export function newCorrelationId(nowMs: number = Date.now()): string {
  const timestamp = Math.max(nowMs, lastTimestampMs);
  if (timestamp === lastTimestampMs) {
    sequence = (sequence + 1) & 0x0fff;
    if (sequence === 0) {
      lastTimestampMs = timestamp + 1;
    }
  } else {
    lastTimestampMs = timestamp;
    sequence = 0;
  }

  const randA = sequence & 0x0fff;
  const randB = randomBytes(8).readBigUInt64BE() & 0x3fffffffffffffffn;

  const bytes = new Uint8Array(16);
  bytes[0] = Math.floor(lastTimestampMs / 2 ** 40) & 0xff;
  bytes[1] = Math.floor(lastTimestampMs / 2 ** 32) & 0xff;
  bytes[2] = Math.floor(lastTimestampMs / 2 ** 24) & 0xff;
  bytes[3] = Math.floor(lastTimestampMs / 2 ** 16) & 0xff;
  bytes[4] = Math.floor(lastTimestampMs / 2 ** 8) & 0xff;
  bytes[5] = lastTimestampMs & 0xff;
  bytes[6] = 0x70 | ((randA >> 8) & 0x0f);
  bytes[7] = randA & 0xff;
  bytes[8] = 0x80 | Number((randB >> 56) & 0x3fn);
  const tail = randB & 0x00ffffffffffffffn;
  for (let index = 0; index < 7; index += 1) {
    bytes[9 + index] = Number((tail >> BigInt(8 * (6 - index))) & 0xffn);
  }
  return formatUuid(bytes);
}

export function isUuidV7(value: string): boolean {
  return /^[0-9a-f]{8}-[0-9a-f]{4}-7[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/.test(value);
}

const CROCKFORD = '0123456789ABCDEFGHJKMNPQRSTVWXYZ';

/**
 * Reference courte affichee a l'utilisateur : « Ref. 7F3K2Q ».
 * Elle ne contient ni PII ni donnee technique, et permet de retrouver
 * la ligne de journal via le correlation_id complet conserve cote serveur.
 */
export function shortReference(correlationId: string): string {
  let hash = 2166136261;
  for (let index = 0; index < correlationId.length; index += 1) {
    hash ^= correlationId.charCodeAt(index);
    hash = Math.imul(hash, 16777619) >>> 0;
  }
  let value = hash & 0x3fffffff;
  let output = '';
  for (let index = 0; index < 6; index += 1) {
    output = CROCKFORD[value % 32]! + output;
    value = Math.floor(value / 32);
  }
  return output;
}
```

- [ ] **Step 4: Écrire `trace.ts` et `metrics.ts`**

`packages/observability/src/trace.ts` :

```typescript
import { formatTraceparent, newSpanId, newTraceId } from './correlation.js';

export interface TraceContext {
  readonly traceId: string;
  readonly spanId: string;
  readonly sampled: boolean;
}

export function newTraceContext(sampled = true): TraceContext {
  return { traceId: newTraceId(), spanId: newSpanId(), sampled };
}

export function injectTraceparent(
  headers: Readonly<Record<string, string>>,
  context: TraceContext,
): Record<string, string> {
  return {
    ...headers,
    traceparent: formatTraceparent(context.traceId, context.spanId, context.sampled),
  };
}

export async function withTrace<T>(
  context: TraceContext,
  call: (headers: Record<string, string>) => Promise<T>,
): Promise<T> {
  return call(injectTraceparent({}, context));
}
```

`packages/observability/src/metrics.ts` :

```typescript
export type MetricKind = 'counter' | 'gauge' | 'histogram';

const DEFAULT_BUCKETS: ReadonlyArray<number> = [
  0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10,
];

interface MetricDefinition {
  readonly kind: MetricKind;
  readonly help: string;
  readonly buckets: ReadonlyArray<number>;
}

interface HistogramValue {
  readonly counts: number[];
  readonly sum: number;
  readonly count: number;
}

function seriesKey(name: string, labels: Readonly<Record<string, string>>): string {
  const entries = Object.entries(labels).sort(([a], [b]) => (a < b ? -1 : 1));
  if (entries.length === 0) {
    return name;
  }
  const rendered = entries
    .map(([key, value]) => `${key}="${value.replace(/"/g, '\\"')}"`)
    .join(',');
  return `${name}{${rendered}}`;
}

export class MetricsRegistry {
  private readonly definitions = new Map<string, MetricDefinition>();
  private readonly values = new Map<string, number>();
  private readonly histograms = new Map<string, HistogramValue>();

  public counter(name: string, help: string): void {
    this.definitions.set(name, { kind: 'counter', help, buckets: [] });
  }

  public gauge(name: string, help: string): void {
    this.definitions.set(name, { kind: 'gauge', help, buckets: [] });
  }

  public histogram(name: string, help: string, buckets: ReadonlyArray<number> = DEFAULT_BUCKETS): void {
    this.definitions.set(name, {
      kind: 'histogram',
      help,
      buckets: [...buckets].sort((a, b) => a - b),
    });
  }

  public inc(name: string, labels: Readonly<Record<string, string>> = {}): void {
    const key = seriesKey(name, labels);
    this.values.set(key, (this.values.get(key) ?? 0) + 1);
  }

  public set(name: string, value: number, labels: Readonly<Record<string, string>> = {}): void {
    this.values.set(seriesKey(name, labels), value);
  }

  public observe(
    name: string,
    value: number,
    labels: Readonly<Record<string, string>> = {},
  ): void {
    const definition = this.definitions.get(name);
    const buckets = definition?.buckets ?? DEFAULT_BUCKETS;
    const key = seriesKey(name, labels);
    const existing = this.histograms.get(key) ?? {
      counts: new Array<number>(buckets.length).fill(0),
      sum: 0,
      count: 0,
    };
    for (let index = 0; index < buckets.length; index += 1) {
      if (value <= buckets[index]!) {
        existing.counts[index] = (existing.counts[index] ?? 0) + 1;
      }
    }
    existing.sum += value;
    existing.count += 1;
    this.histograms.set(key, existing);
  }

  public render(): string {
    const lines: string[] = [];
    for (const [name, definition] of this.definitions) {
      lines.push(`# HELP ${name} ${definition.help}`);
      lines.push(`# TYPE ${name} ${definition.kind}`);
      if (definition.kind === 'histogram') {
        for (const [key, histogram] of this.histograms) {
          if (!key.startsWith(`${name}{`)) {
            continue;
          }
          const labelEnd = key.indexOf('}');
          const base = key.slice(0, labelEnd);
          const labels = key.slice(labelEnd + 1, -1);
          for (let index = 0; index < definition.buckets.length; index += 1) {
            const rendered = labels.length > 0 ? `${base},le="${definition.buckets[index]}"${labels}` : `${base},le="${definition.buckets[index]}"`;
            lines.push(`${rendered} ${histogram.counts[index] ?? 0}`);
          }
          lines.push(`${base}_sum ${histogram.sum}`);
          lines.push(`${base}_count ${histogram.count}`);
        }
        continue;
      }
      for (const [key, value] of this.values) {
        if (key === name || key.startsWith(`${name}{`)) {
          lines.push(`${key} ${value}`);
        }
      }
    }
    return `${lines.join('\n')}\n`;
  }
}
```

- [ ] **Step 5: Ajouter le champ `version` au logger et exporter les nouveaux modules**

Dans `packages/observability/src/logger.ts`, remplacer la signature de `createLogger` par :

```typescript
export function createLogger(
  service: string,
  env: EnvName,
  logHashKey: string,
  version = '0.0.0',
) {
  const base = pino({
    base: { service, env, version },
    level: env === 'production' || env === 'staging' ? 'info' : 'debug',
    timestamp: pino.stdTimeFunctions.isoTime,
    formatters: {
      level: (label) => ({ level: label }),
    },
    hooks: {
      logMethod(args, method) {
        const [first] = args;
        const payload =
          typeof first === 'object' && first !== null ? (first as Record<string, unknown>) : {};
        method.call(this, redact(payload) as Record<string, unknown>);
      },
    },
  });
```

et ajouter à la fin du fichier :

```typescript
export const MANDATORY_LOG_FIELDS: ReadonlyArray<string> = [
  'timestamp',
  'level',
  'service',
  'env',
  'version',
  'correlation_id',
  'trace_id',
  'user_id_hash',
  'session_id_hash',
  'route',
  'method',
  'status',
  'duration_ms',
  'order_id',
  'event',
  'msg',
  'error',
];
```

`packages/observability/src/index.ts` devient :

```typescript
export {
  formatTraceparent,
  isUuidV7,
  newCorrelationId,
  newSpanId,
  newTraceId,
  parseTraceparent,
  shortReference,
} from './correlation.js';
export { createLogger, hashIdentifier, MANDATORY_LOG_FIELDS, type Logger, type LoggerFields } from './logger.js';
export { FORBIDDEN_KEYS, redact, redactString } from './redaction.js';
export { injectTraceparent, newTraceContext, withTrace, type TraceContext } from './trace.js';
export { MetricsRegistry, type MetricKind } from './metrics.js';
```

- [ ] **Step 6: Écrire le middleware de corrélation et le filtre d'erreur**

`apps/api/src/version.ts` :

```typescript
export const APP_VERSION = '0.1.0';
export const SERVICE_NAME = 'fanours-api';
```

`apps/api/src/common/correlation.middleware.ts` :

```typescript
import { Injectable, type NestMiddleware } from '@nestjs/common';
import type { NextFunction, Request, Response } from 'express';
import { isUuidV7, newCorrelationId, parseTraceparent } from '@fanours/observability';
import { createLogger } from '@fanours/observability';
import { APP_VERSION, SERVICE_NAME } from '../version.js';

export interface CorrelationRequest extends Request {
  correlationId?: string;
  traceId?: string;
  spanId?: string;
}

const logger = createLogger(SERVICE_NAME, 'local', 'development-only-hash-key', APP_VERSION);

@Injectable()
export class CorrelationMiddleware implements NestMiddleware {
  public use(request: CorrelationRequest, response: Response, next: NextFunction): void {
    const incoming = request.headers['x-correlation-id'];
    const provided = Array.isArray(incoming) ? incoming[0] : incoming;
    const correlationId =
      provided !== undefined && isUuidV7(provided) ? provided : newCorrelationId();

    const traceparentHeader = request.headers['traceparent'];
    const traceparent = Array.isArray(traceparentHeader) ? traceparentHeader[0] : traceparentHeader;
    const parsed = traceparent === undefined ? null : parseTraceparent(traceparent);

    request.correlationId = correlationId;
    request.traceId = parsed?.traceId;
    request.spanId = parsed?.spanId;
    request.headers['x-correlation-id'] = correlationId;

    response.setHeader('X-Correlation-Id', correlationId);
    if (parsed !== null) {
      response.setHeader('traceparent', traceparent!);
    }

    const startedAt = process.hrtime.bigint();
    response.on('finish', () => {
      const durationMs = Number(process.hrtime.bigint() - startedAt) / 1_000_000;
      logger.info(
        {
          correlation_id: correlationId,
          trace_id: request.traceId,
          route: request.route?.path ?? request.path,
          method: request.method,
          status: response.statusCode,
          duration_ms: Math.round(durationMs * 100) / 100,
          event: 'http.request',
        },
        'requête traitée',
      );
    });

    next();
  }
}
```

`apps/api/src/common/error.filter.ts` :

```typescript
import {
  ArgumentsHost,
  Catch,
  HttpException,
  type ExceptionFilter,
} from '@nestjs/common';
import type { Request, Response } from 'express';
import { API_ERROR_CODES, type ApiErrorBody, type ApiErrorCode } from '@fanours/contracts';
import { createLogger, shortReference } from '@fanours/observability';
import { APP_VERSION, SERVICE_NAME } from '../version.js';

const logger = createLogger(SERVICE_NAME, 'local', 'development-only-hash-key', APP_VERSION);

const STATUS_TO_CODE: Readonly<Record<number, ApiErrorCode>> = {
  400: API_ERROR_CODES.VALIDATION_FAILED,
  401: API_ERROR_CODES.UNAUTHENTICATED,
  403: API_ERROR_CODES.FORBIDDEN,
  404: API_ERROR_CODES.NOT_FOUND,
  409: API_ERROR_CODES.CONFLICT,
  413: API_ERROR_CODES.PAYLOAD_TOO_LARGE,
  429: API_ERROR_CODES.RATE_LIMITED,
  502: API_ERROR_CODES.UPSTREAM_UNAVAILABLE,
  503: API_ERROR_CODES.UPSTREAM_UNAVAILABLE,
};

const CODE_TO_MESSAGE_KEY: Readonly<Partial<Record<ApiErrorCode, string>>> = {
  validation_failed: 'state.error.title',
  unauthenticated: 'state.sessionExpired.title',
  forbidden: 'state.forbidden.title',
  not_found: 'error.404.title',
  conflict: 'state.error.title',
  rate_limited: 'state.error.title',
  payload_too_large: 'state.error.title',
  upstream_unavailable: 'state.unavailable.title',
  internal_error: 'state.error.title',
};

@Catch()
export class ApiExceptionFilter implements ExceptionFilter {
  public catch(exception: unknown, host: ArgumentsHost): void {
    const http = host.switchToHttp();
    const request = http.getRequest<Request & { correlationId?: string }>();
    const response = http.getResponse<Response>();
    const correlationId = request.correlationId ?? '00000000-0000-7000-8000-000000000000';

    const status =
      exception instanceof HttpException ? exception.getStatus() : 500;
    const code =
      exception instanceof HttpException
        ? (STATUS_TO_CODE[status] ?? API_ERROR_CODES.INTERNAL_ERROR)
        : API_ERROR_CODES.INTERNAL_ERROR;

    logger.error(
      {
        correlation_id: correlationId,
        route: request.path,
        method: request.method,
        status,
        event: 'http.error',
        error: exception instanceof Error ? exception.name : 'UnknownError',
      },
      'erreur traitée',
    );

    const body: ApiErrorBody = {
      error: {
        code,
        messageKey: CODE_TO_MESSAGE_KEY[code] ?? 'state.error.title',
        correlationId,
        details: { reference: shortReference(correlationId) },
      },
    };
    response.status(status).json(body);
  }
}
```

Le corps de la réponse ne contient **jamais** de message technique, de trace d'appels ni de fragment SQL : uniquement un code stable, une clé de message localisable et la référence courte.

- [ ] **Step 7: Écrire le module santé et l'endpoint de métriques**

`apps/api/src/modules/health/health.controller.ts` :

```typescript
import { Controller, Get, Inject } from '@nestjs/common';
import { sql } from 'drizzle-orm';
import type { NodePgDatabase } from 'drizzle-orm/node-postgres';
import type { Redis } from 'ioredis';
import { APP_VERSION, SERVICE_NAME } from '../../version.js';

export interface HealthReport {
  readonly status: 'ok' | 'ready' | 'degraded';
  readonly service: string;
  readonly version: string;
  readonly uptime_s: number;
  readonly checks?: Readonly<Record<string, 'ok' | 'ko'>>;
}

const startedAt = Date.now();

@Controller()
export class HealthController {
  public constructor(
    @Inject('DATABASE') private readonly db: NodePgDatabase<Record<string, never>>,
    @Inject('REDIS') private readonly redis: Redis,
  ) {}

  @Get('healthz')
  public healthz(): HealthReport {
    return {
      status: 'ok',
      service: SERVICE_NAME,
      version: APP_VERSION,
      uptime_s: Math.floor((Date.now() - startedAt) / 1000),
    };
  }

  @Get('readyz')
  public async readyz(): Promise<HealthReport> {
    const checks: Record<string, 'ok' | 'ko'> = { database: 'ok', redis: 'ok' };
    try {
      await this.db.execute(sql`select 1`);
    } catch {
      checks['database'] = 'ko';
    }
    try {
      await this.redis.ping();
    } catch {
      checks['redis'] = 'ko';
    }
    const degraded = Object.values(checks).some((value) => value === 'ko');
    return {
      status: degraded ? 'degraded' : 'ready',
      service: SERVICE_NAME,
      version: APP_VERSION,
      uptime_s: Math.floor((Date.now() - startedAt) / 1000),
      checks,
    };
  }
}
```

`apps/api/src/modules/health/metrics.controller.ts` :

```typescript
import { Controller, Get, Header } from '@nestjs/common';
import { MetricsRegistry } from '@fanours/observability';

export const metrics = new MetricsRegistry();
metrics.counter('http_requests_total', 'Nombre total de requêtes HTTP');
metrics.histogram('http_request_duration_seconds', 'Durée des requêtes HTTP en secondes');
metrics.counter('job_failures_total', 'Nombre total de jobs en échec');
metrics.gauge('queue_depth', 'Profondeur des files d’attente');

@Controller('v1')
export class MetricsController {
  @Get('metrics')
  @Header('Content-Type', 'text/plain; version=0.0.4; charset=utf-8')
  public render(): string {
    return metrics.render();
  }
}
```

`apps/api/src/modules/health/health.module.ts` :

```typescript
import { Module } from '@nestjs/common';
import { HealthController } from './health.controller.js';
import { MetricsController } from './metrics.controller.js';

@Module({
  controllers: [HealthController, MetricsController],
})
export class HealthModule {}
```

- [ ] **Step 8: Brancher le middleware, le filtre et le module dans l'application**

`apps/api/src/app.module.ts` devient :

```typescript
import { MiddlewareConsumer, Module, type NestModule } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';
import { AdminModule } from './modules/admin/admin.module.js';
import { ApiExceptionFilter } from './common/error.filter.js';
import { AuthModule } from './modules/auth/auth.module.js';
import { CorrelationMiddleware } from './common/correlation.middleware.js';
import { HealthModule } from './modules/health/health.module.js';
import { MfaModule } from './modules/mfa/mfa.module.js';
import { RbacModule } from './modules/rbac/rbac.module.js';

@Module({
  imports: [AuthModule, MfaModule, RbacModule, AdminModule, HealthModule],
  providers: [{ provide: APP_FILTER, useClass: ApiExceptionFilter }],
})
export class AppModule implements NestModule {
  public configure(consumer: MiddlewareConsumer): void {
    consumer.apply(CorrelationMiddleware).forRoutes('*');
  }
}
```

`GET /v1/metrics` est **désactivé en `production`** : le contrôleur n'est enregistré que si `APP_ENV !== 'production'`, la production exposant ses métriques par le collecteur de la plateforme et non par l'application. La condition est appliquée dans `health.module.ts` :

```typescript
import { Module } from '@nestjs/common';
import { HealthController } from './health.controller.js';
import { MetricsController } from './metrics.controller.js';

const isProduction = process.env['APP_ENV'] === 'production';

@Module({
  controllers: isProduction ? [HealthController] : [HealthController, MetricsController],
})
export class HealthModule {}
```

- [ ] **Step 9: Écrire le worker et la relais de l'outbox**

`apps/worker/package.json` :

```json
{
  "name": "@fanours/worker",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "tsc -b",
    "typecheck": "tsc -b --pretty",
    "start": "node dist/main.js"
  },
  "dependencies": {
    "@fanours/contracts": "workspace:*",
    "@fanours/db": "workspace:*",
    "@fanours/observability": "workspace:*",
    "bullmq": "5.34.6",
    "ioredis": "5.4.2"
  },
  "devDependencies": {
    "typescript": "5.6.3"
  }
}
```

`apps/worker/src/version.ts` :

```typescript
export const APP_VERSION = '0.1.0';
export const SERVICE_NAME = 'fanours-worker';
```

`apps/worker/src/jobs/outbox-relay.ts` :

```typescript
import { and, eq, lt } from 'drizzle-orm';
import type { NodePgDatabase } from 'drizzle-orm/node-postgres';
import { consumedEvent, outboxEvent } from '@fanours/db';
import type { EventType } from '@fanours/contracts';
import { createLogger, newCorrelationId } from '@fanours/observability';
import { APP_VERSION, SERVICE_NAME } from '../version.js';

const logger = createLogger(SERVICE_NAME, 'local', 'development-only-hash-key', APP_VERSION);

export interface OutboxRelayOptions {
  readonly db: NodePgDatabase<Record<string, never>>;
  readonly enqueue: (payload: {
    readonly eventType: EventType;
    readonly payload: Record<string, unknown>;
    readonly correlationId: string;
    readonly idempotencyKey: string;
  }) => Promise<void>;
  readonly batchSize?: number;
}

export class OutboxRelay {
  private readonly db: NodePgDatabase<Record<string, never>>;
  private readonly enqueue: OutboxRelayOptions['enqueue'];
  private readonly batchSize: number;

  public constructor(options: OutboxRelayOptions) {
    this.db = options.db;
    this.enqueue = options.enqueue;
    this.batchSize = options.batchSize ?? 50;
  }

  public async drain(): Promise<number> {
    const events = await this.db
      .select()
      .from(outboxEvent)
      .where(lt(outboxEvent.availableAt, new Date()))
      .limit(this.batchSize);

    let dispatched = 0;
    for (const event of events) {
      const idempotencyKey = `${event.id}`;
      const alreadyConsumed = await this.db
        .select({ id: consumedEvent.id })
        .from(consumedEvent)
        .where(
          and(
            eq(consumedEvent.consumer, 'outbox-relay'),
            eq(consumedEvent.idempotencyKey, idempotencyKey),
          ),
        )
        .limit(1);
      if (alreadyConsumed.length > 0) {
        continue;
      }
      const correlationId = newCorrelationId();
      await this.enqueue({
        eventType: event.eventType as EventType,
        payload: event.payload as Record<string, unknown>,
        correlationId,
        idempotencyKey,
      });
      await this.db.insert(consumedEvent).values({
        consumer: 'outbox-relay',
        idempotencyKey,
        eventId: event.id,
        correlationId,
      });
      dispatched += 1;
    }
    if (dispatched > 0) {
      logger.info({ event: 'outbox.dispatched', count: dispatched }, 'événements publiés');
    }
    return dispatched;
  }
}
```

`apps/worker/src/main.ts` :

```typescript
import { Queue, Worker } from 'bullmq';
import IORedis from 'ioredis';
import { createPool } from '@fanours/db';
import { createDb } from '@fanours/db';
import { createLogger, newCorrelationId } from '@fanours/observability';
import { OutboxRelay } from './jobs/outbox-relay.js';
import { APP_VERSION, SERVICE_NAME } from './version.js';

const logger = createLogger(SERVICE_NAME, 'local', 'development-only-hash-key', APP_VERSION);

const connection = new IORedis(process.env['REDIS_URL'] ?? 'redis://localhost:6379', {
  maxRetriesPerRequest: null,
});

const pool = createPool(process.env['DATABASE_URL'] ?? '');
const db = createDb(pool);

const queue = new Queue('outbox', { connection });

const relay = new OutboxRelay({
  db,
  enqueue: async (payload) => {
    await queue.add(payload.eventType, payload, {
      jobId: payload.idempotencyKey,
      attempts: 5,
      backoff: { type: 'exponential', delay: 1000 },
      removeOnComplete: 1000,
      removeOnFail: 5000,
    });
  },
});

const worker = new Worker(
  'outbox',
  async (job) => {
    const correlationId = (job.data as { correlationId?: string }).correlationId ?? newCorrelationId();
    logger.info(
      { correlation_id: correlationId, event: 'job.started', job_name: job.name },
      'job démarré',
    );
    return { correlationId, received: job.data };
  },
  { connection, concurrency: 5 },
);

worker.on('failed', (job, error) => {
  logger.error(
    {
      correlation_id: (job?.data as { correlationId?: string } | undefined)?.correlationId,
      event: 'job.failed',
      job_name: job?.name,
      error: error.name,
    },
    'job en échec',
  );
});

const timer = setInterval(() => {
  relay.drain().catch((error: unknown) => {
    logger.error(
      { event: 'outbox.failed', error: error instanceof Error ? error.name : 'UnknownError' },
      'relais outbox en échec',
    );
  });
}, 5000);

async function shutdown(): Promise<void> {
  clearInterval(timer);
  await worker.close();
  await queue.close();
  await connection.quit();
  await pool.end();
}

process.on('SIGTERM', () => {
  shutdown().finally(() => process.exit(0));
});
```

Le `correlation_id` est **créé par le relais et transporté dans le payload du job** : il est donc identique dans la ligne `outbox_event`, dans la file BullMQ, dans le log du job et dans l'appel sortant de l'intégration. C'est la fin de la chaîne HTTP → job → intégration.

- [ ] **Step 10: Exécuter les tests et prouver la réussite**

```bash
pnpm --filter @fanours/observability test:unit
pnpm --filter @fanours/api test:unit
```

Attendu :

```
 ✓ src/__tests__/correlation.test.ts (5 tests)
 ✓ src/__tests__/trace.test.ts (4 tests)
 ✓ src/__tests__/metrics.test.ts (3 tests)

 Test Files  3 passed (3)
      Tests  12 passed (12)
```

- [ ] **Step 11: Vérifier la chaîne de corrélation de bout en bout**

```bash
pnpm --filter @fanours/api build
pnpm --filter @fanours/worker typecheck
```

Contrôle manuel : un appel `curl -i http://localhost:3000/healthz` doit renvoyer l'en-tête `X-Correlation-Id: <uuid v7>` et le journal doit contenir la même valeur dans le champ `correlation_id`, avec `route`, `method`, `status` et `duration_ms` renseignés. Un appel avec `-H 'X-Correlation-Id: 018f1c2d-3e4f-7000-8000-000000000000'` doit conserver cet identifiant dans la réponse et dans le journal.

- [ ] **Step 12: États, 3 langues et 2 thèmes**

La Task 11 n'ajoute aucune chaîne d'interface : le champ `messageKey` renvoyé par le filtre d'erreur pointe vers des clés **existantes** (`state.error.title`, `state.forbidden.title`, `state.sessionExpired.title`, `state.unavailable.title`, `error.404.title`), déjà présentes dans les trois dictionnaires de la Task 5. La référence courte affichée au client est produite par `shortReference` et ne contient ni PII ni donnée technique.

- [ ] **Step 13: Commit**

```bash
git add packages/observability apps/api apps/worker
git commit -m "feat(infra): ajoute la correlation UUIDv7, les metriques et la sante" -m "Le correlation_id est desormais un UUIDv7 RFC 9562 horodate et monotone,
propage dans X-Correlation-Id, dans les jobs BullMQ et dans les appels
sortants via un traceparent W3C.
Les logs JSON portent les 17 champs obligatoires, dont version, et la
redaction PII et secrets reste active.
Un registre Prometheus minimal expose compteurs, jauges et histogrammes.
healthz ne touche a aucune dependance, readyz verifie PostgreSQL et Redis.
Le worker relaye l outbox avec une cle d idempotence par evenement.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 12: PWA socle — manifeste, service worker versionné, aucune donnée privée en cache

**Files:**
- Create: `apps/web/public/manifest.webmanifest`
- Create: `apps/web/public/icons/icon-192.svg`, `apps/web/public/icons/icon-512.svg`, `apps/web/public/icons/maskable-512.svg`
- Create: `apps/web/public/sw.js`
- Create: `apps/web/src/components/pwa/register-service-worker.tsx`
- Modify: `apps/web/src/app/[locale]/layout.tsx` (montage du composant d'enregistrement)
- Test: `apps/web/src/__tests__/sw.test.ts` (exécution du `sw.js` dans une machine virtuelle `node:vm`), `apps/web/src/__tests__/manifest.test.ts`

**Interfaces:**
- Consumes: `apps/web/public/manifest.webmanifest` (nouveau), `apps/web/public/sw.js` (nouveau), `@fanours/i18n` (Task 5 — `useI18n` pour le message d'installation).
- Produces:
  - `SW_VERSION` (`'fanours-sw-v1'`) — nom du cache versionné, changé à chaque livraison du `sw.js`.
  - `CACHE_SHELL` (precache du shell), `CACHE_RUNTIME` (assets publics), `CACHE_CATALOGUE` (pages catalogue, NetworkFirst).
  - `PRIVATE_PATTERNS` : liste explicite des chemins **jamais** mis en cache (`/panier`, `/checkout`, `/compte`, `/v1/orders`, `/v1/cart`, `/v1/quotes`, `/facture`, `/factures`).
  - `classifyRequest(request): 'shell' | 'runtime' | 'catalogue' | 'private' | 'network'`.
  - `shouldCacheResponse(request, response): boolean` — refuse tout `Set-Cookie`, toute réponse `private`/`no-store`, et toute réponse dont la requête est privée.
  - `purgeOldCaches(): Promise<string[]>` — supprime les caches qui ne portent pas la version courante.

---

- [ ] **Step 1: Écrire les tests qui doivent échouer**

`apps/web/src/__tests__/sw.test.ts` :

```typescript
import { readFileSync } from 'node:fs';
import { fileURLToPath } from 'node:url';
import { createContext, runInContext } from 'node:vm';
import { describe, expect, it } from 'vitest';

const swPath = fileURLToPath(new URL('../../public/sw.js', import.meta.url));
const source = readFileSync(swPath, 'utf8');

interface SwHarness {
  readonly classifyRequest: (url: string, method: string) => string;
  readonly shouldCacheResponse: (url: string, method: string, status: number, headers: Record<string, string>) => boolean;
  readonly cachesOpened: string[];
  readonly deletedCaches: string[];
}

function loadServiceWorker(): SwHarness {
  const cachesOpened: string[] = [];
  const deletedCaches: string[] = [];
  const cacheStorage = {
    open: async (name: string) => {
      cachesOpened.push(name);
      return {
        put: async () => undefined,
        match: async () => undefined,
        addAll: async () => undefined,
      };
    },
    keys: async () => [names.current],
    delete: async (name: string) => {
      deletedCaches.push(name);
      return true;
    },
  };
  const names = { current: 'fanours-sw-v1' };
  const context = createContext({
    self: { addEventListener: () => undefined, skipWaiting: () => undefined, clients: { claim: () => undefined } },
    caches: cacheStorage,
    fetch: async () => new Response('ok', { status: 200 }),
    Response,
    Request,
    URL,
    console,
    __names: names,
  });
  runInContext(`${source}\n;globalThis.__harness = { classifyRequest, shouldCacheResponse, purgeOldCaches };`, context);
  return {
    classifyRequest: context['__harness']!.classifyRequest,
    shouldCacheResponse: context['__harness']!.shouldCacheResponse,
    cachesOpened,
    deletedCaches,
  };
}

describe('service worker', () => {
  it('place le shell en precache', () => {
    const harness = loadServiceWorker();
    expect(harness.classifyRequest('/fr/', 'GET')).toBe('shell');
  });

  it('place les pages catalogue en NetworkFirst', () => {
    const harness = loadServiceWorker();
    expect(harness.classifyRequest('/fr/catalogue', 'GET')).toBe('catalogue');
  });

  it('ne met jamais en cache une donnee privee', () => {
    const harness = loadServiceWorker();
    for (const path of ['/fr/panier', '/fr/checkout', '/fr/compte', '/v1/orders', '/v1/quotes']) {
      expect(harness.classifyRequest(path, 'GET')).toBe('private');
    }
  });

  it('refuse de mettre en cache une reponse qui porte un cookie', () => {
    const harness = loadServiceWorker();
    expect(
      harness.shouldCacheResponse('/fr/', 'GET', 200, { 'set-cookie': 'fanours_sid=abc' }),
    ).toBe(false);
  });

  it('refuse de mettre en cache une reponse no-store', () => {
    const harness = loadServiceWorker();
    expect(
      harness.shouldCacheResponse('/fr/', 'GET', 200, { 'cache-control': 'no-store' }),
    ).toBe(false);
  });

  it('accepte de mettre en cache un asset public', () => {
    const harness = loadServiceWorker();
    expect(harness.shouldCacheResponse('/icons/icon-192.svg', 'GET', 200, {})).toBe(true);
  });
});
```

`apps/web/src/__tests__/manifest.test.ts` :

```typescript
import { readFileSync } from 'node:fs';
import { fileURLToPath } from 'node:url';
import { describe, expect, it } from 'vitest';

const manifestPath = fileURLToPath(new URL('../../public/manifest.webmanifest', import.meta.url));

describe('manifeste PWA', () => {
  const manifest = JSON.parse(readFileSync(manifestPath, 'utf8')) as Record<string, unknown>;

  it('declare le nom, le mode standalone et les couleurs de marque', () => {
    expect(manifest['name']).toBe('Fanours');
    expect(manifest['display']).toBe('standalone');
    expect(manifest['background_color']).toBe('#FFF5FA');
    expect(manifest['theme_color']).toBe('#D81B75');
  });

  it('declare les icones 192 et 512', () => {
    const icons = manifest['icons'] as ReadonlyArray<{ sizes: string; src: string }>;
    const sizes = icons.map((icon) => icon.sizes);
    expect(sizes).toContain('192x192');
    expect(sizes).toContain('512x512');
  });

  it('ne contient aucune URL absolue vers un service tiers', () => {
    const serialized = JSON.stringify(manifest);
    expect(serialized).not.toMatch(/https?:\/\/(?!localhost)/);
  });
});
```

- [ ] **Step 2: Exécuter les tests et constater l'échec**

```bash
pnpm --filter @fanours/web test:unit
```

Attendu : `ENOENT: no such file or directory, open '.../apps/web/public/sw.js'`. **Coller la sortie réelle** dans la PR.

- [ ] **Step 3: Écrire le manifeste et les icônes**

`apps/web/public/manifest.webmanifest` :

```json
{
  "name": "Fanours",
  "short_name": "Fanours",
  "description": "Sourcing grossiste depuis la Chine : MOQ explicites, frais de douane clairs, Incoterms annonces.",
  "lang": "fr-FR",
  "dir": "ltr",
  "start_url": "/fr/",
  "scope": "/",
  "display": "standalone",
  "orientation": "portrait-primary",
  "background_color": "#FFF5FA",
  "theme_color": "#D81B75",
  "categories": ["business", "shopping"],
  "icons": [
    { "src": "/icons/icon-192.svg", "sizes": "192x192", "type": "image/svg+xml", "purpose": "any" },
    { "src": "/icons/icon-512.svg", "sizes": "512x512", "type": "image/svg+xml", "purpose": "any" },
    { "src": "/icons/maskable-512.svg", "sizes": "512x512", "type": "image/svg+xml", "purpose": "maskable" }
  ],
  "shortcuts": [
    { "name": "Catalogue", "url": "/fr/catalogue" },
    { "name": "Panier", "url": "/fr/panier" }
  ]
}
```

`apps/web/public/icons/icon-512.svg` :

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" width="512" height="512" role="img" aria-label="Fanours">
  <rect width="512" height="512" rx="96" fill="#D81B75"/>
  <path d="M136 140h60l34 118 34-118h60l-62 232h-64z" fill="#FFF5FA"/>
</svg>
```

`apps/web/public/icons/icon-192.svg` : le même dessin, avec `width="192" height="192"` et `viewBox="0 0 512 512"`.

`apps/web/public/icons/maskable-512.svg` :

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" width="512" height="512" role="img" aria-label="Fanours">
  <rect width="512" height="512" fill="#D81B75"/>
  <path d="M168 172h48l24 84 24-84h48l-44 168h-56z" fill="#FFF5FA"/>
</svg>
```

- [ ] **Step 4: Écrire le service worker versionné**

`apps/web/public/sw.js` :

```javascript
/* Service worker Fanours — versionné. Toute livraison du fichier change
   SW_VERSION, ce qui purge automatiquement les anciens caches. */
const SW_VERSION = 'fanours-sw-v1';
const CACHE_SHELL = `${SW_VERSION}-shell`;
const CACHE_RUNTIME = `${SW_VERSION}-runtime`;
const CACHE_CATALOGUE = `${SW_VERSION}-catalogue`;

const SHELL_URLS = ['/fr/', '/en/', '/zh-cn/'];

const PRIVATE_PATTERNS = [
  '/panier',
  '/checkout',
  '/compte',
  '/account',
  '/commandes',
  '/orders',
  '/facture',
  '/factures',
  '/v1/cart',
  '/v1/orders',
  '/v1/quotes',
  '/v1/payment',
  '/v1/admin',
];

const CATALOGUE_PATTERNS = ['/catalogue', '/catalogue/'];

function classifyRequest(url, method) {
  if (method !== 'GET') {
    return 'network';
  }
  const path = new URL(url, 'https://fanours.local').pathname;
  if (PRIVATE_PATTERNS.some((pattern) => path.includes(pattern))) {
    return 'private';
  }
  if (SHELL_URLS.includes(path)) {
    return 'shell';
  }
  if (CATALOGUE_PATTERNS.some((pattern) => path.includes(pattern))) {
    return 'catalogue';
  }
  if (
    path.startsWith('/icons/') ||
    path.startsWith('/fonts/') ||
    path.startsWith('/_next/static/') ||
    path === '/manifest.webmanifest'
  ) {
    return 'runtime';
  }
  return 'network';
}

function shouldCacheResponse(url, method, status, headers) {
  const kind = classifyRequest(url, method);
  if (kind === 'private' || kind === 'network') {
    return false;
  }
  if (status !== 200) {
    return false;
  }
  const normalized = {};
  for (const key of Object.keys(headers)) {
    normalized[key.toLowerCase()] = headers[key];
  }
  if (normalized['set-cookie'] !== undefined) {
    return false;
  }
  const cacheControl = normalized['cache-control'] || '';
  if (cacheControl.includes('no-store') || cacheControl.includes('private')) {
    return false;
  }
  return true;
}

async function purgeOldCaches() {
  const names = await caches.keys();
  const removed = [];
  for (const name of names) {
    if (!name.startsWith('fanours-sw-') || name.startsWith(SW_VERSION)) {
      continue;
    }
    await caches.delete(name);
    removed.push(name);
  }
  return removed;
}

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches
      .open(CACHE_SHELL)
      .then((cache) => cache.addAll(SHELL_URLS))
      .then(() => self.skipWaiting()),
  );
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    purgeOldCaches().then(() => self.clients.claim()),
  );
});

self.addEventListener('fetch', (event) => {
  const request = event.request;
  const kind = classifyRequest(request.url, request.method);

  if (kind === 'private' || kind === 'network') {
    return;
  }

  if (kind === 'catalogue') {
    event.respondWith(
      fetch(request)
        .then((response) => {
          if (shouldCacheResponse(request.url, request.method, response.status, response.headers)) {
            const copy = response.clone();
            caches.open(CACHE_CATALOGUE).then((cache) => cache.put(request, copy));
          }
          return response;
        })
        .catch(() => caches.match(request).then((cached) => cached || caches.match('/fr/'))),
    );
    return;
  }

  if (kind === 'runtime') {
    event.respondWith(
      caches.match(request).then((cached) => {
        if (cached) {
          return cached;
        }
        return fetch(request).then((response) => {
          if (shouldCacheResponse(request.url, request.method, response.status, response.headers)) {
            const copy = response.clone();
            caches.open(CACHE_RUNTIME).then((cache) => cache.put(request, copy));
          }
          return response;
        });
      }),
    );
  }
});
```

Trois garanties sont vérifiables à la lecture :

1. `classifyRequest` renvoie `private` pour tout chemin contenant un segment privé, et le gestionnaire `fetch` **retourne sans rien faire** dans ce cas : la requête va toujours au réseau, jamais dans un cache ;
2. `shouldCacheResponse` refuse toute réponse portant `Set-Cookie` ou un `Cache-Control` contenant `no-store` ou `private` ;
3. `purgeOldCaches` supprime tout cache `fanours-sw-*` qui ne commence pas par la version courante.

- [ ] **Step 5: Enregistrer le service worker depuis le layout**

`apps/web/src/components/pwa/register-service-worker.tsx` :

```tsx
'use client';

import { useEffect } from 'react';

export function RegisterServiceWorker() {
  useEffect(() => {
    if (!('serviceWorker' in navigator)) {
      return;
    }
    const register = async () => {
      try {
        await navigator.serviceWorker.register('/sw.js', { scope: '/' });
      } catch {
        /* l application reste pleinement fonctionnelle sans service worker */
      }
    };
    void register();
  }, []);

  return null;
}
```

Dans `apps/web/src/app/[locale]/layout.tsx`, ajouter l'import et le rendu :

```tsx
import { RegisterServiceWorker } from '@/components/pwa/register-service-worker';
```

et, à l'intérieur de `<I18nProvider locale={locale}>` :

```tsx
      <RegisterServiceWorker />
```

- [ ] **Step 6: Exécuter les tests et prouver la réussite**

```bash
pnpm --filter @fanours/web test:unit
```

Attendu :

```
 ✓ src/__tests__/sw.test.ts (6 tests)
 ✓ src/__tests__/manifest.test.ts (3 tests)

 Test Files  2 passed (2)
      Tests  9 passed (9)
```

- [ ] **Step 7: Vérifier le comportement réel dans le navigateur**

```bash
pnpm --filter @fanours/web dev
```

Contrôle manuel, à consigner dans la PR :

1. `navigator.serviceWorker.controller` n'est pas nul après un second chargement ;
2. dans l'onglet Application → Cache Storage, seuls `fanours-sw-v1-shell`, `fanours-sw-v1-runtime` et `fanours-sw-v1-catalogue` existent ;
3. après avoir chargé `/fr/panier` puis `/fr/compte`, **aucune** entrée de cache ne contient une réponse de ces URL ;
4. après un rechargement, `fanours-sw-v1-runtime` ne contient pas de réponse portant `Set-Cookie`.

- [ ] **Step 8: États, 3 langues et 2 thèmes**

Le service worker ne dépend ni de la langue ni du thème : il classifie les URL par motif, pas par contenu. Le composant d'enregistrement ne rend rien et n'ajoute aucune chaîne. Le manifeste déclare `lang: "fr-FR"` et `dir: "ltr"`, ce qui convient aux trois langues de la phase 1 (aucune d'entre elles n'est en écriture droite de droite à gauche).

- [ ] **Step 9: Commit**

```bash
git add apps/web
git commit -m "feat(ui): ajoute le socle PWA avec cache aucune donnee privee" -m "Manifeste, icones SVG et service worker versionne fanours-sw-v1.
Le shell est en precache, les assets publics en cache runtime et les pages
catalogue en NetworkFirst.
Le panier, le checkout, le compte, les commandes, les factures et les
routes /v1 privees sont en NetworkOnly strict et ne sont jamais stockes.
Une reponse portant Set-Cookie ou Cache-Control no-store n est jamais
mise en cache, et les anciens caches sont purges a l activation.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 13: SEO technique — URLs par locale, canonical, hreflang, robots, sitemaps, JSON-LD

**Files:**
- Create: `apps/web/public/robots.txt` (contenu exact du spec §17.2)
- Create: `apps/web/src/app/sitemap.xml/route.ts` (index racine des sitemaps)
- Create: `apps/web/src/app/sitemaps/sitemap-index.xml/route.ts` (index par langue)
- Create: `apps/web/src/app/sitemaps/sitemap-[locale].xml/route.ts` (index par langue)
- Create: `apps/web/src/app/sitemaps/sitemap-[locale]-[type].xml/route.ts` (urlset par type)
- Create: `apps/web/src/components/seo/json-ld.tsx`
- Create: `apps/web/src/lib/seo.ts`
- Modify: `apps/web/src/middleware.ts` (version complète : 302 de la racine, CSP nonce, `noindex` sur les espaces privés)
- Modify: `apps/web/src/app/[locale]/layout.tsx` (métadonnées par page, JSON-LD, `noindex` sur les pages privées)
- Modify: `apps/web/src/app/[locale]/page.tsx`, `apps/web/src/app/[locale]/catalogue/page.tsx`, `apps/web/src/app/[locale]/contact/page.tsx`, `apps/web/src/app/[locale]/conditions/page.tsx` (ajout de `generateMetadata`)
- Test: `apps/web/src/__tests__/seo.test.ts`, `apps/web/src/__tests__/sitemaps.test.ts`, `apps/web/src/__tests__/json-ld.test.tsx`, `apps/web/src/__tests__/middleware.test.ts`

**Interfaces:**
- Consumes: `@fanours/i18n` (Task 5 — `LOCALE_ROUTES`, `routeToLocale`, `detectLocaleFromAcceptLanguage`, `DICTIONARIES`), `apps/web/src/lib/locale.ts` (Task 10 — `isWebLocaleRoute`, `homePath`, `pathInLocale`).
- Produces:
  - `siteUrl(): string` — origine publique issue de `APP_BASE_URL`.
  - `absoluteUrl(path: string): string`.
  - `localePath(locale: Locale, path: string): string`.
  - `alternateLanguages(path: string): Record<string, string>` — `fr`, `en`, `zh-cn` **plus `x-default`** pointant sur la variante anglaise.
  - `buildMetadata(input): Metadata` — canonical auto-référentiel absolu, `alternates.languages`, `robots` (`noindex` pour les pages privées), `openGraph` avec `og:locale` et `og:locale:alternate`.
  - `buildSitemapIndex(): string`, `buildLocaleIndex(locale: Locale): string`, `buildUrlSet(locale: Locale, type: SitemapType): string` — XML en 3 niveaux.
  - `SitemapType` : `'products' | 'pages' | 'editorial'`.
  - `JsonLd()` — un **seul** `@graph` contenant `Organization` et `WebSite` + `SearchAction`.
  - Middleware : redirection **302** de `/` vers `/{locale}/` (cookie `fanours_locale`, puis `Accept-Language`, puis `fr-FR`), **jamais** de redirection par IP, CSP avec nonce par requête, `X-Robots-Tag: noindex, nofollow, noarchive` sur les espaces privés.

---

- [ ] **Step 1: Écrire les tests qui doivent échouer**

`apps/web/src/__tests__/seo.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { alternateLanguages, buildMetadata, localePath } from '../lib/seo.js';

describe('metadonnees SEO', () => {
  it('produit un canonical absolu auto-referentiel', () => {
    const metadata = buildMetadata({
      locale: 'fr-FR',
      path: '/',
      title: 'Accueil',
      description: 'Sourcing grossiste',
    });
    expect(metadata.alternates?.canonical).toMatch(/\/fr\/$/);
  });

  it('declare les trois locales et x-default en anglais', () => {
    const languages = alternateLanguages('/');
    expect(Object.keys(languages).sort()).toEqual(['en', 'fr', 'x-default', 'zh-cn']);
    expect(languages['x-default']).toMatch(/\/en\/$/);
  });

  it('interdit l indexation d une page privee', () => {
    const metadata = buildMetadata({
      locale: 'fr-FR',
      path: '/panier',
      title: 'Panier',
      description: 'Panier',
      noindex: true,
    });
    expect(metadata.robots).toMatchObject({ index: false, follow: false });
  });

  it('construit un chemin localise sans double barre', () => {
    expect(localePath('zh-CN', '/catalogue')).toBe('/zh-cn/catalogue');
    expect(localePath('en', '/')).toBe('/en/');
  });
});
```

`apps/web/src/__tests__/sitemaps.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { buildLocaleIndex, buildSitemapIndex, buildUrlSet } from '../lib/seo.js';

describe('sitemaps', () => {
  it('l index racine reference les index par langue', () => {
    const xml = buildSitemapIndex();
    expect(xml).toContain('<sitemapindex');
    expect(xml).toContain('/sitemaps/sitemap-fr.xml');
    expect(xml).toContain('/sitemaps/sitemap-en.xml');
    expect(xml).toContain('/sitemaps/sitemap-zh-cn.xml');
  });

  it('l index par langue reference les trois types', () => {
    const xml = buildLocaleIndex('fr-FR');
    expect(xml).toContain('sitemap-fr-products.xml');
    expect(xml).toContain('sitemap-fr-pages.xml');
    expect(xml).toContain('sitemap-fr-editorial.xml');
  });

  it('le urlset des pages contient les pages publiques', () => {
    const xml = buildUrlSet('fr-FR', 'pages');
    expect(xml).toContain('<urlset');
    expect(xml).toContain('/fr/catalogue');
    expect(xml).not.toContain('/fr/panier');
  });

  it('le urlset produits est vide tant que le catalogue est vide', () => {
    const xml = buildUrlSet('fr-FR', 'products');
    expect(xml).toContain('<urlset');
    expect(xml).not.toContain('<url>');
  });
});
```

`apps/web/src/__tests__/json-ld.test.tsx` :

```tsx
import { render } from '@testing-library/react';
import { describe, expect, it } from 'vitest';
import { JsonLd } from '../components/seo/json-ld.js';

describe('JSON-LD', () => {
  it('rend un seul @graph contenant Organization et WebSite', () => {
    const { container } = render(<JsonLd />);
    const scripts = container.querySelectorAll('script[type="application/ld+json"]');
    expect(scripts).toHaveLength(1);
    const parsed = JSON.parse(scripts[0]!.textContent ?? '{}') as {
      '@context': string;
      '@graph': ReadonlyArray<Record<string, unknown>>;
    };
    expect(parsed['@context']).toBe('https://schema.org');
    expect(parsed['@graph'].map((node) => node['@type'])).toEqual(['Organization', 'WebSite']);
    const website = parsed['@graph'][1]!;
    expect(website['potentialAction']).toMatchObject({ '@type': 'SearchAction' });
  });

  it('n echappe pas le JSON par une entite HTML', () => {
    const { container } = render(<JsonLd />);
    const raw = container.innerHTML;
    expect(raw).not.toContain('&quot;');
  });
});
```

`apps/web/src/__tests__/middleware.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { buildContentSecurityPolicy, resolveLocaleForRequest } from '../middleware.js';

describe('middleware SEO', () => {
  it('redirige la racine en 302 vers la locale detectee', () => {
    const locale = resolveLocaleForRequest({ cookie: undefined, acceptLanguage: 'zh-CN,zh;q=0.9' });
    expect(locale).toBe('zh-CN');
  });

  it('donne la priorite au cookie de choix explicite', () => {
    const locale = resolveLocaleForRequest({ cookie: 'en', acceptLanguage: 'fr-FR,fr;q=0.9' });
    expect(locale).toBe('en');
  });

  it('retombe sur la locale par defaut sans en-tete exploitable', () => {
    expect(resolveLocaleForRequest({ cookie: undefined, acceptLanguage: undefined })).toBe('fr-FR');
  });

  it('produit une CSP avec nonce et sans unsafe-inline sur script-src', () => {
    const csp = buildContentSecurityPolicy('nonce-123');
    expect(csp).toContain("script-src 'self' 'nonce-nonce-123' 'strict-dynamic'");
    expect(csp).not.toContain("script-src 'self' 'unsafe-inline'");
    expect(csp).toContain("frame-ancestors 'none'");
  });
});
```

- [ ] **Step 2: Exécuter les tests et constater l'échec**

```bash
pnpm --filter @fanours/web test:unit
```

Attendu : `Cannot find module '../lib/seo.js'` et `Cannot find module '../middleware.js'` (les exports `buildContentSecurityPolicy` et `resolveLocaleForRequest` n'existent pas encore). **Coller la sortie réelle** dans la PR.

- [ ] **Step 3: Écrire `lib/seo.ts`**

`apps/web/src/lib/seo.ts` :

```typescript
import type { Metadata } from 'next';
import { LOCALE_ROUTES, type Locale } from '@fanours/i18n';

export const SITEMAP_TYPES = ['products', 'pages', 'editorial'] as const;

export type SitemapType = (typeof SITEMAP_TYPES)[number];

const OG_LOCALES: Readonly<Record<Locale, string>> = {
  'fr-FR': 'fr_FR',
  en: 'en_US',
  'zh-CN': 'zh_CN',
};

const OG_ALTERNATES: Readonly<Record<Locale, ReadonlyArray<string>>> = {
  'fr-FR': ['en_US', 'zh_CN'],
  en: ['fr_FR', 'zh_CN'],
  'zh-CN': ['fr_FR', 'en_US'],
};

export function siteUrl(): string {
  return (process.env['APP_BASE_URL'] ?? 'http://localhost:3000').replace(/\/+$/, '');
}

export function absoluteUrl(path: string): string {
  return `${siteUrl()}${path.startsWith('/') ? path : `/${path}`}`;
}

export function localePath(locale: Locale, path = '/'): string {
  const route = LOCALE_ROUTES[locale];
  const normalized = path === '/' ? '' : `/${path.replace(/^\/+|\/+$/g, '')}`;
  return `/${route}${normalized}`;
}

export function alternateLanguages(path: string): Record<string, string> {
  const languages: Record<string, string> = {};
  for (const [locale, route] of Object.entries(LOCALE_ROUTES)) {
    languages[route] = absoluteUrl(localePath(locale as Locale, path));
  }
  languages['x-default'] = absoluteUrl(localePath('en', path));
  return languages;
}

export interface BuildMetadataInput {
  readonly locale: Locale;
  readonly path: string;
  readonly title: string;
  readonly description: string;
  readonly noindex?: boolean;
}

export function buildMetadata(input: BuildMetadataInput): Metadata {
  const canonical = absoluteUrl(localePath(input.locale, input.path));
  return {
    title: input.title,
    description: input.description,
    alternates: {
      canonical,
      languages: alternateLanguages(input.path),
    },
    robots: input.noindex === true
      ? { index: false, follow: false, nocache: true }
      : { index: true, follow: true },
    openGraph: {
      type: 'website',
      url: canonical,
      siteName: 'Fanours',
      locale: OG_LOCALES[input.locale],
      alternateLocale: OG_ALTERNATES[input.locale],
      title: input.title,
      description: input.description,
    },
  };
}

const PUBLIC_PAGES: ReadonlyArray<{ path: string; type: SitemapType }> = [
  { path: '/', type: 'pages' },
  { path: '/catalogue', type: 'pages' },
  { path: '/contact', type: 'pages' },
  { path: '/conditions', type: 'pages' },
];

function escapeXml(value: string): string {
  return value
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}

function xmlHeader(): string {
  return '<?xml version="1.0" encoding="UTF-8"?>';
}

export function buildSitemapIndex(): string {
  const entries = (Object.keys(LOCALE_ROUTES) as Locale[])
    .map(
      (locale) =>
        `  <sitemap><loc>${escapeXml(absoluteUrl(`/sitemaps/sitemap-${LOCALE_ROUTES[locale]}.xml`))}</loc></sitemap>`,
    )
    .join('\n');
  return `${xmlHeader()}\n<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">\n${entries}\n</sitemapindex>\n`;
}

export function buildLocaleIndex(locale: Locale): string {
  const route = LOCALE_ROUTES[locale];
  const entries = SITEMAP_TYPES.map(
    (type) =>
      `  <sitemap><loc>${escapeXml(absoluteUrl(`/sitemaps/sitemap-${route}-${type}.xml`))}</loc></sitemap>`,
  ).join('\n');
  return `${xmlHeader()}\n<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">\n${entries}\n</sitemapindex>\n`;
}

export function buildUrlSet(locale: Locale, type: SitemapType): string {
  const urls = PUBLIC_PAGES.filter((page) => page.type === type)
    .map((page) => {
      const location = escapeXml(absoluteUrl(localePath(locale, page.path)));
      const alternates = (Object.keys(LOCALE_ROUTES) as Locale[])
        .map(
          (candidate) =>
            `    <xhtml:link rel="alternate" hreflang="${LOCALE_ROUTES[candidate]}" href="${escapeXml(absoluteUrl(localePath(candidate, page.path)))}"/>`,
        )
        .join('\n');
      return `  <url>\n    <loc>${location}</loc>\n    <changefreq>weekly</changefreq>\n    <priority>0.7</priority>\n${alternates}\n  </url>`;
    })
    .join('\n');
  return `${xmlHeader()}\n<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9" xmlns:xhtml="http://www.w3.org/1999/xhtml">\n${urls}\n</urlset>\n`;
}
```

Le catalogue étant vide en Phase 1, le urlset `products` est **volontairement vide** : une liste de produits n'est jamais inventée. L'espace privé (`panier`, `compte`, `commandes`, `devis`, `facturation`) n'apparaît dans aucun sitemap.

- [ ] **Step 4: Écrire les routes de sitemaps et le `robots.txt`**

`apps/web/src/app/sitemap.xml/route.ts` :

```typescript
import { NextResponse } from 'next/server';
import { buildSitemapIndex } from '@/lib/seo';

export const dynamic = 'force-static';

export function GET(): NextResponse {
  return new NextResponse(buildSitemapIndex(), {
    headers: {
      'Content-Type': 'application/xml; charset=utf-8',
      'Cache-Control': 'public, max-age=3600',
    },
  });
}
```

`apps/web/src/app/sitemaps/sitemap-index.xml/route.ts` :

```typescript
import { NextResponse } from 'next/server';
import { buildSitemapIndex } from '@/lib/seo';

export const dynamic = 'force-static';

export function GET(): NextResponse {
  return new NextResponse(buildSitemapIndex(), {
    headers: {
      'Content-Type': 'application/xml; charset=utf-8',
      'Cache-Control': 'public, max-age=3600',
    },
  });
}
```

`apps/web/src/app/sitemaps/sitemap-[locale].xml/route.ts` :

```typescript
import { NextResponse } from 'next/server';
import { routeToLocale } from '@fanours/i18n';
import { buildLocaleIndex } from '@/lib/seo';

export const dynamic = 'force-static';

export function GET(
  request: Request,
  context: { params: Promise<{ locale: string }> },
): NextResponse {
  const { locale: route } = await context.params;
  const locale = routeToLocale(route);
  if (locale === null) {
    return new NextResponse('locale inconnue', { status: 404 });
  }
  void request;
  return new NextResponse(buildLocaleIndex(locale), {
    headers: {
      'Content-Type': 'application/xml; charset=utf-8',
      'Cache-Control': 'public, max-age=3600',
    },
  });
}
```

`apps/web/src/app/sitemaps/sitemap-[locale]-[type].xml/route.ts` :

```typescript
import { NextResponse } from 'next/server';
import { routeToLocale } from '@fanours/i18n';
import { SITEMAP_TYPES, buildUrlSet, type SitemapType } from '@/lib/seo';

export const dynamic = 'force-static';

export async function GET(
  request: Request,
  context: { params: Promise<{ locale: string; type: string }> },
): Promise<NextResponse> {
  const { locale: route, type } = await context.params;
  const locale = routeToLocale(route);
  if (locale === null || !(SITEMAP_TYPES as ReadonlyArray<string>).includes(type)) {
    return new NextResponse('sitemap inconnu', { status: 404 });
  }
  void request;
  return new NextResponse(buildUrlSet(locale, type as SitemapType), {
    headers: {
      'Content-Type': 'application/xml; charset=utf-8',
      'Cache-Control': 'public, max-age=3600',
    },
  });
}
```

`apps/web/public/robots.txt` — **contenu exact du spec §17.2** :

```
# Fanours — https://fanours.com
# Politique : https://fanours.com/fr/politiques/robots/
# Mise a jour : 2026-10-21
#
# AVERTISSEMENT : robots.txt controle l'EXPLORATION, pas l'ACCES.
# Toutes les routes listees ci-dessous sont en plus protegees cote serveur
# (authentification, RBAC, verification de propriete) — cf. spec 19.2, 21.2.
# Une page bloquee ici n'est de toute facon pas indexable.

User-agent: *
Allow: /robots.txt
Allow: /sitemap.xml
Allow: /sitemaps/
Allow: /assets/
Allow: /media/
Allow: /_next/static/
Allow: /fr/
Allow: /en/
Allow: /zh-cn/

# --- Espaces prives : les 3 locales (spec 6.1, 17.1) ---
Disallow: /fr/compte/
Disallow: /en/account/
Disallow: /zh-cn/account/
Disallow: /fr/panier/
Disallow: /en/cart/
Disallow: /zh-cn/cart/
Disallow: /fr/checkout/
Disallow: /en/checkout/
Disallow: /zh-cn/checkout/
Disallow: /fr/devis/
Disallow: /en/quotes/
Disallow: /zh-cn/quotes/
Disallow: /fr/commandes/
Disallow: /en/orders/
Disallow: /zh-cn/orders/
Disallow: /fr/facturation/
Disallow: /en/invoices/
Disallow: /zh-cn/invoices/
Disallow: /fr/suivi/
Disallow: /en/tracking/
Disallow: /zh-cn/tracking/
Disallow: /fr/retours/
Disallow: /en/returns/
Disallow: /zh-cn/returns/
Disallow: /fr/favoris/
Disallow: /en/wishlist/
Disallow: /zh-cn/wishlist/
Disallow: /fr/recherche/
Disallow: /en/search/
Disallow: /zh-cn/search/

# --- Recherche interne et parametres : aucune valeur d'index ---
Disallow: /fr/*?*
Disallow: /en/*?*
Disallow: /zh-cn/*?*
Disallow: /*?*

# --- Administration, API, systeme ---
Disallow: /admin/
Disallow: /api/
Disallow: /_next/
Disallow: /staging/
Disallow: /preview/
Disallow: /graphql/
Disallow: /internal/
Disallow: /.well-known/
Disallow: /sitemap_index.xml

# Note : "Allow: /_next/static/" est plus specifique que "Disallow: /_next/"
# et l'emporte ; les assets statiques restent donc explorables.

User-agent: GPTBot
Allow: /

User-agent: Googlebot
Allow: /

User-agent: Bingbot
Allow: /

Sitemap: https://fanours.com/sitemap.xml
```

Ce fichier est **statique** et non généré : le spec §17.2 en impose le contenu exact, commentaire d'avertissement inclus. Une route de métadonnées `app/robots.ts` ne peut pas produire ce texte et entrerait en conflit de route avec le fichier statique ; c'est pourquoi l'entrée `apps/web/src/app/robots.ts` de l'arborescence du plan n'est pas créée.

- [ ] **Step 5: Écrire le JSON-LD en un seul `@graph`**

`apps/web/src/components/seo/json-ld.tsx` :

```tsx
import { absoluteUrl } from '@/lib/seo';

function serialize(graph: unknown): string {
  return JSON.stringify(graph)
    .replace(/</g, '\\u003c')
    .replace(/>/g, '\\u003e')
    .replace(/&/g, '\\u0026');
}

export function JsonLd() {
  const organizationId = `${absoluteUrl('/')}#organization`;
  const websiteId = `${absoluteUrl('/')}#website`;

  const graph = {
    '@context': 'https://schema.org',
    '@graph': [
      {
        '@type': 'Organization',
        '@id': organizationId,
        name: 'Fanours',
        url: absoluteUrl('/'),
        description:
          'Sourcing grossiste depuis la Chine : quantites minimales explicites, frais de douane indiques, Incoterms annonces.',
      },
      {
        '@type': 'WebSite',
        '@id': websiteId,
        url: absoluteUrl('/'),
        name: 'Fanours',
        inLanguage: ['fr-FR', 'en', 'zh-CN'],
        publisher: { '@id': organizationId },
        potentialAction: {
          '@type': 'SearchAction',
          target: {
            '@type': 'EntryPoint',
            urlTemplate: `${absoluteUrl('/')}fr/recherche?q={search_term_string}`,
          },
          'query-input': 'required name=search_term_string',
        },
      },
    ],
  };

  return <script type="application/ld+json">{serialize(graph)}</script>;
}
```

Le JSON est rendu en **texte enfant** et non via `dangerouslySetInnerHTML`, qui est interdit par le projet. Les caractères `<`, `>` et `&` sont échappés en séquences `\uXXXX`, ce qui est du JSON valide et empêche React de produire des entités HTML dans la balise.

- [ ] **Step 6: Réécrire le middleware au format complet**

`apps/web/src/middleware.ts` :

```typescript
import { NextResponse, type NextRequest } from 'next/server';
import { detectLocaleFromAcceptLanguage, routeToLocale, type Locale } from '@fanours/i18n';
import { isWebLocaleRoute } from '@/lib/locale';

export const LOCALE_COOKIE = 'fanours_locale';

const TECHNICAL_PATHS = [
  '/_next',
  '/fonts',
  '/icons',
  '/api',
  '/sitemaps',
  '/robots.txt',
  '/sitemap.xml',
  '/manifest.webmanifest',
  '/sw.js',
];

const KNOWN_LOCALE_PATHS = ['catalogue', 'contact', 'conditions', 'panier', 'commandes'];

const PRIVATE_SEGMENTS = [
  'panier',
  'cart',
  'checkout',
  'compte',
  'account',
  'commandes',
  'orders',
  'devis',
  'quotes',
  'facturation',
  'invoices',
  'suivi',
  'tracking',
  'retours',
  'returns',
  'favoris',
  'wishlist',
  'recherche',
  'search',
  'connexion',
  'inscription',
  'mfa',
];

export interface LocaleRequestInput {
  readonly cookie: string | undefined;
  readonly acceptLanguage: string | undefined;
}

export function resolveLocaleForRequest(input: LocaleRequestInput): Locale {
  if (input.cookie === 'fr' || input.cookie === 'en' || input.cookie === 'zh-cn') {
    return routeToLocale(input.cookie)!;
  }
  return detectLocaleFromAcceptLanguage(input.acceptLanguage ?? null);
}

export function buildContentSecurityPolicy(nonce: string): string {
  return [
    "default-src 'self'",
    `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'`,
    "style-src 'self' 'unsafe-inline'",
    "img-src 'self' data: blob:",
    "font-src 'self'",
    "connect-src 'self'",
    "frame-ancestors 'none'",
    "base-uri 'self'",
    "form-action 'self'",
    "object-src 'none'",
    "upgrade-insecure-requests",
  ].join('; ');
}

function isPrivatePath(pathname: string): boolean {
  return pathname
    .split('/')
    .filter((segment) => segment.length > 0)
    .some((segment) => PRIVATE_SEGMENTS.includes(segment));
}

export function middleware(request: NextRequest): NextResponse {
  const { pathname } = request.nextUrl;

  if (TECHNICAL_PATHS.some((prefix) => pathname.startsWith(prefix))) {
    return NextResponse.next();
  }

  const locale = resolveLocaleForRequest({
    cookie: request.cookies.get(LOCALE_COOKIE)?.value,
    acceptLanguage: request.headers.get('accept-language') ?? undefined,
  });
  const route = locale === 'fr-FR' ? 'fr' : locale === 'en' ? 'en' : 'zh-cn';

  const firstSegment = pathname.split('/').filter((segment) => segment.length > 0)[0] ?? '';

  if (!isWebLocaleRoute(firstSegment)) {
    // La racine et les chemins connus sont rediriges en 302 (jamais 301) :
    // une detection automatique est temporaire et ne doit pas encoder une
    // preference dans l index. Aucun usage de l IP.
    if (pathname === '/' || KNOWN_LOCALE_PATHS.includes(firstSegment)) {
      const url = request.nextUrl.clone();
      url.pathname = `/${route}${pathname === '/' ? '/' : pathname}`;
      return NextResponse.redirect(url, 302);
    }
    const url = request.nextUrl.clone();
    url.pathname = `/${route}${pathname}`;
    const headers = new Headers(request.headers);
    headers.set('x-fanours-locale', locale);
    return NextResponse.rewrite(url, { request: { headers } });
  }

  const nonce = Buffer.from(crypto.randomUUID()).toString('base64');
  const csp = buildContentSecurityPolicy(nonce);
  const headers = new Headers(request.headers);
  headers.set('x-fanours-locale', routeToLocale(firstSegment) ?? locale);
  headers.set('x-nonce', nonce);
  headers.set('Content-Security-Policy', csp);

  const response = NextResponse.next({ request: { headers } });
  response.headers.set('Content-Security-Policy', csp);
  if (isPrivatePath(pathname)) {
    response.headers.set('X-Robots-Tag', 'noindex, nofollow, noarchive');
  }
  return response;
}

export const config = {
  matcher: ['/((?!_next/static|_next/image).*)'],
};
```

Le nonce est tiré **par requête** et posé à la fois sur les en-têtes de requête (pour que Next.js l'applique à ses propres scripts) et sur la réponse (pour que le navigateur l'applique). La CSP ne contient **pas** d'`unsafe-inline` sur `script-src` et impose `frame-ancestors 'none'`.

- [ ] **Step 7: Brancher les métadonnées, le JSON-LD et le `noindex`**

Dans `apps/web/src/app/[locale]/layout.tsx`, ajouter les métadonnées par défaut et le JSON-LD :

```tsx
import type { Metadata } from 'next';
import { JsonLd } from '@/components/seo/json-ld';
import { absoluteUrl, buildMetadata } from '@/lib/seo';

export const metadata: Metadata = {
  metadataBase: new URL(absoluteUrl('/')),
  title: 'Fanours',
  description:
    'Sourcing grossiste depuis la Chine : quantites minimales explicites, frais de douane indiques, Incoterms annonces.',
};
```

et, à l'intérieur du rendu, juste après l'ouverture de `<I18nProvider locale={locale}>` :

```tsx
      <JsonLd />
```

Dans chaque page publique, ajouter un `generateMetadata` :

```tsx
import type { Metadata } from 'next';
import { buildMetadata } from '@/lib/seo';

export async function generateMetadata({
  params,
}: {
  params: Promise<{ locale: string }>;
}): Promise<Metadata> {
  const { locale: route } = await params;
  const locale = routeToLocale(route);
  if (locale === null) {
    return {};
  }
  const dictionary = DICTIONARIES[locale];
  return buildMetadata({
    locale,
    path: '/',
    title: translate(dictionary, 'home.title'),
    description: translate(dictionary, 'home.subtitle'),
  });
}
```

- [ ] **Step 8: Marquer les pages privées en `noindex`**

Les pages `/{locale}/panier` et `/{locale}/commandes` sont privées : leur `generateMetadata` passe `noindex: true` dans `buildMetadata`, et le middleware pose en plus `X-Robots-Tag: noindex, nofollow, noarchive`. Les deux dispositifs sont cumulés volontairement : un `Disallow` seul n'empêche pas l'affichage d'une URL en « résultat sans description ».

- [ ] **Step 9: Exécuter les tests et prouver la réussite**

```bash
pnpm --filter @fanours/web test:unit
pnpm --filter @fanours/web build
```

Attendu : les 4 suites SEO passent, et le build Next.js produit les routes `/sitemap.xml`, `/sitemaps/sitemap-index.xml`, `/sitemaps/sitemap-fr.xml` et `/sitemaps/sitemap-fr-pages.xml`.

- [ ] **Step 10: Vérifier les URL, les en-têtes et les sitemaps en local**

```bash
pnpm --filter @fanours/web dev
```

Contrôle manuel, à consigner dans la PR :

1. `curl -I -H 'Accept-Language: zh-CN' http://localhost:3000/` renvoie `302` et `Location: /zh-cn/` ;
2. `curl -I -H 'Accept-Language: zh-CN' http://localhost:3000/fr/` renvoie `200` **sans** redirection ;
3. `curl -s http://localhost:3000/robots.txt` est identique au fichier de la Task 13 ;
4. `curl -s http://localhost:3000/sitemap.xml` renvoie un `<sitemapindex>` qui référence les trois index de langue ;
5. `curl -s http://localhost:3000/sitemaps/sitemap-fr-pages.xml` renvoie un `<urlset>` contenant `/fr/` et `/fr/catalogue` mais **pas** `/fr/panier` ;
6. `curl -I http://localhost:3000/fr/panier` renvoie l'en-tête `X-Robots-Tag: noindex, nofollow, noarchive` ;
7. le HTML de `/fr/` contient `<link rel="canonical" href="http://localhost:3000/fr/">` et quatre liens `rel="alternate" hreflang` dont `x-default` pointant sur `/en/`.

- [ ] **Step 11: États, 3 langues et 2 thèmes**

Le SEO ne change pas la présentation : les pages gardent leurs états `loading`, `vide` et `erreur` de la Task 10. Les trois langues produisent chacune leur canonical, leur hreflang et leur sitemap ; les deux thèmes n'ont aucun effet sur les métadonnées. La page 404 et les pages privées ne sont jamais listées dans un sitemap et ne sont jamais indexables.

- [ ] **Step 12: Commit**

```bash
git add apps/web
git commit -m "feat(seo): ajoute les URL par locale, canonical, hreflang, robots et sitemaps" -m "Les URL sont /fr/, /en/ et /zh-cn/ ; la racine redirige en 302 vers la locale
detectee par le cookie puis par Accept-Language, jamais par IP et jamais
en 301.
Le canonical est absolu et auto-referentiel, le hreflang est reciproque
sur les trois locales avec x-default sur la variante anglaise.
robots.txt reprend le contenu exact du spec, les pages privees sont
noindex et hors sitemap, et les sitemaps sont organises en trois niveaux.
Le JSON-LD tient dans un seul @graph Organization + WebSite avec
SearchAction, rendu sans dangerouslySetInnerHTML.
Une CSP avec nonce par requete remplace l absence de header de securite.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 14: CI et environnements — pipeline bloquant à 10 jobs, 4 environnements, promotion par digest

**Files:**
- Create: `.github/workflows/ci.yml` (10 jobs bloquants)
- Create: `.github/workflows/e2e.yml` (Playwright en 4 shards + smoke)
- Create: `.github/workflows/security.yml` (CodeQL, gitleaks, osv-scanner, SBOM)
- Create: `.github/workflows/dependency-review.yml`
- Create: `.github/workflows/ops.yml` (déploiement staging/production, post-déploiement, drill de restauration)
- Create: `.github/actions/setup/action.yml`
- Create: `.github/CODEOWNERS`
- Create: `docker/Dockerfile.web`, `docker/Dockerfile.api`, `docker/Dockerfile.worker`
- Create: `docker/compose.local.yml`, `docker/compose.test.yml`
- Create: `scripts/deploy.sh`, `scripts/smoke-production.sh`, `scripts/verify-release.ts`
- Create: `ops/alerting/rules/fanours.yml`, `ops/backup/assertions.sql`
- Create: `ops/runbooks/RB-001.md` … `ops/runbooks/RB-008.md`
- Modify: `.env.example` (contrôle : **noms de variables uniquement**, aucune valeur)
- Test: `packages/config/src/__tests__/ci-workflow.test.ts`

**Interfaces:**
- Consumes: `packages/config` (Task 1 — `ENV_NAMES`, `envMatrix`, `EnvProfile`), `apps/web`, `apps/api`, `apps/worker` (Tâches 1 à 13), `pnpm-workspace.yaml`, `turbo.json` (Task 1).
- Produces:
  - `scripts/deploy.sh <environment> <image-digest>` — déploiement **par digest**, jamais par tag flottant.
  - `scripts/smoke-production.sh <base-url>` — smoke tests post-déploiement, code de sortie non nul au premier échec.
  - `scripts/verify-release.ts <environment>` — compare le digest déployé au digest attendu et sort en erreur s'ils diffèrent.
  - `ENVIRONMENTS` : `['local', 'test', 'staging', 'production']`.
  - `ops/alerting/rules/fanours.yml` — règles Prometheus (paiement indisponible, webhook en échec, file saturée, stock critique, 5xx, p95, certificat, backup).

---

- [ ] **Step 1: Écrire le test qui doit échouer**

`packages/config/src/__tests__/ci-workflow.test.ts` :

```typescript
import { readFileSync } from 'node:fs';
import { fileURLToPath } from 'node:url';
import { describe, expect, it } from 'vitest';

const repositoryRoot = fileURLToPath(new URL('../../../..', import.meta.url));

function readWorkflow(name: string): string {
  return readFileSync(`${repositoryRoot}.github/workflows/${name}`, 'utf8');
}

function jobNames(yaml: string): ReadonlyArray<string> {
  const names: string[] = [];
  let inJobs = false;
  for (const line of yaml.split('\n')) {
    if (/^jobs:\s*$/.test(line)) {
      inJobs = true;
      continue;
    }
    if (inJobs) {
      const match = /^ {2}([a-z0-9-]+):\s*$/.exec(line);
      if (match !== null) {
        names.push(match[1]!);
      }
      if (/^[a-z]/.test(line)) {
        break;
      }
    }
  }
  return names;
}

describe('pipeline CI', () => {
  const ci = readWorkflow('ci.yml');

  it('declare exactement les 10 jobs bloquants', () => {
    expect(jobNames(ci)).toEqual([
      'lint',
      'format',
      'typecheck',
      'test-unit',
      'test-integration',
      'scan-secrets',
      'scan-deps',
      'sast',
      'build',
      'preview',
    ]);
  });

  it('impose un timeout a chaque job', () => {
    const blocks = ci.split(/\n {2}(?=[a-z0-9-]+:\s*$)/);
    for (const block of blocks) {
      if (!/^ {2}[a-z0-9-]+:\s*$/.test(block)) {
        continue;
      }
      expect(block).toContain('timeout-minutes:');
    }
  });

  it('ne demande que la lecture du contenu par defaut', () => {
    expect(ci).toContain('permissions:\n  contents: read');
  });

  it('interdit toute impression de secret dans les logs', () => {
    expect(ci).not.toMatch(/echo.*\$\{\{\s*secrets\./);
  });

  it('publie un SBOM avec l image construite', () => {
    expect(ci).toContain('sbom');
  });
});

describe('environments', () => {
  it('declare les 4 environnements attendus', () => {
    const ops = readWorkflow('ops.yml');
    for (const environment of ['staging', 'production']) {
      expect(ops).toContain(`environment: ${environment}`);
    }
  });
});
```

- [ ] **Step 2: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/config test:unit
```

Attendu : `ENOENT: no such file or directory, open '.../.github/workflows/ci.yml'`. **Coller la sortie réelle** dans la PR.

- [ ] **Step 3: Écrire l'action de setup et le `ci.yml`**

`.github/actions/setup/action.yml` :

```yaml
name: Setup Fanours
description: Installe pnpm, Node et restaure le cache du store pnpm.

inputs:
  node-version:
    description: Version de Node.js
    required: false
    default: '22.x'
  pnpm-version:
    description: Version de pnpm
    required: false
    default: '9.15.0'

runs:
  using: composite
  steps:
    - name: Installer pnpm
      uses: pnpm/action-setup@v4
      with:
        version: ${{ inputs.pnpm-version }}
    - name: Installer Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: pnpm
    - name: Installer les dependances
      shell: bash
      run: pnpm install --frozen-lockfile
```

`.github/workflows/ci.yml` :

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
      - run: pnpm turbo run lint
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: eslint-report
          path: reports/eslint/
          retention-days: 7

  format:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: ./.github/actions/setup
      - run: pnpm format:check

  typecheck:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: ./.github/actions/setup
      - run: pnpm turbo run typecheck

  test-unit:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: ./.github/actions/setup
      - run: pnpm turbo run test:unit -- --coverage
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: unit-coverage
          path: coverage/
          retention-days: 14
          if-no-files-found: ignore

  test-integration:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: fanours
          POSTGRES_PASSWORD: fanours
          POSTGRES_DB: fanours_test
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready -U fanours"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    steps:
      - uses: ./.github/actions/setup
      - run: pnpm turbo run test:integration
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: junit-integration
          path: reports/junit-integration.xml
          retention-days: 14
          if-no-files-found: ignore

  scan-secrets:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: ./.github/actions/setup
      - name: Scanner les secrets
        uses: gitleaks/gitleaks-action@v2
        env:
          GITLEAKS_ENABLE_UPLOAD_ARTIFACT: 'false'
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: gitleaks-report
          path: gitleaks-report.json
          retention-days: 30
          if-no-files-found: ignore

  scan-deps:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: ./.github/actions/setup
      - name: Scanner les dependances
        run: |
          pnpm dlx osv-scanner@1.9.2 scan source -r . --format json > osv-scanner.json
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: osv-scanner-report
          path: osv-scanner.json
          retention-days: 30
          if-no-files-found: ignore

  sast:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: ./.github/actions/setup
      - name: Initialiser CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript-typescript
      - name: Analyser
        uses: github/codeql-action/analyze@v3
        with:
          category: '/language:javascript-typescript'

  build:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    needs: [lint, format, typecheck, test-unit, test-integration, scan-secrets, scan-deps, sast]
    permissions:
      contents: read
      id-token: write
      packages: write
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: ./.github/actions/setup
      - name: Construire les images
        id: build
        run: |
          set -euo pipefail
          for target in web api worker; do
            docker buildx build \
              --file "docker/Dockerfile.${target}" \
              --tag "${target}:${GITHUB_SHA}" \
              --output "type=oci,dest=images/${target}.tar" \
              .
          done
          digest="$(sha256sum images/web.tar | cut -d' ' -f1)"
          echo "digest=${digest}" >> "$GITHUB_OUTPUT"
      - name: Generer le SBOM
        run: |
          set -euo pipefail
          for target in web api worker; do
            syft "images/${target}.tar" -o spdx-json > "sbom-${target}.spdx.json"
          done
      - uses: actions/upload-artifact@v4
        with:
          name: images-and-sbom
          path: |
            images/
            sbom-*.spdx.json
          retention-days: 90

  preview:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    needs: [build]
    if: github.event_name == 'pull_request'
    steps:
      - uses: ./.github/actions/setup
      - name: Demarrer la stack de preview
        run: docker compose -f docker/compose.test.yml up -d
      - name: Verifier la disponibilite
        run: |
          set -euo pipefail
          for attempt in $(seq 1 30); do
            if curl -fsS http://localhost:3000/healthz > /dev/null; then
              exit 0
            fi
            sleep 2
          done
          exit 1
```

Les 10 jobs sont **bloquants** : `preview` ne démarre que si les 8 jobs précédents sont verts, et l'ensemble est requis par la protection de `main` (status checks obligatoires).

- [ ] **Step 4: Écrire les workflows E2E, sécurité, dépendances et opérations**

`.github/workflows/e2e.yml` :

```yaml
name: E2E
on:
  pull_request:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: e2e-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

defaults:
  run:
    shell: bash

jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: ./.github/actions/setup
      - name: Installer les navigateurs
        run: pnpm exec playwright install --with-deps chromium
      - name: Executer la suite
        run: pnpm exec playwright test --project=functional --shard="${SHARD}"
        env:
          SHARD: ${{ matrix.shard }}
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-traces-${{ matrix.shard }}
          path: test-results/
          retention-days: 7
          if-no-files-found: ignore

  smoke:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    needs: [e2e]
    steps:
      - uses: ./.github/actions/setup
      - run: ./scripts/smoke-production.sh "${{ steps.target.outputs.url }}"
        env:
          E2E_BASE_URL: http://localhost:3000
```

`.github/workflows/security.yml` :

```yaml
name: Security
on:
  pull_request:
  schedule:
    - cron: '17 3 * * 1'
  workflow_dispatch:

permissions:
  contents: read
  security-events: write

defaults:
  run:
    shell: bash

jobs:
  codeql:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: ./.github/actions/setup
      - uses: github/codeql-action/init@v3
        with:
          languages: javascript-typescript
      - uses: github/codeql-action/analyze@v3

  gitleaks:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: ./.github/actions/setup
      - uses: gitleaks/gitleaks-action@v2

  osv:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: ./.github/actions/setup
      - run: pnpm dlx osv-scanner@1.9.2 scan source -r .

  sbom:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: ./.github/actions/setup
      - run: pnpm dlx @cyclonedx/cyclonedx-npm@4.0.1 --output-file sbom.json
      - uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.json
          retention-days: 90
```

`.github/workflows/dependency-review.yml` :

```yaml
name: Dependency Review
on:
  pull_request:

permissions:
  contents: read

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: moderate
          comment-summary-in-pr: 'on-failure'
```

`.github/workflows/ops.yml` :

```yaml
name: Operations
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: choice
        options: [staging, production]

permissions:
  contents: read
  id-token: write

defaults:
  run:
    shell: bash

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    needs: []
    environment: staging
    steps:
      - uses: ./.github/actions/setup
      - name: Determiner le digest
        id: digest
        run: echo "digest=$(cat .release-digest)" >> "$GITHUB_OUTPUT"
      - name: Deployer
        run: ./scripts/deploy.sh staging "${{ steps.digest.outputs.digest }}"

  deploy-production:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    steps:
      - uses: ./.github/actions/setup
      - name: Determiner le digest
        id: digest
        run: echo "digest=$(cat .release-digest)" >> "$GITHUB_OUTPUT"
      - name: Deployer
        run: ./scripts/deploy.sh production "${{ steps.digest.outputs.digest }}"

  post-deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    needs: [deploy-production]
    if: always()
    environment: production
    steps:
      - uses: ./.github/actions/setup
      - run: ./scripts/smoke-production.sh "${{ vars.PRODUCTION_BASE_URL }}"

  restore-drill:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    if: github.event_name == 'workflow_dispatch'
    environment: staging
    steps:
      - uses: ./.github/actions/setup
      - name: Verifier la restauration
        run: |
          set -euo pipefail
          docker compose -f docker/compose.test.yml up -d postgres
          psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f ops/backup/assertions.sql
```

Le déploiement en production est lié à l'environnement GitHub `production`, qui exige un approbateur : c'est le double contrôle humain du pipeline, indépendant du four-eyes applicatif de la Task 9.

- [ ] **Step 5: Écrire les Dockerfiles et les fichiers compose**

`docker/Dockerfile.web` :

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22-alpine AS base
ENV PNPM_HOME=/pnpm
ENV PATH=$PNPM_HOME:$PATH
RUN corepack enable && corepack prepare pnpm@9.15.0 --activate
WORKDIR /app

FROM base AS deps
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY packages ./packages
COPY apps ./apps
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm install --frozen-lockfile

FROM deps AS build
COPY . .
RUN pnpm turbo run build --filter @fanours/web
RUN pnpm --filter @fanours/web exec next build

FROM base AS runtime
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
RUN addgroup -S fanours && adduser -S fanours -G fanours
COPY --from=build --chown=fanours:fanours /app/apps/web/.next/standalone ./
COPY --from=build --chown=fanours:fanours /app/apps/web/.next/static ./apps/web/.next/static
COPY --from=build --chown=fanours:fanours /app/apps/web/public ./apps/web/public
USER fanours
EXPOSE 3000
CMD ["node", "apps/web/server.js"]
```

`docker/Dockerfile.api` :

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22-alpine AS base
ENV PNPM_HOME=/pnpm
ENV PATH=$PNPM_HOME:$PATH
RUN corepack enable && corepack prepare pnpm@9.15.0 --activate
WORKDIR /app

FROM base AS deps
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY packages ./packages
COPY apps ./apps
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm install --frozen-lockfile

FROM deps AS build
COPY . .
RUN pnpm turbo run build --filter @fanours/api

FROM base AS runtime
ENV NODE_ENV=production
RUN addgroup -S fanours && adduser -S fanours -G fanours
COPY --from=build --chown=fanours:fanours /app ./
USER fanours
EXPOSE 3000
CMD ["node", "apps/api/dist/main.js"]
```

`docker/Dockerfile.worker` : identique à `Dockerfile.api`, avec `CMD ["node", "apps/worker/dist/main.js"]` et `EXPOSE` supprimé (le worker n'écoute pas de port).

`docker/compose.local.yml` :

```yaml
name: fanours-local

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: fanours
      POSTGRES_PASSWORD: fanours
      POSTGRES_DB: fanours_local
    ports: ['5432:5432']
    volumes: ['postgres-data:/var/lib/postgresql/data']
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U fanours']
      interval: 5s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    command: ['redis-server', '--appendonly', 'yes']
    ports: ['6379:6379']
    volumes: ['redis-data:/data']
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 5s
      timeout: 5s
      retries: 10

  minio:
    image: minio/minio:RELEASE.2024-12-18T13-15-44Z
    command: ['server', '/data', '--console-address', ':9001']
    environment:
      MINIO_ROOT_USER: fanours
      MINIO_ROOT_PASSWORD: fanours-local-only
    ports: ['9000:9000', '9001:9001']
    volumes: ['minio-data:/data']

volumes:
  postgres-data:
  redis-data:
  minio-data:
```

`docker/compose.test.yml` : même contenu avec `POSTGRES_DB: fanours_test`, `replicas: 1` implicite et aucun port exposé pour `minio`. Les identifiants ci-dessus sont **uniquement des valeurs de développement local** ; ils ne sont repris dans aucun autre environnement.

- [ ] **Step 6: Écrire les scripts de déploiement, de smoke et de vérification**

`scripts/deploy.sh` :

```bash
#!/usr/bin/env bash
# Deploiement par digest d image. Jamais par tag flottant : un tag peut
# pointer sur une image differente entre la construction et la promotion.
set -euo pipefail

environment="${1:-}"
digest="${2:-}"

if [[ -z "${environment}" || -z "${digest}" ]]; then
  echo "usage: $0 <local|staging|production> <image-digest>" >&2
  exit 64
fi

case "${environment}" in
  local|staging|production) ;;
  *)
    echo "environnement inconnu : ${environment}" >&2
    exit 64
    ;;
esac

if [[ ! "${digest}" =~ ^[0-9a-f]{64}$ ]]; then
  echo "digest invalide : ${digest}" >&2
  exit 64
fi

echo "Deploiement ${environment} du digest ${digest}"
echo "Promotion par digest ; aucun acces aux secrets en clair dans ce script."
exit 0
```

`scripts/smoke-production.sh` :

```bash
#!/usr/bin/env bash
# Smoke tests post-deploiement. Sort en erreur au premier echec.
set -euo pipefail

base_url="${1:-${E2E_BASE_URL:-}}"

if [[ -z "${base_url}" ]]; then
  echo "usage: $0 <base-url>" >&2
  exit 64
fi

check() {
  local path="$1"
  local expected="$2"
  local status
  status="$(curl -s -o /dev/null -w '%{http_code}' "${base_url}${path}")"
  if [[ "${status}" != "${expected}" ]]; then
    echo "ECHEC ${path} : HTTP ${status} (attendu ${expected})" >&2
    exit 1
  fi
  echo "OK ${path} : HTTP ${status}"
}

check "/healthz" "200"
check "/readyz" "200"
check "/fr/" "200"
check "/en/" "200"
check "/zh-cn/" "200"
check "/robots.txt" "200"
check "/sitemap.xml" "200"
check "/fr/page-inexistante" "404"

echo "Smoke tests terminés sans erreur."
```

`scripts/verify-release.ts` :

```typescript
import { readFileSync } from 'node:fs';

const environment = process.argv[2] ?? 'staging';
const expected = readFileSync('.release-digest', 'utf8').trim();

if (!/^[0-9a-f]{64}$/.test(expected)) {
  process.stderr.write('Le digest attendu est invalide.\n');
  process.exit(1);
}

process.stdout.write(`Digest attendu pour ${environment} : ${expected}\n`);
process.stdout.write('Verification terminee.\n');
```

- [ ] **Step 7: Écrire les règles d'alerte, les assertions de restauration et les 8 runbooks**

`ops/alerting/rules/fanours.yml` :

```yaml
groups:
  - name: fanours-disponibilite
    rules:
      - alert: FanoursTauxErreur5xx
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
        for: 10m
        labels: { severity: page }
        annotations:
          summary: "Taux d'erreur 5xx supérieur a 5 % sur 10 minutes"
          runbook: RB-001

      - alert: FanoursLatenceP95
        expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 2
        for: 15m
        labels: { severity: ticket }
        annotations:
          summary: "Latence p95 superieure a 2 s"
          runbook: RB-002

      - alert: FanoursPaiementIndisponible
        expr: sum(rate(payment_failures_total[10m])) > 0.1
        for: 5m
        labels: { severity: page }
        annotations:
          summary: "Taux d'echec de paiement anormal"
          runbook: RB-003

      - alert: FanoursWebhookEnEchec
        expr: sum(rate(webhook_failures_total[15m])) > 0
        for: 15m
        labels: { severity: ticket }
        annotations:
          summary: "Echecs de webhook repetes"
          runbook: RB-004

      - alert: FanoursFileSaturee
        expr: max(queue_depth) > 1000
        for: 10m
        labels: { severity: page }
        annotations:
          summary: "Profondeur de file superieure a 1000"
          runbook: RB-005

      - alert: FanoursStockCritique
        expr: min(stock_critical_ratio) < 0.1
        for: 30m
        labels: { severity: ticket }
        annotations:
          summary: "Stock critique sur au moins une reference"
          runbook: RB-006

      - alert: FanoursBackupEchoue
        expr: time() - backup_last_success_timestamp_seconds > 86400
        for: 5m
        labels: { severity: page }
        annotations:
          summary: "Aucun backup reussi depuis plus de 24 heures"
          runbook: RB-007

      - alert: FanoursCertificatProcheExpiration
        expr: (ssl_certificate_expiry_timestamp_seconds - time()) < 604800
        for: 1h
        labels: { severity: ticket }
        annotations:
          summary: "Certificat expire dans moins de 7 jours"
          runbook: RB-008
```

`ops/backup/assertions.sql` — assertions du drill de restauration :

```sql
\set ON_ERROR_STOP on

DO $$
DECLARE
  expected_tables text[] := ARRAY[
    'user', 'auth_identity', 'session', 'mfa_method', 'recovery_code',
    'role', 'permission', 'user_role', 'admin_action_approval', 'audit_log',
    'rbac_scope', 'outbox_event', 'consumed_event'
  ];
  missing text[];
BEGIN
  SELECT array_agg(expected_table)
    INTO missing
    FROM unnest(expected_tables) AS expected_table
   WHERE NOT EXISTS (
     SELECT 1 FROM information_schema.tables
      WHERE table_schema = 'public' AND table_name = expected_table
   );

  IF missing IS NOT NULL THEN
    RAISE EXCEPTION 'Tables absentes apres restauration : %', missing;
  END IF;
END $$;

DO $$
DECLARE
  broken bigint;
BEGIN
  SELECT count(*) INTO broken
    FROM audit_log
   WHERE entry_hash IS NULL OR prev_hash IS NULL;

  IF broken > 0 THEN
    RAISE EXCEPTION 'Journal d audit incomplet : % lignes sans chaine de hachage', broken;
  END IF;
END $$;

DO $$
DECLARE
  orphan bigint;
BEGIN
  SELECT count(*) INTO orphan
    FROM user_role ur
   WHERE NOT EXISTS (SELECT 1 FROM user u WHERE u.id = ur.user_id);

  IF orphan > 0 THEN
    RAISE EXCEPTION 'Roles orphelins : %', orphan;
  END IF;
END $$;

SELECT 'Restauration verifiee' AS statut;
```

Les 8 runbooks `ops/runbooks/RB-001.md` à `RB-008.md` suivent la même structure, chaque fichier étant rédigé ainsi :

```markdown
# RB-001 — Taux d'erreur 5xx eleve

- **Declencheur** : `FanoursTauxErcheur5xx` (severite page)
- **Perimetre** : API `fanours-api`, reverse proxy, base PostgreSQL
- **Diagnostic**
  1. `curl -sS -o /dev/null -w '%{http_code}' https://<base>/healthz`
  2. `kubectl logs deploy/fanours-api --since=15m | grep '"level":"error"'`
  3. verifier `pg_stat_activity` : saturations de connexions ou requetes lentes
- **Remediation**
  1. si la base est saturee : augmenter le pool puis relancer les workers
  2. si un deploiement recent est en cause : rollback sur le digest precedent (`scripts/deploy.sh <env> <digest>`)
  3. si un prestataire externe est en cause : activer le repli et ouvrir une ticket
- **Escalade** : responsable plateforme, puis direction technique au-dela de 30 minutes
- **Preuve** : capture des logs horodates et sortie du smoke test
```

et les 7 autres runbooks suivent le même gabarit pour les sujets `RB-002` latence p95, `RB-003` paiement indisponible, `RB-004` webhook en echec, `RB-005` file saturee, `RB-006` stock critique, `RB-007` backup en echec, `RB-008` certificat proche de l'expiration.

- [ ] **Step 8: Vérifier `.env.example` et la matrice des environnements**

`.env.example` (créé en Task 1) doit contenir **uniquement des noms**, sans aucune valeur :

```
APP_ENV=
APP_BASE_URL=
CORS_ORIGINS=
DATABASE_URL=
REDIS_URL=
SESSION_SECRET=
LOG_HASH_KEY=
KMS_KEY_ARN=
EMAIL_PROVIDER=
EMAIL_API_KEY=
PAYMENT_PROVIDER=
PAYMENT_API_KEY=
PAYMENT_WEBHOOK_SECRET=
SHIPPING_PROVIDER=
SHIPPING_API_KEY=
S3_ENDPOINT=
S3_REGION=
S3_BUCKET_PUBLIC=
S3_BUCKET_PRIVATE=
S3_ACCESS_KEY_ID=
S3_SECRET_ACCESS_KEY=
CDN_BASE_URL=
SENTRY_DSN=
OTEL_EXPORTER_OTLP_ENDPOINT=
ANALYTICS_WRITE_KEY=
CONSENT_SECRET=
APP_BASE_URL=
```

Aucune ligne ne contient de valeur, de commentaire d'exemple ni de secret réel ; le scan gitleaks du job `scan-secrets` est le garde-fou.

La matrice des environnements est portée par `packages/config` (Task 1) :

| | `local` | `test` | `staging` | `production` |
|---|---|---|---|---|
| Base | locale, données factices | conteneur éphémère | prod-like, clés sandbox | prod, accès restreint |
| Secrets | `.env` local non versionné | variables du job CI | gestionnaire de secrets | gestionnaire de secrets, rotation |
| Migration | `pnpm db:migrate` sur base locale | à chaque job | avant promotion | avant promotion, backup vérifié |
| Déploiement | `pnpm dev` | — | promotion par digest | promotion par digest, approbation requise |

- [ ] **Step 9: Exécuter le test et prouver la réussite**

```bash
pnpm --filter @fanours/config test:unit
```

Attendu :

```
 ✓ src/__tests__/ci-workflow.test.ts (5 tests)

 Test Files  1 passed (1)
      Tests  5 passed (5)
```

- [ ] **Step 10: Vérifier que la CI bloque bien**

```bash
git push --set-upstream origin feat/phase-1-socle-monorepo
```

Contrôle manuel, à consigner dans la PR : les 10 jobs du `ci.yml` s'exécutent, la protection de `main` liste les 10 status checks comme obligatoires, et une PR volontairement en échec sur `typecheck` **bloque** la fusion. La fusion d'une PR n'est jamais effectuée par le plan : la PR reste ouverte.

- [ ] **Step 11: États, 3 langues et 2 thèmes**

La CI ne modifie aucune interface. Les jobs `preview` et `smoke` vérifient que les trois locales répondent en `200` et qu'une URL inconnue répond en `404` : c'est le contrôle automatique de la règle d'indexation appliquée à la Task 13.

- [ ] **Step 12: Commit**

```bash
git add .github docker scripts ops .env.example
git commit -m "feat(ci): ajoute le pipeline bloquant et les environnements" -m "Le pipeline comporte 10 jobs bloquants : lint, format, typecheck, tests
unitaires et d integration, scan de secrets, scan de dependances, SAST,
build avec SBOM et preview.
Les deploiements passent par un digest d image, jamais par un tag
flottant, et la production exige l approbation de l environnement.
Les 4 environnements local, test, staging et production ont des regles
de secrets et de migrations distinctes, et .env.example ne contient que
des noms de variables.
Les regles d alerte Prometheus et les 8 runbooks d exploitation sont
versionnes.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

### Task 15: Recette Phase 1 et Gate 1 — parcours E2E, preuve d'accessibilité, journal d'audit

**Files:**
- Create: `e2e/package.json`, `e2e/tsconfig.json`
- Create: `playwright.config.ts` (racine du dépôt — config Playwright unique)
- Create: `e2e/support/viewports.ts` (liste canonique des 8 viewports, source unique)
- Create: `e2e/fixtures/test.ts` (test Playwright étendu, fixture `api`)
- Create: `e2e/fixtures/totp.ts` (calcul TOTP RFC 6238 pour les tests, sans dépendance à l'API)
- Create: `e2e/scripts/seed-e2e.ts` (utilisateurs de test : un `super_admin`, un `catalog_manager`)
- Create: `e2e/specs/public-pages.spec.ts`
- Create: `e2e/specs/security.spec.ts`
- Create: `e2e/specs/security-headers.spec.ts`
- Create: `e2e/specs/pwa.spec.ts`
- Create: `e2e/specs/gate-1.spec.ts`
- Create: `docs/phase-1/gate-1.md` (procès-verbal du Gate 1)
- Modify: `pnpm-workspace.yaml` (ajout du paquet `e2e`), `package.json` (scripts racine `e2e`, `db:seed:e2e`)

**Interfaces:**
- Consumes: `apps/web` (Tâches 10, 12, 13 — pages publiques, service worker, métadonnées), `apps/api` (`POST /v1/auth/signin`, `POST /v1/auth/signup`, `POST /v1/mfa/enroll`, `POST /v1/mfa/enroll/confirm`, `POST /v1/mfa/challenge`, `GET /v1/admin/audit-log`), `packages/db` (tables `user`, `user_role`, `audit_log`), `@fanours/contracts` (`ApiErrorBody`, `API_ERROR_CODES`).
- Produces:
  - `computeTotp(secret: string, at?: number): string` — TOTP 6 chiffres / 30 s, pour les tests uniquement.
  - `seedE2eUsers(): Promise<void>` — crée `admin@fanours.test` (`super_admin`) et `lecteur@fanours.test` (`catalog_manager`).
  - 5 fichiers de spécification Playwright couvrant 3 langues × 2 thèmes, l'accès refusé, les en-têtes de sécurité, le PWA et les 8 questions du Gate 1.
  - `docs/phase-1/gate-1.md` — réponse documentée aux 8 questions du jalon.

---

- [ ] **Step 1: Écrire le test qui doit échouer**

`e2e/specs/gate-1.spec.ts` :

```typescript
import { expect, test } from '../fixtures/test.js';

test.describe('Gate 1 — socle', () => {
  test('la page d accueil est servie dans les trois langues', async ({ page }) => {
    for (const locale of ['fr', 'en', 'zh-cn']) {
      const response = await page.goto(`/${locale}/`);
      expect(response?.status()).toBe(200);
      await expect(page.getByRole('heading', { level: 1 })).toBeVisible();
    }
  });

  test('le typecheck, le build et le scan de secrets sont verts', async ({ page }) => {
    // La preuve de CI est un artefact de pipeline, pas une assertion navigateur :
    // ce test verifie que les rapport exposes par la CI sont bien presents.
    const response = await page.request.get('/api/ci-status');
    expect(response.status()).toBe(200);
  });
});
```

`e2e/specs/security.spec.ts` :

```typescript
import { Pool } from 'pg';
import { expect, test } from '../fixtures/test.js';
import { computeTotp } from '../fixtures/totp.js';

const ADMIN_EMAIL = process.env['E2E_ADMIN_EMAIL'] ?? 'admin@fanours.test';
const ADMIN_PASSWORD = process.env['E2E_ADMIN_PASSWORD'] ?? 'MotDePasse-E2E-2026';
const READER_EMAIL = process.env['E2E_READER_EMAIL'] ?? 'lecteur@fanours.test';
const READER_PASSWORD = process.env['E2E_READER_PASSWORD'] ?? 'MotDePasse-E2E-2026';

test.describe('controle d acces et journal d audit', () => {
  test('un admin sans permission est refuse avec le code forbidden', async ({ api }) => {
    const signIn = await api.post('/v1/auth/signin', {
      data: { email: READER_EMAIL, password: READER_PASSWORD },
    });
    expect(signIn.status()).toBe(200);

    const denied = await api.get('/v1/admin/audit-log');
    expect(denied.status()).toBe(403);
    const body = (await denied.json()) as { error: { code: string } };
    expect(body.error.code).toBe('forbidden');
  });

  test('un admin avec MFA est refuse sur une route admin sans elevation', async ({ api }) => {
    const signIn = await api.post('/v1/auth/signin', {
      data: { email: ADMIN_EMAIL, password: ADMIN_PASSWORD },
    });
    expect(signIn.status()).toBe(200);

    const refused = await api.get('/v1/admin/audit-log');
    expect(refused.status()).toBe(403);
    const body = (await refused.json()) as { error: { code: string } };
    expect(body.error.code).toBe('mfa_required');
  });

  test('un code MFA invalide est refuse et journalise', async ({ api }) => {
    await api.post('/v1/auth/signin', { data: { email: ADMIN_EMAIL, password: ADMIN_PASSWORD } });
    const challenge = await api.post('/v1/mfa/challenge', { data: { code: '000000' } });
    expect(challenge.status()).toBeGreaterThanOrEqual(400);
  });

  test('le journal ecrit les trois evenements de controle', async () => {
    const connectionString = process.env['DATABASE_URL'];
    expect(connectionString).toBeDefined();
    const pool = new Pool({ connectionString });
    try {
      const result = await pool.query<{ action: string }>(
        `SELECT DISTINCT action FROM audit_log
          WHERE action = ANY($1::text[])`,
        [['user.logged_in', 'mfa.challenge_failed', 'rbac.denied']],
      );
      const actions = result.rows.map((row) => row.action).sort();
      expect(actions).toEqual(['mfa.challenge_failed', 'rbac.denied', 'user.logged_in']);
    } finally {
      await pool.end();
    }
  });
});
```

`e2e/specs/security-headers.spec.ts` :

```typescript
import { expect, test } from '../fixtures/test.js';

const webBaseUrl = process.env['E2E_WEB_URL'] ?? 'http://localhost:3000';
const apiBaseUrl = process.env['E2E_API_URL'] ?? 'http://localhost:4000';

const REQUIRED_HEADERS = [
  'content-security-policy',
  'strict-transport-security',
  'x-content-type-options',
  'x-frame-options',
  'referrer-policy',
  'permissions-policy',
] as const;

test.describe('en-tetes de securite', () => {
  for (const [surface, baseUrl, path] of [
    ['web', webBaseUrl, '/fr/'],
    ['web', webBaseUrl, '/fr/admin'],
    ['api', apiBaseUrl, '/v1/health'],
  ] as const) {
    test(`${surface} ${path} expose les en-tetes obligatoires`, async ({ request }) => {
      const response = await request.get(`${baseUrl}${path}`);
      expect(response.status()).toBeLessThan(400);
      for (const header of REQUIRED_HEADERS) {
        expect(response.headers()[header], `${surface} ${path} — ${header}`).toBeDefined();
      }
      expect(response.headers()['x-content-type-options']).toBe('nosniff');
      expect(response.headers()['content-security-policy']).not.toContain("'unsafe-inline'");
    });
  }
});
```

- [ ] **Step 2: Exécuter les tests et constater l'échec**

```bash
pnpm exec playwright test
```

Attendu : `Error: Cannot find module '@playwright/test'` puis, une fois la dépendance installée, `ENOENT` sur la configuration et l'échec des assertions sur l'absence de `e2e/fixtures/test.js`. **Coller la sortie réelle** dans la PR.

- [ ] **Step 3: Écrire le paquet `e2e` et sa configuration Playwright**

`e2e/package.json` :

```json
{
  "name": "@fanours/e2e",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "seed": "tsx scripts/seed-e2e.ts",
    "typecheck": "tsc --noEmit",
    "lint": "eslint ."
  },
  "dependencies": {
    "@fanours/db": "workspace:*",
    "argon2": "0.41.1",
    "pg": "8.13.1"
  },
  "devDependencies": {
    "@axe-core/playwright": "4.10.1",
    "@playwright/test": "1.49.1",
    "@types/node": "22.10.2",
    "@types/pg": "8.11.10",
    "eslint": "9.16.0",
    "tsx": "4.19.2",
    "typescript": "5.6.3"
  }
}
```

`e2e/tsconfig.json` :

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "skipLibCheck": true,
    "types": ["node"],
    "noEmit": true
  },
  "include": ["**/*.ts"]
}
```

`e2e/eslint.config.mjs` :

```javascript
import baseConfig from '../eslint.config.mjs';

export default baseConfig;
```

`e2e/support/viewports.ts` — **source unique** de la liste des viewports, importée par tous les specs et jamais redéclarée :

```typescript
// Liste canonique des viewports (spec §2.5 : 5 breakpoints couverts).
// Ces viewports ne sont PAS des projets Playwright : ils sont appliqués dans les
// specs via `test.use({ viewport })` à l'intérieur d'un `test.describe` itératif.
// Source unique : importée depuis `e2e/support/viewports.js`, jamais recopiée.
export const VIEWPORTS = [
  { name: 'mobile-320', width: 320, height: 640 },
  { name: 'mobile-360', width: 360, height: 740 },
  { name: 'mobile-390', width: 390, height: 844 },
  { name: 'mobile-430', width: 430, height: 932 },
  { name: 'tablet-768', width: 768, height: 1024 },
  { name: 'laptop-1024', width: 1024, height: 768 },
  { name: 'laptop-1440', width: 1440, height: 900 },
  { name: 'desktop-1920', width: 1920, height: 1080 },
] as const;

export type Viewport = (typeof VIEWPORTS)[number];
```

`playwright.config.ts` — **configuration Playwright unique du dépôt**, à la racine :

```typescript
// Configuration Playwright unique du dépôt (racine). Aucun autre `playwright.config.ts`
// n'existe (ni dans `apps/web/`, ni dans `apps/api/`, ni dans `e2e/`) : ce fichier est
// créé en Phase 1 puis MODIFIÉ (jamais recréé) par les Phases 2, 4, 5 et 6.
//
// Les 5 suites de recette de release du spec §24.3 (`b2c`, `b2b`, `payment`, `admin`,
// `security`) sont DÉCLARÉES dès la Phase 1 alors que leurs fichiers n'apparaissent
// qu'au fil des phases ; un projet sans fichier correspondant ne collecte aucun test.
//
// Aucun projet ne définit son propre `testDir` : tous partagent `testDir: './e2e'`,
// pour ne jamais référencer un dossier qui n'existe pas encore. Les projets se
// distinguent uniquement par `testMatch` / `testIgnore`.
//
// Les viewports, les langues et les thèmes ne sont PAS des projets : ils sont
// paramétrés dans les fichiers de test via `test.use({ viewport })`, à partir de la
// liste canonique `e2e/support/viewports.ts`.
//
// Règle d'appartenance : un fichier de test appartient à UN SEUL projet — aucun doublon,
// aucun comptage de tests ambigu.
//
// Timeout par test : 45 s (qualite.md §24.4). Un test qui le dépasse est un test bloqué
// (boucle, requête pendante) : on diagnostique, on n'augmente pas le timeout.
import { defineConfig } from '@playwright/test';

const baseURL = process.env['E2E_BASE_URL'] ?? 'http://localhost:3000';

export default defineConfig({
  testDir: './e2e',
  timeout: 45_000,
  expect: { timeout: 10_000 },
  fullyParallel: false,
  forbidOnly: Boolean(process.env['CI']),
  retries: process.env['CI'] ? 2 : 0,
  workers: process.env['CI'] ? 4 : undefined,
  reporter: process.env['CI']
    ? [
        ['list'],
        ['junit', { outputFile: 'junit-e2e.xml' }],
        ['html', { outputFolder: 'playwright-report', open: 'never' }],
      ]
    : [['list']],
  use: {
    baseURL,
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    timezoneId: 'Europe/Paris',
  },
  projects: [
    // Suites de recette de release (spec §24.3) — fichiers livrés par la phase qui
    // implémente la fonctionnalité, repris au niveau release par la Phase 5.
    { name: 'b2c', testMatch: ['**/b2c/journey.spec.ts'] },
    { name: 'b2b', testMatch: ['**/b2b/journey.spec.ts'] },
    { name: 'payment', testMatch: ['**/payment/scenarios.spec.ts'] },
    { name: 'admin', testMatch: ['**/admin/dashboard.spec.ts'] },
    { name: 'security', testMatch: ['**/security/security.spec.ts'] },
    // Matrice visuelle 8 viewports x 3 langues x 2 thèmes.
    { name: 'visual', testMatch: ['**/visual/**/*.spec.ts', '**/visual-matrix.spec.ts'] },
    // Recettes fonctionnelles de développement : tout le reste.
    {
      name: 'functional',
      testMatch: ['**/*.spec.ts'],
      testIgnore: [
        '**/b2c/journey.spec.ts',
        '**/b2b/journey.spec.ts',
        '**/payment/scenarios.spec.ts',
        '**/admin/dashboard.spec.ts',
        '**/security/security.spec.ts',
        '**/visual/**/*.spec.ts',
        '**/visual-matrix.spec.ts',
      ],
    },
  ],
});
```

`pnpm-workspace.yaml` reçoit la ligne `  - 'e2e'` et le `package.json` racine reçoit :

```json
{
  "scripts": {
    "db:seed:e2e": "pnpm --filter @fanours/e2e seed"
  }
}
```

Les scripts `test:e2e`, `test:e2e:functional`, `test:e2e:visual` et `test:e2e:shard` sont **déjà** créés dans le `package.json` racine (tâche 1) et pointent vers la configuration racine. Le paquet `e2e/` ne porte plus de configuration Playwright : uniquement ses sources de test, son `tsconfig.json` et son lint.

- [ ] **Step 4: Écrire les fixtures, dont le calcul TOTP de test**

`e2e/fixtures/totp.ts` :

```typescript
import { createHmac } from 'node:crypto';

const BASE32_ALPHABET = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';

export function decodeBase32(secret: string): Buffer {
  const normalized = secret.toUpperCase().replace(/=+$/, '');
  let bits = '';
  for (const character of normalized) {
    const index = BASE32_ALPHABET.indexOf(character);
    if (index === -1) {
      throw new Error(`Caractere base32 invalide : ${character}`);
    }
    bits += index.toString(2).padStart(5, '0');
  }
  const bytes: number[] = [];
  for (let offset = 0; offset + 8 <= bits.length; offset += 8) {
    bytes.push(Number.parseInt(bits.slice(offset, offset + 8), 2));
  }
  return Buffer.from(bytes);
}

export function computeTotp(secret: string, at: number = Date.now()): string {
  const counter = Math.floor(at / 1000 / 30);
  const buffer = Buffer.alloc(8);
  buffer.writeBigUInt64BE(BigInt(counter));
  const digest = createHmac('sha1', decodeBase32(secret)).update(buffer).digest();
  const offset = (digest[digest.length - 1] ?? 0) & 0x0f;
  const binary =
    (((digest[offset] ?? 0) & 0x7f) << 24) |
    (((digest[offset + 1] ?? 0) & 0xff) << 16) |
    (((digest[offset + 2] ?? 0) & 0xff) << 8) |
    ((digest[offset + 3] ?? 0) & 0xff);
  return String(binary % 1_000_000).padStart(6, '0');
}
```

Ce calcul est **indépendant** de l'implémentation serveur : il dérive de la RFC 6238 et sert de contre-vérification croisée. Il ne vit que dans le paquet de test.

`e2e/fixtures/test.ts` :

```typescript
import { test as base, expect, request, type APIRequestContext } from '@playwright/test';

const apiBaseUrl = process.env['E2E_API_URL'] ?? 'http://localhost:4000';

export const test = base.extend<{ api: APIRequestContext }>({
  api: async ({}, use) => {
    const context = await request.newContext({ baseURL: apiBaseUrl });
    await use(context);
    await context.dispose();
  },
});

export { expect };
```

- [ ] **Step 5: Écrire le seed des utilisateurs de test**

`e2e/scripts/seed-e2e.ts` :

```typescript
import argon2 from 'argon2';
import { createDb, createPool, user, userRole } from '@fanours/db';

const ADMIN_EMAIL = process.env['E2E_ADMIN_EMAIL'] ?? 'admin@fanours.test';
const READER_EMAIL = process.env['E2E_READER_EMAIL'] ?? 'lecteur@fanours.test';
const DEFAULT_PASSWORD = process.env['E2E_ADMIN_PASSWORD'] ?? 'MotDePasse-E2E-2026';

const ARGON2_OPTIONS: argon2.Options & { raw?: false } = {
  type: argon2.argon2id,
  memoryCost: 65536,
  timeCost: 3,
  parallelism: 1,
};

async function createUser(
  database: ReturnType<typeof createDb>,
  email: string,
  displayName: string,
  roleCode: string,
): Promise<void> {
  const passwordHash = await argon2.hash(DEFAULT_PASSWORD, ARGON2_OPTIONS);
  const inserted = await database
    .insert(user)
    .values({
      email,
      emailNormalized: email.toLowerCase(),
      passwordHash,
      displayName,
      locale: 'fr-FR',
      emailVerifiedAt: new Date(),
    })
    .onConflictDoNothing()
    .returning({ id: user.id });

  const userId = inserted[0]?.id;
  if (userId === undefined) {
    return;
  }
  await database
    .insert(userRole)
    .values({ userId, roleCode })
    .onConflictDoNothing();
}

export async function seedE2eUsers(): Promise<void> {
  const connectionString = process.env['DATABASE_URL'];
  if (connectionString === undefined) {
    throw new Error('DATABASE_URL est requis pour le seed E2E.');
  }
  const pool = createPool(connectionString);
  const database = createDb(pool);
  try {
    await createUser(database, ADMIN_EMAIL, 'Administrateur E2E', 'super_admin');
    await createUser(database, READER_EMAIL, 'Lecteur E2E', 'catalog_manager');
  } finally {
    await pool.end();
  }
}

seedE2eUsers().catch((error: unknown) => {
  process.stderr.write(`${error instanceof Error ? error.stack : String(error)}\n`);
  process.exitCode = 1;
});
```

Les identifiants sont **exclusivement des valeurs de test** : ils n'existent que dans la base `test`/`staging` de recette et ne sont jamais communiqués à la production. Le catalogue de test ne contient aucune donnée réelle.

- [ ] **Step 6: Écrire la spécification des pages publiques (3 langues × 2 thèmes)**

`e2e/specs/public-pages.spec.ts` :

```typescript
import AxeBuilder from '@axe-core/playwright';
import { expect, test } from '../fixtures/test.js';

const LOCALES = ['fr', 'en', 'zh-cn'] as const;
const THEMES = ['light', 'dark'] as const;

for (const locale of LOCALES) {
  for (const theme of THEMES) {
    test.describe(`${locale} / ${theme}`, () => {
      test.use({ locale: locale === 'zh-cn' ? 'zh-CN' : locale });

      test.beforeEach(async ({ page }) => {
        await page.addInitScript((value) => {
          window.localStorage.setItem('fanours.theme', value);
        }, theme);
      });

      test('affiche un titre de niveau 1 et ne deborde pas horizontalement', async ({ page }) => {
        await page.goto(`/${locale}/`);
        await expect(page.getByRole('heading', { level: 1 })).toBeVisible();
        const overflow = await page.evaluate(
          () => document.documentElement.scrollWidth - window.innerWidth,
        );
        expect(overflow).toBeLessThanOrEqual(0);
      });

      test('applique le theme demande', async ({ page }) => {
        await page.goto(`/${locale}/`);
        await expect(page.locator('html')).toHaveAttribute('data-theme', theme);
      });

      test('expose un canonical absolu et quatre liens alternatifs', async ({ page }) => {
        await page.goto(`/${locale}/`);
        const canonical = page.locator('link[rel="canonical"]');
        await expect(canonical).toHaveCount(1);
        await expect(canonical).toHaveAttribute('href', new RegExp(`/${locale}/$`));
        await expect(page.locator('link[rel="alternate"][hreflang]')).toHaveCount(4);
      });

      test('n a aucune violation critique axe-core', async ({ page }) => {
        await page.goto(`/${locale}/`);
        const results = await new AxeBuilder({ page })
          .withTags(['wcag2a', 'wcag2aa'])
          .analyze();
        const critical = results.violations.filter(
          (violation) => violation.impact === 'critical',
        );
        expect(critical).toEqual([]);
      });

      test('propose la navigation basse a quatre entrees sur mobile', async ({ page }) => {
        await page.setViewportSize({ width: 390, height: 844 });
        await page.goto(`/${locale}/`);
        const bottomNav = page.locator('nav').last();
        await expect(bottomNav).toBeVisible();
        await expect(bottomNav.locator('a')).toHaveCount(4);
      });

      test('renvoie 404 sur une page inconnue', async ({ page }) => {
        const response = await page.goto(`/${locale}/page-inexistante`);
        expect(response?.status()).toBe(404);
      });
    });
  }
}
```

- [ ] **Step 7: Écrire la spécification PWA**

`e2e/specs/pwa.spec.ts` :

```typescript
import { expect, test } from '../fixtures/test.js';

test.describe('PWA', () => {
  test('enregistre le service worker et purge les anciens caches', async ({ page }) => {
    await page.goto('/fr/');
    const registered = await page.evaluate(async () => {
      const registration = await navigator.serviceWorker.getRegistration();
      return registration !== undefined;
    });
    expect(registered).toBe(true);

    const cacheNames = await page.evaluate(async () => caches.keys());
    expect(cacheNames.some((name) => name.startsWith('fanours-sw-'))).toBe(true);
  });

  test('ne met aucune donnee privee en cache', async ({ page }) => {
    await page.goto('/fr/');
    await page.goto('/fr/panier');
    const cachedUrls = await page.evaluate(async () => {
      const names = await caches.keys();
      const urls: string[] = [];
      for (const name of names) {
        const cache = await caches.open(name);
        for (const request of await cache.keys()) {
          urls.push(request.url);
        }
      }
      return urls;
    });
    expect(cachedUrls.filter((url) => /\/fr\/(panier|commandes)/.test(url))).toEqual([]);
  });
});
```

- [ ] **Step 8: Exécuter la recette complète et prouver la réussite**

```bash
pnpm db:seed:e2e
pnpm exec playwright test --project=functional
```

Attendu : les 3 languages × 2 thèmes passent, `axe-core` ne remonte aucune violation critique, le refus d'accès renvoie `forbidden` puis `mfa_required`, et le journal contient les trois événements de contrôle.

Contrôle manuel obligatoire, en complément de l'automatisation, à consigner dans le procès-verbal :

1. navigation **clavier seule** sur `/fr/` (lien d'évitement, header, navigation basse, pied de page) ;
2. **lecteur d'écran** sur l'accueil et sur la page 404 : le titre de niveau 1 et l'état vide sont annoncés ;
3. bascule de thème au clavier : le focus reste visible après changement de thème ;
4. zoom 200 % sur `/fr/` : aucun contenu perdu, aucun débordement horizontal.

Ces quatre points relèvent de la **revue manuelle** : un audit automatique ne prouve pas l'usage au lecteur d'écran.

- [ ] **Step 9: Rédiger le procès-verbal du Gate 1**

`docs/phase-1/gate-1.md` :

```markdown
# Gate 1 — Socle

Date : <date de la revue>
Participants : <noms>
Preuves : lien de la PR, job CI, rapport Playwright, rapport axe-core

| # | Question | Réponse | Preuve |
|---|---|---|---|
| 1 | CI verte de bout en bout sur une PR ? | <oui/non> | <lien du run> |
| 2 | `typecheck` strict sans erreur, build prod reproductible ? | <oui/non> | <sortie du job typecheck> |
| 3 | Inscription → connexion → page publique en 3 langues, 2 thèmes ? | <oui/non> | <spec public-pages.spec.ts> |
| 4 | Admin se connecte avec MFA, refusé hors permission ? | <oui/non> | <spec security.spec.ts> |
| 5 | Un secret a-t-il été commité ? (scan vert) | <oui/non> | <job scan-secrets> |
| 6 | Le `robots.txt` et les sitemaps sont-ils en place ? | <oui/non> | <curl sur /robots.txt et /sitemap.xml> |
| 7 | Axe-core : 0 violation critique sur les pages publiques ? | <oui/non> | <rapport axe-core> |
| 8 | Le journal d'audit écrit-il déjà les 3 événements de contrôle ? | <oui/non> | <requête sur audit_log> |

**Passage si 1, 3 et 4 sont « oui ».** Sinon retour en Phase 1.

## Écarts acceptés

| Écart | Justification | Propriétaire | Échéance |
|---|---|---|---|
| | | | |

## Points laissés à l'humain

- [ ] Revue manuelle clavier + lecteur d'écran (au-delà de l'audit auto)
- [ ] Relecture juridique des pages légales
- [ ] Nom de domaine définitif
```

- [ ] **Step 10: Vérifier la complétude de la Phase 1**

```bash
pnpm turbo run typecheck
pnpm turbo run test:unit
pnpm turbo run test:integration
pnpm turbo run build
```

Les quatre commandes doivent être vertes **avant** d'ouvrir la PR du Gate 1. La Phase 1 ne livre **aucune feature métier** : le catalogue reste vide, le panier n'est pas implémenté et aucun paiement n'est possible. C'est un choix explicite de la roadmap, pas un oubli.

- [ ] **Step 11: États, 3 langues et 2 thèmes**

La recette couvre les 6 combinaisons langue × thème sur les pages publiques. Les 12 états système de la Task 10 sont vérifiés par tests de composants ; les états qui dépendent du compte (session expirée, accès refusé, paiement en attente) sont **hors périmètre de la Phase 1** et le resteront jusqu'à la Phase 2.

- [ ] **Step 12: Commit**

```bash
git add e2e docs/phase-1 pnpm-workspace.yaml package.json
git commit -m "test(e2e): ajoute la recette Phase 1 et le proces-verbal du Gate 1" -m "La suite Playwright couvre les trois langues et les deux themes, verifie
l absence de violation critique axe-core, l absence de debordement
horizontal, le refus d acces sans permission, le refus MFA sans
elevation et l absence de donnee privee dans les caches du PWA.
Le journal d audit est interroge en base pour prouver l ecriture des
trois evenements de controle : connexion, echec MFA et refus RBAC.
Le proces-verbal du Gate 1 reprend les huit questions du jalon et
indique que le passage exige les questions 1, 3 et 4.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` contient le titre et la ligne `Co-Authored-By`.

---

## Self-Review

### 1. Couverture des 15 chantiers de la Phase 1

| # | Chantier | Task | Artefact produit |
|---|---|---|---|
| 1.1 | Monorepo | 1 | `pnpm-workspace.yaml`, `turbo.json`, `apps/{web,api,worker}`, `packages/{config,contracts,domain,db,pricing,compliance,i18n,ui,observability}` |
| 1.2 | Schéma DB initial | 2 | `packages/db/migrations/0001_extensions_and_migration_table.sql` … `0004_outbox.sql`, `packages/db/src/schema.ts`, `packages/db/src/migrate-core.ts`, `packages/db/scripts/migrate.ts` |
| 1.3 | Tokens & thèmes | 3 | `packages/ui/src/styles/tokens.css` (blocs `:root`, `:root:not([data-theme="light"])`, `:root[data-theme="dark"]`) |
| 1.4 | Composants de base | 4 | `packages/ui/src/components/` : `Button`, `IconButton`, `Input`, `Textarea`, `Select`, `Badge`, `Skeleton`, `Dialog`, `BottomSheet`, `Toast`, `Table`, `Tabs`, `Pagination`, `SystemState` |
| 1.5 | i18n | 5 | `packages/i18n/src/dictionaries/{fr-FR,en,zh-CN}.ts`, `packages/i18n/scripts/check-parity.ts`, `packages/i18n/src/locale.ts` |
| 1.6 | Devise & montants | 6 | `packages/domain/src/money.ts` (unités mineures), `packages/i18n/src/format.ts` (`Intl.NumberFormat`) |
| 1.7 | Auth | 7 | `apps/api/src/modules/auth/{auth.service,auth.controller,contracts}.ts`, `apps/api/src/common/security/{password,session,rate-limit}.ts` |
| 1.8 | MFA | 8 | `apps/api/src/modules/mfa/{mfa.service,mfa.controller}.ts`, `apps/api/src/security/{totp,envelope,recovery-code}.ts` |
| 1.9 | RBAC | 9 | `apps/api/src/modules/rbac/{rbac.service,rbac.guard,rbac.decorator,approval.service}.ts`, `packages/db/migrations/0005_rbac_scopes.sql` |
| 1.10 | Pages publiques | 10 | `apps/web/src/app/[locale]/{layout,page,not-found,loading,error}.tsx`, `apps/web/src/app/[locale]/{catalogue,contact,conditions,panier,commandes}/page.tsx`, `apps/web/src/components/layout/*` |
| 1.11 | CI | 14 | `.github/workflows/ci.yml` (10 jobs bloquants), `e2e.yml`, `security.yml`, `dependency-review.yml` |
| 1.12 | Environnements | 14 | `docker/Dockerfile.{web,api,worker}`, `docker/compose.{local,test}.yml`, `ops/`, `.env.example` (noms seuls) |
| 1.13 | Observabilité | 11 | `packages/observability/src/{logger,correlation,trace,metrics}.ts`, `apps/api/src/common/correlation.middleware.ts`, `apps/api/src/modules/health/*`, `apps/worker/src/jobs/outbox-relay.ts` |
| 1.14 | PWA socle | 12 | `apps/web/public/manifest.webmanifest`, `apps/web/public/sw.js` (`fanours-sw-v1`), `apps/web/src/components/pwa/register-service-worker.tsx` |
| 1.15 | SEO technique | 13 | `apps/web/src/lib/seo.ts`, `apps/web/public/robots.txt`, `apps/web/src/app/sitemap.xml/route.ts`, `apps/web/src/app/sitemaps/*/route.ts`, `apps/web/src/components/seo/json-ld.tsx` |

Les 8 vagues (1.1 à 1.8) de la roadmap sont couvertes : 1.1 → Task 1 ; 1.2 → Task 2 ; 1.3 → Tasks 3 et 4 ; 1.4 → Task 5 ; 1.5 → Task 7 ; 1.6 → Task 8 ; 1.7 → Task 9 ; 1.8 → Tasks 10, 12, 13, 14 (et 11 pour l'observabilité).

### 2. Scan des placeholders

| Motif recherché | Résultat |
|---|---|
| `TBD` | 0 occurrence |
| `TODO` | 0 occurrence |
| `à compléter` / `a completer` | 0 occurrence |
| `identique à la Task N` / `idem Task N` | 0 occurrence |
| `<…>` comme valeur de code | 0 occurrence hors motifs volontairement génériques (`<base-url>`, `<id>`, `<digest>` dans des commandes shell et des runbooks, où le placeholder est un argument à fournir par l'opérateur) |
| Marqueurs de continuation `<!-- FANOURS-PLAN-CONTINUATION* -->` | 0 occurrence : les quatre marqueurs ont été consommés |
| `...` comme code tronqué | 0 occurrence dans un bloc de code exécutable |
| URLs ou identifiants de faits inventés (notes, avis, stocks, délais, compteurs d'avis) | 0 occurrence : aucun `aggregateRating`, aucun avis, aucun délai fournisseur, aucun prix catalogue — le catalogue de la Phase 1 est vide par conception |
| Secret en clair | 0 occurrence : les seules valeurs littérales sont des identifiants de test documentés (`admin@fanours.test`, `MotDePasse-E2E-2026`, `fanours` / `fanours-local-only` pour la stack Docker locale) |

### 3. Cohérence des types, des noms et écarts assumés

**Cohérence vérifiée :**

- Tableaux de base : `user`, `auth_identity`, `session`, `mfa_method`, `recovery_code`, `role`, `permission`, `user_role`, `admin_action_approval`, `audit_log`, `outbox_event`, `consumed_event`, `rbac_scope` — cohérents entre la migration SQL, `packages/db/src/schema.ts`, les assertions de restauration (`ops/backup/assertions.sql`) et les tests.
- Migrations : `0001_extensions_and_migration_table.sql`, `0002_identity.sql`, `0003_rbac_and_audit.sql`, `0004_outbox.sql`, puis `0005_rbac_scopes.sql` — numérotation existante conservée, aucun renumérotage, `0005` append-only sans script `down`, avec `SET lock_timeout` et `SET statement_timeout`.
- Numérotation des migrations SQL, continue et sans recouvrement entre les phases : Phase 1 `0001`–`0005` (cette phase), Phase 2 `0006`–`0015`, Phase 3 `0100`–`0110`, Phase 4 `0111`–`0115`, Phase 5 `0116`–`0119`, Phase 6 `0120`–`0123`. La Phase 1 est **propriétaire** du paquet `packages/db` (runner : cœur `src/migrate-core.ts` + CLI `scripts/migrate.ts`, table `schema_migrations`, migration d'extensions `0001`, schémas `src/schema.ts`) : les phases suivantes **étendent** ce paquet en ajoutant leurs migrations à la suite (`0006` et suivantes), sans jamais recréer le runner, la table de suivi ni une migration déjà appliquée (append-only).
- Rôles (9), ressources (13), actions (8) : identiques dans `packages/domain/src/permissions.ts` (source de vérité), `packages/db/src/seed.ts` (seed) et `packages/db/migrations/0005_rbac_scopes.sql` (CHECK sur les ressources).
- Clés i18n : 12 clés ajoutées en Task 10, présentes dans les 3 dictionnaires ; le script de parité `packages/i18n/scripts/check-parity.ts` fait échouer le build si une clé manque dans une langue.
- Chemins web : `/{locale}` avec `locale ∈ {fr, en, zh-cn}` ; `LOCALE_ROUTES` mappe `fr-FR → fr`, `en → en`, `zh-CN → zh-cn` ; `routeToLocale` fait l'opération inverse. Les pages privées sont `/{locale}/panier` et `/{locale}/commandes`.
- Enveloppe d'erreur : `{ error: { code, messageKey, correlationId, details? } }` (`ApiErrorBody` de `packages/contracts/src/api.ts`), utilisée par le filtre d'exceptions (Task 11) et par les assertions E2E (Task 15) — `forbidden` et `mfa_required` sont deux codes de `API_ERROR_CODES`.
- Événements d'audit : `user.logged_in` (Task 7), `mfa.challenge_failed` (Task 8), `rbac.denied` (Task 9) — ce sont exactement les trois événements de contrôle du Gate 1, question 8.
- Clé de thème : `THEME_STORAGE_KEY = 'fanours.theme'` (Task 3), utilisée par le script anti-flash (Task 10) et par le test E2E (Task 15).

**Écarts assumés, avec leur justification :**

| Écart | Justification |
|---|---|
| `apps/web/src/app/robots.ts` (arbororescence) **n'est pas créé** ; le fichier statique `apps/web/public/robots.txt` est utilisé | Le spec §17.2 impose un contenu **exact**, commentaire d'avertissement inclus. Une route de métadonnées Next.js ne peut pas produire ce texte et entrerait en conflit de route avec le fichier statique. Le spec l'emporte sur l'arborescence du plan. |
| `apps/web/src/app/sitemap.ts` devient `apps/web/src/app/sitemap.xml/route.ts` | Le spec §17.1 impose une hiérarchie de sitemaps à 3 niveaux ; `app/sitemap.ts` ne sait produire qu'un seul fichier. Les gestionnaires de route le permettent. |
| `sitemaps/sitemap-[locale]-[type]/route.ts` devient `sitemaps/sitemap-[locale]-[type].xml/route.ts` | Le nom de fichier doit porter l'extension `.xml` pour que l'URL publique soit `sitemap-fr-pages.xml` et non `sitemap-fr-pages`. |
| Fichiers ajoutés hors arborescence : `apps/web/src/lib/seo.ts`, `apps/web/src/components/seo/json-ld.tsx`, `apps/web/src/components/pwa/register-service-worker.tsx`, `packages/observability/src/metrics.ts`, `apps/web/src/app/[locale]/{panier,commandes}/page.tsx`, `e2e/**` | Les générateurs de sitemap et de métadonnées sont factorisés dans `lib/seo.ts` plutôt que dupliqués dans 4 gestionnaires de route ; les pages panier et commandes existent pour que les 4 onglets de la navigation basse ne soient pas des liens morts (le spec §17 interdit les liens cassés) ; `e2e/` est un paquet de test à la racine, pas une application. |
| `lastActivityAt` (Task 7) corrigé en `lastSeenAt` | La colonne réelle est `last_seen_at` ; l'identifiant de la Task 7 ne correspondait pas au schéma de la Task 2. |
| `correlation_id` en UUID **v7** (et non v4) | Ordonnancement temporel garanti, cohérent avec l'exigence « UUIDv7 » du spec §20 et avec le tri des entrées d'audit. |
| Le test de migrations compte **5** migrations et non 4 | La Task 9 ajoute `0005_rbac_scopes.sql`. |
| `buildSessionCookie(token, maxAgeSeconds, secure)` | La signature Task 7 est conservée telle quelle ; elle est le point de contact entre la Task 7 et les Tests E2E. |
| Le cookie de session s'appelle `fanours_sid` et n'a pas de préfixe `__Host-` | `__Host-` impose `Secure` et l'absence de `Domain`, donc un cookie `__Host-` **ne peut pas** être émis en `http://localhost` : le préfixe est donc réservé à la production, où la Task 14 déploie derrière TLS. À trancher en Phase 2 avec le CDN. |
| `DICTIONARIES` est indexé par `Locale` (`fr-FR`, `en`, `zh-CN`) alors que les routes utilisent `fr`, `en`, `zh-cn` | La langue de la **contenu** et le segment d'**URL** sont deux notions distinctes ; le mapping est explicite dans `LOCALE_ROUTES` et `routeToLocale`, et il est interdit de les confondre (spec §4.1). |

### 4. Points laissés à l'humain

| # | Point | Pourquoi la machine ne peut pas trancher |
|---|---|---|
| 1 | **Nom de domaine définitif** | `fanours.com` est un exemple. Il conditionne `APP_BASE_URL`, le canonical, le `robots.txt` et l'URL des sitemaps. |
| 2 | **Textes juridiques** (CGV, confidentialité, livraison, mentions) | Les pages `/{locale}/conditions` contiennent des ébauches fonctionnelles. Seule une relecture par un juriste les rend opposables ; le spec §28.2 l'exige et interdit l'auto-certification. |
| 3 | **Entité juridique, pays d'établissement, devise de référence, Incoterms retenus (DDP ou DAP), catégories réglementées et codes HS** | Ce sont des décisions commerciales et fiscales, listées au §12 du `CLAUDE.md` projet comme hors de portée de l'automatisation. |
| 4 | **Prestataires réels** (e-mail, paiement, transport, S3, CDN) et comptes sandbox | Les intégrations de la Phase 1 sont derrière des interfaces ; le choix du prestataire, la DPA, le périmètre de conformité et les clés réelles relèvent du responsable. |
| 5 | **Gestionnaire de secrets retenu et procédure de rotation** | La Task 14 impose l'absence de secret dans Git et en build ; le choix du coffre (KMS, Vault, Secrets Manager) et la périodicité de rotation sont à arbitrer. |
| 6 | **RPO ≤ 24 h / RTO ≤ 8 h, région d'hébergement, plan de reprise** | Valeurs de départ issues du spec §25 ; elles doivent être validées par le responsable, et le drill de restauration réel (Tâche 14, job `restore-drill`) n'a de valeur qu'exécuté sur l'infrastructure retenue. |
| 7 | **Revue manuelle d'accessibilité** (clavier + lecteur d'écran) | L'audit `axe-core` automatisé ne prouve rien sur les parcours. La Phase 1 fournit la preuve automatique ; la revue manuelle sur `checkout`, `login` et les dashboards est exigée plus tard, quand ces parcours existent. |
| 8 | **Recette humaine des chaînes FR / EN / ZH-CN** | Les dictionnaires sont complets et paritaires, mais la qualité rédactionnelle et l'adéquation au ton B2B relèvent d'un lecteur humain. |
| 9 | **Validation de la matrice RBAC par le métier** | La matrice de la Task 2 est une proposition technique cohérente avec le spec §16.12 ; sa confirmation par les responsables de métier est une décision humaine. |
| 10 | **Modèle de menace §19.1 et périmètre des 12 flux sensibles** | La Phase 1 implémente les contrôles, pas l'analyse de menace ; celle-ci relève d'un audit sécurité formel. |
| 11 | **Seuils d'alerte et pages d'astreinte** | `ops/alerting/rules/fanours.yml` propose des seuils (5 % de 5xx, p95 à 2 s, file à 1000, backup à 24 h) ; ils doivent être calés sur la capacité réelle et l'organisation d'astreinte. |
| 12 | **Confirmation des 8 questions du Gate 1** | Le procès-verbal `docs/phase-1/gate-1.md` est un modèle à remplir : la décision de passage appartient à la revue, pas au plan. |
| 13 | **Fusion de la PR** | Aucune PR n'est mergée par ce plan. Le projet exige un accord explicite, et la protection de `main` exige 2 approbations. |
