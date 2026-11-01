## Architecture, stack et modele de donnees

Décisions figées pour Fanours. Chaque choix est justifié, l'alternative écartée est nommée, la contrainte induite est explicite. Les exigences citées renvoient au cahier des charges (CDC).

---

### 1. Décisions de stack figées

| Domaine | Choix figé | Raison | Alternative écartée | Contrainte induite |
|---|---|---|---|---|
| Front | Next.js 15 (App Router) + TypeScript `strict` + React 19 | CDC 5.1 impose Next.js App Router ; RSC réduit le JS client, ISR natif (CDC 23.1) | SPA Vite (SEO/LCP hors budget CDC 17.1) | Tout ce qui touche au compte/checkout/admin passe par des Server Components + Server Actions ou appels API ; zéro logique métier côté client (CDC 5.4) |
| UI | Tailwind v4 (config CSS-first) + tokens CSS en variables `--font-*`/couleurs CDC 3.3 | Les tokens CDC 3.3/3.4 sont des variables CSS ; Tailwind v4 les consomme nativement | CSS Modules, vanilla-extract (pas de pipeline de tokens partagé), shadcn/ui en socle (design non propriétaire, cf. risque « template IA ») | `@theme` en source unique des couleurs ; aucun hex littéral dans les composants ; `@apply` interdit (interdit le drift de design) |
| API | NestJS 11 (modules, DI, guards, interceptors) + Zod | CDC 5.1 : NestJS « ou équivalent » avec validation schémas, RBAC, OpenAPI ; NestJS livre les 3 nativement | Express/Fastify maison (RBAC, validation, OpenAPI à réécrire) | Contrôleurs minces ; logique métier dans les services de module ; DTO Zod = source unique pour validation + OpenAPI (zod → JSON Schema) |
| Base | PostgreSQL 16 (+RDS/Aurora ou Neon) + PgBouncer | CDC 5.1 ; besoin de `CHECK`, `EXCLUDE`, index partiels, JSONB, `tstzrange` (tarifs, règles datées) | MySQL (pas d'`EXCLUDE`, JSONB faible) | Schéma public unique par service ; `search_path` maîtrisé ; isolation `READ COMMITTED` + `SELECT … FOR UPDATE` sur les écritures concurrentes (CDC 7.3) |
| ORM | **Drizzle ORM** (+ SQL versionné à la main) | Voir §1.1 | Prisma (voir §1.1) | Pas de `push` : tout schéma passe par un fichier SQL reviewé |
| Cache / file | Redis 7 (Upstash ou ElastiCache) + **BullMQ** | CDC 5.1 : Redis + file de tâches ; BullMQ donne jobs retardés (expiration réservation, `trade_rule.review_due`), retry backoff, concurrence, flows | SQS+EventBridge (infra supplémentaire, isolation mais coût), Inngest/Temporal (surcharge Phase 1-3), Cron DB (impossible à hascler à l'échelle) | Worker = processus long-running obligatoire ; pas de serverless pour le worker |
| Stockage | S3-compatible (bucket public média + bucket privé documents) + CDN | CDC 5.1 et 11.8 | Disque local, stockage objet managé sans CDN | Octets jamais transités par l'API ; documents privés servis par URL signée courte durée ; région de données alignée sur l'entité (CDC 19.6) |
| Recherche | PostgreSQL (`pg_trgm` + `tsvector` multilingue) | CDC 5.1 : moteur PG au lancement | Elasticsearch/Typesense (infra + synchronisation pour un catalogue < 100k SKU) | Index GIN ; la bascule vers un moteur dédié se fait derrière la même interface `SearchProvider` |
| E-mail | Fournisseur transactionnel + SPF/DKIM/DMARC (CDC 5.1) | Exigence explicite | SMTP auto-hébergé (délivrabilité non garantie) | Envoi 100 % asynchrone via `EmailProvider` (CDC 14.4) |
| Paiement | Prestataire encapsulé (Stripe si éligible, sinon PSP autorisé — CDC 10.1) | CDC 10.1 interdit de coder en dur un prestataire non éligible | Stripe en dur | Interface unique `PaymentProvider` (§9) |
| Observabilité | Logs JSON (pino) + OpenTelemetry traces + Sentry + métriques Prometheus/Grafana | CDC 23.2 : logs structurés, IDs de corrélation, alertes | Logs texte, APM SaaS unique | `correlation_id` propagé web → API → worker → intégrations |
| Tests | Vitest + Testing Library + Playwright + supertest | CDC 5.1 et 24 | Jest (lente, ESM pénible) | Voir §7 |

#### 1.1 ORM : Drizzle choisi, Prisma écarté

**Drizzle :**
- Le CDC exige des migrations zero-downtime, expand/contract (§8), des `CHECK`/contraintes d'unicité métier en base, des index partiels et des colonnes `tstzrange`. Drizzle est une couche fine au-dessus de SQL : ces objets sont du SQL natif, versionné dans le repo, reviewable par un DBA.
- Pas de moteur binaire embarqué → cold start faible, image Docker slim, pas de problème de plateforme serveurless.

**Prisma écarté :** son moteur de migration prend un lock DDL et exige une shadow DB pour le diff ; la génération de SQL arbitraire (contraintes d'exclusion, index partiels, `tstzrange`) passe par du SQL non tracké, ce qui casse la garantie « schéma = source de vérité versionnée ». Son bénéfice principal (client typé fermé) est neutralisé par le fait que le pricing (§3) et la résolution des règles douanières (§4) se font en SQL (CTE, jointures fenêtrées) : on aurait de toute façon du SQL brut à maintenir.

**Contrainte :** Drizzle n'est pas un garde-fou. Le projet impose `drizzle-kit generate` **uniquement** pour produire un diff, chaque fichier `.sql` est relu, et les requêtes métier critiques (prix, conformité, disponibilité) sont écrites en SQL explicite via `sql` template avec paramètres, pas via le query builder chaîné.

#### 1.2 Stratégie de déploiement : conteneur unique multi-services (pas Vercel)

**Décision :** une seule plateforme conteneur (Railway / Fly.io / k8s interne selon CDC 28.2 §10), **une image Docker multi-stage, trois services** issus du même monorepo : `web` (Next.js `output: standalone`), `api` (NestJS), `worker` (BullMQ). CDN (Cloudflare/CloudFront) devant, cache HTTP pour les pages publiques, `next start` avec `custom cache handler` Redis pour l'ISR.

**Pourquoi pas Vercel :**

| Contrainte | Vercel | Conteneur |
|---|---|---|
| Worker BullMQ | Impossible : pas de processus long-lived ni socket Redis fiable | Processus dédié, scale horizontal sur `concurrency` |
| API NestJS | Second déploiement, donc second fournisseur, second coût, latence inter-appels | Même image, même réseau, `correlation_id` natif |
| Webhooks | Signature vérifiée sur une fonction serverless : cold start + `maxDuration` contraignant | Processus toujours chaud, `maxDuration` libre |
| PWA (CDC 22.3) | Service worker servi en statique : OK | Identique ; aucune contrainte supplémentaire |
| Coût | 3 offres/3 fournisseurs minimum | 1 offre, 3 services, un seul pipeline CI |
| Repli | Si le web tombe, l'API est ailleurs | DNS + reverse proxy pilotés, retour arrière image précédente |

**Contraintes retenues :** (a) le service worker ne met **jamais** en cache factures/commandes/données privées (CDC 22.3) ; (b) l'API n'est jamais exposée publiquement sans reverse proxy + rate limiting ; (c) le worker est déployé indépendamment et peut être mis à l'échelle à 0 sans arrêter les webhooks (ils sont consommés par l'API) ; (d) toute déploiement suit CDC 25.2 (build reproductible, E2E sur staging, rollback).

#### 1.3 Moteur de tâches : BullMQ

Jobs retardés requis par le CDC : expiration de réservation de stock (7.3), `trade_rule.review_due` (11.2, 21.3), e-mails transactionnels (14.4), exports (16.3), réconciliation transporteur (12.2), génération PDF (14.5), déblocage de shipments. BullMQ fournit la file à retardement + retry/backoff + DLQ + flow de commande sans dépendance supplémentaire à un service managé supplémentaire. Règle : **tout job est idempotent** et porte une `idempotency_key` (§6).

---

### 2. Architecture de code

**Monorepo pnpm + Turborepo : oui.** Les règles de tarification (§3) et le moteur de conformité (§4) doivent être exécutés à l'identique par le front (affichage du palier), l'API (prix au checkout) et le worker (revue de règle) ; le contrat OpenAPI et les clients doivent être générés depuis la même source ; les E2E traversent les trois applications. Un monorepo garantit une version unique, un commit atomique, un pipeline unique. **Contrainte :** frontières de packages contrôlées par `eslint-plugin-boundaries` + *project references* TS ; aucun import profond inter-apps ; `turbo` avec cache de build et pipeline `lint → typecheck → test → build` en dépendance.

```
fanours/
├─ apps/
│  ├─ web/                    # Next.js App Router (RSC). Pages publiques, compte, dashboard admin, PWA (manifest+SW).
│  │  ├─ src/app/[locale]/    # Routes FR / EN / ZH-CN (CDC 17.2)
│  │  ├─ src/app/(shop)/      # Catalogue, fiche produit, panier, checkout
│  │  ├─ src/app/(account)/   # Portail client (CDC 15)
│  │  ├─ src/app/(admin)/     # Dashboard admin, MFA obligatoire (CDC 16.1)
│  │  ├─ src/app/api/         # BFF mince: proxy/cache/ISR vers l'API + routes Next.js purement statiques
│  │  ├─ src/components/      # Design system (tokens CDC 3), aucune logique métier
│  │  ├─ src/lib/             # Clients HTTP typés, i18n (FR/EN/ZH-CN), formatteurs monétaires/date
│  │  └─ public/              # manifest PWA, service worker, icônes
│  ├─ api/                    # NestJS. Modules = domaines CDC 5.2 (15 modules).
│  │  ├─ src/modules/<domain>/ # controllers/, services/, dto/ (Zod), repository/, events/
│  │  ├─ src/common/          # guards RBAC, intercepteurs corrélation/erreurs, filtres, pagination
│  │  ├─ src/providers/        # implémentations PaymentProvider, ShippingProvider, EmailProvider, TaxRulesProvider, StorageProvider (§9)
│  │  └─ src/worker/          # Bootstrap BullMQ + handlers de jobs
│  └─ worker/                 # Processus BullMQ dédié: handlers, schedulers, DLQ replay
│     └─ src/jobs/            # par domaine: payments, orders, inventory, compliance, mail, shipments, reconciliation
├─ packages/
│  ├─ db/                     # Schéma Drizzle, clients, helpers de transaction, seeds, SQL de migration
│  │  ├─ migrations/          # Fichiers .sql versionnés (append-only) + registre des checksums
│  │  └─ src/queries/          # SQL métier paramétré (pricing, compliance, availability)
│  ├─ domain/                 # Types, enums, state machine (CDC 9.3), calculateurs purs (paliers, MOQ, arrondis)
│  ├─ contracts/              # Zod schemas partagés API↔web, types d'événements (CDC 21.3), OpenAPI généré
│  ├─ compliance/             # Moteur TradeCompliance: schéma de règles, résolution, niveaux de certitude (§4)
│  ├─ pricing/                # Résolution de prix: priorité CustomerPriceRule > PriceTier > promo (§3)
│  ├─ i18n/                   # Dictionnaires FR/EN/ZH-CN, validation de parité des clés
│  ├─ ui/                     # Design system React + tokens Tailwind (partagé web/admin)
│  ├─ observability/          # Logger, corrélation, métriques, trace helpers
│  └─ config/                 # tsconfig, eslint, prettier, env schemas (Zod), config turbo
├─ e2e/                       # Playwright: parcours CDC 24.3, multi-locale, multi-viewport
├─ infra/                     # Dockerfile multi-stage, compose local, config reverse proxy, dashboards Grafana
├─ ops/                       # Runbooks (CDC 23.2), seed staging, scripts de smoke test
├─ docs/                      # ADR, architecture, glossaire, runbooks
├─ playwright.config.ts        # Config Playwright UNIQUE (racine) — 7 projets, cf. qualite.md §24.4
├─ package.json               # Workspace root, scripts turbo
├─ pnpm-workspace.yaml
├─ turbo.json
└─ .env.example               # Noms de variables uniquement, jamais de valeurs
```

Règles de dépendance (flèches = imports autorisés) : `domain ← compliance ← {pricing, api, web}` ; `db → {pricing, compliance, api, worker}` ; `ui → web` ; `contracts → {api, web, worker}`. Un module `domain` n'importe **jamais** de framework ni d'ORM (CDC 5.4 : le frontend n'est pas source de vérité des prix/taxes).

---

### 3. Modèle de données

Conventions : `id UUID` (v7 pour tri temporel), `created_at/updated_at timestamptz NOT NULL DEFAULT now()`, montants `NUMERIC(18,4)` (jamais de flottant — CDC 4.3), devises `CHAR(3)` ISO, suppression logique `deleted_at timestamptz` quand pertinente (CDC 20.2). `tstzrange` pour les fenêtres de validité.

#### 3.1 Identité et accès

| Table | Colonnes clés | Index / uniques | Invariants base |
|---|---|---|---|
| `user` | `email citext NOT NULL`, `name`, `locale`, `theme`, `status`, `email_verified_at`, `password_hash` | UNIQUE(`email`); idx(`status`), idx(`created_at`) | `CHECK (status IN ('pending','active','suspended','closed'))`; e-mail normalisé minuscules; jamais de PAN/tokens ici (CDC 10.3) |
| `auth_identity` | `user_id`, `provider`, `provider_subject`, `linked_at` | UNIQUE(`provider`,`provider_subject`); idx(`user_id`) | `CHECK (provider IN ('password','google'))`; liaison seulement après preuve d'identité (CDC 13.2) |
| `session` | `user_id`, `token_hash`, `device`, `ip`, `user_agent`, `expires_at`, `revoked_at`, `last_seen_at` | UNIQUE(`token_hash`); idx(`user_id`,`revoked_at`); idx(`expires_at`) | Stockage du hash, jamais du token brut ; révocation en cascade sur changement de mot de passe (CDC 13.4) |
| `mfa_method` | `user_id`, `type`, `secret_ciphertext`, `status`, `confirmed_at` | UNIQUE(`user_id`,`type`) | Secret chiffré (CDC 13.3) ; jamais de code TOTP en clair |
| `recovery_code` | `user_id`, `code_hash`, `used_at` | idx(`user_id`) | `CHECK (used_at IS NULL OR used_at > created_at)`; usage unique |
| `role` / `permission` / `user_role` | `code`, `resource`, `action` | UNIQUE(`role.code`); UNIQUE(`permission.resource`,`permission.action`); UNIQUE(`user_id`,`role_id`) | Rôles CDC 16.12 ; permissions évaluées **côté serveur** (CDC 2.4, 19.2) |
| `business_account` | `legal_name`, `country`, `tax_id`, `verification_status`, `verified_at`, `verified_by` | idx(`verification_status`); idx(`country`) | `CHECK (verification_status IN ('unverified','pending','approved','rejected','suspended'))` |
| `business_membership` | `business_id`, `user_id`, `internal_role` | UNIQUE(`business_id`,`user_id`) | Le rôle interne n'accorde aucun privilège admin |
| `address` | `user_id`, `type`, `lines jsonb`, `city`, `region`, `postal_code`, `country_code`, `phone_e164`, `is_default` | idx(`user_id`); idx(`country_code`) | `CHECK (country_code ~ '^[A-Z]{2}$')`; snapshot figé dans la commande (CDC 20.2) |

#### 3.2 Catalogue, tarification et stock

| Table | Colonnes clés | Index / uniques | Invariants base |
|---|---|---|---|
| `product` | `sku UNIQUE`, `status`, `sales_modes text[]`, `moq int`, `order_multiple int`, `origin_country`, `hs_code`, `net_weight_g`, `gross_weight_g`, `length_mm/width_mm/height_mm`, `package_type` | UNIQUE(`sku`); idx(`status`); idx(`origin_country`); GIN(`sales_modes`) | `CHECK (moq > 0)`, `CHECK (order_multiple > 0)`, `CHECK (status IN ('draft','in_review','published','suspended','archived'))`; vente détail non contournable API (CDC 2.3) |
| `product_translation` | `product_id`, `locale`, `name`, `short_desc`, `description`, `slug`, `seo_title`, `seo_desc` | UNIQUE(`product_id`,`locale`); UNIQUE(`locale`,`slug`) | `CHECK (locale IN ('fr-FR','en','zh-CN'))`; nom non vide |
| `product_variant` | `product_id`, `sku UNIQUE`, `attributes jsonb`, `barcode`, `weight_g` | UNIQUE(`sku`); idx(`product_id`) | `CHECK (jsonb_typeof(attributes)='object')` |
| `product_media` | `product_id`, `variant_id`, `storage_key`, `media_type`, `position`, `alt_text`, `width`, `height` | idx(`product_id`,`position`); UNIQUE(`storage_key`) | `CHECK (media_type IN ('image','video'))`; alt_text non vide si image significative (CDC 22.4) |
| `category` / `category_translation` | `parent_id`, `position`, `locale`, `name`, `slug` | UNIQUE(`locale`,`slug`); idx(`parent_id`) | Pas de cycle : trigger ou contrôle applicatif |
| **`price_tier`** | `product_id`, `variant_id NULL`, `channel` ('wholesale'/'retail'), `currency`, `min_qty int`, `max_qty int NULL`, `unit_price NUMERIC(18,4)`, `valid_from`, `valid_to` | UNIQUE(`product_id`,`variant_id`,`channel`,`currency`,`min_qty`,`valid_from`); idx(`product_id`,`channel`,`valid_from` DESC) | `CHECK (unit_price > 0)`, `CHECK (min_qty > 0)`, `CHECK (max_qty IS NULL OR max_qty >= min_qty)`, `CHECK (valid_to IS NULL OR valid_to > valid_from)` ; **plages non chevauchantes** garanties par `EXCLUDE USING gist (product_id WITH =, COALESCE(variant_id,'-') WITH =, channel WITH =, currency WITH =, int4range(min_qty, COALESCE(max_qty, 2147483647),'[]') WITH &&) WHERE (valid_to IS NULL OR valid_to > now())` (impose `btree_gist`) — invariant clé CDC 7.2 |
| **`customer_price_rule`** | `customer_id` (business_account), `product_id` NULL, `variant_id` NULL, `scope` ('customer'/'group'), `min_qty`, `unit_price` ou `discount_pct`, `currency`, `priority int`, `approved_by`, `valid_from`, `valid_to` | UNIQUE(`customer_id`,`product_id`,`variant_id`,`min_qty`,`valid_from`); idx(`customer_id`,`valid_to`); idx(`priority`) | `CHECK (unit_price IS NOT NULL OR discount_pct IS NOT NULL)`, `CHECK (discount_pct BETWEEN 0 AND 100)`, `CHECK (valid_to IS NULL OR valid_to > valid_from)`, `approved_by NOT NULL` ; priorité **1 = client > 2 = groupe > 3 = palier standard > 4 = promo** (CDC 8.4, testé) |
| `warehouse` | `code`, `name`, `country`, `timezone` | UNIQUE(`code`) | Chaque stock est rattaché à un entrepôt |
| `inventory_item` | `warehouse_id`, `variant_id`, `on_hand`, `reserved`, `damaged`, `in_quality_control`, `incoming`, `safety_stock`, `version` | UNIQUE(`warehouse_id`,`variant_id`); idx(`variant_id`) | `CHECK (on_hand >= 0)`, `CHECK (reserved >= 0)`, `CHECK (reserved <= on_hand)`, `CHECK (incoming >= 0)` ; `version` pour optimistic lock |
| `inventory_movement` | `item_id`, `kind` ('in','out','reserve','release','adjust','transfer'), `qty`, `reason`, `actor_id`, `order_id` NULL, `occurred_at` | idx(`item_id`,`occurred_at` DESC); idx(`order_id`) | Append-only : jamais `UPDATE`/`DELETE` (CDC 7.3) ; `CHECK (qty <> 0)` |
| **`stock_reservation`** | `cart_id`/`user_id`, `variant_id`, `warehouse_id`, `qty`, `state` ('held','converted','released','expired'), `expires_at`, `idempotency_key` | UNIQUE(`idempotency_key`); idx(`expires_at`) WHERE `state='held'`; idx(`variant_id`,`state`) | `CHECK (qty > 0)` ; `CHECK (expires_at > created_at)` ; **l'anti-survente est assure par `SELECT … FOR UPDATE` sur `inventory_item` + `CHECK (reserved + qty <= on_hand)` dans la même transaction** (CDC 7.3, 26.2) ; un job planifié passe `held` → `expired` et libère le stock (CDC 24.2) |
| `product_compliance` | `product_id`, `variant_id`, `hs_code_proposed`, `hs_code_validated`, `hs_version`, `hs_authority`, `origin_country`, `restricted_countries text[]`, `required_documents text[]`, `notes`, `reviewed_by`, `reviewed_at` | UNIQUE(`product_id`,`variant_id`); idx(`hs_code_validated`) | `CHECK (origin_country <> '')`; `hs_code_validated IS NULL` ⇒ revue manuelle obligatoire (CDC 11.5) ; aucun code HS deviné par IA seul |

#### 3.3 B2B, panier, commandes

| Table | Colonnes clés | Index / uniques | Invariants base |
|---|---|---|---|
| `cart` / `cart_item` | `cart_id`, `variant_id`, `qty`, `price_snapshot jsonb`, `idempotency_key` | UNIQUE(`cart_id`,`variant_id`); UNIQUE(`idempotency_key`) | `CHECK (qty > 0)`; MOQ et multiple **recalculés serveur** à chaque action (CDC 9.1) |
| `promotion` / `promotion_redemption` | code, pays, segment, produits, min_amount, max_uses, used_count, valid_from/to | UNIQUE(`code`); UNIQUE(`promotion_id`,`order_id`) | `CHECK (used_count <= max_uses)`; non-cumulable sauf règles explicites (CDC 17.3) |
| **`quote`** | `number UNIQUE`, `business_id`, `user_id`, `status`, `currency`, `valid_until`, `incoterm`, `payment_terms`, `total_amount` | idx(`business_id`,`status`); idx(`valid_until`) | `CHECK (status IN ('draft','sent','reviewing','negotiating','accepted','rejected','expired','converted'))`; `CHECK (total_amount >= 0)`; `valid_until > created_at` |
| **`quote_line`** | `quote_id`, `revision_id`, `variant_id`, `qty`, `unit_price`, `price_basis jsonb` | UNIQUE(`revision_id`,`variant_id`); idx(`quote_id`) | `CHECK (qty > 0)`, `CHECK (unit_price >= 0)`; instantané du palier appliqué (CDC 7.2) |
| **`quote_revision`** | `quote_id`, `revision_no`, `author_id`, `total_amount`, `change_reason`, `created_at` | UNIQUE(`quote_id`,`revision_no`) | Append-only ; le client compare les révisions (CDC 8.3) |
| `quote_message` | `quote_id`, `author_id`, `visibility` ('client'/'internal'), `body` | idx(`quote_id`,`created_at`) | Notes internes invisibles du client (CDC 12.5) |
| **`order`** | `number UNIQUE`, `user_id`, `business_id`, `status`, `currency`, `total_amount`, `refunded_amount`, `idempotency_key`, `placed_at`, `paid_at`, `snapshot jsonb` | UNIQUE(`number`); UNIQUE(`idempotency_key`); idx(`user_id`,`status`); idx(`status`,`placed_at`); idx(`business_id`) | `CHECK (status IN (20 états CDC 9.3))`; `CHECK (total_amount >= 0)`; `CHECK (refunded_amount >= 0 AND refunded_amount <= total_amount)`; `snapshot` fige adresse, prix, taxes, CGV, langue, Incoterm, version de règle (CDC 11.4, 20.2) ; **commande payée jamais supprimée** (CDC 9.3) |
| **`order_line`** | `order_id`, `variant_id`, `sku`, `qty`, `unit_price`, `price_tier_ref`, `tax_amount`, `duty_amount`, `line_total`, `snapshot jsonb` | idx(`order_id`); idx(`variant_id`); idx(`sku`) | `CHECK (qty > 0)`, `CHECK (unit_price >= 0)`, `CHECK (line_total = qty*unit_price + tax_amount + duty_amount)` (arrondi commun documenté, CDC 4.3) |
| **`order_status_event`** | `order_id`, `from_status`, `to_status`, `actor_type` ('system','customer','admin','provider'), `actor_id`, `reason`, `correlation_id`, `occurred_at` | idx(`order_id`,`occurred_at`) | Append-only ; source de vérité de la machine à états (§5) ; `CHECK (from_status <> to_status)` |

#### 3.4 Paiements et remboursements

| Table | Colonnes clés | Index / uniques | Invariants base |
|---|---|---|---|
| **`payment`** | `order_id`, `provider`, `provider_payment_id`, `method`, `currency`, `amount`, `status`, `idempotency_key`, `paid_at` | UNIQUE(`provider`,`provider_payment_id`); UNIQUE(`idempotency_key`); idx(`order_id`); idx(`status`,`created_at`) | `CHECK (amount > 0)`; `CHECK (status IN ('pending','processing','succeeded','failed','expired','partially_refunded','refunded','disputed'))`; **aucune donnée de carte** (CDC 10.3, 19.4) ; `paid` jamais posé sur redirection navigateur (CDC 26.3) |
| **`payment_event`** | `payment_id`, `provider_event_id`, `type`, `raw_payload jsonb`, `signature_valid`, `received_at`, `processed_at`, `processing_result` | UNIQUE(`provider`,`provider_event_id`); idx(`payment_id`,`received_at`) | Idempotence par `provider_event_id` ; payload **chiffré/redigé** (pas de PAN dans les logs, CDC 19.7) |
| **`refund`** | `payment_id`, `provider_refund_id`, `amount`, `currency`, `reason`, `status`, `requested_by`, `approved_by`, `idempotency_key` | UNIQUE(`provider`,`provider_refund_id`); UNIQUE(`idempotency_key`); idx(`payment_id`) | `CHECK (amount > 0)`; `CHECK (status IN ('pending','succeeded','failed','cancelled'))`; **statut issu du prestataire**, jamais du clic admin (CDC 9.3, 10.4) ; double remboursement impossible (CDC 16.7) |
| `dispute` | `payment_id`, `provider_dispute_id`, `status`, `reason_code`, `amount`, `evidence_due_at` | UNIQUE(`provider_dispute_id`); idx(`status`) | Alerte support/comptabilité (CDC 10.3) |
| `invoice` / `credit_note` | `number UNIQUE`, `order_id`, `type` ('commercial_export','fiscal','receipt','credit_note'), `currency`, `total`, `issued_at`, `pdf_storage_key`, `locale`, `template_version` | UNIQUE(`number`); UNIQUE(`order_id`,`type`); idx(`issued_at`) | Numérotation non modifiable (CDC 14.5) ; **facture commerciale d'export ≠ facture fiscale** (CDC 11.8) ; PDF archivé, polices CJK embarquées (CDC 14.5) |
| `commercial_document` | `order_id`, `shipment_id`, `kind` ('packing_list','export_declaration','certificate_of_origin','customs_data'), `version`, `storage_key`, `sent_at` | UNIQUE(`order_id`,`kind`,`version`); idx(`shipment_id`) | Réutilise les données validées de la commande ; version envoyée conservée (CDC 11.8) |

#### 3.5 Logistique

| Table | Colonnes clés | Index / uniques | Invariants base |
|---|---|---|---|
| **`shipment`** | `order_id`, `carrier`, `service`, `tracking_number`, `status`, `incoterm`, `weight_g`, `dims jsonb`, `customs_data jsonb`, `dispatched_at`, `delivered_at` | UNIQUE(`carrier`,`tracking_number`); idx(`order_id`); idx(`status`); idx(`tracking_number`) | `CHECK (status IN ('label_created','handed_over','in_transit','customs_hold','action_required','out_for_delivery','delivered','exception','returned'))`; **pas de `shipped` sans shipment** (CDC 9.3) ; `customs_data` non vide ⇒ blocage expédition (CDC 11.9) |
| `shipment_item` | `shipment_id`, `order_line_id`, `qty` | UNIQUE(`shipment_id`,`order_line_id`); idx(`order_line_id`) | `CHECK (qty > 0)` ; somme par ligne ≤ quantité commandée (contrôle applicatif + test) |
| `package` | `shipment_id`, `reference`, `weight_g`, `dims` | UNIQUE(`shipment_id`,`reference`) | Colis multiple supporté (CDC 12.1) |
| **`tracking_event`** | `shipment_id`, `provider_event_id`, `status`, `location`, `occurred_at`, `received_at`, `raw_payload jsonb` | UNIQUE(`shipment_id`,`provider_event_id`); idx(`shipment_id`,`occurred_at`) | Idempotent + **protection contre le désordre** : on ne rétrograde pas un statut (comparaison de rang, CDC 12.2) |

#### 3.6 Conformité, communication, administration

| Table | Colonnes clés | Index / uniques | Invariants base |
|---|---|---|---|
| `country_profile` | `code CHAR(2) PK`, `name`, `delivery_enabled`, `incoterms text[]`, `currencies text[]`, `payment_methods text[]`, `tax_registration_required`, `required_docs text[]`, `restricted_categories text[]`, `operational_status` | PK(`code`) | `CHECK (code ~ '^[A-Z]{2}$')` ; suspension = `delivery_enabled=false` + `operational_status='suspended'` + justification auditée (CDC 11.6) |
| **`trade_rule`** | `key UNIQUE`, `country_code`, `scope` ('category'/'product'/'hs_prefix'/'global'), `category_id`/`product_id`/`hs_prefix`, `rule_type`, `certainty`, `priority`, `active` | UNIQUE(`key`); idx(`country_code`,`rule_type`); idx(`scope`,`category_id`) ; idx(`hs_prefix` text_pattern_ops) | `CHECK (rule_type IN ('vat','duty','threshold','flat_fee','license','prohibition','documentation','warning'))` ; `CHECK (certainty IN ('validated','estimated','incomplete','specialist_review'))` |
| **`trade_rule_version`** | `trade_rule_id`, `version_no`, `valid_from`, `valid_to`, `params jsonb`, `calculation_mode`, `taxable_base`, `thresholds jsonb`, `source_id`, `validated_by`, `validated_at`, `review_due_at` | UNIQUE(`trade_rule_id`,`version_no`); idx(`trade_rule_id`,`valid_from` DESC); idx(`review_due_at`) | `CHECK (valid_to IS NULL OR valid_to > valid_from)` ; `valid_to` **obligatoire** pour les seuils monétaires (CDC 11.7 : aucun seuil sans date) ; `EXCLUDE USING gist (trade_rule_id WITH =, tstzrange(valid_from, COALESCE(valid_to,'infinity'),'[)') WITH &&)` ⇒ pas de chevauchement de versions |
| `trade_rule_source` | `url`, `authority`, `title`, `consulted_at`, `retrieved_at`, `snapshot_key` | idx(`authority`) | Toute version de règle doit référencer une source (CDC 26.4) |
| `tax_estimate` / `duty_estimate` | `order_id`/`cart_id`, `country_code`, `currency`, `amount`, `low`, `high`, `certainty`, `rule_refs jsonb`, `computed_at`, `text_shown` | idx(`order_id`); idx(`country_code`,`computed_at` DESC) | `CHECK (certainty IN (...4 niveaux...))` ; `CHECK (low <= high)` ; fige source, version, horodatage et **texte présenté** (CDC 11.4) |
| `shipping_quote` | `origin`, `destination`, `weight_g`, `dims`, `provider`, `provider_quote_id`, `amount`, `currency`, `transit_days_min/max`, `expires_at` | idx(`destination`,`provider`); idx(`expires_at`) | `CHECK (amount >= 0)` ; `CHECK (transit_days_min <= transit_days_max)` ; **plages, pas promesses** (CDC 12.1) |
| `compliance_review` / `compliance_alert` | `product_id`, `order_id` NULL, `kind`, `severity`, `status`, `assignee_id`, `resolved_at` | idx(`status`,`severity`); idx(`order_id`) | Blocage expédition si `severity='blocker'` (CDC 11.9) |
| `incoterm_configuration` | `country_code`, `incoterm`, `tax_collection_mode` ('prepaid'/'at_delivery'), `enabled`, `effective_from` | UNIQUE(`country_code`,`incoterm`,`effective_from`) | Un Incoterm n'est activable que si l'organisation contractuelle existe (CDC 11.4) |
| `notification` / `notification_preference` | `user_id`, `type`, `channel`, `payload jsonb`, `read_at`, `locale` | idx(`user_id`,`read_at`); idx(`type`) | Les notifications de sécurité ne sont pas désactivables par une préférence marketing (CDC 15.7) |
| `email_template` / `email_delivery` | `template_key`, `locale`, `version`, `subject`, `body_html`; `delivery_id`, `to_email`, `template_version`, `locale`, `idempotency_key`, `status`, `provider_message_id`, `sent_at` | UNIQUE(`template_key`,`locale`,`version`); UNIQUE(`idempotency_key`); idx(`status`,`created_at`) | `CHECK (locale IN ('fr-FR','en','zh-CN'))` ; **clé d'idempotence par événement** ⇒ pas de doublon (CDC 14.4, 26.3) ; version de modèle conservée (CDC 14.3) |
| **`audit_log`** | `actor_type`, `actor_id`, `action`, `resource_type`, `resource_id`, `before jsonb`, `after jsonb`, `result`, `ip`, `user_agent`, `correlation_id`, `occurred_at` | idx(`actor_id`,`occurred_at` DESC); idx(`resource_type`,`resource_id`,`occurred_at` DESC); idx(`action`,`occurred_at` DESC) | **append-only** : `REVOKE UPDATE, DELETE` sur le rôle applicatif, trigger `BEFORE UPDATE OR DELETE` → `RAISE EXCEPTION` ; aucun secret/PAN (CDC 19.7) ; les rôles métier ne peuvent ni lire ni modifier (CDC 19.7) |
| `admin_action_approval` | `action`, `resource_type`, `resource_id`, `requested_by`, `approved_by`, `status`, `amount_threshold` | idx(`status`); idx(`resource_id`) | Double contrôle pour actions financières au-delà du seuil (CDC 19.2) |
| `feature_flag` | `key UNIQUE`, `enabled`, `rules jsonb`, `rollout_pct` | UNIQUE(`key`) | Flag critique ne peut pas être activé en prod sans revue |
| `webhook_event` | `provider`, `provider_event_id`, `type`, `signature_valid`, `payload jsonb`, `status`, `attempts`, `last_error` | UNIQUE(`provider`,`provider_event_id`); idx(`status`,`created_at`) | Idempotence ; `signature_valid=false` ⇒ rejeté et journalisé (CDC 19.4) |
| `background_job` | `queue`, `name`, `idempotency_key`, `state`, `attempts`, `payload jsonb`, `last_error` | UNIQUE(`queue`,`idempotency_key`); idx(`state`,`run_at`) | Miroir applicatif de BullMQ pour observabilité et DLQ (CDC 16.2) |
| `integration_health` | `provider`, `state`, `last_success_at`, `last_error`, `error_rate` | UNIQUE(`provider`) | Source du tableau de bord d'intégrations (CDC 16.2) |
| `support_ticket` / `support_message` / `attachment` | `order_id`/`payment_id`/`shipment_id`, `category`, `priority`, `status`, `sla_due_at`; `body`, `visibility`; `storage_key`, `mime`, `size` | idx(`status`,`priority`); idx(`order_id`); idx(`ticket_id`) | Notes internes invisibles du client ; upload MIME validé, taille max, nom sûr (CDC 19.3) |

**Index transverses à créer d'emblée :** `order(status, placed_at)`, `order(user_id, created_at DESC)`, `payment(status, created_at)`, `shipment(status, created_at)`, `product(status, category_id)`, `audit_log(occurred_at DESC)` pour purge/partitionnement futur, `tracking_event(shipment_id, occurred_at)`, `inventory_item(variant_id, warehouse_id)`.

**Invariants transverses garantis en base (et non seulement en code)** : quantités > 0 ; prix ≥ 0 ; lignes sans produit valide impossibles (FK `NOT NULL`) ; `order_line.line_total` = somme recalculée (CHECK) ; snapshot non vide sur commande confirmée ; idempotence par `UNIQUE` sur `order`, `payment`, `refund`, `webhook_event`, `email_delivery`, `stock_reservation` ; audit log non modifiable.

---

### 4. Moteur de règles douanières `TradeCompliance`

**Principe (CDC 11.2, 26.4, 28.1) :** aucun tarif, seuil, taux de TVA ou frais de courtage n'est écrit dans le code. Le code ne contient que l'**algorithme de résolution** et l'**interprétation de `params`**. La garantie est mécanique : un contrôle CI refuse toute constante de taux ou de seuil dans `apps/` et `packages/` (`/(vat|duty|tariff|threshold)\s*[:=]\s*0?\.\d+/i`), toute exception devant être déclarée explicitement dans `compliance/constants.allowlist.ts`, revu par le compliance manager.

**Tables :** `trade_rule`, `trade_rule_version`, `trade_rule_source` (§3.6) + `country_profile` + `tax_estimate`/`duty_estimate`.

**Résolution d'une règle applicable (déterminisme, ordre strict) :**

1. **Contexte d'entrée** : `country_code` (destination), `category_id` et `product_id` (ou `hs_code`), `origin_country`, `date` (date de commande ; date de livraison estimée si la règle l'exige), `mode` (`B2B`/`B2C`), `incoterm`, `carrier`, `is_business_importer`.
2. **Sélection des candidats** : `trade_rule.active = true` **et** une version `trade_rule_version` telle que `valid_from <= date` et (`valid_to IS NULL` ou `valid_to > date`). Les versions expirées **ne sont jamais appliquées silencieusement** (CDC 11.2) : une version expirée produit un `compliance_alert` et fait passer le résultat en `incomplete` si aucune version valide ne couvre la date.
3. **Filtrage de portée**, du plus spécifique au plus général : `product` > `variant` > `hs_prefix` > `category` > `global` ; à portée égale, priorité = `priority` décroissante, puis `valid_from` décroissante, puis `id` — l'ordre total garantit un résultat unique et reproductible.
4. **Filtrage contextuel** : `params.mode` (B2B/B2C), `params.incoterm`, `params.carrier`, `params.import_role`, `params.product_scope`.
5. **Contrôle d'expiration métier** : si `review_due_at < now()` → `compliance_alert` de criticité `warning` (CDC 11.2) ; si la règle est `rule_type IN ('prohibition','license')` et la version est expirée ⇒ **blocage** (la commande ne peut pas être confirmée).
6. **Calcul** via `calculation_mode` piloté par `params` : `percentage` (taux + base), `flat_amount` + `threshold`, `per_unit` + `unit_ref`, `progressive_tiers` (tranches), `component_sum`, `deferred_to_carrier` (valeur nulle ⇒ « à confirmer par le transporteur », CDC 11.3). `taxable_base` ∈ {`goods_value`, `goods+shipping`, `goods+shipping+insurance`, `custom`}. Aucun double comptage : une taxe déjà collectée au checkout est marquée `collected_by_fanours` et exclue du total dû à destination.
7. **Sortie** : pour chaque composante, `{amount_low, amount_high, currency, certainty, rule_refs[], source_url, computed_at, text_shown}`.

**Niveaux de certitude (CDC 11.2) et conséquences :**

| Niveau | Définition | Effet produit |
|---|---|---|
| `validated` | Source officielle identifiée, version validée par un responsable, date de revue future | Montant affiché comme certain ; utilisable en DDP |
| `estimated` | Source identifiée mais calcul dépendant d'une donnée manquante (poids exact, transporteur, HS définitif) | Fourchette `low..high` + mention « estimation » |
| `incomplete` | Données nécessaires absentes (HS non validé, origine inconnue, poids manquant) | Aucune valeur chiffrée ; « à confirmer » + **blocage d'expédition** si la donnée est requise par le transporteur (CDC 11.9) |
| `specialist_review` | Cas à traiter par un expert-comptable/déclarant | Panneau « avis professionnel requis », pas de montant, notification au compliance manager |

**Versionnage :** une règle n'est jamais modifiée en place. Modification = nouvelle `trade_rule_version` (`version_no` incrémenté) avec `valid_from` = date d'entrée en vigueur du texte source, `valid_to` = date de fin. Le `EXCLUDE USING gist` empêche deux versions actives qui se chevauchent. La commande fige les `rule_refs` (rule_id + version_id + params + source_url) dans `order.snapshot` et `tax_estimate` : la relecture d'une règle en 2027 ne modifie pas une commande de 2026 (CDC 1.2, 11.4).

**Alertes :** job quotidien `trade_rule.review_due` (CDC 21.3) : `review_due_at` dépassé → `compliance_alert` + e-mail au `compliance_manager` (CDC 11.2, 16.9) ; criticité `blocker` → blocage des confirmations de commande et des expéditions pour la portée concernée.

---

### 5. Machine à états des commandes

20 états, ordre de rang croissant : `draft(10) → pending_payment(20) → payment_processing(25) → paid(30) → payment_failed(28) → awaiting_review(40) → confirmed(50) → processing(60) → partially_fulfilled(70) → shipped(80) → in_transit(85) → customs_action_required(88) → delivered(90) → completed(100)`, plus `cancellation_requested(45)`, `cancelled(35)`, `refund_pending(95)`, `partially_refunded(97)`, `refunded(99)`, `disputed(96)`.

Règles transverses (CDC 9.3, 9.5) : toute transition écrit une ligne `order_status_event` (append-only) **dans la même transaction** que la mise à jour de `order.status` ; un rang ne peut jamais diminuer (interdit le retour arrière arbitraire, sauf transitions explicites listées) ; chaque transition déclenche ses effets de bord via l'**outbox** (§6) dans la même transaction ; le frontend ne peut proposer que les transitions autorisées pour son rôle.

| # | De | Vers | Acteurs autorisés | Effets de bord (side effects) |
|---|---|---|---|---|
| T01 | `draft` | `pending_payment` | client (serveur) | Snapshot prix/taxes figé ; création `payment` (`pending`) + `order` update ; événement `order.created` ; e-mail « commande créée, paiement en attente » ; job `payment.expire` (délai) |
| T02 | `pending_payment` | `payment_processing` | système (webhook PSP signé) | `payment.status='processing'` ; événement `payment.processing` (interne) ; aucune notif client |
| T03 | `payment_processing` | `paid` | système (webhook PSP signé **uniquement**) | `payment.status='succeeded'` + `paid_at` ; conversion des `stock_reservation` (`held`→`converted`, mouvement `inventory_movement` de type `out`) ; `order_line.snapshot` figé ; événements `payment.succeeded` + `order.paid` ; génération facture/reçu (job) ; e-mail confirmation ; autorisation de téléchargement des documents |
| T04 | `pending_payment`/`payment_processing` | `payment_failed` | système (webhook) | `payment.status='failed'` ; maintien des réservations jusqu'à expiration ; événement `payment.failed` ; e-mail d'échec avec lien de nouvelle tentative |
| T05 | `payment_failed` | `pending_payment` | client | Nouvelle tentative ⇒ nouvelle `payment` (jamais réutilisation de l'`idempotency_key`), `order_line` inchangées |
| T06 | `paid` | `awaiting_review` | système (règle métier : B2B, montant > seuil, HS non validé, Incoterm DDP) | Gel de l'expédition ; `compliance_review` créé ; notification au compliance manager + message client « en revue » ; blocage d'expédition |
| T07 | `awaiting_review` | `confirmed` | `compliance_manager`, `order_manager` | Levée du blocage ; événement `order.confirmed` ; e-mail client ; Incoterm et mode de collecte des taxes figés dans le snapshot |
| T08 | `awaiting_review` | `cancelled` | `compliance_manager` | Remboursement : `refund` créé (`pending`) → `refund_pending` si non confirmé ; événement `order.cancelled` ; e-mail |
| T09 | `confirmed` | `processing` | `logistics_operator` | `shipment`/liste de préparation créée (non obligatoire) ; statut interne « en préparation » ; notification interne |
| T10 | `confirmed` | `awaiting_review` | `compliance_manager` | Retour en revue (nouveau contrôle) — **seule** exception au rang non-décroissant, journalisée et motivée |
| T11 | `processing` | `partially_fulfilled` | `logistics_operator` | Une `shipment` avec `shipment_item` partiels créée ; `shipment.status='label_created'` ; événement `shipment.dispatched` partiel |
| T12 | `processing` | `shipped` | `logistics_operator` | **Précondition bloquante :** au moins une `shipment` existante + `customs_data` complète (CDC 9.3, 11.9) ; `shipment.status='handed_over'` ; documents d'export générés (`commercial_document`) ; événement `shipment.dispatched` ; e-mail + notification tracking |
| T13 | `processing` | `cancelled` | `order_manager` (+ MFA, CDC 13.3) | `refund` total `pending` ; mouvement de stock `in` (restitution) ; événement `order.cancelled` ; e-mail |
| T14 | `partially_fulfilled` | `shipped` | `logistics_operator` | Idem T12 pour le solde |
| T15 | `partially_fulfilled` | `partially_fulfilled` | `logistics_operator` | Nouvel envoi partiel (expéditions multiples, CDC 12.1) |
| T16 | `shipped` | `in_transit` | système (tracking) | `tracking_event` upsert idempotent ; mise à jour du statut shipment ; notification client aux jalons |
| T17 | `in_transit` | `customs_action_required` | système (tracking) / `logistics_operator` | `shipment.status='action_required'` ; `compliance_alert` ; **blocage** de la confirmation de livraison ; e-mail client « action requise » (documents/frais) + escalade support (CDC 11.9) ; événement `shipment.customs_action_required` |
| T18 | `customs_action_required` | `in_transit` | `logistics_operator` (levée) | Levée du blocage, note de résolution jointe ; journal d'incident (frais supplémentaires, colis retenu) |
| T19 | `in_transit` | `delivered` | système (tracking signé) | `shipment.status='delivered'`, `delivered_at` ; événement `shipment.delivered` ; e-mail de livraison ; relance automatique différée (J+3/J+10) si non réclamation |
| T20 | `delivered` | `completed` | système (délai + confirmation) ou client | Clôture du cycle de vente ; KPI mis à jour ; la commande devient non modifiable hors retour/litige |
| T21 | `delivered` | `disputed` | client / `support_agent` | Ouverture ticket lié ; gel des remboursements automatiques ; événement `order.disputed_opened` ; information comptable |
| T22 | `disputed` | `completed` | `support_agent` (résolution) | Clôture du litige ; décision + pièces conservées |
| T23 | `delivered`/`completed` | `cancellation_requested` | client | Demande enregistrée + ticket ; décision de retour à `cancelled` (avec remboursement) ou retour à l'état précédent (motif `request_rejected`) |
| T24 | `cancellation_requested` | `cancelled` | `order_manager` | Remboursement total : `refund` `pending` → `refund_pending` ; stock restitué si non expédié ; événement `order.cancelled` ; e-mail ; **commande non supprimée** (CDC 9.3) |
| T25 | `cancellation_requested` | `delivered`/`completed` | `order_manager` | Rejet motivé de la demande ; notification client |
| T26 | `cancelled` | `refund_pending` | `finance_manager` | `refund.status='pending'` ; le stock est déjà restitué ; événement `refund.requested` |
| T27 | `paid`/`confirmed`/`processing` | `refund_pending` | `finance_manager` (+ double contrôle si montant > seuil, CDC 19.2) | `refund` créé ; le passage à `refunded` dépend **exclusivement** de la confirmation du prestataire (CDC 9.3) |
| T28 | `refund_pending` | `partially_refunded` | système (webhook PSP) | `payment.amount` inchangé, `refunded_amount` mis à jour ; `payment.status='partially_refunded'` ; avoir généré si requis (CDC 10.4) ; événement `refund.completed` partiel ; e-mail |
| T29 | `refund_pending` | `refunded` | système (webhook PSP) | `refunded_amount = total_amount` ; `payment.status='refunded'` ; avoir final ; événement `refund.completed` ; e-mail ; le stock n'est pas restitué si le produit a quitté l'entrepôt |
| T30 | `partially_refunded` | `refunded` | système (webhook PSP) | Idem T29 pour le solde |
| T31 | `partially_refunded`/`refunded` | `partially_refunded`/`refunded` | `finance_manager` | Remboursement complémentaire (multi-remboursements autorisés, traçables dans `refund`) |
| T32 | `refund_pending` | `cancelled` (retour arrière) | `finance_manager` | Échec du remboursement : `refund.status='failed'` ; la commande **reste** `refund_pending` ou revient à l'état payée d'origine ; alerte `integration_health` |
| T33 | `delivered` | `completed` | `support_agent` (retour clos) | Fait partie de T20 ; les retours gèrent le remboursement via T27 |
| T34 | `disputed` | `partially_refunded`/`refunded` | `finance_manager` | Résolution de litige avec remboursement ; `dispute.status` mis à jour ; notification comptable |

**Garde-fous d'implémentation :** la table de transitions est **une seule source de vérité**, exportée en TypeScript (`packages/domain/src/order-state-machine.ts`) et dérivée vers un `CHECK` SQL sur `order.status` ; les routes API et le dashboard admin l'interrogent via une fonction `allowedTransitions(status, role)` — aucun `if (status === ...)` dispersé. Un test unitaire vérifie que **toutes** les paires (état, action) sont couvertes et que les 20 états du CDC 9.3 sont présents.

---

### 6. Événements et outbox

**Problème résolu (CDC 21.3) :** l'ordre « mise à jour de commande » et « publication d'événement » ne doivent pas se désynchroniser. Le pattern retenu est l'**outbox transactionnelle** : la mutation métier écrit l'événement dans `outbox_event` **dans la même transaction** que l'écriture métier ; un *relay* (job BullMQ récurrent ou process du worker) publie vers le broker ; les consommateurs sont idempotents.

**Schéma `outbox_event` :**

| Colonne | Type | Rôle |
|---|---|---|
| `id` | `uuid` PK | Identifiant technique |
| `event_id` | `uuid` UNIQUE | Identifiant de l'événement, propagé dans les headers et stocké par le consommateur (clé d'idempotence) |
| `event_type` | `text` | `order.created`, `payment.succeeded`… (CDC 21.3) |
| `event_version` | `int` | Version du schéma de payload (rétrocompatibilité) |
| `aggregate_type` / `aggregate_id` | `text` / `uuid` | `order`/`payment`/`shipment`/`quote` |
| `payload` | `jsonb` | Données de l'événement (jamais de PAN ni de secret, CDC 19.7) |
| `correlation_id` | `uuid` | Traverse web → API → worker → intégration (CDC 23.2) |
| `idempotency_key` | `text` UNIQUE | Clé d'idempotence **côté producteur** (ex. `payment.succeeded:payment:<id>`) |
| `state` | `text` | `pending` / `published` / `failed` / `dead` |
| `attempts` | `int` | Nombre de tentatives |
| `available_at` | `timestamptz` | Support du délai (retry exponentiel) |
| `published_at` | `timestamptz NULL` | Horodatage de publication |
| `last_error` | `text NULL` | Diagnostic |

**Indexes :** `idx(outbox_event_state_available_at) ON outbox_event (state, available_at) WHERE state IN ('pending','failed')` (extraction rapide, pas de scan de l'historique publié) ; `UNIQUE(idempotency_key)` ; `idx(payload->>'aggregate_id')` en GIN si recherche par agrégat ; partitionnement par `created_at` mensuelle à partir de 10 M lignes.

**Pattern publish/consume :**
1. Producteur : mutation métier + `INSERT INTO outbox_event` dans **une seule transaction** (garantie d'atomicité). Aucun appel réseau dans la transaction.
2. Relay : `SELECT … FOR UPDATE SKIP LOCKED` sur `(state, available_at)`, publication vers le transport (Redis Streams en interne, avec pont possible vers Kafka/SQS si le volume le justifie), puis `state='published'`, `published_at=now()`.
3. Consommateur : reçoit l'événement, calcule/porte sa **clé d'idempotence** = `event_id` (stocké dans `consumed_event(event_id, consumer, consumed_at)` avec `UNIQUE(event_id, consumer)` ou, pour les effets idempotents par nature, sur la clé métier : `refund.idempotency_key`, `email_delivery.idempotency_key`). Si la clé existe → acquittement immédiat, aucun effet de bord.
4. Échec : retry exponentiel (ex. 1s, 5s, 30s, 2 min, 10 min), puis `state='failed'` + alerte ; au-delà de 10 tentatives → `dead` + DLQ rejouable depuis le dashboard admin (CDC 14.4, 16.2).
5. Les webhooks entrants sont traités par le **même** mécanisme d'idempotence : `webhook_event(provider, provider_event_id)` UNIQUE (CDC 24.2 : webhook valide, invalide, dupliqué, désordonné).

**Les 16 événements du CDC 21.3 et leurs consommateurs :**

| Événement | Déclencheur | Consommateurs et actions requises |
|---|---|---|
| `user.email_verified` | Validation e-mail | E-mail de bienvenue ; déblocage des actions requérant un e-mail vérifié ; activation de la suggestion catalogue |
| `business.review_requested` | Demande d'accès B2B | File d'attente admin (badge dashboard) ; e-mail au compliance/support ; journal d'audit |
| `business.approved` | Approbation | Activation des `customer_price_rule` associées ; bascule du canal tarifaire en `wholesale` ; e-mail client ; invalidation du cache des prix du client |
| `quote.sent` | Envoi d'un devis | E-mail au client (PDF joint) ; notification au commercial ; `quote_revision` figée |
| `quote.accepted` | Acceptation | Passage en `accepted` ; job de conversion en commande (snapshot accepté conservé) ; e-mail client + commercial |
| `order.created` | Commande créée | E-mail « commande créée, paiement en attente » ; réservation de stock (job) ; analytics (non identifiant) |
| `order.confirmed` | Confirmation après revue | E-mail de confirmation ; génération facture/reçu (job PDF) ; levée des blocages conformité ; analytics CA |
| `payment.succeeded` | Webhook PSP signé | Transition `paid` ; conversion des réservations de stock ; facturation ; analytics paiement ; **aucune** donnée de carte transmise |
| `payment.failed` | Webhook PSP | E-mail d'échec + nouvelle tentative ; conservation des réservations jusqu'à expiration ; alerte si taux d'échec anormal |
| `refund.completed` | Webhook PSP | Transition `refunded`/`partially_refunded` ; mise à jour `refunded_amount` ; avoir ; e-mail client ; notification finance ; mise à jour KPI |
| `shipment.dispatched` | Expédition | Génération des documents d'export ; e-mail + notification tracking ; retrait des réservations de stock restantes ; mise à jour KPI logistique |
| `shipment.customs_action_required` | Tracking / contrôle | Alerte support ; e-mail client « action requise » (documents, frais) ; blocage de la confirmation de livraison ; journal d'incident |
| `shipment.delivered` | Tracking livré | E-mail de livraison ; relance satisfaction différée ; KPI délai de livraison |
| `invoice.issued` | Facture générée | Rattachement du PDF au dossier client ; e-mail avec lien sécurisé ; mise à jour du rapprochement |
| `inventory.low_stock` | Seuil franchi | Alerte catalogue/achats ; badge dashboard ; event tracking pour audit ; **jamais** d'e-mail au client |
| `trade_rule.review_due` | Job quotidien | E-mail + alerte au compliance manager ; si critique, blocage des confirmations de commande concernées ; mise à jour de `integration_health` |

**Événements supplémentaires requis par le reste du CDC** (même mécanisme) : `order.status_changed` (vue temps réel du dashboard), `order.cancelled`, `order.cancellation_requested`, `order.partially_fulfilled`, `shipment.tracking_updated`, `shipment.label_created`, `quote.revised`, `quote.expired`, `business.rejected`, `stock.reservation_expired`, `stock.reservation_released`, `user.mfa_enabled`, `user.password_changed`, `user.suspicious_login`, `admin.role_changed`, `compliance.alert_raised`, `support.ticket_opened`, `support.sla_breached`, `promotion.redeemed`.

---

### 7. Stratégie de tests par couche

Outils : **Vitest** (unitaires + intégration, un seul runner, ESM natif, compatible `pnpm` workspaces), **@testing-library/react** + **@testing-library/user-event**, **supertest** pour les tests d'API NestJS, **Playwright** pour l'E2E, **axe-core / @axe-core/playwright** pour l'accessibilité, **k6** pour la charge.

| Couche | Contenu | Outils | Requis |
|---|---|---|---|
| **Unitaire** | MOQ et multiples ; application des paliers et non-chevauchement ; priorités tarifaires client/groupe/palier/promo ; arrondis monétaires et devise (§4.3) ; transitions de la machine à états (§5) ; expiration et libération des réservations ; RBAC par rôle/permission ; résolution des règles douanières, dates d'effet, niveaux de certitude ; traitement des paiements/remboursements ; localisation FR/EN/ZH-CN (dates, montants, pluriels) ; idempotence des clés | Vitest, Testing Library | Couverture seuil **≥ 90 %** sur `packages/domain`, `packages/pricing`, `packages/compliance` (logique financière/douanière, CDC 24.6) ; ≥ 70 % global. **Aucun accès réseau, aucun accès DB** — fonctions pures |
| **Intégration** | API + PostgreSQL réelle (conteneur éphémère) : contraintes, transactions, verrous ; webhooks paiement (signature valide / invalide / dupliqué / désordonné) ; e-mails + file BullMQ (worker réel sur Redis de test) ; génération PDF (FR/EN/ZH) ; recalcul du checkout après changement d'adresse/quantité ; expiration de réservation et libération de stock ; import/export catalogue ; concurrence sur stock limité | Vitest + supertest + **testcontainers** (PostgreSQL + Redis) | `testcontainers` est justifié ici : les invariants critiques (contraintes `EXCLUDE`, `CHECK`, concurrence `SELECT FOR UPDATE`) **n'existent pas** sur un SQLite/mock ; un mock SQL ne prouverait rien |
| **E2E** | Les parcours du CDC 24.3 : B2C, B2B (inscription → vérification → approbation → devis → conversion → commande), paiement (succès/refus/en attente/webhook x2/retour navigateur sans webhook/double soumission/incohérence montant-devise), administration (MFA, refus de rôle, produit 3 langues, MOQ/prix, stock bas, expédition refusée si données douanières manquantes, règle mise à jour + alerte, export restreint, audit d'action financière), sécurité (IDOR par changement d'ID, jeton expiré, rate limit, CSRF/XSS, upload refusé, MFA erroné, route admin sans rôle) | Playwright | 6 viewports (320/360/390/430/768/1440) ; 3 locales ; thèmes clair/sombre ; run sur **staging** avant release (CDC 25.2 étape 7) |
| **Charge / résilience** (CDC 24.5) | Montée en charge catalogue/recherche/login/panier/checkout/webhooks ; accès concurrents à un stock limité ; retries et indisponibilité d'un prestataire ; e-mails retardés/échoués ; restauration de sauvegarde | k6 | Seuil : checkout p95 < 400 ms API, webhook < 200 ms, aucune survente |
| **Non testé en unitaire (explicitement)** | Requêtes SQL et plans d'exécution ; verrouillage et concurrence réelle ; rendering React complet (SSR vs CSR) ; comportement d'un `webhook` HTTP réel ; file BullMQ et ses retries ; PDF et polices CJK ; migrations et expansion/contract ; contrats de l'API publique ; parcours navigateur ; accessibilité réelle (clavier, lecteur d'écran) ; comportement des prestataires en sandbox | — | Ces éléments sont couverts en intégration/E2E ou manuellement, pas simulés en unitaire |
| **Statique / CI** | `tsc --noEmit` strict, ESLint (+ boundaries), Prettier, `npm audit`/OSV, gitleaks (secrets), `docker scan`, contrôle de non-cumul des `EXCLUDE`/CHECK, test de non-régression sur « aucun taux en dur » | CI | Aucun déploiement si un check bloquant échoue (CDC 24.6) |

---

### 8. Migrations

**Stratégie : SQL versionné à la main, en append-only.** Raison : les invariants du §3 (contraintes d'exclusion, index partiels, `tstzrange`, triggers d'immuabilité de l'audit log, `CHECK` de cohérence de totaux) sont du SQL avancé que ni Prisma ni l'ORM ne modélisent nativement ; et la garantie « schéma = source de vérité versionnée » exige un diff relu par un humain. Drizzle sert à typer l'accès ; `drizzle-kit generate` est utilisé **uniquement** pour produire un diff, jamais `push`.

**Organisation :** `packages/db/migrations/NNNN_nom.sql` (numéro croissant, jamais réécrit), un `schema_migrations` avec **checksum** du fichier appliqué (détection de migration modifiée a posteriori = erreur CI). Chaque migration est **irréversible par convention** : on n'écrit jamais de `down` ; le retour arrière se fait par code applicatif compatible avec l'ancien et le nouveau schéma (CDC 25.3).

**Zero-downtime :** cycle *expand / migrate / contract* en trois déploiements :
- **Expand** : ajout de colonnes `NULL` ou avec `DEFAULT` non volatil, nouvelles tables, nouveaux index `CREATE INDEX CONCURRENTLY`, nouvelle contrainte `NOT VALID`. Aucun changement de comportement applicatif.
- **Migrate** : le code écrit dans l'ancien **et** le nouveau format (dual write), backfill asynchrone par job (jamais un `UPDATE` massif dans la migration), puis bascule de lecture. Suppression des colonnes en lecture duale impossible : on garde les deux colonnes un cycle de déploiement.
- **Contract** : suppression de l'ancienne colonne, retrait de l'ancienne contrainte, `VALIDATE CONSTRAINT` (non `NOT VALID` bloquant). Un cycle complet est d'au moins 2 déploiements.

**Règles bloquantes :** (a) aucun index non-`CONCURRENTLY` sur une table > 100 000 lignes ; (b) jamais de `ALTER TYPE` sans `USING` explicite ni de modification de type de colonne en une étape ; (c) tout `NOT NULL` nouveau passe par `CHECK ... NOT VALID` puis `VALIDATE` ; (d) `SET lock_timeout` et `statement_timeout` posés sur chaque migration pour qu'un verrou bloquant échoue vite au lieu de tuer l'applicatif ; (e) jamais de transaction implicite longue autour d'un `CREATE INDEX CONCURRENTLY`.

**Garantie « aucune migration destructive en prod » (CDC 25.3) :** quatre verrous cumulés.
1. **Détection statique en CI** : un script analyse chaque `.sql` et refuse (échec build) la présence de `DROP TABLE`, `DROP COLUMN`, `TRUNCATE`, `RENAME`, `ALTER COLUMN … TYPE`, `DELETE FROM` sans `WHERE`, sauf si le fichier contient le marqueur explicite `-- @allow-destructive` **et** un `-- @adr: ADR-xxx` pointant vers une décision approuvée.
2. **Classification automatique de la migration** : fichier marqué `expand` / `contract` / `destructive` ; le pipeline exige une revue d'un second mainteneur pour la catégorie `destructive`.
3. **Environnement de test de migration** : chaque migration est appliquée sur une base **répliquée de la production** (anonymisée) puis la suite d'intégration est rejouée ; une migration qui échoue bloque le pipeline.
4. **Preuve de restauration** : avant toute migration `destructive`, snapshot PITR + test de restauration exécuté et horodaté dans le run de pipeline ; pas de snapshot vérifié, pas de déploiement.

**Rollback :** rollback **par code, pas par migration** (une migration destructive n'est pas réversible de façon fiable). Procédure : (1) déploiement de l'image précédente en quelques minutes, le schéma étant rétro-compatible grâce à expand/contract ; (2) si le rollback applicatif ne suffit pas (donnée corrompue), restauration PITR depuis le snapshot horodaté, avec une durée d'immobilisation à valider (RPO ≤ 24 h / RTO ≤ 8 h en première cible, CDC 19.8) ; (3) les données irréversibles (paiement, audit) ne sont jamais « annulées » par une migration : on les rectifie par un *backfill correctif* scripté et audité. Sauvegardes : quotidiennes PITR + hebdo complètes chiffrées, restauration testée mensuellement (CDC 19.8, 25.4).

---

### 9. Intégrations externes

Règle forte : **une seule interface interne par famille**, définie dans `packages/contracts` (Zod), implémentée dans `apps/api/src/providers/`, sélectionnée par configuration d'environnement. Le checkout, les documents, le worker et les tests ne connaissent que l'interface. Changer de prestataire = écrire une seconde implémentation + changer une variable d'environnement, sans réécrire le checkout (CDC 10.1).

| Intégration | Interface interne | Rôle | Simulé en local / staging | Secrets requis (noms, jamais en clair) |
|---|---|---|---|---|
| Paiement | `PaymentProvider` : `createIntent`, `confirm`, `refund`, `getStatus`, `verifyWebhook`, `getCapabilities(country, currency)` | Encaissement, 3DS, remboursements, réconciliation (§10) | `MockPaymentProvider` : intents simulés, webhooks rejouables, carte de test refusée ; staging utilise les clés sandbox du prestataire | `PAYMENT_PROVIDER`, `PAYMENT_API_KEY`, `PAYMENT_WEBHOOK_SECRET`, `PAYMENT_API_KEY_SANDBOX` |
| E-mail | `EmailProvider` : `send(template, locale, to, vars, idempotencyKey)` | Transactionnel FR/EN/ZH-CN, SPF/DKIM/DMARC (§14) | `LogEmailProvider` (écrit dans `email_delivery`, visible au dashboard) ; staging : boîte de réception interne | `EMAIL_PROVIDER`, `EMAIL_API_KEY`, `EMAIL_FROM_DOMAIN`, `EMAIL_DKIM_PRIVATE_KEY` |
| Transporteur / tracking | `ShippingProvider` : `getRates`, `createShipment`, `getTracking`, `cancel`, `parseWebhook` | Devis transport, étiquettes, tracking (§12) | `MockShippingProvider` : tarifs déterministes par zone, transporteurs simulés, événements de tracking injectables | `SHIPPING_PROVIDER`, `SHIPPING_API_KEY`, `SHIPPING_WEBHOOK_SECRET` |
| Calcul de transport | Même interface `ShippingProvider` (mode `rates`) + `TaxRulesProvider` | Estimation des frais (§12.1, §11.3) | Table tarifaire locale versionnée en base pour le dev | inclus |
| Règles fiscales / douane | `TaxRulesProvider` : `resolve(context) → TaxComponent[]` | Moteur `TradeCompliance` (§4) | Base de règles seedée, aucun appel réseau ; **jamais de taux en dur dans le code** | aucun secret (données en base, versionnées) |
| Stockage objet + CDN | `StorageProvider` : `put`, `getSignedUrl`, `delete`, `head` | Médias publics, documents privés, PDF, exports (§11.8, 14.5) | `LocalStorageProvider` : disque + serveur de fichiers de dev ; staging : bucket dédié | `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET_PUBLIC`, `S3_BUCKET_PRIVATE`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `CDN_BASE_URL` |
| File de tâches | Interne : BullMQ + Redis | Jobs retardés, retries, DLQ (§1.3) | Redis local en Docker, file en base locale | `REDIS_URL` |
| Monitoring / erreurs | `TelemetryProvider` : logs JSON, traces OTel, métriques | Alertes taux d'erreur, latence, files, webhooks (§23.2) | Sentry en mode dev + console ; Grafana/Prometheus locaux | `SENTRY_DSN`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `SENTRY_AUTH_TOKEN` |
| Analytics | `AnalyticsProvider` : `track(name, props)` | KPI business, respect du consentement (§23.3) | `NoopAnalyticsProvider` (no-op) + table `analytics_event` en base | `ANALYTICS_WRITE_KEY`, `ANALYTICS_PROVIDER` |
| Consentement cookies | `ConsentProvider` : `getCategory`, `setConsent`, `isAllowed` | Bandeau cookies, catégories, désinscription newsletter (§19.6, 17.3) | Implémentation interne (cookie signé + stockage local) — pas de tiers obligatoire | `CONSENT_SECRET` (signature du cookie) |
| Images produit (AVIF/WebP, CDN) | Même `StorageProvider` | Tailles multiples, `srcset`, `sizes` (§23.1) | Idem | idem |
| PDF / documents | Service interne `DocumentRenderer` (HTML → PDF, polices CJK embarquées) | Factures, avoirs, packing lists, certificats (§14.5, 11.8) | Rendu local identique à la prod (même image Docker) | aucun |
| Traduction / i18n | Fichiers de dictionnaire versionnés (§4.2) | FR/EN/ZH-CN | Aucun — fichiers dans le repo | aucun |

**Règles transverses d'intégration (CDC 21.4) :** chaque intégration a un *owner*, une clé de test, une procédure de staging, une politique de retry, une alerte, une procédure de rotation de secrets et un plan de secours ; `integration_health` suit `last_success_at` et `error_rate` ; aucune donnée de carte, aucun secret dans les logs (CDC 19.7) ; les webhooks entrants sont toujours vérifiés par signature **avant** toute écriture (CDC 19.4).

---

### 10. Performance et scalabilité

Cibles CDC 23.1 : **LCP ≤ 2,5 s, INP ≤ 200 ms, CLS ≤ 0,1 au p75**. Budget de performance mesuré en RUM (web-vitals), segmenté mobile/desktop.

**Stratégie de rendu par type de page (App Router) :**

| Type de page | Rendu | Cache | Justification |
|---|---|---|---|
| Accueil, catégories, pages éditoriales | **SSG** (build) + ISR (revalidate 3600) | CDN + `revalidateTag` à la publication | Contenu stable, indexable (CDC 17.1) |
| Fiche produit | **ISR** (revalidate 300 s) + revalidation on-demand à la modification produit | CDN | Le prix et le panier ne sont **pas** ISR (données personnalisées) ; la fiche est personnalisé côté client via Server Action |
| Catalogue / recherche | **SSR** avec `revalidate` court (60 s) et streaming des résultats (`Suspense` + skeletons) | CDN court + `unstable_cache` sur les agrégats | Pagination par curseur ; pas de scroll infini inaccessible (CDC 6.3) |
| Panier / checkout / compte / admin | **SSR dynamique**, `no-store` sur les données privées ; mise en cache du cadre (layout, tokens) uniquement | Aucun cache HTTP public | Données privées ; `Cache-Control: private, no-store` ; jamais de CDN sur ces routes |
| Pages statiques (CGV, politiques) | SSG | CDN long | Rarement modifiées |
| Tracking de commande | SSR + page `noindex` | Aucun | Lien sécurisé (CDC 6.1) |

**Cache (4 couches) :**
1. **CDN HTTP** : `Cache-Control: public, s-maxage=…` + `stale-while-revalidate` sur le public ; purge taguée à la publication.
2. **Cache Next.js / Redis** : `unstable_cache` + `cacheTag` pour les agrégats catalogue, catégories, pages statiques ; invalidation par tag.
3. **Cache applicatif Redis** : snapshots de prix par `(product, channel, currency)`, résolution de règles par `(country, category, date)`, rate limiting, sessions ; TTL court (60–300 s) et invalidation par événement (`business.approved`, `order.paid` invalide les clés concernées).
4. **Cache PostgreSQL** : catalogue public via vues matérialisées rafraîchies (`REFRESH MATERIALIZED VIEW CONCURRENTLY`) pour les compteurs du dashboard ; les KPI internes (CDC 16.2) sont recalculés hors chemin critique, par job planifié dans des tables de faits.

**Base de données — éviter les N+1 :**
- Requêtes d'agrégats en **une seule requête** : `SELECT … FROM order_line JOIN order … GROUP BY` plutôt qu'une boucle par ligne. Interdiction explicite des requêtes dans une boucle sur une collection (revue de code + règle ESLint sur l'usage de `await` dans un `for…of` sur un repository).
- **Agrégats pré-calculés** dans `daily_metric(date, country, currency, segment, metric, value)` alimentés par un job nocturne (CA, panier moyen, marge estimée — CDC 16.2) : le dashboard ne fait pas d'agrégation temps réel sur `order`.
- **Index** alignés sur les filtres du dashboard : `order(status, placed_at)`, `order_line(order_id)`, `payment(status, created_at)`, `shipment(status, created_at)`, `audit_log(actor_id, occurred_at DESC)`, GIN sur `payload` de l'outbox si filtrage.
- **Pagination** : *keyset* (curseur sur `(created_at, id)`) pour les listes croissantes ; *offset* interdit au-delà de 10 000 lignes. Pagination explicite dans toutes les réponses API (CDC 21.1).
- **Connexions** : PgBouncer en mode *transaction pooling* ; `statement_timeout` par rôle ; requêtes API ≤ 200 ms p95 (hors requêtes analytiques).
- **Reads** : les lectures admin peuvent passer par une réplica en lecture si la latence l'exige ; les lectures de checkout restent sur la primaire (cohérence du stock).

**Images et ressources (LCP/CLS) :** `next/image` avec AVIF/WebP, `sizes` explicite par point de rupture, `width`/`height` obligatoires (CLS), `priority` sur l'image LCP uniquement, `loading="lazy"` sous la ligne de flottaison, preload limité au LCP (CDC 3.2) ; polices auto-hébergées `next/font` avec `display: swap` et `preload` restreint (2 graisses max) ; SVG inline pour les icônes ; aucun script tiers sur le chemin critique (analytics chargé après consentement, donc absent du LCP) ; glassmorphism et animations bornées (CDC 3.6 : limiter les zones floutées simultanées sur mobile).

**INP ≤ 200 ms :** JavaScript client minimal — les Server Components portent l'affichage ; les actions de formulaire passent par des Server Actions ; le client n'est hydrate que sur les zones interactives (panier, checkout, dashboard) ; les listes longues utilisent la pagination, pas le rendu massif ; `useTransition` pour les recalculs (prix, taxes) ; bundle initial visé < 120 kB gzip sur le chemin d'entrée ; les composants d'admin ne sont jamais dans le bundle du site public (boundaries monorepo).

**Ce qui est mesuré :** web-vitals en RUM (p75 par page et par pays), `EXPLAIN (ANALYZE, BUFFERS)` revu pour toute requête > 100 ms, budget CI sur la taille du bundle, alertes sur p95 API, taux d'erreur 5xx, profondeur de file BullMQ, latence des webhooks, `integration_health`. Rappel : les seuils sont des objectifs ; la disponibilité cible 99,5 % et les RPO/RTO restent à valider par le propriétaire (CDC 23.2, 19.8, 28.2).



