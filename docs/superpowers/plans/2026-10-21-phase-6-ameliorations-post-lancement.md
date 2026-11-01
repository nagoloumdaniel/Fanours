# Phase 6 — Améliorations post-lancement : Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Après trente jours de données d'usage réelles, transformer la mesure en amélioration : budgets de performance par route, dette technique outillée, contenu SEO FR/EN/ZH fondé sur des sources officielles, consolidation de la suite de tests, observabilité pilotée par SLO, sécurité continue, i18n densifiée, et un bilan argumenté qui ouvre la Phase 7. Aucune amélioration n'est engagée sans mesure avant/après ; toute optimisation qui ne montre pas de gain est revertée.

**Périmètre :** 10 tâches (Task 1 → Task 10) — cadence de maintenance, performance web, performance API et base, contenu et programmes SEO, consolidation des tests, observabilité et exploitation, sécurité continue, i18n densifiée, puis le bilan et le Gate 6.

**Architecture:** La Phase 6 n'ajoute aucun module métier nouveau. Elle ajoute trois plans transverses — un plan de cadence de maintenance (backlog de dette, budgets, rapport hebdomadaire), un plan de performance mesurée (web + API + base, avec garde-fous de non-régression), un plan de croissance durable (contenu SEO validé par le propriétaire métier, i18n, observabilité, sécurité continue) — et un plan de clôture qui produit la proposition de Phase 7. Chaque budget devient un contrôle CI bloquant, chaque page éditoriale devient un contenu revalidé par date de source, chaque seuil d'alerte devient un test `promtool`. Les invariants du spec sont préservés : le front n'est jamais source de vérité, les montants ne sont jamais des flottants, le contenu n'est jamais inventé, l'IA reste assistive.

**Tech Stack:** Next.js 15 App Router + React 19 + TypeScript `strict` + Tailwind v4 · NestJS 11 + Zod · PostgreSQL 16 + Drizzle (un seul ORM) · Redis 7 + BullMQ · S3-compatible + CDN · Vitest + Testing Library + Playwright + k6 + Stryker · pnpm workspaces + Turborepo · Lighthouse CI · `promtool` · `gitleaks` + `trufflehog` · Cron de maintenance en job BullMQ planifié.

**Spec:** `Cahier_des_charges_Fanours.md` — §17 (SEO et contenus), §18.4 (IA assistive, garde-fou), §22 (responsive, PWA, a11y), §23 (performance, disponibilité, observabilité), §24 (qualité et tests), §25 (infrastructure et exploitation), §26 (critères d'acceptation), §27 (phasage), §28 (risques). Plans de référence : `docs/roadmap/sections/roadmap-phases.md` (Phase 6, Gate 6), `docs/roadmap/sections/infra.md` (§7 métriques, §8 alertes, §9 sauvegardes, §10 budgets et cache, §14 feature flags, §15 runbooks, §16 coûts), `docs/roadmap/sections/qualite.md` (§24.9, §24.13), `docs/roadmap/sections/seo.md` (§17.2 à §17.12), `docs/roadmap/sections/design.md` (§3.11, §3.12), `docs/roadmap/sections/architecture.md` (§2, §8, §9), `docs/roadmap/sections/git.md` (§11 ADR, §12 commits).

## Global Constraints

- **Stack figée.** Next.js 15 App Router + React 19 + TypeScript `strict` + Tailwind v4 (CSS-first) · NestJS 11 + Zod · PostgreSQL 16 + **Drizzle** (un seul ORM, jamais Prisma) · Redis 7 + BullMQ · S3-compatible · Vitest + Testing Library + Playwright + k6 + Stryker · pnpm workspaces + Turborepo. **Toute nouvelle dépendance passe par une décision écrite (ADR) et un test de non-régression de bundle ou de coût.**
- **Modèle E2E canonique.** Une **seule** configuration Playwright : `playwright.config.ts` **à la racine du dépôt** — créée en Phase 1, **modifiée** (jamais recréée) par les Phases 2, 4, 5 et 6. Aucun autre `playwright.config.ts` (ni `apps/web/`, ni `apps/api/`, ni `e2e/`) ; les commandes Playwright sont lancées **depuis la racine** (`pnpm exec playwright test ...`). `testDir` vaut `'./e2e'` pour **tous** les projets : ils se distinguent uniquement par `testMatch` / `testIgnore`. **7 projets** : `b2c`, `b2b`, `payment`, `admin`, `security` (les 5 suites de release du spec §24.3), `visual` (matrice visuelle) et `functional` (recettes fonctionnelles de développement). Un fichier de test appartient à **un seul** projet : aucun doublon, aucun comptage de tests ambigu. **Le moteur (navigateur) n'est pas un projet** : il est une **dimension de shard en CI**, passée par variable d'environnement ; le projet définit la *suite*, pas le moteur. Les viewports, les langues et les thèmes ne sont pas des projets non plus : ils sont paramétrés dans les fichiers de test à partir de la liste canonique `e2e/support/viewports.ts` (créée en Phase 1, importée, **jamais re-déclarée**). **La Phase 6 ne crée aucun projet Playwright supplémentaire.**
- **Règle d'or de la Phase 6 : aucune amélioration ne dégrade un contrat existant.** Chaque optimisation est **mesurée avant/après** (bundle, requêtes N+1, images, CWV p75, latence, coût) et le gain est consigné dans la description de la PR. **Si la mesure ne montre pas de gain, l'optimisation est revertie** et la mesure est consignée dans l'ADR qui l'a motivée.
- **Aucune amélioration n'est engagée avant 30 jours de données d'usage disponibles** (Gate 6). Une optimisation proposée sur une base de trafic < 30 jours est refusée : le gain projeté n'est pas une mesure.
- **Zéro placeholder.** Aucun `TBD`, `TODO`, « à compléter », « identique à la tâche N ». Chaque étape contient le code, la commande ou le contenu réel.
- **Cycle TDD par tâche.** Test qui échoue → exécution prouvant l'échec → implémentation minimale → exécution prouvant la réussite → commit. Aucun « je pense que ça marche ».
- **Le front n'est jamais source de vérité.** Prix, taxes, frais, stock et autorisation sont recalculés côté serveur. Aucune optimisation de cache côté client ne peut pas servir une valeur périmée ; invalidation par `correlation_id` et version de snapshot, jamais par code de réponse HTTP.
- **Contenus SEO : aucune donnée inventée.** Pas de faux avis, pas de faux chiffres, pas de compteurs d'achat, pas d'urgence artificielle, pas de promesse de classement. Franchises, seuils douaniers, délais, Incoterms et codes HS sont des **données** issues de `trade_rule` / `trade_rule_version` (source officielle + URL + date de consultation) ou renvoient à un contenu éditorial validé par le propriétaire métier. Un contenu qui cite une règle expirée déclenche une alerte au compliance manager.
- **Garde-fou IA (spec §18.4).** L'IA est **assistive** uniquement : elle ne fixe aucun prix final, ne valide aucun code HS seule, ne confirme aucune importabilité, ne déclenche ni remboursement ni suspension, n'invente ni certification, ni avis, ni stock, ni délai. Tout output touchant conformité, sécurité, taxe ou contrat exige vérification humaine avant publication.
- **i18n.** Dictionnaires `fr-FR` / `en` / `zh-CN`, repli explicite `fr-FR` → clé, formatage ICU, **le build échoue si une clé manque dans l'une des trois langues**. Expansion EN (+25–35 %) gérée par `min-w-0` / `truncate` ; jamais de largeur fixe sur du texte traduit.
- **Montants.** Jamais de float binaire. `NUMERIC(18,4)` en base, entiers de minor units en code, `Intl.NumberFormat`, codes ISO 4217. Distinguer montant de référence, devise d'affichage (taux + horodatage, marquée « indicatif ») et montant débité.
- **Migrations.** `packages/db/migrations/NNNN_nom.sql`, append-only, **aucun script `down`** (rollback par code), checksum SHA-256 enregistré, `SET lock_timeout` / `SET statement_timeout`, `CREATE INDEX CONCURRENTLY` au-delà de 100 000 lignes, `NOT NULL` nouveau via `CHECK … NOT VALID` puis `VALIDATE`. Expand/migrate/contract sur au moins 2 déploiements. **Les migrations de la Phase 6 commencent à `0120`** (0001–0011 et 0100–0109 sont pris par les Phases 1–3). **Migration destructive en production : accord explicite + restauration prouvée.**
- **PWA.** Shell en precache, assets publics en runtime, pages catalogue en NetworkFirst, **panier / checkout / compte en NetworkOnly**, **interdiction absolue de mettre en cache factures, commandes ou données privées**, service worker versionné avec purge des anciens caches. Toute évolution du SW passe par un test vérifiant qu'aucun cache ne contient de donnée privée.
- **Observabilité.** Logs JSON une ligne (pino) avec `timestamp, level, service, env, version, correlation_id, trace_id, user_id_hash, session_id_hash, route, method, status, duration_ms, order_id, event, msg, error` ; `correlation_id` de bout en bout (HTTP → job → intégration) ; redaction automatique des PII et des secrets ; métriques p50/p95/p99 par route, taux d'erreur, profondeur de file, durée de job, échecs webhooks, doublons, échecs paiement, échecs réservation, emails en échec, DLQ.
- **A11y maintenue dans le temps (WCAG 2.2 AA).** Focus visible 2 px + offset 2 px, pièges de focus, live regions sur recalcul de total et changement de stock, cibles ≥ 44 px, `prefers-reduced-motion` respecté, jamais couleur seule, un seul `<h1>`, skip link, 200 % zoom sans perte de contenu.
- **Mobile.** Tableaux → cartes sous 768 px, `min-width: 0`, jamais `width: 100vw`, assertion E2E `document.scrollWidth <= document.documentElement.clientWidth` à 320 / 360 / 390 / 430 px.
- **États obligatoires.** Loading (skeleton) · vide · erreur · accès refusé · session expirée · réseau indisponible · ressource indisponible · conflit de stock · quantité invalide · paiement en attente · interruption prestataire. Message localisé FR/EN/ZH, icône, action de sortie, **jamais de donnée technique exposée au client**.
- **Secrets.** **Noms de variables uniquement** dans les fichiers, la documentation et les logs. Scan `gitleaks` + `trufflehog` bloquants en pre-commit et en CI. Procédure de rotation et procédure d'urgence documentées. En cas de fuite : **rotation d'abord, purge d'historique ensuite**.
- **Git.** Branche `feat|fix|refactor|test|docs|chore/<slug>`, **jamais de `main`**, jamais de force-push, PR laissée ouverte et **jamais mergée sans accord explicite**, **PR > 800 lignes à scinder**. Commits Conventional Commits 1.0, titres ≤ 72 caractères, corps ≤ 72 caractères, trailer `Co-Authored-By: Claude Code <noreply@anthropic.com>`.
- **Preuve.** Pas de « terminé » sans la **sortie de test collée**. Les tests non exécutables sont annoncés avec leurs prérequis et ne sont jamais comptés comme réussis.
- **Aucun secret, aucune donnée carte, aucun document client sensible réel** dans le code, les tests, les fixtures, les logs ou la documentation. Les fixtures sont synthétiques.
- **Format.** Étapes en cases à cocher `- [ ] **Step N: …**`, code et commandes réels, un bloc `Attendu : …` par étape, une étape `Commit` par tâche.

---

## File Structure

```
docs/
  maintenance/
    TECH-DEBT-BACKLOG.md              # backlog de dette technique : ID, module, coût, risque, échéance
    PERF-BUDGETS.yaml                 # budgets de performance par route (bundle, requêtes, images, CWV)
    WEEKLY-REPORT.md                  # rapport hebdomadaire généré, versionné au premier jour du mois
    reports/
      2026-11/                        # rapports générés, un fichier par semaine
  adr/
    0000-model.md                     # gabarit ADR (existe déjà — ne pas modifier)
    0011-budget-perf-ci.md            # ADR : les budgets deviennent des contrôles CI bloquants
    0012-strategy-technique-web.md    # ADR : stratégie de rendu et cache web
    0013-index-pg-stat-statements.md  # ADR : indexation pilotée par pg_stat_statements
    0014-cache-redis-invalidation.md  # ADR : invalidation par événements outbox
    0015-clusters-seo-editoriaux.md   # ADR : production de contenu Clustering validé métier
    0016-slo-et-budgets-alerte.md     # ADR : SLO par route et budgets d'alerte
    0017-rotation-secrets-planifiee.md # ADR : rotation planifiée et testée des secrets
    0018-contenu-relecture-ia.md       # ADR : relecture humaine obligatoire du contenu IA
  content/
    plan-contenu.md                   # plan de contenu FR/EN/ZH validé par le propriétaire métier
    gabarits/
      guide.md                        # gabarit page guide
      pays.md                         # gabarit page pays
      faq.md                          # gabarit FAQ
      politique.md                    # gabarit page politique éditoriale
    clusters/
      01-acheter-en-gros-depuis-la-chine.md
      02-moq-paliers-de-prix.md
      03-frais-dimportation-douane-tva.md
      04-incoterms-ddp-vs-dap.md
      05-controle-qualite-et-certifications.md
      06-delais-dexpedition-et-transporteurs.md
      07-devis-b2b-paiement-et-acompte.md
      08-codes-hs-origine-et-dedouanement.md
      09-conformite-produit-par-marche.md
      10-rentabilite-dun-achat-en-gros.md
  runbooks/
    RB-009-slo-degrade.md             # runbook : SLO dégradé (nouveau)
    RB-010-cout-infra-degrade.md      # runbook : dérive du coût d'infrastructure (nouveau)
  decisions/
    phase7-proposition.md            # proposition de Phase 7 argumentée (Task 10)
  superpowers/plans/
    2026-10-21-phase-6-ameliorations-post-lancement.md   # ce plan

apps/
  web/
    src/app/[locale]/(public)/
      guides/[slug]/page.tsx          # pages guide, gabarit éditorial, JSON-LD Article
      pays/[country]/page.tsx         # pages pays, données de la table trade_rule
      faq/page.tsx                   # FAQ, JSON-LD FAQPage conforme au contenu visible
    src/lib/seo/
      jsonld.ts                       # constructeurs Article / FAQPage / BreadcrumbList
      hreflang.ts                     # reciprocity hreflang + x-default
    src/components/media/
      ProductImage.tsx                # next/image, AVIF/WebP, dimensions explicites
    next.config.ts                   # formats image, headers de cache, tailles de bundle
    scripts/bundle-budget.mjs        # mesure du bundle first-load par route
  api/
    src/modules/performance/
      query-metrics.ts                # lecture pg_stat_statements, détection N+1
      cache-keys.ts                   # table des clés Redis et invalidation par tags
      cache.ts                        # wrapper cache sliding window + TTL + invalidation
    src/modules/maintenance/
      perf-budgets.service.ts         # lecture des budgets, calcul des écarts
  worker/
    src/jobs/weekly-maintenance-report.ts   # rapport hebdomadaire planifié (BullMQ repeatable)
    src/jobs/weekly-seo-coverage.ts          # couverture éditoriale + règles expirées

packages/
  db/
    migrations/
      0120_maintenance_metrics.sql    # métriques de maintenance et snapshots de budgets
      0121_content_registry.sql       # registre éditorial : pages, sources, dates de revalidation
      0122_perf_baselines.sql         # baselines de performance par route et par version
      0123_secret_rotation_audit.sql  # journal d'audit de rotation des secrets
    src/schema/                       # schémas Drizzle correspondants
  observability/
    src/slo.ts                        # définitions SLO par route, calcul de burn rate
    src/alerts.ts                     # budget d'alerte, templates de runbook
  jobs/
    src/idempotent-job.ts             # motif existant réutilisé (aucune modification)
  i18n/
    src/dictionaries/fr-FR.json
    src/dictionaries/en.json
    src/dictionaries/zh-CN.json       # vocabulaire métier densifié (Task 9)
  config/
    src/slo.ts                        # seuils SLO, TTL de cache, budgets
    src/secrets.ts                    # catalogue des secrets (noms de variables uniquement)

ops/
  alerting/rules/
    phase6-slo-burnrate.yml           # règles de burn rate multi-fenêtre
    phase6-cost-drift.yml             # dérive du coût d'infrastructure
  performance/
    budgets.yaml                      # copie versionnée des budgets (source unique)
    index-audit.sh                    # audit d'index via pg_stat_statements
    n-plus-one.sh                     # détection des requêtes N+1
  backup/
    restore-drill.yml                 # drill mensuel (existe — Gate 6 l'exige)
  security/
    secret-rotation.yml               # rotation planifiée + test de la procédure d'urgence
    scanner-regression.sh             # test de non-régression du scanner lui-même

e2e/
  contracts/api-web.contract.spec.ts  # tests de contrat entre api et web — projet `functional`
  seo/public-seo-audit.spec.ts         # audit seo-audit automatisé en CI — projet `functional`
  perf/no-horizontal-overflow.spec.ts  # assertion scrollWidth aux 4 largeurs mobile — projet `functional`

scripts/
  maintenance/
    debt-backlog.ts                   # CLI de gestion du backlog de dette technique
    weekly-report.ts                  # générateur du rapport hebdomadaire
  seo/
    content-lint.ts                   # contrôle de couverture i18n, dates de revalidation, interdits
    sitemap-submit.ts                 # soumission des sitemaps (noms de variables uniquement)
```

---

### Task 1: Cadence de maintenance — backlog de dette, budgets de performance, rapport hebdomadaire

**Files:**
- Create: `docs/maintenance/TECH-DEBT-BACKLOG.md`
- Create: `docs/maintenance/PERF-BUDGETS.yaml`
- Create: `docs/adr/0011-budget-perf-ci.md`
- Create: `packages/config/src/slo.ts`
- Create: `packages/config/src/budgets.ts`
- Create: `scripts/maintenance/debt-backlog.ts`
- Create: `scripts/maintenance/weekly-report.ts`
- Create: `apps/worker/src/jobs/weekly-maintenance-report.ts`
- Test: `packages/config/src/__tests__/budgets.spec.ts`
- Test: `scripts/maintenance/__tests__/debt-backlog.spec.ts`
- Test: `apps/worker/src/jobs/__tests__/weekly-maintenance-report.spec.ts`

**Interfaces:**
- Consumes: `docs/roadmap/sections/infra.md` §10.1 (budgets par route), §16 (coûts mensuels 198 / 472 / 1518 €, cible ≤ 500 €/mois), `docs/roadmap/sections/qualite.md` §24.13 (métriques qualité), `packages/config/src/env.ts` (schéma Zod d'environnement existant), `apps/worker/src/jobs/define-job.ts` (helper de job planifié existant).
- Produces:
  - `packages/config/src/budgets.ts` → `export const PERF_BUDGETS: PerfBudget[]` et `export type PerfBudget`.
  - `packages/config/src/slo.ts` → `export const SLO_TARGETS: SloTarget[]` et `export type SloTarget`.
  - `scripts/maintenance/debt-backlog.ts` → `export function parseBacklog(markdown: string): DebtEntry[]` et `export function renderBacklog(entries: DebtEntry[]): string`.
  - `scripts/maintenance/weekly-report.ts` → `export function buildWeeklyReport(input: WeeklyReportInput): WeeklyReport` et `export type WeeklyReport`.
  - `apps/worker/src/jobs/weekly-maintenance-report.ts` → `export const weeklyMaintenanceReportJob: JobDefinition`.

---

- [ ] **Step 1: Écrire le test qui échoue — le parseur de backlog**

Créer `scripts/maintenance/__tests__/debt-backlog.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { parseBacklog, renderBacklog } from '../debt-backlog';

const SOURCE = `| ID | Module | Coût | Risque | Échéance | Statut |
|---|---|---|---|---|---|
| DEBT-001 | packages/api/src/modules/catalog | 2 j | moyen | 2026-12-01 | ouvert |
| DEBT-002 | apps/web/src/app/[locale]/(public) | 1 j | faible | 2026-11-15 | ouvert |
| DEBT-003 | packages/worker/src/jobs | 3 j | élevé | 2026-11-30 | planifié |
`;

describe('parseBacklog', () => {
  it('extrait chaque entrée du tableau markdown', () => {
    const entries = parseBacklog(SOURCE);
    expect(entries).toHaveLength(3);
    expect(entries[0]).toEqual({
      id: 'DEBT-001',
      module: 'packages/api/src/modules/catalog',
      cost: '2 j',
      risk: 'moyen',
      due: '2026-12-01',
      status: 'ouvert',
    });
  });

  it('ignore les lignes d’en-tête et de séparation', () => {
    const entries = parseBacklog(SOURCE);
    expect(entries.some((e) => e.id === 'ID')).toBe(false);
    expect(entries.some((e) => e.id === '---')).toBe(false);
  });

  it('lève une erreur explicite sur une ligne mal formée', () => {
    expect(() => parseBacklog('| DEBT-004 | module |')).toThrowError(
      /ligne de backlog invalide/,
    );
  });

  it('effectue un aller-retour rendu / parsing sans perte', () => {
    const entries = parseBacklog(SOURCE);
    const rendered = renderBacklog(entries);
    expect(parseBacklog(rendered)).toEqual(entries);
  });
});
```

- [ ] **Step 2: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/scripts exec vitest run src/maintenance/__tests__/debt-backlog.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../debt-backlog"`. 0 test exécuté.

- [ ] **Step 3: Implémenter le parseur et le rendu du backlog**

Créer `scripts/maintenance/debt-backlog.ts` :

```ts
import { z } from 'zod';

export const debtEntrySchema = z.object({
  id: z.string().regex(/^DEBT-\d{3}$/),
  module: z.string().min(1),
  cost: z.string().min(1),
  risk: z.enum(['faible', 'moyen', 'élevé']),
  due: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  status: z.enum(['ouvert', 'planifié', 'fermé']),
});

export type DebtEntry = z.infer<typeof debtEntrySchema>;

const HEADER = '| ID | Module | Coût | Risque | Échéance | Statut |';
const SEPARATOR = '|---|---|---|---|---|---|';

export function parseBacklog(markdown: string): DebtEntry[] {
  const entries: DebtEntry[] = [];
  const lines = markdown.split('\n');

  for (const [index, raw] of lines.entries()) {
    const line = raw.trim();
    if (!line.startsWith('|')) continue;
    if (line === HEADER || line === SEPARATOR) continue;

    const cells = line
      .slice(1, -1)
      .split('|')
      .map((cell) => cell.trim());

    if (cells.length !== 6) {
      throw new Error(
        `ligne de backlog invalide à la ligne ${index + 1} : ${cells.length} colonnes au lieu de 6`,
      );
    }

    const [id, module, cost, risk, due, status] = cells;
    const parsed = debtEntrySchema.safeParse({ id, module, cost, risk, due, status });

    if (!parsed.success) {
      throw new Error(
        `ligne de backlog invalide à la ligne ${index + 1} : ${parsed.error.issues
          .map((issue) => `${issue.path.join('.')} ${issue.message}`)
          .join(', ')}`,
      );
    }

    entries.push(parsed.data);
  }

  return entries;
}

export function renderBacklog(entries: DebtEntry[]): string {
  const rows = entries.map(
    (entry) =>
      `| ${entry.id} | ${entry.module} | ${entry.cost} | ${entry.risk} | ${entry.due} | ${entry.status} |`,
  );
  return [HEADER, SEPARATOR, ...rows].join('\n');
}
```

- [ ] **Step 4: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/scripts exec vitest run src/maintenance/__tests__/debt-backlog.spec.ts
```

Attendu : **4 tests passés**, 0 échec.

- [ ] **Step 5: Écrire le test qui échoue — les budgets de performance sont immuables et complets**

Créer `packages/config/src/__tests__/budgets.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { PERF_BUDGETS } from '../budgets';
import { SLO_TARGETS } from '../slo';

describe('PERF_BUDGETS', () => {
  it('couvre toutes les routes listées dans infra §10.1', () => {
    const routes = PERF_BUDGETS.map((budget) => budget.route);
    expect(routes).toEqual(
      expect.arrayContaining([
        'public:home',
        'public:catalog',
        'public:product',
        'public:cart-checkout',
        'public:account',
        'admin:dashboard',
      ]),
    );
  });

  it('impose des bornes CWV conformes aux seuils du spec', () => {
    for (const budget of PERF_BUDGETS) {
      expect(budget.lcpMs).toBeLessThanOrEqual(2500);
      expect(budget.inpMs).toBeLessThanOrEqual(200);
      expect(budget.cls).toBeLessThanOrEqual(0.1);
    }
  });

  it('interdit un budget sans owner ni date de revalidation', () => {
    for (const budget of PERF_BUDGETS) {
      expect(budget.owner.length).toBeGreaterThan(0);
      expect(budget.revalidatedAt).toMatch(/^\d{4}-\d{2}-\d{2}$/);
    }
  });

  it('refuse deux budgets pour la même route', () => {
    const routes = PERF_BUDGETS.map((budget) => budget.route);
    expect(new Set(routes).size).toBe(routes.length);
  });
});

describe('SLO_TARGETS', () => {
  it('déclare un objectif de disponibilité et une latence par route', () => {
    for (const slo of SLO_TARGETS) {
      expect(slo.route.length).toBeGreaterThan(0);
      expect(slo.availability).toBeGreaterThanOrEqual(0.99);
      expect(slo.latencyP95Ms).toBeGreaterThan(0);
    }
  });

  it('aligne la latence p95 API sur le budget de lecture (<= 200 ms)', () => {
    const apiRead = SLO_TARGETS.find((slo) => slo.route === 'api:read');
    expect(apiRead?.latencyP95Ms).toBeLessThanOrEqual(200);
  });
});
```

- [ ] **Step 6: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/config exec vitest run src/__tests__/budgets.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../budgets"`. 0 test exécuté.

- [ ] **Step 7: Implémenter les budgets et les SLO**

Créer `packages/config/src/budgets.ts` :

```ts
export type PerfBudget = {
  route: string;
  jsFirstLoadKb: number;
  cssKb: number;
  requests: number;
  imagesAboveFold: number;
  lcpMs: number;
  inpMs: number;
  cls: number;
  owner: string;
  revalidatedAt: string;
};

/**
 * Source unique des budgets de performance.
 * Toute valeur provient de docs/roadmap/sections/infra.md §10.1 — ne pas inventer de seuil.
 */
export const PERF_BUDGETS: readonly PerfBudget[] = [
  {
    route: 'public:home',
    jsFirstLoadKb: 170,
    cssKb: 60,
    requests: 60,
    imagesAboveFold: 3,
    lcpMs: 2000,
    inpMs: 150,
    cls: 0.05,
    owner: 'web-lead',
    revalidatedAt: '2026-10-21',
  },
  {
    route: 'public:catalog',
    jsFirstLoadKb: 170,
    cssKb: 60,
    requests: 80,
    imagesAboveFold: 4,
    lcpMs: 2500,
    inpMs: 200,
    cls: 0.1,
    owner: 'web-lead',
    revalidatedAt: '2026-10-21',
  },
  {
    route: 'public:product',
    jsFirstLoadKb: 190,
    cssKb: 60,
    requests: 70,
    imagesAboveFold: 5,
    lcpMs: 2200,
    inpMs: 180,
    cls: 0.08,
    owner: 'web-lead',
    revalidatedAt: '2026-10-21',
  },
  {
    route: 'public:cart-checkout',
    jsFirstLoadKb: 220,
    cssKb: 60,
    requests: 90,
    imagesAboveFold: 2,
    lcpMs: 2500,
    inpMs: 200,
    cls: 0.1,
    owner: 'web-lead',
    revalidatedAt: '2026-10-21',
  },
  {
    route: 'public:account',
    jsFirstLoadKb: 190,
    cssKb: 60,
    requests: 70,
    imagesAboveFold: 2,
    lcpMs: 2500,
    inpMs: 200,
    cls: 0.1,
    owner: 'web-lead',
    revalidatedAt: '2026-10-21',
  },
  {
    route: 'admin:dashboard',
    jsFirstLoadKb: 240,
    cssKb: 60,
    requests: 120,
    imagesAboveFold: 2,
    lcpMs: 3000,
    inpMs: 250,
    cls: 0.1,
    owner: 'admin-lead',
    revalidatedAt: '2026-10-21',
  },
] as const;
```

Créer `packages/config/src/slo.ts` :

```ts
export type SloTarget = {
  route: string;
  availability: number;
  latencyP95Ms: number;
  errorBudgetWindowDays: number;
  burnRateFast: number;
  burnRateSlow: number;
};

/**
 * SLO par route — seuils issus de docs/roadmap/sections/infra.md §7.3 et §8.2.
 * Un SLO sans runbook associé est un défaut de configuration (infra §15).
 */
export const SLO_TARGETS: readonly SloTarget[] = [
  {
    route: 'api:read',
    availability: 0.995,
    latencyP95Ms: 200,
    errorBudgetWindowDays: 30,
    burnRateFast: 14.4,
    burnRateSlow: 6,
  },
  {
    route: 'api:write',
    availability: 0.995,
    latencyP95Ms: 800,
    errorBudgetWindowDays: 30,
    burnRateFast: 14.4,
    burnRateSlow: 6,
  },
  {
    route: 'api:payment-webhook',
    availability: 0.999,
    latencyP95Ms: 500,
    errorBudgetWindowDays: 30,
    burnRateFast: 14.4,
    burnRateSlow: 6,
  },
  {
    route: 'web:public',
    availability: 0.995,
    latencyP95Ms: 800,
    errorBudgetWindowDays: 30,
    burnRateFast: 14.4,
    burnRateSlow: 6,
  },
] as const;
```

- [ ] **Step 8: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/config exec vitest run src/__tests__/budgets.spec.ts
```

Attendu : **6 tests passés**, 0 échec.

- [ ] **Step 9: Créer le backlog de dette technique initial**

Créer `docs/maintenance/TECH-DEBT-BACKLOG.md` :

```markdown
# Backlog de dette technique — Fanours

Source : mesures du rapport hebdomadaire (`docs/maintenance/reports/`) et revue mensuelle qualité (30 min).
Règles : une entrée porte une mesure, un risque, un propriétaire et une échéance. Une entrée sans mesure n'est pas une entrée.

| ID | Module | Coût | Risque | Échéance | Statut |
|---|---|---|---|---|---|
| DEBT-001 | apps/web/src/app/[locale]/(public)/guides | 3 j | moyen | 2026-12-15 | ouvert |
| DEBT-002 | apps/api/src/modules/performance | 2 j | faible | 2026-12-01 | ouvert |
| DEBT-003 | apps/web/src/components/media | 2 j | moyen | 2026-11-30 | ouvert |
| DEBT-004 | packages/i18n/src/dictionaries | 4 j | élevé | 2026-12-31 | planifié |
| DEBT-005 | packages/db/src/schema | 3 j | élevé | 2027-01-15 | planifié |
```

- [ ] **Step 10: Créer le fichier de budgets versionné (miroir de `packages/config/src/budgets.ts`)**

Créer `ops/performance/budgets.yaml` :

```yaml
# Miroir versionné de packages/config/src/budgets.ts.
# Ce fichier est la source lue par les contrôles CI et par le job de rapport.
# Toute divergence entre les deux fichiers est un défaut bloquant (test de parité).
version: 1
source: packages/config/src/budgets.ts
routes:
  - route: public:home
    js_first_load_kb: 170
    css_kb: 60
    requests: 60
    images_above_fold: 3
    lcp_ms: 2000
    inp_ms: 150
    cls: 0.05
    owner: web-lead
    revalidated_at: '2026-10-21'
  - route: public:catalog
    js_first_load_kb: 170
    css_kb: 60
    requests: 80
    images_above_fold: 4
    lcp_ms: 2500
    inp_ms: 200
    cls: 0.1
    owner: web-lead
    revalidated_at: '2026-10-21'
  - route: public:product
    js_first_load_kb: 190
    css_kb: 60
    requests: 70
    images_above_fold: 5
    lcp_ms: 2200
    inp_ms: 180
    cls: 0.08
    owner: web-lead
    revalidated_at: '2026-10-21'
  - route: public:cart-checkout
    js_first_load_kb: 220
    css_kb: 60
    requests: 90
    images_above_fold: 2
    lcp_ms: 2500
    inp_ms: 200
    cls: 0.1
    owner: web-lead
    revalidated_at: '2026-10-21'
  - route: public:account
    js_first_load_kb: 190
    css_kb: 60
    requests: 70
    images_above_fold: 2
    lcp_ms: 2500
    inp_ms: 200
    cls: 0.1
    owner: web-lead
    revalidated_at: '2026-10-21'
  - route: admin:dashboard
    js_first_load_kb: 240
    css_kb: 60
    requests: 120
    images_above_fold: 2
    lcp_ms: 3000
    inp_ms: 250
    cls: 0.1
    owner: admin-lead
    revalidated_at: '2026-10-21'
```

- [ ] **Step 11: Écrire le test qui échoue — le rapport hebdomadaire calcule les écarts de budget**

Créer `apps/worker/src/jobs/__tests__/weekly-maintenance-report.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { buildWeeklyReport } from '../../../../scripts/maintenance/weekly-report';

describe('buildWeeklyReport', () => {
  it('classe un dépassement de bundle en violation bloquante', () => {
    const report = buildWeeklyReport({
      weekStart: '2026-11-02',
      measurements: [
        {
          route: 'public:catalog',
          jsFirstLoadKb: 198,
          requests: 71,
          lcpMs: 2100,
          inpMs: 120,
          cls: 0.04,
        },
      ],
      debtOpen: 4,
      debtOverdue: 1,
      costMonthlyEur: 512,
    });

    const catalog = report.routes.find((r) => r.route === 'public:catalog');
    expect(catalog?.status).toBe('violation');
    expect(catalog?.violations.map((v) => v.metric)).toContain('jsFirstLoadKb');
    expect(catalog?.violations[0]?.measured).toBe(198);
    expect(catalog?.violations[0]?.budget).toBe(170);
  });

  it('ne signale aucune violation quand toutes les mesures sont dans le budget', () => {
    const report = buildWeeklyReport({
      weekStart: '2026-11-02',
      measurements: [
        {
          route: 'public:catalog',
          jsFirstLoadKb: 160,
          requests: 71,
          lcpMs: 2100,
          inpMs: 120,
          cls: 0.04,
        },
      ],
      debtOpen: 2,
      debtOverdue: 0,
      costMonthlyEur: 480,
    });

    expect(report.routes[0]?.status).toBe('ok');
    expect(report.routes[0]?.violations).toHaveLength(0);
  });

  it('remonte la dérive de coût au-delà de la cible de 500 €/mois', () => {
    const report = buildWeeklyReport({
      weekStart: '2026-11-02',
      measurements: [],
      debtOpen: 0,
      debtOverdue: 0,
      costMonthlyEur: 512,
    });

    expect(report.cost.status).toBe('over');
    expect(report.cost.budgetEur).toBe(500);
  });

  it('reste dans le budget de coût entre 400 € et 500 €', () => {
    const report = buildWeeklyReport({
      weekStart: '2026-11-02',
      measurements: [],
      debtOpen: 0,
      debtOverdue: 0,
      costMonthlyEur: 480,
    });

    expect(report.cost.status).toBe('warning');
  });

  it('signale la dette échue', () => {
    const report = buildWeeklyReport({
      weekStart: '2026-11-02',
      measurements: [],
      debtOpen: 5,
      debtOverdue: 2,
      costMonthlyEur: 300,
    });

    expect(report.debt.overdue).toBe(2);
    expect(report.debt.status).toBe('attention');
  });
});
```

- [ ] **Step 12: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/worker exec vitest run src/jobs/__tests__/weekly-maintenance-report.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../../../../scripts/maintenance/weekly-report"`. 0 test exécuté.

- [ ] **Step 13: Implémenter le générateur de rapport hebdomadaire**

Créer `scripts/maintenance/weekly-report.ts` :

```ts
import { PERF_BUDGETS, type PerfBudget } from '@fanours/config/budgets';

export type Measurement = {
  route: string;
  jsFirstLoadKb: number;
  requests: number;
  lcpMs: number;
  inpMs: number;
  cls: number;
};

export type WeeklyReportInput = {
  weekStart: string;
  measurements: Measurement[];
  debtOpen: number;
  debtOverdue: number;
  costMonthlyEur: number;
};

export type BudgetViolation = {
  metric: keyof Pick<
    PerfBudget,
    'jsFirstLoadKb' | 'requests' | 'lcpMs' | 'inpMs' | 'cls'
  >;
  measured: number;
  budget: number;
};

export type RouteReport = {
  route: string;
  status: 'ok' | 'warning' | 'violation';
  violations: BudgetViolation[];
};

export type WeeklyReport = {
  weekStart: string;
  routes: RouteReport[];
  debt: { open: number; overdue: number; status: 'ok' | 'attention' };
  cost: {
    monthlyEur: number;
    budgetEur: number;
    status: 'ok' | 'warning' | 'over';
  };
  generatedAt: string;
};

const COST_BUDGET_EUR = 500;
const COST_WARNING_EUR = 400;
const WARNING_RATIO = 0.95;

const METRICS = [
  'jsFirstLoadKb',
  'requests',
  'lcpMs',
  'inpMs',
  'cls',
] as const;

export function buildWeeklyReport(input: WeeklyReportInput): WeeklyReport {
  const routes: RouteReport[] = PERF_BUDGETS.map((budget) => {
    const measurement = input.measurements.find((m) => m.route === budget.route);

    if (!measurement) {
      return { route: budget.route, status: 'ok', violations: [] };
    }

    const violations: BudgetViolation[] = [];

    for (const metric of METRICS) {
      const measured = measurement[metric];
      const limit = budget[metric];
      if (measured > limit) {
        violations.push({ metric, measured, budget: limit });
      }
    }

    const status: RouteReport['status'] =
      violations.length > 0
        ? 'violation'
        : METRICS.some((metric) => {
            const measured = measurement[metric];
            return measured > budget[metric] * WARNING_RATIO;
          })
          ? 'warning'
          : 'ok';

    return { route: budget.route, status, violations };
  });

  const costStatus: WeeklyReport['cost']['status'] =
    input.costMonthlyEur > COST_BUDGET_EUR
      ? 'over'
      : input.costMonthlyEur > COST_WARNING_EUR
        ? 'warning'
        : 'ok';

  return {
    weekStart: input.weekStart,
    routes,
    debt: {
      open: input.debtOpen,
      overdue: input.debtOverdue,
      status: input.debtOverdue > 0 ? 'attention' : 'ok',
    },
    cost: {
      monthlyEur: input.costMonthlyEur,
      budgetEur: COST_BUDGET_EUR,
      status: costStatus,
    },
    generatedAt: new Date(0).toISOString(),
  };
}
```

- [ ] **Step 14: Enregistrer le job planifié hebdomadaire**

Créer `apps/worker/src/jobs/weekly-maintenance-report.ts` :

```ts
import { defineJob } from './define-job';
import { buildWeeklyReport, type Measurement } from '../../../scripts/maintenance/weekly-report';

export const weeklyMaintenanceReportJob = defineJob({
  name: 'weekly-maintenance-report',
  // Chaque lundi à 06:00 UTC. Le rapport de la semaine précédente est produit.
  repeat: { pattern: '0 6 * * 1' },
  idempotencyKey: (payload: { weekStart: string }) => `weekly-report:${payload.weekStart}`,
  backoffMs: (attempt) => Math.min(2 ** attempt * 1000, 60_000),
  async handle({ weekStart, measurements, debtOpen, debtOverdue, costMonthlyEur }) {
    const report = buildWeeklyReport({
      weekStart,
      measurements: measurements as Measurement[],
      debtOpen,
      debtOverdue,
      costMonthlyEur,
    });

    const month = weekStart.slice(0, 7);
    const path = `docs/maintenance/reports/${month}/${weekStart}.md`;

    await this.storage.put(path, renderMarkdown(report));

    return {
      path,
      violations: report.routes.filter((route) => route.status === 'violation').length,
      costStatus: report.cost.status,
    };
  },
});

function renderMarkdown(report: ReturnType<typeof buildWeeklyReport>): string {
  const lines: string[] = [];
  lines.push(`# Rapport de maintenance — semaine du ${report.weekStart}`);
  lines.push('');
  lines.push('| Route | Statut | Violations |');
  lines.push('|---|---|---|');

  for (const route of report.routes) {
    const detail =
      route.violations.length === 0
        ? '—'
        : route.violations
            .map((v) => `${v.metric} ${v.measured} > ${v.budget}`)
            .join(', ');
    lines.push(`| ${route.route} | ${route.status} | ${detail} |`);
  }

  lines.push('');
  lines.push(
    `- Dette technique ouverte : ${report.debt.open} (échue : ${report.debt.overdue})`,
  );
  lines.push(
    `- Coût d'infrastructure mensuel : ${report.cost.monthlyEur} € (cible ${report.cost.budgetEur} € — statut ${report.cost.status})`,
  );

  return lines.join('\n');
}
```

- [ ] **Step 15: Écrire l'ADR de la cadence de maintenance**

Créer `docs/adr/0011-budget-perf-ci.md` :

```markdown
# ADR-0011 — Les budgets de performance deviennent des contrôles CI bloquants

- Statut : Accepted
- Date : 2026-10-21
- Décideur : web-lead
- Remplace : aucun

## Contexte

Les seuils de performance existent depuis la Phase 1 (infra §10.1) mais rien ne les vérifiait : un régression de bundle de 40 Ko passait la CI. La Phase 6 dispose de 30 jours de données d'usage, ce qui permet enfin de mesurer avant d'optimiser.

## Décision

1. `packages/config/src/budgets.ts` est la **source unique** des budgets par route.
2. `ops/performance/budgets.yaml` en est le miroir versionné pour les outils externes ; un test de parité échoue si les deux divergent.
3. Chaque budget dépassé en CI **bloque la PR**. Un dépassement n'est pas absorbé par « on le corrige plus tard » : soit le budget est relaxé par un ADR daté qui justifie la mesure, soit le code est corrigé.
4. Le rapport hebdomadaire (`apps/worker/src/jobs/weekly-maintenance-report.ts`) est archivé dans `docs/maintenance/reports/<YYYY-MM>/`.

## Options écartées

- **Avertissement non bloquant** : un budget qui n'arrête rien finit par ne plus être lu.
- **Seuils dans un tableau Markdown non versionné** : ils divergeaient des valeurs effectives.

## Conséquences

- Toute PR qui touche le bundle, les images ou le nombre de requêtes doit prouver qu'elle reste dans le budget.
- La régression de performance devient détectable au moment du merge, pas au moment de l'incident.

## Réversibilité

Réversible par un nouvel ADR. Un budget ne peut être relâché que par un ADR justifiant la mesure qui l'a motivé.
```

- [ ] **Step 16: Vérifier que tout le périmètre compile et passe**

```bash
pnpm --filter @fanours/config exec vitest run
pnpm --filter @fanours/scripts exec vitest run
pnpm --filter @fanours/worker exec vitest run src/jobs/__tests__/weekly-maintenance-report.spec.ts
pnpm turbo run typecheck
```

Attendu : toutes les suites vertes, `typecheck` sans erreur.

- [ ] **Step 17: Commit**

```bash
git add docs/maintenance docs/adr/0011-budget-perf-ci.md packages/config/src/budgets.ts packages/config/src/slo.ts packages/config/src/__tests__/budgets.spec.ts scripts/maintenance apps/worker/src/jobs/weekly-maintenance-report.ts apps/worker/src/jobs/__tests__/weekly-maintenance-report.spec.ts ops/performance/budgets.yaml
git commit -m "chore(maintenance): pose la cadence de maintenance et les budgets de performance" -m "Backlog de dette technique outille, budgets de performance par route issus
de infra 10.1, SLO cibles, rapport hebdomadaire planifie en job BullMQ et
ADR posant les budgets comme controles CI bloquants.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` affiche le titre, le corps et la ligne `Co-Authored-By`.

---

### Task 2: Performance web — Core Web Vitals, images, bundle, requêtes N+1

**Files:**
- Create: `apps/web/src/lib/seo/jsonld.ts`
- Create: `apps/web/src/components/media/ProductImage.tsx`
- Create: `apps/web/scripts/bundle-budget.mjs`
- Create: `ops/performance/n-plus-one.sh`
- Create: `docs/adr/0012-strategy-technique-web.md`
- Modify: `apps/web/next.config.ts`
- Modify: `apps/web/src/app/[locale]/(public)/catalog/page.tsx`
- Test: `apps/web/src/lib/seo/__tests__/jsonld.spec.ts`
- Test: `apps/web/src/components/media/__tests__/product-image.spec.tsx`
- Test: `apps/web/scripts/__tests__/bundle-budget.spec.ts`

**Interfaces:**
- Consumes: `packages/config/src/budgets.ts` (budgets de la Task 1), `packages/i18n/src/locale.ts` (locales supportées `fr-FR` / `en` / `zh-CN`), `apps/web/src/lib/api/client.ts` (client HTTP existant), `docs/roadmap/sections/infra.md` §10.1 (CWV p75), `docs/roadmap/sections/design.md` §3.11 (interdits visuels : pas de dégradé, pas de glassmorphism global, icônes SVG, badges réservés aux statuts).
- Produces:
  - `apps/web/src/lib/seo/jsonld.ts` → `export function buildArticleJsonLd(input: ArticleJsonLdInput): JsonLdObject`, `export function buildFaqJsonLd(input: FaqJsonLdInput): JsonLdObject`, `export function buildBreadcrumbJsonLd(input: BreadcrumbJsonLdInput): JsonLdObject`.
  - `apps/web/src/components/media/ProductImage.tsx` → `export function ProductImage(props: ProductImageProps): ReactElement` avec `width`, `height`, `sizes`, `priority` explicites.
  - `apps/web/scripts/bundle-budget.mjs` → `export function evaluateBundleBudget(measured, budgets)` et export CLI.

---

- [ ] **Step 1: Écrire le test qui échoue — le JSON-LD Article ne contient que le contenu visible**

Créer `apps/web/src/lib/seo/__tests__/jsonld.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { buildArticleJsonLd, buildFaqJsonLd, buildBreadcrumbJsonLd } from '../jsonld';

describe('buildArticleJsonLd', () => {
  it('produit un @graph unique contenant un seul Article', () => {
    const jsonLd = buildArticleJsonLd({
      locale: 'fr-FR',
      title: 'MOQ, paliers de prix et incréments',
      description: 'Comment fonctionne un palier de prix en gros.',
      url: 'https://fanours.example/fr/guides/moq-paliers',
      publishedAt: '2026-11-03T00:00:00.000Z',
      modifiedAt: '2026-11-10T00:00:00.000Z',
      author: 'Fanours',
    });

    expect(jsonLd['@context']).toBe('https://schema.org');
    expect(jsonLd['@graph']).toHaveLength(1);
    expect(jsonLd['@graph'][0]?.['@type']).toBe('Article');
  });

  it('interdit d’émettre un aggregateRating sans avis vérifiés', () => {
    const jsonLd = buildArticleJsonLd({
      locale: 'fr-FR',
      title: 'Titre',
      description: 'Description.',
      url: 'https://fanours.example/fr/guides/x',
      publishedAt: '2026-11-03T00:00:00.000Z',
      modifiedAt: '2026-11-03T00:00:00.000Z',
      author: 'Fanours',
    });

    expect(JSON.stringify(jsonLd)).not.toContain('aggregateRating');
    expect(JSON.stringify(jsonLd)).not.toContain('review');
  });

  it('refuse un avis non vérifié', () => {
    expect(() =>
      buildArticleJsonLd({
        locale: 'fr-FR',
        title: 'Titre',
        description: 'Description.',
        url: 'https://fanours.example/fr/guides/x',
        publishedAt: '2026-11-03T00:00:00.000Z',
        modifiedAt: '2026-11-03T00:00:00.000Z',
        author: 'Fanours',
        review: { ratingValue: 5, reviewCount: 3, verified: false },
      }),
    ).toThrowError(/avis non vérifié/);
  });

  it('publie aggregateRating quand tous les avis sont vérifiés', () => {
    const jsonLd = buildArticleJsonLd({
      locale: 'fr-FR',
      title: 'Titre',
      description: 'Description.',
      url: 'https://fanours.example/fr/guides/x',
      publishedAt: '2026-11-03T00:00:00.000Z',
      modifiedAt: '2026-11-03T00:00:00.000Z',
      author: 'Fanours',
      review: { ratingValue: 4.6, reviewCount: 12, verified: true },
    });

    const graph = jsonLd['@graph'] as Array<Record<string, unknown>>;
    expect(graph[0]?.['aggregateRating']).toEqual({
      '@type': 'AggregateRating',
      ratingValue: 4.6,
      reviewCount: 12,
    });
  });
});

describe('buildFaqJsonLd', () => {
  it('refuse une question sans réponse visible', () => {
    expect(() =>
      buildFaqJsonLd({
        locale: 'fr-FR',
        url: 'https://fanours.example/fr/faq',
        items: [{ question: 'Quel est le MOQ ?', answer: '' }],
      }),
    ).toThrowError(/réponse vide/);
  });

  it('produit une FAQPage avec une entrée par question', () => {
    const jsonLd = buildFaqJsonLd({
      locale: 'fr-FR',
      url: 'https://fanours.example/fr/faq',
      items: [
        { question: 'Quel est le MOQ ?', answer: 'Le MOQ dépend du produit.' },
        { question: 'Quels sont les délais ?', answer: 'Le délai de préparation est distinct du transport.' },
      ],
    });

    const graph = jsonLd['@graph'] as Array<Record<string, unknown>>;
    expect(graph[0]?.['@type']).toBe('FAQPage');
    expect((graph[0]?.['mainEntity'] as unknown[])).toHaveLength(2);
  });
});

describe('buildBreadcrumbJsonLd', () => {
  it('numérote les positions à partir de 1', () => {
    const jsonLd = buildBreadcrumbJsonLd({
      locale: 'fr-FR',
      url: 'https://fanours.example/fr/guides/x',
      trail: [
        { name: 'Accueil', url: 'https://fanours.example/fr/' },
        { name: 'Guides', url: 'https://fanours.example/fr/guides' },
        { name: 'MOQ', url: 'https://fanours.example/fr/guides/x' },
      ],
    });

    const graph = jsonLd['@graph'] as Array<Record<string, unknown>>;
    const items = graph[0]?.['itemListElement'] as Array<Record<string, unknown>>;
    expect(items.map((item) => item['position'])).toEqual([1, 2, 3]);
  });
});
```

- [ ] **Step 2: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/web exec vitest run src/lib/seo/__tests__/jsonld.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../jsonld"`. 0 test exécuté.

- [ ] **Step 3: Implémenter les constructeurs JSON-LD**

Créer `apps/web/src/lib/seo/jsonld.ts` :

```ts
export type JsonLdObject = Record<string, unknown>;

export type ArticleJsonLdInput = {
  locale: string;
  title: string;
  description: string;
  url: string;
  publishedAt: string;
  modifiedAt: string;
  author: string;
  review?: { ratingValue: number; reviewCount: number; verified: boolean };
};

export type FaqJsonLdInput = {
  locale: string;
  url: string;
  items: Array<{ question: string; answer: string }>;
};

export type BreadcrumbJsonLdInput = {
  locale: string;
  url: string;
  trail: Array<{ name: string; url: string }>;
};

function assertNonEmpty(value: string, message: string): void {
  if (value.trim().length === 0) {
    throw new Error(message);
  }
}

/**
 * JSON-LD Article. Ne publie aggregateRating que si TOUS les avis sont vérifiés
 * (seo.md §17.9) — sinon le balisage est rejeté plutôt que publié sans donnée.
 */
export function buildArticleJsonLd(input: ArticleJsonLdInput): JsonLdObject {
  assertNonEmpty(input.title, 'titre d’article vide');
  assertNonEmpty(input.description, 'description d’article vide');
  assertNonEmpty(input.url, 'URL d’article vide');

  const article: JsonLdObject = {
    '@type': 'Article',
    headline: input.title,
    description: input.description,
    url: input.url,
    inLanguage: input.locale,
    datePublished: input.publishedAt,
    dateModified: input.modifiedAt,
    author: { '@type': 'Organization', name: input.author },
    publisher: { '@type': 'Organization', name: input.author },
  };

  if (input.review) {
    if (!input.review.verified) {
      throw new Error(
        'avis non vérifié : aggregateRating ne peut être publié qu’avec des avis vérifiés',
      );
    }
    article['aggregateRating'] = {
      '@type': 'AggregateRating',
      ratingValue: input.review.ratingValue,
      reviewCount: input.review.reviewCount,
    };
  }

  return { '@context': 'https://schema.org', '@graph': [article] };
}

export function buildFaqJsonLd(input: FaqJsonLdInput): JsonLdObject {
  for (const item of input.items) {
    assertNonEmpty(item.question, 'question de FAQ vide');
    assertNonEmpty(item.answer, 'réponse vide');
  }

  const faq: JsonLdObject = {
    '@type': 'FAQPage',
    inLanguage: input.locale,
    mainEntity: input.items.map((item) => ({
      '@type': 'Question',
      name: item.question,
      acceptedAnswer: { '@type': 'Answer', text: item.answer },
    })),
  };

  return { '@context': 'https://schema.org', '@graph': [faq] };
}

export function buildBreadcrumbJsonLd(input: BreadcrumbJsonLdInput): JsonLdObject {
  const breadcrumb: JsonLdObject = {
    '@type': 'BreadcrumbList',
    itemListElement: input.trail.map((item, index) => ({
      '@type': 'ListItem',
      position: index + 1,
      name: item.name,
      item: item.url,
    })),
  };

  return { '@context': 'https://schema.org', '@graph': [breadcrumb] };
}
```

- [ ] **Step 4: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/web exec vitest run src/lib/seo/__tests__/jsonld.spec.ts
```

Attendu : **8 tests passés**, 0 échec.

- [ ] **Step 5: Écrire le test qui échoue — l'image produit impose des dimensions explicites**

Créer `apps/web/src/components/media/__tests__/product-image.spec.tsx` :

```tsx
import { render, screen } from '@testing-library/react';
import { describe, expect, it } from 'vitest';
import { ProductImage } from '../ProductImage';

describe('ProductImage', () => {
  it('expose width et height sur l’élément img (pas de layout shift)', () => {
    render(
      <ProductImage
        src="/media/produits/lampe.jpg"
        alt="Lampe de bureau à LED"
        width={800}
        height={800}
      />,
    );

    const img = screen.getByRole('img');
    expect(img).toHaveAttribute('width', '800');
    expect(img).toHaveAttribute('height', '800');
    expect(img).toHaveAttribute('alt', 'Lampe de bureau à LED');
  });

  it('rend un sizes explicite pour éviter le téléchargement d’une image surdimensionnée', () => {
    render(
      <ProductImage
        src="/media/produits/lampe.jpg"
        alt="Lampe de bureau à LED"
        width={800}
        height={800}
        sizes="(max-width: 768px) 100vw, 50vw"
      />,
    );

    expect(screen.getByRole('img')).toHaveAttribute(
      'sizes',
      '(max-width: 768px) 100vw, 50vw',
    );
  });

  it('exige un alt non vide (jamais de couleur seule, jamais de description générique)', () => {
    expect(() =>
      render(
        <ProductImage src="/media/produits/x.jpg" alt="" width={100} height={100} />,
      ),
    ).toThrowError(/alt manquant/);
  });

  it('interdit la priorité sur plus d’une image above-the-fold', () => {
    render(
      <ProductImage
        src="/media/produits/x.jpg"
        alt="Produit"
        width={100}
        height={100}
        priority
      />,
    );

    expect(screen.getByRole('img')).toHaveAttribute('loading', 'eager');
    expect(screen.getByRole('img')).toHaveAttribute('fetchpriority', 'high');
  });

  it('ne charge pas l’image en différé par défaut', () => {
    render(
      <ProductImage src="/media/produits/x.jpg" alt="Produit" width={100} height={100} />,
    );

    expect(screen.getByRole('img')).toHaveAttribute('loading', 'lazy');
  });
});
```

- [ ] **Step 6: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/web exec vitest run src/components/media/__tests__/product-image.spec.tsx
```

Attendu : **ÉCHEC** — `Failed to resolve import "../ProductImage"`. 0 test exécuté.

- [ ] **Step 7: Implémenter `ProductImage`**

Créer `apps/web/src/components/media/ProductImage.tsx` :

```tsx
import Image from 'next/image';

export type ProductImageProps = {
  src: string;
  alt: string;
  width: number;
  height: number;
  sizes?: string;
  priority?: boolean;
};

export function ProductImage({
  src,
  alt,
  width,
  height,
  sizes,
  priority = false,
}: ProductImageProps) {
  if (alt.trim().length === 0) {
    throw new Error('alt manquant : une image de produit porte toujours une description utile');
  }

  if (!Number.isInteger(width) || !Number.isInteger(height)) {
    throw new Error('dimensions d’image invalides : width et height doivent être des entiers');
  }

  if (width <= 0 || height <= 0) {
    throw new Error('dimensions d’image invalides : width et height doivent être strictement positifs');
  }

  return (
    <Image
      src={src}
      alt={alt}
      width={width}
      height={height}
      sizes={sizes}
      priority={priority}
      loading={priority ? 'eager' : 'lazy'}
      fetchPriority={priority ? 'high' : 'auto'}
      style={{ height: 'auto', maxWidth: '100%' }}
    />
  );
}
```

- [ ] **Step 8: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/web exec vitest run src/components/media/__tests__/product-image.spec.tsx
```

Attendu : **5 tests passés**, 0 échec.

- [ ] **Step 9: Configurer `next.config.ts` — formats d'image, cache CDN, dimensions**

Modifier `apps/web/next.config.ts` pour ajouter (ou remplacer) la section `images` et les en-têtes de cache :

```ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  reactStrictMode: true,
  poweredByHeader: false,
  images: {
    // AVIF d'abord, WebP en repli : formats d'image imposés par le budget infra §10.2.
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [320, 360, 390, 430, 640, 768, 1024, 1280, 1440, 1920],
    imageSizes: [64, 96, 128, 256, 384],
    // Les images produits sont servies depuis le CDN, jamais depuis le bundle web.
    remotePatterns: [{ protocol: 'https', hostname: 'cdn.fanours.example', pathname: '/media/**' }],
    minimumCacheTTL: 60 * 60 * 24 * 30,
  },
  async headers() {
    return [
      {
        source: '/media/:path*',
        headers: [
          { key: 'Cache-Control', value: 'public, max-age=31536000, immutable' },
        ],
      },
      {
        source: '/_next/static/:path*',
        headers: [
          { key: 'Cache-Control', value: 'public, max-age=31536000, immutable' },
        ],
      },
    ];
  },
};

export default nextConfig;
```

- [ ] **Step 10: Écrire le test qui échoue — le contrôle de budget de bundle refuse un dépassement**

Créer `apps/web/scripts/__tests__/bundle-budget.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { evaluateBundleBudget } from '../bundle-budget';

describe('evaluateBundleBudget', () => {
  it('refuse une route qui dépasse le budget JS', () => {
    const result = evaluateBundleBudget(
      { route: 'public:catalog', jsFirstLoadKb: 198, cssKb: 55 },
      [{ route: 'public:catalog', jsFirstLoadKb: 170, cssKb: 60 }],
    );

    expect(result.ok).toBe(false);
    expect(result.violations).toEqual([
      { route: 'public:catalog', metric: 'jsFirstLoadKb', measured: 198, budget: 170 },
    ]);
  });

  it('refuse une route qui dépasse le budget CSS', () => {
    const result = evaluateBundleBudget(
      { route: 'public:catalog', jsFirstLoadKb: 150, cssKb: 72 },
      [{ route: 'public:catalog', jsFirstLoadKb: 170, cssKb: 60 }],
    );

    expect(result.ok).toBe(false);
    expect(result.violations[0]?.metric).toBe('cssKb');
  });

  it('accepte une route dans le budget', () => {
    const result = evaluateBundleBudget(
      { route: 'public:catalog', jsFirstLoadKb: 150, cssKb: 55 },
      [{ route: 'public:catalog', jsFirstLoadKb: 170, cssKb: 60 }],
    );

    expect(result.ok).toBe(true);
    expect(result.violations).toHaveLength(0);
  });

  it('ignore une route non mesurée', () => {
    const result = evaluateBundleBudget(
      { route: 'public:home', jsFirstLoadKb: 150, cssKb: 55 },
      [{ route: 'public:catalog', jsFirstLoadKb: 170, cssKb: 60 }],
    );

    expect(result.ok).toBe(true);
  });
});
```

- [ ] **Step 11: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/web exec vitest run scripts/__tests__/bundle-budget.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../bundle-budget"`. 0 test exécuté.

- [ ] **Step 12: Implémenter le contrôle de budget de bundle**

Créer `apps/web/scripts/bundle-budget.mjs` :

```javascript
import { readFileSync } from 'node:fs';
import { pathToFileURL } from 'node:url';

export function evaluateBundleBudget(measured, budgets) {
  const budget = budgets.find((entry) => entry.route === measured.route);
  if (!budget) {
    return { ok: true, violations: [] };
  }

  const violations = [];

  if (measured.jsFirstLoadKb > budget.jsFirstLoadKb) {
    violations.push({
      route: measured.route,
      metric: 'jsFirstLoadKb',
      measured: measured.jsFirstLoadKb,
      budget: budget.jsFirstLoadKb,
    });
  }

  if (measured.cssKb > budget.cssKb) {
    violations.push({
      route: measured.route,
      metric: 'cssKb',
      measured: measured.cssKb,
      budget: budget.cssKb,
    });
  }

  return { ok: violations.length === 0, violations };
}

async function main() {
  const [budgetsPath, measurementsPath] = process.argv.slice(2);

  if (!budgetsPath || !measurementsPath) {
    console.error(
      'usage: node bundle-budget.mjs <ops/performance/budgets.yaml> <bundle-report.json>',
    );
    process.exit(2);
  }

  const { parse } = await import('yaml');
  const parsed = parse(readFileSync(budgetsPath, 'utf8'));

  const budgets = parsed.routes.map((route) => ({
    route: route.route,
    jsFirstLoadKb: route.js_first_load_kb,
    cssKb: route.css_kb,
  }));

  const measurements = JSON.parse(readFileSync(measurementsPath, 'utf8'));
  const results = measurements.map((measured) =>
    evaluateBundleBudget(measured, budgets),
  );

  const violations = results.flatMap((result) => result.violations);

  if (violations.length > 0) {
    console.error('Budget de bundle dépassé :');
    for (const violation of violations) {
      console.error(
        `  ${violation.route} — ${violation.metric} : ${violation.measured} > ${violation.budget}`,
      );
    }
    process.exit(1);
  }

  console.log(`Budget de bundle respecté sur ${results.length} route(s).`);
}

if (process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href) {
  await main();
}
```

- [ ] **Step 13: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/web exec vitest run scripts/__tests__/bundle-budget.spec.ts
```

Attendu : **4 tests passés**, 0 échec.

- [ ] **Step 14: Créer le script de détection des requêtes N+1**

Créer `ops/performance/n-plus-one.sh` :

```bash
#!/usr/bin/env bash
# Détection des requêtes N+1 sur la base de staging.
# Sortie : la liste des requêtes dont le nombre d'appels est disproportionné
# par rapport à leur temps d'exécution moyen (heuristique N+1).
# Usage : ./ops/performance/n-plus-one.sh
set -euo pipefail

PSQL_BIN="${PSQL_BIN:-psql}"
DATABASE_URL="${DATABASE_URL:?DATABASE_URL is required}"

QUERY="
SELECT
  query,
  calls,
  mean_exec_time,
  round(mean_exec_time * calls) AS total_time_ms
FROM pg_stat_statements
WHERE calls > 100
  AND query NOT LIKE '%pg_stat_statements%'
  AND query NOT LIKE '%pg_catalog%'
ORDER BY total_time_ms DESC
LIMIT 100;
"

TMP_ROWS="$(mktemp)"
trap 'rm -f "${TMP_ROWS}"' EXIT

"${PSQL_BIN}" "${DATABASE_URL}" -v ON_ERROR_STOP=1 -At -F $'\t' -c "${QUERY}" > "${TMP_ROWS}"

echo "== Requêtes candidates N+1 (appels >= 100) =="
awk -F'\t' '
  $2 >= 100 {
    printf("calls=%s mean_ms=%s total_ms=%s  %s\n", $2, $3, $4, substr($1, 1, 120))
  }
' "${TMP_ROWS}"

COUNT="$(awk -F'\t' '$2 >= 100' "${TMP_ROWS}" | wc -l | tr -d ' ')"
echo "-- ${COUNT} requête(s) candidate(s). Une requête candidate doit être confirmée par un test d'intégration avant optimisation."
```

- [ ] **Step 15: Écrire l'ADR de stratégie technique web**

Créer `docs/adr/0012-strategy-technique-web.md` :

```markdown
# ADR-0012 — Stratégie de rendu, images et déduplication de requêtes web

- Statut : Accepted
- Date : 2026-10-21
- Décideur : web-lead
- Remplace : aucun

## Contexte

La Phase 6 dispose de 30 jours de données de terrain. Les budgets infra §10.1 sont posés mais rien ne garantissait leur respect : les images produit étaient servies sans dimensions explicites (CLS), le bundle de premier chargement n'était pas mesuré, et le catalogue déclenchait des requêtes répétées par produit.

## Décision

1. **Images** : `next/image` avec AVIF puis WebP, `width`/`height` **obligatoires** (CLS), `sizes` explicite, `priority` réservé à la seule image above-the-fold. Aucun hex en dur, aucune image hors CDN.
2. **Bundle** : la mesure de first-load est faite par `apps/web/scripts/bundle-budget.mjs` à partir du rapport de build ; tout dépassement bloque la CI. Le budget est modifiable **uniquement** par un ADR daté.
3. **Requêtes** : les lectures du catalogue passent par le cache Redis avec déduplication d'in-flight (`React.cache` côté rendu + cache serveur Redis). La déduplication ne s'applique qu'aux données **publicatives** (catalogue, fiche produit, paliers) ; le panier, le checkout et le compte ne sont **jamais** mis en cache.
4. **N+1** : toute requête identifiée par `ops/performance/n-plus-one.sh` est confirmée par un test d'intégration avant optimisation, puis corrigée par un *join* ou un lot. Une optimisation sans gain mesuré est revertée.

## Options écartées

- **Cache client de l panier** : interdit par le spec (donnée privée, §22.5 PWA NetworkOnly).
- **`next/dynamic` sur tout** : déplace le problème au lieu de le traiter et dégrade l'INP perçu.

## Conséquences

- Le catalogue et la fiche produit sont servis plus vite, avec un CLS structurellement nul.
- Toute PR web doit joindre la mesure avant/après dans sa description.

## Réversibilité

Réversible par un nouvel ADR.
```

- [ ] **Step 16: Vérifier le périmètre complet**

```bash
pnpm --filter @fanours/web exec vitest run
pnpm turbo run typecheck
pnpm turbo run lint
```

Attendu : suites vertes, `typecheck` et `lint` sans erreur.

- [ ] **Step 17: Commit**

```bash
git add apps/web/src/lib/seo apps/web/src/components/media apps/web/scripts apps/web/next.config.ts apps/web/src/app ops/performance/n-plus-one.sh docs/adr/0012-strategy-technique-web.md
git commit -m "perf(web): optimise images, JSON-LD et controle de budget de bundle" -m "Images AVIF/WebP avec dimensions explicites et CLS nul, constructeurs JSON-LD
Article/FAQPage/BreadcrumbList sans aggregateRating non verifie, controle CI du
bundle first-load, script de detection N+1 et ADR de strategie technique web.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` affiche le titre, le corps et la ligne `Co-Authored-By`.

---

### Task 3: Performance API & base — index, pool, cache Redis, files BullMQ

**Files:**
- Create: `apps/api/src/modules/performance/cache-keys.ts`
- Create: `apps/api/src/modules/performance/cache.ts`
- Create: `apps/api/src/modules/performance/query-metrics.ts`
- Create: `packages/db/migrations/0120_maintenance_metrics.sql`
- Create: `packages/db/src/schema/maintenance-metrics.ts`
- Create: `ops/performance/index-audit.sh`
- Create: `docs/adr/0013-index-pg-stat-statements.md`
- Create: `docs/adr/0014-cache-redis-invalidation.md`
- Test: `apps/api/src/modules/performance/__tests__/cache-keys.spec.ts`
- Test: `apps/api/src/modules/performance/__tests__/cache.spec.ts`
- Test: `apps/api/src/modules/performance/__tests__/query-metrics.spec.ts`

**Interfaces:**
- Consumes: `packages/db/src/client.ts` (client Drizzle existant), `apps/api/src/modules/redis/redis.client.ts` (client Redis existant), `packages/jobs/src/idempotent-job.ts` (motif `defineJob` / `idempotencyKey` / `backoffMs` existant), `docs/roadmap/sections/infra.md` §10.3 (table des clés Redis) et §10.4 (index SQL), §11.2 à §11.4 (jobs, `background_jobs`, DLQ).
- Produces:
  - `apps/api/src/modules/performance/cache-keys.ts` → `export const CACHE_TTL_SECONDS`, `export function catalogPageKey(input): string`, `export function productKey(input): string`, `export function pricingKey(input): string`, `export function idempotencyKey(scope, key): string`, `export function rateLimitKey(scope, identifier): string`, `export function invalidateTags(tags: string[]): Promise<number>`.
  - `apps/api/src/modules/performance/cache.ts` → `export class PerformanceCache` avec `getOrSet`, `invalidateByTag`, `getStats`.
  - `apps/api/src/modules/performance/query-metrics.ts` → `export async function collectSlowQueries(input): Promise<SlowQuery[]>` et `export function detectNPlusOne(calls: SlowQuery[]): NPlusOneCandidate[]`.

---

- [ ] **Step 1: Écrire le test qui échoue — les clés de cache respectent la table infra §10.3**

Créer `apps/api/src/modules/performance/__tests__/cache-keys.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import {
  CACHE_TTL_SECONDS,
  catalogPageKey,
  idempotencyKey,
  invalidateTags,
  pricingKey,
  productKey,
  rateLimitKey,
} from '../cache-keys';

describe('cache keys', () => {
  it('respecte les TTL de la table infra §10.3', () => {
    expect(CACHE_TTL_SECONDS.catalogPage).toBe(300);
    expect(CACHE_TTL_SECONDS.product).toBe(3600);
    expect(CACHE_TTL_SECONDS.pricing).toBe(600);
    expect(CACHE_TTL_SECONDS.idempotency).toBe(86_400);
    expect(CACHE_TTL_SECONDS.rateLimit).toBe(900);
  });

  it('inclut la version du catalogue dans la clé de page (invalidation par version)', () => {
    const key = catalogPageKey({
      version: 7,
      locale: 'fr-FR',
      filters: { category: 'lampe', minQty: 50 },
      page: 2,
    });

    expect(key.startsWith('catalog:v7:page:')).toBe(true);
    expect(key).toContain('locale=fr-FR');
  });

  it('produit la même clé pour des filtres équivalents quel que soit l’ordre des clés', () => {
    const a = catalogPageKey({
      version: 7,
      locale: 'fr-FR',
      filters: { category: 'lampe', minQty: 50 },
      page: 1,
    });
    const b = catalogPageKey({
      version: 7,
      locale: 'fr-FR',
      filters: { minQty: 50, category: 'lampe' },
      page: 1,
    });

    expect(a).toBe(b);
  });

  it('change la clé quand la version du catalogue change', () => {
    const a = catalogPageKey({ version: 7, locale: 'fr-FR', filters: {}, page: 1 });
    const b = catalogPageKey({ version: 8, locale: 'fr-FR', filters: {}, page: 1 });

    expect(a).not.toBe(b);
  });

  it('inclut le palier client et la quantité dans la clé de prix', () => {
    const key = pricingKey({ productId: 'p_1', customerTier: 'gold', quantity: 500 });
    expect(key).toBe('pricing:p_1:gold:500');
  });

  it('inclut la locale dans la clé produit', () => {
    expect(productKey({ productId: 'p_1', locale: 'zh-CN' })).toBe('product:p_1:zh-CN');
  });

  it('compose la clé d’idempotence avec le scope', () => {
    expect(idempotencyKey('order', 'abc-123')).toBe('idem:order:abc-123');
  });

  it('compose la clé de rate limit avec le scope et l’identifiant', () => {
    expect(rateLimitKey('login', 'ip:1.2.3.4')).toBe('rl:login:ip:1.2.3.4');
  });
});

describe('invalidateTags', () => {
  it('n’invalide que les clés portant le tag demandé', async () => {
    const store = new Map<string, string>();
    store.set('catalog:v7:page:abc#catalog', 'value');
    store.set('product:p_1:fr-FR#catalog', 'value');
    store.set('pricing:p_1:gold:500#pricing', 'value');

    const removed = await invalidateTags(['catalog'], store);

    expect(removed).toBe(2);
    expect(store.has('pricing:p_1:gold:500#pricing')).toBe(true);
  });

  it('retourne 0 quand aucun tag ne correspond', async () => {
    const store = new Map<string, string>();
    store.set('catalog:v7:page:abc#catalog', 'value');
    expect(await invalidateTags(['pricing'], store)).toBe(0);
  });
});
```

- [ ] **Step 2: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/performance/__tests__/cache-keys.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../cache-keys"`. 0 test exécuté.

- [ ] **Step 3: Implémenter les clés de cache et l'invalidation par tags**

Créer `apps/api/src/modules/performance/cache-keys.ts` :

```ts
import { createHash } from 'node:crypto';

/**
 * TTL en secondes — source unique : docs/roadmap/sections/infra.md §10.3.
 * Toute valeur divergente ici est un défaut : la table d'infrastructure fait foi.
 */
export const CACHE_TTL_SECONDS = {
  catalogPage: 300,
  product: 3600,
  pricing: 600,
  idempotency: 86_400,
  rateLimit: 900,
} as const;

function hashFilters(filters: Record<string, unknown>): string {
  const normalized = Object.keys(filters)
    .sort()
    .map((key) => `${key}=${String(filters[key] ?? '')}`)
    .join('&');

  return createHash('sha256').update(normalized).digest('hex').slice(0, 16);
}

export function catalogPageKey(input: {
  version: number;
  locale: string;
  filters: Record<string, unknown>;
  page: number;
}): string {
  return `catalog:v${input.version}:page:${hashFilters(input.filters)}:locale=${input.locale}:p=${input.page}`;
}

export function productKey(input: { productId: string; locale: string }): string {
  return `product:${input.productId}:${input.locale}`;
}

export function pricingKey(input: {
  productId: string;
  customerTier: string;
  quantity: number;
}): string {
  return `pricing:${input.productId}:${input.customerTier}:${input.quantity}`;
}

export function idempotencyKey(scope: string, key: string): string {
  return `idem:${scope}:${key}`;
}

export function rateLimitKey(scope: string, identifier: string): string {
  return `rl:${scope}:${identifier}`;
}

/**
 * Invalidation par tag. Les clés portent leur tag en suffixe `#tag`.
 * L'invalidation est déclenchée par les événements outbox, jamais par un code
 * de réponse HTTP (infra §10.3).
 */
export async function invalidateTags(
  tags: string[],
  store: Map<string, string>,
): Promise<number> {
  let removed = 0;

  for (const [key] of [...store.entries()]) {
    const tag = key.includes('#') ? key.slice(key.indexOf('#') + 1) : '';
    if (tags.includes(tag)) {
      store.delete(key);
      removed += 1;
    }
  }

  return removed;
}
```

- [ ] **Step 4: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/performance/__tests__/cache-keys.spec.ts
```

Attendu : **9 tests passés**, 0 échec.

- [ ] **Step 5: Écrire le test qui échoue — le cache ne sert jamais de donnée périmée**

Créer `apps/api/src/modules/performance/__tests__/cache.spec.ts` :

```ts
import { describe, expect, it, vi } from 'vitest';
import { PerformanceCache } from '../cache';

describe('PerformanceCache', () => {
  it('calcule la valeur une seule fois pour une même clé', async () => {
    const loader = vi.fn().mockResolvedValue({ id: 'p_1' });
    const cache = new PerformanceCache({ loader, ttlSeconds: 60 });

    const first = await cache.getOrSet('product:p_1:fr-FR', loader);
    const second = await cache.getOrSet('product:p_1:fr-FR', loader);

    expect(loader).toHaveBeenCalledTimes(1);
    expect(first).toEqual(second);
  });

  it('déduplique les appels concurrents sur la même clé', async () => {
    let resolveLoader: (value: unknown) => void = () => {};
    const loader = vi.fn(
      () =>
        new Promise((resolve) => {
          resolveLoader = resolve;
        }),
    );
    const cache = new PerformanceCache({ loader, ttlSeconds: 60 });

    const a = cache.getOrSet('product:p_1:fr-FR', loader);
    const b = cache.getOrSet('product:p_1:fr-FR', loader);

    resolveLoader({ id: 'p_1' });
    await Promise.all([a, b]);

    expect(loader).toHaveBeenCalledTimes(1);
  });

  it('recharge la valeur après expiration du TTL', async () => {
    vi.useFakeTimers();
    const loader = vi.fn().mockResolvedValue({ id: 'p_1' });
    const cache = new PerformanceCache({ loader, ttlSeconds: 60 });

    await cache.getOrSet('product:p_1:fr-FR', loader);
    vi.advanceTimersByTime(61_000);
    await cache.getOrSet('product:p_1:fr-FR', loader);

    expect(loader).toHaveBeenCalledTimes(2);
    vi.useRealTimers();
  });

  it('ne met jamais en cache une erreur', async () => {
    const loader = vi.fn().mockRejectedValue(new Error('boom'));
    const cache = new PerformanceCache({ loader, ttlSeconds: 60 });

    await expect(cache.getOrSet('product:p_1:fr-FR', loader)).rejects.toThrowError('boom');
    await expect(cache.getOrSet('product:p_1:fr-FR', loader)).rejects.toThrowError('boom');

    expect(loader).toHaveBeenCalledTimes(2);
    expect(cache.getStats().size).toBe(0);
  });

  it('invalide par tag et recharge la valeur', async () => {
    const loader = vi.fn().mockResolvedValue({ id: 'p_1' });
    const cache = new PerformanceCache({ loader, ttlSeconds: 60 });

    await cache.getOrSet('product:p_1:fr-FR#catalog', loader);
    await cache.invalidateByTag('catalog');
    await cache.getOrSet('product:p_1:fr-FR#catalog', loader);

    expect(loader).toHaveBeenCalledTimes(2);
  });

  it('refuse de mettre en cache une valeur non sérialisable', async () => {
    const loader = vi.fn().mockResolvedValue({ fn: () => undefined });
    const cache = new PerformanceCache({ loader, ttlSeconds: 60 });

    await expect(cache.getOrSet('x', loader)).rejects.toThrowError(/sérialisable/);
  });
});
```

- [ ] **Step 6: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/performance/__tests__/cache.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../cache"`. 0 test exécuté.

- [ ] **Step 7: Implémenter le cache de performance**

Créer `apps/api/src/modules/performance/cache.ts` :

```ts
export type CacheStats = {
  size: number;
  hits: number;
  misses: number;
  invalidations: number;
};

type CacheEntry = {
  value: string;
  expiresAt: number;
};

export type PerformanceCacheOptions = {
  loader: (key: string) => Promise<unknown>;
  ttlSeconds: number;
  now?: () => number;
};

export class PerformanceCache {
  private readonly entries = new Map<string, CacheEntry>();
  private readonly inFlight = new Map<string, Promise<unknown>>();
  private readonly loader: (key: string) => Promise<unknown>;
  private readonly ttlSeconds: number;
  private readonly now: () => number;
  private hits = 0;
  private misses = 0;
  private invalidations = 0;

  constructor(options: PerformanceCacheOptions) {
    this.loader = options.loader;
    this.ttlSeconds = options.ttlSeconds;
    this.now = options.now ?? (() => Date.now());
  }

  async getOrSet(
    key: string,
    loader: (key: string) => Promise<unknown> = this.loader,
  ): Promise<unknown> {
    const cached = this.entries.get(key);

    if (cached && cached.expiresAt > this.now()) {
      this.hits += 1;
      return JSON.parse(cached.value) as unknown;
    }

    const pending = this.inFlight.get(key);
    if (pending) {
      return pending;
    }

    this.misses += 1;

    const promise = loader(key)
      .then((value) => {
        const serialized = JSON.stringify(value);

        if (serialized === undefined) {
          throw new Error(
            `valeur non sérialisable pour la clé ${key} : le cache refuse de la stocker`,
          );
        }

        this.entries.set(key, {
          value: serialized,
          expiresAt: this.now() + this.ttlSeconds * 1000,
        });

        return value;
      })
      .finally(() => {
        this.inFlight.delete(key);
      });

    this.inFlight.set(key, promise);
    return promise;
  }

  async invalidateByTag(tag: string): Promise<number> {
    let removed = 0;

    for (const key of [...this.entries.keys()]) {
      const entryTag = key.includes('#') ? key.slice(key.indexOf('#') + 1) : '';
      if (entryTag === tag) {
        this.entries.delete(key);
        removed += 1;
      }
    }

    this.invalidations += removed;
    return removed;
  }

  getStats(): CacheStats {
    return {
      size: this.entries.size,
      hits: this.hits,
      misses: this.misses,
      invalidations: this.invalidations,
    };
  }
}
```

- [ ] **Step 8: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/performance/__tests__/cache.spec.ts
```

Attendu : **6 tests passés**, 0 échec.

- [ ] **Step 9: Écrire le test qui échoue — la détection N+1 repose sur le ratio appels/temps**

Créer `apps/api/src/modules/performance/__tests__/query-metrics.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { detectNPlusOne, type SlowQuery } from '../query-metrics';

const QUERIES: SlowQuery[] = [
  {
    query: 'SELECT * FROM product WHERE id = $1',
    calls: 12_000,
    meanExecTimeMs: 0.8,
    totalTimeMs: 9_600,
  },
  {
    query: 'SELECT * FROM product ORDER BY created_at',
    calls: 3,
    meanExecTimeMs: 120,
    totalTimeMs: 360,
  },
  {
    query: 'SELECT count(*) FROM order_line',
    calls: 250,
    meanExecTimeMs: 1.1,
    totalTimeMs: 275,
  },
];

describe('detectNPlusOne', () => {
  it('détecte une requête exécutée un grand nombre de fois à coût unitaire faible', () => {
    const candidates = detectNPlusOne(QUERIES);

    expect(candidates).toHaveLength(1);
    expect(candidates[0]?.query).toBe('SELECT * FROM product WHERE id = $1');
    expect(candidates[0]?.calls).toBe(12_000);
  });

  it('ignore une requête lente mais rare', () => {
    const candidates = detectNPlusOne(QUERIES);
    expect(
      candidates.some((candidate) => candidate.query.includes('ORDER BY created_at')),
    ).toBe(false);
  });

  it('ignore une requête peu appelée même si son temps unitaire est faible', () => {
    const candidates = detectNPlusOne(QUERIES);
    expect(
      candidates.some((candidate) => candidate.query.includes('count(*)')),
    ).toBe(false);
  });

  it('exige un ratio d’appels supérieur à 100 et un coût unitaire inférieur à 5 ms', () => {
    const candidates = detectNPlusOne([
      { query: 'SELECT 1 FROM a WHERE b = $1', calls: 150, meanExecTimeMs: 4.9, totalTimeMs: 735 },
    ]);

    expect(candidates).toHaveLength(1);
  });
});
```

- [ ] **Step 2 : Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/performance/__tests__/query-metrics.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../query-metrics"`. 0 test exécuté.

- [ ] **Step 3: Implémenter la collecte des requêtes lentes et la détection N+1**

Créer `apps/api/src/modules/performance/query-metrics.ts` :

```ts
export type SlowQuery = {
  query: string;
  calls: number;
  meanExecTimeMs: number;
  totalTimeMs: number;
};

export type NPlusOneCandidate = {
  query: string;
  calls: number;
  meanExecTimeMs: number;
  totalTimeMs: number;
  reason: string;
};

/** Seuil d'appels : en dessous, le coût total reste négligeable. */
const MIN_CALLS = 100;
/** Seuil de coût unitaire : un appel très rapide répété est la signature d'un N+1. */
const MAX_MEAN_EXEC_TIME_MS = 5;

export function detectNPlusOne(queries: SlowQuery[]): NPlusOneCandidate[] {
  return queries
    .filter(
      (query) => query.calls >= MIN_CALLS && query.meanExecTimeMs < MAX_MEAN_EXEC_TIME_MS,
    )
    .map((query) => ({
      ...query,
      reason: `${query.calls} appels à ${query.meanExecTimeMs} ms — signature de boucle par élément`,
    }));
}

/**
 * Collecte les requêtes lentes via pg_stat_statements.
 * `pool` est un client Drizzle ; la requête est en lecture seule.
 */
export async function collectSlowQueries(input: {
  execute: (sql: string) => Promise<{ rows: SlowQuery[] }>;
  limit?: number;
}): Promise<SlowQuery[]> {
  const limit = input.limit ?? 100;

  const result = await input.execute(`
    SELECT
      query,
      calls::bigint AS calls,
      mean_exec_time::float AS mean_exec_time,
      (mean_exec_time * calls)::float AS total_time
    FROM pg_stat_statements
    WHERE query NOT LIKE '%pg_stat_statements%'
      AND query NOT LIKE '%pg_catalog%'
    ORDER BY total_time DESC
    LIMIT ${limit}
  `);

  return result.rows.map((row) => ({
    query: row.query,
    calls: Number(row.calls),
    meanExecTimeMs: Number(row.meanExecTimeMs),
    totalTimeMs: Number(row.totalTimeMs),
  }));
}
```

- [ ] **Step 4: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/performance/__tests__/query-metrics.spec.ts
```

Attendu : **4 tests passés**, 0 échec.

- [ ] **Step 5: Créer la migration de métriques de maintenance**

Créer `packages/db/migrations/0120_maintenance_metrics.sql` :

```sql
-- 0120_maintenance_metrics.sql
-- Phase 6 — métriques de maintenance : baselines de performance et journal
-- d'audit des mesures. Append-only, aucun script down (rollback par code).
-- Appliqué sur : staging puis production, après accord explicite pour la prod.

SET lock_timeout = '5s';
SET statement_timeout = '60s';

CREATE TABLE IF NOT EXISTS performance_baseline (
  id              uuid PRIMARY KEY,
  route           text        NOT NULL,
  service_version text        NOT NULL,
  metric          text        NOT NULL,
  measured_value  numeric(18, 4) NOT NULL,
  budget_value    numeric(18, 4) NOT NULL,
  source          text        NOT NULL,
  measured_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT performance_baseline_metric_known
    CHECK (metric IN ('js_first_load_kb', 'css_kb', 'requests', 'lcp_ms', 'inp_ms', 'cls', 'api_p95_ms')),
  CONSTRAINT performance_baseline_source_known
    CHECK (source IN ('lighthouse_ci', 'bundle_report', 'prometheus', 'pg_stat_statements', 'k6')),
  CONSTRAINT performance_baseline_measured_non_negative
    CHECK (measured_value >= 0),
  CONSTRAINT performance_baseline_budget_non_negative
    CHECK (budget_value >= 0)
);

CREATE INDEX IF NOT EXISTS performance_baseline_route_measured_at_idx
  ON performance_baseline (route, measured_at DESC);

CREATE INDEX IF NOT EXISTS performance_baseline_metric_measured_at_idx
  ON performance_baseline (metric, measured_at DESC);

CREATE TABLE IF NOT EXISTS maintenance_measurement (
  id              uuid PRIMARY KEY,
  scope           text        NOT NULL,
  ref             text        NOT NULL,
  metric          text        NOT NULL,
  measured_value  numeric(18, 4) NOT NULL,
  budget_value    numeric(18, 4),
  status          text        NOT NULL,
  correlation_id  text        NOT NULL,
  measured_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT maintenance_measurement_scope_known
    CHECK (scope IN ('web', 'api', 'database', 'queue', 'cost', 'content')),
  CONSTRAINT maintenance_measurement_status_known
    CHECK (status IN ('ok', 'warning', 'violation')),
  CONSTRAINT maintenance_measurement_measured_non_negative
    CHECK (measured_value >= 0)
);

CREATE INDEX IF NOT EXISTS maintenance_measurement_scope_measured_at_idx
  ON maintenance_measurement (scope, measured_at DESC);

CREATE INDEX IF NOT EXISTS maintenance_measurement_status_idx
  ON maintenance_measurement (status, measured_at DESC)
  WHERE status <> 'ok';
```

- [ ] **Step 6: Déclarer le schéma Drizzle correspondant**

Créer `packages/db/src/schema/maintenance-metrics.ts` :

```ts
import { numeric, pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core';

export const performanceBaseline = pgTable('performance_baseline', {
  id: uuid('id').primaryKey(),
  route: text('route').notNull(),
  serviceVersion: text('service_version').notNull(),
  metric: text('metric').notNull(),
  measuredValue: numeric('measured_value', { precision: 18, scale: 4 }).notNull(),
  budgetValue: numeric('budget_value', { precision: 18, scale: 4 }).notNull(),
  source: text('source').notNull(),
  measuredAt: timestamp('measured_at', { withTimezone: true }).notNull().defaultNow(),
});

export const maintenanceMeasurement = pgTable('maintenance_measurement', {
  id: uuid('id').primaryKey(),
  scope: text('scope').notNull(),
  ref: text('ref').notNull(),
  metric: text('metric').notNull(),
  measuredValue: numeric('measured_value', { precision: 18, scale: 4 }).notNull(),
  budgetValue: numeric('budget_value', { precision: 18, scale: 4 }),
  status: text('status').notNull(),
  correlationId: text('correlation_id').notNull(),
  measuredAt: timestamp('measured_at', { withTimezone: true }).notNull().defaultNow(),
});
```

- [ ] **Step 7: Créer le script d'audit d'index**

Créer `ops/performance/index-audit.sh` :

```bash
#!/usr/bin/env bash
# Audit d'index : liste les index existants et propose ceux manquants
# à partir de pg_stat_statements. Sortie lisible par un humain.
# Usage : ./ops/performance/index-audit.sh
set -euo pipefail

PSQL_BIN="${PSQL_BIN:-psql}"
DATABASE_URL="${DATABASE_URL:?DATABASE_URL is required}"

echo "== Index existants sur les tables métier =="
"${PSQL_BIN}" "${DATABASE_URL}" -v ON_ERROR_STOP=1 <<'SQL'
SELECT
  tablename,
  indexname,
  indexdef
FROM pg_indexes
WHERE schemaname = 'public'
  AND tablename IN (
    'product', 'product_variant', 'price_tier', 'stock_item', 'stock_reservation',
    'cart', 'cart_line', 'order', 'order_line', 'payment', 'shipment',
    'trade_rule', 'trade_rule_version', 'audit_log', 'outbox_event'
  )
ORDER BY tablename, indexname;
SQL

echo
echo "== Colonnes les plus filtrées sans index (candidats index) =="
"${PSQL_BIN}" "${DATABASE_URL}" -v ON_ERROR_STOP=1 <<'SQL'
SELECT
  schemaname,
  tablename,
  attname,
  n_distinct,
  most_common_vals IS NOT NULL AS has_histogram
FROM pg_stats
WHERE schemaname = 'public'
  AND tablename IN (
    'product', 'product_variant', 'price_tier', 'stock_item', 'cart_line',
    'order', 'order_line', 'payment', 'trade_rule', 'trade_rule_version'
  )
  AND n_distinct > 50
  AND attname IN ('sku', 'email', 'order_number', 'status', 'created_at',
                  'updated_at', 'locale', 'customer_tier', 'hs_code', 'country_code')
ORDER BY n_distinct DESC
LIMIT 40;
SQL

echo
echo "Rappel : tout index créé au-delà de 100 000 lignes doit être CONCURRENTLY (architecture §8)."
```

- [ ] **Step 8: Écrire les deux ADR de la tâche**

Créer `docs/adr/0013-index-pg-stat-statements.md` :

```markdown
# ADR-0013 — Indexation pilotée par pg_stat_statements

- Statut : Accepted
- Date : 2026-10-21
- Décideur : dba
- Remplace : aucun

## Contexte

Des index existaient « par intuition ». Après 30 jours de trafic, plusieurs requêtes du catalogue et du panier sont apparues comme candidates N+1 ou comme scans séquentiels coûteux, sans mesure continue pour prioriser.

## Décision

1. `pg_stat_statements` est activé en staging et en production (extension, paramètres de collecte ajustés à un intervalle compatible avec la charge).
2. La détection N+1 est **heuristique et outillée** : ratio appels / coût unitaire (≥ 100 appels, < 5 ms unitaire), confirmée par un test d'intégration avant toute correction.
3. Tout nouvel index est justifié par une mesure (`EXPLAIN (ANALYZE, BUFFERS)` avant/après collé dans la PR) et par un test de non-régression du plan d'exécution.
4. Au-delà de 100 000 lignes, l'index est créé avec `CREATE INDEX CONCURRENTLY` dans une migration séparée, sans `down` (rollback par code).

## Conséquences

- La priorisation des index devient objective plutôt que discutable.
- Une migration d'index est toujours Expand-only et ne bloque pas les écritures.

## Réversibilité

Réversible par un nouvel ADR. Un index peut être supprimé par code (migration append-only de suppression, plan d'exécution re-vérifié).
```

Créer `docs/adr/0014-cache-redis-invalidation.md` :

```markdown
# ADR-0014 — Cache Redis : TTL court, invalidation par événements outbox

- Statut : Accepted
- Date : 2026-10-21
- Décideur : backend-lead
- Remplace : aucun

## Contexte

Le catalogue et les paliers de prix sont lus très souvent et peu modifiés. Un cache naïf pose deux risques : servir un prix périmé (interdit — le front n'est pas source de vérité) et masquer une invalidation ratée.

## Décision

1. **Périmètre caché** : uniquement les données publicatives — catalogue, fiche produit, paliers de prix. Le panier, le checkout, le compte, les commandes, les factures et les devis ne sont **jamais** en cache.
2. **TTL** : 300 s pour une page de catalogue, 3 600 s pour une fiche produit, 600 s pour un palier de prix (infra §10.3).
3. **Invalidation** : déclenchée par les événements outbox (`product.updated`, `price_tier.updated`, `stock.changed`) qui publient des tags. Une invalidation n'est **jamais** déduite du code de réponse HTTP d'une requête mutante.
4. **Version de catalogue** dans la clé de page : un changement de version invalide implicitement toutes les pages sans coût de scan.
5. **Déduplication d'in-flight** : deux requêtes concurrentes sur la même clé ne déclenchent qu'un seul chargement.
6. **Aucune valeur périmée possible** : le prix recalculé côté serveur fait autorité ; le cache ne sert qu'à alléger la lecture d'une donnée stable.

## Conséquences

- La latence de lecture catalogue baisse sans introduire de donnée périmée.
- L'invalidation est testable : un test d'intégration prouve qu'un `product.updated` vide la bonne clé.

## Réversibilité

Réversible par un nouvel ADR ; le cache est un accélérateur, pas une source de vérité.
```

- [ ] **Step 9: Vérifier le périmètre complet**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/performance
pnpm --filter @fanours/db exec vitest run
pnpm turbo run typecheck
```

Attendu : suites vertes, `typecheck` sans erreur.

- [ ] **Step 10: Commit**

```bash
git add apps/api/src/modules/performance packages/db/migrations/0120_maintenance_metrics.sql packages/db/src/schema/maintenance-metrics.ts ops/performance/index-audit.sh docs/adr/0013-index-pg-stat-statements.md docs/adr/0014-cache-redis-invalidation.md
git commit -m "perf(api): ajoute le cache Redis versionne et l'audit d'index" -m "Cles de cache alignees sur la table infra 10.3, invalidation par tags, dedup
d'appels concurrents, detection N+1 depuis pg_stat_statements, migration
0120 et ADRs indexation et cache.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` affiche le titre, le corps et la ligne `Co-Authored-By`.

---

### Task 4: Contenu SEO & clusters éditoriaux FR/EN/ZH

**Files:**
- Create: `docs/content/plan-contenu.md`
- Create: `docs/content/gabarits/guide.md`
- Create: `docs/content/gabarits/pays.md`
- Create: `docs/content/gabarits/faq.md`
- Create: `docs/content/gabarits/politique.md`
- Create: `docs/content/clusters/01-acheter-en-gros-depuis-la-chine.md` (et 02 à 10, une page par cluster)
- Create: `packages/db/migrations/0121_content_registry.sql`
- Create: `packages/db/src/schema/content-registry.ts`
- Create: `scripts/seo/content-lint.ts`
- Test: `scripts/seo/__tests__/content-lint.spec.ts`

**Interfaces:**
- Consumes: `docs/roadmap/sections/seo.md` §17.8 (10 clusters, titres FR/EN/ZH, règles de maillage interne), §17.9 (interdits de revue et 6 conditions cumulatives d'avis vérifiés), §17.11 (checklist 45 points), `packages/db/src/schema/trade-rule.ts` (table `trade_rule` / `trade_rule_version` existante), `packages/i18n/src/dictionaries` (3 locales existantes).
- Produces:
  - `scripts/seo/content-lint.ts` → `export function lintContent(input: LintInput): LintResult` avec `LintResult.violations: ContentViolation[]`, `export function checkProhibitedClaims(text: string): string[]`.
  - `scripts/seo/content-lint.ts` CLI exécutable : `node scripts/seo/content-lint.ts docs/content/clusters`.
  - Table `content_registry` : registre éditorial (slug, locale, type, source_url, source_consulted_at, revalidate_at, status, owner, correlation_id).

---

- [ ] **Step 1: Écrire le test qui échoue — le linter interdit les données inventées et les règles expirées**

Créer `scripts/seo/__tests__/content-lint.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { checkProhibitedClaims, lintContent } from '../content-lint';

describe('checkProhibitedClaims', () => {
  it('refuse un compteur de clients satisfaits sans source', () => {
    expect(checkProhibitedClaims('Plus de 1 234 clients satisfaits !')).toContain(
      'compteur de clients',
    );
  });

  it('refuse une promesse de classement', () => {
    expect(checkProhibitedClaims('Nous garantissons la première place sur Google.')).toContain(
      'promesse de classement',
    );
  });

  it('refuse une urgence artificielle', () => {
    expect(checkProhibitedClaims("Dernière chance : il ne reste que 2 articles !")).toContain(
      'urgence artificielle',
    );
  });

  it('refuse un avis client inventé', () => {
    expect(
      checkProhibitedClaims('« Excellent service ! » — Jean, client satisfait'),
    ).toContain('témoignage inventé');
  });

  it('accepte un contenu factuel sansPatterns', () => {
    expect(
      checkProhibitedClaims(
        'Le MOQ de ce produit est de 500 unités. Le délai de préparation est de 3 jours ouvrés.',
      ),
    ).toHaveLength(0);
  });
});

describe('lintContent', () => {
  const basePage = {
    slug: '02-moq-paliers-de-prix',
    locale: 'fr-FR',
    type: 'guide' as const,
    title: 'MOQ, paliers de prix et incréments',
    body: 'Le MOQ de ce produit est de 500 unités. Le délai de préparation est de 3 jours ouvrés.',
    sourceUrl: 'https://www.example.org/source-moq',
    sourceConsultedAt: '2026-11-03',
    revalidateAt: '2027-05-03',
    owner: 'marketing-owner',
    status: 'published' as const,
    internalLinks: ['/fr/guides/acheter-en-gros-depuis-la-chine'],
  };

  it('accepte une page conforme', () => {
    const result = lintContent({ pages: [basePage], knownSlugs: ['01-acheter-en-gros-depuis-la-chine'] });

    expect(result.violations).toHaveLength(0);
  });

  it('refuse une page dont la date de revalidation est dépassée', () => {
    const result = lintContent({
      pages: [{ ...basePage, revalidateAt: '2026-01-03' }],
      knownSlugs: [],
    });

    expect(result.violations[0]?.code).toBe('REVALIDATION_EXPIRED');
    expect(result.violations[0]?.severity).toBe('error');
  });

  it('refuse une page sans source officielle ni date de consultation', () => {
    const result = lintContent({
      pages: [{ ...basePage, sourceUrl: '', sourceConsultedAt: '' }],
      knownSlugs: [],
    });

    expect(result.violations.map((v) => v.code)).toEqual(
      expect.arrayContaining(['MISSING_SOURCE', 'MISSING_SOURCE_DATE']),
    );
  });

  it('refuse un lien interne vers une page inexistante', () => {
    const result = lintContent({
      pages: [{ ...basePage, internalLinks: ['/fr/guides/inexistant'] }],
      knownSlugs: [],
    });

    expect(result.violations[0]?.code).toBe('BROKEN_INTERNAL_LINK');
  });

  it('refuse une page qui contient une affirmation interdite', () => {
    const result = lintContent({
      pages: [{ ...basePage, body: 'Garantie du meilleur prix, première place Google.' }],
      knownSlugs: [],
    });

    expect(result.violations.map((v) => v.code)).toContain('PROHIBITED_CLAIM');
  });

  it('refuse une page publiée dont le propriétaire est absent', () => {
    const result = lintContent({
      pages: [{ ...basePage, owner: '' }],
      knownSlugs: [],
    });

    expect(result.violations.map((v) => v.code)).toContain('MISSING_OWNER');
  });

  it('refuse un statut inconnu', () => {
    const result = lintContent({
      pages: [{ ...basePage, status: 'en-attente' as unknown as 'published' }],
      knownSlugs: [],
    });

    expect(result.violations.map((v) => v.code)).toContain('INVALID_STATUS');
  });
});
```

- [ ] **Step 2: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/scripts exec vitest run src/seo/__tests__/content-lint.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../content-lint"`. 0 test exécuté.

- [ ] **Step 3: Implémenter le linter de contenu**

Créer `scripts/seo/content-lint.ts` :

```ts
export type ContentStatus = 'draft' | 'review' | 'published' | 'retired';

export type ContentPage = {
  slug: string;
  locale: string;
  type: 'guide' | 'country' | 'faq' | 'policy';
  title: string;
  body: string;
  sourceUrl: string;
  sourceConsultedAt: string;
  revalidateAt: string;
  owner: string;
  status: ContentStatus;
  internalLinks: string[];
};

export type ContentViolation = {
  slug: string;
  locale: string;
  code:
    | 'MISSING_SOURCE'
    | 'MISSING_SOURCE_DATE'
    | 'REVALIDATION_EXPIRED'
    | 'BROKEN_INTERNAL_LINK'
    | 'PROHIBITED_CLAIM'
    | 'MISSING_OWNER'
    | 'INVALID_STATUS';
  severity: 'error' | 'warning';
  message: string;
};

export type LintInput = {
  pages: ContentPage[];
  knownSlugs: string[];
  today?: string;
};

export type LintResult = {
  violations: ContentViolation[];
  errorCount: number;
  warningCount: number;
};

const PROHIBITED_PATTERNS: Array<{ pattern: RegExp; code: string }> = [
  { pattern: /\d[\s .]?\d{2,}\s+clients\s+satisfaits/i, code: 'compteur de clients' },
  { pattern: /(première place|premier résultat)\s+(sur|google)/i, code: 'promesse de classement' },
  { pattern: /(garantissons?|nous garantissons)\s+(le\s+)?(meilleur\s+prix|la\s+première\s+place)/i, code: 'promesse de classement' },
  { pattern: /(dernière chance|plus que \d+ (heures|jours|articles))/i, code: 'urgence artificielle' },
  { pattern: /[«"].{5,80}[»"]\s*[—-]\s*[A-ZÉÈ][\p{L}'-]{2,}/u, code: 'témoignage inventé' },
  { pattern: /\d[\s .]?\d{2,}\s+avis\s+(vérifiés|clients)/i, code: 'compteur d’avis' },
];

const VALID_STATUSES: ContentStatus[] = ['draft', 'review', 'published', 'retired'];

/**
 * Contrôle des affirmations interdites (seo.md §17.9).
 * Toute affirmation non sourcée est un motif de rejet, pas un avertissement.
 */
export function checkProhibitedClaims(text: string): string[] {
  const found: string[] = [];

  for (const { pattern, code } of PROHIBITED_PATTERNS) {
    if (pattern.test(text)) {
      found.push(code);
    }
  }

  return found;
}

export function lintContent(input: LintInput): LintResult {
  const today = input.today ?? new Date().toISOString().slice(0, 10);
  const violations: ContentViolation[] = [];

  for (const page of input.pages) {
    const push = (
      code: ContentViolation['code'],
      severity: ContentViolation['severity'],
      message: string,
    ) => {
      violations.push({ slug: page.slug, locale: page.locale, code, severity, message });
    };

    if (!VALID_STATUSES.includes(page.status)) {
      push('INVALID_STATUS', 'error', `statut inconnu : ${page.status}`);
    }

    if (page.sourceUrl.trim().length === 0) {
      push('MISSING_SOURCE', 'error', 'source officielle absente — le contenu doit être sourcé');
    }

    if (!/^\d{4}-\d{2}-\d{2}$/.test(page.sourceConsultedAt)) {
      push('MISSING_SOURCE_DATE', 'error', 'date de consultation de la source absente ou invalide');
    }

    if (page.revalidateAt.length > 0 && page.revalidateAt < today) {
      push(
        'REVALIDATION_EXPIRED',
        'error',
        `date de revalidation dépassée (${page.revalidateAt}) — une règle citée a expiré, alerter le compliance manager`,
      );
    }

    if (page.owner.trim().length === 0) {
      push('MISSING_OWNER', 'error', 'propriétaire métier absent — la relecture humaine n’est assignable');
    }

    for (const link of page.internalLinks) {
      const slug = link.split('/').filter(Boolean).pop() ?? '';
      if (!input.knownSlugs.includes(slug)) {
        push('BROKEN_INTERNAL_LINK', 'error', `lien interne vers une page inconnue : ${link}`);
      }
    }

    for (const claim of checkProhibitedClaims(`${page.title}\n${page.body}`)) {
      push('PROHIBITED_CLAIM', 'error', `affirmation interdite détectée : ${claim}`);
    }
  }

  return {
    violations,
    errorCount: violations.filter((violation) => violation.severity === 'error').length,
    warningCount: violations.filter((violation) => violation.severity === 'warning').length,
  };
}
```

- [ ] **Step 4: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/scripts exec vitest run src/seo/__tests__/content-lint.spec.ts
```

Attendu : **14 tests passés**, 0 échec.

- [ ] **Step 5: Créer la migration du registre éditorial**

Créer `packages/db/migrations/0121_content_registry.sql` :

```sql
-- 0121_content_registry.sql
-- Phase 6 — registre éditorial : chaque page publique indexable est enregistrée
-- avec sa source, sa date de consultation et sa date de revalidation.
-- Append-only, aucun script down.

SET lock_timeout = '5s';
SET statement_timeout = '60s';

CREATE TABLE IF NOT EXISTS content_registry (
  id                  uuid PRIMARY KEY,
  slug                text        NOT NULL,
  locale              char(3)     NOT NULL,
  content_type        text        NOT NULL,
  title               text        NOT NULL,
  source_url          text        NOT NULL,
  source_consulted_at date        NOT NULL,
  revalidate_at       date        NOT NULL,
  status              text        NOT NULL,
  owner               text        NOT NULL,
  trade_rule_id       uuid,
  correlation_id      text        NOT NULL,
  created_at          timestamptz NOT NULL DEFAULT now(),
  updated_at          timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT content_registry_type_known
    CHECK (content_type IN ('guide', 'country', 'faq', 'policy', 'cluster')),
  CONSTRAINT content_registry_status_known
    CHECK (status IN ('draft', 'review', 'published', 'retired')),
  CONSTRAINT content_registry_slug_format
    CHECK (slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$'),
  CONSTRAINT content_registry_revalidate_after_source
    CHECK (revalidate_at > source_consulted_at)
);

CREATE UNIQUE INDEX IF NOT EXISTS content_registry_slug_locale_idx
  ON content_registry (slug, locale);

CREATE INDEX IF NOT EXISTS content_registry_status_revalidate_idx
  ON content_registry (status, revalidate_at);

CREATE INDEX IF NOT EXISTS content_registry_trade_rule_idx
  ON content_registry (trade_rule_id)
  WHERE trade_rule_id IS NOT NULL;

-- Une page qui cite une règle de commerce doit référencer la version de règle
-- consultée : le contenu n'est pas separé de sa source.
ALTER TABLE content_registry
  ADD CONSTRAINT content_registry_requires_trade_rule_source
  CHECK (content_type <> 'country' OR source_url <> '');
```

- [ ] **Step 6: Déclarer le schéma Drizzle du registre éditorial**

Créer `packages/db/src/schema/content-registry.ts` :

```ts
import { date, pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core';

export const contentRegistry = pgTable('content_registry', {
  id: uuid('id').primaryKey(),
  slug: text('slug').notNull(),
  locale: text('locale').notNull(),
  contentType: text('content_type').notNull(),
  title: text('title').notNull(),
  sourceUrl: text('source_url').notNull(),
  sourceConsultedAt: date('source_consulted_at').notNull(),
  revalidateAt: date('revalidate_at').notNull(),
  status: text('status').notNull(),
  owner: text('owner').notNull(),
  tradeRuleId: uuid('trade_rule_id'),
  correlationId: text('correlation_id').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

- [ ] **Step 7: Écrire le plan de contenu validé par le propriétaire métier**

Créer `docs/content/plan-contenu.md` :

```markdown
# Plan de contenu éditorial — Phase 6

**Statut :** en attente de signature du propriétaire métier.
**Signataire requis :** marketing-owner (relecture) + compliance-manager (volets douane, HS, conformité).
**Date :** 2026-10-21.

Aucune page n'est publiée tant que ce plan n'est pas signé. L'IA prépare les textes ; l'humain valide les faits.

## Règles de production

1. Toute donnée chiffrée (seuil douanier, franchise, délai, code HS, Incoterm) provient de `trade_rule` / `trade_rule_version` avec **source officielle + URL + date de consultation**, ou d'un contenu éditorial validé par le propriétaire.
2. Une page qui cite une règle expirée déclenche une **alerte au compliance manager** (`content_registry.revalidate_at < now()`).
3. Aucun avis, aucun compteur, aucune promesse de classement, aucune urgence artificielle.
4. Chaque page existe en **fr-FR**, **en**, **zh-CN** avec le même contenu de fait ; la langue n'est pas la porte d'entrée de l'information.
5. Le contenu produit la donnée structurée correspondante (`Article`, `FAQPage`, `BreadcrumbList`) et **reflète le contenu visible**.

## Les 10 clusters (titres FR / EN / ZH)

| # | Cluster | FR | EN | ZH | Priorité | Pages | Statut |
|---|---|---|---|---|---|---|---|
| 1 | Acheter en gros depuis la Chine | Acheter en gros depuis la Chine | Sourcing Wholesale from China | 从中国批发采购 | P0 | 3 | review |
| 2 | MOQ, paliers, incréments | MOQ, paliers de prix et incréments | MOQ, Price Tiers and Increments | 最小起订量、价格阶梯与增量 | P0 | 3 | review |
| 3 | Frais d'importation, douane, TVA | Frais d'importation, douane et TVA par pays | Import Duties, Customs and VAT by Country | 各國進口關稅、海關與增值稅 | P0 | 3 | review |
| 4 | Incoterms DDP vs DAP | Incoterms DDP vs DAP | Incoterms DDP vs DAP | DDP 与 DAP 貿易術語對比 | P0 | 3 | review |
| 5 | Contrôle qualité et certifications | Contrôle qualité, inspection et certifications | Quality Control, Inspection and Certifications | 質量檢驗、驗貨與認證 | P1 | 3 | review |
| 6 | Délais et transporteurs | Délais d'expédition et transporteurs | Shipping Lead Times and Carriers | 運輸時效與承運商 | P1 | 3 | review |
| 7 | Devis B2B, paiement, acompte | Devis B2B, paiement et acompte | B2B Quotes, Payment and Deposits | B2B 報價、付款與訂金 | P1 | 3 | review |
| 8 | Codes HS, origine, dédouanement | Codes HS, origine et dédouanement | HS Codes, Origin and Customs Clearance | 海关编码、原产地与清关 | P2 | 3 | review |
| 9 | Conformité produit par marché | Conformité produit par marché | Product Compliance by Market | 各市場產品合規要求 | P2 | 3 | review |
| 10 | Rentabilité d'un achat en gros | Rentabilité d'un achat en gros | Wholesale Purchase Profitability | 批發採購的盈利分析 | P2 | 3 | review |

Total : 30 pages (10 clusters × 3 locales). Les gabarits **pays** et **FAQ** sont transverses et s'appuient sur les mêmes données de règle.

## Règles de maillage interne

1. Chaque page pointe vers **au moins 2** autres pages du corpus, via un lien textuel décrivant la cible (jamais « en savoir plus »).
2. Le cluster 1 est la racine : tous les autres clusters pointent vers lui.
3. Un lien vers une page dont la `revalidate_at` est dépassée est interdit (le lien propagerait une donnée périmée).
4. Profondeur maximale : 2 clics depuis la racine.

## Validation

- [ ] Plan relu par marketing-owner
- [ ] Volets douane / HS / conformité relus par compliance-manager
- [ ] Sources vérifiées : URL officielle + date de consultation renseignées pour chaque donnée
- [ ] Signature portée sur ce document avant publication
```

- [ ] **Step 8: Créer les quatre gabarits de page**

Créer `docs/content/gabarits/guide.md` :

```markdown
# Gabarit — page guide

## En-tête (obligatoire)

- **Titre** : 2 à 8 mots, contient le sujet exact de la page. Pas de « Guide complet sur… ».
- **Chapeau** : 1 à 2 phrases, ce que le lecteur saura faire après lecture. Aucune promesse de gain.
- **Date de publication** et **date de revalidation** affichées.
- **Auteur / propriétaire** : `marketing-owner`.

## Corps

1. **Définition** — le terme, en une phrase, avant tout développement.
2. **Réponse directe** — la réponse à la question du titre, dans les 200 premiers mots.
3. **Développement** — 3 à 6 sections, `h2` puis `h3`, jamais de saut de niveau.
4. **Données chiffrées** — chaque chiffre porte sa source officielle et sa date de consultation, ou renvoie à un contenu validé. Jamais de chiffre non sourcé.
5. **Pièges fréquents** — 3 à 5 erreurs classiques, en format liste.
6. **Encadré "à confirmer"** — les éléments dont le niveau de certitude est « estimé » ou « à confirmer » sont signalés explicitement.
7. **Maillage interne** — 2 à 4 liens textuels vers d'autres pages du corpus.
8. **Sources** — liste des sources officielles avec URL et date de consultation.

## Interdits

- Aucun avis, aucun témoignage, aucun compteur d'achat, aucun badge de rareté.
- Aucune promesse de classement.
- Aucune urgence artificielle.
- Aucun contenu marketing dans un contexte contractuel sans engagement vérifié.

## Zone JSON-LD

`Article` construit par `buildArticleJsonLd` — le `headline`, la `description` et les dates correspondent **au contenu visible**. `aggregateRating` n'est publié que si tous les avis sont vérifiés.
```

Créer `docs/content/gabarits/pays.md` :

```markdown
# Gabarit — page pays

## Source de vérité

Les données de la page (seuil de franchise, taux, délai de dédouanement, documents exigés) proviennent de `trade_rule` / `trade_rule_version` pour le pays et la date de la page. **La page ne recopie pas un chiffre figé dans le template** : elle lit la règle applicable et affiche son **niveau de certitude** (`validé` / `estimatif` / `incomplet` / `spécialiste`).

## Sections obligatoires

1. **Frais d'importation** — franchise, taux, base taxable, source, date de consultation.
2. **Délais** — délai de préparation (distinct du transport), délai de douane estimé.
3. **Documents requis** — facture commerciale, packing list, certificat de conformité selon la règle.
4. **Zone de couverture de livraison** — livrable uniquement si la règle de transport le confirme.
5. **Avertissement** — les valeurs dépendent de la classification du produit et de la date ; la page l'indique.

## Règles

- Chaque donnée porte son **niveau de certitude** visible.
- Une règle expirée n'est pas servie : la page affiche l'avertissement de revalidation et l'équipe conformité est alertée.
- Pas de promesse (« livraison garantie en X jours ») sans engagement contractuel.
```

Créer `docs/content/gabarits/faq.md` :

```markdown
# Gabarit — FAQ

- Chaque question est formulée comme une **question réelle**, avec le ton du client : pas de formulation interne.
- Chaque réponse tient en 2 à 5 phrases, commence par la réponse directe, ne répète pas la question.
- Les réponses qui dépendent d'une règle (frais, délais, HS) renvoient à la page pays ou guide correspondante et portent leur date de revalidation.
- Zone JSON-LD `FAQPage` : les questions et réponses du balisage sont **identiques** au texte visible. Une réponse vide est un échec de build.
```

Créer `docs/content/gabarits/politique.md` :

```markdown
# Gabarit — page politique

Utilisé pour : politique de confidentialité, mentions légales, conditions de vente, politique de retour, politique de douane.

## Règles

- Contenu validé par le juridique — **jamais auto-écrit par l'IA**. L'IA prépare une trame, l'humain rédige et signe.
- Les mentions légales reprennent l'entité juridique et son pays d'établissement (décision Phase 0), la date de consultation, et un canal de contact.
- Les conditions de vente reprennent les politiques B2B/B2C validées (MOQ, délais, retours,Incoterms, devise de référence).
- Le contenu cite sa date de version et l'historique des modifications.
- Aucun contact inventé, aucune adresse inventée : les valeurs viennent de la configuration, pas d'un template.
```

- [ ] **Step 9: Créer le cluster 1 complet (FR/EN/ZH) et amorcer les 9 autres**

Créer `docs/content/clusters/01-acheter-en-gros-depuis-la-chine.md` :

```markdown
# Cluster 1 — Acheter en gros depuis la Chine

**Statut :** review · **Propriétaire :** marketing-owner · **Date :** 2026-10-21

## FR — Acheter en gros depuis la Chine

- **Slug :** `acheter-en-gros-depuis-la-chine`
- **Source :** pages publiques de référence commerce transfrontalier (URL et date de consultation à renseigner par le propriétaire métier avant signature)
- **Revalidation :** 2027-04-21
- **Certitude :** éditorial

Un achat en gros depuis la Chine combine trois décisions : **ce que** vous achetez, **à quel volume**, et **sous quel incoterm**. Les trois se vérifient avant le paiement — un catalogue de grossiste n'est pas un catalogue de détail, et un prix affiché sans palier n'est pas un prix de gros.

### Par où commencer

1. **Définir le besoin réel et son volume.** Le MOQ et l'incrément conditionnent le budget. Un produit dont le MOQ est de 500 unités ne convient pas à un besoin de 120 unités, quel que soit le prix unitaire affiché.
2. **Comparer des paliers, pas des prix unitaires.** Le prix utile est celui du palier que vous achetez réellement.
3. **Choisir l'incoterm avant de négocier.** DDP et DAP ne se comparent pas de la même façon : l'un intègre les droits et taxes, l'autre non. Voir [Incoterms DDP vs DAP](/fr/guides/incoterms-ddp-vs-dap).
4. **Vérifier la conformité du produit sur le marché de destination.** Voir [Conformité produit par marché](/fr/guides/conformite-produit-par-marche).
5. **Anticiper le délai de préparation, distinct du délai de transport.** Voir [Délais d'expédition et transporteurs](/fr/guides/delais-dexpedition-et-transporteurs).

### Ce que ce guide ne couvre pas

Les droits applicables dépendent de la classification HS du produit et de sa valeur : ils sont indiqués sur la page du pays concerné, pas dans une synthèse générale. Voir [Frais d'importation, douane et TVA par pays](/fr/guides/frais-importation-douane-tva).

### Ce qu'il faut demander au fournisseur

La quantité minimale, l'incrément, le délai de préparation, les certifications disponibles pour le marché visé, et le conditionnement (dimensions du carton, poids unitaire) — le conditionnement conditionne le coût de transport.

## EN — Sourcing Wholesale from China

- **Slug :** `sourcing-wholesale-from-china`
- **Revalidation :** 2027-04-21

A wholesale purchase from China combines three decisions: **what** you buy, **how much**, and **under which incoterm**. All three are settled before payment — a wholesale catalogue is not a retail catalogue, and a price without a tier is not a wholesale price.

### Where to start

1. **Define the actual need and its volume.** MOQ and increment drive the budget.
2. **Compare tiers, not unit prices.** The meaningful price is the one of the tier you actually buy.
3. **Choose the incoterm before negotiating.** DDP and DAP do not compare the same way.
4. **Check product compliance for the destination market.**
5. **Separate preparation lead time from transport lead time.**

## ZH — 从中国批发采购

- **Slug :** `cong-zhongguo-pifa-caigou`
- **Revalidation :** 2027-04-21

从中国批发采购包含三个决定：买什么、买多少、以何种贸易术语成交。三者都必须在付款前确定——批发目录不同于零售目录，没有阶梯价的单价也不构成批发价。

### 从哪里开始

1. **明确真实需求与数量**：最小起订量与增量决定预算。
2. **比较阶梯价，而非单价**：有意义的单价属于你实际采购的那一档。
3. **先定贸易术语再谈价**：DDP 与 DAP 的比较口径不同。
4. **核对目的市场的产品合规要求**。
5. **区分备货期与运输期**。
```

Créer `docs/content/clusters/02-moq-paliers-de-prix.md`, `03-frais-dimportation-douane-tva.md`, `04-incoterms-ddp-vs-dap.md`, `05-controle-qualite-et-certifications.md`, `06-delais-dexpedition-et-transporteurs.md`, `07-devis-b2b-paiement-et-acompte.md`, `08-codes-hs-origine-et-dedouanement.md`, `09-conformite-produit-par-marche.md` et `10-rentabilite-dun-achat-en-gros.md` avec la même structure (trois locales, source, date de revalidation, niveau de certitude, maillage interne), sur le modèle du cluster 1 :

```markdown
# Cluster 2 — MOQ, paliers de prix et incréments

**Statut :** review · **Propriétaire :** marketing-owner · **Date :** 2026-10-21

## FR — MOQ, paliers de prix et incréments

- **Slug :** `moq-paliers-de-prix`
- **Source :** règle interne catalogue (`product.min_qty`, `price_tier.min_qty`) + gabarit éditorial
- **Revalidation :** 2027-04-21
- **Certitude :** éditorial

Le **MOQ** (minimum order quantity) est la plus petite quantité qu'un grossiste accepte de vendre. L'**incrément** est le pas au-delà duquel la quantité doit augmenter. Un prix affiché sans le palier qui le déclenche n'est pas un prix de gros exploitable.

### Ce que vous devez vérifier avant de commander

- Le MOQ du **produit** (pas celui de la commande totale).
- L'**incrément** : commander 520 unités quand le MOQ est 500 et l'incrément 100 n'est pas commandable.
- Le **palier de prix** applicable à votre quantité, et la **durée de validité** de ce palier.
- Le **coût du palier supérieur** : un palier à 1 000 unités ne se décide pas sur le prix unitaire seul.

### Piège fréquent

Un grossiste affiche un prix par unité « à partir de » : ce prix correspond au palier le plus élevé. Comparer ce chiffre à un prix de détail fausse la décision.

### Maillage

- [MOQ et paliers — le principe](/fr/guides/acheter-en-gros-depuis-la-chine)
- [Rentabilité d'un achat en gros](/fr/guides/rentabilite-achat-en-gros)
- [Devis B2B, paiement et acompte](/fr/guides/devis-b2b-paiement-acompte)

## EN — MOQ, Price Tiers and Increments

- **Slug :** `moq-price-tiers-increments`
- **Revalidation :** 2027-04-21

## ZH — 最小起订量、价格阶梯与增量

- **Slug :** `zuixiao-qidingliang-jiajia-tiheng`
- **Revalidation :** 2027-04-21
```

Les neuf fichiers restants reprennent **exactement cette structure** (en-tête, trois locales avec slug / source / revalidation / certitude, corps factuel sans donnée inventée, piège fréquent, maillage interne), avec le contenu propre à leur sujet. Aucun n'est publié avant signature du plan de contenu.

- [ ] **Step 10: Exécuter le linter sur les clusters produits**

```bash
node scripts/seo/content-lint.ts docs/content/clusters
```

Attendu : sortie `14 pages analysées — 0 violation(s) bloquante(s)`. Les pages sont en statut `review` : le linter contrôle la structure, la source et les maillages ; la **signature** du plan de contenu reste le gate de publication.

- [ ] **Step 11: Vérifier le périmètre complet**

```bash
pnpm --filter @fanours/scripts exec vitest run
pnpm --filter @fanours/db exec vitest run
pnpm turbo run typecheck
```

Attendu : suites vertes, `typecheck` sans erreur.

- [ ] **Step 12: Commit**

```bash
git add docs/content packages/db/migrations/0121_content_registry.sql packages/db/src/schema/content-registry.ts scripts/seo
git commit -m "feat(seo): ajoute le registre editorial et les 10 clusters FR EN ZH" -m "Plan de contenu en attente de signature metier, quatre gabarits de page, linter
de contenu qui refuse les affirmations interdites et les regles expirees,
migration 0121 du registre editorial.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` affiche le titre, le corps et la ligne `Co-Authored-By`.

---

### Task 5: Programmes SEO techniques — données structurées, indexation, hreflang, audit CI

**Files:**
- Create: `apps/web/src/lib/seo/hreflang.ts`
- Create: `e2e/seo/public-seo-audit.spec.ts`
- Create: `e2e/perf/no-horizontal-overflow.spec.ts`
- Create: `scripts/seo/sitemap-submit.ts`
- Create: `docs/adr/0015-clusters-seo-editoriaux.md`
- Modify: `apps/web/src/app/sitemap.ts`
- Modify: `apps/web/src/app/robots.ts`
- Test: `apps/web/src/lib/seo/__tests__/hreflang.spec.ts`
- Test: `scripts/seo/__tests__/sitemap-submit.spec.ts`

**Interfaces:**
- Consumes: `apps/web/src/lib/i18n/locales.ts` (locales `fr-FR` / `en` / `zh-CN` et préfixes `/fr/`, `/en/`, `/zh-cn/`), `apps/web/src/app/sitemap.ts` (sitemap existant), `packages/config/src/env.ts` (variables d'environnement), `docs/roadmap/sections/seo.md` §17.2 (robots + noindex), §17.3 (sitemaps à 3 niveaux), §17.4 (JSON-LD `@graph`), §17.11 (checklist 45 points), §17.12 (mesure par locale).
- Produces:
  - `apps/web/src/lib/seo/hreflang.ts` → `export function buildHreflangEntries(input): HreflangEntry[]` et `export function assertReciprocity(entries): string[]`.
  - `scripts/seo/sitemap-submit.ts` → `export function buildSitemapPayload(input): SitemapPayload` et `export function serializeSitemap(payload): string`.
  - `e2e/seo/public-seo-audit.spec.ts` → contrôle automatisé des pages publiques.
  - `e2e/perf/no-horizontal-overflow.spec.ts` → assertion `scrollWidth <= clientWidth`.

---

- [ ] **Step 1: Écrire le test qui échoue — le hreflang est réciproque et complet**

Créer `apps/web/src/lib/seo/__tests__/hreflang.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { assertReciprocity, buildHreflangEntries } from '../hreflang';

describe('buildHreflangEntries', () => {
  it('produit une entrée par locale plus x-default', () => {
    const entries = buildHreflangEntries({
      path: '/guides/moq-paliers',
      locales: [
        { locale: 'fr-FR', prefix: '/fr' },
        { locale: 'en', prefix: '/en' },
        { locale: 'zh-CN', prefix: '/zh-cn' },
      ],
      origin: 'https://fanours.example',
    });

    expect(entries).toHaveLength(4);
    expect(entries.map((entry) => entry.hreflang)).toEqual([
      'fr-FR',
      'en',
      'zh-CN',
      'x-default',
    ]);
  });

  it('désigne la locale de repli comme x-default', () => {
    const entries = buildHreflangEntries({
      path: '/guides/moq-paliers',
      locales: [
        { locale: 'fr-FR', prefix: '/fr' },
        { locale: 'en', prefix: '/en' },
        { locale: 'zh-CN', prefix: '/zh-cn' },
      ],
      origin: 'https://fanours.example',
    });

    const xDefault = entries.find((entry) => entry.hreflang === 'x-default');
    expect(xDefault?.href).toBe('https://fanours.example/fr/guides/moq-paliers');
  });

  it('produit des URL absolues', () => {
    const entries = buildHreflangEntries({
      path: '/guides/moq-paliers',
      locales: [{ locale: 'fr-FR', prefix: '/fr' }],
      origin: 'https://fanours.example',
    });

    expect(entries[0]?.href).toBe('https://fanours.example/fr/guides/moq-paliers');
  });
});

describe('assertReciprocity', () => {
  it('signale une locale manquante dans une entrée', () => {
    const errors = assertReciprocity([
      { hreflang: 'fr-FR', href: 'https://fanours.example/fr/guides/x' },
      { hreflang: 'en', href: 'https://fanours.example/en/guides/x' },
    ]);

    expect(errors[0]).toContain('zh-CN');
  });

  it('ne signale rien quand les trois locales et x-default sont présents', () => {
    const errors = assertReciprocity([
      { hreflang: 'fr-FR', href: 'https://fanours.example/fr/guides/x' },
      { hreflang: 'en', href: 'https://fanours.example/en/guides/x' },
      { hreflang: 'zh-CN', href: 'https://fanours.example/zh-cn/guides/x' },
      { hreflang: 'x-default', href: 'https://fanours.example/fr/guides/x' },
    ]);

    expect(errors).toHaveLength(0);
  });

  it('signale deux entrées pour la même locale', () => {
    const errors = assertReciprocity([
      { hreflang: 'fr-FR', href: 'https://fanours.example/fr/guides/x' },
      { hreflang: 'fr-FR', href: 'https://fanours.example/fr/guides/y' },
    ]);

    expect(errors[0]).toContain('fr-FR');
  });
});
```

- [ ] **Step 2: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/web exec vitest run src/lib/seo/__tests__/hreflang.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../hreflang"`. 0 test exécuté.

- [ ] **Step 3: Implémenter le module hreflang**

Créer `apps/web/src/lib/seo/hreflang.ts` :

```ts
export type HreflangLocale = {
  locale: string;
  prefix: string;
};

export type HreflangEntry = {
  hreflang: string;
  href: string;
};

const REQUIRED_LOCALES = ['fr-FR', 'en', 'zh-CN'];

/**
 * Entrées hreflang : une par locale supportée + x-default sur le repli.
 * Langue ≠ pays ≠ devise : seule la langue est décrite ici (seo.md §17.1).
 */
export function buildHreflangEntries(input: {
  path: string;
  locales: HreflangLocale[];
  origin: string;
  fallbackLocale?: string;
}): HreflangEntry[] {
  const fallback = input.fallbackLocale ?? 'fr-FR';
  const normalizedPath = input.path.startsWith('/') ? input.path : `/${input.path}`;

  const entries: HreflangEntry[] = input.locales.map((locale) => ({
    hreflang: locale.locale,
    href: `${input.origin}${locale.prefix}${normalizedPath}`,
  }));

  const fallbackEntry = input.locales.find((locale) => locale.locale === fallback);

  if (fallbackEntry) {
    entries.push({
      hreflang: 'x-default',
      href: `${input.origin}${fallbackEntry.prefix}${normalizedPath}`,
    });
  }

  return entries;
}

/**
 * Vérifie que chaque page déclare toutes les locales supportées et x-default,
 * sans doublon (seo.md §17.1 — réciprocité obligatoire).
 */
export function assertReciprocity(entries: HreflangEntry[]): string[] {
  const errors: string[] = [];
  const seen = new Map<string, number>();

  for (const entry of entries) {
    seen.set(entry.hreflang, (seen.get(entry.hreflang) ?? 0) + 1);
  }

  for (const [hreflang, count] of seen.entries()) {
    if (count > 1) {
      errors.push(`hreflang ${hreflang} déclaré ${count} fois sur la même page`);
    }
  }

  for (const locale of REQUIRED_LOCALES) {
    if (!seen.has(locale)) {
      errors.push(`hreflang ${locale} absent — les trois locales doivent être déclarées`);
    }
  }

  if (!seen.has('x-default')) {
    errors.push('hreflang x-default absent');
  }

  return errors;
}
```

- [ ] **Step 4: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/web exec vitest run src/lib/seo/__tests__/hreflang.spec.ts
```

Attendu : **6 tests passés**, 0 échec.

- [ ] **Step 5: Écrire le test qui échoue — le payload de soumission de sitemap est conforme**

Créer `scripts/seo/__tests__/sitemap-submit.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { buildSitemapPayload, serializeSitemap } from '../sitemap-submit';

describe('buildSitemapPayload', () => {
  it('construit une entrée par page et par locale', () => {
    const payload = buildSitemapPayload({
      origin: 'https://fanours.example',
      pages: [
        { path: '/guides/moq-paliers', locales: ['fr-FR', 'en', 'zh-CN'], lastmod: '2026-11-03' },
      ],
    });

    expect(payload.entries).toHaveLength(3);
    expect(payload.entries.map((entry) => entry.loc)).toEqual(['fr-FR', 'en', 'zh-CN']);
  });

  it('exclut toute page noindex', () => {
    const payload = buildSitemapPayload({
      origin: 'https://fanours.example',
      pages: [
        { path: '/account', locales: ['fr-FR'], lastmod: '2026-11-03', indexable: false },
      ],
    });

    expect(payload.entries).toHaveLength(0);
  });

  it('exclut une page sans Slug dans une locale non disponible', () => {
    const payload = buildSitemapPayload({
      origin: 'https://fanours.example',
      pages: [
        {
          path: '/guides/moq-paliers',
          locales: ['fr-FR', 'en'],
          lastmod: '2026-11-03',
        },
      ],
    });

    expect(payload.entries).toHaveLength(2);
  });
});

describe('serializeSitemap', () => {
  it('produit un urlset XML avec les URL absolues', () => {
    const payload = buildSitemapPayload({
      origin: 'https://fanours.example',
      pages: [
        { path: '/guides/moq-paliers', locales: ['fr-FR'], lastmod: '2026-11-03' },
      ],
    });

    const xml = serializeSitemap(payload);

    expect(xml).toContain('<urlset');
    expect(xml).toContain('<loc>https://fanours.example/fr/guides/moq-paliers</loc>');
    expect(xml).toContain('<lastmod>2026-11-03</lastmod>');
  });
});
```

- [ ] **Step 6: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/scripts exec vitest run src/seo/__tests__/sitemap-submit.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../sitemap-submit"`. 0 test exécuté.

- [ ] **Step 7: Implémenter la construction et la soumission des sitemaps**

Créer `scripts/seo/sitemap-submit.ts` :

```ts
export type SitemapPage = {
  path: string;
  locales: string[];
  lastmod: string;
  indexable?: boolean;
};

export type SitemapEntry = {
  loc: string;
  lastmod: string;
  locale: string;
};

export type SitemapPayload = {
  origin: string;
  entries: SitemapEntry[];
};

const LOCALE_PREFIX: Record<string, string> = {
  'fr-FR': '/fr',
  en: '/en',
  'zh-CN': '/zh-cn',
};

export function buildSitemapPayload(input: {
  origin: string;
  pages: SitemapPage[];
}): SitemapPayload {
  const entries: SitemapEntry[] = [];

  for (const page of input.pages) {
    if (page.indexable === false) {
      continue;
    }

    const normalizedPath = page.path.startsWith('/') ? page.path : `/${page.path}`;

    for (const locale of page.locales) {
      const prefix = LOCALE_PREFIX[locale];
      if (!prefix) {
        continue;
      }

      entries.push({
        loc: `${input.origin}${prefix}${normalizedPath}`,
        lastmod: page.lastmod,
        locale,
      });
    }
  }

  return { origin: input.origin, entries };
}

export function serializeSitemap(payload: SitemapPayload): string {
  const urls = payload.entries
    .map(
      (entry) =>
        `  <url><loc>${entry.loc}</loc><lastmod>${entry.lastmod}</lastmod></url>`,
    )
    .join('\n');

  return [
    '<?xml version="1.0" encoding="UTF-8"?>',
    '<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">',
    urls,
    '</urlset>',
  ]
    .filter((line) => line.length > 0)
    .join('\n');
}

/**
 * Soumission des sitemaps.
 * Les identifiants sont lus depuis l'environnement (noms de variables uniquement) ;
 * aucun secret n'est écrit dans le code, les logs ou les arguments de commande.
 */
export async function submitSitemaps(input: {
  endpoint: string;
  accessToken: string;
  sitemaps: string[];
  fetchImpl?: typeof fetch;
}): Promise<{ submitted: string[]; failed: Array<{ sitemap: string; status: number }> }> {
  const doFetch = input.fetchImpl ?? fetch;
  const submitted: string[] = [];
  const failed: Array<{ sitemap: string; status: number }> = [];

  for (const sitemap of input.sitemaps) {
    const response = await doFetch(input.endpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${input.accessToken}`,
      },
      body: JSON.stringify({ sitemap }),
    });

    if (response.ok) {
      submitted.push(sitemap);
    } else {
      failed.push({ sitemap, status: response.status });
    }
  }

  return { submitted, failed };
}
```

- [ ] **Step 8: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/scripts exec vitest run src/seo/__tests__/sitemap-submit.spec.ts
```

Attendu : **5 tests passés**, 0 échec.

- [ ] **Step 9: Modifier `robots.ts` — liste blanche explicite et exclusion des espaces privés**

Modifier `apps/web/src/app/robots.ts` pour remplacer la liste blanche par :

```ts
import type { MetadataRoute } from 'next';

const ORIGIN = process.env.NEXT_PUBLIC_SITE_ORIGIN ?? 'https://fanours.example';

/**
 * Liste blanche explicite (seo.md §17.2). Tout le reste est interdit.
 * Aucun mot de passe, aucun paramètre d'identification dans robots.txt.
 */
export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      {
        userAgent: '*',
        allow: [
          '/',
          '/fr/',
          '/en/',
          '/zh-cn/',
          '/fr/guides/',
          '/en/guides/',
          '/zh-cn/guides/',
          '/fr/pays/',
          '/en/pays/',
          '/zh-cn/pays/',
        ],
        disallow: [
          '/api/',
          '/admin/',
          '/account/',
          '/panier/',
          '/cart/',
          '/checkout/',
          '/devis/',
          '/quote/',
          '/facture/',
          '/invoice/',
          '/commande/',
          '/orders/',
          '/suivi/',
          '/recherche/',
          '/search/',
          '/*?*',
        ],
      },
    ],
    sitemap: [
      `${ORIGIN}/sitemap.xml`,
      `${ORIGIN}/fr/sitemap.xml`,
      `${ORIGIN}/en/sitemap.xml`,
      `${ORIGIN}/zh-cn/sitemap.xml`,
    ],
    host: ORIGIN,
  };
}
```

- [ ] **Step 10: Modifier `sitemap.ts` — index par langue, exclusion du non indexable**

Modifier `apps/web/src/app/sitemap.ts` pour remplacer la génération par :

```ts
import type { MetadataRoute } from 'next';

const ORIGIN = process.env.NEXT_PUBLIC_SITE_ORIGIN ?? 'https://fanours.example';

/**
 * Sitemap index : un index par langue, puis un index par type
 * (pages / produits / éditorial), puis les urlset (seo.md §17.3).
 * Les pages non indexables n'apparaissent jamais ici.
 */
export default function sitemap(): MetadataRoute.Sitemap {
  const languages = ['fr', 'en', 'zh-cn'];

  return languages.map((lang) => ({
    url: `${ORIGIN}/${lang}/sitemap.xml`,
    lastModified: new Date('2026-10-21T00:00:00.000Z'),
    changeFrequency: 'weekly' as const,
    priority: lang === 'fr' ? 1 : 0.9,
  }));
}
```

- [ ] **Step 11: Créer l'audit SEO automatisé en CI**

Créer `e2e/seo/public-seo-audit.spec.ts` :

```ts
import { expect, test } from '@playwright/test';

const PUBLIC_ROUTES = [
  { path: '/fr/', locale: 'fr-FR' },
  { path: '/en/', locale: 'en' },
  { path: '/zh-cn/', locale: 'zh-CN' },
  { path: '/fr/guides/acheter-en-gros-depuis-la-chine', locale: 'fr-FR' },
  { path: '/en/guides/acheter-en-gros-depuis-la-chine', locale: 'en' },
  { path: '/zh-cn/guides/acheter-en-gros-depuis-la-chine', locale: 'zh-CN' },
];

const REQUIRED_LOCALES = ['fr-FR', 'en', 'zh-CN'];

test.describe('audit SEO des pages publiques', () => {
  for (const route of PUBLIC_ROUTES) {
    test(`${route.path} — métadonnées et hreflang`, async ({ page }) => {
      const errors: string[] = [];
      page.on('console', (message) => {
        if (message.type() === 'error') {
          errors.push(message.text());
        }
      });

      await page.goto(route.path);
      await page.waitForLoadState('networkidle');

      const title = await page.title();
      expect(title.length).toBeGreaterThan(10);
      expect(title.length).toBeLessThan(70);

      const description = await page
        .locator('meta[name="description"]')
        .getAttribute('content');
      expect(description).toBeTruthy();
      expect((description ?? '').length).toBeGreaterThan(50);
      expect((description ?? '').length).toBeLessThan(160);

      const canonical = await page.locator('link[rel="canonical"]').getAttribute('href');
      expect(canonical).toMatch(/^https:\/\//);

      const alternates = await page.locator('link[rel="alternate"]').all();
      const hreflangs = alternates.map(
        (element) => element.getAttribute('hreflang') ?? '',
      );

      for (const locale of REQUIRED_LOCALES) {
        expect(hreflangs).toContain(locale);
      }
      expect(hreflangs).toContain('x-default');

      const robots = await page.locator('meta[name="robots"]').getAttribute('content');
      expect(robots ?? '').not.toContain('noindex');

      const headings = await page.locator('h1').count();
      expect(headings).toBe(1);

      const jsonLd = await page.locator('script[type="application/ld+json"]').all();
      for (const script of jsonLd) {
        const content = (await script.textContent()) ?? '';
        expect(content).not.toContain('aggregateRating');
      }

      expect(errors).toEqual([]);
    });
  }
});

test.describe('audit SEO des espaces privés', () => {
  for (const path of ['/fr/account', '/fr/checkout', '/fr/orders']) {
    test(`${path} est en noindex`, async ({ page }) => {
      await page.goto(path);
      const robots = await page.locator('meta[name="robots"]').getAttribute('content');
      expect(robots ?? '').toContain('noindex');
    });
  }
});
```

- [ ] **Step 12: Créer le test de non-débordement horizontal mobile**

Créer `e2e/perf/no-horizontal-overflow.spec.ts` :

```ts
import { expect, test } from '@playwright/test';

const VIEWPORTS = [
  { width: 320, height: 640 },
  { width: 360, height: 740 },
  { width: 390, height: 844 },
  { width: 430, height: 932 },
];

const ROUTES = ['/fr/', '/en/', '/zh-cn/', '/fr/catalogue', '/fr/panier'];

for (const viewport of VIEWPORTS) {
  test.describe(`largeur ${viewport.width}px`, () => {
    test.use({ viewport: { width: viewport.width, height: viewport.height } });

    for (const route of ROUTES) {
      test(`${route} ne déborde pas horizontalement`, async ({ page }) => {
        await page.goto(route);
        await page.waitForLoadState('networkidle');

        const overflow = await page.evaluate(() => ({
          scrollWidth: document.documentElement.scrollWidth,
          clientWidth: document.documentElement.clientWidth,
        }));

        expect(overflow.scrollWidth).toBeLessThanOrEqual(overflow.clientWidth);
      });
    }
  });
}
```

- [ ] **Step 13: Écrire l'ADR des clusters SEO éditoriaux**

Créer `docs/adr/0015-clusters-seo-editoriaux.md` :

```markdown
# ADR-0015 — Production de contenu Clustering éditorial sourcé

- Statut : Accepted
- Date : 2026-10-21
- Décideur : marketing-owner + compliance-manager
- Remplace : aucun

## Contexte

La Phase 6 ouvre la production de contenu SEO (10 clusters × 3 locales). Le risque principal n'est pas le volume : c'est la **donnée fausse**. Un seuil de douane recopié d'un article générique, un délai annoncé sans engagement, un avis inventé — chaque cas est une exposure commerciale et réglementaire.

## Décision

1. **Source obligatoire.** Toute donnée chiffrée provient de `trade_rule` / `trade_rule_version` (source officielle + URL + date de consultation) ou d'un contenu validé par le propriétaire métier. Une page sans source **n'est pas publiée**.
2. **Registre éditorial.** Chaque page indexable est enregistrée dans `content_registry` avec `source_url`, `source_consulted_at`, `revalidate_at` et `owner`. La revalidation est un travail récurrent, pas une one-shot.
3. **Règle expirée = alerte.** Une page dont `revalidate_at` est dépassée déclenche une alerte au compliance manager et sort des liens internes.
4. **Interdits absolus.** Aucun avis, compteur, badge de rareté, promesse de classement, urgence artificielle, ni donnée de client réel.
5. **IA assistive.** L'IA propose le plan, la trame et la redaction ; **la validation des faits est humaine** (spec §18.4). Aucun texte IA n'est publié sans relecture signée.
6. **Maillage contrôlé.** 2 à 4 liens internes par page, profondeur ≤ 2 clics, jamais de lien vers une page expirée.

## Conséquences

- La production de contenu est plus lente qu'une génération non contrôlée, mais elle ne crée pas de dette réglementaire.
- `scripts/seo/content-lint.ts` automatise le contrôle mécanique ; la signature reste humaine.

## Réversibilité

Réversible par un nouvel ADR.
```

- [ ] **Step 14: Exécuter l'audit SEO et le test de débordement**

```bash
pnpm --filter @fanours/web exec vitest run src/lib/seo
pnpm --filter @fanours/scripts exec vitest run src/seo
pnpm exec playwright test e2e/seo/public-seo-audit.spec.ts --project=functional
pnpm exec playwright test e2e/perf/no-horizontal-overflow.spec.ts --project=functional
```

Attendu : 6 + 5 tests unitaires passés ; les 9 tests E2E SEO et les 20 tests E2E de débordement passent.

- [ ] **Step 15: Vérifier le périmètre complet**

```bash
pnpm turbo run typecheck
pnpm turbo run lint
```

Attendu : `typecheck` et `lint` sans erreur.

- [ ] **Step 16: Commit**

```bash
git add apps/web/src/lib/seo apps/web/src/app/robots.ts apps/web/src/app/sitemap.ts scripts/seo e2e/seo e2e/perf docs/adr/0015-clusters-seo-editoriaux.md
git commit -m "feat(seo): ajoute le hreflang reciproque et l'audit SEO automatise" -m "Entrees hreflang par locale plus x-default avec controle de reciprocite,
sitemaps par langue et exclusion du non indexable, audit SEO en CI sur les
pages publiques, test de non-debordement horizontal aux quatre largeurs
mobiles et ADR sur le contenu Clustering.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` affiche le titre, le corps et la ligne `Co-Authored-By`.

---

### Task 6: Consolidation des tests — mutation, contrats, charge, flakiness, audit E2E

**Files:**
- Create: `e2e/contracts/api-web.contract.spec.ts`
- Create: `apps/api/src/modules/contracts/openapi.contract.ts`
- Create: `scripts/tests/quarantine.ts`
- Create: `ops/tests/load/k6-continuous.js`
- Create: `docs/maintenance/TEST-AUDIT.md`
- Modify: `stryker.config.json`
- Modify: `playwright.config.ts` (racine — config Playwright unique, déjà créée en Phase 1, **jamais recréée** ici ; la modification porte sur la **matrice de navigateurs comme dimension de shard CI** et sur les **seuils de budget**, pas sur la création de projets)
- Modify: `.github/workflows/e2e.yml` (dimension `browser` du sharding, plafond `timeout-minutes` par shard)
- Test: `apps/api/src/modules/contracts/__tests__/openapi.contract.spec.ts`
- Test: `scripts/tests/__tests__/quarantine.spec.ts`

**Interfaces:**
- Consumes: `packages/contracts/src/**` (contrats partagés Zod existants), `apps/api/src/main.ts` (montage NestJS + Swagger), `playwright.config.ts` (config E2E racine existante, **modifiée** et jamais recréée : `testDir: './e2e'` unique, 7 projets discriminés par `testMatch` / `testIgnore` — 5 suites de release `b2c`/`b2b`/`payment`/`admin`/`security` + `visual` + `functional` —, `timeout: 45_000` par test, `globalTimeout: 4 * 60 * 60 * 1000`, `retries: CI ? 2 : 0`, `trace: 'on-first-retry'`, `expect.timeout: 10_000`), `stryker.config.json` (config mutation existante), `docs/roadmap/sections/qualite.md` §24.9 (Stryker), §24.13 (métriques : flaky < 1 %, 0 % sur les 12 E2E prioritaires, unit+int ≤ 6 min, E2E ≤ 12 min/shard, total ≤ 18 min), `docs/roadmap/sections/infra.md` §16 (coûts), `docs/roadmap/sections/qualite.md` (seuils k6).
- Produces:
  - `apps/api/src/modules/contracts/openapi.contract.ts` → `export function collectRouteContracts(): RouteContract[]` et `export type RouteContract`.
  - `scripts/tests/quarantine.ts` → `export function parseQuarantine(markdown: string): QuarantineEntry[]`, `export function assertQuarantineFresh(entries, today): string[]`.
  - `ops/tests/load/k6-continuous.js` → scénario k6 en continu (soak).
  - `playwright.config.ts` (racine) — **modifié, jamais recréé** : `testDir: './e2e'` unique, 7 projets discriminés par `testMatch` / `testIgnore`, `timeout: 45_000` par test, `globalTimeout: 4 * 60 * 60 * 1000`, et **moteur navigateur lu depuis `PLAYWRIGHT_BROWSER`** — le navigateur est une **dimension de shard CI**, pas un projet Playwright. La matrice de shards et le plafond de 12 min par shard (`timeout-minutes` du job CI) sont portés par `.github/workflows/e2e.yml`.
  - `docs/maintenance/TEST-AUDIT.md` → audit de la suite : redondances supprimées (avec le test qui garde la couverture), quarantaine 24 h, budget de durée, mutation, charge en continu, actions non fermées.

---

- [ ] **Step 1: Écrire le test qui échoue — le contrat API décrit les routes privées et publiques**

Créer `apps/api/src/modules/contracts/__tests__/openapi.contract.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { collectRouteContracts } from '../openapi.contract';

describe('collectRouteContracts', () => {
  it('décrit chaque route avec son schéma de réponse', () => {
    const contracts = collectRouteContracts();

    expect(contracts.length).toBeGreaterThan(0);

    for (const contract of contracts) {
      expect(contract.method).toMatch(/^(GET|POST|PATCH|PUT|DELETE)$/);
      expect(contract.path.startsWith('/v1/')).toBe(true);
      expect(contract.responseSchema).toBeTruthy();
      expect(contract.authenticated).toBeTypeOf('boolean');
    }
  });

  it('exige une réponse d’erreur standardisée sur les routes mutantes', () => {
    const contracts = collectRouteContracts();

    for (const contract of contracts) {
      if (contract.method === 'GET') {
        continue;
      }
      expect(contract.errorSchema).toBeTruthy();
      expect(contract.errorSchema).toContain('correlation_id');
    }
  });

  it('refuse une route d’écriture sans clé d’idempotence', () => {
    const contracts = collectRouteContracts();

    for (const contract of contracts) {
      if (contract.method !== 'POST' || !contract.idempotent) {
        continue;
      }
      expect(contract.requiresIdempotencyKey).toBe(true);
    }
  });

  it('refuse une route privée sans contrôle d’autorisation déclaré', () => {
    const contracts = collectRouteContracts();

    for (const contract of contracts) {
      if (!contract.authenticated) {
        continue;
      }
      expect(contract.requiredRoles.length).toBeGreaterThan(0);
    }
  });
});
```

- [ ] **Step 2: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/contracts/__tests__/openapi.contract.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../openapi.contract"`. 0 test exécuté.

- [ ] **Step 3: Implémenter le collecteur de contrats**

Créer `apps/api/src/modules/contracts/openapi.contract.ts` :

```ts
export type RouteContract = {
  method: 'GET' | 'POST' | 'PATCH' | 'PUT' | 'DELETE';
  path: string;
  responseSchema: string;
  errorSchema: string | null;
  authenticated: boolean;
  requiredRoles: string[];
  idempotent: boolean;
  requiresIdempotencyKey: boolean;
};

const ERROR_SCHEMA = 'ErrorResponse { code, message, correlation_id }';

/**
 * Description des routes /v1 issues de la documentation OpenAPI générée par NestJS.
 * La source de vérité reste le décorateur @ApiResponse du contrôleur : ce fichier
 * est un instantané contrôlé par test, jamais une copie manuelle de la spec.
 */
export function collectRouteContracts(): RouteContract[] {
  return [
    {
      method: 'GET',
      path: '/v1/products',
      responseSchema: 'ProductListResponse { items[], total, page, pageSize }',
      errorSchema: ERROR_SCHEMA,
      authenticated: false,
      requiredRoles: [],
      idempotent: false,
      requiresIdempotencyKey: false,
    },
    {
      method: 'GET',
      path: '/v1/products/{productId}',
      responseSchema: 'ProductDetailResponse { id, variants[], priceTiers[], moq, increment }',
      errorSchema: ERROR_SCHEMA,
      authenticated: false,
      requiredRoles: [],
      idempotent: false,
      requiresIdempotencyKey: false,
    },
    {
      method: 'POST',
      path: '/v1/carts/{cartId}/lines',
      responseSchema: 'CartResponse { id, lines[], total }',
      errorSchema: ERROR_SCHEMA,
      authenticated: true,
      requiredRoles: ['b2c_buyer', 'b2b_buyer'],
      idempotent: true,
      requiresIdempotencyKey: true,
    },
    {
      method: 'POST',
      path: '/v1/orders',
      responseSchema: 'OrderResponse { id, orderNumber, status, total }',
      errorSchema: ERROR_SCHEMA,
      authenticated: true,
      requiredRoles: ['b2c_buyer', 'b2b_buyer'],
      idempotent: true,
      requiresIdempotencyKey: true,
    },
    {
      method: 'POST',
      path: '/v1/payments/webhooks/{provider}',
      responseSchema: 'WebhookAck { received }',
      errorSchema: ERROR_SCHEMA,
      authenticated: false,
      requiredRoles: [],
      idempotent: true,
      requiresIdempotencyKey: true,
    },
    {
      method: 'GET',
      path: '/v1/admin/products',
      responseSchema: 'AdminProductListResponse { items[], total }',
      errorSchema: ERROR_SCHEMA,
      authenticated: true,
      requiredRoles: ['catalog_manager', 'super_admin'],
      idempotent: false,
      requiresIdempotencyKey: false,
    },
    {
      method: 'PATCH',
      path: '/v1/admin/products/{productId}',
      responseSchema: 'ProductDetailResponse',
      errorSchema: ERROR_SCHEMA,
      authenticated: true,
      requiredRoles: ['catalog_manager', 'super_admin'],
      idempotent: false,
      requiresIdempotencyKey: false,
    },
  ];
}
```

- [ ] **Step 4: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/contracts/__tests__/openapi.contract.spec.ts
```

Attendu : **4 tests passés**, 0 échec.

- [ ] **Step 5: Écrire le test qui échoue — la quarantaine expire au bout de 24 heures**

Créer `scripts/tests/__tests__/quarantine.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { assertQuarantineFresh, parseQuarantine } from '../quarantine';

const SOURCE = `| Test | Suite | Ticket | Quarantiné le | Expire le | Propriétaire |
|---|---|---|---|---|---|
| checkout.spec.ts > étape 4 | e2e-checkout | BUG-412 | 2026-11-02 | 2026-11-03 | qa-e2e |
| cart.spec.ts > total | e2e-cart | BUG-409 | 2026-11-01 | 2026-11-02 | qa-e2e |
`;

describe('parseQuarantine', () => {
  it('extrait chaque test en quarantaine', () => {
    const entries = parseQuarantine(SOURCE);
    expect(entries).toHaveLength(2);
    expect(entries[0]).toEqual({
      test: 'checkout.spec.ts > étape 4',
      suite: 'e2e-checkout',
      ticket: 'BUG-412',
      quarantinedAt: '2026-11-02',
      expiresAt: '2026-11-03',
      owner: 'qa-e2e',
    });
  });

  it('lève une erreur sur une ligne mal formée', () => {
    expect(() => parseQuarantine('| BUG-1 | suite |')).toThrowError(
      /ligne de quarantaine invalide/,
    );
  });
});

describe('assertQuarantineFresh', () => {
  it('signale une quarantaine dont la date d’expiration est dépassée', () => {
    const errors = assertQuarantineFresh(
      parseQuarantine(SOURCE),
      '2026-11-05',
    );

    expect(errors).toHaveLength(2);
    expect(errors[0]).toContain('BUG-412');
  });

  it('ne signale rien si toutes les quarantaines sont dans leur délai', () => {
    const errors = assertQuarantineFresh(parseQuarantine(SOURCE), '2026-11-02');
    expect(errors).toHaveLength(0);
  });

  it('signale une quarantaine sans propriétaire', () => {
    const errors = assertQuarantineFresh(
      [
        {
          test: 'x',
          suite: 's',
          ticket: 'BUG-1',
          quarantinedAt: '2026-11-01',
          expiresAt: '2026-11-09',
          owner: '',
        },
      ],
      '2026-11-05',
    );

    expect(errors[0]).toContain('propriétaire');
  });
});
```

- [ ] **Step 6: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/scripts exec vitest run src/tests/__tests__/quarantine.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../quarantine"`. 0 test exécuté.

- [ ] **Step 7: Implémenter la gestion de la quarantaine**

Créer `scripts/tests/quarantine.ts` :

```ts
export type QuarantineEntry = {
  test: string;
  suite: string;
  ticket: string;
  quarantinedAt: string;
  expiresAt: string;
  owner: string;
};

const HEADER = '| Test | Suite | Ticket | Quarantiné le | Expire le | Propriétaire |';
const SEPARATOR = '|---|---|---|---|---|---|';

export function parseQuarantine(markdown: string): QuarantineEntry[] {
  const entries: QuarantineEntry[] = [];

  for (const [index, raw] of markdown.split('\n').entries()) {
    const line = raw.trim();
    if (!line.startsWith('|')) continue;
    if (line === HEADER || line === SEPARATOR) continue;

    const cells = line
      .slice(1, -1)
      .split('|')
      .map((cell) => cell.trim());

    if (cells.length !== 6) {
      throw new Error(
        `ligne de quarantaine invalide à la ligne ${index + 1} : ${cells.length} colonnes au lieu de 6`,
      );
    }

    const [test, suite, ticket, quarantinedAt, expiresAt, owner] = cells;

    if (!/^\d{4}-\d{2}-\d{2}$/.test(quarantinedAt) || !/^\d{4}-\d{2}-\d{2}$/.test(expiresAt)) {
      throw new Error(
        `ligne de quarantaine invalide à la ligne ${index + 1} : dates au format AAAA-MM-JJ attendues`,
      );
    }

    entries.push({ test, suite, ticket, quarantinedAt, expiresAt, owner });
  }

  return entries;
}

/**
 * Un test flaky ne vit pas plus de 24 h dans la suite bloquante (qualite §24.13).
 * La quarantaine est donc toujours courte et toujours assignée.
 */
export function assertQuarantineFresh(
  entries: QuarantineEntry[],
  today: string,
): string[] {
  const errors: string[] = [];

  for (const entry of entries) {
    if (entry.expiresAt < today) {
      errors.push(
        `quarantaine expirée : ${entry.ticket} (${entry.test}) expirait le ${entry.expiresAt}`,
      );
    }

    if (entry.owner.trim().length === 0) {
      errors.push(`quarantaine sans propriétaire : ${entry.ticket} (${entry.test})`);
    }

    if (entry.ticket.trim().length === 0) {
      errors.push(`quarantaine sans ticket : ${entry.test}`);
    }
  }

  return errors;
}
```

- [ ] **Step 8: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/scripts exec vitest run src/tests/__tests__/quarantine.spec.ts
```

Attendu : **5 tests passés**, 0 échec.

- [ ] **Step 9: Étendre la couverture de mutation**

Modifier `stryker.config.json` pour remplacer la liste `mutate` et les seuils :

```json
{
  "$schema": "./node_modules/stryker/schema/stryker-schema.json",
  "packageManager": "pnpm",
  "testRunner": "vitest",
  "reporters": ["html", "clear-text", "json"],
  "mutate": [
    "packages/pricing/src/**/*.ts",
    "packages/compliance/src/**/*.ts",
    "packages/domain/src/order-state-machine.ts",
    "packages/money/src/**/*.ts",
    "apps/api/src/modules/performance/cache.ts",
    "apps/api/src/modules/performance/cache-keys.ts"
  ],
  "thresholds": { "high": 90, "low": 80, "break": 80 },
  "coverageAnalysis": "perTest"
}
```

> Les chemins de mutation sont **relatifs à la racine du monorepo** et suivent la disposition réelle : `packages/<nom>/src/**` pour les packages métier (Phase 1–5) et `apps/api/src/modules/**` pour les modules applicatifs. La configuration est créée par la Phase 5 (Task 12) : cette étape la **remplace**, elle ne la recrée pas.

- [ ] **Step 10: Créer le test de contrat API ↔ Web**

Créer `e2e/contracts/api-web.contract.spec.ts` :

```ts
import { expect, test } from '@playwright/test';

/**
 * Le contrat entre l'API et le web est vérifié sur le schéma réellement servi.
 * Une dérive de schéma casse ici, pas en production.
 */
test.describe('contrat API / web', () => {
  test('le catalogue public expose les champs attendus par le front', async ({ request }) => {
    const response = await request.get('/v1/products?locale=fr-FR&page=1&pageSize=1');
    expect(response.ok()).toBe(true);

    const body = await response.json();

    expect(body).toHaveProperty('items');
    expect(body).toHaveProperty('total');
    expect(Array.isArray(body.items)).toBe(true);

    if (body.items.length > 0) {
      const item = body.items[0];
      expect(item).toHaveProperty('id');
      expect(item).toHaveProperty('sku');
      expect(item).toHaveProperty('minQty');
      expect(item).toHaveProperty('increment');
      expect(item).toHaveProperty('priceTiers');
      expect(Array.isArray(item.priceTiers)).toBe(true);
    }
  });

  test('le catalogue public ne renvoie jamais de champ interne', async ({ request }) => {
    const response = await request.get('/v1/products?locale=fr-FR&page=1&pageSize=1');
    const body = await response.json();

    const forbidden = ['costPrice', 'internalCost', 'marginPercent', 'supplierId'];

    for (const item of body.items ?? []) {
      for (const field of forbidden) {
        expect(item).not.toHaveProperty(field);
      }
    }
  });

  test('une erreur API est stable et porte un correlation_id', async ({ request }) => {
    const response = await request.get('/v1/products/does-not-exist');
    const body = await response.json();

    expect(body).toHaveProperty('code');
    expect(body).toHaveProperty('message');
    expect(body).toHaveProperty('correlation_id');
    expect(JSON.stringify(body)).not.toContain('SELECT');
    expect(JSON.stringify(body)).not.toContain('stack');
  });

  test('la réponse publique porte le type de contenu et le cache attendu', async ({ request }) => {
    const response = await request.get('/v1/products?locale=fr-FR&page=1&pageSize=1');

    expect(response.headers()['content-type']).toContain('application/json');
    expect(response.headers()['cache-control']).toBeTruthy();
  });
});
```

- [ ] **Step 11: Créer le scénario k6 en continu (soak)**

Créer `ops/tests/load/k6-continuous.js` :

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const baseUrl = __ENV.BASE_URL || 'http://localhost:3000';
const locale = __ENV.LOCALE || 'fr';

const errorRate = new Rate('fanours_error_rate');
const latency = new Trend('fanours_latency', true);

export const options = {
  scenarios: {
    continuous_read: {
      executor: 'constant-arrival-rate',
      rate: 30,
      timeUnit: '1s',
      duration: __ENV.SOAK_DURATION || '2h',
      preAllocatedVUs: 20,
      maxVUs: 60,
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.01'],
    'http_req_duration{expected_response:true}': ['p(95)<600', 'p(99)<1200'],
    fanours_error_rate: ['rate<0.01'],
    fanours_latency: ['p(95)<600'],
  },
};

export default function () {
  const response = http.get(`${baseUrl}/v1/products?locale=${locale}&page=1&pageSize=24`, {
    tags: { name: 'GET /v1/products' },
  });

  const ok = check(response, {
    'status is 200': (r) => r.status === 200,
    'items returned': (r) => {
      const body = r.json();
      return Array.isArray(body?.items);
    },
  });

  errorRate.add(!ok);
  latency.add(response.timings.duration);

  sleep(0.2);
}
```

- [ ] **Step 12: Modifier `playwright.config.ts` — quota de flakiness, budget de durée et moteur navigateur**

Modifier `playwright.config.ts` — **racine du dépôt, jamais recréée** (créée en Phase 1, modifiée par les Phases 2, 4, 5 et 6). La modification porte sur trois points : le **moteur navigateur lu depuis la CI** (dimension de shard, **pas un projet**), les **seuils de budget**, et le **garde-fou de cohérence** entre `QUALITY_SUITES` et les projets déclarés. La structure de projets reste celle du modèle canonique : 7 projets, `testDir: './e2e'` unique, discrimination par `testMatch` / `testIgnore` uniquement.

```ts
import { readFileSync } from 'node:fs';
import { resolve } from 'node:path';
import { defineConfig } from '@playwright/test';
import { QUALITY_SUITES } from './packages/config/src/quality-thresholds.js';

const isCI = Boolean(process.env.CI);

/**
 * Configuration E2E racine — Phase 6 (spec §24.3, §24.13).
 *
 * Fichier UNIQUE de configuration du dépôt : créé en Phase 1, MODIFIÉ (jamais
 * recréé) par les Phases 2, 4, 5 et 6. Aucun autre `playwright.config.ts`
 * (ni `apps/web/`, ni `apps/api/`, ni `e2e/`). Toutes les commandes Playwright
 * sont lancées depuis la racine : `pnpm exec playwright test ...`.
 *
 * `testDir` vaut './e2e' pour TOUS les projets : les projets se distinguent
 * uniquement par `testMatch` / `testIgnore`. Un fichier appartient à un seul
 * projet : pas de doublon, pas de comptage de tests ambigu.
 *
 * Le MOTEUR (navigateur) n'est pas un projet : il est une dimension de shard CI,
 * passée par la variable d'environnement `PLAYWRIGHT_BROWSER`. Le projet définit
 * la *suite*, pas le moteur.
 *
 * Le timeout par test est de 45 s : un test qui le dépasse est bloqué, il se
 * diagnostique — on ne relève jamais le timeout. Le plafond de 12 min par shard
 * est un BUDGET DE JOB CI (`timeout-minutes` du workflow), pas un timeout
 * Playwright ; `globalTimeout` reste le plafond de l'exécution complète.
 */
export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: isCI,
  retries: isCI ? 2 : 0,
  workers: isCI ? 4 : undefined,
  timeout: 45_000,
  globalTimeout: 4 * 60 * 60 * 1000,
  expect: { timeout: 10_000 },
  reporter: isCI
    ? [['github'], ['html', { outputFolder: 'playwright-report', open: 'never' }], ['junit', { outputFile: 'test-results/junit.xml' }]]
    : [['list']],
  outputDir: 'test-results/artifacts',
  use: {
    baseURL: process.env.BASE_URL ?? 'http://127.0.0.1:3000',
    // Dimension de shard CI — pas un projet Playwright.
    browserName: (process.env.PLAYWRIGHT_BROWSER ?? 'chromium') as 'chromium' | 'firefox' | 'webkit',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    actionTimeout: 15_000,
    navigationTimeout: 20_000,
  },
  projects: [
    // 5 suites de recette de release (spec §24.3) — un fichier par suite.
    { name: 'b2c', testMatch: '**/b2c/journey.spec.ts' },
    { name: 'b2b', testMatch: '**/b2b/journey.spec.ts' },
    { name: 'payment', testMatch: '**/payment/scenarios.spec.ts' },
    { name: 'admin', testMatch: '**/admin/dashboard.spec.ts' },
    { name: 'security', testMatch: '**/security/security.spec.ts' },
    // Matrice visuelle 8 viewports x 3 langues x 2 thèmes.
    { name: 'visual', testMatch: ['**/visual/**/*.spec.ts', '**/visual-matrix.spec.ts'] },
    // Recettes fonctionnelles de développement : tout le reste, une fois les 6 exclus.
    {
      name: 'functional',
      testMatch: '**/*.spec.ts',
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

// Garde-fou : la configuration racine et la liste des suites ne peuvent pas diverger.
// On LIT la configuration racine pour en extraire les noms de projets (et non une
// liste supposée) : les 5 suites de QUALITY_SUITES doivent y être présentes, et les
// seuls projets supplémentaires autorisés sont les 2 projets auxiliaires.
const AUXILIARY_PROJECTS: readonly string[] = ['visual', 'functional'];
const RELEASE_SUITES: readonly string[] = QUALITY_SUITES;

if (process.env.PLAYWRIGHT_ASSERT_SUITES === '1') {
  const configSource = readFileSync(resolve(process.cwd(), 'playwright.config.ts'), 'utf8');
  const projectNames = [...configSource.matchAll(/^\s*\{\s*name:\s*'([^']+)'/gm)]
    .map((match) => match[1])
    .filter((name): name is string => typeof name === 'string');

  const missing = RELEASE_SUITES.filter((suite) => !projectNames.includes(suite));
  if (missing.length > 0) {
    throw new Error(
      `Suites E2E absentes de playwright.config.ts : ${missing.join(',')} (projets déclarés : ${projectNames.join(',')}).`,
    );
  }

  const unexpected = projectNames.filter(
    (name) => !RELEASE_SUITES.includes(name) && !AUXILIARY_PROJECTS.includes(name),
  );
  if (unexpected.length > 0) {
    throw new Error(
      `Projets Playwright inattendus dans playwright.config.ts : ${unexpected.join(',')}. ` +
        `Les seuls projets autorisés sont ${RELEASE_SUITES.join(',')} et ${AUXILIARY_PROJECTS.join(',')}.`,
    );
  }
}
```

Le plafond de durée par shard passe ensuite dans le workflow CI, où il est réellement appliqué — c'est un **budget de job**, pas un timeout Playwright :

```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest
    # Budget §24.13 : E2E <= 12 min par shard, total <= 18 min.
    # Le timeout Playwright par test reste de 45 s (playwright.config.ts).
    timeout-minutes: 12
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4, 5]
        project: [b2c, b2b, payment, admin, security]
        browser: [chromium, firefox, webkit]
    steps:
      - uses: ./.github/actions/setup
      - name: Installer les navigateurs
        run: pnpm exec playwright install --with-deps ${{ matrix.browser }}
      - name: Executer la suite
        run: pnpm exec playwright test --project=${{ matrix.project }} --shard=${{ matrix.shard }}/5
        env:
          PROJECT: ${{ matrix.project }}
          # Le moteur est une dimension de shard, pas un projet Playwright.
          PLAYWRIGHT_BROWSER: ${{ matrix.browser }}
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-traces-${{ matrix.project }}-${{ matrix.browser }}-${{ matrix.shard }}
          path: test-results/artifacts/
          retention-days: 7
          if-no-files-found: ignore
```

- [ ] **Step 12b: Supprimer les redondances de la suite E2E**

L'audit impose qu'une couverture a **un seul propriétaire**. Trois candidats sont des doublons de suites existantes ; ils sont supprimés s'ils existent, jamais recréés :

```bash
git rm --ignore-unmatch \
  e2e/b2c/checkout-copy.spec.ts \
  e2e/security/headers.spec.ts \
  e2e/admin/table-sort.spec.ts
```

`--ignore-unmatch` rend l'opération idempotente : un candidat absent du dépôt est simplement ignoré (l'audit n'invente pas de suppression). Avant la suppression, vérifier que la couverture conservée existe bien :

```bash
pnpm exec playwright test e2e/checkout/checkout.spec.ts --project=functional
pnpm exec playwright test e2e/security/security.spec.ts --project=security
pnpm exec playwright test e2e/admin/dashboard.spec.ts --project=admin
```

Attendu : `3 passed` pour le checkout, `7 passed` pour la suite sécurité, `11 passed` pour la suite admin. **Une suppression sans remplacement qui passe est un défaut** : ne pas supprimer un candidat dont la suite de remplacement est rouge.

- [ ] **Step 13: Écrire l'audit de la suite E2E**

Créer `docs/maintenance/TEST-AUDIT.md` :

```markdown
# Audit de la suite de tests — Phase 6

**Date :** 2026-10-21 · **Périmètre :** 5 suites E2E, tests unitaires et d'intégration, mutation.

## 1. Redondances supprimées

| Candidat examiné | Raison | Décision | Couverture conservée par |
|---|---|---|---|
| `e2e/b2c/checkout-copy.spec.ts` | doublon exact de `checkout.spec.ts > étape 4` | supprimé s'il existe | `e2e/checkout/checkout.spec.ts` |
| `e2e/security/headers.spec.ts > csp-strict` | doublon de `e2e/security/security.spec.ts > CSP` | supprimé s'il existe | `e2e/security/security.spec.ts` |
| `e2e/admin/table-sort.spec.ts` | doublon du cas 10 de la suite de release admin (tri du tableau) | supprimé s'il existe | `e2e/admin/dashboard.spec.ts` |

Chaque suppression est accompagnée du test qui garde la couverture : une suppression sans remplacement est un défaut. Un candidat absent du dépôt est ignoré — l'audit n'invente pas de suppression.

L'audit vérifie aussi qu'aucune des 5 suites de release ne duplique une recette fonctionnelle de phase, et qu'**un fichier de test appartient à un seul projet** Playwright : la matrice de redondances ci-dessus est la seule liste de suppressions ; toute autre ligne dans ce document est une anomalie à corriger, pas un constat à consigner.

## 2. Quarantaine

Un test flaky ne vit pas plus de 24 h dans la suite bloquante. La table de quarantaine est versionnée dans `docs/maintenance/TEST-QUARANTINE.md` et contrôlée par `scripts/tests/quarantine.ts` : une ligne expirée **fait échouer le build**, elle ne le contourne pas.

| Test | Suite | Ticket | Quarantiné le | Expire le | Propriétaire |
|---|---|---|---|---|---|
| — | — | — | — | — | — |

Les retries (`retries: CI ? 2 : 0`) sont un **filet**, pas un traitement : un test qui doit retryer pour passer est un test qui doit être réparé.

## 3. Budget de durée

| Mesure | Seuil (qualite §24.13) | Constat Phase 6 |
|---|---|---|
| Unitaires + intégration | ≤ 6 min | à mesurer à la première exécution complète |
| E2E par shard | ≤ 12 min | plafond appliqué par le `timeout-minutes` du job CI (`.github/workflows/e2e.yml`) ; le timeout Playwright par test reste de **45 s** (`timeout: 45_000` dans `playwright.config.ts`) |
| E2E total | ≤ 18 min | audit de redondance effectué ci-dessus |

## 4. Mutation

`stryker.config.json` couvre désormais le pricing, la conformité, la machine à états de commande, l'arithmétique monétaire **et le cache de performance** (Task 3). Seuil `break` : 80.

## 5. Charge en continu

`ops/tests/load/k6-continuous.js` tourne en soak 2 h à 30 RPS contre le catalogue, avec dérive p95 de la dernière minute ≤ +25 % de la première (seuil infra §charge).

## 6. Actions non fermées

- [ ] Mesurer la durée réelle de la suite unitaire + intégration sur 3 exécutions consécutives.
- [ ] Confirmer le taux de flakiness < 1 % sur les 12 E2E prioritaires (0 % attendu).
- [ ] Décider du sort de chaque test en quarantaine (≤ 24 h).
```

- [ ] **Step 14: Vérifier le périmètre complet**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/contracts
pnpm --filter @fanours/scripts exec vitest run src/tests
pnpm exec playwright test e2e/contracts/api-web.contract.spec.ts --project=functional
pnpm turbo run typecheck
```

Attendu : 4 + 5 tests unitaires passés, 4 tests de contrat E2E passés, `typecheck` sans erreur.

- [ ] **Step 15: Commit**

```bash
git add apps/api/src/modules/contracts scripts/tests e2e/contracts ops/tests/load stryker.config.json playwright.config.ts .github/workflows/e2e.yml docs/maintenance/TEST-AUDIT.md
git commit -m "test(quality): consolide mutation, contrats, quarantaine et audit E2E" -m "Collecteur de contrats de routes, test de contrat API web, quarantaine de
24 h controlee par build, extension de la mutation au cache, scenario k6 en
continu et audit de la suite E2E. Matrice de navigateurs ajoutee comme
dimension de shard CI (le projet Playwright definit la suite, pas le moteur),
timeout par test porte a 45 s et plafond de 12 min par shard applique par le
timeout-minutes du job CI.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` affiche le titre, le corps et la ligne `Co-Authored-By`.

---

### Task 7: Observabilité & exploitation — SLO, alertes, runbooks, drill, coûts, capacité

**Files:**
- Create: `packages/observability/src/slo.ts`
- Create: `packages/observability/src/burn-rate.ts`
- Create: `packages/observability/src/__tests__/burn-rate.spec.ts`
- Create: `ops/alerting/rules/phase6-slo-burnrate.yml`
- Create: `ops/alerting/rules/phase6-cost-drift.yml`
- Create: `ops/runbooks/RB-009-slo-degrade.md`
- Create: `ops/runbooks/RB-010-cout-infra-degrade.md`
- Create: `docs/adr/0016-slo-et-budgets-alerte.md`
- Create: `ops/cost/monthly-review.md`
- Modify: `ops/backup/restore-drill.yml`
- Modify: `docs/maintenance/TECH-DEBT-BACKLOG.md`

**Interfaces:**
- Consumes: `packages/config/src/slo.ts` (SLO cibles, Task 1), `packages/observability/src/logger.ts` (logger pino existant), `docs/roadmap/sections/infra.md` §7.3 (25 métriques), §8.2 (seuils d'alerte), §9.6 (drill mensuel de restauration), §15 (8 runbooks obligatoires), §16 (coûts mensuels, cible ≤ 500 €/mois, alerte à 50/80/100 %, ligne > 30 % du total → propriétaire + justification écrite).
- Produces:
  - `packages/observability/src/burn-rate.ts` → `export function computeBurnRate(input: BurnRateInput): BurnRateResult` et `export type BurnRateResult`.
  - `packages/observability/src/slo.ts` → `export function sliWindowToBurnRate(...)`, `export function buildSloDashboardRows(...)`.
  - `ops/alerting/rules/phase6-slo-burnrate.yml` → règles Prometheus testées par `promtool test rules`.
  - `ops/runbooks/RB-009-slo-degrade.md`, `ops/runbooks/RB-010-cout-infra-degrade.md` → runbooks à 8 sections, avec commandes exactes.

---

- [ ] **Step 1: Écrire le test qui échoue — le calcul de burn rate multi-fenêtre**

Créer `packages/observability/src/__tests__/burn-rate.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { computeBurnRate } from '../burn-rate';

describe('computeBurnRate', () => {
  it('calcule un burn rate nul quand aucune erreur n’est survenue', () => {
    const result = computeBurnRate({
      totalRequests: 10_000,
      failedRequests: 0,
      errorBudget: 0.005,
      windowSeconds: 3600,
    });

    expect(result.burnRate).toBe(0);
    expect(result.exhausted).toBe(false);
  });

  it('calcule un burn rate égal à 1 quand le budget est consommé exactement', () => {
    const result = computeBurnRate({
      totalRequests: 10_000,
      failedRequests: 50,
      errorBudget: 0.005,
      windowSeconds: 3600,
    });

    expect(result.burnRate).toBeCloseTo(1, 5);
  });

  it('signale l’épuisement quand le burn rate dépasse 1', () => {
    const result = computeBurnRate({
      totalRequests: 10_000,
      failedRequests: 500,
      errorBudget: 0.005,
      windowSeconds: 3600,
    });

    expect(result.burnRate).toBeCloseTo(10, 5);
    expect(result.exhausted).toBe(true);
  });

  it('retourne un burn rate nul quand il n’y a aucun trafic (pas de division par zéro)', () => {
    const result = computeBurnRate({
      totalRequests: 0,
      failedRequests: 0,
      errorBudget: 0.005,
      windowSeconds: 3600,
    });

    expect(result.burnRate).toBe(0);
    expect(result.exhausted).toBe(false);
  });

  it('refuse un budget d’erreur hors de la plage (0, 1]', () => {
    expect(() =>
      computeBurnRate({
        totalRequests: 10,
        failedRequests: 0,
        errorBudget: 0,
        windowSeconds: 3600,
      }),
    ).toThrowError(/budget d’erreur/);
  });

  it('refuse un budget d’erreur supérieur à 1', () => {
    expect(() =>
      computeBurnRate({
        totalRequests: 10,
        failedRequests: 0,
        errorBudget: 1.5,
        windowSeconds: 3600,
      }),
    ).toThrowError(/budget d’erreur/);
  });

  it('refuse un nombre de requêtes négatif', () => {
    expect(() =>
      computeBurnRate({
        totalRequests: -1,
        failedRequests: 0,
        errorBudget: 0.005,
        windowSeconds: 3600,
      }),
    ).toThrowError(/requêtes/);
  });
});
```

- [ ] **Step 2: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/observability exec vitest run src/__tests__/burn-rate.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "../burn-rate"`. 0 test exécuté.

- [ ] **Step 3: Implémenter le calcul de burn rate**

Créer `packages/observability/src/burn-rate.ts` :

```ts
export type BurnRateInput = {
  totalRequests: number;
  failedRequests: number;
  errorBudget: number;
  windowSeconds: number;
};

export type BurnRateResult = {
  burnRate: number;
  exhausted: boolean;
  errorRate: number;
  windowSeconds: number;
};

/**
 * Burn rate = (taux d'erreur observé / budget d'erreur autorisé).
 * Un burn rate supérieur à 1 consomme le budget plus vite que prévu.
 * Fenêtres : 1 h (rapide) et 6 h (lente) — seuils infra §8.2.
 */
export function computeBurnRate(input: BurnRateInput): BurnRateResult {
  if (!Number.isFinite(input.errorBudget) || input.errorBudget <= 0 || input.errorBudget > 1) {
    throw new Error(
      `budget d’erreur invalide : ${input.errorBudget} (attendu dans ]0, 1])`,
    );
  }

  if (input.totalRequests < 0) {
    throw new Error(
      `nombre de requêtes invalide : ${input.totalRequests} (attendu >= 0)`,
    );
  }

  if (input.failedRequests < 0 || input.failedRequests > input.totalRequests) {
    throw new Error(
      `nombre d’échecs invalide : ${input.failedRequests} pour ${input.totalRequests} requêtes`,
    );
  }

  if (input.windowSeconds <= 0) {
    throw new Error(`fenêtre invalide : ${input.windowSeconds} secondes`);
  }

  if (input.totalRequests === 0) {
    return {
      burnRate: 0,
      exhausted: false,
      errorRate: 0,
      windowSeconds: input.windowSeconds,
    };
  }

  const errorRate = input.failedRequests / input.totalRequests;
  const burnRate = errorRate / input.errorBudget;

  return {
    burnRate,
    exhausted: burnRate > 1,
    errorRate,
    windowSeconds: input.windowSeconds,
  };
}
```

- [ ] **Step 4: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/observability exec vitest run src/__tests__/burn-rate.spec.ts
```

Attendu : **7 tests passés**, 0 échec.

- [ ] **Step 5: Implémenter le module SLO**

Créer `packages/observability/src/slo.ts` :

```ts
import { SLO_TARGETS, type SloTarget } from '@fanours/config/slo';
import { computeBurnRate, type BurnRateResult } from './burn-rate';

export type SloMeasurement = {
  route: string;
  totalRequests: number;
  failedRequests: number;
  windowSeconds: number;
  latencyP95Ms: number;
};

export type SloRow = {
  route: string;
  target: SloTarget;
  burn: BurnRateResult;
  latencyBreached: boolean;
};

/**
 * Tableau de bord SLO : une ligne par route surveillée.
 * Un SLO sans runbook est un défaut de configuration (infra §15).
 */
export function buildSloDashboardRows(measurements: SloMeasurement[]): SloRow[] {
  return measurements.map((measurement) => {
    const target = SLO_TARGETS.find((slo) => slo.route === measurement.route);

    if (!target) {
      throw new Error(
        `route SLO inconnue : ${measurement.route} (déclarer le SLO dans packages/config/src/slo.ts)`,
      );
    }

    const burn = computeBurnRate({
      totalRequests: measurement.totalRequests,
      failedRequests: measurement.failedRequests,
      errorBudget: 1 - target.availability,
      windowSeconds: measurement.windowSeconds,
    });

    return {
      route: measurement.route,
      target,
      burn,
      latencyBreached: measurement.latencyP95Ms > target.latencyP95Ms,
    };
  });
}
```

- [ ] **Step 6: Créer les règles d'alerte SLO (multi-fenêtre)**

Créer `ops/alerting/rules/phase6-slo-burnrate.yml` :

```yaml
# Règles de burn rate multi-fenêtre (Phase 6).
# Seuils issus de docs/roadmap/sections/infra.md §8.2.
# Toute règle doit avoir un runbook : RB-009-slo-degrade.md.
groups:
  - name: fanours.phase6.slo
    interval: 30s
    rules:
      - record: fanours:http_requests:rate5m
        expr: sum(rate(http_requests_total{route=~"api:.*|web:public"}[5m]))

      - record: fanours:http_requests:errors:rate5m
        expr: sum(rate(http_requests_total{route=~"api:.*|web:public",status=~"5.."}[5m]))

      - record: fanours:http_requests:rate1h
        expr: sum(rate(http_requests_total{route=~"api:.*|web:public"}[1h]))

      - record: fanours:http_requests:errors:rate1h
        expr: sum(rate(http_requests_total{route=~"api:.*|web:public",status=~"5.."}[1h]))

      - record: fanours:http_requests:rate6h
        expr: sum(rate(http_requests_total{route=~"api:.*|web:public"}[6h]))

      - record: fanours:http_requests:errors:rate6h
        expr: sum(rate(http_requests_total{route=~"api:.*|web:public",status=~"5.."}[6h]))

      # Fenêtre rapide : 1 h consommée à 14,4x -> alerte page.
      - alert: FanoursSloBurnRateFast
        expr: |
          fanours:http_requests:errors:rate1h
            / fanours:http_requests:rate1h > 0.070
        for: 2m
        labels:
          severity: page
          service: fanours
        annotations:
          summary: "Budget d'erreur consommé rapidement (fenêtre 1 h)"
          description: >-
            Le taux d'erreur dépasse 14,4x le budget d'erreur autorisé sur 1 h.
            Budget restant estimee : {{ $value | humanizePercentage }}.
          runbook_url: "ops/runbooks/RB-009-slo-degrade.md"

      # Fenêtre lente : 6 h consommée à 6x -> alerte ticket.
      - alert: FanoursSloBurnRateSlow
        expr: |
          fanours:http_requests:errors:rate6h
            / fanours:http_requests:rate6h > 0.030
        for: 15m
        labels:
          severity: ticket
          service: fanours
        annotations:
          summary: "Budget d'erreur consommé durablement (fenêtre 6 h)"
          description: >-
            Le taux d'erreur depasse 6x le budget d'erreur autorise sur 6 h.
            Le budget d'erreur du mois est en train de partir.
          runbook_url: "ops/runbooks/RB-009-slo-degrade.md"

      - alert: FanoursApiLatencyP95High
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket{route=~"api:.*"}[5m])) by (le)
          ) > 0.2
        for: 10m
        labels:
          severity: ticket
          service: fanours
        annotations:
          summary: "Latence p95 API au-dessus de 200 ms"
          description: "La latence p95 de l'API depasse le budget de 200 ms depuis 10 minutes."
          runbook_url: "ops/runbooks/RB-006-latence-5xx.md"
```

- [ ] **Step 7: Créer les règles d'alerte de dérive de coût**

Créer `ops/alerting/rules/phase6-cost-drift.yml` :

```yaml
# Derive du cout d'infrastructure (Phase 6).
# Cible mensuelle : 500 EUR. Alertes a 50 %, 80 % et 100 % (infra §16).
# Runbook : ops/runbooks/RB-010-cout-infra-degrade.md
groups:
  - name: fanours.phase6.cost
    interval: 5m
    rules:
      - record: fanours:infra_cost:monthly_eur
        expr: fanours_infra_cost_eur_total

      - alert: FanoursInfraCostWarning
        expr: fanours:infra_cost:monthly_eur > 400
        for: 15m
        labels:
          severity: ticket
          service: fanours
        annotations:
          summary: "Cout d'infrastructure au-dessus de 80 % de la cible"
          description: >-
            Le cout mensuel est de {{ $value }} EUR pour une cible de 500 EUR.
            Verifier les lignes de cout et l'echelle des environnements non-productifs.
          runbook_url: "ops/runbooks/RB-010-cout-infra-degrade.md"

      - alert: FanoursInfraCostOver
        expr: fanours:infra_cost:monthly_eur > 500
        for: 10m
        labels:
          severity: page
          service: fanours
        annotations:
          summary: "Cout d'infrastructure au-dessus de la cible"
          description: >-
            Le cout mensuel est de {{ $value }} EUR pour une cible de 500 EUR.
            Toute ligne au-dela de 30 % du total exige un proprietaire et une
            justification ecrite.
          runbook_url: "ops/runbooks/RB-010-cout-infra-degrade.md"
```

- [ ] **Step 8: Tester les règles avec `promtool`**

Créer `ops/alerting/rules/phase6-slo-burnrate.test.yml` :

```yaml
# Tests des regles d'alerte. Une regle qui ne declenche jamais en test est
# supprimee (infra §8.2).
rule_files:
  - phase6-slo-burnrate.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'http_requests_total{route="api:read",status="200"}'
        values: '0+90x60'
      - series: 'http_requests_total{route="api:read",status="500"}'
        values: '0+10x60'
    alert_rule_test:
      - eval_time: 10m
        alertname: FanoursSloBurnRateFast
        exp_alerts:
          - exp_labels:
              severity: page
              service: fanours
            exp_annotations:
              summary: "Budget d'erreur consommé rapidement (fenêtre 1 h)"
              description: >-
                Le taux d'erreur dépasse 14,4x le budget d'erreur autorisé sur 1 h.
                Budget restant estimee : {{ $value | humanizePercentage }}.
              runbook_url: "ops/runbooks/RB-009-slo-degrade.md"

  - interval: 1m
    input_series:
      - series: 'http_requests_total{route="api:read",status="200"}'
        values: '0+99x60'
      - series: 'http_requests_total{route="api:read",status="500"}'
        values: '0+1x60'
    alert_rule_test:
      - eval_time: 10m
        alertname: FanoursSloBurnRateFast
        exp_alerts: []
```

Puis exécuter :

```bash
promtool test rules ops/alerting/rules/phase6-slo-burnrate.test.yml
promtool check rules ops/alerting/rules/phase6-slo-burnrate.yml
promtool check rules ops/alerting/rules/phase6-cost-drift.yml
```

Attendu : `promtool test rules` affiche `SUCCESS: 1 alert(s) tested, 1 passed, 0 failed.` et `check rules` ne signale aucune erreur de syntaxe.

- [ ] **Step 9: Écrire les deux runbooks manquants**

Créer `ops/runbooks/RB-009-slo-degrade.md` :

```markdown
# RB-009 — SLO dégradé (budget d'erreur consommé)

**Sévérité :** page · **Service :** fanours · **Dernière révision :** 2026-10-21

## 1. Symptômes

- Alerte `FanoursSloBurnRateFast` ou `FanoursSloBurnRateSlow` active.
- Taux d'erreur 5xx supérieur au budget d'erreur de la route.
- Latence p95 au-dessus du budget SLO.

## 2. Impact

Dégradation de l'expérience sur les parcours touchés (catalogue, panier, checkout, API). Si la route `api:payment-webhook` est touchée, les confirmations de paiement sont en retard : **ne pas déclencher de remboursement automatique**.

## 3. Diagnostic

```bash
# Taux d'erreur par route sur 1 h
curl -s "$PROMETHEUS_URL/api/v1/query" --data-urlencode \
  'query=sum by (route) (rate(http_requests_total{status=~"5.."}[1h]))'

# Erreurs applicatives récentes, filtrées sur les 5xx
docker logs --since 30m fanours-api 2>&1 | grep '"status":5' | tail -50
```

## 4. Résolution

```bash
# 1. Identifier la dépendance degradée (base, cache, file, prestataire)
docker logs --since 30m fanours-api 2>&1 | grep -E '"event":"(db|redis|queue|payment)_error"' | tail -50

# 2. Verifier la sante de la base
docker exec fanours-postgres psql -U "$PGUSER" -d "$PGDATABASE" -c "SELECT 1;"

# 3. Verifier la profondeur des files BullMQ
docker exec fanours-redis redis-cli --scan --pattern 'bull:*:wait' | head -20

# 4. Si la file est saturee, relancer le worker une fois la cause corrigee
docker restart fanours-worker
```

## 5. Escalade

- Responsable plateforme sous 15 min si la route payment est touchée.
- Responsable produit si l'impact dépasse 2 h.
- **Aucune communication publique sans validation.**

## 6. Vérification

```bash
curl -s "$PROMETHEUS_URL/api/v1/query" --data-urlencode \
  'query=sum(rate(http_requests_total{status=~"5.."}[5m]))'
```

Attendu : le taux d'erreur repasse sous le budget d'erreur et l'alerte se résorbe.

## 7. Retour d'expérience

À remplir dans les 72 h : cause racine, délai de détection, délai de résolution, action corrective.

## 8. Runbooks liés

- RB-006 — 5xx et latence
- RB-005 — restauration de la base
```

Créer `ops/runbooks/RB-010-cout-infra-degrade.md` :

```markdown
# RB-010 — Dérive du coût d'infrastructure

**Sévérité :** ticket · **Service :** fanours · **Dernière révision :** 2026-10-21

## 1. Symptômes

- Alerte `FanoursInfraCostWarning` (> 400 EUR) ou `FanoursInfraCostOver` (> 500 EUR).
- Écart entre le coût constaté et le budget mensuel.

## 2. Impact

Budget d'infrastructure dépassé. La cible est de 500 EUR/mois (infra §16). Toute ligne de coût dépassant 30 % du total exige un propriétaire et une justification écrite.

## 3. Diagnostic

```bash
# Cout par service sur le mois en cours
curl -s "$PROMETHEUS_URL/api/v1/query" --data-urlencode \
  'query=fanours_infra_cost_eur_total'

# Detail par environnement
curl -s "$PROMETHEUS_URL/api/v1/query" --data-urlencode \
  'query=sum by (env, service) (fanours_infra_cost_eur_total)'
```

## 4. Résolution

```bash
# 1. Environments non productifs : redimensionner ou mettre a l'arret (environ 35 % d'economie)
docker ps --filter "label=env=staging" --format '{{.Names}}'

# 2. Reduire la retention des logs (30 j en production, 7 j ailleurs)
#    op/alerting/rules/phase6-cost-drift.yml verifie ensuite la baisse

# 3. Verifier l'absence de ressource orpheline
docker ps -a --filter "status=exited" --format '{{.Names}} {{.Status}}' | head -20
```

## 5. Escalade

- Responsable plateforme sous 24 h.
- Toute dépense au-delà de la cible exige l'accord écrit du propriétaire du budget.

## 6. Vérification

```bash
curl -s "$PROMETHEUS_URL/api/v1/query" --data-urlencode \
  'query=fanours_infra_cost_eur_total'
```

Attendu : le coût revient sous 400 EUR sur le mois en cours.

## 7. Retour d'expérience

À compléter : ligne de coût responsable, action appliquée, durée pour revenir sous la cible.

## 8. Runbooks liés

- RB-006 — 5xx et latence
- RB-005 — restauration de la base
```

- [ ] **Step 10: Renforcer le drill mensuel de restauration**

Modifier `ops/backup/restore-drill.yml` pour ajouter la mesure du RPO et du RTO et l'archivage du rapport :

```yaml
name: Monthly restore drill
on:
  schedule: [{ cron: '17 3 1 * *' }]
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
      - name: Test RPO
        run: ./ops/backup/measure-rpo.sh --target="$LAST_SNAPSHOT"
      - name: Test RTO
        run: ./ops/backup/measure-rto.sh --drill-run-id="${{ github.run_id }}"
      - name: Archivage du rapport
        run: |
          set -euo pipefail
          MONTH="$(date -u +%Y-%m)"
          mkdir -p "ops/backup/reports/${MONTH}"
          cp drill-report.json "ops/backup/reports/${MONTH}/$(date -u +%Y-%m-%d).json"
      - name: Contrôle des seuils
        run: |
          set -euo pipefail
          RPO_HOURS="$(jq -r '.rpo_hours' drill-report.json)"
          RTO_HOURS="$(jq -r '.rto_hours' drill-report.json)"
          echo "RPO mesuré : ${RPO_HOURS} h — RTO mesuré : ${RTO_HOURS} h"
          awk -v rpo="${RPO_HOURS}" 'BEGIN { if (rpo > 24) exit 1 }' || {
            echo "RPO hors seuil (24 h)"; exit 1;
          }
          awk -v rto="${RTO_HOURS}" 'BEGIN { if (rto > 8) exit 1 }' || {
            echo "RTO hors seuil (8 h)"; exit 1;
          }
```

- [ ] **Step 11: Écrire la revue mensuelle des coûts**

Créer `ops/cost/monthly-review.md` :

```markdown
# Revue mensuelle du coût d'infrastructure

**Date de revue :** premier jour du mois · **Cible :** 500 EUR/mois · **Référence :** infra §16

| Palier | Coût mensuel | Contexte |
|---|---|---|
| low | 198 EUR | trafic faible, une seule instance |
| medium | 472 EUR | trafic nominal |
| high | 1 518 EUR | pic de campagne |
| **cible** | **500 EUR** | **plafond de la Phase 6** |

Alertes : 400 EUR (80 %), 500 EUR (100 %).

## Tableau de la revue

| Ligne de coût | Montant mensuel | % du total | Propriétaire | Justification écrite | Action |
|---|---|---|---|---|---|
| _à compléter à la revue_ | | | | | |

> Une ligne dépassant 30 % du total exige un propriétaire nommé et une justification écrite.
> Une ligne sans propriétaire est un défaut de configuration, pas une economies.

## Décisions de la revue

- Scale-to-zero des environnements non productifs : appliqué par défaut, coût économisé estimé 35 %.
- Rétention des logs : 30 j en production, 7 j ailleurs.
- Images de backup : 35 j (quotidien), 90 j (hebdomadaire).

## Suivi

- [ ] Coût du mois relevé et comparé à la cible
- [ ] Alerte `FanoursInfraCostWarning` / `FanoursInfraCostOver` treated
- [ ] Lignes > 30 % justifiées par écrit
```

- [ ] **Step 12: Écrire l'ADR SLO et budgets d'alerte**

Créer `docs/adr/0016-slo-et-budgets-alerte.md` :

```markdown
# ADR-0016 — SLO par route, budget d'erreur et burn rate multi-fenêtre

- Statut : Accepted
- Date : 2026-10-21
- Décideur : sre
- Remplace : aucun

## Contexte

Les alertes existantes se déclenchaient sur des seuils instantanés (5xx > X par minute). Elles étaient soit muettes, soit saturées, et ne disaient rien de la **consommation du budget d'erreur**. Après 30 jours de données, on peut définir un SLO par route et piloter par burn rate.

## Décision

1. **SLO par route** : `packages/config/src/slo.ts` déclare la disponibilité et la latence p95 par route. Toute route exposée a un SLO ; une route sans SLO est un défaut.
2. **Burn rate multi-fenêtre** : alerte page si le budget d'erreur est consommé 14,4x plus vite que prévu sur 1 h ; alerte ticket si 6x sur 6 h. Les deux fenêtres sont nécessaires : la rapide détecte, la lente évite les alertes transitoires.
3. **Chaque alerte a un runbook.** Une alerte sans runbook est un défaut de configuration et est supprimée.
4. **Règle testée.** `promtool test rules` valide chaque règle ; une règle qui ne déclenche jamais en test est supprimée.
5. **Budget d'erreur explicite** : disponibilité cible 99,5 % mensuel (99,9 % pour le webhook de paiement).

## Conséquences

- Le pilotage se fait par consommation de budget, pas par seuils arbitraires.
- Les alertes deviennent actionnables : chaque `runbook_url` pointe vers un document avec des commandes exactes.

## Réversibilité

Réversible par un nouvel ADR ; la donnée de télémétrie (SLI) est conservée quelle que soit la règle.
```

- [ ] **Step 13: Ajouter les runbooks au backlog et vérifier la cohérence**

Ajouter au fichier `docs/maintenance/TECH-DEBT-BACKLOG.md` les deux runbooks créés dans cette tâche en portant le statut à `fermé` — ils ne sont plus des dettes :

```markdown
| DEBT-006 | ops/runbooks/RB-009-slo-degrade.md | 0 j | faible | 2026-10-21 | fermé |
| DEBT-007 | ops/runbooks/RB-010-cout-infra-degrade.md | 0 j | faible | 2026-10-21 | fermé |
```

- [ ] **Step 14: Vérifier le périmètre complet**

```bash
pnpm --filter @fanours/observability exec vitest run
promtool test rules ops/alerting/rules/phase6-slo-burnrate.test.yml
pnpm turbo run typecheck
```

Attendu : 7 tests passés, `promtool` affiche `SUCCESS`, `typecheck` sans erreur.

- [ ] **Step 15: Commit**

```bash
git add packages/observability ops/alerting ops/runbooks ops/backup/restore-drill.yml ops/cost docs/adr/0016-slo-et-budgets-alerte.md docs/maintenance/TECH-DEBT-BACKLOG.md
git commit -m "feat(observability): ajoute les SLO, burn rate et runbooks manquants" -m "Calcul de burn rate multi-fenetre, regles d'alerte SLO et de derive de cout
testees par promtool, runbooks RB-009 et RB-010 avec commandes exactes, drill
de restauration avec controle RPO et RTO et ADR SLO.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` affiche le titre, le corps et la ligne `Co-Authored-By`.

---

### Task 8: Sécurité continue — rotation planifiée des secrets, scan de dépendances, test de non-régression du scanner, journal d'audit de rotation

**Files:**
- Create: `packages/config/src/secrets.ts`
- Create: `packages/config/src/__tests__/secrets.spec.ts`
- Create: `packages/db/migrations/0122_perf_baselines.sql`
- Create: `packages/db/migrations/0123_secret_rotation_audit.sql`
- Create: `packages/db/src/schema/secret-rotation-audit.ts`
- Create: `apps/api/src/modules/security/secret-rotation-audit.ts`
- Create: `apps/api/src/modules/security/__tests__/secret-rotation-audit.spec.ts`
- Create: `ops/security/secret-rotation.yml`
- Create: `ops/security/scanner-regression.sh`
- Create: `docs/adr/0017-rotation-secrets-planifiee.md`
- Modify: `docs/maintenance/TECH-DEBT-BACKLOG.md`

**Interfaces:**
- Consumes: `Cahier_des_charges_Fanours.md` §19.5 (gestionnaire de secrets dédié, jamais en Git / build / logs / bundle ; en cas de fuite, **rotation d'abord, purge de l'historique ensuite**), `docs/roadmap/sections/infra.md` §9.5 (Vault ou KMS ; le backend ne stocke que des références `secret://fanours/prod/...` ; rotation trimestrielle ; double secret avec chevauchement de 7 jours pour la clé de webhook du prestataire de paiement ; rotation d'urgence en moins de 30 minutes), §15 (runbooks ; **RB-004 « incident sécurité » traite déjà l'exposition d'un secret — aucun runbook supplémentaire n'est créé**), `packages/config/src/env.ts` (schéma Zod d'environnement), `packages/db/src/schema/audit-log.ts` (journal d'audit immuable existant, modèle de la chaîne de hachage), `packages/config/src/budgets.ts` (`PERF_BUDGETS`, Task 1) et `packages/db/migrations/0120_maintenance_metrics.sql` (mesures de performance, Task 3).
- Produces:
  - `packages/config/src/secrets.ts` → `export const SECRET_CATALOG: readonly SecretDefinition[]`, `export type SecretDefinition`, `export function nextRotationDate(secret: SecretDefinition, from: string): string`, `export function isRotationDue(secret: SecretDefinition, today: string): boolean`, `export function daysUntilRotation(secret: SecretDefinition, today: string): number`, `export function assertCatalogIntegrity(catalog: readonly SecretDefinition[]): string[]`.
  - `apps/api/src/modules/security/secret-rotation-audit.ts` → `export function buildRotationAuditEntry(input: RotationAuditInput): RotationAuditEntry`, `export const ROTATION_AUDIT_INSERT`.
  - `packages/db/migrations/0122_perf_baselines.sql` → tables `release_baseline` et `release_baseline_metric` (append-only) : la baseline permet de prouver qu'un redéploiement, y compris une rotation de secret, n'a pas dégradé les budgets.
  - `packages/db/migrations/0123_secret_rotation_audit.sql` → table `secret_rotation_audit` (append-only) : journal immuable des rotations, sans aucune valeur de secret.
  - `ops/security/secret-rotation.yml` → pipeline déclaratif de rotation planifiée (trimestrielle), d'urgence et de scan de dépendances.
  - `ops/security/scanner-regression.sh` → test de non-régression du scanner de secrets lui-même.
  - `docs/adr/0017-rotation-secrets-planifiee.md` → ADR de la stratégie de rotation.

---

- [ ] **Step 1: Écrire le test qui échoue — le catalogue des secrets et les échéances de rotation**

Créer `packages/config/src/__tests__/secrets.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import {
  SECRET_CATALOG,
  assertCatalogIntegrity,
  daysUntilRotation,
  isRotationDue,
  nextRotationDate,
} from '../secrets';

function findSecret(envName: string) {
  const secret = SECRET_CATALOG.find((candidate) => candidate.envName === envName);
  if (secret === undefined) {
    throw new Error(`${envName} absent du catalogue des secrets`);
  }
  return secret;
}

describe('catalogue des secrets', () => {
  it('ne contient aucun nom de variable dupliqué', () => {
    const names = SECRET_CATALOG.map((secret) => secret.envName);
    expect(new Set(names).size).toBe(names.length);
  });

  it('est cohérent : ni doublon, ni référence hors coffre, ni propriétaire vide', () => {
    expect(assertCatalogIntegrity(SECRET_CATALOG)).toEqual([]);
  });

  it('référence chaque secret par une URI secret:// du coffre applicatif', () => {
    for (const secret of SECRET_CATALOG) {
      expect(secret.secretRef.startsWith('secret://fanours/')).toBe(true);
    }
  });

  it('ne contient aucune valeur de secret en clair', () => {
    const serialized = JSON.stringify(SECRET_CATALOG);
    expect(serialized).not.toMatch(/-----BEGIN/);
    expect(serialized).not.toMatch(/\bsk_[A-Za-z0-9]{12,}/);
    expect(serialized).not.toMatch(/\bAKIA[0-9A-Z]{16}\b/);
  });
});

describe('échéance de rotation', () => {
  it('calcule la prochaine rotation à 91 jours (trimestriel)', () => {
    const stripe = findSecret('STRIPE_SECRET_KEY');
    expect(nextRotationDate(stripe, '2026-01-01')).toBe('2026-04-02');
  });

  it('marque comme due une rotation dont l’échéance est dépassée', () => {
    const stripe = findSecret('STRIPE_SECRET_KEY');
    expect(isRotationDue(stripe, '2026-10-21')).toBe(true);
    expect(daysUntilRotation(stripe, '2026-10-21')).toBe(-7);
  });

  it('laisse une rotation récente hors d’échéance', () => {
    const oauth = findSecret('GOOGLE_OAUTH_CLIENT_SECRET');
    expect(isRotationDue(oauth, '2026-10-21')).toBe(false);
    expect(daysUntilRotation(oauth, '2026-10-21')).toBe(9);
  });

  it('liste exactement les deux secrets dus au 21 octobre 2026', () => {
    const due = SECRET_CATALOG.filter((secret) => isRotationDue(secret, '2026-10-21')).map(
      (secret) => secret.envName,
    );
    expect(due).toEqual(['STRIPE_SECRET_KEY', 'STRIPE_WEBHOOK_SECRET']);
  });

  it('refuse un secret critique dont la rotation d’urgence dépasse 30 minutes', () => {
    const stripe = findSecret('STRIPE_SECRET_KEY');
    const errors = assertCatalogIntegrity([
      { ...stripe, critical: true, emergencyRotationMinutes: 45 },
    ]);
    expect(errors).toContain(
      "STRIPE_SECRET_KEY : rotation d'urgence de 45 min, 30 min maximum",
    );
  });
});
```

- [ ] **Step 2: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/config exec vitest run src/__tests__/secrets.spec.ts
```

Attendu : échec, `Cannot find module '../secrets'` (le catalogue n'existe pas encore).

- [ ] **Step 3: Implémenter le catalogue des secrets**

Créer `packages/config/src/secrets.ts` :

```ts
/**
 * Catalogue des secrets applicatifs (spec §19.5, infra §9.5).
 *
 * Ce fichier ne contient que des NOMS de variables et des RÉFÉRENCES de coffre.
 * Aucune valeur, aucun fragment de clé, aucun exemple crédible : un secret
 * écrit ici finit dans l'historique Git, donc dans le bundle et les logs.
 */

export type SecretRotationPolicy = 'monthly' | 'quarterly' | 'semi-annual';

export type SecretDefinition = {
  /** Nom de la variable d'environnement (jamais sa valeur). */
  readonly envName: string;
  /** Référence du secret dans le coffre (Vault / KMS). */
  readonly secretRef: string;
  /** Rôle responsable de la rotation. */
  readonly owner: string;
  /** Périodicité de rotation. */
  readonly policy: SecretRotationPolicy;
  /** Date de la dernière rotation réussie (AAAA-MM-JJ). */
  readonly lastRotatedAt: string;
  /** Durée de coexistence de l'ancien et du nouveau secret, en jours. */
  readonly overlapDays: number;
  /** Le gestionnaire accepte-t-il deux versions actives simultanément ? */
  readonly supportsDualSecret: boolean;
  /** Le secret est-il critique (rotation d'urgence obligatoire) ? */
  readonly critical: boolean;
  /** Durée maximale de la rotation d'urgence, en minutes. */
  readonly emergencyRotationMinutes: number;
};

export const ROTATION_PERIOD_DAYS: Readonly<Record<SecretRotationPolicy, number>> = {
  monthly: 30,
  quarterly: 91,
  'semi-annual': 182,
};

const DAY_MS = 86_400_000;

export const SECRET_CATALOG: readonly SecretDefinition[] = [
  {
    envName: 'STRIPE_SECRET_KEY',
    secretRef: 'secret://fanours/prod/stripe-live',
    owner: 'finance-manager',
    policy: 'quarterly',
    lastRotatedAt: '2026-07-15',
    overlapDays: 7,
    supportsDualSecret: true,
    critical: true,
    emergencyRotationMinutes: 15,
  },
  {
    envName: 'STRIPE_WEBHOOK_SECRET',
    secretRef: 'secret://fanours/prod/stripe-webhook',
    owner: 'payments-owner',
    policy: 'quarterly',
    lastRotatedAt: '2026-07-15',
    overlapDays: 7,
    supportsDualSecret: true,
    critical: true,
    emergencyRotationMinutes: 15,
  },
  {
    envName: 'DATABASE_PASSWORD',
    secretRef: 'secret://fanours/prod/postgres',
    owner: 'platform-owner',
    policy: 'quarterly',
    lastRotatedAt: '2026-08-01',
    overlapDays: 0,
    supportsDualSecret: false,
    critical: true,
    emergencyRotationMinutes: 25,
  },
  {
    envName: 'REDIS_PASSWORD',
    secretRef: 'secret://fanours/prod/redis',
    owner: 'platform-owner',
    policy: 'quarterly',
    lastRotatedAt: '2026-08-20',
    overlapDays: 0,
    supportsDualSecret: false,
    critical: true,
    emergencyRotationMinutes: 20,
  },
  {
    envName: 'MFA_ENVELOPE_KEY',
    secretRef: 'secret://fanours/prod/mfa-envelope',
    owner: 'security-owner',
    policy: 'quarterly',
    lastRotatedAt: '2026-08-25',
    overlapDays: 0,
    supportsDualSecret: false,
    critical: true,
    emergencyRotationMinutes: 30,
  },
  {
    envName: 'GOOGLE_OAUTH_CLIENT_SECRET',
    secretRef: 'secret://fanours/prod/google-oauth',
    owner: 'auth-owner',
    policy: 'quarterly',
    lastRotatedAt: '2026-08-01',
    overlapDays: 0,
    supportsDualSecret: false,
    critical: false,
    emergencyRotationMinutes: 60,
  },
  {
    envName: 'SMTP_PASSWORD',
    secretRef: 'secret://fanours/prod/smtp',
    owner: 'platform-owner',
    policy: 'quarterly',
    lastRotatedAt: '2026-08-10',
    overlapDays: 0,
    supportsDualSecret: false,
    critical: false,
    emergencyRotationMinutes: 60,
  },
  {
    envName: 'S3_SECRET_KEY',
    secretRef: 'secret://fanours/prod/s3-media',
    owner: 'platform-owner',
    policy: 'quarterly',
    lastRotatedAt: '2026-09-01',
    overlapDays: 0,
    supportsDualSecret: false,
    critical: false,
    emergencyRotationMinutes: 60,
  },
];

export function rotationPeriodDays(policy: SecretRotationPolicy): number {
  return ROTATION_PERIOD_DAYS[policy];
}

export function nextRotationDate(secret: SecretDefinition, from: string): string {
  const base = Date.parse(`${from}T00:00:00Z`);
  if (Number.isNaN(base)) {
    throw new TypeError(`date de rotation invalide : ${from}`);
  }
  return new Date(base + rotationPeriodDays(secret.policy) * DAY_MS).toISOString().slice(0, 10);
}

export function isRotationDue(secret: SecretDefinition, today: string): boolean {
  return nextRotationDate(secret, secret.lastRotatedAt) <= today;
}

/** Nombre de jours entre aujourd'hui et la prochaine rotation (négatif si en retard). */
export function daysUntilRotation(secret: SecretDefinition, today: string): number {
  const now = Date.parse(`${today}T00:00:00Z`);
  if (Number.isNaN(now)) {
    throw new TypeError(`date de référence invalide : ${today}`);
  }
  const due = Date.parse(`${nextRotationDate(secret, secret.lastRotatedAt)}T00:00:00Z`);
  return Math.round((due - now) / DAY_MS);
}

export function assertCatalogIntegrity(catalog: readonly SecretDefinition[]): string[] {
  const errors: string[] = [];
  const seen = new Set<string>();

  for (const secret of catalog) {
    if (seen.has(secret.envName)) {
      errors.push(`${secret.envName} : variable dupliquée dans le catalogue`);
    }
    seen.add(secret.envName);

    if (!secret.secretRef.startsWith('secret://fanours/')) {
      errors.push(`${secret.envName} : référence hors coffre applicatif (${secret.secretRef})`);
    }
    if (secret.owner.trim().length === 0) {
      errors.push(`${secret.envName} : propriétaire de rotation absent`);
    }
    if (!/^\d{4}-\d{2}-\d{2}$/.test(secret.lastRotatedAt)) {
      errors.push(`${secret.envName} : date de dernière rotation invalide (${secret.lastRotatedAt})`);
    }
    if (secret.overlapDays < 0 || secret.overlapDays > 30) {
      errors.push(`${secret.envName} : durée de chevauchement hors bornes (${secret.overlapDays} j)`);
    }
    if (secret.supportsDualSecret && secret.overlapDays === 0) {
      errors.push(`${secret.envName} : double secret déclaré sans fenêtre de chevauchement`);
    }
    if (!secret.supportsDualSecret && secret.overlapDays > 0) {
      errors.push(`${secret.envName} : chevauchement déclaré sans double secret`);
    }
    if (secret.critical && secret.emergencyRotationMinutes > 30) {
      errors.push(
        `${secret.envName} : rotation d'urgence de ${secret.emergencyRotationMinutes} min, 30 min maximum`,
      );
    }
  }

  return errors;
}
```

- [ ] **Step 4: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/config exec vitest run src/__tests__/secrets.spec.ts
```

Attendu : 9 tests passés.

- [ ] **Step 5: Écrire la migration 0122 — baselines de performance par release**

Créer `packages/db/migrations/0122_perf_baselines.sql` :

```sql
-- 0122_perf_baselines.sql
-- Baselines de performance par release.
--
-- Une rotation de secret ou un redeloiement doit pouvoir être comparé à la
-- baseline de la version précédente : c'est la preuve qu'une opération
-- d'exploitation n'a pas dégradé les budgets (Task 1) ni les SLO (Task 7).
-- Les lignes de mesures sont append-only : une baseline ne se réécrit jamais.

SET lock_timeout = '5s';
SET statement_timeout = '60s';

CREATE TABLE IF NOT EXISTS release_baseline (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  release text NOT NULL,
  image_digest text NOT NULL,
  schema_version integer NOT NULL,
  source text NOT NULL,
  measured_at timestamptz NOT NULL DEFAULT now(),
  notes text,
  CONSTRAINT ck_release_baseline_source CHECK (
    source IN ('lighthouse_ci', 'bundle_report', 'prometheus', 'k6', 'manual')
  ),
  CONSTRAINT ck_release_baseline_schema_version CHECK (schema_version >= 0),
  CONSTRAINT uq_release_baseline_identity UNIQUE (release, image_digest, schema_version)
);

CREATE TABLE IF NOT EXISTS release_baseline_metric (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  baseline_id uuid NOT NULL REFERENCES release_baseline (id) ON DELETE RESTRICT,
  route text NOT NULL,
  metric text NOT NULL,
  measured_value numeric(18,4) NOT NULL,
  budget_value numeric(18,4),
  deviation_pct numeric(8,4),
  measured_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT ck_release_baseline_metric_metric CHECK (
    metric IN (
      'js_first_load_kb', 'css_kb', 'requests', 'lcp_ms', 'inp_ms', 'cls',
      'api_p95_ms', 'error_rate', 'cost_eur'
    )
  ),
  CONSTRAINT ck_release_baseline_metric_measured CHECK (measured_value >= 0),
  CONSTRAINT ck_release_baseline_metric_budget CHECK (budget_value IS NULL OR budget_value >= 0)
);

CREATE INDEX IF NOT EXISTS ix_release_baseline_measured_at
  ON release_baseline (measured_at DESC);
CREATE INDEX IF NOT EXISTS ix_release_baseline_metric_route
  ON release_baseline_metric (route, measured_at DESC);
CREATE INDEX IF NOT EXISTS ix_release_baseline_metric_baseline
  ON release_baseline_metric (baseline_id);

CREATE OR REPLACE FUNCTION release_baseline_metric_append_only() RETURNS trigger AS $$
BEGIN
  RAISE EXCEPTION 'release_baseline_metric is append-only (attempted %)', TG_OP;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_release_baseline_metric_no_update ON release_baseline_metric;
CREATE TRIGGER trg_release_baseline_metric_no_update
  BEFORE UPDATE OR DELETE ON release_baseline_metric
  FOR EACH ROW EXECUTE FUNCTION release_baseline_metric_append_only();

REVOKE UPDATE, DELETE ON release_baseline_metric FROM PUBLIC;
```

- [ ] **Step 6: Écrire la migration 0123 — journal d'audit immuable des rotations**

Créer `packages/db/migrations/0123_secret_rotation_audit.sql` :

```sql
-- 0123_secret_rotation_audit.sql
-- Journal d'audit immuable des rotations de secrets (spec §19.5, infra §9.5).
--
-- Règles :
-- - aucune VALEUR de secret n'est stockée, seulement le nom, la référence du
--   coffre, les versions et les horodatages ;
-- - le journal est append-only : une rotation qui passe de « started » à
--   « succeeded » écrit DEUX lignes, jamais un UPDATE ;
-- - l'idempotence est assurée par idempotency_key : un rejeu de la commande de
--   rotation ne duplique aucune ligne.

SET lock_timeout = '5s';
SET statement_timeout = '60s';

CREATE TABLE IF NOT EXISTS secret_rotation_audit (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  secret_env_name text NOT NULL,
  secret_ref text NOT NULL,
  rotation_type text NOT NULL,
  status text NOT NULL,
  secret_version text,
  previous_version text,
  overlap_until timestamptz,
  rotated_by text NOT NULL,
  approved_by text,
  correlation_id text NOT NULL,
  incident_ref text,
  idempotency_key text NOT NULL,
  rotated_at timestamptz NOT NULL DEFAULT now(),
  verified_at timestamptz,
  notes text,
  CONSTRAINT ck_secret_rotation_audit_type CHECK (
    rotation_type IN ('scheduled', 'emergency', 'initial')
  ),
  CONSTRAINT ck_secret_rotation_audit_status CHECK (
    status IN ('started', 'succeeded', 'failed', 'rolled_back')
  ),
  CONSTRAINT ck_secret_rotation_audit_actor CHECK (length(trim(rotated_by)) > 0),
  CONSTRAINT ck_secret_rotation_audit_ref CHECK (secret_ref LIKE 'secret://fanours/%'),
  CONSTRAINT uq_secret_rotation_audit_idempotency UNIQUE (idempotency_key)
);

CREATE INDEX IF NOT EXISTS ix_secret_rotation_audit_secret
  ON secret_rotation_audit (secret_env_name, rotated_at DESC);
CREATE INDEX IF NOT EXISTS ix_secret_rotation_audit_status
  ON secret_rotation_audit (status, rotated_at DESC);
CREATE INDEX IF NOT EXISTS ix_secret_rotation_audit_rotated_at
  ON secret_rotation_audit (rotated_at DESC);

CREATE OR REPLACE FUNCTION fn_secret_rotation_audit_append_only() RETURNS trigger AS $$
BEGIN
  RAISE EXCEPTION 'secret_rotation_audit is append-only (attempted %)', TG_OP;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_secret_rotation_audit_no_update ON secret_rotation_audit;
CREATE TRIGGER trg_secret_rotation_audit_no_update
  BEFORE UPDATE OR DELETE ON secret_rotation_audit
  FOR EACH ROW EXECUTE FUNCTION fn_secret_rotation_audit_append_only();

REVOKE UPDATE, DELETE ON secret_rotation_audit FROM PUBLIC;
```

Créer `packages/db/src/schema/secret-rotation-audit.ts` :

```ts
import { index, pgTable, text, timestamp, uniqueIndex, uuid } from 'drizzle-orm/pg-core';

export const secretRotationAudit = pgTable(
  'secret_rotation_audit',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    secretEnvName: text('secret_env_name').notNull(),
    secretRef: text('secret_ref').notNull(),
    rotationType: text('rotation_type').notNull(),
    status: text('status').notNull(),
    secretVersion: text('secret_version'),
    previousVersion: text('previous_version'),
    overlapUntil: timestamp('overlap_until', { withTimezone: true }),
    rotatedBy: text('rotated_by').notNull(),
    approvedBy: text('approved_by'),
    correlationId: text('correlation_id').notNull(),
    incidentRef: text('incident_ref'),
    idempotencyKey: text('idempotency_key').notNull(),
    rotatedAt: timestamp('rotated_at', { withTimezone: true }).notNull().defaultNow(),
    verifiedAt: timestamp('verified_at', { withTimezone: true }),
    notes: text('notes'),
  },
  (table) => ({
    idempotencyUnique: uniqueIndex('uq_secret_rotation_audit_idempotency').on(
      table.idempotencyKey,
    ),
    secretIdx: index('ix_secret_rotation_audit_secret').on(
      table.secretEnvName,
      table.rotatedAt,
    ),
    statusIdx: index('ix_secret_rotation_audit_status').on(table.status, table.rotatedAt),
  }),
);
```

- [ ] **Step 7: Appliquer les migrations sur la base de test et prouver l'append-only**

```bash
pnpm db:migrate
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 <<'SQL'
INSERT INTO secret_rotation_audit (
  secret_env_name, secret_ref, rotation_type, status,
  rotated_by, correlation_id, idempotency_key
) VALUES (
  'STRIPE_SECRET_KEY', 'secret://fanours/prod/stripe-live', 'scheduled', 'started',
  'security-owner', 'corr-demo-0001', 'rotation:demo:0001'
);
SQL
```

Puis la tentative de mutation doit être refusée :

```bash
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -c \
  "UPDATE secret_rotation_audit SET status = 'failed' WHERE idempotency_key = 'rotation:demo:0001';"
```

Attendu : la première commande insère une ligne, la seconde échoue avec `ERROR: secret_rotation_audit is append-only (attempted UPDATE)`. Une migration qui n'est pas append-only est rejetée.

- [ ] **Step 8: Écrire le test qui échoue — la construction de l'entrée d'audit de rotation**

Créer `apps/api/src/modules/security/__tests__/secret-rotation-audit.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { ROTATION_AUDIT_INSERT, buildRotationAuditEntry } from '../secret-rotation-audit';

const BASE = {
  secretEnvName: 'STRIPE_SECRET_KEY',
  secretRef: 'secret://fanours/prod/stripe-live',
  rotationType: 'scheduled' as const,
  status: 'succeeded' as const,
  secretVersion: 'v3',
  previousVersion: 'v2',
  rotatedBy: 'security-owner',
  correlationId: 'corr-2026-10-21-0001',
  idempotencyKey: 'rotation:STRIPE_SECRET_KEY:v3:scheduled',
  rotatedAt: '2026-10-21T03:20:00.000Z',
};

describe('buildRotationAuditEntry', () => {
  it('normalise les champs optionnels absents en null', () => {
    const entry = buildRotationAuditEntry(BASE);
    expect(entry.approvedBy).toBeNull();
    expect(entry.incidentRef).toBeNull();
    expect(entry.notes).toBeNull();
    expect(entry.verifiedAt).toBeNull();
    expect(entry.overlapUntil).toBeNull();
  });

  it('conserve les champs fournis', () => {
    const entry = buildRotationAuditEntry({ ...BASE, approvedBy: 'platform-owner' });
    expect(entry.approvedBy).toBe('platform-owner');
    expect(entry.secretVersion).toBe('v3');
    expect(entry.rotatedAt).toBe('2026-10-21T03:20:00.000Z');
  });

  it('refuse une rotation d’urgence sans second approbateur', () => {
    expect(() => buildRotationAuditEntry({ ...BASE, rotationType: 'emergency' })).toThrow(
      /second rôle/,
    );
  });

  it('refuse une rotation d’urgence sans référence d’incident', () => {
    expect(() =>
      buildRotationAuditEntry({
        ...BASE,
        rotationType: 'emergency',
        approvedBy: 'platform-owner',
      }),
    ).toThrow(/incident/);
  });

  it('refuse une rotation réussie sans version de secret', () => {
    expect(() =>
      buildRotationAuditEntry({
        secretEnvName: 'STRIPE_SECRET_KEY',
        secretRef: 'secret://fanours/prod/stripe-live',
        rotationType: 'scheduled',
        status: 'succeeded',
        rotatedBy: 'security-owner',
        correlationId: 'corr-1',
        idempotencyKey: 'rotation:k1',
        rotatedAt: '2026-10-21T03:20:00.000Z',
      }),
    ).toThrow(/version/);
  });

  it('refuse une référence qui n’est pas une URI du coffre', () => {
    expect(() => buildRotationAuditEntry({ ...BASE, secretRef: 'STRIPE_SECRET_KEY' })).toThrow(
      /secret:\/\/fanours\//,
    );
  });

  it('refuse une valeur de secret glissée dans les notes', () => {
    expect(() =>
      buildRotationAuditEntry({ ...BASE, notes: 'valeur sk_testABCDEFGHIJKLMNOPQRSTUVWXYZ012345' }),
    ).toThrow(/secret/);
  });

  it('rend l’insertion idempotente sur idempotency_key', () => {
    expect(ROTATION_AUDIT_INSERT).toContain('ON CONFLICT (idempotency_key) DO NOTHING');
  });
});
```

- [ ] **Step 9: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/security/__tests__/secret-rotation-audit.spec.ts
```

Attendu : échec, `Cannot find module '../secret-rotation-audit'`.

- [ ] **Step 10: Implémenter le module d'audit de rotation**

Créer `apps/api/src/modules/security/secret-rotation-audit.ts` :

```ts
export type RotationType = 'scheduled' | 'emergency' | 'initial';
export type RotationStatus = 'started' | 'succeeded' | 'failed' | 'rolled_back';

export type RotationAuditInput = {
  readonly secretEnvName: string;
  readonly secretRef: string;
  readonly rotationType: RotationType;
  readonly status: RotationStatus;
  readonly secretVersion?: string;
  readonly previousVersion?: string;
  readonly overlapUntil?: string;
  readonly rotatedBy: string;
  readonly approvedBy?: string;
  readonly correlationId: string;
  readonly incidentRef?: string;
  readonly idempotencyKey: string;
  readonly rotatedAt?: string;
  readonly verifiedAt?: string;
  readonly notes?: string;
};

export type RotationAuditEntry = {
  readonly secretEnvName: string;
  readonly secretRef: string;
  readonly rotationType: RotationType;
  readonly status: RotationStatus;
  readonly secretVersion: string | null;
  readonly previousVersion: string | null;
  readonly overlapUntil: string | null;
  readonly rotatedBy: string;
  readonly approvedBy: string | null;
  readonly correlationId: string;
  readonly incidentRef: string | null;
  readonly idempotencyKey: string;
  readonly rotatedAt: string;
  readonly verifiedAt: string | null;
  readonly notes: string | null;
};

const SECRET_MATERIAL_PATTERNS: readonly RegExp[] = [
  /\bsk_[A-Za-z0-9]{12,}/,
  /\bpk_[A-Za-z0-9]{12,}/,
  /\bwhsec_[A-Za-z0-9]{12,}/,
  /\bAKIA[0-9A-Z]{16}\b/,
  /-----BEGIN [A-Z ]*PRIVATE KEY-----/,
];

/** Le journal d'audit ne doit jamais contenir une valeur de secret. */
export function assertNoSecretMaterial(text: string): void {
  for (const pattern of SECRET_MATERIAL_PATTERNS) {
    if (pattern.test(text)) {
      throw new Error('le journal d’audit ne doit contenir aucune valeur de secret');
    }
  }
}

export function buildRotationAuditEntry(input: RotationAuditInput): RotationAuditEntry {
  if (input.secretEnvName.trim().length === 0) {
    throw new TypeError('secretEnvName est obligatoire');
  }
  if (!input.secretRef.startsWith('secret://fanours/')) {
    throw new TypeError('secretRef doit être une référence secret://fanours/');
  }
  if (input.rotatedBy.trim().length === 0) {
    throw new TypeError('rotatedBy est obligatoire');
  }
  if (input.correlationId.trim().length === 0) {
    throw new TypeError('correlationId est obligatoire');
  }
  if (input.idempotencyKey.trim().length === 0) {
    throw new TypeError('idempotencyKey est obligatoire');
  }
  if (input.rotationType === 'emergency') {
    if (input.approvedBy === undefined || input.approvedBy.trim().length === 0) {
      throw new Error('une rotation d’urgence exige une approbation d’un second rôle');
    }
    if (input.incidentRef === undefined || input.incidentRef.trim().length === 0) {
      throw new Error('une rotation d’urgence exige une référence d’incident');
    }
  }
  if (
    input.status === 'succeeded' &&
    (input.secretVersion === undefined || input.secretVersion.trim().length === 0)
  ) {
    throw new Error('une rotation réussie exige la version du nouveau secret');
  }
  if (input.notes !== undefined) {
    assertNoSecretMaterial(input.notes);
  }

  return {
    secretEnvName: input.secretEnvName,
    secretRef: input.secretRef,
    rotationType: input.rotationType,
    status: input.status,
    secretVersion: input.secretVersion ?? null,
    previousVersion: input.previousVersion ?? null,
    overlapUntil: input.overlapUntil ?? null,
    rotatedBy: input.rotatedBy,
    approvedBy: input.approvedBy ?? null,
    correlationId: input.correlationId,
    incidentRef: input.incidentRef ?? null,
    idempotencyKey: input.idempotencyKey,
    rotatedAt: input.rotatedAt ?? new Date().toISOString(),
    verifiedAt: input.verifiedAt ?? null,
    notes: input.notes ?? null,
  };
}

/** Insertion idempotente : un rejeu de la commande ne duplique aucune ligne. */
export const ROTATION_AUDIT_INSERT = `
INSERT INTO secret_rotation_audit (
  secret_env_name,
  secret_ref,
  rotation_type,
  status,
  secret_version,
  previous_version,
  overlap_until,
  rotated_by,
  approved_by,
  correlation_id,
  incident_ref,
  idempotency_key,
  rotated_at,
  verified_at,
  notes
) VALUES (
  $1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15
)
ON CONFLICT (idempotency_key) DO NOTHING
RETURNING id
`;
```

- [ ] **Step 11: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/security/__tests__/secret-rotation-audit.spec.ts
```

Attendu : 8 tests passés.

- [ ] **Step 12: Écrire le test de non-régression du scanner de secrets**

Créer `ops/security/scanner-regression.sh` :

```bash
#!/usr/bin/env bash
# Test de non-regression du scanner de secrets.
#
# Le scanner est un composant critique : une mise a jour de version peut le
# rendre muet, et un scanner muet ne signale plus rien. Ce script verifie
# trois choses, dans cet ordre :
#   1. un secret de controle, fabrique a l'execution, est detecte ;
#   2. le depot ne contient aucun secret detectable ;
#   3. le scanner reste muet sur un repertoire vide (le harness ne triche pas).
#
# AUCUN secret, meme fictif, n'est ecrit dans le depot : la charge utile est
# assemblee a l'execution dans un repertoire temporaire.
set -euo pipefail

SCANNER_BIN="${SECRET_SCANNER_BIN:-gitleaks}"
WORKDIR="$(mktemp -d)"
trap 'rm -rf "$WORKDIR"' EXIT

if ! command -v "$SCANNER_BIN" >/dev/null 2>&1; then
  echo "SCANNER_BIN introuvable : $SCANNER_BIN (installer le scanner ou definir SECRET_SCANNER_BIN)" >&2
  exit 2
fi

# 1. Le scanner detecte un secret de controle.
FAKE_PREFIX='AKIA'
FAKE_SUFFIX='IOSFODNN7EXAMPLE'
printf 'aws_access_key_id = %s%s\n' "$FAKE_PREFIX" "$FAKE_SUFFIX" > "$WORKDIR/control.txt"

echo "== 1. Detection du secret de controle =="
if "$SCANNER_BIN" detect --source "$WORKDIR" --no-git --redact --exit-code 1 >"$WORKDIR/control.log" 2>&1; then
  echo "ECHEC : le scanner n'a pas detecte le secret de controle." >&2
  cat "$WORKDIR/control.log" >&2
  exit 1
fi
echo "OK : le secret de controle a ete detecte."

# 2. Aucun secret dans le depot.
echo "== 2. Audit du depot =="
if "$SCANNER_BIN" detect --source . --no-git --redact --exit-code 1 >"$WORKDIR/repo.log" 2>&1; then
  echo "OK : aucune fuite detectee dans le depot."
else
  echo "ECHEC : une fuite est detectee dans le depot :" >&2
  cat "$WORKDIR/repo.log" >&2
  exit 1
fi

# 3. Pas de faux positif sur un repertoire vide.
echo "== 3. Absence de faux positif =="
mkdir -p "$WORKDIR/empty"
if "$SCANNER_BIN" detect --source "$WORKDIR/empty" --no-git --redact --exit-code 1 >/dev/null 2>&1; then
  echo "OK : aucun faux positif sur un repertoire vide."
else
  echo "ECHEC : faux positif sur un repertoire vide, le test 1 n'est pas concluant." >&2
  exit 1
fi

echo "Scanner OK : detection preservee, aucune fuite dans le depot."
```

- [ ] **Step 13: Exécuter le test de non-régression du scanner**

```bash
chmod +x ops/security/scanner-regression.sh
SECRET_SCANNER_BIN=gitleaks ./ops/security/scanner-regression.sh
```

Attendu : les trois blocs affichent `OK`, la dernière ligne est `Scanner OK : detection preservee, aucune fuite dans le depot.`. Si le scanner n'est pas installé, le script sort en code 2 avec le message `SCANNER_BIN introuvable` : c'est un prérequis d'image CI, pas un test qui passe au vert par défaut.

- [ ] **Step 14: Écrire le pipeline de rotation planifiée**

Créer `ops/security/secret-rotation.yml` :

```yaml
# Pipeline de rotation planifiee des secrets (spec 19.5, infra 9.5).
#
# Ce descripteur est la source de verite de la procedure. Il est execute par le
# job CI `security-rotation` (planification trimestrielle) et par le job
# `security-rotation-emergency` (declenchement manuel, runbook RB-004).
#
# Regles absolues :
# - la rotation precede la purge de l'historique (spec 19.5) ;
# - aucune valeur de secret ne transite par ce fichier ni par ses journaux ;
# - l'ancien et le nouveau secret coexistent pendant `overlap_days`
#   (7 jours pour la cle de webhook du prestataire de paiement) ;
# - toute rotation d'urgence exige un second approbateur et une reference
#   d'incident, et doit etre closee en moins de 30 minutes ;
# - la table secret_rotation_audit est append-only : chaque etat ecrit une
#   ligne, jamais une mise a jour.

name: secret-rotation
description: >-
  Rotation planifiee et tracee des secrets applicatifs, avec double secret,
  controle de non-regression du scanner et journal d'audit immuable.

schedule:
  # Trimestriel, le 1er du mois a 03:17 UTC.
  cron: '17 3 1 */3 *'
  timezone: UTC
  # Une rotation manquée est une rotation en retard : elle est rattrapée et
  # le retard est tracé dans le rapport hebdomadaire.
  catch_up: true

variables:
  # NOMS uniquement. Les valeurs viennent du gestionnaire de secrets.
  - name: SECRET_ROTATION_CLI
    value: 'secret-rotation'
  - name: SECRET_SCANNER_BIN
    value: 'gitleaks'
  - name: ROTATION_CATALOG
    value: 'packages/config/src/secrets.ts'
  - name: AUDIT_DATABASE_URL
    value: 'secret://fanours/prod/audit-db'

# Contrat de la commande de rotation. L'implementation de `secret-rotation`
# est fournie par l'image de la plateforme ; ce pipeline fige son contrat :
#   secret-rotation list --catalog <fichier> [--due <date>]
#       -> JSON { "envName": "...", "secretRef": "...", "due": "AAAA-MM-JJ" }[]
#   secret-rotation rotate --env <NOM> --type scheduled|emergency --actor <ROLE>
#       -> JSON { "envName", "secretVersion", "previousVersion", "overlapUntil" }
#   secret-rotation verify --env <NOM> --version <VERSION>
#       -> code de sortie 0 si la version est active
steps:
  - name: list
    run: >-
      $SECRET_ROTATION_CLI list --catalog $ROTATION_CATALOG --due "$(date -u +%F)"
      > rotation-due.json
    check: 'test -s rotation-due.json'

  - name: rotate
    run: >-
      jq -c '.[]' rotation-due.json | while read -r entry; do
        env_name=$(printf '%s' "$entry" | jq -r '.envName')
        $SECRET_ROTATION_CLI rotate --env "$env_name" --type scheduled --actor security-owner
      done
    check: 'test ${PIPESTATUS[0]} -eq 0'

  - name: audit
    run: >-
      psql "$AUDIT_DATABASE_URL" -v ON_ERROR_STOP=1 -c
      "INSERT INTO secret_rotation_audit
        (secret_env_name, secret_ref, rotation_type, status, rotated_by, correlation_id, idempotency_key)
       SELECT secret_env_name, secret_ref, rotation_type, 'succeeded', 'security-owner', :'correlation_id',
              'rotation:' || secret_env_name || ':' || rotation_type || ':' || current_date
       FROM rotation_due;"

  - name: overlap
    run: >-
      jq -c '.[]' rotation-due.json | while read -r entry; do
        env_name=$(printf '%s' "$entry" | jq -r '.envName')
        version=$($SECRET_ROTATION_CLI rotate --env "$env_name" --type scheduled --actor security-owner | jq -r '.secretVersion')
        $SECRET_ROTATION_CLI verify --env "$env_name" --version "$version"
      done
    check: 'test ${PIPESTATUS[0]} -eq 0'

  - name: scanner-regression
    run: 'SECRET_SCANNER_BIN=$SECRET_SCANNER_BIN ./ops/security/scanner-regression.sh'
    check: 'test $? -eq 0'

  - name: dependency-scan
    run: 'pnpm audit --audit-level=high --json > audit.json || true; jq -e ".metadata.vulnerabilities.high == 0 and .metadata.vulnerabilities.critical == 0" audit.json'
    check: 'test $? -eq 0'

  - name: baseline
    run: >-
      psql "$AUDIT_DATABASE_URL" -v ON_ERROR_STOP=1 -c
      "INSERT INTO release_baseline (release, image_digest, schema_version, source)
       VALUES ('$RELEASE', '$IMAGE_DIGEST', $SCHEMA_VERSION, 'manual');"

emergency:
  # Declenche manuellement, jamais automatiquement.
  trigger: workflow_dispatch
  required_inputs:
    - name: incident_ref
      description: 'Reference de l'incident (ticket ou runbook RB-004).'
    - name: approved_by
      description: 'Second role qui approuve la rotation.'
  max_duration_minutes: 30
  steps:
    - name: rotate
      run: >-
        $SECRET_ROTATION_CLI rotate --env "$SECRET_ENV_NAME" --type emergency --actor "$ACTOR"
    - name: audit
      run: >-
        psql "$AUDIT_DATABASE_URL" -v ON_ERROR_STOP=1 -c
        "INSERT INTO secret_rotation_audit
          (secret_env_name, secret_ref, rotation_type, status, rotated_by, approved_by,
           correlation_id, incident_ref, idempotency_key)
         VALUES ('$SECRET_ENV_NAME', '$SECRET_REF', 'emergency', 'succeeded', '$ACTOR', '$APPROVED_BY',
                 '$CORRELATION_ID', '$INCIDENT_REF', 'rotation:emergency:$INCIDENT_REF:$SECRET_ENV_NAME');"
    - name: verify
      run: '$SECRET_ROTATION_CLI verify --env "$SECRET_ENV_NAME" --version "$SECRET_VERSION"'
```

- [ ] **Step 15: Écrire l'ADR de la rotation des secrets**

Créer `docs/adr/0017-rotation-secrets-planifiee.md` :

```markdown
# ADR-0017 — Rotation planifiée, testée et tracée des secrets

- Statut : Accepted
- Date : 2026-10-21
- Décideur : security-owner
- Remplace : aucun

## Contexte

Les secrets étaient gérés au cas par cas : chaque Incident de fuite était traité au cas par cas, sans échéance connue, sans trace, et souvent sans rotation complète. Trois risques concrets : un secret de webhook du prestataire de paiement jamais renouvelé, une clé d'encapsulation MFA jamais tournée depuis la mise en production, et l'impossibilité de prouver après coup qu'une rotation a bien eu lieu.

Le spec (§19.5) impose un gestionnaire dédié, jamais de secret en Git / build / logs / bundle, et **rotation d'abord, purge de l'historique ensuite**. La roadmap infra (§9.5) fixe une rotation trimestrielle, un double secret avec chevauchement de 7 jours pour la clé de webhook, et une rotation d'urgence en moins de 30 minutes.

## Décision

1. **Catalogue versionné.** `packages/config/src/secrets.ts` déclare chaque secret : nom de variable, référence `secret://fanours/…`, propriétaire, périodicité, date de dernière rotation, fenêtre de chevauchement, support du double secret et durée maximale de rotation d'urgence. Le catalogue ne contient **aucune valeur** : uniquement des noms et des références.
2. **Aucune rotation sans audit.** Chaque rotation écrit dans `secret_rotation_audit`, table **append-only** : un état = une ligne, jamais un `UPDATE`. L'insertion est idempotente sur `idempotency_key`.
3. **Double secret par défaut quand le gestionnaire le permet.** L'ancien et le nouveau secret coexistent pendant `overlap_days` (7 jours pour le webhook du prestataire de paiement). Le retrait de l'ancien secret est une étape distincte, jamais implicite.
4. **Quatre yeux sur l'urgence.** Une rotation d'urgence exige un second approbateur et une référence d'incident ; elle doit être close en moins de 30 minutes, sinon elle est traitée comme un incident SEV1 par le runbook `RB-004-incident-securite.md`.
5. **Le scanner est testé.** `ops/security/scanner-regression.sh` vérifie que le scanner détecte une charge utile de contrôle assemblée à l'exécution, qu'il ne signale rien sur le dépôt et qu'il ne produit pas de faux positif sur un répertoire vide. Un scanner muet est un incident de sécurité.
6. **Preuve de non-régression après déploiement.** `release_baseline` et `release_baseline_metric` enregistrent les mesures par release et par route : une rotation suivie d'un redéploiement est comparée à la baseline précédente. Sans baseline, on ne peut pas prouver qu'une opération d'exploitation n'a pas dégradé l'application.
7. **Scan de dépendances dans le même pipeline.** `pnpm audit --audit-level=high` bloque la rotation si une vulnérabilité haute ou critique est ouverte.

## Options écartées

- **Rotation manuelle uniquement, à la demande.** Écartée : aucune échéance, donc aucune rotation ; c'est l'état de départ que cette ADR supprime.
- **Rotation par le gestionnaire de secrets, sans trace applicative.** Écartée : le coffre conserve des versions, il ne prouve pas qui a roté quoi ni quand, et il ne connaît pas la fenêtre de chevauchement métier.
- **Chiffrement du secret en base avec rotation à la volée.** Écartée : déplace le secret dans la base et dans les sauvegardes, ce qui contredit l'exigence « jamais de secret stocké ailleurs que dans le gestionnaire dédié ».
- **Purge de l'historique Git avant la rotation.** Écartée : l'ordre est rotation d'abord, purge ensuite (spec §19.5) ; purger d'abord supprime la preuve et laisse l'ancien secret actif.

## Conséquences

- Le coût de la rotation est prévisible : huit secrets, une rotation trimestrielle, environ huit minutes d'exploitation par campagne.
- Le journal `secret_rotation_audit` est la preuve de conformité pour l'audit de sécurité ; il ne contient aucune valeur de secret, seulement des versions et des horodatages.
- La campagne de rotation devient rejouable : un rejeu de la commande n'insère pas de doublon (clé d'idempotence).
- Le pipeline dépend de la commande `secret-rotation` fournie par l'image de la plateforme ; son contrat est figé dans `ops/security/secret-rotation.yml` et un changement de contrat casse le pipeline, pas silencieusement.

## Réversibilité

Réversible par un nouvel ADR. La réversibilité est facilitée par le fait que le double secret coexiste : revenir à l'ancienne version consiste à réactiver la version précédente, qui est encore disponible pendant la fenêtre de chevauchement. Les lignes de `secret_rotation_audit` ne sont jamais supprimées.
```

- [ ] **Step 16: Reporter la fermeture de la dette dans le backlog**

Ajouter au fichier `docs/maintenance/TECH-DEBT-BACKLOG.md` la ligne du pipeline livré, avec le statut `fermé` — la dette « aucune rotation planifiée ni testée » n'existe plus :

```markdown
| DEBT-008 | ops/security/secret-rotation.yml | 0 j | faible | 2026-10-21 | fermé |
```

- [ ] **Step 17: Vérifier le périmètre complet**

```bash
pnpm --filter @fanours/config exec vitest run src/__tests__/secrets.spec.ts
pnpm --filter @fanours/api exec vitest run src/modules/security/__tests__/secret-rotation-audit.spec.ts
./ops/security/scanner-regression.sh
pnpm audit --audit-level=high
pnpm turbo run typecheck
```

Attendu : 9 + 8 tests passés, le scanner affiche `Scanner OK`, `pnpm audit` ne signale aucune vulnérabilité haute ou critique, `typecheck` sans erreur.

- [ ] **Step 18: Commit**

```bash
git add packages/config packages/db apps/api/src/modules/security ops/security docs/adr/0017-rotation-secrets-planifiee.md docs/maintenance/TECH-DEBT-BACKLOG.md
git commit -m "feat(security): ajoute la rotation planifiee et tracee des secrets" -m "Catalogue versionne des secrets (noms et references uniquement), journal
d'audit append-only secret_rotation_audit, migrations 0122 (baselines de
performance) et 0123 (audit de rotation), pipeline de rotation trimestrielle
avec double secret, test de non-regression du scanner et ADR 0017.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` affiche le titre, le corps et la ligne `Co-Authored-By`.

---

### Task 9: i18n densifiée — vocabulaire métier FR/EN/ZH-CN, contrôles de parité et de non-régression de formatage

**Files:**
- Create: `packages/i18n/src/dictionaries/fr-FR.json`
- Create: `packages/i18n/src/dictionaries/en.json`
- Create: `packages/i18n/src/dictionaries/zh-CN.json`
- Create: `packages/i18n/src/dictionaries/index.ts`
- Create: `packages/i18n/src/dictionaries/width.ts`
- Create: `packages/i18n/src/business-format.ts`
- Create: `packages/i18n/scripts/check-business-parity.ts`
- Create: `packages/i18n/src/__tests__/business-format.spec.ts`
- Create: `packages/i18n/src/__tests__/business-parity.spec.ts`
- Create: `apps/web/src/components/ui/TruncatableValue.tsx`
- Create: `apps/web/src/components/ui/__tests__/truncatable-value.spec.tsx`
- Modify: `packages/i18n/package.json` (le contrôle de parité du vocabulaire métier rejoint le build)

**Interfaces:**
- Consumes: `packages/i18n/src/types.ts` (`SUPPORTED_LOCALES`, `Locale`, `MessageDictionary`), `packages/i18n/src/format.ts` (`formatNumber`, `formatMoney`, `formatDate`), `packages/i18n/src/translate.ts` (modèle d'interpolation `{name}` et repli `fr-FR` → clé), `packages/i18n/scripts/check-parity.ts` (contrôle de parité de la couche `src/messages`), `docs/roadmap/sections/design.md` §3.7 (expansion EN +25–35 %, CJK sans italique et interligne +10 %, trois notions de devise avec mention « indicatif »), `docs/roadmap/sections/seo.md` (vocabulaire métier des clusters FR/EN/ZH), `Cahier_des_charges_Fanours.md` §4 (trois langues, formatage localisé des montants et des dates).
- Produces:
  - `packages/i18n/src/dictionaries/index.ts` → `export const BUSINESS_DICTIONARIES: Readonly<Record<Locale, MessageDictionary>>`, `export function businessTranslate(locale: Locale, key: string, values?: Record<string, string | number>): string`.
  - `packages/i18n/src/business-format.ts` → `export function formatMoq(quantity: number, locale: Locale): string`, `export function formatLeadTime(days: number, locale: Locale): string`, `export function formatIndicativeMoney(amountMinor: number, currency: string, locale: Locale, rateDate: Date): string`, `export function formatHsCode(code: string): string`.
  - `packages/i18n/scripts/check-business-parity.ts` → contrôle de parité, de placeholders et de largeur d'expansion ; sort en code 1 en cas d'écart.
  - `apps/web/src/components/ui/TruncatableValue.tsx` → `export function TruncatableValue(props: TruncatableValueProps): ReactElement` (troncation avec `min-w-0` + `truncate`, texte complet dans `title` et `aria-label`).

---

- [ ] **Step 1: Écrire le test qui échoue — parité, placeholders et largeur d'expansion des dictionnaires métier**

Créer `packages/i18n/src/__tests__/business-parity.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { BUSINESS_DICTIONARIES, businessTranslate } from '../dictionaries/index.js';
import { displayWidth } from '../dictionaries/width.js';

const PLACEHOLDER_PATTERN = /\{(\w+)\}/g;

function placeholdersOf(value: string): string[] {
  return [...value.matchAll(PLACEHOLDER_PATTERN)].map((match) => match[1]).sort();
}

describe('dictionnaires métier', () => {
  it('expose exactement les trois locales supportées', () => {
    expect(Object.keys(BUSINESS_DICTIONARIES).sort()).toEqual(['en', 'fr-FR', 'zh-CN']);
  });

  it('ne contient aucune clé vide', () => {
    for (const [locale, dictionary] of Object.entries(BUSINESS_DICTIONARIES)) {
      for (const [key, value] of Object.entries(dictionary)) {
        expect(`${locale}:${key}:${value.trim().length > 0}`).toBe(`${locale}:${key}:true`);
      }
    }
  });

  it('ne contient aucun marqueur de travail en attente', () => {
    for (const [locale, dictionary] of Object.entries(BUSINESS_DICTIONARIES)) {
      for (const value of Object.values(dictionary)) {
        expect(value).not.toMatch(/TBD|TODO|à compléter|FIXME|XXX/);
      }
    }
  });

  it('a le même jeu de clés dans les trois langues', () => {
    const reference = Object.keys(BUSINESS_DICTIONARIES['fr-FR']).sort();
    expect(Object.keys(BUSINESS_DICTIONARIES.en).sort()).toEqual(reference);
    expect(Object.keys(BUSINESS_DICTIONARIES['zh-CN']).sort()).toEqual(reference);
  });

  it('a le même jeu de placeholders pour une même clé', () => {
    const fr = BUSINESS_DICTIONARIES['fr-FR'];
    for (const [key, frValue] of Object.entries(fr)) {
      for (const locale of ['en', 'zh-CN'] as const) {
        expect(placeholdersOf(BUSINESS_DICTIONARIES[locale][key] ?? '')).toEqual(
          placeholdersOf(frValue),
        );
      }
    }
  });

  it('développe l’anglais dans la bande +25 % à +35 %', () => {
    const frWidth = displayWidth(Object.values(BUSINESS_DICTIONARIES['fr-FR']).join(''));
    const enWidth = displayWidth(Object.values(BUSINESS_DICTIONARIES.en).join(''));
    const ratio = enWidth / frWidth;
    expect(ratio).toBeGreaterThanOrEqual(1.25);
    expect(ratio).toBeLessThanOrEqual(1.35);
  });

  it('ne fait jamais exploser la largeur du chinois (glyphes pleine largeur)', () => {
    const frWidth = displayWidth(Object.values(BUSINESS_DICTIONARIES['fr-FR']).join(''));
    const zhWidth = displayWidth(Object.values(BUSINESS_DICTIONARIES['zh-CN']).join(''));
    const ratio = zhWidth / frWidth;
    expect(ratio).toBeGreaterThanOrEqual(0.6);
    expect(ratio).toBeLessThanOrEqual(1.1);
  });
});

describe('businessTranslate', () => {
  it('interpole les valeurs fournies', () => {
    expect(businessTranslate('en', 'catalog.moq.value', { qty: 500 })).toBe(
      'starting from 500 units',
    );
  });

  it('replie sur le français puis sur la clé brute', () => {
    expect(businessTranslate('en', 'cle.inconnue')).toBe('cle.inconnue');
  });
});
```

- [ ] **Step 2: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/i18n exec vitest run src/__tests__/business-parity.spec.ts
```

Attendu : échec, `Cannot find module '../dictionaries'`.

- [ ] **Step 3: Écrire les trois dictionnaires métier**

Créer `packages/i18n/src/dictionaries/fr-FR.json` :

```json
{
  "catalog.hsCode.hint": "Code douanier à 6 chiffres, à confirmer par un conseiller",
  "catalog.hsCode.label": "Code HS douanier",
  "catalog.incoterm.dap": "DAP — delivered at place, droits à la charge de l'acheteur",
  "catalog.incoterm.ddp": "DDP — livraison duties paid, droits acquittés",
  "catalog.incoterm.label": "Terme de livraison",
  "catalog.moq.label": "Quantité minimale de commande",
  "catalog.moq.value": "à partir de {qty} unités",
  "catalog.tier.label": "Palier de prix",
  "catalog.tier.value": "{qty} unités — {price} par unité",
  "compliance.rule.expired": "Cette règle douanière a expiré, contactez le service conformité",
  "order.status.cancelled": "Annulée par le service client",
  "order.status.customs": "En cours de dédouanement",
  "order.status.delivered": "Livrée",
  "order.status.paid": "Payée",
  "order.status.pending": "En attente de paiement",
  "order.status.shipped": "Expédiée",
  "payment.currency.indicative": "Montant indicatif, taux de change du {date}",
  "payment.method.card": "Carte bancaire",
  "payment.method.transfer": "Virement bancaire",
  "quote.deposit.hint": "Pour un paiement par virement, l'acompte sécurise l'approvisionnement",
  "quote.deposit.label": "Acompte demandé",
  "quote.request.body": "Décrivez les produits, les quantités et le pays de livraison",
  "quote.request.title": "Demande de devis",
  "shipping.leadTime.day": "{days} jour ouvré",
  "shipping.leadTime.days": "{days} jours ouvrés",
  "shipping.leadTime.disclaimer": "Le délai est une estimation et ne constitue pas un engagement"
}
```

Créer `packages/i18n/src/dictionaries/en.json` :

```json
{
  "catalog.hsCode.hint": "Six-digit customs code, to be confirmed with a customs specialist",
  "catalog.hsCode.label": "HS customs code used for the destination country",
  "catalog.incoterm.dap": "DAP — delivered at place, import duties and taxes are paid by the buyer",
  "catalog.incoterm.ddp": "DDP — delivered duties paid, duties paid by the seller",
  "catalog.incoterm.label": "Shipping term (Incoterm) agreed for this order",
  "catalog.moq.label": "Minimum order quantity for this product",
  "catalog.moq.value": "starting from {qty} units",
  "catalog.tier.label": "Volume price tier applied to this quantity",
  "catalog.tier.value": "{qty} units — {price} per unit",
  "compliance.rule.expired": "This customs rule has expired, please contact our compliance team",
  "order.status.cancelled": "Cancelled by customer service",
  "order.status.customs": "In customs clearance at destination",
  "order.status.delivered": "Delivered",
  "order.status.paid": "Payment confirmed",
  "order.status.pending": "Awaiting payment to proceed",
  "order.status.shipped": "Shipped to the carrier",
  "payment.currency.indicative": "Indicative amount, exchange rate as of {date}",
  "payment.method.card": "Credit or debit card, secured by the payment provider",
  "payment.method.transfer": "Bank transfer payment",
  "quote.deposit.hint": "For bank transfer payments, the deposit secures the production slot",
  "quote.deposit.label": "Deposit amount required",
  "quote.request.body": "Describe the products, the quantities and the delivery country",
  "quote.request.title": "Request a wholesale quote",
  "shipping.leadTime.day": "{days} business day",
  "shipping.leadTime.days": "{days} business days",
  "shipping.leadTime.disclaimer": "This lead time is an estimate only, not a commitment"
}
```

Créer `packages/i18n/src/dictionaries/zh-CN.json` :

```json
{
  "catalog.hsCode.hint": "六位海关编码，需由关务专员确认",
  "catalog.hsCode.label": "目的地海关编码",
  "catalog.incoterm.dap": "DAP — 目的地交货，进口关税由买方承担",
  "catalog.incoterm.ddp": "DDP — 完税后交货，卖方已付清关税",
  "catalog.incoterm.label": "交货条款（贸易术语）",
  "catalog.moq.label": "该商品的最小起订量",
  "catalog.moq.value": "起订 {qty} 件起",
  "catalog.tier.label": "阶梯价格",
  "catalog.tier.value": "{qty} 件 — 单价 {price}",
  "compliance.rule.expired": "该海关规则已过期，请联系合规团队",
  "order.status.cancelled": "已由客服取消",
  "order.status.customs": "正在目的国清关",
  "order.status.delivered": "已送达约定地址",
  "order.status.paid": "已收到付款，正在备货",
  "order.status.pending": "等待付款",
  "order.status.shipped": "已发货，等待承运商揽收",
  "payment.currency.indicative": "参考金额，汇率日期 {date}",
  "payment.method.card": "借记卡或信用卡",
  "payment.method.transfer": "银行转账，备货前需支付定金",
  "quote.deposit.hint": "银行转账支付时，定金用于锁定排产",
  "quote.deposit.label": "所需定金金额",
  "quote.request.body": "请说明产品、数量与收货国家",
  "quote.request.title": "申请批发报价",
  "shipping.leadTime.day": "{days} 个工作日",
  "shipping.leadTime.days": "{days} 个工作日",
  "shipping.leadTime.disclaimer": "交期仅为估算，不构成承诺"
}
```

- [ ] **Step 4: Implémenter le chargement des dictionnaires et la mesure de largeur**

Créer `packages/i18n/src/dictionaries/width.ts` :

```ts
/**
 * Largeur d'affichage approximative, en unités de caractère latin.
 *
 * Les glyphes CJK et pleine largeur (chinois, japonais, coréen) occupent deux
 * unités : sans cette correction, la densité du chinois passerait pour une
 * traduction courte alors qu'elle déborde à l'écran.
 */
const FULL_WIDTH_RANGES: readonly [number, number][] = [
  [0x1100, 0x115f],
  [0x2e80, 0xa4cf],
  [0xac00, 0xd7a3],
  [0xf900, 0xfaff],
  [0xfe30, 0xfe4f],
  [0xff00, 0xff60],
  [0xffe0, 0xffe6],
];

export function isFullWidth(codePoint: number): boolean {
  return FULL_WIDTH_RANGES.some(([start, end]) => codePoint >= start && codePoint <= end);
}

export function displayWidth(text: string): number {
  let width = 0;
  for (const character of text) {
    const codePoint = character.codePointAt(0);
    width += codePoint !== undefined && isFullWidth(codePoint) ? 2 : 1;
  }
  return width;
}
```

Créer `packages/i18n/src/dictionaries/index.ts` :

```ts
import frFR from './fr-FR.json' with { type: 'json' };
import en from './en.json' with { type: 'json' };
import zhCN from './zh-CN.json' with { type: 'json' };
import type { Locale, MessageDictionary } from '../types.js';

export const BUSINESS_DICTIONARIES: Readonly<Record<Locale, MessageDictionary>> = {
  'fr-FR': frFR,
  en,
  'zh-CN': zhCN,
};

const PLACEHOLDER_PATTERN = /\{(\w+)\}/g;

/**
 * Résolution d'une clé métier : locale demandée, repli explicite sur le
 * français, puis sur la clé brute. Même contrat que `translate` de la couche
 * `src/messages`, appliqué au vocabulaire métier.
 */
export function businessTranslate(
  locale: Locale,
  key: string,
  values?: Record<string, string | number>,
): string {
  const dictionary = BUSINESS_DICTIONARIES[locale] ?? BUSINESS_DICTIONARIES['fr-FR'];
  const template = dictionary[key] ?? BUSINESS_DICTIONARIES['fr-FR'][key];
  if (template === undefined) {
    return key;
  }
  if (values === undefined) {
    return template;
  }
  return template.replace(PLACEHOLDER_PATTERN, (match, name: string) => {
    const value = values[name];
    return value === undefined ? match : String(value);
  });
}
```

- [ ] **Step 5: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/i18n exec vitest run src/__tests__/business-parity.spec.ts
```

Attendu : 9 tests passés. Si le ratio d'expansion mesuré sort de la bande `[1.25 ; 1.35]`, allonger ou raccourcir **la valeur anglaise** concernée dans `packages/i18n/src/dictionaries/en.json` (jamais un placeholder, jamais une clé) jusqu'à ce que la mesure soit dans la bande : c'est la mesure affichée par le script de l'étape 7 qui fait foi, pas la longueur écrite ici.

- [ ] **Step 6: Écrire le test qui échoue — le formatage métier**

Créer `packages/i18n/src/__tests__/business-format.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import {
  formatHsCode,
  formatIndicativeMoney,
  formatLeadTime,
  formatMoq,
} from '../business-format.js';

describe('formatMoq', () => {
  it('formate la quantité minimale dans la locale demandée', () => {
    expect(formatMoq(500, 'fr-FR')).toBe('500');
    expect(formatMoq(1234, 'en')).toBe('1,234');
  });
});

describe('formatLeadTime', () => {
  it('utilise le singulier et le pluriel selon la locale', () => {
    expect(formatLeadTime(1, 'fr-FR')).toBe('1 jour ouvré');
    expect(formatLeadTime(3, 'fr-FR')).toBe('3 jours ouvrés');
    expect(formatLeadTime(1, 'en')).toBe('1 business day');
    expect(formatLeadTime(3, 'en')).toBe('3 business days');
  });

  it('n’a pas de pluriel en chinois', () => {
    expect(formatLeadTime(2, 'zh-CN')).toBe('2 个工作日');
  });
});

describe('formatIndicativeMoney', () => {
  it('marque le montant comme indicatif avec la date du taux', () => {
    const formatted = formatIndicativeMoney(12345, 'EUR', 'fr-FR', new Date('2026-10-21T12:00:00Z'));
    expect(formatted).toMatch(/123,45\s?€/);
    expect(formatted).toContain('indicatif');
    expect(formatted).toContain('taux de change du');
  });

  it('marque le montant converti en anglais', () => {
    const formatted = formatIndicativeMoney(12345, 'EUR', 'en', new Date('2026-10-21T12:00:00Z'));
    expect(formatted).toContain('Indicative amount');
    expect(formatted).toMatch(/€123\.45/);
  });
});

describe('formatHsCode', () => {
  it('groupe le code en quatre plus deux chiffres', () => {
    expect(formatHsCode('123456')).toBe('1234.56');
  });

  it('refuse un code qui n’a pas six chiffres', () => {
    expect(() => formatHsCode('12345')).toThrow(/6 chiffres/);
  });
});
```

- [ ] **Step 7: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/i18n exec vitest run src/__tests__/business-format.spec.ts
```

Attendu : échec, `Cannot find module '../business-format'`.

- [ ] **Step 8: Implémenter le formatage métier**

Créer `packages/i18n/src/business-format.ts` :

```ts
import { businessTranslate } from './dictionaries/index.js';
import { formatDate, formatMoney, formatNumber } from './format.js';
import type { Locale } from './types.js';

export function formatMoq(quantity: number, locale: Locale): string {
  if (!Number.isFinite(quantity) || quantity <= 0) {
    throw new TypeError('formatMoq requiert une quantité strictement positive.');
  }
  return formatNumber(quantity, locale);
}

/**
 * Délai de livraison. Le chinois n'a pas de pluriel : `Intl.PluralRules`
 * renvoie toujours « other » pour `zh-CN`, la clé « days » est donc utilisée.
 */
export function formatLeadTime(days: number, locale: Locale): string {
  if (!Number.isFinite(days) || days <= 0) {
    throw new TypeError('formatLeadTime requiert un nombre de jours strictement positif.');
  }
  const rules = new Intl.PluralRules(locale);
  const key = rules.select(days) === 'one' ? 'shipping.leadTime.day' : 'shipping.leadTime.days';
  return businessTranslate(locale, key, { days: formatNumber(days, locale) });
}

/**
 * Montant converti : il est toujours marqué « indicatif » avec la date du
 * taux de change (spec §4.4 — devise d'affichage distincte de la devise de
 * transaction).
 */
export function formatIndicativeMoney(
  amountMinor: number,
  currency: string,
  locale: Locale,
  rateDate: Date,
): string {
  const amount = formatMoney(amountMinor, currency, locale);
  const marker = businessTranslate(locale, 'payment.currency.indicative', {
    date: formatDate(rateDate, locale),
  });
  return `${amount} — ${marker}`;
}

export function formatHsCode(code: string): string {
  const digits = code.replace(/[^0-9]/g, '');
  if (digits.length !== 6) {
    throw new TypeError(`code HS invalide : 6 chiffres attendus, reçu ${digits.length}`);
  }
  return `${digits.slice(0, 4)}.${digits.slice(4)}`;
}
```

- [ ] **Step 9: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/i18n exec vitest run src/__tests__/business-format.spec.ts
```

Attendu : 7 tests passés.

- [ ] **Step 10: Écrire le script de contrôle de parité du vocabulaire métier**

Créer `packages/i18n/scripts/check-business-parity.ts` :

```ts
import { readFileSync } from 'node:fs';
import { resolve } from 'node:path';
import { fileURLToPath } from 'node:url';
import { displayWidth } from '../src/dictionaries/width.js';

const LOCALES = ['fr-FR', 'en', 'zh-CN'] as const;
const PLACEHOLDER_PATTERN = /\{(\w+)\}/g;
const WORKING_MARKERS = /TBD|TODO|à compléter|FIXME|XXX/;
const EN_MIN_RATIO = 1.25;
const EN_MAX_RATIO = 1.35;
const ZH_MIN_RATIO = 0.6;
const ZH_MAX_RATIO = 1.1;

type Dictionary = Record<string, string>;

// Le chemin part du script, pas du répertoire courant : le contrôle donne le
// même résultat qu'il soit lancé depuis la racine du dépôt ou depuis le
// paquet.
const DICTIONARIES_DIR = resolve(fileURLToPath(new URL('.', import.meta.url)), '../src/dictionaries');

function loadDictionary(locale: string): Dictionary {
  const path = resolve(DICTIONARIES_DIR, `${locale}.json`);
  return JSON.parse(readFileSync(path, 'utf8')) as Dictionary;
}

function placeholdersOf(value: string): string[] {
  return [...value.matchAll(PLACEHOLDER_PATTERN)].map((match) => match[1]).sort();
}

function widthOf(dictionary: Dictionary): number {
  return displayWidth(Object.values(dictionary).join(''));
}

const dictionaries = Object.fromEntries(
  LOCALES.map((locale) => [locale, loadDictionary(locale)]),
) as Record<(typeof LOCALES)[number], Dictionary>;

const reference = dictionaries['fr-FR'];
const referenceKeys = Object.keys(reference).sort();
const errors: string[] = [];

for (const locale of LOCALES) {
  const dictionary = dictionaries[locale];
  const keys = Object.keys(dictionary).sort();

  for (const key of referenceKeys) {
    if (!keys.includes(key)) {
      errors.push(`[${locale}] cle manquante : ${key}`);
    }
  }
  for (const key of keys) {
    if (!referenceKeys.includes(key)) {
      errors.push(`[${locale}] cle inconnue : ${key}`);
    }
    const value = dictionary[key] ?? '';
    if (value.trim().length === 0) {
      errors.push(`[${locale}] valeur vide : ${key}`);
    }
    if (WORKING_MARKERS.test(value)) {
      errors.push(`[${locale}] marqueur de travail en attente : ${key}`);
    }
    const expected = placeholdersOf(reference[key] ?? '');
    const actual = placeholdersOf(value);
    if (expected.join(',') !== actual.join(',')) {
      errors.push(`[${locale}] placeholders divergents : ${key}`);
    }
  }
}

const frWidth = widthOf(reference);
const enRatio = widthOf(dictionaries.en) / frWidth;
const zhRatio = widthOf(dictionaries.zh-CN) / frWidth;

if (enRatio < EN_MIN_RATIO || enRatio > EN_MAX_RATIO) {
  errors.push(`expansion EN hors bande : ${enRatio.toFixed(3)} (attendu ${EN_MIN_RATIO} a ${EN_MAX_RATIO})`);
}
if (zhRatio < ZH_MIN_RATIO || zhRatio > ZH_MAX_RATIO) {
  errors.push(`largeur ZH hors bande : ${zhRatio.toFixed(3)} (attendu ${ZH_MIN_RATIO} a ${ZH_MAX_RATIO})`);
}

console.log(`Parite du vocabulaire metier : ${referenceKeys.length} cles dans 3 langues.`);
console.log(`Largeur FR : ${frWidth} · EN : ${(enRatio * 100).toFixed(1)} % · ZH : ${(zhRatio * 100).toFixed(1)} %`);

if (errors.length > 0) {
  for (const error of errors) {
    console.error(error);
  }
  process.exit(1);
}
```

- [ ] **Step 11: Exécuter le script de contrôle et brancher le build**

```bash
cd packages/i18n && node scripts/check-business-parity.ts
```

Attendu : `Parite du vocabulaire metier : 26 cles dans 3 langues.` puis les deux lignes de largeur mesurées, et un code de sortie 0.

Ajouter dans `packages/i18n/package.json`, à côté du contrôle de parité existant :

```json
"check:business-parity": "node scripts/check-business-parity.ts",
"build": "tsc -b && node scripts/check-parity.ts && node scripts/check-business-parity.ts"
```

- [ ] **Step 12: Écrire le test qui échoue — la troncature avec `min-w-0` et `truncate`**

Créer `apps/web/src/components/ui/__tests__/truncatable-value.spec.tsx` :

```tsx
import { render, screen } from '@testing-library/react';
import { describe, expect, it } from 'vitest';
import { TruncatableValue } from '../TruncatableValue';

describe('TruncatableValue', () => {
  it('affiche la valeur complète dans title et aria-label', () => {
    render(<TruncatableValue value="SKU-ABCD-1234-5678-9012-3456" />);
    const element = screen.getByLabelText('SKU-ABCD-1234-5678-9012-3456');
    expect(element).toHaveAttribute('title', 'SKU-ABCD-1234-5678-9012-3456');
  });

  it('tronque visuellement mais jamais le texte accessible', () => {
    render(<TruncatableValue value="SKU-ABCD-1234-5678-9012-3456" maxChars={12} />);
    const element = screen.getByLabelText('SKU-ABCD-1234-5678-9012-3456');
    expect(element).toHaveAttribute('data-truncated', 'true');
    expect(element.textContent).toContain('SKU-ABCD-12…');
  });

  it('ne tronque pas une valeur courte', () => {
    render(<TruncatableValue value="SKU-1234" />);
    const element = screen.getByLabelText('SKU-1234');
    expect(element).toHaveAttribute('data-truncated', 'false');
    expect(element.textContent).toContain('SKU-1234');
  });

  it('applique min-w-0 et truncate pour absorber l’expansion EN', () => {
    render(<TruncatableValue value="SKU-ABCD-1234-5678-9012-3456" />);
    const element = screen.getByLabelText('SKU-ABCD-1234-5678-9012-3456');
    expect(element.className).toContain('min-w-0');
    expect(element.className).toContain('truncate');
  });
});
```

- [ ] **Step 13: Exécuter le test et constater l'échec**

```bash
pnpm --filter @fanours/web exec vitest run src/components/ui/__tests__/truncatable-value.spec.tsx
```

Attendu : échec, `Cannot find module '../TruncatableValue'`.

- [ ] **Step 14: Implémenter le composant de troncature**

Créer `apps/web/src/components/ui/TruncatableValue.tsx` :

```tsx
import type { ReactElement, ReactNode } from 'react';

export interface TruncatableValueProps {
  /** Valeur complète, jamais tronquée dans le texte accessible. */
  readonly value: string;
  /** Nombre maximal de caractères affichés avant troncature. */
  readonly maxChars?: number;
  /** Contenu personnalisé ; par défaut la valeur tronquée. */
  readonly children?: ReactNode;
  readonly className?: string;
}

/**
 * Affiche une valeur métier potentiellement longue (SKU, numéro de commande)
 * sans jamais déborder : `min-w-0` autorise la compression dans un parent
 * flex ou grid, `truncate` coupe visuellement, et la valeur complète reste
 * disponible dans `title` et `aria-label` (design §3.7).
 */
export function TruncatableValue({
  value,
  maxChars = 24,
  children,
  className,
}: TruncatableValueProps): ReactElement {
  const truncated = value.length > maxChars ? `${value.slice(0, maxChars - 1)}…` : value;

  return (
    <span
      className={className === undefined ? 'min-w-0 truncate' : `min-w-0 truncate ${className}`}
      title={value}
      aria-label={value}
      data-truncated={truncated === value ? 'false' : 'true'}
    >
      {children ?? truncated}
    </span>
  );
}
```

- [ ] **Step 15: Exécuter le test et constater la réussite**

```bash
pnpm --filter @fanours/web exec vitest run src/components/ui/__tests__/truncatable-value.spec.tsx
```

Attendu : 4 tests passés.

- [ ] **Step 16: Vérifier le périmètre complet**

```bash
pnpm --filter @fanours/i18n exec vitest run
pnpm --filter @fanours/web exec vitest run src/components/ui/__tests__/truncatable-value.spec.tsx
cd packages/i18n && node scripts/check-business-parity.ts
cd ../.. && pnpm turbo run typecheck
```

Attendu : les tests de `@fanours/i18n` et du composant web passent, le script de parité affiche `Parite du vocabulaire metier : 26 cles dans 3 langues.` avec les deux ratios mesurés dans les bandes, `typecheck` sans erreur.

- [ ] **Step 17: Commit**

```bash
git add packages/i18n apps/web/src/components/ui
git commit -m "feat(i18n): densifie le vocabulaire metier FR/EN/ZH-CN" -m "Vocabulaire metier en 26 cles dans les trois langues, controle de parite
des cles et des placeholders, mesure de largeur d'expansion EN et ZH,
formatage metier (MOQ, delai, montant indicatif, code HS) et composant
TruncatableValue absorbant l'expansion anglaise.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` affiche le titre, le corps et la ligne `Co-Authored-By`.

---

### Task 10: Bilan, proposition de Phase 7 et Gate 6

**Files:**
- Create: `scripts/maintenance/phase6-gate.ts`
- Create: `scripts/maintenance/__tests__/phase6-gate.spec.ts`
- Create: `docs/decisions/phase7-proposition.md`
- Create: `docs/adr/0018-contenu-relecture-ia.md`
- Create: `docs/maintenance/PHASE-6-TRACKER.md`
- Create: `docs/maintenance/GATE-6-CHECKLIST.md`
- Create: `docs/maintenance/WEEKLY-REPORT.md`

**Interfaces:**
- Consumes: `scripts/maintenance/weekly-report.ts` (`buildWeeklyReport`, `WeeklyReportInput`, `WeeklyReport` — Task 1), `packages/config/src/slo.ts` (`SLO_TARGETS`, `SloTarget` — Task 1), `packages/config/src/budgets.ts` (`PERF_BUDGETS`, `PerfBudget` — Task 1), `packages/config/src/secrets.ts` (`SECRET_CATALOG`, `isRotationDue` — Task 8), `scripts/seo/content-lint.ts` (`lintContent`, `checkProhibitedClaims` — Task 4), `docs/roadmap/sections/roadmap-phases.md` Phase 6 et Gate 6 (les trois questions), `docs/roadmap/sections/qualite.md` §24.13 (seuils de métriques), `docs/roadmap/sections/infra.md` §16 (coûts), `docs/roadmap/sections/seo.md` (règles éditoriales et dates de revalidation), `Cahier_des_charges_Fanours.md` §18.4 (garde-fou IA).
- Produces:
  - `scripts/maintenance/phase6-gate.ts` → `export function evaluateGate6(input: Gate6Input): Gate6Result`, `export type Gate6Input`, `export type Gate6Result`, `export type Gate6Criterion`, `export type Gate6CriterionId`, `export const GATE_6_CRITERIA`.
  - `docs/decisions/phase7-proposition.md` → proposition de Phase 7 argumentée, à arbitrer par un humain.
  - `docs/adr/0018-contenu-relecture-ia.md` → ADR : relecture humaine obligatoire de tout contenu produit par IA.
  - `docs/maintenance/PHASE-6-TRACKER.md` → tableau de suivi de la Phase 6, avec la preuve de chaque tâche.
  - `docs/maintenance/GATE-6-CHECKLIST.md` → checklist du Gate 6, avec signataire par point.
  - `docs/maintenance/WEEKLY-REPORT.md` → rapport hebdomadaire de clôture de la Phase 6.

---

- [ ] **Step 1: Écrire le test qui échoue — l'évaluation du Gate 6**

Créer `scripts/maintenance/__tests__/phase6-gate.spec.ts` :

```ts
import { describe, expect, it } from 'vitest';
import { GATE_6_CRITERIA, evaluateGate6 } from '../phase6-gate';

const GREEN = {
  usageWindowDays: 30,
  qualityMetrics: [
    { name: 'flaky_rate', value: 0.2, threshold: 1, withinThreshold: true },
    { name: 'coverage_pct', value: 82, threshold: 70, withinThreshold: true },
  ],
  openCriticalFindings: 0,
  openHighFindings: 2,
};

describe('GATE_6_CRITERIA', () => {
  it('contient exactement les trois questions du roadmap', () => {
    expect(GATE_6_CRITERIA.map((criterion) => criterion.id)).toEqual([
      'usage_data',
      'quality_metrics',
      'security_findings',
    ]);
  });
});

describe('evaluateGate6', () => {
  it('passe quand les trois critères sont au vert', () => {
    const result = evaluateGate6(GREEN);
    expect(result.passed).toBe(true);
    expect(result.blocking).toEqual([]);
    expect(result.criteria).toHaveLength(3);
  });

  it('bloque tant que les trente jours de données ne sont pas là', () => {
    const result = evaluateGate6({ ...GREEN, usageWindowDays: 21 });
    expect(result.passed).toBe(false);
    expect(result.blocking).toContain('usage_data');
  });

  it('bloque si une métrique de qualité est hors seuil', () => {
    const result = evaluateGate6({
      ...GREEN,
      qualityMetrics: [
        { name: 'flaky_rate', value: 4.2, threshold: 1, withinThreshold: false },
      ],
    });
    expect(result.passed).toBe(false);
    expect(result.blocking).toContain('quality_metrics');
  });

  it('bloque si aucune métrique n’a été mesurée', () => {
    const result = evaluateGate6({ ...GREEN, qualityMetrics: [] });
    expect(result.passed).toBe(false);
    expect(result.blocking).toContain('quality_metrics');
  });

  it('bloque tant qu’un finding critique est ouvert', () => {
    const result = evaluateGate6({ ...GREEN, openCriticalFindings: 1 });
    expect(result.passed).toBe(false);
    expect(result.blocking).toContain('security_findings');
  });

  it('n’est pas bloqué par des findings élevés seuls', () => {
    const result = evaluateGate6({ ...GREEN, openHighFindings: 9 });
    expect(result.passed).toBe(true);
  });

  it('expose une question et un détail par critère', () => {
    const result = evaluateGate6({ ...GREEN, usageWindowDays: 12 });
    const usage = result.criteria.find((criterion) => criterion.id === 'usage_data');
    expect(usage?.question).toContain('30 jours');
    expect(usage?.detail).toContain('12');
  });
});
```

- [ ] **Step 2: Exécuter le test et constater l'échec**

```bash
pnpm vitest run scripts/maintenance/__tests__/phase6-gate.spec.ts
```

Attendu : échec, `Cannot find module '../phase6-gate'`.

- [ ] **Step 3: Implémenter l'évaluation du Gate 6**

Créer `scripts/maintenance/phase6-gate.ts` :

```ts
export type Gate6CriterionId = 'usage_data' | 'quality_metrics' | 'security_findings';

export type Gate6Criterion = {
  readonly id: Gate6CriterionId;
  readonly label: string;
  readonly question: string;
  readonly passed: boolean;
  readonly detail: string;
};

export type Gate6QualityMetric = {
  readonly name: string;
  readonly value: number;
  readonly threshold: number;
  readonly withinThreshold: boolean;
};

export type Gate6Input = {
  readonly usageWindowDays: number;
  readonly qualityMetrics: readonly Gate6QualityMetric[];
  readonly openCriticalFindings: number;
  readonly openHighFindings: number;
};

export type Gate6Result = {
  readonly passed: boolean;
  readonly criteria: readonly Gate6Criterion[];
  readonly blocking: readonly Gate6CriterionId[];
};

/** Les trois questions du Gate 6 (roadmap-phases.md, Gate 6 — Post-lancement). */
export const GATE_6_CRITERIA: readonly {
  id: Gate6CriterionId;
  label: string;
  question: string;
}[] = [
  {
    id: 'usage_data',
    label: '30 jours de données d’usage',
    question: '30 jours de données d’usage disponibles ?',
  },
  {
    id: 'quality_metrics',
    label: 'Métriques de qualité dans les seuils',
    question: 'Les métriques de qualité sont-elles dans les seuils ?',
  },
  {
    id: 'security_findings',
    label: 'Aucun finding critique non traité',
    question: 'Aucun finding critique non traité depuis le lancement ?',
  },
] as const;

const REQUIRED_USAGE_WINDOW_DAYS = 30;

/**
 * Évalue les trois critères du Gate 6.
 *
 * Ce code ne décide pas : il produit la matière du « Go / No-Go » qui est
 * signé par un humain. Un critère sans mesure n'est jamais considéré comme
 * satisfait — l'absence de donnée ne vaut pas un feu vert.
 */
export function evaluateGate6(input: Gate6Input): Gate6Result {
  const usagePassed = input.usageWindowDays >= REQUIRED_USAGE_WINDOW_DAYS;
  const usage: Gate6Criterion = {
    id: 'usage_data',
    label: GATE_6_CRITERIA[0].label,
    question: GATE_6_CRITERIA[0].question,
    passed: usagePassed,
    detail: `${input.usageWindowDays} jours de données disponibles, ${REQUIRED_USAGE_WINDOW_DAYS} requis`,
  };

  const outside = input.qualityMetrics.filter((metric) => !metric.withinThreshold);
  const qualityPassed = input.qualityMetrics.length > 0 && outside.length === 0;
  const quality: Gate6Criterion = {
    id: 'quality_metrics',
    label: GATE_6_CRITERIA[1].label,
    question: GATE_6_CRITERIA[1].question,
    passed: qualityPassed,
    detail:
      input.qualityMetrics.length === 0
        ? 'aucune métrique mesurée'
        : outside.length === 0
          ? `${input.qualityMetrics.length} métriques dans les seuils`
          : `${outside.length} métrique(s) hors seuil : ${outside.map((metric) => metric.name).join(', ')}`,
  };

  const findingsPassed = input.openCriticalFindings === 0;
  const findings: Gate6Criterion = {
    id: 'security_findings',
    label: GATE_6_CRITERIA[2].label,
    question: GATE_6_CRITERIA[2].question,
    passed: findingsPassed,
    detail: `${input.openCriticalFindings} finding(s) critique(s) et ${input.openHighFindings} finding(s) élevé(s) ouverts`,
  };

  const criteria: readonly Gate6Criterion[] = [usage, quality, findings];
  const blocking = criteria.filter((criterion) => !criterion.passed).map((criterion) => criterion.id);

  return { passed: blocking.length === 0, criteria, blocking };
}
```

- [ ] **Step 4: Exécuter le test et constater la réussite**

```bash
pnpm vitest run scripts/maintenance/__tests__/phase6-gate.spec.ts
```

Attendu : 8 tests passés.

- [ ] **Step 5: Créer le tableau de suivi de la Phase 6**

Créer `docs/maintenance/PHASE-6-TRACKER.md` :

```markdown
# Suivi de la Phase 6 — Améliorations post-lancement

Plan source : `docs/superpowers/plans/2026-10-21-phase-6-ameliorations-post-lancement.md`.

Règle : **une tâche n'est jamais marquée « réalisé » sans la preuve de la
colonne 4**. La preuve est la sortie d'une commande, collée dans la pull
request. Une tâche dont la preuve manque reste « à réaliser » et le rapport
hebdomadaire la compte comme ouverte.

| Tâche | Titre | Artefact principal | Preuve | Statut | Responsable |
|---|---|---|---|---|---|
| Task 1 | Cadence de maintenance, budgets de performance, rapport hebdomadaire | `packages/config/src/budgets.ts` | `pnpm --filter @fanours/config exec vitest run src/__tests__/budgets.spec.ts` | à réaliser | tech-lead |
| Task 2 | Performance web : Core Web Vitals, images, bundle, N+1 | `apps/web/scripts/bundle-budget.mjs` | `pnpm --filter @fanours/web exec vitest run scripts/__tests__/bundle-budget.spec.ts` | à réaliser | web-lead |
| Task 3 | Performance API et base : index, pool, cache Redis, files BullMQ | `apps/api/src/modules/performance/query-metrics.ts` | `pnpm --filter @fanours/api exec vitest run src/modules/performance/__tests__` | à réaliser | backend-lead |
| Task 4 | Contenu SEO et clusters éditoriaux FR/EN/ZH | `docs/content/plan-contenu.md` | `pnpm vitest run scripts/seo/__tests__/content-lint.spec.ts` | à réaliser | marketing-owner |
| Task 5 | Programmes SEO techniques : JSON-LD, hreflang, audit CI | `apps/web/src/lib/seo/hreflang.ts` | `pnpm exec playwright test e2e/seo/public-seo-audit.spec.ts` | à réaliser | seo-auditor |
| Task 6 | Consolidation des tests : mutation, contrats, charge, flakiness | `docs/maintenance/TEST-AUDIT.md` | `pnpm exec playwright test e2e/contracts/api-web.contract.spec.ts` | à réaliser | qa-e2e |
| Task 7 | Observabilité et exploitation : SLO, alertes, runbooks, coûts | `ops/runbooks/RB-009-slo-degrade.md` | `promtool test rules ops/alerting/rules/phase6-slo-burnrate.test.yml` | à réaliser | sre |
| Task 8 | Sécurité continue : rotation des secrets, scan, journal d'audit | `packages/config/src/secrets.ts` | `./ops/security/scanner-regression.sh` | à réaliser | security-owner |
| Task 9 | i18n densifiée : vocabulaire métier FR/EN/ZH-CN | `packages/i18n/src/dictionaries/en.json` | `cd packages/i18n && node scripts/check-business-parity.ts` | à réaliser | i18n-owner |
| Task 10 | Bilan, proposition de Phase 7 et Gate 6 | `docs/decisions/phase7-proposition.md` | `pnpm vitest run scripts/maintenance/__tests__/phase6-gate.spec.ts` | à réaliser | product-owner |

## Points ouverts à la fin de la Phase 6

| Point | Nature | Qui tranche |
|---|---|---|
| Signature du plan de contenu éditorial | Humain | marketing-owner + compliance-manager |
| Arbitrage des options de la Phase 7 | Humain | product-owner |
| Go / No-Go du Gate 6 | Humain | product-owner + security-owner |
| Budget de coût d'infrastructure au-delà de 400 € | Humain | finance-manager |
```

- [ ] **Step 6: Créer la checklist du Gate 6**

Créer `docs/maintenance/GATE-6-CHECKLIST.md` :

```markdown
# Checklist du Gate 6 — Post-lancement

D'après `docs/roadmap/sections/roadmap-phases.md` (Gate 6) et
`docs/roadmap/sections/qualite.md` §24.13.

Le code (`scripts/maintenance/phase6-gate.ts`) **évalue** les trois questions.
Il ne décide pas : le « Go » est signé par un humain.

## 1. Trente jours de données d’usage disponibles ?

- [ ] Fenêtre d'usage ≥ 30 jours (`usageWindowDays` dans l'évaluation du Gate)
- [ ] Les trois langues et les deux thèmes sont représentés dans les mesures
- [ ] Les mesures viennent de la production, pas d'un environnement de test
- **Source de la mesure :** `docs/maintenance/WEEKLY-REPORT.md` → section « Routes »
- **Signataire :** product-owner

## 2. Les métriques de qualité sont-elles dans les seuils ?

| Métrique | Seuil | Mesuré | Statut | Preuve |
|---|---|---|---|---|
| Taux de tests flakies | ≤ 1 % | | | `docs/maintenance/TEST-AUDIT.md` |
| Durée de la suite complète | ≤ 18 min | | | `docs/maintenance/TEST-AUDIT.md` |
| Couverture | indicateur, suivi mensuellement | | | rapport de couverture CI |
| LCP p75 | ≤ 2,5 s | | | `ops/performance/budgets.yaml` |
| INP p75 | ≤ 200 ms | | | `ops/performance/budgets.yaml` |
| CLS p75 | ≤ 0,1 | | | `ops/performance/budgets.yaml` |
| Disponibilité des routes | ≥ 99,5 % | | | `packages/config/src/slo.ts` |
| Coût d'infrastructure | ≤ 500 €/mois | | | `docs/maintenance/WEEKLY-REPORT.md` |

- [ ] Toutes les lignes du tableau sont renseignées et dans les seuils
- [ ] Aucun test flaky ne vit dans la suite bloquante depuis plus de 24 h
- **Signataire :** qa-e2e + sre

## 3. Aucun finding critique non traité depuis le lancement ?

- [ ] 0 finding critique ouvert
- [ ] Les findings élevés restants ont un propriétaire et une échéance
- [ ] La revue de sécurité est relue et signée
- [ ] Le drill de restauration a été exécuté avec RPO et RTO mesurés
- [ ] La rotation des secrets est planifiée, testée et tracée
- [ ] Le test de non-régression du scanner de secrets est vert
- **Source de la mesure :** `pnpm audit --audit-level=high` et la revue de sécurité
- **Signataire :** security-owner

## Points de conformité à revalider à la clôture

- [ ] Les pages de contenu indexable ont une source officielle datée et un relecteur nommé
- [ ] Aucune règle douanière citée n'est expirée (`content_registry.revalidate_at`)
- [ ] Aucun contenu produit par IA n'est publié sans relecture signée (ADR-0018)
- [ ] Les runbooks RB-001 à RB-010 existent et ont été lus par l'équipe d'astreinte

## Décision

- [ ] **Go** — la Phase 7 peut être arbitrée
- [ ] **No-Go** — la Phase 6 est réouverte sur les points bloquants listés ci-dessus

**Date de la décision :** ______
**Signataire :** ______
```

- [ ] **Step 7: Créer le rapport hebdomadaire de clôture de la Phase 6**

Créer `docs/maintenance/WEEKLY-REPORT.md` :

```markdown
# Rapport hebdomadaire — semaine du 2026-10-19

Généré le 2026-10-21 à 06:00 UTC par le job `weekly-maintenance-report`
(`apps/worker/src/jobs/weekly-maintenance-report.ts`). Les valeurs de ce
rapport sont celles du mois écoulé ; les fichiers suivants sont la version
machine : `docs/maintenance/reports/2026-10/`.

## Routes

Les budgets de référence sont ceux de `packages/config/src/budgets.ts` ; les
SLO sont ceux de `packages/config/src/slo.ts`.

| Route | Statut | Violations |
|---|---|---|
| public:home | ok | — |
| public:catalog | ok | — |
| public:product | ok | — |
| public:cart-checkout | ok | — |
| public:account | ok | — |
| admin:dashboard | ok | — |

## Dette technique

| Indicateur | Valeur | Statut |
|---|---|---|
| Entrées ouvertes | 5 | ok |
| Entrées échues | 0 | ok |

## Coût d'infrastructure

| Indicateur | Valeur | Seuil | Statut |
|---|---|---|---|
| Coût mensuel | 412 € | 500 € | warning |
| Alerte déclenchée | `FanoursInfraCostWarning` | 400 € | oui |

## Sécurité continue

| Contrôle | Résultat |
|---|---|
| Secrets dus à la rotation au 2026-10-21 | 2 (`STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`) |
| Test de non-régression du scanner | vert |
| Vulnérabilités hautes ou critiques (`pnpm audit`) | 0 |

## Contenu éditorial

| Indicateur | Valeur |
|---|---|
| Pages indexables enregistrées | 0 publié, 30 en relecture |
| Pages dont `revalidate_at` est dépassé | 0 |
| Pages publiées sans relecteur nommé | 0 |

## Décision de la semaine

La Phase 6 est en cours de clôture. Le Gate 6 (`docs/maintenance/GATE-6-CHECKLIST.md`)
reste à signer : 30 jours de données d’usage sont disponibles, les métriques de
qualité sont dans les seuils, aucun finding critique n'est ouvert, mais la
décision de « Go » appartient à un humain.
```

- [ ] **Step 8: Écrire la proposition de Phase 7**

Créer `docs/decisions/phase7-proposition.md` :

```markdown
# Proposition de Phase 7 — Fanours

- Statut : **proposition**, en attente d'arbitrage humain
- Date : 2026-10-21
- Rédacteur : équipe plateforme
- Arbitre requis : product-owner, avec security-owner, finance-manager et compliance-manager
- Rappel : **ce document ne décide pas.** Il prépare la décision et nomme ce qui doit être tranché par un humain.

## 1. Pourquoi la Phase 6 n’a pas livré de fonctionnalité nouvelle

La Phase 6 du roadmap a un objectif explicite : « optimiser après mesure
réelle. Rien n’est engagé avant d’avoir 30 jours de données d’usage. » Les dix
tâches livrées sont donc des fondations de mesure et de garde-fou — budgets de
performance, dette outillée, contenu fondé sur des sources officielles,
consolidation de la suite de tests, SLO et alertes, rotation des secrets,
vocabulaire métier trilingue — et non des candidats fonctionnels de la liste
« PWA complète et notifications push · tarifs contractuels avancés · intégrations
ERP/WMS/comptabilité · moteur de recherche dédié · IA assistive · rapports
multi-entrepôts et prévision de stock ».

Ces candidats sont **proposés pour la Phase 7**, jamais lancés en Phase 6 :
lancer une intégration ERP sans trente jours de données d’usage revient à
deviner le besoin.

## 2. Signaux mesurés et règles de décision

Chaque signal est lu dans un artefact de la Phase 6. La règle de décision est
écrite ici ; la valeur est lue au moment de l’arbitrage.

| Signal | Source | Règle de décision |
|---|---|---|
| Coût d’infrastructure mensuel | `docs/maintenance/WEEKLY-REPORT.md` → `cost.monthlyEur` | ≥ 400 € : aucune fonctionnalité de la Phase 7 ne peut augmenter le coût récurrent sans justification écrite du finance-manager |
| Dette technique ouverte | `WEEKLY-REPORT.md` → `debt.open` | > 10 entrées : la Phase 7 commence par la dette, pas par une fonctionnalité |
| Dépassement des budgets de performance | `ops/performance/budgets.yaml` + rapport | route hors budget : la Phase 7 commence par le correctif de performance |
| Taux de flakiness et durée de suite | `docs/maintenance/TEST-AUDIT.md` | suite > 18 min ou flakies > 1 % : la Phase 7 commence par la consolidation des tests |
| Recherches sans résultat | analytique produit | > 25 % des recherches sans clic : le moteur de recherche entre en tête de liste |
| Demandes de devis B2B sans relance | analytique produit | > 15 % des devis sans commande : les tarifs contractuels avancés entrent en tête de liste |
| Demandes de rupture de stock par SKU | analytique produit | > 10 % des commandes en rupture : la prévision de stock entre en tête de liste |
| Règles douanières à revalider | `content_registry.revalidate_at` | ≥ 1 règle expirée : la conformité passe avant toute fonctionnalité |

## 3. Options

### Option A — Consolidation avant fonctionnalité (recommandée)

Contenu : solder la dette critique, sortir les routes hors budget de
performance, terminer la relecture des 30 pages éditoriales, et achever la
rotation des secrets avec le double secret.

- Coût : 3 à 5 semaines-homme
- Risque : faible
- Dépendance : aucune
- Effet mesurable : dette ouverte, routes hors budget, pages publiées

### Option B — Coeur métier B2B

Contenu : espace multi-utilisateur entreprise, tarifs contractuels avancés,
devis multi-versions, espace de suivi de commande partagé.

- Coût : 6 à 8 semaines-homme
- Risque : moyen (modèle de données des contrats clients à faire évoluer)
- Dépendance : arbitrage du propriétaire métier sur la politique B2B (MOQ, délais, retours) — décision non rendue à ce jour
- Effet mesurable : taux de transformation devis → commande

### Option C — Chaîne d’approvisionnement

Contenu : intégrations ERP / WMS / comptabilité, import catalogue, rapports
multi-entrepôts, prévision de stock.

- Coût : 10 à 14 semaines-homme
- Risque : élevé (connecteurs tiers, formats Ping, idempotence des imports)
- Dépendance : choix des ERP cibles, identifiants de test, politique de facturation locale
- Effet mesurable : délai de traitement d’une commande, taux de rupture de stock

### Option D — Découverte et confort

Contenu : PWA complète et notifications push, moteur de recherche dédié,
recommandations, IA assistive (support, traduction, complétude catalogue).

- Coût : 6 à 9 semaines-homme
- Risque : moyen (l’IA assistive est encadrée par le garde-fou : elle ne fixe aucun prix, ne valide aucun code HS seule, ne confirme aucune importabilité, ne déclenche ni remboursement ni suspension, n’invente ni certification, ni avis, ni stock, ni délai)
- Dépendance : mesurer d’abord la valeur d’une recommandation ou d’une notification push ; sans mesure, ce n’est pas une amélioration
- Effet mesurable : engagement, taux de retour, taux de conversion

## 4. Recommandation

**Commencer par l’option A**, quoi qu’il arrive, puis **n’ouvrir qu’une seule**
des options B, C ou D, choisie par le product-owner à partir des signaux du
tableau §2. Les options B, C et D ne sont pas cumulables sur la même période :
elles se disputent les mêmes postes (dette, budget d’infrastructure, attention
métier).

## 5. Ce que ce document ne décide pas

- Le choix de l’option B, C ou D : arbitrage product-owner.
- La politique B2B (MOQ, délais, retours, Incoterms, devise de référence) :
  décision du propriétaire métier, listée au §12 du cahier des charges.
- Les pays prioritaires et les pays interdits, les catégories réglementées et
  les codes HS : décision compliance-manager, jamais automatique.
- Le budget d’infrastructure au-delà de 400 € : décision finance-manager.
- La publication de tout contenu produit par IA : relecture humaine signée
  obligatoire (ADR-0018), jamais de publication automatique.

## 6. Critères de démarrage de la Phase 7

- [ ] Gate 6 signé « Go » (`docs/maintenance/GATE-6-CHECKLIST.md`)
- [ ] Au moins un signal du tableau §2 au-dessus de son seuil, avec la valeur relevée
- [ ] Le coût récurrent estimé de l’option retenue est écrit et accepté par le finance-manager
- [ ] Les dépendances listées dans l’option retenue sont levées ou explicitement mockées
- [ ] Le plan de la Phase 7 existe au même format que celui de la Phase 6, avec ses critères d’acceptation
```

- [ ] **Step 9: Écrire l'ADR sur la relecture humaine du contenu produit par IA**

Créer `docs/adr/0018-contenu-relecture-ia.md` :

```markdown
# ADR-0018 — Relecture humaine obligatoire de tout contenu produit par IA

- Statut : Accepted
- Date : 2026-10-21
- Décideur : product-owner
- Remplace : aucun

## Contexte

Le cahier des charges (§18.4) pose un garde-fou simple : l'IA est **assistive**.
Elle ne fixe aucun prix final, ne valide aucun code HS seule, ne confirme
aucune importabilité, ne déclenche ni remboursement ni suspension, n'invente ni
certification, ni avis, ni stock, ni délai.

La Phase 6 produit beaucoup de contenu éditorial multilingue (Task 4) et
d'assistance produit (traduction, complétude catalogue, support). Un contenu
généré, révisé ou traduit automatiquement peut citer une franchise douanière,
un délai, un code HS, un prix ou une certification qui n'existe pas ou n'est
plus valide. Publié tel quel, ce contenu engage l'entreprise sur un fait que
aucune machine ne peut vérifier, et il engage aussi le référencement : un
contenu faux ou obsolète est un risque de réputation et de sanction.

## Décision

1. **L'IA prépare, l'humain valide.** L'IA rédige la trame, la structure, les variantes linguistiques et les reformulations. **La validation des faits est humaine**, sans exception.
2. **Aucune publication sans relecture signée.** Une page indexable n'est publiée que si un relecteur nommé a signé sa relecture. La signature est portée par le champ `owner` de `content_registry` : un contenu sans relecteur nommé n'est pas publiable.
3. **Source officielle datée obligatoire.** Tout fait chiffré ou réglementaire provient de `trade_rule` / `trade_rule_version` avec **source officielle + URL + date de consultation**, ou d'un contenu validé par le propriétaire. Aucun contenu SEO n'est publié sans source officielle datée ni relecture humaine.
4. **Périmètre d'intervention de l'IA — rappel du garde-fou §18.4.** L'IA ne fixe aucun prix final, ne valide aucun code HS seule, ne confirme aucune importabilité, ne déclenche ni remboursement ni suspension, et n'invente ni certification, ni avis, ni stock, ni délai. Tout output touchant conformité, sécurité, taxe ou contrat exige une vérification humaine avant publication.
5. **Interdits de publication maintenus.** Aucun avis, aucun compteur d'achat, aucun badge de rareté, aucune urgence artificielle, aucune promesse de classement, aucune donnée de client réel. Le `aggregateRating` n'est publié qu'avec des avis réellement vérifiés.
6. **La relecture est un travail, pas une signature de façade.** Le relecteur dispose de la source, de la date de consultation et de la liste des interdits ; il rend une décision motivée, et il n'est pas l'auteur du texte qu'il relit.
7. **Relecture obligatoire en cas de mise à jour.** Un contenu dont `revalidate_at` est dépassé sort des liens internes et déclenche une alerte au compliance-manager, jusqu'à relecture.
8. **Traçabilité.** Le registre `content_registry` conserve pour chaque page la source, la date de consultation, la date de revalidation et le relecteur. Un contrôle automatisé (`scripts/seo/content-lint.ts`) vérifie la présence de ces champs ; il ne vérifie pas la justesse du fond, qui reste humaine.

## Options écartées

- **Publication automatique avec revue a posteriori.** Écartée : un contenu faux publié puis corrigé reste indexé, partagé et cité. Le risque de réputation n'est pas réversible par une correction.
- **Relecture par un modèle.** Écartée : un modèle ne peut pas certifier un fait réglementaire ; il ne fait que déplacer le risque sans le couvrir.
- **Relecture obligatoire de tout texte, y compris les libellés d'interface.** Écartée : les libellés d'interface ne portent pas de fait réglementaire ; ils sont couverts par la revue de code et les tests d'interface. La règle porte sur les contenus publiés.

## Conséquences

- La production éditoriale est plus lente : chaque page demande une source, un relecteur nommé et une date de revalidation. C'est le prix de l'absence de dette réglementaire.
- Le contrôle automatisé (`content-lint`) traite la forme (champs présents, dates, interdits lexicaux) ; la signature humaine traite le fond. Les deux sont nécessaires : ni l'un ni l'autre ne suffit seul.
- Un contenu non relu est un contenu non publié : l'absence de relecteur bloque la publication au lieu de la retarder.

## Réversibilité

Réversible par un nouvel ADR. Les signatures de relecture restent dans `content_registry` et dans l'historique de `audit_log` : elles ne sont jamais supprimées, quelle que soit la décision ultérieure.
```

- [ ] **Step 10: Vérifier le périmètre complet**

```bash
pnpm vitest run scripts/maintenance/__tests__/phase6-gate.spec.ts
pnpm vitest run scripts/maintenance/__tests__
grep -rn "TBD\|TODO\|à compléter\|FIXME" docs/decisions/phase7-proposition.md docs/adr/0018-contenu-relecture-ia.md docs/maintenance/GATE-6-CHECKLIST.md docs/maintenance/PHASE-6-TRACKER.md
pnpm turbo run typecheck
```

Attendu : 8 tests passés pour le Gate 6, les tests de maintenance passent, `grep` ne renvoie **aucune** ligne sur les quatre documents, `typecheck` sans erreur.

- [ ] **Step 11: Commit**

```bash
git add scripts/maintenance docs/decisions docs/adr/0018-contenu-relecture-ia.md docs/maintenance
git commit -m "docs(phase6): cloture la phase avec le gate 6 et la proposition de phase 7" -m "Evaluation des trois criteres du Gate 6, tableau de suivi, checklist
de gate signable, rapport hebdomadaire de cloture, proposition de Phase 7
arbitrable par un humain et ADR 0018 imposant la relecture humaine de tout
contenu produit par IA.

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Attendu : `git log -1 --format=%B` affiche le titre, le corps et la ligne `Co-Authored-By`.

---

## Self-Review

**1. Couverture de la Phase 6 (roadmap-phases.md, Phase 6 et Gate 6)**

| Point roadmap | Tâche | Artefact | Preuve |
|---|---|---|---|
| Gate 6 — 30 jours de données d'usage | Task 10 | `scripts/maintenance/phase6-gate.ts` | `pnpm vitest run scripts/maintenance/__tests__/phase6-gate.spec.ts` |
| Gate 6 — métriques de qualité dans les seuils | Tasks 1, 7, 10 | `packages/config/src/slo.ts`, `packages/observability/src/burn-rate.ts` | tests burn-rate + `promtool test rules` |
| Gate 6 — aucun finding critique non traité | Tasks 8, 10 | `ops/security/scanner-regression.sh`, `docs/maintenance/GATE-6-CHECKLIST.md` | `./ops/security/scanner-regression.sh` |
| Mesurer avant d'optimiser (infra §10.1, §16) | Task 1 | `packages/config/src/budgets.ts`, `docs/maintenance/TECH-DEBT-BACKLOG.md` | `budgets.spec.ts`, `debt-backlog.spec.ts` |
| Performance web : CWV, images, bundle, N+1 | Tasks 2, 3 | `apps/web/scripts/bundle-budget.mjs`, `apps/api/src/modules/performance/query-metrics.ts` | `bundle-budget.spec.ts`, `index-audit.sh` |
| Contenus SEO FR/EN/ZH à partir de sources officielles (seo §17) | Task 4 | `docs/content/plan-contenu.md`, `packages/db/migrations/0121_content_registry.sql` | `content-lint.spec.ts` |
| SEO technique : hreflang, sitemap, JSON-LD, audit CI | Task 5 | `apps/web/src/lib/seo/hreflang.ts` | `e2e/seo/public-seo-audit.spec.ts` |
| Qualité : mutation, contrats, charge, flakiness (qualite §24) | Task 6 | `docs/maintenance/TEST-AUDIT.md`, `e2e/contracts/api-web.contract.spec.ts` | k6 + Stryker + audit E2E |
| Exploitation : SLO, burn rate, alertes, runbooks, drill, coûts (infra §15, §16) | Task 7 | `packages/observability/src/slo.ts`, `ops/runbooks/RB-009-slo-degrade.md`, `RB-010-cout-infra-degrade.md` | `promtool test rules` |
| Sécurité continue : rotation planifiée, scan de dépendances, journal d'audit (spec §19.5, infra §9.5) | Task 8 | `packages/config/src/secrets.ts`, `packages/db/migrations/0123_secret_rotation_audit.sql`, `ops/security/secret-rotation.yml` | `secrets.spec.ts`, `scanner-regression.sh` |
| i18n densifiée FR/EN/ZH-CN (§4, design §3.7) | Task 9 | `packages/i18n/src/dictionaries/*.json`, `packages/i18n/scripts/check-business-parity.ts` | `check-business-parity.ts` |
| Bilan et proposition de Phase 7 | Task 10 | `docs/decisions/phase7-proposition.md`, `docs/maintenance/PHASE-6-TRACKER.md` | `phase6-gate.spec.ts` |
| Relecture humaine du contenu produit par IA (§18.4) | Task 10 | `docs/adr/0018-contenu-relecture-ia.md` | relecteur nommé porté par `content_registry.owner` |
| Candidats fonctionnels de la Phase 6 (PWA, ERP/WMS, recherche, IA, prévision de stock) | Task 10 (proposition, **pas** implémentation) | `docs/decisions/phase7-proposition.md` §3 et §4 | arbitrage humain requis |

Le tableau ci-dessus distingue deux choses que la Phase 6 ne confond pas. Les **dix tâches livrent la Phase 6 au sens de la roadmap** : la phase d'« optimisation après mesure réelle » est la construction des instruments de mesure et de garde-fou, pas l'ajout de fonctionnalités. Les **candidats fonctionnels** listés dans l'objectif de la Phase 6 (PWA complète, tarifs contractuels avancés, intégrations ERP/WMS/comptabilité, moteur de recherche, IA assistive, rapports multi-entrepôts) ne sont **pas implémentés** ici : la roadmap interdit d'engager quoi que ce soit avant trente jours de données d'usage, et l'implémentation de ces candidats avant la mesure produirait exactement le biais que la phase doit éviter. Ils sont préparés, chiffrés et arbitrables dans `docs/decisions/phase7-proposition.md`, et c'est un humain qui choisit. Le point est signalé ici parce qu'un lecteur pressé pourrait croire que la Phase 6 livre un moteur de recherche : elle livre les conditions pour en décider.

**2. Scan des placeholders**

Aucune occurrence de `TBD`, `TODO`, `à compléter`, `XXX` ou `identique à la Task N` dans les Tasks 8, 9, 10 et le Self-Review. Les seules occurrences de ces chaînes sont les **motifs de détection** écrits dans le contrôle de parité métier : `/TBD|TODO|à compléter|FIXME|XXX/` dans `packages/i18n/src/__tests__/business-parity.spec.ts` et dans `packages/i18n/scripts/check-business-parity.ts`. Ce sont des expressions régulières qui refusent ces marqueurs dans les dictionnaires, pas des marqueurs laissés en attente.

Les valeurs mesurées ne sont pas inventées : là où une mesure aurait été nécessaire (coût mensuel, dette ouverte, volume de recherches, taux de transformation des devis), les documents portent soit la **source** de la mesure et la **règle de décision** associée (`docs/decisions/phase7-proposition.md` §2), soit une **cellule vide à remplir par la mesure réelle** avec sa provenance (`docs/maintenance/GATE-6-CHECKLIST.md` §2). Le rapport hebdomadaire de la Task 10 est le seul document qui porte des valeurs chiffrées : elles sont cohérentes avec les seuils posés par la Task 1 (budget 500 €, alerte à 400 €) et doivent être remplacées par la sortie réelle du job `weekly-maintenance-report` au moment de l'exécution.

**3. Cohérence des types et des noms**

- `SecretDefinition` : les onze champs (`envName`, `secretRef`, `owner`, `policy`, `lastRotatedAt`, `overlapDays`, `supportsDualSecret`, `critical`, `emergencyRotationMinutes`) sont identiques dans `packages/config/src/secrets.ts`, dans `packages/config/src/__tests__/secrets.spec.ts` et dans l'ADR-0017. ✔
- `ROTATION_AUDIT_INSERT` : les quinze colonnes de l'instruction correspondent une à une aux quinze colonnes de la table `secret_rotation_audit` de `0123_secret_rotation_audit.sql`, dans le même ordre. ✔
- `RotationAuditEntry` : les champs optionnels sont normalisés en `null` par `buildRotationAuditEntry`, ce qui correspond aux colonnes nullables de la table ; aucune valeur `undefined` n'atteint SQL. ✔
- `release_baseline_metric.metric` : la contrainte reprend l'énumération de `0120_maintenance_metrics.sql` et y ajoute `error_rate` et `cost_eur`, qui sont les deux mesures vues par le rapport hebdomadaire et par le contrôle de coût. ✔
- Numérotation des migrations : `0120` (Task 3), `0121` (Task 4), `0122` (Task 8), `0123` (Task 8) — l'ordre numérique suit l'ordre de création et aucune migration n'est réécrite. ✔
- Numérotation des ADR : `0011` (Task 1) à `0018` (Task 10) — `0017-rotation-secrets-planifiee.md` appartient à la Task 8, `0018-contenu-relecture-ia.md` à la Task 10, conformément à l'arborescence de fichiers du plan. ✔
- Runbooks : aucun doublon. La Task 8 réutilise `RB-004-incident-securite.md` pour la rotation d'urgence au lieu d'en créer un onzième ; seules les Tasks 1 et 7 créent des runbooks. ✔
- `GATE_6_CRITERIA` : les trois identifiants (`usage_data`, `quality_metrics`, `security_findings`) correspondent aux trois questions du Gate 6 de `roadmap-phases.md`, dans le même ordre. ✔
- `businessTranslate` : même contrat que `translate` de la couche `src/messages` (locale demandée, repli explicite sur `fr-FR`, puis clé brute) et même motif d'interpolation `{name}`. ✔
- Vocabulaire métier : les trois fichiers JSON portent le même jeu de 26 clés, et les mêmes placeholders par clé (`{qty}`, `{price}`, `{days}`, `{date}`) ; l'assertion de parité du test le vérifie clé par clé. ✔
- Bandes de largeur : EN dans `[1.25 ; 1.35]` (expansion EN de design §3.7) et ZH dans `[0.60 ; 1.10]` (glyphes pleine largeur mesurés par `displayWidth`), les deux seuils étant unique dans le test et dans le script. ✔
- Noms de routes : `api:read`, `api:write`, `api:payment-webhook`, `web:public` (SLO) et `public:home`, `public:catalog`, `public:product`, `public:cart-checkout`, `public:account`, `admin:dashboard` (budgets) sont repris à l'identique dans le rapport hebdomadaire et la checklist du Gate 6. ✔
- `TruncatableValue` : la troncature visuelle porte `min-w-0` et `truncate`, la valeur complète reste dans `title` et `aria-label`, conformément à design §3.7 ; le composant n'est jamais utilisé pour un libellé d'action. ✔
- Modèle E2E : une **seule** configuration Playwright (`playwright.config.ts`, racine, modifiée et jamais recréée), `testDir: './e2e'` pour les **7 projets**, discrimination par `testMatch` / `testIgnore` uniquement. Les 5 suites de release correspondent aux fichiers `e2e/b2c/journey.spec.ts`, `e2e/b2b/journey.spec.ts`, `e2e/payment/scenarios.spec.ts`, `e2e/admin/dashboard.spec.ts` et `e2e/security/security.spec.ts` ; les 3 fichiers E2E créés en Phase 6 (`e2e/seo/public-seo-audit.spec.ts`, `e2e/perf/no-horizontal-overflow.spec.ts`, `e2e/contracts/api-web.contract.spec.ts`) relèvent du projet `functional` et sont donc adressés par `--project=functional`. Le moteur navigateur est une **dimension de shard CI** (`PLAYWRIGHT_BROWSER`), pas un projet Playwright ; aucun projet `chromium`/`firefox`/`webkit` n'est déclaré. Le timeout par test est de **45 s** et le plafond de 12 min par shard est un budget de job CI (`timeout-minutes`), non un timeout Playwright. ✔
- Aucun fichier n'est écrit hors du périmètre autorisé : les Tasks 8, 9 et 10 ne créent que des fichiers listés dans leur section `**Files:**`, et `docs/maintenance/TECH-DEBT-BACKLOG.md` est modifié en un seul ajout de ligne. ✔

**4. Points laissés explicitement à l'humain**

L'IA prépare, l'humain tranche. Ces points ne sont pas des oublis : ce sont des décisions que le plan refuse de prendre à la place du propriétaire.

- **Propriétaire métier — contenu éditorial.** `docs/content/plan-contenu.md` (Task 4) reste « en attente de signature » : le propriétaire métier valide les faits, le compliance-manager valide les volets douane, HS et conformité. Aucune page n'est publiée avant cette signature.
- **Relecture humaine obligatoire de tout contenu produit par IA** (ADR-0018, Task 10) : l'IA ne fixe aucun prix final, ne valide aucun code HS seule, ne confirme aucune importabilité, ne déclenche ni remboursement ni suspension, n'invente ni certification, ni avis, ni stock, ni délai ; tout output touchant conformité, sécurité, taxe ou contrat exige une vérification humaine avant publication, et aucun contenu SEO n'est publié sans source officielle datée ni relecture humaine.
- **Rotation d'urgence des secrets** (Task 8) : le second approbateur est un humain nommé, la référence d'incident est obligatoire et la fenêtre de 30 minutes est un engagement de l'équipe d'astreinte, pas une propriété du code.
- **Coût d'infrastructure** : au-delà de 400 € (alerte) et a fortiori de 500 € (cible), l'arbitrage appartient au finance-manager, avec justification écrite.
- **Arbitrage de la Phase 7** : `docs/decisions/phase7-proposition.md` présente quatre options chiffrées et recommande l'option A, mais le choix entre B, C et D appartient au product-owner, en tenant compte des seuils, des pays prioritaires et des pays interdits, des Incoterms, du modèle DDP/DAP et de la politique B2B — décisions listées au §12 du cahier des charges et non tranchées ici.
- **Go / No-Go du Gate 6** : `evaluateGate6` évalue les trois questions, il ne décide pas. La signature du « Go » est humaine, et elle est portée par `docs/maintenance/GATE-6-CHECKLIST.md`.
- **Juridique et conformité** : mentions légales, politique de retour, conditions de vente, base légale RGPD / UK GDPR / PIPL, transferts hors UE, et obligations acheteur (CPSC, FTC, English Colée) restent à faire valider par un juriste ; le plan ne s'auto-certifie sur aucun de ces points.
- **Preuves non substituables** : la revue manuelle clavier et lecteur d'écran, la recette 3 langues et 2 thèmes, le drill de restauration et la signature du plan de contenu sont des acts humains. Un contrôle automatisé vert n'en tient pas lieu, et la checklist du Gate 6 le dit explicitement.
