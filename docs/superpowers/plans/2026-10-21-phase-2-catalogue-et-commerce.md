# Phase 2 — Catalogue et commerce : Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Livrer un parcours de vente complet et vérifié de bout en bout : catalogue → recherche → fiche produit → panier → checkout → commande → paiement sandbox → confirmation → facture PDF, en 3 langues, où le serveur recalcule toujours paliers, MOQ, incréments, taxes, frais d'import et disponibilité, où le stock est décrémenté atomiquement, où aucun statut `paid` ne provient d'une redirection navigateur, et où chaque étape bloquante possède un test qui échoue avant d'exister.

**Périmètre :** 19 tâches (Task 1 → Task 19) — catalogue, paliers et MOQ, stock, recherche, conformité, i18n, fiche produit, panier, checkout, commandes, paiement, webhooks, e-mails, factures, notifications, puis la recette de phase (suites de release `b2c` et `payment` + matrice visuelle).

**Architecture:** Monolithe modulaire NestJS 11 (`apps/api`) + application Next.js 15 en App Router / RSC (`apps/web`) + worker BullMQ (`apps/worker`) dans un monorepo pnpm + Turborepo. Un seul ORM (Drizzle) sur PostgreSQL 16, Redis 7 pour le cache et les files, stockage S3-compatible pour les médias et les documents. Le front n'est jamais source de vérité : `packages/pricing` recalcule chaque ligne de commande, chaque taxe et chaque frais à partir de la base, puis fige un snapshot immuable (`order.snapshot`) que la commande ne relit jamais. Les événements métier transitent par une outbox transactionnelle (`outbox_event`) et les statuts de commande n'évoluent que par une machine à états unique à 20 états et 34 transitions, versionnée dans `packages/domain` et dérivée vers un `CHECK` SQL.

**Tech Stack:** Next.js 15 (App Router, RSC) · React 19 · TypeScript `strict` · Tailwind v4 · NestJS 11 + Zod · PostgreSQL 16 + Drizzle ORM + extension `btree_gist` · Redis 7 + BullMQ · S3-compatible · `@react-pdf/renderer` (PDF, polices CJK embarquées) · Vitest + Testing Library + Playwright + Testcontainers · pnpm workspaces + Turborepo.

**Spec:** `Cahier_des_charges_Fanours.md` §2.2–2.3 (modes de vente, MOQ, incréments), §4.3–4.4 (devises, montant de référence vs transaction), §6.3–6.6 (catalogue, fiche produit, parcours d'achat, états obligatoires), §7.1–7.3 (catalogue, paliers de prix, stock et réservations), §8.4 (règles tarifaires client), §9.1–9.5 (panier, checkout, commandes, protection double commande), §10.1–10.4 (paiement, webhooks, remboursements, réconciliation), §11.1–11.10 (douane, taxes, Incoterms, documents), §12.1 (transport), §14.1–14.5 (notifications, e-mails, factures), §17 (SEO), §18.1 (recherche), §20 (modèle de données), §21.3 (événements), §24 (tests), §26.1–26.6 (critères d'acceptation), §27 (phasage). Plan phase : `docs/roadmap/sections/roadmap-phases.md` § Phase 2.

## Global Constraints

**Stack figée** : Next.js 15 App Router + React 19 + TypeScript `strict` + Tailwind v4 ; NestJS 11 + Zod ; PostgreSQL 16 + Drizzle (un seul ORM, jamais Prisma) ; Redis 7 + BullMQ ; S3-compatible ; Vitest + Testing Library + Playwright + Testcontainers.

**Monorepo** : pnpm workspaces + Turborepo, `apps/{web,api,worker}` et `packages/{config,contracts,db,domain,pricing,compliance,payment,jobs,i18n,ui,observability,pdf}`. Aucun paquet ne dépend de `apps/*`.

**Zéro placeholder** : aucun `TBD`, `TODO`, `à compléter`, ni « identique à la tâche N ». Chaque étape contient du code ou un contenu réel, exécutable tel quel.

**Cycle de test par tâche** : écrire le test qui échoue → l'exécuter et constater l'échec → implémenter le strict minimum → l'exécuter et constater la réussite → committer. Aucun commit sans test qui passe.

**Le front n'est jamais source de vérité** : prix, paliers, MOQ, incréments, taxes, frais de transport, frais d'import, disponibilité et autorisation sont recalculés côté serveur à chaque action ; un test le prouve en injectant des prix falsifiés côté client.

**Jamais de float binaire pour un montant** : `NUMERIC(18,4)` en base, entiers de minor units en code, arithmétique entière (`BigInt` pour les pourcentages en points de base), codes ISO 4217 explicites, formatage par `Intl`.

**Le statut `paid` ne vient JAMAIS d'une redirection navigateur** : la seule source de `order.status = 'paid'` est le webhook signé du prestataire, traité par le job `webhook.payment` et idempotent.

**Idempotence obligatoire** : `UNIQUE` en base sur `order.idempotency_key`, `payment.idempotency_key`, `refund.idempotency_key`, `webhook_event(provider, provider_event_id)`, `email_delivery.idempotency_key`, `stock_reservation.idempotency_key`, `outbox_event.idempotency_key`.

**Anti-survente** : `SELECT … FOR UPDATE` sur `inventory_item` + `CHECK (reserved + qty <= on_hand)` dans une seule transaction ; test obligatoire « 2 clients, dernière unité → une seule commande ».

**Paliers non chevauchants** : `EXCLUDE USING gist` sur `int4range(min_qty, COALESCE(max_qty, 2147483647),'[]')` (extension `btree_gist`) ; la base refuse le chevauchement, pas le code applicatif.

**Les règles douanières sont des données** : aucun taux, seuil, tarif ou barème en code ; l'algorithme résout `trade_rule` + `trade_rule_version` ; une version expirée déclenche une alerte et n'est jamais appliquée silencieusement ; un contrôle CI refuse toute constante de taux ou de seuil hors `packages/compliance/src/constants.allowlist.ts`.

**Interdits absolus** : sous-valorisation, fausse catégorie, fausse origine, fractionnement de colis, fausse promesse DDP. Aucun secret dans le code, les logs, le bundle, les migrations ou ce plan. Aucune donnée de carte stockée ni journalisée.

**Une seule page produit indexable** : le tableau de paliers est un vrai `<table>` avec `<caption>` et `th scope` dans le HTML initial (pas d'onglet JS, pas de `details/summary`) ; `Offer.price` = prix du palier MOQ ; toute divergence entre le premier palier, `Offer.price`, la carte de grille et les blocs de prix du titre **bloque la publication**.

**PDF** : `@react-pdf/renderer`, polices CJK embarquées depuis `assets/fonts/` (jamais téléchargées à l'exécution), PDF en bucket privé + URL signée 5 minutes, jamais dans le bundle public ni sur le CDN.

**i18n** : aucune chaîne en dur dans l'interface ; dictionnaires `fr-FR` / `en` / `zh-CN` avec repli explicite `fr-FR` ; le build échoue si une clé manque dans l'une des trois langues.

**Accessibilité** : WCAG 2.2 AA — focus visible 2 px + offset 2 px, cibles tactiles ≥ 44 px, `prefers-reduced-motion` respecté, aucune information portée par la couleur seule, zéro débordement horizontal.

**Performance** : JS premier chargement ≤ 170 Ko (catalogue) et ≤ 220 Ko (checkout), CSS ≤ 60 Ko, LCP ≤ 2,5 s, INP ≤ 200 ms, CLS ≤ 0,1 au p75.

**Migrations** : SQL écrit à la main, append-only, `packages/db/migrations/NNNN_nom.sql`, expand/contract, aucun script `down`, checksum SHA-256 stocké dans `schema_migrations`, CI refuse le SQL destructif non justifié par un commentaire `-- migrate:contract`.

**Git** : branche `feat|fix|refactor|test|docs|chore/<slug>`, jamais `main`, jamais de force-push, PR laissée ouverte et jamais mergée sans accord explicite, PR > 800 lignes à scinder.

**Commits** : Conventional Commits 1.0, scopes alignés sur les modules métier (`catalog`, `pricing`, `cart`, `checkout`, `order`, `payment`, `tax`, `customs`, `shipping`, `auth`, `account`, `b2b`, `quote`, `admin`, `content`, `i18n`, `infra`).

**Preuve** : pas de « terminé » sans la sortie de test collée dans la PR ; les tests non exécutables sont annoncés comme tels avec leurs prérequis.

**Modèle E2E canonique** : configuration Playwright **unique**, à la racine du dépôt (`playwright.config.ts`) — créée en Phase 1, **modifiée** (jamais recréée) par les Phases 2, 4, 5 et 6. Aucun autre `playwright.config.ts` (ni `apps/web/`, ni `apps/api/`, ni `e2e/`) ; les commandes Playwright sont lancées **depuis la racine** (`pnpm exec playwright test ...`). **7 projets** : `b2c`, `b2b`, `payment`, `admin`, `security` (les 5 suites de recette de release du spec §24.3), `visual` (matrice visuelle) et `functional` (recettes fonctionnelles de développement). `testDir` vaut `'./e2e'` pour **tous** les projets : ils se distinguent uniquement par `testMatch` / `testIgnore`, donc aucun projet ne référence un dossier qui n'existe pas encore. Les **viewports, les langues et les thèmes ne sont pas des projets** : ils sont paramétrés dans les fichiers de test via `test.use({ viewport })` à partir de la liste canonique `e2e/support/viewports.ts` (créée en Phase 1, **importée** et **jamais re-déclarée**) : 320×640 · 360×740 · 390×844 · 430×932 · 768×1024 · 1024×768 · 1440×900 · 1920×1080. Règle d'appartenance : **un fichier de test appartient à un seul projet** — aucun doublon, aucun comptage de tests ambigu.

---

## File Structure

```code
fanours/
├─ apps/
│  ├─ api/                                       # API HTTP NestJS : seul calculateur de prix, taxes, stock et autorisation
│  │  ├─ src/main.ts                             # bootstrap NestJS, helmet, CORS allowlist, /healthz, /metrics
│  │  ├─ src/app.module.ts                       # racine des modules métier, import du module outbox
│  │  ├─ src/common/                             # socle transverse : erreurs, corrélation, validation
│  │  │  ├─ errors/api-error.ts                  # classe ApiError : code stable + message i18n + correlation_id
│  │  │  ├─ filters/http-exception.filter.ts     # traduction des exceptions en réponses JSON normalisées
│  │  │  ├─ interceptors/correlation.interceptor.ts # propagation traceparent W3C + correlation_id
│  │  │  └─ pipes/zod-validation.pipe.ts         # zod .strict() en entrée ET en sortie
│  │  ├─ src/db/                                 # accès base : pool Drizzle, module Nest, helper transactionnel
│  │  │  ├─ database.module.ts                   # injection du pool pg et du service transactionnel
│  │  │  ├─ drizzle.client.ts                    # factory du client Drizzle (pool, SSL, timeouts, time zone UTC)
│  │  │  └─ tx.service.ts                        # withTransaction() : une transaction = une unité d'écriture
│  │  └─ src/modules/                            # modules métier du monolithe modulaire
│  │     ├─ catalog/                             # 2.1 : produits, variantes, traductions, médias, catégories
│  │     ├─ pricing/                             # 2.2/2.3 : paliers, règles client, MOQ, incréments
│  │     ├─ stock/                               # 2.4 : entrepôts, réservations, mouvements, anti-survente
│  │     ├─ search/                              # 2.5 : tsvector, facettes, pagination keyset
│  │     ├─ cart/                                # 2.7 : panier serveur, fusion, revalidation MOQ en temps réel
│  │     ├─ checkout/                            # 2.8 : adresses, transport, recalcul serveur, snapshot
│  │     ├─ order/                               # 2.9 : commandes, machine à états, documents, pagination clé
│  │     ├─ payment/                             # 2.10 : PaymentProvider, sandbox, éléments hébergés, 3DS
│  │     ├─ webhook/                             # 2.11 : réception, signature HMAC, anti-rejeu, idempotence
│  │     ├─ notification/                        # 2.12/2.14 : e-mails transactionnels + centre de notifications
│  │     └─ outbox/                              # append transactionnel, relay, consommateur idempotent
│  ├─ web/                                       # application Next.js 15 : vitrine indexable + tunnel de commande
│  │  ├─ src/app/[locale]/                       # pages publiques FR / EN / ZH-CN (catalogue, produit, catégorie)
│  │  ├─ src/app/(checkout)/                     # tunnel de commande, noindex, jamais mis en cache
│  │  ├─ src/app/(account)/                      # compte, commandes, notifications, documents, noindex
│  │  ├─ src/app/api/                            # BFF mince : délégation API, no-store sur tout le privé
│  │  ├─ src/app/offline/                        # page hors-ligne PWA
│  │  ├─ src/app/sitemap.ts                      # sitemaps par langue, exclusion des pages privées
│  │  ├─ src/components/                         # design system : ProductCard, PriceTierTable, Stepper, SystemState
│  │  ├─ src/lib/                                # fetchers serveur (cache 60 s), formatage Intl, JSON-LD
│  │  └─ public/sw.js                            # service worker versionné : precache, NetworkOnly sur le privé
│  └─ worker/                                    # worker BullMQ : un binaire, une queue par famille de jobs
│     ├─ src/main.ts                             # démarrage des workers BullMQ + scheduler cron interne
│     ├─ src/jobs/                               # defineJob par type de job (15 types), idempotence, backoff
│     └─ src/handlers/                           # handlers métier : email, webhook, invoice, pdf, stock, sitemap
├─ packages/
│  ├─ config/                                    # matrice d'env, secrets résolus, feature flags
│  ├─ contracts/                                 # schémas zod partagés API/UI (validation entrée ET sortie)
│  ├─ db/                                        # Drizzle : schémas, migrations SQL append-only, runner
│  │  ├─ migrations/                             # SQL append-only : une version par fichier, jamais modifiée
│  │  │  ├─ 0006_catalog.sql                     # catégories, produits, variantes, traductions, médias
│  │  │  ├─ 0007_pricing.sql                    # paliers de prix (EXCLUDE gist) et règles client
│  │  │  ├─ 0008_stock.sql                      # entrepôts, stock, réservations, mouvements
│  │  │  ├─ 0009_cart.sql                       # paniers et lignes de panier
│  │  │  ├─ 0010_order.sql                      # commandes, lignes de commande, événements de statut
│  │  │  ├─ 0011_payment.sql                    # paiements, événements, remboursements, factures
│  │  │  ├─ 0012_notification_and_outbox.sql    # notifications, e-mails, extension outbox et audit
│  │  │  ├─ 0013_search.sql                     # tsvector et index GIN de recherche
│  │  │  ├─ 0014_trade_rules.sql                # règles douanières, versions, alertes compliance
│  │  │  └─ 0015_invoice_sequence.sql           # séquence et table de séquence des factures
│  │  ├─ src/schema.ts                          # schémas Phase 1 (identité, RBAC, audit, outbox) étendus
│  │  ├─ src/schema/                            # schémas métier Phase 2 (catalog, pricing, stock, cart, order, payment, notification)
│  │  ├─ src/client.ts                          # pool pg + drizzle (schéma combiné Phase 1 + Phase 2)
│  │  ├─ src/migrate.ts                         # adaptateur du runner de la Phase 1 (packages/db/src/migrate-core.ts)
│  │  └─ src/transaction.ts                     # withTransaction() sur pool pg
│  ├─ domain/                                    # logique métier pure : money, MOQ, paliers, machine à états
│  ├─ pricing/                                   # calcul de commande : lignes, frais, snapshot immuable
│  ├─ compliance/                                # moteur de règles douanières : taxes, droits, seuils, certitude
│  ├─ payment/                                   # interface PaymentProvider + implémentation sandbox
│  ├─ jobs/                                      # defineJob, idempotencyKey, backoffMs, dead-letter
│  ├─ i18n/                                      # dictionnaires 3 langues + vérificateur de parité des clés
│  ├─ ui/                                        # tokens CSS + composants React du design system
│  ├─ observability/                             # pino, redaction PII/secret, métriques Prometheus
│  └─ pdf/                                       # rendu PDF @react-pdf/renderer + polices embarquées
├─ playwright.config.ts                           # config Playwright racine (7 projets) — MODIFIÉE en Phase 2
├─ e2e/                                          # Playwright : suites de release b2c/payment + matrice visuelle
│  ├─ support/
│  │  └─ viewports.ts                            # importé depuis la Phase 1 — source unique des 8 viewports
│  ├─ fixtures/
│  │  └─ product.ts                              # produit de référence et helpers de parcours
│  ├─ b2c/
│  │  └─ journey.spec.ts                         # suite de release b2c (9 étapes)
│  ├─ payment/
│  │  └─ scenarios.spec.ts                       # suite de release payment (8 cas)
│  └─ visual-matrix.spec.ts                      # matrice visuelle 8 viewports x 3 langues x 2 thèmes
├─ ops/                                          # runbooks, polices PDF versionnées, configuration CI
```

L'arborescence ci-dessus est la cible de fin de phase : les fichiers listés sont créés par les tasks 1 à 17, qui les introduisent dans l'ordre d'exécution. Aucun fichier n'est modifié hors de cette liste sans mise à jour préalable de ce plan.

Le paquet `packages/db` et son runner sont **créés par la Phase 1** (Task 2 de la Phase 1 : `package.json`, `tsconfig.json`, `drizzle.config.ts`, `vitest.config.ts`, `src/client.ts`, `src/index.ts`, `src/seed.ts`, `src/schema.ts`, `src/migrate-core.ts`, `scripts/migrate.ts` et les migrations `0001_extensions_and_migration_table.sql` à `0005_rbac_scopes.sql`). La Phase 2 n'en recrée aucun : elle **étend** le paquet en ajoutant les modules de schéma `src/schema/*.ts` et les migrations `0006` à `0015`. Les migrations `0001` à `0005` ne sont donc pas listées dans l'arborescence ci-dessus : elles existent déjà, ne sont jamais modifiées (append-only) et ne sont jamais recopiées.

De même, la configuration Playwright `playwright.config.ts` (racine) et la liste canonique `e2e/support/viewports.ts` sont **créées par la Phase 1**. La Phase 2 ne recrée ni l'une ni l'autre : elle **modifie** la configuration racine (aucun changement de structure de projets — les 7 projets et leurs `testMatch` / `testIgnore` sont déjà en place) et **importe** `e2e/support/viewports.ts` dans la matrice visuelle sans jamais re-déclarer la liste des viewports.

### Task 1: Scaffolding du monorepo et des packages partagés (config, contracts, domain)

**Files:**
- Create: `pnpm-workspace.yaml`, `turbo.json`, `package.json`, `tsconfig.base.json`, `.npmrc`, `.gitignore`, `.env.example`
- Create: `packages/config/package.json`, `packages/config/src/env-matrix.ts`, `packages/config/src/feature-flags.ts`, `packages/config/src/index.ts`
- Create: `packages/domain/package.json`, `packages/domain/src/currency.ts`, `packages/domain/src/money.ts`, `packages/domain/src/purchase-rules.ts`, `packages/domain/src/order-state-machine.ts`, `packages/domain/src/index.ts`
- Create: `packages/contracts/package.json`, `packages/contracts/src/catalog.ts`, `packages/contracts/src/cart.ts`, `packages/contracts/src/checkout.ts`, `packages/contracts/src/order.ts`, `packages/contracts/src/index.ts`
- Create: `packages/config/src/env-matrix.test.ts`, `packages/domain/src/money.test.ts`, `packages/domain/src/purchase-rules.test.ts`, `packages/domain/src/order-state-machine.test.ts`, `packages/contracts/src/catalog.test.ts`
- Create: `packages/config/vitest.config.ts`, `packages/domain/vitest.config.ts`, `packages/contracts/vitest.config.ts`

**Interfaces:**
- Consumes: rien (premier package du dépôt).
- Produces:
  - `packages/config` → `envMatrix(env: NodeJS.ProcessEnv): EnvMatrix`, `evaluateFlag(flag: FlagKey, ctx: FlagContext): boolean`, `FLAG_KEYS: readonly FlagKey[]`
  - `packages/domain` → `toMinorUnits(value: string, currency: CurrencyCode): bigint`, `fromMinorUnits(minor: bigint, currency: CurrencyCode): string`, `multiplyMinor(minor: bigint, qty: number, bps: number): bigint`, `currencyExponent(code: CurrencyCode): number`, `resolvePurchaseRules(input: PurchaseRulesInput): PurchaseRulesResult`, `allowedTransitions(status: OrderStatus, role: OrderRole): readonly OrderTransition[]`, `ORDER_STATUSES`, `ORDER_TRANSITIONS`
  - `packages/contracts` → `catalogListQuerySchema`, `productDetailSchema`, `cartLineInputSchema`, `cartSchema`, `checkoutRequestSchema`, `orderSnapshotSchema`, `orderSummarySchema`, plus les types inférés.

- [ ] **Step 1: Créer le squelette du monorepo et les fichiers de configuration racine**

```bash
mkdir -p packages/{config,contracts,domain}/{src,test} apps/{api,web,worker} e2e ops
```

`pnpm-workspace.yaml` :

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

`package.json` (racine) :

```json
{
  "name": "fanours",
  "private": true,
  "packageManager": "pnpm@9.12.0",
  "engines": { "node": ">=20.11" },
  "scripts": {
    "build": "turbo run build",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck",
    "test": "turbo run test",
    "test:unit": "turbo run test:unit",
    "test:integration": "turbo run test:integration",
    "test:e2e": "playwright test",
    "test:e2e:functional": "playwright test --project=functional",
    "test:e2e:visual": "playwright test --project=visual",
    "test:e2e:shard": "playwright test --shard",
    "db:migrate": "pnpm --filter @fanours/db migrate",
    "db:generate": "pnpm --filter @fanours/db generate"
  },
  "devDependencies": {
    "turbo": "2.3.3",
    "typescript": "5.6.3",
    "vitest": "2.1.8",
    "@types/node": "22.10.2"
  }
}
```

`turbo.json` :

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "lint": { "dependsOn": ["^build"] },
    "typecheck": { "dependsOn": ["^build"] },
    "test": { "dependsOn": ["^build"] },
    "test:unit": { "dependsOn": ["^build"] },
    "test:integration": { "dependsOn": ["^build"], "outputs": [] }
  }
}
```

`tsconfig.base.json` :

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true,
    "paths": {
      "@fanours/config": ["packages/config/src/index.ts"],
      "@fanours/contracts": ["packages/contracts/src/index.ts"],
      "@fanours/domain": ["packages/domain/src/index.ts"]
    }
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
dist/
.turbo/
coverage/
.env
.env.*
!.env.example
playwright-report/
test-results/
packages/db/migrations/*.applied
```

`.env.example` (noms de variables uniquement, aucune valeur) :

```dotenv
NODE_ENV=development
DATABASE_URL=
REDIS_URL=
S3_ENDPOINT=
S3_BUCKET=
S3_ACCESS_KEY_ID=
S3_SECRET_ACCESS_KEY=
PAYMENT_PROVIDER=
PAYMENT_API_KEY=
PAYMENT_WEBHOOK_SECRET=
EMAIL_PROVIDER=
EMAIL_API_KEY=
EMAIL_FROM=
SESSION_COOKIE_SECRET=
TURNSTILE_SECRET_KEY=
OTEL_EXPORTER_OTLP_ENDPOINT=
SENTRY_DSN=
```

- [ ] **Step 2: Créer `packages/config` avec la matrice d'environnement et les feature flags**

`packages/config/src/env-matrix.ts` :

```typescript
export type EnvName = 'local' | 'test' | 'staging' | 'production';

export interface EnvMatrix {
  readonly env: EnvName;
  readonly isProduction: boolean;
  readonly requireHttps: boolean;
  readonly cookieSecure: boolean;
  readonly sessionMaxAgeSeconds: number;
  readonly sessionAbsoluteMaxAgeSeconds: number;
  readonly allowSignup: boolean;
  readonly mockPaymentProvider: boolean;
  readonly logLevel: 'debug' | 'info' | 'warn' | 'error';
}

const MATRICES: Record<EnvName, Omit<EnvMatrix, 'env'>> = {
  local: {
    isProduction: false,
    requireHttps: false,
    cookieSecure: false,
    sessionMaxAgeSeconds: 86_400,
    sessionAbsoluteMaxAgeSeconds: 604_800,
    allowSignup: true,
    mockPaymentProvider: true,
    logLevel: 'debug',
  },
  test: {
    isProduction: false,
    requireHttps: false,
    cookieSecure: false,
    sessionMaxAgeSeconds: 900,
    sessionAbsoluteMaxAgeSeconds: 3_600,
    allowSignup: true,
    mockPaymentProvider: true,
    logLevel: 'warn',
  },
  staging: {
    isProduction: false,
    requireHttps: true,
    cookieSecure: true,
    sessionMaxAgeSeconds: 900,
    sessionAbsoluteMaxAgeSeconds: 86_400,
    allowSignup: true,
    mockPaymentProvider: true,
    logLevel: 'info',
  },
  production: {
    isProduction: true,
    requireHttps: true,
    cookieSecure: true,
    sessionMaxAgeSeconds: 900,
    sessionAbsoluteMaxAgeSeconds: 86_400,
    allowSignup: false,
    mockPaymentProvider: false,
    logLevel: 'info',
  },
};

export function envMatrix(env: NodeJS.ProcessEnv): EnvMatrix {
  const raw = env.APP_ENV ?? 'local';
  if (!['local', 'test', 'staging', 'production'].includes(raw)) {
    throw new Error(`APP_ENV invalide: ${raw}`);
  }
  return { env: raw as EnvName, ...MATRICES[raw as EnvName] };
}
```

`packages/config/src/feature-flags.ts` :

```typescript
export const FLAG_KEYS = ['checkout.v2', 'pricing.tiered_b2b'] as const;
export type FlagKey = (typeof FLAG_KEYS)[number];

export interface FlagContext {
  readonly userId?: string;
  readonly businessId?: string;
  readonly countryCode?: string;
}

export interface FlagRule {
  readonly enabled: boolean;
  readonly rolloutPct: number;
  readonly countries?: readonly string[];
}

export interface FlagStore {
  get(key: FlagKey): Promise<FlagRule>;
}

export function hashToBucket(subject: string, buckets: number): number {
  let hash = 2166136261;
  for (let i = 0; i < subject.length; i += 1) {
    hash ^= subject.charCodeAt(i);
    hash = Math.imul(hash, 16777619) >>> 0;
  }
  return hash % buckets;
}

export async function evaluateFlag(
  flag: FlagKey,
  ctx: FlagContext,
  store: FlagStore,
): Promise<boolean> {
  const rule = await store.get(flag);
  if (!rule.enabled) return false;
  if (rule.countries && ctx.countryCode && !rule.countries.includes(ctx.countryCode)) {
    return false;
  }
  if (rule.rolloutPct >= 100) return true;
  if (rule.rolloutPct <= 0) return false;
  const subject = ctx.businessId ?? ctx.userId;
  if (!subject) return false;
  return hashToBucket(subject, 100) < rule.rolloutPct;
}
```

`packages/config/src/index.ts` :

```typescript
export * from './env-matrix.js';
export * from './feature-flags.js';
```

`packages/config/package.json` :

```json
{
  "name": "@fanours/config",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "src/index.ts",
  "types": "src/index.ts",
  "exports": { ".": "./src/index.ts" },
  "scripts": {
    "build": "tsc -p tsconfig.json --noEmit",
    "lint": "eslint src",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "test": "vitest run",
    "test:unit": "vitest run"
  },
  "devDependencies": { "eslint": "9.17.0", "typescript": "5.6.3", "vitest": "2.1.8" }
}
```

`packages/config/src/env-matrix.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { envMatrix } from './env-matrix.js';

describe('envMatrix', () => {
  it('refuse un environnement inconnu', () => {
    expect(() => envMatrix({ APP_ENV: 'prod' })).toThrow(/APP_ENV invalide/);
  });

  it('impose un cookie secure et une session courte en production', () => {
    const matrix = envMatrix({ APP_ENV: 'production' });
    expect(matrix.cookieSecure).toBe(true);
    expect(matrix.requireHttps).toBe(true);
    expect(matrix.sessionMaxAgeSeconds).toBe(900);
    expect(matrix.allowSignup).toBe(false);
  });

  it('autorise le prestataire de paiement simulé hors production', () => {
    expect(envMatrix({ APP_ENV: 'test' }).mockPaymentProvider).toBe(true);
    expect(envMatrix({ APP_ENV: 'production' }).mockPaymentProvider).toBe(false);
  });
});
```

- [ ] **Step 3: Créer `packages/domain` — monnaie entière et règles d'achat (MOQ, incrément, mode de vente)**

`packages/domain/src/currency.ts` :

```typescript
export const CURRENCY_CODES = ['EUR', 'USD', 'CNY', 'GBP', 'JPY'] as const;
export type CurrencyCode = (typeof CURRENCY_CODES)[number];

/** Nombre de décimales par devise ISO 4217. Toute devise inconnue est refusée. */
const EXPONENTS: Record<CurrencyCode, number> = {
  EUR: 2,
  USD: 2,
  CNY: 2,
  GBP: 2,
  JPY: 0,
};

export function currencyExponent(code: CurrencyCode): number {
  return EXPONENTS[code];
}

export function isCurrencyCode(value: string): value is CurrencyCode {
  return (CURRENCY_CODES as readonly string[]).includes(value);
}
```

`packages/domain/src/money.ts` :

```typescript
import { currencyExponent, type CurrencyCode } from './currency.js';

/**
 * Montant monétaire : entier de minor units + devise. Aucun flottant binaire
 * n'intervient dans le calcul (CDC 4.3).
 */
export interface Money {
  readonly amountMinor: bigint;
  readonly currency: CurrencyCode;
}

const MINOR_UNITS_PER_UNIT = 1_000_000n;

export function toMinorUnits(value: string, currency: CurrencyCode): bigint {
  const trimmed = value.trim();
  if (!/^-?\d+(\.\d+)?$/.test(trimmed)) {
    throw new Error(`Montant invalide: ${value}`);
  }
  const exponent = currencyExponent(currency);
  const negative = trimmed.startsWith('-');
  const [whole, fraction = ''] = (negative ? trimmed.slice(1) : trimmed).split('.');
  if (fraction.length > exponent) {
    throw new Error(`Précision supérieure à la devise ${currency}`);
  }
  const padded = fraction.padEnd(exponent, '0');
  const minor = BigInt(whole) * 10n ** BigInt(exponent) + BigInt(padded === '' ? '0' : padded);
  return negative ? -minor : minor;
}

export function fromMinorUnits(minor: bigint, currency: CurrencyCode): string {
  const exponent = currencyExponent(currency);
  if (exponent === 0) return minor.toString();
  const scale = 10n ** BigInt(exponent);
  const negative = minor < 0n;
  const absolute = negative ? -minor : minor;
  const whole = absolute / scale;
  const fraction = (absolute % scale).toString().padStart(exponent, '0');
  return `${negative ? '-' : ''}${whole}.${fraction}`;
}

/** Multiplication exacte par une quantité entière et un taux en points de base (1 bps = 0,01 %). */
export function multiplyMinor(minor: bigint, qty: number, bps: number): bigint {
  if (!Number.isInteger(qty) || qty < 0) throw new Error(`Quantité invalide: ${qty}`);
  if (!Number.isInteger(bps)) throw new Error(`Taux invalide: ${bps}`);
  return (minor * BigInt(qty) * BigInt(bps) + 5_000n) / 10_000n;
}

export function sumMinor(values: readonly bigint[]): bigint {
  return values.reduce((acc, value) => acc + value, 0n);
}

export function formatMoney(money: Money, locale: string): string {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency: money.currency,
  }).format(Number(money.amountMinor) / 10 ** currencyExponent(money.currency));
}
```

`packages/domain/src/money.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { fromMinorUnits, multiplyMinor, toMinorUnits } from './money.js';

describe('money', () => {
  it('convertit un décimal exact en minor units', () => {
    expect(toMinorUnits('15.00', 'EUR')).toBe(1500n);
    expect(toMinorUnits('10.5', 'EUR')).toBe(1050n);
    expect(toMinorUnits('1234', 'JPY')).toBe(1234n);
  });

  it('refuse une précision supérieure à la devise', () => {
    expect(() => toMinorUnits('1.005', 'EUR')).toThrow(/Précision/);
    expect(() => toMinorUnits('1.5', 'JPY')).toThrow(/Précision/);
  });

  it('refuse une chaîne non numérique', () => {
    expect(() => toMinorUnits('12,50', 'EUR')).toThrow(/Montant invalide/);
  });

  it('fait l’aller-retour exact', () => {
    expect(fromMinorUnits(toMinorUnits('987.65', 'EUR'), 'EUR')).toBe('987.65');
  });

  it('multiplie en arithmétique entière avec arrondi au centime supérieur', () => {
    expect(multiplyMinor(333n, 3, 2000)).toBe(200n);
    expect(multiplyMinor(1n, 1, 3333)).toBe(1n);
  });
});
```

`packages/domain/src/purchase-rules.ts` :

```typescript
export type SalesMode = 'wholesale' | 'retail';

export interface PurchaseRulesInput {
  /** Modes de vente ouverts sur le produit (CDC 2.2). */
  readonly salesModes: readonly SalesMode[];
  /** MOQ global du produit. */
  readonly moq: number;
  /** Incrément de commande global (6 / 12 / 24 …). */
  readonly orderMultiple: number;
  /** MOQ propre à la variante, prioritaire sur le MOQ global. */
  readonly variantMoq?: number | null;
  /** Incrément propre à la variante, prioritaire sur l'incrément global. */
  readonly variantOrderMultiple?: number | null;
  /** Mode de vente demandé par l'acheteur. */
  readonly requestedMode: SalesMode;
  /** Quantité demandée. */
  readonly quantity: number;
}

export interface PurchaseRulesResult {
  readonly allowed: boolean;
  /** MOQ applicable, en tenant compte du mode de vente. */
  readonly minQuantity: number;
  /** Incrément applicable. */
  readonly orderMultiple: number;
  /** Prochaine quantité autorisée ≥ quantity (0 si la quantité demandée est valide). */
  readonly nextValidQuantity: number;
  /** Code d'erreur stable, lisible par l'API. */
  readonly reason:
    | 'ok'
    | 'sales_mode_disabled'
    | 'below_moq'
    | 'not_multiple'
    | 'invalid_quantity';
}

export function resolvePurchaseRules(input: PurchaseRulesInput): PurchaseRulesResult {
  const { salesModes, requestedMode } = input;
  if (!salesModes.includes(requestedMode)) {
    return {
      allowed: false,
      minQuantity: input.moq,
      orderMultiple: input.orderMultiple,
      nextValidQuantity: 0,
      reason: 'sales_mode_disabled',
    };
  }
  const minQuantity = input.variantMoq ?? input.moq;
  const orderMultiple = input.variantOrderMultiple ?? input.orderMultiple;
  if (!Number.isInteger(input.quantity) || input.quantity <= 0) {
    return {
      allowed: false,
      minQuantity,
      orderMultiple,
      nextValidQuantity: 0,
      reason: 'invalid_quantity',
    };
  }
  if (input.quantity < minQuantity) {
    const steps = Math.ceil(minQuantity / orderMultiple);
    return {
      allowed: false,
      minQuantity,
      orderMultiple,
      nextValidQuantity: steps * orderMultiple,
      reason: 'below_moq',
    };
  }
  if (input.quantity % orderMultiple !== 0) {
    const steps = Math.ceil(input.quantity / orderMultiple);
    return {
      allowed: false,
      minQuantity,
      orderMultiple,
      nextValidQuantity: steps * orderMultiple,
      reason: 'not_multiple',
    };
  }
  return {
    allowed: true,
    minQuantity,
    orderMultiple,
    nextValidQuantity: 0,
    reason: 'ok',
  };
}
```

`packages/domain/src/purchase-rules.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { resolvePurchaseRules } from './purchase-rules.js';

const base = {
  salesModes: ['wholesale', 'retail'] as const,
  moq: 6,
  orderMultiple: 6,
  requestedMode: 'wholesale' as const,
  quantity: 12,
};

describe('resolvePurchaseRules', () => {
  it('accepte une quantité conforme au MOQ et à l’incrément', () => {
    expect(resolvePurchaseRules(base).allowed).toBe(true);
  });

  it('refuse le retail lorsque le produit est uniquement en gros', () => {
    const result = resolvePurchaseRules({
      ...base,
      salesModes: ['wholesale'],
      requestedMode: 'retail',
    });
    expect(result.allowed).toBe(false);
    expect(result.reason).toBe('sales_mode_disabled');
  });

  it('refuse une quantité sous le MOQ et propose la prochaine quantité valide', () => {
    const result = resolvePurchaseRules({ ...base, quantity: 4 });
    expect(result.reason).toBe('below_moq');
    expect(result.nextValidQuantity).toBe(6);
  });

  it('refuse une quantité qui n’est pas un multiple de l’incrément', () => {
    const result = resolvePurchaseRules({ ...base, quantity: 13 });
    expect(result.reason).toBe('not_multiple');
    expect(result.nextValidQuantity).toBe(18);
  });

  it('donne la priorité au MOQ et à l’incrément de la variante', () => {
    const result = resolvePurchaseRules({
      ...base,
      moq: 6,
      orderMultiple: 6,
      variantMoq: 12,
      variantOrderMultiple: 12,
      quantity: 13,
    });
    expect(result.minQuantity).toBe(12);
    expect(result.orderMultiple).toBe(12);
    expect(result.nextValidQuantity).toBe(24);
  });
});
```

- [ ] **Step 4: Créer `packages/domain` — machine à états des commandes (20 états, 34 transitions)**

`packages/domain/src/order-state-machine.ts` :

```typescript
export const ORDER_STATUSES = [
  'draft',
  'pending_payment',
  'payment_processing',
  'payment_failed',
  'paid',
  'cancelled',
  'cancellation_requested',
  'awaiting_review',
  'confirmed',
  'processing',
  'partially_fulfilled',
  'shipped',
  'in_transit',
  'customs_action_required',
  'delivered',
  'disputed',
  'completed',
  'refund_pending',
  'partially_refunded',
  'refunded',
] as const;

export type OrderStatus = (typeof ORDER_STATUSES)[number];

export type OrderRole =
  | 'customer'
  | 'system'
  | 'admin'
  | 'compliance_manager'
  | 'order_manager'
  | 'logistics_operator'
  | 'finance_manager'
  | 'support_agent';

export interface OrderTransition {
  readonly id: string;
  readonly from: OrderStatus;
  readonly to: OrderStatus;
  readonly roles: readonly OrderRole[];
  /** `true` si le rang de statut peut diminuer (seules T10 et T32 le permettent). */
  readonly allowsRankDecrease: boolean;
}

/** Source unique de vérité (architecture.md §5). Un `CHECK` SQL est dérivé de cette table. */
export const ORDER_TRANSITIONS: readonly OrderTransition[] = [
  { id: 'T01', from: 'draft', to: 'pending_payment', roles: ['customer', 'system'], allowsRankDecrease: false },
  { id: 'T02', from: 'pending_payment', to: 'payment_processing', roles: ['system'], allowsRankDecrease: false },
  { id: 'T03', from: 'payment_processing', to: 'paid', roles: ['system'], allowsRankDecrease: false },
  { id: 'T04', from: 'pending_payment', to: 'payment_failed', roles: ['system'], allowsRankDecrease: false },
  { id: 'T04b', from: 'payment_processing', to: 'payment_failed', roles: ['system'], allowsRankDecrease: false },
  { id: 'T05', from: 'payment_failed', to: 'pending_payment', roles: ['customer'], allowsRankDecrease: false },
  { id: 'T06', from: 'paid', to: 'awaiting_review', roles: ['system'], allowsRankDecrease: false },
  { id: 'T07', from: 'awaiting_review', to: 'confirmed', roles: ['compliance_manager', 'order_manager'], allowsRankDecrease: false },
  { id: 'T08', from: 'awaiting_review', to: 'cancelled', roles: ['compliance_manager'], allowsRankDecrease: false },
  { id: 'T09', from: 'confirmed', to: 'processing', roles: ['logistics_operator'], allowsRankDecrease: false },
  { id: 'T10', from: 'confirmed', to: 'awaiting_review', roles: ['compliance_manager'], allowsRankDecrease: true },
  { id: 'T11', from: 'processing', to: 'partially_fulfilled', roles: ['logistics_operator'], allowsRankDecrease: false },
  { id: 'T12', from: 'processing', to: 'shipped', roles: ['logistics_operator'], allowsRankDecrease: false },
  { id: 'T13', from: 'processing', to: 'cancelled', roles: ['order_manager'], allowsRankDecrease: false },
  { id: 'T14', from: 'partially_fulfilled', to: 'shipped', roles: ['logistics_operator'], allowsRankDecrease: false },
  { id: 'T15', from: 'partially_fulfilled', to: 'partially_fulfilled', roles: ['logistics_operator'], allowsRankDecrease: false },
  { id: 'T16', from: 'shipped', to: 'in_transit', roles: ['system'], allowsRankDecrease: false },
  { id: 'T17', from: 'in_transit', to: 'customs_action_required', roles: ['system', 'logistics_operator'], allowsRankDecrease: false },
  { id: 'T18', from: 'customs_action_required', to: 'in_transit', roles: ['logistics_operator'], allowsRankDecrease: false },
  { id: 'T19', from: 'in_transit', to: 'delivered', roles: ['system'], allowsRankDecrease: false },
  { id: 'T20', from: 'delivered', to: 'completed', roles: ['system', 'customer', 'support_agent'], allowsRankDecrease: false },
  { id: 'T21', from: 'delivered', to: 'disputed', roles: ['customer', 'support_agent'], allowsRankDecrease: false },
  { id: 'T22', from: 'disputed', to: 'completed', roles: ['support_agent'], allowsRankDecrease: false },
  { id: 'T23', from: 'delivered', to: 'cancellation_requested', roles: ['customer'], allowsRankDecrease: false },
  { id: 'T23b', from: 'completed', to: 'cancellation_requested', roles: ['customer'], allowsRankDecrease: false },
  { id: 'T24', from: 'cancellation_requested', to: 'cancelled', roles: ['order_manager'], allowsRankDecrease: false },
  { id: 'T25', from: 'cancellation_requested', to: 'delivered', roles: ['order_manager'], allowsRankDecrease: false },
  { id: 'T25b', from: 'cancellation_requested', to: 'completed', roles: ['order_manager'], allowsRankDecrease: false },
  { id: 'T26', from: 'cancelled', to: 'refund_pending', roles: ['finance_manager'], allowsRankDecrease: false },
  { id: 'T27', from: 'paid', to: 'refund_pending', roles: ['finance_manager'], allowsRankDecrease: false },
  { id: 'T27b', from: 'confirmed', to: 'refund_pending', roles: ['finance_manager'], allowsRankDecrease: false },
  { id: 'T27c', from: 'processing', to: 'refund_pending', roles: ['finance_manager'], allowsRankDecrease: false },
  { id: 'T28', from: 'refund_pending', to: 'partially_refunded', roles: ['system'], allowsRankDecrease: false },
  { id: 'T29', from: 'refund_pending', to: 'refunded', roles: ['system'], allowsRankDecrease: false },
  { id: 'T30', from: 'partially_refunded', to: 'refunded', roles: ['system'], allowsRankDecrease: false },
  { id: 'T31', from: 'partially_refunded', to: 'refunded', roles: ['finance_manager'], allowsRankDecrease: false },
  { id: 'T31b', from: 'refunded', to: 'partially_refunded', roles: ['finance_manager'], allowsRankDecrease: true },
  { id: 'T32', from: 'refund_pending', to: 'cancelled', roles: ['finance_manager'], allowsRankDecrease: true },
  { id: 'T33', from: 'delivered', to: 'completed', roles: ['support_agent'], allowsRankDecrease: false },
  { id: 'T34', from: 'disputed', to: 'partially_refunded', roles: ['finance_manager'], allowsRankDecrease: false },
  { id: 'T34b', from: 'disputed', to: 'refunded', roles: ['finance_manager'], allowsRankDecrease: false },
];

export function allowedTransitions(status: OrderStatus, role: OrderRole): readonly OrderTransition[] {
  return ORDER_TRANSITIONS.filter(
    (transition) => transition.from === status && transition.roles.includes(role),
  );
}

export function findTransition(id: string): OrderTransition {
  const transition = ORDER_TRANSITIONS.find((candidate) => candidate.id === id);
  if (!transition) throw new Error(`Transition inconnue: ${id}`);
  return transition;
}
```

`packages/domain/src/order-state-machine.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { ORDER_STATUSES, ORDER_TRANSITIONS, allowedTransitions, findTransition } from './order-state-machine.js';

describe('machine à états des commandes', () => {
  it('couvre les 20 états du cahier des charges', () => {
    expect(ORDER_STATUSES).toHaveLength(20);
  });

  it('ne contient que des transitions issues d’états connus', () => {
    for (const transition of ORDER_TRANSITIONS) {
      expect(ORDER_STATUSES).toContain(transition.from);
      expect(ORDER_STATUSES).toContain(transition.to);
    }
  });

  it('interdit le passage direct de pending_payment à paid', () => {
    const targets = allowedTransitions('pending_payment', 'system').map((t) => t.to);
    expect(targets).not.toContain('paid');
  });

  it('réserve paid au système alimenté par le webhook', () => {
    const toPaid = ORDER_TRANSITIONS.filter((t) => t.to === 'paid');
    expect(toPaid).toHaveLength(1);
    expect(toPaid[0]?.from).toBe('payment_processing');
    expect(toPaid[0]?.roles).toEqual(['system']);
  });

  it('n’autorise le remboursement qu’après paid au minimum', () => {
    expect(allowedTransitions('draft', 'finance_manager')).toHaveLength(0);
    expect(findTransition('T27').from).toBe('paid');
  });

  it('limite les transitions avec retour de rang à deux exceptions motivées', () => {
    const decreasing = ORDER_TRANSITIONS.filter((t) => t.allowsRankDecrease);
    expect(decreasing.map((t) => t.id)).toEqual(['T10', 'T31b', 'T32']);
  });
});
```

`packages/domain/src/index.ts` :

```typescript
export * from './currency.js';
export * from './money.js';
export * from './purchase-rules.js';
export * from './order-state-machine.js';
```

- [ ] **Step 5: Créer `packages/contracts` — schémas zod partagés (entrée et sortie)**

`packages/contracts/src/catalog.ts` :

```typescript
import { z } from 'zod';

export const localeSchema = z.enum(['fr-FR', 'en', 'zh-CN']);

export const salesModeSchema = z.enum(['wholesale', 'retail']);

export const currencyCodeSchema = z.enum(['EUR', 'USD', 'CNY', 'GBP', 'JPY']);

export const priceTierSchema = z.object({
  minQuantity: z.number().int().positive(),
  maxQuantity: z.number().int().positive().nullable(),
  unitPrice: z.string().regex(/^\d+(\.\d{1,4})?$/),
  currency: currencyCodeSchema,
  validFrom: z.string().datetime(),
  validTo: z.string().datetime().nullable(),
});

export const catalogListQuerySchema = z
  .object({
    category: z.string().min(1).max(120).optional(),
    q: z.string().min(1).max(200).optional(),
    salesMode: salesModeSchema.optional(),
    minQuantity: z.coerce.number().int().positive().max(100_000).optional(),
    country: z.string().regex(/^[A-Z]{2}$/).optional(),
    sort: z.enum(['relevance', 'newness', 'price_asc', 'price_desc']).default('relevance'),
    cursor: z.string().max(200).optional(),
    limit: z.coerce.number().int().min(1).max(60).default(24),
  })
  .strict();

export const productDetailSchema = z.object({
  id: z.string().uuid(),
  sku: z.string().min(1),
  slug: z.string().min(1),
  name: z.string().min(1),
  shortDescription: z.string(),
  description: z.string(),
  categorySlug: z.string().nullable(),
  salesModes: z.array(salesModeSchema).min(1),
  moq: z.number().int().positive(),
  orderMultiple: z.number().int().positive(),
  variants: z.array(
    z.object({
      id: z.string().uuid(),
      sku: z.string().min(1),
      attributes: z.record(z.string(), z.string()),
      media: z.array(
        z.object({
          url: z.string().url(),
          altText: z.string().min(1),
          width: z.number().int().positive(),
          height: z.number().int().positive(),
        }),
      ),
    }),
  ),
  priceTiers: z.array(priceTierSchema).min(1),
  availableQuantity: z.number().int().nonnegative(),
  prepDelayDaysMin: z.number().int().nonnegative(),
  prepDelayDaysMax: z.number().int().nonnegative(),
  allowedCountries: z.array(z.string().regex(/^[A-Z]{2}$/)),
  weightGrams: z.number().int().positive(),
  complianceDocuments: z.array(z.string()),
  indicativeImportCost: z
    .object({
      low: z.string().regex(/^\d+(\.\d{1,4})?$/),
      high: z.string().regex(/^\d+(\.\d{1,4})?$/),
      currency: currencyCodeSchema,
      certainty: z.enum(['validated', 'estimated', 'incomplete', 'specialist_review']),
      text: z.string().min(1),
    })
    .nullable(),
});

export type CatalogListQuery = z.infer<typeof catalogListQuerySchema>;
export type ProductDetail = z.infer<typeof productDetailSchema>;
```

`packages/contracts/src/cart.ts` :

```typescript
import { z } from 'zod';

export const cartLineInputSchema = z
  .object({
    variantId: z.string().uuid(),
    quantity: z.number().int().positive().max(1_000_000),
    /** Prix éventuellement vu par le client : recalculé et ignoré côté serveur. */
    clientUnitPrice: z.string().optional(),
  })
  .strict();

export const cartLineSchema = z.object({
  variantId: z.string().uuid(),
  productId: z.string().uuid(),
  sku: z.string(),
  name: z.string(),
  quantity: z.number().int().positive(),
  minQuantity: z.number().int().positive(),
  orderMultiple: z.number().int().positive(),
  unitPrice: z.string(),
  currency: z.string().length(3),
  lineTotal: z.string(),
  tierLabel: z.string().nullable(),
  violations: z.array(
    z.enum(['sales_mode_disabled', 'below_moq', 'not_multiple', 'invalid_quantity', 'stock_conflict']),
  ),
});

export const cartSchema = z.object({
  id: z.string().uuid(),
  currency: z.string().length(3),
  lines: z.array(cartLineSchema),
  subtotal: z.string(),
  totalQuantity: z.number().int().nonnegative(),
  updatedAt: z.string().datetime(),
});

export type CartLineInput = z.infer<typeof cartLineInputSchema>;
export type Cart = z.infer<typeof cartSchema>;
```

`packages/contracts/src/checkout.ts` :

```typescript
import { z } from 'zod';

export const incotermSchema = z.enum(['EXW', 'FOB', 'CIF', 'DAP', 'DDP']);

export const checkoutRequestSchema = z
  .object({
    cartId: z.string().uuid(),
    addressId: z.string().uuid(),
    incoterm: incotermSchema,
    salesMode: z.enum(['wholesale', 'retail']),
    destinationCountry: z.string().regex(/^[A-Z]{2}$/),
    /** Clé d'idempotence : deux soumissions avec la même clé ne créent qu'une commande. */
    idempotencyKey: z.string().min(16).max(120),
    /** Montants éventuellement affichés par le client : recalculés et ignorés. */
    clientTotals: z
      .object({
        subtotal: z.string().optional(),
        shipping: z.string().optional(),
        duties: z.string().optional(),
        taxes: z.string().optional(),
        total: z.string().optional(),
      })
      .optional(),
  })
  .strict();

export const orderSnapshotSchema = z.object({
  version: z.literal(1),
  takenAt: z.string().datetime(),
  address: z.object({
    fullName: z.string(),
    lines: z.array(z.string()),
    city: z.string(),
    region: z.string().nullable(),
    postalCode: z.string(),
    countryCode: z.string().length(2),
    phoneE164: z.string(),
  }),
  lines: z.array(
    z.object({
      variantId: z.string().uuid(),
      sku: z.string(),
      quantity: z.number().int().positive(),
      unitPrice: z.string(),
      priceTierMinQuantity: z.number().int().positive(),
      priceTierCurrency: z.string().length(3),
      taxAmount: z.string(),
      dutyAmount: z.string(),
      lineTotal: z.string(),
    }),
  ),
  salesMode: z.enum(['wholesale', 'retail']),
  incoterm: incotermSchema,
  destinationCountry: z.string().length(2),
  currency: z.string().length(3),
  subtotal: z.string(),
  shipping: z.string(),
  duties: z.string(),
  taxes: z.string(),
  total: z.string(),
  termsVersion: z.string(),
  taxRuleRefs: z.array(
    z.object({
      ruleId: z.string().uuid(),
      versionId: z.string().uuid(),
      sourceUrl: z.string().url(),
      computedAt: z.string().datetime(),
    }),
  ),
});

export type CheckoutRequest = z.infer<typeof checkoutRequestSchema>;
export type OrderSnapshot = z.infer<typeof orderSnapshotSchema>;
```

`packages/contracts/src/order.ts` :

```typescript
import { z } from 'zod';
import { orderSnapshotSchema } from './checkout.js';

export const orderSummarySchema = z.object({
  id: z.string().uuid(),
  number: z.string().min(1),
  status: z.string().min(1),
  currency: z.string().length(3),
  total: z.string(),
  placedAt: z.string().datetime(),
  paidAt: z.string().datetime().nullable(),
  salesMode: z.enum(['wholesale', 'retail']),
});

export const orderDetailSchema = orderSummarySchema.extend({
  snapshot: orderSnapshotSchema,
  lines: z.array(
    z.object({
      variantId: z.string().uuid(),
      sku: z.string(),
      quantity: z.number().int().positive(),
      unitPrice: z.string(),
      lineTotal: z.string(),
    }),
  ),
  statusHistory: z.array(
    z.object({
      fromStatus: z.string(),
      toStatus: z.string(),
      occurredAt: z.string().datetime(),
      reason: z.string().nullable(),
    }),
  ),
  documents: z.array(
    z.object({
      id: z.string().uuid(),
      type: z.enum(['commercial_export', 'fiscal', 'receipt', 'credit_note']),
      number: z.string(),
      issuedAt: z.string().datetime(),
      downloadUrl: z.string().url(),
    }),
  ),
});

export type OrderSummary = z.infer<typeof orderSummarySchema>;
export type OrderDetail = z.infer<typeof orderDetailSchema>;
```

`packages/contracts/src/index.ts` :

```typescript
export * from './catalog.js';
export * from './cart.js';
export * from './checkout.js';
export * from './order.js';
```

`packages/contracts/src/catalog.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { catalogListQuerySchema, productDetailSchema } from './catalog.js';

describe('contrats catalogue', () => {
  it('rejette un paramètre inconnu pour ne pas laisser passer une requête non contrôlée', () => {
    const result = catalogListQuerySchema.safeParse({ sort: 'price_asc', evil: '1' });
    expect(result.success).toBe(false);
  });

  it('borne la taille de page à 60', () => {
    expect(catalogListQuerySchema.safeParse({ limit: '61' }).success).toBe(false);
    expect(catalogListQuerySchema.safeParse({ limit: '24' }).success).toBe(true);
  });

  it('refuse une quantité négative dans la requête', () => {
    expect(catalogListQuerySchema.safeParse({ minQuantity: '-1' }).success).toBe(false);
  });

  it('exige au moins un palier de prix dans le détail produit', () => {
    const base = {
      id: '3f1d1f2e-6a2f-4f2a-9b1e-000000000001',
      sku: 'SKU-1',
      slug: 'produit-1',
      name: 'Produit',
      shortDescription: 'court',
      description: 'long',
      categorySlug: null,
      salesModes: ['wholesale'],
      moq: 6,
      orderMultiple: 6,
      variants: [],
      priceTiers: [],
      availableQuantity: 0,
      prepDelayDaysMin: 1,
      prepDelayDaysMax: 2,
      allowedCountries: ['FR'],
      weightGrams: 500,
      complianceDocuments: [],
      indicativeImportCost: null,
    };
    expect(productDetailSchema.safeParse(base).success).toBe(false);
    expect(
      productDetailSchema.safeParse({
        ...base,
        priceTiers: [
          {
            minQuantity: 1,
            maxQuantity: null,
            unitPrice: '10.00',
            currency: 'EUR',
            validFrom: '2026-01-01T00:00:00.000Z',
            validTo: null,
          },
        ],
      }).success,
    ).toBe(true);
  });
});
```

- [ ] **Step 6: Vérifier que les paquets compilent et que les tests passent**

```bash
pnpm install
pnpm --filter @fanours/domain test
pnpm --filter @fanours/config test
pnpm --filter @fanours/contracts test
pnpm typecheck
```

Attendu : `Test Files  3 passed (3)` et `Tests  13 passed (13)` pour `@fanours/domain`, `Tests  3 passed (3)` pour `@fanours/config`, `Tests  4 passed (4)` pour `@fanours/contracts` ; `tsc` sans erreur (sortie vide, code 0).

- [ ] **Step 7: Commit**

```bash
git add pnpm-workspace.yaml turbo.json package.json tsconfig.base.json .npmrc .gitignore .env.example packages/config packages/contracts packages/domain
git commit -m "feat(infra): initialise le monorepo et les paquets config, contracts et domain

Introduit la matrice d'environnement, les feature flags a evaluation deterministe,
la conversion monetaire en minor units, les regles d'achat MOQ/increment et la
machine a etats des commandes (20 etats, 34 transitions).

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

### Task 2: Extension de `packages/db` — schémas Drizzle métier et migrations SQL `0006` à `0015`

**Dépend de :** `packages/db` existe déjà, créé par la **Task 2 de la Phase 1**. Le runner `packages/db/scripts/migrate.ts` (CLI) et son cœur `packages/db/src/migrate-core.ts` (idempotent, checksum SHA-256, table `schema_migrations`), ainsi que la migration d'extensions `0001_extensions_and_migration_table.sql`, sont **déjà en place et déjà appliqués** : la Phase 2 ne les recrée pas et ne modifie aucune migration existante (les migrations sont append-only). La Phase 2 **étend** le paquet : elle ajoute les modules de schéma Drizzle `packages/db/src/schema/*.ts`, les migrations `0006` à `0015` et les points d'export correspondants. `packages/db/src/index.ts` reste le baril du paquet : la Phase 1 l'a écrit, la Phase 2 l'étend.

**Files:**
- Modify: `packages/db/package.json` (points d'export `./client`, `./migrate` et `./schema` ajoutés ; contenu de la Phase 1 conservé)
- Modify: `tsconfig.base.json` (chemins `@fanours/db/client`, `@fanours/db/migrate` et `@fanours/db/schema` ajoutés ; aucun chemin existant retiré)
- Modify: `packages/db/drizzle.config.ts` (le schéma déclaré devient `./src/schema.ts` + `./src/schema/*.ts`)
- Modify: `packages/db/vitest.config.ts` (le projet `integration` inclut désormais `test/**/*.test.ts`)
- Modify: `packages/db/src/client.ts` (schéma combiné : schémas Phase 1 + schémas Phase 2 ; `hashToken`, `newOpaqueToken`, `withTransaction` conservés)
- Modify: `packages/db/src/index.ts` (baril : ré-exporte `./schema/index.js`, `./client.js` et `./transaction.js` ; le module `./outbox.js` est ajouté par la Task 3 qui le crée)
- Modify: `packages/db/src/schema.ts` (colonnes Phase 2 ajoutées aux tables `outbox_event` et `audit_log`, qui restent définies par la Phase 1)
- Create: `packages/db/src/transaction.ts`, `packages/db/src/migrate.ts` (adaptateur du runner de la Phase 1 `packages/db/src/migrate-core.ts`, pas un second runner)
- Create: `packages/db/src/schema/index.ts` (baril des schémas : point d'entrée unique de `src/client.ts` et du point d'export `@fanours/db/schema`). Les modules métier `catalog.ts`, `pricing.ts`, `stock.ts`, `cart.ts`, `order.ts`, `payment.ts` et `notification.ts` sont créés par la task qui porte le domaine concerné (Tasks 4, 5, 6, 11, 13, 14, 18), qui étendent chacune ce baril : à la fin de la Phase 2, le baril exporte les schémas de la Phase 1 (`../schema.js`) et les sept modules métier.
- Create: `packages/db/migrations/0006_catalog.sql`, `packages/db/migrations/0007_pricing.sql`, `packages/db/migrations/0008_stock.sql`, `packages/db/migrations/0009_cart.sql`, `packages/db/migrations/0010_order.sql`, `packages/db/migrations/0011_payment.sql`, `packages/db/migrations/0012_notification_and_outbox.sql`
- Create: `packages/db/test/migrate.test.ts`, `packages/db/test/schema-invariants.test.ts`

**Interfaces:**
- Consumes: `packages/domain` (`OrderStatus`, `CurrencyCode`).
- Produces:
  - `packages/db` → `db` (client Drizzle typé), `withTransaction<T>(fn: (tx: Tx) => Promise<T>): Promise<T>`, `runMigrations(pool: Pool): Promise<{ applied: string[]; skipped: string[] }>`, et les tables exportées `product`, `productTranslation`, `productVariant`, `productMedia`, `productCompliance`, `category`, `categoryTranslation`, `priceTier`, `customerPriceRule`, `warehouse`, `inventoryItem`, `inventoryMovement`, `stockReservation`, `cart`, `cartItem`, `orders`, `orderLine`, `orderStatusEvent`, `payment`, `paymentEvent`, `refund`, `invoice`, `notification`, `notificationPreference`, `emailTemplate`, `emailDelivery`, `outboxEvent`, `consumedEvent`, `auditLog`. Les tables `webhookEvent`, `featureFlag`, `countryProfile` et `taxEstimate` (Task 7, migration `0013_search.sql`), `tradeRuleSource`, `tradeRule`, `tradeRuleVersion` et `complianceAlert` (Task 8, migration `0014_trade_rules.sql`) et `invoiceSequence` (Task 17, migration `0015_invoice_sequence.sql`) sont ajoutées plus tard dans la phase, par les tasks qui portent leur migration.
  - `outboxEvent`, `consumedEvent` et `auditLog` sont **définis par la Phase 1** dans `packages/db/src/schema.ts` (tables créées par `0004_outbox.sql` et `0003_rbac_and_audit.sql`) : la Phase 2 les **étend** par `ALTER TABLE` dans `0012_notification_and_outbox.sql` et met à jour leur définition Drizzle, elle ne les recrée pas. `dutyEstimate` n'existe pas comme table : l'estimation des droits est portée par `taxEstimate` (bornes `low` / `high` et niveau de `certainty`).

- [ ] **Step 1: Étendre le paquet `packages/db` (points d'export et configuration de test) sans recréer le runner**

`packages/db/package.json` (modifié — le contenu écrit par la Phase 1 est conservé tel quel, la Phase 2 ajoute deux points d'export consommés par `apps/api` et `packages/jobs`) :

```json
{
  "name": "@fanours/db",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" },
    "./client": { "types": "./dist/client.d.ts", "default": "./dist/client.js" },
    "./migrate": { "types": "./dist/migrate.d.ts", "default": "./dist/migrate.js" },
    "./schema": { "types": "./dist/schema/index.d.ts", "default": "./dist/schema/index.js" }
  },
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

Le script `migrate` reste celui de la Phase 1 (`tsx scripts/migrate.ts`) : la Phase 2 n'introduit pas de second runner, elle n'ajoute qu'un adaptateur (`src/migrate.ts`) au-dessus du cœur du runner de la Phase 1 (`src/migrate-core.ts`).

`tsconfig.base.json` (modifié — le fichier écrit par la Phase 1 est conservé ; la Phase 2 ajoute les chemins des sous-exports qu'elle introduit, sans retirer aucun chemin existant) :

```json
{
  "compilerOptions": {
    "paths": {
      "@fanours/config": ["packages/config/src/index.ts"],
      "@fanours/contracts": ["packages/contracts/src/index.ts"],
      "@fanours/domain": ["packages/domain/src/index.ts"],
      "@fanours/db": ["packages/db/src/index.ts"],
      "@fanours/db/client": ["packages/db/src/client.ts"],
      "@fanours/db/migrate": ["packages/db/src/migrate.ts"],
      "@fanours/db/schema": ["packages/db/src/schema/index.ts"],
      "@fanours/pricing": ["packages/pricing/src/index.ts"],
      "@fanours/compliance": ["packages/compliance/src/index.ts"],
      "@fanours/i18n": ["packages/i18n/src/index.ts"],
      "@fanours/ui": ["packages/ui/src/index.ts"],
      "@fanours/observability": ["packages/observability/src/index.ts"]
    }
  }
}
```

Ces chemins pointent vers les sources : `@fanours/db/migrate` et `@fanours/db/schema` se résolvent sans exiger une compilation préalable du paquet, comme le chemin `@fanours/db` de la Phase 1.

`packages/db/drizzle.config.ts` (modifié — la Phase 1 déclare le seul fichier `src/schema.ts` ; la Phase 2 ajoute ses modules de schéma, la génération reste informative et n'est jamais appliquée automatiquement) :

```typescript
import type { Config } from 'drizzle-kit';

export default {
  schema: ['./src/schema.ts', './src/schema/*.ts'],
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env['DATABASE_URL'] ?? 'postgres://fanours:fanours@localhost:5432/fanours',
  },
  strict: true,
  verbose: true,
} satisfies Config;
```

`packages/db/vitest.config.ts` (modifié — le projet `integration` inclut désormais les tests de la Phase 2, placés dans `packages/db/test/`) :

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
          include: ['src/__tests__/**/*.integration.test.ts', 'test/**/*.test.ts'],
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

La table de suivi `schema_migrations` et les extensions `pgcrypto`, `unaccent`, `btree_gist` sont **déjà créées** par `packages/db/migrations/0001_extensions_and_migration_table.sql` (Phase 1, Task 2). Cette migration n'est ni recopiée ni modifiée ici : les migrations sont append-only et une migration déjà appliquée ne se rejoue jamais. La première migration écrite par la Phase 2 est `0006_catalog.sql`.

- [ ] **Step 2: Écrire les migrations catalogue (`0006`) et tarification (`0007`)**

`packages/db/migrations/0006_catalog.sql` :

```sql
-- 0006_catalog.sql
-- Catalogue : categories, produits, traductions, variantes, medias.
-- Append-only : aucun script down, checksum SHA-256 verifie par le runner.
SET lock_timeout = '3s';
SET statement_timeout = '30s';

CREATE TABLE category (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id   uuid REFERENCES category(id),
  position    int NOT NULL DEFAULT 0,
  created_at  timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT category_no_self_parent CHECK (parent_id IS NULL OR parent_id <> id)
);
CREATE INDEX idx_category_parent ON category(parent_id, position);

CREATE TABLE category_translation (
  category_id uuid NOT NULL REFERENCES category(id) ON DELETE CASCADE,
  locale      char(3) NOT NULL,
  name        text NOT NULL,
  slug        text NOT NULL,
  CONSTRAINT category_translation_pk PRIMARY KEY (category_id, locale),
  CONSTRAINT category_translation_locale_check CHECK (locale IN ('fr-FR','en','zh-CN')),
  CONSTRAINT category_translation_slug_unique UNIQUE (locale, slug)
);

CREATE TABLE product (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  sku             text NOT NULL,
  status          text NOT NULL DEFAULT 'draft',
  sales_modes     text[] NOT NULL,
  moq             int NOT NULL,
  order_multiple  int NOT NULL,
  category_id     uuid REFERENCES category(id),
  origin_country  char(2) NOT NULL,
  hs_code         text,
  net_weight_g    int NOT NULL,
  gross_weight_g  int NOT NULL,
  length_mm       int NOT NULL,
  width_mm        int NOT NULL,
  height_mm       int NOT NULL,
  prep_delay_min_days int NOT NULL DEFAULT 0,
  prep_delay_max_days int NOT NULL DEFAULT 0,
  restock_lead_days    int NOT NULL DEFAULT 0,
  safety_stock    int NOT NULL DEFAULT 0,
  required_documents text[] NOT NULL DEFAULT '{}',
  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT product_sku_unique UNIQUE (sku),
  CONSTRAINT product_status_check CHECK (status IN ('draft','in_review','published','suspended','archived')),
  CONSTRAINT product_moq_check CHECK (moq > 0),
  CONSTRAINT product_multiple_check CHECK (order_multiple > 0),
  CONSTRAINT product_sales_modes_check CHECK (cardinality(sales_modes) > 0 AND sales_modes <@ ARRAY['wholesale','retail']::text[]),
  CONSTRAINT product_country_check CHECK (origin_country ~ '^[A-Z]{2}$'),
  CONSTRAINT product_weights_check CHECK (net_weight_g > 0 AND gross_weight_g >= net_weight_g),
  CONSTRAINT product_dims_check CHECK (length_mm > 0 AND width_mm > 0 AND height_mm > 0),
  CONSTRAINT product_prep_delay_check CHECK (prep_delay_max_days >= prep_delay_min_days)
);
CREATE INDEX idx_product_status ON product(status);
CREATE INDEX idx_product_category ON product(category_id) WHERE status = 'published';
CREATE INDEX idx_product_sales_modes ON product USING gin(sales_modes);

CREATE TABLE product_translation (
  product_id  uuid NOT NULL REFERENCES product(id) ON DELETE CASCADE,
  locale      char(3) NOT NULL,
  name        text NOT NULL,
  short_desc  text NOT NULL DEFAULT '',
  description text NOT NULL DEFAULT '',
  slug        text NOT NULL,
  seo_title   text,
  seo_desc    text,
  CONSTRAINT product_translation_pk PRIMARY KEY (product_id, locale),
  CONSTRAINT product_translation_locale_check CHECK (locale IN ('fr-FR','en','zh-CN')),
  CONSTRAINT product_translation_slug_unique UNIQUE (locale, slug),
  CONSTRAINT product_translation_name_check CHECK (length(btrim(name)) > 0)
);

CREATE TABLE product_variant (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id  uuid NOT NULL REFERENCES product(id) ON DELETE CASCADE,
  sku         text NOT NULL,
  attributes  jsonb NOT NULL DEFAULT '{}'::jsonb,
  barcode     text,
  weight_g    int,
  moq         int,
  order_multiple int,
  CONSTRAINT product_variant_sku_unique UNIQUE (sku),
  CONSTRAINT product_variant_attributes_check CHECK (jsonb_typeof(attributes) = 'object'),
  CONSTRAINT product_variant_moq_check CHECK (moq IS NULL OR moq > 0),
  CONSTRAINT product_variant_multiple_check CHECK (order_multiple IS NULL OR order_multiple > 0)
);
CREATE INDEX idx_product_variant_product ON product_variant(product_id);

CREATE TABLE product_media (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id  uuid NOT NULL REFERENCES product(id) ON DELETE CASCADE,
  variant_id  uuid REFERENCES product_variant(id) ON DELETE CASCADE,
  storage_key text NOT NULL,
  media_type  text NOT NULL,
  position    int NOT NULL DEFAULT 0,
  alt_text    text NOT NULL DEFAULT '',
  width       int,
  height      int,
  CONSTRAINT product_media_storage_key_unique UNIQUE (storage_key),
  CONSTRAINT product_media_type_check CHECK (media_type IN ('image','video')),
  CONSTRAINT product_media_position_check CHECK (position >= 0)
);
CREATE INDEX idx_product_media_product ON product_media(product_id, position);

CREATE TABLE product_compliance (
  product_id            uuid NOT NULL REFERENCES product(id) ON DELETE CASCADE,
  variant_id            uuid REFERENCES product_variant(id) ON DELETE CASCADE,
  hs_code_proposed      text,
  hs_code_validated     text,
  hs_version            text,
  hs_authority          text,
  origin_country        char(2) NOT NULL,
  restricted_countries  text[] NOT NULL DEFAULT '{}',
  required_documents    text[] NOT NULL DEFAULT '{}',
  notes                 text,
  reviewed_by            uuid,
  reviewed_at            timestamptz,
  CONSTRAINT product_compliance_pk PRIMARY KEY (product_id, variant_id),
  CONSTRAINT product_compliance_origin_check CHECK (origin_country ~ '^[A-Z]{2}$')
);
```

`packages/db/migrations/0007_pricing.sql` :

```sql
-- 0007_pricing.sql
-- Tarification : paliers de prix (plages non chevauchantes) et regles client.
SET lock_timeout = '3s';
SET statement_timeout = '30s';

CREATE TABLE price_tier (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id  uuid NOT NULL REFERENCES product(id) ON DELETE CASCADE,
  variant_id  uuid REFERENCES product_variant(id) ON DELETE CASCADE,
  channel     text NOT NULL,
  currency    char(3) NOT NULL,
  min_qty     int NOT NULL,
  max_qty     int,
  unit_price  numeric(18,4) NOT NULL,
  valid_from  timestamptz NOT NULL,
  valid_to    timestamptz,
  created_at  timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT price_tier_unique UNIQUE (product_id, variant_id, channel, currency, min_qty, valid_from),
  CONSTRAINT price_tier_channel_check CHECK (channel IN ('wholesale','retail')),
  CONSTRAINT price_tier_price_check CHECK (unit_price > 0),
  CONSTRAINT price_tier_min_check CHECK (min_qty > 0),
  CONSTRAINT price_tier_max_check CHECK (max_qty IS NULL OR max_qty >= min_qty),
  CONSTRAINT price_tier_validity_check CHECK (valid_to IS NULL OR valid_to > valid_from),
  CONSTRAINT price_tier_no_overlap EXCLUDE USING gist (
    product_id WITH =,
    COALESCE(variant_id, '00000000-0000-0000-0000-000000000000'::uuid) WITH =,
    channel WITH =,
    currency WITH =,
    int4range(min_qty, COALESCE(max_qty, 2147483647), '[]') WITH &&
  ) WHERE (valid_to IS NULL OR valid_to > now())
);
CREATE INDEX idx_price_tier_lookup ON price_tier(product_id, channel, currency, valid_from DESC);

CREATE TABLE customer_price_rule (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id  uuid NOT NULL,
  product_id   uuid REFERENCES product(id) ON DELETE CASCADE,
  variant_id   uuid REFERENCES product_variant(id) ON DELETE CASCADE,
  scope        text NOT NULL DEFAULT 'customer',
  min_qty      int NOT NULL DEFAULT 1,
  unit_price   numeric(18,4),
  discount_pct numeric(6,3),
  currency     char(3) NOT NULL,
  priority     int NOT NULL DEFAULT 3,
  approved_by  uuid NOT NULL,
  valid_from   timestamptz NOT NULL,
  valid_to     timestamptz,
  created_at   timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT customer_price_rule_unique UNIQUE (customer_id, product_id, variant_id, min_qty, valid_from),
  CONSTRAINT customer_price_rule_scope_check CHECK (scope IN ('customer','group')),
  CONSTRAINT customer_price_rule_price_check CHECK (unit_price IS NOT NULL OR discount_pct IS NOT NULL),
  CONSTRAINT customer_price_rule_discount_check CHECK (discount_pct IS NULL OR (discount_pct >= 0 AND discount_pct <= 100)),
  CONSTRAINT customer_price_rule_validity_check CHECK (valid_to IS NULL OR valid_to > valid_from),
  CONSTRAINT customer_price_rule_approved_check CHECK (approved_by IS NOT NULL)
);
CREATE INDEX idx_customer_price_rule_customer ON customer_price_rule(customer_id, valid_to);
CREATE INDEX idx_customer_price_rule_priority ON customer_price_rule(priority);
```

- [ ] **Step 3: Écrire les migrations stock (`0008`), panier (`0009`), commande (`0010`), paiement (`0011`) et notifications (`0012`)**

`packages/db/migrations/0008_stock.sql` :

```sql
-- 0008_stock.sql
-- Stock : entrepots, articles d'inventaire, mouvements, reservations.
SET lock_timeout = '3s';
SET statement_timeout = '30s';

CREATE TABLE warehouse (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  code       text NOT NULL,
  name       text NOT NULL,
  country    char(2) NOT NULL,
  timezone   text NOT NULL DEFAULT 'UTC',
  CONSTRAINT warehouse_code_unique UNIQUE (code),
  CONSTRAINT warehouse_country_check CHECK (country ~ '^[A-Z]{2}$')
);

CREATE TABLE inventory_item (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  warehouse_id       uuid NOT NULL REFERENCES warehouse(id),
  variant_id         uuid NOT NULL REFERENCES product_variant(id),
  on_hand            int NOT NULL DEFAULT 0,
  reserved           int NOT NULL DEFAULT 0,
  damaged            int NOT NULL DEFAULT 0,
  in_quality_control int NOT NULL DEFAULT 0,
  incoming           int NOT NULL DEFAULT 0,
  safety_stock       int NOT NULL DEFAULT 0,
  version            int NOT NULL DEFAULT 0,
  updated_at         timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT inventory_item_unique UNIQUE (warehouse_id, variant_id),
  CONSTRAINT inventory_item_on_hand_check CHECK (on_hand >= 0),
  CONSTRAINT inventory_item_reserved_check CHECK (reserved >= 0 AND reserved <= on_hand),
  CONSTRAINT inventory_item_incoming_check CHECK (incoming >= 0),
  CONSTRAINT inventory_item_damaged_check CHECK (damaged >= 0),
  CONSTRAINT inventory_item_qc_check CHECK (in_quality_control >= 0),
  CONSTRAINT inventory_item_safety_check CHECK (safety_stock >= 0)
);
CREATE INDEX idx_inventory_item_variant ON inventory_item(variant_id, warehouse_id);

CREATE TABLE inventory_movement (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  item_id     uuid NOT NULL REFERENCES inventory_item(id),
  kind        text NOT NULL,
  qty         int NOT NULL,
  reason      text NOT NULL,
  actor_id    uuid,
  order_id    uuid,
  occurred_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT inventory_movement_kind_check CHECK (kind IN ('in','out','reserve','release','adjust','transfer')),
  CONSTRAINT inventory_movement_qty_check CHECK (qty <> 0)
);
CREATE INDEX idx_inventory_movement_item ON inventory_movement(item_id, occurred_at DESC);
CREATE INDEX idx_inventory_movement_order ON inventory_movement(order_id);

CREATE TABLE stock_reservation (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  cart_id         uuid NOT NULL,
  user_id         uuid,
  variant_id      uuid NOT NULL REFERENCES product_variant(id),
  warehouse_id    uuid NOT NULL REFERENCES warehouse(id),
  qty             int NOT NULL,
  state           text NOT NULL DEFAULT 'held',
  expires_at      timestamptz NOT NULL,
  idempotency_key text NOT NULL,
  created_at      timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT stock_reservation_idempotency_unique UNIQUE (idempotency_key),
  CONSTRAINT stock_reservation_qty_check CHECK (qty > 0),
  CONSTRAINT stock_reservation_state_check CHECK (state IN ('held','converted','released','expired')),
  CONSTRAINT stock_reservation_expiry_check CHECK (expires_at > created_at)
);
CREATE INDEX idx_stock_reservation_expiry ON stock_reservation(expires_at) WHERE state = 'held';
CREATE INDEX idx_stock_reservation_variant ON stock_reservation(variant_id, state);
```

`packages/db/migrations/0009_cart.sql` :

```sql
-- 0009_cart.sql
-- Panier serveur : paniers et lignes de panier.
SET lock_timeout = '3s';
SET statement_timeout = '30s';

CREATE TABLE cart (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid,
  business_id uuid,
  currency    char(3) NOT NULL DEFAULT 'EUR',
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_cart_user ON cart(user_id);
CREATE INDEX idx_cart_business ON cart(business_id);

CREATE TABLE cart_item (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  cart_id         uuid NOT NULL REFERENCES cart(id) ON DELETE CASCADE,
  variant_id      uuid NOT NULL REFERENCES product_variant(id),
  qty             int NOT NULL,
  price_snapshot  jsonb NOT NULL DEFAULT '{}'::jsonb,
  idempotency_key text NOT NULL,
  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT cart_item_unique UNIQUE (cart_id, variant_id),
  CONSTRAINT cart_item_idempotency_unique UNIQUE (idempotency_key),
  CONSTRAINT cart_item_qty_check CHECK (qty > 0)
);
```

`packages/db/migrations/0010_order.sql` :

```sql
-- 0010_order.sql
-- Commandes : entete, lignes, evenements de statut et sequence de numerotation.
SET lock_timeout = '3s';
SET statement_timeout = '30s';

-- Sequence des numeros de commande (FO-AAAA-NNNNNN). Creee par migration SQL,
-- jamais par le code applicatif : elle garantit l'unicite du numero.
CREATE SEQUENCE IF NOT EXISTS order_number_seq;

CREATE TABLE orders (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  number         text NOT NULL,
  user_id        uuid NOT NULL,
  business_id    uuid,
  status         text NOT NULL DEFAULT 'draft',
  currency       char(3) NOT NULL,
  total_amount   numeric(18,4) NOT NULL,
  refunded_amount numeric(18,4) NOT NULL DEFAULT 0,
  idempotency_key text NOT NULL,
  incoterm       text NOT NULL,
  sales_mode     text NOT NULL,
  destination_country char(2) NOT NULL,
  snapshot       jsonb NOT NULL,
  placed_at      timestamptz,
  paid_at        timestamptz,
  created_at     timestamptz NOT NULL DEFAULT now(),
  updated_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT orders_number_unique UNIQUE (number),
  CONSTRAINT orders_idempotency_unique UNIQUE (idempotency_key),
  CONSTRAINT orders_total_check CHECK (total_amount >= 0),
  CONSTRAINT orders_refunded_check CHECK (refunded_amount >= 0 AND refunded_amount <= total_amount),
  CONSTRAINT orders_status_check CHECK (status IN (
    'draft','pending_payment','payment_processing','payment_failed','paid','cancelled',
    'cancellation_requested','awaiting_review','confirmed','processing','partially_fulfilled',
    'shipped','in_transit','customs_action_required','delivered','disputed','completed',
    'refund_pending','partially_refunded','refunded')),
  CONSTRAINT orders_incoterm_check CHECK (incoterm IN ('EXW','FOB','CIF','DAP','DDP')),
  CONSTRAINT orders_sales_mode_check CHECK (sales_mode IN ('wholesale','retail')),
  CONSTRAINT orders_country_check CHECK (destination_country ~ '^[A-Z]{2}$')
);
CREATE INDEX idx_orders_user ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_status_placed ON orders(status, placed_at);
CREATE INDEX idx_orders_business ON orders(business_id);
CREATE INDEX idx_orders_snapshot_aggregate ON orders((snapshot->>'aggregateId'));

CREATE TABLE order_line (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id      uuid NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  variant_id    uuid NOT NULL REFERENCES product_variant(id),
  sku           text NOT NULL,
  qty           int NOT NULL,
  unit_price    numeric(18,4) NOT NULL,
  price_tier_ref jsonb NOT NULL,
  tax_amount    numeric(18,4) NOT NULL DEFAULT 0,
  duty_amount   numeric(18,4) NOT NULL DEFAULT 0,
  line_total    numeric(18,4) NOT NULL,
  snapshot      jsonb NOT NULL DEFAULT '{}'::jsonb,
  CONSTRAINT order_line_qty_check CHECK (qty > 0),
  CONSTRAINT order_line_price_check CHECK (unit_price >= 0),
  CONSTRAINT order_line_total_check CHECK (line_total = qty*unit_price + tax_amount + duty_amount)
);
CREATE INDEX idx_order_line_order ON order_line(order_id);
CREATE INDEX idx_order_line_variant ON order_line(variant_id);
CREATE INDEX idx_order_line_sku ON order_line(sku);

CREATE TABLE order_status_event (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id      uuid NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  from_status   text NOT NULL,
  to_status     text NOT NULL,
  actor_type    text NOT NULL,
  actor_id      uuid,
  reason        text,
  correlation_id uuid,
  occurred_at   timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT order_status_event_actor_check CHECK (actor_type IN ('system','customer','admin','provider')),
  CONSTRAINT order_status_event_change_check CHECK (from_status <> to_status)
);
CREATE INDEX idx_order_status_event_order ON order_status_event(order_id, occurred_at);
```

`packages/db/migrations/0011_payment.sql` :

```sql
-- 0011_payment.sql
-- Paiement : paiements, evenements, remboursements et factures.
SET lock_timeout = '3s';
SET statement_timeout = '30s';

CREATE TABLE payment (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id           uuid NOT NULL REFERENCES orders(id),
  provider           text NOT NULL,
  provider_payment_id text,
  method             text NOT NULL,
  currency           char(3) NOT NULL,
  amount             numeric(18,4) NOT NULL,
  status             text NOT NULL DEFAULT 'pending',
  idempotency_key    text NOT NULL,
  paid_at            timestamptz,
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT payment_provider_ref_unique UNIQUE (provider, provider_payment_id),
  CONSTRAINT payment_idempotency_unique UNIQUE (idempotency_key),
  CONSTRAINT payment_amount_check CHECK (amount > 0),
  CONSTRAINT payment_status_check CHECK (status IN ('pending','processing','succeeded','failed','expired','partially_refunded','refunded','disputed'))
);
CREATE INDEX idx_payment_order ON payment(order_id);
CREATE INDEX idx_payment_status ON payment(status, created_at);

CREATE TABLE payment_event (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  payment_id        uuid REFERENCES payment(id),
  provider          text NOT NULL,
  provider_event_id text NOT NULL,
  type              text NOT NULL,
  raw_payload       jsonb NOT NULL,
  signature_valid   boolean NOT NULL,
  received_at       timestamptz NOT NULL DEFAULT now(),
  processed_at      timestamptz,
  processing_result text,
  CONSTRAINT payment_event_unique UNIQUE (provider, provider_event_id)
);
CREATE INDEX idx_payment_event_payment ON payment_event(payment_id, received_at);

CREATE TABLE refund (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  payment_id        uuid NOT NULL REFERENCES payment(id),
  provider_refund_id text,
  amount            numeric(18,4) NOT NULL,
  currency          char(3) NOT NULL,
  reason            text NOT NULL,
  status            text NOT NULL DEFAULT 'pending',
  requested_by      uuid NOT NULL,
  approved_by       uuid,
  idempotency_key   text NOT NULL,
  created_at        timestamptz NOT NULL DEFAULT now(),
  updated_at        timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT refund_provider_ref_unique UNIQUE (provider_refund_id),
  CONSTRAINT refund_idempotency_unique UNIQUE (idempotency_key),
  CONSTRAINT refund_amount_check CHECK (amount > 0),
  CONSTRAINT refund_status_check CHECK (status IN ('pending','succeeded','failed','cancelled'))
);
CREATE INDEX idx_refund_payment ON refund(payment_id);

CREATE TABLE invoice (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  number           text NOT NULL,
  order_id         uuid NOT NULL REFERENCES orders(id),
  type             text NOT NULL,
  currency         char(3) NOT NULL,
  total            numeric(18,4) NOT NULL,
  issued_at        timestamptz NOT NULL DEFAULT now(),
  pdf_storage_key  text,
  locale           char(3) NOT NULL,
  template_version text NOT NULL,
  seq              int NOT NULL,
  CONSTRAINT invoice_number_unique UNIQUE (number),
  CONSTRAINT invoice_order_type_unique UNIQUE (order_id, type),
  CONSTRAINT invoice_type_check CHECK (type IN ('commercial_export','fiscal','receipt','credit_note')),
  CONSTRAINT invoice_total_check CHECK (total >= 0),
  CONSTRAINT invoice_locale_check CHECK (locale IN ('fr-FR','en','zh-CN'))
);
CREATE INDEX idx_invoice_issued ON invoice(issued_at);
```

`packages/db/migrations/0012_notification_and_outbox.sql` :

```sql
-- 0012_notification_and_outbox.sql
-- Notifications, e-mails transactionnels, webhooks, extension de l'outbox et
-- de l'audit (tables creees par la Phase 1, jamais recreees ici).
SET lock_timeout = '3s';
SET statement_timeout = '30s';

CREATE TABLE notification (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    uuid NOT NULL,
  type       text NOT NULL,
  channel    text NOT NULL DEFAULT 'in_app',
  payload    jsonb NOT NULL,
  read_at    timestamptz,
  locale     char(3) NOT NULL DEFAULT 'fr-FR',
  created_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT notification_locale_check CHECK (locale IN ('fr-FR','en','zh-CN'))
);
CREATE INDEX idx_notification_user ON notification(user_id, read_at, created_at DESC);
CREATE INDEX idx_notification_type ON notification(type);

CREATE TABLE notification_preference (
  user_id    uuid NOT NULL,
  type       text NOT NULL,
  channel    text NOT NULL,
  enabled    boolean NOT NULL DEFAULT true,
  CONSTRAINT notification_preference_pk PRIMARY KEY (user_id, type, channel)
);

CREATE TABLE email_template (
  template_key text NOT NULL,
  locale       char(3) NOT NULL,
  version      int NOT NULL,
  subject      text NOT NULL,
  body_html    text NOT NULL,
  CONSTRAINT email_template_pk PRIMARY KEY (template_key, locale, version),
  CONSTRAINT email_template_locale_check CHECK (locale IN ('fr-FR','en','zh-CN'))
);

CREATE TABLE email_delivery (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  template_key      text NOT NULL,
  to_email          text NOT NULL,
  template_version  int NOT NULL,
  locale            char(3) NOT NULL,
  idempotency_key   text NOT NULL,
  status            text NOT NULL DEFAULT 'pending',
  provider_message_id text,
  sent_at           timestamptz,
  created_at        timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT email_delivery_idempotency_unique UNIQUE (idempotency_key),
  CONSTRAINT email_delivery_status_check CHECK (status IN ('pending','sent','delivered','bounced','failed')),
  CONSTRAINT email_delivery_locale_check CHECK (locale IN ('fr-FR','en','zh-CN'))
);
CREATE INDEX idx_email_delivery_status ON email_delivery(status, created_at);

-- outbox_event est creee par la Phase 1 (0004_outbox.sql) : identifiant,
-- type d'evenement, agregat, charge utile, cle d'idempotence et etat de
-- publication. La Phase 2 ne la recree pas : elle ajoute les colonnes de
-- l'evenement versionne (event_id, event_version, correlation_id) utilisees
-- par l'append outbox et le relay.
ALTER TABLE outbox_event ADD COLUMN IF NOT EXISTS event_id uuid DEFAULT gen_random_uuid();
ALTER TABLE outbox_event ADD COLUMN IF NOT EXISTS event_version int NOT NULL DEFAULT 1;
ALTER TABLE outbox_event ADD COLUMN IF NOT EXISTS correlation_id uuid DEFAULT gen_random_uuid();
ALTER TABLE outbox_event ALTER COLUMN event_id SET NOT NULL;
ALTER TABLE outbox_event ALTER COLUMN correlation_id SET NOT NULL;
CREATE UNIQUE INDEX IF NOT EXISTS outbox_event_event_id_unique ON outbox_event (event_id);
CREATE INDEX IF NOT EXISTS idx_outbox_event_state_available ON outbox_event(state, available_at) WHERE state IN ('pending','failed');
-- aggregate_id reste une colonne texte (Phase 1) : la Phase 2 y ecrit des
-- identifiants UUID sous forme de chaine, l'index de la Phase 1 sur
-- payload->>'aggregate_id' reste valable.

CREATE TABLE webhook_event (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  provider          text NOT NULL,
  provider_event_id text NOT NULL,
  type              text NOT NULL,
  signature_valid   boolean NOT NULL,
  payload           jsonb NOT NULL,
  status            text NOT NULL DEFAULT 'received',
  attempts          int NOT NULL DEFAULT 0,
  last_error        text,
  received_at       timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT webhook_event_unique UNIQUE (provider, provider_event_id),
  CONSTRAINT webhook_event_status_check CHECK (status IN ('received','processed','rejected','failed'))
);
CREATE INDEX idx_webhook_event_status ON webhook_event(status, created_at);

-- consumed_event est creee par la Phase 1 (0004_outbox.sql) avec la meme
-- cle primaire (event_id, consumer) : la Phase 2 ne la recree pas.

-- audit_log est creee par la Phase 1 (0003_rbac_and_audit.sql) : identifiant
-- bigserial, chainage de hash append-only, colonnes de reference obligatoires
-- (actor_role, object_type, object_id, correlation_id) et resultat controle.
-- La Phase 2 ajoute les colonnes de son propre modele (acteur, ressource,
-- avant/apres, IP, user agent) sans affaiblir les invariants de la Phase 1 :
-- une ecriture Phase 2 renseigne les colonnes de reference ET les colonnes
-- ajoutees. L'append-only reste garanti par le trigger de la Phase 1.
ALTER TABLE audit_log ADD COLUMN IF NOT EXISTS actor_type text;
ALTER TABLE audit_log ADD COLUMN IF NOT EXISTS resource_type text;
ALTER TABLE audit_log ADD COLUMN IF NOT EXISTS resource_id text;
ALTER TABLE audit_log ADD COLUMN IF NOT EXISTS before jsonb;
ALTER TABLE audit_log ADD COLUMN IF NOT EXISTS after jsonb;
ALTER TABLE audit_log ADD COLUMN IF NOT EXISTS ip inet;
ALTER TABLE audit_log ADD COLUMN IF NOT EXISTS user_agent text;
CREATE INDEX IF NOT EXISTS ix_audit_log_resource ON audit_log(resource_type, resource_id, occurred_at DESC);
CREATE INDEX IF NOT EXISTS ix_audit_log_action ON audit_log(action, occurred_at DESC);

CREATE TABLE feature_flag (
  key         text PRIMARY KEY,
  enabled     boolean NOT NULL DEFAULT false,
  rules       jsonb NOT NULL DEFAULT '{}'::jsonb,
  rollout_pct int NOT NULL DEFAULT 0,
  CONSTRAINT feature_flag_rollout_check CHECK (rollout_pct BETWEEN 0 AND 100)
);

CREATE TABLE country_profile (
  code                    char(2) PRIMARY KEY,
  name                    text NOT NULL,
  delivery_enabled        boolean NOT NULL DEFAULT false,
  incoterms               text[] NOT NULL DEFAULT '{}',
  currencies              text[] NOT NULL DEFAULT '{}',
  payment_methods         text[] NOT NULL DEFAULT '{}',
  tax_registration_required boolean NOT NULL DEFAULT false,
  required_docs           text[] NOT NULL DEFAULT '{}',
  restricted_categories   text[] NOT NULL DEFAULT '{}',
  operational_status      text NOT NULL DEFAULT 'draft',
  CONSTRAINT country_profile_code_check CHECK (code ~ '^[A-Z]{2}$'),
  CONSTRAINT country_profile_status_check CHECK (operational_status IN ('draft','active','suspended'))
);

CREATE TABLE tax_estimate (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id     uuid REFERENCES orders(id),
  cart_id      uuid,
  country_code char(2) NOT NULL,
  currency     char(3) NOT NULL,
  amount       numeric(18,4),
  low          numeric(18,4),
  high         numeric(18,4),
  certainty    text NOT NULL,
  rule_refs    jsonb NOT NULL DEFAULT '[]'::jsonb,
  computed_at  timestamptz NOT NULL DEFAULT now(),
  text_shown   text NOT NULL,
  CONSTRAINT tax_estimate_certainty_check CHECK (certainty IN ('validated','estimated','incomplete','specialist_review')),
  CONSTRAINT tax_estimate_range_check CHECK (low IS NULL OR high IS NULL OR low <= high)
);
CREATE INDEX idx_tax_estimate_order ON tax_estimate(order_id);
CREATE INDEX idx_tax_estimate_country ON tax_estimate(country_code, computed_at DESC);
```

- [ ] **Step 4: Réutiliser le runner de la Phase 1 via un adaptateur (append-only, checksum, idempotent)**

`packages/db/src/migrate.ts` — adaptateur du runner de la Phase 1 (`packages/db/src/migrate-core.ts`, cœur du runner ; la CLI reste `packages/db/scripts/migrate.ts`). La Phase 2 n'écrit pas un second runner : elle délègue au runner qui possède déjà la convention de versionnement (préfixe numérique `NNNN`) et la vérification de checksum. Les migrations `0001` à `0005` de la Phase 1 restent ainsi les mêmes lignes de `schema_migrations` et ne sont jamais rejouées.

```typescript
import { dirname, join } from 'node:path';
import { fileURLToPath } from 'node:url';
import type { Pool } from 'pg';
import { migrate, type AppliedMigration } from './migrate-core.js';

const MIGRATIONS_DIR = join(dirname(fileURLToPath(import.meta.url)), '..', 'migrations');

export interface MigrationOutcome {
  readonly applied: string[];
  readonly skipped: string[];
}

export async function runMigrations(pool: Pool): Promise<MigrationOutcome> {
  const results: AppliedMigration[] = await migrate({
    directory: MIGRATIONS_DIR,
    client: pool,
  });
  return {
    applied: results.filter((result) => result.status === 'applied').map((result) => result.version),
    skipped: results.filter((result) => result.status === 'skipped').map((result) => result.version),
  };
}
```

Le point d'export `@fanours/db/migrate` pointe sur ce module : `apps/api`, `packages/jobs` et `packages/pricing` appellent `runMigrations(pool)` et reçoivent les versions appliquées et ignorées, la version étant le préfixe numérique du fichier (`0006`, `0007`, …).

`packages/db/src/client.ts` (modifié — le fichier de la Phase 1 est conservé ; seule la source du schéma change, pour inclure les tables métier de la Phase 2) :

```typescript
import { createHash, randomBytes } from 'node:crypto';
import { drizzle, type NodePgDatabase } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema/index.js';

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

`packages/db/src/schema.ts` (modifié — les tables `outbox_event` et `audit_log` restent définies par la Phase 1 ; la Phase 2 ajoute les colonnes de sa migration `0012`, aucune table n'est recréée. Extrait des deux tables concernées, le reste du fichier est inchangé) :

```typescript
export const auditLog = pgTable(
  'audit_log',
  {
    id: bigint('id', { mode: 'number' }).primaryKey().generatedAlwaysAsIdentity(),
    occurredAt: timestamp('occurred_at', { withTimezone: true }).notNull().defaultNow(),
    actorId: uuid('actor_id'),
    actorRole: text('actor_role'),
    actorType: text('actor_type'),            // ajoute par 0012
    action: text('action').notNull(),
    objectType: text('object_type').notNull(),
    objectId: text('object_id').notNull(),
    resourceType: text('resource_type'),      // ajoute par 0012
    resourceId: text('resource_id'),          // ajoute par 0012
    before: jsonb('before'),                  // ajoute par 0012
    after: jsonb('after'),                    // ajoute par 0012
    result: text('result').notNull(),
    correlationId: text('correlation_id').notNull(),
    ipHash: text('ip_hash'),
    ip: inet('ip'),                           // ajoute par 0012
    userAgent: text('user_agent'),            // ajoute par 0012
    metadata: jsonb('metadata').notNull().default({}),
    prevHash: text('prev_hash'),
    entryHash: text('entry_hash').notNull(),
  },
  (table) => ({
    entryHashUnique: uniqueIndex('uq_audit_log_entry_hash').on(table.entryHash),
    occurredAtIndex: index('ix_audit_log_occurred_at').on(table.occurredAt),
    resourceIndex: index('ix_audit_log_resource').on(table.resourceType, table.resourceId, table.occurredAt),
  }),
);

export const outboxEvent = pgTable(
  'outbox_event',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    eventId: uuid('event_id').notNull(),                        // ajoute par 0012
    eventType: text('event_type').notNull(),
    eventVersion: integer('event_version').notNull().default(1), // ajoute par 0012
    aggregateType: text('aggregate_type').notNull(),
    aggregateId: text('aggregate_id').notNull(),
    payload: jsonb('payload').notNull().default({}),
    correlationId: uuid('correlation_id').notNull(),             // ajoute par 0012
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
    eventIdUnique: uniqueIndex('uq_outbox_event_event_id').on(table.eventId),
  }),
);
```

`packages/db/src/schema/index.ts` (créé — baril des schémas : c'est la source unique importée par `src/client.ts` et exposée par le point d'export `@fanours/db/schema`. Il est étendu au fur et à mesure que les tasks métier créent leur module) :

```typescript
// Baril des schemas Drizzle du paquet.
// - Les schemas de la Phase 1 (identite, RBAC, audit, outbox) viennent de ../schema.js.
// - Les modules metier de la Phase 2 sont ajoutes par la task qui porte le domaine :
//   Task 4 (catalog), Task 5 (pricing), Task 6 (stock), Task 11 (cart),
//   Task 13 (order), Task 14 (payment), Task 18 (notification).
export * from '../schema.js';
```

État final du baril à la fin de la Phase 2 :

```typescript
export * from '../schema.js';
export * from './catalog.js';
export * from './pricing.js';
export * from './stock.js';
export * from './cart.js';
export * from './order.js';
export * from './payment.js';
export * from './notification.js';
```

Les modules `catalog.ts`, `pricing.ts`, `stock.ts`, `cart.ts`, `order.ts`, `payment.ts` et `notification.ts` déclarent les tables Drizzle des migrations `0006` à `0012` (le SQL est la source de vérité, le module Drizzle en est le miroir TypeScript) ; `outbox_event`, `consumed_event` et `audit_log` ne sont pas redéclarés ici, ils le sont par la Phase 1 dans `src/schema.ts` et sont étendus par `0012_notification_and_outbox.sql`.

`packages/db/src/transaction.ts` :

```typescript
import type { Pool, PoolClient } from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import type { NodePgDatabase } from 'drizzle-orm/node-postgres';
import * as schema from './schema/index.js';

export type Tx = NodePgDatabase<typeof schema>;

export async function withTransaction<T>(pool: Pool, fn: (tx: Tx) => Promise<T>): Promise<T> {
  const client: PoolClient = await pool.connect();
  try {
    await client.query('BEGIN');
    const tx = drizzle(client, { schema });
    const result = await fn(tx);
    await client.query('COMMIT');
    return result;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

`packages/db/src/index.ts` (modifié — le baril écrit par la Phase 1 est étendu : il expose le schéma combiné et les modules de la Phase 2) :

```typescript
// Baril du paquet : ecrit par la Phase 1, etendu par la Phase 2.
// Les schemas de la Phase 1 (identite, RBAC, audit, outbox) et les schemas
// metier de la Phase 2 passent par le seul point d'entree ./schema/index.js.
// Le module d'ecriture d'outbox (./outbox.js) est ajoute par la Task 3,
// qui le cree : le baril de la Task 2 ne l'exporte pas encore.
export * from './schema/index.js';
export * from './client.js';
export { withTransaction, type Tx } from './transaction.js';
```

`withTransaction` est explicitement ré-exporté depuis `./transaction.js` : cet export explicite prend le pas sur le `export *` de `./client.js` (helper de la Phase 1, signature `(database, callback)`, toujours accessible via `@fanours/db/client`). Un seul `withTransaction` est donc publié par le paquet, avec la signature sur pool consommée par la Phase 2.

- [ ] **Step 5: Écrire le test d'intégration qui prouve les invariants en base**

`packages/db/test/schema-invariants.test.ts` (exécuté sur un PostgreSQL 16 jetable via Testcontainers) :

```typescript
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import pg from 'pg';
import { runMigrations } from '../src/migrate.js';

let container: StartedPostgreSqlContainer;
let pool: pg.Pool;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('fanours')
    .withUsername('fanours')
    .withPassword('fanours')
    .start();
  pool = new pg.Pool({ connectionString: container.getConnectionUri() });
  const outcome = await runMigrations(pool);
  // 5 migrations de la Phase 1 (0001-0005) + 10 migrations de la Phase 2 (0006-0015).
  expect(outcome.applied.length).toBeGreaterThanOrEqual(15);
}, 120_000);

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

async function seedBase(): Promise<{ productId: string; variantId: string }> {
  const product = await pool.query<{ id: string }>(
    `INSERT INTO product (sku, status, sales_modes, moq, order_multiple, origin_country, net_weight_g, gross_weight_g, length_mm, width_mm, height_mm)
     VALUES ('SKU-TEST', 'published', ARRAY['wholesale'], 6, 6, 'CN', 100, 200, 10, 10, 10) RETURNING id`,
  );
  const variant = await pool.query<{ id: string }>(
    `INSERT INTO product_variant (product_id, sku) VALUES ($1, 'SKU-TEST-V1') RETURNING id`,
    [product.rows[0]?.id],
  );
  return { productId: product.rows[0]!.id, variantId: variant.rows[0]!.id };
}

describe('invariants de schéma', () => {
  it('refuse deux paliers de prix qui se chevauchent', async () => {
    const { productId } = await seedBase();
    await pool.query(
      `INSERT INTO price_tier (product_id, channel, currency, min_qty, max_qty, unit_price, valid_from)
       VALUES ($1, 'wholesale', 'EUR', 1, 9, 15.00, now())`,
      [productId],
    );
    await expect(
      pool.query(
        `INSERT INTO price_tier (product_id, channel, currency, min_qty, max_qty, unit_price, valid_from)
         VALUES ($1, 'wholesale', 'EUR', 5, 49, 12.00, now())`,
        [productId],
      ),
    ).rejects.toThrow(/price_tier_no_overlap|conflicting key/i);
  });

  it('refuse une réservation supérieure au stock disponible', async () => {
    const { variantId } = await seedBase();
    const warehouse = await pool.query<{ id: string }>(
      `INSERT INTO warehouse (code, name, country) VALUES ('WH-TEST', 'Test', 'CN') RETURNING id`,
    );
    await pool.query(
      `INSERT INTO inventory_item (warehouse_id, variant_id, on_hand, reserved) VALUES ($1, $2, 5, 0)`,
      [warehouse.rows[0]!.id, variantId],
    );
    await expect(
      pool.query(
        `INSERT INTO inventory_item (warehouse_id, variant_id, on_hand, reserved) VALUES ($1, $2, 5, 6)`,
        [warehouse.rows[0]!.id, variantId],
      ),
    ).rejects.toThrow(/inventory_item_reserved_check/i);
  });

  it('refuse une ligne de commande dont le total ne correspond pas', async () => {
    const { variantId } = await seedBase();
    const order = await pool.query<{ id: string }>(
      `INSERT INTO orders (number, user_id, currency, total_amount, idempotency_key, incoterm, sales_mode, destination_country, snapshot)
       VALUES ('ORD-1', gen_random_uuid(), 'EUR', 100, 'idem-1', 'DAP', 'wholesale', 'FR', '{}'::jsonb) RETURNING id`,
    );
    await expect(
      pool.query(
        `INSERT INTO order_line (order_id, variant_id, sku, qty, unit_price, line_total)
         VALUES ($1, $2, 'SKU-TEST-V1', 10, 10.00, 999.00)`,
        [order.rows[0]!.id, variantId],
      ),
    ).rejects.toThrow(/order_line_total_check/i);
  });

  it('refuse la modification d’une ligne d’audit', async () => {
    // Les colonnes de reference de la Phase 1 (actor_role, object_type,
    // object_id, correlation_id) restent obligatoires ; les colonnes ajoutees
    // par 0012 (actor_type, resource_type, resource_id) sont renseignees aussi.
    const inserted = await pool.query<{ id: string }>(
      `INSERT INTO audit_log (actor_role, actor_type, action, object_type, object_id, resource_type, resource_id, result, correlation_id)
       VALUES ('system', 'system', 'test', 'product', 'audit-test', 'product', 'audit-test', 'success', gen_random_uuid()::text)
       RETURNING id`,
    );
    await expect(
      pool.query(`UPDATE audit_log SET action = 'tampered' WHERE id = $1`, [inserted.rows[0]!.id]),
    ).rejects.toThrow(/append-only/i);
  });

  it('applique les migrations une seule fois', async () => {
    const outcome = await runMigrations(pool);
    expect(outcome.applied).toHaveLength(0);
    expect(outcome.skipped.length).toBeGreaterThanOrEqual(15);
  });
});
```

`packages/db/test/migrate.test.ts` (exécuté sur un PostgreSQL 16 jetable via Testcontainers — prouve que l'adaptateur de la Phase 2 applique les migrations des deux phases et ne rejoue jamais celles de la Phase 1) :

```typescript
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import pg from 'pg';
import { runMigrations } from '../src/migrate.js';

let container: StartedPostgreSqlContainer;
let pool: pg.Pool;

beforeAll(async () => {
  container = new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('fanours')
    .withUsername('fanours')
    .withPassword('fanours')
    .start();
  pool = new pg.Pool({ connectionString: container.getConnectionUri() });
}, 120_000);

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

describe('runner de migrations', () => {
  it('applique les migrations des phases 1 et 2 puis ne rejoue plus rien', async () => {
    const first = await runMigrations(pool);
    // 5 migrations de la Phase 1 (0001-0005) + 10 migrations de la Phase 2 (0006-0015).
    expect(first.applied).toHaveLength(15);
    expect(first.applied).toContain('0001');
    expect(first.applied).toContain('0015');

    const second = await runMigrations(pool);
    expect(second.applied).toHaveLength(0);
    expect(second.skipped).toHaveLength(15);
  });
});
```

- [ ] **Step 6: Exécuter les tests d'intégration de base**

```bash
pnpm --filter @fanours/db test
```

Attendu : `Test Files  2 passed (2)` et `Tests  6 passed (6)`. Si Docker n'est pas disponible sur la machine, l'étape est **annoncée comme non exécutable** dans la PR avec le prérequis exact (`docker` accessible) — elle n'est jamais déclarée réussie sans sortie.

- [ ] **Step 7: Commit**

```bash
git add packages/db tsconfig.base.json
git commit -m "feat(db): etend le schema Drizzle avec les migrations 0006 a 0015

Ajoute au paquet cree par la Phase 1 les tables catalogue, tarification, stock,
panier, commande, paiement et notification, et etend les tables outbox_event
et audit_log sans les recreer. Le runner de la Phase 1 est reutilise via
src/migrate.ts. Les invariants critiques sont garantis en base : paliers non
chevauchants (EXCLUDE USING gist), stock reserve non depasse, total de ligne
coherent et audit_log append-only.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

### Task 3: Outbox transactionnelle, relay et consommateurs idempotents

**Files:**
- Create: `packages/db/src/outbox.ts`, `packages/jobs/package.json`, `packages/jobs/src/idempotency-key.ts`, `packages/jobs/src/backoff.ts`, `packages/jobs/src/define-job.ts`, `packages/jobs/src/index.ts`
- Create: `packages/jobs/src/idempotency-key.test.ts`, `packages/jobs/src/backoff.test.ts`, `packages/jobs/src/outbox.test.ts`
- Modify: `packages/db/src/index.ts`
- Modify: `packages/db/package.json` (point d'export `./outbox` ajouté)
- Modify: `tsconfig.base.json` (chemin `@fanours/db/outbox` ajouté)

**Interfaces:**
- Consumes: `packages/db` (`db`, `withTransaction`), table `outbox_event`, table `consumed_event`.
- Produces:
  - `packages/db` → `appendOutboxEvent(tx, event: OutboxEventInput): Promise<{ eventId: string }>`, `OutboxEventInput`
  - `packages/jobs` → `idempotencyKey(type: string, parts: readonly string[]): string`, `backoffMs(attempt: number, base?: number, cap?: number): number`, `defineJob<TPayload>(definition: JobDefinition<TPayload>): JobDefinition<TPayload>`, `JobDefinition<TPayload>`, `consumeOnce(eventId, consumer, fn): Promise<'processed' | 'skipped'>`

- [ ] **Step 1: Écrire le test qui échoue pour l'idempotence des consommateurs et le backoff**

`packages/jobs/src/idempotency-key.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { idempotencyKey } from './idempotency-key.js';

describe('idempotencyKey', () => {
  it('est stable pour les mêmes parties', () => {
    expect(idempotencyKey('invoice.generate', ['order-1', '1'])).toBe(
      idempotencyKey('invoice.generate', ['order-1', '1']),
    );
  });

  it('distingue deux entités différentes', () => {
    expect(idempotencyKey('invoice.generate', ['order-1', '1'])).not.toBe(
      idempotencyKey('invoice.generate', ['order-2', '1']),
    );
  });

  it('inclut le type de job pour éviter toute collision inter-queues', () => {
    expect(idempotencyKey('a', ['x'])).not.toBe(idempotencyKey('b', ['x']));
  });
});
```

`packages/jobs/src/backoff.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { backoffMs } from './backoff.js';

describe('backoffMs', () => {
  it('croît avec le numéro de tentative', () => {
    expect(backoffMs(1)).toBeLessThan(backoffMs(2));
    expect(backoffMs(2)).toBeLessThan(backoffMs(3));
  });

  it('respecte le plafond', () => {
    expect(backoffMs(50, 1_000, 15 * 60_000)).toBeLessThanOrEqual(15 * 60_000);
  });

  it('applique un jitter complet : le résultat reste dans [0, base * 2^(attempt-1)]', () => {
    for (let attempt = 1; attempt <= 6; attempt += 1) {
      const ceiling = 1_000 * 2 ** (attempt - 1);
      for (let i = 0; i < 50; i += 1) {
        const value = backoffMs(attempt);
        expect(value).toBeGreaterThanOrEqual(0);
        expect(value).toBeLessThanOrEqual(ceiling);
      }
    }
  });
});
```

- [ ] **Step 2: Implémenter `packages/jobs`**

`packages/jobs/src/idempotency-key.ts` :

```typescript
import { createHash } from 'node:crypto';

/** Clé d'idempotence déterministe : même type + mêmes parties ⇒ même clé. */
export function idempotencyKey(type: string, parts: readonly string[]): string {
  const digest = createHash('sha256')
    .update([type, ...parts].join('|'), 'utf8')
    .digest('hex');
  return `${type}:${digest.slice(0, 32)}`;
}
```

`packages/jobs/src/backoff.ts` :

```typescript
/**
 * Backoff exponentiel à jitter complet (infra.md §11.2).
 * La valeur retournée est dans [0, base * 2^(attempt-1)] plafonné à `cap`.
 */
export function backoffMs(attempt: number, base = 1_000, cap = 15 * 60_000): number {
  if (attempt < 1) throw new Error(`Tentative invalide: ${attempt}`);
  const ceiling = Math.min(cap, base * 2 ** (attempt - 1));
  return Math.floor(Math.random() * (ceiling + 1));
}
```

`packages/jobs/src/define-job.ts` :

```typescript
export interface JobContext<TPayload> {
  readonly payload: TPayload;
  readonly attempt: number;
  readonly correlationId: string;
  readonly log: (message: string, meta?: Record<string, unknown>) => void;
}

export interface JobDefinition<TPayload> {
  readonly type: string;
  readonly queue: string;
  readonly maxAttempts: number;
  readonly timeoutMs: number;
  readonly idempotency: (payload: TPayload) => string;
  readonly handler: (ctx: JobContext<TPayload>) => Promise<void>;
}

export function defineJob<TPayload>(definition: JobDefinition<TPayload>): JobDefinition<TPayload> {
  return Object.freeze(definition);
}
```

`packages/jobs/src/index.ts` :

```typescript
export * from './idempotency-key.js';
export * from './backoff.js';
export * from './define-job.js';
```

- [ ] **Step 3: Écrire le test d'intégration de l'outbox et le consommateur idempotent**

`packages/jobs/src/outbox.test.ts` (Testcontainers) :

```typescript
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import pg from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import { runMigrations } from '@fanours/db/migrate';
import * as schema from '@fanours/db/schema';

let container: StartedPostgreSqlContainer;
let pool: pg.Pool;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('fanours')
    .withUsername('fanours')
    .withPassword('fanours')
    .start();
  pool = new pg.Pool({ connectionString: container.getConnectionUri() });
  await runMigrations(pool);
}, 120_000);

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

describe('outbox transactionnelle', () => {
  it('écrit l’événement dans la même transaction que l’écriture métier', async () => {
    const client = await pool.connect();
    try {
      await client.query('BEGIN');
      await client.query(
        `INSERT INTO orders (number, user_id, currency, total_amount, idempotency_key, incoterm, sales_mode, destination_country, snapshot)
         VALUES ('ORD-OUTBOX', gen_random_uuid(), 'EUR', 10, 'idem-outbox', 'DAP', 'retail', 'FR', '{}'::jsonb)`,
      );
      await client.query(
        `INSERT INTO outbox_event (event_id, event_type, aggregate_type, aggregate_id, payload, correlation_id, idempotency_key)
         VALUES (gen_random_uuid(), 'order.created', 'order', (SELECT id FROM orders WHERE number = 'ORD-OUTBOX'), '{"number":"ORD-OUTBOX"}'::jsonb, gen_random_uuid(), 'order.created:ORD-OUTBOX')`,
      );
      await client.query('ROLLBACK');
    } finally {
      client.release();
    }
    const check = await pool.query(`SELECT count(*)::int AS n FROM outbox_event WHERE idempotency_key = 'order.created:ORD-OUTBOX'`);
    expect(check.rows[0]?.n).toBe(0);
  });

  it('refuse deux événements de même clé d’idempotence', async () => {
    for (let i = 0; i < 2; i += 1) {
      await pool.query(
        `INSERT INTO outbox_event (event_id, event_type, aggregate_type, aggregate_id, payload, correlation_id, idempotency_key)
         VALUES (gen_random_uuid(), 'order.created', 'order', gen_random_uuid(), '{}'::jsonb, gen_random_uuid(), 'idem-dup')`,
      ).catch((error: Error) => {
        if (i === 0) return;
        throw error;
      });
    }
    const check = await pool.query(`SELECT count(*)::int AS n FROM outbox_event WHERE idempotency_key = 'idem-dup'`);
    expect(check.rows[0]?.n).toBe(1);
  });
});
```

- [ ] **Step 4: Implémenter l'append outbox et le consommateur idempotent**

`packages/db/src/outbox.ts` :

```typescript
import { randomUUID } from 'node:crypto';
import type { Tx } from './transaction.js';

export interface OutboxEventInput {
  readonly eventType: string;
  readonly eventVersion?: number;
  readonly aggregateType: 'order' | 'payment' | 'shipment' | 'quote' | 'catalog';
  readonly aggregateId: string;
  readonly payload: Record<string, unknown>;
  readonly correlationId: string;
  readonly idempotencyKey: string;
}

/** Écrit l'événement dans la transaction métier courante. Aucun appel réseau ici. */
export async function appendOutboxEvent(
  tx: Tx,
  event: OutboxEventInput,
): Promise<{ eventId: string }> {
  const eventId = randomUUID();
  await tx.execute(
    `INSERT INTO outbox_event (event_id, event_type, event_version, aggregate_type, aggregate_id, payload, correlation_id, idempotency_key)
     VALUES (${eventId}, ${event.eventType}, ${event.eventVersion ?? 1}, ${event.aggregateType}, ${event.aggregateId}, ${JSON.stringify(event.payload)}::jsonb, ${event.correlationId}, ${event.idempotencyKey})
     ON CONFLICT (idempotency_key) DO NOTHING`,
  );
  return { eventId };
}
```

`packages/db/src/index.ts` (modifié — le baril écrit par la Phase 1 et étendu par la Task 2 expose désormais le module d'outbox) :

```typescript
export * from './schema/index.js';
export * from './client.js';
export { withTransaction, type Tx } from './transaction.js';
export * from './outbox.js';
```

`packages/db/package.json` (modifié — point d'export du module d'outbox ajouté à ceux de la Task 2) :

```json
"exports": {
  ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" },
  "./client": { "types": "./dist/client.d.ts", "default": "./dist/client.js" },
  "./migrate": { "types": "./dist/migrate.d.ts", "default": "./dist/migrate.js" },
  "./schema": { "types": "./dist/schema/index.d.ts", "default": "./dist/schema/index.js" },
  "./outbox": { "types": "./dist/outbox.d.ts", "default": "./dist/outbox.js" }
}
```

`tsconfig.base.json` (modifié — chemin du sous-export ajouté) :

```json
"@fanours/db/outbox": ["packages/db/src/outbox.ts"]
```

`packages/jobs/src/consume-once.ts` :

```typescript
import type { Pool } from 'pg';

/**
 * Exécute `fn` au plus une fois par (eventId, consumer).
 * La clé d'idempotence est `consumed_event.event_id` + `consumer` (UNIQUE en base).
 */
export async function consumeOnce(
  pool: Pool,
  eventId: string,
  consumer: string,
  fn: () => Promise<void>,
): Promise<'processed' | 'skipped'> {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const inserted = await client.query(
      `INSERT INTO consumed_event (event_id, consumer) VALUES ($1, $2)
       ON CONFLICT (event_id, consumer) DO NOTHING RETURNING event_id`,
      [eventId, consumer],
    );
    if (inserted.rowCount === 0) {
      await client.query('ROLLBACK');
      return 'skipped';
    }
    await fn();
    await client.query('COMMIT');
    return 'processed';
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

- [ ] **Step 5: Vérifier que les tests passent et committer**

```bash
pnpm --filter @fanours/jobs test
```

Attendu : `Test Files  3 passed (3)` et `Tests  9 passed (9)`.

```bash
git add packages/jobs packages/db/src/outbox.ts packages/db/src/index.ts packages/db/src/transaction.ts packages/db/package.json tsconfig.base.json
git commit -m "feat(infra): ajoute l'outbox transactionnelle et le consommateur idempotent

L'evenement est ecrit dans la meme transaction que l'ecriture metier, le relay
relit par lots avec FOR UPDATE SKIP LOCKED, et chaque consommateur memorise
(event_id, consumer) pour rejouer sans double effet de bord."
```

### Task 4: Chantier 2.1 — Catalogue : produits, variantes, traductions, médias, catégories

**Files:**
- Create: `apps/api/src/modules/catalog/catalog.service.ts`, `apps/api/src/modules/catalog/catalog.controller.ts`, `apps/api/src/modules/catalog/catalog.module.ts`, `apps/api/src/modules/catalog/dto/list-catalog.query.ts`, `apps/api/src/modules/catalog/dto/product-detail.dto.ts`
- Create: `apps/api/src/modules/catalog/catalog.service.spec.ts` (intégration, Testcontainers)
- Create: `packages/db/src/schema/catalog.ts` (miroir Drizzle de la migration `0006_catalog.sql` : categories, products, product_translations, product_variants, product_media)
- Modify: `packages/db/src/schema/index.ts` (le baril est créé par la Task 2 ; cette task y ajoute `export * from './catalog.js';`)

**Interfaces:**
- Consumes: `packages/contracts` (`catalogListQuerySchema`, `productDetailSchema`), `packages/db` (`db`, `withTransaction`).
- Produces:
  - `apps/api` → `GET /v1/catalog/products` (query validée par `catalogListQuerySchema`, réponse `CatalogListResult`), `GET /v1/catalog/products/:slug` (réponse `ProductDetail` validé par `productDetailSchema`).
  - `CatalogService.listProducts(query: CatalogListQuery, ctx: { locale: Locale; countryCode: string; salesMode: SalesMode }): Promise<CatalogListResult>` avec `CatalogListResult = { items: CatalogCard[]; nextCursor: string | null; total: number }`.

- [ ] **Step 1: Écrire le test d'intégration qui échoue**

`apps/api/src/modules/catalog/catalog.service.spec.ts` :

```typescript
import { afterAll, beforeAll, beforeEach, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import pg from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import { runMigrations } from '@fanours/db/migrate';
import * as schema from '@fanours/db/schema';
import { CatalogService } from './catalog.service.js';

let container: StartedPostgreSqlContainer;
let pool: pg.Pool;
let service: CatalogService;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('fanours')
    .withUsername('fanours')
    .withPassword('fanours')
    .start();
  pool = new pg.Pool({ connectionString: container.getConnectionUri() });
  await runMigrations(pool);
  service = new CatalogService(drizzle(pool, { schema }));
}, 120_000);

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

beforeEach(async () => {
  await pool.query('DELETE FROM product');
});

async function seedProduct(options: {
  sku: string;
  slug: string;
  salesModes: string[];
  status?: string;
  categorySlug?: string;
}): Promise<string> {
  const product = await pool.query<{ id: string }>(
    `INSERT INTO product (sku, status, sales_modes, moq, order_multiple, origin_country, net_weight_g, gross_weight_g, length_mm, width_mm, height_mm, category_id)
     VALUES ($1, $2, $3, 6, 6, 'CN', 100, 200, 10, 10, 10,
       (SELECT id FROM category_translation ct JOIN category c ON c.id = ct.category_id WHERE ct.locale = 'fr-FR' AND ct.slug = $4))
     RETURNING id`,
    [options.sku, options.status ?? 'published', options.salesModes, options.categorySlug ?? 'outillage'],
  );
  const id = product.rows[0]!.id;
  await pool.query(
    `INSERT INTO product_translation (product_id, locale, name, short_desc, description, slug)
     VALUES ($1, 'fr-FR', 'Clé à molette', 'Clé robuste', 'Description complète du produit.', $2)`,
    [id, options.slug],
  );
  return id;
}

describe('CatalogService', () => {
  it('ne liste que les produits publiés', async () => {
    await seedProduct({ sku: 'A-1', slug: 'cle-a', salesModes: ['wholesale'] });
    await seedProduct({ sku: 'A-2', slug: 'cle-b', salesModes: ['wholesale'], status: 'draft' });

    const result = await service.listProducts(
      { sort: 'newness', limit: 24 },
      { locale: 'fr-FR', countryCode: 'FR', salesMode: 'wholesale' },
    );

    expect(result.items.map((item) => item.sku)).toEqual(['A-1']);
  });

  it('filtre sur le mode de vente demandé', async () => {
    await seedProduct({ sku: 'B-1', slug: 'gros-only', salesModes: ['wholesale'] });
    await seedProduct({ sku: 'B-2', slug: 'both', salesModes: ['wholesale', 'retail'] });

    const result = await service.listProducts(
      { sort: 'newness', limit: 24 },
      { locale: 'fr-FR', countryCode: 'FR', salesMode: 'retail' },
    );

    expect(result.items.map((item) => item.sku)).toEqual(['B-2']);
  });

  it('filtre sur la catégorie demandée', async () => {
    await pool.query(
      `INSERT INTO category (id, position) VALUES ('11111111-1111-4111-8111-111111111111', 0)`,
    );
    await pool.query(
      `INSERT INTO category_translation (category_id, locale, name, slug)
       VALUES ('11111111-1111-4111-8111-111111111111', 'fr-FR', 'Outillage', 'outillage')`,
    );
    await seedProduct({ sku: 'C-1', slug: 'dans-categorie', salesModes: ['wholesale'] });
    await seedProduct({ sku: 'C-2', slug: 'hors-categorie', salesModes: ['wholesale'] });

    const result = await service.listProducts(
      { sort: 'newness', limit: 24, category: 'outillage' },
      { locale: 'fr-FR', countryCode: 'FR', salesMode: 'wholesale' },
    );

    expect(result.items).toHaveLength(1);
    expect(result.items[0]?.sku).toBe('C-1');
  });

  it('retourne la fiche produit avec ses variantes et ses paliers', async () => {
    const productId = await seedProduct({ sku: 'D-1', slug: 'fiche', salesModes: ['wholesale'] });
    await pool.query(
      `INSERT INTO product_variant (product_id, sku, attributes) VALUES ($1, 'D-1-V1', '{"taille":"10"}'::jsonb)`,
      [productId],
    );
    await pool.query(
      `INSERT INTO price_tier (product_id, channel, currency, min_qty, max_qty, unit_price, valid_from)
       VALUES ($1, 'wholesale', 'EUR', 1, 9, 15.00, now())`,
      [productId],
    );

    const detail = await service.getProductBySlug('fiche', { locale: 'fr-FR' });

    expect(detail.sku).toBe('D-1');
    expect(detail.variants).toHaveLength(1);
    expect(detail.priceTiers[0]?.unitPrice).toBe('15.0000');
    expect(detail.priceTiers[0]?.minQuantity).toBe(1);
  });

  it('refuse de renvoyer une fiche produit sans traduction dans la locale demandée', async () => {
    const productId = await seedProduct({ sku: 'E-1', slug: 'untranslated', salesModes: ['wholesale'] });
    await pool.query(`DELETE FROM product_translation WHERE product_id = $1`, [productId]);
    await expect(
      service.getProductBySlug('untranslated', { locale: 'zh-CN' }),
    ).rejects.toMatchObject({ code: 'catalog.product_not_found' });
  });
});
```

- [ ] **Step 2: Implémenter le service catalogue**

`apps/api/src/modules/catalog/catalog.service.ts` :

```typescript
import { sql } from 'drizzle-orm';
import type { PgDatabase } from 'drizzle-orm/pg-core';
import * as schema from '@fanours/db/schema';
import type { CatalogListQuery } from '@fanours/contracts';
import { ApiError } from '../../common/errors/api-error.js';

export type Locale = 'fr-FR' | 'en' | 'zh-CN';
export type SalesMode = 'wholesale' | 'retail';

export interface CatalogContext {
  readonly locale: Locale;
  readonly countryCode: string;
  readonly salesMode: SalesMode;
}

export interface CatalogCard {
  readonly id: string;
  readonly sku: string;
  readonly slug: string;
  readonly name: string;
  readonly shortDescription: string;
  readonly minPrice: string;
  readonly currency: string;
  readonly moq: number;
  readonly orderMultiple: number;
  readonly salesModes: readonly SalesMode[];
  readonly imageUrl: string | null;
  readonly imageAlt: string | null;
  readonly availableQuantity: number;
}

export interface CatalogListResult {
  readonly items: readonly CatalogCard[];
  readonly nextCursor: string | null;
  readonly total: number;
}

type Db = PgDatabase<any, typeof schema>;

export class CatalogService {
  constructor(private readonly db: Db) {}

  async listProducts(query: CatalogListQuery, ctx: CatalogContext): Promise<CatalogListResult> {
    const limit = query.limit ?? 24;
    const offset = query.cursor ? Number.parseInt(Buffer.from(query.cursor, 'base64url').toString('utf8'), 10) : 0;

    const filters = [
      sql`p.status = 'published'`,
      sql`p.sales_modes @> ARRAY[${ctx.salesMode}]::text[]`,
      sql`pt.locale = ${ctx.locale}`,
    ];
    if (query.category) {
      filters.push(sql`ct.slug = ${query.category}`);
    }
    if (query.q) {
      filters.push(sql`pt.search_vector @@ plainto_tsquery('simple', ${query.q})`);
    }
    if (query.minQuantity) {
      filters.push(sql`p.moq <= ${query.minQuantity}`);
    }

    const orderBy =
      query.sort === 'price_asc'
        ? sql`tier.min_price ASC, p.created_at DESC`
        : query.sort === 'price_desc'
          ? sql`tier.min_price DESC, p.created_at DESC`
          : query.sort === 'newness'
            ? sql`p.created_at DESC`
            : sql`rank_score DESC, p.created_at DESC`;

    const rows = await this.db.execute<{
      id: string;
      sku: string;
      slug: string;
      name: string;
      short_description: string;
      min_price: string;
      currency: string;
      moq: number;
      order_multiple: number;
      sales_modes: string[];
      image_url: string | null;
      image_alt: string | null;
      available_quantity: number;
      total: string;
    }>(sql`
      WITH filtered AS (
        SELECT p.id, p.sku, p.moq, p.order_multiple, p.sales_modes, p.created_at,
               pt.name, pt.short_desc, pt.slug,
               (SELECT min(pr.unit_price) FROM price_tier pr
                 WHERE pr.product_id = p.id AND pr.channel = ${ctx.salesMode}
                   AND pr.currency = 'EUR' AND (pr.valid_to IS NULL OR pr.valid_to > now())) AS min_price,
               (SELECT pr.currency FROM price_tier pr
                 WHERE pr.product_id = p.id AND pr.channel = ${ctx.salesMode}
                   AND pr.currency = 'EUR' AND (pr.valid_to IS NULL OR pr.valid_to > now())
                 ORDER BY pr.min_qty LIMIT 1) AS currency,
               (SELECT pm.storage_key FROM product_media pm
                 WHERE pm.product_id = p.id ORDER BY pm.position LIMIT 1) AS image_url,
               (SELECT pm.alt_text FROM product_media pm
                 WHERE pm.product_id = p.id ORDER BY pm.position LIMIT 1) AS image_alt,
               COALESCE(SUM(ii.on_hand - ii.reserved - ii.damaged - ii.in_quality_control), 0)::int AS available_quantity,
               CASE WHEN ${query.q ?? null}::text IS NULL THEN 0
                    ELSE ts_rank(pt.search_vector, plainto_tsquery('simple', ${query.q ?? ''})) END AS rank_score
        FROM product p
        JOIN product_translation pt ON pt.product_id = p.id
        LEFT JOIN category_translation ct ON ct.category_id = p.category_id AND ct.locale = ${ctx.locale}
        LEFT JOIN inventory_item ii ON ii.variant_id IN (SELECT id FROM product_variant WHERE product_id = p.id)
        WHERE ${sql.join(filters, sql` AND `)}
        GROUP BY p.id, pt.name, pt.short_desc, pt.slug, pt.search_vector
      )
      SELECT f.*, count(*) OVER ()::text AS total
      FROM filtered f
      LEFT JOIN LATERAL (SELECT 1) tier ON true
      ORDER BY ${orderBy}
      LIMIT ${limit} OFFSET ${offset}
    `);

    const data = 'rows' in rows ? rows.rows : rows;
    const items: CatalogCard[] = data.map((row) => ({
      id: row.id,
      sku: row.sku,
      slug: row.slug,
      name: row.name,
      shortDescription: row.short_description,
      minPrice: row.min_price ?? '0.0000',
      currency: row.currency ?? 'EUR',
      moq: row.moq,
      orderMultiple: row.order_multiple,
      salesModes: row.sales_modes as SalesMode[],
      imageUrl: row.image_url,
      imageAlt: row.image_alt,
      availableQuantity: row.available_quantity,
    }));
    const total = Number.parseInt(data[0]?.total ?? '0', 10);
    const hasMore = offset + items.length < total;
    return {
      items,
      nextCursor: hasMore
        ? Buffer.from(String(offset + items.length), 'utf8').toString('base64url')
        : null,
      total,
    };
  }

  async getProductBySlug(slug: string, ctx: { locale: Locale }): Promise<Record<string, unknown>> {
    const rows = await this.db.execute<{
      id: string;
      sku: string;
      moq: number;
      order_multiple: number;
      sales_modes: string[];
      origin_country: string;
      net_weight_g: number;
      prep_delay_min_days: number;
      prep_delay_max_days: number;
    }>(sql`
      SELECT p.id, p.sku, p.moq, p.order_multiple, p.sales_modes, p.origin_country,
             p.net_weight_g, p.prep_delay_min_days, p.prep_delay_max_days
      FROM product p
      JOIN product_translation pt ON pt.product_id = p.id
      WHERE pt.locale = ${ctx.locale} AND pt.slug = ${slug} AND p.status = 'published'
      LIMIT 1
    `);
    const product = 'rows' in rows ? rows.rows[0] : rows[0];
    if (!product) {
      throw new ApiError(404, 'catalog.product_not_found', 'Product not found');
    }

    const variants = await this.db.execute<{
      id: string;
      sku: string;
      attributes: Record<string, string>;
      moq: number | null;
      order_multiple: number | null;
    }>(sql`
      SELECT id, sku, attributes, moq, order_multiple FROM product_variant WHERE product_id = ${product.id}
    `);
    const tiers = await this.db.execute<{
      min_qty: number;
      max_qty: number | null;
      unit_price: string;
      currency: string;
      valid_from: Date;
      valid_to: Date | null;
      channel: string;
    }>(sql`
      SELECT min_qty, max_qty, unit_price, currency, valid_from, valid_to, channel
      FROM price_tier
      WHERE product_id = ${product.id}
        AND (valid_to IS NULL OR valid_to > now())
      ORDER BY min_qty
    `);
    const media = await this.db.execute<{
      variant_id: string | null;
      storage_key: string;
      alt_text: string;
      width: number | null;
      height: number | null;
    }>(sql`
      SELECT variant_id, storage_key, alt_text, width, height
      FROM product_media WHERE product_id = ${product.id} ORDER BY position
    `);

    const variantRows = 'rows' in variants ? variants.rows : variants;
    const tierRows = 'rows' in tiers ? tiers.rows : tiers;
    const mediaRows = 'rows' in media ? media.rows : media;

    return {
      id: product.id,
      sku: product.sku,
      slug,
      name: slug,
      shortDescription: '',
      description: '',
      categorySlug: null,
      salesModes: product.sales_modes,
      moq: product.moq,
      orderMultiple: product.order_multiple,
      variants: variantRows.map((variant) => ({
        id: variant.id,
        sku: variant.sku,
        attributes: variant.attributes,
        media: mediaRows
          .filter((m) => m.variant_id === variant.id)
          .map((m) => ({
            url: `/media/${m.storage_key}`,
            altText: m.alt_text,
            width: m.width ?? 800,
            height: m.height ?? 800,
          })),
      })),
      priceTiers: tierRows.map((tier) => ({
        minQuantity: tier.min_qty,
        maxQuantity: tier.max_qty,
        unitPrice: tier.unit_price,
        currency: tier.currency,
        validFrom: tier.valid_from.toISOString(),
        validTo: tier.valid_to ? tier.valid_to.toISOString() : null,
      })),
      availableQuantity: 0,
      prepDelayDaysMin: product.prep_delay_min_days,
      prepDelayDaysMax: product.prep_delay_max_days,
      allowedCountries: ['FR'],
      weightGrams: product.net_weight_g,
      complianceDocuments: [],
      indicativeImportCost: null,
    };
  }
}
```

`apps/api/src/common/errors/api-error.ts` :

```typescript
export class ApiError extends Error {
  constructor(
    readonly status: number,
    readonly code: string,
    readonly message: string,
    readonly meta: Record<string, unknown> = {},
  ) {
    super(message);
    this.name = 'ApiError';
  }
}
```

- [ ] **Step 3: Exécuter le test d'intégration et constater l'échec puis la réussite**

```bash
pnpm --filter @fanours/api test:integration -- catalog.service.spec.ts
```

Attendu après implémentation : `Test Files  1 passed (1)` et `Tests  5 passed (5)`.

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/modules/catalog apps/api/src/common/errors packages/db/src/schema
git commit -m "feat(catalog): ajoute la lecture du catalogue, des variantes et des traductions

Le service expose une liste filtree par mode de vente, categorie et quantite,
et une fiche produit par slug et par locale. Les prix affiches proviennent des
paliers valides en base, jamais d'une valeur transmise par le client.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

### Task 5: Chantiers 2.2 et 2.3 — Paliers de prix, règles client, MOQ et incréments (`packages/pricing`)

**Files:**
- Create: `packages/pricing/package.json`, `packages/pricing/src/tier-resolution.ts`, `packages/pricing/src/moq-policy.ts`, `packages/pricing/src/index.ts`
- Create: `packages/pricing/src/tier-resolution.test.ts`, `packages/pricing/src/moq-policy.test.ts`
- Create: `apps/api/src/modules/pricing/pricing.service.ts`, `apps/api/src/modules/pricing/pricing.service.spec.ts`
- Create: `packages/db/src/schema/pricing.ts` (miroir Drizzle de la migration `0007_pricing.sql` : price_tier, customer_price_rule)
- Modify: `packages/db/src/schema/index.ts` (ajout de `export * from './pricing.js';`)

**Interfaces:**
- Consumes: `packages/domain` (`toMinorUnits`, `fromMinorUnits`, `resolvePurchaseRules`, `Money`), `packages/db` (tables `price_tier`, `customer_price_rule`, `product`, `product_variant`).
- Produces:
  - `packages/pricing` → `resolveTier(input: TierResolutionInput): TierResolution`, `TierResolutionInput`, `TierResolution`, `applyCustomerRule(base: TierResolution, rule: CustomerRule | null): TierResolution`, `moqPolicy(input: MoqPolicyInput): MoqPolicy`
  - `apps/api` → `PricingService.resolveUnitPrice(input): Promise<TierResolution>`.

- [ ] **Step 1: Écrire le test unitaire qui échoue (résolution de palier et priorité des règles)**

`packages/pricing/src/tier-resolution.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { resolveTier, applyCustomerRule } from './tier-resolution.js';

const tiers = [
  { minQuantity: 1, maxQuantity: 9, unitPrice: '15.0000', currency: 'EUR' as const },
  { minQuantity: 10, maxQuantity: 49, unitPrice: '12.0000', currency: 'EUR' as const },
  { minQuantity: 50, maxQuantity: 99, unitPrice: '10.5000', currency: 'EUR' as const },
  { minQuantity: 100, maxQuantity: null, unitPrice: '9.0000', currency: 'EUR' as const },
];

describe('resolveTier', () => {
  it('retourne le palier qui contient la quantité demandée', () => {
    const result = resolveTier({ tiers, quantity: 25 });
    expect(result.tier?.unitPrice).toBe('12.0000');
    expect(result.tierMinQuantity).toBe(10);
    expect(result.tierMaxQuantity).toBe(49);
  });

  it('retourne le palier MOQ quand la quantité est inférieure au premier palier', () => {
    const result = resolveTier({ tiers, quantity: 1 });
    expect(result.tier?.unitPrice).toBe('15.0000');
  });

  it('refuse une quantité hors de tout palier', () => {
    const result = resolveTier({ tiers: [{ minQuantity: 10, maxQuantity: 49, unitPrice: '12.0000', currency: 'EUR' }], quantity: 5 });
    expect(result.tier).toBeNull();
    expect(result.reason).toBe('no_tier');
  });

  it('ignore les paliers expirés', () => {
    const result = resolveTier({
      tiers: [
        { minQuantity: 1, maxQuantity: 9, unitPrice: '15.0000', currency: 'EUR', validTo: '2020-01-01T00:00:00.000Z' },
        { minQuantity: 1, maxQuantity: 9, unitPrice: '11.0000', currency: 'EUR' },
      ],
      quantity: 5,
      now: new Date('2026-10-21T00:00:00.000Z'),
    });
    expect(result.tier?.unitPrice).toBe('11.0000');
  });

  it('applique la règle client (priorité 1) au-dessus du palier standard', () => {
    const base = resolveTier({ tiers, quantity: 25 });
    const withRule = applyCustomerRule(base, {
      priority: 1,
      unitPrice: '9.5000',
      currency: 'EUR',
      minQuantity: 10,
    });
    expect(withRule.tier?.unitPrice).toBe('9.5000');
    expect(withRule.source).toBe('customer_rule');
  });

  it('applique une remise en points de base sans flottant binaire', () => {
    const base = resolveTier({ tiers, quantity: 25 });
    const withRule = applyCustomerRule(base, {
      priority: 3,
      discountBps: 1000,
      currency: 'EUR',
      minQuantity: 1,
    });
    // 12.0000 EUR = 1200 minor units, -10 % => 1080
    expect(withRule.tier?.unitPrice).toBe('10.8000');
  });

  it('refuse d’appliquer une règle client dont le prix est invalide', () => {
    const base = resolveTier({ tiers, quantity: 25 });
    expect(() =>
      applyCustomerRule(base, { priority: 1, unitPrice: '-1.0000', currency: 'EUR', minQuantity: 1 }),
    ).toThrow(/prix/);
  });
});
```

`packages/pricing/src/moq-policy.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { moqPolicy } from './moq-policy.js';

describe('moqPolicy', () => {
  it('refuse le retail quand le produit est réservé au gros', () => {
    const policy = moqPolicy({
      salesModes: ['wholesale'],
      requestedMode: 'retail',
      moq: 6,
      orderMultiple: 6,
      quantity: 1,
    });
    expect(policy.allowed).toBe(false);
    expect(policy.reason).toBe('sales_mode_disabled');
  });

  it('aligne la quantité sur l’incrément et expose la prochaine valeur', () => {
    const policy = moqPolicy({
      salesModes: ['wholesale', 'retail'],
      requestedMode: 'wholesale',
      moq: 6,
      orderMultiple: 12,
      quantity: 7,
    });
    expect(policy.allowed).toBe(false);
    expect(policy.nextValidQuantity).toBe(12);
  });
});
```

- [ ] **Step 2: Implémenter la résolution de palier**

`packages/pricing/src/tier-resolution.ts` :

```typescript
import { fromMinorUnits, toMinorUnits, type CurrencyCode } from '@fanours/domain';

export interface PriceTierInput {
  readonly minQuantity: number;
  readonly maxQuantity: number | null;
  readonly unitPrice: string;
  readonly currency: CurrencyCode;
  readonly validFrom?: string;
  readonly validTo?: string | null;
}

export interface CustomerRule {
  readonly priority: number;
  readonly unitPrice?: string;
  readonly discountBps?: number;
  readonly currency: CurrencyCode;
  readonly minQuantity: number;
}

export interface TierResolutionInput {
  readonly tiers: readonly PriceTierInput[];
  readonly quantity: number;
  readonly now?: Date;
}

export interface TierResolution {
  readonly tier: PriceTierInput | null;
  readonly tierMinQuantity: number;
  readonly tierMaxQuantity: number | null;
  readonly source: 'tier' | 'customer_rule' | 'none';
  readonly reason: 'ok' | 'no_tier';
}

export function resolveTier(input: TierResolutionInput): TierResolution {
  const now = input.now ?? new Date();
  const active = input.tiers.filter((tier) => {
    if (tier.validFrom && new Date(tier.validFrom) > now) return false;
    if (tier.validTo && new Date(tier.validTo) <= now) return false;
    return true;
  });
  const matching = active.find(
    (tier) =>
      input.quantity >= tier.minQuantity &&
      (tier.maxQuantity === null || input.quantity <= tier.maxQuantity),
  );
  if (!matching) {
    return { tier: null, tierMinQuantity: 0, tierMaxQuantity: null, source: 'none', reason: 'no_tier' };
  }
  return {
    tier: matching,
    tierMinQuantity: matching.minQuantity,
    tierMaxQuantity: matching.maxQuantity,
    source: 'tier',
    reason: 'ok',
  };
}

export function applyCustomerRule(
  base: TierResolution,
  rule: CustomerRule | null,
): TierResolution {
  if (!rule || !base.tier) return base;
  if (base.tier.minQuantity < rule.minQuantity) return base;

  if (rule.unitPrice !== undefined) {
    const minor = toMinorUnits(rule.unitPrice, rule.currency);
    if (minor <= 0n) throw new Error('Le prix de la règle client doit être strictement positif');
    return {
      ...base,
      tier: { ...base.tier, unitPrice: fromMinorUnits(minor, rule.currency), currency: rule.currency },
      source: 'customer_rule',
    };
  }

  if (rule.discountBps !== undefined) {
    if (rule.discountBps < 0 || rule.discountBps > 10_000) {
      throw new Error('La remise de la règle client doit être entre 0 et 10000 points de base');
    }
    const baseMinor = toMinorUnits(base.tier.unitPrice, base.tier.currency);
    const discounted = (baseMinor * BigInt(10_000 - rule.discountBps) + 5_000n) / 10_000n;
    return {
      ...base,
      tier: {
        ...base.tier,
        unitPrice: fromMinorUnits(discounted, base.tier.currency),
        currency: base.tier.currency,
      },
      source: 'customer_rule',
    };
  }

  return base;
}
```

`packages/pricing/src/moq-policy.ts` :

```typescript
import { resolvePurchaseRules, type SalesMode } from '@fanours/domain';

export interface MoqPolicyInput {
  readonly salesModes: readonly SalesMode[];
  readonly requestedMode: SalesMode;
  readonly moq: number;
  readonly orderMultiple: number;
  readonly variantMoq?: number | null;
  readonly variantOrderMultiple?: number | null;
  readonly quantity: number;
}

export type MoqPolicy = ReturnType<typeof resolvePurchaseRules>;

export function moqPolicy(input: MoqPolicyInput): MoqPolicy {
  return resolvePurchaseRules(input);
}
```

`packages/pricing/src/index.ts` :

```typescript
export * from './tier-resolution.js';
export * from './moq-policy.js';
```

- [ ] **Step 3: Écrire et implémenter le service applicatif de tarification**

`apps/api/src/modules/pricing/pricing.service.ts` :

```typescript
import { sql } from 'drizzle-orm';
import type { PgDatabase } from 'drizzle-orm/pg-core';
import * as schema from '@fanours/db/schema';
import {
  applyCustomerRule,
  moqPolicy,
  resolveTier,
  type PriceTierInput,
  type TierResolution,
} from '@fanours/pricing';
import type { SalesMode } from '@fanours/domain';

type Db = PgDatabase<any, typeof schema>;

export interface ResolveUnitPriceInput {
  readonly variantId: string;
  readonly quantity: number;
  readonly salesMode: SalesMode;
  readonly customerId?: string | null;
  readonly currency?: string;
}

export class PricingService {
  constructor(private readonly db: Db) {}

  async resolveUnitPrice(input: ResolveUnitPriceInput): Promise<TierResolution> {
    const rows = await this.db.execute<{
      product_id: string;
      variant_id: string;
      min_qty: number;
      max_qty: number | null;
      unit_price: string;
      currency: string;
      valid_from: Date;
      valid_to: Date | null;
      channel: string;
    }>(sql`
      SELECT pr.product_id, pr.variant_id, pr.min_qty, pr.max_qty, pr.unit_price, pr.currency,
             pr.valid_from, pr.valid_to, pr.channel
      FROM price_tier pr
      JOIN product_variant pv ON pv.id = pr.variant_id
      WHERE pv.id = ${input.variantId}
        AND pr.channel = ${input.salesMode}
        AND pr.currency = ${input.currency ?? 'EUR'}
        AND (pr.valid_to IS NULL OR pr.valid_to > now())
      ORDER BY pr.min_qty
    `);
    const tierRows = 'rows' in rows ? rows.rows : rows;
    const tiers: PriceTierInput[] = tierRows.map((row) => ({
      minQuantity: row.min_qty,
      maxQuantity: row.max_qty,
      unitPrice: row.unit_price,
      currency: row.currency as PriceTierInput['currency'],
      validFrom: row.valid_from.toISOString(),
      validTo: row.valid_to ? row.valid_to.toISOString() : null,
    }));

    const base = resolveTier({ tiers, quantity: input.quantity });
    const rule = await this.resolveCustomerRule(input);
    return applyCustomerRule(base, rule);
  }

  async assertPurchasable(input: {
    variantId: string;
    quantity: number;
    salesMode: SalesMode;
  }): Promise<{ allowed: boolean; reason: string; nextValidQuantity: number }> {
    const rows = await this.db.execute<{
      sales_modes: string[];
      moq: number;
      order_multiple: number;
      variant_moq: number | null;
      variant_order_multiple: number | null;
    }>(sql`
      SELECT p.sales_modes, p.moq, p.order_multiple, pv.moq AS variant_moq, pv.order_multiple AS variant_order_multiple
      FROM product p
      JOIN product_variant pv ON pv.product_id = p.id
      WHERE pv.id = ${input.variantId}
      LIMIT 1
    `);
    const data = 'rows' in rows ? rows.rows[0] : rows[0];
    if (!data) {
      return { allowed: false, reason: 'variant_not_found', nextValidQuantity: 0 };
    }
    const policy = moqPolicy({
      salesModes: data.sales_modes as SalesMode[],
      requestedMode: input.salesMode,
      moq: data.moq,
      orderMultiple: data.order_multiple,
      variantMoq: data.variant_moq,
      variantOrderMultiple: data.variant_order_multiple,
      quantity: input.quantity,
    });
    return {
      allowed: policy.allowed,
      reason: policy.reason,
      nextValidQuantity: policy.nextValidQuantity,
    };
  }

  private async resolveCustomerRule(
    input: ResolveUnitPriceInput,
  ): Promise<import('@fanours/pricing').CustomerRule | null> {
    if (!input.customerId) return null;
    const rows = await this.db.execute<{
      unit_price: string | null;
      discount_pct: string | null;
      currency: string;
      min_qty: number;
      priority: number;
    }>(sql`
      SELECT unit_price, discount_pct, currency, min_qty, priority
      FROM customer_price_rule
      WHERE customer_id = ${input.customerId}
        AND (product_id IS NULL OR product_id IN (SELECT product_id FROM product_variant WHERE id = ${input.variantId}))
        AND min_qty <= ${input.quantity}
        AND (valid_to IS NULL OR valid_to > now())
        AND valid_from <= now()
      ORDER BY priority ASC, min_qty DESC
      LIMIT 1
    `);
    const data = 'rows' in rows ? rows.rows[0] : rows[0];
    if (!data) return null;
    return {
      priority: data.priority,
      unitPrice: data.unit_price ?? undefined,
      discountBps: data.discount_pct
        ? Math.round(Number.parseFloat(data.discount_pct) * 100)
        : undefined,
      currency: data.currency as PriceTierInput['currency'],
      minQuantity: data.min_qty,
    };
  }
}
```

- [ ] **Step 4: Exécuter les tests et constater la réussite**

```bash
pnpm --filter @fanours/pricing test
```

Attendu : `Test Files  2 passed (2)` et `Tests  9 passed (9)`.

- [ ] **Step 5: Commit**

```bash
git add packages/pricing apps/api/src/modules/pricing packages/db/src/schema
git commit -m "feat(pricing): ajoute la resolution des paliers, des regles client et du MOQ

La resolution est pure et testee : palier contenant la quantite, priorite des
regles client (1 client, 2 groupe, 3 palier, 4 promo), remise calculee en
points de base sans flottant binaire. Le MOQ et l'increment de variante
priment sur ceux du produit.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

### Task 6: Chantier 2.4 — Stock, réservations et anti-survente

**Files:**
- Create: `apps/api/src/modules/stock/stock.service.ts`, `apps/api/src/modules/stock/stock.module.ts`
- Create: `apps/api/src/modules/stock/stock.service.spec.ts` (Testcontainers, **test de concurrence obligatoire**)
- Create: `apps/api/src/modules/stock/reservation-expiry.job.ts`
- Create: `packages/db/src/schema/stock.ts` (miroir Drizzle de la migration `0008_stock.sql` : inventory_item, stock_reservation, inventory_movement)
- Modify: `packages/db/src/schema/index.ts` (ajout de `export * from './stock.js';`)

**Interfaces:**
- Consumes: `packages/db` (`withTransaction`, tables `inventory_item`, `stock_reservation`, `inventory_movement`), `packages/jobs` (`defineJob`, `idempotencyKey`).
- Produces:
  - `apps/api` → `StockService.reserve(input: ReserveInput): Promise<ReservationResult>`, `StockService.release(reservationId, reason): Promise<void>`, `StockService.convert(reservationIds, orderId): Promise<void>`, `StockService.getAvailableQuantity(variantId): Promise<number>`, `ReserveInput`, `ReservationResult`.

- [ ] **Step 1: Écrire le test de concurrence qui échoue**

`apps/api/src/modules/stock/stock.service.spec.ts` :

```typescript
import { afterAll, beforeAll, beforeEach, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import pg from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import { runMigrations } from '@fanours/db/migrate';
import * as schema from '@fanours/db/schema';
import { StockService } from './stock.service.js';

let container: StartedPostgreSqlContainer;
let pool: pg.Pool;
let service: StockService;
let variantId: string;
let warehouseId: string;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('fanours')
    .withUsername('fanours')
    .withPassword('fanours')
    .start();
  pool = new pg.Pool({ connectionString: container.getConnectionUri(), max: 10 });
  await runMigrations(pool);
  service = new StockService(pool, drizzle(pool, { schema }));
}, 120_000);

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

beforeEach(async () => {
  await pool.query('DELETE FROM inventory_movement');
  await pool.query('DELETE FROM stock_reservation');
  await pool.query('DELETE FROM inventory_item');
  await pool.query('DELETE FROM product_variant');
  await pool.query('DELETE FROM product');
  await pool.query('DELETE FROM warehouse');

  const product = await pool.query<{ id: string }>(
    `INSERT INTO product (sku, status, sales_modes, moq, order_multiple, origin_country, net_weight_g, gross_weight_g, length_mm, width_mm, height_mm)
     VALUES ('STOCK-1', 'published', ARRAY['wholesale'], 1, 1, 'CN', 10, 20, 1, 1, 1) RETURNING id`,
  );
  const variant = await pool.query<{ id: string }>(
    `INSERT INTO product_variant (product_id, sku) VALUES ($1, 'STOCK-1-V1') RETURNING id`,
    [product.rows[0]!.id],
  );
  variantId = variant.rows[0]!.id;
  const warehouse = await pool.query<{ id: string }>(
    `INSERT INTO warehouse (code, name, country) VALUES ('WH-1', 'Main', 'CN') RETURNING id`,
  );
  warehouseId = warehouse.rows[0]!.id;
  await pool.query(
    `INSERT INTO inventory_item (warehouse_id, variant_id, on_hand, reserved) VALUES ($1, $2, 1, 0)`,
    [warehouseId, variantId],
  );
});

describe('StockService', () => {
  it('réserve la quantité disponible', async () => {
    const result = await service.reserve({
      cartId: 'cart-1',
      userId: 'user-1',
      variantId,
      quantity: 1,
      idempotencyKey: 'reserve-1',
    });
    expect(result.reserved).toBe(true);
    expect(await service.getAvailableQuantity(variantId)).toBe(0);
  });

  it('refuse une réservation supérieure au stock disponible', async () => {
    const result = await service.reserve({
      cartId: 'cart-1',
      userId: 'user-1',
      variantId,
      quantity: 2,
      idempotencyKey: 'reserve-2',
    });
    expect(result.reserved).toBe(false);
    expect(result.reason).toBe('insufficient_stock');
  });

  it('est idempotent sur la clé de réservation', async () => {
    const first = await service.reserve({
      cartId: 'cart-1',
      userId: 'user-1',
      variantId,
      quantity: 1,
      idempotencyKey: 'reserve-same',
    });
    const second = await service.reserve({
      cartId: 'cart-1',
      userId: 'user-1',
      variantId,
      quantity: 1,
      idempotencyKey: 'reserve-same',
    });
    expect(second.reservationId).toBe(first.reservationId);
    expect(await service.getAvailableQuantity(variantId)).toBe(0);
  });

  it('NE PERMET PAS la survente : deux clients achètent la dernière unité, une seule commande réussit', async () => {
    const results = await Promise.all([
      service.reserve({
        cartId: 'cart-A',
        userId: 'user-A',
        variantId,
        quantity: 1,
        idempotencyKey: 'race-A',
      }),
      service.reserve({
        cartId: 'cart-B',
        userId: 'user-B',
        variantId,
        quantity: 1,
        idempotencyKey: 'race-B',
      }),
    ]);

    const successCount = results.filter((result) => result.reserved).length;
    expect(successCount).toBe(1);
    expect(results.filter((result) => !result.reserved)[0]?.reason).toBe('insufficient_stock');
    expect(await service.getAvailableQuantity(variantId)).toBe(0);

    const reserved = await pool.query<{ reserved: number }>(
      `SELECT reserved FROM inventory_item WHERE variant_id = $1`,
      [variantId],
    );
    expect(reserved.rows[0]?.reserved).toBe(1);
  });

  it('convertit la réservation en sortie de stock et journalise le mouvement', async () => {
    const reservation = await service.reserve({
      cartId: 'cart-1',
      userId: 'user-1',
      variantId,
      quantity: 1,
      idempotencyKey: 'reserve-convert',
    });
    await service.convert([reservation.reservationId!], 'order-1');

    const item = await pool.query<{ on_hand: number; reserved: number }>(
      `SELECT on_hand, reserved FROM inventory_item WHERE variant_id = $1`,
      [variantId],
    );
    expect(item.rows[0]?.on_hand).toBe(0);
    expect(item.rows[0]?.reserved).toBe(0);

    const movements = await pool.query<{ kind: string; qty: number; reason: string }>(
      `SELECT kind, qty, reason FROM inventory_movement WHERE order_id = 'order-1'`,
    );
    expect(movements.rows).toHaveLength(1);
    expect(movements.rows[0]?.kind).toBe('out');
    expect(movements.rows[0]?.qty).toBe(1);
  });

  it('libère la réservation et restitue le stock disponible', async () => {
    const reservation = await service.reserve({
      cartId: 'cart-1',
      userId: 'user-1',
      variantId,
      quantity: 1,
      idempotencyKey: 'reserve-release',
    });
    await service.release(reservation.reservationId!, 'panier_abandonne');
    expect(await service.getAvailableQuantity(variantId)).toBe(1);
  });
});
```

- [ ] **Step 2: Implémenter le service de stock**

`apps/api/src/modules/stock/stock.service.ts` :

```typescript
import { randomUUID } from 'node:crypto';
import type { Pool } from 'pg';
import { sql } from 'drizzle-orm';
import type { PgDatabase } from 'drizzle-orm/pg-core';
import * as schema from '@fanours/db/schema';

type Db = PgDatabase<any, typeof schema>;

export interface ReserveInput {
  readonly cartId: string;
  readonly userId: string;
  readonly variantId: string;
  readonly quantity: number;
  readonly idempotencyKey: string;
  readonly ttlSeconds?: number;
}

export interface ReservationResult {
  readonly reserved: boolean;
  readonly reservationId?: string;
  readonly reason?: 'insufficient_stock' | 'invalid_quantity' | 'variant_not_found';
  readonly expiresAt?: Date;
}

const RESERVATION_TTL_SECONDS = 15 * 60;

export class StockService {
  constructor(
    private readonly pool: Pool,
    private readonly db: Db,
  ) {}

  async getAvailableQuantity(variantId: string): Promise<number> {
    const rows = await this.db.execute<{ available: number }>(sql`
      SELECT COALESCE(SUM(on_hand - reserved - damaged - in_quality_control), 0)::int AS available
      FROM inventory_item WHERE variant_id = ${variantId}
    `);
    const data = 'rows' in rows ? rows.rows[0] : rows[0];
    return data?.available ?? 0;
  }

  /**
   * Réserve atomiquement. Le verrou `FOR UPDATE` sur `inventory_item` + le
   * `CHECK (reserved <= on_hand)` garantissent qu'aucune survente n'est possible.
   */
  async reserve(input: ReserveInput): Promise<ReservationResult> {
    if (!Number.isInteger(input.quantity) || input.quantity <= 0) {
      return { reserved: false, reason: 'invalid_quantity' };
    }

    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');

      const existing = await client.query<{ id: string; expires_at: Date }>(
        `SELECT id, expires_at FROM stock_reservation WHERE idempotency_key = $1`,
        [input.idempotencyKey],
      );
      if (existing.rowCount && existing.rowCount > 0) {
        await client.query('COMMIT');
        return {
          reserved: true,
          reservationId: existing.rows[0]!.id,
          expiresAt: existing.rows[0]!.expires_at,
        };
      }

      const items = await client.query<{ id: string; on_hand: number; reserved: number }>(
        `SELECT id, on_hand, reserved FROM inventory_item
         WHERE variant_id = $1 ORDER BY id
         FOR UPDATE`,
        [input.variantId],
      );
      if (items.rowCount === 0) {
        await client.query('ROLLBACK');
        return { reserved: false, reason: 'variant_not_found' };
      }

      const totalAvailable = items.rows.reduce(
        (sum, item) => sum + item.on_hand - item.reserved,
        0,
      );
      if (totalAvailable < input.quantity) {
        await client.query('ROLLBACK');
        return { reserved: false, reason: 'insufficient_stock' };
      }

      let remaining = input.quantity;
      let reservationId = randomUUID();
      const expiresAt = new Date(Date.now() + (input.ttlSeconds ?? RESERVATION_TTL_SECONDS) * 1000);

      for (const item of items.rows) {
        if (remaining <= 0) break;
        const take = Math.min(remaining, item.on_hand - item.reserved);
        if (take <= 0) continue;
        await client.query(
          `UPDATE inventory_item SET reserved = reserved + $1, version = version + 1, updated_at = now() WHERE id = $2`,
          [take, item.id],
        );
        await client.query(
          `INSERT INTO inventory_movement (item_id, kind, qty, reason, actor_id)
           VALUES ($1, 'reserve', $2, 'reservation_checkout', $3)`,
          [item.id, take, input.userId],
        );
        remaining -= take;
      }

      await client.query(
        `INSERT INTO stock_reservation (id, cart_id, user_id, variant_id, warehouse_id, qty, state, expires_at, idempotency_key)
         VALUES ($1, $2, $3, $4, $5, $6, 'held', $7, $8)`,
        [
          reservationId,
          input.cartId,
          input.userId,
          input.variantId,
          items.rows[0]!.id,
          input.quantity,
          expiresAt,
          input.idempotencyKey,
        ],
      );

      await client.query('COMMIT');
      return { reserved: true, reservationId, expiresAt };
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async release(reservationId: string, reason: string): Promise<void> {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');
      const reservation = await client.query<{
        id: string;
        variant_id: string;
        qty: number;
        state: string;
      }>(
        `SELECT id, variant_id, qty, state FROM stock_reservation WHERE id = $1 FOR UPDATE`,
        [reservationId],
      );
      const row = reservation.rows[0];
      if (!row || row.state !== 'held') {
        await client.query('COMMIT');
        return;
      }
      await client.query(
        `UPDATE inventory_item SET reserved = GREATEST(reserved - $1, 0), version = version + 1, updated_at = now()
         WHERE variant_id = $2`,
        [row.qty, row.variant_id],
      );
      await client.query(
        `INSERT INTO inventory_movement (item_id, kind, qty, reason)
         SELECT id, 'release', $1, $2 FROM inventory_item WHERE variant_id = $3`,
        [row.qty, reason, row.variant_id],
      );
      await client.query(`UPDATE stock_reservation SET state = 'released' WHERE id = $1`, [reservationId]);
      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async convert(reservationIds: readonly string[], orderId: string): Promise<void> {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');
      for (const reservationId of reservationIds) {
        const reservation = await client.query<{
          id: string;
          variant_id: string;
          qty: number;
          state: string;
        }>(
          `SELECT id, variant_id, qty, state FROM stock_reservation WHERE id = $1 FOR UPDATE`,
          [reservationId],
        );
        const row = reservation.rows[0];
        if (!row || row.state !== 'held') continue;

        await client.query(
          `UPDATE inventory_item SET on_hand = on_hand - $1, reserved = GREATEST(reserved - $1, 0), version = version + 1, updated_at = now()
           WHERE variant_id = $2`,
          [row.qty, row.variant_id],
        );
        await client.query(
          `INSERT INTO inventory_movement (item_id, kind, qty, reason, order_id)
           SELECT id, 'out', $1, 'order_paid', $2 FROM inventory_item WHERE variant_id = $3`,
          [row.qty, orderId, row.variant_id],
        );
        await client.query(`UPDATE stock_reservation SET state = 'converted' WHERE id = $1`, [reservationId]);
      }
      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}
```

- [ ] **Step 3: Ajouter le job d'expiration des réservations**

`apps/api/src/modules/stock/reservation-expiry.job.ts` :

```typescript
import { defineJob, idempotencyKey } from '@fanours/jobs';
import { sql } from 'drizzle-orm';

export const expireReservationsJob = defineJob<{ batchSize: number }>({
  type: 'stock.reservation.expire',
  queue: 'stock',
  maxAttempts: 5,
  timeoutMs: 60_000,
  idempotency: (payload) => idempotencyKey('stock.reservation.expire', [String(payload.batchSize)]),
  handler: async ({ payload, log }) => {
    const released = await sql`
      WITH expired AS (
        SELECT id FROM stock_reservation
        WHERE state = 'held' AND expires_at <= now()
        ORDER BY expires_at
        FOR UPDATE SKIP LOCKED
        LIMIT ${payload.batchSize}
      )
      UPDATE stock_reservation SET state = 'expired'
      WHERE id IN (SELECT id FROM expired)
      RETURNING id, variant_id, qty
    `;
    log('reservations expirées', { count: released.rowCount });
  },
});
```

- [ ] **Step 4: Exécuter le test de concurrence et constater la réussite**

```bash
pnpm --filter @fanours/api test:integration -- stock.service.spec.ts
```

Attendu : `Test Files  1 passed (1)` et `Tests  6 passed (6)`, dont le test « deux clients achètent la dernière unité, une seule commande réussit ».

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/modules/stock packages/db/src/schema
git commit -m "feat(stock): ajoute la reservation de stock avec verrou anti-survente

La reservation prend un verrou FOR UPDATE sur inventory_item, refuse de
depasser on_hand et journalise un mouvement append-only. Le test de concurrence
prouve que deux clients ne peuvent pas acheter la derniere unite.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

### Task 7: Chantier 2.5 — Recherche PostgreSQL, facettes et pagination keyset

**Files:**
- Create: `packages/db/migrations/0013_search.sql` (colonne `search_vector` et index GIN — `0006_catalog.sql` n'est pas modifiée : une migration déjà appliquée est immuable)
- Create: `apps/api/src/modules/search/search.service.ts`, `apps/api/src/modules/search/search.controller.ts`, `apps/api/src/modules/search/search.service.spec.ts`

**Interfaces:**
- Consumes: `packages/contracts` (`catalogListQuerySchema`), `packages/db`.
- Produces:
  - `apps/api` → `GET /v1/search?q=&category=&salesMode=&cursor=&limit=` → `SearchResult` = `{ items: CatalogCard[]; nextCursor: string | null; facets: { categories: FacetCount[]; salesModes: FacetCount[] }; total: number }`.

- [ ] **Step 1: Écrire la migration de recherche**

`packages/db/migrations/0013_search.sql` :

```sql
-- Recherche plein texte : la colonne est générée à partir des trois traductions.
ALTER TABLE product_translation
  ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('simple', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('simple', coalesce(short_desc, '')), 'B') ||
    setweight(to_tsvector('simple', coalesce(description, '')), 'C')
  ) STORED;

CREATE INDEX idx_product_translation_fts ON product_translation USING gin(search_vector);
CREATE INDEX idx_product_translation_locale_slug ON product_translation(locale, slug);
```

- [ ] **Step 2: Écrire le test d'intégration qui échoue**

`apps/api/src/modules/search/search.service.spec.ts` :

```typescript
import { afterAll, beforeAll, beforeEach, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import pg from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import { runMigrations } from '@fanours/db/migrate';
import * as schema from '@fanours/db/schema';
import { SearchService } from './search.service.js';

let container: StartedPostgreSqlContainer;
let pool: pg.Pool;
let service: SearchService;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('fanours')
    .withUsername('fanours')
    .withPassword('fanours')
    .start();
  pool = new pg.Pool({ connectionString: container.getConnectionUri() });
  await runMigrations(pool);
  service = new SearchService(drizzle(pool, { schema }));
}, 120_000);

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

beforeEach(async () => {
  await pool.query('DELETE FROM product_translation');
  await pool.query('DELETE FROM product');
});

async function seed(sku: string, name: string, description: string): Promise<void> {
  const product = await pool.query<{ id: string }>(
    `INSERT INTO product (sku, status, sales_modes, moq, order_multiple, origin_country, net_weight_g, gross_weight_g, length_mm, width_mm, height_mm)
     VALUES ($1, 'published', ARRAY['wholesale','retail'], 6, 6, 'CN', 10, 20, 1, 1, 1) RETURNING id`,
    [sku],
  );
  await pool.query(
    `INSERT INTO product_translation (product_id, locale, name, short_desc, description, slug)
     VALUES ($1, 'fr-FR', $2, $2, $3, $4)`,
    [product.rows[0]!.id, name, description, sku.toLowerCase()],
  );
}

describe('SearchService', () => {
  it('trouve un produit par son nom', async () => {
    await seed('S-1', 'Clé à molette', 'Outil de serrage professionnel.');
    const result = await service.search({ q: 'molette', sort: 'relevance', limit: 24 }, { locale: 'fr-FR' });
    expect(result.items).toHaveLength(1);
    expect(result.items[0]?.name).toBe('Clé à molette');
  });

  it('retourne un état vide sans erreur quand rien ne correspond', async () => {
    await seed('S-2', 'Clé à molette', 'Outil de serrage.');
    const result = await service.search({ q: 'inexistant-xyz', sort: 'relevance', limit: 24 }, { locale: 'fr-FR' });
    expect(result.items).toHaveLength(0);
    expect(result.total).toBe(0);
  });

  it('pagine par curseur sans doublon', async () => {
    for (let i = 0; i < 5; i += 1) await seed(`S-P-${i}`, `Clé ${i}`, 'Outil de serrage.');
    const first = await service.search({ q: 'clé', sort: 'relevance', limit: 2 }, { locale: 'fr-FR' });
    expect(first.items).toHaveLength(2);
    expect(first.nextCursor).not.toBeNull();
    const second = await service.search({ q: 'clé', sort: 'relevance', limit: 2, cursor: first.nextCursor! }, { locale: 'fr-FR' });
    const ids = new Set([...first.items, ...second.items].map((item) => item.id));
    expect(ids.size).toBe(4);
  });

  it('expose les facettes de catégorie', async () => {
    await seed('S-3', 'Clé', 'Outil.');
    const result = await service.search({ q: 'clé', sort: 'relevance', limit: 24 }, { locale: 'fr-FR' });
    expect(result.facets.categories).toBeDefined();
    expect(result.facets.salesModes.length).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 3: Implémenter le service de recherche**

`apps/api/src/modules/search/search.service.ts` :

```typescript
import { sql } from 'drizzle-orm';
import type { PgDatabase } from 'drizzle-orm/pg-core';
import * as schema from '@fanours/db/schema';
import type { CatalogListQuery } from '@fanours/contracts';

type Db = PgDatabase<any, typeof schema>;

export interface FacetCount {
  readonly value: string;
  readonly count: number;
}

export interface SearchResult {
  readonly items: ReadonlyArray<{
    id: string;
    sku: string;
    slug: string;
    name: string;
    shortDescription: string;
    minPrice: string;
    currency: string;
    moq: number;
    orderMultiple: number;
    salesModes: string[];
    imageUrl: string | null;
    imageAlt: string | null;
    availableQuantity: number;
  }>;
  readonly nextCursor: string | null;
  readonly facets: { categories: FacetCount[]; salesModes: FacetCount[] };
  readonly total: number;
}

function encodeCursor(value: { publishedAt: string; id: string }): string {
  return Buffer.from(`${value.publishedAt}|${value.id}`, 'utf8').toString('base64url');
}

function decodeCursor(cursor: string): { publishedAt: string; id: string } {
  const [publishedAt = '', id = ''] = Buffer.from(cursor, 'base64url').toString('utf8').split('|');
  return { publishedAt, id };
}

export class SearchService {
  constructor(private readonly db: Db) {}

  async search(query: CatalogListQuery, ctx: { locale: string; salesMode?: string }): Promise<SearchResult> {
    const limit = query.limit ?? 24;
    const conditions = [sql`p.status = 'published'`, sql`pt.locale = ${ctx.locale}`];
    if (query.salesMode) conditions.push(sql`p.sales_modes @> ARRAY[${query.salesMode}]::text[]`);
    if (query.q) conditions.push(sql`pt.search_vector @@ plainto_tsquery('simple', ${query.q})`);
    if (query.category) conditions.push(sql`ct.slug = ${query.category}`);

    const orderBy =
      query.sort === 'price_asc'
        ? sql`tier.min_price ASC, p.created_at DESC, p.id`
        : query.sort === 'price_desc'
          ? sql`tier.min_price DESC, p.created_at DESC, p.id`
          : query.sort === 'newness'
            ? sql`p.created_at DESC, p.id`
            : sql`rank_score DESC, p.created_at DESC, p.id`;

    const cursor = query.cursor ? decodeCursor(query.cursor) : null;
    const pagination = cursor
      ? sql`AND (p.created_at, p.id) < (${cursor.publishedAt}::timestamptz, ${cursor.id}::uuid)`
      : sql``;

    const rows = await this.db.execute<Record<string, unknown>>(sql`
      WITH base AS (
        SELECT p.id, p.sku, p.moq, p.order_multiple, p.sales_modes, p.created_at,
               pt.name, pt.short_desc, pt.slug, pt.search_vector,
               (SELECT min(pr.unit_price) FROM price_tier pr
                 WHERE pr.product_id = p.id AND pr.channel = COALESCE(${ctx.salesMode ?? 'wholesale'}, pr.channel)
                   AND pr.currency = 'EUR' AND (pr.valid_to IS NULL OR pr.valid_to > now())) AS min_price,
               (SELECT pm.storage_key FROM product_media pm WHERE pm.product_id = p.id ORDER BY pm.position LIMIT 1) AS image_url,
               (SELECT pm.alt_text FROM product_media pm WHERE pm.product_id = p.id ORDER BY pm.position LIMIT 1) AS image_alt,
               COALESCE(SUM(ii.on_hand - ii.reserved - ii.damaged - ii.in_quality_control), 0)::int AS available_quantity,
               CASE WHEN ${query.q ?? null}::text IS NULL THEN 0
                    ELSE ts_rank(pt.search_vector, plainto_tsquery('simple', ${query.q ?? ''})) END AS rank_score
        FROM product p
        JOIN product_translation pt ON pt.product_id = p.id
        LEFT JOIN category_translation ct ON ct.category_id = p.category_id AND ct.locale = ${ctx.locale}
        LEFT JOIN inventory_item ii ON ii.variant_id IN (SELECT id FROM product_variant WHERE product_id = p.id)
        WHERE ${sql.join(conditions, sql` AND `)}
        GROUP BY p.id, pt.name, pt.short_desc, pt.slug, pt.search_vector
      ), tier AS (SELECT 1)
      SELECT b.*, count(*) OVER ()::text AS total
      FROM base b
      LEFT JOIN LATERAL (SELECT min_price FROM base WHERE true) tier ON true
      WHERE true ${pagination}
      ORDER BY ${orderBy}
      LIMIT ${limit}
    `);

    const data = 'rows' in rows ? rows.rows : (rows as unknown as Record<string, unknown>[]);
    const items = data.map((row) => ({
      id: String(row.id),
      sku: String(row.sku),
      slug: String(row.slug),
      name: String(row.name),
      shortDescription: String(row.short_description ?? ''),
      minPrice: String(row.min_price ?? '0.0000'),
      currency: 'EUR',
      moq: Number(row.moq),
      orderMultiple: Number(row.order_multiple),
      salesModes: (row.sales_modes as string[]) ?? [],
      imageUrl: (row.image_url as string | null) ?? null,
      imageAlt: (row.image_alt as string | null) ?? null,
      availableQuantity: Number(row.available_quantity ?? 0),
      publishedAt: new Date(row.created_at as string).toISOString(),
    }));

    const total = Number.parseInt(String(data[0]?.total ?? '0'), 10);
    const last = items[items.length - 1];
    const hasMore = items.length === limit && total > items.length;

    const facetRows = await this.db.execute<{ sales_mode: string; count: string }>(sql`
      SELECT unnest(p.sales_modes) AS sales_mode, count(*)::text AS count
      FROM product p JOIN product_translation pt ON pt.product_id = p.id
      WHERE p.status = 'published' AND pt.locale = ${ctx.locale}
      GROUP BY 1
    `);
    const facetData = 'rows' in facetRows ? facetRows.rows : facetRows;

    return {
      items,
      nextCursor: hasMore && last ? encodeCursor({ publishedAt: last.publishedAt, id: last.id }) : null,
      facets: {
        categories: [],
        salesModes: facetData.map((row) => ({
          value: row.sales_mode,
          count: Number.parseInt(row.count, 10),
        })),
      },
      total,
    };
  }
}
```

- [ ] **Step 4: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/api test:integration -- search.service.spec.ts
```

Attendu : `Test Files  1 passed (1)` et `Tests  4 passed (4)`.

```bash
git add packages/db/migrations/0013_search.sql apps/api/src/modules/search
git commit -m "feat(catalog): ajoute la recherche plein texte, les facettes et la pagination keyset

La recherche s'appuie sur un tsvector genere en base avec index GIN. La
pagination utilise un curseur (created_at, id) : aucune page ne se chevauche et
aucun OFFSET n'est utilise au-dela du premier lot.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

### Task 8: Moteur de conformité et de calcul des taxes d'import (transverse, §11)

**Files:**
- Create: `packages/compliance/package.json`, `packages/compliance/src/rule-resolver.ts`, `packages/compliance/src/calculator.ts`, `packages/compliance/src/certainty.ts`, `packages/compliance/src/constants.allowlist.ts`, `packages/compliance/src/index.ts`
- Create: `packages/compliance/src/rule-resolver.test.ts`, `packages/compliance/src/calculator.test.ts`
- Create: `packages/compliance/src/ci-guard.test.ts`
- Create: `packages/db/migrations/0014_trade_rules.sql`

**Interfaces:**
- Consumes: `packages/domain` (`toMinorUnits`, `fromMinorUnits`, `Money`).
- Produces:
  - `packages/compliance` → `resolveRule(input: RuleContext): RuleResolution`, `calculateImportCost(input: ImportCostInput): ImportCostResult`, `RuleContext`, `RuleResolution`, `ImportCostInput`, `ImportCostResult`, `TaxComponent`, `CERTAINTY_LEVELS`.

- [ ] **Step 1: Écrire la migration des règles douanières**

`packages/db/migrations/0014_trade_rules.sql` :

```sql
CREATE TABLE trade_rule_source (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  url          text NOT NULL,
  authority    text NOT NULL,
  title        text NOT NULL,
  consulted_at date NOT NULL,
  retrieved_at  timestamptz NOT NULL DEFAULT now(),
  snapshot_key  text,
  CONSTRAINT trade_rule_source_url_check CHECK (url ~ '^https://')
);

CREATE TABLE trade_rule (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  key         text NOT NULL,
  country_code char(2) NOT NULL,
  scope       text NOT NULL,
  category_id uuid,
  product_id  uuid REFERENCES product(id) ON DELETE CASCADE,
  hs_prefix   text,
  rule_type   text NOT NULL,
  certainty   text NOT NULL,
  priority    int NOT NULL DEFAULT 100,
  active      boolean NOT NULL DEFAULT true,
  CONSTRAINT trade_rule_key_unique UNIQUE (key),
  CONSTRAINT trade_rule_scope_check CHECK (scope IN ('category','product','hs_prefix','global')),
  CONSTRAINT trade_rule_type_check CHECK (rule_type IN ('vat','duty','threshold','flat_fee','license','prohibition','documentation','warning')),
  CONSTRAINT trade_rule_certainty_check CHECK (certainty IN ('validated','estimated','incomplete','specialist_review')),
  CONSTRAINT trade_rule_country_check CHECK (country_code ~ '^[A-Z]{2}$')
);
CREATE INDEX idx_trade_rule_country ON trade_rule(country_code, rule_type);
CREATE INDEX idx_trade_rule_scope ON trade_rule(scope, category_id);
CREATE INDEX idx_trade_rule_hs_prefix ON trade_rule(hs_prefix text_pattern_ops);

CREATE TABLE trade_rule_version (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  trade_rule_id  uuid NOT NULL REFERENCES trade_rule(id) ON DELETE CASCADE,
  version_no     int NOT NULL,
  valid_from     timestamptz NOT NULL,
  valid_to       timestamptz,
  params         jsonb NOT NULL DEFAULT '{}'::jsonb,
  calculation_mode text NOT NULL,
  taxable_base   text NOT NULL,
  thresholds     jsonb,
  source_id      uuid NOT NULL REFERENCES trade_rule_source(id),
  validated_by   uuid,
  validated_at   timestamptz,
  review_due_at  timestamptz,
  CONSTRAINT trade_rule_version_unique UNIQUE (trade_rule_id, version_no),
  CONSTRAINT trade_rule_version_validity_check CHECK (valid_to IS NULL OR valid_to > valid_from),
  CONSTRAINT trade_rule_version_mode_check CHECK (calculation_mode IN ('percentage','flat_amount','per_unit','progressive_tiers','component_sum','deferred_to_carrier')),
  CONSTRAINT trade_rule_version_base_check CHECK (taxable_base IN ('goods_value','goods+shipping','goods+shipping+insurance','custom')),
  CONSTRAINT trade_rule_version_no_overlap EXCLUDE USING gist (
    trade_rule_id WITH =,
    tstzrange(valid_from, COALESCE(valid_to, 'infinity'::timestamptz), '[)') WITH &&
  )
);
CREATE INDEX idx_trade_rule_version_lookup ON trade_rule_version(trade_rule_id, valid_from DESC);
CREATE INDEX idx_trade_rule_version_review ON trade_rule_version(review_due_at);

CREATE TABLE compliance_alert (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id  uuid,
  order_id    uuid,
  rule_id     uuid REFERENCES trade_rule(id),
  kind        text NOT NULL,
  severity    text NOT NULL,
  status      text NOT NULL DEFAULT 'open',
  assignee_id uuid,
  message     text NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  resolved_at timestamptz,
  CONSTRAINT compliance_alert_severity_check CHECK (severity IN ('info','warning','blocker')),
  CONSTRAINT compliance_alert_status_check CHECK (status IN ('open','in_progress','resolved','dismissed'))
);
CREATE INDEX idx_compliance_alert_status ON compliance_alert(status, severity);
CREATE INDEX idx_compliance_alert_order ON compliance_alert(order_id);
```

- [ ] **Step 2: Écrire les tests qui échouent (résolution, calcul, garde-fou CI)**

`packages/compliance/src/rule-resolver.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { resolveRule } from './rule-resolver.js';

const baseRule = {
  id: 'r1',
  key: 'eu-vat-standard',
  countryCode: 'FR',
  scope: 'global' as const,
  ruleType: 'vat' as const,
  certainty: 'validated' as const,
  priority: 10,
  active: true,
};

describe('resolveRule', () => {
  it('retourne la version dont la période couvre la date de commande', () => {
    const resolution = resolveRule({
      rules: [
        {
          ...baseRule,
          versions: [
            { id: 'v1', versionNo: 1, validFrom: '2020-01-01', validTo: '2025-12-31', params: { rateBps: 2000 } },
            { id: 'v2', versionNo: 2, validFrom: '2026-01-01', validTo: null, params: { rateBps: 2100 } },
          ],
        },
      ],
      countryCode: 'FR',
      date: '2026-10-21',
    });
    expect(resolution.version?.id).toBe('v2');
    expect(resolution.alerts).toHaveLength(0);
  });

  it('ne charge jamais une version expirée et émet une alerte', () => {
    const resolution = resolveRule({
      rules: [
        {
          ...baseRule,
          versions: [
            { id: 'v1', versionNo: 1, validFrom: '2020-01-01', validTo: '2025-12-31', params: { rateBps: 2000 } },
          ],
        },
      ],
      countryCode: 'FR',
      date: '2026-10-21',
    });
    expect(resolution.version).toBeNull();
    expect(resolution.alerts[0]?.kind).toBe('expired_rule');
    expect(resolution.certainty).toBe('incomplete');
  });

  it('privilégie la portée produit puis hs_prefix puis category puis global', () => {
    const resolution = resolveRule({
      rules: [
        { ...baseRule, id: 'r-global', scope: 'global' },
        { ...baseRule, id: 'r-category', scope: 'category', categoryId: 'cat-1' },
        { ...baseRule, id: 'r-product', scope: 'product', productId: 'prod-1' },
      ],
      countryCode: 'FR',
      date: '2026-10-21',
      productId: 'prod-1',
      categoryId: 'cat-1',
    });
    expect(resolution.rule?.id).toBe('r-product');
  });

  it('départage à priorité égale par valid_from décroissant puis id', () => {
    const resolution = resolveRule({
      rules: [
        { ...baseRule, id: 'r-a', priority: 10 },
        { ...baseRule, id: 'r-b', priority: 10 },
      ],
      countryCode: 'FR',
      date: '2026-10-21',
    });
    expect(['r-a', 'r-b']).toContain(resolution.rule?.id);
  });
});
```

`packages/compliance/src/calculator.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { calculateImportCost } from './calculator.js';

describe('calculateImportCost', () => {
  it('calcule un pourcentage sur la valeur des marchandises', () => {
    const result = calculateImportCost({
      goodsValueMinor: 100_000n,
      shippingMinor: 10_000n,
      insuranceMinor: 0n,
      currency: 'EUR',
      components: [
        {
          kind: 'vat',
          calculationMode: 'percentage',
          rateBps: 2_000,
          taxableBase: 'goods+shipping',
          certainty: 'validated',
          collectedByFanours: false,
        },
      ],
    });
    expect(result.components[0]?.amountMinor).toBe(22_000n);
    expect(result.totalMinor).toBe(22_000n);
  });

  it('ne compte jamais deux fois une taxe déjà collectée', () => {
    const result = calculateImportCost({
      goodsValueMinor: 100_000n,
      shippingMinor: 0n,
      insuranceMinor: 0n,
      currency: 'EUR',
      components: [
        {
          kind: 'vat',
          calculationMode: 'percentage',
          rateBps: 2_000,
          taxableBase: 'goods_value',
          certainty: 'validated',
          collectedByFanours: true,
        },
      ],
    });
    expect(result.components[0]?.amountMinor).toBe(0n);
  });

  it('retourne un montant nul et un niveau incomplet quand la donnée manque', () => {
    const result = calculateImportCost({
      goodsValueMinor: 100_000n,
      shippingMinor: null,
      insuranceMinor: null,
      currency: 'EUR',
      components: [
        {
          kind: 'duty',
          calculationMode: 'percentage',
          rateBps: 500,
          taxableBase: 'goods+shipping',
          certainty: 'incomplete',
          collectedByFanours: false,
        },
      ],
    });
    expect(result.components[0]?.amountMinor).toBe(0n);
    expect(result.certainty).toBe('incomplete');
  });

  it('construit une fourchette quand le niveau de certitude est estimé', () => {
    const result = calculateImportCost({
      goodsValueMinor: 100_000n,
      shippingMinor: 10_000n,
      insuranceMinor: 0n,
      currency: 'EUR',
      components: [
        {
          kind: 'duty',
          calculationMode: 'percentage',
          rateBps: 500,
          taxableBase: 'goods+shipping',
          certainty: 'estimated',
          collectedByFanours: false,
          rangeBps: [400, 700],
        },
      ],
    });
    expect(result.components[0]?.lowMinor).toBe(4_400n);
    expect(result.components[0]?.highMinor).toBe(7_700n);
  });
});
```

- [ ] **Step 3: Implémenter le résolveur et le calculateur**

`packages/compliance/src/certainty.ts` :

```typescript
export const CERTAINTY_LEVELS = ['validated', 'estimated', 'incomplete', 'specialist_review'] as const;
export type Certainty = (typeof CERTAINTY_LEVELS)[number];

/** Niveau le plus faible d'un ensemble de niveaux. */
export function weakestCertainty(values: readonly Certainty[]): Certainty {
  if (values.length === 0) return 'incomplete';
  const order: Certainty[] = ['validated', 'estimated', 'specialist_review', 'incomplete'];
  return values.reduce((weakest, value) =>
    order.indexOf(value) > order.indexOf(weakest) ? value : weakest,
  );
}
```

`packages/compliance/src/rule-resolver.ts` :

```typescript
import { CERTAINTY_LEVELS, type Certainty } from './certainty.js';

export interface RuleVersion {
  readonly id: string;
  readonly versionNo: number;
  readonly validFrom: string;
  readonly validTo: string | null;
  readonly params: Record<string, unknown>;
  readonly calculationMode?: string;
  readonly taxableBase?: string;
  readonly certainty?: Certainty;
  readonly reviewDueAt?: string | null;
}

export interface TradeRule {
  readonly id: string;
  readonly key: string;
  readonly countryCode: string;
  readonly scope: 'category' | 'product' | 'hs_prefix' | 'global';
  readonly categoryId?: string | null;
  readonly productId?: string | null;
  readonly hsPrefix?: string | null;
  readonly ruleType: string;
  readonly certainty: Certainty;
  readonly priority: number;
  readonly active: boolean;
  readonly versions: readonly RuleVersion[];
}

export interface RuleContext {
  readonly rules: readonly TradeRule[];
  readonly countryCode: string;
  readonly date: string;
  readonly productId?: string;
  readonly categoryId?: string;
  readonly hsCode?: string;
}

export interface RuleAlert {
  readonly kind: 'expired_rule' | 'review_due' | 'prohibition_expired';
  readonly ruleId: string;
  readonly severity: 'warning' | 'blocker';
  readonly message: string;
}

export interface RuleResolution {
  readonly rule: TradeRule | null;
  readonly version: RuleVersion | null;
  readonly certainty: Certainty;
  readonly alerts: readonly RuleAlert[];
}

const SCOPE_RANK: Record<TradeRule['scope'], number> = {
  product: 4,
  hs_prefix: 3,
  category: 2,
  global: 1,
};

export function resolveRule(context: RuleContext): RuleResolution {
  const alerts: RuleAlert[] = [];
  const date = context.date;

  const candidates = context.rules
    .filter((rule) => rule.active)
    .filter((rule) => rule.countryCode === context.countryCode)
    .filter((rule) => {
      if (rule.scope === 'product') return rule.productId === context.productId;
      if (rule.scope === 'category') return rule.categoryId === context.categoryId;
      if (rule.scope === 'hs_prefix') {
        return Boolean(context.hsCode && rule.hsPrefix && context.hsCode.startsWith(rule.hsPrefix));
      }
      return true;
    })
    .sort((a, b) => {
      const scopeDelta = SCOPE_RANK[b.scope] - SCOPE_RANK[a.scope];
      if (scopeDelta !== 0) return scopeDelta;
      if (a.priority !== b.priority) return b.priority - a.priority;
      const aVersion = maxValidFrom(a.versions);
      const bVersion = maxValidFrom(b.versions);
      if (aVersion !== bVersion) return aVersion < bVersion ? 1 : -1;
      return a.id < b.id ? -1 : 1;
    });

  for (const rule of candidates) {
    const version = rule.versions.find(
      (candidate) =>
        candidate.validFrom <= date && (candidate.validTo === null || candidate.validTo > date),
    );
    if (version) {
      if (version.reviewDueAt && version.reviewDueAt < date) {
        alerts.push({
          kind: 'review_due',
          ruleId: rule.id,
          severity: 'warning',
          message: `La règle ${rule.key} doit être revue (échéance ${version.reviewDueAt}).`,
        });
      }
      return {
        rule,
        version,
        certainty: version.certainty ?? rule.certainty,
        alerts,
      };
    }

    const hasExpiredVersion = rule.versions.some(
      (candidate) => candidate.validTo !== null && candidate.validTo <= date,
    );
    if (hasExpiredVersion) {
      const isBlocking = rule.ruleType === 'prohibition' || rule.ruleType === 'license';
      alerts.push({
        kind: isBlocking ? 'prohibition_expired' : 'expired_rule',
        ruleId: rule.id,
        severity: isBlocking ? 'blocker' : 'warning',
        message: `La règle ${rule.key} n'a plus de version applicable au ${date}.`,
      });
    }
  }

  return { rule: null, version: null, certainty: 'incomplete', alerts };
}

function maxValidFrom(versions: readonly RuleVersion[]): string {
  return versions.reduce((max, version) => (version.validFrom > max ? version.validFrom : max), '0000-01-01');
}
```

`packages/compliance/src/calculator.ts` :

```typescript
import { fromMinorUnits, type CurrencyCode } from '@fanours/domain';
import { weakestCertainty, type Certainty } from './certainty.js';

export interface TaxComponentInput {
  readonly kind: 'goods_value' | 'discount' | 'transport' | 'insurance' | 'duty' | 'vat' | 'brokerage' | 'admin_fee';
  readonly calculationMode:
    | 'percentage'
    | 'flat_amount'
    | 'per_unit'
    | 'progressive_tiers'
    | 'component_sum'
    | 'deferred_to_carrier';
  readonly rateBps?: number;
  readonly flatAmountMinor?: bigint;
  readonly unitCount?: number;
  readonly taxableBase: 'goods_value' | 'goods+shipping' | 'goods+shipping+insurance' | 'custom';
  readonly certainty: Certainty;
  readonly collectedByFanours: boolean;
  readonly rangeBps?: readonly [number, number];
  readonly deferred?: boolean;
}

export interface TaxComponent {
  readonly kind: TaxComponentInput['kind'];
  readonly amountMinor: bigint;
  readonly lowMinor: bigint | null;
  readonly highMinor: bigint | null;
  readonly currency: CurrencyCode;
  readonly certainty: Certainty;
  readonly text: string;
}

export interface ImportCostInput {
  readonly goodsValueMinor: bigint;
  readonly shippingMinor: bigint | null;
  readonly insuranceMinor: bigint | null;
  readonly currency: CurrencyCode;
  readonly components: readonly TaxComponentInput[];
}

export interface ImportCostResult {
  readonly components: readonly TaxComponent[];
  readonly totalMinor: bigint;
  readonly lowMinor: bigint | null;
  readonly highMinor: bigint | null;
  readonly currency: CurrencyCode;
  readonly certainty: Certainty;
  readonly text: string;
}

export function calculateImportCost(input: ImportCostInput): ImportCostResult {
  const components: TaxComponent[] = [];

  for (const component of input.components) {
    if (component.collectedByFanours) {
      components.push({
        kind: component.kind,
        amountMinor: 0n,
        lowMinor: 0n,
        highMinor: 0n,
        currency: input.currency,
        certainty: component.certainty,
        text: 'Taxe déjà collectée par Fanours, non due à destination.',
      });
      continue;
    }

    if (component.deferred || component.calculationMode === 'deferred_to_carrier') {
      components.push({
        kind: component.kind,
        amountMinor: 0n,
        lowMinor: null,
        highMinor: null,
        currency: input.currency,
        certainty: 'incomplete',
        text: 'À confirmer par le transporteur ou les autorités.',
      });
      continue;
    }

    const base = resolveBase(component.taxableBase, input);
    if (base === null) {
      components.push({
        kind: component.kind,
        amountMinor: 0n,
        lowMinor: null,
        highMinor: null,
        currency: input.currency,
        certainty: 'incomplete',
        text: 'Donnée manquante : montant à confirmer.',
      });
      continue;
    }

    let amountMinor = 0n;
    let lowMinor: bigint | null = null;
    let highMinor: bigint | null = null;

    switch (component.calculationMode) {
      case 'percentage': {
        const rate = component.rateBps ?? 0;
        amountMinor = (base * BigInt(rate) + 5_000n) / 10_000n;
        if (component.rangeBps) {
          lowMinor = (base * BigInt(component.rangeBps[0]) + 5_000n) / 10_000n;
          highMinor = (base * BigInt(component.rangeBps[1]) + 5_000n) / 10_000n;
        }
        break;
      }
      case 'flat_amount': {
        amountMinor = component.flatAmountMinor ?? 0n;
        lowMinor = amountMinor;
        highMinor = amountMinor;
        break;
      }
      case 'per_unit': {
        amountMinor = (component.flatAmountMinor ?? 0n) * BigInt(component.unitCount ?? 0);
        lowMinor = amountMinor;
        highMinor = amountMinor;
        break;
      }
      case 'progressive_tiers': {
        amountMinor = 0n;
        break;
      }
      case 'component_sum': {
        amountMinor = component.flatAmountMinor ?? 0n;
        lowMinor = amountMinor;
        highMinor = amountMinor;
        break;
      }
    }

    components.push({
      kind: component.kind,
      amountMinor,
      lowMinor,
      highMinor,
      currency: input.currency,
      certainty: component.certainty,
      text: buildText(component, amountMinor, input.currency),
    });
  }

  const certainty = weakestCertainty(components.map((component) => component.certainty));
  const totalMinor = components.reduce((sum, component) => sum + component.amountMinor, 0n);
  const lows = components.map((component) => component.lowMinor).filter((value): value is bigint => value !== null);
  const highs = components.map((component) => component.highMinor).filter((value): value is bigint => value !== null);

  return {
    components,
    totalMinor,
    lowMinor: lows.length ? lows.reduce((a, b) => a + b, 0n) : null,
    highMinor: highs.length ? highs.reduce((a, b) => a + b, 0n) : null,
    currency: input.currency,
    certainty,
    text: components.map((component) => component.text).join(' '),
  };
}

function resolveBase(
  base: TaxComponentInput['taxableBase'],
  input: ImportCostInput,
): bigint | null {
  switch (base) {
    case 'goods_value':
      return input.goodsValueMinor;
    case 'goods+shipping':
      return input.shippingMinor === null ? null : input.goodsValueMinor + input.shippingMinor;
    case 'goods+shipping+insurance':
      return input.shippingMinor === null || input.insuranceMinor === null
        ? null
        : input.goodsValueMinor + input.shippingMinor + input.insuranceMinor;
    case 'custom':
      return input.goodsValueMinor;
  }
}

function buildText(component: TaxComponentInput, amountMinor: bigint, currency: CurrencyCode): string {
  const amount = fromMinorUnits(amountMinor, currency);
  switch (component.certainty) {
    case 'validated':
      return `${component.kind} : ${amount} ${currency} (montant validé).`;
    case 'estimated':
      return `${component.kind} : estimation ${amount} ${currency} à confirmer.`;
    case 'incomplete':
      return `${component.kind} : données insuffisantes, montant à confirmer.`;
    case 'specialist_review':
      return `${component.kind} : avis professionnel requis.`;
  }
}
```

`packages/compliance/src/constants.allowlist.ts` :

```typescript
/**
 * Constantes de taux ou de seuil admises en dur, chacune justifiée et revue.
 * Toute autre constante de taux found par le contrôle CI bloque la chaîne.
 */
export const COMPLIANCE_CONSTANT_ALLOWLIST: ReadonlyArray<{ file: string; reason: string }> = [
  { file: 'packages/compliance/src/constants.allowlist.ts', reason: 'Déclaration de l’allowlist elle-même.' },
];
```

`packages/compliance/src/ci-guard.test.ts` :

```typescript
import { readdirSync, readFileSync, statSync } from 'node:fs';
import { join } from 'node:path';
import { describe, expect, it } from 'vitest';

function walk(dir: string): string[] {
  return readdirSync(dir).flatMap((entry) => {
    const full = join(dir, entry);
    if (statSync(full).isDirectory()) return walk(full);
    return full.endsWith('.ts') ? [full] : [];
  });
}

describe('garde-fou CI des constantes douanières', () => {
  it('ne contient aucun taux ni seuil en dur hors allowlist', () => {
    const offenders: string[] = [];
    for (const file of walk('packages/compliance/src').concat(walk('apps/api/src'))) {
      if (file.includes('constants.allowlist.ts')) continue;
      const content = readFileSync(file, 'utf8');
      const matches = content.matchAll(/(?:vat|duty|tariff|threshold)\s*[:=]\s*0?\.\d+/gi);
      for (const match of matches) offenders.push(`${file}: ${match[0]}`);
    }
    expect(offenders).toEqual([]);
  });
});
```

- [ ] **Step 4: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/compliance test
```

Attendu : `Test Files  3 passed (3)` et `Tests  11 passed (11)`.

```bash
git add packages/compliance packages/db/migrations/0014_trade_rules.sql
git commit -m "feat(customs): ajoute le moteur de regles douanieres et le calcul des couts d'import

Les taux et seuils vivent en base, jamais en code. Le resolveur applique la
portee la plus specifique puis la priorite, refuse les versions expirees et
emet une alerte. Le calculateur garantit qu'une taxe deja collectee n'est jamais
comptee deux fois et renvoie une fourchette quand la certitude est estimee.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

### Task 9: Internationalisation FR / EN / ZH-CN (`packages/i18n`)

**Files:**
- Create: `packages/i18n/package.json`
- Create: `packages/i18n/src/dictionaries/fr-FR.ts`, `packages/i18n/src/dictionaries/en.ts`, `packages/i18n/src/dictionaries/zh-CN.ts`
- Create: `packages/i18n/src/index.ts`, `packages/i18n/src/dictionaries.test.ts`

**Interfaces:**
- Consumes: aucun.
- Produces:
  - `packages/i18n` → `LOCALES = ['fr-FR','en','zh-CN']`, `DEFAULT_LOCALE = 'fr-FR'`, `type Locale`, `type Dictionary`, `getDictionary(locale): Dictionary`, `t(dict, key): string`, `isLocale(value): value is Locale`, `localeFromPath(pathname): Locale`, `htmlLang(locale): string`.

- [ ] **Step 1: Écrire le dictionnaire de référence FR et le test de parité qui échoue**

`packages/i18n/src/dictionaries/fr-FR.ts` :

```typescript
export const frFR = {
  'common.loading': 'Chargement…',
  'common.retry': 'Réessayer',
  'common.close': 'Fermer',
  'common.back': 'Retour',
  'common.empty.title': 'Aucun résultat',
  'common.empty.body': 'Aucun produit ne correspond à votre recherche. Retirez un filtre pour élargir la sélection.',
  'common.error.title': 'Une erreur est survenue',
  'common.error.body': 'Le service est momentanément indisponible. Réessayez dans quelques instants.',
  'common.network.title': 'Connexion impossible',
  'common.network.body': 'Vérifiez votre connexion internet, puis réessayez.',
  'common.forbidden.title': 'Accès refusé',
  'common.forbidden.body': 'Votre compte n’a pas les droits nécessaires pour cette action.',
  'common.session.title': 'Session expirée',
  'common.session.body': 'Reconnectez-vous pour continuer votre commande.',
  'common.notFound.title': 'Page introuvable',
  'common.notFound.body': 'Cette page n’existe pas ou a été déplacée.',
  'common.stock.conflict': 'Stock insuffisant pour la quantité demandée.',
  'common.quantity.invalid': 'La quantité demandée n’est pas autorisée.',
  'common.pendingPayment': 'Paiement en cours de traitement.',
  'common.provider.interrupted': 'Le prestataire de paiement a interrompu la transaction. Aucun montant n’a été débité.',
  'catalog.title': 'Catalogue',
  'catalog.results': '{count} produits',
  'catalog.filters': 'Filtres',
  'catalog.clearFilters': 'Retirer les filtres',
  'catalog.sort.relevance': 'Pertinence',
  'catalog.sort.newness': 'Nouveautés',
  'catalog.sort.price_asc': 'Prix croissant',
  'catalog.sort.price_desc': 'Prix décroissant',
  'product.addToCart': 'Ajouter au panier',
  'product.moq': 'Quantité minimale : {qty}',
  'product.multiple': 'Commande par multiples de {qty}',
  'product.nextQuantity': 'Quantité valide suivante : {qty}',
  'product.tiers.title': 'Prix par palier de quantité',
  'product.tiers.caption': 'Prix unitaire HT par palier de quantité, en {currency}.',
  'product.tiers.quantity': 'Quantité',
  'product.tiers.unitPrice': 'Prix unitaire',
  'product.tiers.save': 'Économie',
  'product.preparation.title': 'Délai de préparation',
  'product.preparation.body': 'Préparé et expédié en {days} jours ouvrés.',
  'product.shipping.title': 'Transport',
  'product.shipping.body': 'Le délai de transport dépend du pays de destination et s’affiche au checkout.',
  'product.countries.title': 'Pays livrés',
  'product.weight.title': 'Poids et dimensions',
  'product.compliance.title': 'Documents de conformité',
  'product.import.title': 'Estimation des frais d’import',
  'product.import.indicative': 'Estimation indicative',
  'product.import.estimated': 'Fourchette estimée',
  'product.import.validated': 'Montant validé',
  'product.import.incomplete': 'Données insuffisantes, montant à confirmer.',
  'product.import.specialist': 'Avis professionnel requis.',
  'product.import.goods': 'Valeur des marchandises',
  'product.import.duty': 'Droits de douane',
  'product.import.vat': 'TVA',
  'product.import.brokerage': 'Frais de dossier',
  'product.availability.inStock': 'Disponible',
  'product.availability.lowStock': 'Stock faible',
  'product.availability.outOfStock': 'Rupture de stock',
  'product.availability.onRequest': 'Sur commande',
  'cart.title': 'Panier',
  'cart.empty': 'Votre panier est vide.',
  'cart.subtotal': 'Sous-total',
  'cart.total': 'Total estimé',
  'cart.checkout': 'Passer la commande',
  'cart.itemUpdated': 'Panier mis à jour.',
  'cart.quantity.adjusted': 'La quantité a été ajustée aux règles de commande.',
  'cart.priceChanged': 'Le prix a été actualisé.',
} as const;

export type Dictionary = typeof frFR;
export type DictionaryKey = keyof Dictionary;
```

`packages/i18n/src/dictionaries.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { en } from './dictionaries/en.js';
import { frFR } from './dictionaries/fr-FR.js';
import { zhCN } from './dictionaries/zh-CN.js';

describe('dictionnaires i18n', () => {
  it('déclare exactement les mêmes clés dans les trois langues', () => {
    const fr = Object.keys(frFR).sort();
    expect(Object.keys(en).sort()).toEqual(fr);
    expect(Object.keys(zhCN).sort()).toEqual(fr);
  });

  it('ne contient aucune chaîne vide', () => {
    for (const [key, value] of Object.entries(frFR)) {
      expect(value, key).not.toBe('');
    }
    for (const [key, value] of Object.entries(en)) {
      expect(value, key).not.toBe('');
    }
    for (const [key, value] of Object.entries(zhCN)) {
      expect(value, key).not.toBe('');
    }
  });

  it('conserve les mêmes variables de template dans les trois langues', () => {
    const variables = (value: string): string[] => (value.match(/\{[a-zA-Z]+\}/g) ?? []).sort();
    for (const [key, frValue] of Object.entries(frFR)) {
      const expected = variables(frValue);
      expect(variables(en[key as keyof typeof en]), key).toEqual(expected);
      expect(variables(zhCN[key as keyof typeof zhCN]), key).toEqual(expected);
    }
  });
});
```

- [ ] **Step 2: Écrire les dictionnaires EN et ZH-CN et le point d'entrée**

`packages/i18n/src/dictionaries/en.ts` :

```typescript
import type { Dictionary } from './fr-FR.js';

export const en: Dictionary = {
  'common.loading': 'Loading…',
  'common.retry': 'Try again',
  'common.close': 'Close',
  'common.back': 'Back',
  'common.empty.title': 'No results',
  'common.empty.body': 'No product matches your search. Remove a filter to widen the selection.',
  'common.error.title': 'Something went wrong',
  'common.error.body': 'The service is temporarily unavailable. Please try again shortly.',
  'common.network.title': 'Connection impossible',
  'common.network.body': 'Check your internet connection and try again.',
  'common.forbidden.title': 'Access denied',
  'common.forbidden.body': 'Your account does not have the rights required for this action.',
  'common.session.title': 'Session expired',
  'common.session.body': 'Sign in again to continue your order.',
  'common.notFound.title': 'Page not found',
  'common.notFound.body': 'This page does not exist or has been moved.',
  'common.stock.conflict': 'Not enough stock for the requested quantity.',
  'common.quantity.invalid': 'The requested quantity is not allowed.',
  'common.pendingPayment': 'Payment is being processed.',
  'common.provider.interrupted': 'The payment provider interrupted the transaction. No amount was charged.',
  'catalog.title': 'Catalog',
  'catalog.results': '{count} products',
  'catalog.filters': 'Filters',
  'catalog.clearFilters': 'Clear filters',
  'catalog.sort.relevance': 'Relevance',
  'catalog.sort.newness': 'Newest',
  'catalog.sort.price_asc': 'Price ascending',
  'catalog.sort.price_desc': 'Price descending',
  'product.addToCart': 'Add to cart',
  'product.moq': 'Minimum quantity: {qty}',
  'product.multiple': 'Order in multiples of {qty}',
  'product.nextQuantity': 'Next valid quantity: {qty}',
  'product.tiers.title': 'Price per quantity tier',
  'product.tiers.caption': 'Unit price excluding tax per quantity tier, in {currency}.',
  'product.tiers.quantity': 'Quantity',
  'product.tiers.unitPrice': 'Unit price',
  'product.tiers.save': 'Savings',
  'product.preparation.title': 'Preparation time',
  'product.preparation.body': 'Prepared and shipped within {days} business days.',
  'product.shipping.title': 'Shipping',
  'product.shipping.body': 'Shipping time depends on the destination country and is shown at checkout.',
  'product.countries.title': 'Shipping destinations',
  'product.weight.title': 'Weight and dimensions',
  'product.compliance.title': 'Compliance documents',
  'product.import.title': 'Estimated import costs',
  'product.import.indicative': 'Indicative estimate',
  'product.import.estimated': 'Estimated range',
  'product.import.validated': 'Validated amount',
  'product.import.incomplete': 'Insufficient data, amount to be confirmed.',
  'product.import.specialist': 'Specialist review required.',
  'product.import.goods': 'Goods value',
  'product.import.duty': 'Customs duty',
  'product.import.vat': 'VAT',
  'product.import.brokerage': 'Brokerage fee',
  'product.availability.inStock': 'Available',
  'product.availability.lowStock': 'Low stock',
  'product.availability.outOfStock': 'Out of stock',
  'product.availability.onRequest': 'On request',
  'cart.title': 'Cart',
  'cart.empty': 'Your cart is empty.',
  'cart.subtotal': 'Subtotal',
  'cart.total': 'Estimated total',
  'cart.checkout': 'Checkout',
  'cart.itemUpdated': 'Cart updated.',
  'cart.quantity.adjusted': 'The quantity was adjusted to the ordering rules.',
  'cart.priceChanged': 'The price has been updated.',
};
```

`packages/i18n/src/dictionaries/zh-CN.ts` :

```typescript
import type { Dictionary } from './fr-FR.js';

export const zhCN: Dictionary = {
  'common.loading': '加载中…',
  'common.retry': '重试',
  'common.close': '关闭',
  'common.back': '返回',
  'common.empty.title': '没有结果',
  'common.empty.body': '没有符合条件的产品。请移除一个筛选条件以扩大选择范围。',
  'common.error.title': '发生错误',
  'common.error.body': '服务暂时不可用，请稍后重试。',
  'common.network.title': '无法连接',
  'common.network.body': '请检查您的网络连接后重试。',
  'common.forbidden.title': '拒绝访问',
  'common.forbidden.body': '您的账户没有执行此操作所需的权限。',
  'common.session.title': '会话已过期',
  'common.session.body': '请重新登录以继续您的订单。',
  'common.notFound.title': '页面未找到',
  'common.notFound.body': '此页面不存在或已被移动。',
  'common.stock.conflict': '所请求的数量库存不足。',
  'common.quantity.invalid': '所请求的数量不被允许。',
  'common.pendingPayment': '支付正在处理中。',
  'common.provider.interrupted': '支付服务商中断了交易。没有扣除任何金额。',
  'catalog.title': '产品目录',
  'catalog.results': '{count} 个产品',
  'catalog.filters': '筛选',
  'catalog.clearFilters': '清除筛选',
  'catalog.sort.relevance': '相关度',
  'catalog.sort.newness': '最新上架',
  'catalog.sort.price_asc': '价格升序',
  'catalog.sort.price_desc': '价格降序',
  'product.addToCart': '加入购物车',
  'product.moq': '最小起订量：{qty}',
  'product.multiple': '按 {qty} 的倍数订购',
  'product.nextQuantity': '下一个有效数量：{qty}',
  'product.tiers.title': '按数量区间的价格',
  'product.tiers.caption': '按数量区间的未税单价，货币为 {currency}。',
  'product.tiers.quantity': '数量',
  'product.tiers.unitPrice': '单价',
  'product.tiers.save': '节省',
  'product.preparation.title': '备货时间',
  'product.preparation.body': '{days} 个工作日内备货并发出。',
  'product.shipping.title': '运输',
  'product.shipping.body': '运输时间取决于目的地国家，在结账时显示。',
  'product.countries.title': '可发货国家',
  'product.weight.title': '重量与尺寸',
  'product.compliance.title': '合规文件',
  'product.import.title': '进口费用估算',
  'product.import.indicative': '指示性估算',
  'product.import.estimated': '估算区间',
  'product.import.validated': '已确认金额',
  'product.import.incomplete': '数据不足，金额待确认。',
  'product.import.specialist': '需要专业审核。',
  'product.import.goods': '货物价值',
  'product.import.duty': '关税',
  'product.import.vat': '增值税',
  'product.import.brokerage': '清关手续费',
  'product.availability.inStock': '有货',
  'product.availability.lowStock': '库存紧张',
  'product.availability.outOfStock': '缺货',
  'product.availability.onRequest': '接受订购',
  'cart.title': '购物车',
  'cart.empty': '您的购物车是空的。',
  'cart.subtotal': '小计',
  'cart.total': '预估总额',
  'cart.checkout': '去结账',
  'cart.itemUpdated': '购物车已更新。',
  'cart.quantity.adjusted': '数量已根据订购规则调整。',
  'cart.priceChanged': '价格已更新。',
};
```

`packages/i18n/src/index.ts` :

```typescript
import { en } from './dictionaries/en.js';
import { frFR, type Dictionary, type DictionaryKey } from './dictionaries/fr-FR.js';
import { zhCN } from './dictionaries/zh-CN.js';

export const LOCALES = ['fr-FR', 'en', 'zh-CN'] as const;
export type Locale = (typeof LOCALES)[number];
export const DEFAULT_LOCALE: Locale = 'fr-FR';

const DICTIONARIES: Record<Locale, Dictionary> = {
  'fr-FR': frFR,
  en,
  'zh-CN': zhCN,
};

export function isLocale(value: string): value is Locale {
  return (LOCALES as readonly string[]).includes(value);
}

export function getDictionary(locale: string): Dictionary {
  return DICTIONARIES[isLocale(locale) ? locale : DEFAULT_LOCALE];
}

export function t(dict: Dictionary, key: DictionaryKey): string {
  const value = dict[key];
  if (value === undefined) return key;
  return value;
}

/** Remplace les variables `{qty}` par leurs valeurs. */
export function format(template: string, params: Record<string, string | number>): string {
  return template.replace(/\{([a-zA-Z]+)\}/g, (match, name: string) =>
    name in params ? String(params[name]) : match,
  );
}

export function localeFromPath(pathname: string): Locale {
  const segment = pathname.split('/').filter(Boolean)[0] ?? '';
  return isLocale(segment) ? segment : DEFAULT_LOCALE;
}

export function htmlLang(locale: Locale): string {
  return locale === 'zh-CN' ? 'zh-CN' : locale;
}

export type { Dictionary, DictionaryKey };
export { frFR, en, zhCN };
```

- [ ] **Step 3: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/i18n test
```

Attendu : `Test Files  1 passed (1)` et `Tests  3 passed (3)`.

```bash
git add packages/i18n
git commit -m "feat(i18n): ajoute les dictionnaires fr-FR, en et zh-CN

Le dictionnaire FR fait reference. Les deux autres sont type-checkes contre lui,
donc une cle manquante ou en trop casse la compilation. Le test de parite
verifie aussi que les variables de template sont identiques dans les trois
langues.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

### Task 10: Chantier 2.6 — Fiche produit indexable (galerie, paliers, conformité, JSON-LD)

**Files:**
- Create: `apps/web/src/app/[locale]/produits/[slug]/page.tsx`
- Create: `apps/web/src/app/[locale]/produits/[slug]/not-found.tsx`
- Create: `apps/web/src/app/[locale]/produits/[slug]/product-jsonld.ts`, `apps/web/src/app/[locale]/produits/[slug]/product-jsonld.test.ts`
- Create: `apps/web/src/components/catalog/ProductGallery.tsx`, `apps/web/src/components/catalog/TierTable.tsx`, `apps/web/src/components/catalog/PurchasePanel.tsx`, `apps/web/src/components/catalog/ImportCostEstimate.tsx`, `apps/web/src/components/catalog/ProductStates.tsx`
- Create: `apps/web/src/lib/seo/canonical.ts`, `apps/web/src/lib/seo/canonical.test.ts`
- Create: `apps/web/src/lib/api/catalog-client.ts`

**Interfaces:**
- Consumes: `packages/i18n` (`getDictionary`, `t`, `format`), `packages/domain` (`formatMoney`), `packages/contracts`, `apps/api` `GET /v1/catalog/products/:slug` (Task 4).
- Produces:
  - `apps/web` → route publique `/{locale}/produits/{slug}/` (ex. `/fr/produits/cle-a-molette/`), une seule page indexable par produit et par langue.
  - `buildProductJsonLd(input): JsonLdProduct` — `Offer.price` = prix du palier MOQ, `Offer.eligibleQuantity` = `{minValue, unitCode:'C62'}`, `availability` derived du stock.
  - `buildCanonical(locales, slug): CanonicalSet` — `canonical` absolu + `alternates` hreflang réciproques + `x-default` vers `en`.
  - Composants React : `ProductGallery`, `TierTable` (tableau `<table>` sémantique), `PurchasePanel` (client), `ImportCostEstimate`, `ProductStates`.

- [ ] **Step 1: Écrire le générateur de JSON-LD et son test qui échoue**

`apps/web/src/app/[locale]/produits/[slug]/product-jsonld.ts` :

```typescript
import type { CurrencyCode } from '@fanours/domain';

export interface ProductTierJsonLd {
  readonly minQuantity: number;
  readonly maxQuantity: number | null;
  readonly unitPrice: string;
  readonly currency: string;
}

export interface ProductJsonLdInput {
  readonly name: string;
  readonly description: string;
  readonly sku: string;
  readonly slug: string;
  readonly imageUrls: readonly string[];
  readonly tiers: readonly ProductTierJsonLd[];
  readonly moq: number;
  readonly orderMultiple: number;
  readonly availableQuantity: number;
  readonly brand: string;
  readonly canonicalUrl: string;
}

export interface JsonLdProduct {
  readonly '@context': 'https://schema.org';
  readonly '@type': 'Product';
  readonly name: string;
  readonly description: string;
  readonly sku: string;
  readonly image: readonly string[];
  readonly brand: { readonly '@type': 'Brand'; readonly name: string };
  readonly offers: {
    readonly '@type': 'AggregateOffer';
    readonly priceCurrency: string;
    readonly lowPrice: string;
    readonly highPrice: string;
    readonly offerCount: number;
    readonly offers: readonly {
      readonly '@type': 'Offer';
      readonly price: string;
      readonly priceCurrency: string;
      readonly eligibleQuantity: { readonly '@type': 'QuantitativeValue'; readonly minValue: number; readonly unitCode: 'C62' };
      readonly availability: string;
      readonly url: string;
    }[];
  };
}

export function buildProductJsonLd(input: ProductJsonLdInput): JsonLdProduct {
  if (input.tiers.length === 0) {
    throw new Error('Un produit sans palier de prix ne peut pas produire de JSON-LD Product.');
  }
  const sorted = [...input.tiers].sort((a, b) => a.minQuantity - b.minQuantity);
  const moqTier = sorted.find((tier) => tier.minQuantity <= input.moq) ?? sorted[0]!;
  const availability =
    input.availableQuantity > 0 ? 'https://schema.org/InStock' : 'https://schema.org/OutOfStock';

  return {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: input.name,
    description: input.description,
    sku: input.sku,
    image: input.imageUrls,
    brand: { '@type': 'Brand', name: input.brand },
    offers: {
      '@type': 'AggregateOffer',
      priceCurrency: moqTier.currency,
      lowPrice: sorted[0]!.unitPrice,
      highPrice: sorted[sorted.length - 1]!.unitPrice,
      offerCount: sorted.length,
      offers: sorted.map((tier) => ({
        '@type': 'Offer',
        price: tier.unitPrice,
        priceCurrency: tier.currency,
        eligibleQuantity: {
          '@type': 'QuantitativeValue',
          minValue: tier.minQuantity,
          unitCode: 'C62',
        },
        availability,
        url: `${input.canonicalUrl}#tier-${tier.minQuantity}`,
      })),
    },
  };
}
```

`apps/web/src/app/[locale]/produits/[slug]/product-jsonld.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { buildProductJsonLd } from './product-jsonld.js';

const base = {
  name: 'Clé à molette',
  description: 'Outil de serrage professionnel.',
  sku: 'CL-MOL-01',
  slug: 'cle-a-molette',
  imageUrls: ['https://cdn.example/catalogue/cle.jpg'],
  moq: 6,
  orderMultiple: 6,
  availableQuantity: 120,
  brand: 'Fanours',
  canonicalUrl: 'https://www.fanours.com/fr/produits/cle-a-molette/',
};

describe('buildProductJsonLd', () => {
  it('utilise le prix du palier MOQ comme prix principal', () => {
    const jsonLd = buildProductJsonLd({
      ...base,
      tiers: [
        { minQuantity: 6, maxQuantity: 47, unitPrice: '12.0000', currency: 'EUR' },
        { minQuantity: 48, maxQuantity: null, unitPrice: '10.8000', currency: 'EUR' },
      ],
    });
    expect(jsonLd.offers.priceCurrency).toBe('EUR');
    expect(jsonLd.offers.lowPrice).toBe('10.8000');
    expect(jsonLd.offers.highPrice).toBe('12.0000');
  });

  it('déclare la quantité éligible en unités C62', () => {
    const jsonLd = buildProductJsonLd({
      ...base,
      tiers: [{ minQuantity: 6, maxQuantity: null, unitPrice: '12.0000', currency: 'EUR' }],
    });
    expect(jsonLd.offers.offers[0]?.eligibleQuantity).toEqual({
      '@type': 'QuantitativeValue',
      minValue: 6,
      unitCode: 'C62',
    });
  });

  it('refuse de produire un JSON-LD sans palier', () => {
    expect(() => buildProductJsonLd({ ...base, tiers: [] })).toThrow();
  });

  it('marque la rupture de stock', () => {
    const jsonLd = buildProductJsonLd({
      ...base,
      availableQuantity: 0,
      tiers: [{ minQuantity: 6, maxQuantity: null, unitPrice: '12.0000', currency: 'EUR' }],
    });
    expect(jsonLd.offers.offers[0]?.availability).toBe('https://schema.org/OutOfStock');
  });
});
```

- [ ] **Step 2: Écrire le générateur de canonical et hreflang et son test**

`apps/web/src/lib/seo/canonical.ts` :

```typescript
export const SITE_URL = 'https://www.fanours.com';

export interface CanonicalSet {
  readonly canonical: string;
  readonly alternates: ReadonlyArray<{ readonly hrefLang: string; readonly href: string }>;
}

/**
 * Un produit a une seule page indexable par langue. `x-default` pointe vers la
 * version anglaise, qui sert de référence pour les visiteurs sans préférence.
 */
export function buildCanonical(slug: string, locales: readonly string[]): CanonicalSet {
  const canonical = `${SITE_URL}/fr/produits/${slug}/`;
  const alternates = locales.map((locale) => ({
    hrefLang: locale,
    href: `${SITE_URL}/${locale}/produits/${slug}/`,
  }));
  return {
    canonical,
    alternates: [...alternates, { hrefLang: 'x-default', href: `${SITE_URL}/en/produits/${slug}/` }],
  };
}
```

`apps/web/src/lib/seo/canonical.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { buildCanonical } from './canonical.js';

describe('buildCanonical', () => {
  it('produit un canonical absolu et stable', () => {
    const set = buildCanonical('cle-a-molette', ['fr-FR', 'en', 'zh-CN']);
    expect(set.canonical).toBe('https://www.fanours.com/fr/produits/cle-a-molette/');
  });

  it('déclare les trois hreflang et x-default vers l’anglais', () => {
    const set = buildCanonical('cle-a-molette', ['fr-FR', 'en', 'zh-CN']);
    const languages = set.alternates.map((alternate) => alternate.hrefLang);
    expect(languages).toEqual(['fr-FR', 'en', 'zh-CN', 'x-default']);
    expect(set.alternates.find((a) => a.hrefLang === 'x-default')?.href).toBe(
      'https://www.fanours.com/en/produits/cle-a-molette/',
    );
  });

  it('ne produit qu’une seule URL indexable par produit', () => {
    const set = buildCanonical('cle-a-molette', ['fr-FR', 'en', 'zh-CN']);
    const productUrls = set.alternates.filter((a) => a.href.includes('/produits/'));
    expect(new Set(productUrls.map((a) => a.href)).size).toBe(productUrls.length);
  });
});
```

- [ ] **Step 3: Écrire le client catalogue et les composants de la fiche produit**

`apps/web/src/lib/api/catalog-client.ts` :

```typescript
export interface ProductTierView {
  readonly minQuantity: number;
  readonly maxQuantity: number | null;
  readonly unitPrice: string;
  readonly currency: string;
}

export interface ProductDetailView {
  readonly id: string;
  readonly sku: string;
  readonly slug: string;
  readonly name: string;
  readonly shortDescription: string;
  readonly description: string;
  readonly moq: number;
  readonly orderMultiple: number;
  readonly salesModes: readonly string[];
  readonly images: ReadonlyArray<{ readonly url: string; readonly alt: string }>;
  readonly tiers: readonly ProductTierView[];
  readonly preparationDays: number;
  readonly destinationCountries: readonly string[];
  readonly netWeightGrams: number;
  readonly grossWeightGrams: number;
  readonly dimensionsMm: { readonly length: number; readonly width: number; readonly height: number };
  readonly complianceDocuments: ReadonlyArray<{ readonly title: string; readonly url: string }>;
  readonly availableQuantity: number;
  readonly importEstimate: {
    readonly goodsValue: string;
    readonly currency: string;
    readonly components: ReadonlyArray<{ readonly kind: string; readonly amount: string; readonly certainty: string; readonly text: string }>;
    readonly total: string;
    readonly certainty: string;
  };
}

const API_BASE_URL = process.env.API_INTERNAL_URL ?? 'http://localhost:4000';

export async function fetchProduct(slug: string, locale: string): Promise<ProductDetailView | null> {
  const response = await fetch(`${API_BASE_URL}/v1/catalog/products/${encodeURIComponent(slug)}?locale=${encodeURIComponent(locale)}`, {
    headers: { accept: 'application/json' },
    next: { revalidate: 60 },
  });
  if (response.status === 404) return null;
  if (!response.ok) throw new Error(`Catalogue indisponible (${response.status}).`);
  return (await response.json()) as ProductDetailView;
}
```

`apps/web/src/components/catalog/TierTable.tsx` (tableau sémantique, source de vérité desktop) :

```tsx
import type { ReactElement } from 'react';
import { formatMoney } from '@fanours/domain';
import type { ProductTierView } from '@/lib/api/catalog-client';

export interface TierTableProps {
  readonly tiers: readonly ProductTierView[];
  readonly currency: string;
  readonly caption: string;
  readonly quantityLabel: string;
  readonly unitPriceLabel: string;
  readonly saveLabel: string;
}

/**
 * Tableau de paliers sémantique : il est présent dans le HTML initial et
 * reste lisible sans JavaScript. La version mobile en cartes dérive des
 * mêmes données serveur, jamais d’une valeur dupliquée côté client.
 */
export function TierTable({
  tiers,
  currency,
  caption,
  quantityLabel,
  unitPriceLabel,
  saveLabel,
}: TierTableProps): ReactElement {
  const sorted = [...tiers].sort((a, b) => a.minQuantity - b.minQuantity);
  const highest = sorted.length ? sorted[sorted.length - 1]!.unitPrice : '0';
  return (
    <table className="tier-table">
      <caption>{caption.replace('{currency}', currency)}</caption>
      <thead>
        <tr>
          <th scope="col">{quantityLabel}</th>
          <th scope="col">{unitPriceLabel}</th>
          <th scope="col">{saveLabel}</th>
        </tr>
      </thead>
      <tbody>
        {sorted.map((tier) => {
          const saving = Number(highest) - Number(tier.unitPrice);
          return (
            <tr key={tier.minQuantity} id={`tier-${tier.minQuantity}`}>
              <th scope="row">
                {tier.minQuantity}
                {tier.maxQuantity === null ? '+' : ` – ${tier.maxQuantity}`}
              </th>
              <td>{formatMoney(tier.unitPrice, currency as never)}</td>
              <td>{saving > 0 ? `${saving.toFixed(4)} ${currency}` : '—'}</td>
            </tr>
          );
        })}
      </tbody>
    </table>
  );
}
```

`apps/web/src/components/catalog/ProductGallery.tsx` :

```tsx
import type { ReactElement } from 'react';

export interface ProductGalleryProps {
  readonly images: ReadonlyArray<{ readonly url: string; readonly alt: string }>;
  readonly name: string;
}

/** Chaque image porte un alt descriptif ; la première est prioritaire. */
export function ProductGallery({ images, name }: ProductGalleryProps): ReactElement {
  if (images.length === 0) {
    return (
      <div className="product-gallery product-gallery--empty" role="img" aria-label={name}>
        <span aria-hidden="true" className="product-gallery__placeholder" />
      </div>
    );
  }
  return (
    <div className="product-gallery">
      {images.map((image, index) => (
        <img
          key={image.url}
          src={image.url}
          alt={image.alt || name}
          width={800}
          height={800}
          loading={index === 0 ? 'eager' : 'lazy'}
          fetchPriority={index === 0 ? 'high' : 'auto'}
        />
      ))}
    </div>
  );
}
```

`apps/web/src/components/catalog/ImportCostEstimate.tsx` :

```tsx
import type { ReactElement } from 'react';

export interface ImportCostEstimateProps {
  readonly estimate: {
    readonly goodsValue: string;
    readonly currency: string;
    readonly components: ReadonlyArray<{ readonly kind: string; readonly amount: string; readonly certainty: string; readonly text: string }>;
    readonly total: string;
    readonly certainty: string;
  };
  readonly labels: {
    readonly title: string;
    readonly indicative: string;
    readonly goods: string;
    readonly duty: string;
    readonly vat: string;
    readonly brokerage: string;
    readonly incomplete: string;
    readonly specialist: string;
    readonly validated: string;
  };
}

const KIND_LABELS: Record<string, 'goods' | 'duty' | 'vat' | 'brokerage'> = {
  goods_value: 'goods',
  duty: 'duty',
  vat: 'vat',
  admin_fee: 'brokerage',
  brokerage: 'brokerage',
};

/**
 * L'estimation est toujours marquée comme indicative : la douane et le
 * transporteur font foi. Aucun montant n'est présenté comme définitif.
 */
export function ImportCostEstimate({ estimate, labels }: ImportCostEstimateProps): ReactElement {
  return (
    <section className="import-estimate" aria-labelledby="import-estimate-title">
      <h2 id="import-estimate-title">{labels.title}</h2>
      <p className="import-estimate__badge">{labels.indicative}</p>
      <dl>
        {estimate.components.map((component) => (
          <div key={component.kind} className="import-estimate__row">
            <dt>{labels[KIND_LABELS[component.kind] ?? 'goods']}</dt>
            <dd>
              {component.amount} {estimate.currency}
              <span className="import-estimate__certainty">
                {component.certainty === 'incomplete'
                  ? labels.incomplete
                  : component.certainty === 'specialist_review'
                    ? labels.specialist
                    : component.certainty === 'estimated'
                      ? labels.indicative
                      : labels.validated}
              </span>
            </dd>
          </div>
        ))}
      </dl>
      <p className="import-estimate__total">
        {labels.title} : {estimate.total} {estimate.currency}
      </p>
    </section>
  );
}
```

`apps/web/src/components/catalog/PurchasePanel.tsx` (composant client) :

```tsx
'use client';

import { useMemo, useState, type ReactElement } from 'react';
import { useRouter } from 'next/navigation';

export interface PurchasePanelProps {
  readonly productId: string;
  readonly slug: string;
  readonly moq: number;
  readonly orderMultiple: number;
  readonly availableQuantity: number;
  readonly salesModes: readonly string[];
  readonly labels: {
    readonly addToCart: string;
    readonly moq: string;
    readonly multiple: string;
    readonly nextQuantity: string;
    readonly inStock: string;
    readonly lowStock: string;
    readonly outOfStock: string;
    readonly quantityInvalid: string;
  };
}

/** Quantité valide la plus proche au-dessus de la quantité saisie. */
export function nextValidQuantity(quantity: number, moq: number, multiple: number): number {
  if (quantity < moq) return moq;
  const remainder = quantity % multiple;
  return remainder === 0 ? quantity : quantity + (multiple - remainder);
}

export function PurchasePanel(props: PurchasePanelProps): ReactElement {
  const router = useRouter();
  const [quantity, setQuantity] = useState(props.moq);
  const [pending, setPending] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const valid = useMemo(
    () => nextValidQuantity(quantity, props.moq, props.orderMultiple),
    [quantity, props.moq, props.orderMultiple],
  );
  const invalid = valid !== quantity;

  const availability =
    props.availableQuantity === 0
      ? props.labels.outOfStock
      : props.availableQuantity < props.moq
        ? props.labels.lowStock
        : props.labels.inStock;

  async function submit(): Promise<void> {
    setPending(true);
    setError(null);
    try {
      const response = await fetch('/api/cart/lines', {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ productId: props.productId, quantity: valid }),
      });
      if (!response.ok) {
        setError(props.labels.quantityInvalid);
        return;
      }
      router.push('/fr/panier');
    } finally {
      setPending(false);
    }
  }

  return (
    <section className="purchase-panel" aria-labelledby="purchase-panel-title">
      <h2 id="purchase-panel-title" className="sr-only">
        {props.labels.addToCart}
      </h2>
      <p className="purchase-panel__moq">{props.labels.moq.replace('{qty}', String(props.moq))}</p>
      <p className="purchase-panel__multiple">
        {props.labels.multiple.replace('{qty}', String(props.orderMultiple))}
      </p>
      <p className="purchase-panel__availability" data-state={props.availableQuantity === 0 ? 'out' : 'in'}>
        {availability}
      </p>
      <label htmlFor="quantity" className="sr-only">
        {props.labels.addToCart}
      </label>
      <input
        id="quantity"
        name="quantity"
        type="number"
        inputMode="numeric"
        min={props.moq}
        step={props.orderMultiple}
        value={quantity}
        onChange={(event) => setQuantity(Number.parseInt(event.target.value, 10) || 0)}
        aria-describedby={invalid ? 'quantity-hint' : undefined}
        aria-invalid={invalid}
      />
      {invalid ? (
        <p id="quantity-hint" className="purchase-panel__hint" role="status">
          {props.labels.nextQuantity.replace('{qty}', String(valid))}
        </p>
      ) : null}
      {error ? (
        <p className="purchase-panel__error" role="alert">
          {error}
        </p>
      ) : null}
      <button type="button" onClick={submit} disabled={pending || props.availableQuantity === 0}>
        {props.labels.addToCart}
      </button>
    </section>
  );
}
```

`apps/web/src/components/catalog/ProductStates.tsx` :

```tsx
import type { ReactElement } from 'react';

export interface ProductStateProps {
  readonly title: string;
  readonly body: string;
  readonly actionLabel?: string;
  readonly variant: 'empty' | 'error' | 'network' | 'forbidden' | 'session' | 'notFound';
}

/** États obligatoires : chaque situation a une icône, un texte et une sortie. */
export function ProductState({ title, body, actionLabel, variant }: ProductStateProps): ReactElement {
  return (
    <div className={`product-state product-state--${variant}`} role={variant === 'error' ? 'alert' : 'status'}>
      <span className="product-state__icon" aria-hidden="true" data-variant={variant} />
      <h2>{title}</h2>
      <p>{body}</p>
      {actionLabel ? (
        <a className="product-state__action" href="/fr/">
          {actionLabel}
        </a>
      ) : null}
    </div>
  );
}
```

- [ ] **Step 4: Écrire la page serveur de la fiche produit**

`apps/web/src/app/[locale]/produits/[slug]/page.tsx` :

```tsx
import type { Metadata } from 'next';
import { notFound } from 'next/navigation';
import { getDictionary, t } from '@fanours/i18n';
import { LOCALES } from '@fanours/i18n';
import { fetchProduct } from '@/lib/api/catalog-client';
import { buildCanonical, SITE_URL } from '@/lib/seo/canonical';
import { buildProductJsonLd } from './product-jsonld';
import { ProductGallery } from '@/components/catalog/ProductGallery';
import { TierTable } from '@/components/catalog/TierTable';
import { PurchasePanel } from '@/components/catalog/PurchasePanel';
import { ImportCostEstimate } from '@/components/catalog/ImportCostEstimate';

export const dynamicParams = true;

interface PageProps {
  readonly params: Promise<{ locale: string; slug: string }>;
}

export async function generateMetadata({ params }: PageProps): Promise<Metadata> {
  const { locale, slug } = await params;
  const product = await fetchProduct(slug, locale);
  if (!product) return { title: 'Produit introuvable', robots: { index: false, follow: false } };

  const { canonical, alternates } = buildCanonical(product.slug, LOCALES);
  return {
    title: `${product.name} | Fanours`,
    description: product.shortDescription,
    alternates: {
      canonical,
      languages: Object.fromEntries(alternates.map((alternate) => [alternate.hrefLang, alternate.href])),
    },
    openGraph: {
      type: 'website',
      url: canonical,
      title: product.name,
      description: product.shortDescription,
      images: product.images.map((image) => ({ url: image.url, alt: image.alt })),
    },
    robots: { index: true, follow: true },
  };
}

export default async function ProductPage({ params }: PageProps) {
  const { locale, slug } = await params;
  const dict = getDictionary(locale);
  const product = await fetchProduct(slug, locale);
  if (!product) notFound();

  const jsonLd = buildProductJsonLd({
    name: product.name,
    description: product.shortDescription,
    sku: product.sku,
    slug: product.slug,
    imageUrls: product.images.map((image) => image.url),
    tiers: product.tiers.map((tier) => ({
      minQuantity: tier.minQuantity,
      maxQuantity: tier.maxQuantity,
      unitPrice: tier.unitPrice,
      currency: tier.currency,
    })),
    moq: product.moq,
    orderMultiple: product.orderMultiple,
    availableQuantity: product.availableQuantity,
    brand: 'Fanours',
    canonicalUrl: `${SITE_URL}/${locale}/produits/${product.slug}/`,
  });

  return (
    <main className="product-page">
      <script
        type="application/ld+json"
        // JSON-LD produit : contenu structuré dérivé des mêmes données serveur que la page.
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
      />
      <h1>{product.name}</h1>
      <p className="product-page__sku">{product.sku}</p>
      <ProductGallery images={product.images} name={product.name} />

      <section aria-labelledby="tiers-title">
        <h2 id="tiers-title">{t(dict, 'product.tiers.title')}</h2>
        <TierTable
          tiers={product.tiers}
          currency={product.tiers[0]?.currency ?? 'EUR'}
          caption={t(dict, 'product.tiers.caption')}
          quantityLabel={t(dict, 'product.tiers.quantity')}
          unitPriceLabel={t(dict, 'product.tiers.unitPrice')}
          saveLabel={t(dict, 'product.tiers.save')}
        />
      </section>

      <PurchasePanel
        productId={product.id}
        slug={product.slug}
        moq={product.moq}
        orderMultiple={product.orderMultiple}
        availableQuantity={product.availableQuantity}
        salesModes={product.salesModes}
        labels={{
          addToCart: t(dict, 'product.addToCart'),
          moq: t(dict, 'product.moq'),
          multiple: t(dict, 'product.multiple'),
          nextQuantity: t(dict, 'product.nextQuantity'),
          inStock: t(dict, 'product.availability.inStock'),
          lowStock: t(dict, 'product.availability.lowStock'),
          outOfStock: t(dict, 'product.availability.outOfStock'),
          quantityInvalid: t(dict, 'common.quantity.invalid'),
        }}
      />

      <ImportCostEstimate
        estimate={product.importEstimate}
        labels={{
          title: t(dict, 'product.import.title'),
          indicative: t(dict, 'product.import.indicative'),
          goods: t(dict, 'product.import.goods'),
          duty: t(dict, 'product.import.duty'),
          vat: t(dict, 'product.import.vat'),
          brokerage: t(dict, 'product.import.brokerage'),
          incomplete: t(dict, 'product.import.incomplete'),
          specialist: t(dict, 'product.import.specialist'),
          validated: t(dict, 'product.import.validated'),
        }}
      />

      <section aria-labelledby="preparation-title">
        <h2 id="preparation-title">{t(dict, 'product.preparation.title')}</h2>
        <p>{t(dict, 'product.preparation.body').replace('{days}', String(product.preparationDays))}</p>
      </section>

      <section aria-labelledby="shipping-title">
        <h2 id="shipping-title">{t(dict, 'product.shipping.title')}</h2>
        <p>{t(dict, 'product.shipping.body')}</p>
      </section>

      <section aria-labelledby="countries-title">
        <h2 id="countries-title">{t(dict, 'product.countries.title')}</h2>
        <ul>
          {product.destinationCountries.map((country) => (
            <li key={country}>{country}</li>
          ))}
        </ul>
      </section>

      <section aria-labelledby="weight-title">
        <h2 id="weight-title">{t(dict, 'product.weight.title')}</h2>
        <p>
          {product.netWeightGrams} g net · {product.grossWeightGrams} g brut ·{' '}
          {product.dimensionsMm.length}×{product.dimensionsMm.width}×{product.dimensionsMm.height} mm
        </p>
      </section>

      <section aria-labelledby="compliance-title">
        <h2 id="compliance-title">{t(dict, 'product.compliance.title')}</h2>
        <ul>
          {product.complianceDocuments.map((document) => (
            <li key={document.url}>
              <a href={document.url}>{document.title}</a>
            </li>
          ))}
        </ul>
      </section>
    </main>
  );
}
```

- [ ] **Step 5: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/web test -- product-jsonld.test.ts canonical.test.ts
```

Attendu : `Test Files  2 passed (2)` et `Tests  7 passed (7)`.

```bash
pnpm --filter @fanours/web typecheck
```

Attendu : aucune erreur.

```bash
git add apps/web/src/app/[locale]/produits apps/web/src/components/catalog apps/web/src/lib
git commit -m "feat(catalog): ajoute la fiche produit indexable et son JSON-LD

Une seule page indexable par produit et par langue. Le tableau des paliers est
un vrai tableau semantique present dans le HTML initial ; la version mobile
derive des memes donnees serveur. Le JSON-LD Product porte le prix du palier
MOQ et la quantite eligible en unites C62.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

### Task 11: Chantier 2.7 — Panier serveur, revalidation des règles et fusion

**Files:**
- Create: `apps/api/src/modules/cart/cart.service.ts`, `apps/api/src/modules/cart/cart.controller.ts`, `apps/api/src/modules/cart/cart.module.ts`, `apps/api/src/modules/cart/cart.service.spec.ts`
- Create: `apps/api/src/modules/cart/cart.repository.ts`
- Create: `packages/db/src/schema/cart.ts` (miroir Drizzle de la migration `0009_cart.sql` : cart, cart_line)
- Modify: `packages/db/src/schema/index.ts` (ajout de `export * from './cart.js';`)

**Interfaces:**
- Consumes: `packages/contracts` (`cartLineInputSchema`, `cartSchema`), `packages/pricing` (`resolveTier`, `assertPurchasable` — Task 5), `packages/domain` (`resolvePurchaseRules`).
- Produces:
  - `apps/api` → `GET /v1/cart`, `POST /v1/cart/lines`, `PATCH /v1/cart/lines/:lineId`, `DELETE /v1/cart/lines/:lineId`, `POST /v1/cart/merge`.
  - `CartService.getCart(ctx): Promise<CartView>`, `CartService.addLine(ctx, input): Promise<CartView>`, `CartService.updateLine(ctx, lineId, quantity): Promise<CartView>`, `CartService.removeLine(ctx, lineId): Promise<CartView>`, `CartService.merge(ctx, sourceCartId): Promise<CartView>`.
  - `CartView` = `{ id, currency, lines: CartLineView[], subtotal, adjustments: CartAdjustment[], total, updatedAt }`.
  - `CartLineView` = `{ id, productId, variantId, slug, name, quantity, unitPrice, lineTotal, minQuantity, orderMultiple, quantityAdjusted, priceChanged, availableQuantity }`.
  - `CartAdjustment` = `{ kind: 'quantity'|'price'|'stock'; reason: string; lineId: string; previousValue: string; currentValue: string }`.

- [ ] **Step 1: Écrire le test d'intégration qui échoue**

`apps/api/src/modules/cart/cart.service.spec.ts` :

```typescript
import { afterAll, beforeAll, beforeEach, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import pg from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import { runMigrations } from '@fanours/db/migrate';
import * as schema from '@fanours/db/schema';
import { CartService } from './cart.service.js';

let container: StartedPostgreSqlContainer;
let pool: pg.Pool;
let service: CartService;

const USER_ID = '11111111-1111-4111-8111-111111111111';

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('fanours')
    .withUsername('fanours')
    .withPassword('fanours')
    .start();
  pool = new pg.Pool({ connectionString: container.getConnectionUri() });
  await runMigrations(pool);
  service = new CartService(drizzle(pool, { schema }));
}, 120_000);

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

beforeEach(async () => {
  await pool.query('DELETE FROM cart_line');
  await pool.query('DELETE FROM cart');
  await pool.query('DELETE FROM price_tier');
  await pool.query('DELETE FROM product_variant');
  await pool.query('DELETE FROM product_translation');
  await pool.query('DELETE FROM product');

  const product = await pool.query<{ id: string }>(
    `INSERT INTO product (sku, status, sales_modes, moq, order_multiple, origin_country, net_weight_g, gross_weight_g, length_mm, width_mm, height_mm)
     VALUES ('CL-MOL-01', 'published', ARRAY['wholesale','retail'], 6, 6, 'CN', 10, 20, 1, 1, 1) RETURNING id`,
  );
  await pool.query(
    `INSERT INTO product_translation (product_id, locale, name, short_desc, description, slug)
     VALUES ($1, 'fr-FR', 'Clé à molette', 'Outil', 'Outil de serrage', 'cle-a-molette')`,
    [product.rows[0]!.id],
  );
  const variant = await pool.query<{ id: string }>(
    `INSERT INTO product_variant (product_id, sku, sales_mode, min_qty, order_multiple) VALUES ($1, 'CL-MOL-01-STD', 'wholesale', 6, 6) RETURNING id`,
    [product.rows[0]!.id],
  );
  await pool.query(
    `INSERT INTO price_tier (product_id, variant_id, channel, currency, min_qty, max_qty, unit_price)
     VALUES ($1, $2, 'wholesale', 'EUR', 6, 47, 12.0000)`,
    [product.rows[0]!.id, variant.rows[0]!.id],
  );
  await pool.query(
    `INSERT INTO price_tier (product_id, variant_id, channel, currency, min_qty, max_qty, unit_price)
     VALUES ($1, $2, 'wholesale', 'EUR', 48, NULL, 10.8000)`,
    [product.rows[0]!.id, variant.rows[0]!.id],
  );
});

describe('CartService', () => {
  it('ajoute une ligne et recalcule le total côté serveur', async () => {
    const cart = await service.addLine({ userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' }, {
      productId: 'x',
      sku: 'CL-MOL-01-STD',
      quantity: 12,
    });
    expect(cart.lines).toHaveLength(1);
    expect(cart.lines[0]?.unitPrice).toBe('12.0000');
    expect(cart.lines[0]?.lineTotal).toBe('144.0000');
    expect(cart.subtotal).toBe('144.0000');
  });

  it('ajuste une quantité hors MOQ et le signale', async () => {
    const cart = await service.addLine({ userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' }, {
      productId: 'x',
      sku: 'CL-MOL-01-STD',
      quantity: 4,
    });
    expect(cart.lines[0]?.quantity).toBe(6);
    expect(cart.lines[0]?.quantityAdjusted).toBe(true);
    expect(cart.adjustments.some((adjustment) => adjustment.kind === 'quantity')).toBe(true);
  });

  it('aligne une quantité sur un multiple', async () => {
    const cart = await service.addLine({ userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' }, {
      productId: 'x',
      sku: 'CL-MOL-01-STD',
      quantity: 7,
    });
    expect(cart.lines[0]?.quantity).toBe(12);
  });

  it('recalcule le prix au palier supérieur quand la quantité change', async () => {
    await service.addLine({ userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' }, {
      productId: 'x',
      sku: 'CL-MOL-01-STD',
      quantity: 12,
    });
    const cart = await service.getCart({ userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' });
    const lineId = cart.lines[0]!.id;
    const updated = await service.updateLine({ userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' }, lineId, 60);
    expect(updated.lines[0]?.unitPrice).toBe('10.8000');
    expect(updated.lines[0]?.lineTotal).toBe('648.0000');
    expect(updated.adjustments.some((adjustment) => adjustment.kind === 'price')).toBe(true);
  });

  it('fusionne un panier invité dans le panier de l’utilisateur', async () => {
    const guest = await service.addLine({ userId: 'guest:abc', locale: 'fr-FR', salesMode: 'wholesale' }, {
      productId: 'x',
      sku: 'CL-MOL-01-STD',
      quantity: 12,
    });
    const merged = await service.merge({ userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' }, guest.id);
    expect(merged.lines).toHaveLength(1);
    expect(merged.lines[0]?.quantity).toBe(12);
  });

  it('refuse une quantité négative', async () => {
    await expect(
      service.addLine({ userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' }, {
        productId: 'x',
        sku: 'CL-MOL-01-STD',
        quantity: -3,
      }),
    ).rejects.toThrow();
  });
});
```

- [ ] **Step 2: Implémenter le dépôt et le service de panier**

`apps/api/src/modules/cart/cart.repository.ts` :

```typescript
import { sql, type PgDatabase } from 'drizzle-orm';
import * as schema from '@fanours/db/schema';

type Db = PgDatabase<any, typeof schema>;

export interface CartRow {
  readonly id: string;
  readonly currency: string;
  readonly updated_at: string;
}

export interface CartLineRow {
  readonly id: string;
  readonly cart_id: string;
  readonly product_id: string;
  readonly variant_id: string;
  readonly sku: string;
  readonly slug: string;
  readonly name: string;
  readonly quantity: number;
  readonly min_qty: number;
  readonly order_multiple: number;
  readonly available_quantity: number;
}

export class CartRepository {
  constructor(private readonly db: Db) {}

  async findCart(userId: string): Promise<CartRow | null> {
    const result = await this.db.execute<CartRow>(sql`
      SELECT c.id, c.currency, c.updated_at
      FROM cart c
      WHERE c.user_id = ${userId} AND c.status = 'active'
      ORDER BY c.updated_at DESC
      LIMIT 1
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as CartRow[]);
    return rows[0] ?? null;
  }

  async createCart(userId: string, currency: string): Promise<CartRow> {
    const result = await this.db.execute<CartRow>(sql`
      INSERT INTO cart (user_id, currency, status) VALUES (${userId}, ${currency}, 'active')
      RETURNING id, currency, updated_at
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as CartRow[]);
    return rows[0]!;
  }

  async listLines(cartId: string): Promise<CartLineRow[]> {
    const result = await this.db.execute<CartLineRow>(sql`
      SELECT cl.id, cl.cart_id, cl.product_id, cl.variant_id, cl.sku,
             cl.quantity, cl.min_qty, cl.order_multiple,
             COALESCE(SUM(ii.on_hand - ii.reserved - ii.damaged - ii.in_quality_control), 0)::int AS available_quantity,
             pt.slug, pt.name
      FROM cart_line cl
      JOIN product p ON p.id = cl.product_id
      JOIN product_translation pt ON pt.product_id = p.id AND pt.locale = cl.locale
      LEFT JOIN product_variant pv ON pv.id = cl.variant_id
      LEFT JOIN inventory_item ii ON ii.variant_id = cl.variant_id
      WHERE cl.cart_id = ${cartId}
      GROUP BY cl.id, pt.slug, pt.name
      ORDER BY cl.created_at ASC
    `);
    return 'rows' in result ? result.rows : (result as unknown as CartLineRow[]);
  }

  async upsertLine(input: {
    cartId: string;
    locale: string;
    productId: string;
    variantId: string;
    sku: string;
    quantity: number;
    minQty: number;
    orderMultiple: number;
  }): Promise<void> {
    await this.db.execute(sql`
      INSERT INTO cart_line (cart_id, locale, product_id, variant_id, sku, quantity, min_qty, order_multiple)
      VALUES (${input.cartId}, ${input.locale}, ${input.productId}, ${input.variantId}, ${input.sku}, ${input.quantity}, ${input.minQty}, ${input.orderMultiple})
      ON CONFLICT (cart_id, product_id, variant_id)
      DO UPDATE SET quantity = cart_line.quantity + EXCLUDED.quantity, updated_at = now()
    `);
  }

  async setLineQuantity(lineId: string, quantity: number): Promise<void> {
    await this.db.execute(sql`
      UPDATE cart_line SET quantity = ${quantity}, updated_at = now() WHERE id = ${lineId}
    `);
  }

  async deleteLine(lineId: string): Promise<void> {
    await this.db.execute(sql`DELETE FROM cart_line WHERE id = ${lineId}`);
  }

  async moveLines(sourceCartId: string, targetCartId: string): Promise<void> {
    await this.db.execute(sql`
      UPDATE cart_line SET cart_id = ${targetCartId}, updated_at = now() WHERE cart_id = ${sourceCartId}
    `);
  }
}
```

`apps/api/src/modules/cart/cart.service.ts` :

```typescript
import { sql, type PgDatabase } from 'drizzle-orm';
import * as schema from '@fanours/db/schema';
import { ApiError } from '@/common/errors/api-error';
import { resolvePurchaseRules } from '@fanours/domain';
import { resolveTier } from '@fanours/pricing';
import { CartRepository } from './cart.repository.js';

type Db = PgDatabase<any, typeof schema>;

export interface CartContext {
  readonly userId: string;
  readonly locale: string;
  readonly salesMode: string;
}

export interface CartAdjustment {
  readonly kind: 'quantity' | 'price' | 'stock';
  readonly reason: string;
  readonly lineId: string;
  readonly previousValue: string;
  readonly currentValue: string;
}

export interface CartLineView {
  readonly id: string;
  readonly productId: string;
  readonly variantId: string;
  readonly sku: string;
  readonly slug: string;
  readonly name: string;
  readonly quantity: number;
  readonly unitPrice: string;
  readonly lineTotal: string;
  readonly minQuantity: number;
  readonly orderMultiple: number;
  readonly quantityAdjusted: boolean;
  readonly priceChanged: boolean;
  readonly availableQuantity: number;
}

export interface CartView {
  readonly id: string;
  readonly currency: string;
  readonly lines: readonly CartLineView[];
  readonly subtotal: string;
  readonly adjustments: readonly CartAdjustment[];
  readonly total: string;
  readonly updatedAt: string;
}

interface VariantContext {
  readonly productId: string;
  readonly variantId: string;
  readonly minQty: number;
  readonly orderMultiple: number;
}

export class CartService {
  private readonly repository: CartRepository;

  constructor(private readonly db: Db) {
    this.repository = new CartRepository(db);
  }

  async getCart(ctx: CartContext): Promise<CartView> {
    const cart = (await this.repository.findCart(ctx.userId)) ?? (await this.repository.createCart(ctx.userId, 'EUR'));
    return this.rebuild(cart.id, ctx);
  }

  async addLine(ctx: CartContext, input: { productId: string; sku: string; quantity: number }): Promise<CartView> {
    if (!Number.isInteger(input.quantity) || input.quantity <= 0) {
      throw new ApiError(422, 'cart.invalid_quantity', 'La quantité doit être un entier positif.');
    }
    const variant = await this.loadVariant(input.sku, ctx.salesMode);
    if (!variant) {
      throw new ApiError(404, 'cart.variant_not_found', 'Cette référence n’existe pas pour ce canal de vente.');
    }

    const cart = (await this.repository.findCart(ctx.userId)) ?? (await this.repository.createCart(ctx.userId, 'EUR'));
    const existing = (await this.repository.listLines(cart.id)).find((line) => line.sku === input.sku);
    const requested = (existing?.quantity ?? 0) + input.quantity;

    const rules = resolvePurchaseRules({
      quantity: requested,
      minQuantity: variant.minQty,
      orderMultiple: variant.orderMultiple,
      salesMode: ctx.salesMode,
    });
    if (!rules.allowed) {
      throw new ApiError(422, 'cart.purchase_not_allowed', rules.reason, { nextValidQuantity: rules.nextValidQuantity });
    }

    await this.repository.upsertLine({
      cartId: cart.id,
      locale: ctx.locale,
      productId: variant.productId,
      variantId: variant.variantId,
      sku: input.sku,
      quantity: rules.nextValidQuantity,
      minQty: variant.minQty,
      orderMultiple: variant.orderMultiple,
    });
    return this.rebuild(cart.id, ctx);
  }

  async updateLine(ctx: CartContext, lineId: string, quantity: number): Promise<CartView> {
    if (!Number.isInteger(quantity) || quantity <= 0) {
      throw new ApiError(422, 'cart.invalid_quantity', 'La quantité doit être un entier positif.');
    }
    const cart = await this.requireCart(ctx);
    const lines = await this.repository.listLines(cart.id);
    const line = lines.find((candidate) => candidate.id === lineId);
    if (!line) throw new ApiError(404, 'cart.line_not_found', 'Cette ligne de panier n’existe pas.');

    const rules = resolvePurchaseRules({
      quantity,
      minQuantity: line.min_qty,
      orderMultiple: line.order_multiple,
      salesMode: ctx.salesMode,
    });
    await this.repository.setLineQuantity(lineId, rules.nextValidQuantity);
    return this.rebuild(cart.id, ctx);
  }

  async removeLine(ctx: CartContext, lineId: string): Promise<CartView> {
    const cart = await this.requireCart(ctx);
    await this.repository.deleteLine(lineId);
    return this.rebuild(cart.id, ctx);
  }

  async merge(ctx: CartContext, sourceCartId: string): Promise<CartView> {
    const target = (await this.repository.findCart(ctx.userId)) ?? (await this.repository.createCart(ctx.userId, 'EUR'));
    if (target.id === sourceCartId) return this.rebuild(target.id, ctx);
    await this.repository.moveLines(sourceCartId, target.id);
    return this.rebuild(target.id, ctx);
  }

  private async requireCart(ctx: CartContext) {
    const cart = await this.repository.findCart(ctx.userId);
    if (!cart) throw new ApiError(404, 'cart.not_found', 'Aucun panier actif.');
    return cart;
  }

  private async loadVariant(sku: string, salesMode: string): Promise<VariantContext | null> {
    const result = await this.db.execute<{
      product_id: string;
      variant_id: string;
      min_qty: number;
      order_multiple: number;
    }>(sql`
      SELECT pv.product_id, pv.id AS variant_id, pv.min_qty, pv.order_multiple
      FROM product_variant pv
      JOIN product p ON p.id = pv.product_id
      WHERE pv.sku = ${sku} AND pv.sales_mode = ${salesMode} AND p.status = 'published'
      LIMIT 1
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { product_id: string; variant_id: string; min_qty: number; order_multiple: number }[]);
    const row = rows[0];
    if (!row) return null;
    return {
      productId: row.product_id,
      variantId: row.variant_id,
      minQty: row.min_qty,
      orderMultiple: row.order_multiple,
    };
  }

  /**
   * Recalcule intégralement le panier : prix, MOQ, incréments et stock sont
   * relus en base à chaque appel. Le panier affiché n’est jamais une vérité.
   */
  private async rebuild(cartId: string, ctx: CartContext): Promise<CartView> {
    const cart = await this.repository.findCartById(cartId);
    const lines = await this.repository.listLines(cartId);
    const adjustments: CartAdjustment[] = [];
    const viewLines: CartLineView[] = [];
    let subtotalMinor = 0n;

    for (const line of lines) {
      const tier = await this.resolvePrice(line.variant_id, line.product_id, line.quantity, ctx);
      const rules = resolvePurchaseRules({
        quantity: line.quantity,
        minQuantity: line.min_qty,
        orderMultiple: line.order_multiple,
        salesMode: ctx.salesMode,
      });
      const quantity = rules.nextValidQuantity;
      if (quantity !== line.quantity) {
        adjustments.push({
          kind: 'quantity',
          reason: rules.reason,
          lineId: line.id,
          previousValue: String(line.quantity),
          currentValue: String(quantity),
        });
        await this.repository.setLineQuantity(line.id, quantity);
      }
      if (line.available_quantity > 0 && quantity > line.available_quantity) {
        adjustments.push({
          kind: 'stock',
          reason: 'insufficient_stock',
          lineId: line.id,
          previousValue: String(quantity),
          currentValue: String(line.available_quantity),
        });
      }

      const unitPrice = tier.unitPrice;
      const lineTotal = (BigInt(Math.round(Number(unitPrice) * 100 * quantity)) / 100n).toString();
      subtotalMinor += BigInt(Math.round(Number(unitPrice) * 100)) * BigInt(quantity);
      viewLines.push({
        id: line.id,
        productId: line.product_id,
        variantId: line.variant_id,
        sku: line.sku,
        slug: line.slug,
        name: line.name,
        quantity,
        unitPrice,
        lineTotal,
        minQuantity: line.min_qty,
        orderMultiple: line.order_multiple,
        quantityAdjusted: quantity !== line.quantity,
        priceChanged: false,
        availableQuantity: line.available_quantity,
      });
    }

    return {
      id: cart.id,
      currency: cart.currency,
      lines: viewLines,
      subtotal: (subtotalMinor / 100n).toString(),
      adjustments,
      total: (subtotalMinor / 100n).toString(),
      updatedAt: cart.updated_at,
    };
  }

  private async resolvePrice(
    variantId: string,
    productId: string,
    quantity: number,
    ctx: CartContext,
  ): Promise<{ unitPrice: string; changed: boolean }> {
    const result = await this.db.execute<{ unit_price: string; min_qty: number; max_qty: number | null }>(sql`
      SELECT unit_price, min_qty, max_qty
      FROM price_tier
      WHERE (variant_id = ${variantId} OR (variant_id IS NULL AND product_id = ${productId}))
        AND channel = ${ctx.salesMode}
        AND currency = 'EUR'
        AND (valid_to IS NULL OR valid_to > now())
      ORDER BY min_qty ASC
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { unit_price: string; min_qty: number; max_qty: number | null }[]);
    const resolution = resolveTier({
      tiers: rows.map((row) => ({
        minQuantity: row.min_qty,
        maxQuantity: row.max_qty,
        unitPrice: row.unit_price,
        currency: 'EUR',
      })),
      quantity,
    });
    return { unitPrice: resolution.unitPrice, changed: resolution.changed };
  }
}
```

- [ ] **Step 3: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/api test:integration -- cart.service.spec.ts
```

Attendu : `Test Files  1 passed (1)` et `Tests  6 passed (6)`.

```bash
git add apps/api/src/modules/cart packages/db/src/schema
git commit -m "feat(cart): ajoute le panier serveur avec revalidation des regles

Le panier est une vue recalculee en base a chaque lecture : prix au palier
correspondant, MOQ, multiples et stock sont relus cote serveur. Les ecarts
constantes sont exposes dans adjustments pour etre affiches au client.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

### Task 12: Chantier 2.8 — Checkout : recalcul serveur, snapshot de commande et taxes

**Files:**
- Create: `apps/api/src/modules/checkout/checkout.service.ts`, `apps/api/src/modules/checkout/checkout.controller.ts`, `apps/api/src/modules/checkout/checkout.module.ts`, `apps/api/src/modules/checkout/checkout.service.spec.ts`
- Create: `apps/api/src/modules/checkout/order-snapshot.ts`
- Create: `apps/web/src/app/[locale]/checkout/page.tsx`, `apps/web/src/app/[locale]/checkout/checkout-form.tsx`

**Interfaces:**
- Consumes: `packages/contracts` (`checkoutRequestSchema`, `orderSnapshotSchema`), `packages/pricing` (`resolveTier`), `packages/compliance` (`resolveRule`, `calculateImportCost` — Task 8), `packages/domain` (`resolvePurchaseRules`), `apps/api` `StockService` (Task 6).
- Produces:
  - `apps/api` → `POST /v1/checkout/quote` (devis non engageant), `POST /v1/checkout` (crée la commande — Task 13).
  - `CheckoutService.quote(ctx, input): Promise<CheckoutQuote>`, `CheckoutService.createOrder(ctx, input): Promise<OrderSnapshot>`.
  - `CheckoutQuote` = `{ currency, lines: QuoteLine[], subtotal, shipping, importCosts, taxes, total, incoterm, warnings: string[] }`.
  - `OrderSnapshot` = `{ orderId, orderNumber, snapshot, quote }` — le snapshot est figé et immuable.

- [ ] **Step 1: Écrire le test d'intégration qui échoue**

`apps/api/src/modules/checkout/checkout.service.spec.ts` :

```typescript
import { afterAll, beforeAll, beforeEach, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import pg from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import { runMigrations } from '@fanours/db/migrate';
import * as schema from '@fanours/db/schema';
import { CheckoutService } from './checkout.service.js';

let container: StartedPostgreSqlContainer;
let pool: pg.Pool;
let service: CheckoutService;

const USER_ID = '11111111-1111-4111-8111-111111111111';

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('fanours')
    .withUsername('fanours')
    .withPassword('fanours')
    .start();
  pool = new pg.Pool({ connectionString: container.getConnectionUri() });
  await runMigrations(pool);
  service = new CheckoutService(drizzle(pool, { schema }));
}, 120_000);

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

beforeEach(async () => {
  await pool.query('DELETE FROM cart_line');
  await pool.query('DELETE FROM cart');
  await pool.query('DELETE FROM price_tier');
  await pool.query('DELETE FROM product_variant');
  await pool.query('DELETE FROM product_translation');
  await pool.query('DELETE FROM product');
  await pool.query('DELETE FROM trade_rule_version');
  await pool.query('DELETE FROM trade_rule');
  await pool.query('DELETE FROM trade_rule_source');
});

async function seedCatalogue(): Promise<void> {
  const product = await pool.query<{ id: string }>(
    `INSERT INTO product (sku, status, sales_modes, moq, order_multiple, origin_country, net_weight_g, gross_weight_g, length_mm, width_mm, height_mm)
     VALUES ('CL-MOL-01', 'published', ARRAY['wholesale','retail'], 6, 6, 'CN', 10, 20, 1, 1, 1) RETURNING id`,
  );
  await pool.query(
    `INSERT INTO product_translation (product_id, locale, name, short_desc, description, slug)
     VALUES ($1, 'fr-FR', 'Clé à molette', 'Outil', 'Outil de serrage', 'cle-a-molette')`,
    [product.rows[0]!.id],
  );
  const variant = await pool.query<{ id: string }>(
    `INSERT INTO product_variant (product_id, sku, sales_mode, min_qty, order_multiple) VALUES ($1, 'CL-MOL-01-STD', 'wholesale', 6, 6) RETURNING id`,
    [product.rows[0]!.id],
  );
  await pool.query(
    `INSERT INTO price_tier (product_id, variant_id, channel, currency, min_qty, max_qty, unit_price)
     VALUES ($1, $2, 'wholesale', 'EUR', 6, NULL, 12.0000)`,
    [product.rows[0]!.id, variant.rows[0]!.id],
  );
}

describe('CheckoutService.quote', () => {
  it('recalcule le prix côté serveur et ne fait jamais confiance au client', async () => {
    await seedCatalogue();
    const quote = await service.quote(
      { userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' },
      {
        lines: [{ productId: 'ignored-by-client', sku: 'CL-MOL-01-STD', quantity: 12, claimedUnitPrice: '0.0100' }],
        country: 'FR',
        incoterm: 'DAP',
      },
    );
    expect(quote.lines[0]?.unitPrice).toBe('12.0000');
    expect(quote.subtotal).toBe('144.0000');
  });

  it('refuse une quantité qui ne respecte ni le MOQ ni le multiple', async () => {
    await seedCatalogue();
    await expect(
      service.quote({ userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' }, {
        lines: [{ productId: 'x', sku: 'CL-MOL-01-STD', quantity: 3 }],
        country: 'FR',
        incoterm: 'DAP',
      }),
    ).rejects.toThrow();
  });

  it('calcule les droits et la TVA à partir des règles en base', async () => {
    await seedCatalogue();
    const source = await pool.query<{ id: string }>(
      `INSERT INTO trade_rule_source (url, authority, title, consulted_at)
       VALUES ('https://taxation-customs.ec.europa.eu/dds2/taric/', 'Commission européenne', 'TARIC', '2026-10-01') RETURNING id`,
    );
    const rule = await pool.query<{ id: string }>(
      `INSERT INTO trade_rule (key, country_code, scope, rule_type, certainty, priority, source_id)
       VALUES ('fr-duty-standard', 'FR', 'global', 'duty', 'validated', 10, $1) RETURNING id`,
      [source.rows[0]!.id],
    );
    await pool.query(
      `INSERT INTO trade_rule_version (trade_rule_id, version_no, valid_from, params, calculation_mode, taxable_base, source_id, certainty)
       VALUES ($1, 1, '2020-01-01', '{"rateBps": 500}', 'percentage', 'goods+shipping', $2, 'validated')`,
      [rule.rows[0]!.id, source.rows[0]!.id],
    );
    const quote = await service.quote(
      { userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' },
      {
        lines: [{ productId: 'x', sku: 'CL-MOL-01-STD', quantity: 12 }],
        country: 'FR',
        incoterm: 'DAP',
        shippingMinor: '10.0000',
      },
    );
    const duty = quote.importCosts.components.find((component) => component.kind === 'duty');
    expect(duty?.amountMinor).toBe('7700');
  });

  it('produit un snapshot figé contenant adresse, prix, taxes et version de règle', async () => {
    await seedCatalogue();
    const order = await service.createOrder(
      { userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' },
      {
        lines: [{ productId: 'x', sku: 'CL-MOL-01-STD', quantity: 12 }],
        country: 'FR',
        incoterm: 'DAP',
        shippingAddress: {
          recipient: 'Ada Lovelace',
          line1: '10 rue de Rivoli',
          postalCode: '75001',
          city: 'Paris',
          country: 'FR',
        },
        idempotencyKey: 'idem-checkout-1',
      },
    );
    expect(order.snapshot.address.postalCode).toBe('75001');
    expect(order.snapshot.lines[0]?.unitPrice).toBe('12.0000');
    expect(order.snapshot.incoterm).toBe('DAP');
    expect(order.snapshot.snapshotVersion).toBe(1);
  });
});
```

- [ ] **Step 2: Implémenter le snapshot de commande**

`apps/api/src/modules/checkout/order-snapshot.ts` :

```typescript
import { randomUUID } from 'node:crypto';

export interface SnapshotAddress {
  readonly recipient: string;
  readonly line1: string;
  readonly line2?: string;
  readonly postalCode: string;
  readonly city: string;
  readonly country: string;
}

export interface SnapshotLine {
  readonly productId: string;
  readonly variantId: string;
  readonly sku: string;
  readonly name: string;
  readonly quantity: number;
  readonly unitPrice: string;
  readonly lineTotal: string;
  readonly taxAmount: string;
  readonly dutyAmount: string;
}

export interface OrderSnapshot {
  readonly snapshotVersion: number;
  readonly capturedAt: string;
  readonly address: SnapshotAddress;
  readonly incoterm: 'EXW' | 'FOB' | 'CIF' | 'DAP' | 'DDP';
  readonly currency: string;
  readonly lines: readonly SnapshotLine[];
  readonly subtotal: string;
  readonly shipping: string;
  readonly taxTotal: string;
  readonly dutyTotal: string;
  readonly total: string;
  readonly taxRules: readonly { readonly ruleKey: string; readonly versionNo: number; readonly certainty: string }[];
  readonly termsVersion: string;
  readonly termsAcceptedAt: string;
}

/**
 * Le snapshot fige tout ce qui a servi à calculer la commande : adresse, prix
 * unitaires, règles de taxe, conditions de vente. La commande ne relit jamais
 * le catalogue courant.
 */
export function buildOrderSnapshot(input: {
  address: SnapshotAddress;
  incoterm: OrderSnapshot['incoterm'];
  currency: string;
  lines: readonly SnapshotLine[];
  subtotal: string;
  shipping: string;
  taxTotal: string;
  dutyTotal: string;
  total: string;
  taxRules: readonly { ruleKey: string; versionNo: number; certainty: string }[];
  termsVersion: string;
}): OrderSnapshot {
  return {
    snapshotVersion: 1,
    capturedAt: new Date().toISOString(),
    address: input.address,
    incoterm: input.incoterm,
    currency: input.currency,
    lines: input.lines,
    subtotal: input.subtotal,
    shipping: input.shipping,
    taxTotal: input.taxTotal,
    dutyTotal: input.dutyTotal,
    total: input.total,
    taxRules: input.taxRules,
    termsVersion: input.termsVersion,
    termsAcceptedAt: new Date().toISOString(),
  };
}

export function newIdempotencyKey(): string {
  return randomUUID();
}
```

- [ ] **Step 3: Implémenter le service de checkout**

`apps/api/src/modules/checkout/checkout.service.ts` :

```typescript
import { sql, type PgDatabase } from 'drizzle-orm';
import * as schema from '@fanours/db/schema';
import { ApiError } from '@/common/errors/api-error';
import { resolvePurchaseRules } from '@fanours/domain';
import { resolveTier } from '@fanours/pricing';
import { calculateImportCost, resolveRule, type TaxComponentInput } from '@fanours/compliance';
import { buildOrderSnapshot, type OrderSnapshot, type SnapshotAddress, type SnapshotLine } from './order-snapshot.js';

type Db = PgDatabase<any, typeof schema>;

export interface CheckoutContext {
  readonly userId: string;
  readonly locale: string;
  readonly salesMode: string;
}

export interface CheckoutLineInput {
  readonly productId: string;
  readonly sku: string;
  readonly quantity: number;
  /** Montant éventuellement annoncé par le client : il est ignoré, jamais utilisé. */
  readonly claimedUnitPrice?: string;
}

export interface CheckoutInput {
  readonly lines: readonly CheckoutLineInput[];
  readonly country: string;
  readonly incoterm: 'EXW' | 'FOB' | 'CIF' | 'DAP' | 'DDP';
  readonly shippingAddress?: SnapshotAddress;
  readonly shippingMinor?: string;
  readonly idempotencyKey?: string;
}

export interface QuoteLine {
  readonly productId: string;
  readonly variantId: string;
  readonly sku: string;
  readonly name: string;
  readonly quantity: number;
  readonly unitPrice: string;
  readonly lineTotal: string;
  readonly minQuantity: number;
  readonly orderMultiple: number;
}

export interface CheckoutQuote {
  readonly currency: string;
  readonly lines: readonly QuoteLine[];
  readonly subtotal: string;
  readonly shipping: string;
  readonly importCosts: {
    readonly components: readonly { readonly kind: string; readonly amountMinor: string; readonly certainty: string; readonly text: string }[];
    readonly totalMinor: string;
    readonly certainty: string;
  };
  readonly taxes: { readonly totalMinor: string; readonly certainty: string };
  readonly total: string;
  readonly incoterm: string;
  readonly warnings: readonly string[];
}

export interface CreatedOrder {
  readonly orderId: string;
  readonly orderNumber: string;
  readonly snapshot: OrderSnapshot;
  readonly quote: CheckoutQuote;
}

export class CheckoutService {
  constructor(private readonly db: Db) {}

  /**
   * Devis non engageant : tout est recalculé en base, aucune valeur envoyée
   * par le navigateur n’est utilisée.
   */
  async quote(ctx: CheckoutContext, input: CheckoutInput): Promise<CheckoutQuote> {
    const lines: QuoteLine[] = [];
    let subtotalMinor = 0n;
    const warnings: string[] = [];

    for (const line of input.lines) {
      const variant = await this.loadVariant(line.sku, ctx.salesMode);
      if (!variant) {
        throw new ApiError(404, 'checkout.variant_not_found', 'Cette référence n’existe pas pour ce canal de vente.');
      }
      const rules = resolvePurchaseRules({
        quantity: line.quantity,
        minQuantity: variant.minQty,
        orderMultiple: variant.orderMultiple,
        salesMode: ctx.salesMode,
      });
      if (!rules.allowed) {
        throw new ApiError(422, 'checkout.purchase_not_allowed', rules.reason, {
          nextValidQuantity: rules.nextValidQuantity,
        });
      }
      const tier = await this.resolvePrice(variant.variantId, variant.productId, rules.nextValidQuantity, ctx);
      const lineTotalMinor = BigInt(Math.round(Number(tier.unitPrice) * 100)) * BigInt(rules.nextValidQuantity);
      subtotalMinor += lineTotalMinor;
      lines.push({
        productId: variant.productId,
        variantId: variant.variantId,
        sku: line.sku,
        name: variant.name,
        quantity: rules.nextValidQuantity,
        unitPrice: tier.unitPrice,
        lineTotal: (lineTotalMinor / 100n).toString(),
        minQuantity: variant.minQty,
        orderMultiple: variant.orderMultiple,
      });
    }

    const shippingMinor = BigInt(Math.round(Number(input.shippingMinor ?? '0') * 100));
    const ruleResolutions = await this.resolveTradeRules(input.country);
    for (const resolution of ruleResolutions) {
      for (const alert of resolution.alerts) warnings.push(alert.message);
    }

    const components: TaxComponentInput[] = ruleResolutions
      .filter((resolution) => resolution.version !== null)
      .map((resolution) => ({
        kind: resolution.rule!.ruleType === 'duty' ? 'duty' : 'vat',
        calculationMode: (resolution.version!.calculationMode ?? 'percentage') as TaxComponentInput['calculationMode'],
        rateBps: Number((resolution.version!.params as Record<string, unknown>).rateBps ?? 0),
        taxableBase: (resolution.version!.taxableBase ?? 'goods_value') as TaxComponentInput['taxableBase'],
        certainty: resolution.certainty,
        collectedByFanours: false,
      }));

    const importCosts = calculateImportCost({
      goodsValueMinor: subtotalMinor,
      shippingMinor: input.shippingMinor === undefined ? null : shippingMinor,
      insuranceMinor: 0n,
      currency: 'EUR',
      components,
    });

    const taxTotal = importCosts.components
      .filter((component) => component.kind === 'vat')
      .reduce((sum, component) => sum + component.amountMinor, 0n);
    const dutyTotal = importCosts.components
      .filter((component) => component.kind === 'duty')
      .reduce((sum, component) => sum + component.amountMinor, 0n);
    const totalMinor = subtotalMinor + shippingMinor + taxTotal + dutyTotal;

    return {
      currency: 'EUR',
      lines,
      subtotal: (subtotalMinor / 100n).toString(),
      shipping: (shippingMinor / 100n).toString(),
      importCosts: {
        components: importCosts.components.map((component) => ({
          kind: component.kind,
          amountMinor: component.amountMinor.toString(),
          certainty: component.certainty,
          text: component.text,
        })),
        totalMinor: importCosts.totalMinor.toString(),
        certainty: importCosts.certainty,
      },
      taxes: { totalMinor: taxTotal.toString(), certainty: importCosts.certainty },
      total: (totalMinor / 100n).toString(),
      incoterm: input.incoterm,
      warnings,
    };
  }

  /**
   * Création de commande : le snapshot est figé au même instant que le devis.
   * L’idempotence est garantie par la clé unique `orders.idempotency_key`.
   */
  async createOrder(ctx: CheckoutContext, input: CheckoutInput): Promise<CreatedOrder> {
    if (!input.shippingAddress) {
      throw new ApiError(422, 'checkout.address_required', 'Une adresse de livraison est obligatoire.');
    }
    const quote = await this.quote(ctx, input);
    const ruleResolutions = await this.resolveTradeRules(input.country);

    const snapshot = buildOrderSnapshot({
      address: input.shippingAddress,
      incoterm: input.incoterm,
      currency: quote.currency,
      lines: quote.lines.map(
        (line): SnapshotLine => ({
          productId: line.productId,
          variantId: line.variantId,
          sku: line.sku,
          name: line.name,
          quantity: line.quantity,
          unitPrice: line.unitPrice,
          lineTotal: line.lineTotal,
          taxAmount: '0.0000',
          dutyAmount: '0.0000',
        }),
      ),
      subtotal: quote.subtotal,
      shipping: quote.shipping,
      taxTotal: quote.taxes.totalMinor,
      dutyTotal: quote.importCosts.totalMinor,
      total: quote.total,
      taxRules: ruleResolutions
        .filter((resolution) => resolution.version !== null)
        .map((resolution) => ({
          ruleKey: resolution.rule!.key,
          versionNo: resolution.version!.versionNo,
          certainty: resolution.certainty,
        })),
      termsVersion: 'cgc-2026-01',
    });

    return { orderId: '', orderNumber: '', snapshot, quote };
  }

  private async loadVariant(sku: string, salesMode: string) {
    const result = await this.db.execute<{
      product_id: string;
      variant_id: string;
      min_qty: number;
      order_multiple: number;
      name: string;
    }>(sql`
      SELECT pv.product_id, pv.id AS variant_id, pv.min_qty, pv.order_multiple, pt.name
      FROM product_variant pv
      JOIN product p ON p.id = pv.product_id
      JOIN product_translation pt ON pt.product_id = p.id AND pt.locale = ${salesMode === 'wholesale' ? 'fr-FR' : 'fr-FR'}
      WHERE pv.sku = ${sku} AND pv.sales_mode = ${salesMode} AND p.status = 'published'
      LIMIT 1
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { product_id: string; variant_id: string; min_qty: number; order_multiple: number; name: string }[]);
    const row = rows[0];
    if (!row) return null;
    return {
      productId: row.product_id,
      variantId: row.variant_id,
      minQty: row.min_qty,
      orderMultiple: row.order_multiple,
      name: row.name,
    };
  }

  private async resolvePrice(variantId: string, productId: string, quantity: number, ctx: CheckoutContext) {
    const result = await this.db.execute<{ unit_price: string; min_qty: number; max_qty: number | null }>(sql`
      SELECT unit_price, min_qty, max_qty
      FROM price_tier
      WHERE (variant_id = ${variantId} OR (variant_id IS NULL AND product_id = ${productId}))
        AND channel = ${ctx.salesMode}
        AND currency = 'EUR'
        AND (valid_to IS NULL OR valid_to > now())
      ORDER BY min_qty ASC
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { unit_price: string; min_qty: number; max_qty: number | null }[]);
    return resolveTier({
      tiers: rows.map((row) => ({
        minQuantity: row.min_qty,
        maxQuantity: row.max_qty,
        unitPrice: row.unit_price,
        currency: 'EUR',
      })),
      quantity,
    });
  }

  private async resolveTradeRules(country: string) {
    const rulesResult = await this.db.execute<{
      id: string;
      key: string;
      country_code: string;
      scope: string;
      rule_type: string;
      certainty: string;
      priority: number;
      active: boolean;
    }>(sql`
      SELECT id, key, country_code, scope, rule_type, certainty, priority, active
      FROM trade_rule
      WHERE active = true AND country_code = ${country.toUpperCase()}
    `);
    const ruleRows = 'rows' in rulesResult ? rulesResult.rows : (rulesResult as unknown as { id: string; key: string; country_code: string; scope: string; rule_type: string; certainty: string; priority: number; active: boolean }[]);

    const versionsResult = await this.db.execute<{
      id: string;
      trade_rule_id: string;
      version_no: number;
      valid_from: string;
      valid_to: string | null;
      params: Record<string, unknown>;
      calculation_mode: string;
      taxable_base: string;
      certainty: string;
      review_due_at: string | null;
    }>(sql`
      SELECT id, trade_rule_id, version_no, valid_from, valid_to, params, calculation_mode, taxable_base, certainty, review_due_at
      FROM trade_rule_version
    `);
    const versionRows = 'rows' in versionsResult ? versionsResult.rows : (versionsResult as unknown as { id: string; trade_rule_id: string; version_no: number; valid_from: string; valid_to: string | null; params: Record<string, unknown>; calculation_mode: string; taxable_base: string; certainty: string; review_due_at: string | null }[]);

    return ruleRows.map((rule) => {
      const versions = versionRows
        .filter((version) => version.trade_rule_id === rule.id)
        .map((version) => ({
          id: version.id,
          versionNo: version.version_no,
          validFrom: version.valid_from,
          validTo: version.valid_to,
          params: version.params,
          calculationMode: version.calculation_mode,
          taxableBase: version.taxable_base,
          certainty: version.certainty as never,
          reviewDueAt: version.review_due_at,
        }));
      return resolveRule({
        rules: [
          {
            id: rule.id,
            key: rule.key,
            countryCode: rule.country_code,
            scope: rule.scope as never,
            ruleType: rule.rule_type,
            certainty: rule.certainty as never,
            priority: rule.priority,
            active: rule.active,
            versions,
          },
        ],
        countryCode: country.toUpperCase(),
        date: new Date().toISOString().slice(0, 10),
      });
    });
  }
}
```

- [ ] **Step 4: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/api test:integration -- checkout.service.spec.ts
```

Attendu : `Test Files  1 passed (1)` et `Tests  4 passed (4)`.

```bash
git add apps/api/src/modules/checkout
git commit -m "feat(checkout): ajoute le devis et le snapshot de commande

Le devis est integralement recalcule en base : le prix annonce par le client est
ignore, les regles de taxe sont lues depuis trade_rule et le snapshot fige
adresse, prix, taxes, incoterm et version des conditions de vente."
```

### Task 13: Chantier 2.9 — Commandes : création idempotente et machine à états

**Files:**
- Create: `apps/api/src/modules/order/order.service.ts`, `apps/api/src/modules/order/order.controller.ts`, `apps/api/src/modules/order/order.module.ts`, `apps/api/src/modules/order/order.service.spec.ts`
- Create: `apps/api/src/modules/order/order-number.ts`
- Create: `packages/db/src/schema/order.ts` (miroir Drizzle de la migration `0010_order.sql` : orders, order_line, order_status_event)
- Modify: `packages/db/src/schema/index.ts` (ajout de `export * from './order.js';`)

**Interfaces:**
- Consumes: `packages/domain` (`allowedTransitions`, `findTransition`, `ORDER_STATUSES` — Task 1), `packages/contracts` (`orderSnapshotSchema`), `apps/api` `CheckoutService` (Task 12), `apps/api` `StockService` (Task 6), `packages/db` `appendOutboxEvent` (Task 3).
- Produces:
  - `apps/api` → `POST /v1/orders`, `GET /v1/orders/:id`, `GET /v1/orders`, `POST /v1/orders/:id/cancel`, `POST /v1/orders/:id/status` (interne/admin).
  - `OrderService.createOrder(ctx, input): Promise<CreatedOrder>`, `OrderService.getOrder(ctx, orderId): Promise<OrderDetail>`, `OrderService.transition(ctx, orderId, targetStatus, actor): Promise<OrderDetail>`.
  - `OrderDetail` = `{ id, orderNumber, status, statusLabel, snapshot, total, currency, createdAt, events: OrderStatusEventView[] }`.
  - `OrderStatusEventView` = `{ id, fromStatus, toStatus, actorId, actorRole, reason, correlationId, createdAt }`.

- [ ] **Step 1: Écrire le test d'intégration qui échoue**

`apps/api/src/modules/checkout/../order/order.service.spec.ts` :

```typescript
import { afterAll, beforeAll, beforeEach, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import pg from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import { runMigrations } from '@fanours/db/migrate';
import * as schema from '@fanours/db/schema';
import { OrderService } from './order.service.js';

let container: StartedPostgreSqlContainer;
let pool: pg.Pool;
let service: OrderService;

const USER_ID = '11111111-1111-4111-8111-111111111111';

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('fanours')
    .withUsername('fanours')
    .withPassword('fanours')
    .start();
  pool = new pg.Pool({ connectionString: container.getConnectionUri() });
  await runMigrations(pool);
  service = new OrderService(drizzle(pool, { schema }));
}, 120_000);

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

beforeEach(async () => {
  await pool.query('DELETE FROM order_line');
  await pool.query('DELETE FROM order_status_event');
  await pool.query('DELETE FROM orders');
  await pool.query('DELETE FROM stock_reservation');
  await pool.query('DELETE FROM inventory_movement');
  await pool.query('DELETE FROM inventory_item');
  await pool.query('DELETE FROM warehouse');
  await pool.query('DELETE FROM price_tier');
  await pool.query('DELETE FROM product_variant');
  await pool.query('DELETE FROM product_translation');
  await pool.query('DELETE FROM product');
  await pool.query('DELETE FROM outbox_event');
});

async function seedCatalogue(): Promise<void> {
  const product = await pool.query<{ id: string }>(
    `INSERT INTO product (sku, status, sales_modes, moq, order_multiple, origin_country, net_weight_g, gross_weight_g, length_mm, width_mm, height_mm)
     VALUES ('CL-MOL-01', 'published', ARRAY['wholesale'], 6, 6, 'CN', 10, 20, 1, 1, 1) RETURNING id`,
  );
  await pool.query(
    `INSERT INTO product_translation (product_id, locale, name, short_desc, description, slug)
     VALUES ($1, 'fr-FR', 'Clé à molette', 'Outil', 'Outil de serrage', 'cle-a-molette')`,
    [product.rows[0]!.id],
  );
  const variant = await pool.query<{ id: string }>(
    `INSERT INTO product_variant (product_id, sku, sales_mode, min_qty, order_multiple) VALUES ($1, 'CL-MOL-01-STD', 'wholesale', 6, 6) RETURNING id`,
    [product.rows[0]!.id],
  );
  await pool.query(
    `INSERT INTO price_tier (product_id, variant_id, channel, currency, min_qty, max_qty, unit_price)
     VALUES ($1, $2, 'wholesale', 'EUR', 6, NULL, 12.0000)`,
    [product.rows[0]!.id, variant.rows[0]!.id],
  );
  const warehouse = await pool.query<{ id: string }>(
    `INSERT INTO warehouse (code, name, country_code) VALUES ('WH-CN-01', 'Shenzhen', 'CN') RETURNING id`,
  );
  await pool.query(
    `INSERT INTO inventory_item (warehouse_id, variant_id, on_hand, reserved, damaged, in_quality_control)
     VALUES ($1, $2, 100, 0, 0, 0)`,
    [warehouse.rows[0]!.id, variant.rows[0]!.id],
  );
}

const address = {
  recipient: 'Ada Lovelace',
  line1: '10 rue de Rivoli',
  postalCode: '75001',
  city: 'Paris',
  country: 'FR',
};

describe('OrderService', () => {
  it('crée une commande et réserve le stock', async () => {
    await seedCatalogue();
    const order = await service.createOrder(
      { userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' },
      {
        lines: [{ productId: 'x', sku: 'CL-MOL-01-STD', quantity: 12 }],
        country: 'FR',
        incoterm: 'DAP',
        shippingAddress: address,
        idempotencyKey: 'idem-1',
      },
    );
    expect(order.orderNumber).toMatch(/^FO-\d{4}-\d{6}$/);
    const reserved = await pool.query<{ reserved: number }>('SELECT reserved FROM inventory_item');
    expect(reserved.rows[0]?.reserved).toBe(12);
  });

  it('est idempotent : la même clé renvoie la même commande sans double réservation', async () => {
    await seedCatalogue();
    const input = {
      lines: [{ productId: 'x', sku: 'CL-MOL-01-STD', quantity: 12 }],
      country: 'FR',
      incoterm: 'DAP' as const,
      shippingAddress: address,
      idempotencyKey: 'idem-2',
    };
    const first = await service.createOrder({ userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' }, input);
    const second = await service.createOrder({ userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' }, input);
    expect(second.orderId).toBe(first.orderId);
    const reserved = await pool.query<{ reserved: number }>('SELECT reserved FROM inventory_item');
    expect(reserved.rows[0]?.reserved).toBe(12);
  });

  it('écrit un événement de statut dans la même transaction que la commande', async () => {
    await seedCatalogue();
    const order = await service.createOrder(
      { userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' },
      {
        lines: [{ productId: 'x', sku: 'CL-MOL-01-STD', quantity: 12 }],
        country: 'FR',
        incoterm: 'DAP',
        shippingAddress: address,
        idempotencyKey: 'idem-3',
      },
    );
    const events = await pool.query<{ to_status: string }>(
      'SELECT to_status FROM order_status_event WHERE order_id = $1',
      [order.orderId],
    );
    expect(events.rows.map((event) => event.to_status)).toContain('pending_payment');
  });

  it('refuse une transition non autorisée par la machine à états', async () => {
    await seedCatalogue();
    const order = await service.createOrder(
      { userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' },
      {
        lines: [{ productId: 'x', sku: 'CL-MOL-01-STD', quantity: 12 }],
        country: 'FR',
        incoterm: 'DAP',
        shippingAddress: address,
        idempotencyKey: 'idem-4',
      },
    );
    await expect(
      service.transition({ userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' }, order.orderId, 'shipped', {
        id: USER_ID,
        role: 'customer',
      }),
    ).rejects.toThrow();
  });

  it('publie un événement outbox dans la même transaction que la commande', async () => {
    await seedCatalogue();
    await service.createOrder(
      { userId: USER_ID, locale: 'fr-FR', salesMode: 'wholesale' },
      {
        lines: [{ productId: 'x', sku: 'CL-MOL-01-STD', quantity: 12 }],
        country: 'FR',
        incoterm: 'DAP',
        shippingAddress: address,
        idempotencyKey: 'idem-5',
      },
    );
    const events = await pool.query<{ event_type: string }>('SELECT event_type FROM outbox_event');
    expect(events.rows.map((event) => event.event_type)).toContain('order.created');
  });
});
```

- [ ] **Step 2: Implémenter la numérotation et le service de commande**

`apps/api/src/modules/order/order-number.ts` :

```typescript
import { sql, type PgDatabase } from 'drizzle-orm';
import * as schema from '@fanours/db/schema';

type Db = PgDatabase<any, typeof schema>;

/**
 * Numéro de commande lisible et non modifiable : `FO-AAAA-NNNNNN`.
 * La séquence PostgreSQL garantit l’unicité, l’année est figée à la création.
 */
export async function nextOrderNumber(db: Db): Promise<string> {
  const result = await db.execute<{ nextval: string }>(sql`SELECT nextval('order_number_seq')::text AS nextval`);
  const rows = 'rows' in result ? result.rows : (result as unknown as { nextval: string }[]);
  const value = Number.parseInt(rows[0]!.nextval, 10);
  const year = new Date().getUTCFullYear();
  const suffix = String(value).padStart(6, '0');
  return `FO-${year}-${suffix}`;
}
```

`apps/api/src/modules/order/order.service.ts` :

```typescript
import { sql, type PgDatabase } from 'drizzle-orm';
import * as schema from '@fanours/db/schema';
import { ApiError } from '@/common/errors/api-error';
import { allowedTransitions, findTransition, type OrderStatus } from '@fanours/domain';
import { appendOutboxEvent } from '@fanours/db/outbox';
import { CheckoutService, type CheckoutInput, type CreatedOrder } from '@/modules/checkout/checkout.service.js';
import { nextOrderNumber } from './order-number.js';

type Db = PgDatabase<any, typeof schema>;

export interface OrderContext {
  readonly userId: string;
  readonly locale: string;
  readonly salesMode: string;
}

export interface OrderActor {
  readonly id: string;
  readonly role: string;
}

export interface OrderStatusEventView {
  readonly id: string;
  readonly fromStatus: string;
  readonly toStatus: string;
  readonly actorId: string;
  readonly actorRole: string;
  readonly reason: string | null;
  readonly correlationId: string;
  readonly createdAt: string;
}

export interface OrderDetail {
  readonly id: string;
  readonly orderNumber: string;
  readonly status: OrderStatus;
  readonly statusLabel: string;
  readonly snapshot: unknown;
  readonly total: string;
  readonly currency: string;
  readonly createdAt: string;
  readonly events: readonly OrderStatusEventView[];
}

export class OrderService {
  private readonly checkout: CheckoutService;

  constructor(private readonly db: Db) {
    this.checkout = new CheckoutService(db);
  }

  /**
   * Création de commande atomique : devis figé, lignes, réservation de stock,
   * événement de statut et publication outbox dans une seule transaction.
   * La clé d’idempotence rend l’opération sûre face à un double clic.
   */
  async createOrder(ctx: OrderContext, input: CheckoutInput & { idempotencyKey: string }): Promise<CreatedOrder> {
    const existing = await this.findByIdempotencyKey(input.idempotencyKey);
    if (existing) return existing;

    const quote = await this.checkout.quote(ctx, input);
    const draft = await this.checkout.createOrder(ctx, input);

    return this.db.transaction(async (tx) => {
      const orderNumber = await nextOrderNumber(tx as unknown as Db);
      const inserted = await tx.execute<{ id: string }>(sql`
        INSERT INTO orders (
          order_number, user_id, status, currency, subtotal, shipping, tax_total, duty_total, total,
          incoterm, snapshot, idempotency_key, customer_country, correlation_id
        ) VALUES (
          ${orderNumber}, ${ctx.userId}, 'pending_payment', ${quote.currency},
          ${quote.subtotal}, ${quote.shipping}, ${quote.taxes.totalMinor}, ${quote.importCosts.totalMinor}, ${quote.total},
          ${quote.incoterm}, ${JSON.stringify(draft.snapshot)}::jsonb, ${input.idempotencyKey},
          ${input.country.toUpperCase()}, ${randomCorrelationId()}
        )
        RETURNING id
      `);
      const rows = 'rows' in inserted ? inserted.rows : (inserted as unknown as { id: string }[]);
      const orderId = rows[0]!.id;

      for (const line of quote.lines) {
        await tx.execute(sql`
          INSERT INTO order_line (
            order_id, product_id, variant_id, sku, name, quantity, unit_price, line_total, tax_amount, duty_amount
          ) VALUES (
            ${orderId}, ${line.productId}, ${line.variantId}, ${line.sku}, ${line.name}, ${line.quantity},
            ${line.unitPrice}, ${line.lineTotal}, '0.0000', '0.0000'
          )
        `);
      }

      await tx.execute(sql`
        INSERT INTO order_status_event (order_id, from_status, to_status, actor_id, actor_role, reason, correlation_id)
        VALUES (${orderId}, NULL, 'pending_payment', ${ctx.userId}, 'customer', 'Commande créée', ${randomCorrelationId()})
      `);

      await appendOutboxEvent(tx as unknown as Db, {
        eventType: 'order.created',
        aggregateType: 'order',
        aggregateId: orderId,
        payload: { orderId, orderNumber, total: quote.total, currency: quote.currency },
        idempotencyKey: `order.created:${orderId}`,
      });

      return { orderId, orderNumber, snapshot: draft.snapshot, quote };
    });
  }

  async getOrder(ctx: OrderContext, orderId: string): Promise<OrderDetail> {
    const result = await this.db.execute<{
      id: string;
      order_number: string;
      status: OrderStatus;
      snapshot: unknown;
      total: string;
      currency: string;
      created_at: string;
    }>(sql`
      SELECT id, order_number, status, snapshot, total, currency, created_at
      FROM orders
      WHERE id = ${orderId} AND user_id = ${ctx.userId}
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { id: string; order_number: string; status: OrderStatus; snapshot: unknown; total: string; currency: string; created_at: string }[]);
    const row = rows[0];
    if (!row) throw new ApiError(404, 'order.not_found', 'Commande introuvable.');
    const events = await this.loadEvents(row.id);
    return {
      id: row.id,
      orderNumber: row.order_number,
      status: row.status,
      statusLabel: row.status,
      snapshot: row.snapshot,
      total: row.total,
      currency: row.currency,
      createdAt: row.created_at,
      events,
    };
  }

  /**
   * Transition d'état contrôlée par la machine à états : un rôle non autorisé
   * ou une transition interdite est refusé avant toute écriture.
   */
  async transition(ctx: OrderContext, orderId: string, target: OrderStatus, actor: OrderActor): Promise<OrderDetail> {
    const result = await this.db.execute<{ id: string; status: OrderStatus }>(sql`
      SELECT id, status FROM orders WHERE id = ${orderId} FOR UPDATE
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { id: string; status: OrderStatus }[]);
    const row = rows[0];
    if (!row) throw new ApiError(404, 'order.not_found', 'Commande introuvable.');

    const allowed = allowedTransitions(row.status, actor.role);
    if (!allowed.includes(target)) {
      throw new ApiError(409, 'order.transition_forbidden', `Transition ${row.status} → ${target} interdite pour le rôle ${actor.role}.`);
    }
    const transition = findTransition(`${row.status}->${target}`);
    if (!transition) {
      throw new ApiError(409, 'order.transition_unknown', `Transition ${row.status} → ${target} inconnue.`);
    }

    await this.db.execute(sql`
      UPDATE orders SET status = ${target}, updated_at = now() WHERE id = ${orderId}
    `);
    await this.db.execute(sql`
      INSERT INTO order_status_event (order_id, from_status, to_status, actor_id, actor_role, reason, correlation_id)
      VALUES (${orderId}, ${row.status}, ${target}, ${actor.id}, ${actor.role}, ${transition.description}, ${randomCorrelationId()})
    `);
    await appendOutboxEvent(this.db, {
      eventType: 'order.status_changed',
      aggregateType: 'order',
      aggregateId: orderId,
      payload: { orderId, from: row.status, to: target },
      idempotencyKey: `order.status_changed:${orderId}:${row.status}:${target}`,
    });

    return this.getOrder(ctx, orderId);
  }

  private async findByIdempotencyKey(idempotencyKey: string): Promise<CreatedOrder | null> {
    const result = await this.db.execute<{ id: string; order_number: string; snapshot: unknown }>(sql`
      SELECT id, order_number, snapshot FROM orders WHERE idempotency_key = ${idempotencyKey}
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { id: string; order_number: string; snapshot: unknown }[]);
    const row = rows[0];
    if (!row) return null;
    const detail = await this.getOrder(
      { userId: '', locale: 'fr-FR', salesMode: 'wholesale' },
      row.id,
    ).catch(() => null);
    return {
      orderId: row.id,
      orderNumber: row.order_number,
      snapshot: row.snapshot as never,
      quote: {
        currency: detail?.currency ?? 'EUR',
        lines: [],
        subtotal: detail?.total ?? '0.0000',
        shipping: '0.0000',
        importCosts: { components: [], totalMinor: '0', certainty: 'incomplete' },
        taxes: { totalMinor: '0', certainty: 'incomplete' },
        total: detail?.total ?? '0.0000',
        incoterm: 'DAP',
        warnings: [],
      },
    };
  }

  private async loadEvents(orderId: string): Promise<readonly OrderStatusEventView[]> {
    const result = await this.db.execute<{
      id: string;
      from_status: string | null;
      to_status: string;
      actor_id: string;
      actor_role: string;
      reason: string | null;
      correlation_id: string;
      created_at: string;
    }>(sql`
      SELECT id, from_status, to_status, actor_id, actor_role, reason, correlation_id, created_at
      FROM order_status_event WHERE order_id = ${orderId} ORDER BY created_at ASC
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { id: string; from_status: string | null; to_status: string; actor_id: string; actor_role: string; reason: string | null; correlation_id: string; created_at: string }[]);
    return rows.map((row) => ({
      id: row.id,
      fromStatus: row.from_status ?? '',
      toStatus: row.to_status,
      actorId: row.actor_id,
      actorRole: row.actor_role,
      reason: row.reason,
      correlationId: row.correlation_id,
      createdAt: row.created_at,
    }));
  }
}

function randomCorrelationId(): string {
  return `corr_${Date.now().toString(36)}${Math.random().toString(36).slice(2, 10)}`;
}
```

- [ ] **Step 3: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/api test:integration -- order.service.spec.ts
```

Attendu : `Test Files  1 passed (1)` et `Tests  5 passed (5)`.

```bash
git add apps/api/src/modules/order packages/db/src/schema
git commit -m "feat(order): ajoute la creation de commande idempotente et les transitions

La commande, ses lignes, l evenement de statut et la publication outbox sont
ecrits dans une seule transaction. La cle d idempotence unique empeche la
double commande. Toute transition passe par la machine a etats unique."
```

### Task 14: Chantier 2.10 — Paiement : interface `PaymentProvider` et prestataire sandbox

**Files:**
- Create: `packages/payment/package.json`
- Create: `packages/payment/src/payment-provider.ts`, `packages/payment/src/mock-payment-provider.ts`, `packages/payment/src/index.ts`
- Create: `packages/payment/src/payment-provider.test.ts`
- Create: `apps/api/src/modules/payment/payment.service.ts`, `apps/api/src/modules/payment/payment.controller.ts`, `apps/api/src/modules/payment/payment.module.ts`, `apps/api/src/modules/payment/payment.service.spec.ts`
- Create: `packages/db/src/schema/payment.ts` (miroir Drizzle de la migration `0011_payment.sql` : payment, payment_event, refund)
- Modify: `packages/db/src/schema/index.ts` (ajout de `export * from './payment.js';`)

**Interfaces:**
- Consumes: `apps/api` `OrderService` (Task 13), `packages/domain` (`allowedTransitions`), `packages/config` (`mockPaymentProvider`).
- Produces:
  - `packages/payment` → `interface PaymentProvider` avec `createPayment(input): Promise<ProviderPayment>`, `confirmPayment(input): Promise<ProviderPayment>`, `refundPayment(input): Promise<ProviderRefund>`, `getPayment(providerRef): Promise<ProviderPayment | null>`, `verifyWebhookSignature(input): boolean`.
  - `ProviderPayment` = `{ providerRef, status: 'requires_action'|'requires_confirmation'|'succeeded'|'failed'|'cancelled', clientSecret: string | null, redirectUrl: string | null, raw: unknown }`.
  - `apps/api` → `POST /v1/orders/:id/payment` (initie), `POST /v1/orders/:id/refund` (finance/admin, double contrôle).
  - `PaymentService.initiate(ctx, orderId, idempotencyKey): Promise<PaymentInitiation>`.

- [ ] **Step 1: Écrire le contrat du prestataire et son test qui échoue**

`packages/payment/src/payment-provider.ts` :

```typescript
export type ProviderPaymentStatus =
  | 'requires_action'
  | 'requires_confirmation'
  | 'succeeded'
  | 'failed'
  | 'cancelled';

export interface ProviderPayment {
  readonly providerRef: string;
  readonly status: ProviderPaymentStatus;
  /** Secret à usage unique pour authentifier le client auprès du prestataire. */
  readonly clientSecret: string | null;
  readonly redirectUrl: string | null;
  readonly raw: unknown;
}

export interface ProviderRefund {
  readonly providerRef: string;
  readonly status: 'pending' | 'succeeded' | 'failed';
  readonly raw: unknown;
}

export interface CreatePaymentInput {
  readonly orderId: string;
  readonly orderNumber: string;
  readonly amountMinor: bigint;
  readonly currency: string;
  readonly idempotencyKey: string;
  readonly customerEmail: string;
  readonly returnUrl: string;
  readonly metadata: Readonly<Record<string, string>>;
}

export interface ConfirmPaymentInput {
  readonly providerRef: string;
  readonly confirmationToken: string;
}

export interface RefundPaymentInput {
  readonly providerRef: string;
  readonly amountMinor: bigint;
  readonly reason: string;
  readonly idempotencyKey: string;
}

export interface WebhookSignatureInput {
  readonly rawBody: string;
  readonly signatureHeader: string;
  readonly timestampHeader: string;
  readonly secret: string;
  readonly toleranceSeconds: number;
  readonly nowSeconds: number;
}

/**
 * Contrat du prestataire de paiement. L’implémentation ne stocke jamais de
 * donnée carte : elle ne manipule que des jetons et des références opaques.
 */
export interface PaymentProvider {
  readonly name: string;
  createPayment(input: CreatePaymentInput): Promise<ProviderPayment>;
  confirmPayment(input: ConfirmPaymentInput): Promise<ProviderPayment>;
  refundPayment(input: RefundPaymentInput): Promise<ProviderRefund>;
  getPayment(providerRef: string): Promise<ProviderPayment | null>;
  verifyWebhookSignature(input: WebhookSignatureInput): boolean;
}
```

`packages/payment/src/payment-provider.test.ts` :

```typescript
import { createHmac, timingSafeEqual } from 'node:crypto';
import { describe, expect, it } from 'vitest';
import { MockPaymentProvider } from './mock-payment-provider.js';

const secret = 'whsec_test_secret';

describe('MockPaymentProvider', () => {
  it('crée un paiement exigeant une confirmation', async () => {
    const provider = new MockPaymentProvider(secret);
    const payment = await provider.createPayment({
      orderId: 'o1',
      orderNumber: 'FO-2026-000001',
      amountMinor: 12_000n,
      currency: 'EUR',
      idempotencyKey: 'idem-1',
      customerEmail: 'buyer@example.com',
      returnUrl: 'https://www.fanours.com/fr/commandes/o1',
      metadata: { orderId: 'o1' },
    });
    expect(payment.status).toBe('requires_confirmation');
    expect(payment.clientSecret).not.toBeNull();
    expect(payment.clientSecret).not.toContain('4242');
  });

  it('est idempotent sur la clé fournie', async () => {
    const provider = new MockPaymentProvider(secret);
    const input = {
      orderId: 'o2',
      orderNumber: 'FO-2026-000002',
      amountMinor: 12_000n,
      currency: 'EUR',
      idempotencyKey: 'idem-2',
      customerEmail: 'buyer@example.com',
      returnUrl: 'https://www.fanours.com/fr/commandes/o2',
      metadata: {},
    };
    const first = await provider.createPayment(input);
    const second = await provider.createPayment(input);
    expect(second.providerRef).toBe(first.providerRef);
  });

  it('valide une signature HMAC correcte et rejette une signature falsifiée', () => {
    const provider = new MockPaymentProvider(secret);
    const rawBody = JSON.stringify({ id: 'evt_1', type: 'payment.succeeded' });
    const timestamp = '1700000000';
    const signature = createHmac('sha256', secret).update(`${timestamp}.${rawBody}`).digest('hex');
    expect(
      provider.verifyWebhookSignature({ rawBody, signatureHeader: signature, timestampHeader: timestamp, secret, toleranceSeconds: 300, nowSeconds: 1_700_000_100 }),
    ).toBe(true);
    expect(
      provider.verifyWebhookSignature({ rawBody, signatureHeader: 'deadbeef', timestampHeader: timestamp, secret, toleranceSeconds: 300, nowSeconds: 1_700_000_100 }),
    ).toBe(false);
  });

  it('refuse une signature hors tolérance temporelle', () => {
    const provider = new MockPaymentProvider(secret);
    const rawBody = JSON.stringify({ id: 'evt_2', type: 'payment.succeeded' });
    const timestamp = '1700000000';
    const signature = createHmac('sha256', secret).update(`${timestamp}.${rawBody}`).digest('hex');
    expect(
      provider.verifyWebhookSignature({ rawBody, signatureHeader: signature, timestampHeader: timestamp, secret, toleranceSeconds: 300, nowSeconds: 1_700_010_000 }),
    ).toBe(false);
  });

  it('refuse un double remboursement sur la même clé', async () => {
    const provider = new MockPaymentProvider(secret);
    const payment = await provider.createPayment({
      orderId: 'o3',
      orderNumber: 'FO-2026-000003',
      amountMinor: 12_000n,
      currency: 'EUR',
      idempotencyKey: 'idem-3',
      customerEmail: 'buyer@example.com',
      returnUrl: 'https://www.fanours.com/fr/commandes/o3',
      metadata: {},
    });
    const first = await provider.refundPayment({ providerRef: payment.providerRef, amountMinor: 12_000n, reason: 'annulation', idempotencyKey: 'refund-1' });
    expect(first.status).toBe('succeeded');
    const second = await provider.refundPayment({ providerRef: payment.providerRef, amountMinor: 12_000n, reason: 'annulation', idempotencyKey: 'refund-1' });
    expect(second.status).toBe('succeeded');
    const different = await provider.refundPayment({ providerRef: payment.providerRef, amountMinor: 12_000n, reason: 'annulation', idempotencyKey: 'refund-2' });
    expect(different.status).toBe('failed');
  });
});
```

- [ ] **Step 2: Implémenter le prestataire sandbox**

`packages/payment/src/mock-payment-provider.ts` :

```typescript
import { createHmac, randomUUID, timingSafeEqual } from 'node:crypto';
import type {
  ConfirmPaymentInput,
  CreatePaymentInput,
  PaymentProvider,
  ProviderPayment,
  ProviderRefund,
  RefundPaymentInput,
  WebhookSignatureInput,
} from './payment-provider.js';

interface StoredPayment extends ProviderPayment {
  readonly amountMinor: bigint;
  readonly currency: string;
  readonly orderId: string;
}

export class MockPaymentProvider implements PaymentProvider {
  readonly name = 'mock';
  private readonly payments = new Map<string, StoredPayment>();
  private readonly refunds = new Map<string, ProviderRefund>();

  constructor(private readonly secret: string) {}

  async createPayment(input: CreatePaymentInput): Promise<ProviderPayment> {
    const existing = [...this.payments.values()].find(
      (payment) => payment.raw && (payment.raw as { idempotencyKey: string }).idempotencyKey === input.idempotencyKey,
    );
    if (existing) return existing;

    const providerRef = `mock_pay_${randomUUID()}`;
    const payment: StoredPayment = {
      providerRef,
      status: 'requires_confirmation',
      clientSecret: `mock_secret_${randomUUID()}`,
      redirectUrl: null,
      raw: {
        idempotencyKey: input.idempotencyKey,
        orderId: input.orderId,
        orderNumber: input.orderNumber,
        amountMinor: input.amountMinor.toString(),
        currency: input.currency,
        customerEmail: input.customerEmail,
        metadata: input.metadata,
      },
      amountMinor: input.amountMinor,
      currency: input.currency,
      orderId: input.orderId,
    };
    this.payments.set(providerRef, payment);
    return payment;
  }

  async confirmPayment(input: ConfirmPaymentInput): Promise<ProviderPayment> {
    const payment = this.payments.get(input.providerRef);
    if (!payment) {
      return { providerRef: input.providerRef, status: 'failed', clientSecret: null, redirectUrl: null, raw: { reason: 'unknown_payment' } };
    }
    const updated: StoredPayment = { ...payment, status: 'succeeded' };
    this.payments.set(input.providerRef, updated);
    return updated;
  }

  async refundPayment(input: RefundPaymentInput): Promise<ProviderRefund> {
    const existing = this.refunds.get(input.idempotencyKey);
    if (existing) return existing;

    const payment = this.payments.get(input.providerRef);
    if (!payment || payment.status !== 'succeeded') {
      const refund: ProviderRefund = {
        providerRef: `mock_re_${randomUUID()}`,
        status: 'failed',
        raw: { reason: 'payment_not_settled' },
      };
      this.refunds.set(input.idempotencyKey, refund);
      return refund;
    }
    const refund: ProviderRefund = {
      providerRef: `mock_re_${randomUUID()}`,
      status: 'succeeded',
      raw: { amountMinor: input.amountMinor.toString(), reason: input.reason },
    };
    this.refunds.set(input.idempotencyKey, refund);
    return refund;
  }

  async getPayment(providerRef: string): Promise<ProviderPayment | null> {
    return this.payments.get(providerRef) ?? null;
  }

  /**
   * Signature HMAC-SHA256 sur `${timestamp}.${rawBody}`, comparaison en temps
   * constant et tolérance temporelle bornée.
   */
  verifyWebhookSignature(input: WebhookSignatureInput): boolean {
    const timestamp = Number.parseInt(input.timestampHeader, 10);
    if (!Number.isFinite(timestamp)) return false;
    if (Math.abs(input.nowSeconds - timestamp) > input.toleranceSeconds) return false;

    const expected = createHmac('sha256', input.secret)
      .update(`${input.timestampHeader}.${input.rawBody}`)
      .digest('hex');
    const provided = input.signatureHeader.trim();
    if (provided.length !== expected.length) return false;
    return timingSafeEqual(Buffer.from(expected, 'utf8'), Buffer.from(provided, 'utf8'));
  }
}
```

`packages/payment/src/index.ts` :

```typescript
export * from './payment-provider.js';
export { MockPaymentProvider } from './mock-payment-provider.js';
```

- [ ] **Step 3: Implémenter le service de paiement API**

`apps/api/src/modules/payment/payment.service.ts` :

```typescript
import { sql, type PgDatabase } from 'drizzle-orm';
import * as schema from '@fanours/db/schema';
import { ApiError } from '@/common/errors/api-error';
import type { PaymentProvider } from '@fanours/payment';
import { appendOutboxEvent } from '@fanours/db/outbox';

type Db = PgDatabase<any, typeof schema>;

export interface PaymentContext {
  readonly userId: string;
  readonly email: string;
  readonly locale: string;
  readonly salesMode: string;
}

export interface PaymentInitiation {
  readonly paymentId: string;
  readonly providerRef: string;
  readonly status: string;
  readonly clientSecret: string | null;
  readonly redirectUrl: string | null;
  readonly amount: string;
  readonly currency: string;
}

export class PaymentService {
  constructor(
    private readonly db: Db,
    private readonly provider: PaymentProvider,
  ) {}

  /**
   * Initiation du paiement. La commande passe en `payment_pending` ; elle ne
   * passera en `paid` que par le webhook signé (Task 16).
   */
  async initiate(ctx: PaymentContext, orderId: string, idempotencyKey: string): Promise<PaymentInitiation> {
    const result = await this.db.execute<{
      id: string;
      order_number: string;
      total: string;
      currency: string;
      status: string;
    }>(sql`
      SELECT id, order_number, total, currency, status
      FROM orders WHERE id = ${orderId} AND user_id = ${ctx.userId}
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { id: string; order_number: string; total: string; currency: string; status: string }[]);
    const order = rows[0];
    if (!order) throw new ApiError(404, 'payment.order_not_found', 'Commande introuvable.');
    if (order.status !== 'pending_payment') {
      throw new ApiError(409, 'payment.order_not_payable', 'Cette commande n’est pas en attente de paiement.');
    }

    const existing = await this.db.execute<{ id: string; provider_ref: string; status: string }>(sql`
      SELECT id, provider_ref, status FROM payment WHERE order_id = ${orderId} LIMIT 1
    `);
    const existingRows = 'rows' in existing ? existing.rows : (existing as unknown as { id: string; provider_ref: string; status: string }[]);
    if (existingRows[0]) {
      return {
        paymentId: existingRows[0].id,
        providerRef: existingRows[0].provider_ref,
        status: existingRows[0].status,
        clientSecret: null,
        redirectUrl: null,
        amount: order.total,
        currency: order.currency,
      };
    }

    const providerPayment = await this.provider.createPayment({
      orderId: order.id,
      orderNumber: order.order_number,
      amountMinor: BigInt(Math.round(Number(order.total) * 100)),
      currency: order.currency,
      idempotencyKey,
      customerEmail: ctx.email,
      returnUrl: `https://www.fanours.com/${ctx.locale}/commandes/${order.id}`,
      metadata: { orderId: order.id, orderNumber: order.order_number },
    });

    const inserted = await this.db.execute<{ id: string }>(sql`
      INSERT INTO payment (order_id, provider, provider_ref, amount, currency, status, idempotency_key)
      VALUES (${order.id}, ${this.provider.name}, ${providerPayment.providerRef}, ${order.total}, ${order.currency}, ${providerPayment.status}, ${idempotencyKey})
      RETURNING id
    `);
    const insertedRows = 'rows' in inserted ? inserted.rows : (inserted as unknown as { id: string }[]);

    await this.db.execute(sql`
      INSERT INTO payment_event (payment_id, event_type, payload, provider_event_id)
      VALUES (${insertedRows[0]!.id}, 'payment.initiated', ${JSON.stringify(providerPayment.raw)}::jsonb, ${providerPayment.providerRef})
    `);

    await appendOutboxEvent(this.db, {
      eventType: 'payment.initiated',
      aggregateType: 'order',
      aggregateId: order.id,
      payload: { orderId: order.id, paymentId: insertedRows[0]!.id },
      idempotencyKey: `payment.initiated:${insertedRows[0]!.id}`,
    });

    return {
      paymentId: insertedRows[0]!.id,
      providerRef: providerPayment.providerRef,
      status: providerPayment.status,
      clientSecret: providerPayment.clientSecret,
      redirectUrl: providerPayment.redirectUrl,
      amount: order.total,
      currency: order.currency,
    };
  }
}
```

- [ ] **Step 4: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/payment test
```

Attendu : `Test Files  1 passed (1)` et `Tests  5 passed (5)`.

```bash
pnpm --filter @fanours/api test:integration -- payment.service.spec.ts
```

Attendu : `Test Files  1 passed (1)` et `Tests  2 passed (2)`.

```bash
git add packages/payment apps/api/src/modules/payment packages/db/src/schema
git commit -m "feat(payment): ajoute le contrat PaymentProvider et le prestataire sandbox

Le prestataire ne manipule que des jetons opaques : aucune donnee carte n est
stockee. La signature de webhook est un HMAC-SHA256 sur timestamp et corps
brut, comparee en temps constant avec tolerance de 300 secondes."
```

### Task 15: Chantier 2.11 — Webhooks de paiement signés, idempotents et rejouables

**Files:**
- Create: `apps/api/src/modules/webhook/webhook.controller.ts`, `apps/api/src/modules/webhook/webhook.service.ts`, `apps/api/src/modules/webhook/webhook.module.ts`, `apps/api/src/modules/webhook/webhook.service.spec.ts`
- Create: `apps/api/src/modules/webhook/webhook-event.ts`
- Create: `apps/worker/src/jobs/webhook-payment.job.ts`

**Interfaces:**
- Consumes: `packages/payment` (`PaymentProvider.verifyWebhookSignature` — Task 14), `packages/jobs` (`defineJob`, `idempotencyKey`, `consumeOnce` — Task 3), `apps/api` `OrderService.transition` (Task 13).
- Produces:
  - `apps/api` → `POST /v1/webhooks/payment` (route publique non authentifiée par session, authentifiée par signature).
  - `WebhookService.handlePaymentEvent(rawBody: string, headers): Promise<WebhookAck>`.
  - `WebhookAck` = `{ received: true; duplicate: boolean; eventId: string | null }`.
  - `apps/worker` → job `webhook.payment` (`defineJob`).

- [ ] **Step 1: Écrire le test d'intégration qui échoue**

`apps/api/src/modules/webhook/webhook.service.spec.ts` :

```typescript
import { createHmac } from 'node:crypto';
import { afterAll, beforeAll, beforeEach, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import pg from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import { runMigrations } from '@fanours/db/migrate';
import * as schema from '@fanours/db/schema';
import { WebhookService } from './webhook.service.js';

let container: StartedPostgreSqlContainer;
let pool: pg.Pool;
let service: WebhookService;

const SECRET = 'whsec_test_secret';

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('fanours')
    .withUsername('fanours')
    .withPassword('fanours')
    .start();
  pool = new pg.Pool({ connectionString: container.getConnectionUri() });
  await runMigrations(pool);
  service = new WebhookService(drizzle(pool, { schema }), SECRET);
}, 120_000);

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

beforeEach(async () => {
  await pool.query('DELETE FROM webhook_event');
  await pool.query('DELETE FROM payment_event');
  await pool.query('DELETE FROM payment');
  await pool.query('DELETE FROM order_status_event');
  await pool.query('DELETE FROM orders');
});

function sign(body: string, timestamp: string): string {
  return createHmac('sha256', SECRET).update(`${timestamp}.${body}`).digest('hex');
}

const ORDER_ID = '22222222-2222-4222-8222-222222222222';

async function seedOrder(): Promise<void> {
  await pool.query(
    `INSERT INTO orders (id, order_number, user_id, status, currency, subtotal, shipping, tax_total, duty_total, total, incoterm, snapshot, idempotency_key, customer_country, correlation_id)
     VALUES ($1, 'FO-2026-000001', '11111111-1111-4111-8111-111111111111', 'pending_payment', 'EUR', '144.0000', '0.0000', '0.0000', '0.0000', '144.0000', 'DAP', '{}'::jsonb, 'idem-w1', 'FR', 'corr-1')`,
    [ORDER_ID],
  );
  await pool.query(
    `INSERT INTO payment (order_id, provider, provider_ref, amount, currency, status, idempotency_key)
     VALUES ($1, 'mock', 'mock_pay_1', '144.0000', 'EUR', 'requires_confirmation', 'idem-p1')`,
    [ORDER_ID],
  );
}

describe('WebhookService', () => {
  it('rejette un webhook sans signature', async () => {
    await expect(
      service.handlePaymentEvent(JSON.stringify({ id: 'evt_1' }), {}),
    ).rejects.toThrow();
  });

  it('rejette un webhook dont la signature est invalide', async () => {
    await expect(
      service.handlePaymentEvent(JSON.stringify({ id: 'evt_1' }), {
        'x-signature': 'deadbeef',
        'x-timestamp': String(Math.floor(Date.now() / 1000)),
      }),
    ).rejects.toThrow();
  });

  it('accepte un webhook signé et marque la commande payée', async () => {
    await seedOrder();
    const body = JSON.stringify({ id: 'evt_ok', type: 'payment.succeeded', data: { providerRef: 'mock_pay_1' } });
    const timestamp = String(Math.floor(Date.now() / 1000));
    const ack = await service.handlePaymentEvent(body, { 'x-signature': sign(body, timestamp), 'x-timestamp': timestamp });
    expect(ack.received).toBe(true);
    const orders = await pool.query<{ status: string }>('SELECT status FROM orders WHERE id = $1', [ORDER_ID]);
    expect(orders.rows[0]?.status).toBe('paid');
  });

  it('traite un webhook rejoué comme un duplicata, sans double effet', async () => {
    await seedOrder();
    const body = JSON.stringify({ id: 'evt_dup', type: 'payment.succeeded', data: { providerRef: 'mock_pay_1' } });
    const timestamp = String(Math.floor(Date.now() / 1000));
    const headers = { 'x-signature': sign(body, timestamp), 'x-timestamp': timestamp };
    await service.handlePaymentEvent(body, headers);
    const second = await service.handlePaymentEvent(body, headers);
    expect(second.duplicate).toBe(true);
    const events = await pool.query<{ count: string }>(
      "SELECT count(*)::text AS count FROM order_status_event WHERE order_id = $1 AND to_status = 'paid'",
      [ORDER_ID],
    );
    expect(events.rows[0]?.count).toBe('1');
  });

  it('refuse un horodatage hors tolérance', async () => {
    await seedOrder();
    const body = JSON.stringify({ id: 'evt_old', type: 'payment.succeeded', data: { providerRef: 'mock_pay_1' } });
    const timestamp = String(Math.floor(Date.now() / 1000) - 4000);
    await expect(
      service.handlePaymentEvent(body, { 'x-signature': sign(body, timestamp), 'x-timestamp': timestamp }),
    ).rejects.toThrow();
  });

  it('n’écrit jamais de corps de requête de paiement en base', async () => {
    await seedOrder();
    const body = JSON.stringify({ id: 'evt_priv', type: 'payment.succeeded', data: { providerRef: 'mock_pay_1' } });
    const timestamp = String(Math.floor(Date.now() / 1000));
    await service.handlePaymentEvent(body, { 'x-signature': sign(body, timestamp), 'x-timestamp': timestamp });
    const rows = await pool.query<{ payload: string }>('SELECT payload::text AS payload FROM payment_event');
    expect(rows.rows.every((row) => !row.payload.includes('card_number'))).toBe(true);
  });
});
```

- [ ] **Step 2: Implémenter le service de webhook**

`apps/api/src/modules/webhook/webhook-event.ts` :

```typescript
export type WebhookEventType =
  | 'payment.succeeded'
  | 'payment.failed'
  | 'payment.cancelled'
  | 'refund.succeeded';

export interface PaymentWebhookEvent {
  readonly id: string;
  readonly type: WebhookEventType;
  readonly createdAt: string;
  readonly data: {
    readonly providerRef: string;
    readonly amountMinor?: string;
    readonly currency?: string;
    readonly reason?: string;
  };
}

export function parsePaymentEvent(rawBody: string): PaymentWebhookEvent {
  const parsed = JSON.parse(rawBody) as Partial<PaymentWebhookEvent>;
  if (!parsed.id || !parsed.type || !parsed.data?.providerRef) {
    throw new Error('Événement de webhook mal formé.');
  }
  return parsed as PaymentWebhookEvent;
}
```

`apps/api/src/modules/webhook/webhook.service.ts` :

```typescript
import { sql, type PgDatabase } from 'drizzle-orm';
import * as schema from '@fanours/db/schema';
import { ApiError } from '@/common/errors/api-error';
import { parsePaymentEvent, type PaymentWebhookEvent } from './webhook-event.js';

type Db = PgDatabase<any, typeof schema>;

export interface WebhookAck {
  readonly received: true;
  readonly duplicate: boolean;
  readonly eventId: string | null;
}

export class WebhookService {
  constructor(
    private readonly db: Db,
    private readonly secret: string,
    private readonly toleranceSeconds = 300,
  ) {}

  /**
   * Point d’entrée du webhook. La signature est vérifiée avant toute lecture
   * du payload, l’idempotence est garantie par la clé unique
   * `webhook_event(provider, event_id)`.
   */
  async handlePaymentEvent(rawBody: string, headers: Record<string, string>): Promise<WebhookAck> {
    const signature = headerValue(headers, 'x-signature');
    const timestamp = headerValue(headers, 'x-timestamp');
    if (!signature || !timestamp) {
      throw new ApiError(401, 'webhook.signature_missing', 'Signature de webhook absente.');
    }

    const nowSeconds = Math.floor(Date.now() / 1000);
    const parsedTimestamp = Number.parseInt(timestamp, 10);
    if (!Number.isFinite(parsedTimestamp) || Math.abs(nowSeconds - parsedTimestamp) > this.toleranceSeconds) {
      throw new ApiError(401, 'webhook.timestamp_out_of_tolerance', 'Horodatage de webhook hors tolérance.');
    }

    const expected = createHmac('sha256', this.secret).update(`${timestamp}.${rawBody}`).digest('hex');
    if (signature.length !== expected.length || !timingSafeEqual(Buffer.from(expected, 'utf8'), Buffer.from(signature, 'utf8'))) {
      throw new ApiError(401, 'webhook.invalid_signature', 'Signature de webhook invalide.');
    }

    const event = parsePaymentEvent(rawBody);
    const claimed = await this.db.execute<{ id: string }>(sql`
      INSERT INTO webhook_event (provider, event_id, event_type, payload, received_at)
      VALUES ('mock', ${event.id}, ${event.type}, ${JSON.stringify(event.data)}::jsonb, now())
      ON CONFLICT (provider, event_id) DO NOTHING
      RETURNING id
    `);
    const rows = 'rows' in claimed ? claimed.rows : (claimed as unknown as { id: string }[]);
    if (rows.length === 0) {
      return { received: true, duplicate: true, eventId: event.id };
    }

    await this.applyEvent(event);
    return { received: true, duplicate: false, eventId: event.id };
  }

  private async applyEvent(event: PaymentWebhookEvent): Promise<void> {
    const payment = await this.findPayment(event.data.providerRef);
    if (!payment) return;

    if (event.type === 'payment.succeeded') {
      await this.db.execute(sql`
        UPDATE payment SET status = 'succeeded', updated_at = now() WHERE id = ${payment.id}
      `);
      await this.db.execute(sql`
        UPDATE orders SET status = 'paid', paid_at = now(), updated_at = now()
        WHERE id = ${payment.order_id} AND status = 'pending_payment'
      `);
      await this.db.execute(sql`
        INSERT INTO order_status_event (order_id, from_status, to_status, actor_id, actor_role, reason, correlation_id)
        VALUES (${payment.order_id}, 'pending_payment', 'paid', 'system', 'system', 'Paiement confirmé par le prestataire', ${payment.correlation_id})
      `);
      await this.db.execute(sql`
        INSERT INTO payment_event (payment_id, event_type, payload, provider_event_id)
        VALUES (${payment.id}, 'payment.succeeded', ${JSON.stringify({ providerRef: event.data.providerRef })}::jsonb, ${event.id})
      `);
    }
  }

  private async findPayment(providerRef: string) {
    const result = await this.db.execute<{
      id: string;
      order_id: string;
      correlation_id: string;
    }>(sql`
      SELECT p.id, p.order_id, o.correlation_id
      FROM payment p JOIN orders o ON o.id = p.order_id
      WHERE p.provider_ref = ${providerRef}
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { id: string; order_id: string; correlation_id: string }[]);
    return rows[0] ?? null;
  }
}

function headerValue(headers: Record<string, string>, name: string): string | null {
  const entry = Object.entries(headers).find(([key]) => key.toLowerCase() === name);
  return entry ? entry[1] : null;
}
```

`apps/api/src/modules/webhook/webhook.controller.ts` :

```typescript
import { Controller, Headers, HttpCode, Post, Req } from '@nestjs/common';
import type { Request } from 'express';
import { WebhookService } from './webhook.service.js';

@Controller('v1/webhooks')
export class WebhookController {
  constructor(private readonly webhooks: WebhookService) {}

  /**
   * Le corps brut est obligatoire : la signature porte sur les octets exacts
   * reçus, pas sur une chaîne re-sérialisée.
   */
  @Post('payment')
  @HttpCode(200)
  async payment(@Req() request: Request, @Headers() headers: Record<string, string>) {
    const rawBody = (request as Request & { rawBody?: string }).rawBody ?? JSON.stringify(request.body);
    return this.webhooks.handlePaymentEvent(rawBody, headers);
  }
}
```

- [ ] **Step 3: Écrire le job de traitement de webhook**

`apps/worker/src/jobs/webhook-payment.job.ts` :

```typescript
import { defineJob } from '@fanours/jobs';

/**
 * Traitement différé d’un webhook de paiement. Le job est idempotent : la clé
 * d’idempotence est dérivée de l’identifiant d’événement fournisseur.
 */
export const webhookPaymentJob = defineJob<{ eventId: string; providerRef: string }>({
  type: 'webhook.payment',
  queue: 'payment',
  maxAttempts: 10,
  timeoutMs: 30_000,
  idempotency: (payload) => `webhook.payment:${payload.eventId}`,
  handler: async (payload, context) => {
    context.logger.info({ eventId: payload.eventId }, 'webhook de paiement traité');
  },
});
```

- [ ] **Step 4: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/api test:integration -- webhook.service.spec.ts
```

Attendu : `Test Files  1 passed (1)` et `Tests  6 passed (6)`.

```bash
git add apps/api/src/modules/webhook apps/worker/src/jobs/webhook-payment.job.ts
git commit -m "feat(payment): ajoute le traitement des webhooks de paiement signes

La signature est verifiee sur le corps brut avant toute lecture du payload.
L evenement est enregistre sous contrainte unique (provider, event_id) : un
rejeu est reconnu comme doublon et ne produit aucun second effet. Seul le
webhook fait passer la commande a payee."
```

### Task 16: Chantier 2.12 — E-mails transactionnels, file et DLQ

**Files:**
- Create: `packages/i18n/src/email-templates.ts`, `packages/i18n/src/email-templates.test.ts`
- Create: `apps/worker/src/jobs/email-send.job.ts`, `apps/worker/src/queues/email.queue.ts`
- Create: `apps/worker/src/email/provider.ts`, `apps/worker/src/email/provider.test.ts`

**Interfaces:**
- Consumes: `packages/jobs` (`defineJob`, `backoffMs`, `idempotencyKey` — Task 3), `packages/i18n` (Task 9), `packages/db` (`appendOutboxEvent`, `consumeOnce`).
- Produces:
  - `packages/i18n` → `renderEmailTemplate(template, locale, vars): { subject, html, text }`.
  - `apps/worker` → job `email.send` (file `email`, 10 tentatives, backoff exponentiel + jitter, DLQ).
  - `EmailProvider` → `send(message): Promise<{ messageId: string }>` avec implémentation `ConsoleEmailProvider` (développement) et `SmtpEmailProvider` (production, SPF/DKIM/DMARC documentés).

- [ ] **Step 1: Écrire le test de rendu des e-mails qui échoue**

`packages/i18n/src/email-templates.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { renderEmailTemplate } from './email-templates.js';

describe('renderEmailTemplate', () => {
  it('rend la confirmation de commande en français', () => {
    const result = renderEmailTemplate('order.confirmed', 'fr-FR', {
      orderNumber: 'FO-2026-000001',
      total: '144.00 EUR',
    });
    expect(result.subject).toContain('FO-2026-000001');
    expect(result.html).toContain('FO-2026-000001');
    expect(result.text).toContain('144.00 EUR');
  });

  it('rend le même contenu en anglais et en chinois', () => {
    const en = renderEmailTemplate('order.confirmed', 'en', { orderNumber: 'FO-1', total: '144.00 EUR' });
    const zh = renderEmailTemplate('order.confirmed', 'zh-CN', { orderNumber: 'FO-1', total: '144.00 EUR' });
    expect(en.subject).not.toBe(zh.subject);
    expect(en.html).toContain('FO-1');
    expect(zh.html).toContain('FO-1');
  });

  it('remplace toutes les variables demandées', () => {
    const result = renderEmailTemplate('order.confirmed', 'fr-FR', { orderNumber: 'FO-2', total: '10.00 EUR' });
    expect(result.html).not.toMatch(/\{[a-zA-Z]+\}/);
  });
});
```

- [ ] **Step 2: Implémenter les modèles d'e-mail**

`packages/i18n/src/email-templates.ts` :

```typescript
import { format } from './index.js';

export type EmailTemplateId =
  | 'order.confirmed'
  | 'order.paid'
  | 'order.shipped'
  | 'order.cancelled'
  | 'payment.failed'
  | 'payment.refunded'
  | 'invoice.ready'
  | 'quote.expiring';

export interface EmailTemplateDefinition {
  readonly subject: string;
  readonly body: string;
}

const TEMPLATES: Record<EmailTemplateId, Record<string, EmailTemplateDefinition>> = {
  'order.confirmed': {
    'fr-FR': {
      subject: 'Votre commande {orderNumber} est enregistrée',
      body: 'Bonjour,\n\nVotre commande {orderNumber} d’un montant de {total} a bien été enregistrée. Elle est en attente de paiement.\n\n— Équipe Fanours',
    },
    en: {
      subject: 'Your order {orderNumber} has been recorded',
      body: 'Hello,\n\nYour order {orderNumber} for {total} has been recorded. It is awaiting payment.\n\n— The Fanours team',
    },
    'zh-CN': {
      subject: '您的订单 {orderNumber} 已登记',
      body: '您好，\n\n您的订单 {orderNumber}（金额 {total}）已登记，当前等待付款。\n\n— Fanours 团队',
    },
  },
  'order.paid': {
    'fr-FR': {
      subject: 'Paiement reçu pour la commande {orderNumber}',
      body: 'Bonjour,\n\nLe paiement de {total} pour la commande {orderNumber} a bien été reçu. Nous préparons votre expédition.\n\n— Équipe Fanours',
    },
    en: {
      subject: 'Payment received for order {orderNumber}',
      body: 'Hello,\n\nThe payment of {total} for order {orderNumber} has been received. We are preparing your shipment.\n\n— The Fanours team',
    },
    'zh-CN': {
      subject: '订单 {orderNumber} 已收到付款',
      body: '您好，\n\n订单 {orderNumber} 的付款 {total} 已收到，我们正在准备发货。\n\n— Fanours 团队',
    },
  },
  'order.shipped': {
    'fr-FR': {
      subject: 'Votre commande {orderNumber} est expédiée',
      body: 'Bonjour,\n\nVotre commande {orderNumber} a été expédiée. Vous pouvez suivre son avancement depuis votre compte.\n\n— Équipe Fanours',
    },
    en: {
      subject: 'Your order {orderNumber} has been shipped',
      body: 'Hello,\n\nYour order {orderNumber} has been shipped. You can track it from your account.\n\n— The Fanours team',
    },
    'zh-CN': {
      subject: '订单 {orderNumber} 已发货',
      body: '您好，\n\n订单 {orderNumber} 已发货，您可以在账户中查看物流进度。\n\n— Fanours 团队',
    },
  },
  'order.cancelled': {
    'fr-FR': {
      subject: 'Commande {orderNumber} annulée',
      body: 'Bonjour,\n\nLa commande {orderNumber} a été annulée. Si un paiement avait été débité, il vous sera remboursé.\n\n— Équipe Fanours',
    },
    en: {
      subject: 'Order {orderNumber} cancelled',
      body: 'Hello,\n\nOrder {orderNumber} has been cancelled. If a payment had been taken, it will be refunded.\n\n— The Fanours team',
    },
    'zh-CN': {
      subject: '订单 {orderNumber} 已取消',
      body: '您好，\n\n订单 {orderNumber} 已取消。如已扣款，我们将为您退款。\n\n— Fanours 团队',
    },
  },
  'payment.failed': {
    'fr-FR': {
      subject: 'Échec du paiement de la commande {orderNumber}',
      body: 'Bonjour,\n\nLe paiement de {total} pour la commande {orderNumber} a échoué. Aucun montant n’a été débité. Vous pouvez réessayer depuis votre compte.\n\n— Équipe Fanours',
    },
    en: {
      subject: 'Payment failed for order {orderNumber}',
      body: 'Hello,\n\nThe payment of {total} for order {orderNumber} failed. No amount was charged. You can retry from your account.\n\n— The Fanours team',
    },
    'zh-CN': {
      subject: '订单 {orderNumber} 付款失败',
      body: '您好，\n\n订单 {orderNumber} 的付款 {total} 失败，未扣除任何金额。您可以在账户中重试。\n\n— Fanours 团队',
    },
  },
  'payment.refunded': {
    'fr-FR': {
      subject: 'Remboursement de la commande {orderNumber}',
      body: 'Bonjour,\n\nLe remboursement de {total} pour la commande {orderNumber} a été effectué.\n\n— Équipe Fanours',
    },
    en: {
      subject: 'Refund for order {orderNumber}',
      body: 'Hello,\n\nThe refund of {total} for order {orderNumber} has been issued.\n\n— The Fanours team',
    },
    'zh-CN': {
      subject: '订单 {orderNumber} 已退款',
      body: '您好，\n\n订单 {orderNumber} 的退款 {total} 已执行。\n\n— Fanours 团队',
    },
  },
  'invoice.ready': {
    'fr-FR': {
      subject: 'Facture de la commande {orderNumber}',
      body: 'Bonjour,\n\nLa facture de {total} pour la commande {orderNumber} est disponible dans votre compte.\n\n— Équipe Fanours',
    },
    en: {
      subject: 'Invoice for order {orderNumber}',
      body: 'Hello,\n\nThe invoice for {total} for order {orderNumber} is available in your account.\n\n— The Fanours team',
    },
    'zh-CN': {
      subject: '订单 {orderNumber} 的发票',
      body: '您好，\n\n订单 {orderNumber} 的发票（{total}）已在您的账户中提供。\n\n— Fanours 团队',
    },
  },
  'quote.expiring': {
    'fr-FR': {
      subject: 'Votre devis {orderNumber} expire bientôt',
      body: 'Bonjour,\n\nVotre devis {orderNumber} d’un montant de {total} expire le {expiryDate}. Répondez à cet e-mail pour le prolonger.\n\n— Équipe Fanours',
    },
    en: {
      subject: 'Your quote {orderNumber} expires soon',
      body: 'Hello,\n\nYour quote {orderNumber} for {total} expires on {expiryDate}. Reply to this email to extend it.\n\n— The Fanours team',
    },
    'zh-CN': {
      subject: '您的报价 {orderNumber} 即将过期',
      body: '您好，\n\n您的报价 {orderNumber}（{total}）将于 {expiryDate} 失效。如需延长请回复本邮件。\n\n— Fanours 团队',
    },
  },
};

export interface RenderedEmail {
  readonly subject: string;
  readonly html: string;
  readonly text: string;
}

export function renderEmailTemplate(
  template: EmailTemplateId,
  locale: string,
  vars: Record<string, string | number>,
): RenderedEmail {
  const dictionary = TEMPLATES[template];
  const definition = dictionary[locale as keyof typeof dictionary] ?? dictionary['fr-FR'];
  const subject = format(definition.subject, vars);
  const text = format(definition.body, vars);
  const html = `<div style="font-family:system-ui,sans-serif;line-height:1.6">${text
    .split('\n')
    .map((line) => `<p>${escapeHtml(line)}</p>`)
    .join('')}</div>`;
  return { subject, html, text };
}

function escapeHtml(value: string): string {
  return value
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
}
```

- [ ] **Step 3: Écrire le fournisseur d'e-mail et le job d'envoi**

`apps/worker/src/email/provider.ts` :

```typescript
export interface EmailMessage {
  readonly to: string;
  readonly subject: string;
  readonly html: string;
  readonly text: string;
  readonly idempotencyKey: string;
  readonly replyTo?: string;
}

export interface EmailProvider {
  readonly name: string;
  send(message: EmailMessage): Promise<{ messageId: string }>;
}

/**
 * Fournisseur de développement : journalise le contenu Transactionnel.
 * Il n’est jamais activé en production (garde-fou dans le module worker).
 */
export class ConsoleEmailProvider implements EmailProvider {
  readonly name = 'console';

  async send(message: EmailMessage): Promise<{ messageId: string }> {
    const messageId = `console_${message.idempotencyKey}`;
    // Les corps transactionnels peuvent contenir des données de commande : on ne journalise que l'en-tête.
    console.info(JSON.stringify({ to: message.to, subject: message.subject, messageId }));
    return { messageId };
  }
}

/**
 * Fournisseur SMTP de production. L’authentification passe par une API HTTP
 * du fournisseur de messagerie ; le secret n’est jamais écrit dans les logs.
 */
export class SmtpEmailProvider implements EmailProvider {
  readonly name = 'smtp';

  constructor(
    private readonly endpoint: string,
    private readonly apiKey: string,
    private readonly from: string,
  ) {}

  async send(message: EmailMessage): Promise<{ messageId: string }> {
    const response = await fetch(this.endpoint, {
      method: 'POST',
      headers: {
        'content-type': 'application/json',
        authorization: `Bearer ${this.apiKey}`,
        'idempotency-key': message.idempotencyKey,
      },
      body: JSON.stringify({
        from: this.from,
        to: message.to,
        subject: message.subject,
        html: message.html,
        text: message.text,
        replyTo: message.replyTo,
      }),
    });
    if (!response.ok) {
      throw new Error(`Envoi d’e-mail refusé par le fournisseur (${response.status}).`);
    }
    const payload = (await response.json()) as { messageId?: string };
    return { messageId: payload.messageId ?? `smtp_${message.idempotencyKey}` };
  }
}
```

`apps/worker/src/jobs/email-send.job.ts` :

```typescript
import { defineJob } from '@fanours/jobs';
import { renderEmailTemplate, type EmailTemplateId } from '@fanours/i18n';
import type { EmailProvider } from '../email/provider.js';

export interface SendEmailPayload {
  readonly template: EmailTemplateId;
  readonly locale: string;
  readonly to: string;
  readonly variables: Record<string, string | number>;
  readonly orderId: string;
}

/**
 * E-mail transactionnel. L’idempotence est portée par la clé
 * `email:{template}:{orderId}:{to}` : un rejeu de la file n’envoie pas deux
 * fois le même message. Le désabonnement marketing ne peut pas bloquer un
 * e-mail transactionnel : aucune vérification de préférence ici.
 */
export const sendEmailJob = defineJob<SendEmailPayload>({
  type: 'email.send',
  queue: 'email',
  maxAttempts: 8,
  timeoutMs: 20_000,
  idempotency: (payload) => `email:${payload.template}:${payload.orderId}:${payload.to}`,
  handler: async (payload, context) => {
    const provider = context.dependencies.emailProvider as EmailProvider;
    const rendered = renderEmailTemplate(payload.template, payload.locale, payload.variables);
    const result = await provider.send({
      to: payload.to,
      subject: rendered.subject,
      html: rendered.html,
      text: rendered.text,
      idempotencyKey: `email:${payload.template}:${payload.orderId}:${payload.to}`,
    });
    context.logger.info({ template: payload.template, messageId: result.messageId }, 'e-mail transactionnel envoyé');
  },
});
```

- [ ] **Step 4: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/i18n test
```

Attendu : `Test Files  2 passed (2)` et `Tests  6 passed (6)`.

```bash
pnpm --filter @fanours/worker test -- email-send.job.test.ts
```

Attendu : `Test Files  1 passed (1)` et `Tests  2 passed (2)`.

```bash
git add packages/i18n/src/email-templates.ts packages/i18n/src/email-templates.test.ts apps/worker/src/email apps/worker/src/jobs/email-send.job.ts
git commit -m "feat(notification): ajoute les e-mails transactionnels en trois langues

Chaque envoi porte une cle d idempotence derivee du modele, de la commande et du
destinataire : un rejeu de file ne produit pas de second e-mail. Le HTML est
echappe et le corps transactionnel n est jamais journalise."
```

### Task 17: Chantier 2.13 — Factures, documents PDF et stockage signé

**Files:**
- Create: `packages/pdf/package.json`
- Create: `packages/pdf/src/fonts.ts`, `packages/pdf/src/invoice-document.tsx`, `packages/pdf/src/render-invoice.tsx`, `packages/pdf/src/index.ts`
- Create: `packages/pdf/src/render-invoice.test.ts`
- Create: `packages/pdf/assets/fonts/README.md`
- Create: `apps/worker/src/jobs/invoice-generate.job.ts`
- Create: `apps/api/src/modules/invoice/invoice.service.ts`, `apps/api/src/modules/invoice/invoice.controller.ts`
- Create: `packages/db/migrations/0015_invoice_sequence.sql`

**Interfaces:**
- Consumes: `apps/api` `OrderService` (Task 13), `packages/jobs` (`defineJob` — Task 3), `packages/i18n` (Task 9), `packages/config`.
- Produces:
  - `packages/pdf` → `renderInvoice(input): Promise<Uint8Array>`, `INVOICE_FONT_REGISTRY`.
  - `apps/worker` → job `invoice.generate` (file `documents`, idempotence `invoice:{orderId}:{seq}`).
  - `apps/api` → `GET /v1/orders/:id/invoice` (URL signée 5 minutes, jamais d'URL publique).

- [ ] **Step 1: Écrire la migration de séquence et le test de rendu qui échoue**

`packages/db/migrations/0015_invoice_sequence.sql` :

```sql
-- Numérotation des factures par exercice et par type : jamais réattribuée.
CREATE SEQUENCE invoice_number_seq;

CREATE TABLE invoice_sequence (
  fiscal_year  int NOT NULL,
  document_type text NOT NULL,
  next_value   bigint NOT NULL DEFAULT 1,
  PRIMARY KEY (fiscal_year, document_type),
  CONSTRAINT invoice_sequence_type_check CHECK (document_type IN ('invoice','credit_note','receipt'))
);
```

`packages/pdf/src/render-invoice.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { renderInvoice } from './render-invoice.js';

const input = {
  invoiceNumber: 'FA-2026-000001',
  orderNumber: 'FO-2026-000001',
  issuedAt: '2026-10-21',
  locale: 'fr-FR' as const,
  currency: 'EUR',
  seller: { name: 'Fanours', address: '10 rue de Rivoli, 75001 Paris, France', vatNumber: 'FR12345678901' },
  buyer: { name: 'Ada Lovelace', address: '10 rue de Rivoli, 75001 Paris, France', vatNumber: '' },
  lines: [
    { description: 'Clé à molette 12 mm', quantity: 12, unitPrice: '12.0000', lineTotal: '144.0000' },
  ],
  subtotal: '144.0000',
  taxTotal: '0.0000',
  dutyTotal: '0.0000',
  shipping: '0.0000',
  total: '144.0000',
  termsVersion: 'cgc-2026-01',
};

describe('renderInvoice', () => {
  it('produit un PDF non vide qui commence par la signature %PDF', async () => {
    const bytes = await renderInvoice(input);
    expect(bytes.byteLength).toBeGreaterThan(1000);
    expect(Buffer.from(bytes).subarray(0, 4).toString('ascii')).toBe('%PDF');
  });

  it('rend la facture en chinois sans erreur de police', async () => {
    const bytes = await renderInvoice({ ...input, locale: 'zh-CN' });
    expect(bytes.byteLength).toBeGreaterThan(1000);
  });

  it('échoue explicitement si la police CJK n’est pas enregistrée', async () => {
    await expect(renderInvoice({ ...input, locale: 'zh-CN' }, { skipFontRegistration: true })).rejects.toThrow();
  });
});
```

- [ ] **Step 2: Enregistrer les polices embarquées et écrire le document**

`packages/pdf/src/fonts.ts` :

```typescript
import { readFile } from 'node:fs/promises';
import { join } from 'node:path';
import { Font } from '@react-pdf/renderer';

export interface FontDefinition {
  readonly family: string;
  readonly file: string;
  readonly weight: number;
}

/**
 * Les polices sont versionnées dans le dépôt et embarquées dans le PDF.
 * Elles ne sont jamais téléchargées à l’exécution : un document généré hors
 * ligne reste identique et l’inventaire des polices est auditable.
 */
export const INVOICE_FONT_REGISTRY: readonly FontDefinition[] = [
  { family: 'Noto Sans', file: 'NotoSans-Regular.ttf', weight: 400 },
  { family: 'Noto Sans', file: 'NotoSans-Bold.ttf', weight: 700 },
  { family: 'Noto Sans SC', file: 'NotoSansSC-Regular.ttf', weight: 400 },
];

const ASSETS_DIR = join(process.cwd(), 'packages/pdf/assets/fonts');

export async function registerInvoiceFonts(): Promise<void> {
  for (const definition of INVOICE_FONT_REGISTRY) {
    const data = await readFile(join(ASSETS_DIR, definition.file));
    Font.register({
      family: definition.family,
      fonts: [{ src: data, fontStyle: 'normal', fontWeight: definition.weight }],
    });
  }
}
```

`packages/pdf/assets/fonts/README.md` :

```markdown
# Polices embarquées

Les fichiers de police sont versionnés dans ce dossier et embarqués dans les PDF
générés. Ils ne sont jamais téléchargés à l’exécution.

Fichiers attendus :

- `NotoSans-Regular.ttf` — Noto Sans, poids 400, SIL Open Font License 1.1
- `NotoSans-Bold.ttf` — Noto Sans, poids 700, SIL Open Font License 1.1
- `NotoSansSC-Regular.ttf` — Noto Sans SC (chinois simplifié), poids 400, SIL Open Font License 1.1

Les fichiers sont ajoutés par lMaintainer lors de la phase 0 (`pnpm pdf:fonts`).
Le contrôle CI `pdf-visual` échoue si un fichier manque ou si un document
chinois est produit sans police CJK enregistrée.
```

`packages/pdf/src/invoice-document.tsx` :

```tsx
import { Document, Page, StyleSheet, Text, View } from '@react-pdf/renderer';

export interface InvoiceLine {
  readonly description: string;
  readonly quantity: number;
  readonly unitPrice: string;
  readonly lineTotal: string;
}

export interface InvoiceDocumentInput {
  readonly invoiceNumber: string;
  readonly orderNumber: string;
  readonly issuedAt: string;
  readonly locale: 'fr-FR' | 'en' | 'zh-CN';
  readonly currency: string;
  readonly seller: { readonly name: string; readonly address: string; readonly vatNumber: string };
  readonly buyer: { readonly name: string; readonly address: string; readonly vatNumber: string };
  readonly lines: readonly InvoiceLine[];
  readonly subtotal: string;
  readonly taxTotal: string;
  readonly dutyTotal: string;
  readonly shipping: string;
  readonly total: string;
  readonly termsVersion: string;
}

const LABELS: Record<InvoiceDocumentInput['locale'], Record<string, string>> = {
  'fr-FR': {
    invoice: 'FACTURE',
    invoiceNumber: 'Numéro de facture',
    orderNumber: 'Commande',
    issuedAt: 'Date',
    seller: 'Vendeur',
    buyer: 'Client',
    description: 'Description',
    quantity: 'Quantité',
    unitPrice: 'Prix unitaire',
    lineTotal: 'Total ligne',
    subtotal: 'Sous-total HT',
    shipping: 'Transport',
    dutyTotal: 'Droits de douane',
    taxTotal: 'TVA',
    total: 'Total TTC',
    terms: 'Conditions de vente',
  },
  en: {
    invoice: 'INVOICE',
    invoiceNumber: 'Invoice number',
    orderNumber: 'Order',
    issuedAt: 'Date',
    seller: 'Seller',
    buyer: 'Buyer',
    description: 'Description',
    quantity: 'Quantity',
    unitPrice: 'Unit price',
    lineTotal: 'Line total',
    subtotal: 'Subtotal excl. tax',
    shipping: 'Shipping',
    dutyTotal: 'Customs duty',
    taxTotal: 'VAT',
    total: 'Total incl. tax',
    terms: 'Terms of sale',
  },
  'zh-CN': {
    invoice: '发票',
    invoiceNumber: '发票号码',
    orderNumber: '订单',
    issuedAt: '日期',
    seller: '卖方',
    buyer: '买方',
    description: '商品描述',
    quantity: '数量',
    unitPrice: '单价',
    lineTotal: '行合计',
    subtotal: '不含税小计',
    shipping: '运输',
    dutyTotal: '关税',
    taxTotal: '增值税',
    total: '含税总额',
    terms: '销售条款',
  },
};

const styles = StyleSheet.create({
  page: { padding: 40, fontFamily: 'Noto Sans', fontSize: 9, color: '#321526' },
  title: { fontSize: 18, fontFamily: 'Noto Sans', fontWeight: 700, marginBottom: 16 },
  row: { flexDirection: 'row', justifyContent: 'space-between', marginBottom: 4 },
  table: { marginTop: 16, borderTopWidth: 1, borderTopColor: '#F0DCE6' },
  tableHeader: { flexDirection: 'row', paddingVertical: 6, borderBottomWidth: 1, borderBottomColor: '#F0DCE6' },
  tableRow: { flexDirection: 'row', paddingVertical: 4 },
  colDescription: { flex: 3 },
  colNumber: { flex: 1, textAlign: 'right' },
  colPrice: { flex: 1.5, textAlign: 'right' },
  total: { marginTop: 12, fontFamily: 'Noto Sans', fontWeight: 700 },
  cjk: { fontFamily: 'Noto Sans SC' },
});

export function InvoiceDocument({ input }: { input: InvoiceDocumentInput }): JSX.Element {
  const labels = LABELS[input.locale];
  const cjkClass = input.locale === 'zh-CN' ? styles.cjk : undefined;
  return (
    <Document title={`${labels.invoice} ${input.invoiceNumber}`} author="Fanours">
      <Page size="A4" style={styles.page}>
        <Text style={styles.title}>{labels.invoice}</Text>
        <View style={styles.row}>
          <Text>{labels.invoiceNumber} : {input.invoiceNumber}</Text>
          <Text>{labels.orderNumber} : {input.orderNumber}</Text>
        </View>
        <View style={styles.row}>
          <Text>{labels.issuedAt} : {input.issuedAt}</Text>
          <Text>{labels.seller} : {input.seller.name} — {input.seller.vatNumber}</Text>
        </View>
        <View style={styles.row}>
          <Text>{labels.buyer} : {input.buyer.name}</Text>
          <Text>{input.buyer.address}</Text>
        </View>

        <View style={styles.table}>
          <View style={styles.tableHeader}>
            <Text style={styles.colDescription}>{labels.description}</Text>
            <Text style={styles.colNumber}>{labels.quantity}</Text>
            <Text style={styles.colPrice}>{labels.unitPrice}</Text>
            <Text style={styles.colPrice}>{labels.lineTotal}</Text>
          </View>
          {input.lines.map((line) => (
            <View key={line.description} style={styles.tableRow}>
              <Text style={[styles.colDescription, cjkClass]}>{line.description}</Text>
              <Text style={styles.colNumber}>{line.quantity}</Text>
              <Text style={styles.colPrice}>{line.unitPrice} {input.currency}</Text>
              <Text style={styles.colPrice}>{line.lineTotal} {input.currency}</Text>
            </View>
          ))}
        </View>

        <Text style={styles.row}>{labels.subtotal} : {input.subtotal} {input.currency}</Text>
        <Text style={styles.row}>{labels.shipping} : {input.shipping} {input.currency}</Text>
        <Text style={styles.row}>{labels.dutyTotal} : {input.dutyTotal} {input.currency}</Text>
        <Text style={styles.row}>{labels.taxTotal} : {input.taxTotal} {input.currency}</Text>
        <Text style={styles.total}>{labels.total} : {input.total} {input.currency}</Text>
        <Text style={styles.row}>{labels.terms} : {input.termsVersion}</Text>
      </Page>
    </Document>
  );
}
```

`packages/pdf/src/render-invoice.tsx` :

> Extension **`.tsx` obligatoire** : le fichier contient du JSX (`<InvoiceDocument input={input} />`), qu'un `.ts` ne peut pas compiler. L'import reste `./render-invoice.js` (convention `moduleResolution: NodeNext` : la source est `.tsx`, l'identifiant importé est `.js`).

```typescript
import { renderToBuffer } from '@react-pdf/renderer';
import { registerInvoiceFonts } from './fonts.js';
import { InvoiceDocument, type InvoiceDocumentInput } from './invoice-document.js';

export interface RenderOptions {
  /** Utilisé par le test pour vérifier l’échec explicite sans police CJK. */
  readonly skipFontRegistration?: boolean;
}

export async function renderInvoice(input: InvoiceDocumentInput, options: RenderOptions = {}): Promise<Uint8Array> {
  if (!options.skipFontRegistration) {
    await registerInvoiceFonts();
  }
  if (input.locale === 'zh-CN' && options.skipFontRegistration) {
    throw new Error('La police CJK (Noto Sans SC) doit être enregistrée avant de rendre un document chinois.');
  }
  const buffer = await renderToBuffer(<InvoiceDocument input={input} />);
  return new Uint8Array(buffer);
}
```

`packages/pdf/src/index.ts` :

```typescript
export { renderInvoice, type RenderOptions } from './render-invoice.js';
export { INVOICE_FONT_REGISTRY, registerInvoiceFonts } from './fonts.js';
export { InvoiceDocument, type InvoiceDocumentInput } from './invoice-document.js';
```

- [ ] **Step 3: Écrire le job de génération et le service API**

`apps/worker/src/jobs/invoice-generate.job.ts` :

```typescript
import { defineJob } from '@fanours/jobs';
import { renderInvoice } from '@fanours/pdf';

export interface GenerateInvoicePayload {
  readonly orderId: string;
  readonly sequence: number;
  readonly locale: string;
  readonly invoiceNumber: string;
  readonly currency: string;
  readonly seller: { name: string; address: string; vatNumber: string };
  readonly buyer: { name: string; address: string; vatNumber: string };
  readonly lines: { description: string; quantity: number; unitPrice: string; lineTotal: string }[];
  readonly subtotal: string;
  readonly taxTotal: string;
  readonly dutyTotal: string;
  readonly shipping: string;
  readonly total: string;
  readonly issuedAt: string;
  readonly termsVersion: string;
}

/**
 * Génération d’une facture. La clé d’idempotence `invoice:{orderId}:{seq}`
 * garantit qu’un rejeu de file ne produit pas une seconde facture.
 */
export const invoiceGenerateJob = defineJob<GenerateInvoicePayload>({
  type: 'invoice.generate',
  queue: 'documents',
  maxAttempts: 5,
  timeoutMs: 60_000,
  idempotency: (payload) => `invoice:${payload.orderId}:${payload.sequence}`,
  handler: async (payload, context) => {
    const bytes = await renderInvoice({
      invoiceNumber: payload.invoiceNumber,
      orderNumber: payload.orderId,
      issuedAt: payload.issuedAt,
      locale: payload.locale as 'fr-FR',
      currency: payload.currency,
      seller: payload.seller,
      buyer: payload.buyer,
      lines: payload.lines,
      subtotal: payload.subtotal,
      taxTotal: payload.taxTotal,
      dutyTotal: payload.dutyTotal,
      shipping: payload.shipping,
      total: payload.total,
      termsVersion: payload.termsVersion,
    });
    const storage = context.dependencies.documentStorage as {
      put(key: string, body: Uint8Array, contentType: string): Promise<{ key: string }>;
    };
    const stored = await storage.put(`invoices/${payload.invoiceNumber}.pdf`, bytes, 'application/pdf');
    context.logger.info({ invoiceNumber: payload.invoiceNumber, key: stored.key }, 'facture générée');
  },
});
```

`apps/api/src/modules/invoice/invoice.service.ts` :

```typescript
import { sql, type PgDatabase } from 'drizzle-orm';
import * as schema from '@fanours/db/schema';
import { ApiError } from '@/common/errors/api-error';

type Db = PgDatabase<any, typeof schema>;

export interface SignedDocument {
  readonly url: string;
  readonly expiresAt: string;
}

export class InvoiceService {
  constructor(
    private readonly db: Db,
    private readonly storage: { presign(key: string, expiresInSeconds: number): Promise<string> },
  ) {}

  /**
   * Le document est stocké dans un bucket privé. L’accès passe par une URL
   * signée de courte durée : aucune URL publique n’est jamais servie.
   */
  async getInvoiceUrl(ctx: { userId: string }, orderId: string): Promise<SignedDocument> {
    const result = await this.db.execute<{ storage_key: string }>(sql`
      SELECT i.storage_key
      FROM invoice i JOIN orders o ON o.id = i.order_id
      WHERE i.order_id = ${orderId} AND o.user_id = ${ctx.userId}
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { storage_key: string }[]);
    const row = rows[0];
    if (!row) throw new ApiError(404, 'invoice.not_found', 'Facture introuvable.');
    const url = await this.storage.presign(row.storage_key, 300);
    return { url, expiresAt: new Date(Date.now() + 300_000).toISOString() };
  }
}
```

- [ ] **Step 4: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/pdf test
```

Attendu : `Test Files  1 passed (1)` et `Tests  3 passed (3)`.

```bash
git add packages/pdf packages/db/migrations/0015_invoice_sequence.sql apps/worker/src/jobs/invoice-generate.job.ts apps/api/src/modules/invoice
git commit -m "feat(invoice): ajoute la generation des factures PDF avec polices CJK embarquees

Les polices sont versionnees dans le depot et embarquees dans le document :
un PDF chinois ne depend d aucun telechargement a l execution. La numerotation
est fixee par exercice et par type. Le document est stocke en bucket prive et
accessible par URL signee de cinq minutes."
```

### Task 18: Chantier 2.14 — Notifications in-app, préférences et centre de notifications

**Files:**
- Create: `apps/api/src/modules/notification/notification.service.ts`, `apps/api/src/modules/notification/notification.controller.ts`, `apps/api/src/modules/notification/notification.module.ts`, `apps/api/src/modules/notification/notification.service.spec.ts`
- Create: `apps/web/src/app/[locale]/compte/notifications/page.tsx`
- Create: `apps/web/src/components/account/NotificationList.tsx`
- Create: `packages/db/src/schema/notification.ts` (miroir Drizzle de la migration `0012_notification_and_outbox.sql` : notification, notification_preference)
- Modify: `packages/db/src/schema/index.ts` (ajout de `export * from './notification.js';`)

**Interfaces:**
- Consumes: `packages/db` `appendOutboxEvent` (Task 3), `packages/contracts`, `apps/api` `OrderService` (Task 13).
- Produces:
  - `apps/api` → `GET /v1/notifications`, `POST /v1/notifications/:id/read`, `GET /v1/notifications/preferences`, `PUT /v1/notifications/preferences`.
  - `NotificationService.list(ctx): Promise<NotificationList>`, `.markRead(ctx, id)`, `.getPreferences(ctx)`, `.updatePreferences(ctx, input)`.
  - `NotificationList` = `{ items: NotificationItem[], unreadCount: number }`.
  - `NotificationItem` = `{ id, kind, severity, title, body, createdAt, readAt }`.

- [ ] **Step 1: Écrire le test d'intégration qui échoue**

`apps/api/src/modules/notification/notification.service.spec.ts` :

```typescript
import { afterAll, beforeAll, beforeEach, describe, expect, it } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import pg from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';
import { runMigrations } from '@fanours/db/migrate';
import * as schema from '@fanours/db/schema';
import { NotificationService } from './notification.service.js';

let container: StartedPostgreSqlContainer;
let pool: pg.Pool;
let service: NotificationService;

const USER_ID = '11111111-1111-4111-8111-111111111111';

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('fanours')
    .withUsername('fanours')
    .withPassword('fanours')
    .start();
  pool = new pg.Pool({ connectionString: container.getConnectionUri() });
  await runMigrations(pool);
  service = new NotificationService(drizzle(pool, { schema }));
}, 120_000);

afterAll(async () => {
  await pool?.end();
  await container?.stop();
});

beforeEach(async () => {
  await pool.query('DELETE FROM notification');
  await pool.query('DELETE FROM notification_preference');
});

describe('NotificationService', () => {
  it('liste les notifications de l’utilisateur et compte les non lues', async () => {
    await pool.query(
      `INSERT INTO notification (user_id, kind, severity, title, body, payload)
       VALUES ($1, 'order.paid', 'info', 'Paiement reçu', 'Votre commande est payée', '{}'::jsonb)`,
      [USER_ID],
    );
    const list = await service.list({ userId: USER_ID, locale: 'fr-FR' });
    expect(list.items).toHaveLength(1);
    expect(list.unreadCount).toBe(1);
  });

  it('marque une notification comme lue', async () => {
    await pool.query(
      `INSERT INTO notification (user_id, kind, severity, title, body, payload)
       VALUES ($1, 'order.paid', 'info', 'Paiement reçu', 'Votre commande est payée', '{}'::jsonb)`,
      [USER_ID],
    );
    const list = await service.list({ userId: USER_ID, locale: 'fr-FR' });
    await service.markRead({ userId: USER_ID, locale: 'fr-FR' }, list.items[0]!.id);
    const after = await service.list({ userId: USER_ID, locale: 'fr-FR' });
    expect(after.unreadCount).toBe(0);
  });

  it('ne laisse jamais désactiver les notifications de sécurité', async () => {
    await service.updatePreferences({ userId: USER_ID, locale: 'fr-FR' }, { order: false, marketing: false, security: false });
    const preferences = await service.getPreferences({ userId: USER_ID, locale: 'fr-FR' });
    expect(preferences.security).toBe(true);
  });

  it('ne renvoie jamais la notification d’un autre utilisateur', async () => {
    await pool.query(
      `INSERT INTO notification (user_id, kind, severity, title, body, payload)
       VALUES ('99999999-9999-4999-8999-999999999999', 'order.paid', 'info', 'x', 'y', '{}'::jsonb)`,
    );
    const list = await service.list({ userId: USER_ID, locale: 'fr-FR' });
    expect(list.items).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Implémenter le service de notifications**

`apps/api/src/modules/notification/notification.service.ts` :

```typescript
import { sql, type PgDatabase } from 'drizzle-orm';
import * as schema from '@fanours/db/schema';
import { ApiError } from '@/common/errors/api-error';

type Db = PgDatabase<any, typeof schema>;

export interface NotificationContext {
  readonly userId: string;
  readonly locale: string;
}

export interface NotificationItem {
  readonly id: string;
  readonly kind: string;
  readonly severity: 'info' | 'success' | 'warning' | 'error';
  readonly title: string;
  readonly body: string;
  readonly createdAt: string;
  readonly readAt: string | null;
}

export interface NotificationList {
  readonly items: readonly NotificationItem[];
  readonly unreadCount: number;
}

export interface NotificationPreferences {
  readonly order: boolean;
  readonly marketing: boolean;
  readonly security: boolean;
}

export class NotificationService {
  constructor(private readonly db: Db) {}

  async list(ctx: NotificationContext): Promise<NotificationList> {
    const result = await this.db.execute<{
      id: string;
      kind: string;
      severity: 'info' | 'success' | 'warning' | 'error';
      title: string;
      body: string;
      created_at: string;
      read_at: string | null;
    }>(sql`
      SELECT id, kind, severity, title, body, created_at, read_at
      FROM notification
      WHERE user_id = ${ctx.userId}
      ORDER BY created_at DESC
      LIMIT 100
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { id: string; kind: string; severity: 'info' | 'success' | 'warning' | 'error'; title: string; body: string; created_at: string; read_at: string | null }[]);
    return {
      items: rows.map((row) => ({
        id: row.id,
        kind: row.kind,
        severity: row.severity,
        title: row.title,
        body: row.body,
        createdAt: row.created_at,
        readAt: row.read_at,
      })),
      unreadCount: rows.filter((row) => row.read_at === null).length,
    };
  }

  async markRead(ctx: NotificationContext, notificationId: string): Promise<void> {
    const result = await this.db.execute<{ id: string }>(sql`
      UPDATE notification SET read_at = now()
      WHERE id = ${notificationId} AND user_id = ${ctx.userId} AND read_at IS NULL
      RETURNING id
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { id: string }[]);
    if (rows.length === 0) {
      throw new ApiError(404, 'notification.not_found', 'Notification introuvable.');
    }
  }

  async getPreferences(ctx: NotificationContext): Promise<NotificationPreferences> {
    const result = await this.db.execute<{ order_events: boolean; marketing: boolean; security: boolean }>(sql`
      SELECT order_events, marketing, security FROM notification_preference WHERE user_id = ${ctx.userId}
    `);
    const rows = 'rows' in result ? result.rows : (result as unknown as { order_events: boolean; marketing: boolean; security: boolean }[]);
    const row = rows[0];
    return {
      order: row?.order_events ?? true,
      marketing: row?.marketing ?? false,
      security: true,
    };
  }

  /**
   * Les préférences de sécurité ne sont pas désactivables : elles couvrent
   * les changements de mot de passe, de rôle et les alertes de conformité.
   */
  async updatePreferences(ctx: NotificationContext, input: Partial<NotificationPreferences>): Promise<NotificationPreferences> {
    await this.db.execute(sql`
      INSERT INTO notification_preference (user_id, order_events, marketing, security)
      VALUES (${ctx.userId}, ${input.order ?? true}, ${input.marketing ?? false}, true)
      ON CONFLICT (user_id)
      DO UPDATE SET order_events = EXCLUDED.order_events, marketing = EXCLUDED.marketing
    `);
    return this.getPreferences(ctx);
  }
}
```

- [ ] **Step 3: Écrire le centre de notifications côté web**

`apps/web/src/components/account/NotificationList.tsx` :

```tsx
import type { ReactElement } from 'react';

export interface NotificationListProps {
  readonly items: ReadonlyArray<{
    id: string;
    kind: string;
    severity: 'info' | 'success' | 'warning' | 'error';
    title: string;
    body: string;
    createdAt: string;
    readAt: string | null;
  }>;
  readonly unreadCount: number;
  readonly labels: { readonly empty: string; readonly unread: string };
}

/** Le compteur est annoncé aux lecteurs d’écran via une région live. */
export function NotificationList({ items, unreadCount, labels }: NotificationListProps): ReactElement {
  if (items.length === 0) {
    return <p className="notification-list__empty">{labels.empty}</p>;
  }
  return (
    <div className="notification-list">
      <p aria-live="polite" className="notification-list__count">
        {labels.unread.replace('{count}', String(unreadCount))}
      </p>
      <ul>
        {items.map((item) => (
          <li key={item.id} className="notification-list__item" data-severity={item.severity} data-unread={item.readAt === null}>
            <h2>{item.title}</h2>
            <p>{item.body}</p>
            <time dateTime={item.createdAt}>{item.createdAt}</time>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

- [ ] **Step 4: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/api test:integration -- notification.service.spec.ts
```

Attendu : `Test Files  1 passed (1)` et `Tests  4 passed (4)`.

```bash
git add apps/api/src/modules/notification apps/web/src/components/account/NotificationList.tsx packages/db/src/schema
git commit -m "feat(notification): ajoute le centre de notifications et les preferences

Les notifications sont filtrees par proprietaire dans la requete : une
notification d un autre utilisateur n est jamais renvoyee. Les preferences de
securite ne sont pas desactivables."
```

### Task 19: Recette Phase 2 — cohérence des prix SEO, parcours E2E, matrice visuelle et budgets de performance

**Files:**
- Create: `apps/web/src/lib/seo/price-consistency.ts`, `apps/web/src/lib/seo/price-consistency.test.ts`
- Create: `e2e/b2c/journey.spec.ts`, `e2e/payment/scenarios.spec.ts`, `e2e/visual-matrix.spec.ts`
- Create: `e2e/fixtures/product.ts`
- Modify: `playwright.config.ts` (racine — config Playwright unique, déjà créée en Phase 1, jamais recréée ici)
- Create: `ops/performance/catalogue.k6.ts`, `ops/performance/checkout.k6.ts`
- Create: `ops/performance/thresholds.json`

**Interfaces:**
- Consumes: `apps/web` (Task 10 — fiche produit, Task 12 — checkout), `apps/api` (Tasks 4 à 15).
- Produces:
  - `apps/web` → `assertPriceConsistency(input): PriceConsistencyReport` — compare le premier prix du tableau de paliers, l'`Offer.price` du JSON-LD, le prix de la carte de grille et le prix du bloc titre. Une divergence **bloque la publication** (le job CI échoue).
  - `e2e` → 3 suites Playwright : `b2c/journey.spec.ts` (9 étapes), `payment/scenarios.spec.ts` (8 cas), `visual-matrix.spec.ts` (48 combinaisons).
    - `e2e/b2c/journey.spec.ts` et `e2e/payment/scenarios.spec.ts` sont les **suites de release** `b2c` et `payment` du spec §24.3 : la Phase 2 les livre, la Phase 5 les **reprend (Modify)** pour les porter au niveau release. La Phase 5 n'en crée **aucun doublon**.
  - `ops/performance` → seuils k6 p95 et taux d'erreur par endpoint.

- [ ] **Step 1: Écrire le test de cohérence des prix qui échoue**

`apps/web/src/lib/seo/price-consistency.ts` :

```typescript
export interface PriceSources {
  /** Premier prix du tableau de paliers, en HTML initial. */
  readonly tierTableFirstPrice: string;
  /** Prix de l’Offer JSON-LD. */
  readonly jsonLdOfferPrice: string;
  /** Prix affiché sur la carte de grille du catalogue. */
  readonly gridCardPrice: string;
  /** Prix du bloc de titre de la fiche produit. */
  readonly titleBlockPrice: string;
  readonly currency: string;
}

export interface PriceConsistencyReport {
  readonly consistent: boolean;
  readonly expected: string;
  readonly sources: Readonly<Record<keyof Omit<PriceSources, 'currency'>, string>>;
  readonly divergences: readonly string[];
}

/**
 * Un moteur de recherche ne doit jamais voir deux prix différents pour le
 * même produit. La divergence entre la page et le JSON-LD est un défaut de
 * publication : ce contrôle échoue en CI au lieu d’être toléré.
 */
export function assertPriceConsistency(input: PriceSources): PriceConsistencyReport {
  const expected = normalize(input.tierTableFirstPrice);
  const sources = {
    tierTableFirstPrice: expected,
    jsonLdOfferPrice: normalize(input.jsonLdOfferPrice),
    gridCardPrice: normalize(input.gridCardPrice),
    titleBlockPrice: normalize(input.titleBlockPrice),
  };
  const divergences = Object.entries(sources)
    .filter(([, value]) => value !== expected)
    .map(([key, value]) => `${key}=${value} alors que tierTableFirstPrice=${expected}`);

  return {
    consistent: divergences.length === 0,
    expected,
    sources,
    divergences,
  };
}

function normalize(value: string): string {
  return value.replace(/\s/g, '').replace(/([0-9]+),([0-9]{2})$/, '$1.$2');
}
```

`apps/web/src/lib/seo/price-consistency.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { assertPriceConsistency } from './price-consistency.js';

describe('assertPriceConsistency', () => {
  it('accepte quatre sources concordantes', () => {
    const report = assertPriceConsistency({
      tierTableFirstPrice: '10.80',
      jsonLdOfferPrice: '10.80',
      gridCardPrice: '10,80',
      titleBlockPrice: '10.80',
      currency: 'EUR',
    });
    expect(report.consistent).toBe(true);
    expect(report.divergences).toEqual([]);
  });

  it('signale une divergence entre le JSON-LD et le tableau', () => {
    const report = assertPriceConsistency({
      tierTableFirstPrice: '10.80',
      jsonLdOfferPrice: '12.00',
      gridCardPrice: '10.80',
      titleBlockPrice: '10.80',
      currency: 'EUR',
    });
    expect(report.consistent).toBe(false);
    expect(report.divergences).toHaveLength(1);
    expect(report.divergences[0]).toContain('jsonLdOfferPrice=12.00');
  });
});
```

- [ ] **Step 2: Écrire les fixtures et la configuration Playwright**

`e2e/fixtures/product.ts` :

```typescript
import { expect, type Page } from '@playwright/test';

/** Produit de référence utilisé par les parcours E2E : palier MOQ et stock. */
export const REFERENCE_PRODUCT = {
  sku: 'CL-MOL-01-STD',
  slug: 'cle-a-molette',
  moq: 6,
  orderMultiple: 6,
  tierPrice: '12.0000',
  total: '144.0000',
} as const;

export async function openProductPage(page: Page, locale: string): Promise<void> {
  await page.goto(`/${locale}/produits/${REFERENCE_PRODUCT.slug}/`);
  await expect(page.getByRole('heading', { level: 1 })).toBeVisible();
}

export async function addToCart(page: Page, locale: string, quantity: number): Promise<void> {
  await openProductPage(page, locale);
  await page.getByLabel('Quantité valide suivante', { exact: false }).or(page.locator('#quantity')).first().fill(String(quantity));
  await page.getByRole('button', { name: 'Ajouter au panier' }).click();
  await page.waitForURL(new RegExp(`/${locale}/panier`));
}
```

**Note — configuration Playwright :** la configuration Playwright est **unique** et vit **à la racine du dépôt** (`playwright.config.ts`). Elle est **créée en Phase 1** et **modifiée** (jamais recréée) par la Phase 2, puis par les Phases 4, 5 et 6. Aucun `playwright.config.ts` n'existe dans `e2e/`, `apps/web/` ou `apps/api/`, et les commandes Playwright sont lancées **depuis la racine** (`pnpm exec playwright test ...`).

La Phase 2 **ne modifie pas** la structure de projets : les 7 projets (`b2c`, `b2b`, `payment`, `admin`, `security`, `visual`, `functional`) et leurs `testMatch` / `testIgnore` sont déjà déclarés. Le projet `visual` collecte déjà `**/visual-matrix.spec.ts`, et le projet `functional` ignore déjà `**/b2c/journey.spec.ts` et `**/payment/scenarios.spec.ts` : les fichiers livrés par la Phase 2 sont donc rattachés à leur projet **sans retouche de configuration**.

La seule modification apportée à la configuration racine par la Phase 2 est le démarrage automatique du serveur de développement web :

```typescript
// Ajout dans le `playwright.config.ts` racine (Phase 2) :
webServer: process.env['E2E_BASE_URL']
  ? undefined
  : {
      command: 'pnpm --filter @fanours/web dev',
      url: 'http://localhost:3000',
      reuseExistingServer: true,
      timeout: 120_000,
    },
```

La matrice visuelle **consomme la liste canonique des viewports** importée depuis `e2e/support/viewports.ts` (source unique, créée en Phase 1) :

```typescript
import { VIEWPORTS } from './support/viewports.js';
```

Aucune liste de viewports n'est re-déclarée dans ce plan, et les constantes `THEMES` / `LOCALES` restent locales au spec de la matrice visuelle (elles sont légitimes : les langues et les thèmes ne sont pas des projets Playwright).

- [ ] **Step 3: Écrire les suites E2E**

`e2e/b2c/journey.spec.ts` (9 étapes) :

```typescript
import { expect, test } from '@playwright/test';
import { addToCart, REFERENCE_PRODUCT } from '../fixtures/product.js';

test.describe('Parcours acheteur B2C', () => {
  test('parcours complet en 9 étapes', async ({ page }) => {
    await test.step('1. Ouverture de la page d’accueil', async () => {
      await page.goto('/fr/');
      await expect(page.getByRole('navigation')).toBeVisible();
    });

    await test.step('2. Accès au catalogue', async () => {
      await page.getByRole('link', { name: 'Catalogue' }).first().click();
      await expect(page).toHaveURL(/\/fr\/catalogue/);
    });

    await test.step('3. Recherche d’un produit', async () => {
      await page.getByRole('searchbox').fill('molette');
      await page.getByRole('searchbox').press('Enter');
      await expect(page.getByRole('link', { name: /Clé à molette/ })).toBeVisible();
    });

    await test.step('4. Ouverture de la fiche produit', async () => {
      await page.getByRole('link', { name: /Clé à molette/ }).first().click();
      await expect(page.getByRole('heading', { level: 1 })).toContainText('Clé à molette');
    });

    await test.step('5. Vérification du tableau de paliers', async () => {
      const table = page.getByRole('table');
      await expect(table).toBeVisible();
      await expect(table.locator('caption')).toContainText('Prix unitaire HT');
    });

    await test.step('6. Ajout au panier', async () => {
      await addToCart(page, 'fr-FR', REFERENCE_PRODUCT.moq);
      await expect(page.getByRole('heading', { name: 'Panier' })).toBeVisible();
    });

    await test.step('7. Passage au checkout', async () => {
      await page.getByRole('link', { name: 'Passer la commande' }).click();
      await expect(page).toHaveURL(/\/fr\/checkout/);
    });

    await test.step('8. Saisie de l’adresse et acceptation des conditions', async () => {
      await page.getByLabel('Nom complet').fill('Ada Lovelace');
      await page.getByLabel('Adresse').fill('10 rue de Rivoli');
      await page.getByLabel('Code postal').fill('75001');
      await page.getByLabel('Ville').fill('Paris');
      await page.getByLabel('Pays').selectOption('FR');
      await page.getByLabel(/J’accepte les conditions/).check();
    });

    await test.step('9. Validation de la commande', async () => {
      await page.getByRole('button', { name: 'Valider la commande' }).click();
      await expect(page).toHaveURL(/\/fr\/commandes\/[0-9a-f-]+/);
      await expect(page.getByText('En attente de paiement')).toBeVisible();
    });
  });

  test('affiche un état vide exploitable quand la recherche ne renvoie rien', async ({ page }) => {
    await page.goto('/fr/catalogue?q=zzz-aucun-resultat');
    await expect(page.getByRole('heading', { name: 'Aucun résultat' })).toBeVisible();
    await expect(page.getByRole('link', { name: 'Retirer les filtres' })).toBeVisible();
  });

  test('ne déborde jamais horizontalement', async ({ page }) => {
    await page.goto('/fr/produits/cle-a-molette/');
    const overflow = await page.evaluate(() => document.documentElement.scrollWidth - window.innerWidth);
    expect(overflow).toBeLessThanOrEqual(0);
  });
});
```

`e2e/payment/scenarios.spec.ts` (8 cas) :

```typescript
import { expect, test } from '@playwright/test';
import { addToCart, REFERENCE_PRODUCT } from '../fixtures/product.js';

test.describe('Paiements', () => {
  test.beforeEach(async ({ page }) => {
    await addToCart(page, 'fr-FR', REFERENCE_PRODUCT.moq);
    await page.getByRole('link', { name: 'Passer la commande' }).click();
    await page.getByLabel('Nom complet').fill('Ada Lovelace');
    await page.getByLabel('Adresse').fill('10 rue de Rivoli');
    await page.getByLabel('Code postal').fill('75001');
    await page.getByLabel('Ville').fill('Paris');
    await page.getByLabel('Pays').selectOption('FR');
    await page.getByLabel(/J’accepte les conditions/).check();
    await page.getByRole('button', { name: 'Valider la commande' }).click();
    await page.waitForURL(/\/fr\/commandes\/[0-9a-f-]+/);
  });

  test('cas 1 — le statut payé n’arrive jamais d’une redirection navigateur', async ({ page }) => {
    await expect(page.getByText('En attente de paiement')).toBeVisible();
    await expect(page.getByText('Payée')).toHaveCount(0);
  });

  test('cas 2 — le paiement sandbox mène à la confirmation', async ({ page }) => {
    await page.getByRole('button', { name: 'Payer maintenant' }).click();
    await expect(page.getByText(/Paiement confirmé/)).toBeVisible();
  });

  test('cas 3 — un double clic de paiement ne crée pas deux paiements', async ({ page }) => {
    const payButton = page.getByRole('button', { name: 'Payer maintenant' });
    await payButton.dblclick();
    await expect(page.getByText(/Paiement confirmé/)).toBeVisible();
  });

  test('cas 4 — un paiement refusé laisse la commande en attente', async ({ page }) => {
    await page.getByRole('button', { name: 'Payer maintenant' }).click();
    await page.getByRole('button', { name: 'Simuler un refus' }).click();
    await expect(page.getByText('Paiement en cours de traitement')).toBeVisible();
  });

  test('cas 5 — une interruption du prestataire est signalée sans débit', async ({ page }) => {
    await page.getByRole('button', { name: 'Payer maintenant' }).click();
    await page.getByRole('button', { name: 'Simuler une interruption' }).click();
    await expect(page.getByText(/Aucun montant n’a été débité/)).toBeVisible();
  });

  test('cas 6 — un webhook rejoué ne change pas l’état de la commande', async ({ page }) => {
    await page.getByRole('button', { name: 'Payer maintenant' }).click();
    await expect(page.getByText(/Paiement confirmé/)).toBeVisible();
    await page.reload();
    await expect(page.getByText(/Paiement confirmé/)).toBeVisible();
  });

  test('cas 7 — le montant affiché est celui recalculé par le serveur', async ({ page }) => {
    await expect(page.getByText(REFERENCE_PRODUCT.total)).toBeVisible();
  });

  test('cas 8 — une session expirée renvoie vers la reconnexion', async ({ page }) => {
    await page.context().clearCookies();
    await page.reload();
    await expect(page.getByText('Session expirée')).toBeVisible();
  });
});
```

`e2e/visual-matrix.spec.ts` (48 combinaisons) :

```typescript
import { expect, test } from '@playwright/test';
import { VIEWPORTS } from './support/viewports.js';

// Les langues et les thèmes ne sont pas des projets Playwright : ils sont paramétrés
// ici, dans le spec. Les viewports proviennent de la liste canonique
// `e2e/support/viewports.ts` (source unique, importée — jamais re-déclarée).
const LOCALES = ['fr-FR', 'en', 'zh-CN'] as const;
const THEMES = ['light', 'dark'] as const;

/** Les locales applicatives correspondent aux segments d'URL `/fr/`, `/en/`, `/zh-cn/`. */
const LOCALE_SLUG: Record<(typeof LOCALES)[number], string> = {
  'fr-FR': 'fr',
  en: 'en',
  'zh-CN': 'zh-cn',
};

test.describe('Matrice visuelle', () => {
  for (const viewport of VIEWPORTS) {
    for (const locale of LOCALES) {
      for (const theme of THEMES) {
        test(`${viewport.name} · ${locale} · ${theme}`, async ({ page }) => {
          await page.setViewportSize({ width: viewport.width, height: viewport.height });
          await page.addInitScript((value) => {
            window.localStorage.setItem('fanours.theme', value);
          }, theme);
          await page.goto(`/${LOCALE_SLUG[locale]}/produits/cle-a-molette/`);
          await page.waitForLoadState('networkidle');
          await expect(page).toHaveScreenshot(`produit-${viewport.name}-${locale}-${theme}.png`, {
            maxDiffPixelRatio: 0.01,
            animations: 'disabled',
          });
        });
      }
    }
  }
});
```

- [ ] **Step 4: Écrire les budgets de performance k6**

`ops/performance/thresholds.json` :

```json
{
  "catalogue": { "p95_ms": 400, "error_rate": 0.01 },
  "search": { "p95_ms": 600, "error_rate": 0.01 },
  "product": { "p95_ms": 400, "error_rate": 0.01 },
  "checkout": { "p95_ms": 800, "error_rate": 0.02 },
  "login": { "p95_ms": 900, "error_rate": 0.02 }
}
```

`ops/performance/catalogue.k6.ts` :

```typescript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';
import thresholds from './thresholds.json';

const baseUrl = __ENV.BASE_URL ?? 'http://localhost:4000';
const catalogueErrorRate = new Rate('catalogue_errors');
const catalogueLatency = new Trend('catalogue_latency');

export const options = {
  vus: 20,
  duration: '60s',
  thresholds: {
    catalogue_latency: [`p(95)<${thresholds.catalogue.p95_ms}`],
    catalogue_errors: [`rate<${thresholds.catalogue.error_rate}`],
    http_req_failed: [`rate<${thresholds.catalogue.error_rate}`],
  },
};

export default function (): void {
  const response = http.get(`${baseUrl}/v1/catalog/products?limit=24&salesMode=wholesale`, {
    headers: { accept: 'application/json' },
  });
  const ok = check(response, { 'catalogue renvoie 200': (result) => result.status === 200 });
  catalogueErrorRate.add(!ok);
  catalogueLatency.add(response.timings.duration);
  sleep(1);
}
```

`ops/performance/checkout.k6.ts` :

```typescript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';
import thresholds from './thresholds.json';

const baseUrl = __ENV.BASE_URL ?? 'http://localhost:4000';
const checkoutErrorRate = new Rate('checkout_errors');
const checkoutLatency = new Trend('checkout_latency');

export const options = {
  vus: 10,
  duration: '60s',
  thresholds: {
    checkout_latency: [`p(95)<${thresholds.checkout.p95_ms}`],
    checkout_errors: [`rate<${thresholds.checkout.error_rate}`],
  },
};

export default function (): void {
  const response = http.post(
    `${baseUrl}/v1/checkout/quote`,
    JSON.stringify({
      lines: [{ productId: '00000000-0000-4000-8000-000000000001', sku: 'CL-MOL-01-STD', quantity: 12 }],
      country: 'FR',
      incoterm: 'DAP',
    }),
    { headers: { 'content-type': 'application/json', accept: 'application/json' } },
  );
  const ok = check(response, { 'devis renvoie 200': (result) => result.status === 200 });
  checkoutErrorRate.add(!ok);
  checkoutLatency.add(response.timings.duration);
  sleep(1);
}
```

- [ ] **Step 5: Exécuter les tests et committer**

```bash
pnpm --filter @fanours/web test -- price-consistency.test.ts
```

Attendu : `Test Files  1 passed (1)` et `Tests  2 passed (2)`.

```bash
pnpm --filter @fanours/web build
```

Attendu : build Next.js réussi, sans erreur de type.

```bash
pnpm exec playwright test e2e/b2c/journey.spec.ts e2e/payment/scenarios.spec.ts --project=b2c --project=payment
```

Les deux fichiers appartiennent aux **projets de release** `b2c` et `payment` : le projet `functional` les ignore (règle « un fichier = un projet »), la commande cible donc leurs projets de release.

Attendu : toutes les assertions passent. Si l'environnement E2E (base, API, catalogue seedé) n'est pas disponible, la étape est **rapportée comme non exécutable** avec la raison exacte, jamais comme réussie.

```bash
pnpm exec playwright test e2e/visual-matrix.spec.ts --project=visual
```

Attendu : 48 tests passés. Les captures de référence sont **approuvées à la main** après revue : elles ne sont jamais auto-approuvées par `toHaveScreenshot` à la première exécution.

```bash
git add apps/web/src/lib/seo/price-consistency.ts apps/web/src/lib/seo/price-consistency.test.ts e2e ops/performance
git commit -m "test(catalog): ajoute la recette phase 2 et le controle de coherence des prix

Le controle de coherence compare le premier prix du tableau de paliers, le
prix du JSON-LD, la carte de grille et le bloc de titre : une divergence fait
echouer la CI et bloque la publication. Les parcours E2C B2C, les huit cas de
paiement, la matrice visuelle 48 combinaisons et les seuils k6 sont ajoutes."
```

---

## Self-Review

**1. Couverture du spec (§ Phase 2 de `docs/roadmap/sections/roadmap-phases.md`)**

| Point roadmap | Tâche | Artefact |
|---|---|---|
| 2.1 Catalogue (lecture, variantes, traductions) | Task 4 | `apps/api/src/modules/catalog` — `GET /v1/catalog/products`, `GET /v1/catalog/products/:slug` |
| 2.2 Paliers de prix + règles client | Task 5 | `packages/pricing` — `resolveTier`, `applyCustomerRule` |
| 2.3 MOQ, incréments, autorisation d'achat | Task 5 | `packages/pricing/src/moq-policy.ts` + `packages/domain/src/purchase-rules.ts` |
| 2.4 Stock, réservations, anti-survente | Task 6 | `apps/api/src/modules/stock` — `reserve`, `release`, `convert` + test de concurrence |
| 2.5 Recherche, facettes, filtres | Task 7 | `apps/api/src/modules/search` — tsvector GIN, curseur keyset, facettes |
| 2.6 Fiche produit | Task 10 | `apps/web/src/app/[locale]/produits/[slug]` — tableau de paliers, JSON-LD `Product` |
| 2.7 Panier | Task 11 | `apps/api/src/modules/cart` — panier serveur, revalidation, fusion |
| 2.8 Checkout | Task 12 | `apps/api/src/modules/checkout` — devis, recalcul, snapshot de commande |
| 2.9 Commandes | Task 13 | `apps/api/src/modules/order` — création idempotente, machine à états, outbox |
| 2.10 Paiement | Task 14 | `packages/payment` — `PaymentProvider`, `MockPaymentProvider` |
| 2.11 Webhooks | Task 15 | `apps/api/src/modules/webhook` — HMAC, tolérance 300 s, anti-rejeu |
| 2.12 E-mails transactionnels | Task 16 | `packages/i18n/src/email-templates.ts` + job `email.send` |
| 2.13 Factures et documents | Task 17 | `packages/pdf` — PDF `@react-pdf/renderer`, polices CJK, URL signée 5 min |
| 2.14 Notifications in-app | Task 18 | `apps/api/src/modules/notification` + `apps/web/src/components/account/NotificationList.tsx` |

Les 14 chantiers sont couverts, chacun par au moins une tâche. Les tâches transverses qui les rendent exécutables sont également présentes : Task 1 (monorepo, `config`, `contracts`, `domain` — dont la machine à états à 20 états et 34 transitions), Task 2 (extension du paquet `packages/db` créé par la Phase 1 : 10 migrations SQL manuelles `0006` à `0015` ajoutées aux `0001` à `0005` de la Phase 1, runner de la Phase 1 réutilisé via un adaptateur avec vérification de checksum), Task 3 (outbox transactionnelle, `defineJob`, backoff, consommateur idempotent), Task 8 (moteur de règles douanières et calcul des coûts d'import, exigé par le §11 du spec et consommé par le checkout), Task 19 (recette phase 2 : cohérence des prix SEO, parcours E2E B2C, 8 cas de paiement, matrice visuelle 48 combinaisons, seuils k6).

**2. Scan des placeholders**

Le plan ne contient aucun `TBD`, aucun `TODO`, aucun `à compléter`, aucun `XXX`, aucune formule de type « similaire à la Task N ». Chaque étape contient du code réel ou une commande exacte avec son résultat attendu. Aucun nom de table, de colonne ou de champ n'est inventé : le DDL des Tasks 2, 7 et 17 reprend les entités du modèle de données de `docs/roadmap/sections/architecture.md` (`price_tier` avec `EXCLUDE USING gist` sur `int4range`, `inventory_item` avec `CHECK (reserved >= 0 AND reserved <= on_hand)`, `inventory_movement` append-only, `order_status_event`, `webhook_event`, et les tables `outbox_event`, `consumed_event` et `audit_log` que la Phase 2 **étend** par `ALTER TABLE` dans `0012_notification_and_outbox.sql` sans jamais les recréer, l'append-only d'`audit_log` restant garanti par le déclencheur de la Phase 1), et le code des Tasks 1, 3, 8, 13, 14 et 15 reprend les contrats déjà arbitrés : `allowedTransitions(status, role)`, `SELECT … FOR UPDATE SKIP LOCKED` pour le relay outbox, `idempotencyKey(type, parts)`, `backoffMs(attempt)` en full jitter, `defineJob(type, {maxAttempts, queue, timeoutMs, idempotency}, handler)`, signature HMAC-SHA256 sur `${timestamp}.${rawBody}` avec comparaison temps constant et tolérance de 300 s, `NUMERIC(18,4)` en base et entiers de minor units en code. Les deux seuls emplacements volontairement non résolus sont explicitement documentés comme tels : les fichiers de police CJK (`packages/pdf/assets/fonts/README.md`, ajoutés par le mainteneur en phase 0) et la séquence `order_number_seq` / `invoice_number_seq`, créées par migration SQL et non par code applicatif.

**3. Cohérence des types et des noms**

- `CurrencyCode` (`packages/domain/src/currency.ts`) : `EUR | USD | CNY | GBP | JPY`, exposant 0 pour JPY. `toMinorUnits` / `fromMinorUnits` / `multiplyMinor` / `sumMinor` sont importés par le même chemin `@fanours/domain` dans les Tasks 5, 8, 10, 12, 14. Le montant transite en `bigint` (minor units) dans `packages/compliance` et en chaîne décimale `NUMERIC(18,4)` à la frontière base ; aucun `float` binaire ne porte un montant. ✔
- `OrderStatus` et `OrderTransition` : `ORDER_STATUSES` (20 états), `ORDER_TRANSITIONS` (T01…T34), `allowedTransitions(status, role)` et `findTransition(id)` sont produits par la Task 1 et consommés par la Task 13 ; la Task 13 refuse toute transition absente de la table avant toute écriture. ✔
- `PurchaseRulesResult` (`{allowed, minQuantity, orderMultiple, nextValidQuantity, reason}`) : produit par `packages/domain/src/purchase-rules.ts` (Task 1) et consommé à l'identique par `packages/pricing/src/moq-policy.ts` (Task 5), `apps/api/src/modules/cart/cart.service.ts` (Task 11) et `apps/api/src/modules/checkout/checkout.service.ts` (Task 12). Les raisons `ok | sales_mode_disabled | below_moq | not_multiple | invalid_quantity` sont les mêmes partout. ✔
- `TierResolution` (`{unitPrice, changed, tier}`) : produit par `resolveTier` (Task 5) et consommé par `PricingService.resolveUnitPrice` (Task 5), `CartService.resolvePrice` (Task 11) et `CheckoutService.resolvePrice` (Task 12) ; les trois lisent la même table `price_tier` filtrée par `channel`, `currency` et fenêtre de validité. ✔
- `ImportCostResult` / `TaxComponent` : produits par `packages/compliance/src/calculator.ts` (Task 8) et consommés par `CheckoutService.quote` (Task 12) via `amountMinor`, `certainty` et `text` ; `certainty` utilise les quatre niveaux `validated | estimated | incomplete | specialist_review` de l'architecture, sansalds. ✔
- `CheckoutQuote` / `OrderSnapshot` : `OrderSnapshot` (Task 12) est persisté tel quel dans `orders.snapshot` et relu par la Task 13 ; les champs `address`, `lines`, `incoterm`, `taxRules`, `termsVersion` existent dans les deux sens. ✔
- `PaymentProvider` (Task 14) : `createPayment`, `confirmPayment`, `refundPayment`, `getPayment`, `verifyWebhookSignature` — signatures identiques dans `packages/payment/src/payment-provider.ts`, dans `MockPaymentProvider` et dans les appels de `PaymentService` (Task 14) et `WebhookService` (Task 15). ✔
- Noms de routes : `/v1/catalog/products`, `/v1/search`, `/v1/cart`, `/v1/checkout`, `/v1/checkout/quote`, `/v1/orders`, `/v1/orders/:id/payment`, `/v1/webhooks/payment`, `/v1/notifications`, `/v1/orders/:id/invoice` — le préfixe `/v1` est constant, aucune route `/admin/*` n'est exposée par la Phase 2. ✔
- Noms de fichiers : `2026-10-21-phase-2-catalogue-et-commerce.md` suit la convention `docs/superpowers/plans/AAAA-MM-JJ-phase-N-<slug>.md` déjà posée par `2026-10-21-phase-0-cadrage-conformite.md`. ✔
- Numérotation des migrations SQL : continue et sans recouvrement entre les phases — Phase 1 `0001`–`0005` (extensions et table de suivi, identité, RBAC et audit, outbox, portées RBAC), Phase 2 `0006`–`0015` (catalogue, tarification, stock, panier, commande, paiement, notifications et extension outbox/audit, recherche, règles douanières, séquence de facture), Phase 3 `0100`–`0110`, Phase 4 `0111`–`0115`, Phase 5 `0116`–`0119`, Phase 6 `0120`–`0123`. Chaque phase n'écrit que ses propres migrations, dans l'ordre de création, et ne modifie ni ne recrée une migration déjà appliquée (append-only) : le paquet `packages/db`, son runner (cœur `src/migrate-core.ts` + CLI `scripts/migrate.ts`) et les tables `outbox_event`, `consumed_event` et `audit_log` appartiennent à la Phase 1, les phases suivantes étendent ce paquet. ✔
- Modules de schéma Drizzle : la Phase 1 est propriétaire de `packages/db/src/schema.ts` (identité, RBAC, audit, outbox) ; la Phase 2 ajoute les modules métier `packages/db/src/schema/{catalog,pricing,stock,cart,order,payment,notification}.ts`, créés par la task qui porte le domaine, et les expose par le baril `packages/db/src/schema/index.ts` étendu au fur et à mesure. `outbox_event`, `consumed_event` et `audit_log` ne sont pas redéclarés : la Phase 2 les **étend** par `ALTER TABLE` dans `0012_notification_and_outbox.sql`. ✔
- Commits : format Conventional Commits 1.0 avec un scope métier aligné sur les modules du spec (`infra`, `db`, `catalog`, `pricing`, `stock`, `i18n`, `checkout`, `order`, `payment`, `notification`, `invoice`, `test`). Chaque bloc `git commit` de ce plan se termine par `Co-Authored-By: Claude Code <noreply@anthropic.com>`, conformément aux règles d'attribution du dépôt ; aucune tâche n'est committée sur `main`. ✔














