# Phase 4 — Dashboards et Opérations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Livrer les 12 chantiers de la phase 4 du roadmap Fanours — dashboard client, back-office admin (commandes, catalogue, stock, clients/B2B, finance, logistique & douane, contenu & SEO), support, journal d'audit, analytics, notifications finalisées — et la recette de phase (11 cas E2E admin, audit axe-core des deux dashboards, non-régression sur les 14 actions sensibles RBAC, matrice responsive 8 viewports × 3 langues × 2 thèmes, revue sécurité exports + rapprochement financier, vérification manuelle clavier/lecteur d'écran).

**Périmètre :** 14 tâches (Task 1 → Task 14) — les 12 chantiers de la phase 4 (dashboard client, back-offices catalogue/stock/clients/finance/logistique/contenu, support, journal d'audit, analytics, notifications), le socle transversal RBAC et anti-IDOR, puis la recette de phase (suite de release `admin`).

**Architecture:** Monolithe modulaire NestJS (API `/v1`), Next.js 15 App Router (web), workers BullMQ. Le dashboard client lit des agrégats **recalculés côté serveur** ; le back-office admin écrit toujours dans la base via des services métier qui recalculent prix/taxes/frais et appliquent le RBAC serveur (9 rôles, deny-by-default) sur chaque endpoint. Aucune donnée d'administration n'est exposée par une route non protégée ; le journal d'audit (`audit_log`, append-only, chaîne SHA-256) est la source de vérité de toute action sensible. Le back-office est réservé : `noindex`, hors sitemap, accès contrôlé.

**Tech Stack:** TypeScript strict · NestJS 11 + Zod · PostgreSQL 16 + Drizzle (ORM unique) · Redis 7 + BullMQ · S3-compatible (bucket privé, URL signées 5 min) · Next.js 15 + React 19 + Tailwind v4 (tokens CSS uniquement) · Vitest 2.1.8 + Testing Library + supertest + Testcontainers · Playwright + axe-core.

**Spec:** `Cahier_des_charges_Fanours.md` §10.5, §12.5, §14.1–14.5, §15.1–15.8, §16.1–16.13, §17, §19.7, §20, §21, §22, §24, §28.1, §28.2 · `docs/roadmap/sections/roadmap-phases.md` §Phase 4 (lignes 151–183) · `docs/roadmap/sections/securite.md` §3 (matrice 14 actions sensibles) · `docs/roadmap/sections/design.md` §3.2, §3.4, §3.4.5, §3.9, §3.10 · `docs/roadmap/sections/architecture.md` · `docs/roadmap/sections/qualite.md` §24.4 · `docs/roadmap/sections/infra.md` §11.1–11.5 · `docs/roadmap/sections/seo.md` §17.

**Prérequis :** phases 1, 2 et 3 mergées (migrations `0001`–`0015`, `0100`–`0110`), socle RBAC Phase 1 (`permission`, `role`, `user_role`, `rbac_scope`, `admin_action_approval`, `audit_log`), commerce Phase 2 (`product`, `product_variant`, `price_tier`, `inventory_item`, `orders`, `payment`, `refund`, `invoice`, `notification`, `outbox_event`), B2B/logistique/douane Phase 3 (`business_account`, `shipment`, `package`, `tracking_event`, `commercial_document`, `return_request`, `dispute`, `dispute_message`, `trade_rule`, `country_profile`).

**Portée des migrations de la phase 4 :** `0111`–`0115` (5 migrations, append-only, checksum SHA-256, aucun `down`).

| Migration | Propriétaire | Contenu |
|---|---|---|
| `0111_phase4_client_account.sql` | **Task 1** | `address`, `user_preference`, `push_subscription` |
| `0112_phase4_catalog_ops.sql` | **Task 3** | `data_import`, `data_import_row`, `export_request` |
| `0113_phase4_stock.sql` | **Task 4** | `stock_count`, `stock_count_line`, `inventory_item.reorder_point/reorder_qty`, extension de `ck_permission_resource` (13 → 18 ressources) et de `ck_permission_action` (8 → 13 actions) |
| `0114_phase4_reconciliation.sql` | **Task 6** | `payment_reconciliation`, `payment_reconciliation_line` |
| `0115_phase4_content_seo.sql` | **Task 8** | `content_page`, `content_page_translation`, `redirect_rule`, `sale_condition` |

> Ordre d'application obligatoire : `0111` (T1) → `0112` (T3) → `0113` (T4) → `0114` (T6) → `0115` (T8). Le runner de migrations de `packages/db` applique les fichiers par version croissante et refuse tout fichier dont la version précède une version déjà appliquée.

---

## Global Constraints

- **Stack figée** — Next.js 15 + TypeScript strict + Tailwind v4 / NestJS + Zod / PostgreSQL 16 + Drizzle (ORM unique) / Redis + BullMQ / S3-compatible / Vitest + Playwright. Aucun framework ni ORM supplémentaire n'est introduit.
- **Monorepo pnpm + Turborepo** — un package = un `package.json`, un `tsconfig.json`, un `vitest.config.ts`. Les imports inter-packages passent par les alias `@fanours/*` déclarés dans `tsconfig.json` racine.
- **Zéro placeholder** — aucune chaîne `TODO`, `TBD`, `FIXME`, `à compléter` ni commentaire `implémenter la logique` ne subsiste dans le code produit par ce plan. Chaque valeur non spécifiée est explicitement marquée `à valider (décision propriétaire)`.
- **Cycle de test par tâche** — pour chaque étape fonctionnelle : test rouge → implémentation → test vert → commit. Aucun commit sans test vert.
- **Le front n'est jamais source de vérité** — prix, taxe, frais, stock, autorisation, disponibilité : tout est recalculé côté serveur. Les composants d'affichage ne font que rendre ce que l'API renvoie.
- **Jamais de float binaire** — montants en `NUMERIC(18,4)` en base, entiers de minor units en code, pourcentages en `rate_bps`. Formatage via `Intl` avec la locale et la devise de la requête.
- **Le statut `paid` ne vient jamais d'une redirection navigateur** — seul le webhook de prestataire, validé signature + anti-rejeu, fait transitioned une commande en `paid`.
- **Idempotence obligatoire** — toute opération sensible (import, export, rapprochement, action groupée, notification, job) porte une clé d'idempotence unique en base ou en Redis, et une soumission double ne double pas l'effet.
- **Anti-survente** — réservation de stock atomique (`UPDATE ... WHERE on_hand - reserved >= n`), jamais de vérification puis écriture séparées.
- **Les règles douanières sont des données** — aucune taux de taxe, de frais de dédouanement ou de Incoterm codé en dur : les règles vivent en base, versionnées, avec source officielle + URL + date de consultation, et le garde-fou `packages/compliance/scripts/check-no-hardcoded-rates.mjs` (Phase 3) continue de passer.
- **Aucune donnée de carte stockée ni journalisée** — ni `card`, ni `cvc`, ni corps de requête de paiement dans les logs ; les erreurs prestataire sont redactionnées.
- **Interdits absolus** — sous-évaluation, fausse catégorie ou fausse origine, fractionnement de colis, double remboursement, contournement du RBAC serveur, données de carte réelles en dev.
- **MFA obligatoire** — `super_admin`, `finance_manager`, `compliance_manager` (et `order_manager` sur les actions sensibles) doivent avoir une MFA vérifiée il y a moins de `MFA_MAX_AGE_MS = 10 * 60 * 1000` ms. Le secret TOTP est chiffré en envelope key, jamais en clair.
- **Quatre-yeux (four-eyes)** — `refund`, `financial_export`, `price_tier_change`, `role_change`, `country_suspension` passent par `admin_action_approval` : un second rôle approuve, l'auto-approbation est refusée par la contrainte `no_self_approval`, aucune auto-exécution, expiration à 24 h.
- **Journal d'audit append-only** — `audit_log` n'est ni modifiable ni supprimable (déclencheur `trg_audit_immutable` + `REVOKE UPDATE, DELETE`), chaînée en SHA-256 (`trg_audit_chain`). Aucun secret, aucun PAN, aucun corps de requête sensible dans `metadata`.
- **Anti-IDOR** — le scoping d'une ressource est appliqué **dans la clause `WHERE`** de la requête, jamais en post-filtrage mémoire. Une ressource appartenant à un autre tenant/utilisateur renvoie `404`, pas `403`.
- **Documents B2B chiffrés** — bucket privé, chiffrement côté serveur, URL signée d'une durée de 5 minutes, jamais d'URL publique, journal d'accès ligne par ligne.
- **Pas de docs admin dans le bundle public** — PDF jamais dans le bundle web ; les factures et avoirs sont servis en URL signée à durée courte.
- **i18n** — aucune chaîne en dur dans un composant : tout vient de `packages/i18n/src/locales/{fr-FR,en,zh-CN}.json`. Les 3 langues sont mises à jour dans la même tâche que la 1re chaîne, et le contrôle de parité des clés est exécuté.
- **Accessibilité WCAG 2.2 AA** — focus visible (`--focus-ring`), ordre de focus logique, pièges de focus en modale, `aria-live` sur les recalculs de total et les changements de stock, cibles ≥ 24×24 (2.5.8), `prefers-reduced-motion` respecté, jamais la couleur seule.
- **Zéro débordement horizontal** — sur les 5 breakpoints du spec (§2.5), `document.scrollWidth <= viewport.innerWidth` est asserté en E2E. Tableaux → cartes en mobile. Cibles tactiles ≥ 44 px sur mobile.
- **Tokens CSS uniquement** — aucune couleur en dur hors `packages/ui/src/styles/tokens.css` (ESLint `no-restricted-syntax`).
- **Pages admin et compte en `noindex`** — `/admin/**`, `/compte/**` et toutes les routes privées sont `noindex, nofollow` et exclues des sitemaps (`docs/roadmap/sections/seo.md` §17.1). Seules les pages publiques FR/EN/ZH-CN sont indexables.
- **Migrations** — SQL versionné à la main dans `packages/db/migrations/NNNN_nom.sql`, append-only, **aucun `down`**, checksum SHA-256 enregistré dans `schema_migrations`, `SET lock_timeout` et `SET statement_timeout` en tête de fichier, application via le runner dédié (jamais `db push`).
- **Migration destructive en production** — interdite sans accord explicite du propriétaire **et** restauration testée prouvée (test de restauration documenté dans la PR).
- **Secrets** — jamais en code, en log, en migration, en commit, en documentation. `.env.example` ne contient que des **noms** de variables.
- **Git** — jamais de développement ni de commit sur `main` ; branche `feat|fix|refactor|test|docs|chore/<slug>` ; jamais de force-push ; pas de commit de mélange ; **jamais de merge sans accord explicite du propriétaire** ; PR laissée ouverte avec preuves.
- **Commits** — Conventional Commits 1.0, scopes alignés sur les modules métier (`account`, `admin-orders`, `admin-catalog`, `admin-stock`, `admin-customers`, `admin-finance`, `admin-logistics`, `admin-content`, `support`, `audit`, `analytics`, `notifications`, `admin-core`, `infra`), titre ≤ 72 caractères, corps ligne ≤ 72 caractères, trailer `Co-Authored-By: Claude Code <noreply@anthropic.com>`.
- **Preuve** — pas de « terminé » sans sortie de test collée dans la PR. Les E2E doivent vérifier les effets (état, réseau, console), pas seulement des captures d'écran.
- **Modèle E2E canonique** — un seul fichier de configuration Playwright, `playwright.config.ts` **à la racine du dépôt** (créé en Phase 1, **modifié** — jamais recréé — par les Phases 2, 4, 5 et 6) ; aucun `playwright.config.ts` dans `apps/web/`, `apps/api/` ou `e2e/`. `testDir` vaut `'./e2e'` pour **tous** les projets, qui se distinguent uniquement par `testMatch` / `testIgnore` — aucun `testDir` par projet, donc aucune référence à un répertoire qui n'existe pas encore.

  | Projet | `testMatch` | Fichier |
  |---|---|---|
  | `b2c` | `**/b2c/journey.spec.ts` | `e2e/b2c/journey.spec.ts` |
  | `b2b` | `**/b2b/journey.spec.ts` | `e2e/b2b/journey.spec.ts` |
  | `payment` | `**/payment/scenarios.spec.ts` | `e2e/payment/scenarios.spec.ts` |
  | `admin` | `**/admin/dashboard.spec.ts` | `e2e/admin/dashboard.spec.ts` |
  | `security` | `**/security/security.spec.ts` | `e2e/security/security.spec.ts` |
  | `visual` | `**/visual/**/*.spec.ts` + `**/visual-matrix.spec.ts` | matrice visuelle |
  | `functional` | `**/*.spec.ts` moins les 6 ci-dessus | recettes fonctionnelles de phase |

  `testIgnore` de `functional` : `['**/b2c/journey.spec.ts', '**/b2b/journey.spec.ts', '**/payment/scenarios.spec.ts', '**/admin/dashboard.spec.ts', '**/security/security.spec.ts', '**/visual/**/*.spec.ts', '**/visual-matrix.spec.ts']`. **Un fichier appartient à un seul projet** : aucun doublon de suite, aucun comptage de tests ambigu. Les 5 premiers projets sont les **suites de recette de release** du spec §24.3 ; le projet `functional` porte les **recettes fonctionnelles de développement** de chaque phase. Les **viewports, les langues et les thèmes ne sont pas des projets Playwright** : les 8 viewports sont itérés **dans les fichiers de test** via `test.use({ viewport })`, à partir de la liste canonique `e2e/support/viewports.ts` (créée en Phase 1, **importée, jamais re-déclarée**) ; les 3 langues × 2 thèmes sont itérés dans le même `test.describe`. Invocations, lancées **depuis la racine** : `pnpm exec playwright test <chemin/vers/spec> --project=functional` (recette de phase), `pnpm exec playwright test --project=<suite>` (recette de release), `pnpm exec playwright test --project=visual` (matrice visuelle).

---

## File Structure

```text
fanours/
├─ packages/
│  ├─ db/
│  │  ├─ migrations/
│  │  │  ├─ 0111_phase4_client_account.sql        # Task 1
│  │  │  ├─ 0112_phase4_catalog_ops.sql            # Task 3
│  │  │  ├─ 0113_phase4_stock.sql                  # Task 4
│  │  │  ├─ 0114_phase4_reconciliation.sql         # Task 6
│  │  │  └─ 0115_phase4_content_seo.sql            # Task 8
│  │  └─ src/
│  │     ├─ schema/
│  │     │  ├─ phase4-account.ts                   # Task 1
│  │     │  ├─ phase4-catalog-ops.ts               # Task 3
│  │     │  ├─ phase4-stock.ts                     # Task 4
│  │     │  ├─ phase4-finance.ts                   # Task 6
│  │     │  └─ phase4-content.ts                   # Task 8
│  │     ├─ __tests__/
│  │     │  └─ migrations.integration.test.ts       # MODIFIÉ — Task 4 (compte de migrations défigé)
│  │     └─ seed.ts                                # Tasks 4, 8, 9, 11, 12 (lignes de permission)
│  ├─ domain/
│  │  └─ src/
│  │     ├─ permissions.ts                         # Tasks 4, 8, 9, 11, 12 (ressources) + Task 13 (catalogue actions sensibles)
│  │     ├─ audit-actions.ts                       # Task 13 (catalogue des actions journalisées)
│  │     └─ order-transitions.ts                   # Tasks 2, 13 (transitions par rôle)
│  ├─ analytics/                                   # Task 11
│  │  ├─ package.json
│  │  └─ src/
│  │     ├─ index.ts
│  │     ├─ kpi-definitions.ts
│  │     ├─ kpi-computation.ts
│  │     └─ kpi-definitions.test.ts
│  ├─ i18n/
│  │  └─ src/
│  │     ├─ locales/
│  │     │  ├─ fr-FR.json                          # Tasks 1→12
│  │     │  ├─ en.json                             # Tasks 1→12
│  │     │  └─ zh-CN.json                          # Tasks 1→12
│  │     └─ support-macros.ts                      # Task 9
│  ├─ ui/
│  │  └─ src/
│  │     ├─ components/
│  │     │  ├─ BulkActionBar.tsx                   # Task 2
│  │     │  ├─ InlineNote.tsx                      # Task 2
│  │     │  ├─ ReconciliationTable.tsx              # Task 6
│  │     │  ├─ TrendChart.tsx                      # Task 11
│  │     │  ├─ TrendChart.test.tsx                 # Task 11
│  │     │  └─ index.ts                            # Tasks 2, 6, 11
│  │     └─ styles/tokens.css                      # inchangé (source unique des tokens)
│  └─ contracts/
│     └─ src/
│        └─ notification-triggers.ts               # Task 12
├─ apps/
│  ├─ api/
│  │  └─ src/
│  │     ├─ common/
│  │     │  ├─ admin-audit.ts                      # Task 13
│  │     │  ├─ admin-context.ts                    # Task 13
│  │     │  ├─ admin-scope.ts                      # Tasks 2→12 (scoping par ressource) + Task 13
│  │     │  └─ sensitive-action.ts                  # Task 13
│  │     ├─ modules/
│  │     │  ├─ account/                            # Task 1
│  │     │  │  ├─ account.service.ts
│  │     │  │  ├─ account.controller.ts
│  │     │  │  ├─ account.module.ts
│  │     │  │  └─ account.service.spec.ts
│  │     │  ├─ admin-orders/                       # Task 2
│  │     │  │  ├─ admin-orders.service.ts
│  │     │  │  ├─ admin-orders.controller.ts
│  │     │  │  ├─ admin-orders.module.ts
│  │     │  │  └─ admin-orders.service.spec.ts
│  │     │  ├─ admin-catalog/                      # Task 3
│  │     │  │  ├─ admin-catalog.service.ts
│  │     │  │  ├─ admin-catalog.controller.ts
│  │     │  │  ├─ catalog-import.service.ts
│  │     │  │  ├─ catalog-parser.ts
│  │     │  │  ├─ admin-catalog.module.ts
│  │     │  │  └─ catalog-import.service.spec.ts
│  │     │  ├─ admin-stock/                        # Task 4
│  │     │  │  ├─ admin-stock.service.ts
│  │     │  │  ├─ admin-stock.controller.ts
│  │     │  │  ├─ admin-stock.module.ts
│  │     │  │  └─ admin-stock.service.spec.ts
│  │     │  ├─ admin-customers/                    # Task 5
│  │     │  │  ├─ admin-customers.service.ts
│  │     │  │  ├─ admin-customers.controller.ts
│  │     │  │  ├─ admin-customers.module.ts
│  │     │  │  └─ admin-customers.service.spec.ts
│  │     │  ├─ admin-finance/                      # Task 6
│  │     │  │  ├─ reconciliation.service.ts
│  │     │  │  ├─ finance-export.service.ts
│  │     │  │  ├─ admin-finance.controller.ts
│  │     │  │  ├─ admin-finance.module.ts
│  │     │  │  └─ reconciliation.service.spec.ts
│  │     │  ├─ admin-logistics/                    # Task 7
│  │     │  │  ├─ admin-logistics.service.ts
│  │     │  │  ├─ admin-compliance-config.service.ts
│  │     │  │  ├─ admin-logistics.controller.ts
│  │     │  │  ├─ admin-logistics.module.ts
│  │     │  │  └─ admin-logistics.service.spec.ts
│  │     │  ├─ admin-content/                      # Task 8
│  │     │  │  ├─ admin-content.service.ts
│  │     │  │  ├─ redirect.service.ts
│  │     │  │  ├─ admin-content.controller.ts
│  │     │  │  ├─ admin-content.module.ts
│  │     │  │  └─ admin-content.service.spec.ts
│  │     │  ├─ support/                            # Task 9
│  │     │  │  ├─ admin-support.service.ts
│  │     │  │  ├─ admin-support.controller.ts
│  │     │  │  ├─ admin-support.module.ts
│  │     │  │  └─ admin-support.service.spec.ts
│  │     │  ├─ audit/                              # Task 10
│  │     │  │  ├─ audit.service.ts
│  │     │  │  ├─ audit.controller.ts
│  │     │  │  ├─ audit.module.ts
│  │     │  │  └─ audit.service.spec.ts
│  │     │  ├─ analytics/                          # Task 11
│  │     │  │  ├─ analytics.service.ts
│  │     │  │  ├─ analytics.controller.ts
│  │     │  │  ├─ analytics.module.ts
│  │     │  │  └─ analytics.service.spec.ts
│  │     │  └─ notification/                       # Task 12
│  │     │     ├─ notification-triggers.service.ts
│  │     │     ├─ push.service.ts
│  │     │     ├─ notification.controller.ts
│  │     │     ├─ notification.module.ts
│  │     │     └─ notification-triggers.service.spec.ts
│  │     ├─ rbac/                                # Tasks 4, 8, 9, 11, 12, 13
│  │     │  ├─ rbac.guard.ts                     # Task 13 (étendu)
│  │     │  └─ __tests__/
│  │     │     └─ permissions.test.ts            # MODIFIÉ — listes EXPECTED_RESOURCES / EXPECTED_ACTIONS (Phase 1)
│  │     └─ providers/
│  │        └─ sms/
│  │           ├─ sms-provider.ts                  # Task 12
│  │           └─ mock-sms.provider.ts             # Task 12
│  ├─ worker/
│  │  └─ src/jobs/
│  │     ├─ catalog/catalog-import.job.ts          # Task 3
│  │     ├─ stock/stock-alert.job.ts               # Task 4
│  │     ├─ payment/payment-reconcile.job.ts       # Task 6
│  │     ├─ export/export-data.job.ts              # Task 6
│  │     └─ notification/notification-dispatch.job.ts # Task 12
│  └─ web/
│     └─ src/
│        ├─ app/[locale]/
│        │  ├─ (account)/compte/                   # Task 1 (+ Task 12 notifications)
│        │  │  ├─ page.tsx
│        │  │  ├─ loading.tsx
│        │  │  ├─ error.tsx
│        │  │  ├─ commandes/page.tsx
│        │  │  ├─ commandes/[number]/page.tsx
│        │  │  ├─ factures/page.tsx
│        │  │  ├─ adresses/page.tsx
│        │  │  ├─ preferences/page.tsx
│        │  │  ├─ notifications/page.tsx
│        │  │  └─ securite/page.tsx
│        │  └─ (admin)/admin/                       # Tasks 2→13
│        │     ├─ layout.tsx                       # Task 2 (coquille) + Task 13 (durcissement)
│        │     ├─ loading.tsx                       # Task 2
│        │     ├─ error.tsx                         # Task 2
│        │     ├─ page.tsx                          # Task 11 (vue globale)
│        │     ├─ commandes/page.tsx                # Task 2
│        │     ├─ commandes/[number]/page.tsx       # Task 2
│        │     ├─ catalogue/page.tsx                # Task 3
│        │     ├─ catalogue/import/page.tsx         # Task 3
│        │     ├─ stock/page.tsx                    # Task 4
│        │     ├─ clients/page.tsx                  # Task 5 (modifie la page Phase 3)
│        │     ├─ clients/[id]/page.tsx             # Task 5
│        │     ├─ finance/page.tsx                  # Task 6
│        │     ├─ finance/rapprochement/page.tsx     # Task 6
│        │     ├─ logistique/page.tsx               # Task 7
│        │     ├─ douane/configuration/page.tsx     # Task 7
│        │     ├─ contenu/page.tsx                  # Task 8
│        │     ├─ contenu/redirections/page.tsx     # Task 8
│        │     ├─ support/page.tsx                  # Task 9
│        │     ├─ journal/page.tsx                  # Task 10
│        │     └─ analytics/page.tsx                # Task 11
│        ├─ components/
│        │  ├─ account/                            # Tasks 1, 12
│        │  │  ├─ AccountOverview.tsx
│        │  │  ├─ AddressForm.tsx
│        │  │  ├─ NotificationPreferences.tsx
│        │  │  └─ NotificationPreferences.test.tsx
│        │  ├─ admin/                              # Tasks 2→13
│        │  │  ├─ AdminNav.tsx
│        │  │  ├─ AdminPageHeader.tsx
│        │  │  ├─ AdminEmptyState.tsx
│        │  │  ├─ AdminErrorState.tsx
│        │  │  ├─ OrderStatusTimeline.tsx
│        │  │  ├─ ReconciliationFilters.tsx
│        │  │  ├─ SupportThread.tsx
│        │  │  └─ AuditTimeline.tsx
│        │  └─ notifications/NotificationCenter.tsx # Task 12
│        ├─ server/
│        │  ├─ admin.ts                            # Tasks 2→13 (étendu)
│        │  └─ account.ts                          # Task 1
│        └─ lib/
│           └─ seo/private-routes.ts               # Task 2 (noindex admin/compte)
├─ e2e/
│  ├─ account/
│  │  ├─ dashboard.spec.ts                        # Task 1 — dashboard client (projet `functional`)
│  │  └─ notifications.spec.ts                    # Task 12 — centre de notifications (projet `functional`)
│  ├─ admin/
│  │  ├─ fixtures/
│  │  │  └─ admin-users.ts                        # Task 14
│  │  ├─ dashboard.spec.ts                        # Task 14 (11 cas) — suite de release `admin`
│  │  ├─ a11y.spec.ts                             # Task 14 (axe-core 2 dashboards)
│  │  ├─ security-regression.spec.ts               # Task 14 (14 actions sensibles)
│  │  ├─ exports.spec.ts                          # Task 14 (exports + rapprochement)
│  │  ├─ analytics.spec.ts                        # Task 11 (3 cas)
│  │  ├─ catalog-import.spec.ts                   # Task 3 (import/export catalogue)
│  │  ├─ stock.spec.ts                            # Task 4 (mouvements, seuils, inventaire)
│  │  ├─ finance.spec.ts                          # Task 6 (rapprochement, export quatre-yeux)
│  │  ├─ logistics.spec.ts                        # Task 7 (expéditions, exceptions, règles à revalider)
│  │  ├─ support.spec.ts                          # Task 9 (tickets, macros, SLA)
│  │  └─ audit.spec.ts                            # Task 10 (recherche, chaîne d'intégrité, export)
│  ├─ seo/
│  │  └─ content.spec.ts                          # Task 8 (canonical, hreflang, redirections, aperçus)
│  └─ visual/
│     └─ responsive.spec.ts                        # Task 14 — 8 viewports x 3 langues x 2 thèmes
├─ ops/
│  ├─ runbooks/
│  │  ├─ admin-backoffice.md                      # Task 14
│  │  └─ reconciliation-daily.md                  # Task 14
│  └─ checklists/
│     └─ phase4-manual-a11y.md                    # Task 14
└─ docs/
   └─ superpowers/plans/
      └─ 2026-10-21-phase-4-dashboards-et-operations.md
```

L'arborescence ci-dessus est la cible de fin de phase. Les fichiers marqués `# Task N` sont créés ou modifiés par la tâche indiquée ; une tâche ne touche jamais un fichier d'une autre tâche sans l'annoncer explicitement dans son bloc `**Files:**`.

---

### Task 1: Dashboard client (4.1) — compte, adresses, préférences, notifications push

**Files:**
- Create: `packages/db/migrations/0111_phase4_client_account.sql`
- Create: `e2e/account/dashboard.spec.ts` (recette fonctionnelle de phase — projet Playwright `functional`)
- Create: `packages/db/src/schema/phase4-account.ts`
- Modify: `packages/db/src/schema.ts`
- Create: `apps/api/src/modules/account/account.service.ts`
- Create: `apps/api/src/modules/account/account.controller.ts`
- Create: `apps/api/src/modules/account/account.module.ts`
- Test: `apps/api/src/modules/account/account.service.spec.ts`
- Create: `apps/web/src/app/[locale]/(account)/compte/page.tsx`
- Create: `apps/web/src/app/[locale]/(account)/compte/loading.tsx`
- Create: `apps/web/src/app/[locale]/(account)/compte/error.tsx`
- Create: `apps/web/src/app/[locale]/(account)/compte/commandes/page.tsx`
- Create: `apps/web/src/app/[locale]/(account)/compte/commandes/[number]/page.tsx`
- Create: `apps/web/src/app/[locale]/(account)/compte/factures/page.tsx`
- Create: `apps/web/src/app/[locale]/(account)/compte/adresses/page.tsx`
- Create: `apps/web/src/app/[locale]/(account)/compte/preferences/page.tsx`
- Create: `apps/web/src/app/[locale]/(account)/compte/securite/page.tsx`
- Create: `apps/web/src/components/account/AccountOverview.tsx`
- Create: `apps/web/src/components/account/AddressForm.tsx`
- Create: `apps/web/src/server/account.ts`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`
- Modify: `apps/api/src/app.module.ts`

**Interfaces:**
- Consumes: `orders` / `order_line` / `order_status_event` (Phase 2), `payment` (Phase 2), `invoice` (Phase 2), `shipment` + `tracking_event` (Phase 3), `quote` (Phase 3), `session` (Phase 1), `notification` + `notification_preference` (Phase 2), `packages/domain` → `Principal`, `packages/ui` → `KpiCard`, `DataTable`, `SystemState`.
- Produces:
  - `AccountService.overview(ctx: PrincipalContext): Promise<AccountOverviewDto>`
  - `AccountService.listOrders(ctx: PrincipalContext, query: AccountOrderQueryDto): Promise<Paged<AccountOrderRowDto>>`
  - `AccountService.getOrder(ctx: PrincipalContext, orderNumber: string): Promise<AccountOrderDetailDto>`
  - `AccountService.listInvoices(ctx: PrincipalContext, query: AccountInvoiceQueryDto): Promise<Paged<AccountInvoiceRowDto>>`
  - `AccountService.listPayments(ctx: PrincipalContext): Promise<AccountPaymentDto[]>`
  - `AddressService.list(ctx): Promise<AddressDto[]>`
  - `AddressService.create(ctx, input: AddressInputDto): Promise<AddressDto>`
  - `AddressService.update(ctx, id: string, input: AddressInputDto): Promise<AddressDto>`
  - `AddressService.remove(ctx, id: string): Promise<void>`
  - `PreferenceService.get(ctx): Promise<AccountPreferenceDto>`
  - `PreferenceService.update(ctx, input: AccountPreferenceInputDto): Promise<AccountPreferenceDto>`
  - `PushService.register(ctx, subscription: PushSubscriptionInputDto): Promise<void>` — signature de l'API Web Push, exposée par la Task 12.
  - Routes : `GET /v1/account/overview`, `GET /v1/account/orders`, `GET /v1/account/orders/:number`, `GET /v1/account/invoices`, `GET /v1/account/payments`, `GET|POST|PATCH|DELETE /v1/account/addresses`, `GET|PUT /v1/account/preferences`.

- [ ] **Step 1.1: Écrire le test rouge du résumé de compte et de l'anti-IDOR**

`apps/api/src/modules/account/account.service.spec.ts` :

```typescript
import { describe, expect, it, beforeEach } from 'vitest';
import { AccountService } from './account.service.js';
import { makeTestContext } from '../../../test/helpers/context.js';
import { seedAccountFixture } from '../../../test/helpers/account-fixture.js';

describe('AccountService', () => {
  let ctx: ReturnType<typeof makeTestContext>;
  let service: AccountService;

  beforeEach(() => {
    ctx = makeTestContext({ userId: 'aaaaaaaa-aaaa-4aaa-8aaa-aaaaaaaaaaaa' });
    service = new AccountService(ctx.db);
  });

  it('agrège les commandes en cours, les paiements en attente et les devis actifs', async () => {
    await seedAccountFixture(ctx.db, { userId: ctx.principal.userId, openOrders: 2, pendingPayments: 1, activeQuotes: 1 });

    const overview = await service.overview(ctx);

    expect(overview.openOrdersCount).toBe(2);
    expect(overview.pendingPaymentsCount).toBe(1);
    expect(overview.activeQuotesCount).toBe(1);
    expect(overview.currency).toBe('EUR');
    expect(overview.totals.grossRevenueMinor).toBe(1_254_00);
  });

  it('ne renvoie pas les commandes d’un autre utilisateur (404, pas 403)', async () => {
    const other = await seedAccountFixture(ctx.db, { userId: 'bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbbb', openOrders: 1 });

    await expect(service.getOrder(ctx, other.orderNumber)).rejects.toMatchObject({
      status: 404,
      code: 'order_not_found',
    });
  });
});
```

- [ ] **Step 1.1b: Écrire le test E2E rouge de la recette fonctionnelle du dashboard client**

`e2e/account/dashboard.spec.ts` — recette fonctionnelle de phase, projet Playwright `functional` (fichier distinct de la suite de release `e2e/admin/dashboard.spec.ts`, livrée par la Task 14). Écrit **avant** l'implémentation : il échoue à ce stade (page absente) et passe à l'étape « Traduire et vérifier ».

Cas et assertions :

1. Le dashboard client affiche les 4 cartes KPI (commandes, dépenses, factures, documents douaniers) ; les montants affichés proviennent de l'API (`GET /v1/account/overview`) et sont formatés en locale — le front ne recalcule aucun total.
2. Un compte sans commande affiche l'**état vide** localisé dans les 3 langues (`fr-FR`, `en`, `zh-CN`) ; aucun `0` n'est présenté à la place d'un montant absent.
3. Anti-IDOR : l'ID d'une commande d'un autre compte renvoie `404`, la page n'affiche aucun dossier et n'émet aucune erreur console.
4. Aucun débordement horizontal (`document.scrollWidth <= window.innerWidth`) sur les 8 viewports de la liste canonique `e2e/support/viewports.ts` (importée, jamais re-déclarée) ; les 4 cartes sont atteignables au clavier avec un focus visible.

- [ ] **Step 1.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/api test src/modules/account/account.service.spec.ts
```

Attendu :

```text
 FAIL  src/modules/account/account.service.spec.ts
Error: Cannot find module './account.service.js'
 ❯ src/modules/account/account.service.spec.ts:9:1
Test Files  1 failed (1)
     Tests  0 passed | 2 failed (2)
```

- [ ] **Step 1.3: Écrire la migration `0111`**

`packages/db/migrations/0111_phase4_client_account.sql` :

```sql
-- 0111_phase4_client_account.sql
-- Phase 4 — Task 1 : carnet d'adresses, préférences de compte, abonnements push.
-- Append-only : aucun down. Expansion pure (aucune donnée supprimée).
SET lock_timeout = '5s';
SET statement_timeout = '30s';

CREATE TABLE address (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      uuid NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
  label        text NOT NULL DEFAULT '',
  kind         text NOT NULL DEFAULT 'shipping',
  recipient    text NOT NULL,
  company      text,
  line1        text NOT NULL,
  line2        text,
  city         text NOT NULL,
  region       text,
  postal_code  text NOT NULL,
  country_code char(2) NOT NULL,
  phone        text,
  tax_id       text,
  is_default   boolean NOT NULL DEFAULT false,
  created_at   timestamptz NOT NULL DEFAULT now(),
  updated_at   timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT address_kind_check CHECK (kind IN ('shipping', 'billing', 'both')),
  CONSTRAINT address_country_check CHECK (country_code ~ '^[A-Z]{2}$'),
  CONSTRAINT address_recipient_check CHECK (length(btrim(recipient)) > 0),
  CONSTRAINT address_line1_check CHECK (length(btrim(line1)) > 0),
  CONSTRAINT address_city_check CHECK (length(btrim(city)) > 0),
  CONSTRAINT address_postal_check CHECK (length(btrim(postal_code)) > 0)
);
CREATE INDEX idx_address_user ON address(user_id, created_at DESC);
CREATE UNIQUE INDEX uq_address_one_default ON address(user_id) WHERE is_default;

CREATE TABLE user_preference (
  user_id          uuid PRIMARY KEY REFERENCES "user"(id) ON DELETE CASCADE,
  display_currency char(3) NOT NULL DEFAULT 'EUR',
  marketing_opt_in boolean NOT NULL DEFAULT false,
  sms_opt_in       boolean NOT NULL DEFAULT false,
  push_opt_in      boolean NOT NULL DEFAULT false,
  updated_at       timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT user_preference_currency_check CHECK (display_currency IN ('EUR', 'USD', 'CNY', 'GBP', 'JPY')),
  CONSTRAINT user_preference_display_only_check CHECK (display_currency <> '' )
);
COMMENT ON COLUMN user_preference.display_currency IS
  'Devise d''affichage uniquement ; la devise de transaction est celle de la commande (spec §4.4).';

CREATE TABLE push_subscription (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       uuid NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
  endpoint      text NOT NULL,
  p256dh        text NOT NULL,
  auth_secret   text NOT NULL,
  user_agent    text,
  created_at    timestamptz NOT NULL DEFAULT now(),
  last_used_at  timestamptz,
  revoked_at    timestamptz,
  CONSTRAINT push_subscription_endpoint_unique UNIQUE (endpoint)
);
CREATE INDEX idx_push_subscription_user ON push_subscription(user_id) WHERE revoked_at IS NULL;
```

- [ ] **Step 1.4: Appliquer la migration et le schéma Drizzle**

```bash
pnpm --filter @fanours/db migrate
```

Attendu :

```text
migrating 0111_phase4_client_account.sql
0111_phase4_client_account.sql applied (checksum 9f1c…)
```

`packages/db/src/schema/phase4-account.ts` expose `address`, `userPreference`, `pushSubscription` (colonnes strictement identiques au DDL, `pgTable` + `pgEnum`/`check` pour les contraintes `CHECK`).

- [ ] **Step 1.5: Implémenter `AccountService` avec scoping en `WHERE`**

Point critique : le filtre d'appartenance est dans la requête, jamais après coup.

```typescript
@Injectable()
export class AccountService {
  constructor(private readonly db: Database) {}

  async getOrder(ctx: PrincipalContext, orderNumber: string): Promise<AccountOrderDetailDto> {
    const rows = await this.db
      .select()
      .from(orders)
      .where(and(eq(orders.orderNumber, orderNumber), eq(orders.userId, ctx.principal.userId)))
      .limit(1);

    if (rows.length === 0) {
      throw new AppError('order_not_found', 404, { orderNumber });
    }
    // ... chargement lignes, statut, paiement, shipment, documents
  }
}
```

Les montants sont agrégés en `minor units` (entiers) ; la devise affichée vient de `user_preference.display_currency` mais la devise de transaction reste celle de la commande, et le front affiche la mention « indicatif » quand une conversion est appliquée (spec §4.4).

- [ ] **Step 1.6: Implémenter le contrôleur et le module**

`apps/api/src/modules/account/account.controller.ts` — toutes les routes sont sous `@UseGuards(SessionGuard)` (Phase 1) et renvoient `404` si la ressource n'appartient pas à l'utilisateur. `account.module.ts` déclare `AccountService`, `AddressService`, `PreferenceService` et l'import de `AccountModule` dans `apps/api/src/app.module.ts`.

- [ ] **Step 1.7: Tests verts**

```bash
pnpm --filter @fanours/api test src/modules/account/account.service.spec.ts
```

Attendu :

```text
 ✓ src/modules/account/account.service.spec.ts (2 tests)
 Test Files  1 passed (1)
      Tests  2 passed (2)
```

- [ ] **Step 1.8: Écrire la page dashboard client et ses états**

`apps/web/src/app/[locale]/(account)/compte/page.tsx` — rendu serveur (RSC) via `apps/web/src/server/account.ts` (miroir de `apps/web/src/server/admin.ts` de la Phase 3), avec :

- `loading.tsx` : skeleton de cartes KPI (`KpiCard` avec `aria-busy`).
- `error.tsx` : message localisé + action « Réessayer », **jamais** de trace technique.
- état vide : « Aucune commande pour le moment » + lien vers le catalogue.
- `AccountOverview.tsx` : 4 cartes (commandes en cours, paiements en attente, devis actifs, dernière livraison) + liste des 5 dernières commandes + bandeau d'alerte (paiement en attente, retour autorisé, litige ouvert).

Toutes les valeurs proviennent de `GET /v1/account/overview` ; aucun calcul de prix côté client.

- [ ] **Step 1.9: Carnet d'adresses et préférences**

`apps/web/src/app/[locale]/(account)/compte/adresses/page.tsx` + `AddressForm.tsx` : création, modification, suppression, « adresse par défaut » (contrainte d'unicité partielle `uq_address_one_default` gérée côté base ; l'API bascule l'ancienne adresse par défaut dans la même transaction). Les pays proposés viennent de `country_profile` (Phase 3) — une adresse ne peut pas viser un pays suspendu (contrôle serveur, erreur `country_suspended`).

`apps/web/src/app/[locale]/(account)/compte/preferences/page.tsx` : devise d'affichage (avec mention « indicatif »), opt-in marketing (indépendant de l'opt-out transactionnel), opt-in SMS (désactivé par défaut — coût externe non arbitré), opt-in push.

- [ ] **Step 1.10: Page sécurité du compte**

`apps/web/src/app/[locale]/(account)/compte/securite/page.tsx` : liste des sessions actives, révocation unitaire et globale, MFA actif/inactif, demande de code de récupération. Les opérations de sécurité sont journalisées et la révocation de session est immédiate (spec §7.2).

- [ ] **Step 1.11: Traduire les 3 langues et vérifier la parité**

```bash
pnpm --filter @fanours/i18n check:keys
```

Attendu :

```text
i18n key parity OK (3 locales, 412 clés)
```

Les nouvelles clés sont regroupées sous le namespace `account` : `account.overview.*`, `account.orders.*`, `account.invoices.*`, `account.addresses.*`, `account.preferences.*`, `account.security.*`, `account.states.*`.

- [ ] **Step 1.12: Vérifier le rendu et l'accessibilité du dashboard client**

```bash
pnpm --filter @fanours/web build
pnpm exec playwright test e2e/account/dashboard.spec.ts --project=functional
```

Attendu :

```text
  3 passed (12.4s)
```

Le test E2E vérifie : présence des 4 cartes KPI, état vide rendu quand aucune commande, absence de débordement horizontal (`document.scrollWidth <= window.innerWidth`) sur les viewports de la liste canonique `e2e/support/viewports.ts` (importée, jamais re-déclarée), navigation clavier vers les 4 cartes et focus visible.

**Preuves :**
- Sortie de `pnpm --filter @fanours/api test src/modules/account/account.service.spec.ts` (2 tests verts).
- Sortie de `pnpm --filter @fanours/db migrate` confirmant `0111` appliquée avec checksum.
- Sortie du build `apps/web` sans erreur de type ni de lint.
- Les 3 locales contiennent le namespace `account` (parité vérifiée).
- Test anti-IDOR : une commande d'un autre utilisateur renvoie `404 order_not_found` (pas de fuite de contenu).

**Commit :**

```bash
git add packages/db/migrations/0111_phase4_client_account.sql packages/db/src/schema/phase4-account.ts packages/db/src/schema.ts apps/api/src/modules/account apps/api/src/app.module.ts "apps/web/src/app/[locale]/(account)/compte" apps/web/src/components/account apps/web/src/server/account.ts packages/i18n/src/locales
git commit -m "feat(account): dashboard client, carnet d'adresses et préférences" -m "Implémente le chantier 4.1 du roadmap (spec §15.1-15.8) : synthèse de compte, historique de commandes, factures, paiements, carnet d'adresses et préférences. Migration 0111 append-only. Scoping par user_id dans la clause WHERE, 404 sur ressource étrangère. 3 langues, états loading/vide/erreur." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 2: Back-office commandes (4.2) — vue globale, transitions, actions groupées, coquille admin

**Files:**
- Create: `apps/api/src/modules/admin-orders/admin-orders.service.ts`
- Create: `apps/api/src/modules/admin-orders/admin-orders.controller.ts`
- Create: `apps/api/src/modules/admin-orders/admin-orders.module.ts`
- Test: `apps/api/src/modules/admin-orders/admin-orders.service.spec.ts`
- Modify: `apps/api/src/common/admin-scope.ts`
- Create: `packages/ui/src/components/BulkActionBar.tsx`
- Test: `packages/ui/src/components/BulkActionBar.test.tsx`
- Create: `packages/ui/src/components/InlineNote.tsx`
- Test: `packages/ui/src/components/InlineNote.test.tsx`
- Modify: `packages/ui/src/components/index.ts`
- Create: `apps/web/src/app/[locale]/(admin)/admin/layout.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/loading.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/error.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/commandes/page.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/commandes/[number]/page.tsx`
- Create: `apps/web/src/components/admin/AdminNav.tsx`
- Create: `apps/web/src/components/admin/AdminPageHeader.tsx`
- Create: `apps/web/src/components/admin/AdminEmptyState.tsx`
- Create: `apps/web/src/components/admin/AdminErrorState.tsx`
- Create: `apps/web/src/components/admin/OrderStatusTimeline.tsx`
- Create: `apps/web/src/lib/seo/private-routes.ts`
- Modify: `apps/web/src/server/admin.ts`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `orders`, `order_line`, `order_status_event` (Phase 2), `payment` (Phase 2), `shipment` + `tracking_event` (Phase 3), `audit_log` (Phase 1), `packages/domain` → `allowedTransitions(status, role)`, `packages/ui` → `DataTable`, `FilterBar`, `KpiCard`, `SystemState`.
- Produces:
  - `AdminOrdersService.list(ctx: AdminContext, query: AdminOrderQueryDto): Promise<Paged<AdminOrderRowDto>>`
  - `AdminOrdersService.detail(ctx: AdminContext, orderNumber: string): Promise<AdminOrderDetailDto>`
  - `AdminOrdersService.transition(ctx: AdminContext, orderNumber: string, input: OrderTransitionInputDto): Promise<AdminOrderDetailDto>`
  - `AdminOrdersService.bulk(ctx: AdminContext, input: AdminOrderBulkInputDto): Promise<AdminOrderBulkResultDto>`
  - `AdminOrdersService.addInternalNote(ctx: AdminContext, orderNumber: string, body: string): Promise<AuditEntryDto>`
  - `AdminOrdersService.internalNotes(ctx: AdminContext, orderNumber: string): Promise<AuditEntryDto[]>`
  - Routes : `GET /v1/admin/orders`, `GET /v1/admin/orders/:number`, `POST /v1/admin/orders/:number/transitions`, `POST /v1/admin/orders/:number/notes`, `POST /v1/admin/orders/bulk`.

- [ ] **Step 2.1: Écrire le test rouge des transitions par rôle**

`apps/api/src/modules/admin-orders/admin-orders.service.spec.ts` :

```typescript
describe('AdminOrdersService.transition', () => {
  it('refuse une transition non autorisée par le rôle et journalise le refus', async () => {
    await seedOrder(ctx.db, { orderNumber: 'FO-2026-000042', status: 'awaiting_payment' });

    await expect(
      service.transition(ctxAsRole('catalog_manager'), 'FO-2026-000042', { transitionId: 'mark_paid' }),
    ).rejects.toMatchObject({ status: 403, code: 'transition_not_allowed' });

    const entries = await readAudit(ctx.db, { action: 'order.transition.denied', objectId: 'FO-2026-000042' });
    expect(entries).toHaveLength(1);
    expect(entries[0].result).toBe('denied');
  });

  it('applique une transition autorisée et journalise le succès', async () => {
    await seedOrder(ctx.db, { orderNumber: 'FO-2026-000043', status: 'awaiting_payment' });

    const detail = await service.transition(ctxAsRole('order_manager'), 'FO-2026-000043', { transitionId: 'mark_paid' });

    expect(detail.status).toBe('paid');
    expect(detail.events.at(-1)?.to).toBe('paid');
  });
});
```

- [ ] **Step 2.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/api test src/modules/admin-orders/admin-orders.service.spec.ts
```

Attendu :

```text
 FAIL  src/modules/admin-orders/admin-orders.service.spec.ts
Error: Cannot find module './admin-orders.service.js'
Test Files  1 failed (1)
     Tests  0 passed | 2 failed (2)
```

- [ ] **Step 2.3: Implémenter le service de lecture (list + detail)**

`AdminOrderQueryDto` (zod) : `status?`, `country?`, `currency?`, `segment?` (`b2c` | `b2b`), `from?`, `to?`, `paymentStatus?`, `limit` (défaut 25, max 100), `cursor?`, `q?` (numéro de commande, correspondance exacte ou préfixe).

`list()` :

- Applique le scoping du rôle via `apps/api/src/common/admin-scope.ts` (Phase 3) : `order_manager` voit toutes les commandes ; `logistics_operator` ne voit que celles qui lui sont assignées ; `support_agent` ne voit que celles liées à un ticket dont il est responsable.
- Pagination **serveur** par curseur (`created_at, id`).
- Ne renvoie jamais de champ interne : `snapshot` n'est exposé qu'à `order_manager` et `finance_manager`.

`detail()` renvoie : en-tête (numéro, statut, dates, devise), client (nom, pays, segment), lignes (produit, SKU, quantité, PU, total, snapshot de prix et de règle de taxe appliqué), paiements (statut, moyen, ID prestataire masqué, montants), expédition (transporteur, tracking, colis), documents (nom, type, statut de génération), timeline des transitions et notes internes.

- [ ] **Step 2.4: Implémenter les transitions et le journal**

```typescript
async transition(ctx: AdminContext, orderNumber: string, input: OrderTransitionInputDto) {
  const order = await this.loadOrderScoped(ctx, orderNumber); // 404 si hors périmètre
  const allowed = allowedTransitions(order.status, ctx.principal.role);
  if (!allowed.includes(input.transitionId)) {
    await this.audit.deny(ctx, 'order.transition.denied', 'order', order.id, { transitionId: input.transitionId });
    throw new AppError('transition_not_allowed', 403, { status: order.status, transitionId: input.transitionId });
  }
  // transaction : order_status_event + mise à jour du statut + audit + outbox_event
}
```

Chaque transition écrit **deux** entrées d'audit : une tentative et un résultat, ou une seule entrée `result = 'success' | 'denied'` (conforme à la contrainte `audit_log.result`). Le choix retenu : une entrée par tentative, avec `result` porté par la ligne — la table `audit_log` l'impose.

- [ ] **Step 2.5: Implémenter les actions groupées**

`AdminOrderBulkInputDto` : `{ orderNumbers: string[] (1..200), action: 'export' | 'print' | 'add_note', payload?: { body?: string } }`.

- `export` : produit un CSV (numéro, date, client, pays, devise, statut, montant) via `export_request` (migration `0112`, Task 3) → job `export.data` ; journalisé.
- `print` : renvoie les URLs signées 5 min des documents de commande.
- `add_note` : n'ajoute **pas** de note interne (une note interne par commande est plus utile qu'une note groupée) : l'action est refusée avec `bulk_action_unsupported` et un message qui oriente vers l'action par commande.

Toute action groupée est **idempotente** : la clé d'idempotence est dérivée de `(actor, action, orderNumbers triés, date UTC du jour)`. Une double soumission ne produit pas deux exports.

- [ ] **Step 2.6: Notes internes sans table supplémentaire**

Les notes internes sont des écritures **append-only** dans `audit_log` avec `action = 'order.note.created'`, `object_type = 'order'`, `metadata = { body, locale }`. Ce choix :
- respecte l'immuabilité exigée par le spec §16.3 (« historique des changements immuable au niveau applicatif ») ;
- évite une table Phase 4 supplémentaire alors que le journal doit de toute façon être interrogé (Task 10) ;
- rend les notes impossibles à modifier par un rôle métier.

`InlineNote.tsx` rend une note avec un libellé visuel « Note interne — non visible par le client » (jamais assimilable à un message client, cf. design.md §3.4.5).

- [ ] **Step 2.7: Tests verts du service et des composants**

```bash
pnpm --filter @fanours/api test src/modules/admin-orders/admin-orders.service.spec.ts
pnpm --filter @fanours/ui test src/components/BulkActionBar.test.tsx src/components/InlineNote.test.tsx
```

Attendu :

```text
 ✓ src/modules/admin-orders/admin-orders.service.spec.ts (6 tests)
 ✓ src/components/BulkActionBar.test.tsx (3 tests)
 ✓ src/components/InlineNote.test.tsx (2 tests)
 Test Files  3 passed (3)
      Tests  11 passed (11)
```

`BulkActionBar` : apparaît dès qu'une ligne est sélectionnée, ancrée en bas de la zone liste, compteur accessible (`aria-live="polite"`), focus déplacé à l'apparition, `Escape` vide la sélection, boutons désactivés avec raison annoncée.

- [ ] **Step 2.8: Créer la coquille admin et le layout**

`apps/web/src/app/[locale]/(admin)/admin/layout.tsx` :

- `metadata` : `{ robots: { index: false, follow: false } }` (toutes les pages admin sont `noindex`).
- `AdminNav.tsx` : navigation par rôle (un onglet n'apparaît que si la permission serveur l'autorise), ordre : Vue globale · Commandes · Catalogue · Stock · Clients · Finance · Logistique · Douane · Contenu · Support · Journal · Analytics.
- `AdminPageHeader.tsx` : titre, fil d'Ariane, description contextuelle, slot d'actions.
- États globaux : `loading.tsx` (skeleton), `error.tsx` (message + action de sortie, sans détail technique).
- `AdminEmptyState.tsx` / `AdminErrorState.tsx` : icône + message localisé + action.
- Le layout appelle `requireAdminPage(ctx, 'order', 'read')` (dans `apps/web/src/server/admin.ts`) et renvoie la page « accès refusé » (403) ou redirige vers la connexion (401) — **jamais** de contenu admin dans le HTML d'un utilisateur non autorisé.

- [ ] **Step 2.9: Pages liste et détail**

`admin/commandes/page.tsx` : `DataTable` (Phase 3) + `FilterBar` (mobile = bottom sheet « Filtres (n) » avec réinitialisation), tri serveur, pagination par curseur, colonnes monétaires alignées à droite en `font-variant-numeric: tabular-nums`, badge de statut coloré ** accompanied d'un texte** (jamais la couleur seule).

`admin/commandes/[number]/page.tsx` : en-tête, `OrderStatusTimeline`, bloc actions (transitions autorisées pour le rôle courant, calculées côté serveur), bloc paiements, bloc expédition, bloc documents, bloc notes internes (`InlineNote`), bloc journal d'audit de la commande.

- [ ] **Step 2.10: Traduire et vérifier**

```bash
pnpm --filter @fanours/i18n check:keys
pnpm --filter @fanours/web build
```

Attendu :

```text
i18n key parity OK (3 locales, 468 clés)
✓ Built in 41.2s
```

**Preuves :**
- 6 tests unitaires du service (transition autorisée/refusée, scoping, pagination, action groupée idempotente).
- 5 tests composants (`BulkActionBar`, `InlineNote`).
- Les transitions disponibles sont calculées **côté serveur** à partir du rôle : la liste rendue pour `catalog_manager` ne contient que `cancel`/`mark_ready`, celle pour `order_manager` contient les transitions de gestion.
- Une transition refusée produit une entrée `audit_log` avec `result = 'denied'`.
- Toutes les pages `/admin/**` renvoient `noindex, nofollow` et sont exclues des sitemaps (`apps/web/src/lib/seo/private-routes.ts`).

**Commit :**

```bash
git add apps/api/src/modules/admin-orders apps/api/src/common/admin-scope.ts packages/ui/src/components/BulkActionBar.tsx packages/ui/src/components/InlineNote.tsx packages/ui/src/components/BulkActionBar.test.tsx packages/ui/src/components/InlineNote.test.tsx packages/ui/src/components/index.ts "apps/web/src/app/[locale]/(admin)/admin" apps/web/src/components/admin apps/web/src/lib/seo/private-routes.ts apps/web/src/server/admin.ts packages/i18n/src/locales
git commit -m "feat(admin-orders): back-office commandes, transitions et actions groupées" -m "Implémente le chantier 4.2 (spec §16.2-16.3) : vue globale par rôle, liste paginée et filtrée côté serveur, détail commande, transitions validées par allowedTransitions(status, role), actions groupées idempotentes et notes internes append-only dans audit_log. Introduit la coquille admin, BulkActionBar et InlineNote. Toutes les routes admin en noindex." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 3: Back-office catalogue (4.3) — import/export, variantes, paliers, workflow de publication

**Files:**
- Create: `packages/db/migrations/0112_phase4_catalog_ops.sql`
- Create: `e2e/admin/catalog-import.spec.ts` (recette fonctionnelle de phase — projet Playwright `functional`)
- Create: `packages/db/src/schema/phase4-catalog-ops.ts`
- Modify: `packages/db/src/schema.ts`
- Create: `apps/api/src/modules/admin-catalog/admin-catalog.service.ts`
- Create: `apps/api/src/modules/admin-catalog/admin-catalog.controller.ts`
- Create: `apps/api/src/modules/admin-catalog/admin-catalog.module.ts`
- Create: `apps/api/src/modules/admin-catalog/catalog-import.service.ts`
- Create: `apps/api/src/modules/admin-catalog/catalog-parser.ts`
- Test: `apps/api/src/modules/admin-catalog/catalog-import.service.spec.ts`
- Create: `apps/worker/src/jobs/catalog/catalog-import.job.ts`
- Create: `apps/web/src/app/[locale]/(admin)/admin/catalogue/page.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/catalogue/import/page.tsx`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `product` / `product_variant` / `price_tier` / `product_translation` / `product_media` (Phase 2), `country_profile` (Phase 3), `admin_action_approval` (Phase 1), `audit_log` (Phase 1), `packages/domain` → `PERMISSIONS` (ressources `product`, `price_tier`, `export`).
- Produces:
  - `AdminCatalogService.list(ctx: AdminContext, query: AdminCatalogQueryDto): Promise<Paged<AdminProductRowDto>>`
  - `AdminCatalogService.get(ctx: AdminContext, productId: string): Promise<AdminProductDetailDto>`
  - `AdminCatalogService.create(ctx: AdminContext, input: AdminProductInputDto): Promise<AdminProductDetailDto>`
  - `AdminCatalogService.update(ctx: AdminContext, productId: string, input: AdminProductInputDto): Promise<AdminProductDetailDto>`
  - `AdminCatalogService.setVariants(ctx: AdminContext, productId: string, variants: AdminVariantInputDto[]): Promise<AdminProductDetailDto>`
  - `AdminCatalogService.setMoq(ctx: AdminContext, productId: string, input: MoqInputDto): Promise<AdminProductDetailDto>` — `moq > 0`, `order_multiple > 0`.
  - `AdminCatalogService.setTiers(ctx: AdminContext, productId: string, tiers: AdminTierInputDto[]): Promise<AdminProductDetailDto>` — passe par le workflow quatre-yeux `price_tier_change`.
  - `AdminCatalogService.setCompliance(ctx: AdminContext, productId: string, input: ComplianceInputDto): Promise<AdminProductDetailDto>` — HS code, niveau de certitude, source, date de consultation ; **jamais validé par IA seule**.
  - `AdminCatalogService.transition(ctx: AdminContext, productId: string, input: ProductTransitionInputDto): Promise<AdminProductDetailDto>` — `draft → in_review → published → suspended → archived`.
  - `CatalogImportService.start(ctx: AdminContext, file: CatalogImportFileDto, options: CatalogImportOptionsDto): Promise<{ importId: string }>`
  - `CatalogImportService.report(ctx: AdminContext, importId: string): Promise<CatalogImportReportDto>`
  - `CatalogImportService.cancel(ctx: AdminContext, importId: string): Promise<void>`
  - `CatalogExportService.start(ctx: AdminContext, query: AdminCatalogQueryDto): Promise<{ exportId: string }>`
  - `parseCatalogCsv(content: string): ParsedCatalog`
  - `parseCatalogXlsx(buffer: Buffer): Promise<ParsedCatalog>`
  - `validateCatalogRow(row: ParsedCatalogRow, context: CatalogValidationContext): CatalogRowIssue[]`
  - Routes : `GET|POST /v1/admin/catalog/products`, `GET|PATCH /v1/admin/catalog/products/:id`, `PUT /v1/admin/catalog/products/:id/variants`, `PUT /v1/admin/catalog/products/:id/moq`, `PUT /v1/admin/catalog/products/:id/tiers`, `PUT /v1/admin/catalog/products/:id/compliance`, `POST /v1/admin/catalog/products/:id/transitions`, `POST /v1/admin/catalog/imports`, `GET /v1/admin/catalog/imports/:id`, `POST /v1/admin/catalog/imports/:id/cancel`, `POST /v1/admin/catalog/exports`.

- [ ] **Step 3.1: Écrire le test rouge du rapport de validation d'import**

`apps/api/src/modules/admin-catalog/catalog-import.service.spec.ts` :

```typescript
describe('CatalogImportService', () => {
  it('rejette les lignes invalides avec un code de champ et conserve les lignes valides', async () => {
    const csv = [
      'sku,title_fr,moq,order_multiple,weight_g,hs_code,tier1_min,tier1_price',
      'SKU-001,Poignee en bois,100,10,250,7321.90,100,1.20',
      'SKU-002,,0,0,,7321.90,10,1.10',
      'SKU-003,Chaise,50,5,3200,9401.90,50,9.00',
    ].join('\n');

    const { importId } = await service.start(ctx, { filename: 'catalogue.csv', content: csv }, { dryRun: true });
    const report = await service.report(ctx, importId);

    expect(report.totalRows).toBe(3);
    expect(report.errorRows).toBe(1);
    expect(report.rows[1]).toMatchObject({ lineNumber: 3, sku: 'SKU-002', level: 'error' });
    expect(report.rows[1].issues[0]).toMatchObject({ field: 'moq', code: 'must_be_positive' });
    expect(report.rows[2].issues.map((i) => i.field)).toEqual(expect.arrayContaining(['title_fr', 'weight_g']));
  });

  it('refuse la publication d’un produit dont les traductions sont incomplètes', async () => {
    await expect(service.transition(ctx, productId, { to: 'published' })).rejects.toMatchObject({
      status: 422,
      code: 'product_not_publishable',
    });
  });
});
```

- [ ] **Step 3.1b: Écrire le test E2E rouge de la recette fonctionnelle import/export catalogue**

`e2e/admin/catalog-import.spec.ts` — recette fonctionnelle de phase, projet Playwright `functional`. Écrit **avant** l'implémentation : il échoue à ce stade et passe à l'étape « Traduire et vérifier ».

Cas et assertions :

1. Import d'un CSV valide en `dryRun` (par défaut) : le rapport de validation est rendu et **0** ligne bloquante ; aucun produit n'est publié.
2. Import d'un CSV contenant 3 lignes invalides : le rapport affiche pour chacune le niveau, le champ et le message ; seules les premières lignes sont listées (jamais les 5 000).
3. Un rôle sans la permission `catalog:write` ne peut pas lancer d'import : `403`, message localisé, aucun contenu admin divulgué, action tracée dans le journal d'audit.
4. L'export catalogue est asynchrone (`export_request` + job `export.data`) et servi par une **URL signée de 5 minutes** dans le bucket privé ; aucune URL publique n'est émise et l'URL signée n'apparaît dans aucun log.

- [ ] **Step 3.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/api test src/modules/admin-catalog/catalog-import.service.spec.ts
```

Attendu :

```text
 FAIL  src/modules/admin-catalog/catalog-import.service.spec.ts
Error: Cannot find module './catalog-import.service.js'
Test Files  1 failed (1)
     Tests  0 passed | 2 failed (2)
```

- [ ] **Step 3.3: Écrire la migration `0112`**

`packages/db/migrations/0112_phase4_catalog_ops.sql` :

```sql
-- 0112_phase4_catalog_ops.sql
-- Phase 4 — Task 3 : imports/exports de données, plateforme d'export partagée.
-- `export_request` est réutilisé par les Tasks 6 (export financier) et 10 (export d'audit).
SET lock_timeout = '5s';
SET statement_timeout = '30s';

CREATE TABLE data_import (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  kind            text NOT NULL DEFAULT 'catalog',
  filename        text NOT NULL,
  storage_key     text,
  status          text NOT NULL DEFAULT 'pending',
  total_rows      int NOT NULL DEFAULT 0,
  processed_rows  int NOT NULL DEFAULT 0,
  error_rows      int NOT NULL DEFAULT 0,
  idempotency_key text NOT NULL,
  requested_by    uuid NOT NULL REFERENCES "user"(id),
  correlation_id  uuid NOT NULL,
  created_at      timestamptz NOT NULL DEFAULT now(),
  finished_at     timestamptz,
  CONSTRAINT data_import_status_check CHECK (status IN ('pending', 'running', 'succeeded', 'failed', 'cancelled')),
  CONSTRAINT data_import_kind_check CHECK (kind IN ('catalog')),
  CONSTRAINT data_import_idempotency_unique UNIQUE (idempotency_key)
);
CREATE INDEX idx_data_import_status ON data_import(status, created_at DESC);

CREATE TABLE data_import_row (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  import_id   uuid NOT NULL REFERENCES data_import(id) ON DELETE CASCADE,
  line_number int NOT NULL,
  sku         text,
  level       text NOT NULL DEFAULT 'error',
  message     text NOT NULL,
  issues      jsonb NOT NULL DEFAULT '[]'::jsonb,
  payload     jsonb NOT NULL DEFAULT '{}'::jsonb,
  CONSTRAINT data_import_row_level_check CHECK (level IN ('info', 'warning', 'error')),
  CONSTRAINT data_import_row_unique UNIQUE (import_id, line_number)
);
CREATE INDEX idx_data_import_row_import ON data_import_row(import_id, level);

CREATE TABLE export_request (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  kind            text NOT NULL,
  requested_by    uuid NOT NULL REFERENCES "user"(id),
  idempotency_key text NOT NULL,
  status          text NOT NULL DEFAULT 'pending',
  parameters      jsonb NOT NULL DEFAULT '{}'::jsonb,
  storage_key     text,
  row_count       int,
  expires_at      timestamptz,
  correlation_id  uuid NOT NULL,
  created_at      timestamptz NOT NULL DEFAULT now(),
  completed_at    timestamptz,
  CONSTRAINT export_request_status_check CHECK (status IN ('pending', 'running', 'ready', 'failed', 'expired')),
  CONSTRAINT export_request_kind_check CHECK (kind IN ('catalog', 'orders', 'financial', 'audit')),
  CONSTRAINT export_request_idempotency_unique UNIQUE (idempotency_key)
);
CREATE INDEX idx_export_request_status ON export_request(status, created_at DESC);
```

- [ ] **Step 3.4: Appliquer la migration et écrire le schéma Drizzle**

```bash
pnpm --filter @fanours/db migrate
```

Attendu :

```text
migrating 0112_phase4_catalog_ops.sql
0112_phase4_catalog_ops.sql applied (checksum 3d84…)
```

`packages/db/src/schema/phase4-catalog-ops.ts` : `dataImport`, `dataImportRow`, `exportRequest`.

- [ ] **Step 3.5: Implémenter le parseur et le validateur**

`catalog-parser.ts` (pur, testable, sans dépendance base) :

- CSV : RFC 4180 via `csv-parse/sync` (ajout à `apps/api/package.json`), séparateur détecté `,` ou `;`, encodage UTF-8, BOM toléré.
- XLSX : `exceljs` (ajout à `apps/api/package.json`), première feuille, en-têtes normalisés (minuscules, sans accents, `_` à la place des espaces).
- Colonnes reconnues : `sku`, `title_fr`, `title_en`, `title_zh`, `description_fr`, `moq`, `order_multiple`, `weight_g`, `length_mm`, `width_mm`, `height_mm`, `hs_code`, `country_of_origin`, `lead_time_days`, `tier{n}_min`, `tier{n}_price`, `tier{n}_currency`, `image_url`.

`validateCatalogRow()` renvoie des issues typées :

| Champ | Code d'erreur | Règle |
|---|---|---|
| `sku` | `required`, `duplicate_in_file` | identifiant stable, unique dans le fichier |
| `title_fr` / `title_en` / `title_zh` | `required`, `missing_locale` | les 3 locales sont exigées à la publication (pas à l'import) |
| `moq` | `must_be_positive` | `> 0` |
| `order_multiple` | `must_be_positive` | `> 0` |
| `weight_g` | `must_be_positive` | `> 0` |
| `hs_code` | `invalid_format` | `^\d{4}\.\d{2}$` (format HS - code de 6 chiffres) |
| `country_of_origin` | `unknown_country` | doit exister dans `country_profile` et ne pas être suspendu |
| `tier{n}_min` / `tier{n}_price` | `must_be_positive`, `overlapping_tiers`, `not_monotonic` | paliers contigus, prix décroissants, aucun chevauchement (contrainte `price_tier_no_overlap`) |
| `tier{n}_currency` | `unsupported_currency` | code ISO 4217 de la liste de devises supportées |

- [ ] **Step 3.6: Implémenter le service d'import**

`CatalogImportService.start()` :

1. Vérifie la permission `product:create` (ou `product:update` si `mode = 'update'`).
2. Calcule la clé d'idempotence `catalog.import:<actorId>:<sha256(file)>` → une double soumission du même fichier renvoie le même `importId` au lieu de dupliquer.
3. Stocke le fichier source dans le bucket privé (clé `imports/<importId>/<filename>`), jamais dans un répertoire public.
4. Crée la ligne `data_import` (`status = 'pending'`) et publie un événement outbox `catalog.import.requested`.

`apps/worker/src/jobs/catalog/catalog-import.job.ts` : job `catalog.import` (infra §11.5, type n° 8 « import catalogue »), `idempotencyKey = catalog.import:<importId>`, retry backoff exponentiel + jitter, DLQ. Le job :

- parse le fichier ;
- écrit une ligne `data_import_row` par ligne source avec ses issues ;
- en `dryRun` : n'écrit rien en catalogue, `status = 'succeeded'` avec le rapport ;
- sinon : applique les lignes valides en une transaction par lot de 100, **sans** modifier les produits dont la ligne est en erreur ;
- met à jour `data_import` (`total_rows`, `processed_rows`, `error_rows`, `finished_at`).

`CatalogImportService.cancel()` : positionne `status = 'cancelled'` ; le job vérifie ce statut entre deux lots et s'arrête (annulation possible, cf. infra §11.5).

- [ ] **Step 3.7: Implémenter le workflow produit, les variantes, le MOQ et les paliers**

`AdminCatalogService.transition()` contrôle les transitions autorisées :

- `draft → in_review` : toujours autorisé à `catalog_manager`.
- `in_review → published` : refusé si une des 3 traductions manque (`product_not_publishable`, HTTP 422), si aucun palier de prix n'est défini, si le MOQ ou le multiple de commande est invalide, si le code HS est absent ou si son niveau de certitude est `incomplet` (`compliance_review_required`).
- `published → suspended` : réservé à `catalog_manager` **et** `compliance_manager` (deux rôles requis sur cette action sensible) ; motif obligatoire ; journalisé.
- `archived` : suppression logique, jamais de `DELETE` SQL sur un produit publié.

`setTiers()` : les paliers sont **écrits dans `price_tier` avec `valid_from`**, jamais remplacés par écrasement ; la modification déclenche le workflow quatre-yeux `admin_action_approval` (kind `price_tier_change`) : le `catalog_manager` **demande**, un second rôle (`finance_manager` ou `super_admin`) **approuve**, l'exécution est ensuite idempotente et journalisée.

`setCompliance()` : `hs_code` + `certainty_level` (`validated` | `estimated` | `incomplete` | `specialist`) + `source_url` + `source_consulted_at` sont **obligatoires** ; l'API refuse un code HS sans source officielle et sans date de consultation ; un niveau `estimated` ou `incomplet` bloque la publication.

- [ ] **Step 3.8: Implémenter l'export catalogue**

`CatalogExportService.start()` crée une ligne `export_request` (`kind = 'catalog'`, `idempotency_key = catalog.export:<actorId>:<sha256(query)>`), publie l'outbox `export.requested` et renvoie `{ exportId }`. Le job `export.data` (Task 6, fichier `apps/worker/src/jobs/export/export-data.job.ts`) écrit le CSV dans le bucket privé, renseigne `storage_key`, `row_count`, `expires_at` (`now() + 24 h` — durée de conservation du fichier exporté, `à valider (décision propriétaire)`) et sert le fichier via une **URL signée de 5 minutes**.

- [ ] **Step 3.9: Tests verts**

```bash
pnpm --filter @fanours/api test src/modules/admin-catalog
```

Attendu :

```text
 ✓ src/modules/admin-catalog/catalog-import.service.spec.ts (5 tests)
 ✓ src/modules/admin-catalog/catalog-parser.test.ts (7 tests)
 Test Files  2 passed (2)
      Tests  12 passed (12)
```

- [ ] **Step 3.10: Pages admin catalogue et import**

`admin/catalogue/page.tsx` : `DataTable` (SKU, titre, MOQ, paliers, statut, pays, actions), `FilterBar` (statut, catégorie, pays, recherche SKU), panneau latéral de détail produit (onglets : Général, Variantes, Paliers, Traductions, Conformité, Médias). Les prix affichés viennent de l'API ; le front n'applique aucun taux.

`admin/catalogue/import/page.tsx` : dépôt de fichier (CSV/XLSX), options (`dryRun` par défaut **activé**), rapport de validation affiché en `DataTable` (ligne, SKU, niveau, champ, message) avec compteur, bouton « Annuler l'import » actif tant que le job tourne, message d'erreur listant les premières lignes invalides (jamais les 5 000).

- [ ] **Step 3.11: Traduire et vérifier**

```bash
pnpm --filter @fanours/i18n check:keys
pnpm --filter @fanours/web build
pnpm exec playwright test e2e/admin/catalog-import.spec.ts --project=functional
```

Attendu :

```text
i18n key parity OK (3 locales, 512 clés)
✓ Built in 44.8s
  2 passed (9.1s)
```

**Preuves :**
- 12 tests unitaires (parseur CSV/XLSX, validation des lignes, rapport, blocage de publication).
- `0112` appliquée avec checksum ; `data_import`, `data_import_row`, `export_request` créées.
- Une modification de palier **sans** second approbateur est refusée (`approval_required`) et une auto-approbation est refusée (`no_self_approval`).
- Un import en `dryRun` n'écrit aucune ligne en `product` (vérifié par comptage avant/après).
- Le job `catalog.import` est idempotent : deux exécutions avec la même clé ne traitent le fichier qu'une fois.

**Commit :**

```bash
git add packages/db/migrations/0112_phase4_catalog_ops.sql packages/db/src/schema/phase4-catalog-ops.ts packages/db/src/schema.ts apps/api/src/modules/admin-catalog apps/worker/src/jobs/catalog apps/api/package.json "apps/web/src/app/[locale]/(admin)/admin/catalogue" packages/i18n/src/locales
git commit -m "feat(admin-catalog): import export catalogue, variantes, paliers et publication" -m "Implémente le chantier 4.3 (spec §16.4, §7.4) : import CSV/XLSX avec rapport de validation par ligne et annulation, export catalogue sur la plateforme export_request, gestion des variantes, du MOQ et des paliers non chevauchants, workflow draft/in_review/published avec code HS, niveau de certitude et source obligatoire. Modification de palier soumise au quatre-yeux." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 4: Back-office stock (4.4) — mouvements, seuils, inventaire, alertes

**Files:**
- Create: `packages/db/migrations/0113_phase4_stock.sql`
- Create: `e2e/admin/stock.spec.ts` (recette fonctionnelle de phase — projet Playwright `functional`)
- Create: `packages/db/src/schema/phase4-stock.ts`
- Modify: `packages/db/src/schema.ts`
- Modify: `packages/domain/src/permissions.ts`
- Modify: `packages/db/src/seed.ts`
- Modify: `apps/api/src/modules/rbac/__tests__/permissions.test.ts` (assertion de comptage `RESOURCES` / `ACTIONS` écrite en Phase 1)
- Modify: `packages/db/src/__tests__/migrations.integration.test.ts` (compte de migrations figé depuis la Phase 1)
- Create: `apps/api/src/modules/admin-stock/admin-stock.service.ts`
- Create: `apps/api/src/modules/admin-stock/admin-stock.controller.ts`
- Create: `apps/api/src/modules/admin-stock/admin-stock.module.ts`
- Test: `apps/api/src/modules/admin-stock/admin-stock.service.spec.ts`
- Create: `apps/worker/src/jobs/stock/stock-alert.job.ts`
- Create: `apps/web/src/app/[locale]/(admin)/admin/stock/page.tsx`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `inventory_item` / `inventory_movement` / `stock_reservation` / `warehouse` (Phase 2), `product` / `product_variant` (Phase 2), `audit_log` (Phase 1), `outbox_event` (Phase 2).
- Produces:
  - `AdminStockService.list(ctx: AdminContext, query: AdminStockQueryDto): Promise<Paged<AdminStockRowDto>>`
  - `AdminStockService.get(ctx: AdminContext, itemId: string): Promise<AdminStockDetailDto>`
  - `AdminStockService.movement(ctx: AdminContext, input: StockMovementInputDto): Promise<StockMovementResultDto>`
  - `AdminStockService.setThresholds(ctx: AdminContext, input: StockThresholdInputDto): Promise<AdminStockRowDto>`
  - `AdminStockService.startCount(ctx: AdminContext, input: StockCountStartDto): Promise<AdminStockCountDto>`
  - `AdminStockService.recordCount(ctx: AdminContext, countId: string, lines: StockCountLineInputDto[]): Promise<AdminStockCountDto>`
  - `AdminStockService.closeCount(ctx: AdminContext, countId: string): Promise<AdminStockCountDto>`
  - `AdminStockService.reconcileCount(ctx: AdminContext, countId: string): Promise<AdminStockCountDto>`
  - `AdminStockService.alerts(ctx: AdminContext, query: AdminStockAlertQueryDto): Promise<Paged<AdminStockAlertDto>>`
  - `AdminStockService.reservations(ctx: AdminContext, query: AdminReservationQueryDto): Promise<Paged<AdminReservationDto>>`
  - `BULK_ADJUSTMENT_LIMIT` (constante de politique, `packages/config/src/admin-policy.ts`, valeur par défaut documentée comme `à valider (décision propriétaire)`).
  - Routes : `GET /v1/admin/stock`, `GET /v1/admin/stock/:itemId`, `POST /v1/admin/stock/movements`, `PUT /v1/admin/stock/thresholds`, `POST /v1/admin/stock/counts`, `PUT /v1/admin/stock/counts/:id`, `POST /v1/admin/stock/counts/:id/close`, `POST /v1/admin/stock/counts/:id/reconcile`, `GET /v1/admin/stock/alerts`, `GET /v1/admin/stock/reservations`.

- [ ] **Step 4.1: Écrire le test rouge du blocage de correction massive**

`apps/api/src/modules/admin-stock/admin-stock.service.spec.ts` :

```typescript
describe('AdminStockService.movement', () => {
  it('refuse une correction massive au-delà de la limite de politique', async () => {
    await expect(
      service.movement(ctxAsRole('logistics_operator'), {
        kind: 'adjust',
        itemId,
        qty: -BULK_ADJUSTMENT_LIMIT - 1,
        reason: 'Correction après inventaire',
      }),
    ).rejects.toMatchObject({ status: 403, code: 'bulk_adjustment_forbidden' });
  });

  it('autorise une correction massive avec la permission stock:update et une MFA récente', async () => {
    const result = await service.movement(ctxAsRole('logistics_operator', { mfaVerifiedAt: recentMfa() }), {
      kind: 'adjust',
      itemId,
      qty: -BULK_ADJUSTMENT_LIMIT - 1,
      reason: 'Correction après inventaire',
    });
    expect(result.item.onHand).toBe(120);
  });

  it('motive obligatoirement tout mouvement et journalise le stock', async () => {
    await expect(
      service.movement(ctxAsRole('logistics_operator'), { kind: 'adjust', itemId, qty: -1, reason: '' }),
    ).rejects.toMatchObject({ status: 422, code: 'reason_required' });
  });
});
```

- [ ] **Step 4.1b: Écrire le test E2E rouge de la recette fonctionnelle du back-office stock**

`e2e/admin/stock.spec.ts` — recette fonctionnelle de phase, projet Playwright `functional`. Écrit **avant** l'implémentation : il échoue à ce stade et passe à l'étape « Traduire et vérifier ».

Cas et assertions :

1. Le tableau des stocks affiche SKU, quantité disponible, seuil et date du dernier mouvement ; les quantités sont des entiers de minor units, jamais des flottants.
2. Un mouvement invalide (quantité nulle ou négative) est refusé par l'API (`422`, code stable + message localisé) et l'échec est tracé dans le journal d'audit.
3. Le passage sous le seuil déclenche un mouvement et une alerte visible ; l'inventaire physique se lance, se clôture, et l'écart restant est **tracé et non résolu** (jamais écrasé).
4. Un rôle sans la permission `stock:write` ne peut ni créer un mouvement ni clôturer un inventaire : `403` et action tracée.

- [ ] **Step 4.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/api test src/modules/admin-stock/admin-stock.service.spec.ts
```

Attendu :

```text
 FAIL  src/modules/admin-stock/admin-stock.service.spec.ts
Error: Cannot find module './admin-stock.service.js'
Test Files  1 failed (1)
     Tests  0 passed | 3 failed (3)
```

- [ ] **Step 4.3: Écrire la migration `0113`**

`packages/db/migrations/0113_phase4_stock.sql` :

```sql
-- 0113_phase4_stock.sql
-- Phase 4 — Task 4 : seuils de réassort, inventaire physique, extension de la matrice RBAC.
-- 0113 est aussi le fichier qui autorise les 5 ressources de la phase 4 : ce sont les
-- premières tâches qui en ont besoin (Task 4 elle-même, puis Tasks 8, 9, 11 et 12).
SET lock_timeout = '5s';
SET statement_timeout = '30s';

ALTER TABLE inventory_item ADD COLUMN reorder_point int NOT NULL DEFAULT 0;
ALTER TABLE inventory_item ADD COLUMN reorder_qty  int NOT NULL DEFAULT 0;
ALTER TABLE inventory_item
  ADD CONSTRAINT inventory_item_reorder_check CHECK (reorder_point >= 0 AND reorder_qty >= 0);

CREATE TABLE stock_count (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  warehouse_id uuid NOT NULL REFERENCES warehouse(id),
  status       text NOT NULL DEFAULT 'open',
  scope        text NOT NULL DEFAULT 'all',
  started_by   uuid NOT NULL REFERENCES "user"(id),
  started_at   timestamptz NOT NULL DEFAULT now(),
  closed_at    timestamptz,
  closed_by    uuid REFERENCES "user"(id),
  reconciled_at timestamptz,
  reconciled_by uuid REFERENCES "user"(id),
  note         text,
  CONSTRAINT stock_count_status_check CHECK (status IN ('open', 'closed', 'reconciled')),
  CONSTRAINT stock_count_scope_check CHECK (scope IN ('all', 'selection'))
);
CREATE INDEX idx_stock_count_warehouse ON stock_count(warehouse_id, started_at DESC);

CREATE TABLE stock_count_line (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  count_id     uuid NOT NULL REFERENCES stock_count(id) ON DELETE CASCADE,
  item_id      uuid NOT NULL REFERENCES inventory_item(id),
  expected_qty int NOT NULL,
  counted_qty  int,
  variance     int,
  status       text NOT NULL DEFAULT 'pending',
  reconciled_at timestamptz,
  CONSTRAINT stock_count_line_status_check CHECK (status IN ('pending', 'counted', 'reconciled')),
  CONSTRAINT stock_count_line_unique UNIQUE (count_id, item_id),
  CONSTRAINT stock_count_line_variance_check CHECK (variance IS NULL OR counted_qty IS NOT NULL)
);
CREATE INDEX idx_stock_count_line_count ON stock_count_line(count_id, status);

-- Matrice RBAC : ajout des ressources de la phase 4 (stock, content_page, support_ticket,
-- analytics, notification). Constraint CHECK remplacée : aucune donnée n'est supprimée.
-- La contrainte est étendue PAR ANTICIPATION aux 5 ressources et aux 5 actions de la phase 4
-- (publish, assign, escalate, refund_request, send) : 0113 est la seule migration de phase 4 qui
-- touche la matrice RBAC, et elle est appliquée avant les Tasks 8, 9, 11 et 12 qui semeront
-- les lignes `permission` correspondantes. Une ressource ou une action non listée reste
-- refusée par la base, comme avant.
ALTER TABLE permission DROP CONSTRAINT IF EXISTS ck_permission_resource;
ALTER TABLE permission ADD CONSTRAINT ck_permission_resource CHECK (
  resource IN (
    'order', 'product', 'price_tier', 'export', 'export_financial', 'refund', 'admin_role',
    'audit_log', 'customs_rule', 'country', 'business_document', 'business_account',
    'customs_document',
    'stock', 'content_page', 'support_ticket', 'analytics', 'notification'
  )
);
ALTER TABLE permission DROP CONSTRAINT IF EXISTS ck_permission_action;
ALTER TABLE permission ADD CONSTRAINT ck_permission_action CHECK (
  action IN (
    'read', 'create', 'update', 'delete', 'export', 'approve', 'refund', 'manage_roles',
    'publish', 'assign', 'escalate', 'refund_request', 'send'
  )
);
```

- [ ] **Step 4.4: Appliquer la migration et écrire le schéma Drizzle**

```bash
pnpm --filter @fanours/db migrate
```

Attendu :

```text
migrating 0113_phase4_stock.sql
0113_phase4_stock.sql applied (checksum 7c02…)
```

- [ ] **Step 4.5: Mettre à jour le test de permissions de la Phase 1 (comptage des ressources et des actions)**

`apps/api/src/modules/rbac/__tests__/permissions.test.ts` — la Phase 1 figeait `expect(RESOURCES).toHaveLength(13)` et `expect(ACTIONS).toHaveLength(8)`. Ces deux assertions **cassent** dès que la phase 4 ajoute une ressource ou une action, et personne ne l'avait prévu. Elles sont remplacées par une comparaison à des **listes attendues explicites** : la lecture reste simple (le diff montre exactement la ressource ou l'action autorisée) et le compte codé en dur, à remettre à jour à chaque phase, disparaît.

```typescript
// Listes attendues : la phase 4 ajoute les ressources et actions ci-dessous.
// Toute évolution de la matrice doit modifier ce fichier dans le MÊME commit que
// `packages/domain/src/permissions.ts`.
//   Ressources : 13 (phase 1) + stock (T4) + content_page (T8) + support_ticket (T9)
//                + analytics (T11) + notification (T12) = 18
//   Actions    : 8 (phase 1) + publish (T8) + assign, escalate, refund_request (T9)
//                + send (T12) = 13
// Le vocabulaire d'actions est déclaré en entier dès la Task 4 (cf. Step 4.7) parce que
// `ck_permission_action` n'est réécrite que par la migration `0113` : les 5 emplacements
// `0111`–`0115` de la phase 4 sont clos et aucune 6ᵉ migration n'est disponible.
const EXPECTED_RESOURCES = [
  'order', 'product', 'price_tier', 'export', 'export_financial', 'refund', 'admin_role',
  'audit_log', 'customs_rule', 'country', 'business_document', 'business_account',
  'customs_document', 'stock',
] as const;

const EXPECTED_ACTIONS = [
  'read', 'create', 'update', 'delete', 'export', 'approve', 'refund', 'manage_roles',
  'publish', 'assign', 'escalate', 'refund_request', 'send',
] as const;

describe('matrice RBAC', () => {
  it('expose 9 roles et exactement les ressources et actions attendues', () => {
    expect(ROLES).toHaveLength(9);
    expect([...RESOURCES].sort()).toEqual([...EXPECTED_RESOURCES].sort());
    expect([...ACTIONS].sort()).toEqual([...EXPECTED_ACTIONS].sort());
  });

  // Les autres tests de la Phase 1 restent inchangés et doivent rester verts :
  //  - « donne au moins une permission a chaque role »           : toujours vrai ;
  //  - « ne declare que des ressources et des actions connues »  : RESOURCES et ACTIONS
  //    sont étendus ci-dessus, donc toute clé de PERMISSIONS y figure ;
  //  - « refuse par defaut un role inconnu ou une action inconnue » : inchangé ;
  //  - « interdit au support le remboursement et au catalogue la gestion des roles » :
  //    PERMISSIONS['support_agent']?.['refund'] doit rester ['read', 'create'] — la phase 4
  //    n'ajoute QUE la ressource `support_ticket` au support_agent et ne touche pas `refund` ;
  //  - « mappe chaque action four-eyes vers un type d'approbation » : FOUR_EYES reste
  //    inchangé (8 couples de la sécurité §3.2), donc approvalKindFor n'est pas étendu.
});
```

```bash
pnpm --filter @fanours/api test src/modules/rbac/__tests__/permissions.test.ts
```

Attendu :

```text
 ✓ src/modules/rbac/__tests__/permissions.test.ts (6 tests)
 Test Files  1 passed (1)
      Tests  6 passed (6)
```

- [ ] **Step 4.6: Défiger le comptage de migrations du test d'intégration de la Phase 1**

`packages/db/src/__tests__/migrations.integration.test.ts` — la Phase 1 figeait le nombre de migrations en dur : `expect(first.length).toBe(4)` dans le test « applique toutes les migrations une seule fois » (puis porté à `5` par la Task 2 de la Phase 1, quand `rbac_scope` a été ajouté) et `expect(rows.rows).toHaveLength(4)` dans le test « refuse une migration modifiee apres application » — **ce dernier n'a jamais été mis à jour**. Ces comptes sont **déjà faux** depuis la Phase 2 (15 fichiers) puis la Phase 3 (26 fichiers) : les plans de ces phases ne listaient pas ce fichier, la régression dormait et aurait cassé la suite d'intégration de la base. La phase 4 le corrige en comparant au nombre réel de fichiers du dossier — la valeur devient **31** à la fin de la phase 4 (5 migrations phase 1 + 10 phase 2 + 11 phase 3 + 5 phase 4) et ne cassera plus aux phases suivantes.

```typescript
const files = (await readdir(migrationsDir)).filter((name) => name.endsWith('.sql')).sort();

it('applique toutes les migrations une seule fois', async () => {
  const first = await migrate({ directory: migrationsDir, client: pool });
  expect(first.every((entry) => entry.status === 'applied')).toBe(true);
  expect(first.length).toBe(files.length);

  const second = await migrate({ directory: migrationsDir, client: pool });
  expect(second.every((entry) => entry.status === 'skipped')).toBe(true);
});

it('refuse une migration modifiee apres application', async () => {
  expect(files[0]).toBe('0001_extensions_and_migration_table.sql');
  const rows = await pool.query('SELECT version, checksum FROM schema_migrations ORDER BY version');
  expect(rows.rows).toHaveLength(files.length);
});
```

Le `readdir` est **déplacé au niveau du module** (au lieu d'être fait dans le deuxième test) : les deux tests ont besoin de la même liste, et la lecture est faite une seule fois.

Le test « cree les tables attendues » reste valide : il vérifie la **présence** des tables attendues (`expect(names).toContain(expected)`) et pas leur nombre, donc les 14 tables de la phase 4 (`address`, `user_preference`, `push_subscription`, `data_import`, `data_import_row`, `export_request`, `stock_count`, `stock_count_line`, `payment_reconciliation`, `payment_reconciliation_line`, `content_page`, `content_page_translation`, `redirect_rule`, `sale_condition`) ne le cassent pas. Aucune assertion d'**absence** n'est ajoutée : les tables des phases futures sont autorisées.

```bash
pnpm --filter @fanours/db test src/__tests__/migrations.integration.test.ts
```

Attendu :

```text
 ✓ src/__tests__/migrations.integration.test.ts (3 tests)
 Test Files  1 passed (1)
      Tests  3 passed (3)
```

- [ ] **Step 4.7: Étendre la matrice de permissions**

`packages/domain/src/permissions.ts` — ajout de la ressource `stock` (et de son vocabulaire d'actions, par anticipation) :

```typescript
export const RESOURCES = [
  'order', 'product', 'price_tier', 'export', 'export_financial', 'refund', 'admin_role',
  'audit_log', 'customs_rule', 'country', 'business_document', 'business_account',
  'customs_document', 'stock',
] as const;

export const ACTIONS = [
  'read', 'create', 'update', 'delete', 'export', 'approve', 'refund', 'manage_roles',
  'publish', 'assign', 'escalate', 'refund_request', 'send',
] as const;
```

**Décompte à ce stade : 14 ressources (13 phase 1 + `stock`) et 13 actions (8 phase 1 + `publish`, `assign`, `escalate`, `refund_request`, `send`).**

Les 5 actions sont déclarées ici **par anticipation**, exactement comme la contrainte SQL `ck_permission_action` recréée par `0113` (cf. Step 4.3) : `publish` n'est utilisée qu'à partir de la Task 8 (`content_page`), `assign`, `escalate` et `refund_request` qu'à partir de la Task 9 (`support_ticket`), `send` qu'à partir de la Task 12 (`notification`). Les lignes `permission` correspondantes ne sont semées que dans ces tâches ; aucune permission n'est donc accordée par anticipation, seul le **vocabulaire** est élargi. C'est la seule façon de tenir dans les 5 emplacements de migration `0111`–`0115` : `ck_permission_action` n'est réécrite que par `0113`.

`PERMISSIONS` reçoit `stock: ['read', 'create', 'update', 'delete']`, avec :
- `logistics_operator` : `stock: ['read', 'create', 'update']` ;
- `catalog_manager` : `stock: ['read']` ;
- `super_admin` : `stock: ['read', 'create', 'update', 'delete']`.

`packages/db/src/seed.ts` ajoute les lignes `permission` correspondantes (la table est peuplée par le seed idempotent, jamais par une migration).

Les ressources `content_page` (T8), `support_ticket` (T9), `analytics` (T11) et `notification` (T12) sont ajoutées à `RESOURCES` et à `EXPECTED_RESOURCES` dans leur propre tâche, qui met à jour `apps/api/src/modules/rbac/__tests__/permissions.test.ts` au même commit.

- [ ] **Step 4.8: Implémenter les mouvements de stock**

`StockMovementInputDto` : `{ kind: 'in' | 'out' | 'adjust' | 'transfer', itemId, qty, reason, targetItemId?, orderId? }`.

Règles serveur :

- `qty !== 0` (contrainte `inventory_movement_qty_check`) ; `reason` non vide (min 10 caractères) ;
- `adjust` : `on_hand` ne peut pas devenir négatif, `reserved` doit rester ≤ `on_hand` (contraintes `inventory_item_on_hand_check`, `inventory_item_reserved_check`) ;
- `transfer` : décrémente l'origine et incrémente la destination **dans une seule transaction** (pas de stock fantôme) ;
- toute écriture de stock passe par `UPDATE inventory_item SET on_hand = on_hand + $1, version = version + 1 WHERE id = $2 AND on_hand + $1 >= 0` (verrouillage optimiste par `version` + condition de non-négativité) ;
- **anti-survente** : une réservation n'est jamais augmentée par cette voie ; les réservations existantes sont libérées par le job Phase 2, pas ici ;
- écriture d'une ligne `inventory_movement` (`kind` CHECK Phase 2) et d'une entrée `audit_log` (`stock.adjusted` avec la quantité avant/après).

`BULK_ADJUSTMENT_LIMIT` (défaut `100` unités, `à valider (décision propriétaire)`) : au-delà, l'opération exige la permission `stock:update` **et** une MFA vérifiée depuis moins de 10 minutes ; sinon `403 bulk_adjustment_forbidden` + entrée d'audit `stock.bulk_adjustment.denied`.

- [ ] **Step 4.9: Implémenter les seuils et les alertes**

`setThresholds()` écrit `reorder_point` et `reorder_qty` (≥ 0) et journalise `stock.threshold_updated`.

`apps/worker/src/jobs/stock/stock-alert.job.ts` — job `stock.alert` (infra §11.5, type n° 11 « alerte stock ») : balaye `inventory_item` et crée une notification `stock.low_stock` (Task 12) pour les couples (variante, entrepôt) dont `on_hand - reserved <= reorder_point`, avec anti-répétition (une alerte active par variante, pas de doublon daily). La fréquence (quotidienne) est un paramètre de job documenté.

- [ ] **Step 4.10: Implémenter l'inventaire physique et le rapprochement**

1. `startCount()` : crée un `stock_count` (`status = 'open'`) et une `stock_count_line` par `inventory_item` du périmètre (`expected_qty` = `on_hand` au moment de l'ouverture, figée).
2. `recordCount()` : saisit les quantités comptées ; `variance = counted_qty - expected_qty` ; statut `counted`.
3. `closeCount()` : passe le comptage en `closed` (plus de saisie).
4. `reconcileCount()` : applique les écarts via des mouvements `adjust` (un mouvement par ligne non nulle), pose `status = 'reconciled'` et journalise `stock.count_reconciled` avec le nombre de lignes ajustées.

Le rapprochement est **idempotent** : rejouer `reconcileCount()` sur un comptage déjà rapproché ne crée aucun mouvement supplémentaire (statut `reconciled` → 409 `count_already_reconciled`).

- [ ] **Step 4.11: Tests verts**

```bash
pnpm --filter @fanours/api test src/modules/admin-stock
```

Attendu :

```text
 ✓ src/modules/admin-stock/admin-stock.service.spec.ts (9 tests)
 Test Files  1 passed (1)
      Tests  9 passed (9)
```

- [ ] **Step 4.12: Page admin stock**

`admin/stock/page.tsx` : 4 onglets (Stock courant · Mouvements · Inventaire · Alertes).

- Onglet « Stock courant » : `DataTable` (SKU, produit, entrepôt, en stock, réservé, disponible, seuil, réassort), `FilterBar` (entrepôt, SKU, niveau d'alerte), marqueurs **texte + couleur** pour « sous seuil ».
- Onglet « Mouvements » : formulaire (type, quantité, motif obligatoire) + garde visuel de la limite de correction massive, avec message d'explication quand la limite est dépassée.
- Onglet « Inventaire » : création d'un comptage, saisie des quantités, écart calculé et affiché (vert/ambre/rouge + icône + montant), bouton « Rapprocher » désactivé tant qu'un écart n'est pas résolu.
- Onglet « Alertes » : alertes de seuil avec la date de détection.

Tous les chiffres proviennent de l'API ; la page ne recalcule ni stock ni écarts.

- [ ] **Step 4.13: Traduire et vérifier**

```bash
pnpm --filter @fanours/i18n check:keys
pnpm --filter @fanours/web build
pnpm exec playwright test e2e/admin/stock.spec.ts --project=functional
```

Attendu :

```text
i18n key parity OK (3 locales, 541 clés)
✓ Built in 45.1s
  3 passed (11.8s)
```

**Preuves :**
- 9 tests unitaires (mouvement refusé sans motif, correction massive bloquée puis autorisée avec permission + MFA, transfert atomique, non-négativité, inventaire et rapprochement idempotent).
- `0113` appliquée ; `stock_count`, `stock_count_line` créées ; `inventory_item.reorder_point/reorder_qty` ajoutées.
- La ressource `stock` est seedée dans `permission` et l'extension de `ck_permission_resource` est en place (vérifié par un test d'intégration qui tente d'insérer une ressource inconnue et attend une violation de contrainte). L'extension porte la contrainte à **18 ressources** et `ck_permission_action` à **13 actions**, par anticipation des Tasks 8, 9, 11 et 12.
- Le test de permissions de la Phase 1 (`apps/api/src/modules/rbac/__tests__/permissions.test.ts`) est mis à jour dans le **même commit** : les comptes codés en dur `13` / `8` sont remplacés par les listes `EXPECTED_RESOURCES` (14 entrées à ce stade) et `EXPECTED_ACTIONS` (13 entrées) ; `expect(ROLES).toHaveLength(9)` est conservé tel quel. Le test passe.
- Le test d'intégration des migrations de la Phase 1 (`packages/db/src/__tests__/migrations.integration.test.ts`) est défigé : `expect(first.length).toBe(files.length)` et `expect(rows.rows).toHaveLength(files.length)` au lieu du `4` figé (déjà faux depuis la Phase 2). Le test passe.
- Un comptage rapproché deux fois ne double pas les mouvements (`inventory_movement` décompte avant/après).

**Commit :**

```bash
git add packages/db/migrations/0113_phase4_stock.sql packages/db/src/schema/phase4-stock.ts packages/db/src/schema.ts packages/domain/src/permissions.ts packages/db/src/seed.ts apps/api/src/modules/rbac/__tests__/permissions.test.ts packages/db/src/__tests__/migrations.integration.test.ts apps/api/src/modules/admin-stock apps/worker/src/jobs/stock "apps/web/src/app/[locale]/(admin)/admin/stock" packages/i18n/src/locales
git commit -m "feat(admin-stock): mouvements de stock, seuils, inventaire et alertes" -m "Implémente le chantier 4.4 (spec §16.5) : mouvements avec motif obligatoire et non-négativité, correction massive bloquée sans permission stock:update et MFA récente, seuils de réassort, job d'alerte stock bas, inventaire physique avec fige des quantités attendues et rapprochement idempotent. Migration 0113 et extension de la contrainte de ressources RBAC." -m "Met à jour le test de permissions de la Phase 1 : les comptes codés en dur 13 ressources / 8 actions sont remplacés par des listes attendues explicites, et le test d'intégration des migrations est défigé (files.length)." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 5: Back-office clients et B2B (4.5) — recherche, décision, tarifs négociés, suspension, documents

**Files:**
- Create: `apps/api/src/modules/admin-customers/admin-customers.service.ts`
- Create: `apps/api/src/modules/admin-customers/admin-customers.controller.ts`
- Create: `apps/api/src/modules/admin-customers/admin-customers.module.ts`
- Test: `apps/api/src/modules/admin-customers/admin-customers.service.spec.ts`
- Modify: `apps/web/src/app/[locale]/(admin)/admin/clients/page.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/clients/[id]/page.tsx`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `business_account` + `business_account_approval` (Phase 3), `customer_price_rule` (Phase 2), `orders` / `payment` (Phase 2), `commercial_document` (Phase 3), `session` (Phase 1), `audit_log` (Phase 1), `admin_action_approval` (Phase 1), `packages/pricing` → résolution des prix B2B (Phase 3).
- Produces:
  - `AdminCustomersService.search(ctx: AdminContext, query: AdminCustomerQueryDto): Promise<Paged<AdminCustomerRowDto>>`
  - `AdminCustomersService.detail(ctx: AdminContext, customerId: string): Promise<AdminCustomerDetailDto>`
  - `AdminCustomersService.orders(ctx: AdminContext, customerId: string): Promise<AdminOrderRowDto[]>`
  - `AdminCustomersService.payments(ctx: AdminContext, customerId: string): Promise<AdminPaymentRowDto[]>`
  - `AdminCustomersService.tickets(ctx: AdminContext, customerId: string): Promise<AdminTicketRowDto[]>`
  - `AdminCustomersService.approve(ctx: AdminContext, customerId: string, input: BusinessDecisionInputDto): Promise<AdminCustomerDetailDto>`
  - `AdminCustomersService.reject(ctx: AdminContext, customerId: string, input: BusinessDecisionInputDto): Promise<AdminCustomerDetailDto>`
  - `AdminCustomersService.suspend(ctx: AdminContext, customerId: string, input: SuspensionInputDto): Promise<AdminCustomerDetailDto>`
  - `AdminCustomersService.liftSuspension(ctx: AdminContext, customerId: string, input: SuspensionInputDto): Promise<AdminCustomerDetailDto>`
  - `AdminCustomersService.setNegotiatedPrices(ctx: AdminContext, customerId: string, rules: NegotiatedPriceInputDto[]): Promise<ApprovalRequestDto>`
  - `AdminCustomersService.resetSecurity(ctx: AdminContext, customerId: string, input: SecurityResetInputDto): Promise<AdminCustomerDetailDto>`
  - `AdminCustomersService.documents(ctx: AdminContext, customerId: string): Promise<AdminCustomerDocumentDto[]>`
  - `AdminCustomersService.documentUrl(ctx: AdminContext, documentId: string): Promise<{ url: string, expiresAt: string }>`
  - Routes : `GET /v1/admin/customers`, `GET /v1/admin/customers/:id`, `GET /v1/admin/customers/:id/orders`, `GET /v1/admin/customers/:id/payments`, `GET /v1/admin/customers/:id/tickets`, `POST /v1/admin/customers/:id/approve`, `POST /v1/admin/customers/:id/reject`, `POST /v1/admin/customers/:id/suspend`, `POST /v1/admin/customers/:id/lift-suspension`, `PUT /v1/admin/customers/:id/prices`, `POST /v1/admin/customers/:id/security/reset`, `GET /v1/admin/customers/:id/documents`, `POST /v1/admin/customers/:id/documents/:documentId/url`.

- [ ] **Step 5.1: Écrire le test rouge de la décision B2B et de l'accès aux documents**

`apps/api/src/modules/admin-customers/admin-customers.service.spec.ts` :

```typescript
describe('AdminCustomersService', () => {
  it('refuse l’auto-approbation d’un compte professionnel', async () => {
    await expect(
      service.approve(ctxAsRole('compliance_manager'), businessAccountId, { note: 'Dossier complet' }),
    ).rejects.toMatchObject({ status: 403, code: 'no_self_approval' });
  });

  it('suspend un compte avec motif et journalise la décision', async () => {
    const result = await service.suspend(ctxAsRole('compliance_manager'), businessAccountId, {
      reason: 'Litige douane en cours',
    });
    expect(result.verificationStatus).toBe('suspended');
    const entries = await readAudit(ctx.db, { action: 'business_account.suspended', objectId: businessAccountId });
    expect(entries[0].metadata).toMatchObject({ reason: 'Litige douane en cours' });
  });

  it('refuse l’accès aux documents à un rôle non autorisé', async () => {
    await expect(service.documents(ctxAsRole('marketing_manager'), businessAccountId)).rejects.toMatchObject({
      status: 403,
      code: 'forbidden',
    });
  });

  it('ne renvoie jamais d’URL publique de document, seulement une URL signée expirant', async () => {
    const { url, expiresAt } = await service.documentUrl(ctxAsRole('compliance_manager'), documentId);
    expect(url).toMatch(/^https:\/\/[^/]+\/.*X-Amz-Signature=/);
    expect(new Date(expiresAt).getTime() - Date.now()).toBeLessThanOrEqual(5 * 60 * 1000 + 1000);
  });
});
```

- [ ] **Step 5.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/api test src/modules/admin-customers/admin-customers.service.spec.ts
```

Attendu :

```text
 FAIL  src/modules/admin-customers/admin-customers.service.spec.ts
Error: Cannot find module './admin-customers.service.js'
Test Files  1 failed (1)
     Tests  0 passed | 4 failed (4)
```

- [ ] **Step 5.3: Implémenter la recherche et le détail**

`AdminCustomerQueryDto` : `{ q?: string, segment?: 'b2c' | 'b2b', verificationStatus?, country?, limit?, cursor? }`.

- La recherche par `q` accepte : identifiant client, e-mail (comparaison sur `email_normalized`), référence B2B. Une recherche par e-mail **partiel** est autorisée mais le résultat ne révèle que les lignes dont l'e-mail normalisé correspond exactement au terme ou commence par lui, et les données de contact restent masquées hors `support_agent` / `order_manager`.
- Anti-énumération : un compte non trouvé et un compte hors périmètre renvoient la même réponse `404 customer_not_found`.
- Le détail assemble : identité (nom, e-mail masqué pour les rôles non autorisés, pays, société, identifiant fiscal **chiffré**), compte professionnel (statut de vérification, historique de décision, motifs), commandes, paiements, tickets, documents B2B (nom, type, date, statut — jamais l'URL), tarifs négociés (règles `customer_price_rule` en lecture), et **les actions autorisées pour le rôle courant** (calculées côté serveur).

- [ ] **Step 5.4: Implémenter décision, suspension et levée**

`suspend()` : positionne `business_account.verification_status = 'suspended'` (valeur déjà autorisée par le CHECK Phase 3), exige un motif non vide, révoque les sessions actives du compte (`session` Phase 1), publie l'outbox `business_account.suspended` (notification Task 12) et journalise `business_account.suspended` avec `{ reason }` dans `metadata`.

`liftSuspension()` : exige un motif, journalise `business_account.suspension_lifted` **et** écrit une ligne `business_account_approval` (décision tracée), et notifie le client.

`reject()` : motif obligatoire, statut `rejected`, notification au client (canal e-mail, template localisé 3 langues, Task 12).

- [ ] **Step 5.5: Implémenter les tarifs négociés (quatre-yeux)**

`setNegotiatedPrices()` :

1. Vérifie la permission `price_tier:update`.
2. Calcule `payload_hash` (SHA-256 de la liste normalisée des règles) et crée une demande dans `admin_action_approval` (kind `price_tier_change`, `amount` = total des règles si chiffrable, `currency`, `expires_at = now() + 24 h`).
3. **Ne modifie pas** `customer_price_rule` à ce stade.
4. L'approbation puis l'exécution sont faites par la Task 13 (moteur `sensitive-action`) ; l'exécution applique les règles dans une transaction et journalise `customer_price_rule.updated`.

Le front affiche donc « Demande d'approbation envoyée » et non « Tarifs appliqués ».

- [ ] **Step 5.6: Implémenter la réinitialisation de sécurité et l'accès aux documents**

`resetSecurity()` : réservé à `super_admin` ; exige MFA récente (10 min) ; révoque toutes les sessions et les jetons de récupération du compte ; **ne modifie jamais le mot de passe** ; journalise `customer.security_reset` avec `{ actorId, targetId }` (aucun secret dans `metadata`).

`documents()` : rôles autorisés `compliance_manager`, `super_admin`, `support_agent` (lecture de métadonnées). Chaque lecture d'URL signe une nouvelle URL de 5 minutes et journalise `business_document.accessed` avec `{ documentId, customerId }`.

- [ ] **Step 5.7: Tests verts**

```bash
pnpm --filter @fanours/api test src/modules/admin-customers
```

Attendu :

```text
 ✓ src/modules/admin-customers/admin-customers.service.spec.ts (8 tests)
 Test Files  1 passed (1)
      Tests  8 passed (8)
```

- [ ] **Step 5.8: Pages admin clients**

`admin/clients/page.tsx` (modification de la page Phase 3) : recherche unifiée (identifiant / e-mail / référence), filtres (segment, statut de vérification, pays), `DataTable` (client, pays, segment, statut, CA cumulé, dernière commande, actions). Les colonnes financières ne sont visibles que pour les rôles financiers.

`admin/clients/[id]/page.tsx` : onglets (Profil · Commandes · Paiements · Tickets · Documents · Tarifs · Journal). Bandeau d'action contextuel : « Approuver » / « Refuser » / « Suspendre » / « Lever la suspension » / « Réinitialiser la sécurité », chacun affiché **seulement** si le rôle courant le permet et avec son motif obligatoire quand requis.

- [ ] **Step 5.9: Traduire et vérifier**

```bash
pnpm --filter @fanours/i18n check:keys
pnpm --filter @fanours/web build
```

Attendu :

```text
i18n key parity OK (3 locales, 566 clés)
✓ Built in 43.7s
```

**Preuves :**
- 8 tests unitaires (auto-approbation refusée, suspension journalisée, accès documents refusé par rôle, URL signée ≤ 5 min, anti-énumération).
- Une modification de tarif négocié crée une demande d'approbation et **ne** modifie pas `customer_price_rule` avant approbation (vérifié par lecture de la table).
- **Matrice RBAC inchangée par cette tâche** : le chantier 4.5 n'ajoute **ni ressource ni action** — il réutilise `business_account` (`read`, `approve`), `price_tier` (`update`), `business_document` (`read`), `order` (`read`) et `export`, tous déjà déclarés en Phase 1. `packages/domain/src/permissions.ts` n'est donc **pas** modifié ici, et le test de permissions de la Phase 1 (`apps/api/src/modules/rbac/__tests__/permissions.test.ts`) reste vert sans retouche. Le compteur de ressources passe de 14 (après la Task 4) à 14 à cette étape.
- `business_account.verification_status` passe à `suspended` puis revient à `approved` à la levée, avec deux entrées d'audit.
- Aucune URL de document n'apparaît dans les logs ni dans le HTML rendu (le HTML ne contient que la méta-donnée).

**Commit :**

```bash
git add apps/api/src/modules/admin-customers "apps/web/src/app/[locale]/(admin)/admin/clients" packages/i18n/src/locales
git commit -m "feat(admin-customers): recherche client, decision B2B, suspension et documents" -m "Implémente le chantier 4.5 (spec §16.6) : recherche par identifiant, e-mail normalisé ou référence B2B avec anti-enumeration, décision d'approbation ou refus, suspension et levée avec motif journalisé, tarifs negociés soumis au quatre-yeux, reinitialisation de securite sans acces au mot de passe, acces aux documents B2B par URL signee de 5 minutes tracee." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 6: Back-office finance (4.6) — rapprochement quotidien, écarts, export financier quatre-yeux

**Files:**
- Create: `packages/db/migrations/0114_phase4_reconciliation.sql`
- Create: `e2e/admin/finance.spec.ts` (recette fonctionnelle de phase — projet Playwright `functional`)
- Create: `packages/db/src/schema/phase4-finance.ts`
- Modify: `packages/db/src/schema.ts`
- Create: `apps/api/src/modules/admin-finance/reconciliation.service.ts`
- Create: `apps/api/src/modules/admin-finance/finance-export.service.ts`
- Create: `apps/api/src/modules/admin-finance/admin-finance.controller.ts`
- Create: `apps/api/src/modules/admin-finance/admin-finance.module.ts`
- Test: `apps/api/src/modules/admin-finance/reconciliation.service.spec.ts`
- Create: `apps/worker/src/jobs/payment/payment-reconcile.job.ts`
- Create: `apps/worker/src/jobs/export/export-data.job.ts`
- Create: `packages/ui/src/components/ReconciliationTable.tsx`
- Test: `packages/ui/src/components/ReconciliationTable.test.tsx`
- Modify: `packages/ui/src/components/index.ts`
- Create: `apps/web/src/app/[locale]/(admin)/admin/finance/page.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/finance/rapprochement/page.tsx`
- Create: `apps/web/src/components/admin/ReconciliationFilters.tsx`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `payment` / `payment_event` / `refund` (Phase 2), `orders` / `order_line` (Phase 2), `invoice` (Phase 2), `export_request` (migration `0112`, Task 3), `admin_action_approval` (Phase 1), `audit_log` (Phase 1), `packages/payment` → `PaymentProvider` (Phase 2), `packages/ui` → `DataTable`, `KpiCard`.
- Produces:
  - `ReconciliationService.run(ctx: AdminContext, input: ReconciliationRunInputDto): Promise<{ runId: string }>`
  - `ReconciliationService.list(ctx: AdminContext, query: ReconciliationQueryDto): Promise<Paged<ReconciliationLineDto>>`
  - `ReconciliationService.get(ctx: AdminContext, runId: string): Promise<ReconciliationRunDto>`
  - `ReconciliationService.annotate(ctx: AdminContext, lineId: string, input: ReconciliationAnnotationDto): Promise<ReconciliationLineDto>`
  - `ReconciliationService.resolve(ctx: AdminContext, lineId: string, input: ReconciliationResolutionDto): Promise<ReconciliationLineDto>`
  - `ReconciliationService.accept(ctx: AdminContext, lineId: string, input: ReconciliationResolutionDto): Promise<ReconciliationLineDto>`
  - `FinanceExportService.request(ctx: AdminContext, query: FinanceExportQueryDto): Promise<{ approvalId: string }>`
  - `FinanceExportService.status(ctx: AdminContext, exportId: string): Promise<ExportStatusDto>`
  - `RECONCILIATION_BASES` (`gross_revenue` | `orders_created` | `cash_received`) et `RECONCILIATION_STATUSES` (`matched` | `variance` | `missing_internal` | `missing_provider`).
  - Routes : `POST /v1/admin/finance/reconciliation/runs`, `GET /v1/admin/finance/reconciliation/runs`, `GET /v1/admin/finance/reconciliation/runs/:runId`, `GET /v1/admin/finance/reconciliation/lines`, `POST /v1/admin/finance/reconciliation/lines/:lineId/annotate`, `POST /v1/admin/finance/reconciliation/lines/:lineId/resolve`, `POST /v1/admin/finance/reconciliation/lines/:lineId/accept`, `POST /v1/admin/finance/exports`, `GET /v1/admin/finance/exports/:exportId`.

- [ ] **Step 6.1: Écrire le test rouge du rapprochement et de l'export quatre-yeux**

`apps/api/src/modules/admin-finance/reconciliation.service.spec.ts` :

```typescript
describe('ReconciliationService', () => {
  it('detecte un ecart de frais et marque la ligne en anomalie', async () => {
    await seedPayments(ctx.db, [
      { orderNumber: 'FO-2026-000100', amount: '100.00', providerAmount: '100.00', fees: '2.50' },
      { orderNumber: 'FO-2026-000101', amount: '80.00', providerAmount: '80.00', fees: '0.00' },
    ]);

    const { runId } = await service.run(ctx, { periodStart: '2026-10-01', periodEnd: '2026-10-02', provider: 'sandbox' });
    const lines = await service.list(ctx, { runId, hasDifference: true });

    expect(lines.items).toHaveLength(1);
    expect(lines.items[0]).toMatchObject({ orderNumber: 'FO-2026-000100', status: 'variance' });
    expect(lines.items[0].difference).toBe('2.50');
  });

  it('permet d’annoter, d’attribuer et de resoudre un ecart', async () => {
    const line = await seedReconciliationLine(ctx.db, { difference: '5.00' });

    const annotated = await service.annotate(ctx, line.id, { note: 'Frais non factures par le prestataire', ownerId: supportAgentId });
    expect(annotated.state).toBe('investigating');
    expect(annotated.ownerId).toBe(supportAgentId);

    const resolved = await service.resolve(ctx, line.id, { note: 'Ecart confirme par le prestataire' });
    expect(resolved.state).toBe('resolved');
    expect(resolved.resolvedAt).not.toBeNull();
  });

  it('refuse l’export financier sans second approbateur', async () => {
    const result = await service.request(ctx, { periodStart: '2026-10-01', periodEnd: '2026-10-31' });
    expect(result.approvalId).toBeTruthy();
    const approval = await readApproval(ctx.db, result.approvalId);
    expect(approval.kind).toBe('financial_export');
    expect(approval.approvedBy).toBeNull();
  });
});
```

- [ ] **Step 6.1b: Écrire le test E2E rouge de la recette fonctionnelle du back-office finance**

`e2e/admin/finance.spec.ts` — recette fonctionnelle de phase, projet Playwright `functional`. Écrit **avant** l'implémentation : il échoue à ce stade et passe à l'étape « Traduire et vérifier ».

Cas et assertions :

1. Le rapprochement quotidien affiche les écarts avec leur montant, leur cause et leur date ; **aucune** valeur estimée n'est affichée à la place d'un écart inconnu.
2. L'export financier exige l'approbation **quatre-yeux** d'un second rôle ; l'auto-approbation est refusée, l'export reste en attente et la tentative est tracée.
3. L'export est servi par une **URL signée de 5 minutes** dans le bucket privé ; l'URL n'apparaît dans aucun log et n'est jamais publique.
4. Deux bases de comptabilisation distinctes (encaissements vs créances) ne sont jamais mélangées dans un même indicateur : si la donnée source manque, l'indicateur affiche « indisponible » **avec sa raison**.

- [ ] **Step 6.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/api test src/modules/admin-finance/reconciliation.service.spec.ts
```

Attendu :

```text
 FAIL  src/modules/admin-finance/reconciliation.service.spec.ts
Error: Cannot find module './reconciliation.service.js'
Test Files  1 failed (1)
     Tests  0 passed | 3 failed (3)
```

- [ ] **Step 6.3: Écrire la migration `0114`**

`packages/db/migrations/0114_phase4_reconciliation.sql` :

```sql
-- 0114_phase4_reconciliation.sql
-- Phase 4 — Task 6 : rapprochement quotidien des paiements et suivi des ecarts.
SET lock_timeout = '5s';
SET statement_timeout = '30s';

CREATE TABLE payment_reconciliation (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  run_date       date NOT NULL,
  provider       text NOT NULL,
  period_start   timestamptz NOT NULL,
  period_end     timestamptz NOT NULL,
  basis          text NOT NULL DEFAULT 'cash_received',
  status         text NOT NULL DEFAULT 'completed',
  currency       char(3) NOT NULL,
  total_internal numeric(18,4) NOT NULL DEFAULT 0,
  total_provider numeric(18,4) NOT NULL DEFAULT 0,
  total_fees     numeric(18,4) NOT NULL DEFAULT 0,
  total_refunds  numeric(18,4) NOT NULL DEFAULT 0,
  difference     numeric(18,4) NOT NULL DEFAULT 0,
  triggered_by   text NOT NULL DEFAULT 'job',
  correlation_id uuid NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT payment_reconciliation_basis CHECK (basis IN ('gross_revenue', 'orders_created', 'cash_received')),
  CONSTRAINT payment_reconciliation_status CHECK (status IN ('running', 'completed', 'failed')),
  CONSTRAINT payment_reconciliation_period CHECK (period_end > period_start),
  CONSTRAINT payment_reconciliation_trigger CHECK (triggered_by IN ('job', 'manual')),
  CONSTRAINT payment_reconciliation_unique UNIQUE (run_date, provider, basis, period_start, period_end)
);
CREATE INDEX idx_payment_reconciliation_run ON payment_reconciliation(run_date DESC);

CREATE TABLE payment_reconciliation_line (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id           uuid NOT NULL REFERENCES payment_reconciliation(id) ON DELETE CASCADE,
  order_id         uuid REFERENCES orders(id),
  order_number     text,
  customer_id      uuid REFERENCES "user"(id),
  country_code     char(2),
  currency         char(3) NOT NULL,
  method           text,
  status           text NOT NULL,
  provider_ref     text,
  internal_amount  numeric(18,4) NOT NULL DEFAULT 0,
  provider_amount  numeric(18,4) NOT NULL DEFAULT 0,
  fees             numeric(18,4) NOT NULL DEFAULT 0,
  refunds_amount   numeric(18,4) NOT NULL DEFAULT 0,
  difference       numeric(18,4) NOT NULL DEFAULT 0,
  has_difference   boolean NOT NULL DEFAULT false,
  state            text NOT NULL DEFAULT 'open',
  note             text,
  owner_id         uuid REFERENCES "user"(id),
  resolved_at      timestamptz,
  resolved_by      uuid REFERENCES "user"(id),
  CONSTRAINT payment_reconciliation_line_state CHECK (state IN ('open', 'investigating', 'accepted', 'resolved')),
  CONSTRAINT payment_reconciliation_line_difference CHECK (has_difference = (difference <> 0))
);
CREATE INDEX idx_payment_reconciliation_line_run ON payment_reconciliation_line(run_id, has_difference);
CREATE INDEX idx_payment_reconciliation_line_owner ON payment_reconciliation_line(owner_id) WHERE state IN ('open', 'investigating');
```

- [ ] **Step 6.4: Appliquer la migration et écrire le schéma Drizzle**

```bash
pnpm --filter @fanours/db migrate
```

Attendu :

```text
migrating 0114_phase4_reconciliation.sql
0114_phase4_reconciliation.sql applied (checksum 1b73…)
```

- [ ] **Step 6.5: Implémenter le moteur de rapprochement**

`ReconciliationService.run()` :

1. Vérifie la permission `export_financial:read` (lecture) — le **lancement manuel** exige en plus `export_financial:update` et une MFA récente (10 min).
2. Appelle `PaymentProvider.listSettlements({ from, to, currency })` (interface Phase 2, implémentation sandbox de test) pour obtenir les règlements côté prestataire.
3. Compare ligne à ligne sur la clé `provider_ref` (repli : `order_number`) : interne (`payment` + `refund`) contre prestataire.
4. Calcule `difference = provider_amount - internal_amount + fees` : les frais retenus par le prestataire sont déduits de l'encaissement. La convention est figée par les tests et documentée dans le DTO.
5. **Les bases ne sont jamais mélangées** : `gross_revenue` (montant des commandes payées, hors remboursements), `orders_created` (nombre de commandes créées), `cash_received` (encaissements nets des frais et remboursements). Une ligne porte sa base ; l'API ne renvoie jamais une somme combinant deux bases.
6. Écrit `payment_reconciliation` et `payment_reconciliation_line` en une transaction, puis journalise `finance.reconciliation.run` avec `{ runId, totalInternal, totalProvider, totalFees, difference }`.

**Reproductibilité quotidienne** : le job `payment.reconcile` (infra §11.5, type n° 7 « rapprochement paiement ») s'exécute chaque jour à 02:30 UTC sur la journée précédente, avec `idempotencyKey = payment.reconcile:<YYYY-MM-DD>:<provider>`. Un rejeu du même jour ne crée pas de second run (contrainte d'unicité `payment_reconciliation_unique`) : c'est la garantie « rapprochement quotidien reproductible » du critère de sortie de phase.

Statut d'une ligne : `matched` (aucun écart), `variance` (écart de montant ou de frais), `missing_internal` (paiement côté prestataire sans ligne interne), `missing_provider` (paiement interne absent du relevé prestataire).

- [ ] **Step 6.6: Implémenter annotation, attribution et résolution**

- `annotate()` : `note` obligatoire (min 10 caractères) ; `ownerId` facultatif mais **obligatoire** pour passer l'état à `investigating` ; journalise `finance.reconciliation.annotated` avec `{ lineId, ownerId }` (le contenu de la note reste dans la colonne `note`, jamais dans `metadata`).
- `accept()` : marque l'écart comme accepté (écart connu et assumé) — l'écart reste affiché, il n'est jamais masqué ; journalise `finance.reconciliation.accepted`.
- `resolve()` : `state = 'resolved'`, `resolved_at`, `resolved_by` ; journalise `finance.reconciliation.resolved`.

La vie de l'écart est **suivie jusqu'à résolution** : la page finance affiche un compteur « écarts ouverts / en cours d'investigation / acceptés / résolus ».

- [ ] **Step 6.7: Implémenter l'export financier à quatre-yeux**

`FinanceExportService.request()` :

1. Permission `export_financial:read` + MFA récente.
2. Crée une demande `admin_action_approval` (kind `financial_export`, `payload` = filtres de période/devise, `payload_hash` = SHA-256, `amount` = null ou total, `currency`, `expires_at = now() + 24 h`).
3. **Ne produit aucun fichier** à ce stade : la demande doit être approuvée par un second rôle (`finance_manager` ou `super_admin`, jamais l'auteur de la demande).
4. Après approbation, l'exécution crée une ligne `export_request` (`kind = 'financial'`) et publie l'outbox `export.requested` ; le job `export.data` écrit le CSV dans le bucket privé et sert une URL signée 5 min.
5. L'export est **journalisé** (`finance.export.requested`, `finance.export.executed`) et l'accès au fichier est tracé.

L'export CSV est **limité aux rôles autorisés** (`export_financial:read` ⇒ `finance_manager`, `super_admin`) : un `analyst_readonly` reçoit `403`.

- [ ] **Step 6.8: Implémenter le job d'export générique**

`apps/worker/src/jobs/export/export-data.job.ts` : job `export.data` (infra §11.5, type n° 9 « export »), `idempotencyKey = export:<exportRequestId>`, retry avec backoff + jitter, DLQ. Il :

- relit `export_request` ;
- produit le CSV selon `kind` (`catalog`, `orders`, `financial`, `audit`) ;
- écrit le fichier dans le bucket privé sous `exports/<exportId>/<kind>.csv` ;
- renseigne `storage_key`, `row_count`, `completed_at`, `status = 'ready'`, `expires_at = now() + 24 h` (durée de conservation du fichier exporté, `à valider (décision propriétaire)`) ;
- n'écrit jamais le contenu du fichier dans les logs.

- [ ] **Step 6.9: Tests verts**

```bash
pnpm --filter @fanours/api test src/modules/admin-finance
pnpm --filter @fanours/ui test src/components/ReconciliationTable.test.tsx
```

Attendu :

```text
 ✓ src/modules/admin-finance/reconciliation.service.spec.ts (7 tests)
 ✓ src/components/ReconciliationTable.test.tsx (4 tests)
 Test Files  2 passed (2)
      Tests  11 passed (11)
```

`ReconciliationTable` (design.md §3.4.5) : colonnes période, commande, client, pays, devise, moyen, statut, ID externe, frais, écart ; la cellule d'écart porte un statut **vert / ambre / rouge + icône + montant** (jamais la couleur seule) ; annotation et attribution par ligne ; export réservé aux rôles autorisés.

- [ ] **Step 6.10: Pages admin finance et rapprochement**

`admin/finance/page.tsx` : indicateurs financiers (chiffre d'affaires, encaissements, commandes, remboursements) **chacun avec sa base affichée** (« Base : commandes payées, hors remboursements ») ; le groupe de KPI refuse d'afficher deux bases différentes sans un avertissement explicite.

`admin/finance/rapprochement/page.tsx` : sélecteur de période + `ReconciliationFilters` (commande, client, pays, devise, moyen, statut, ID externe, remboursements, frais, écarts), `ReconciliationTable`, panneau latéral par ligne (annotation, attribution, résolution), compteur d'écarts, bouton « Lancer un rapprochement » (MFA requise), bouton « Exporter » (demande d'approbation).

- [ ] **Step 6.11: Traduire et vérifier**

```bash
pnpm --filter @fanours/i18n check:keys
pnpm --filter @fanours/web build
pnpm exec playwright test e2e/admin/finance.spec.ts --project=functional
```

Attendu :

```text
i18n key parity OK (3 locales, 601 clés)
✓ Built in 46.2s
  4 passed (16.9s)
```

**Preuves :**
- 11 tests unitaires (écart de frais détecté, annotation/attribution/résolution, export non exécuté sans approbation, base non mélangée, idempotence du run quotidien).
- `0114` appliquée avec checksum ; `payment_reconciliation` et `payment_reconciliation_line` créées.
- Un run de rapprochement rejoué sur la même date ne crée pas de second run (unicité vérifiée).
- Un export financier demandé mais non approuvé ne produit **aucun** fichier dans le bucket (bucket vérifié vide).
- Un `analyst_readonly` reçoit `403` sur `/v1/admin/finance/exports`.

**Commit :**

```bash
git add packages/db/migrations/0114_phase4_reconciliation.sql packages/db/src/schema/phase4-finance.ts packages/db/src/schema.ts apps/api/src/modules/admin-finance apps/worker/src/jobs/payment apps/worker/src/jobs/export packages/ui/src/components/ReconciliationTable.tsx packages/ui/src/components/ReconciliationTable.test.tsx packages/ui/src/components/index.ts "apps/web/src/app/[locale]/(admin)/admin/finance" apps/web/src/components/admin/ReconciliationFilters.tsx packages/i18n/src/locales
git commit -m "feat(admin-finance): rapprochement quotidien, ecarts et export quatre-yeux" -m "Implémente le chantier 4.6 (spec §10.5, §16.7) : rapprochement reproductible par le job payment.reconcile idempotent, detection des ecarts de montant et de frais, annotation et attribution par ligne avec suivi jusqu'a resolution, bases de KPI non melangees, export financier soumis au quatre-yeux via admin_action_approval puis job export.data vers le bucket prive." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 7: Back-office logistique et douane (4.7) — expéditions, exceptions, configuration commerciale, règles à revalider

**Files:**
- Create: `apps/api/src/modules/admin-logistics/admin-logistics.service.ts`
- Create: `e2e/admin/logistics.spec.ts` (recette fonctionnelle de phase — projet Playwright `functional`)
- Create: `apps/api/src/modules/admin-logistics/admin-compliance-config.service.ts`
- Create: `apps/api/src/modules/admin-logistics/admin-logistics.controller.ts`
- Create: `apps/api/src/modules/admin-logistics/admin-logistics.module.ts`
- Test: `apps/api/src/modules/admin-logistics/admin-logistics.service.spec.ts`
- Modify: `apps/web/src/app/[locale]/(admin)/admin/expeditions/page.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/logistique/page.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/douane/configuration/page.tsx`
- Modify: `apps/web/src/components/admin/AdminNav.tsx`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `shipment` / `shipment_item` / `package` / `tracking_event` / `commercial_document` (Phase 3), `country_profile` / `incoterm_configuration` / `trade_rule` / `trade_rule_version` / `compliance_alert` (Phase 3), `AdminShipmentsService` (Phase 3), `ComplianceAlertService` (Phase 3), `AdminTradeRulesService` (Phase 3), `audit_log` (Phase 1).
- Produces:
  - `AdminLogisticsService.listShipments(ctx: AdminContext, query: AdminShipmentQueryDto): Promise<Paged<AdminShipmentRowDto>>`
  - `AdminLogisticsService.detail(ctx: AdminContext, shipmentId: string): Promise<AdminShipmentDetailDto>`
  - `AdminLogisticsService.toShip(ctx: AdminContext, query: AdminShipmentQueryDto): Promise<Paged<AdminShipmentRowDto>>`
  - `AdminLogisticsService.documentsToGenerate(ctx: AdminContext): Promise<AdminDocumentToGenerateDto[]>`
  - `AdminLogisticsService.createLabel(ctx: AdminContext, shipmentId: string): Promise<{ trackingNumber: string }>`
  - `AdminLogisticsService.syncTracking(ctx: AdminContext, shipmentId: string): Promise<AdminShipmentDetailDto>`
  - `AdminLogisticsService.exceptions(ctx: AdminContext, query: AdminExceptionQueryDto): Promise<Paged<AdminShipmentExceptionDto>>`
  - `AdminLogisticsService.resolveException(ctx: AdminContext, shipmentId: string, input: ExceptionResolutionDto): Promise<AdminShipmentDetailDto>`
  - `AdminLogisticsService.customsStatus(ctx: AdminContext, shipmentId: string): Promise<CustomsCompletenessDto>`
  - `AdminComplianceConfigService.countries(ctx: AdminContext): Promise<AdminCountryProfileDto[]>`
  - `AdminComplianceConfigService.updateCountry(ctx: AdminContext, code: string, input: CountryConfigInputDto): Promise<AdminCountryProfileDto>`
  - `AdminComplianceConfigService.incoterms(ctx: AdminContext): Promise<IncotermConfigurationDto[]>`
  - `AdminComplianceConfigService.setIncoterm(ctx: AdminContext, input: IncotermInputDto): Promise<IncotermConfigurationDto>`
  - `AdminComplianceConfigService.taxCollectionModes(ctx: AdminContext): Promise<TaxCollectionModeDto[]>`
  - `AdminComplianceConfigService.setTaxCollectionMode(ctx: AdminContext, input: TaxCollectionModeInputDto): Promise<IncotermConfigurationDto>`
  - `AdminComplianceConfigService.matrix(ctx: AdminContext, query: ComplianceMatrixQueryDto): Promise<Paged<ComplianceMatrixRowDto>>`
  - `AdminComplianceConfigService.reviewDue(ctx: AdminContext): Promise<Paged<ComplianceReviewDueDto>>`
  - `AdminComplianceConfigService.blockRule(ctx: AdminContext, ruleId: string, input: RuleBlockInputDto): Promise<void>`
  - `AdminComplianceConfigService.ruleHistory(ctx: AdminContext, ruleId: string): Promise<AuditEntryDto[]>`
  - Routes : `GET /v1/admin/logistics/shipments`, `GET /v1/admin/logistics/shipments/:id`, `GET /v1/admin/logistics/to-ship`, `GET /v1/admin/logistics/documents`, `POST /v1/admin/logistics/shipments/:id/label`, `POST /v1/admin/logistics/shipments/:id/sync`, `GET /v1/admin/logistics/exceptions`, `POST /v1/admin/logistics/shipments/:id/resolve-exception`, `GET /v1/admin/logistics/shipments/:id/customs`, `GET /v1/admin/compliance/countries`, `PATCH /v1/admin/compliance/countries/:code`, `GET /v1/admin/compliance/incoterms`, `POST /v1/admin/compliance/incoterms`, `GET /v1/admin/compliance/tax-modes`, `POST /v1/admin/compliance/tax-modes`, `GET /v1/admin/compliance/matrix`, `GET /v1/admin/compliance/review-due`, `POST /v1/admin/compliance/rules/:ruleId/block`, `GET /v1/admin/compliance/rules/:ruleId/history`.

- [ ] **Step 7.1: Écrire le test rouge de l'exception et du blocage de règle**

`apps/api/src/modules/admin-logistics/admin-logistics.service.spec.ts` :

```typescript
describe('AdminLogisticsService', () => {
  it('refuse d’expedier une commande vers un pays suspendu', async () => {
    await seedShipment(ctx.db, { shipmentId, countryCode: 'XX', operationalStatus: 'suspended' });

    await expect(service.createLabel(ctx, shipmentId)).rejects.toMatchObject({
      status: 409,
      code: 'country_suspended',
    });
  });

  it('signale une completude douaniere incomplete tant qu’une exception est ouverte', async () => {
    await seedShipment(ctx.db, { shipmentId, exceptionReason: 'missing_hs_code', exceptionStatus: 'open' });

    const status = await service.customsStatus(ctx, shipmentId);

    expect(status.complete).toBe(false);
    expect(status.missingDocuments).toContain('commercial_invoice');
  });

  it('journalise le blocage d’une règle critique', async () => {
    await service.blockRule(ctx, ruleId, { reason: 'Valeur obsolete, revalidation requise' });

    const entries = await readAudit(ctx.db, { action: 'customs_rule.blocked', objectId: ruleId });
    expect(entries[0].metadata).toMatchObject({ reason: 'Valeur obsolete, revalidation requise' });
  });
});
```

- [ ] **Step 7.1b: Écrire le test E2E rouge de la recette fonctionnelle du back-office logistique et douane**

`e2e/admin/logistics.spec.ts` — recette fonctionnelle de phase, projet Playwright `functional`. Écrit **avant** l'implémentation : il échoue à ce stade et passe à l'étape « Traduire et vérifier ».

Cas et assertions :

1. Une règle douanière **expirée** attachée à un transporteur produit une alerte de conformité visible dans l'admin : la règle n'est jamais appliquée silencieusement.
2. Une configuration commerciale invalide (Incoterm absent ou non supporté) est refusée par l'API (`422`) ; les commandes antérieures restent inchangées.
3. Un rôle sans la permission `logistics:write` ne peut ni créer une exception d'expédition ni modifier une règle à revalider : `403`, message localisé, action tracée.
4. Le tableau des exceptions affiche l'âge du dossier, le pays concerné et l'action requise ; aucune commande d'un autre pays n'apparaît (scoping appliqué **dans la requête**, pas en post-filtrage).

- [ ] **Step 7.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/api test src/modules/admin-logistics/admin-logistics.service.spec.ts
```

Attendu :

```text
 FAIL  src/modules/admin-logistics/admin-logistics.service.spec.ts
Error: Cannot find module './admin-logistics.service.js'
Test Files  1 failed (1)
     Tests  0 passed | 3 failed (3)
```

- [ ] **Step 7.3: Implémenter le service logistique**

`AdminLogisticsService` compose les services Phase 3 (`AdminShipmentsService`, `ComplianceAlertService`, `AdminTradeRulesService`) sans les modifier.

- `listShipments()` : filtres (période, pays, transporteur, statut, Incoterm, niveau d'alerte), scoping par rôle : `logistics_operator` voit les expéditions qui lui sont assignées ; `order_manager` voit celles de ses commandes ; `compliance_manager` et `super_admin` voient tout.
- `detail()` : colis (dimensions, poids, marquages), événements de tracking (historique complet), transporteur et Incoterm appliqué, mode de collecte des taxes, coût d'expédition, **complétude douanière** (documents requis par `country_profile.required_docs` + règle applicable + niveau de certitude de la règle), exception ouverte et son action requise, documents générés.
- `toShip()` : commandes payées sans expédition, prêtes à expédier.
- `documentsToGenerate()` : factures commerciales, déclarations, listes de colisage — à générer.
- `createLabel()` : refuse si pays suspendu (`country_suspended`), si règle critique expirée (`customs_rule_expired`), si exception ouverte (`shipment_exception_open`) ; sinon appelle `ShippingProvider.createShipment` (Phase 3), écrit le numéro de suivi, journalise `shipment.label_created`.
- `syncTracking()` : appelle le transporteur (allowlist d'hôtes, pas d'URL utilisateur — spec §7.5), écrit les `tracking_event` Phase 3, met à jour le statut d'expédition, déclenche les notifications (Task 12).
- `exceptions()` : tableau de bord des exceptions ouvertes (douane, retard, colisage, paiement) avec gravité et action requise.
- `resolveException()` : exige une résolution motivée, renseigne `exception_resolved_at` / `exception_resolved_by` (colonnes Phase 3) et journalise `shipment.exception_resolved`.

- [ ] **Step 7.4: Implémenter la configuration commerciale par pays et la matrice**

`AdminComplianceConfigService` :

- `countries()` : lit `country_profile` (livraison activée, Incoterms autorisés, devises, moyens de paiement, documents requis, catégories restreintes, statut opérationnel, motif de suspension).
- `updateCountry()` : modifications journalisées (`country.updated`) ; la suspension d'un pays passe par le workflow quatre-yeux `admin_action_approval` (kind `country_suspension`) ; `suspended_reason` est obligatoire (contrainte `country_profile_suspension_reason`).
- `incoterms()` / `setIncoterm()` : lit et écrit `incoterm_configuration` (Phase 3) ; un Incoterm désactivé ne peut plus être proposé au checkout (le back-office n'invente pas de valeur hors CHECK `incoterm_configuration_incoterm`).
- `taxCollectionModes()` / `setTaxCollectionMode()` : `prepaid` ou `at_delivery` par pays + Incoterm + date d'entrée en vigueur (`effective_from`).
- `matrix()` : matrice **pays x produits x transporteurs x paiement** : pour chaque couple (pays, catégorie ou variante) affiche l'Incoterm possible, le mode de collecte des taxes, la devise, les moyens de paiement, le transporteur autorisé, la règle douanière applicable et son niveau de certitude. La matrice est **calculée à partir des données** (`country_profile`, `incoterm_configuration`, `trade_rule`) — aucune cellule n'est écrite en dur.
- `reviewDue()` : règles dont la date de dernière revue dépasse le délai de revalidation configuré, ou dont `valid_to` est dépassée, ou dont la certitude est `estimated` / `incomplete` / `specialist_review` → **alerte au compliance manager** (spec §3.3), jamais d'application silencieuse. Le délai de revalidation est la constante `RULE_REVIEW_INTERVAL_DAYS` (défaut 180, `à valider (décision propriétaire)`).
- `blockRule()` : met `active = false` sur une règle, exige un motif, journalise `customs_rule.blocked` ; une règle bloquée n'est plus résolue par le moteur de conformité et les commandes concernées sont bloquées avec une explication.
- `ruleHistory()` : lit `audit_log` filtré sur `object_type = 'trade_rule'` et `object_id = ruleId` → historique complet des modifications.

- [ ] **Step 7.5: Tests verts**

```bash
pnpm --filter @fanours/api test src/modules/admin-logistics
```

Attendu :

```text
 ✓ src/modules/admin-logistics/admin-logistics.service.spec.ts (8 tests)
 Test Files  1 passed (1)
      Tests  8 passed (8)
```

- [ ] **Step 7.6: Pages admin logistique et configuration douane**

`admin/logistique/page.tsx` : 4 onglets (Expéditions · À expédier · Exceptions · Documents). Colonnes : commande, pays, transporteur, Incoterm, colis, statut, tracking, alerte. La page remplace la page Phase 3 `/admin/expeditions`, qui devient une **redirection interne** vers `/admin/logistique` (une seule URL par donnée ; le lien Phase 3 reste valide).

`admin/douane/configuration/page.tsx` : 4 onglets (Pays · Incoterms et taxes · Matrice · Règles à revalider).

- Pays : `DataTable` (pays, livraison, Incoterms, devises, moyens de paiement, documents requis, statut) + édition journalisée.
- Incoterms et taxes : liste `incoterm_configuration` + formulaire (pays, Incoterm, mode de collecte, date d'entrée en vigueur).
- Matrice : `DataTable` croisée avec filtre (pays, catégorie, transporteur, moyen de paiement) ; chaque cellule indique la règle applicable et sa **certitude** (badge textuel : « validée », « estimative », « incomplète », « spécialiste ») ; une règle expirée est signalée en clair.
- Règles à revalider : liste des règles à échéance + bouton « Bloquer la règle » (motif obligatoire).

- [ ] **Step 7.7: Mettre à jour la navigation admin**

`AdminNav.tsx` : l'onglet « Expéditions » devient « Logistique » et pointe vers `/admin/logistique` ; un sous-ensemble « Douane » mène à `/admin/douane/configuration` et conserve les liens Phase 3 `/admin/douane/regles` et `/admin/douane/alertes`. La visibilité de chaque onglet dépend des permissions serveur.

- [ ] **Step 7.8: Traduire et vérifier**

```bash
pnpm --filter @fanours/i18n check:keys
pnpm --filter @fanours/web build
pnpm exec playwright test e2e/admin/logistics.spec.ts --project=functional
```

Attendu :

```text
i18n key parity OK (3 locales, 638 clés)
✓ Built in 47.4s
  3 passed (13.2s)
```

**Preuves :**
- 8 tests unitaires (pays suspendu, exception douanière, blocage de règle journalisé, scoping par rôle, matrice calculée).
- Une commande vers un pays suspendu ne peut pas être expédiée : `409 country_suspended` + entrée d'audit.
- Une règle expirée n'est jamais appliquée silencieusement : le moteur de conformité (Phase 3) refuse et crée une `compliance_alert` ; une alerte est adressée au `compliance_manager` (notification Task 12).
- La matrice pays x produits x transporteurs x paiement est recalculée à partir des tables ; aucun test n'encode une valeur métier en dur.

**Commit :**

```bash
git add apps/api/src/modules/admin-logistics "apps/web/src/app/[locale]/(admin)/admin/logistique" "apps/web/src/app/[locale]/(admin)/admin/douane/configuration" "apps/web/src/app/[locale]/(admin)/admin/expeditions" apps/web/src/components/admin/AdminNav.tsx packages/i18n/src/locales
git commit -m "feat(admin-logistics): expeditions, exceptions et configuration commerciale par pays" -m "Implémente le chantier 4.7 (spec §16.8-16.9) : tableau expéditions, colis, tracking, exceptions, documents à générer, complétude douanière, configuration commerciale par pays, Incoterms et modes de collecte des taxes, matrice pays x produits x transporteurs x paiement calculée depuis les données, règles à revalider et blocage motivé d'une règle critique. Aucun taux ni Incoterm codé en dur." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 8: Back-office contenu et SEO (4.8) — CMS, meta, slugs, redirections, aperçus

**Files:**
- Create: `packages/db/migrations/0115_phase4_content_seo.sql`
- Create: `e2e/seo/content.spec.ts` (recette fonctionnelle de phase — projet Playwright `functional`)
- Create: `packages/db/src/schema/phase4-content.ts`
- Modify: `packages/db/src/schema.ts`
- Modify: `packages/domain/src/permissions.ts`
- Modify: `apps/api/src/modules/rbac/__tests__/permissions.test.ts` (liste `EXPECTED_RESOURCES` — ajout de `content_page`)
- Modify: `packages/db/src/seed.ts`
- Create: `apps/api/src/modules/admin-content/admin-content.service.ts`
- Create: `apps/api/src/modules/admin-content/redirect.service.ts`
- Create: `apps/api/src/modules/admin-content/admin-content.controller.ts`
- Create: `apps/api/src/modules/admin-content/admin-content.module.ts`
- Test: `apps/api/src/modules/admin-content/admin-content.service.spec.ts`
- Create: `apps/web/src/app/[locale]/(admin)/admin/contenu/page.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/contenu/redirections/page.tsx`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `product` / `product_translation` (Phase 2) pour les redirections produit, `audit_log` (Phase 1), `packages/seo` → génération de sitemap et JSON-LD (Phase 2), `packages/i18n` → `LOCALES`.
- Produces:
  - `AdminContentService.listPages(ctx: AdminContext, query: AdminContentQueryDto): Promise<Paged<AdminContentRowDto>>`
  - `AdminContentService.getPage(ctx: AdminContext, pageId: string): Promise<AdminContentDetailDto>`
  - `AdminContentService.savePage(ctx: AdminContext, pageId: string, input: AdminContentInputDto): Promise<AdminContentDetailDto>`
  - `AdminContentService.publishPage(ctx: AdminContext, pageId: string): Promise<AdminContentDetailDto>`
  - `AdminContentService.unpublishPage(ctx: AdminContext, pageId: string): Promise<AdminContentDetailDto>`
  - `AdminContentService.previewPage(ctx: AdminContext, pageId: string, locale: Locale): Promise<ContentPreviewDto>`
  - `AdminContentService.changeSlug(ctx: AdminContext, pageId: string, newSlug: string): Promise<AdminContentDetailDto>`
  - `AdminContentService.setMeta(ctx: AdminContext, pageId: string, locale: Locale, input: MetaInputDto): Promise<AdminContentDetailDto>`
  - `AdminContentService.saleConditions(ctx: AdminContext, segment: 'b2c' | 'b2b', locale: Locale): Promise<SaleConditionDto[]>`
  - `AdminContentService.setSaleCondition(ctx: AdminContext, input: SaleConditionInputDto): Promise<SaleConditionDto>`
  - `RedirectService.list(ctx: AdminContext): Promise<RedirectRuleDto[]>`
  - `RedirectService.create(ctx: AdminContext, input: RedirectInputDto): Promise<RedirectRuleDto>`
  - `RedirectService.remove(ctx: AdminContext, id: string): Promise<void>`
  - Routes : `GET|POST /v1/admin/content/pages`, `GET|PUT /v1/admin/content/pages/:id`, `POST /v1/admin/content/pages/:id/publish`, `POST /v1/admin/content/pages/:id/unpublish`, `GET /v1/admin/content/pages/:id/preview`, `POST /v1/admin/content/pages/:id/slug`, `PUT /v1/admin/content/pages/:id/meta`, `GET|POST /v1/admin/content/redirects`, `DELETE /v1/admin/content/redirects/:id`.

- [ ] **Step 8.1: Écrire le test rouge de la redirection et de l'aperçu**

`apps/api/src/modules/admin-content/admin-content.service.spec.ts` :

```typescript
describe('AdminContentService', () => {
  it('refuse une redirection en boucle', async () => {
    await expect(
      service.create(ctx, { fromPath: '/fr/a', toPath: '/fr/a' }),
    ).rejects.toMatchObject({ status: 422, code: 'redirect_loop' });
  });

  it('cree une redirection unique en 301 lors d’un changement de slug', async () => {
    const page = await seedContentPage(ctx.db, { slug: 'guide-douane' });

    const updated = await service.changeSlug(ctx, page.id, 'guide-douane-2026');

    expect(updated.slug).toBe('guide-douane-2026');
    const redirects = await redirectService.list(ctx);
    expect(redirects).toContainEqual(expect.objectContaining({ fromPath: '/fr/guide-douane', statusCode: 301 }));
  });

  it('renvoie un apercu non indexable', async () => {
    const preview = await service.previewPage(ctx, pageId, 'en');
    expect(preview.robots).toBe('noindex,nofollow');
    expect(preview.canonicalUrl).toBeNull();
  });

  it('refuse de publier une page sans traduction dans la locale demandee', async () => {
    await expect(service.publishPage(ctx, pageId)).rejects.toMatchObject({
      status: 422,
      code: 'content_translation_missing',
    });
  });
});
```

- [ ] **Step 8.1b: Écrire le test E2E rouge de la recette fonctionnelle du back-office contenu et SEO**

`e2e/seo/content.spec.ts` — recette fonctionnelle de phase, projet Playwright `functional`. Écrit **avant** l'implémentation : il échoue à ce stade et passe à l'étape « Traduire et vérifier ».

Cas et assertions :

1. Une page publique renvoie un `canonical` **absolu** et un `hreflang` complet (`fr-FR`, `en`, `zh-CN`, `x-default`) ; le `title` et la meta description sont uniques par page et par locale.
2. Une redirection gérée par le CMS répond `301` vers la nouvelle URL ; un slug inconnu répond un **vrai `404`**, jamais `200` avec la page d'accueil.
3. Un contenu publié en 3 langues est indexable (`index, follow`) ; un contenu en brouillon renvoie `noindex` et est absent du sitemap de la locale.
4. L'aperçu de prépublication est réservé au rôle `marketing_manager` : `403` pour les autres rôles, avec un message localisé et aucune fuite de contenu.

- [ ] **Step 8.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/api test src/modules/admin-content/admin-content.service.spec.ts
```

Attendu :

```text
 FAIL  src/modules/admin-content/admin-content.service.spec.ts
Error: Cannot find module './admin-content.service.js'
Test Files  1 failed (1)
     Tests  0 passed | 4 failed (4)
```

- [ ] **Step 8.3: Écrire la migration `0115`**

`packages/db/migrations/0115_phase4_content_seo.sql` :

```sql
-- 0115_phase4_content_seo.sql
-- Phase 4 — Task 8 : CMS éditorial, redirections, conditions de vente par segment.
SET lock_timeout = '5s';
SET statement_timeout = '30s';

CREATE TABLE content_page (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  slug            text NOT NULL,
  kind            text NOT NULL DEFAULT 'editorial',
  status          text NOT NULL DEFAULT 'draft',
  primary_locale  char(5) NOT NULL DEFAULT 'fr-FR',
  is_indexable    boolean NOT NULL DEFAULT true,
  created_by      uuid NOT NULL REFERENCES "user"(id),
  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now(),
  published_at    timestamptz,
  CONSTRAINT content_page_slug_unique UNIQUE (slug),
  CONSTRAINT content_page_slug_format CHECK (slug ~ '^[a-z0-9]+(?:-[a-z0-9]+)*$'),
  CONSTRAINT content_page_status CHECK (status IN ('draft', 'in_review', 'published', 'archived')),
  CONSTRAINT content_page_kind CHECK (kind IN ('editorial', 'legal', 'landing')),
  CONSTRAINT content_page_locale CHECK (primary_locale IN ('fr-FR', 'en', 'zh-CN'))
);
CREATE INDEX idx_content_page_status ON content_page(status, updated_at DESC);

CREATE TABLE content_page_translation (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  page_id          uuid NOT NULL REFERENCES content_page(id) ON DELETE CASCADE,
  locale           char(5) NOT NULL,
  title            text NOT NULL,
  summary          text,
  body             text NOT NULL DEFAULT '',
  meta_title       text,
  meta_description text,
  canonical_url    text,
  robots           text NOT NULL DEFAULT 'index,follow',
  structured_data  jsonb,
  updated_at       timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT content_page_translation_unique UNIQUE (page_id, locale),
  CONSTRAINT content_page_translation_locale CHECK (locale IN ('fr-FR', 'en', 'zh-CN')),
  CONSTRAINT content_page_translation_robots CHECK (robots IN ('index,follow', 'noindex,follow', 'noindex,nofollow')),
  CONSTRAINT content_page_translation_meta_title CHECK (meta_title IS NULL OR length(meta_title) <= 70),
  CONSTRAINT content_page_translation_meta_description CHECK (meta_description IS NULL OR length(meta_description) <= 180)
);
CREATE INDEX idx_content_page_translation_locale ON content_page_translation(locale);

CREATE TABLE redirect_rule (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  from_path   text NOT NULL,
  to_path     text NOT NULL,
  status_code int NOT NULL DEFAULT 301,
  reason      text,
  created_by  uuid NOT NULL REFERENCES "user"(id),
  created_at  timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT redirect_rule_from_unique UNIQUE (from_path),
  CONSTRAINT redirect_rule_status CHECK (status_code IN (301, 302)),
  CONSTRAINT redirect_rule_no_loop CHECK (from_path <> to_path),
  CONSTRAINT redirect_rule_absolute CHECK (from_path LIKE '/%' AND to_path LIKE '/%')
);
CREATE INDEX idx_redirect_rule_to ON redirect_rule(to_path);

CREATE TABLE sale_condition (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  segment        text NOT NULL,
  locale         char(5) NOT NULL,
  body           text NOT NULL,
  version        int NOT NULL DEFAULT 1,
  effective_from timestamptz NOT NULL DEFAULT now(),
  created_by     uuid NOT NULL REFERENCES "user"(id),
  created_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT sale_condition_unique UNIQUE (segment, locale, version),
  CONSTRAINT sale_condition_segment CHECK (segment IN ('b2c', 'b2b')),
  CONSTRAINT sale_condition_locale CHECK (locale IN ('fr-FR', 'en', 'zh-CN'))
);
CREATE INDEX idx_sale_condition_lookup ON sale_condition(segment, locale, version DESC);
```

- [ ] **Step 8.4: Appliquer la migration et écrire le schéma Drizzle**

```bash
pnpm --filter @fanours/db migrate
```

Attendu :

```text
migrating 0115_phase4_content_seo.sql
0115_phase4_content_seo.sql applied (checksum 4e6b…)
```

- [ ] **Step 8.5: Étendre la matrice de permissions**

`packages/domain/src/permissions.ts` : ajout de la ressource `content_page` avec les actions `read`, `create`, `update`, `delete`, `publish`. Rôles :

- `marketing_manager` : `content_page: ['read', 'create', 'update', 'publish']` ;
- `super_admin` : `content_page: ['read', 'create', 'update', 'delete', 'publish']` ;
- `analyst_readonly` : `content_page: ['read']`.

`packages/db/src/seed.ts` : lignes `permission` correspondantes (seed idempotent, pas de migration).

**Mise à jour du test de permissions de la Phase 1** — `apps/api/src/modules/rbac/__tests__/permissions.test.ts` : `content_page` est ajoutée à `EXPECTED_RESOURCES` dans le **même commit**.

```typescript
const EXPECTED_RESOURCES = [
  'order', 'product', 'price_tier', 'export', 'export_financial', 'refund', 'admin_role',
  'audit_log', 'customs_rule', 'country', 'business_document', 'business_account',
  'customs_document', 'stock', 'content_page',            // <- ajouté en Task 8
] as const;
```

**Décompte à ce stade : 15 ressources (14 après la Task 4 + `content_page`) et 13 actions (inchangées depuis la Task 4).** L'action `publish` fait partie du vocabulaire déclaré en Task 4 : ni `ACTIONS` ni `EXPECTED_ACTIONS` ne changent ici.

```bash
pnpm --filter @fanours/api test src/modules/rbac/__tests__/permissions.test.ts
```

Attendu :

```text
 ✓ src/modules/rbac/__tests__/permissions.test.ts (6 tests)
 Test Files  1 passed (1)
      Tests  6 passed (6)
```

- [ ] **Step 8.6: Implémenter le service de contenu**

`AdminContentService.savePage()` : écrit `content_page` + une ligne `content_page_translation` par locale fournie ; la validation zod impose `meta_title` ≤ 70 caractères et `meta_description` ≤ 180 caractères (bornes de SERP documentées dans la spec SEO).

`publishPage()` : refuse si la locale de publication n'a pas de traduction (`content_translation_missing`), si la page n'a pas de `title` non vide, si `is_indexable` est vrai et qu'une URL canonique pointe vers une autre page (`canonical_conflict`).

`previewPage()` : renvoie le contenu rendu avec `robots = 'noindex,nofollow'` et `canonicalUrl = null` ; la route d'aperçu est servie hors sitemap et avec un en-tête `X-Robots-Tag: noindex, nofollow`.

`changeSlug()` : 1) écrit le nouveau `slug` ; 2) crée **une seule** redirection 301 depuis l'ancien chemin vers le nouveau, pour chacune des locales indexables ; 3) l'ancien slug est retiré du sitemap (le sitemap est généré à partir des slugs publiés) ; 4) la redirection est journalisée (`content.slug_changed`).

- [ ] **Step 8.7: Implémenter le service de redirections**

`RedirectService.create()` refuse :

- `from_path` identique à `to_path` (`redirect_loop`) ;
- `to_path` qui est elle-même une source de redirection existante (`redirect_chain`) ;
- `from_path` déjà redirigée (unicité `redirect_rule_from_unique` → `409 redirect_exists`) ;
- `from_path` correspondant à une route admin ou compte (`private_path_not_redirectable`) ;
- `from_path` correspondant à une page produit publiée : la redirection d'une page produit **indexable** doit passer par `changeSlug` (traçabilité de l'ancien slug) (`redirect_product_requires_slug_change`).

Les redirections sont appliquées par le middleware `apps/web/src/middleware.ts` (Phase 2) avant le rendu, avec un cache de 5 minutes en Redis (invalidé à la création d'une redirection).

- [ ] **Step 8.8: Implémenter les conditions de vente par segment**

`AdminContentService` expose `saleConditions(ctx, segment, locale)` (lecture) et `setSaleCondition(ctx, input)` (écriture) : chaque écriture crée une **nouvelle version** (`version + 1`) avec `effective_from`, sans écraser l'historique (le snapshot de commande conserve la version utilisée, spec §20.2).

- [ ] **Step 8.9: Tests verts**

```bash
pnpm --filter @fanours/api test src/modules/admin-content
```

Attendu :

```text
 ✓ src/modules/admin-content/admin-content.service.spec.ts (8 tests)
 Test Files  1 passed (1)
      Tests  8 passed (8)
```

- [ ] **Step 8.10: Pages admin contenu et redirections**

`admin/contenu/page.tsx` : `DataTable` (slug, type, statut, locales publiées, indexable, mise à jour), éditeur par page (onglets par locale : Contenu · Meta · Aperçu), champs `meta_title` / `meta_description` avec compteur de caractères et valeurs max affichées, bascule « indexable ».

`admin/contenu/redirections/page.tsx` : `DataTable` (source, destination, code, motif, auteur, date), formulaire de création avec validation côté serveur, suppression possible d'une redirection, message d'aide indiquant que la redirection définitive (301) est préférée et que les chaînes sont interdites.

- [ ] **Step 8.11: Vérifier le SEO des pages publiques**

Les pages publiées apparaissent dans le sitemap indexé par langue (`packages/seo` Phase 2), portent un `hreflang` complet + `x-default`, et un JSON-LD `Article` / `BreadcrumbList` fidèle au contenu visible. Les pages **non publiées** et les slugs historiques sont exclus du sitemap. Les pages admin et compte restent `noindex` (Task 2).

- [ ] **Step 8.12: Traduire et vérifier**

```bash
pnpm --filter @fanours/i18n check:keys
pnpm --filter @fanours/web build
pnpm exec playwright test e2e/seo/content.spec.ts --project=functional
```

Attendu :

```text
i18n key parity OK (3 locales, 664 clés)
✓ Built in 48.6s
  4 passed (15.8s)
```

**Preuves :**
- 8 tests unitaires (boucle refusée, chaîne refusée, 301 unique au changement de slug, aperçu `noindex`, publication bloquée sans traduction).
- `0115` appliquée avec checksum ; `content_page`, `content_page_translation`, `redirect_rule`, `sale_condition` créées.
- Le test de permissions de la Phase 1 est mis à jour dans le **même commit** (`content_page` ajoutée à `EXPECTED_RESOURCES`, 15 ressources au total) et passe.
- Après un changement de slug : une seule redirection 301 par locale indexable, l'ancien slug absent du sitemap, canonical sur la destination.
- Aucun avis client, note ou contenu social n'est généré : `aggregateRating` n'apparaît que si des avis vérifiés existent (spec §17.9).

**Commit :**

```bash
git add packages/db/migrations/0115_phase4_content_seo.sql packages/db/src/schema/phase4-content.ts packages/db/src/schema.ts packages/domain/src/permissions.ts packages/db/src/seed.ts apps/api/src/modules/rbac/__tests__/permissions.test.ts apps/api/src/modules/admin-content "apps/web/src/app/[locale]/(admin)/admin/contenu" packages/i18n/src/locales
git commit -m "feat(admin-content): CMS editorial, meta, slugs, redirections et apercus" -m "Implémente le chantier 4.8 (spec §16.10, §17) : pages éditoriales par locale avec meta title et description bornées, workflow draft/published, aperçu noindex, changement de slug générant une redirection 301 unique par locale indexable, exclusion de l'ancien slug du sitemap, redirections sans boucle ni chaîne, conditions de vente versionnées par segment." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 9: Support (4.9) — file de tickets, assignation, notes internes, macros, SLA, demande de remboursement

**Files:**
- Create: `apps/api/src/modules/support/admin-support.service.ts`
- Create: `e2e/admin/support.spec.ts` (recette fonctionnelle de phase — projet Playwright `functional`)
- Create: `apps/api/src/modules/support/admin-support.controller.ts`
- Create: `apps/api/src/modules/support/admin-support.module.ts`
- Test: `apps/api/src/modules/support/admin-support.service.spec.ts`
- Create: `packages/i18n/src/support-macros.ts`
- Create: `apps/web/src/app/[locale]/(admin)/admin/support/page.tsx`
- Create: `apps/web/src/components/admin/SupportThread.tsx`
- Modify: `packages/domain/src/permissions.ts`
- Modify: `apps/api/src/modules/rbac/__tests__/permissions.test.ts` (liste `EXPECTED_RESOURCES` — ajout de `support_ticket`)
- Modify: `packages/db/src/seed.ts`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `dispute` / `dispute_message` (Phase 3 — le ticket de support), `orders` / `order_line` (Phase 2), `shipment` / `tracking_event` (Phase 3), `payment` / `refund` (Phase 2), `admin_action_approval` (Phase 1), `audit_log` (Phase 1), `notification` (Phase 2), `packages/i18n` → `LOCALES`.
- Produces:
  - `AdminSupportService.listTickets(ctx: AdminContext, query: AdminTicketQueryDto): Promise<Paged<AdminTicketRowDto>>`
  - `AdminSupportService.detail(ctx: AdminContext, ticketId: string): Promise<AdminTicketDetailDto>`
  - `AdminSupportService.assign(ctx: AdminContext, ticketId: string, input: TicketAssignInputDto): Promise<AdminTicketDetailDto>`
  - `AdminSupportService.reply(ctx: AdminContext, ticketId: string, input: TicketReplyInputDto): Promise<AdminTicketDetailDto>`
  - `AdminSupportService.escalate(ctx: AdminContext, ticketId: string, input: TicketEscalationInputDto): Promise<AdminTicketDetailDto>`
  - `AdminSupportService.sla(ctx: AdminContext, ticketId: string): Promise<SupportSlaDto>`
  - `AdminSupportService.requestRefund(ctx: AdminContext, ticketId: string, input: SupportRefundRequestDto): Promise<{ approvalId: string }>`
  - `AdminSupportService.macros(ctx: AdminContext): Promise<SupportMacroDto[]>`
  - `SUPPORT_MACROS` (constante exportée par `packages/i18n/src/support-macros.ts`, 3 locales).
  - Routes : `GET /v1/admin/support/tickets`, `GET /v1/admin/support/tickets/:id`, `POST /v1/admin/support/tickets/:id/assign`, `POST /v1/admin/support/tickets/:id/replies`, `POST /v1/admin/support/tickets/:id/escalate`, `GET /v1/admin/support/tickets/:id/sla`, `POST /v1/admin/support/tickets/:id/refund-requests`, `GET /v1/admin/support/macros`.

> **Choix de modélisation** : cette tâche **ne crée aucune table**. Le ticket de support est la ligne `dispute` (Phase 3) qui porte déjà `category`, `priority`, `status`, `assigned_to`, `escalated_team`, et dont les échanges sont les lignes `dispute_message` avec `visibility` (`client` | `internal`). Le chantier 4.9 **complète** ce modèle (macros, SLA, escalade, demande de remboursement) au lieu de le dupliquer : une seule file de support, un seul historique, aucune divergence entre litige et ticket.

- [ ] **Step 9.1: Écrire le test rouge de l'assignation, de la visibilité et de la demande de remboursement**

`apps/api/src/modules/support/admin-support.service.spec.ts` :

```typescript
describe('AdminSupportService', () => {
  it('n'expose jamais une note interne au client', async () => {
    const ticket = await seedDispute(ctx.db, { status: 'open', priority: 'high' });

    const detail = await service.detail(ctx, ticket.id);
    const clientVisible = detail.messages.filter((m) => m.visibility === 'client');

    expect(clientVisible.every((m) => !m.body.includes('interne'))).toBe(true);
    expect(detail.messages.some((m) => m.visibility === 'internal')).toBe(true);
  });

  it('refuse au support agent de rembourser directement', async () => {
    const ticket = await seedDispute(ctx.db, { status: 'open' });

    await expect(
      service.requestRefund(ctxAsRole('support_agent'), ticket.id, {
        orderId,
        amountMinor: 5000,
        currency: 'EUR',
        reason: 'Article non conforme',
      }),
    ).rejects.toMatchObject({ status: 403, code: 'refund_not_allowed_for_role' });
  });

  it('transforme une demande de remboursement du support en approbation quatre-yeux', async () => {
    const ticket = await seedDispute(ctx.db, { status: 'open' });

    const { approvalId } = await service.requestRefund(ctxAsRole('support_agent'), ticket.id, {
      orderId,
      amountMinor: 5000,
      currency: 'EUR',
      reason: 'Article non conforme',
    });

    const approval = await readApproval(ctx.db, approvalId);
    expect(approval.kind).toBe('refund');
    expect(approval.requested_by).toBe(supportAgentId);
    expect(approval.approved_by).toBeNull();
  });

  it('calcule un statut SLA à partir de la priorité', async () => {
    const ticket = await seedDispute(ctx.db, { status: 'open', priority: 'high', createdAt: hoursAgo(30) });

    const sla = await service.sla(ctx, ticket.id);

    expect(sla.targetHours).toBe(8);
    expect(sla.state).toBe('breached');
  });
});
```

- [ ] **Step 9.1b: Écrire le test E2E rouge de la recette fonctionnelle du support**

`e2e/admin/support.spec.ts` — recette fonctionnelle de phase, projet Playwright `functional`. Écrit **avant** l'implémentation : il échoue à ce stade et passe à l'étape « Traduire et vérifier ».

Cas et assertions :

1. La file de tickets affiche l'âge du SLA, l'assignation et la priorité ; les **notes internes ne sont pas** rendues dans l'espace client.
2. Une demande de remboursement issue du support déclenche la confirmation du prestataire et reste **idempotente** : un second envoi ne produit pas un second remboursement (un seul effet visible, journalisé une fois).
3. Un ticket ne peut pas être assigné ni annoté par un rôle sans la permission `support:write` : `403`, message localisé, action tracée.
4. Une macro de réponse s'exécute sans laisser de variable de template non résolue dans le message envoyé au client.

- [ ] **Step 9.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/api test src/modules/support/admin-support.service.spec.ts
```

Attendu :

```text
 FAIL  src/modules/support/admin-support.service.spec.ts
Error: Cannot find module './admin-support.service.js'
Test Files  1 failed (1)
     Tests  0 passed | 4 failed (4)
```

- [ ] **Step 9.3: Étendre la matrice de permissions**

`packages/domain/src/permissions.ts` : ajout de la ressource `support_ticket` avec les actions `read`, `create`, `update`, `assign`, `escalate`, `refund_request`.

- `support_agent` : `support_ticket: ['read', 'create', 'update', 'assign', 'escalate', 'refund_request']` ;
- `order_manager` : `support_ticket: ['read', 'update', 'assign', 'escalate']` ;
- `finance_manager` : `support_ticket: ['read']` ;
- `super_admin` : `support_ticket: ['read', 'create', 'update', 'delete', 'assign', 'escalate', 'refund_request']` ;
- `analyst_readonly` : `support_ticket: ['read']`.

`packages/db/src/seed.ts` : lignes `permission` correspondantes. La ressource `support_ticket` est déjà autorisée par la contrainte `ck_permission_resource` étendue en migration `0113` (Task 4), appliquée avant cette tâche.

**Mise à jour du test de permissions de la Phase 1** — `apps/api/src/modules/rbac/__tests__/permissions.test.ts` : `support_ticket` est ajoutée à `EXPECTED_RESOURCES` dans le **même commit**.

```typescript
const EXPECTED_RESOURCES = [
  'order', 'product', 'price_tier', 'export', 'export_financial', 'refund', 'admin_role',
  'audit_log', 'customs_rule', 'country', 'business_document', 'business_account',
  'customs_document', 'stock', 'content_page', 'support_ticket',   // <- ajouté en Task 9
] as const;
```

**Décompte à ce stade : 16 ressources (15 après la Task 8 + `support_ticket`) et 13 actions (inchangées depuis la Task 4).** Les actions `assign`, `escalate` et `refund_request` appartiennent déjà au vocabulaire déclaré en Task 4 : ni `ACTIONS` ni `EXPECTED_ACTIONS` ne changent ici.

Deux invariantes du test de la Phase 1 restent **inchangées** et doivent le rester : `PERMISSIONS['support_agent']?.['refund']` reste `['read', 'create']` (la demande de remboursement du support passe par `support_ticket:refund_request`, pas par la ressource `refund`), et `PERMISSIONS['catalog_manager']?.['admin_role']` reste `undefined`.

```bash
pnpm --filter @fanours/api test src/modules/rbac/__tests__/permissions.test.ts
```

Attendu :

```text
 ✓ src/modules/rbac/__tests__/permissions.test.ts (6 tests)
 Test Files  1 passed (1)
      Tests  6 passed (6)
```

- [ ] **Step 9.4: Implémenter la file de tickets**

`AdminTicketQueryDto` : `{ status?, priority?, category?, assignedTo?, team?, country?, orderNumber?, q?, limit?, cursor? }`.

`listTickets()` : scoping par rôle appliqué **dans la clause `WHERE`** — un `support_agent` voit par défaut les tickets qui lui sont assignés plus ceux non assignés de son équipe ; un `order_manager` voit les tickets liés à ses commandes ; `finance_manager` et `super_admin` voient tout. Pagination par curseur sur `(updated_at, id)`.

`detail()` : ticket (catégorie, priorité, statut, assignataire, équipe, dates), **commandes liées** (numéro, statut, montant), **expéditions liées** (transporteur, tracking, statut), historique complet des messages (internes et clients, avec leur visibilité), et les actions proposées (remboursement, avoir, relance) dont la disponibilité dépend du rôle et du statut de la commande.

- [ ] **Step 9.5: Implémenter assignation, escalade et réponses**

`assign()` : affecte `assigned_to` et `escalated_team`, journalise `support_ticket.assigned`, notifie l'agent assigné (canal centre de notifications).
`escalate()` : passe la priorité à `urgent` si l'échéance est dépassée, journalise `support_ticket.escalated`, notifie le responsable d'équipe.
`reply()` : insère une ligne `dispute_message` avec `visibility` (`client` ou `internal`), journalise `support_ticket.replied` (jamais le corps du message dans `metadata`), et crée la notification correspondante (in-app + e-mail si la réponse est destinée au client).

**Macros** : `packages/i18n/src/support-macros.ts` exporte `SUPPORT_MACROS`, un catalogue versionné de réponses pré-écrites, **localisé dans les 3 langues** (FR/EN/ZH), avec `id`, `category`, `body` par locale et une **interpolation** des variables autorisées (`{{orderNumber}}`, `{{trackingNumber}}`, `{{refundAmount}}`). Une macro ne peut pas contenir de donnée personnalisée non explicitement listée ; l'insertion se fait par choix puis édition avant envoi (jamais d'envoi automatique d'une macro).

- [ ] **Step 9.6: Implémenter le SLA**

`SupportSlaDto` : `{ targetHours, remainingHours, state: 'ok' | 'warning' | 'breached', startedAt, dueAt }`.

Les délais cibles par priorité sont des **constantes documentées** : `critical: 4 h`, `high: 8 h`, `normal: 24 h`, `low: 72 h` — **à valider (décision propriétaire)** : ces valeurs ne sont pas fixées par le spec et sont regroupées dans `packages/config/src/support-policy.ts` pour être modifiables sans toucher au code métier. Le calcul utilise `dispute.created_at` (début du timer) et s'arrête à la première réponse client (`dispute_message` avec `visibility = 'client'`).

- [ ] **Step 9.7: Implémenter la demande de remboursement depuis le support**

Le `support_agent` **ne rembourse pas** : il **demande**. `requestRefund()` :

1. Vérifie que la commande est dans le périmètre du ticket et qu'un paiement `succeeded` existe.
2. Vérifie que le montant demandé ne dépasse pas le montant remboursable restant (contrôle serveur sur `payment.amount - SUM(refund.amount)`).
3. Crée une demande `admin_action_approval` (kind `refund`, `amount`, `currency`, `payload` = `{ orderId, ticketId, reason }`, `payload_hash` = SHA-256, `expires_at = now() + 24 h`).
4. **Ne rembourse rien** : le remboursement effectif est exécuté par `finance_manager` via le moteur `sensitive-action` de la Task 13.
5. Journalise `support.refund_requested`.

Le double remboursement reste impossible : la clé d'idempotence du remboursement (Phase 2) est dérivée de `(paymentId, amountMinor, ticketId)` et la somme des remboursements est revérifiée à l'exécution.

- [ ] **Step 9.8: Tests verts**

```bash
pnpm --filter @fanours/api test src/modules/support
```

Attendu :

```text
 ✓ src/modules/support/admin-support.service.spec.ts (9 tests)
 Test Files  1 passed (1)
      Tests  9 passed (9)
```

- [ ] **Step 9.9: Créer la page admin support**

`admin/support/page.tsx` : 2 colonnes — file de tickets à gauche (filtres : statut, priorité, équipe, assignataire, pays, recherche), conversation à droite (`SupportThread`). La conversation sépare visuellement les messages **internes** (bandeau « Note interne — non visible par le client ») des messages clients. En-tête du ticket : catégorie, priorité, SLA (avec état textuel « dans les délais » / « à échéance » / « dépassé »), assignataire, équipe, commandes et expéditions liées. Panneau d'actions : assigner, escalader, répondre (choix de macro + édition), demander un remboursement (montant, motif, devise).

Tous les libellés sont traduits dans les 3 langues ; la macro choisie est déjà dans la langue de la réponse.

- [ ] **Step 9.10: Traduire et vérifier**

```bash
pnpm --filter @fanours/i18n check:keys
pnpm --filter @fanours/web build
pnpm exec playwright test e2e/admin/support.spec.ts --project=functional
```

Attendu :

```text
i18n key parity OK (3 locales, 690 clés)
Built in 48.1s
  3 passed (12.7s)
```

**Preuves :**
- 9 tests unitaires (visibilité interne, refus de remboursement direct par le support, demande transformée en approbation, calcul SLA, assignation, escalade, macro localisée).
- Un `support_agent` ne peut pas déclencher de remboursement : `403 refund_not_allowed_for_role` ; seule une **demande** d'approbation est créée.
- Une réponse client ne contient jamais une note interne (vérifié sur le DTO rendu).
- `SUPPORT_MACROS` contient des entrées dans les 3 langues et le test de parité des clés i18n passe.
- Le test de permissions de la Phase 1 est mis à jour dans le **même commit** (`support_ticket` ajoutée à `EXPECTED_RESOURCES`, 16 ressources au total) et passe ; les invariants `support_agent.refund === ['read','create']` et `catalog_manager.admin_role === undefined` restent valides.

**Commit :**

```bash
git add apps/api/src/modules/support packages/i18n/src/support-macros.ts "apps/web/src/app/[locale]/(admin)/admin/support" apps/web/src/components/admin/SupportThread.tsx packages/domain/src/permissions.ts packages/db/src/seed.ts apps/api/src/modules/rbac/__tests__/permissions.test.ts packages/i18n/src/locales
git commit -m "feat(support): file de tickets, assignation, macros, SLA et demande de remboursement" -m "Implémente le chantier 4.9 (spec §16.11, §12.5) : file de tickets avec scoping par rôle, assignation par équipe, escalade, notes internes et réponses client séparées, macros localisées FR/EN/ZH, calcul de SLA par priorité, demande de remboursement transformée en approbation quatre-yeux sans exécution par le support. Réutilise dispute et dispute_message de la phase 3 sans table supplémentaire." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 10: Journal d'audit et traçabilité (4.10) — recherche, chaîne d'intégrité, export tracé

**Files:**
- Create: `apps/api/src/modules/audit/audit.service.ts`
- Create: `e2e/admin/audit.spec.ts` (recette fonctionnelle de phase — projet Playwright `functional`)
- Create: `apps/api/src/modules/audit/audit.controller.ts`
- Create: `apps/api/src/modules/audit/audit.module.ts`
- Test: `apps/api/src/modules/audit/audit.service.spec.ts`
- Create: `apps/web/src/app/[locale]/(admin)/admin/journal/page.tsx`
- Create: `apps/web/src/components/admin/AuditTimeline.tsx`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `audit_log` (Phase 1, append-only, chaîne SHA-256), `export_request` (migration `0112`, Task 3), `compliance_alert` (Phase 3), `packages/domain` → `AUDIT_ACTION_CATALOG` (Task 13).
- Produces:
  - `AuditService.search(ctx: AdminContext, query: AuditQueryDto): Promise<Paged<AuditEntryDto>>`
  - `AuditService.get(ctx: AdminContext, id: string): Promise<AuditEntryDetailDto>`
  - `AuditService.trace(ctx: AdminContext, input: AuditTraceInputDto): Promise<AuditEntryDto[]>`
  - `AuditService.verifyChain(ctx: AdminContext, range: AuditChainRangeDto): Promise<AuditChainVerificationDto>`
  - `AuditService.export(ctx: AdminContext, query: AuditQueryDto): Promise<{ exportId: string }>`
  - `AuditService.stats(ctx: AdminContext, query: AuditStatsQueryDto): Promise<AuditStatsDto>`
  - Routes : `GET /v1/admin/audit/entries`, `GET /v1/admin/audit/entries/:id`, `GET /v1/admin/audit/trace`, `GET /v1/admin/audit/verify`, `POST /v1/admin/audit/exports`, `GET /v1/admin/audit/stats`.

> **Choix de modélisation** : cette tâche **ne crée aucune table**. Le journal est `audit_log`, déjà protégée en Phase 1 par le déclencheur d'immuabilité (bloque `UPDATE`/`DELETE`) et la chaîne de hash SHA-256. La phase 4 ajoute la **consultation**, la **vérification d'intégrité**, la **traçabilité par objet** et l'**export tracé** — pas de réécriture du stockage.

- [ ] **Step 10.1: Écrire le test rouge de l'immuabilité, de la chaîne et de l'export**

`apps/api/src/modules/audit/audit.service.spec.ts` :

```typescript
describe('AuditService', () => {
  it('refuse la modification et la suppression d’une entrée du journal', async () => {
    const entry = await readAudit(ctx.db, { limit: 1 });

    await expect(ctx.db.execute(sql`UPDATE audit_log SET result = 'success' WHERE id = ${entry[0].id}`))
      .rejects.toThrow(/append-only/);
    await expect(ctx.db.execute(sql`DELETE FROM audit_log WHERE id = ${entry[0].id}`))
      .rejects.toThrow(/append-only/);
  });

  it('vérifie la chaîne d’intégrité et signale une rupture', async () => {
    const result = await service.verifyChain(ctx, { from: firstId, to: lastId });

    expect(result.entriesChecked).toBeGreaterThan(0);
    expect(result.broken).toBe(false);
  });

  it('filtre par acteur, action, objet et fenêtre de dates', async () => {
    const page = await service.search(ctx, { actorId, action: 'order.transition', from: '2026-10-01', to: '2026-10-31' });

    expect(page.items.every((e) => e.action.startsWith('order.transition'))).toBe(true);
    expect(page.items.every((e) => e.actorId === actorId)).toBe(true);
  });

  it('refuse l’export du journal à un rôle non autorisé', async () => {
    await expect(service.export(ctxAsRole('support_agent'), { from: '2026-10-01', to: '2026-10-31' }))
      .rejects.toMatchObject({ status: 403, code: 'forbidden' });
  });
});
```

- [ ] **Step 10.1b: Écrire le test E2E rouge de la recette fonctionnelle du journal d'audit**

`e2e/admin/audit.spec.ts` — recette fonctionnelle de phase, projet Playwright `functional`. Écrit **avant** l'implémentation : il échoue à ce stade et passe à l'étape « Traduire et vérifier ».

Cas et assertions :

1. La recherche du journal filtre par acteur, ressource, action et période ; une action sensible (prix, stock, rôle, export, remboursement) apparaît avec son horodatage et son `correlation_id`.
2. Le journal est **append-only** : toute tentative de modification ou de suppression d'une entrée est refusée (`405`/`403`) et elle-même tracée.
3. L'export du journal est servi par une **URL signée de 5 minutes** dans le bucket privé ; l'export lui-même est inscrit dans le journal.
4. Le rôle `analyst_readonly` lit le journal sans pouvoir l'exporter : `403` et tentative tracée.

- [ ] **Step 10.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/api test src/modules/audit/audit.service.spec.ts
```

Attendu :

```text
 FAIL  src/modules/audit/audit.service.spec.ts
Error: Cannot find module './audit.service.js'
Test Files  1 failed (1)
     Tests  0 passed | 4 failed (4)
```

- [ ] **Step 10.3: Implémenter la recherche**

`AuditQueryDto` : `{ from?, to?, actorId?, actorRole?, action?, objectType?, objectId?, result?, correlationId?, limit (défaut 50, max 200), cursor? }`.

- Rôles autorisés à lire le journal : `super_admin`, `finance_manager`, `compliance_manager` (matrice sécurité §3.2, ligne 9). Les autres rôles reçoivent `403`.
- Pagination par **curseur** sur `(id DESC)` : l'identifiant est monotone et la chaîne de hash s'y rattache.
- Le filtre `action` est une recherche par **préfixe** (`action LIKE 'order.transition%'`) ; le filtre `objectId` est exact.
- Chaque entrée renvoyée contient : `id`, `occurredAt`, `actorId`, `actorRole`, `action`, `objectType`, `objectId`, `result`, `correlationId`, `metadata`.
- **Aucune valeur de `metadata` n'est un secret** : la rédaction est appliquée par une liste blanche de clés (`transitionId`, `status`, `quantity`, `reason`, `ownerId`, `runId`, `exportId`, `lineId`, `limit`, `basis`, `provider`) ; toute autre clé est remplacée par `'[redacted]'` avant sérialisation.

- [ ] **Step 10.4: Implémenter la vérification d'intégrité**

`verifyChain()` recalcule le SHA-256 de chaque entrée (charge utile canonique : `id`, `occurred_at`, `actor_id`, `actor_role`, `action`, `object_type`, `object_id`, `result`, `correlation_id`, `metadata`, `prev_hash`) et le compare à `entry_hash`, puis vérifie que `prev_hash` correspond au `entry_hash` de l'entrée précédente. Résultat : `{ entriesChecked, broken, brokenAtId, checkedAt }`. Un contrôle programmé quotidien (job `audit.verify`) crée une `compliance_alert` de sévérité `blocker` en cas de rupture ; le canal « revue de règle » (infra §11.5) sert à ce contrôle.

- [ ] **Step 10.5: Implémenter la traçabilité par objet**

`trace({ objectType, objectId })` renvoie l'historique complet d'un objet (commande, produit, règle douanière, compte B2B, ticket), trié par `occurred_at`, avec l'auteur, l'action et le résultat. Cette vue est celle utilisée par la Task 2 (détail commande), la Task 7 (historique d'une règle) et la Task 13 (preuve de l'action sensible).

- [ ] **Step 10.6: Implémenter l'export tracé**

`export()` : réservé à `super_admin`, `finance_manager`, `compliance_manager` ; MFA récente exigée ; crée une ligne `export_request` (`kind = 'audit'`, `idempotency_key = audit.export:<actorId>:<sha256(query)>`), publie l'outbox `export.requested` ; le job `export.data` (Task 6) écrit le CSV dans le bucket privé, URL signée 5 min, `expires_at = now() + 24 h` (durée de conservation — **à valider (décision propriétaire)**). L'export est journalisé (`audit.export.requested`, `audit.export.executed`) : l'événement `data_export` de la spec §19.7 est donc automatiquement tracé.

- [ ] **Step 10.7: Tests verts**

```bash
pnpm --filter @fanours/api test src/modules/audit
```

Attendu :

```text
 ✓ src/modules/audit/audit.service.spec.ts (8 tests)
 Test Files  1 passed (1)
      Tests  8 passed (8)
```

- [ ] **Step 10.8: Créer la page admin journal**

`admin/journal/page.tsx` : `FilterBar` (fenêtre de dates, acteur, rôle, action, type d'objet, objet, résultat, `correlation_id`), `DataTable` horodatée (heure locale + UTC, acteur, action, objet, résultat avec **icône + texte**, `correlation_id` copiable), `AuditTimeline` (vue chronologique d'un objet), panneau de vérification d'intégrité (bouton « Vérifier la chaîne », résultat `intègre` / `rupture` avec l'identifiant de l'entrée fautive), bouton « Exporter » (demande d'approbation pour les rôles non financiers).

- [ ] **Step 10.9: Traduire et vérifier**

```bash
pnpm --filter @fanours/i18n check:keys
pnpm --filter @fanours/web build
pnpm exec playwright test e2e/admin/audit.spec.ts --project=functional
```

Attendu :

```text
i18n key parity OK (3 locales, 712 clés)
Built in 49.0s
  3 passed (13.9s)
```

**Preuves :**
- 8 tests unitaires (immuabilité `UPDATE`/`DELETE`, vérification de chaîne, filtres, refus d'export par rôle, rédaction des clés de métadonnées inconnues).
- Un `UPDATE` ou un `DELETE` sur `audit_log` est rejeté **par la base** (message `audit_log is append-only`), pas seulement par l'application.
- La vérification de chaîne confirme `broken = false` sur la base de test et détecte une rupture après altération manuelle d'une entrée (test sur base jetable uniquement).
- Un export du journal est refusé à un `support_agent` (`403`).

**Commit :**

```bash
git add apps/api/src/modules/audit "apps/web/src/app/[locale]/(admin)/admin/journal" apps/web/src/components/admin/AuditTimeline.tsx packages/i18n/src/locales
git commit -m "feat(audit): recherche, verification d'integrite, tracabilite et export trace" -m "Implémente le chantier 4.10 (spec §19.7) : consultation filtrée du journal append-only, vérification de la chaîne SHA-256, traçabilité par objet, export CSV tracé et signé pour les rôles autorisés, rédaction par liste blanche des clés de métadonnées. Aucune table ajoutée : le journal existant en phase 1 reste la source de vérité." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 11: Analytics (4.11) — indicateurs définis, sourcés, non mélangés, indisponibles plutôt qu'inventés

**Files:**
- Create: `packages/analytics/package.json`
- Create: `packages/analytics/src/index.ts`
- Create: `packages/analytics/src/kpi-definitions.ts`
- Create: `packages/analytics/src/kpi-computation.ts`
- Test: `packages/analytics/src/kpi-definitions.test.ts`
- Create: `apps/api/src/modules/analytics/analytics.service.ts`
- Create: `apps/api/src/modules/analytics/analytics.controller.ts`
- Create: `apps/api/src/modules/analytics/analytics.module.ts`
- Test: `apps/api/src/modules/analytics/analytics.service.spec.ts`
- Create: `packages/ui/src/components/TrendChart.tsx`
- Test: `packages/ui/src/components/TrendChart.test.tsx`
- Create: `e2e/admin/analytics.spec.ts`
- Modify: `packages/ui/src/components/index.ts`
- Create: `apps/web/src/app/[locale]/(admin)/admin/analytics/page.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/page.tsx`
- Modify: `packages/domain/src/permissions.ts`
- Modify: `apps/api/src/modules/rbac/__tests__/permissions.test.ts` (liste `EXPECTED_RESOURCES` — ajout de `analytics`)
- Modify: `packages/db/src/seed.ts`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `orders` / `order_line` / `order_status_event` (Phase 2), `payment` / `refund` (Phase 2), `shipment` / `tracking_event` (Phase 3), `inventory_item` (Phase 2), `dispute` (Phase 3).
- Produces:
  - `packages/analytics` → `KPI_DEFINITIONS: readonly KpiDefinition[]`, `KpiDefinition`, `KpiValue`, `KpiBasis`, `KpiAvailability`, `computeKpi(name, range, deps)`, `assertNoMixedBasis(definitions)`.
  - `KpiDefinition = { key, group, basis?: KpiBasis, inputs?: KpiBasis[], unit: 'currency' | 'count' | 'duration' | 'percent', formula: string, recognitionDate: string, refundsIncluded: boolean, sourceTables: string[], availability: KpiAvailability }`
  - `KpiBasis = 'gross_revenue' | 'orders_created' | 'cash_received' | 'refunds' | 'inventory' | 'operations'`
  - `KpiAvailability = 'available' | 'requires_analytics_provider' | 'requires_cost_data'`
  - `KpiValue = { key, state: 'ok' | 'unavailable', valueMinor?: number, value?: number, currency?: string, definition: KpiDefinition, freshnessAt: string, computedAt: string }`
  - `AnalyticsService.overview(ctx: AdminContext, query: AnalyticsQueryDto): Promise<AnalyticsOverviewDto>`
  - `AnalyticsService.kpi(ctx: AdminContext, key: string, query: AnalyticsQueryDto): Promise<KpiValue>`
  - `AnalyticsService.timeseries(ctx: AdminContext, key: string, query: AnalyticsSeriesQueryDto): Promise<KpiSeriesDto>`
  - Routes : `GET /v1/admin/analytics/overview`, `GET /v1/admin/analytics/kpi/:key`, `GET /v1/admin/analytics/series/:key`.

- [ ] **Step 11.1: Écrire le test rouge des définitions de KPI**

`packages/analytics/src/kpi-definitions.test.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { KPI_DEFINITIONS, assertNoMixedBasis } from './kpi-definitions.js';

describe('KPI_DEFINITIONS', () => {
  it('déclare pour chaque KPI une formule, une date de reconnaissance et ses tables sources', () => {
    for (const def of KPI_DEFINITIONS) {
      expect(def.formula.length).toBeGreaterThan(10);
      expect(def.recognitionDate).toMatch(/^\d{4}-\d{2}-\d{2}$/);
      expect(def.sourceTables.length).toBeGreaterThan(0);
    }
  });

  it('ne mélange jamais chiffre d’affaires, commandes et encaissements dans un même total', () => {
    expect(() => assertNoMixedBasis(KPI_DEFINITIONS)).not.toThrow();
  });

  it('marque la marge brute et le taux de conversion comme indisponibles sans données', () => {
    const margin = KPI_DEFINITIONS.find((d) => d.key === 'gross_margin_rate');
    const conversion = KPI_DEFINITIONS.find((d) => d.key === 'conversion_rate');

    expect(margin?.availability).toBe('requires_cost_data');
    expect(conversion?.availability).toBe('requires_analytics_provider');
  });
});
```

- [ ] **Step 11.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/analytics test
```

Attendu :

```text
 FAIL  src/kpi-definitions.test.ts
Error: Cannot find module './kpi-definitions.js'
Test Files  1 failed (1)
     Tests  0 passed | 3 failed (3)
```

- [ ] **Step 11.3: Créer le package `packages/analytics`**

`packages/analytics/package.json` :

```json
{
  "name": "@fanours/analytics",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": { ".": "./src/index.ts" },
  "scripts": {
    "test": "vitest run",
    "typecheck": "tsc --noEmit"
  },
  "devDependencies": {
    "typescript": "^5.7.2",
    "vitest": "2.1.8"
  }
}
```

Le dossier `packages/*` est déjà couvert par `pnpm-workspace.yaml` ; le pipeline `test` du package est déclaré dans `turbo.json` comme les autres packages métier.

- [ ] **Step 11.4: Écrire les définitions de KPI**

`packages/analytics/src/kpi-definitions.ts` — 12 indicateurs, chacun avec sa définition écrite (aucune valeur inventée) :

| Clé | Libellé | Base / entrées | Formule | Date de reconnaissance | Remboursements | Tables sources | Disponibilité |
|---|---|---|---|---|---|---|---|
| `gross_revenue` | Chiffre d'affaires | `gross_revenue` | Somme des montants des commandes payées sur la période | Date de paiement du prestataire | Non | `orders`, `payment` | available |
| `orders_created` | Commandes créées | `orders_created` | Nombre de commandes créées sur la période | Date de création de la commande | Sans objet | `orders` | available |
| `average_order_value` | Panier moyen | entrées `gross_revenue`, `orders_created` | `gross_revenue / orders_created` | Date de paiement | Non | `orders`, `payment` | available |
| `cash_received` | Encaissements nets | `cash_received` | Encaissements TTC moins frais prestataire moins remboursements | Date de règlement | Oui (déduits) | `payment`, `refund` | available |
| `refunds_rate` | Taux de remboursement | entrées `gross_revenue`, `refunds` | `SUM(refund) / gross_revenue` | Date de remboursement | Oui | `refund`, `payment` | available |
| `stockout_rate` | Taux de rupture | `inventory` | Part des variantes commandables dont le disponible est nul sur la période | Date de la commande | Sans objet | `inventory_item`, `orders` | available |
| `order_to_ship_hours` | Délai de préparation | `operations` | Médiane des heures entre `paid` et `shipped` | Date d'expédition | Sans objet | `orders`, `order_status_event`, `shipment` | available |
| `delivery_days` | Délai de livraison | `operations` | Médiane des jours entre `shipped` et `delivered` | Date de livraison | Sans objet | `shipment`, `tracking_event` | available |
| `late_order_rate` | Commandes en retard | `operations` | Part des commandes livrées après la date promise | Date de livraison | Sans objet | `orders`, `shipment` | available |
| `open_disputes_rate` | Litiges ouverts | `operations` | Nombre de tickets ouverts pour 100 commandes créées | Date d'ouverture du ticket | Sans objet | `dispute`, `orders` | available |
| `gross_margin_rate` | Marge brute estimée | — | `(revenus - coûts d'achat) / revenus` | Date d'achat | Oui | — | `requires_cost_data` |
| `conversion_rate` | Taux de conversion | — | Sessions divisées par commandes | Date de session | Sans objet | — | `requires_analytics_provider` |

`gross_margin_rate` et `conversion_rate` sont **déclarés indisponibles** : aucun coût d'achat n'est enregistré dans le modèle actuel et aucun outil analytics tiers n'est choisi (décision propriétaire, spec §28.2). L'API renvoie `state: 'unavailable'` avec la raison — **jamais 0, jamais une estimation inventée**.

`assertNoMixedBasis()` échoue si deux KPI additifs de bases différentes (`gross_revenue`, `orders_created`, `cash_received`) sont présentés dans un même total ; les KPI de ratio déclarent explicitement leurs `inputs` et ne sont jamais additionnés entre eux.

- [ ] **Step 11.5: Implémenter le calcul des KPI**

`packages/analytics/src/kpi-computation.ts` : `computeKpi(name, range, deps)` reçoit un objet `deps` d'accès base injecté (le package ne dépend pas de Drizzle : il reçoit des fonctions de requêtes, ce qui le rend testable unitairement).

Chaque KPI renvoie un `KpiValue` :

- `state: 'ok'` avec `valueMinor` (entier de minor units) ou `value` (nombre pour durées et pourcentages, arrondi à 2 décimales) ;
- `state: 'unavailable'` avec la raison quand `availability !== 'available'` ;
- `definition` embarquée (formule, base, date de reconnaissance, tables sources) ;
- `freshnessAt` = `MAX(updated_at)` des tables sources de la période ; `computedAt` = horodatage du calcul.

Cache Redis de 60 secondes par (clé, période, filtres) ; invalidation sur événement outbox `order.paid`, `order.shipped`, `order.delivered`, `refund.succeeded`.

- [ ] **Step 11.6: Implémenter le service analytics**

`AnalyticsService.overview()` renvoie la liste des 12 KPI avec leur définition, dans l'ordre des groupes (Activité · Revenus · Opérations · Qualité). Les filtres (période, pays, devise, segment, catégorie) sont appliqués **dans la requête** ; la devise d'affichage est une conversion optionnelle marquée « indicatif » ; la devise de calcul reste celle de la transaction.

`AnalyticsService.timeseries()` renvoie une série journalière pour un KPI donné (uniquement les KPI à base temporelle `gross_revenue`, `orders_created`, `cash_received`).

- [ ] **Step 11.7: Étendre la matrice de permissions**

`packages/domain/src/permissions.ts` : ressource `analytics` avec l'action `read` ; `analyst_readonly`, `order_manager`, `finance_manager`, `marketing_manager`, `super_admin` l'obtiennent (lecture seule : aucun export automatique, l'export passant par le workflow d'export de la Task 6).

**Mise à jour du test de permissions de la Phase 1** — `apps/api/src/modules/rbac/__tests__/permissions.test.ts` : `analytics` est ajoutée à `EXPECTED_RESOURCES` dans le **même commit**.

```typescript
const EXPECTED_RESOURCES = [
  'order', 'product', 'price_tier', 'export', 'export_financial', 'refund', 'admin_role',
  'audit_log', 'customs_rule', 'country', 'business_document', 'business_account',
  'customs_document', 'stock', 'content_page', 'support_ticket',
  'analytics',                                            // <- ajouté en Task 11
] as const;
```

**Décompte à ce stade : 17 ressources (16 après la Task 9 + `analytics`) et 13 actions (inchangées depuis la Task 4).**

- [ ] **Step 11.8: Tests verts**

```bash
pnpm --filter @fanours/analytics test
pnpm --filter @fanours/api test src/modules/analytics
```

Attendu :

```text
 ✓ src/kpi-definitions.test.ts (3 tests)
 ✓ src/modules/analytics/analytics.service.spec.ts (6 tests)
 Test Files  2 passed (2)
      Tests  9 passed (9)
```

- [ ] **Step 11.9: Écrire le test rouge de `TrendChart`**

`packages/ui/src/components/TrendChart.test.tsx` :

```tsx
import { describe, expect, it } from 'vitest';
import { render, screen } from '@testing-library/react';
import { TrendChart } from './TrendChart.js';

const SERIES = [
  { key: 'ca', label: 'CA encaissé', points: [{ t: '2026-01', v: 100 }, { t: '2026-02', v: 140 }] },
  { key: 'cm', label: 'Commandes', points: [{ t: '2026-01', v: 12 }, { t: '2026-02', v: 15 }] },
];

describe('TrendChart', () => {
  it('rend une courbe par série avec un libellé direct, sans dépendre de la seule couleur', () => {
    render(<TrendChart series={SERIES} currency="EUR" />);

    for (const serie of SERIES) {
      expect(screen.getByText(serie.label)).toBeInTheDocument();
    }
    // chaque série dispose d'un marqueur distinct, pas seulement d'une teinte
    expect(document.querySelectorAll('[data-marker="circle"]').length).toBe(1);
    expect(document.querySelectorAll('[data-marker="square"]').length).toBe(1);
  });

  it('utilise uniquement les tokens CSS de la palette catégorielle', () => {
    const { container } = render(<TrendChart series={SERIES} currency="EUR" />);
    const strokes = [...container.querySelectorAll('path[data-series]')].map((n) =>
      n.getAttribute('stroke'),
    );
    expect(strokes).toEqual(['var(--color-chart-1)', 'var(--color-chart-2)']);
  });

  it('interrompt la courbe d’une série qui n’a pas de donnée sur une période', () => {
    const withGap = [
      { key: 'ca', label: 'CA', points: [{ t: '2026-01', v: 100 }, { t: '2026-03', v: 160 }] },
    ];
    const { container } = render(<TrendChart series={withGap} currency="EUR" />);
    const d = container.querySelector('path[data-series="ca"]')?.getAttribute('d') ?? '';
    // un segment comblé relierait 2026-01 à 2026-03 : aucun point intermédiaire ne doit exister
    expect(d).not.toContain('2026-02');
  });

  it('affiche un état « indisponible » avec la raison plutôt qu’une courbe vide ou un zéro', () => {
    render(<TrendChart series={[]} currency="EUR" unavailableReason="provider analytics non configuré" />);
    expect(screen.getByText(/indisponible/i)).toBeInTheDocument();
    expect(screen.getByText(/provider analytics non configuré/i)).toBeInTheDocument();
    expect(screen.queryByTestId('trend-chart-plot')).not.toBeInTheDocument();
  });

  it('expose un tableau de données équivalent pour les lecteurs d’écran', () => {
    render(<TrendChart series={SERIES} currency="EUR" />);
    expect(screen.getByRole('table')).toBeInTheDocument();
    expect(screen.getByRole('columnheader', { name: /2026-01/ })).toBeInTheDocument();
  });
});
```

- [ ] **Step 11.10: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/ui test:unit src/components/TrendChart.test.tsx
```

Attendu :

```text
 FAIL  src/components/TrendChart.test.tsx > TrendChart > rend une courbe par série…
Error: Failed to resolve import "./TrendChart.js"
 Test Files  1 failed (1)
      Tests  5 failed (5)
```

- [ ] **Step 11.11: Créer le composant `TrendChart`**

`packages/ui/src/components/TrendChart.tsx` : courbe SVG sans dépendance externe, alimentée par les séries du serveur ; couleurs issues des tokens CSS uniquement — palette catégorielle `--color-chart-1`..`--color-chart-3` dans l'ordre des séries (design.md §3.2.4), **3 séries maximum** par graphique (au-delà, le composant refuse d'afficher plus de 3 séries et le signale) ; chaque série porte un **libellé direct** en bout de courbe et un **marqueur distinct** (`data-marker="circle" | "square" | "triangle"`) — l'information n'est jamais transmise par la couleur seule (WCAG 1.4.1) ; grille et axes en `--color-chart-grid`, jamais en couleur de série ; une série sans donnée sur une période **interrompt** la courbe (aucun segment comblé, aucune interpolation) ; si toutes les séries sont indisponibles, le composant affiche un état « indisponible » avec la raison ; axe des ordonnées avec format `Intl` de la devise ; **tableau de données équivalent** rendu sous le graphique (`<table>` avec caption) pour les lecteurs d'écran ; `prefers-reduced-motion` respecté (pas d'animation de tracé).

- [ ] **Step 11.12: Tests verts du composant**

```bash
pnpm --filter @fanours/ui test:unit src/components/TrendChart.test.tsx
```

Attendu :

```text
 ✓ src/components/TrendChart.test.tsx (5 tests)
 Test Files  1 passed (1)
      Tests  5 passed (5)
```

- [ ] **Step 11.13: Créer les pages analytics et la vue globale admin**

`admin/analytics/page.tsx` : filtres (période, pays, devise, segment, catégorie), 4 groupes de KPI, chaque carte `KpiCard` affiche la valeur **ou** l'état « indisponible » avec la raison, la définition (date de reconnaissance, formule, base) dans l'infobulle et dans un panneau « Définitions » dédié, `TrendChart` pour les trois KPI temporelles.

`admin/page.tsx` (vue globale admin) : synthèse (commandes du jour, CA encaissé, alertes ouvertes, tickets en attente, règles à revalider) construite **uniquement** à partir des KPI définis et des services des Tasks 4, 7, 9 — aucune valeur calculée à la main dans le composant.

- [ ] **Step 11.14: Écrire le test E2E analytics**

`e2e/admin/analytics.spec.ts` — 3 cas :

1. Le dashboard analytics affiche les KPI du rôle `analyst_readonly` sans contrôle de modification ; la navigation ne propose aucune action d'écriture.
2. Un KPI déclaré indisponible (`gross_margin_rate`, `conversion_rate`) affiche « indisponible » **avec sa raison** ; aucun `0` n'est affiché à la place, et la capture ne contient ni courbe ni valeur estimative.
3. Le `TrendChart` rend au maximum 3 séries, chacune avec un libellé direct et un marqueur distinct ; le tableau de données équivalent est présent (contrôle a11y de `@axe-core/playwright` sur la page).

```typescript
import { expect, test } from '@playwright/test';
import { loginAsAdmin } from './fixtures/admin-users.js';

test.describe('Analytics admin', () => {
  test.beforeEach(async ({ page }) => {
    await loginAsAdmin(page, 'analyst_readonly');
    await page.goto('/fr/admin/analytics');
  });

  test('affiche les KPI et leurs définitions sans action de modification', async ({ page }) => {
    await expect(page.getByRole('heading', { name: /analytics/i })).toBeVisible();
    await expect(page.getByRole('link', { name: /exporter/i })).toHaveCount(0);
  });

  test('affiche « indisponible » avec la raison plutôt qu’une valeur estimée', async ({ page }) => {
    const marge = page.getByTestId('kpi-gross_margin_rate');
    await expect(marge).toContainText(/indisponible/i);
    await expect(marge).not.toContainText('0');
  });

  test('rend au plus trois séries, chacune étiquetée, avec un tableau équivalent', async ({ page }) => {
    const chart = page.getByTestId('trend-chart-plot');
    await expect(chart.locator('path[data-series]')).toHaveCount(3);
    await expect(page.getByRole('table')).toBeVisible();
  });
});
```

- [ ] **Step 11.15: Traduire et vérifier**

```bash
pnpm --filter @fanours/i18n check:keys
pnpm --filter @fanours/web build
pnpm exec playwright test e2e/admin/analytics.spec.ts --project=functional
```

Attendu :

```text
i18n key parity OK (3 locales, 734 clés)
Built in 49.8s
  3 passed (14.4s)
```

**Preuves :**
- 9 tests unitaires (définitions complètes, pas de mélange de bases, indisponibilité déclarée, calcul d'un KPI disponible, `state: 'unavailable'` avec raison).
- 5 tests de `TrendChart` (libellés directs + marqueurs distincts, tokens CSS uniquement, interruption de courbe sur trou de données, état « indisponible » avec raison, tableau de données équivalent).
- 3 tests E2E analytics (rôle lecture seule, indisponibilité affichée avec raison, 3 séries maximum étiquetées + tableau équivalent).
- `gross_margin_rate` et `conversion_rate` s'affichent « indisponible » avec leur raison, **jamais 0** et jamais une valeur estimée.
- Le chiffre d'affaires, le nombre de commandes et les encaissements ne sont jamais additionnés dans un même total (garde `assertNoMixedBasis` + test de rendu).
- Le test de permissions de la Phase 1 est mis à jour dans le **même commit** (`analytics` ajoutée à `EXPECTED_RESOURCES`, 17 ressources au total) et passe.
- Chaque KPI affiche sa date de reconnaissance et ses tables sources.

**Commit :**

```bash
git add packages/analytics apps/api/src/modules/analytics packages/ui/src/components/TrendChart.tsx packages/ui/src/components/TrendChart.test.tsx packages/ui/src/components/index.ts "apps/web/src/app/[locale]/(admin)/admin/analytics" "apps/web/src/app/[locale]/(admin)/admin/page.tsx" packages/domain/src/permissions.ts packages/db/src/seed.ts apps/api/src/modules/rbac/__tests__/permissions.test.ts packages/i18n/src/locales
git commit -m "feat(analytics): indicateurs KPI definis, sources et non melanges" -m "Implémente le chantier 4.11 (spec §16.2) : package analytics avec 12 indicateurs définis par formule, date de reconnaissance, base et tables sources, garde interdisant de mélanger chiffre d'affaires, commandes et encaissements, état indisponible avec raison pour la marge brute et le taux de conversion plutôt qu'une valeur inventée. Ajoute TrendChart accessible et la vue globale admin." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 12: Notifications finalisées (4.12) — catalogue de déclencheurs, canaux, préférences, push, centre de notifications

**Files:**
- Create: `packages/contracts/src/notification-triggers.ts`
- Create: `e2e/account/notifications.spec.ts` (recette fonctionnelle de phase — projet Playwright `functional`)
- Create: `apps/api/src/modules/notification/notification-triggers.service.ts`
- Create: `apps/api/src/modules/notification/push.service.ts`
- Create: `apps/api/src/modules/notification/notification.controller.ts`
- Create: `apps/api/src/modules/notification/notification.module.ts`
- Test: `apps/api/src/modules/notification/notification-triggers.service.spec.ts`
- Create: `apps/api/src/providers/sms/sms-provider.ts`
- Create: `apps/api/src/providers/sms/mock-sms.provider.ts`
- Create: `apps/worker/src/jobs/notification/notification-dispatch.job.ts`
- Create: `apps/web/src/app/[locale]/(account)/compte/notifications/page.tsx` (modifie la page Phase 2)
- Create: `apps/web/src/components/account/NotificationPreferences.tsx`
- Test: `apps/web/src/components/account/NotificationPreferences.test.tsx`
- Create: `apps/web/src/components/notifications/NotificationCenter.tsx`
- Modify: `packages/domain/src/permissions.ts`
- Modify: `apps/api/src/modules/rbac/__tests__/permissions.test.ts` (liste `EXPECTED_RESOURCES` — ajout de `notification`)
- Modify: `packages/db/src/seed.ts`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `notification` (Phase 2), `notification_preference` (Phase 2), `push_subscription` (migration `0111`, Task 1), `user_preference` (migration `0111`, Task 1), `outbox_event` (Phase 1), `packages/jobs` → `defineJob`, `idempotencyKey`.
- Produces:
  - `packages/contracts/src/notification-triggers.ts` → `NOTIFICATION_TRIGGERS: readonly NotificationTrigger[]`, `NotificationTrigger`, `NotificationChannel = 'email' | 'in_app' | 'push' | 'sms'`, `NotificationCategory = 'transactional' | 'security' | 'stock_internal' | 'marketing'`, `NotificationAudience = 'client' | 'internal'`, `triggerById(id)`.
  - `NotificationTrigger = { id, category, audience, channels: NotificationChannel[], mandatory: boolean, templateKey, i18nKey, dedupeWindowSeconds, requiresConsent?: boolean }`
  - `NotificationTriggersService.dispatch(triggerId: string, payload: NotificationPayload): Promise<NotificationDispatchResult>`
  - `NotificationTriggersService.resolveRecipients(trigger: NotificationTrigger, payload: NotificationPayload): Promise<NotificationRecipient[]>`
  - `NotificationTriggersService.resolveChannels(trigger: NotificationTrigger, recipient: NotificationRecipient): Promise<NotificationChannel[]>`
  - `PushService.register(ctx: AccountContext, subscription: PushSubscriptionInputDto): Promise<PushSubscriptionDto>`
  - `PushService.unregister(ctx: AccountContext, subscriptionId: string): Promise<void>`
  - `PushService.send(userId: string, payload: PushPayload): Promise<PushSendResult>`
  - `SmsProvider` (interface) : `send(to: string, body: string): Promise<{ providerMessageId: string }>` ; `MockSmsProvider` (implémentation de test/staging, journalise sans envoyer).
  - Routes : `GET /v1/notifications`, `POST /v1/notifications/:id/read`, `POST /v1/notifications/read-all`, `GET /v1/notifications/preferences`, `PUT /v1/notifications/preferences`, `POST /v1/notifications/push-subscriptions`, `DELETE /v1/notifications/push-subscriptions/:id`.

> **Choix de modélisation** : cette tâche **ne crée aucune table**. La table `push_subscription` a été créée en migration `0111` (Task 1) et les préférences existent en Phase 2 (`notification_preference`) et en `0111` (`user_preference`). Le chantier 4.12 **finalise** le dispositif : catalogue explicite des déclencheurs, règles de consentement, routage multi-canal, job de distribution idempotent.

- [ ] **Step 12.1: Écrire le test rouge du catalogue et de la règle de consentement**

`apps/api/src/modules/notification/notification-triggers.service.spec.ts` :

```typescript
describe('NotificationTriggersService', () => {
  it('décline les 15 événements du spec §14.2 en déclencheurs atomiques connus', () => {
    expect(NOTIFICATION_TRIGGERS.length).toBe(36);
    expect(new Set(NOTIFICATION_TRIGGERS.map((t) => t.id)).size).toBe(36);
  });

  it('n’envoie jamais une notification transactionnelle désactivée par l’opt-out marketing', async () => {
    const user = await seedUser(ctx.db, { marketingOptIn: false, transactionalOptIn: true });

    const result = await service.dispatch('order.paid', { orderId, userId: user.id });

    expect(result.sent.some((s) => s.channel === 'email')).toBe(true);
  });

  it('n’envoie pas de push sans consentement explicite', async () => {
    const user = await seedUser(ctx.db, { pushConsent: false });

    const result = await service.dispatch('order.shipped', { orderId, userId: user.id });

    expect(result.sent.some((s) => s.channel === 'push')).toBe(false);
  });

  it('déduplique deux envois du même événement dans la fenêtre', async () => {
    const user = await seedUser(ctx.db, {});

    await service.dispatch('order.shipped', { orderId, userId: user.id });
    const second = await service.dispatch('order.shipped', { orderId, userId: user.id });

    expect(second.deduplicated).toBe(true);
  });
});
```

- [ ] **Step 12.1b: Écrire le test E2E rouge de la recette fonctionnelle des notifications**

`e2e/account/notifications.spec.ts` — recette fonctionnelle de phase, projet Playwright `functional`. Écrit **avant** l'implémentation : il échoue à ce stade et passe à l'étape « Traduire et vérifier ».

Cas et assertions :

1. Le centre de notifications liste les notifications **du seul** utilisateur authentifié (anti-IDOR) et le compteur « non lues » est exact ; marquer comme lue persiste après rechargement.
2. Une préférence de canal désactivée (e-mail, SMS ou push) empêche l'envoi sur ce canal ; le journal d'audit de l'envoi reste vide.
3. L'opt-out marketing n'empêche **jamais** une notification transactionnelle (compte, commande, paiement) : l'e-mail transactionnel part quand même.
4. L'enregistrement du push est refusé sans permission granted ; aucune erreur console n'est produite pendant la recette.

- [ ] **Step 12.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/api test src/modules/notification/notification-triggers.service.spec.ts
```

Attendu :

```text
 FAIL  src/modules/notification/notification-triggers.service.spec.ts
Error: Cannot find module '@fanours/contracts/dist/notification-triggers.js'
Test Files  1 failed (1)
     Tests  0 passed | 4 failed (4)
```

- [ ] **Step 12.3: Écrire le catalogue de déclencheurs**

`packages/contracts/src/notification-triggers.ts` : les 15 événements du spec §14.2 sont déclinés en **36 déclencheurs atomiques** (identifiant unique, catégorie, audience, canaux autorisés, caractère obligatoire, clé de modèle d'e-mail, clé i18n, fenêtre de déduplication) :

| Événement spec §14.2 | Déclencheurs |
|---|---|
| Vérification e-mail | `email.verification` |
| Connexion inhabituelle ou changement de sécurité | `security.unusual_login`, `security.password_changed`, `security.mfa_changed` |
| Demande/approbation de compte professionnel | `b2b.application_submitted`, `b2b.application_approved`, `b2b.application_rejected` |
| Devis créé, envoyé, modifié, accepté, refusé ou expiré | `quote.created`, `quote.sent`, `quote.modified`, `quote.accepted`, `quote.rejected`, `quote.expired` |
| Commande créée et paiement en attente | `order.created`, `order.payment_pending` |
| Paiement accepté, refusé, expiré ou contesté | `payment.accepted`, `payment.rejected`, `payment.expired`, `payment.disputed` |
| Confirmation de commande | `order.confirmed` |
| Préparation, expédition, tracking, retard, action douanière | `order.preparing`, `order.shipped`, `order.tracking_updated`, `order.delayed`, `order.customs_action_required` |
| Livraison | `order.delivered` |
| Facture ou avoir disponible | `invoice.available`, `credit_note.available` |
| Remboursement initié et terminé | `refund.initiated`, `refund.completed` |
| Retour autorisé, reçu ou décidé | `return.authorized`, `return.received`, `return.decided` |
| Nouveau message support | `support.message_created` |
| Stock bas pour les rôles internes | `stock.low` |
| Erreur d'intégration critique | `integration.critical_error` |

Règles inscrites dans le catalogue :

- `transactional` = `mandatory: true` (l'opt-out marketing ne peut pas les supprimer — spec §14.3) ;
- `security` = audience client + interne selon le cas, `requiresConsent: false` (une alerte de sécurité ne dépend pas d'un opt-in) ;
- `stock.low` et `integration.critical_error` = audience interne, e-mail aux rôles concernés (spec §14.2), jamais au client ;
- `push` = `requiresConsent: true` (spec §14.1 : notifications navigateur/PWA uniquement après consentement et lorsque pris en charge) ;
- `sms` = canal **désactivé par défaut** sur tous les déclencheurs : le SMS est « option ultérieure, si coût, consentement et conformité le permettent » (spec §14.1) — l'activation par déclencheur est une **décision propriétaire**.

- [ ] **Step 12.4: Implémenter le routage multi-canal**

`dispatch(triggerId, payload)` :

1. Résout le déclencheur (id inconnu → erreur `unknown_notification_trigger`).
2. Résout les destinataires selon l'audience (client : propriétaire de la commande / du devis / du ticket ; interne : rôles ciblés par le `payload`, résolus par `packages/domain`).
3. Résout les canaux : canaux du catalogue ∩ canaux autorisés par les préférences du destinataire ∩ consentement (push) ∩ capacité du fournisseur (SMS désactivé par défaut).
4. **Garde transactionnel** : un déclencheur `mandatory: true` est envoyé sur `email` + `in_app` même si l'utilisateur a désactivé les notifications marketing ; un déclencheur marketing sans consentement est ignoré (aucun envoi, aucun échec).
5. Déduplication : clé `(triggerId, destinataire, payload.resourceId)` dans une fenêtre `dedupeWindowSeconds` ; un doublon renvoie `deduplicated: true` et ne crée pas de notification.
6. Crée les lignes `notification` (Phase 2) avec `locale` de l'utilisateur, `channel`, `payload` (clé i18n + variables minimales, **jamais de donnée de carte ni de secret**), et publie l'outbox `notification.requested` pour l'envoi e-mail asynchrone (file, spec §14.4).

- [ ] **Step 12.5: Implémenter le service push**

`PushService.register()` : valide le endpoint (doit être `https`), stocke une ligne `push_subscription` (clé composite `user_id + endpoint` unique), horodate `created_at`/`last_used_at`, journalise `push.subscription.registered` (sans l'endpoint complet dans `metadata` : seul l'hôte).
`PushService.send()` : iterate les abonnements de l'utilisateur, envoie via le `PushProvider` (interface, implémentation `WebPushProvider` en Phase 6 — en Phase 4 le service journalise et renvoie `skipped: true` avec la raison `push_provider_not_configured` si aucun fournisseur n'est configuré) ; supprime les abonnements qui retournent `410 Gone` (consentement révoqué ou abonnement expiré).

- [ ] **Step 12.6: Implémenter le job de distribution**

`apps/worker/src/jobs/notification/notification-dispatch.job.ts` : job BullMQ `notification.dispatch`, `idempotencyKey` par notification (`notification:<id>`), backoff exponentiel + jitter, `attempts: 5`, DLQ. Le job lit les notifications en attente, envoie l'e-mail via le provider transactionnel, met à jour `status` (`queued` → `sent` | `failed` | `bounced`) et `sent_at`. Le template utilisé est enregistré dans l'historique avec `locale` et `template_version` (spec §14.3 : « date, langue et version du modèle conservées dans l'historique »).

- [ ] **Step 12.7: Implémenter le provider SMS (interface + mock)**

`apps/api/src/providers/sms/sms-provider.ts` définit l'interface `SmsProvider` ; `mock-sms.provider.ts` journalise le message et retourne un `providerMessageId` factice sans aucun envoi réel (staging et tests). Le choix du prestataire SMS réel est une **décision propriétaire** (spec §28.2) : tant qu'elle n'est pas prise, le canal SMS reste désactivé et aucun numéro n'est transmis à un tiers.

- [ ] **Step 12.8: Étendre la matrice de permissions**

`packages/domain/src/permissions.ts` : ressource `notification` avec les actions `read` (auto), `send` (interne) ; `super_admin` et `marketing_manager` obtiennent `notification: ['read', 'send']` pour l'envoi de notifications internes.

**Mise à jour du test de permissions de la Phase 1** — `apps/api/src/modules/rbac/__tests__/permissions.test.ts` : `notification` est ajoutée à `EXPECTED_RESOURCES` dans le **même commit**.

```typescript
const EXPECTED_RESOURCES = [
  'order', 'product', 'price_tier', 'export', 'export_financial', 'refund', 'admin_role',
  'audit_log', 'customs_rule', 'country', 'business_document', 'business_account',
  'customs_document', 'stock', 'content_page', 'support_ticket', 'analytics',
  'notification',                                       // <- ajouté en Task 12
] as const;
```

**Décompte final : 18 ressources (17 après la Task 11 + `notification`) et 13 actions.** Le test de permissions de la Phase 1 est désormais à jour et vert.

```bash
pnpm --filter @fanours/api test src/modules/rbac/__tests__/permissions.test.ts
```

Attendu :

```text
 ✓ src/modules/rbac/__tests__/permissions.test.ts (6 tests)
 Test Files  1 passed (1)
      Tests  6 passed (6)
```

- [ ] **Step 12.9: Tests verts**

```bash
pnpm --filter @fanours/api test src/modules/notification
pnpm --filter @fanours/web test src/components/account/NotificationPreferences.test.tsx
```

Attendu :

```text
 ✓ src/modules/notification/notification-triggers.service.spec.ts (12 tests)
 ✓ src/components/account/NotificationPreferences.test.tsx (4 tests)
 Test Files  2 passed (2)
      Tests  16 passed (16)
```

- [ ] **Step 12.10: Créer le centre de notifications et la page préférences**

`compte/notifications/page.tsx` : liste paginée des notifications de l'utilisateur (les plus récentes d'abord), regroupées par jour, avec état lu/non lu (indiqué par texte et icône, jamais par la couleur seule), actions « marquer comme lu » et « tout marquer comme lu », lien « configurer mes notifications ».

`NotificationPreferences.tsx` : interrupteurs par catégorie (`transactional` non désactivable — explained why, `security`, `stock_internal` non exposé au client, `marketing`), par canal (e-mail, push) ; l'enregistrement est optimiste avec retour arrière sur erreur ; le composant expose les libellés traduits.

`NotificationCenter.tsx` : cloche de notifications dans l'en-tête du compte (badge avec le nombre de non-lus, `aria-live="polite"`), panneau déroulant listant les 5 dernières notifications et lien « voir tout ».

- [ ] **Step 12.11: Traduire et vérifier**

```bash
pnpm --filter @fanours/i18n check:keys
pnpm --filter @fanours/web build
pnpm exec playwright test e2e/account/notifications.spec.ts --project=functional
```

Attendu :

```text
i18n key parity OK (3 locales, 758 clés)
Built in 50.2s
  3 passed (13.1s)
```

**Preuves :**
- 16 tests unitaires (catalogue de 36 déclencheurs uniques, transactionnel insupprimable, push sans consentement, déduplication, routage des canaux, inscription de subscription).
- Un opt-out marketing ne supprime **aucun** message transactionnel (test explicite) et un opt-out push n'entraîne aucun envoi navigateur.
- Deux envois du même événement dans la fenêtre de déduplication produisent **un seul** message.
- Aucun SMS n'est envoyé vers un tiers tant que le prestataire n'est pas choisi (canal désactivé par défaut, test à l'appui).
- Le test de permissions de la Phase 1 est mis à jour dans le **même commit** (`notification` ajoutée à `EXPECTED_RESOURCES`, 18 ressources au total) et passe.

**Commit :**

```bash
git add packages/contracts/src/notification-triggers.ts apps/api/src/modules/notification apps/api/src/providers/sms apps/worker/src/jobs/notification "apps/web/src/app/[locale]/(account)/compte/notifications" apps/web/src/components/account apps/web/src/components/notifications packages/domain/src/permissions.ts packages/db/src/seed.ts apps/api/src/modules/rbac/__tests__/permissions.test.ts packages/i18n/src/locales
git commit -m "feat(notification): catalogue de declencheurs, canaux, consentement et push" -m "Implémente le chantier 4.12 (spec §14.1, §14.2, §14.3) : les 15 événements déclencheurs du spec déclinés en 36 déclencheurs atomiques, routage multi-canal avec règle transactionnel insupprimable, déduplication par fenêtre, consentement obligatoire pour le push, job de distribution idempotent avec DLQ, provider SMS en interface plus mock, centre de notifications et page de préférences. Aucune table ajoutée." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 13: Socle transversal (RBAC, anti-IDOR, journalisation, coquille admin) — application uniforme aux 12 chantiers

**Files:**
- Create: `apps/api/src/common/admin-context.ts`
- Create: `apps/api/src/common/admin-audit.ts`
- Create: `apps/api/src/common/sensitive-action.ts`
- Modify: `apps/api/src/common/admin-scope.ts`
- Modify: `apps/api/src/modules/rbac/rbac.guard.ts`
- Modify: `packages/domain/src/permissions.ts`
- Modify: `apps/api/src/modules/rbac/__tests__/permissions.test.ts` (contrôle final : 18 ressources, 13 actions, 9 rôles)
- Create: `packages/domain/src/audit-actions.ts`
- Test: `apps/api/src/common/sensitive-action.spec.ts`
- Test: `apps/api/src/common/admin-scope.spec.ts`
- Modify: `apps/web/src/app/[locale]/(admin)/admin/layout.tsx`
- Modify: `apps/web/src/server/admin.ts`

**Interfaces:**
- Consumes: `audit_log` (Phase 1), `admin_action_approval` (Phase 1), `session` / `mfa_method` (Phase 1), `packages/domain` → `ROLES`, `RESOURCES`, `ACTIONS`, `PERMISSIONS`, `FOUR_EYES`, `decide` (Phase 1) · `apps/api/src/modules/rbac/require-permission.ts` → `requirePermission`, `Principal`, `assertRecentMfa`, `MFA_MAX_AGE_MS` (Phase 3) · `apps/api/src/modules/rbac/rbac.guard.ts` (Phase 1) · `apps/api/src/modules/rbac/approval.service.ts` → `approvalKindFor`, `requiresApproval`, `ApprovalService` (Phase 1).
- Produces:
  - `packages/domain/src/permissions.ts` → `RESOURCES` (18 ressources), `Resource`, `Role`, `Action` (13 actions), `PERMISSIONS: PermissionTable`, `FOUR_EYES: ReadonlySet<string>`, `PRIVILEGED: ReadonlySet<string>` (**nouveau** — la liste de la sécurité §3.3, qui n'était qu'une référence de CDC jusqu'ici et n'existait dans aucun code), `decide(role, resource, action): 'allow' | 'four_eyes' | 'deny'`. `MFA_MAX_AGE_MS` **n'est pas** redéclaré ici : il vit dans `apps/api/src/modules/rbac/require-permission.ts` (Phase 3) et est ré-exporté tel quel, pour ne pas créer deux constantes divergentes.
  - `packages/domain/src/audit-actions.ts` → `AUDIT_ACTION_CATALOG: readonly AuditActionDefinition[]`, `AuditActionDefinition = { id, family, objectType, mandatory, description }`, `auditActionById(id)`.
  - `apps/api/src/common/admin-context.ts` → `AdminContext = { userId, role, sessionId, mfaVerifiedAt: number | null, correlationId, locale, ip }`, `buildAdminContext(req, principal)`.
  - `apps/api/src/common/admin-audit.ts` → `recordAdminAction(ctx: AdminContext, entry: AdminAuditInput): Promise<void>`, `AdminAuditInput = { action, objectType, objectId?, result: 'success' | 'failure', metadata?: Record<string, string | number | boolean | null> }`.
  - `apps/api/src/common/sensitive-action.ts` → `runSensitiveAction<T>(ctx: AdminContext, opts: { resource, action, fourEyes?, auditObjectType? }, fn: () => Promise<T>): Promise<T>`, `assertFourEyes(ctx, opts)`.
  - `apps/api/src/common/admin-scope.ts` (étendu) → `scopeByCountry`, `scopeByAssignee`, `scopeByOwner`, `scopeByVisibility`, `scopeByB2bAccount`, `applyScope(where, scope)`, `ScopeFragment`.
  - `apps/api/src/modules/rbac/rbac.guard.ts` (étendu) → application de `requirePermission` à **toutes** les routes `/v1/admin/*`, y compris les sous-ressources.

> **Pourquoi cette tâche existe** : les 11 chantiers admin précédents ont chacun écrit leur propre vérification. Cette tâche les **factorise** et les **prouve** : une seule matrice de permissions, un seul point d'application du guard, un seul point de journalisation, un seul moteur d'action sensible (MFA + quatre-yeux), et un test paramétrique sur les 14 actions sensibles de la matrice sécurité §3.2. Sans cela, la garantie « deny by default » n'est pas démontrable.

- [ ] **Step 13.1: Écrire le test rouge paramétrique sur les 14 actions sensibles**

`apps/api/src/common/sensitive-action.spec.ts` — la table de test est **la matrice sécurité §3.2 transposée** ; toute divergence entre la matrice et le code est un échec de test, pas une documentation obsolète :

```typescript
const SENSITIVE_ACTIONS = [
  // [role, resource, action, attendu]
  ['support_agent', 'order', 'read', 'allow'],
  ['order_manager', 'order', 'update', 'allow'],
  ['catalog_manager', 'order', 'update', 'deny'],
  ['catalog_manager', 'product', 'create', 'allow'],
  ['order_manager', 'product', 'create', 'deny'],
  ['catalog_manager', 'price_tier', 'update', 'four_eyes'],
  ['super_admin', 'price_tier', 'update', 'four_eyes'],
  ['support_agent', 'export', 'export', 'allow'],
  ['support_agent', 'export_financial', 'export', 'deny'],
  ['finance_manager', 'export_financial', 'export', 'four_eyes'],
  ['support_agent', 'refund', 'refund', 'deny'],
  ['finance_manager', 'refund', 'refund', 'four_eyes'],
  ['order_manager', 'admin_role', 'manage_roles', 'deny'],
  ['marketing_manager', 'audit_log', 'read', 'deny'],
  ['compliance_manager', 'customs_rule', 'update', 'allow'],
  ['super_admin', 'country', 'update', 'four_eyes'],
  ['support_agent', 'business_document', 'read', 'allow'],
  ['order_manager', 'business_account', 'approve', 'deny'],
  ['compliance_manager', 'business_account', 'approve', 'allow'],
  ['logistics_operator', 'customs_document', 'create', 'allow'],
  ['analyst_readonly', 'order', 'update', 'deny'],
  ['analyst_readonly', 'export_financial', 'export', 'deny'],
] as const;

describe('decide (matrice sécurité §3.2)', () => {
  it.each(SENSITIVE_ACTIONS)('%s / %s / %s -> %s', (role, resource, action, expected) => {
    expect(decide(role, resource, action)).toBe(expected);
  });
});

describe('runSensitiveAction', () => {
  it('exige un MFA récent sur une action privilégiée', async () => {
    const ctx = { ...baseCtx, role: 'finance_manager' as const, mfaVerifiedAt: Date.now() - 11 * 60 * 1000 };

    await expect(runSensitiveAction(ctx, { resource: 'refund', action: 'refund' }, async () => 'ok'))
      .rejects.toMatchObject({ code: 'mfa_required' });
  });

  it('refuse l’auto-approbation d’une action à quatre yeux', async () => {
    const ctx = { ...baseCtx, role: 'finance_manager' as const, mfaVerifiedAt: Date.now() };

    await expect(
      runSensitiveAction(ctx, {
        resource: 'refund',
        action: 'refund',
        fourEyes: { requesterId: ctx.userId, approverId: ctx.userId },
      }, async () => 'ok'),
    ).rejects.toMatchObject({ code: 'self_approval_forbidden' });
  });

  it('journalise l’échec sans secret dans les métadonnées', async () => {
    const ctx = { ...baseCtx, role: 'order_manager' as const };

    await expect(
      runSensitiveAction(ctx, { resource: 'refund', action: 'refund' }, async () => 'ok'),
    ).rejects.toMatchObject({ status: 403 });

    const entry = await readLastAudit(ctx.db);
    expect(entry.result).toBe('failure');
    expect(JSON.stringify(entry.metadata)).not.toMatch(/password|token|card/i);
  });
});
```

- [ ] **Step 13.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/api test src/common/sensitive-action.spec.ts
```

Attendu :

```text
 FAIL  src/common/sensitive-action.spec.ts
Error: Cannot find module './sensitive-action.js'
 Test Files  1 failed (1)
      Tests  0 passed | 22 failed (22)
```

- [ ] **Step 13.3: Centraliser la matrice de permissions**

`packages/domain/src/permissions.ts` devient l'unique source de vérité :

- `RESOURCES` compte **18 ressources** : les 13 de la sécurité §3.3 plus les 5 ajoutées par la phase 4 — `stock` (Task 4), `content_page` (Task 8), `support_ticket` (Task 9), `analytics` (Task 11), `notification` (Task 12) — toutes déjà autorisées par la contrainte `ck_permission_resource` étendue en `0113` (Task 4). L'ajout est **progressif et tracé** : chaque ressource est ajoutée à `RESOURCES` **et** à `EXPECTED_RESOURCES` (test de la Phase 1) dans le commit de sa tâche ;
- `ACTIONS` compte **13 actions** : les 8 de la sécurité §3.3 plus `publish` (Task 8), `assign`, `escalate`, `refund_request` (Task 9) et `send` (Task 12). Le vocabulaire est déclaré **en entier dès la Task 4** parce que `ck_permission_action` n'est réécrite que par `0113` et que les 5 emplacements `0111`–`0115` sont clos ;
- `PERMISSIONS` reprend **exactement** la table de la sécurité §3.3, augmentée des 5 ressources de phase 4 selon les règles écrites dans les Tasks 4, 8, 9, 11 et 12 ;
- `FOUR_EYES` contient **les 8 couples de la sécurité §3.3, inchangés**. La demande de remboursement du support (Task 9) est matérialisée par une ligne `admin_action_approval` de kind `refund` avec **requester ≠ approver** (`no_self_approval`), pas par un neuvième couple : `support_agent` n'a pas l'action `refund` sur la ressource `refund`, il a l'action `refund_request` sur `support_ticket` — il **demande**, `finance_manager` ou `super_admin` **rembourse** ;
- `PRIVILEGED` est **créé** dans ce fichier : la liste des actions privilégiées de la sécurité §3.3 (`admin_role:manage_roles`, `refund:refund`, `export_financial:export`, `price_tier:update`, `country:update`, `customs_rule:update`, `business_account:approve`, `business_document:read`) n'existait jusqu'ici que comme référence de CDC, jamais comme constante de code. Elle est **élargie** aux actions de phase 4 qui exigent un MFA récent : `stock:update` (correction massive, Task 4), `content_page:publish` (Task 8), `support_ticket:refund_request` (Task 9), `notification:send` (Task 12). `MFA_MAX_AGE_MS` n'est pas redéclaré ici : il reste dans `apps/api/src/modules/rbac/require-permission.ts` (Phase 3) et est ré-exporté ;
- `decide(role, resource, action)` est **centralisé ici** : les phases 1 à 3 deciding au niveau service via `requirePermission` (Phase 3) ; cette tâche introduit la fonction de décision unique `'allow' | 'four_eyes' | 'deny'` du CDC sécurité §3.3, **deny by default** — une permission absente est un refus, jamais une hésitation. `requirePermission` (Phase 3) est **réécrit** pour déléguer à `decide()`, sans changement de comportement observable.

Aucune exception ajoutée : si un besoin métier n'est pas dans la matrice, il se demande au propriétaire (décision propriétaire), il ne se code pas en dur dans un service.

**Contrôle final du test de permissions de la Phase 1** — `apps/api/src/modules/rbac/__tests__/permissions.test.ts` : à ce stade, la liste `EXPECTED_RESOURCES` doit contenir les 18 ressources et `EXPECTED_ACTIONS` les 13 actions. Le test de la Phase 1 est **rouge** si une des Tasks 4, 8, 9, 11 ou 12 a oublié de mettre à jour la liste attendue : c'est le garde-fou qui empêche une dérive silencieuse entre la matrice du code et le test.

```typescript
// État attendu à la fin de la Task 12, vérifié par la Task 13.
const EXPECTED_RESOURCES = [
  'order', 'product', 'price_tier', 'export', 'export_financial', 'refund', 'admin_role',
  'audit_log', 'customs_rule', 'country', 'business_document', 'business_account',
  'customs_document', 'stock', 'content_page', 'support_ticket', 'analytics', 'notification',
] as const;                                                     // 18 ressources

const EXPECTED_ACTIONS = [
  'read', 'create', 'update', 'delete', 'export', 'approve', 'refund', 'manage_roles',
  'publish', 'assign', 'escalate', 'refund_request', 'send',
] as const;                                                    // 13 actions
```

```bash
pnpm --filter @fanours/api test src/modules/rbac/__tests__/permissions.test.ts
```

Attendu :

```text
 ✓ src/modules/rbac/__tests__/permissions.test.ts (6 tests)
 Test Files  1 passed (1)
      Tests  6 passed (6)
```

- [ ] **Step 13.4: Créer le contexte admin et le point unique de journalisation**

`apps/api/src/common/admin-context.ts` : `AdminContext` porte `userId`, `role`, `sessionId`, `mfaVerifiedAt`, `correlationId`, `locale`, `ip`. `buildAdminContext(req, principal)` lit le `correlation_id` de la requête (ou en génère un, format `traceparent` W3C) et le propage.

`apps/api/src/common/admin-audit.ts` : `recordAdminAction()` écrit **une** ligne `audit_log` (acteur, rôle, action, objet, horodatage, résultat, `correlation_id`, métadonnées filtrées) et publie l'outbox `audit.recorded`. Le filtrage des métadonnées applique une liste blanche de clés et **interdit** toute valeur qui ressemble à un secret (`password`, `token`, `secret`, `card`, `pan`, `cvv`) — si une clé correspond, la valeur est remplacée par `'[redacted]'` et une métrique `audit_redaction_total` est incrémentée.

`packages/domain/src/audit-actions.ts` : catalogue des **12 familles** d'événements du spec §19.7 — connexion admin, échec de MFA, création/modification produit, changement de prix, changement de règle douanière, modification de stock, décision B2B, remboursement, annulation, changement de rôle, export de données, modification de configuration sensible — avec, pour chaque action, son type d'objet et son caractère obligatoire. Toute écriture dans `audit_log` passe par une action du catalogue : une action inconnue est rejetée (`unknown_audit_action`).

- [ ] **Step 13.5: Créer le moteur d'action sensible**

`apps/api/src/common/sensitive-action.ts` : `runSensitiveAction(ctx, { resource, action, fourEyes?, auditObjectType? }, fn)` :

1. `decide(ctx.role, resource, action)` → `deny` ⇒ `403 action_not_permitted` **sans révéler l'existence de la ressource** ;
2. action privilégiée ⇒ MFA vérifié et `MFA_MAX_AGE_MS` respecté, sinon `403 mfa_required` ;
3. `four_eyes` ⇒ approbation d'un **second rôle** requise ; `approverId === requesterId` ⇒ `403 self_approval_forbidden` ; l'approbation n'est pas expirée (`expires_at` dans le futur) ;
4. exécution de `fn()` dans une transaction (rollback en cas d'échec) ;
5. journalisation systématique : `success` ou `failure`, avec l'objet et le `correlation_id`.

Toutes les actions sensibles des Tasks 2 à 12 passent par ce moteur ; les routes déjà implémentées sont refactorisées pour l'utiliser (le comportement observable ne change pas, les tests des tâches précédentes restent verts).

- [ ] **Step 13.6: Étendre l'anti-IDOR et le guard RBAC**

`apps/api/src/common/admin-scope.ts` : les fragments de scoping existants (`scopeByCountry`, `scopeByAssignee`, `scopeByOwner`, `scopeByVisibility`) sont complétés par `scopeByB2bAccount` et `scopeByRole`, et appliqués **dans la clause `WHERE`** — jamais en post-filtrage en mémoire. Toute liste admin dont la ressource appartient à un client (commande, ticket, document B2B) applique le scoping ; une ressource absente du scope renvoie `404` (l'existence n'est pas révélée).

`apps/api/src/modules/rbac/rbac.guard.ts` : le guard est appliqué à **toutes** les routes `/v1/admin/*` via un décorateur de contrôleur global, y compris les sous-ressources (`/v1/admin/orders/:number/notes`, `/v1/admin/support/tickets/:id/refund-requests`, `/v1/admin/audit/exports`…). Une route sans `resource`/`action` déclarés **échoue** au démarrage de l'application (`missing_permission_declaration`) — l'oubli devient impossible.

- [ ] **Step 13.7: Durcir la coquille admin**

`apps/web/src/app/[locale]/(admin)/admin/layout.tsx` :

- redirection vers la connexion si session absente ; message « session expirée » avec action de reconnexion si la session a expiré pendant l'usage ;
- bandeau MFA lorsque `mfaVerifiedAt` est absent ou trop ancien, avec bouton de revalidation MFA sur les écrans sensibles ;
- navigation filtrée par rôle (`AdminNav` ne propose que les entrées autorisées par la matrice — la matrice fait foi, pas la liste de liens) ;
- bandeau de contexte « journalisé » rappelant que les actions de cette session sont tracées (transparence, spec §19.7) ;
- `<meta name="robots" content="noindex, nofollow">` et en-tête `X-Robots-Tag` sur toutes les réponses admin ;
- gestion des états loading / erreur / accès refusé / session expirée.

`apps/web/src/server/admin.ts` : chaque appel serveur admin joint le `correlation_id` de la requête, impose `cache: 'no-store'` (aucune donnée d'admin dans un cache partagé ou un CDN) et n'expose jamais de champ interne.

- [ ] **Step 13.8: Tests verts et non-régression des tâches précédentes**

```bash
pnpm --filter @fanours/api test src/common
pnpm --filter @fanours/api test
pnpm --filter @fanours/web test
```

Attendu :

```text
 ✓ src/common/sensitive-action.spec.ts (25 tests)
 ✓ src/common/admin-scope.spec.ts (11 tests)
 Test Files  2 passed (2)
      Tests  36 passed (36)

 Test Files  87 passed (87)
      Tests  412 passed (412)
```

- [ ] **Step 13.9: Vérifier que la matrice n'a pas de trou**

```bash
pnpm --filter @fanours/api test src/common/sensitive-action.spec.ts -- --reporter=verbose
```

Attendu : les 22 lignes de la table paramétrique, les 3 cas `runSensitiveAction` et les 11 cas `admin-scope` passent, et **aucune** ligne de la matrice sécurité §3.2 n'est laissée sans assertion.

**Preuves :**
- 36 tests unitaires : 22 cas de la matrice §3.2 (rôle × ressource × action → allow / deny / four_eyes), MFA expirée refusée, auto-approbation refusée, échec journalisé sans secret.
- La suite complète de l'API (412 tests) et du web passe après factorisation : **aucune régression** introduite par le socle transversal.
- Toute route `/v1/admin/*` sans déclaration de permission fait échouer le démarrage de l'application (test de garde sur le catalogue de routes).
- Le refus renvoie `403` sans révéler l'existence de la ressource ; l'absence de propriété renvoie `404`.

**Commit :**

```bash
git add apps/api/src/common apps/api/src/modules/rbac/rbac.guard.ts apps/api/src/modules/rbac/__tests__/permissions.test.ts packages/domain/src/permissions.ts packages/domain/src/audit-actions.ts "apps/web/src/app/[locale]/(admin)/admin/layout.tsx" apps/web/src/server/admin.ts
git commit -m "refactor(admin): socle transversal RBAC, anti-IDOR, journalisation et coquille admin" -m "Implémente le socle transversal de la phase 4 : matrice de permissions unique de 18 ressources alignée sur la matrice sécurité §3.2, deny by default, moteur d'action sensible avec MFA de 10 minutes et quatre-yeux, point unique de journalisation append-only avec redaction des secrets, scoping anti-IDOR appliqué dans la clause WHERE, coquille admin durcie (MFA, navigation par rôle, noindex, no-store) et test paramétrique sur les 14 actions sensibles." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 14: Recette Phase 4 — 11 cas E2E admin, a11y, non-régression sécurité, exports, responsive, runbooks

**Files:**
- Create: `e2e/admin/fixtures/admin-users.ts`
- Create: `e2e/admin/dashboard.spec.ts`
- Create: `e2e/admin/a11y.spec.ts`
- Create: `e2e/admin/security-regression.spec.ts`
- Create: `e2e/admin/exports.spec.ts`
- Create: `e2e/visual/responsive.spec.ts`
- Create: `ops/runbooks/admin-backoffice.md`
- Create: `ops/runbooks/reconciliation-daily.md`
- Create: `ops/checklists/phase4-manual-a11y.md`

**Interfaces:**
- Consumes: les 12 chantiers livrés par les Tasks 1 à 13, `@playwright/test`, `@axe-core/playwright`, `apps/api` (base de test), `packages/contracts`, `e2e/support/viewports.ts` (liste canonique des 8 viewports, Phase 1).
- Produces:
  - `e2e/admin/fixtures/admin-users.ts` → `ADMIN_USERS: Record<Role, AdminUserFixture>`, `loginAsAdmin(page, role, opts?)`, `resolveAdminCredentials(role)`. Les identifiants de test proviennent de l'environnement de test (`E2E_ADMIN_PASSWORD` — **jamais** de secret en dur dans le dépôt) ; l'absence de variable fait échouer la suite avec un message explicite, elle ne contourne pas l'authentification.
  - `e2e/admin/dashboard.spec.ts` → 11 cas. C'est le **seul** fichier capturé par le projet Playwright `admin` (`testMatch: '**/admin/dashboard.spec.ts'`) : la Phase 4 livre ce fichier, la Phase 5 le reprend en **Modify** pour le porter au niveau release — elle n'en crée pas de doublon.
  - `e2e/admin/a11y.spec.ts` → audit axe-core sur les 2 dashboards (client et admin), 3 langues, 2 thèmes. Recette fonctionnelle de phase → projet `functional`.
  - `e2e/admin/security-regression.spec.ts` → non-régression sur les 14 actions sensibles. Recette fonctionnelle de phase → projet `functional`.
  - `e2e/admin/exports.spec.ts` → exports (catalogue, commandes, financier, audit) et rapprochement quotidien. Recette fonctionnelle de phase → projet `functional`.
  - `e2e/visual/responsive.spec.ts` → matrice responsive 8 viewports × 3 langues × 2 thèmes sur les 2 dashboards, projet `visual`.
  - `ops/runbooks/admin-backoffice.md`, `ops/runbooks/reconciliation-daily.md` → procédures opératoires.
  - `ops/checklists/phase4-manual-a11y.md` → checklist de vérification manuelle clavier / lecteur d'écran.

- [ ] **Step 14.1: Créer les fixtures d'administration**

`e2e/admin/fixtures/admin-users.ts` définit un compte de test par rôle (les 9 rôles de la matrice §3.1) et l'helper `loginAsAdmin(page, role)` qui exécute le parcours complet : connexion → **challenge MFA** (le code TOTP est calculé par le test à partir du secret de test lu dans l'environnement de test) → session admin.

Contraintes :

- aucun mot de passe ni secret TOTP en dur dans le dépôt ; la variable d'environnement `E2E_ADMIN_PASSWORD` et le secret TOTP de test sont requis ;
- si la variable est absente, la suite **échoue** avec `E2E_ADMIN_PASSWORD is required` (aucun contournement, aucun mode dégradé) ;
- les comptes de test sont créés et révoqués par la fixture, jamais réutilisés en production.

- [ ] **Step 14.2: Écrire les 11 cas E2E d'administration**

`e2e/admin/dashboard.spec.ts` — les 10 parcours du spec §24.3 « Parcours administration », plus un onzième cas propre à la phase 4. **Fichier des 11 cas** : `e2e/admin/dashboard.spec.ts`, seul fichier capturé par le projet Playwright `admin` (`testMatch: '**/admin/dashboard.spec.ts'`). Conformément à `docs/roadmap/sections/qualite.md` §24.4, la Phase 4 livre ces 11 cas et la Phase 5 rejoue la suite complète sur staging : elle reprend ce fichier en **Modify** pour le niveau release, elle n'en crée pas de doublon.

| # | Cas | Fichier | Attendu |
|---|---|---|---|
| 1 | Connexion admin avec MFA | `e2e/admin/dashboard.spec.ts` | session créée, `mfaVerifiedAt` frais, badge « MFA active » |
| 2 | Refus d'accès pour un rôle trop faible | `e2e/admin/dashboard.spec.ts` | `403` + message localisé, aucun contenu admin divulgué |
| 3 | Création et publication d'un produit traduit en 3 langues | `e2e/admin/dashboard.spec.ts` | produit `published` + 3 traductions, sitemap mis à jour |
| 4 | Modification de MOQ et de prix | `e2e/admin/dashboard.spec.ts` | palier créé en `draft`, **approbation quatre-yeux requise**, audit `price_tier.updated` |
| 5 | Gestion d'un stock bas | `e2e/admin/dashboard.spec.ts` | seuil franchi → alerte interne créée, notification `stock.low` émise |
| 6 | Traitement d'une commande et création d'une expédition | `e2e/admin/dashboard.spec.ts` | transition autorisée, réservation de stock décrémentée, audit |
| 7 | Expédition refusée si les données douanières obligatoires manquent | `e2e/admin/dashboard.spec.ts` | blocage serveur, message actionnable, aucun statut avancé |
| 8 | Mise à jour d'une règle, historique et alerte de révision | `e2e/admin/dashboard.spec.ts` | règle versionnée, `compliance_alert` ouverte, historique consultable |
| 9 | Export restreint selon le rôle | `e2e/admin/dashboard.spec.ts` | `analyst_readonly` → `403` sur export financier ; `finance_manager` → demande d'approbation |
| 10 | Journal d'audit pour action financière sensible | `e2e/admin/dashboard.spec.ts` | entrée présente, chaîne intègre, `correlation_id` renseigné |
| 11 | Navigation admin limitée au rôle + action non déclarée refusée | `e2e/admin/dashboard.spec.ts` | menu conforme à la matrice, route non déclarée refusée au démarrage |

Chaque cas vérifie aussi **l'absence d'erreur console** et **l'absence de requête réseau en échec**.

- [ ] **Step 14.3: Écrire le test d'accessibilité des deux dashboards**

`e2e/admin/a11y.spec.ts` : `@axe-core/playwright` sur le dashboard **client** (`/fr/compte`) et le dashboard **admin** (`/fr/admin`), en 3 langues (`fr-FR`, `en`, `zh-CN`) et 2 thèmes (`light`, `dark`). Règles : **zéro violation critique ou sérieuse** ; les violations modérées et mineures sont listées dans le rapport et suivies, pas ignorées en silence. Le test vérifie aussi l'absence de débordement horizontal (`document.scrollWidth <= window.innerWidth`), la présence d'un H1 unique et d'un ordre de focus logique. Les 3 langues et les 2 thèmes sont itérés **dans le même `test.describe`** ; les viewports utilisés sont ceux de la liste canonique `e2e/support/viewports.ts` (créée en Phase 1, **importée, jamais re-déclarée**). Ce fichier est une recette fonctionnelle de phase : il relève du projet `functional`, pas de la suite de release `admin` (`e2e/admin/dashboard.spec.ts`).

- [ ] **Step 14.4: Écrire le test responsive multi-viewports (8 viewports × 3 langues × 2 thèmes)**

`e2e/visual/responsive.spec.ts` — projet Playwright `visual` (matrice visuelle). Les 8 viewports sont itérés **dans le spec** via `test.use({ viewport })`, à partir de la liste canonique `e2e/support/viewports.ts` (créée en Phase 1, **importée, jamais re-déclarée**) ; les 3 langues × 2 thèmes sont itérés dans le même `test.describe`. **Les viewports ne sont pas des projets Playwright** et aucun `testDir` n'est déclaré par projet.

```ts
import { expect, test } from "@playwright/test";
import { VIEWPORTS } from "../support/viewports.js";

const PAGES = [
  { name: "dashboard client", path: "compte" },
  { name: "back-office admin", path: "admin" },
] as const;

// segments d'URL publics : /fr/, /en/, /zh-cn/
const LOCALES = ["fr", "en", "zh-cn"] as const;
const THEMES = ["light", "dark"] as const;

for (const viewport of VIEWPORTS) {
  test.describe(`responsive ${viewport.width}x${viewport.height}`, () => {
    test.use({ viewport: { width: viewport.width, height: viewport.height } });

    for (const locale of LOCALES) {
      for (const theme of THEMES) {
        for (const target of PAGES) {
          test(`aucun debordement horizontal - ${target.name} - ${locale} - ${theme}`, async ({ page }) => {
            // cle lue par le script anti-flash de la Phase 1
            await page.addInitScript((value) => {
              window.localStorage.setItem("theme", value);
            }, theme);
            await page.goto(`/${locale}/${target.path}`, { waitUntil: "networkidle" });
            const { scrollWidth, innerWidth } = await page.evaluate(() => ({
              scrollWidth: document.documentElement.scrollWidth,
              innerWidth: window.innerWidth,
            }));
            expect(scrollWidth).toBeLessThanOrEqual(innerWidth);
          });
        }
      }
    }
  });
}
```

Les 2 pages couvertes sont le dashboard client (`/{locale}/compte`) et le back-office (`/{locale}/admin`). Le volume de la matrice est `8 viewports × 3 langues × 2 thèmes × 2 pages = 96 tests`, tous dans le projet `visual`.

- [ ] **Step 14.5: Écrire la non-régression des 14 actions sensibles**

`e2e/admin/security-regression.spec.ts` : un test par ligne de la matrice sécurité §3.2, en conducteur de bout en bout (HTTP + navigateur) :

- ligne autorisée → `2xx` et présence de la preuve attendue (entrée d'audit, ressource créée) ;
- ligne interdite → `403` **sans** indice sur l'existence de la ressource ;
- ligne à quatre yeux → `409 approval_required` ou création d'une demande d'approbation, et **exécution bloquée** tant que l'approbation n'est pas donnée ;
- ligne privilégiée sans MFA frais → `403 mfa_required`.

Le test vérifie également que l'anti-IDOR fonctionne : un `order_manager` qui modifie l'identifiant d'une commande d'un autre client obtient `404`, jamais `403` (l'existence n'est pas révélée).

- [ ] **Step 14.6: Écrire les tests d'export et de rapprochement**

`e2e/admin/exports.spec.ts` :

- export catalogue (CSV) : contenu conforme à la liste blanche de champs, **URL signée expirant après 5 minutes**, fichier stocké dans le bucket privé (aucune URL publique) ;
- export financier : refusé sans approbation, produit après approbation d'un second rôle, journalisé (`audit.export.requested`, `audit.export.executed`) ;
- export du journal : refusé à `marketing_manager`, autorisé à `compliance_manager` ;
- rapprochement quotidien : un écart volontaire (paiement simulé manquant) apparaît en ligne `open` avec `has_difference = true`, l'annotation et la résolution ferment la ligne ; le job est idempotent (deux exécutions le même jour ne dupliquent pas les lignes).

- [ ] **Step 14.7: Exécuter la recette complète**

```bash
pnpm --filter @fanours/api test
pnpm --filter @fanours/web test
pnpm --filter @fanours/analytics test
pnpm exec playwright test e2e/admin/dashboard.spec.ts --project=admin    # suite de release admin — les 11 cas
pnpm exec playwright test e2e/admin --project=functional                 # autres specs admin de la phase (a11y, security-regression, exports, analytics)
pnpm exec playwright test e2e/account --project=functional               # dashboard client
pnpm exec playwright test e2e/specs/security.spec.ts --project=functional # non-régression sécurité des phases 1 à 3
pnpm exec playwright test e2e/visual/responsive.spec.ts --project=visual  # matrice 8 viewports × 3 langues × 2 thèmes
```

Attendu :

```text
 Test Files  87 passed (87)
      Tests  412 passed (412)

 Test Files  21 passed (21)
      Tests  118 passed (118)

 ✓ src/kpi-definitions.test.ts (3 tests)

Running 27 tests using 1 worker
  ✓ admin/dashboard.spec.ts:9:1 › 1. Connexion admin avec MFA
  ✓ admin/dashboard.spec.ts:9:1 › 2. Refus d'accès pour un rôle trop faible
  ✓ admin/dashboard.spec.ts:9:1 › 3. Création et publication d'un produit traduit en trois langues
  ✓ admin/dashboard.spec.ts:9:1 › 4. Modification de MOQ et de prix
  ✓ admin/dashboard.spec.ts:9:1 › 5. Gestion d'un stock bas
  ✓ admin/dashboard.spec.ts:9:1 › 6. Traitement d'une commande et création d'une expédition
  ✓ admin/dashboard.spec.ts:9:1 › 7. Refus d'expédition si les données douanières obligatoires manquent
  ✓ admin/dashboard.spec.ts:9:1 › 8. Mise à jour d'une règle, historique et alerte de révision
  ✓ admin/dashboard.spec.ts:9:1 › 9. Export restreint selon le rôle
  ✓ admin/dashboard.spec.ts:9:1 › 10. Journal d'audit pour action financière sensible
  ✓ admin/dashboard.spec.ts:9:1 › 11. Navigation admin limitée au rôle
  ✓ admin/a11y.spec.ts:12:1 › axe-core — dashboard client, fr-FR, light
  ✓ admin/a11y.spec.ts:12:1 › axe-core — dashboard client, en, dark
  ✓ admin/a11y.spec.ts:12:1 › axe-core — dashboard client, zh-CN, light
  ✓ admin/a11y.spec.ts:12:1 › axe-core — dashboard admin, fr-FR, light
  ✓ admin/a11y.spec.ts:12:1 › axe-core — dashboard admin, en, dark
  ✓ admin/a11y.spec.ts:12:1 › axe-core — dashboard admin, zh-CN, light
  ✓ admin/security-regression.spec.ts:14:1 › 1. Lire une commande
  ✓ admin/security-regression.spec.ts:14:1 › 2. Modifier une commande
  ...
  ✓ admin/exports.spec.ts:10:1 › export catalogue — liste blanche et URL signée 5 min
  ✓ admin/exports.spec.ts:10:1 › export financier — quatre-yeux
  ✓ admin/exports.spec.ts:10:1 › export journal — rôles autorisés
  ✓ admin/exports.spec.ts:10:1 › rapprochement quotidien — écart, annotation, résolution

  27 passed (48.7s)
```

Les 27 tests de la phase sont répartis sur **deux exécutions** : les 11 cas de `e2e/admin/dashboard.spec.ts` sont capturés par le projet `admin` (`--project=admin`), les 16 autres specs `e2e/admin/*` (a11y, security-regression, exports) par le projet `functional` (`--project=functional`, dont le `testIgnore` exclut `**/admin/dashboard.spec.ts`). La matrice `e2e/visual/responsive.spec.ts` est exécutée séparément par le projet `visual` : aucun test n'est compté deux fois.

- [ ] **Step 14.8: Rédiger les runbooks**

`ops/runbooks/admin-backoffice.md` : procédures opérationnelles du back-office — accès et MFA, gestion d'une commande, inventaire, gestion d'un litige, escalade, répondre à une alerte de conformité, procédure d'urgence si le back-office est indisponible (file e-mail saturée, prestataire de paiement HS), et **qui appeler** (rôles du spec §23.4).

`ops/runbooks/reconciliation-daily.md` : procédure quotidienne du rapprochement — exécution du job de 02:30 UTC, lecture des écarts, qualification d'un écart (doublon, frais, remise non appliquée, paiement non reçu), traitement d'un écart par état (`open` → `investigating` → `accepted` / `resolved`), escalade au-delà d'un seuil, et procédure de **double remboursement** (vérification de la somme avant toute action).

- [ ] **Step 14.9: Rédiger la checklist de vérification manuelle**

`ops/checklists/phase4-manual-a11y.md` : parcours à vérifier **manuellement** (l'audit automatique ne prouve rien) — clavier seul et lecteur d'écran sur : connexion admin, MFA, dashboard client, liste et détail de commande admin, changement de statut, import catalogue, inventaire, rapprochement, ticket support, journal d'audit, centre de notifications, préférences ; pour chaque écran : ordre de focus, piège de focus des modales, annonces des changements de stock et de total, contraste, zoom 200 %, `prefers-reduced-motion`, et **absence de débordement horizontal** sur les 8 viewports de la liste canonique `e2e/support/viewports.ts` (importée depuis le dépôt, jamais re-déclarée).

- [ ] **Step 14.10: Revue sécurité ciblée de la phase 4**

```bash
pnpm exec gitleaks detect --no-banner --redact
pnpm audit --audit-level=high
pnpm exec semgrep scan --config auto --error
```

Attendu :

```text
no leaks found (0 bytes scanned)
found 0 vulnerabilities
Scan Summary: 0 findings
```

Cette revue couvre spécifiquement : les exports (liste blanche de champs, URL signée courte durée, absence d'export pour `analyst_readonly`), le rapprochement financier (contrôle serveur des sommes, journalisation, quatre-yeux), l'accès aux documents B2B (chiffrement, bucket privé, URL 5 min, traçabilité ligne par ligne), et l'absence de secret dans les journaux et les exports.

**Preuves :**
- 27 tests E2E admin (11 cas d'administration, 6 audits a11y, non-régression des 14 actions sensibles, 4 tests d'export et de rapprochement) au vert.
- 96 tests de la matrice responsive (`e2e/visual/responsive.spec.ts`, projet `visual` : 8 viewports × 3 langues × 2 thèmes × 2 pages) au vert.
- 0 violation axe-core critique ou sérieuse sur les 2 dashboards, en 3 langues et 2 thèmes.
- Suite complète verte : 412 tests API, 118 tests web, 3 tests analytics, **aucune régression** sur les phases 1 à 3.
- Revue sécurité : 0 fuite de secret détectée, 0 vulnérabilité haute, 0 finding semgrep.
- Runbooks et checklist manuelle rédigés et versés dans `ops/`.

**Commit :**

```bash
git add e2e/admin e2e/visual ops/runbooks ops/checklists
git commit -m "test(admin): recette phase 4, a11y, non-regression securite et runbooks" -m "Implémente la recette du chantier 4 : 11 cas E2E d'administration conformes au spec §24.3, audit axe-core sur les dashboards client et admin en 3 langues et 2 thèmes, non-régression sur les 14 actions sensibles de la matrice sécurité §3.2, tests d'export et de rapprochement, matrice responsive 8 viewports × 3 langues × 2 thèmes, runbooks d'exploitation et checklist de vérification manuelle clavier et lecteur d'écran." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Self-Review

### 1. Conformité au spec — les 12 chantiers de la phase 4

| Chantier | Sections spec | Tâche | Artefacts produits |
|---|---|---|---|
| 4.1 Dashboard client | §4.8, §16.3, §16.4, §16.5 | **Task 1** | migration `0111`, `apps/api/src/modules/account/*`, `apps/web/.../compte/*` (commandes, factures, adresses, préférences, sécurité) |
| 4.2 Back-office commandes | §16.6, §5.1 | **Task 2** | `apps/api/src/modules/admin-orders/*`, coquille admin `admin/layout.tsx`, `AdminNav`, `OrderStatusTimeline`, actions groupées |
| 4.3 Back-office catalogue | §16.7, §16.8 | **Task 3** | migration `0112`, `admin-catalog/*`, `catalog-parser.ts`, job `catalog-import.job.ts`, import/export |
| 4.4 Back-office stock | §16.9, §5.3 | **Task 4** | migration `0113`, `admin-stock/*`, job `stock-alert.job.ts`, inventaire et réconciliation de stock |
| 4.5 Back-office clients et B2B | §16.10, §8.2, §7.6 | **Task 5** | `admin-customers/*`, `admin/clients/[id]/page.tsx`, approbation B2B, suspension, documents signés 5 min |
| 4.6 Back-office finance | §16.7, §10.4, §23.5 | **Task 6** | migration `0114`, `reconciliation.service.ts`, `finance-export.service.ts`, job `payment-reconcile.job.ts` (02:30 UTC) |
| 4.7 Back-office logistique et douane | §11.2, §11.6, §16.9 | **Task 7** | `admin-logistics/*`, `admin-compliance-config.service.ts`, matrice pays × produits × transporteurs |
| 4.8 Back-office contenu et SEO | §16.10, §17 | **Task 8** | migration `0115`, `admin-content/*`, `redirect.service.ts`, conditions de vente versionnées |
| 4.9 Support | §16.11, §12.5 | **Task 9** | `support/*`, `SUPPORT_MACROS`, SLA, demande de remboursement quatre-yeux |
| 4.10 Audit et journal | §19.7 | **Task 10** | `audit/*`, `AuditTimeline`, vérification de chaîne SHA-256, export tracé |
| 4.11 Analytics | §16.2 | **Task 11** | package `packages/analytics`, 12 KPI définis, `TrendChart`, vue globale admin |
| 4.12 Notifications finalisées | §14.1, §14.2, §14.3 | **Task 12** | `notification-triggers.ts` (36 déclencheurs), `push.service.ts`, job `notification-dispatch.job.ts`, centre de notifications |
| Socle transversal | §19.2, §19.3, sécurité §3.2, §3.3, §3.4 | **Task 13** | matrice de permissions unique, moteur d'action sensible, point unique de journalisation, coquille admin durcie |
| Recette | §24.3, §24.4, §22.4 | **Task 14** | 11 cas E2E admin (`e2e/admin/dashboard.spec.ts`, projet `admin`), axe-core sur 2 dashboards, non-régression des 14 actions sensibles, matrice responsive 8 viewports × 3 langues × 2 thèmes (`e2e/visual/responsive.spec.ts`, projet `visual`), runbooks, checklist manuelle |

**Répartition des migrations** (5 numéros, 5 tâches propriétaires, une seule migration par tâche) :

| N° | Fichier | Tâche propriétaire | Contenu |
|---|---|---|---|
| `0111` | `0111_phase4_client_account.sql` | Task 1 | `address`, `user_preference`, `push_subscription` |
| `0112` | `0112_phase4_catalog_ops.sql` | Task 3 | `data_import`, `data_import_row`, `export_request` |
| `0113` | `0113_phase4_stock.sql` | Task 4 | `stock_count`, `stock_count_line`, `inventory_item.reorder_point/reorder_qty`, extension de `ck_permission_resource` (13 → 18 ressources) et de `ck_permission_action` (8 → 13 actions) |
| `0114` | `0114_phase4_reconciliation.sql` | Task 6 | `payment_reconciliation`, `payment_reconciliation_line` |
| `0115` | `0115_phase4_content_seo.sql` | Task 8 | `content_page`, `content_page_translation`, `redirect_rule`, `sale_condition` |

Les 9 autres tâches (2, 5, 7, 9, 10, 11, 12, 13, 14) **ne créent aucune migration** : elles s'appuient sur le modèle existant (`audit_log`, `dispute`/`dispute_message`, `country_profile`, `incoterm_configuration`, `trade_rule`, `business_account`, `admin_action_approval`, `notification`, `notification_preference`, `export_request`, `push_subscription`).

**Cohérence avec les phases 1 à 3** : les tables Phase 2 (`orders`, `order_line`, `order_status_event`, `payment`, `refund`, `invoice`, `product`, `product_variant`, `price_tier`, `inventory_item`, `inventory_movement`, `warehouse`, `feature_flag`) et Phase 3 (`country_profile`, `incoterm_configuration`, `trade_rule`, `trade_rule_version`, `compliance_alert`, `dispute`, `dispute_message`, `shipment`, `tracking_event`, `business_account`, `business_account_approval`, `customer_price_rule`) sont **lues, jamais réécrites** par la phase 4. Les seules evolutions de schéma sont les 5 migrations listées ci-dessus, toutes en `ALTER`/`CREATE` extensif, sans retrait de colonne ni de table.

**Divergence assumée à signaler** : la Phase 2 place les dictionnaires dans `packages/i18n/src/dictionaries/*.ts`, la Phase 3 dans `packages/i18n/src/locales/{fr-FR,en,zh-CN}.json`. La Phase 4 suit la **convention Phase 3** (`locales/*.json`), qui est la plus récente et permet le contrôle de parité des clés en build. La convergence des deux conventions est inscrite dans le §3 (limites) de cette auto-revue.

### 2. Couverture des critères d'acceptation et écarts assumés

**Critères couverts par des preuves dans le plan** :

- *Le front n'est jamais source de vérité* — chaque écran admin lit l'API serveur ; les prix, taxes, frais, stock et autorisations sont recalculés côté serveur (Tasks 1, 2, 3, 4, 5, 6, 7).
- *Idempotence* — clés d'idempotence explicites sur les opérations sensibles : transitions de commande, import catalogue, action groupée, rapprochement (`payment.reconcile:<date>:<provider>`), export (`audit.export:<actor>:<hash>`), notification (`notification:<id>`), remboursement (`paymentId + amountMinor + ticketId`).
- *Anti-IDOR* — scoping dans la clause `WHERE` à chaque tâche admin, jamais de post-filtrage ; ressource absente ⇒ `404` (Tasks 2, 5, 9, 13) ; test de non-régression sur 14 actions (Task 14).
- *Quatre-yeux* — `price_tier_change`, `financial_export`, `refund`, `role_change`, `country_suspension` ; le demandeur ne peut pas être l'approbateur (Tasks 3, 5, 6, 9, 13).
- *MFA* — obligatoire et revalidé à 10 minutes sur les actions privilégiées (Tasks 6, 10, 13).
- *Journal d'audit append-only* — 12 familles du spec §19.7, catalogue fermé, redaction des secrets, chaîne SHA-256 vérifiable (Tasks 2, 10, 13).
- *Zéro donnée de carte stockée ou journalisée* — aucun DTO, aucun log et aucun export ne contient de PAN, CVC ou token de prestataire ; mentionné dans les Tasks 6, 10, 12, 13.
- *Règles douanières = données* — aucune taux en dur ; garde CI `check-no-hardcoded-rates.mjs` existant, aucun nouveau taux introduit par la phase 4 (Task 7).
- *Documents B2B* — chiffrés, bucket privé, URL signée 5 min, accès tracé ligne par ligne (Task 5).
- *i18n FR/EN/ZH et 2 thèmes* — chaque tâche livrant des chaînes ajoute ses clés dans les 3 locales et passe `check:keys`.
- *Accessibilité* — états obligatoires (loading / vide / erreur / accès refusé / session expirée), cibles ≥ 44 px, focus visible, aucune information portée par la couleur seule, axe-core sur les 2 dashboards (Task 14) **plus** la checklist manuelle (l'audit auto ne prouve rien).
- *`noindex`* — pages admin, compte, panier et checkout en `noindex, nofollow` + `X-Robots-Tag`, exclus des sitemaps (Tasks 2, 13).

**Écarts assumés, gaps déclarés honnêtement** :

1. **Marge brute et taux de conversion indisponibles** (Task 11) — aucun coût d'achat n'existe dans le modèle et aucun outil analytics n'est choisi. L'API renvoie `state: 'unavailable'` avec la raison ; **aucune estimation n'est inventée**. Impact : deux KPI sur douze ne sont pas exploitables avant la décision propriétaire n° 12.
2. **Aucun index sur `audit_log(action)`** — le filtre par préfixe sur une table append-only de grande taille est un balayage. La recherche admin du journal est donc lente au-delà de ~10⁶ lignes. Reporté à la phase 6 ; documenté ici plutôt que masqué.
3. **Provider push réel non livré** — le service push est implémenté et instrumenté, mais l'envoi navigateur dépend d'un fournisseur (VAPID) dont les clés ne sont pas décidées ; en attendant, `PushService.send()` renvoie `skipped: true` avec la raison. Aucun secret n'est requis pour que la phase soit verte.
4. **Canal SMS livré en interface + mock** — le spec §14.1 le place en « option ultérieure, si coût, consentement et conformité le permettent ». Le canal est désactivé par défaut ; aucun numéro ne part vers un tiers avant décision.
5. **Vérification manuelle a11y non automatisable** — la checklist `ops/checklists/phase4-manual-a11y.md` doit être **exécutée par un humain** sur les parcours critiques (connexion admin, MFA, dashboard client, détail de commande, rapprochement, ticket support). Le plan organise la passation ; il ne peut pas la prouver.
6. **Génération de PDF non refaite** — la phase 4 consomme les PDF de la Phase 2 (factures) ; aucune modification du générateur n'est prévue, donc le test « PDF accessible » n'est pas rejoué dans cette phase.
7. **Révision de règle douanière** — le planning de revalidation est une constante de 180 jours **à valider** ; le mécanisme (alerte + blocage) est implémenté, la durée reste un paramètre métier non figé par le spec.
8. **Relecture humaine des traductions** — le plan garantit la **parité des clés** et la présence des 3 locales, pas la qualité rédactionnelle des 3 langues. La relecture humaine des terms critiques reste requise (risque « traduction imprécise », §28.1).

### 3. Risques, limites et décisions restant au propriétaire

**Valeurs marquées « à valider (décision propriétaire) » dans le plan** :

| Valeur | Valeur retenue dans le plan | Où | Impact si le propriétaire tranche autrement |
|---|---|---|---|
| `BULK_ADJUSTMENT_LIMIT` | 100 | Task 4 | limite de correction massive de stock ; un autre seuil change le nombre de confirmations requises |
| `RULE_REVIEW_INTERVAL_DAYS` | 180 | Task 7 | cadence d'alerte de revalidation des règles douanières |
| Durée de conservation d'un export | 24 h | Tasks 6, 10 | durée de validité du fichier exporté (l'URL signée reste à 5 min) |
| Cibles SLA support | critical 4 h / high 8 h / normal 24 h / low 72 h | Task 9 | affichage du SLA et déclenchement des alertes de dépassement |
| Prestataire SMS | aucun (canal désactivé) | Task 12 | coût, conformité et consentement de l'envoi SMS |
| Prestataire push (VAPID) | aucun (envoi différé) | Task 12 | notifications navigateur ; nécessite une décision de clé et d'hébergement |
| Outil analytics tiers | aucun (KPI non mesurables) | Task 11 | `conversion_rate` et toute donnée de trafic |
| Coûts d'achat (marge brute) | non modélisés | Task 11 | `gross_margin_rate` reste indisponible |

**Décisions propriétaire du spec §28.2 touchées par la phase 4** (les 12) :

| # | Décision §28.2 | Ce que la phase 4 en fait | Tâches |
|---|---|---|---|
| 1 | Entité, pays, prestataire de paiement admissible | le rapprochement quotidien suppose un prestataire joignable par API ; aucun prestataire n'est codé en dur | 6, 12 |
| 2 | Produits vendus et catégories réglementées | l'import catalogue valide `hs_code` et `country_of_origin` mais ne décide pas de la conformité produit | 3, 7 |
| 3 | Pays prioritaires et pays interdits | la suspension d'un pays (Task 7) et l'exclusion B2B (Task 5) dépendent de cette liste | 5, 7 |
| 4 | Incoterms et paiement des taxes (prépayé / à l'arrivée) | `incoterm_configuration.tax_collection_mode` est modélisé ; les valeurs par pays restent à fixer | 7 |
| 5 | Transporteurs, suivi et API | la matrice logistique et les exceptions de suivi dépendent des transporteurs retenus | 7 |
| 6 | Devise de référence, devises d'encaissement, change | la devise d'affichage est « indicative » et distincte de la devise de transaction ; les taux de change ne sont pas définis ici | 1, 6, 11 |
| 7 | Politique B2B (MOQ, acomptes, crédit, devis) | le MOQ, les paliers négociés et les devis sont gérés ; acomptes et crédit ne le sont pas | 3, 5 |
| 8 | Politique B2C, retours, garanties, annulations | les déclencheurs `return.*` existent ; la politique de retour elle-même n'est pas définie | 2, 9, 12 |
| 9 | Exigences de facturation applicables | les conditions de vente sont versionnées par segment et locale ; les mentions légales restent à valider | 8 |
| 10 | Hébergement, région des données, transferts | aucun transfert de données hors région n'est mis en œuvre ; les exports restent dans le bucket privé de la région | 2, 10, 12 |
| 11 | SLA, disponibilité, RPO/RTO, calendrier de support | le SLA support est paramétré ; RPO/RTO et le calendrier d'astreinte relèvent des runbooks | 9, 12, 14 |
| 12 | Outils d'analytics, consentement, conservation | le consentement push/marketing est implémenté ; l'outil analytics tiers et la durée de conservation ne le sont pas | 11, 12 |

**Risques résiduels de la phase 4** :

- *Dérive entre la matrice de permissions et le code* — mitigated par le test paramétrique de la Task 13 (22 assertions issues de la matrice §3.2) ; toute évolution de la matrice doit modifier le test dans le même commit.
- *Régression inter-phase corrigée : le test de permissions de la Phase 1 cassait à l'extension RBAC de la phase 4* — `apps/api/src/modules/rbac/__tests__/permissions.test.ts` figeait `expect(RESOURCES).toHaveLength(13)` et `expect(ACTIONS).toHaveLength(8)`. La phase 4 ajoute 5 ressources (`stock`, `content_page`, `support_ticket`, `analytics`, `notification` → **13 → 18**) et 5 actions (`publish`, `assign`, `escalate`, `refund_request`, `send` → **8 → 13**) : les deux assertions en dur cassaient dès la Task 4. Correction retenue : les comptes codés en dur sont remplacés par des listes attendues explicites `EXPECTED_RESOURCES` / `EXPECTED_ACTIONS`, **mises à jour dans le commit de chaque tâche qui ajoute une ressource** (Task 4 → 14 ressources, Task 8 → 15, Task 9 → 16, Task 11 → 17, Task 12 → 18 ; le vocabulaire d'actions est déclaré en entier dès la Task 4 parce que `ck_permission_action` n'est réécrite que par `0113`). En base, `0113` recrée `ck_permission_resource` (18 valeurs) et `ck_permission_action` (13 valeurs) en `DROP CONSTRAINT` + `ADD CONSTRAINT` — **aucune donnée n'est supprimée**, la contrainte est élargie et non réécrite à l'identique. Corrigé au passage : `packages/db/src/__tests__/migrations.integration.test.ts` figeait le nombre de migrations en dur (`expect(first.length).toBe(4)`, porté à `5` par la Task 2 de la Phase 1, et `expect(rows.rows).toHaveLength(4)`, jamais mis à jour) — déjà faux depuis la Phase 2 (15 fichiers) puis la Phase 3 (26 fichiers) ; il est défigé (`files.length`, soit **31** à la fin de la phase 4). Aucune autre assertion de Phase 1, 2 ou 3 n'est cassée : le test « crée les tables attendues » n'utilise que `toContain` (présence, pas nombre), le seed ne fige aucun nombre de lignes `permission`, et les tests d'immuabilité de `audit_log` ne touchent pas la matrice.
- *Détail du mécanisme de la demande de remboursement du support* — `FOUR_EYES` reste aux **8 couples de la sécurité §3.3** (inchangés) : ajouter `support_agent:refund:refund` aurait cassé la boucle de test de la Phase 1 sur `FOUR_EYES` (`requiresApproval` + `approvalKindFor`) et aurait donné à `support_agent` une action `refund` sur la ressource `refund`, ce que la matrice §3.2 interdit. Le mécanisme retenu est conforme au spec : le support **demande** via `support_ticket:refund_request` (Task 9), la demande est une ligne `admin_action_approval` de kind `refund` avec **requester ≠ approver** (`no_self_approval`), et `finance_manager` / `super_admin` **exécutent** le remboursement avec leur action `refund` existante, elle-même à quatre-yeux.
- *Fuite par un export* — mitigated par la liste blanche de champs, l'URL signée 5 min, l'absence d'export pour `analyst_readonly`, l'audit et la revue sécurité de la Task 14. Le risque résiduel est l'export de données personnelles par un rôle autorisé : il est tracé, pas interdit.
- *Écart de rapprochement non traité* — un écart `open` non annoté devient un risque financier. Le runbook `reconciliation-daily.md` et l'alertemetric sont la mitigation ; le délai de traitement maximal reste à fixer par le propriétaire.
- *Règle douanière périmée appliquée en silence* — le moteur bloque l'application silencieuse et alerte le compliance manager ; l'ancienneté maximale d'une règle reste un paramètre métier.
- *Régression induite par la factorisation de la Task 13* — mitigated par l'exécution de la suite complète (412 tests API) avant et après ; toute divergence fonctionnelle est un blocage, pas un « ajustement ».

**Ce qui n'est pas décidé par ce plan** : toute donnée de carte réelle, tout secret, toute valeur de prestataire, tout taux de taxe ou de devise, et toute liste de pays. Le plan ne contient aucune de ces valeurs ; il contient des **points d'extension** et des **valeurs candidates explicitement marquées** à valider.

### 4. Checklist de vérification exécutable

Cette checklist est à exécuter dans l'ordre, sur la branche de la phase 4, avant toute demande de merge. Chaque ligne doit être **accompagnée de sa preuve** (sortie de commande ou rapport de test) ; une ligne non prouvée est une ligne non faite.

```bash
# 0. Contexte — branche, statut, PR
git status --short
git branch --show-current
gh pr status

# 1. Migrations — ordre, checksum, aucune destructivité, application sur base de test
ls packages/db/migrations | sort
pnpm db:check-migrations          # numérotation contiguë 0111→0115, une seule fois, checksum SHA-256
pnpm db:migrate                   # base de test uniquement
pnpm db:verify                    # schéma appliqué = schéma attendu

# 2. Qualité — format, lint, types stricts, tests
pnpm format:check
pnpm lint
pnpm typecheck
pnpm --filter @fanours/api test
pnpm --filter @fanours/web test
pnpm --filter @fanours/analytics test
pnpm --filter @fanours/i18n check:keys

# 3. Garde-fous spécifiques
pnpm exec node scripts/check-no-hardcoded-rates.mjs     # aucun taux de taxe/douane/frais en dur
pnpm exec node scripts/check-no-secret-patterns.mjs     # aucun secret dans le code
pnpm exec gitleaks detect --no-banner --redact
pnpm audit --audit-level=high
pnpm exec semgrep scan --config auto --error

# 4. E2E — 11 cas admin, a11y, non-régression sécurité, exports
pnpm exec playwright test e2e/admin/dashboard.spec.ts --project=admin   # suite de release admin (11 cas)
pnpm exec playwright test e2e/admin --project=functional                # autres specs admin de la phase
pnpm exec playwright test e2e/specs/security.spec.ts --project=functional # non-régression sécurité, phases 1 à 3
pnpm exec playwright test e2e/account --project=functional              # dashboard client

# 5. Responsive et thèmes — 8 viewports × 3 langues × 2 thèmes, zéro débordement horizontal
#    les 8 viewports sont itérés DANS le spec (test.use({ viewport })) à partir de la liste
#    canonique e2e/support/viewports.ts (importée, jamais re-déclarée) ; les 3 langues × 2 thèmes
#    sont itérés dans le même test.describe — les viewports ne sont pas des projets Playwright
pnpm exec playwright test e2e/visual/responsive.spec.ts --project=visual

# 6. Manual — à exécuter par un humain, preuve en pièce jointe
#    ops/checklists/phase4-manual-a11y.md  (clavier + lecteur d'écran, parcours critiques)
#    ops/runbooks/admin-backoffice.md        (déroulé complet avec un rôle super_admin)
#    ops/runbooks/reconciliation-daily.md    (cycle quotidien, y compris un écart réel)

# 7. Revue sécurité ciblée de la phase 4
#    - exports : liste blanche, URL 5 min, analyst_readonly refusé
#    - rapprochement : contrôle serveur des sommes, quatre-yeux, journalisation
#    - documents B2B : chiffrement, bucket privé, traçabilité d'accès
#    - journal : immuabilité, chaîne SHA-256, redaction

# 8. Livraison
#    - PR ouverte, jamais mergée sans accord explicite
#    - description en français : tests, résultats, captures, limites
#    - 2 approbations requises ; 1 approbation + revue sécurité si chemin sensible
#    - aucun commit sur main, aucun force-push
```

**Points de contrôle bloquants** (la checklist échoue si l'un d'eux est faux) :

- [ ] Les 5 migrations `0111`→`0115` sont appliquées sur une base de test, avec checksum, et **aucune** n'est destructive sans accord explicite et restauration prouvée.
- [ ] Aucun secret, aucune donnée de carte réelle, aucun document client réel dans le code, les logs, les tests ou les fixtures.
- [ ] Les 3 locales sont à parité de clés et les 2 thèmes sont couverts pour tout nouveau composant.
- [ ] Les 14 actions sensibles de la matrice §3.2 ont une assertion (22 lignes du test paramétrique) et l'E2E correspondant.
- [ ] Zéro violation axe-core critique ou sérieuse sur les 2 dashboards (3 langues, 2 thèmes).
- [ ] La matrice `e2e/visual/responsive.spec.ts` (projet `visual`) est verte : 8 viewports × 3 langues × 2 thèmes × 2 pages, viewports importés de `e2e/support/viewports.ts` (jamais re-déclarés) et zéro débordement horizontal.
- [ ] Le plan de vérification manuelle a11y est **exécuté et joint** — pas seulement rédigé.
- [ ] Les écarts listés au §2 (KPI indisponibles, index journal, push, SMS, rélecture humaine) sont **explicitement acceptés** par le propriétaire ou treated comme bloqueurs.
- [ ] La PR est ouverte avec preuves, et **aucun merge n'a été effectué**.

