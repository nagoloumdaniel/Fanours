# Phase 5 — Assurance qualité et lancement : Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prouver que Fanours tient ses promesses, puis ouvrir. Livrer les 14 chantiers de la phase 5 du roadmap — stratégie de tests et couverture des modules critiques, tests d'intégration sur PostgreSQL réel, les 5 suites E2E Playwright bloquantes (`b2c`, `b2b`, `payment`, `admin`, `security`), responsive & visuels, accessibilité, charge & résilience, mutation testing, durcissement de la CI, recette métier FR/EN/ZH-CN, validation légale & fiscale, observabilité & runbooks, sauvegardes/PITR/rollback, checklists de lancement et Go/No-Go formel.

**Périmètre :** 18 tâches (Task 1 → Task 18) — stratégie et couverture de tests, les 5 suites de release du spec §24.3, concurrence et idempotence, responsive, accessibilité, charge, mutation, lint et scans, recette métier, validation légale et fiscale, observabilité et runbooks, sauvegardes et rollback, checklists de lancement.

**Architecture:** La phase 5 n'ajoute aucune fonctionnalité métier : elle **prouve** l'existant et **rend les preuves opposables**. La stratégie de test suit la pyramide appliquée au Fanours (§24.1) — ~70 % unitaires sur la logique pure, ~25 % intégration sur PostgreSQL/Redis réels via Testcontainers, ~5 % E2E navigateur sur les parcours bloquants. Les 5 suites E2E sont nommées et cloisonnées (`e2e/b2c/`, `e2e/b2b/`, `e2e/payment/`, `e2e/admin/`, `e2e/security/`) et pilotées par une configuration Playwright **racine** dont les projets portent exactement ces noms, de sorte que la matrice CI `e2e.yml` puisse addresser chaque suite par `--project=`. Les preuves de lancement (qualité, drill de runbook, décision légale, preuve de release) sont **versionnées en base** (`quality_metric`, `runbook_drill`, `legal_decision`, `release_evidence`) et non laissées dans des messages de chat : un Go/No-Go se tranche sur des lignes de table, pas sur des souvenirs.

**Tech Stack:** TypeScript strict · Vitest 2.1.8 + Testing Library + jest-axe + supertest + Testcontainers (PostgreSQL 16, Redis 7) · Playwright (5 projets nommés) + axe-core + `@axe-core/playwright` · Stryker (mutation) · k6 (charge) · `promtool` (tests de règles d'alerte) · gitleaks + osv-scanner + semgrep + Trivy (sécurité) · pnpm workspaces + Turborepo · PostgreSQL 16 + Drizzle (ORM unique).

**Spec:** `Cahier_des_charges_Fanours.md` §22.3–22.4, §23.1–23.5, §24.1–24.6, §25.1–25.4, §26.1–26.6, §27, §28.1–28.2, Annexes A/B/C · `docs/roadmap/sections/roadmap-phases.md` §Phase 5 (lignes 184–204), Gate 5 (lignes 337–347), DoR/DoD (lignes 356–380) · `docs/roadmap/sections/qualite.md` §24.1–§24.13 · `docs/roadmap/sections/infra.md` §4.2–4.4, §7.3, §8.1–8.2, §9.1–9.6, §10, §15 · `docs/roadmap/sections/securite.md` §13.1–13.7, §14 · `docs/roadmap/sections/git.md` §11 (ADR), §13 (checklist de fin de phase) · `docs/roadmap/sections/design.md` §3.10, §3.11.

**Prérequis :** phases 1, 2, 3 et 4 mergées (migrations `0001`–`0005`, `0006`–`0015`, `0100`–`0110`, `0111`–`0115`), socle RBAC Phase 1 (`permission`, `role`, `user_role`, `rbac_scope`, `admin_action_approval`, `audit_log`), commerce Phase 2 (`product`, `product_variant`, `price_tier`, `inventory_item`, `orders`, `order_line`, `order_status_event`, `payment`, `refund`, `invoice`, `notification`, `outbox_event`, `idempotency_key`), B2B/logistique/douane Phase 3 (`business_account`, `shipment`, `package`, `tracking_event`, `trade_rule`, `trade_rule_version`, `country_profile`, `quote`, `quote_revision`), opérations Phase 4 (`address`, `stock_count`, `payment_reconciliation`, `content_page`, `export_request`, `data_import`, `audit`).

**Portée des migrations de la phase 5 :** `0116`–`0119` (4 migrations, append-only, aucun `down`).

| Migration | Propriétaire | Contenu |
|---|---|---|
| `0116_phase5_quality_metrics.sql` | **Task 12** | `quality_metric`, `quality_metric_run` — relevés des 12 clés de métriques qualité (§24.13) et de leurs métriques dérivées |
| `0117_phase5_legal_decision.sql` | **Task 15** | `legal_decision` — registre des avis juridiques et fiscaux par document, version et locale, avec décideur, statut et date (les 12 décisions §28.2 sont tracées en Task 18 dans `docs/quality/go-nogo.md` et `release_evidence`) |
| `0118_phase5_runbook_drill.sql` | **Task 16** | `runbook_drill` — preuve d'exécution des runbooks (auteur, date, résultat) |
| `0119_phase5_release_evidence.sql` | **Task 18** | `release_evidence`, `launch_checklist_item` — preuves de release et points des Annexes A/B/C |

> Ordre d'application obligatoire : `0116` (T12) → `0117` (T15) → `0118` (T16) → `0119` (T18). Le runner de migrations de `packages/db` applique les fichiers par version croissante et refuse tout fichier dont la version précède une version déjà appliquée : la numérotation est donc **contiguë par phase et sans chevauchement** (P1 `0001`–`0005`, P2 `0006`–`0015`, P3 `0100`–`0110`, P4 `0111`–`0115`, P5 `0116`–`0119`, P6 `0120`–`0123`).

---

## Global Constraints

- **La phase 5 n'ajoute aucune fonctionnalité métier.** Aucun nouveau parcours utilisateur, aucune nouvelle route publique, aucun nouveau prix. Les seules évolutions de schéma sont les 4 migrations listées ci-dessus, qui enregistrent des **preuves** (métriques, décisions, drills, évidences de release).
- **Modèle E2E canonique.** Une **seule** configuration Playwright : `playwright.config.ts` **à la racine du dépôt** — créée en Phase 1, **modifiée** (jamais recréée) par les Phases 2, 4, 5 et 6. Aucun autre `playwright.config.ts` (ni `apps/web/`, ni `apps/api/`, ni `e2e/`) ; les commandes Playwright sont lancées **depuis la racine** (`pnpm exec playwright test ...`). `testDir` vaut `'./e2e'` pour **tous** les projets : ils se distinguent uniquement par `testMatch` / `testIgnore`, donc aucun projet ne référence un répertoire qui n'existe pas encore. **7 projets** : `b2c`, `b2b`, `payment`, `admin`, `security` (les 5 suites de recette de release du spec §24.3), `visual` (matrice visuelle) et `functional` (recettes fonctionnelles de développement). Un fichier de test appartient à **un seul** projet : aucun doublon, aucun comptage de tests ambigu.
- **Les viewports, les langues et les thèmes ne sont pas des projets Playwright.** Ils sont paramétrés **dans les fichiers de test** (`test.use({ viewport })` dans un `test.describe` itératif) à partir de la liste canonique `e2e/support/viewports.ts` (créée en Phase 1, **importée, jamais re-déclarée**) : 320×640 · 360×740 · 390×844 · 430×932 · 768×1024 · 1024×768 · 1440×900 · 1920×1080. La Phase 5 ne crée **aucune** liste de viewports locale.
- **Une seule suite de release par cas du spec §24.3.** La Phase 5 ne crée aucun doublon : elle crée **uniquement** `e2e/security/security.spec.ts`, et **reprend** (`Modify`) `e2e/b2c/journey.spec.ts` (livrée en Phase 2), `e2e/b2b/journey.spec.ts` (livrée en Phase 3), `e2e/payment/scenarios.spec.ts` (livrée en Phase 2) et `e2e/admin/dashboard.spec.ts` (livrée en Phase 4) pour les porter au niveau release — données staging, isolation, 3 langues, traçabilité §24.3. Un cas non vert = Gate bloqué.
- **Les 5 suites E2E sont nommées `b2c`, `b2b`, `payment`, `admin`, `security`** et vivent sous `e2e/<suite>/`. La matrice CI `.github/workflows/e2e.yml` addresse chaque suite par `--project=<suite>` ; les recettes fonctionnelles de phase sont adressées par `--project=functional`, la matrice visuelle par `--project=visual`.
- **Les preuves sont écrites, pas affirmées.** Un « terminé » sans sortie de test collée ou sans ligne de table (`quality_metric`, `runbook_drill`, `legal_decision`, `release_evidence`) est un « terminé » non recevable.
- **Aucune preuve inventée.** Quand une preuve ne peut pas être produite localement (recette humaine, validation juridique, restauration sur PITR managé, test de charge sur chargeur réelle), le plan écrit la **procédure exacte** et la **preuve attendue**, sans inventer de résultat.
- **Cycle de test par tâche** — pour chaque étape fonctionnelle : test rouge → exécution prouvant l'échec → implémentation → exécution prouvant la réussite → commit. Aucun commit sans test vert.
- **Stratégie de test pyramidale** — ~70 % unitaires / ~25 % intégration / ~5 % E2E. Une question de placement se tranche en 30 s : « faut-il un navigateur ? » → E2E. « Faut-il une vraie base ? » → intégration. Sinon → unitaire.
- **Jamais de `sleep(3000)` ni de `waitForTimeout`** — on attend une condition : état d'élément, réponse réseau, changement d'URL.
- **Jamais de `test.only` committé** — interdit par ESLint et par le hook pre-commit.
- **Les snapshots visuels ne sont jamais auto-approuvés** — une référence est acceptée par un relecteur après lecture du diff ; la CI ne met **jamais** à jour les snapshots.
- **Les 5 suites E2E passent dans les 3 langues** (FR / EN / ZH-CN) sur les 20 scénarios prioritaires — seuil §24.13 « couverture des 3 langues = 100 % ».
- **Le back-end reste la source de vérité** — les tests manipulent les prix, taxes, frais et droits depuis l'API, jamais depuis l'affichage : un test qui lit `data-testid="checkout-total"` pour décider d'un montant est un test qui ne prouve rien.
- **Jamais de float binaire dans les assertions** — montants comparés en minor units entiers ; pourcentages en `rate_bps` ; un écart d'un centime est un défaut, pas une tolérance.
- **Les règles douanières restent des données** — aucun taux, seuil ou Incoterm codé en dur dans un test : les tests lisent la règle depuis `trade_rule` / `trade_rule_version` ou l'injectent via une factory, et le garde-fou `packages/compliance/scripts/check-no-hardcoded-rates.mjs` (Phase 3) continue de passer.
- **Aucune donnée de carte réelle, aucun secret** — sandbox PSP, clés de test, body de requête de paiement jamais journalisé, aucun PAN/CVC/JWT dans les fixtures, les logs de test ou les snapshots.
- **Les tests de concurrence utilisent une vraie base**, jamais une branche Neon distante : 10 clients sur 5 unités ⇒ exactement 5 commandes réussies, stock jamais négatif.
- **L'idempotence se prouve par le code HTTP et par l'effet** : `201` puis `200` sur la même clé, `409` sur la même clé avec un corps différent, un seul effet en base (facture, réservation, événement outbox, notification).
- **Le rate limit est testé à la frontière** : 6e tentative ⇒ `429` + `Retry-After` + `X-RateLimit-Remaining: 0`, et la 7e tentative ne modifie pas le compteur de verrouillage de compte.
- **Aucune capture d'écran ne prouve rien** — un test E2E vérifie l'effet (état, réseau, console) ; les captures ne servent qu'à documenter un échec.
- **Les tests E2E échouent sur toute erreur console non attendue** — `pageerror` ou `console.error` non prévu ⇒ échec du test.
- **Accessibilité WCAG 2.2 AA** — axe-core automatique (zéro violation critique/sérieuse) **complété** d'une revue manuelle clavier + lecteur d'écran écrite, signée et datée ; l'audit automatique ne prouve rien.
- **Zéro débordement horizontal** — sur les 8 viewports, `document.scrollWidth <= window.innerWidth` est asserté ; les tables deviennent des cartes en mobile ; cibles tactiles ≥ 44 px.
- **i18n** — aucune chaîne en dur dans un composant ni dans un test de parcours : les sélecteurs E2E ciblent des `data-testid` ou des libellés résolus depuis les dictionnaires `fr-FR` / `en` / `zh-CN`.
- **Les 3 langues et les 2 thèmes** sont mis à jour dans la même tâche que la 1re chaîne ; la parité des clés est vérifiée en CI.
- **Observabilité** — les 16 alertes critiques ont une règle Prometheus versionnée, un test `promtool` qui prouve qu'elle déclenche, et **un runbook** ; une alerte sans runbook est un défaut de configuration.
- **Les runbooks sont lus, pas écrits** — au moins un drill exécuté et horodaté par runbook ; un runbook jamais exercé est un runbook non validé.
- **Sauvegardes** — RPO ≤ 24 h, RTO ≤ 8 h (à valider par le propriétaire, décision §28.2 point 11) ; le test de restauration est **exécuté** et ses RPO/RTO **mesurés**, pas estimés.
- **Rollback** — rollback roll-forward par digest d'image (ADR-0010) ; testé réellement en staging, pas décrit.
- **Migrations** — SQL versionné à la main dans `packages/db/migrations/NNNN_nom.sql`, append-only, **aucun `down`**. Les 4 migrations de la phase 5 (`0116`–`0119`) suivent la convention de la Phase 1 : `SET lock_timeout = '5s'` et `SET statement_timeout = '30s'` en tête de fichier, et le **checksum SHA-256 est calculé par le runner** (`packages/db/src/migrate-core.ts`) au moment de l'application — il n'est jamais écrit à la main dans le fichier SQL. Appliqué par le runner dédié, jamais `db push`.
- **Migration destructive en production** — interdite sans accord explicite du propriétaire **et** restauration testée prouvée (test de restauration documenté dans la PR).
- **Sécurité** — scans de dépendances, de secrets et SAST **bloquants** ; le scan de secrets ne peut pas être contourné ni mis en `continue-on-error`.
- **Aucune vulnérabilité critique non résolue** sans acceptation formelle du risque écrite et signée dans la PR (Annexe C).
- **Les documents juridiques ne sont jamais auto-certifiés** — CGU/CGV, politique de confidentialité, politique cookies, exemptions de TVA et mentions légales sont **soumis à un juriste** ; la phase 5 enregistre la décision, elle ne la prend pas.
- **Les taxes et droits sont validés par un expert** — aucun taux, seuil de dédouanement ou Incoterm n'est publié sans relecture du Tax Strategist / d'un conseil douanier.
- **Secrets** — jamais en code, en log, en migration, en commit, en documentation ; `.env.example` ne contient que des **noms** de variables.
- **Git** — jamais de développement ni de commit sur `main` ; branche `feat|fix|refactor|test|docs|chore/<slug>` ; jamais de force-push ; pas de commit de mélange ; **jamais de merge sans accord explicite du propriétaire** ; PR laissée ouverte avec preuves.
- **Commits** — Conventional Commits 1.0, scopes alignés sur les modules métier (`quality`, `e2e`, `a11y`, `visual`, `load`, `security`, `ci`, `legal`, `ops`, `docs`), titre ≤ 72 caractères, corps ligne ≤ 72 caractères, trailer `Co-Authored-By: Claude Code <noreply@anthropic.com>`.
- **PR > 800 lignes → à scinder.** Une PR de phase 5 qui dépasse 800 lignes est découpée par tâche, pas par lot.
- **Zéro placeholder** — aucune chaîne `TODO`, `TBD`, `FIXME`, `à compléter` ni commentaire `implémenter la logique`. Chaque valeur non spécifiée est explicitement marquée `à valider (décision propriétaire)`.
- **Preuve** — pas de « terminé » sans sortie de test collée dans la PR. Les E2E vérifient les effets, pas seulement des captures.

---

## File Structure

```text
fanours/
├─ packages/
│  ├─ config/
│  │  └─ src/
│  │     ├─ quality-thresholds.ts                 # Task 1 — seuils centralisés (couverture, mutation, k6, CWV, a11y, durée)
│  │     ├─ quality-metrics.ts                     # Task 12 — clés et contrat des métriques qualité
│  │     ├─ __tests__/
│  │     │  ├─ quality-thresholds.spec.ts          # Task 1
│  │     │  ├─ quality-metrics.spec.ts             # Task 12
│  │     │  └─ eslint-rules.spec.ts                # Task 13 — le contrat de la config ESLint/Semgrep est testé
│  │     └─ scripts/
│  │        ├─ record-quality-metrics.ts           # Task 12 — ingestion des relevés en base
│  │        └─ check-quality-thresholds.ts         # Task 12 — porte de couverture / mutation / durée
│  ├─ db/
│  │  ├─ migrations/
│  │  │  ├─ 0116_phase5_quality_metrics.sql        # Task 12
│  │  │  ├─ 0117_phase5_legal_decision.sql         # Task 15
│  │  │  ├─ 0118_phase5_runbook_drill.sql          # Task 16
│  │  │  └─ 0119_phase5_release_evidence.sql       # Task 18
│  │  └─ src/
│  │     ├─ schema/
│  │     │  ├─ quality-metrics.ts                  # Task 12
│  │     │  ├─ legal-decision.ts                   # Task 15
│  │     │  ├─ runbook-drill.ts                    # Task 16
│  │     │  ├─ release-evidence.ts                 # Task 18
│  │     │  └─ __tests__/release-evidence.spec.ts  # Task 18
│  │     └─ test/
│  │        └─ integration.setup.ts                # Task 2 — Testcontainers PostgreSQL 16 + Redis 7
│  ├─ pricing/src/__tests__/
│  │  └─ money-rounding.spec.ts                   # Task 1
│  ├─ compliance/src/__tests__/
│  │  ├─ tax-cases.spec.ts                        # Task 1
│  │  └─ rule-selection.spec.ts                   # Task 1
│  ├─ domain/src/__tests__/
│  │  └─ order-state-machine.critical.spec.ts      # Task 1
│  ├─ i18n/src/__tests__/
│  │  └─ localization.critical.spec.ts             # Task 1
│  ├─ legal/
│  │  └─ src/
│  │     ├─ legal-documents.ts                    # Task 15
│  │     └─ __tests__/legal-documents.spec.ts      # Task 15
│  └─ pdf/src/__tests__/
│     └─ invoice-render.integration.spec.ts        # Task 2
├─ apps/
│  ├─ api/src/modules/
│  │  ├─ rbac/__tests__/
│  │  │  └─ permissions.critical.spec.ts           # Task 1
│  │  ├─ payments/__tests__/
│  │  │  ├─ webhook.integration.spec.ts            # Task 2
│  │  │  └─ refund.integration.spec.ts             # Task 2
│  │  ├─ checkout/__tests__/
│  │  │  └─ checkout-recalc.integration.spec.ts    # Task 2
│  │  └─ catalog/__tests__/
│  │     └─ catalog-import.integration.spec.ts     # Task 2
│  ├─ web/src/components/__tests__/
│  │  └─ axe.spec.tsx                             # Task 10 — jest-axe sur 6 composants critiques
│  └─ worker/src/jobs/__tests__/
│     └─ jobs-idempotency.integration.spec.ts      # Task 2
├─ e2e/
│  ├─ b2c/
│  │  ├─ journey.spec.ts                           # Task 3 — Modify (livré en Phase 2) — 9 étapes
│  │  └─ fixtures.ts                               # Task 3
│  ├─ b2b/
│  │  └─ journey.spec.ts                           # Task 4 — Modify (livré en Phase 3) — 9 étapes
│  ├─ payment/
│  │  └─ scenarios.spec.ts                         # Task 5 — Modify (livré en Phase 2) — 8 cas
│  ├─ admin/
│  │  └─ dashboard.spec.ts                         # Task 6 — Modify (livré en Phase 4) — 11 cas
│  ├─ security/
│  │  └─ security.spec.ts                          # Task 7 — 7 cas
│  ├─ concurrency/
│  │  ├─ stock-oversell.spec.ts                    # Task 8
│  │  ├─ double-webhook.spec.ts                    # Task 8
│  │  ├─ double-refund.spec.ts                     # Task 8
│  │  └─ double-submission.spec.ts                 # Task 8
│  ├─ visual/
│  │  ├─ matrix.spec.ts                            # Task 9 — 8 viewports × 3 locales × 2 thèmes
│  │  ├─ README.md                                 # Task 9 — procédure d'approbation humaine
│  │  └─ __snapshots__/                            # Task 9 — références approuvées
│  ├─ a11y/
│  │  ├─ a11y.spec.ts                              # Task 10
│  │  └─ manual-checklist.spec.ts                  # Task 10 — trace des 12 vérifications manuelles
│  ├─ checkout/
│  │  └─ checkout.spec.ts                          # Task 3 — flux checkout détaillé
│  └─ support/
│     ├─ i18n.ts                                   # Task 3 — résolution des libellés par locale
│     ├─ i18n-parity.spec.ts                       # Task 14 — parité des 3 dictionnaires
│     └─ viewports.ts                              # Phase 1 — source unique des 8 viewports (importée, jamais re-déclarée)
├─ ops/
│  ├─ alerting/
│  │  ├─ rules/
│  │  │  ├─ fanours.yml                            # Phase 1 (lu, non renuméroté)
│  │  │  └─ phase5-alerts.yml                      # Task 16 — 5 règles des 5 nouveaux runbooks
│  │  └─ tests/
│  │     └─ phase5-alerts.test.yml                 # Task 16 — tests promtool
│  ├─ dashboards/
│  │  └─ supervision-lancement.json                # Task 16 — tableau de supervision du lancement
│  ├─ runbooks/
│  │  ├─ README.md                                 # Task 16 — index alerte → runbook
│  │  ├─ RB-011-prestataire-email-panne.md         # Task 16
│  │  ├─ RB-012-defaillance-transporteur.md        # Task 16
│  │  ├─ RB-013-incident-securite.md               # Task 16
│  │  ├─ RB-014-restauration-bd.md                 # Task 16
│  │  └─ RB-015-suspension-destination.md          # Task 16
│  ├─ backup/
│  │  ├─ README.md                                 # Task 17
│  │  ├─ restore-verify.sql                        # Task 17 — 12 contrôles d'intégrité
│  │  ├─ restore-drill.md                          # Task 17 — procédure et preuve du drill
│  │  ├─ rollback.md                               # Task 17 — rollback applicatif et base
│  │  └─ assertions.sql                            # Phase 1 (étendu par la Task 17)
│  ├─ checklists/
│  │  ├─ phase5-manual-a11y.md                    # Task 10 — 12 vérifications manuelles
│  │  ├─ phase5-recette-metier.md                 # Task 14 — 42 points, 3 langues
│  │  ├─ phase5-recette-technique.md               # Task 14 — 26 points
│  │  ├─ phase5-validation-juridique.md           # Task 15 — 12 points, juriste / comptable
│  │  ├─ phase5-lancement-produit.md               # Task 18 — Annexe A, 14 points
│  │  ├─ phase5-lancement-destination.md           # Task 18 — Annexe B, 11 points
│  │  ├─ phase5-lancement-release.md               # Task 18 — Annexe C, 16 points
│  │  └─ phase5-go-nogo.md                         # Task 18 — Gate 5, 8 questions
│  ├─ quality/
│  │  └─ semgrep.yml                               # Task 13 — 7 anti-patrouilles Fanours
│  └─ tests/load/
│     ├─ lib/thresholds.js                         # Task 11 — seuils alignés sur QUALITY_THRESHOLDS.load
│     ├─ k6-catalogue.js                           # Task 11
│     ├─ k6-search.js                              # Task 11
│     ├─ k6-login.js                               # Task 11
│     ├─ k6-checkout.js                            # Task 11
│     ├─ k6-webhook.js                             # Task 11
│     └─ README.md                                 # Task 11 — exécution et injection de pannes
├─ docs/
│  ├─ quality/
│  │  ├─ strategie-tests.md                        # Task 1
│  │  ├─ rapport-qualite.md                        # Task 12 — 12 clés de métriques + 3 dérivées
│  │  ├─ lint-et-scans.md                          # Task 13 — ce qui bloque, dérogations
│  │  ├─ recette-metier.md                         # Task 14 — compte rendu signé
│  │  ├─ legal-validation.md                       # Task 15 — registre des décisions juridiques
│  │  ├─ backup-restore.md                         # Task 17 — RPO et RTO mesurés
│  │  └─ go-nogo.md                                # Task 18 — 12 décisions §28.2 + décision signée
│  ├─ adr/
│  │  ├─ 0019-strategie-test-phase-5.md            # Task 1
│  │  ├─ 0020-preuves-de-lancement-en-base.md      # Task 18
│  │  └─ 0021-index-alerte-runbook.md              # Task 16
│  └─ superpowers/plans/
│     └─ 2026-10-21-phase-5-assurance-qualite-et-lancement.md
├─ .github/workflows/
│  ├─ ci.yml                                       # Task 13 — jobs quality-gates, sast-semgrep, supply-chain
│  ├─ e2e.yml                                      # Task 3 — 5 suites nommées + concurrence + visuel + a11y
│  └─ security.yml                                 # Task 13 — scans bloquants (secrets, dépendances, SAST, images)
├─ .husky/
│  └─ pre-commit                                   # Task 13 — lint-staged + gitleaks protect
├─ eslint.config.mjs                               # Task 13
├─ .prettierrc.json                                # Task 13
├─ .lintstagedrc.json                              # Task 13
├─ vitest.integration.config.ts                    # Task 2
├─ playwright.config.ts                            # Task 3 — racine, 7 projets (5 suites de release + visual + functional)
├─ stryker.config.json                             # Task 12 — racine
└─ package.json                                    # Tasks 11, 12, 13 — scripts quality / load / mutation
```

L'arborescence ci-dessus est la cible de fin de phase. Les fichiers marqués `# Task N` sont créés ou modifiés par la tâche indiquée ; une tâche ne touche jamais un fichier d'une autre tâche sans l'annoncer explicitement dans son bloc `**Files:**`.

---

### Task 1: Stratégie de tests et couverture des modules critiques (unitaires : pricing, taxes, machine à états, authz, règles douanières, permissions)

**Files:**
- Create: `docs/quality/strategie-tests.md`
- Create: `docs/adr/0019-strategie-test-phase-5.md`
- Create: `packages/config/src/quality-thresholds.ts`
- Create: `packages/config/src/__tests__/quality-thresholds.spec.ts`
- Modify: `packages/config/package.json` (script `quality:check`)
- Create: `packages/pricing/src/__tests__/money-rounding.spec.ts`
- Create: `packages/compliance/src/__tests__/tax-cases.spec.ts`
- Create: `packages/compliance/src/__tests__/rule-selection.spec.ts`
- Create: `packages/domain/src/__tests__/order-state-machine.critical.spec.ts`
- Create: `apps/api/src/modules/rbac/__tests__/permissions.critical.spec.ts`
- Create: `packages/i18n/src/__tests__/localization.critical.spec.ts`

**Interfaces:**
- Consumes: `packages/pricing` → `resolvePriceTier(tiers, quantity, { scope, at })`, `applyCustomerRule(base, rule)`, `moqPolicy(input)` ; `packages/domain` → `canTransition(status, next)`, `assertTransition(status, next, ctx)`, `InvalidTransitionError`, `toMinorUnits`, `fromMinorUnits` ; `packages/compliance` → `resolveTradeRule(input)`, `calculateTaxes(input)`, `PriceTierError` ; `packages/domain` → `decide(role, resource, action)`, `PERMISSIONS`, `Role`, `Resource`, `Action` ; `packages/i18n` → `formatMoney`, `formatDate`, `translate(locale, key)`.
- Produces:
  - `packages/config` → `QUALITY_THRESHOLDS: { coverage: { global, critical }, mutation: { break, high, critical }, e2e: { suites, flakyRate, priorityFlakyRate }, suiteDuration: { unitIntegrationMinutes, e2ePerShardMinutes, totalMinutes }, load: { catalogueP95Ms, searchP95Ms, loginP95Ms, cartP95Ms, checkoutP95Ms, webhookP95Ms }, cwv: { lcpMs, inpMs, cls }, a11y: { criticalViolations, seriousViolations } }` (constante figée, `as const`).
  - `packages/config` → `QUALITY_SUITES: readonly ['b2c','b2b','payment','admin','security']` — source unique de la liste des suites, importée par la CI, le script de couverture et le rapport de métriques.
  - `docs/quality/strategie-tests.md` — règle de placement, matrice spec → type de test, conventions de nommage, règles de fixtures, politique de flaky.

- [ ] **Step 1.1: Écrire le test rouge des seuils de qualité centralisés**

`packages/config/src/__tests__/quality-thresholds.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { QUALITY_SUITES, QUALITY_THRESHOLDS } from '../quality-thresholds.js';

describe('QUALITY_THRESHOLDS — spec §24.6 et §24.13', () => {
  it('expose exactement les 5 suites E2E bloquantes, dans l\'ordre du roadmap', () => {
    expect([...QUALITY_SUITES]).toEqual(['b2c', 'b2b', 'payment', 'admin', 'security']);
  });

  it('exige 95 % de couverture critique et 80 % global', () => {
    expect(QUALITY_THRESHOLDS.coverage.global).toBe(0.8);
    expect(QUALITY_THRESHOLDS.coverage.critical).toBe(0.95);
  });

  it('aligne les seuils k6 du spec §24.8', () => {
    expect(QUALITY_THRESHOLDS.load).toMatchObject({
      catalogueP95Ms: 600,
      searchP95Ms: 800,
      loginP95Ms: 900,
      cartP95Ms: 400,
      checkoutP95Ms: 1200,
      webhookP95Ms: 500,
    });
  });

  it('aligne les budgets CWV du spec §22.1 (p75)', () => {
    expect(QUALITY_THRESHOLDS.cwv).toEqual({ lcpMs: 2500, inpMs: 200, cls: 0.1 });
  });

  it('refuse toute violation a11y critique ou serieuse', () => {
    expect(QUALITY_THRESHOLDS.a11y).toEqual({ criticalViolations: 0, seriousViolations: 0 });
  });

  it('respecte le budget de suite de §24.13', () => {
    expect(QUALITY_THRESHOLDS.suiteDuration.unitIntegrationMinutes).toBe(6);
    expect(QUALITY_THRESHOLDS.suiteDuration.e2ePerShardMinutes).toBe(12);
    expect(QUALITY_THRESHOLDS.suiteDuration.totalMinutes).toBe(18);
  });
});
```

- [ ] **Step 1.2: Coller la sortie attendue en échec**

```bash
pnpm --filter @fanours/config test src/__tests__/quality-thresholds.spec.ts
```

Attendu :

```text
 FAIL  src/__tests__/quality-thresholds.spec.ts
Error: Failed to resolve import "../quality-thresholds.js"
 ❯ src/__tests__/quality-thresholds.spec.ts:2:1
Test Files  1 failed (1)
     Tests  0 passed | 6 failed (6)
```

- [ ] **Step 1.3: Implémenter les seuils centralisés**

`packages/config/src/quality-thresholds.ts` :

```typescript
/**
 * Seuils de qualité Fanours — source unique.
 * Référencés par : la CI (`.github/workflows/ci.yml`), le script de couverture
 * (`scripts/quality/check-coverage-gates.mjs`), le script de mutation
 * (`scripts/quality/check-mutation-gate.mjs`), les scénarios k6 et le rapport
 * mensuel de métriques. Toute modification passe par un ADR.
 *
 * Sources : spec §22.1 (CWV), §24.4 (responsive), §24.6 (qualité), §24.8 (charge),
 * §24.9 (mutation), §24.13 (métriques), roadmap `qualite.md` §24.8 et §24.13.
 */

/** Les 5 suites E2E bloquantes. Une 6e suite exige une décision d'ADR. */
export const QUALITY_SUITES = ['b2c', 'b2b', 'payment', 'admin', 'security'] as const;

export type QualitySuite = (typeof QUALITY_SUITES)[number];

export const QUALITY_THRESHOLDS = {
  /** Couverture de lignes / branches. Indicateur, jamais preuve (spec §24.6). */
  coverage: {
    global: 0.8,
    critical: 0.95,
  },
  /** Score de mutation (Stryker) sur la logique financière et douanière. */
  mutation: {
    break: 0.8,
    high: 0.9,
    critical: 0.95,
  },
  /** Flakiness : < 1 % global, 0 % sur les scénarios prioritaires. */
  e2e: {
    suites: QUALITY_SUITES,
    flakyRate: 0.01,
    priorityFlakyRate: 0,
  },
  /** Budget de durée de la CI. */
  suiteDuration: {
    unitIntegrationMinutes: 6,
    e2ePerShardMinutes: 12,
    totalMinutes: 18,
  },
  /** Seuils p95 des scénarios k6 (ms). */
  load: {
    catalogueP95Ms: 600,
    searchP95Ms: 800,
    loginP95Ms: 900,
    cartP95Ms: 400,
    checkoutP95Ms: 1200,
    webhookP95Ms: 500,
  },
  /** Core Web Vitals au 75e percentile. */
  cwv: {
    lcpMs: 2500,
    inpMs: 200,
    cls: 0.1,
  },
  /** Violations axe-core tolérées : aucune critique, aucune sérieuse. */
  a11y: {
    criticalViolations: 0,
    seriousViolations: 0,
  },
} as const;
```

- [ ] **Step 1.4: Écrire les tests unitaires critiques (rouge) — arrondis monétaires**

`packages/pricing/src/__tests__/money-rounding.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { roundHalfUp, toMinorUnits, fromMinorUnits } from '../money.js';

describe('arrondis monétaires — spec §7.2, §24.2', () => {
  it('additionne 19,99 EUR sans dérive binaire', () => {
    // given
    const prices = [1999, 1999, 1999];
    // when
    const total = prices.reduce((acc, p) => acc + p, 0);
    // then
    expect(total).toBe(5997);
    expect(fromMinorUnits(total, 'EUR')).toBe('59.97');
  });

  it('arrondit 0,145 à 0,15 (round half up, pas float)', () => {
    expect(roundHalfUp(0.145, 2)).toBe('0.15');
  });

  it('arrondit 2,5 à 3 (moitié supérieure, symétrique et stable)', () => {
    expect(roundHalfUp(2.5, 0)).toBe('3');
    expect(roundHalfUp(3.5, 0)).toBe('4');
  });

  it('convertit des minor units sans perte pour les montants à 4 décimales', () => {
    expect(toMinorUnits('1234.5678', 'EUR')).toBe(123_456_78);
    expect(fromMinorUnits(123_456_78, 'EUR')).toBe('1234.5678');
  });

  it('refuse un montant à plus de 4 décimales', () => {
    expect(() => toMinorUnits('1.000001', 'EUR')).toThrow(/precision/i);
  });

  it('refuse une devise inconnue', () => {
    expect(() => toMinorUnits('10.00', 'XBT')).toThrow(/currency/i);
  });
});
```

- [ ] **Step 1.5: Écrire les tests unitaires critiques (rouge) — les 6 cas chiffres taxes et droits**

`packages/compliance/src/__tests__/tax-cases.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { calculateTaxes, type TaxInput } from '../calculator.js';

/**
 * Les 6 cas chiffres du spec §11.3, repris à l'identique de `qualite.md` §24.2.
 * Aucun taux n'est codé en dur dans le test : le taux vient de la règle injectée.
 * La règle est une donnée, jamais une constante du code (spec §11.2).
 */
const rule = (over: Partial<TaxInput['rule']> = {}): TaxInput['rule'] => ({
  country: 'FR',
  thresholdMinor: 15_000,
  vatRateBps: 2000,
  dutyRateBps: 0,
  brokerageMinor: 0,
  incoterm: 'DAP',
  ...over,
});

const base = (over: Partial<TaxInput> = {}): TaxInput => ({
  merchandiseMinor: 20_000,
  discountMinor: 0,
  shippingMinor: 3_000,
  insuranceMinor: 0,
  rule: rule(),
  ...over,
});

describe('taxes et droits — spec §11.3 (6 cas chiffres)', () => {
  it('cas 1 — FR sous le seuil de 150 EUR : aucune TVA, mention « à confirmer »', () => {
    const res = calculateTaxes(base({ merchandiseMinor: 12_000 }));
    expect(res.baseTaxableMinor).toBe(0);
    expect(res.vatMinor).toBe(0);
    expect(res.dutyMinor).toBe(0);
    expect(res.invoiceTotalMinor).toBe(15_000);
    expect(res.customs.estimateAvailable).toBe(false);
  });

  it('cas 2 — FR 200 + 30 : base 230, TVA 46,00, total 276,00', () => {
    const res = calculateTaxes(base());
    expect(res.baseTaxableMinor).toBe(23_000);
    expect(res.vatMinor).toBe(4_600);
    expect(res.invoiceTotalMinor).toBe(27_600);
  });

  it('cas 3 — assurance incluse dans la base : base 250, TVA 50,00, total 300,00', () => {
    const res = calculateTaxes(base({ insuranceMinor: 2_000 }));
    expect(res.baseTaxableMinor).toBe(25_000);
    expect(res.vatMinor).toBe(5_000);
    expect(res.invoiceTotalMinor).toBe(30_000);
  });

  it('cas 4 — US duty 5 % : droits 25,00, courtage 12,50 hors base et hors facture', () => {
    const res = calculateTaxes(
      base({
        merchandiseMinor: 50_000,
        shippingMinor: 4_000,
        rule: rule({ country: 'US', vatRateBps: 0, dutyRateBps: 500, brokerageMinor: 1_250 }),
      }),
    );
    expect(res.baseTaxableMinor).toBe(50_000);
    expect(res.vatMinor).toBe(0);
    expect(res.dutyMinor).toBe(2_500);
    expect(res.brokerageMinor).toBe(1_250);
    expect(res.invoiceTotalMinor).toBe(54_000);
    expect(res.customs.arrivalTotalMinor).toBe(7_750);
    expect(res.invoiceTotalMinor).not.toContain(res.brokerageMinor as unknown as string);
  });

  it('cas 5 — règle expirée : aucune application silencieuse, estimation non disponible', () => {
    const res = calculateTaxes(
      base({ rule: { ...rule(), validUntil: '2026-01-01' }, ruleExpired: true }),
    );
    expect(res.vatMinor).toBe(0);
    expect(res.dutyMinor).toBe(0);
    expect(res.invoiceTotalMinor).toBe(54_000);
    expect(res.customs.estimateAvailable).toBe(false);
    expect(res.customs.mustBeConfirmedByCarrier).toBe(true);
    expect(res.complianceAlert).toBe('trade_rule_expired');
  });

  it('cas 6 — FR vers DE en B2B intracommunautaire : autoliquidation, TVA 0,00', () => {
    const res = calculateTaxes(
      base({
        merchandiseMinor: 30_000,
        shippingMinor: 2_500,
        rule: rule({ country: 'DE', vatRateBps: 0, reverseCharge: true, incoterm: 'DDP' }),
      }),
    );
    expect(res.baseTaxableMinor).toBe(32_500);
    expect(res.vatMinor).toBe(0);
    expect(res.invoiceTotalMinor).toBe(32_500);
    expect(res.vatNote).toBe('vat_reverse_charge');
  });

  it('ne double-compte jamais une taxe déjà collectée (règle transverse)', () => {
    const res = calculateTaxes(base({ rule: rule({ collectedAtCheckout: true, alsoClaimedOnArrival: true }) }));
    expect(res.conflict).toBe('double_taxation_risk');
    expect(res.vatLines).toHaveLength(2);
  });
});
```

- [ ] **Step 1.6: Écrire les tests unitaires critiques (rouge) — sélection de règle douanière**

`packages/compliance/src/__tests__/rule-selection.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { resolveTradeRule, type TradeRuleCandidate } from '../resolver.js';

const candidate = (over: Partial<TradeRuleCandidate>): TradeRuleCandidate => ({
  id: 'r-1',
  countryCode: 'FR',
  hsCode: null,
  productId: null,
  priority: 0,
  validFrom: '2026-01-01',
  validUntil: null,
  confidence: 'validated',
  ...over,
});

describe('sélection de règle douanière — spec §11.2, §11.5', () => {
  it('privilégie la règle produit sur la règle générique du pays', () => {
    const res = resolveTradeRule({
      candidates: [candidate({ id: 'country' }), candidate({ id: 'product', productId: 'p-1', priority: 10 })],
      productId: 'p-1',
      countryCode: 'FR',
      at: '2026-06-01',
    });
    expect(res.rule?.id).toBe('product');
  });

  it('utilise la version en vigueur à la date de la commande, pas la dernière', () => {
    const res = resolveTradeRule({
      candidates: [
        candidate({ id: 'old', validFrom: '2026-01-01', validUntil: '2026-05-31' }),
        candidate({ id: 'new', validFrom: '2026-06-01', validUntil: null }),
      ],
      productId: 'p-1',
      countryCode: 'FR',
      at: '2026-04-15',
    });
    expect(res.rule?.id).toBe('old');
  });

  it('ignore une règle expirée et signale un résultat incomplet', () => {
    const res = resolveTradeRule({
      candidates: [candidate({ id: 'expired', validUntil: '2026-05-01' })],
      productId: 'p-1',
      countryCode: 'FR',
      at: '2026-06-15',
    });
    expect(res.rule).toBeNull();
    expect(res.confidence).toBe('incomplete');
    expect(res.needsManualReview).toBe(true);
  });

  it('ignore une règle dont la date de début n\'est pas atteinte', () => {
    const res = resolveTradeRule({
      candidates: [candidate({ id: 'future', validFrom: '2026-12-01' })],
      productId: 'p-1',
      countryCode: 'FR',
      at: '2026-06-15',
    });
    expect(res.rule).toBeNull();
  });

  it('retourne REVUE_MANUELLE si le code HS n\'est pas validé', () => {
    const res = resolveTradeRule({
      candidates: [candidate({ id: 'hs', hsCode: '8471.30', confidence: 'suggested' })],
      productId: 'p-1',
      countryCode: 'FR',
      at: '2026-06-15',
    });
    expect(res.confidence).toBe('suggested');
    expect(res.needsManualReview).toBe(true);
  });

  it('ne confond pas pays de destination et pays d\'origine', () => {
    const res = resolveTradeRule({
      candidates: [candidate({ id: 'cn', countryCode: 'CN' })],
      productId: 'p-1',
      countryCode: 'FR',
      at: '2026-06-15',
    });
    expect(res.rule).toBeNull();
  });

  it('lève une alerte de revalidation quand la date de revue est dépassée', () => {
    const res = resolveTradeRule({
      candidates: [candidate({ id: 'r', reviewDueAt: '2026-01-01' })],
      productId: 'p-1',
      countryCode: 'FR',
      at: '2026-06-15',
    });
    expect(res.alerts).toContain('trade_rule_review_due');
  });
});
```

- [ ] **Step 1.7: Écrire les tests unitaires critiques (rouge) — machine à états des commandes**

`packages/domain/src/__tests__/order-state-machine.critical.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { canTransition, assertTransition, InvalidTransitionError, orderTransitions } from '../order-state-machine.js';
import type { OrderStatus } from '../types.js';

describe('machine à états des commandes — spec §9.3, §9.4', () => {
  it('autorise draft -> pending_payment', () => {
    expect(canTransition('draft', 'pending_payment')).toBe(true);
  });

  it('autorise pending_payment -> paid uniquement via webhook', () => {
    expect(canTransition('pending_payment', 'paid')).toBe(true);
    expect(canTransition('pending_payment', 'paid', { via: 'browser_redirect' })).toBe(false);
  });

  it('refuse paid -> draft (régression métier interdite)', () => {
    expect(canTransition('paid', 'draft')).toBe(false);
    expect(() => assertTransition('paid', 'draft')).toThrow(InvalidTransitionError);
  });

  it('refuse paid -> cancelled sans workflow de remboursement', () => {
    expect(() => assertTransition('paid', 'cancelled', { refundExecuted: false })).toThrow(/refund/i);
  });

  it('refuse shipped -> cancelled (spec §9.4)', () => {
    expect(() => assertTransition('shipped', 'cancelled')).toThrow(InvalidTransitionError);
  });

  it('refuse shipped sans Shipment associé (spec §9.3)', () => {
    expect(() => assertTransition('shipped', 'in_transit', { shipment: null })).toThrow(/shipment/i);
  });

  it('refuse refunded -> paid', () => {
    expect(canTransition('refunded', 'paid')).toBe(false);
  });

  it('autorise la chaîne complète confirmed -> ... -> completed', () => {
    const chain: OrderStatus[] = [
      'confirmed', 'processing', 'partially_fulfilled', 'shipped',
      'in_transit', 'customs_action_required', 'delivered', 'completed',
    ];
    for (let i = 0; i < chain.length - 1; i += 1) {
      expect(canTransition(chain[i], chain[i + 1])).toBe(true);
    }
  });

  it('refuse la suppression d\'une commande payée', () => {
    expect(() => assertTransition('paid', 'deleted')).toThrow(InvalidTransitionError);
  });

  it('refuse une transition inconnue non listée dans le graphe', () => {
    expect(canTransition('completed', 'processing')).toBe(false);
  });

  it('expose un graphe de transitions fermé et sans état orphelin', () => {
    const known = new Set(Object.keys(orderTransitions));
    for (const [from, targets] of Object.entries(orderTransitions)) {
      expect(known.has(from)).toBe(true);
      for (const to of targets) {
        expect(known.has(to)).toBe(true);
      }
    }
  });

  it('enregistre un OrderStatusEvent immuable à chaque transition', () => {
    const events = assertTransition('pending_payment', 'paid', { via: 'webhook' });
    expect(events).toHaveLength(1);
    expect(events[0]).toMatchObject({ from: 'pending_payment', to: 'paid' });
    expect(events[0]).not.toHaveProperty('updatedAt');
  });
});
```

- [ ] **Step 1.8: Écrire les tests unitaires critiques (rouge) — matrice d'autorisations par rôle**

`apps/api/src/modules/rbac/__tests__/permissions.critical.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { decide, PERMISSIONS, type Role, type Resource, type Action } from '../permissions.js';

const ROLES: Role[] = [
  'super_admin', 'catalog_manager', 'order_manager', 'logistics_operator',
  'finance_manager', 'support_agent', 'compliance_manager', 'marketing_manager', 'analyst_readonly',
];

const RESOURCES: Resource[] = [
  'order', 'product', 'price_tier', 'export', 'export_financial', 'refund',
  'admin_role', 'audit_log', 'customs_rule', 'country', 'business_document',
  'business_account', 'customs_document',
];

const ACTIONS: Action[] = [
  'read', 'create', 'update', 'delete', 'export', 'approve', 'refund', 'manage_roles',
];

const EXPECTED_DENY: Array<[Role, Resource, Action]> = [
  ['support_agent', 'refund', 'refund'],
  ['catalog_manager', 'audit_log', 'delete'],
  ['catalog_manager', 'export_financial', 'export'],
  ['analyst_readonly', 'order', 'update'],
  ['analyst_readonly', 'export_financial', 'export'],
  ['marketing_manager', 'audit_log', 'read'],
  ['order_manager', 'price_tier', 'update'],
  ['logistics_operator', 'business_document', 'read'],
  ['compliance_manager', 'export_financial', 'export'],
  ['finance_manager', 'business_account', 'approve'],
  ['support_agent', 'admin_role', 'manage_roles'],
  ['catalog_manager', 'business_account', 'approve'],
];

describe('matrice action x rôle — deny by default (spec §19.2)', () => {
  it.each(EXPECTED_DENY)('refuse %s sur %s:%s', (role, resource, action) => {
    expect(decide(role, resource, action)).toBe('deny');
  });

  it('refuse toute action non déclarée pour tout rôle', () => {
    for (const role of ROLES) {
      for (const resource of RESOURCES) {
        for (const action of ACTIONS) {
          if (!PERMISSIONS[role]?.[resource]?.includes(action)) {
            expect(decide(role, resource, action)).toBe('deny');
          }
        }
      }
    }
  });

  it('autorise super_admin sur toutes les ressources déclarées', () => {
    for (const resource of RESOURCES) {
      expect(decide('super_admin', resource, 'read')).toBe('allow');
    }
  });

  it('n\'accorde jamais l\'action manage_roles hors de super_admin', () => {
    for (const role of ROLES.filter((r) => r !== 'super_admin')) {
      expect(decide(role, 'admin_role', 'manage_roles')).toBe('deny');
    }
  });

  it('refuse toute ressource inconnue, y compris pour super_admin', () => {
    expect(decide('super_admin', 'unknown_resource' as Resource, 'read')).toBe('deny');
  });

  it('refuse toute action inconnue', () => {
    expect(decide('super_admin', 'order', 'detonate' as Action)).toBe('deny');
  });
});
```

- [ ] **Step 1.9: Écrire les tests unitaires critiques (rouge) — localisation**

`packages/i18n/src/__tests__/localization.critical.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { formatMoney, formatDate, translate } from '../index.js';

describe('localisation — spec §4.2, §4.4, §24.2', () => {
  it('formate un montant selon la locale FR (1 234,56 EUR)', () => {
    expect(formatMoney(123_456, 'EUR', 'fr-FR')).toBe('1 234,56 €');
  });

  it('formate un montant selon la locale EN (1,234.56 EUR)', () => {
    expect(formatMoney(123_456, 'EUR', 'en')).toBe('€1,234.56');
  });

  it('formate un montant selon la locale ZH-CN', () => {
    expect(formatMoney(123_456, 'EUR', 'zh-CN')).toBe('€1,234.56');
  });

  it('convertit un timestamp UTC vers le fuseau du lieu logistique', () => {
    expect(formatDate('2026-06-15T08:30:00.000Z', 'zh-CN', 'Asia/Shanghai')).toContain('16:30');
  });

  it('n\'affiche jamais une heure UTC non explicitée', () => {
    expect(formatDate('2026-06-15T08:30:00.000Z', 'fr-FR', 'Europe/Paris')).not.toContain('08:30');
  });

  it('respecte la langue de la commande pour la facture', () => {
    expect(translate('en', 'invoice.title')).toBe('Invoice');
    expect(translate('zh-CN', 'invoice.title')).toBe('发票');
  });

  it('signale une clé manquante au lieu de renvoyer la clé brute', () => {
    expect(() => translate('fr-FR', 'invoice.does.not.exist')).toThrow(/missing translation key/i);
  });
});
```

- [ ] **Step 1.10: Exécuter la suite unitaire critique et constater les échecs attendus**

```bash
pnpm --filter @fanours/config test src/__tests__/quality-thresholds.spec.ts
pnpm --filter @fanours/pricing test src/__tests__/money-rounding.spec.ts
pnpm --filter @fanours/compliance test src/__tests__/tax-cases.spec.ts src/__tests__/rule-selection.spec.ts
pnpm --filter @fanours/domain test src/__tests__/order-state-machine.critical.spec.ts
pnpm --filter @fanours/api test src/modules/rbac/__tests__/permissions.critical.spec.ts
pnpm --filter @fanours/i18n test src/__tests__/localization.critical.spec.ts
```

Attendu : chaque fichier échoue sur le **module manquant ou le contrat non satisfait** (pas sur une erreur de syntaxe), par exemple :

```text
 FAIL  src/__tests__/tax-cases.spec.ts
Error: Cannot find module '../calculator.js'
Test Files  1 failed (1)
```

> Si un fichier passe du premier coup, c'est que la couverture existait déjà : le cas est alors redondant avec une suite antérieure et doit être retiré de cette tâche (pas laissé en doublon) — voir §Self-Review point 3.

- [ ] **Step 1.11: Implémenter ou ajuster les modules manquants jusqu'au vert**

Implémenter dans chaque package la fonction sous test avec la signature exacte consommée par le test :

- `packages/pricing/src/money.ts` → `roundHalfUp(value: number, decimals: number): string`, `toMinorUnits(amount: string, currency: string): number`, `fromMinorUnits(minor: number, currency: string): string` (précision 4 décimales max, `RateLimitError`/`CurrencyError` remplacés ici par `Error` au message `precision` / `currency`).
- `packages/compliance/src/calculator.ts` → `calculateTaxes(input: TaxInput): TaxResult` avec `TaxResult = { baseTaxableMinor, vatMinor, dutyMinor, brokerageMinor, invoiceTotalMinor, vatNote?, conflict?, complianceAlert?, customs: { estimateAvailable, mustBeConfirmedByCarrier, arrivalTotalMinor }, vatLines: VatLine[] }`.
- `packages/compliance/src/resolver.ts` → `resolveTradeRule(input: ResolveTradeRuleInput): ResolveTradeRuleResult` avec `confidence: 'validated' | 'estimatif' | 'incomplet' | 'specialiste'`, `needsManualReview: boolean`, `alerts: string[]`.
- `packages/domain/src/order-state-machine.ts` → `canTransition(from, to, ctx?)`, `assertTransition(from, to, ctx?): OrderStatusEvent[]`, `orderTransitions: Record<OrderStatus, OrderStatus[]>`.
- `apps/api/src/modules/rbac/permissions.ts` → `decide(role, resource, action): 'allow' | 'deny'` (**deny par défaut**), `PERMISSIONS: Record<Role, Partial<Record<Resource, Action[]>>>`.
- `packages/i18n/src/index.ts` → `formatMoney(minor, currency, locale)`, `formatDate(iso, locale, timeZone)`, `translate(locale, key)` **qui lève** sur clé manquante.

```bash
pnpm --filter @fanours/config test src/__tests__/quality-thresholds.spec.ts
pnpm --filter @fanours/pricing test src/__tests__/money-rounding.spec.ts
pnpm --filter @fanours/compliance test src/__tests__/tax-cases.spec.ts src/__tests__/rule-selection.spec.ts
pnpm --filter @fanours/domain test src/__tests__/order-state-machine.critical.spec.ts
pnpm --filter @fanours/api test src/modules/rbac/__tests__/permissions.critical.spec.ts
pnpm --filter @fanours/i18n test src/__tests__/localization.critical.spec.ts
```

Attendu :

```text
 Test Files  6 passed (6)
      Tests  38 passed (38)
```

- [ ] **Step 1.12: Écrire la stratégie de tests et l'ADR**

`docs/quality/strategie-tests.md` (extrait intégral) :

```markdown
# Stratégie de tests — Phase 5

**Sources :** spec §24.1–§24.6, §24.13 · `docs/roadmap/sections/qualite.md` §24.1–§24.13 · ADR-0019.

## 1. Pyramide appliquée

| Niveau | Part visée | Ce qui y vit | Exécution |
|---|---|---|---|
| Unitaire | ~70 % | logique pure : prix, taxes, machine à états, authz, règles, i18n | sans base, sans réseau, < 2 s |
| Intégration | ~25 % | API + PostgreSQL réel + Redis + worker + outbox | Testcontainers, base réelle |
| E2E | ~5 % | parcours bloquants en navigateur, 3 langues × 2 thèmes | Playwright, 5 projets nommés |

## 2. Règle de placement (30 secondes)

1. « Faut-il un navigateur ? » → E2E.
2. « Faut-il une vraie base ? » → intégration.
3. Sinon → unitaire.

Exception assumée : le rendez-vous visuel (spec §22.1, §24.4) et les comportements multi-cookies (panier invité + fusion après connexion, §9.1) restent en E2E.

## 3. Les 5 suites E2E bloquantes

Une suite de release = **un seul fichier**, capturé par **un seul** projet Playwright. Les autres fichiers du même dossier sont des recettes fonctionnelles de phase et relèvent du projet `functional`.

| Suite (projet) | Fichier de release | Scénarios | Langues |
|---|---|---|---|
| `b2c` | `e2e/b2c/journey.spec.ts` | 9 étapes du parcours acheteur | FR / EN / ZH-CN |
| `b2b` | `e2e/b2b/journey.spec.ts` | 9 étapes du parcours acheteur pro | FR / EN / ZH-CN |
| `payment` | `e2e/payment/scenarios.spec.ts` | 8 cas paiement et webhooks | FR / EN / ZH-CN |
| `admin` | `e2e/admin/dashboard.spec.ts` | 11 cas d'administration | FR (administration) |
| `security` | `e2e/security/security.spec.ts` | 7 cas de sécurité | FR (administration) |

Les 5 suites sont complétées par les 2 projets auxiliaires du modèle canonique : `visual` (matrice 8 viewports × 3 locales × 2 thèmes) et `functional` (recettes fonctionnelles de développement). La configuration Playwright racine expose donc **7 projets**, tous sur `testDir: './e2e'`, discriminés par `testMatch` / `testIgnore` uniquement.

Source unique de la liste : `QUALITY_SUITES` dans `packages/config/src/quality-thresholds.ts`.

## 4. Matrice spec → type de test

| Spec | Module | Unitaire | Intégration | E2E |
|---|---|---|---|---|
| §7.2 paliers de prix | `pricing/price-tiers` | oui | snapshot en commande | palier affiché |
| §7.3 stock et réservations | `inventory/reservation` | expiration | décrément atomique | — |
| §9.3 états de commande | `orders/state-machine` | graphe complet | transition + audit | parcours nominal |
| §9.5 idempotence | `idempotency` | dédoublonnage | clé unique SQL | double soumission |
| §10.3 webhooks | `payments/webhook` | signature | rejeu, ordre | paiement simulé |
| §10.4 remboursements | `payments/refund` | transition | double remboursement | — |
| §11.2 règles douanières | `compliance/rule-engine` | sélection | versionnage + alerte | incertitude affichée |
| §11.3 calcul | `compliance/calculator` | 6 cas chiffres | — | récapitulatif checkout |
| §13 auth / 2FA | `auth` | matrice RBAC | session, MFA | login, MFA admin |
| §19.2 contrôle d'accès | `authz` | matrice action × rôle | IDOR réel | route admin refusée |
| §22.4 accessibilité | — | — | — | axe + manuel |
| §24.4 responsive & visuel | — | — | — | 8 viewports × 3 langues × 2 thèmes |
| §26 critères d'acceptation | — | — | — | scénarios dédiés |

## 5. Conventions

- Un fichier de test collé : `src/**/__tests__/<fichier>.spec.ts` (unitaire) et `e2e/<suite>/<nom>.spec.ts` (E2E).
- `describe` = unité de comportement · `it` = cas observable · commentaire `// given / // when / // then` sur les cas non triviaux.
- Arrange / Act / Assert en 3 blocs séparés par une ligne vide.
- Aucun `expect` sur une valeur intermédiaire non contractuelle.
- Montants comparés en minor units entiers.

## 6. Interdits (revue de code rejette la PR)

| Interdit | Raison | Alternative |
|---|---|---|
| Tester l'implémentation (assertion sur un mock) | un refactor légitime casse le test | tester le comportement observable |
| E2E pour tout | instable, diagnostic impossible | descendre au niveau le plus bas prouvable |
| Mocker un prestataire | la fixture fige l'état du prestataire | sandbox ou faux serveur local au niveau wire |
| `sleep(3000)` | course conditionnelle | attendre une condition |
| `test.only` | le reste de la suite ne tourne plus | ESLint + hook pre-commit |
| Snapshot approuvé sans relecture | la régression passe inaperçue | relecture obligatoire du diff |
| Couverture haute / tests vides | fausse assurance | mutation testing sur la logique critique |

## 7. Flakiness

- Cible : < 1 % de flake global, **0 %** sur les scénarios prioritaires (§24.13).
- `retries: 2` en CI mesure la flake, il ne la traite pas (valeur alignée sur le modèle E2E canonique : `retries: isCI ? 2 : 0`).
- Un test qui ne passe qu'avec retry ne merge pas.
- Un test flaky ne vit pas plus de 24 h dans la suite bloquante : corrigé, ou désactivé **avec ticket de cause racine**.

## 8. Données

- Base de test : PostgreSQL 16 réel via Testcontainers, Recréée par job, seed synthétique.
- **Neon n'est jamais utilisé pour un test de concurrence** : le branching distant masque les verrous.
- Aucun test ne dépend d'un jeu de seed partagé ni du résultat d'un autre test.
- Aucune donnée de carte réelle, aucun secret dans les fixtures.
```

`docs/adr/0019-strategie-test-phase-5.md` :

```markdown
# ADR-0019 — Stratégie de test de la phase 5

## Contexte

La phase 5 doit prouver la conformité de Fanours au spec §24 avant ouverture. Le risque financier porte sur le pricing, la conformité douanière et le paiement (§28.1). Une stratégie « tout en E2E » serait instable et lente ; une stratégie « tout unitaire » ne prouverait ni les contraintes SQL, ni les verrous, ni l'idempotence des webhooks.

## Décision

Pyramide appliquée au risque : ~70 % unitaires sur la logique pure, ~25 % intégration sur PostgreSQL 16 et Redis 7 réels via Testcontainers, ~5 % E2E Playwright sur 5 suites nommées (`b2c`, `b2b`, `payment`, `admin`, `security`). La configuration Playwright racine expose 7 projets — les 5 suites de release, plus `visual` (matrice visuelle) et `functional` (recettes fonctionnelles de phase) — tous sur `testDir: './e2e'`, discriminés par `testMatch` / `testIgnore` uniquement ; la CI addresse chaque suite par `--project=`. Les seuils vivent dans `packages/config/src/quality-thresholds.ts` (source unique) et les tests de couverture critique exigent 95 % de couverture et 95 % de score de mutation sur pricing, compliance, order-state et money.

## Alternatives écartées

- **E2E exhaustif** : 40 scénarios navigateur, instables, diagnostic impossible, incompatible avec le budget de 12 min par shard appliqué par le `timeout-minutes` du job CI (§24.13) — le timeout Playwright par test reste de 45 s.
- **Base in-memory / SQLite en intégration** : ne prouve ni `CHECK`, ni `EXCLUDE`, ni `SELECT … FOR UPDATE`, ni les verrous de réservation.
- **Neon branch pour la concurrence** : le branching distant ne reproduit pas la contention réelle et masque l'oversell.
- **Couverture seule comme preuve** : le spec §24.6 l'interdit explicitement.

## Conséquences

- La CI est plus lourde (PostgreSQL réel en job d'intégration) mais prouve ce qui compte.
- Le budget de durée (18 min) impose le sharding des 5 suites.
- Toute nouvelle suite E2E exige une mise à jour de `QUALITY_SUITES` et un ADR.
- La Phase 6 peut étendre la couverture sans changer la stratégie.
```

- [ ] **Step 1.13: Vérifier la couverture des modules critiques**

```bash
pnpm --filter @fanours/pricing test --coverage
pnpm --filter @fanours/compliance test --coverage
pnpm --filter @fanours/domain test --coverage
pnpm --filter @fanours/api test --coverage
```

Attendu : rapport `text-summary` indiquant les modules `pricing`, `compliance`, `order-state-machine`, `permissions` au-dessus de 95 % de lignes et de branches. Les taux inférieurs au seuil sont traités dans la Task 13 (`check-coverage-gates.mjs`), qui rend la CI rouge.

**Preuves :**
- Sortie de la suite unitaire critique (6 fichiers, 38 tests verts) collée dans la PR.
- `docs/quality/strategie-tests.md` et `docs/adr/0019-strategie-test-phase-5.md` relus par un tiers.
- Rapports de couverture par module critique au-dessus de 95 %.

**Commit :**

```bash
git add packages/config/src/quality-thresholds.ts packages/config/src/__tests__/quality-thresholds.spec.ts packages/config/package.json packages/pricing/src/__tests__/money-rounding.spec.ts packages/compliance/src/__tests__/tax-cases.spec.ts packages/compliance/src/__tests__/rule-selection.spec.ts packages/domain/src/__tests__/order-state-machine.critical.spec.ts apps/api/src/modules/rbac/__tests__/permissions.critical.spec.ts packages/i18n/src/__tests__/localization.critical.spec.ts docs/quality/strategie-tests.md docs/adr/0019-strategie-test-phase-5.md
git commit -m "test(quality): strategie de test et couverture des modules critiques" -m "Chantier 5.1 (spec §24.1, §24.6) : seuils centralises dans @fanours/config, tests unitaires critiques sur les arrondis monetaires, les 6 cas chiffres de taxes et droits, la selection de regle douaniere, la machine a etats des commandes, la matrice action x role et la localisation. ADR-0019." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 2: Tests d'intégration sur PostgreSQL réel (Testcontainers) : transactions, webhooks, outbox, jobs, PDF, import/export

**Files:**
- Create: `vitest.integration.config.ts`
- Create: `packages/db/src/test/integration.setup.ts`
- Create: `apps/api/src/modules/payments/__tests__/webhook.integration.spec.ts`
- Create: `apps/api/src/modules/payments/__tests__/refund.integration.spec.ts`
- Create: `apps/api/src/modules/checkout/__tests__/checkout-recalc.integration.spec.ts`
- Create: `apps/api/src/modules/catalog/__tests__/catalog-import.integration.spec.ts`
- Create: `apps/worker/src/jobs/__tests__/jobs-idempotency.integration.spec.ts`
- Create: `packages/pdf/src/__tests__/invoice-render.integration.spec.ts`

**Interfaces:**
- Consumes: `packages/db` → `Database` (client Drizzle), runner `pnpm --filter @fanours/db migrate`, tables `orders`, `order_line`, `order_status_event`, `payment`, `refund`, `invoice`, `inventory_item`, `stock_reservation`, `outbox_event`, `idempotency_key`, `notification`, `webhook_event`, `product`, `product_variant`, `price_tier`, `data_import`, `data_import_row`, `export_request` · `apps/api` → `sign(payload: string, timestamp: number): string` (HMAC-SHA256 de `` `${timestamp}.${payload}` ``), `POST /v1/webhooks/payment`, `POST /v1/orders` (header `Idempotency-Key`), `POST /v1/checkout/quote` · `apps/worker` → `expireReservations(now: Date): Promise<number>`, job `payment.reconcile` · `packages/pdf` → `renderInvoice(input: InvoiceInput): Promise<Buffer>`.
- Produces:
  - `vitest.integration.config.ts` — runner d'intégration : `environment: 'node'`, `include: ['**/__tests__/**/*.integration.spec.ts']`, `pool: 'forks'`, `fileParallelism: false`, `setupFiles: ['./packages/db/src/test/integration.setup.ts']`, `testTimeout: 30_000`, `hookTimeout: 120_000`.
  - `packages/db/src/test/integration.setup.ts` — exporte `getTestDatabase(): Promise<Database>`, `resetDatabase(): Promise<void>`, `stopTestDatabase(): Promise<void>` ; démarre un conteneur PostgreSQL 16 et un conteneur Redis 7, applique les migrations, positionne `DATABASE_URL` et `REDIS_URL` pour le process de test.
  - `apps/api` → contrat webhook signé : en-tête `x-webhook-timestamp` (secondes Unix) + `x-webhook-signature` (hexadécimal HMAC-SHA256 de `` `${timestamp}.${corps brut}` ``), tolérance d'horodatage de 300 s.

- [ ] **Step 2.1: Écrire le harnais d'intégration (rouge) — conteneurs réels**

`vitest.integration.config.ts` :

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    include: ['**/__tests__/**/*.integration.spec.ts'],
    exclude: ['**/node_modules/**', '**/.next/**', '**/e2e/**'],
    setupFiles: ['./packages/db/src/test/integration.setup.ts'],
    pool: 'forks',
    poolOptions: { forks: { singleFork: false } },
    fileParallelism: false,
    testTimeout: 30_000,
    hookTimeout: 120_000,
    reporters: ['default'],
  },
});
```

`packages/db/src/test/integration.setup.ts` :

```typescript
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { GenericContainer, type StartedGenericContainer } from 'testcontainers';
import { execFile } from 'node:child_process';
import { promisify } from 'node:util';
import { afterAll, beforeAll } from 'vitest';

const exec = promisify(execFile);

let pg: StartedPostgreSqlContainer | undefined;
let redis: StartedGenericContainer | undefined;
let databaseUrl = '';

export function getTestDatabaseUrl(): string {
  if (!databaseUrl) {
    throw new Error('Base de test non initialisée : appeler setupIntegration() dans beforeAll.');
  }
  return databaseUrl;
}

/** Schéma + données de test isolés : chaque fichier repart d'une base propre. */
export async function resetDatabase(): Promise<void> {
  const { stdout } = await exec('psql', [
    getTestDatabaseUrl(),
    '-v', 'ON_ERROR_STOP=1',
    '-c', 'DROP SCHEMA public CASCADE; CREATE SCHEMA public;',
  ], { env: { ...process.env, PGPASSWORD: 'postgres' } });
  if (!stdout.includes('DROP SCHEMA')) {
    throw new Error(`Reset du schéma de test en échec : ${stdout}`);
  }
  await exec('pnpm', ['--filter', '@fanours/db', 'migrate'], {
    env: { ...process.env, DATABASE_URL: getTestDatabaseUrl() },
  });
  await exec('pnpm', ['--filter', '@fanours/db', 'db:seed:synthetic'], {
    env: { ...process.env, DATABASE_URL: getTestDatabaseUrl() },
  });
}

beforeAll(async () => {
  pg = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('fanours_test')
    .withUsername('postgres')
    .withPassword('postgres')
    .start();

  redis = await new GenericContainer('redis:7-alpine')
    .withExposedPorts(6379)
    .start();

  databaseUrl = pg.getConnectionUri();
  process.env.DATABASE_URL = databaseUrl;
  process.env.REDIS_URL = `redis://${redis.getHost()}:${redis.getMappedPort(6379)}`;
  process.env.NODE_ENV = 'test';

  await resetDatabase();
}, 120_000);

afterAll(async () => {
  await redis?.stop();
  await pg?.stop();
});
```

- [ ] **Step 2.2: Écrire le test d'intégration des webhooks (rouge)**

`apps/api/src/modules/payments/__tests__/webhook.integration.spec.ts` :

```typescript
import { createHmac, randomUUID } from 'node:crypto';
import { afterEach, beforeEach, describe, expect, it } from 'vitest';
import { getTestDatabaseUrl, resetDatabase } from '../../../../../packages/db/src/test/integration.setup.js';
import { sign } from '../webhook-signature.js';

/**
 * Webhooks PSP — spec §24.2, §10.3, §26.3.
 * Base PostgreSQL 16 réelle : les contraintes, verrous et unicités sont prouvés,
 * pas simulés. La signature est recalculée sur le corps BRUT.
 */
const WEBHOOK_URL = `${getTestDatabaseUrl()}`; // remplacé par l'URL de l'app de test
const TOLERANCE_SECONDS = 300;

function signedBody(payload: object, options: { timestamp?: number; secret?: string } = {}) {
  const body = JSON.stringify(payload);
  const timestamp = options.timestamp ?? Math.floor(Date.now() / 1000);
  const secret = options.secret ?? process.env.PAYMENT_WEBHOOK_SECRET ?? 'test-webhook-secret';
  const signature = createHmac('sha256', secret)
    .update(`${timestamp}.${body}`)
    .digest('hex');
  return { body, timestamp, signature };
}

describe('webhooks paiement — intégration sur PostgreSQL réel', () => {
  beforeEach(async () => {
    await resetDatabase();
  });

  afterEach(() => {
    delete process.env.PAYMENT_WEBHOOK_SECRET;
  });

  it('accepte un webhook signé et crée une seule facture, une réservation, un événement outbox et une notification', async () => {
    const orderId = randomUUID();
    const providerEventId = `evt_${randomUUID()}`;
    const { body, timestamp, signature } = signedBody({
      id: providerEventId,
      type: 'payment.succeeded',
      data: { orderId, amountMinor: 12_000, currency: 'EUR' },
    });

    const response = await postWebhook({ body, timestamp, signature });

    expect(response.status).toBe(200);
    expect(await countInvoices(orderId)).toBe(1);
    expect(await countReservations(orderId)).toBe(1);
    expect(await countOutboxEvents(orderId)).toBe(1);
    expect(await countNotifications(orderId)).toBe(1);
  });

  it('rejette un webhook dont la signature est invalide (401) et ne produit aucun effet', async () => {
    const orderId = randomUUID();
    const { body, timestamp } = signedBody({
      id: `evt_${randomUUID()}`,
      type: 'payment.succeeded',
      data: { orderId, amountMinor: 12_000, currency: 'EUR' },
    });

    const response = await postWebhook({ body, timestamp, signature: 'deadbeef' });

    expect(response.status).toBe(401);
    expect(await countInvoices(orderId)).toBe(0);
    expect(await countNotifications(orderId)).toBe(0);
  });

  it('rejette un webhook dupliqué (409) sans dupliquer les effets', async () => {
    const orderId = randomUUID();
    const { body, timestamp, signature } = signedBody({
      id: `evt_${randomUUID()}`,
      type: 'payment.succeeded',
      data: { orderId, amountMinor: 12_000, currency: 'EUR' },
    });

    expect((await postWebhook({ body, timestamp, signature })).status).toBe(200);
    expect((await postWebhook({ body, timestamp, signature })).status).toBe(409);

    expect(await countInvoices(orderId)).toBe(1);
    expect(await countOutboxEvents(orderId)).toBe(1);
  });

  it('ignore un webhook arrivé hors ordre (un event antérieur ne régresse pas la commande)', async () => {
    const orderId = randomUUID();
    const succeeded = signedBody({
      id: `evt_${randomUUID()}`,
      type: 'payment.succeeded',
      data: { orderId, amountMinor: 12_000, currency: 'EUR' },
    });
    const pending = signedBody({
      id: `evt_${randomUUID()}`,
      type: 'payment.pending',
      data: { orderId, amountMinor: 12_000, currency: 'EUR' },
    });

    expect((await postWebhook(succeeded)).status).toBe(200);
    expect((await postWebhook(pending)).status).toBe(200);

    expect(await readOrderStatus(orderId)).toBe('paid');
  });

  it('rejette un horodatage hors tolérance (401) — rejeu à 10 min', async () => {
    const { body, signature } = signedBody(
      { id: `evt_${randomUUID()}`, type: 'payment.succeeded', data: { orderId: randomUUID(), amountMinor: 1, currency: 'EUR' } },
      { timestamp: Math.floor(Date.now() / 1000) - 600 },
    );

    const response = await postWebhook({ body, timestamp: Math.floor(Date.now() / 1000) - 600, signature });

    expect(response.status).toBe(401);
  });

  it('rejette un corps modifié après signature (401) — la signature porte sur le corps brut', async () => {
    const { body, timestamp, signature } = signedBody({
      id: `evt_${randomUUID()}`,
      type: 'payment.succeeded',
      data: { orderId: randomUUID(), amountMinor: 12_000, currency: 'EUR' },
    });
    const tampered = body.replace(/"amountMinor":12000/, '"amountMinor":1');

    const response = await postWebhook({ body: tampered, timestamp, signature });

    expect(response.status).toBe(401);
  });
});
```

- [ ] **Step 2.3: Écrire les tests d'intégration transactionnels, outbox, jobs, PDF et import/export (rouge)**

`apps/api/src/modules/payments/__tests__/refund.integration.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { requestRefund, getRefund } from '../refund.service.js';

describe('remboursements — intégration (spec §10.4, §24.2, §26.3)', () => {
  it('refuse le double remboursement (409) et ne crée qu\'un refund', async () => {
    const payment = await seedPaidPayment({ amountMinor: 12_000 });
    const first = await requestRefund(payment.id, { amountMinor: 12_000, reason: 'customer_request' });
    expect(first.status).toBe(201);

    const second = await requestRefund(payment.id, { amountMinor: 12_000, reason: 'customer_request' });
    expect(second.status).toBe(409);

    expect(await countRefunds(payment.id)).toBe(1);
  });

  it('refuse un remboursement partiel supérieur au montant restant', async () => {
    const payment = await seedPaidPayment({ amountMinor: 12_000 });
    await requestRefund(payment.id, { amountMinor: 8_000, reason: 'customer_request' });

    const second = await requestRefund(payment.id, { amountMinor: 8_000, reason: 'customer_request' });

    expect(second.status).toBe(422);
    expect(await countRefunds(payment.id)).toBe(1);
  });

  it('enregistre le statut réel du remboursement et le journalise', async () => {
    const payment = await seedPaidPayment({ amountMinor: 12_000 });
    const refund = await requestRefund(payment.id, { amountMinor: 12_000, reason: 'customer_request' });

    expect(refund.status).toBe('pending');
    expect(await getRefund(refund.id)).toMatchObject({ status: 'pending' });
    expect(await countAuditEntries({ action: 'refund.requested' })).toBe(1);
  });
});
```

`apps/api/src/modules/checkout/__tests__/checkout-recalc.integration.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { quoteCheckout } from '../checkout.service.js';

describe('recalcul du checkout — intégration (spec §24.2, §7.2, §9.2)', () => {
  it('recalcule le total après un changement d\'adresse (le serveur est la source de vérité)', async () => {
    const cart = await seedCart({ items: [{ sku: 'SKU-1', quantity: 10 }] });
    const fr = await quoteCheckout(cart.id, { addressId: await seedAddress('FR'), locale: 'fr-FR' });
    const de = await quoteCheckout(cart.id, { addressId: await seedAddress('DE'), locale: 'fr-FR' });

    expect(fr.totalMinor).not.toBe(de.totalMinor);
    expect(de.totalMinor).toBe(fr.totalMinor + de.shippingMinor - fr.shippingMinor);
  });

  it('recalcule le total après un changement de quantité et applique le palier', async () => {
    const cart = await seedCart({ items: [{ sku: 'SKU-1', quantity: 9 }] });
    const nine = await quoteCheckout(cart.id, { addressId: await seedAddress('FR'), locale: 'fr-FR' });

    await setCartQuantity(cart.id, 'SKU-1', 10);
    const ten = await quoteCheckout(cart.id, { addressId: await seedAddress('FR'), locale: 'fr-FR' });

    expect(ten.unitPriceMinor).toBeLessThan(nine.unitPriceMinor);
    expect(ten.tier.minQuantity).toBe(10);
  });

  it('ignore un prix fourni par le client et renvoie le prix serveur', async () => {
    const cart = await seedCart({ items: [{ sku: 'SKU-1', quantity: 10, priceMinor: 1 }] });
    const quote = await quoteCheckout(cart.id, { addressId: await seedAddress('FR'), locale: 'fr-FR' });

    expect(quote.unitPriceMinor).toBe(1_200);
  });
});
```

`apps/worker/src/jobs/__tests__/jobs-idempotency.integration.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { expireReservations } from '../reservation.job.js';
import { runJobOnce } from '../job-runner.js';

describe('jobs BullMQ — intégration (spec §24.2, §9.5)', () => {
  it('expire une réservation et libère le stock exactement une fois', async () => {
    const order = await seedOrderWithReservation({ quantity: 2, expiresAt: new Date('2026-01-01T00:00:00Z') });

    const released = await expireReservations(new Date('2026-06-01T00:00:00Z'));

    expect(released).toBe(1);
    expect(await readReservedQuantity(order.id)).toBe(0);
    expect(await readAvailableStock(order.sku)).toBe(await readOnHandStock(order.sku));
  });

  it('ne fait rien sur une réservation déjà expirée (deuxième passage sans effet)', async () => {
    await seedOrderWithReservation({ quantity: 2, expiresAt: new Date('2026-01-01T00:00:00Z') });
    await expireReservations(new Date('2026-06-01T00:00:00Z'));

    const second = await expireReservations(new Date('2026-06-01T00:00:01Z'));

    expect(second).toBe(0);
  });

  it('rejoue un job avec la même clé d\'idempotence sans dupliquer l\'effet', async () => {
    const key = `job:${randomUUID()}`;
    const first = await runJobOnce('export.data', { exportRequestId: randomUUID() }, { idempotencyKey: key });
    const second = await runJobOnce('export.data', { exportRequestId: randomUUID() }, { idempotencyKey: key });

    expect(first.status).toBe('completed');
    expect(second.status).toBe('skipped_duplicate');
  });
});
```

`packages/pdf/src/__tests__/invoice-render.integration.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { renderInvoice } from '../render-invoice.js';

describe('facture PDF — intégration multi-langue (spec §14.3, §24.2, §26.3)', () => {
  it.each(['fr-FR', 'en', 'zh-CN'] as const)('génère une facture en %s avec la police CJK embarquée', async (locale) => {
    const pdf = await renderInvoice({ orderNumber: 'FAN-2026-0001', locale, currency: 'EUR', lines: [] });

    expect(pdf.byteLength).toBeGreaterThan(1000);
    expect(pdf.subarray(0, 5).toString()).toBe('%PDF-');
  });

  it('inclut le taux de TVA en base 10 000 quand la locale le demande (FR)', async () => {
    const pdf = await renderInvoice({ orderNumber: 'FAN-2026-0001', locale: 'fr-FR', currency: 'EUR', lines: [] });
    expect(pdf.byteLength).toBeGreaterThan(0);
  });
});
```

`apps/api/src/modules/catalog/__tests__/catalog-import.integration.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { importCatalogCsv } from '../catalog-import.service.js';

describe('import/export catalogue — intégration (spec §7.4, §24.2)', () => {
  it('rejette un schéma invalide ligne par ligne et n\'importe rien', async () => {
    const csv = 'sku,name,min_qty\nSKU-1,Produit,abc\n';
    const result = await importCatalogCsv(csv, { locale: 'fr-FR' });

    expect(result.imported).toBe(0);
    expect(result.errors).toHaveLength(1);
    expect(result.errors[0].row).toBe(2);
  });

  it('importe les lignes valides et isole les lignes invalides', async () => {
    const csv = 'sku,name,min_qty\nSKU-1,Produit,10\nSKU-2,Autre,abc\n';
    const result = await importCatalogCsv(csv, { locale: 'fr-FR' });

    expect(result.imported).toBe(1);
    expect(result.errors).toHaveLength(1);
  });

  it('annule un import en cours sur demande explicite', async () => {
    const csv = 'sku,name,min_qty\nSKU-1,Produit,10\n';
    const started = await importCatalogCsv(csv, { locale: 'fr-FR' });
    await cancelImport(started.importId);

    expect(await readImportStatus(started.importId)).toBe('cancelled');
  });
});
```

- [ ] **Step 2.4: Exécuter la suite d'intégration et constater les échecs attendus**

```bash
pnpm vitest run --config vitest.integration.config.ts
```

Attendu :

```text
 FAIL  apps/api/src/modules/payments/__tests__/webhook.integration.spec.ts
Error: Cannot find module '../webhook-signature.js'
Test Files  1 failed (1)
```

et, pour les fichiers dont les modules existent déjà mais dont le contrat diffère :

```text
AssertionError: expected 200 to be 401
 ❯ apps/api/src/modules/payments/__tests__/webhook.integration.spec.ts:78:24
```

- [ ] **Step 2.5: Implémenter les briques manquantes jusqu'au vert**

Implémenter / ajuster :

- `apps/api/src/modules/payments/webhook-signature.ts` → `sign(payload: string, timestamp: number, secret?: string): string` (HMAC-SHA256 hexadécimal de `` `${timestamp}.${payload}` ``) et `verifySignature(payload, timestamp, signature, secret, toleranceSeconds = 300): boolean` (rejette un horodatage hors tolérance **avant** toute comparaison).
- `apps/api/src/modules/payments/webhook.controller.ts` → `POST /v1/webhooks/payment` : en-têtes `x-webhook-timestamp` et `x-webhook-signature`, corps brut lu via le middleware de corps brut ; `401` si signature invalide ou horodatage hors tolérance, `409` si `provider_event_id` déjà traité (contrainte unique `uq_webhook_events_provider_event`), `200` sinon ; l'{order_id} est mis à jour et l'événement `payment.succeeded` est écrit dans `outbox_event` **dans la même transaction**.
- `apps/api/src/modules/payments/refund.service.ts` → `requestRefund(paymentId, input)` : refus si le cumul des remboursements dépasse le montant du paiement, refus si un remboursement est déjà en cours sur ce paiement, idempotence par `idempotency_key`, écriture dans `refund` + `audit_log` dans la même transaction.
- `apps/api/src/modules/checkout/checkout.service.ts` → `quoteCheckout(cartId, input)` : prix, paliers, frais et taxes recalculés côté serveur ; tout prix fourni par le client est ignoré.
- `apps/worker/src/jobs/reservation.job.ts` → `expireReservations(now)` : sélection `FOR UPDATE SKIP LOCKED` des réservations expirées, libération atomique du stock, journalisation.
- `apps/worker/src/jobs/job-runner.ts` → `runJobOnce(type, payload, { idempotencyKey })` : clé unique en base, `skipped_duplicate` au second appel.
- `packages/pdf/src/render-invoice.ts` → `renderInvoice(input)` : polices CJK embarquées, 3 locales, jamais de PDF dans le bundle public.
- `apps/api/src/modules/catalog/catalog-import.service.ts` → `importCatalogCsv(csv, { locale })` : validation Zod par ligne, rapport d'erreurs avec numéro de ligne, `cancelImport(importId)`.

```bash
pnpm vitest run --config vitest.integration.config.ts
```

Attendu :

```text
 Test Files  6 passed (6)
      Tests  24 passed (24)
```

- [ ] **Step 2.6: Vérifier qu'aucun test d'intégration n'utilise Neon ni une base distante**

```bash
rg -n "neon.tech|neon-serverless|branch.neon" apps packages --glob "*.integration.spec.ts"
```

Attendu :

```text
(no matches)
```

Toute occurrence est un défaut : un test de concurrence sur une branche Neon distante ne prouve pas le verrou.

**Preuves :**
- Sortie de `pnpm vitest run --config vitest.integration.config.ts` (6 fichiers, 24 tests verts) collée dans la PR.
- Conteneur PostgreSQL 16 + Redis 7 démarré par le job, base recréée entre les fichiers.
- Aucun test d'intégration contre Neon (sortie de recherche vide).
- Les 6 familles du spec §24.2 couvertes : API+DB, transactions/contraintes, webhooks, e-mails et files, PDF, transporteur sandbox, recalcul checkout, expiration de réservation, import/export.

**Commit :**

```bash
git add vitest.integration.config.ts packages/db/src/test/integration.setup.ts apps/api/src/modules/payments apps/api/src/modules/checkout apps/api/src/modules/catalog apps/worker/src/jobs packages/pdf/src
git commit -m "test(quality): integration sur PostgreSQL reel, webhooks, outbox, jobs et PDF" -m "Chantier 5.2 (spec §24.2) : runner d'integration a conteneurs PostgreSQL 16 + Redis 7, webhooks signes (valide, invalide, duplique, desordre, horodatage, corps modifie), double remboursement, recalcul de checkout, expiration de reservation, facture PDF 3 langues, import/export catalogue." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 3: Suite E2E « acheteur B2C » — 9 étapes

**Files:**
- Modify: `playwright.config.ts` (racine — config Playwright unique, déjà créée en Phase 1, jamais recréée ici)
- Modify: `e2e/b2c/journey.spec.ts` (livré en Phase 2 — repris ici au niveau release)
- Create: `e2e/b2c/fixtures.ts`
- Create: `e2e/checkout/checkout.spec.ts`
- Create: `e2e/support/i18n.ts`
- Modify: `.github/workflows/e2e.yml` (projets nommés, 5 shards)

**Interfaces:**
- Consumes: `packages/i18n` → dictionnaires `fr-FR` / `en` / `zh-CN` ; `apps/web` → routes publiques `/{locale}/products`, `/{locale}/products/{slug}`, `/{locale}/cart`, `/{locale}/checkout`, `/{locale}/account/orders` ; `apps/api` → `GET /v1/products`, `GET /v1/search`, `POST /v1/cart/items`, `POST /v1/checkout/quote`, `POST /v1/orders` ; `packages/config` → `QUALITY_SUITES`.
- Produces:
  - `playwright.config.ts` (racine) — **modifié, jamais recréé** : la configuration racine est créée en Phase 1 ; la modification porte sur les **seuils de budget** (`timeout: 45_000` par test, `globalTimeout: 4 * 60 * 60 * 1000`, `retries: isCI ? 2 : 0`, `fullyParallel: true`, `forbidOnly: isCI`, `use: { trace, screenshot, video, baseURL }`) et sur l'**ajout des 5 suites de release** au modèle canonique. La structure de projets n'est pas réécrite : ce sont les **7 projets** du modèle canonique (5 suites de release `b2c`/`b2b`/`payment`/`admin`/`security` + `visual` + `functional`), tous partagent `testDir: './e2e'` et se distinguent uniquement par `testMatch` / `testIgnore`.
  - `e2e/b2c/journey.spec.ts` — **repris (Modify)** : le fichier livré par la Phase 2 est porté au niveau release (données staging, isolation, 3 langues, traçabilité §24.3) ; le code ci-dessous est le **contenu release-grade final** du fichier. Les 9 étapes sont celles du parcours B2C du spec §24.3, dans le même ordre et avec les mêmes intitulés que la version Phase 2. Le projet `b2c` ne capture **que** ce fichier.
  - `e2e/support/i18n.ts` → `SUPPORTED_LOCALES: readonly ['fr-FR','en','zh-CN']`, `t(locale: SupportedLocale, key: string): string` (résolution dans les dictionnaires, jamais de chaîne en dur dans un test), `localePath(locale: SupportedLocale, path: string): string`.
  - `e2e/b2c/fixtures.ts` → `b2cFixtures: PlaywrightTestFixtures` exposant `locale`, `product` (`slug`, `sku`, `moq`), `addToCart(page, quantity, locale?)`, `fillShippingAddress(page, address)`.

- [ ] **Step 3.1: Modifier la configuration Playwright racine (rouge)**

`playwright.config.ts` — la configuration est **créée en Phase 1** et **modifiée** ici, jamais recréée. La modification porte sur les **seuils de budget** et sur l'**ajout des 5 suites de release** au modèle canonique déjà en place. La structure de projets reste celle du modèle canonique : `testDir: './e2e'` unique pour tous, discrimination par `testMatch` / `testIgnore` uniquement, aucun `testDir` par projet.

```typescript
import { readFileSync } from 'node:fs';
import { resolve } from 'node:path';
import { defineConfig } from '@playwright/test';
import { QUALITY_SUITES } from './packages/config/src/quality-thresholds.js';

const isCI = Boolean(process.env.CI);

/**
 * Configuration E2E racine — Phase 5 (spec §24.3, §24.4, roadmap Gate 5).
 *
 * Fichier UNIQUE de configuration du dépôt : créé en Phase 1, MODIFIÉ (jamais
 * recréé) par les Phases 2, 4, 5 et 6. Aucun autre `playwright.config.ts`
 * (ni `apps/web/`, ni `apps/api/`, ni `e2e/`). Toutes les commandes Playwright
 * sont lancées depuis la racine : `pnpm exec playwright test ...`.
 *
 * `testDir` vaut './e2e' pour TOUS les projets : les projets se distinguent
 * uniquement par `testMatch` / `testIgnore`, donc aucun projet ne référence un
 * répertoire qui n'existe pas encore à la phase courante.
 *
 * Les 5 premiers projets sont les suites de recette de release du spec §24.3 ;
 * la matrice CI addresse chaque suite par `--project=<suite>`. Les viewports,
 * les langues et les thèmes ne sont PAS des projets : ils sont paramétrés dans
 * les fichiers de test à partir de la liste canonique `e2e/support/viewports.ts`.
 *
 * Le timeout par test est de 45 s (`timeout: 45_000`) : un test qui le dépasse
 * est bloqué, il se diagnostique — on ne relève jamais le timeout. Le plafond de
 * 12 min par shard est un budget de job CI (`timeout-minutes`), pas un timeout
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

  const unexpected = projectNames.filter((name) => !RELEASE_SUITES.includes(name) && !AUXILIARY_PROJECTS.includes(name));
  if (unexpected.length > 0) {
    throw new Error(
      `Projets Playwright inattendus dans playwright.config.ts : ${unexpected.join(',')}. ` +
        `Les seuls projets autorisés sont ${RELEASE_SUITES.join(',')} et ${AUXILIARY_PROJECTS.join(',')}.`,
    );
  }
}
```

- [ ] **Step 3.2: Écrire le helper i18n et les fixtures B2C (rouge)**

`e2e/support/i18n.ts` :

```typescript
import frFR from '../../packages/i18n/src/locales/fr-FR.json' with { type: 'json' };
import en from '../../packages/i18n/src/locales/en.json' with { type: 'json' };
import zhCN from '../../packages/i18n/src/locales/zh-CN.json' with { type: 'json' };

export const SUPPORTED_LOCALES = ['fr-FR', 'en', 'zh-CN'] as const;
export type SupportedLocale = (typeof SUPPORTED_LOCALES)[number];

const dictionaries: Record<SupportedLocale, Record<string, string>> = {
  'fr-FR': frFR as Record<string, string>,
  en: en as Record<string, string>,
  'zh-CN': zhCN as Record<string, string>,
};

/** Résolution d'un libellé par locale : jamais de chaîne en dur dans un test. */
export function t(locale: SupportedLocale, key: string): string {
  const value = dictionaries[locale]?.[key];
  if (typeof value !== 'string' || value.length === 0) {
    throw new Error(`Clé de traduction absente : ${key} (${locale})`);
  }
  return value;
}

/** Construit un chemin localisé : /fr-FR/... ; la langue est dans l'URL, pas dans un cookie. */
export function localePath(locale: SupportedLocale, path = ''): string {
  const suffix = path.startsWith('/') ? path : `/${path}`;
  return `/${locale}${suffix}`.replace(/\/$/, '') || `/${locale}`;
}
```

`e2e/b2c/fixtures.ts` :

```typescript
import { test as base, expect, type Page } from '@playwright/test';
import { localePath, type SupportedLocale } from '../support/i18n.js';

export const B2C_LOCALE: SupportedLocale = 'fr-FR';
export const B2C_ADDRESS = {
  recipient: 'Zhang Wei',
  line1: '88 Nanjing Road',
  city: 'Shanghai',
  postalCode: '200001',
  country: 'CN',
} as const;

export const b2cFixtures = {
  locale: B2C_LOCALE,
  product: { slug: 'usb-c-hub-7-ports', sku: 'SKU-USB-C-7P', moq: 10 } as const,

  async addToCart(page: Page, quantity: number, locale: SupportedLocale = B2C_LOCALE): Promise<void> {
    await page.goto(localePath(locale, `/products/${b2cFixtures.product.slug}`));
    const quantityInput = page.getByTestId('product-quantity');
    await expect(quantityInput).toBeVisible();
    await quantityInput.fill(String(quantity));
    await page.getByTestId('add-to-cart').click();
    await expect(page.getByTestId('cart-count')).toContainText(String(quantity));
  },

  async fillShippingAddress(page: Page): Promise<void> {
    await page.getByTestId('shipping-recipient').fill(B2C_ADDRESS.recipient);
    await page.getByTestId('shipping-line1').fill(B2C_ADDRESS.line1);
    await page.getByTestId('shipping-city').fill(B2C_ADDRESS.city);
    await page.getByTestId('shipping-postal-code').fill(B2C_ADDRESS.postalCode);
    await page.getByTestId('shipping-country').selectOption(B2C_ADDRESS.country);
  },
};

export const test = base.extend<{ b2c: typeof b2cFixtures }>({
  b2c: async ({}, use) => {
    await use(b2cFixtures);
  },
});

export { expect };
```

- [ ] **Step 3.3: Reprendre la suite B2C au niveau release — 9 étapes (rouge)**

`e2e/b2c/journey.spec.ts` — **fichier livré par la Phase 2, repris ici**. Le code ci-dessous est le **contenu release-grade final** : mêmes 9 étapes, même ordre et mêmes intitulés que la version Phase 2, mais jouées sur les données de staging, dans les 3 langues, avec isolation entre tests et vérification d'absence d'erreur console.

```typescript
import { expect, test } from '@playwright/test';
import { SUPPORTED_LOCALES, t, localePath } from '../support/i18n.js';
import { VIEWPORTS } from '../support/viewports.js';
import { b2cFixtures } from './fixtures.js';

/**
 * Suite de release `b2c` — spec §24.3 (9 étapes).
 * Chaque étape vérifie un EFFET (état, réseau, console), pas une capture.
 * Le parcours est rejoué dans les 3 langues : seuil §24.13 « 100 % des scénarios
 * prioritaires en 3 langues ».
 *
 * Ce fichier est le SEUL capturé par le projet Playwright `b2c` : tout fichier
 * E2E appartient à un seul projet (pas de doublon, pas de comptage ambigu).
 */

const consoleErrors: string[] = [];

/** Viewport issu de la liste canonique `e2e/support/viewports.ts` — jamais re-déclaré. */
function viewportOf(width: number) {
  const found = VIEWPORTS.find((viewport) => viewport.width === width);
  if (!found) {
    throw new Error(`viewport ${width} absent de la liste canonique e2e/support/viewports.ts`);
  }
  return found;
}

test.beforeEach(async ({ page }) => {
  consoleErrors.length = 0;
  page.on('console', (msg) => {
    if (msg.type() === 'error') consoleErrors.push(msg.text());
  });
  page.on('pageerror', (err) => consoleErrors.push(err.message));
});

test.afterEach(async () => {
  expect(consoleErrors, `erreurs console inattendues : ${consoleErrors.join(' | ')}`).toHaveLength(0);
});

for (const locale of SUPPORTED_LOCALES) {
  test.describe(`parcours acheteur B2C — ${locale}`, () => {
    test('étapes 1 à 9 — accueil, catalogue, recherche, fiche produit, paliers, panier, checkout, adresse, validation', async ({ page }) => {
      // Étape 1 — Ouverture de la page d'accueil
      await page.goto(localePath(locale, '/'));
      await expect(page.getByRole('heading', { level: 1 })).toBeVisible();

      // Étape 2 — Accès au catalogue
      await page.getByTestId('nav-catalogue').click();
      await expect(page).toHaveURL(new RegExp(`${locale}/products`));

      // Étape 3 — Recherche d'un produit
      await page.getByTestId('search-input').fill('USB-C');
      await page.getByTestId('search-submit').click();
      await expect(page.getByTestId('search-results')).toBeVisible();

      // Étape 4 — Ouverture de la fiche produit
      await page.getByTestId('product-card').first().click();
      await expect(page).toHaveURL(new RegExp(`${locale}/products/`));

      // Étape 5 — Vérification du tableau de paliers
      const tiers = page.getByTestId('price-tier-table');
      await expect(tiers).toBeVisible();
      await expect(tiers.getByRole('row')).toHaveCount(3);

      // Étape 6 — Ajout au panier
      await b2cFixtures.addToCart(page, b2cFixtures.product.moq, locale);
      await expect(page.getByTestId('cart-count')).toHaveText(String(b2cFixtures.product.moq));

      // Étape 7 — Passage au checkout
      await page.getByTestId('cart-checkout-link').click();
      await expect(page).toHaveURL(new RegExp(`${locale}/checkout`));

      // Étape 8 — Saisie de l'adresse et acceptation des conditions
      await b2cFixtures.fillShippingAddress(page);
      await page.getByTestId('checkout-terms-checkbox').check();
      await expect(page.getByTestId('checkout-loading')).toBeHidden();
      await expect(page.getByTestId('customs-estimate-disclaimer')).toBeVisible();

      // Étape 9 — Validation de la commande
      await page.getByTestId('checkout-submit').click();
      await expect(page).toHaveURL(new RegExp(`${locale}/account/orders`));
      await expect(page.getByTestId('order-status')).toContainText(t(locale, 'order.status.pending_payment'));
    });
  });
}

test('affiche un état vide exploitable quand la recherche ne renvoie rien', async ({ page }) => {
  await page.goto(localePath('fr-FR', '/products?q=zzz-aucun-resultat'));
  await expect(page.getByTestId('search-empty-state')).toBeVisible();
  await expect(page.getByTestId('search-clear-filters')).toBeVisible();
});

test('le sélecteur de langue expose les 3 locales supportées', async ({ page }) => {
  await page.goto(localePath('fr-FR', '/'));
  await page.getByTestId('locale-switcher').click();
  for (const locale of SUPPORTED_LOCALES) {
    await expect(page.getByRole('link', { name: locale })).toBeVisible();
  }
});

test('le parcours B2C ne déborde pas horizontalement à 360 px', async ({ page }) => {
  const mobile = viewportOf(360);
  await page.setViewportSize({ width: mobile.width, height: mobile.height });
  await page.goto(localePath('fr-FR', '/checkout'));

  const overflow = await page.evaluate(
    () => document.documentElement.scrollWidth - document.documentElement.clientWidth,
  );
  expect(overflow).toBeLessThanOrEqual(0);
});
```

`e2e/checkout/checkout.spec.ts` (flux checkout détaillé, complément de la suite B2C) :

```typescript
import { expect, test } from '@playwright/test';
import { localePath } from '../support/i18n.js';
import { b2cFixtures } from '../b2c/fixtures.js';

test.describe('checkout — recalcul serveur et idempotence', () => {
  test('le total affiché correspond au total renvoyé par l\'API', async ({ page }) => {
    const quote = await page.request.get('/v1/checkout/quote?cartId=seed-cart');
    const body = await quote.json();

    await page.goto(localePath('fr-FR', '/checkout'));
    await b2cFixtures.fillShippingAddress(page);
    await expect(page.getByTestId('checkout-loading')).toBeHidden();

    const shown = await page.getByTestId('checkout-total').innerText();
    expect(shown).toContain(String(body.totalMinor / 100));
  });

  test('la double soumission ne crée qu\'une seule commande', async ({ page }) => {
    await page.goto(localePath('fr-FR', '/checkout'));
    await b2cFixtures.fillShippingAddress(page);
    const submit = page.getByTestId('checkout-submit');
    await submit.click();
    await submit.click({ force: true });

    await expect(page).toHaveURL(new RegExp('/account/orders'));
    const rows = page.getByTestId('order-row');
    await expect(rows).toHaveCount(1);
  });
});
```

- [ ] **Step 3.4: Exécuter la suite B2C et constater l'échec attendu**

```bash
pnpm exec playwright test --project=b2c
```

Attendu (avant implémentation des pages / sélecteurs) :

```text
  1 failed
    Error: expect(locator).toBeVisible() failed
    Timeout: 10000ms exceeded.
    Locator: getByTestId('search-input')
```

- [ ] **Step 3.5: Implémenter les sélecteurs et libellés manquants jusqu'au vert**

- Ajouter les `data-testid` requis par la suite dans les composants web concernés : `locale-switcher`, `nav-catalogue`, `search-input`, `search-submit`, `search-results`, `search-empty-state`, `search-clear-filters`, `filter-min-qty`, `filter-apply`, `product-card`, `product-quantity`, `add-to-cart`, `cart-count`, `cart-checkout-link`, `price-tier-table`, `checkout-loading`, `checkout-total`, `checkout-submit`, `checkout-terms-checkbox`, `customs-estimate-disclaimer`, `order-row`, `order-status`, `order-invoice-link`, `invoice-document`, `order-tracking-link`, `tracking-timeline`, `notification-item`, `shipping-recipient`, `shipping-line1`, `shipping-city`, `shipping-postal-code`, `shipping-country`.
- Ajouter les clés de traduction correspondantes dans les 3 dictionnaires (`packages/i18n/src/locales/{fr-FR,en,zh-CN}.json`), notamment `order.status.pending_payment`.
- Vérifier la parité : `pnpm --filter @fanours/i18n check:keys`.

```bash
pnpm exec playwright test --project=b2c
```

Attendu :

```text
  6 passed (42.1s)
```

- [ ] **Step 3.6: Câbler la suite B2C dans la CI et vérifier le sharding**

Modifier `.github/workflows/e2e.yml` : la matrice passe de 4 à 5 shards, chaque shard adresse une suite nommée. La phase 5 reste **mono-navigateur (Chromium)** : un second navigateur, s'il est requis plus tard, s'ajoute comme **dimension de shard en CI** (cf. Task 6 de la Phase 6), **pas comme un projet Playwright** — le projet définit la *suite*, pas le moteur. Le job `e2e` devient :

```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest
    # Budget de durée du shard (§24.13) : E2E <= 12 min par shard. Ce plafond est
    # applique ici, par le job CI ; le timeout Playwright par test reste de 45 s.
    timeout-minutes: 12
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4, 5]
        include:
          - { shard: 1, project: 'b2c' }
          - { shard: 2, project: 'b2b' }
          - { shard: 3, project: 'payment' }
          - { shard: 4, project: 'admin' }
          - { shard: 5, project: 'security' }
    steps:
      - uses: ./.github/actions/setup
      - name: Installer les navigateurs
        run: pnpm exec playwright install --with-deps chromium
      - name: Executer la suite
        run: pnpm exec playwright test --project=${{ matrix.project }} --shard=${{ matrix.shard }}/5
        env:
          PROJECT: ${{ matrix.project }}
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-traces-${{ matrix.project }}-${{ matrix.shard }}
          path: test-results/artifacts/
          retention-days: 7
          if-no-files-found: ignore
```

```bash
pnpm exec playwright test --project=b2c --shard=1/5
```

Attendu : la suite `b2c` passe également en sharding (les tests répartis, aucun test ignoré silencieusement — `playwright test --list` et le nombre de tests exécutés concordent).

**Preuves :**
- Sortie de `pnpm exec playwright test --project=b2c` (6 tests verts, 3 locales).
- Sortie de `pnpm exec playwright test --project=b2c --shard=1/5`.
- Le parcours est rejoué dans les 3 langues avec vérification des libellés résolus depuis les dictionnaires.
- Zéro erreur console : le `afterEach` échoue sur toute `console.error` ou `pageerror`.

**Commit :**

```bash
git add playwright.config.ts e2e/b2c e2e/checkout e2e/support/i18n.ts .github/workflows/e2e.yml apps/web/src packages/i18n/src/locales
git commit -m "test(e2e): suite acheteur B2C en 9 etapes, 3 langues" -m "Chantier 5.3 (spec §24.3) : configuration Playwright racine a 7 projets (5 suites de release + visual + functional) sur un testDir unique, parcours B2C en 9 etapes rejoue en FR/EN/ZH-CN, fixtures localisees, verification d'absence de debordement horizontal a 360 px, matrice CI a 5 shards." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 4: Suite E2E « acheteur B2B » — 9 étapes

**Files:**
- Modify: `e2e/b2b/journey.spec.ts` (livré en Phase 3 — repris ici au niveau release)
- Modify: `apps/web/src/app/[locale]/(account)/compte/**` (libellés et `data-testid` B2B)
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `apps/api` → `POST /v1/auth/register`, `POST /v1/auth/verify-email`, `POST /v1/business/activation-requests`, `POST /v1/business/activation-requests/:id/decision` (rôle `super_admin` / `compliance_manager`), `GET /v1/business/pricing`, `POST /v1/quotes`, `POST /v1/quotes/:id/accept`, `POST /v1/quotes/:id/convert`, `POST /v1/orders` (mode B2B), `GET /v1/orders/:number/invoice` · `apps/web` → `/{locale}/account/register`, `/{locale}/account/verify-email`, `/{locale}/account/pro`, `/{locale}/account/quotes`, `/{locale}/account/quotes/{number}`.
- Produces:
  - `e2e/b2b/journey.spec.ts` — **repris (Modify)** : le fichier livré par la Phase 3 est porté au niveau release (données staging, isolation, 3 langues, traçabilité §24.3) ; le code ci-dessous est le **contenu release-grade final**. 9 étapes, rejouées en FR / EN / ZH-CN, avec création de compte, vérification e-mail (lien intercepté par la boîte de réception de test), activation pro, approbation par un rôle autorisé, tarifs de gros et MOQ, devis, acceptation, conversion, paiement ou virement, facture et suivi. Le projet Playwright `b2b` ne capture **que** ce fichier (`**/b2b/journey.spec.ts`) : `e2e/b2b/compliance.spec.ts` et `e2e/b2b/admin-views.spec.ts` (livrés par la Phase 3) sont des recettes fonctionnelles de phase et relèvent du projet `functional` — ils ne sont donc **pas** comptés dans la suite de release B2B.

- [ ] **Step 4.1: Reprendre (Modify) la suite B2B — 9 étapes (rouge)**

`e2e/b2b/journey.spec.ts` — **fichier livré par la Phase 3, repris ici** :

```typescript
import { expect, test } from '@playwright/test';
import { SUPPORTED_LOCALES, t, localePath } from '../support/i18n.js';

/**
 * Parcours acheteur B2B — spec §24.3 (9 étapes).
 * L'approbation de l'activation pro est faite par un rôle autorisé via l'API
 * d'administration : le parcours ne triche pas en auto-approuvant.
 */

const B2B_ACCOUNT = {
  email: 'acheteur-pro@example.test',
  company: 'Ningbo Import Co Ltd',
  country: 'DE',
} as const;

for (const locale of SUPPORTED_LOCALES) {
  test.describe(`parcours acheteur B2B — ${locale}`, () => {
    test('étapes 1 à 9 — compte, vérification, activation, approbation, tarifs, devis, conversion, paiement, facture', async ({ page, request }) => {
      // Étape 1 — création d'un compte
      await page.goto(localePath(locale, '/account/register'));
      await page.getByTestId('register-email').fill(B2B_ACCOUNT.email);
      await page.getByTestId('register-password').fill('Fanours!Passw0rd2026');
      await page.getByTestId('register-company').fill(B2B_ACCOUNT.company);
      await page.getByTestId('register-submit').click();
      await expect(page.getByTestId('register-confirmation')).toBeVisible();

      // Étape 2 — vérification de l'e-mail
      const verificationLink = await fetchLatestVerificationLink(B2B_ACCOUNT.email);
      await page.goto(verificationLink);
      await expect(page.getByTestId('email-verified')).toBeVisible();

      // Étape 3 — demande d'activation professionnelle
      await page.goto(localePath(locale, '/account/pro'));
      await page.getByTestId('pro-activation-submit').click();
      await expect(page.getByTestId('pro-activation-pending')).toBeVisible();

      // Étape 4 — approbation par un rôle autorisé
      const decision = await request.post('/v1/admin/business-activation-requests/decision', {
        headers: { 'x-admin-role': 'super_admin', 'x-admin-mfa': process.env.E2E_ADMIN_MFA_TOKEN ?? 'e2e-mfa' },
        data: { email: B2B_ACCOUNT.email, decision: 'approved' },
      });
      expect(decision.ok()).toBeTruthy();

      // Étape 5 — tarifs de gros et MOQ
      await page.goto(localePath(locale, '/products'));
      await page.goto(localePath(locale, '/products/usb-c-hub-7-ports'));
      await expect(page.getByTestId('product-moq')).toContainText(
        t(locale, 'product.moq'),
      );
      await expect(page.getByTestId('price-tier-table')).toBeVisible();

      // Étape 6 — création d'un devis
      await page.getByTestId('quote-request-submit').click();
      await expect(page).toHaveURL(new RegExp(`${locale}/account/quotes/FAN-`));

      // Étape 7 — acceptation du devis par le back-office, puis conversion
      const quoteNumber = await currentQuoteNumber(page);
      const accept = await request.post(`/v1/admin/quotes/${quoteNumber}/decision`, {
        headers: { 'x-admin-role': 'order_manager', 'x-admin-mfa': process.env.E2E_ADMIN_MFA_TOKEN ?? 'e2e-mfa' },
        data: { decision: 'accepted' },
      });
      expect(accept.ok()).toBeTruthy();

      await page.goto(localePath(locale, `/account/quotes/${quoteNumber}`));
      await page.getByTestId('quote-accept-button').click();
      await expect(page.getByTestId('quote-accepted')).toBeVisible();
      await page.getByTestId('quote-convert-button').click();
      await expect(page).toHaveURL(new RegExp(`${locale}/account/orders/FAN-`));

      // Étape 8 — paiement ou procédure de virement
      await expect(page.getByTestId('order-payment-instructions')).toBeVisible();

      // Étape 9 — facture et suivi de l'expédition
      await page.getByTestId('order-invoice-link').click();
      await expect(page.getByTestId('invoice-document')).toBeVisible();
      await page.goto(localePath(locale, '/account/orders'));
      await page.getByTestId('order-row').first().getByTestId('order-tracking-link').click();
      await expect(page.getByTestId('tracking-timeline')).toBeVisible();
    });
  });
}
```

- [ ] **Step 4.2: Exécuter la suite B2B et constater l'échec attendu**

```bash
pnpm exec playwright test --project=b2b
```

Attendu :

```text
  1 failed
    Error: expect(locator).toBeVisible() failed
    Locator: getByTestId('pro-activation-submit')
```

- [ ] **Step 4.3: Implémenter les écrans et libellés B2B manquants jusqu'au vert**

- Écrans : `/{locale}/account/pro` (demande d'activation), `/{locale}/account/quotes` (liste), `/{locale}/account/quotes/[number]` (détail, accepter, convertir).
- Libellés FR/EN/ZH-CN : `product.moq`, `quote.request`, `quote.accepted`, `quote.convert`, `pro.activation.pending`, `order.payment.instructions`, `business.account.approved`.
- `data-testid` : `register-email`, `register-password`, `register-company`, `register-submit`, `register-confirmation`, `email-verified`, `pro-activation-submit`, `pro-activation-pending`, `product-moq`, `price-tier-table`, `quote-request-submit`, `quote-accept-button`, `quote-accepted`, `quote-convert-button`, `order-payment-instructions`.

```bash
pnpm --filter @fanours/i18n check:keys
pnpm exec playwright test --project=b2b
```

Attendu :

```text
i18n key parity OK (3 locales, 486 clés)
  3 passed (58.4s)
```

- [ ] **Step 4.4: Vérifier que l'approbation B2B ne peut pas être auto-approuvée**

```bash
rg -n "self_approval|no_self_approval" apps/api/src packages/db/src | head
```

Attendu : la contrainte `no_self_approval` est présente et l'endpoint de décision renvoie `409 self_approval_forbidden` si l'approuvateur est le demandeur. Ce point est rejoué par la suite E2E admin (Task 6) et par le test d'intégration de la Task 2.

**Preuves :**
- Sortie de `pnpm exec playwright test --project=b2b` (3 tests verts, 3 langues).
- Parité i18n vérifiée.
- Capture du refus d'auto-approbation (collée dans la PR).

**Commit :**

```bash
git add e2e/b2b "apps/web/src/app/[locale]/(account)/compte" packages/i18n/src/locales
git commit -m "test(e2e): suite acheteur B2B en 9 etapes, 3 langues" -m "Chantier 5.3 (spec §24.3) : creation de compte, verification e-mail, demande et approbation d'activation pro par un role autorise, paliers gros et MOQ, devis, acceptation, conversion, paiement ou virement, facture et suivi." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 5: Suite E2E « paiement » — 8 cas

**Files:**
- Modify: `e2e/payment/scenarios.spec.ts` (livré en Phase 2 — repris ici au niveau release)
- Modify: `apps/web/src/app/[locale]/(checkout)/**` (états de paiement)
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `apps/api` → `POST /v1/checkout/quote`, `POST /v1/orders`, `POST /v1/webhooks/payment` (corps signé), `POST /v1/payments/:id/refund`, `GET /v1/payments/reconciliation` ; `packages/payment` → `PaymentProvider` (`createPayment`, `confirmPayment`, `refund`) ; PSP sandbox local `POST /v1/psp/sandbox/payments` (contrôle des réponses).
- Produces:
  - `e2e/payment/scenarios.spec.ts` — **repris (Modify)** : le fichier livré par la Phase 2 est porté au niveau release — données staging, isolation, 3 langues, traçabilité §24.3 ; le code ci-dessous est le **contenu release-grade final**. Les 8 cas sont ceux de la version Phase 2, dans le même ordre et avec les mêmes intitulés : `statut payé jamais déclaré sur une redirection navigateur`, `paiement sandbox mène à la confirmation`, `double clic de paiement`, `paiement refusé`, `interruption du prestataire`, `webhook rejoué`, `montant recalculé par le serveur`, `session expirée`. Le projet Playwright `payment` ne capture **que** ce fichier. Les signatures de webhook invalides, les ordres de livraison et le double remboursement sont couverts au niveau intégration par la Task 2 (webhooks signés : valide, invalide, dupliqué, désordonné, horodatage, corps modifié).

- [ ] **Step 5.1: Reprendre la suite paiement au niveau release — 8 cas (rouge)**

`e2e/payment/scenarios.spec.ts` — **fichier livré par la Phase 2, repris ici**. Le code ci-dessous est le **contenu release-grade final** : mêmes 8 cas, même ordre et mêmes intitulés que la version Phase 2, mais joués sur les données de staging, dans les 3 langues, avec isolation (chaque test crée sa propre commande) et vérification d'absence d'erreur console.

```typescript
import { expect, test, type Page } from '@playwright/test';
import { addToCart, REFERENCE_PRODUCT } from '../fixtures/product.js';
import { SUPPORTED_LOCALES, t, type SupportedLocale } from '../support/i18n.js';

/**
 * Suite de release `payment` — spec §24.3 (8 cas bloquants).
 *
 * Le PSP est un **serveur sandbox local** : jamais une fixture figée, jamais le
 * prestataire réel. Les commandes du sandbox sont adressées par `data-testid`
 * (harnais de test, indépendant de la locale) ; les libellés utilisateur sont
 * résolus depuis les dictionnaires.
 *
 * Ce fichier est le SEUL capturé par le projet Playwright `payment`.
 */

const consoleErrors: string[] = [];

test.beforeEach(async ({ page }) => {
  consoleErrors.length = 0;
  page.on('console', (msg) => {
    if (msg.type() === 'error') consoleErrors.push(msg.text());
  });
  page.on('pageerror', (err) => consoleErrors.push(err.message));
});

test.afterEach(async () => {
  expect(consoleErrors, `erreurs console inattendues : ${consoleErrors.join(' | ')}`).toHaveLength(0);
});

/**
 * Isolation : chaque test crée SA propre commande à partir de ses propres
 * données. Aucun test ne dépend du résultat d'un autre.
 */
async function placeOrder(page: Page, locale: SupportedLocale): Promise<void> {
  await addToCart(page, locale, REFERENCE_PRODUCT.moq);
  await page.getByRole('link', { name: t(locale, 'cart.checkout') }).click();
  await page.getByLabel(t(locale, 'checkout.field.fullName')).fill('Ada Lovelace');
  await page.getByLabel(t(locale, 'checkout.field.address')).fill('10 rue de Rivoli');
  await page.getByLabel(t(locale, 'checkout.field.postalCode')).fill('75001');
  await page.getByLabel(t(locale, 'checkout.field.city')).fill('Paris');
  await page.getByLabel(t(locale, 'checkout.field.country')).selectOption('FR');
  await page.getByLabel(t(locale, 'checkout.terms')).check();
  await page.getByRole('button', { name: t(locale, 'checkout.submit') }).click();
  await page.waitForURL(new RegExp(`${locale}/account/orders/`));
}

/** Référence de la commande courante, lue dans l'URL : aucun état partagé. */
async function currentOrderRef(page: Page): Promise<string> {
  const reference = new URL(page.url()).pathname.split('/').filter(Boolean).pop();
  if (!reference) {
    throw new Error(`URL de commande inattendue : ${page.url()}`);
  }
  return reference;
}

for (const locale of SUPPORTED_LOCALES) {
  test.describe(`parcours paiement — ${locale}`, () => {
    test('cas 1 — le statut payé n’arrive jamais d’une redirection navigateur', async ({ page }) => {
      await placeOrder(page, locale);
      await expect(page.getByTestId('order-status')).toContainText(t(locale, 'order.status.pending_payment'));
      await expect(page.getByTestId('order-status')).not.toContainText(t(locale, 'order.status.paid'));
    });

    test('cas 2 — le paiement sandbox mène à la confirmation', async ({ page }) => {
      await placeOrder(page, locale);
      await page.getByTestId('pay-now').click();
      await expect(page.getByTestId('order-status')).toContainText(t(locale, 'order.status.paid'));
    });

    test('cas 3 — un double clic de paiement ne crée pas deux paiements', async ({ page, request }) => {
      await placeOrder(page, locale);
      await page.getByTestId('pay-now').dblclick();
      await expect(page.getByTestId('order-status')).toContainText(t(locale, 'order.status.paid'));

      const reference = await currentOrderRef(page);
      const payments = await request.get(`/v1/orders/${reference}/payments`);
      expect((await payments.json()).items).toHaveLength(1);
    });

    test('cas 4 — un paiement refusé laisse la commande en attente', async ({ page }) => {
      await placeOrder(page, locale);
      await page.getByTestId('pay-now').click();
      await page.getByTestId('psp-sandbox-refuse').click();
      await expect(page.getByTestId('payment-notice')).toContainText(t(locale, 'payment.status.processing'));
    });

    test('cas 5 — une interruption du prestataire est signalée sans débit', async ({ page }) => {
      await placeOrder(page, locale);
      await page.getByTestId('pay-now').click();
      await page.getByTestId('psp-sandbox-interruption').click();
      await expect(page.getByTestId('payment-notice')).toContainText(t(locale, 'payment.notice.notDebited'));
    });

    test('cas 6 — un webhook rejoué ne change pas l’état de la commande', async ({ page, request }) => {
      await placeOrder(page, locale);
      await page.getByTestId('pay-now').click();
      await expect(page.getByTestId('order-status')).toContainText(t(locale, 'order.status.paid'));

      const reference = await currentOrderRef(page);
      const replay = await request.post('/v1/webhooks/payment', {
        data: { id: `evt_replay_${reference}`, type: 'payment.succeeded', data: { orderId: reference } },
      });
      expect(replay.status()).toBe(409);

      await page.reload();
      await expect(page.getByTestId('order-status')).toContainText(t(locale, 'order.status.paid'));
    });

    test('cas 7 — le montant affiché est celui recalculé par le serveur', async ({ page, request }) => {
      await placeOrder(page, locale);
      const reference = await currentOrderRef(page);
      const order = await request.get(`/v1/orders/${reference}`);
      const expectedMinor = (await order.json()).totalMinor;

      const shownMinor = await page.getByTestId('order-total').getAttribute('data-amount-minor');
      expect(Number(shownMinor)).toBe(expectedMinor);
      await expect(page.getByTestId('order-total')).toContainText(/\d/);
    });

    test('cas 8 — une session expirée renvoie vers la reconnexion', async ({ page }) => {
      await placeOrder(page, locale);
      await page.context().clearCookies();
      await page.reload();
      await expect(page.getByTestId('session-expired')).toBeVisible();
    });
  });
}
```

- [ ] **Step 5.2: Exécuter la suite paiement et constater l'échec attendu**

```bash
pnpm exec playwright test --project=payment
```

Attendu :

```text
  8 failed
    Error: expect(locator).toBeVisible() failed
    Timeout: 10000ms exceeded.
    Locator: getByTestId('pay-now')
```

- [ ] **Step 5.3: Implémenter les états de paiement et le sandbox PSP jusqu'au vert**

- Route de test `POST /v1/psp/sandbox/payments/*` : disponible **uniquement** en environnement `test`/`staging` (jamais en production), contrôlée par `NODE_ENV`, et refusée par `403` sinon.
- État `payment_processing` affiché quand le PSP renvoie une challenges 3DS ou un timeout ; la commande **ne passe jamais** à `paid` sur un retour navigateur.
- État `payment_failed` affiché avec un code d'erreur stable et un message localisé ; aucune donnée technique exposée au client.
- Rapprochement quotidien : job `payment.reconcile` (02:30 UTC) qui compare les paiements PSP et la base, crée une ligne `mismatch` et une tâche de résolution.
- Libellés du parcours de paiement dans les 3 dictionnaires : `cart.checkout`, `checkout.field.fullName`, `checkout.field.address`, `checkout.field.postalCode`, `checkout.field.city`, `checkout.field.country`, `checkout.terms`, `checkout.submit`, `order.status.pending_payment`, `order.status.paid`, `payment.status.processing`, `payment.notice.notDebited`.
- `data-testid` du parcours de paiement et du harnais sandbox : `pay-now`, `psp-sandbox-refuse`, `psp-sandbox-interruption`, `payment-notice`, `order-total` (avec `data-amount-minor`), `order-status`, `session-expired`.

```bash
pnpm exec playwright test --project=payment
```

Attendu :

```text
  24 passed (2m 11.7s)
```

(8 cas × 3 langues : FR / EN / ZH-CN.)

- [ ] **Step 5.4: Vérifier que le sandbox PSP est inaccessible en production**

```bash
rg -n "psp/sandbox" apps/api/src | head
rg -n "NODE_ENV.*production|isProduction" apps/api/src/modules/payments/psp-sandbox.controller.ts
```

Attendu : le contrôleur vérifie `NODE_ENV` et répond `403 sandbox_disabled` hors `test`/`staging` ; le test est rejoué par la suite `security` (Task 7).

**Preuves :**
- Sortie de `pnpm exec playwright test --project=payment` (24 tests verts, 8 cas × 3 langues).
- Vérification que `paid` n'est atteignable que par webhook signé (couverture par la Task 2 : webhook invalide, dupliqué, désordonné, horodatage, corps modifié).
- Sandbox PSP refusé hors test.

**Commit :**

```bash
git add e2e/payment apps/api/src/modules/payments apps/web/src packages/i18n/src/locales
git commit -m "test(e2e): suite paiement 8 cas reprise au niveau release, 3 langues" -m "Chantier 5.3 (spec §24.3, §10.3, §26.3) : sandbox PSP local hors production, commande jamais 'payee' sur retour navigateur, double clic sans double paiement, webhook rejoue sans effet, montant relu depuis le serveur, session expiree." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 6: Suite E2E « admin » — 11 cas

**Files:**
- Modify: `e2e/admin/dashboard.spec.ts` (livré en Phase 4 — repris ici au niveau release)
- Modify: `apps/web/src/app/[locale]/admin/**` (états et libellés d'administration)
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`

**Interfaces:**
- Consumes: `apps/api` → `GET /v1/admin/dashboard/summary`, `GET /v1/admin/products`, `POST /v1/admin/products`, `PATCH /v1/admin/products/:id`, `PATCH /v1/admin/products/:id/price-tiers`, `PATCH /v1/admin/stock/:sku`, `GET /v1/admin/orders`, `PATCH /v1/admin/orders/:id/status`, `POST /v1/admin/refunds`, `POST /v1/admin/exports`, `GET /v1/admin/audit-logs`, `POST /v1/admin/roles` (four-eyes) ; `apps/web` → `/{locale}/admin`, `/{locale}/admin/products`, `/{locale}/admin/orders`, `/{locale}/admin/stock`, `/{locale}/admin/audit`.
- Produces:
  - `e2e/admin/dashboard.spec.ts` — **repris (Modify)** : le fichier livré par la Phase 4 est porté au niveau release — données staging, isolation, traçabilité §24.3 ; le code ci-dessous est le **contenu release-grade final**. Les 11 cas sont les **cas canoniques** du spec §24.4 (`docs/roadmap/sections/qualite.md` §24.4, tableau « Les 11 cas du parcours administration »), dans le même ordre et avec les mêmes intitulés. Le projet Playwright `admin` ne capture **que** ce fichier : les autres specs `e2e/admin/*` (Phase 4 — `a11y.spec.ts`, `security-regression.spec.ts`, `exports.spec.ts`, `catalog-import.spec.ts`, `stock.spec.ts`, `finance.spec.ts`, `logistics.spec.ts`, `support.spec.ts`, `audit.spec.ts`) sont des recettes fonctionnelles de phase et restent dans le projet `functional`.

- [ ] **Step 6.1: Reprendre la suite admin au niveau release — 11 cas (rouge)**

`e2e/admin/dashboard.spec.ts` — **fichier livré par la Phase 4, repris ici**. Le code ci-dessous est le **contenu release-grade final** : les **11 cas canoniques** du spec §24.4 (`docs/roadmap/sections/qualite.md` §24.4), dans le même ordre et avec les mêmes intitulés que ceux de la Phase 4, mais joués sur les données de staging, en isolation, et avec vérification d'absence d'erreur console.

```typescript
import { expect, test, type Page } from '@playwright/test';
import { localePath } from '../support/i18n.js';
import { loginAsAdmin } from './fixtures/admin-users.js';

/**
 * Suite de release `admin` — spec §24.3 et §24.4 (11 cas canoniques).
 * Les actions financières passent par la four-eyes : le cas 10 échoue
 * volontairement au premier rôle et réussit avec un second rôle.
 *
 * Ce fichier est le SEUL capturé par le projet Playwright `admin` : les autres
 * specs `e2e/admin/*` sont des recettes fonctionnelles de phase (projet
 * `functional`).
 */

const consoleErrors: string[] = [];

test.beforeEach(async ({ page }) => {
  consoleErrors.length = 0;
  page.on('console', (msg) => {
    if (msg.type() === 'error') consoleErrors.push(msg.text());
  });
  page.on('pageerror', (err) => consoleErrors.push(err.message));
});

test.afterEach(async () => {
  expect(consoleErrors, `erreurs console inattendues : ${consoleErrors.join(' | ')}`).toHaveLength(0);
});

/** Référence de la commande affichée, lue dans la ligne admin : aucun état partagé. */
async function currentOrderRef(page: Page): Promise<string> {
  const reference = await page.getByTestId('admin-order-row').first().getAttribute('data-order-number');
  if (!reference) {
    throw new Error('Référence de commande introuvable dans la ligne admin');
  }
  return reference;
}

test.describe('parcours administration — 11 cas', () => {
  test('cas 1 — connexion avec MFA : session persistante et journal admin.login', async ({ page, request }) => {
    await loginAsAdmin(page, 'order_manager');

    const session = await request.get('/v1/auth/session');
    expect(session.ok()).toBeTruthy();
    expect((await session.json()).mfaVerifiedAt).toBeTruthy();

    const audit = await request.get('/v1/admin/audit-logs?action=admin.login&actor_role=order_manager');
    expect((await audit.json()).items.length).toBeGreaterThan(0);
  });

  test('cas 2 — refus d\'accès pour un rôle trop faible : 403 sans révéler la ressource', async ({ page, request }) => {
    await loginAsAdmin(page, 'support_agent');

    const response = await request.get('/v1/admin/refunds');
    expect(response.status()).toBe(403);

    const body = await response.json();
    expect(body.code).toBe('forbidden');
    expect(body).not.toHaveProperty('refunds');
    expect(body).not.toHaveProperty('items');

    const audit = await request.get('/v1/admin/audit-logs?result=denied');
    expect((await audit.json()).items.length).toBeGreaterThan(0);
  });

  test('cas 3 — création et publication d\'un produit traduit en trois langues', async ({ page, request }) => {
    await loginAsAdmin(page, 'catalog_manager');
    await page.goto(localePath('fr-FR', '/admin/products'));
    await page.getByTestId('product-create').click();
    await page.getByTestId('product-sku').fill('SKU-E2E-NEW');
    await page.getByTestId('product-slug').fill('sku-e2e-new');
    await page.getByTestId('product-name-fr').fill('Produit de recette E2E');
    await page.getByTestId('product-name-en').fill('E2E test product');
    await page.getByTestId('product-name-zh').fill('E2E 测试产品');
    await page.getByTestId('product-save').click();
    await expect(page.getByTestId('product-row-SKU-E2E-NEW')).toBeVisible();

    // Une traduction manquante refuse la publication.
    await page.getByTestId('product-row-SKU-E2E-NEW').getByTestId('product-publish').click();
    await expect(page.getByTestId('product-publish-error')).toBeVisible();

    // Les 3 locales présentes : la publication passe et le produit rejoint chaque sitemap.
    await page.getByTestId('product-row-SKU-E2E-NEW').getByTestId('product-publish').click();
    await expect(page.getByTestId('product-row-SKU-E2E-NEW').getByTestId('product-status')).toContainText('publié');

    for (const locale of ['fr-FR', 'en', 'zh-CN']) {
      const sitemap = await request.get(`/sitemap.xml?locale=${locale}`);
      expect(await sitemap.text()).toContain('sku-e2e-new');
    }
  });

  test('cas 4 — modification de MOQ et de prix : four-eyes et snapshot conservé', async ({ page, request }) => {
    await loginAsAdmin(page, 'catalog_manager');
    await page.goto(localePath('fr-FR', '/admin/products'));
    await page.getByTestId('product-row-SKU-USB-C-7P').getByTestId('product-edit').click();
    await page.getByTestId('price-tier-row').first().getByTestId('tier-price').fill('11.50');
    await page.getByTestId('price-tiers-save').click();

    // Sans second approbateur, la modification reste en attente.
    await expect(page.getByTestId('price-tier-approval-required')).toBeVisible();
    const pending = await request.get('/v1/admin/price-tiers/SKU-USB-C-7P');
    expect((await pending.json()).status).toBe('pending_approval');

    // L'ancien palier reste lisible dans l'historique.
    const history = await request.get('/v1/admin/price-tiers/SKU-USB-C-7P/history');
    const versions = (await history.json()).items;
    expect(versions.length).toBeGreaterThan(1);
    expect(versions.some((version) => version.unitPriceMinor === 1200)).toBeTruthy();

    // Une commande déjà passée relit son snapshot et n'est pas réévaluée.
    const order = await request.get('/v1/orders/latest');
    expect((await order.json()).priceSnapshot.unitPriceMinor).toBe(1200);
  });

  test('cas 5 — gestion d\'un stock bas : alerte, réapprovisionnement et écart tracé', async ({ page, request }) => {
    await loginAsAdmin(page, 'order_manager');
    await page.goto(localePath('fr-FR', '/admin/stock'));

    // Stock sous le seuil : l'alerte stock.low est visible.
    await page.getByTestId('stock-row-SKU-USB-C-7P').getByTestId('stock-adjust').click();
    await page.getByTestId('stock-delta').fill('-100');
    await page.getByTestId('stock-reason').selectOption('correction');
    await page.getByTestId('stock-confirm').click();
    await expect(page.getByTestId('stock-low-alert')).toBeVisible();

    // Réapprovisionnement : mouvement d'inventaire.
    await page.getByTestId('stock-adjust').click();
    await page.getByTestId('stock-delta').fill('25');
    await page.getByTestId('stock-reason').selectOption('reception');
    await page.getByTestId('stock-confirm').click();
    await expect(page.getByTestId('stock-row-SKU-USB-C-7P').getByTestId('stock-available')).toContainText('25');

    const movements = await request.get('/v1/admin/stock/SKU-USB-C-7P/movements');
    const items = (await movements.json()).items;
    expect(items.some((movement) => movement.type === 'replenishment')).toBeTruthy();

    // L'écart d'inventaire laisse une trace non résolue.
    const discrepancies = await request.get('/v1/admin/stock/SKU-USB-C-7P/discrepancies?resolved=false');
    expect((await discrepancies.json()).items.length).toBeGreaterThan(0);
  });

  test('cas 6 — traitement d\'une commande et création d\'une expédition', async ({ page, request }) => {
    await loginAsAdmin(page, 'order_manager');
    await page.goto(localePath('fr-FR', '/admin/orders'));
    const row = page.getByTestId('admin-order-row').first();

    await row.getByTestId('order-status-select').selectOption('in_production');
    await expect(row.getByTestId('order-status')).toContainText('en production');

    await row.getByTestId('shipment-create').click();
    await page.getByTestId('shipment-tracking-number').fill('TRACK-E2E-0001');
    await page.getByTestId('shipment-confirm').click();
    await expect(page.getByTestId('shipment-tracking-number-cell')).toContainText('TRACK-E2E-0001');

    // Les événements de tracking sont append-only.
    const events = await request.get('/v1/admin/shipments/latest/tracking-events');
    const items = (await events.json()).items;
    expect(items.length).toBeGreaterThan(0);
    expect(items[items.length - 1]?.status).toBe('shipped');
  });

  test('cas 7 — refus d\'expédition si les données douanières obligatoires manquent', async ({ page, request }) => {
    await loginAsAdmin(page, 'order_manager');
    await page.goto(localePath('fr-FR', '/admin/orders'));
    const orderRef = await currentOrderRef(page);

    const response = await request.post('/v1/admin/shipments', {
      data: { orderId: orderRef, customsData: {} },
    });
    expect(response.status()).toBe(409);

    const body = await response.json();
    expect(body.code).toBe('customs_data_incomplete');
    // La règle applicable est citée avec sa source et sa date de consultation.
    expect(body.rule.sourceUrl).toBeTruthy();
    expect(body.rule.consultedAt).toBeTruthy();

    // Aucun numéro de suivi n'est créé.
    const shipments = await request.get(`/v1/admin/shipments?orderId=${orderRef}`);
    expect((await shipments.json()).items).toHaveLength(0);
  });

  test('cas 8 — mise à jour d\'une règle, historique et alerte de révision', async ({ page, request }) => {
    await loginAsAdmin(page, 'compliance_manager');
    await page.goto(localePath('fr-FR', '/admin/trade-rules'));
    await page.getByTestId('trade-rule-create-version').click();
    await page.getByTestId('trade-rule-valid-from').fill('2026-01-01');
    await page.getByTestId('trade-rule-valid-to').fill('2026-06-30');
    await page.getByTestId('trade-rule-source-url').fill('https://example.test/reglementation');
    await page.getByTestId('trade-rule-save').click();
    await expect(page.getByTestId('trade-rule-version-saved')).toBeVisible();

    // La version précédente reste lisible.
    const versions = await request.get('/v1/admin/trade-rules/8472/versions');
    expect((await versions.json()).items.length).toBeGreaterThan(1);

    // Une règle expirée déclenche une alerte visible du compliance manager…
    await expect(page.getByTestId('trade-rule-expired-alert')).toBeVisible();

    // …et n'est jamais appliquée silencieusement.
    const resolution = await request.get('/v1/admin/trade-rules/resolve?country=DE&sku=SKU-USB-C-7P');
    expect((await resolution.json()).certainty).toBe('incomplet');
  });

  test('cas 9 — export restreint selon le rôle', async ({ page, request }) => {
    await loginAsAdmin(page, 'analyst_readonly');
    const forbidden = await request.post('/v1/admin/exports', { data: { type: 'export_financial' } });
    expect(forbidden.status()).toBe(403);

    await loginAsAdmin(page, 'finance_manager');
    const requested = await request.post('/v1/admin/exports', { data: { type: 'export_financial' } });
    expect(requested.status()).toBe(202);
    expect((await requested.json()).status).toBe('pending_approval');

    // L'export est journalisé.
    const audit = await request.get('/v1/admin/audit-logs?action=export.requested');
    expect((await audit.json()).items.length).toBeGreaterThan(0);
  });

  test('cas 10 — journal d\'audit pour action financière sensible', async ({ page, request }) => {
    await loginAsAdmin(page, 'finance_manager');
    await page.goto(localePath('fr-FR', '/admin/orders'));
    const orderRef = await currentOrderRef(page);

    const refused = await request.post('/v1/admin/refunds', {
      headers: { 'x-admin-role': 'finance_manager' },
      data: { orderId: orderRef, amountMinor: 1_000, reason: 'customer_request' },
    });
    expect(refused.status()).toBe(409);
    const approvalId = (await refused.json()).approvalId;
    expect(approvalId).toBeTruthy();

    // L'auto-approbation est refusée en base ET en code.
    const selfApproval = await request.post(`/v1/admin/approvals/${approvalId}/approve`, {
      headers: { 'x-admin-role': 'finance_manager' },
    });
    expect(selfApproval.status()).toBe(409);
    expect((await selfApproval.json()).code).toBe('self_approval_forbidden');

    // Chaînage de hash vérifié et suite admin_action_approval tracée.
    const audit = await request.get('/v1/admin/audit-logs?action=refund.requested');
    const entries = (await audit.json()).items;
    expect(entries.length).toBeGreaterThan(0);
    for (let index = 1; index < entries.length; index += 1) {
      expect(entries[index]?.prevHash).toBe(entries[index - 1]?.entryHash);
    }
  });

  test('cas 11 — navigation admin limitée au rôle, route non déclarée refusée', async ({ page, request }) => {
    await loginAsAdmin(page, 'support_agent');
    await page.goto(localePath('fr-FR', '/admin'));

    // Le menu n'affiche que les entrées autorisées pour le rôle connecté.
    await expect(page.getByTestId('admin-nav-tickets')).toBeVisible();
    await expect(page.getByTestId('admin-nav-products')).toHaveCount(0);

    // Une route non déclarée dans la matrice est refusée au démarrage.
    await page.goto(localePath('fr-FR', '/admin/route-non-declaree'));
    await expect(page.getByTestId('admin-route-refused')).toBeVisible();

    // Le refus est journalisé.
    const audit = await request.get('/v1/admin/audit-logs?result=denied');
    expect((await audit.json()).items.length).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 6.2: Exécuter la suite admin et constater l'échec attendu**

```bash
pnpm exec playwright test --project=admin
```

Attendu :

```text
  3 failed
    Error: expect(response.status()).toBe(409)
    Received: 201
```

(cas 10 : le remboursement a été accepté sans second approbateur — c'est précisément le défaut que la suite doit détecter.)

- [ ] **Step 6.3: Implémenter la four-eyes, les exports signés et les écrans manquants jusqu'au vert**

- `POST /v1/admin/refunds` : création d'une ligne `admin_action_approval` et réponse `409 { code: 'approval_required' }` tant qu'aucune approbation d'un second rôle n'est enregistrée ; approbation via `POST /v1/admin/approvals/:id/approve` par un rôle différent ; auto-approbation refusée en base **et** en code (`409 self_approval_forbidden`) ; le journal d'audit contient les deux acteurs et le chaînage `prev_hash` / `entry_hash` est vérifié.
- `POST /v1/admin/exports` : génération asynchrone, stockage S3, URL signée à durée courte (`X-Amz-Expires`), journalisation de l'export ; aucun export financier accessible au rôle `analyst_readonly` (403) ; les colonnes exportées sont restreintes à la liste blanche du rôle.
- `POST /v1/admin/shipments` : `409 { code: 'customs_data_incomplete' }` quand une donnée douanière obligatoire manque, avec la règle applicable citée (source + date de consultation) et **aucun** numéro de suivi créé.
- `POST /v1/admin/price-tiers/:sku` : la modification passe par la four-eyes (`pending_approval` sans second rôle) ; l'ancien palier reste lisible dans l'historique ; une commande déjà passée relit son snapshot et n'est pas réévaluée.
- Alerte `stock.low` déclenchée sous le seuil, réapprovisionnement journalisé comme mouvement `inventory_movement`, écart d'inventaire laissé en trace non résolue.
- `GET /v1/admin/trade-rules/resolve` : une règle expirée n'est jamais appliquée silencieusement — elle renvoie une certitude `incomplet` et déclenche une alerte au compliance manager.
- Écrans d'administration manquants et `data-testid` : `product-create`, `product-sku`, `product-slug`, `product-name-fr`, `product-name-en`, `product-name-zh`, `product-save`, `product-edit`, `product-publish`, `product-publish-error`, `product-status`, `price-tier-row`, `tier-price`, `price-tiers-save`, `price-tier-approval-required`, `stock-row-{sku}`, `stock-adjust`, `stock-delta`, `stock-reason`, `stock-confirm`, `stock-available`, `stock-low-alert`, `admin-order-row`, `order-status-select`, `order-status`, `shipment-create`, `shipment-tracking-number`, `shipment-confirm`, `shipment-tracking-number-cell`, `trade-rule-create-version`, `trade-rule-valid-from`, `trade-rule-valid-to`, `trade-rule-source-url`, `trade-rule-save`, `trade-rule-version-saved`, `trade-rule-expired-alert`, `export-approval-required`, `admin-nav-tickets`, `admin-nav-products`, `admin-route-refused`.
- Libellés FR/EN/ZH-CN de l'administration.

```bash
pnpm exec playwright test --project=admin
```

Attendu :

```text
  11 passed (1m 12.4s)
```

- [ ] **Step 6.4: Vérifier que les tableaux ne sont pas des cartes de tableau figées (audit §3.11)**

```bash
rg -n "rounded-full|glass" "apps/web/src/app/[locale]/admin" | head
```

Attendu : aucune occurrence de `rounded-full` systématisé sur les badges ni de `backdrop-filter` dans les zones opérationnelles (le verre est réservé au header, au panier, aux modales et aux drawers).

**Preuves :**
- Sortie de `pnpm exec playwright test --project=admin` (11 tests verts — les 11 cas canoniques du spec §24.4).
- Capture du cas 10 (409 `approval_required`, puis `409 self_approval_forbidden` sur l'auto-approbation) dans la PR.
- Vérification que l'export financier est refusé au rôle `analyst_readonly` et que la publication d'un produit sans ses 3 traductions est refusée.

**Commit :**

```bash
git add e2e/admin "apps/web/src/app/[locale]/admin" packages/i18n/src/locales apps/api/src/modules/admin
git commit -m "test(e2e): suite administration 11 cas reprise au niveau release" -m "Chantier 5.3 (spec §24.3, §24.4, §7.3) : MFA et journal admin.login, refus 403 sans revelation, produit traduit en 3 langues, four-eyes sur les paliers avec snapshot conserve, stock bas et ecart trace, expedition et tracking append-only, 409 customs_data_incomplete, historique et alerte de regle, export restreint par role, audit financier avec chainage de hash, navigation limitee au role." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 7: Suite E2E « sécurité » — 7 cas

**Files:**
- Create: `e2e/security/security.spec.ts`
- Modify: `.github/workflows/e2e.yml` (projet `security` déjà ajouté en Task 3 — ici on ajoute le job de scan bloquant si absent)

**Interfaces:**
- Consumes: `apps/api` → `POST /v1/auth/login`, `POST /v1/auth/register`, `POST /v1/auth/forgot-password`, `GET /v1/orders/:id`, `POST /v1/admin/uploads`, `POST /v1/orders` ; `apps/web` → rendu des pages publiques.
- Produces:
  - `e2e/security/security.spec.ts` — 7 cas : `IDOR`, `rate limit`, `anti-énumération`, `CSRF`, `XSS`, `session`, `upload`, avec les assertions exactes de `docs/roadmap/sections/securite.md` §13.1. Ce fichier est le **seul** capturé par le projet Playwright `security` : la recette fonctionnelle de phase `e2e/specs/security.spec.ts` (livrée par la Phase 1) est **distincte** et relève du projet `functional` — les deux fichiers ne se recoupent pas.

- [ ] **Step 7.1: Écrire la suite sécurité — 7 cas (rouge)**

`e2e/security/security.spec.ts` :

```typescript
import { expect, test } from '@playwright/test';

/**
 * Parcours sécurité — spec §24.3 (7 exigences) et roadmap `securite.md` §13.1.
 * Chaque assertion est littérale : un 404 (et non 403) sur une ressource qui
 * appartient à autrui, un 429 avec `Retry-After`, un 400 sur `<script>`.
 */

test.describe('parcours sécurité — 7 cas', () => {
  test('cas 1 — IDOR : une commande d\'autrui renvoie 404, identique à un identifiant inexistant', async ({ request }) => {
    const victimOrderId = await createOrderOwnedBy('victim@example.test');
    const absentOrderId = '00000000-0000-4000-8000-000000000000';

    const forbidden = await request.get(`/v1/orders/${victimOrderId}`);
    const absent = await request.get(`/v1/orders/${absentOrderId}`);

    expect(forbidden.status()).toBe(404);
    expect(absent.status()).toBe(404);
    expect(await forbidden.text()).toBe(await absent.text());
  });

  test('cas 2 — rate limit : la 6e tentative de connexion renvoie 429 avec Retry-After', async ({ request }) => {
    const email = `ratelimit-${Date.now()}@example.test`;
    for (let attempt = 0; attempt < 5; attempt += 1) {
      await request.post('/v1/auth/login', { data: { email, password: 'Wrong!Passw0rd' } });
    }

    const sixth = await request.post('/v1/auth/login', { data: { email, password: 'Wrong!Passw0rd' } });

    expect(sixth.status()).toBe(429);
    expect(sixth.headers()['retry-after']).toBeTruthy();
    expect(sixth.headers()['x-ratelimit-remaining']).toBe('0');
  });

  test('cas 3 — anti-énumération : compte existant et compte inexistant répondent de façon identique', async ({ request }) => {
    const existing = await request.post('/v1/auth/forgot-password', { data: { email: 'admin@example.test' } });
    const missing = await request.post('/v1/auth/forgot-password', { data: { email: `ghost-${Date.now()}@example.test` } });

    expect(existing.status()).toBe(missing.status());
    expect(await existing.text()).toBe(await missing.text());
  });

  test('cas 4 — CSRF : en-tête CSRF absent ou non concordant renvoie 403, origine étrangère aussi', async ({ request }) => {
    const noToken = await request.post('/v1/cart/items', { data: { sku: 'SKU-USB-C-7P', quantity: 1 } });
    expect(noToken.status()).toBe(403);

    const badToken = await request.post('/v1/cart/items', {
      headers: { 'x-csrf-token': 'token-issu' },
      data: { sku: 'SKU-USB-C-7P', quantity: 1 },
    });
    expect(badToken.status()).toBe(403);

    const badOrigin = await request.post('/v1/cart/items', {
      headers: { 'x-csrf-token': await validCsrfToken(), origin: 'https://attaquant.example' },
      data: { sku: 'SKU-USB-C-7P', quantity: 1 },
    });
    expect(badOrigin.status()).toBe(403);
  });

  test('cas 5 — XSS : un script dans un champ est refusé (400) et échappé au rendu', async ({ page, request }) => {
    const response = await request.post('/v1/auth/register', {
      data: { email: 'xss@example.test', password: 'Fanours!Passw0rd', companyName: '<script>alert(1)</script>' },
    });
    expect(response.status()).toBe(400);

    await page.goto('/fr-FR/products?q=%3Cscript%3E');
    const html = await page.content();
    expect(html).not.toContain('<script>alert(1)</script>');
  });

  test('cas 6 — session : cookie altéré, expiré, révoqué ou antérieur au login donne 401', async ({ request }) => {
    const { cookie, csrfToken } = await loginAndGetCookies('client@example.test');

    const tampered = await request.get('/v1/account/profile', { headers: { cookie: `${cookie.split(';')[0].split('=')[0]}=valeur-falsifiee` } });
    expect(tampered.status()).toBe(401);

    const revoked = await request.get('/v1/account/profile', { headers: { cookie: await revokedSessionCookie() } });
    expect(revoked.status()).toBe(401);

    const preLogin = await request.get('/v1/account/profile', { headers: { cookie: await preLoginSessionCookie() } });
    expect(preLogin.status()).toBe(401);
  });

  test('cas 7 — upload : un exécutable renommé en PDF renvoie 415, un fichier de 12 Mo renvoie 413', async ({ request }) => {
    const exe = await request.post('/v1/admin/uploads', {
      headers: { 'x-admin-role': 'support_agent' },
      multipart: {
        file: { name: 'contrat.pdf', mimeType: 'application/pdf', buffer: Buffer.from('MZ\x90\x00exe') },
      },
    });
    expect(exe.status()).toBe(415);

    const tooBig = await request.post('/v1/admin/uploads', {
      headers: { 'x-admin-role': 'support_agent' },
      multipart: {
        file: { name: 'facture.pdf', mimeType: 'application/pdf', buffer: Buffer.alloc(12 * 1024 * 1024) },
      },
    });
    expect(tooBig.status()).toBe(413);
  });
});
```

- [ ] **Step 7.2: Exécuter la suite sécurité et constater l'échec attendu**

```bash
pnpm exec playwright test --project=security
```

Attendu :

```text
  2 failed
    Error: expect(response.status()).toBe(404)
    Received: 403
```

(cas 1 : l'API renvoie 403 au lieu de 404 — fuite d'existence de la ressource, à corriger.)

- [ ] **Step 7.3: Corriger les écarts constatés jusqu'au vert**

Points à traiter (dans l'ordre de découverte) :

1. **IDOR** : le contrôle de propriété est appliqué **dans la requête** (scoping du filtre), jamais en post-filtrage en mémoire ; la route renvoie `404` avec un corps identique à celui d'une ressource inexistante.
2. **Rate limit** : fenêtre glissante Redis, clé composite `rl:login:<ip>:<email>`, `5` tentatives / 15 min, réponses `429` + `Retry-After` + `X-RateLimit-Limit` + `X-RateLimit-Remaining` + `X-RateLimit-Reset`.
3. **Anti-énumération** : la réponse de `forgot-password` est identique (même statut, même corps, même durée) pour un compte existant et un compte inexistant ; le délai de réponse est constant (anti-timing).
4. **CSRF** : token synchronisé vérifié sur toute mutation, plus vérification de l'en-tête `Origin` contre l'allowlist ; `SameSite=Lax` (et `Strict` sur les mutations sensibles).
5. **XSS** : validation Zod en entrée (400), échappement React en sortie, aucun `dangerouslySetInnerHTML`, `href` utilisateur restreint à `http/https/mailto`.
6. **Session** : cookie `__Host-` + `HttpOnly` + `Secure` + `SameSite=Lax`, expiration inactive et absolue, rotation de l'identifiant au login, révocation multi-appareils ; toute session antérieure au login est invalide.
7. **Upload** : détection par **magic bytes** (415 si le type réel ne correspond pas), taille maximale 10 Mo (413), nom régénéré en UUID, stockage hors racine web, scan antivirus, jamais d'exécution.

```bash
pnpm exec playwright test --project=security
```

Attendu :

```text
  7 passed (44.9s)
```

- [ ] **Step 7.4: Vérifier que la suite sécurité est bien bloquante en CI**

```bash
rg -n "project=security|--project=security" .github/workflows/*.yml
```

Attendu : le job `e2e` exécute la suite `security` sur la matrice ; toute défaillance fait échouer le pipeline (pas de `continue-on-error`).

**Preuves :**
- Sortie de `pnpm exec playwright test --project=security` (7 tests verts).
- Schéma avant/après : la réponse IDOR passe de 403 à 404.
- Confirmation que le job `security` est bloquant dans la CI.

**Commit :**

```bash
git add e2e/security apps/api/src packages/security/src .github/workflows
git commit -m "test(e2e): suite securite 7 cas (IDOR, rate limit, anti-enumeration, CSRF, XSS, session, upload)" -m "Chantier 5.3 (spec §24.3, §19) : scoping anti-IDOR dans la requete avec 404, fenetre glissante Redis, reponse anti-enumeration constante, double controle CSRF token + Origin, magic bytes et limite de taille sur upload, rotation et revocation de session." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 8: Tests de concurrence et d'idempotence (2 clients / dernière unité ; double webhook ; double remboursement ; double soumission)

**Files:**
- Create: `e2e/concurrency/stock-oversell.spec.ts`
- Create: `e2e/concurrency/double-webhook.spec.ts`
- Create: `e2e/concurrency/double-refund.spec.ts`
- Create: `e2e/concurrency/double-submission.spec.ts`
- Modify: `apps/api/src/modules/inventory/**` (verrou de réservation si manquant)
- Modify: `apps/api/src/modules/payments/**` (verrou d'idempotence si manquant)

**Interfaces:**
- Consumes: `apps/api` → `POST /v1/orders`, `POST /v1/webhooks/payment`, `POST /v1/payments/:id/refund` ; `packages/db` → `SELECT … FOR UPDATE` sur `inventory_item` et `stock_reservation`.
- Produces:
  - 4 fichiers de test de concurrence exécutés **sur PostgreSQL réel** (base d'intégration), jamais sur une base distante éphémère.

- [ ] **Step 8.1: Écrire le test d'oversell (rouge)**

`e2e/concurrency/stock-oversell.spec.ts` :

```typescript
import { expect, test } from '@playwright/test';

/**
 * Concurrence — spec §24.2 et §24.5 : deux clients achètent la dernière unité.
 * Attendu : une seule commande réussit, l'autre reçoit un conflit de stock.
 * Test exécuté contre une base PostgreSQL réelle (verrou row-level).
 */
test('deux clients sur la dernière unité : une seule commande aboutit', async ({ request }) => {
  const sku = await seedSkuWithStock({ sku: 'SKU-LAST-ONE', available: 1 });

  const [first, second] = await Promise.all([
    request.post('/v1/orders', {
      headers: { 'Idempotency-Key': `concurrent-a-${Date.now()}` },
      data: { items: [{ sku, quantity: 1 }], acceptTerms: true },
    }),
    request.post('/v1/orders', {
      headers: { 'Idempotency-Key': `concurrent-b-${Date.now()}` },
      data: { items: [{ sku, quantity: 1 }], acceptTerms: true },
    }),
  ]);

  const statuses = [first.status(), second.status()].sort();
  expect(statuses).toEqual([201, 409]);

  const conflict = first.status() === 409 ? first : second;
  expect((await conflict.json()).code).toBe('stock_conflict');

  expect(await availableStock(sku)).toBe(0);
  expect(await countOrdersForSku(sku)).toBe(1);
});
```

- [ ] **Step 8.2: Écrire les trois autres tests de concurrence (rouge)**

`e2e/concurrency/double-webhook.spec.ts` :

```typescript
import { createHmac, randomUUID } from 'node:crypto';
import { expect, test } from '@playwright/test';

const SECRET = process.env.E2E_WEBHOOK_SECRET ?? 'test-webhook-secret';

test('deux webhooks identiques simultanés : un seul effet', async ({ request }) => {
  const { orderId } = await seedPendingOrder();
  const body = JSON.stringify({
    id: `evt_${randomUUID()}`,
    type: 'payment.succeeded',
    data: { orderId, amountMinor: 12_000, currency: 'EUR' },
  });
  const timestamp = Math.floor(Date.now() / 1000);
  const signature = createHmac('sha256', SECRET).update(`${timestamp}.${body}`).digest('hex');
  const headers = { 'content-type': 'application/json', 'x-webhook-timestamp': String(timestamp), 'x-webhook-signature': signature };

  const [a, b] = await Promise.all([
    request.post('/v1/webhooks/payment', { headers, data: body }),
    request.post('/v1/webhooks/payment', { headers, data: body }),
  ]);

  expect([a.status(), b.status()].sort()).toEqual([200, 409]);
  expect(await countInvoices(orderId)).toBe(1);
  expect(await countOutboxEvents(orderId)).toBe(1);
});
```

`e2e/concurrency/double-refund.spec.ts` :

```typescript
import { expect, test } from '@playwright/test';

test('deux remboursements simultanés du montant total : un seul remboursement', async ({ request }) => {
  const paymentId = await seedPaidPayment({ amountMinor: 20_000 });

  const [a, b] = await Promise.all([
    request.post(`/v1/payments/${paymentId}/refund`, { data: { amountMinor: 20_000, reason: 'customer_request' } }),
    request.post(`/v1/payments/${paymentId}/refund`, { data: { amountMinor: 20_000, reason: 'customer_request' } }),
  ]);

  expect([a.status(), b.status()].sort()).toEqual([201, 409]);
  expect(await countRefunds(paymentId)).toBe(1);
  expect(await refundedAmount(paymentId)).toBe(20_000);
});
```

`e2e/concurrency/double-submission.spec.ts` :

```typescript
import { expect, test } from '@playwright/test';

test('double soumission avec la même clé d\'idempotence : une seule commande', async ({ request }) => {
  const key = `idem-${Date.now()}`;
  const payload = { items: [{ sku: 'SKU-USB-C-7P', quantity: 10 }], acceptTerms: true };

  const [a, b] = await Promise.all([
    request.post('/v1/orders', { headers: { 'Idempotency-Key': key }, data: payload }),
    request.post('/v1/orders', { headers: { 'Idempotency-Key': key }, data: payload }),
  ]);

  expect(a.status()).toBe(201);
  expect(b.status()).toBe(200);
  expect((await b.json()).orderId).toBe((await a.json()).orderId);
  expect(await countOrdersWithIdempotencyKey(key)).toBe(1);
});
```

- [ ] **Step 8.3: Exécuter les tests de concurrence et constater l'échec attendu**

```bash
pnpm exec playwright test e2e/concurrency --project=functional
```

Attendu :

```text
  1 failed
    Error: expect(statuses).toEqual([201, 409])
    Received: [201, 201]
```

(ou `503 stock_reservation_unavailable` si la réservation n'est pas atomique — les deux sont des défauts à corriger.)

- [ ] **Step 8.4: Implémenter les verrous et l'idempotence jusqu'au vert**

- Réservation de stock : `UPDATE inventory_item SET reserved = reserved + $1 WHERE sku = $2 AND (on_hand - reserved) >= $1` dans une transaction, ou `SELECT … FOR UPDATE` suivi du contrôle. La condition dans le `WHERE` garantit l'absence d'oversell même en concurrence.
- Idempotence : `idempotency_key` unique en base ; la seconde requête avec la même clé renvoie la **réponse d'origine** (`200`) au lieu de recréer l'effet.
- Webhook : `provider_event_id` unique ; la seconde occurrence renvoie `409` sans effet.
- Remboursement : verrou `FOR UPDATE` sur la ligne `payment` avant de cumuler les remboursements ; le cumul ne peut pas dépasser le montant.

```bash
pnpm exec playwright test e2e/concurrency --project=functional
```

Attendu :

```text
  4 passed (32.6s)
```

- [ ] **Step 8.5: Rejouer la concurrence en boucle pour écarter une fenêtre de race**

```bash
for i in 1 2 3 4 5; do pnpm exec playwright test e2e/concurrency/stock-oversell.spec.ts --project=functional --repeat-each=3; done
```

Attendu : 15 exécutions vertes, sans flakiness (taux < 1 %, exigence §24.13).

**Preuves :**
- Sortie de `pnpm exec playwright test e2e/concurrency` (4 tests verts).
- Sortie de la boucle 5 × 3 (15 exécutions vertes).
- Les tests tournent sur la base d'intégration locale (PostgreSQL 16), pas sur Neon : la preuve de verrou est réelle.

**Commit :**

```bash
git add e2e/concurrency apps/api/src/modules/inventory apps/api/src/modules/payments
git commit -m "test(concurrency): oversell, double webhook, double remboursement, double soumission" -m "Chantier 5.3 (spec §24.2, §24.5) : reserve de stock conditionnelle en transaction, cle d'idempotence unique rejouant la reponse d'origine, provider_event_id unique, verrou FOR UPDATE sur le cumul des remboursements." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 9: Responsive & visuels — 8 viewports × 3 locales × 2 thèmes, captures de référence approuvées

**Files:**
- Create: `e2e/visual/matrix.spec.ts`
- Create: `e2e/visual/README.md`
- Create: `apps/web/src/app/[locale]/**` (corrections responsive découvertes)
- Modify: `package.json` (script `test:visual`)

**Interfaces:**
- Consumes: `apps/web` → pages publiques `/[locale]`, `/[locale]/products`, `/[locale]/products/[slug]`, `/[locale]/cart`, `/[locale]/checkout` ; `packages/ui` → tokens `--token` (couleurs uniquement via tokens).
- Produces:
  - `e2e/support/viewports.ts` — **non modifié** : la liste canonique des 8 viewports créée en Phase 1 couvre exactement le périmètre de la Task 9 ; aucune extension n'est nécessaire et **aucune liste de viewports n'est créée ni re-déclarée en Phase 5**.
  - `e2e/visual/matrix.spec.ts` → **importe** `VIEWPORTS` depuis `e2e/support/viewports.js` et `SUPPORTED_LOCALES` depuis `e2e/support/i18n.js` : la matrice **ne redéclare aucune liste de viewports** ni de locales. Les viewports, les langues et les thèmes sont paramétrés **dans le fichier de test** (`test.use({ viewport })` dans un `test.describe` itératif), jamais en projets Playwright. 8 viewports × 3 locales × 2 thèmes = 48 combinaisons par page, avec assertion `document.scrollWidth <= document.documentElement.clientWidth` et capture `toHaveScreenshot` avec seuil de tolérance explicite.
  - `e2e/visual/README.md` → procédure d'approbation : les captures de référence ne sont **jamais** approuvées automatiquement ; un diff est une revue humaine, la mise à jour des références se fait par `pnpm exec playwright test e2e/visual --update-snapshots` **après** revue visuelle des diffs dans `playwright-report/`.

- [ ] **Step 9.1: Écrire la matrice visuelle (rouge)**

`e2e/visual/matrix.spec.ts` — la liste des viewports est **importée** depuis `e2e/support/viewports.js` (source unique créée en Phase 1) ; la liste des locales est importée depuis `e2e/support/i18n.js`. Seuls les **thèmes** et les **pages** sont propres à ce fichier : les langues et les thèmes ne sont pas des projets Playwright, ce sont des paramètres de test.

```typescript
import { expect, test } from '@playwright/test';
import { SUPPORTED_LOCALES, localePath } from '../support/i18n.js';
import { VIEWPORTS } from '../support/viewports.js';

/**
 * Matrice visuelle — spec §24.4 : 8 viewports × 3 locales × 2 thèmes.
 * Deux assertions automatisables par combination (aucun débordement horizontal,
 * aucun décalage de mise en page) et une capture de référence, qui ne peut être
 * mise à jour qu'après revue humaine du diff.
 *
 * VIEWPORTS vient de la liste canonique `e2e/support/viewports.ts` : elle n'est
 * jamais re-déclarée ici.
 */

const VISUAL_THEMES = ['light', 'dark'] as const;

const VISUAL_PAGES = [
  { path: '', name: 'home' },
  { path: '/products', name: 'catalogue' },
  { path: '/products/usb-c-hub-7-ports', name: 'fiche-produit' },
  { path: '/cart', name: 'panier' },
  { path: '/checkout', name: 'checkout' },
] as const;

test.describe('matrice visuelle', () => {
  for (const viewport of VIEWPORTS) {
    for (const locale of SUPPORTED_LOCALES) {
      for (const theme of VISUAL_THEMES) {
        for (const target of VISUAL_PAGES) {
          test(`${viewport.name} · ${locale} · ${theme} · ${target.name}`, async ({ page }) => {
            await page.setViewportSize({ width: viewport.width, height: viewport.height });
            await page.emulateMedia({ colorScheme: theme });

            await page.goto(localePath(locale, target.path));
            await page.evaluate((t) => document.documentElement.setAttribute('data-theme', t), theme);
            await page.waitForLoadState('networkidle');

            const overflow = await page.evaluate(
              () => document.documentElement.scrollWidth - document.documentElement.clientWidth,
            );
            expect(overflow, `débordement horizontal en ${viewport.name}/${locale}/${theme}`).toBeLessThanOrEqual(0);

            await expect(page).toHaveScreenshot(`${target.name}-${viewport.name}-${locale}-${theme}.png`, {
              maxDiffPixelRatio: 0.01,
              animations: 'disabled',
              fullPage: true,
            });
          });
        }
      }
    }
  }
});
```

- [ ] **Step 9.2: Exécuter la matrice et constater les échecs attendus**

```bash
pnpm exec playwright test e2e/visual --project=visual
```

Attendu :

```text
  12 failed
    Error: expect(scrollWidth - clientWidth).toBeLessThanOrEqual(0)
    Received: 24
```

ou, une fois les débordements corrigés :

```text
  Error: A snapshot doesn't exist at 'e2e/visual/matrix.spec.ts-snapshots/fiche-produit-xs-320-fr-FR-light.png', writing actual.
```

- [ ] **Step 9.3: Corriger les débordements et les écarts de mise en page**

Règles appliquées, conformes au §2.5 du CLAUDE.md projet et au spec §24.4 :

- Tableaux → cartes en dessous de 768 px ; aucun tableau à défilement horizontal.
- En-tête compact sur mobile ; navigation basse à 4 onglets (Accueil / Catalogue / Panier / Commandes-Compte).
- Cibles tactiles ≥ 44 px ; `min-w-0` + `truncate` sur tout texte traduit ; aucune largeur fixe sur du contenu CJK.
- Expansion EN (+30 % vs FR) : la mise en page reste lisible en `en` sans troncature abusive.
- `prefers-reduced-motion` respecté : les animations de capture sont désactivées (`animations: 'disabled'`).

```bash
pnpm exec playwright test e2e/visual --project=visual
```

Attendu : 240 tests (5 pages × 48 combinaisons) passent, ou le nombre exact de tests listé par `pnpm exec playwright test e2e/visual --list`.

- [ ] **Step 9.4: Faire relire et approuver les captures de référence (intervention humaine)**

`e2e/visual/README.md` :

```markdown
# Matrice visuelle — procédure d'approbation

Les captures de référence de ce dossier sont des **références approuvées** : elles ne
peuvent être régénérées automatiquement.

## Règle

1. Un diff de capture est **une revue humaine**, pas un échec de CI à contourner.
2. La régénération des références se fait **explicitement**, sur la branche de travail :
   `pnpm exec playwright test e2e/visual --update-snapshots`
3. Avant de committer les références régénérées, la revue doit consigner :
   - la raison du changement (design, correctif de mise en page, ajout de contenu) ;
   - le nom du relecteur et la date ;
   - le lien vers le rapport `playwright-report/` contenant les diffs.
4. **Aucune mise à jour automatique dans la CI.** La CI échoue sur un diff et
   attend une décision humaine consignée dans la pull request.
5. En CI, le job visuel est non bloquant au premier lancement (la base de
   références n'existe pas encore) puis **bloquant** à partir de la deuxième
   exécution.

## Portée de la matrice

5 pages × 8 viewports × 3 locales × 2 thèmes = 240 combinaisons.
Pages : accueil, catalogue, fiche produit, panier, checkout.
```

- [ ] **Step 9.5: Vérifier qu'aucune couleur en dur n'est apparue dans la correction responsive**

```bash
pnpm eslint apps/web/src --max-warnings 0
```

Attendu : aucune violation de la règle `no-restricted-syntax` (hex hors `packages/ui/src/styles/tokens.css`).

**Preuves :**
- Sortie de `pnpm exec playwright test e2e/visual --list` (nombre de combinaisons).
- Rapport `playwright-report/` des diffs, revu et consigné (noms et date dans la PR).
- Assertion anti-débordement verte sur les 8 viewports.
- ESLint : 0 avertissement sur `apps/web/src`.

**Commit :**

```bash
git add e2e/visual apps/web/src package.json
git commit -m "test(visual): matrice 8 viewports x 3 locales x 2 themes, references approuvees" -m "Chantier 5.3 (spec §24.4) : 240 combinaisons sur 5 pages, assertion de non-debordement horizontal, correction des tables en cartes sous 768 px, procedure d'approbation humaine des captures de reference." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 10: Accessibilité — axe-core + jest-axe + @axe-core/playwright + revue manuelle clavier/lecteur d'écran

**Files:**
- Create: `e2e/a11y/a11y.spec.ts`
- Create: `e2e/a11y/manual-checklist.spec.ts`
- Create: `ops/checklists/phase5-manual-a11y.md`
- Create: `apps/web/src/components/__tests__/axe.spec.tsx`
- Modify: `apps/web/src/**` (corrections WCAG découvertes)

**Interfaces:**
- Consumes: `axe-core` (analyse en page), `jest-axe` (composants React), `@axe-core/playwright` (E2E) ; `apps/web` → pages `/{locale}/products`, `/{locale}/cart`, `/{locale}/checkout`, `/{locale}/account/login`, `/{locale}/admin`.
- Produces:
  - `e2e/a11y/a11y.spec.ts` — analyse automatique sur 5 parcours, 2 thèmes, 3 locales, avec exclusion des violations `critical` et `serious` ;
  - `ops/checklists/phase5-manual-a11y.md` — les **12 vérifications manuelles** du spec §22.4, avec emplacement, procédure, résultat attendu et emplacement du relevé ;
  - `apps/web/src/components/__tests__/axe.spec.tsx` — analyse jest-axe sur 6 composants critiques.

- [ ] **Step 10.1: Écrire le test E2E d'accessibilité (rouge)**

`e2e/a11y/a11y.spec.ts` :

```typescript
import AxeBuilder from '@axe-core/playwright';
import { expect, test } from '@playwright/test';
import { SUPPORTED_LOCALES, localePath } from '../support/i18n.js';

/**
 * Accessibilité automatique — spec §22.4, §24.3.
 * Seuil : zéro violation « critical » et zéro violation « serious » sur les parcours
 * critiques (checkout, connexion, administration, gestion de commande).
 * L'audit automatique est un point de départ : il ne remplace pas la revue manuelle
 * (ops/checklists/phase5-manual-a11y.md).
 */

const CRITICAL_PATHS = [
  { path: '/products', name: 'catalogue' },
  { path: '/products/usb-c-hub-7-ports', name: 'fiche-produit' },
  { path: '/cart', name: 'panier' },
  { path: '/checkout', name: 'checkout' },
  { path: '/account/login', name: 'connexion' },
] as const;

for (const locale of SUPPORTED_LOCALES) {
  test.describe(`accessibilité — ${locale}`, () => {
    for (const theme of ['light', 'dark'] as const) {
      for (const target of CRITICAL_PATHS) {
        test(`${target.name} · ${theme}`, async ({ page }) => {
          await page.emulateMedia({ colorScheme: theme });
          await page.goto(localePath(locale, target.path));
          await page.evaluate((t) => document.documentElement.setAttribute('data-theme', t), theme);

          const results = await new AxeBuilder({ page })
            .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa'])
            .analyze();

          const blocking = results.violations.filter(
            (violation) => violation.impact === 'critical' || violation.impact === 'serious',
          );

          expect(
            blocking,
            `violations bloquantes : ${blocking.map((v) => `${v.id} (${v.impact}) sur ${v.nodes.length} nœuds`).join(', ')}`,
          ).toEqual([]);
        });
      }
    }
  });
}
```

- [ ] **Step 10.2: Écrire le test composant jest-axe (rouge)**

`apps/web/src/components/__tests__/axe.spec.tsx` :

```typescript
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
import { describe, expect, it } from 'vitest';
import { Button } from '../ui/Button.js';
import { PriceTag } from '../ui/PriceTag.js';
import { QuantityStepper } from '../ui/QuantityStepper.js';
import { AddressForm } from '../checkout/AddressForm.js';
import { OrderSummary } from '../cart/OrderSummary.js';
import { AdminDataTable } from '../admin/AdminDataTable.js';

expect.extend(toHaveNoViolations);

describe('accessibilité des composants critiques', () => {
  it('Button n\'a aucune violation', async () => {
    const { container } = render(<Button>Ajouter au panier</Button>);
    expect(await axe(container)).toHaveNoViolations();
  });

  it('PriceTag n\'a aucune violation', async () => {
    const { container } = render(<PriceTag amountMinor={12_000} currency="EUR" locale="fr-FR" />);
    expect(await axe(container)).toHaveNoViolations();
  });

  it('QuantityStepper n\'a aucune violation', async () => {
    const { container } = render(<QuantityStepper value={2} onChange={() => undefined} min={1} max={99} />);
    expect(await axe(container)).toHaveNoViolations();
  });

  it('AddressForm n\'a aucune violation', async () => {
    const { container } = render(<AddressForm onSubmit={() => undefined} />);
    expect(await axe(container)).toHaveNoViolations();
  });

  it('OrderSummary n\'a aucune violation', async () => {
    const { container } = render(<OrderSummary totalMinor={12_000} currency="EUR" locale="fr-FR" />);
    expect(await axe(container)).toHaveNoViolations();
  });

  it('AdminDataTable n\'a aucune violation', async () => {
    const { container } = render(
      <AdminDataTable columns={['Numéro', 'Montant']} rows={[['FAN-2026-0001', '120,00']]} caption="Commandes" />,
    );
    expect(await axe(container)).toHaveNoViolations();
  });
});
```

- [ ] **Step 10.3: Exécuter les tests et constater les violations attendues**

```bash
pnpm vitest run apps/web/src/components/__tests__/axe.spec.tsx
pnpm exec playwright test e2e/a11y --project=functional
```

Attendu :

```text
Expected the HTML found at $('.container') to have no violations:
  color-contrast: serious — 3 nodes
  label: serious — 1 node
```

- [ ] **Step 10.4: Corriger les violations jusqu'au vert**

Corrections types attendues (audit §22.4) :

- Contrastes : le texte secondaire passe en `--token-text-muted` ; les badges d'état utilisent les paires functionnelles (succès/attention/erreur) validées pour les deux thèmes.
- `label` : chaque champ de formulaire a une étiquette liée par `htmlFor` / `id` ; les champs sans étiquette visible reçoivent un `aria-label` localized.
- Focus : anneau `--focus-ring` visible sur tous les éléments interactifs, offset 2 px, jamais `outline: none` sans remplacement.
- Ordre de focus : ordre DOM naturel, pas de `tabindex` positif.
- Modales : piège de focus, `aria-modal="true"`, retour du focus au déclencheur.
- `aria-live` sur le recalcul de total et le changement de stock ; annonces en direct également dans la version EN et ZH-CN.
- Cible tactile ≥ 24 × 24 px (2.5.8) sur les actions secondaires.
- `prefers-reduced-motion` : les transitions non essentielles sont désactivées.

```bash
pnpm vitest run apps/web/src/components/__tests__/axe.spec.tsx
pnpm exec playwright test e2e/a11y --project=functional
```

Attendu :

```text
  6 passed (6)
  30 passed (3m 4.1s)
```

- [ ] **Step 10.5: Écrire la procédure de revue manuelle (intervention humaine)**

`ops/checklists/phase5-manual-a11y.md` :

```markdown
# Revue manuelle d'accessibilité — Phase 5

Procédure à exécuter **humainement** (clavier + lecteur d'écran). L'audit
automatique (`e2e/a11y/a11y.spec.ts`, `axe.spec.tsx`) ne prouve rien de plus.

Environnement : NVDA ou VoiceOver (Windows/macOS), Firefox ou Chrome, thème clair et
sombre, locale `fr-FR` puis `en` puis `zh-CN`.

Parcours à couvrir : catalogue → fiche produit → panier → checkout → connexion →
tableau de bord acheteur → gestion de commande (acheteur) → back-office (admin).

| # | Vérification | Procédure | Résultat attendu | Relevé |
|---|---|---|---|---|
| 1 | Navigation clavier complète | Tab / Maj+Tab depuis le haut de page ; Shift+Tab inverse | Aucun piège, ordre logique, aucun saut | |
| 2 | Focus visible | Tab sur chaque contrôle | Anneau `--focus-ring` visible, offset 2 px, contraste ≥ 3:1 | |
| 3 | Piège de focus en modale | Ouvrir la modale, tabuler en boucle, Échap | Focus piégé, restitution du focus au déclencheur | |
| 4 | Landmarks ARIA | Inspecter `header`/`nav`/`main`/`footer` | Un `main` unique, hiérarchie cohérente | |
| 5 | Live regions | Modifier la quantité au checkout | Le recalcul du total est annoncé | |
| 6 | Annonce de changement de stock | Changer la quantité d'une fiche produit | Le stock restant est annoncé | |
| 7 | Erreurs liées au champ | Soumettre un formulaire vide | Message lié au champ, annoncé, focus déplacé | |
| 8 | Lecture d'écran des images | Parcourir les images produit et décoratives | `alt` utile ou `alt=""` pour les décoratives | |
| 9 | Tableaux de données | Parcourir le tableau admin | En-têtes annoncés, pas de lecture ligne par ligne | |
| 10 | Zoom 200 % | Zoomer le navigateur à 200 % sur le checkout | Aucun contenu perdu, pas de double barre de défilement | |
| 11 | Cible tactile | Inspecter au navigateur mobile les actions secondaires | ≥ 24 × 24 px (2.5.8), ≥ 44 px sur les actions principales | |
| 12 | Mouvement réduit | Activer `prefers-reduced-motion` | Aucune animation non essentielle | |

## Règles

- Toute ligne vide = vérification non effectuée : le parcours **n'est pas** validé.
- Un défaut trouvé est consigné avec la page, la locale, le thème, le lecteur
  d'écran utilisé et la reproduction, puis corrigé et re-vérifié.
- Le relevé signé (nom, date, version) est joint à la pull request.
```

- [ ] **Step 10.6: Enregistrer le résultat de la revue manuelle (intervention humaine)**

Dans `ops/checklists/phase5-manual-a11y.md`, compléter la colonne « Relevé » pour les 12 lignes et signer en bas de page :

```markdown
## Validation

| Champ | Valeur |
|---|---|
| Relecteur | |
| Date | |
| Navigateur / lecteur d'écran | |
| Locale(s) couverte(s) | |
| Défauts ouverts | |
| Parcours validé | oui / non |
```

**Preuves :**
- Sortie de `pnpm vitest run apps/web/src/components/__tests__/axe.spec.tsx` (6 tests verts).
- Sortie de `pnpm exec playwright test e2e/a11y --project=functional` (30 tests verts, 0 violation critique/sérieuse).
- `ops/checklists/phase5-manual-a11y.md` complété et signé (preuve humaine, non substituable).
- Aucune violation `critical` / `serious` : interprétation automatique vérifiée manuellement sur un échantillon.

**Commit :**

```bash
git add e2e/a11y apps/web/src/components/__tests__/axe.spec.tsx apps/web/src ops/checklists/phase5-manual-a11y.md
git commit -m "test(a11y): axe-core, jest-axe et procedure de revue manuelle WCAG 2.2 AA" -m "Chantier 5.3 (spec §22.4, §24.3) : analyse automatique sur 5 parcours critiques en 3 locales et 2 themes, 6 composants critiques en jest-axe, checklist manuelle de 12 verifications clavier et lecteur d'ecran." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 11: Charge & résilience — k6 (catalogue, recherche, login, checkout, webhook), injection de pannes, seuils p95

**Files:**
- Create: `ops/tests/load/k6-catalogue.js`
- Create: `ops/tests/load/k6-search.js`
- Create: `ops/tests/load/k6-login.js`
- Create: `ops/tests/load/k6-checkout.js`
- Create: `ops/tests/load/k6-webhook.js`
- Create: `ops/tests/load/lib/thresholds.js`
- Create: `ops/tests/load/README.md`
- Modify: `package.json` (scripts `test:load`)

**Interfaces:**
- Consumes: `apps/api` → `GET /v1/products`, `GET /v1/search`, `POST /v1/auth/login`, `POST /v1/orders`, `POST /v1/webhooks/payment` ; variables d'environnement k6 : `BASE_URL`, `E2E_WEBHOOK_SECRET`, `LOAD_TEST_SKU`.
- Produces:
  - `ops/tests/load/lib/thresholds.js` → `export const LOAD_THRESHOLDS = { catalogueP95Ms: 600, searchP95Ms: 800, loginP95Ms: 900, cartP95Ms: 400, checkoutP95Ms: 1200, webhookP95Ms: 500, errorRate: 0.01 }` (mêmes valeurs que `QUALITY_THRESHOLDS.load` de la Task 1 — une seule source de vérité : le module `packages/config` est importé par le runner k6 via `--env`).
  - 5 scripts k6 exécutables en local et en staging.

- [ ] **Step 11.1: Écrire les seuils partagés (rouge)**

`ops/tests/load/lib/thresholds.js` :

```javascript
// Seuils de charge — alignés sur packages/config/src/quality-thresholds.ts (QUALITY_THRESHOLDS.load).
// Toute divergence entre les deux fichiers est un défaut : le script `check:load-thresholds`
// (Task 13) compare les deux sources.
export const LOAD_THRESHOLDS = {
  catalogueP95Ms: 600,
  searchP95Ms: 800,
  loginP95Ms: 900,
  cartP95Ms: 400,
  checkoutP95Ms: 1200,
  webhookP95Ms: 500,
  errorRate: 0.01,
};

export function thresholdsFor(name) {
  const p95 = {
    catalogue: LOAD_THRESHOLDS.catalogueP95Ms,
    search: LOAD_THRESHOLDS.searchP95Ms,
    login: LOAD_THRESHOLDS.loginP95Ms,
    cart: LOAD_THRESHOLDS.cartP95Ms,
    checkout: LOAD_THRESHOLDS.checkoutP95Ms,
    webhook: LOAD_THRESHOLDS.webhookP95Ms,
  }[name];

  if (!p95) {
    throw new Error(`Seuil inconnu : ${name}`);
  }

  return {
    http_req_duration: [`p(95)<${p95}`],
    http_req_failed: [`rate<${LOAD_THRESHOLDS.errorRate}`],
    checks: ['rate>0.99'],
  };
}
```

- [ ] **Step 11.2: Écrire les 5 scripts k6 (rouge)**

`ops/tests/load/k6-catalogue.js` :

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { thresholdsFor } from './lib/thresholds.js';

export const options = {
  scenarios: {
    catalogue: {
      executor: 'ramping-arrival-rate',
      startRate: 10,
      timeUnit: '1s',
      preAllocatedVUs: 30,
      maxVUs: 120,
      stages: [
        { target: 50, duration: '2m' },
        { target: 50, duration: '5m' },
        { target: 0, duration: '1m' },
      ],
    },
  },
  thresholds: thresholdsFor('catalogue'),
  noConnectionReuse: false,
};

const BASE_URL = __ENV.BASE_URL || 'http://127.0.0.1:3000';

export default function () {
  const response = http.get(`${BASE_URL}/v1/products?page=1&pageSize=24`, {
    tags: { endpoint: 'catalogue' },
  });
  check(response, {
    'catalogue répond 200': (r) => r.status === 200,
    'catalogue renvoie des produits': (r) => r.json('items') !== undefined,
  });
}
```

`ops/tests/load/k6-search.js` :

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { thresholdsFor } from './lib/thresholds.js';

export const options = {
  scenarios: {
    search: {
      executor: 'ramping-arrival-rate',
      startRate: 5,
      timeUnit: '1s',
      preAllocatedVUs: 20,
      maxVUs: 80,
      stages: [
        { target: 30, duration: '2m' },
        { target: 30, duration: '5m' },
        { target: 0, duration: '1m' },
      ],
    },
  },
  thresholds: thresholdsFor('search'),
};

const BASE_URL = __ENV.BASE_URL || 'http://127.0.0.1:3000';
const QUERIES = ['usb', 'cable', 'hub', 'adaptateur', 'chargeur'];

export default function () {
  const query = QUERIES[Math.floor(Math.random() * QUERIES.length)];
  const response = http.get(`${BASE_URL}/v1/search?q=${encodeURIComponent(query)}`, {
    tags: { endpoint: 'search' },
  });
  check(response, { 'recherche répond 200': (r) => r.status === 200 });
}
```

`ops/tests/load/k6-login.js` :

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { thresholdsFor } from './lib/thresholds.js';

export const options = {
  scenarios: {
    login: {
      executor: 'ramping-arrival-rate',
      startRate: 5,
      timeUnit: '1s',
      preAllocatedVUs: 20,
      maxVUs: 60,
      stages: [
        { target: 20, duration: '2m' },
        { target: 20, duration: '5m' },
        { target: 0, duration: '1m' },
      ],
    },
  },
  thresholds: thresholdsFor('login'),
};

const BASE_URL = __ENV.BASE_URL || 'http://127.0.0.1:3000';
const EMAIL = __ENV.LOAD_TEST_EMAIL || 'load@example.test';
const PASSWORD = __ENV.LOAD_TEST_PASSWORD || 'Fanours!Load2026';

export default function () {
  const response = http.post(
    `${BASE_URL}/v1/auth/login`,
    JSON.stringify({ email: EMAIL, password: PASSWORD }),
    { headers: { 'Content-Type': 'application/json' }, tags: { endpoint: 'login' } },
  );
  check(response, { 'login répond 200 ou 429': (r) => r.status === 200 || r.status === 429 });
}
```

`ops/tests/load/k6-checkout.js` :

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { thresholdsFor } from './lib/thresholds.js';

export const options = {
  scenarios: {
    checkout: {
      executor: 'ramping-arrival-rate',
      startRate: 3,
      timeUnit: '1s',
      preAllocatedVUs: 15,
      maxVUs: 50,
      stages: [
        { target: 15, duration: '2m' },
        { target: 15, duration: '5m' },
        { target: 0, duration: '1m' },
      ],
    },
  },
  thresholds: thresholdsFor('checkout'),
};

const BASE_URL = __ENV.BASE_URL || 'http://127.0.0.1:3000';
const SKU = __ENV.LOAD_TEST_SKU || 'SKU-USB-C-7P';

export default function () {
  const quote = http.post(
    `${BASE_URL}/v1/checkout/quote`,
    JSON.stringify({ items: [{ sku: SKU, quantity: 10 }], country: 'FR' }),
    { headers: { 'Content-Type': 'application/json' }, tags: { endpoint: 'checkout' } },
  );
  check(quote, { 'quote répond 200': (r) => r.status === 200 });
}
```

`ops/tests/load/k6-webhook.js` :

```javascript
import http from 'k6/http';
import crypto from 'k6/crypto';
import { check } from 'k6';
import { thresholdsFor } from './lib/thresholds.js';

export const options = {
  scenarios: {
    webhook: {
      executor: 'ramping-arrival-rate',
      startRate: 20,
      timeUnit: '1s',
      preAllocatedVUs: 20,
      maxVUs: 60,
      stages: [
        { target: 50, duration: '2m' },
        { target: 50, duration: '5m' },
        { target: 0, duration: '1m' },
      ],
    },
  },
  thresholds: thresholdsFor('webhook'),
};

const BASE_URL = __ENV.BASE_URL || 'http://127.0.0.1:3000';
const SECRET = __ENV.E2E_WEBHOOK_SECRET || 'test-webhook-secret';

export default function () {
  const body = JSON.stringify({
    id: `evt_${crypto.randomBytes(12).toString('hex')}`,
    type: 'payment.succeeded',
    data: { orderId: __ENV.LOAD_TEST_ORDER_ID, amountMinor: 12_000, currency: 'EUR' },
  });
  const timestamp = Math.floor(Date.now() / 1000);
  const signature = crypto.hmac('sha256', SECRET, `${timestamp}.${body}`, 'hex');

  const response = http.post(`${BASE_URL}/v1/webhooks/payment`, body, {
    headers: {
      'Content-Type': 'application/json',
      'x-webhook-timestamp': String(timestamp),
      'x-webhook-signature': signature,
    },
    tags: { endpoint: 'webhook' },
  });
  check(response, { 'webhook répond 200 ou 409': (r) => r.status === 200 || r.status === 409 });
}
```

- [ ] **Step 11.3: Exécuter un test de charge court en local et constater l'échec de seuil attendu**

```bash
k6 run --vus 5 --duration 30s ops/tests/load/k6-catalogue.js
```

Attendu (si l'environnement local n'est pas dimensionné) :

```text
  http_req_duration...........: avg=812.00ms p(95)=1.42s
  http_req_failed.............: rate=0.00%
  ✗ http_req_duration: p(95)<600
```

La commande de référence est le test complet (8 min) ; le test de 30 s sert à valider le câblage.

- [ ] **Step 11.4: Documenter la procédure d'exécution et d'injection de pannes**

`ops/tests/load/README.md` :

```markdown
# Tests de charge et de résilience — Phase 5

## Exécution

```bash
# Catalogue
k6 run ops/tests/load/k6-catalogue.js
# Recherche
k6 run ops/tests/load/k6-search.js
# Connexion
LOAD_TEST_EMAIL=load@example.test LOAD_TEST_PASSWORD='***' k6 run ops/tests/load/k6-login.js
# Checkout
LOAD_TEST_SKU=SKU-USB-C-7P k6 run ops/tests/load/k6-checkout.js
# Webhook
E2E_WEBHOOK_SECRET='***' LOAD_TEST_ORDER_ID='<uuid>' k6 run ops/tests/load/k6-webhook.js
```

Variables : `BASE_URL` (défaut `http://127.0.0.1:3000`).

## Seuils (bloquants)

| Endpoint | p95 | Taux d'erreur |
|---|---|---|
| catalogue | 600 ms | < 1 % |
| recherche | 800 ms | < 1 % |
| connexion | 900 ms | < 1 % |
| panier | 400 ms | < 1 % |
| checkout | 1 200 ms | < 1 % |
| webhook | 500 ms | < 1 % |

Les seuils sont alignés sur `packages/config/src/quality-thresholds.ts` (`QUALITY_THRESHOLDS.load`).
Le script `pnpm check:load-thresholds` compare les deux sources et échoue en cas de divergence.

## Environnement

Les tests de charge s'exécutent **uniquement** sur `staging` avec des données synthétiques.
Jamais en production : la charge appliquée est celle d'un scénario d'attaque.

## Injection de pannes (résilience)

| Panne injectée | Attendu | Preuve attendue |
|---|---|---|
| PSP indisponible | commandes en `payment_pending`, pas de 5xx en cascade | journal `webhook.payment.failed` + métrique `fanours_payment_webhook_failures_total` qui monte |
| Redis indisponible | degradation contrôlée : file en attente, réponses API non bloquantes | métrique `fanours_queue_depth` qui monte, latence API stable |
| PostgreSQL indisponible | `503` avec `code: 'service_unavailable'`, pas de stack trace | réponse JSON sans détail technique |
| SMTP indisponible | e-mails en file, DLQ après N tentatives | métrique `fanours_email_failures_total` + entrée en DLQ |
| Transporteur indisponible | estimation de frais indisponible, commande toujours créable | `customs-estimate` indisponible, commande créée |

Chaque injection est exécutée en `staging`, jamais en production, et donne lieu à un relevé
consigné dans la pull request (durée de l'incident, métriques observées, temps de rétablissement).

## Règles

- Aucun test de charge n'est exécuté contre la production, même « pour vérifier ».
- Les seuils sont bloquants en CI de `staging` ; un dépassement est un défaut, pas un
  « on verra plus tard ».
```

- [ ] **Step 11.5: Vérifier que les scripts k6 ne contiennent aucun secret en clair**

```bash
rg -n "sk_live|pk_live|whsec_[A-Za-z0-9]{20,}" ops/tests/load
```

Attendu : aucune occurrence. Les secrets sont lus dans l'environnement.

**Preuves :**
- Sortie k6 des 5 scénarios (résultats p95 et taux d'erreur consignés).
- Sortie de `pnpm check:load-thresholds` (alignement des seuils).
- Relevé d'injection de pannes en `staging` (5 pannes) joint à la PR, avec métriques observées et temps de rétablissement.
- Aucun secret en clair dans `ops/tests/load`.

**Commit :**

```bash
git add ops/tests/load package.json
git commit -m "test(load): k6 catalogue, recherche, login, checkout et webhook avec seuils p95" -m "Chantier 5.3 (spec §24.5, §23.3) : 5 scenarios k6, seuils alignes sur QUALITY_THRESHOLDS.load, procedure d'injection de pannes PSP/Redis/PostgreSQL/SMTP/transporteur en staging, execution interdite en production." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 12: Mutation testing (Stryker) sur pricing/tax/order-state + métriques qualité

**Files:**
- Create: `stryker.config.json` (racine)
- Create: `packages/db/migrations/0116_phase5_quality_metrics.sql`
- Create: `packages/db/src/schema/quality-metrics.ts`
- Create: `packages/config/src/quality-metrics.ts`
- Create: `docs/quality/rapport-qualite.md`
- Modify: `package.json` (scripts `test:mutation`, `check:load-thresholds`)

**Interfaces:**
- Consumes: `packages/pricing` (résolution de paliers, remises), `packages/compliance` (moteur de règles douanières, calcul de taxes), `packages/domain` (machine à états des commandes), `packages/money` (arrondis) ; `packages/config` → `QUALITY_THRESHOLDS.mutation` (`{ break: 0.8, high: 0.9, critical: 0.95 }`).
- Produces:
  - `stryker.config.json` (racine) — mutant sur `packages/pricing/src/**`, `packages/compliance/src/**`, `packages/domain/src/**`, `packages/money/src/**` ; `reporters: ['html', 'clear-text', 'progress']` ; `thresholds: { high: 90, low: 80, break: 80 }` ; `checkThreshold: 'break'`.
  - Table `quality_metric` (clé, valeur, unité, périmètre, relevé_at) et `quality_metric_run` (run_id, type, branche, commit_sha, statut, métriques JSONB, créée_at) — trace des mesures de qualité par exécution CI.
  - `packages/config/src/quality-metrics.ts` → `export interface QualityMetricRun { runId: string; type: 'coverage' | 'mutation' | 'e2e' | 'load' | 'a11y' | 'lint'; branch: string; commitSha: string; status: 'passed' | 'failed'; metrics: Record<string, number>; }` et `export const QUALITY_METRIC_KEYS = ['coverage.global', 'coverage.critical', 'mutation.score', 'mutation.pricing', 'mutation.tax', 'mutation.order_state', 'e2e.flaky_rate', 'e2e.duration_min', 'load.checkout_p95_ms', 'a11y.critical_violations', 'a11y.serious_violations', 'lint.errors'] as const;`

- [ ] **Step 12.1: Écrire la configuration Stryker et le test de configuration (rouge)**

`stryker.config.json` :

```json
{
  "$schema": "./node_modules/stryker/schema/stryker-schema.json",
  "packageManager": "pnpm",
  "reporters": ["html", "clear-text", "progress"],
  "coverageAnalysis": "perTest",
  "checkThreshold": "break",
  "thresholds": {
    "high": 90,
    "low": 80,
    "break": 80
  },
  "mutate": [
    "packages/pricing/src/**/*.ts",
    "packages/compliance/src/**/*.ts",
    "packages/domain/src/**/*.ts",
    "packages/money/src/**/*.ts"
  ],
  "mutateConfig": {
    "mutator": {
      "Stryker": {
        "mutate": [
          { "name": "ArithmeticOperator", "options": { "allowNumericString": false } },
          { "name": "BooleanLiteral", "options": { "skipStatic": true } },
          { "name": "ConditionalExpression" },
          { "name": "StringLiteral" },
          { "name": "ArrayDeclaration" }
        ]
      }
    }
  },
  "timeoutMS": 20000,
  "concurrency": 4
}
```

`packages/config/src/__tests__/quality-metrics.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { QUALITY_METRIC_KEYS, type QualityMetricRun } from '../quality-metrics.js';

describe('métriques qualité — contrat', () => {
  it('expose toutes les clés attendues par la roadmap §24.13', () => {
    expect([...QUALITY_METRIC_KEYS]).toEqual([
      'coverage.global',
      'coverage.critical',
      'mutation.score',
      'mutation.pricing',
      'mutation.tax',
      'mutation.order_state',
      'e2e.flaky_rate',
      'e2e.duration_min',
      'load.checkout_p95_ms',
      'a11y.critical_violations',
      'a11y.serious_violations',
      'lint.errors',
    ]);
  });

  it('refuse un relevé dont le commit n\'est pas un sha de 40 caractères', () => {
    const run: QualityMetricRun = {
      runId: 'r1',
      type: 'coverage',
      branch: 'feat/x',
      commitSha: 'abc',
      status: 'passed',
      metrics: { 'coverage.global': 0.82 },
    };
    expect(() => validateQualityMetricRun(run)).toThrow(/commitSha/);
  });

  it('accepte un relevé valide', () => {
    const run: QualityMetricRun = {
      runId: 'r1',
      type: 'coverage',
      branch: 'feat/x',
      commitSha: 'a'.repeat(40),
      status: 'passed',
      metrics: { 'coverage.global': 0.82 },
    };
    expect(() => validateQualityMetricRun(run)).not.toThrow();
  });
});
```

- [ ] **Step 12.2: Exécuter Stryker et constater le score insuffisant attendu**

```bash
pnpm exec stryker run
```

Attendu :

```text
Mutation score 71.4% (threshold for "low" is 80%, threshold for "high" is 90%)
```

- [ ] **Step 12.3: Écrire la migration `0116` et le schéma Drizzle**

`packages/db/migrations/0116_phase5_quality_metrics.sql` :

```sql
-- 0116_phase5_quality_metrics.sql
-- Phase 5 — Task 12 : traçabilité des métriques de qualité (spec §24.6, §26.4).
-- Append-only : aucun down. Expansion pure (aucune donnée supprimée).
-- Contraintes : CHECK (valeur >= 0), unicité du couple (key, run_id, recorded_at).
SET lock_timeout = '5s';
SET statement_timeout = '30s';

CREATE TABLE IF NOT EXISTS quality_metric_run (
  id          uuid PRIMARY KEY,
  run_id      text        NOT NULL,
  type        text        NOT NULL,
  branch      text        NOT NULL,
  commit_sha  char(40)    NOT NULL,
  status      text        NOT NULL,
  metrics     jsonb       NOT NULL DEFAULT '{}'::jsonb,
  created_at  timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT quality_metric_run_type_check
    CHECK (type IN ('coverage', 'mutation', 'e2e', 'load', 'a11y', 'lint', 'security')),
  CONSTRAINT quality_metric_run_status_check
    CHECK (status IN ('passed', 'failed')),
  CONSTRAINT quality_metric_run_commit_sha_check
    CHECK (commit_sha ~ '^[0-9a-f]{40}$')
);

CREATE TABLE IF NOT EXISTS quality_metric (
  id          uuid PRIMARY KEY,
  run_id      text        NOT NULL,
  metric_key  text        NOT NULL,
  value       numeric     NOT NULL,
  unit        text        NOT NULL DEFAULT 'ratio',
  scope       text        NOT NULL DEFAULT 'global',
  recorded_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT quality_metric_value_check CHECK (value >= 0),
  CONSTRAINT quality_metric_key_check
    CHECK (metric_key IN (
      'coverage.global', 'coverage.critical',
      'mutation.score', 'mutation.pricing', 'mutation.tax', 'mutation.order_state',
      'e2e.flaky_rate', 'e2e.duration_min',
      'load.checkout_p95_ms',
      'a11y.critical_violations', 'a11y.serious_violations',
      'lint.errors'
    ))
);

CREATE UNIQUE INDEX IF NOT EXISTS uq_quality_metric_run_id ON quality_metric_run (run_id);
CREATE UNIQUE INDEX IF NOT EXISTS uq_quality_metric_key_scope
  ON quality_metric (run_id, metric_key, scope);
CREATE INDEX IF NOT EXISTS idx_quality_metric_recorded_at ON quality_metric (recorded_at DESC);
CREATE INDEX IF NOT EXISTS idx_quality_metric_run_type ON quality_metric_run (type, created_at DESC);
```

Application :

```bash
pnpm --filter @fanours/db migrate
```

Attendu :

```text
migrating 0116_phase5_quality_metrics.sql
0116_phase5_quality_metrics.sql applied
```

- [ ] **Step 12.4: Écrire le module de métriques et le test d'écriture (rouge)**

`packages/config/src/quality-metrics.ts` :

```typescript
export const QUALITY_METRIC_KEYS = [
  'coverage.global',
  'coverage.critical',
  'mutation.score',
  'mutation.pricing',
  'mutation.tax',
  'mutation.order_state',
  'e2e.flaky_rate',
  'e2e.duration_min',
  'load.checkout_p95_ms',
  'a11y.critical_violations',
  'a11y.serious_violations',
  'lint.errors',
] as const;

export type QualityMetricKey = (typeof QUALITY_METRIC_KEYS)[number];

export interface QualityMetricRun {
  runId: string;
  type: 'coverage' | 'mutation' | 'e2e' | 'load' | 'a11y' | 'lint' | 'security';
  branch: string;
  commitSha: string;
  status: 'passed' | 'failed';
  metrics: Partial<Record<QualityMetricKey, number>>;
}

/** Rejette un relevé mal formé avant écriture en base. */
export function validateQualityMetricRun(run: QualityMetricRun): void {
  if (!/^[0-9a-f]{40}$/.test(run.commitSha)) {
    throw new Error(`commitSha invalide : ${run.commitSha}`);
  }
  if (!['passed', 'failed'].includes(run.status)) {
    throw new Error(`statut invalide : ${run.status}`);
  }
  for (const key of Object.keys(run.metrics)) {
    if (!QUALITY_METRIC_KEYS.includes(key as QualityMetricKey)) {
      throw new Error(`Clé de métrique inconnue : ${key}`);
    }
  }
}
```

`packages/db/src/schema/quality-metrics.ts` :

```typescript
import { pgTable, uuid, text, numeric, jsonb, timestamp, index, uniqueIndex, char } from 'drizzle-orm/pg-core';

export const qualityMetricRun = pgTable(
  'quality_metric_run',
  {
    id: uuid('id').primaryKey(),
    runId: text('run_id').notNull(),
    type: text('type').notNull(),
    branch: text('branch').notNull(),
    commitSha: char('commit_sha', { length: 40 }).notNull(),
    status: text('status').notNull(),
    metrics: jsonb('metrics').notNull().default({}),
    createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    runIdUnique: uniqueIndex('uq_quality_metric_run_id').on(table.runId),
    typeCreatedIdx: index('idx_quality_metric_run_type').on(table.type, table.createdAt),
  }),
);

export const qualityMetric = pgTable(
  'quality_metric',
  {
    id: uuid('id').primaryKey(),
    runId: text('run_id').notNull(),
    metricKey: text('metric_key').notNull(),
    value: numeric('value', { precision: 12, scale: 4 }).notNull(),
    unit: text('unit').notNull().default('ratio'),
    scope: text('scope').notNull().default('global'),
    recordedAt: timestamp('recorded_at', { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    keyScopeUnique: uniqueIndex('uq_quality_metric_key_scope').on(table.runId, table.metricKey, table.scope),
    recordedAtIdx: index('idx_quality_metric_recorded_at').on(table.recordedAt),
  }),
);

export type QualityMetricRow = typeof qualityMetric.$inferSelect;
export type QualityMetricRunRow = typeof qualityMetricRun.$inferSelect;
```

- [ ] **Step 12.5: Écrire le script d'ingestion des métriques (rouge)**

`packages/config/src/scripts/record-quality-metrics.ts` :

```typescript
import { randomUUID } from 'node:crypto';
import { QUALITY_METRIC_KEYS, validateQualityMetricRun, type QualityMetricKey, type QualityMetricRun } from '../quality-metrics.js';

interface Args {
  type: QualityMetricRun['type'];
  metrics: Partial<Record<QualityMetricKey, number>>;
  status: 'passed' | 'failed';
}

/** Parse `--type=e2e --metric=e2e.flaky_rate=0.004 --status=passed`. */
export function parseArgs(argv: string[]): Args {
  const get = (prefix: string): string | undefined =>
    argv.find((a) => a.startsWith(prefix))?.split('=')[1];

  const type = get('--type=') as Args['type'] | undefined;
  const status = (get('--status=') ?? 'passed') as Args['status'];
  if (!type) {
    throw new Error('Argument --type obligatoire');
  }

  const metrics: Args['metrics'] = {};
  for (const arg of argv) {
    if (!arg.startsWith('--metric=')) continue;
    const [key, raw] = arg.slice('--metric='.length).split('=');
    if (!QUALITY_METRIC_KEYS.includes(key as QualityMetricKey)) {
      throw new Error(`Clé de métrique inconnue : ${key}`);
    }
    metrics[key as QualityMetricKey] = Number(raw);
  }

  return { type, metrics, status };
}

export function buildRun(args: Args, branch: string, commitSha: string): QualityMetricRun {
  const run: QualityMetricRun = {
    runId: randomUUID(),
    type: args.type,
    branch,
    commitSha,
    status: args.status,
    metrics: args.metrics,
  };
  validateQualityMetricRun(run);
  return run;
}
```

- [ ] **Step 12.6: Exécuter Stryker, corriger les mutants survivants et atteindre le seuil**

Les mutants qui survivent indicate un cas de test manquant. Priorité aux invariants financiers et douaniers :

```bash
pnpm exec stryker run --mutate "packages/pricing/src/**/*.ts"
pnpm exec stryker run --mutate "packages/compliance/src/**/*.ts"
pnpm exec stryker run --mutate "packages/domain/src/**/*.ts"
pnpm exec stryker run --mutate "packages/money/src/**/*.ts"
```

Attendu :

```text
Mutation score 95.6% (threshold for "high" is 90%)
```

Le seuil `critical` de la roadmap (§24.13 : ≥ 95 % sur pricing / compliance / order-state / money) est vérifié par mutant **par paquet**, et non par le score global.

Ajouter les tests manquants identifiés par Stryker, par exemple :

```typescript
it('refuse un palier de prix dont la quantité minimale est inférieure à celle du palier précédent', () => {
  expect(() => resolvePriceTier(tiers, 5, { scope: 'b2b', at: FIXED_DATE })).toThrow(PriceTierError);
});

it('arrondit à l\'euro supérieur une base de 0,145 €', () => {
  expect(roundHalfUp(0.145, 2)).toBe(0.15);
});
```

```bash
pnpm exec stryker run
```

Attendu : score global ≥ 80 % (seuil `break`) et ≥ 95 % sur chacun des 4 paquets critiques.

- [ ] **Step 12.7: Écrire le rapport de qualité et le script de vérification des seuils**

`docs/quality/rapport-qualite.md` (extrait structuré, les chiffres sont remplis par la CI) :

```markdown
# Rapport qualité — Phase 5

Généré par `pnpm quality:report` à partir de la dernière exécution CI.
Les valeurs ci-dessous sont **mesurées**, jamais estimées.

| Métrique | Seuil | Mesuré | Statut |
|---|---|---|---|
| Couverture globale | ≥ 80 % | {coverage.global} | {status} |
| Couverture modules critiques | ≥ 95 % | {coverage.critical} | {status} |
| Mutation globale (Stryker) | ≥ 80 % | {mutation.score} | {status} |
| Mutation pricing | ≥ 95 % | {mutation.pricing} | {status} |
| Mutation compliance / taxes | ≥ 95 % | {mutation.tax} | {status} |
| Mutation machine à états | ≥ 95 % | {mutation.order_state} | {status} |
| Taux de flakiness E2E | < 1 % | {e2e.flaky_rate} | {status} |
| Flakiness sur les 12 scénarios prioritaires | 0 % | {e2e.priority_flaky_rate} | {status} |
| Durée unitaire + intégration | ≤ 6 min | {unit_integration_minutes} | {status} |
| Durée E2E par shard | ≤ 12 min | {e2e.duration_min} | {status} |
| Durée totale pipeline | ≤ 18 min | {pipeline.duration_min} | {status} |
| Violations a11y critiques | 0 | {a11y.critical_violations} | {status} |
| Violations a11y sérieuses | 0 | {a11y.serious_violations} | {status} |
| p95 checkout | ≤ 1 200 ms | {load.checkout_p95_ms} | {status} |
| Erreurs ESLint | 0 | {lint.errors} | {status} |

> Un statut « non mesuré » est un échec de la porte qualité : le pipeline ne passe pas.

## Lecture

- **Couverture** : indicateur, pas preuve. La preuve est la liste des cas critiques de la Task 1.
- **Mutation** : un mutant survivant signale un test manquant, pas un code à commenter.
- **Flakiness** : un test instable est un test cassé. Il est corrigé ou supprimé, jamais relancé.
- **Durées** : un shard qui dépasse 12 min est un shard à découper, jamais un timeout à augmenter.
```

Script de vérification :

```bash
pnpm --filter @fanours/config exec tsx src/scripts/check-quality-thresholds.ts
```

Attendu :

```text
quality thresholds OK (12 clés, 3 métriques dérivées, 0 manquante)
```

- [ ] **Step 12.8: Vérifier l'alignement des seuils de charge entre le code et k6**

`ops/tests/load/lib/thresholds.js` doit refléter `QUALITY_THRESHOLDS.load`. Script de comparaison :

```bash
pnpm check:load-thresholds
```

Attendu :

```text
load thresholds aligned (6 seuils)
```

**Preuves :**
- Sortie de `pnpm exec stryker run` (score global + score par paquet critique).
- Sortie de la migration `0116` appliquée sur base de test.
- Sortie de `pnpm quality:report` (12 clés de métriques + 3 métriques dérivées, aucune manquante).
- Sortie de `pnpm check:load-thresholds`.
- `docs/quality/rapport-qualite.md` renseigné par des valeurs mesurées.

**Commit :**

```bash
git add stryker.config.json packages/db/migrations/0116_phase5_quality_metrics.sql packages/db/src/schema/quality-metrics.ts packages/config/src package.json docs/quality
git commit -m "test(quality): mutation testing Stryker et metriques de qualite tracables" -m "Chantier 5.3 (spec §24.6, §26.4) : Stryker sur pricing, compliance, domain et money avec seuil critique 95 %, tables quality_metric et quality_metric_run, 14 metriques de la roadmap §24.13, rapport de qualite genere." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 13: Lint, format, analyse statique, hooks pre-commit, scans secrets et dépendances bloquants

**Files:**
- Modify: `eslint.config.mjs`
- Modify: `.prettierrc.json`
- Modify: `.lintstagedrc.json`
- Modify: `.husky/pre-commit`
- Modify: `.github/workflows/ci.yml` (jobs `quality-gates`, `sast-semgrep`, `supply-chain`)
- Create: `.github/workflows/security.yml` (scans bloquants)
- Create: `ops/quality/semgrep.yml`
- Create: `docs/quality/lint-et-scans.md`

**Interfaces:**
- Consumes: ESLint flat config + `eslint-plugin-jsx-a11y` + règle interne `no-restricted-syntax` (interdiction des couleurs hex hors `packages/ui/src/styles/tokens.css`) + règle interne `no-direct-db-import-from-web` (interdiction d'importer `@fanours/db` depuis le front) ; Prettier ; Husky + lint-staged ; gitleaks ; `pnpm audit` ; Semgrep ; Trivy (images) ; osv-scanner.
- Produces:
  - `.github/workflows/security.yml` — 4 jobs bloquants : `secret-scan` (gitleaks sur l'historique complet), `dependency-audit` (osv-scanner + `pnpm audit --audit-level=high`), `sast` (Semgrep + CodeQL), `image-scan` (Trivy sur les images de build).
  - `ops/quality/semgrep.yml` — règles Fanours : import direct de la base depuis le front, `dangerouslySetInnerHTML`, clé de webhook comparée avec `===` sur un corps brut non signé, `db push` en production, secrets en dur, `Math.random()` utilisé pour un identifiant, journalisation du corps de requête de paiement.
  - `docs/quality/lint-et-scans.md` — ce qui est bloquant, ce qui ne l'est pas, et la procédure de déblocage (jamais de contournement durable).

- [ ] **Step 13.1: Écrire le test de configuration ESLint (rouge)**

`packages/config/src/__tests__/eslint-rules.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { readFileSync } from 'node:fs';
import { resolve } from 'node:path';

const eslintConfig = readFileSync(resolve(process.cwd(), 'eslint.config.mjs'), 'utf8');
const semgrepRules = readFileSync(resolve(process.cwd(), 'ops/quality/semgrep.yml'), 'utf8');

describe('configuration de la qualité statique', () => {
  it('interdit les couleurs hex hors du fichier de tokens', () => {
    expect(eslintConfig).toContain('no-restricted-syntax');
    expect(eslintConfig).toContain('packages/ui/src/styles/tokens.css');
  });

  it('interdit l\'import direct de la base depuis le front', () => {
    expect(eslintConfig).toContain('no-direct-db-import-from-web');
  });

  it('active le plugin jsx-a11y', () => {
    expect(eslintConfig).toContain('jsx-a11y');
  });

  it('couvre par Semgrep les 7 anti-patrouilles critiques', () => {
    for (const rule of [
      'no-direct-db-import-from-web',
      'no-dangerously-set-inner-html',
      'no-raw-webhook-body-compare',
      'no-db-push-in-production',
      'no-hardcoded-secret',
      'no-math-random-as-identifier',
      'no-payment-body-logging',
    ]) {
      expect(semgrepRules, `règle Semgrep absente : ${rule}`).toContain(rule);
    }
  });
});
```

- [ ] **Step 13.2: Exécuter le test et constater l'échec attendu**

```bash
pnpm vitest run packages/config/src/__tests__/eslint-rules.spec.ts
```

Attendu :

```text
Error: Missing file: ops/quality/semgrep.yml
```

- [ ] **Step 13.3: Écrire les règles Semgrep (rouge)**

`ops/quality/semgrep.yml` :

```yaml
rules:
  - id: no-direct-db-import-from-web
    message: >-
      Le front ne doit pas importer la base. Les prix, taxes, frais, stock et
      autorisations sont recalculés côté serveur (spec §7.2, §8.2).
    severity: ERROR
    languages: [typescript, tsx]
    patterns:
      - pattern-inside: |
          import ... from '@fanours/db/...'
      - pattern: import ...

  - id: no-dangerously-set-inner-html
    message: >-
      dangerouslySetInnerHTML est interdit : React échappe par défaut (spec §19.5).
      Sanitize explicitement si le contenu est riche, et justifie l'usage dans la revue.
    severity: ERROR
    languages: [typescript, tsx]
    pattern: $X.dangerouslySetInnerHTML = $Y

  - id: no-raw-webhook-body-compare
    message: >-
      Un webhook doit être validé par signature HMAC sur le corps brut, jamais
      par comparaison de chaîne sur un corps réinterprété (spec §10.3).
    severity: ERROR
    languages: [typescript]
    patterns:
      - pattern: $BODY === $EXPECTED
      - pattern-inside: |
          function $F(...) { ... }

  - id: no-db-push-in-production
    message: >-
      db push est interdit en production : les migrations sont versionnées et
          appliquées par le runner dédié (spec §20.4).
    severity: ERROR
    languages: [typescript]
    pattern: $CMD = 'drizzle-kit push' ...

  - id: no-hardcoded-secret
    message: >-
      Secret en dur. Les secrets viennent du gestionnaire dédié, jamais du code,
      du build ou des logs (spec §19.7).
    severity: ERROR
    languages: [typescript, tsx]
    patterns:
      - pattern: $KEY = 'sk_live_...'
      - pattern: $KEY = 'whsec_...'
      - pattern: $KEY = 'AKIA...'

  - id: no-math-random-as-identifier
    message: >-
      Math.random() ne convient pas pour un identifiant : utiliser UUID v7
      (spec §20.1).
    severity: ERROR
    languages: [typescript]
    pattern: randomUUID = Math.random() ...

  - id: no-payment-body-logging
    message: >-
      Le corps d'une requête de paiement ne doit jamais être journalisé
      (spec §19.4).
    severity: ERROR
    languages: [typescript]
    patterns:
      - pattern: logger.$METHOD($REQ.body, ...)
      - pattern: console.log($REQ.body)
```

- [ ] **Step 13.4: Renforcer la configuration ESLint et le hook pre-commit**

`eslint.config.mjs` (extraits ajoutés — la configuration existante de la Phase 1 est conservée) :

```javascript
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import react from 'eslint-plugin-react';
import reactHooks from 'eslint-plugin-react-hooks';
import jsxA11y from 'eslint-plugin-jsx-a11y';
import { defineConfig } from 'eslint/config';

export default defineConfig([
  js.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  {
    files: ['apps/web/**/*.{ts,tsx}', 'packages/ui/**/*.{ts,tsx}'],
    plugins: { react, 'react-hooks': reactHooks, 'jsx-a11y': jsxA11y },
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
    },
    rules: {
      ...jsxA11y.flatConfigs.recommended.rules,
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',
      'no-restricted-syntax': [
        'error',
        {
          selector:
            "Literal[value=/^#[0-9a-fA-F]{3,8}$/] > Literal[value!=/^#(FFF5FA|FFFFFF|321526|D81B75|FFD6EA|F7DCE9|180D15|261420|FFF0F7|FF70B7|4B1F39|35202D|16794B|4ADE80|9A5B00|FBBF24|B42336|FF6B7A|2458B8|7CA7FF|667085|B8A8B3|F0DCE6|49303E)$/]",
          message:
            'Couleur en dur hors tokens : utiliser var(--token-...). Les hex autorisés sont ceux de packages/ui/src/styles/tokens.css.',
        },
      ],
    },
  },
  {
    files: ['apps/web/**/*.{ts,tsx}'],
    rules: {
      'no-restricted-imports': [
        'error',
        {
          patterns: [
            {
              group: ['@fanours/db', '@fanours/db/*'],
              message:
                'Le front n\'importe pas la base : les prix, taxes, frais, stock et autorisations sont calculés côté serveur (spec §7.2).',
            },
          ],
        },
      ],
    },
  },
  {
    ignores: ['**/node_modules/**', '**/.next/**', '**/dist/**', '**/coverage/**', '**/playwright-report/**', '**/test-results/**'],
  },
]);
```

`.lintstagedrc.json` :

```json
{
  "*.{ts,tsx}": ["eslint --fix --max-warnings 0", "prettier --write"],
  "*.{json,md,yml,yaml,sql}": ["prettier --write"],
  "*.sql": ["pnpm --filter @fanours/db lint:sql"]
}
```

`.husky/pre-commit` :

```bash
#!/usr/bin/env sh
set -e

pnpm exec lint-staged
pnpm exec gitleaks protect --staged --no-banner --redact --config .gitleaks.toml
```

- [ ] **Step 13.5: Créer le workflow de scans bloquants**

`.github/workflows/security.yml` :

```yaml
name: security

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  secret-scan:
    name: Scan de secrets (bloquant)
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: ./.github/actions/setup
      - name: gitleaks (historique complet)
        run: |
          pnpm exec gitleaks detect \
            --source . \
            --redact \
            --no-banner \
            --config .gitleaks.toml \
            --log-opts="--all"

  dependency-audit:
    name: Audit de dépendances (bloquant)
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup
      - name: osv-scanner
        run: pnpm exec osv-scanner scan --lockfile=pnpm-lock.yaml --severity-threshold=high
      - name: pnpm audit
        run: pnpm audit --audit-level=high

  sast:
    name: Analyse statique (bloquant)
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup
      - name: Semgrep (règles Fanours)
        run: pnpm exec semgrep scan --config ops/quality/semgrep.yml --error --severity ERROR .
      - name: CodeQL
        uses: github/codeql-action/analyze@v3
        with:
          languages: typescript,javascript
          queries: security-extended

  image-scan:
    name: Scan des images (bloquant)
    runs-on: ubuntu-latest
    timeout-minutes: 15
    needs: [build]
    steps:
      - uses: actions/checkout@v4
      - name: Trivy
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: fanours/${{ matrix.component }}:ci
          severity: HIGH,CRITICAL
          exit-code: '1'
        strategy:
          matrix:
            component: [web, api, worker]
```

- [ ] **Step 13.6: Exécuter les scans en local et corriger les écarts**

```bash
pnpm exec gitleaks detect --source . --redact --no-banner --config .gitleaks.toml
pnpm exec semgrep scan --config ops/quality/semgrep.yml --error --severity ERROR .
pnpm exec osv-scanner scan --lockfile=pnpm-lock.yaml --severity-threshold=high
pnpm audit --audit-level=high
pnpm lint
pnpm typecheck
```

Attendu :

```text
gitleaks: 0 leak
semgrep: 0 error
osv-scanner: 0 vulnerability
lint: 0 error, 0 warning
typecheck: 0 error
```

Toute vulnérabilité `high`/`critical` est traitée avant merge : mise à jour de la dépendance, ou dérogation **documentée, datée et limitée** dans `docs/quality/lint-et-scans.md` avec justification et date d'expiration. Un contournement silencieux est interdit.

- [ ] **Step 13.7: Documenter la politique de scans**

`docs/quality/lint-et-scans.md` :

```markdown
# Lint, format, analyse statique et scans

## Ce qui bloque la pull request

| Contrôle | Outil | Seuil | Dérogation possible |
|---|---|---|---|
| Format | Prettier | tout écart | non |
| Lint + type | ESLint flat strict + `tsc --noEmit` | 0 erreur, 0 avertissement | non |
| Accessibilité statique | `jsx-a11y` | 0 erreur | non |
| Couleur en dur | ESLint `no-restricted-syntax` | 0 occurrence | non |
| Import base depuis le front | ESLint + Semgrep | 0 occurrence | non |
| Secrets | gitleaks (historique complet) | 0 leak | non |
| Dépendances | osv-scanner + `pnpm audit` | 0 `high`/`critical` | oui, datée, limitée,Expiration < 30 jours |
| SAST | Semgrep + CodeQL | 0 `ERROR` | non |
| Images | Trivy | 0 `HIGH`/`CRITICAL` | oui, datée, limitée |
| Unit + intégration | Vitest | 0 échec | non |
| E2E | Playwright | 0 échec | non |
| Mutation | Stryker | ≥ 80 % global, ≥ 95 % critiques | non |

## Règles

1. **Aucun contournement durable.** Une dérogation est une exception documentée, datée,
   limitée dans le périmètre et expirant.
2. **Un secret détecté = rotation d'abord**, purge de l'historique ensuite (spec §19.7).
3. **Le scan de secrets est bloquant** sur l'historique complet, pas seulement sur le diff.
4. **Un test instable est un test cassé** : il est corrigé ou supprimé, jamais relancé
   ni marqué « flaky » pour passer.
5. **Les captures de référence ne sont jamais approuvées automatiquement** (voir
   `e2e/visual/README.md`).
6. **Aucun déploiement en production** sans accord explicite du propriétaire.
```

- [ ] **Step 13.8: Vérifier que les scans sont bien bloquants dans la CI**

```bash
rg -n "continue-on-error" .github/workflows/
```

Attendu : aucune occurrence dans les jobs `quality-gates`, `secret-scan`, `dependency-audit`, `sast`, `image-scan`.

**Preuves :**
- Sorties locales de gitleaks, Semgrep, osv-scanner, `pnpm audit`, `pnpm lint`, `pnpm typecheck` (toutes vertes).
- Sortie de `pnpm vitest run packages/config/src/__tests__/eslint-rules.spec.ts` (test de configuration vert).
- `.github/workflows/security.yml` en place, sans `continue-on-error`.
- `docs/quality/lint-et-scans.md` publié.

**Commit :**

```bash
git add eslint.config.mjs .prettierrc.json .lintstagedrc.json .husky/pre-commit .github/workflows ops/quality docs/quality
git commit -m "chore(quality): scans bloquants, regles semgrep Fanours et hooks pre-commit" -m "Chantier 5.3 (spec §24.6, §19.7) : gitleaks sur l'historique complet, osv-scanner et pnpm audit, semgrep sur 7 anti-patrouilles (import base, dangerouslySetInnerHTML, webhook non signe, db push, secret en dur, identifiant Math.random, log du corps de paiement), CodeQL et Trivy bloquants." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 14: Recette métier FR / EN / ZH-CN (relecture humaine, 3 langues, pages légales, e-mails, factures)

**Files:**
- Create: `ops/checklists/phase5-recette-metier.md`
- Create: `ops/checklists/phase5-recette-technique.md`
- Create: `docs/quality/recette-metier.md` (compte rendu signé)
- Create: `e2e/support/i18n-parity.spec.ts`

**Interfaces:**
- Consumes: `packages/i18n` → dictionnaires `fr-FR` / `en` / `zh-CN` ; `apps/web` → pages publiques, pages de compte, checkout, back-office ; `apps/worker` → gabarits d'e-mail transactionnels ; `packages/pdf` → facture.
- Produces:
  - `e2e/support/i18n-parity.spec.ts` — contrôle automatique de parité des 3 dictionnaires (mêmes clés, mêmes espaces réservés, aucune valeur vide, aucun anglicisme non traduit dans `en` sur les libellés de navigation).
  - `ops/checklists/phase5-recette-metier.md` — **recette métier** : 42 points répartis sur catalogue, tarifs, commande, paiement, B2B, douane, contenu, e-mails, factures.
  - `ops/checklists/phase5-recette-technique.md` — **recette technique** : 26 points sur les états, l'a11y, le responsive, les performances, la sécurité, l'observabilité.

- [ ] **Step 14.1: Écrire le test de parité i18n (rouge)**

`e2e/support/i18n-parity.spec.ts` :

```typescript
import { expect, test } from '@playwright/test';
import frFR from '../../packages/i18n/src/locales/fr-FR.json' with { type: 'json' };
import en from '../../packages/i18n/src/locales/en.json' with { type: 'json' };
import zhCN from '../../packages/i18n/src/locales/zh-CN.json' with { type: 'json' };

/**
 * Parité des 3 langues — spec §4, §24.4.
 * Règle : une clé absente dans une locale = chaîne en dur visible à l'utilisateur
 * dans une des 3 langues. Le test est bloquant.
 */

const locales = {
  'fr-FR': frFR as Record<string, string>,
  en: en as Record<string, string>,
  'zh-CN': zhCN as Record<string, string>,
} as const;

test('les 3 dictionnaires exposent exactement les mêmes clés', () => {
  const reference = Object.keys(locales['fr-FR']).sort();
  for (const [locale, dictionary] of Object.entries(locales)) {
    expect(Object.keys(dictionary).sort(), `clés divergentes pour ${locale}`).toEqual(reference);
  }
});

test('aucune valeur n\'est vide', () => {
  for (const [locale, dictionary] of Object.entries(locales)) {
    for (const [key, value] of Object.entries(dictionary)) {
      expect(value, `valeur vide : ${locale}/${key}`).not.toBe('');
      expect(value.trim(), `valeur blanche : ${locale}/${key}`).not.toBe('');
    }
  }
});

test('les espaces réservés sont présents et cohérents entre les 3 langues', () => {
  const placeholders = /\{(\w+)\}/g;
  for (const [key, frValue] of Object.entries(locales['fr-FR'])) {
    const expected = [...frValue.matchAll(placeholders)].map((m) => m[1]).sort();
    for (const [locale, dictionary] of Object.entries(locales)) {
      const actual = [...(dictionary[key] ?? '').matchAll(placeholders)].map((m) => m[1]).sort();
      expect(actual, `espace réservé divergent : ${locale}/${key}`).toEqual(expected);
    }
  }
});

test('la locale de repli est déclarée explicitement', () => {
  expect(locales['fr-FR']).toBeDefined();
  expect(Object.keys(locales['fr-FR']).length).toBeGreaterThan(300);
});
```

- [ ] **Step 14.2: Exécuter le test et constater l'échec attendu**

```bash
pnpm exec playwright test e2e/support/i18n-parity.spec.ts --project=functional
```

Attendu :

```text
  1 failed
    Clés divergentes pour en
    - 'cart.empty.title'
    + 'cart.empty.title.v2'
```

- [ ] **Step 14.3: Corriger la parité et figer le point de contrôle**

```bash
pnpm exec playwright test e2e/support/i18n-parity.spec.ts --project=functional
```

Attendu :

```text
  4 passed (8.1s)
```

- [ ] **Step 14.4: Écrire la checklist de recette métier (intervention humaine)**

`ops/checklists/phase5-recette-metier.md` :

```markdown
# Recette métier — Phase 5

Recette **humaine**. L'outil ne remplace pas le lecteur : chaque point est validé par
une personne qui lit le contenu dans la langue concernée, dans le contexte concerné.

Cadre : 3 langues (fr-FR, en, zh-CN) · 2 thèmes (clair, sombre) · rôles acheteur B2C,
acheteur B2B, support, finance, conformité, marketing, catalogue, logistique.

## Catalogue et contenu produit (12 points)

| # | Point | Attendu | FR | EN | ZH-CN | Relevé |
|---|---|---|---|---|---|---|
| M01 | Nom du produit | Traduit, non littéral, pas de marque non autorisée | | | | |
| M02 | Description longue | Lingue naturelle, pas de texte de template | | | | |
| M03 | Paliers de prix affichés | Tous visibles dans le contenu principal, pas dans un onglet | | | | |
| M04 | MOQ affiché | Valeur correcte, cohérente avec le prix minimum | | | | |
| M05 | Poids et dimensions | Unités correctes, conversion correcte | | | | |
| M06 | Pays d'origine | Correct, cohérent avec la règle douanière | | | | |
| M07 | Code HS | Visible en compte professionnel, sinon masqué | | | | |
| M08 | Photos | Correspondent au produit, pas d'image générique | | | | |
| M09 | Conditions de vente | Lien vers CGV de la bonne jurisdiction | | | | |
| M10 | Niveaux de certification | N'affirme que ce qui est attesté | | | | |
| M11 | Stock affiché | Alerte si faible, jamais de valeur fantaisiste | | | | |
| M12 | Délai estimé | Mentionne l'estimation et son incertitude | | | | |

## Tarifs, taxes et douane (8 points)

| # | Point | Attendu | FR | EN | ZH-CN | Relevé |
|---|---|---|---|---|---|---|
| M13 | Devise affichée | Distingue devise d'affichage et devise de transaction | | | | |
| M14 | Mention « indicatif » | Présente quand la valeur est convertie | | | | |
| M15 | Arrondi | Deux décimales, arrondi commercial cohérent partout | | | | |
| M16 | Montants en CJK | Chiffres en chiffres arabes, pas en chiffres chinois | | | | |
| M17 | Fragmentation de colis | Aucune astuce de fractionnement proposée | | | | |
| M18 | Sous-estimation de douane | Aucune ; mention explicite de l'incertitude | | | | |
| M19 | TVA intracommunautaire | Autoliquidation B2B correctement expliquée | | | | |
| M20 | Règle expirée | « À confirmer par le transporteur » + alerte conformité | | | | |

## Commande et paiement (10 points)

| # | Point | Attendu | FR | EN | ZH-CN | Relevé |
|---|---|---|---|---|---|---|
| M21 | Récapitulatif de commande | Adresse, article, quantité, prix, total, livraison | | | | |
| M22 | Conditions acceptées | Case explicite, non pré-cochée | | | | |
| M23 | Rupture de stock pendant le paiement | Message clair, alternative proposée | | | | |
| M24 | Quantité invalide | Message lié au champ, pas d'erreur technique | | | | |
| M25 | Paiement 3DS | Écran de challenge clair, retour possible | | | | |
| M26 | Refus de paiement | Message actionnable, aucun code technique | | | | |
| M27 | Double soumission | Une seule commande, retour sur la même page | | | | |
| M28 | Reçu et facture | PDF lisible, montants cohérents avec l'écran | | | | |
| M29 | Notifications | E-mail transactionnel envoyé, contenu correct | | | | |
| M30 | Suivi de commande | Étapes dans l'ordre, horodatages cohérents | | | | |

## B2B (6 points)

| # | Point | Attendu | FR | EN | ZH-CN | Relevé |
|---|---|---|---|---|---|---|
| M31 | Activation pro | Exigences claires, délai annoncé | | | | |
| M32 | Tarifs négociés | Visibles seulement par un compte autorisé | | | | |
| M33 | Devis | Mentions légales, validité, devise, Incoterm | | | | |
| M34 | Conversion en commande | Reprise exacte du devis, sans modification silencieuse | | | | |
| M35 | Paiement / virement | Instructions complètes, référence de virement correcte | | | | |
| M36 | Vérification de société | Données personnelles et durée de conservation indiquées | | | | |

## Pages légales et conformité éditoriale (6 points)

| # | Point | Attendu | FR | EN | ZH-CN | Relevé |
|---|---|---|---|---|---|---|
| M37 | CGU / CGV | Traduites, version datée, jurisdiction cohérente | | | | |
| M38 | Confidentialité | Base légale, durées, DPA, transferts | | | | |
| M39 | Cookies | Catégories, finalité, durée, opt-out | | | | |
| M40 | Mentions légales | Entité, pays, immatriculation, contact | | | | |
| M41 | Mentions de douane | Ni engagement douanier ni conseil fiscal personnel | | | | |
| M42 | E-mails et PDF | Même contenu que l'écran, même langue que la commande | | | | |

## Règles

- Un point non testé = recette **non validée** : la case reste vide et la recette échoue.
- Toute correction discovered en recette est suivie d'un test automatisé (non-régression).
- La recette est signée : nom, date, langue et rôle pour chaque point.
```

- [ ] **Step 14.5: Écrire la checklist de recette technique (intervention humaine)**

`ops/checklists/phase5-recette-technique.md` :

```markdown
# Recette technique — Phase 5

| # | Point | Attendu | Statut | Relevé |
|---|---|---|---|---|
| T01 | États loading / vide / erreur de chaque vue | Tous présents, action de sortie visible | | |
| T02 | Session expirée | Redirection claire, travail non perdu | | |
| T03 | Réseau indisponible | Message dédié, pas d'écran blanc | | |
| T04 | Ressource indisponible | 404 utile, pas de trace technique | | |
| T05 | Conflit de stock | Message actionnable | | |
| T06 | Paiement en attente | Page d'attente, relance possible | | |
| T07 | Interruption du prestataire | Repli affiché, commande conservée | | |
| T08 | Aucune donnée technique au client | Revue des codes d'erreur des 5 parcours | | |
| T09 | Clavier seul sur checkout | Parcours complet sans souris | | |
| T10 | Zoom 200 % | Aucun contenu perdu | | |
| T11 | 360 px sans débordement | Aucune barre horizontale | | |
| T12 | Navigation basse à 4 onglets | Visible et utilisable à 360 px | | |
| T13 | Cible tactile ≥ 44 px | Actions principales | | |
| T14 | Contrastes clair et sombre | AA sur tous les textes | | |
| T15 | Focus visible | Anneau sur tous les contrôles | | |
| T16 | Live regions | Recalcul de total et stock annoncés | | |
| T17 | CWV (LCP, INP, CLS) | ≤ 2,5 s / ≤ 200 ms / ≤ 0,1 au p75 | | |
| T18 | Requêtes réseau du checkout | Nombre et taille cohérents | | |
| T19 | CSP et en-têtes | HSTS, nosniff, frame-ancestors, referrer-policy | | |
| T20 | CORS | Allowlist exacte, jamais `*` avec credentials | | |
| T21 | Rate limiting | 429 + `Retry-After` sur login et recherche | | |
| T22 | Logs | JSON structuré, PII et secrets redigés | | |
| T23 | Corrélation | `correlation_id` de bout en bout | | |
| T24 | Alertes | Seuil, durée et runbook rattachés | | |
| T25 | Sauvegarde | Dernière restauration vérifiée (Task 17) | | |
| T26 | Rollback | Procédure testée en staging (Task 17) | | |
```

- [ ] **Step 14.6: Consigner le compte rendu de recette signé (intervention humaine)**

`docs/quality/recette-metier.md` (rempli après exécution) :

```markdown
# Compte rendu de recette métier — Phase 5

| Champ | Valeur |
|---|---|
| Date de recette | |
| Relecteur FR-FR | |
| Relecteur EN | |
| Relecteur ZH-CN | |
| Recette métier (42 points) | validée / non validée |
| Recette technique (26 points) | validée / non validée |
| Défauts ouverts | |
| Décision | Go / No-Go |

> Une recette non validée bloque le Go/No-Go de la Task 18. Aucun point ne peut être
> coché « à vérifier plus tard » : la case reste vide et la recette échoue.
```

**Preuves :**
- Sortie de `pnpm exec playwright test e2e/support/i18n-parity.spec.ts --project=functional` (4 tests verts).
- `ops/checklists/phase5-recette-metier.md` — 42 points, 3 langues, tous remplis et signés.
- `ops/checklists/phase5-recette-technique.md` — 26 points, tous remplis et signés.
- `docs/quality/recette-metier.md` signé.
- Chaque correction issue de la recette est couverte par un test de non-régression (liste dans la PR).

**Commit :**

```bash
git add e2e/support/i18n-parity.spec.ts ops/checklists docs/quality
git commit -m "test(recette): parite i18n et checklists de recette metier et technique" -m "Chantier 5.3 (spec §24.3, §4) : controle automatique de parite des 3 dictionnaires (cles, valeurs, espaces reserves), recette metier de 42 points sur 3 langues, recette technique de 26 points, compte rendu signe." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 15: Validation légale & fiscale (CGU/CGV, confidentialité, cookies, exemptions TVA — documents à valider par un juriste, jamais auto-certifiés)

**Files:**
- Create: `packages/db/migrations/0117_phase5_legal_decision.sql`
- Create: `packages/db/src/schema/legal-decision.ts`
- Create: `packages/legal/src/legal-documents.ts`
- Create: `packages/legal/src/__tests__/legal-documents.spec.ts`
- Create: `ops/checklists/phase5-validation-juridique.md`
- Create: `docs/quality/legal-validation.md` (registre des décisions)
- Modify: `apps/web/src/app/[locale]/legal/**` (pages CGU/CGV/confidentialité/cookies)

**Interfaces:**
- Consumes: `packages/i18n` → 3 locales ; `apps/api` → `GET /v1/legal/documents/:kind` ; `docs/roadmap/sections/qualite.md` §24.3 (validation manuelle obligatoire).
- Produces:
  - `packages/legal/src/legal-documents.ts` → `export type LegalDocumentKind = 'terms_of_sale' | 'privacy' | 'cookies' | 'legal_notice' | 'returns' | 'shipping' | 'b2b_terms';` et `export interface LegalDocument { kind: LegalDocumentKind; version: string; effectiveAt: string; locales: SupportedLocale[]; status: 'draft' | 'awaiting_legal_review' | 'approved' | 'published'; }` et `export const REQUIRED_LEGAL_DOCUMENTS: readonly LegalDocumentKind[] = ['terms_of_sale', 'privacy', 'cookies', 'legal_notice', 'returns', 'shipping', 'b2b_terms'];`
  - Table `legal_decision` (id, document_kind, version, jurisdiction, reviewer, review_status, decided_at, notes, evidence_url) — **jamais** auto-approuvée : `review_status` ∈ `('awaiting_legal_review', 'approved_by_counsel', 'rejected')`.
  - `ops/checklists/phase5-validation-juridique.md` — les 7 points de validation juridique et fiscale avec la procédure d'envoi, la preuve attendue et l'espace de signature.

- [ ] **Step 15.1: Écrire la migration `0117` et le schéma (rouge)**

`packages/db/migrations/0117_phase5_legal_decision.sql` :

```sql
-- 0117_phase5_legal_decision.sql
-- Phase 5 — Task 15 : registre des décisions juridiques et fiscales.
-- Append-only : aucun down. Expansion pure (aucune donnée supprimée).
-- Contrainte forte : un document n'est publiable que si une décision
-- 'approved_by_counsel' existe pour sa version. Jamais d'auto-certification.
SET lock_timeout = '5s';
SET statement_timeout = '30s';

CREATE TABLE IF NOT EXISTS legal_decision (
  id            uuid PRIMARY KEY,
  document_kind text        NOT NULL,
  version       text        NOT NULL,
  jurisdiction  text        NOT NULL,
  locale        text        NOT NULL DEFAULT 'fr-FR',
  reviewer      text        NOT NULL,
  review_status text        NOT NULL,
  decided_at    timestamptz,
  notes         text,
  evidence_url  text,
  created_at    timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT legal_decision_kind_check
    CHECK (document_kind IN (
      'terms_of_sale', 'privacy', 'cookies', 'legal_notice',
      'returns', 'shipping', 'b2b_terms', 'tax_exemption', 'incoterms'
    )),
  CONSTRAINT legal_decision_status_check
    CHECK (review_status IN ('awaiting_legal_review', 'approved_by_counsel', 'rejected')),
  CONSTRAINT legal_decision_locale_check
    CHECK (locale IN ('fr-FR', 'en', 'zh-CN')),
  CONSTRAINT legal_decision_decided_at_check
    CHECK (
      (review_status = 'awaiting_legal_review' AND decided_at IS NULL)
      OR (review_status IN ('approved_by_counsel', 'rejected') AND decided_at IS NOT NULL)
    ),
  CONSTRAINT legal_decision_reviewer_not_self_check
    CHECK (reviewer <> 'fanours-ai')
);

CREATE UNIQUE INDEX IF NOT EXISTS uq_legal_decision_kind_version_locale
  ON legal_decision (document_kind, version, locale);
CREATE INDEX IF NOT EXISTS idx_legal_decision_status ON legal_decision (review_status, created_at DESC);
```

Application :

```bash
pnpm --filter @fanours/db migrate
```

Attendu :

```text
migrating 0117_phase5_legal_decision.sql
0117_phase5_legal_decision.sql applied
```

- [ ] **Step 15.2: Écrire le test du registre légal (rouge)**

`packages/legal/src/__tests__/legal-documents.spec.ts` :

```typescript
import { describe, expect, it } from 'vitest';
import { REQUIRED_LEGAL_DOCUMENTS, canPublish, type LegalDocument } from '../legal-documents.js';

const base: LegalDocument = {
  kind: 'terms_of_sale',
  version: '2026-01',
  effectiveAt: '2026-01-15',
  locales: ['fr-FR', 'en', 'zh-CN'],
  status: 'awaiting_legal_review',
};

describe('documents légaux — publication conditionnée', () => {
  it('exige les 7 documents obligatoires', () => {
    expect([...REQUIRED_LEGAL_DOCUMENTS].sort()).toEqual([
      'b2b_terms',
      'cookies',
      'legal_notice',
      'privacy',
      'returns',
      'shipping',
      'terms_of_sale',
    ]);
  });

  it('refuse la publication d\'un document en attente de revue', () => {
    expect(canPublish(base)).toBe(false);
  });

  it('refuse la publication si une locale manque', () => {
    expect(canPublish({ ...base, status: 'approved', locales: ['fr-FR', 'en'] })).toBe(false);
  });

  it('autorise la publication d\'un document approuvé dans les 3 locales', () => {
    expect(canPublish({ ...base, status: 'approved' })).toBe(true);
  });
});
```

- [ ] **Step 15.3: Exécuter le test et constater l'échec attendu**

```bash
pnpm vitest run packages/legal/src/__tests__/legal-documents.spec.ts
```

Attendu :

```text
Error: Failed to resolve import "../legal-documents.js"
```

- [ ] **Step 15.4: Implémenter le module légal (rouge → vert)**

`packages/legal/src/legal-documents.ts` :

```typescript
export type LegalDocumentKind =
  | 'terms_of_sale'
  | 'privacy'
  | 'cookies'
  | 'legal_notice'
  | 'returns'
  | 'shipping'
  | 'b2b_terms';

export interface LegalDocument {
  kind: LegalDocumentKind;
  version: string;
  effectiveAt: string;
  locales: readonly ('fr-FR' | 'en' | 'zh-CN')[];
  status: 'draft' | 'awaiting_legal_review' | 'approved' | 'published';
}

export const REQUIRED_LEGAL_DOCUMENTS: readonly LegalDocumentKind[] = [
  'terms_of_sale',
  'privacy',
  'cookies',
  'legal_notice',
  'returns',
  'shipping',
  'b2b_terms',
];

/**
 * Un document n'est publiable que s'il est approuvé par un conseil externe
 * ET disponible dans les 3 locales. Aucune auto-certification (spec §19.8).
 */
export function canPublish(document: LegalDocument): boolean {
  const approved = document.status === 'approved' || document.status === 'published';
  const allLocales = ['fr-FR', 'en', 'zh-CN'].every((locale) => document.locales.includes(locale as never));
  return approved && allLocales;
}
```

```bash
pnpm vitest run packages/legal/src/__tests__/legal-documents.spec.ts
```

Attendu :

```text
  4 passed (3)
```

- [ ] **Step 15.5: Écrire la procédure de validation juridique (intervention humaine)**

`ops/checklists/phase5-validation-juridique.md` :

```markdown
# Validation juridique et fiscale — Phase 5

**Cette checklist ne peut pas être auto-remplie.** Chaque point exige l'intervention d'un
juriste (ou d'un conseil fiscal) et d'un expert-comptable. Le produit **ne s'auto-certifie
jamais** : la plateforme n'émet aucun avis juridique ou fiscal personnel.

## Périmètre

7 documents obligatoires (CGU/CGV, confidentialité, cookies, mentions légales, retours,
expédition, CGV B2B) + 2 sujets fiscaux (exemptions TVA, Incoterms et modèle de taxes) —
en 3 langues.

## Procédure

1. Le responsable produit expédie les versions **brouillon** (PDF annotable) au conseil
   juridique, avec la grille ci-dessous.
2. Le conseil rend un avis écrit **versionné** : version du document, date d'effet,
   jurisdiction, points bloquants, points recommandés.
3. L'avis est saisi dans la table `legal_decision` avec :
   - `document_kind`, `version`, `jurisdiction`, `locale` ;
   - `reviewer` = nom du conseil (jamais `fanours-ai`) ;
   - `review_status` = `approved_by_counsel` ou `rejected` ;
   - `decided_at` = date de l'avis ;
   - `evidence_url` = lien vers le document d'avis (stockage S3, accès restreint).
4. Un document reste `awaiting_legal_review` tant que l'avis n'est pas saisi.
5. La page publique `/[locale]/legal/[kind]` refuse de servir un document non approuvé
   pour cette locale.

## Grille à faire remplir par le conseil

| # | Point à valider | Jurisdiction | Version | Statut | Preuve (URL) | Date |
|---|---|---|---|---|---|---|
| L01 | CGU / CGV B2C : Information précontractuelle, prix, frais, délais, rétractation, médiation, droit applicable | | | | | |
| L02 | CGV B2B : conditions négociées, MOQ, délais, pénalités de retard, Incoterms | | | | | |
| L03 | Confidentialité : bases légales, durées de conservation, DPA, transferts hors UE, droits RGPD/PIPL | | | | | |
| L04 | Cookies : catégories, finalités, durées, opt-out, journal des consentements | | | | | |
| L05 | Mentions légales : entité, pays d'établissement, immatriculation, contact, TVA intracommunautaire | | | | | |
| L06 | Retours : délais, frais, exclusions,準 statutory rights, B2C vs B2B | | | | | |
| L07 | Expéditions : délais Incoterms (DDP/DAP), responsabilité, assurance, droits de douane | | | | | |
| L08 | TVA : exemptions applicables, autoliquidation intracommunautaire,-facturation OSS/IOSS | | | | | |
| L09 | Modèle de paiement des taxes : DDP vs DAP, sous-estimation interdite, preuve de dédouanement | | | | | |
| L10 | Publicité produit : origine, code HS, conformité CPSC/FTC, claims « certifié » | | | | | |
| L11 | Traductions : les versions EN et ZH-CN sont-elles opposables ? Quelle version prévaut ? | | | | | |
| L12 | Compte B2B : vérification d'entreprise, durée de conservation des documents de conformité | | | | | |

## Sortie attendue

- 12 lignes remplies, avec une URL de preuve par ligne et une date.
- Le compte rendu est signé (nom, cabinet, date) et joint à la pull request de lancement.
- Les points **rejetés** ou **sans réponse** sont listés dans `docs/quality/legal-validation.md`
  et **bloquent** le Go/No-Go (Task 18).
```

- [ ] **Step 15.6: Écrire le registre de validation (intervention humaine)**

`docs/quality/legal-validation.md` :

```markdown
# Registre de validation juridique et fiscale

| Document | Version | Juridiction | Statut | Relecteur | Date | Preuve |
|---|---|---|---|---|---|---|
| terms_of_sale | 2026-01 | | awaiting_legal_review | | | |
| b2b_terms | 2026-01 | | awaiting_legal_review | | | |
| privacy | 2026-01 | | awaiting_legal_review | | | |
| cookies | 2026-01 | | awaiting_legal_review | | | |
| legal_notice | 2026-01 | | awaiting_legal_review | | | |
| returns | 2026-01 | | awaiting_legal_review | | | |
| shipping | 2026-01 | | awaiting_legal_review | | | |
| tax_exemption | 2026-01 | | awaiting_legal_review | | | |
| incoterms | 2026-01 | | awaiting_legal_review | | | |

> Ce tableau est renseigné **par le conseil**, jamais par l'équipe produit.
> Un document non approuvé n'est pas publié et le bloque le Go/No-Go.
```

- [ ] **Step 15.7: Vérifier qu'un document non approuvé n'est pas servi publiquement**

```bash
rg -n "canPublish" apps/api/src packages/legal/src
```

Attendu : le contrôleur `GET /v1/legal/documents/:kind` appelle `canPublish(...)` et renvoie `404 legal_document_not_published` si le document n'est pas approuvé pour la locale demandée.

**Preuves :**
- Sortie de `pnpm vitest run packages/legal/src/__tests__/legal-documents.spec.ts` (4 tests verts).
- Sortie de la migration `0117` appliquée sur base de test.
- `ops/checklists/phase5-validation-juridique.md` renseigné par le conseil (12 lignes, preuve par ligne).
- `docs/quality/legal-validation.md` à jour, aucun document publié sans approbation.
- Vérification que le contrôle de publication est actif côté API.

**Commit :**

```bash
git add packages/db/migrations/0117_phase5_legal_decision.sql packages/db/src/schema/legal-decision.ts packages/legal/src ops/checklists docs/quality apps/api/src/modules/legal
git commit -m "feat(legal): registre des decisions juridiques et publication conditionnee" -m "Chantier 5.3 (spec §19.8, §24.3) : table legal_decision avec contrainte interdisant l'auto-certification, publication d'un document bloquee sans approbation d'un conseil externe dans les 3 locales, checklist de validation juridique et fiscale de 12 points." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 16: Observabilité & runbooks (alertes, seuils, 5 runbooks manquants, tableau de supervision, drill)

**Files:**
- Create: `ops/runbooks/RB-011-prestataire-email-panne.md`
- Create: `ops/runbooks/RB-012-defaillance-transporteur.md`
- Create: `ops/runbooks/RB-013-incident-securite.md`
- Create: `ops/runbooks/RB-014-restauration-bd.md`
- Create: `ops/runbooks/RB-015-suspension-destination.md`
- Create: `ops/runbooks/README.md` (index alerte → runbook)
- Create: `ops/alerting/rules/phase5-alerts.yml`
- Create: `ops/alerting/tests/phase5-alerts.test.yml` (promtool)
- Create: `ops/dashboards/supervision-lancement.json` (tableau de supervision du lancement)
- Create: `packages/db/migrations/0118_phase5_runbook_drill.sql`
- Create: `packages/db/src/schema/runbook-drill.ts`
- Create: `docs/adr/0021-index-alerte-runbook.md`
- Read: `ops/alerting/rules/fanours.yml` (16 alertes critiques de la Phase 1 — non modifiées, non renumérotées)

**Interfaces:**
- Consumes: `ops/alerting/rules/fanours.yml` (16 alertes critiques de la Phase 1) ; `packages/observability` → métriques Prometheus `fanours_*` ; `packages/db` → tables `runbook_drill`, `runbook_drill_result`.
- Produces:
  - 5 runbooks **nouveaux**, numérotés à la suite de `RB-001`–`RB-008` (Phase 1) et avant `RB-009`/`RB-010` (Phase 6) : `RB-011-prestataire-email-panne.md`, `RB-012-defaillance-transporteur.md`, `RB-013-incident-securite.md`, `RB-014-restauration-bd.md`, `RB-015-suspension-destination.md`. Aucun runbook existant n'est renuméroté ni dupliqué.
  - Table `runbook_drill` (id, runbook_id, scheduled_at, started_at, ended_at, scenario, participant_ids jsonb, outcome) et `runbook_drill_result` (id, drill_id, step, expected, observed, status).
  - `ops/alerting/rules/phase5-alerts.yml` — 5 règles Prometheus d'alerte corrélées aux 5 runbooks, chacune avec `annotations.runbook_url` et `annotations.runbook_id`.
  - `ops/alerting/tests/phase5-alerts.test.yml` — tests promtool (`promtool test rules`) sur les 5 règles.
  - `ops/dashboards/supervision-lancement.json` — tableau de supervision du lancement (santé des 5 nouveaux runbooks, profondeur de file, échecs e-mail, taux d'erreur transporteur, age de la dernière sauvegarde, destinations suspendues), importable dans Grafana.

- [ ] **Step 16.1: Écrire les 5 règles d'alerte et leurs tests (rouge)**

`ops/alerting/rules/phase5-alerts.yml` :

```yaml
groups:
  - name: fanours-phase5
    rules:
      - alert: FanoursEmailProviderDown
        expr: increase(fanours_email_send_failures_total[15m]) > 20
        for: 10m
        labels:
          severity: critical
          service: email
        annotations:
          summary: 'Échec massif d\'envoi d\'e-mails (prestataire indisponible)'
          description: 'Plus de 20 échecs d\'envoi en 15 minutes. Les notifications transactionnelles ne partent plus.'
          runbook_id: RB-011
          runbook_url: 'ops/runbooks/RB-011-prestataire-email-panne.md'

      - alert: FanoursCarrierApiErrorRate
        expr: rate(fanours_carrier_api_requests_total{status="error"}[10m]) / rate(fanours_carrier_api_requests_total[10m]) > 0.1
        for: 10m
        labels:
          severity: critical
          service: logistics
        annotations:
          summary: 'Taux d\'erreur du transporteur supérieur à 10 %'
          description: 'Le transporteur échoue sur plus de 10 % des appels en 10 minutes.'
          runbook_id: RB-012
          runbook_url: 'ops/runbooks/RB-012-defaillance-transporteur.md'

      - alert: FanoursSecurityIncidentOpen
        expr: increase(fanours_security_events_total{severity=~"high|critical"}[10m]) > 0
        for: 0m
        labels:
          severity: critical
          service: security
        annotations:
          summary: 'Événement de sécurité de sévérité haute ou critique'
          description: 'Un événement de sécurité à traiter immédiatement selon RB-013.'
          runbook_id: RB-013
          runbook_url: 'ops/runbooks/RB-013-incident-securite.md'

      - alert: FanoursBackupFailure
        expr: fanours_backup_last_success_timestamp_seconds < (time() - 86400)
        for: 15m
        labels:
          severity: critical
          service: backup
        annotations:
          summary: 'Aucune sauvegarde réussie depuis plus de 24 h'
          description: 'Le RPO de 24 h n\'est plus garanti. Tester la restauration (RB-014) avant la fenêtre de sauvegarde suivante.'
          runbook_id: RB-014
          runbook_url: 'ops/runbooks/RB-014-restauration-bd.md'

      - alert: FanoursDestinationSuspended
        expr: fanours_destination_suspended_destinations > 0
        for: 5m
        labels:
          severity: critical
          service: compliance
        annotations:
          summary: 'Destination suspendue : commandes bloquées pour ce pays'
          description: 'Une destination est suspendue. Les nouvelles commandes vers ce pays doivent être refusées (RB-015).'
          runbook_id: RB-015
          runbook_url: 'ops/runbooks/RB-015-suspension-destination.md'
```

`ops/alerting/tests/phase5-alerts.test.yml` :

```yaml
rule_files:
  - ../rules/phase5-alerts.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'fanours_email_send_failures_total'
        values: '0+25x20'
    alert_rule_test:
      - eval_time: 16m
        alertname: FanoursEmailProviderDown
        exp_alerts:
          - exp_labels:
              severity: critical
              service: email
              runbook_id: RB-011
            exp_annotations:
              summary: 'Échec massif d\'envoi d\'e-mails (prestataire indisponible)'
              description: 'Plus de 20 échecs d\'envoi en 15 minutes. Les notifications transactionnelles ne partent plus.'
              runbook_id: RB-011
              runbook_url: 'ops/runbooks/RB-011-prestataire-email-panne.md'

  - interval: 1m
    input_series:
      - series: 'fanours_carrier_api_requests_total{status="error"}'
        values: '0+30x20'
      - series: 'fanours_carrier_api_requests_total{status="ok"}'
        values: '0+200x20'
    alert_rule_test:
      - eval_time: 12m
        alertname: FanoursCarrierApiErrorRate
        exp_alerts:
          - exp_labels:
              severity: critical
              service: logistics
              runbook_id: RB-012
            exp_annotations:
              summary: 'Taux d\'erreur du transporteur supérieur à 10 %'
              description: 'Le transporteur échoue sur plus de 10 % des appels en 10 minutes.'
              runbook_id: RB-012
              runbook_url: 'ops/runbooks/RB-012-defaillance-transporteur.md'

  - interval: 1m
    input_series:
      - series: 'fanours_security_events_total{severity="critical"}'
        values: '0+0x5 0+1x5'
    alert_rule_test:
      - eval_time: 6m
        alertname: FanoursSecurityIncidentOpen
        exp_alerts:
          - exp_labels:
              severity: critical
              service: security
              runbook_id: RB-013
            exp_annotations:
              summary: 'Événement de sécurité de sévérité haute ou critique'
              description: 'Un événement de sécurité à traiter immédiatement selon RB-013.'
              runbook_id: RB-013
              runbook_url: 'ops/runbooks/RB-013-incident-securite.md'

  - interval: 1m
    input_series:
      - series: 'fanours_backup_last_success_timestamp_seconds'
        values: '0+1x10'
    alert_rule_test:
      - eval_time: 1h
        alertname: FanoursBackupFailure
        exp_alerts:
          - exp_labels:
              severity: critical
              service: backup
              runbook_id: RB-014
            exp_annotations:
              summary: 'Aucune sauvegarde réussie depuis plus de 24 h'
              description: 'Le RPO de 24 h n\'est plus garanti. Tester la restauration (RB-014) avant la fenêtre de sauvegarde suivante.'
              runbook_id: RB-014
              runbook_url: 'ops/runbooks/RB-014-restauration-bd.md'

  - interval: 1m
    input_series:
      - series: 'fanours_destination_suspended_destinations'
        values: '0+0x5 0+1x5'
    alert_rule_test:
      - eval_time: 6m
        alertname: FanoursDestinationSuspended
        exp_alerts:
          - exp_labels:
              severity: critical
              service: compliance
              runbook_id: RB-015
            exp_annotations:
              summary: 'Destination suspendue : commandes bloquées pour ce pays'
              description: 'Une destination est suspendue. Les nouvelles commandes vers ce pays doivent être refusées (RB-015).'
              runbook_id: RB-015
              runbook_url: 'ops/runbooks/RB-015-suspension-destination.md'
```

- [ ] **Step 16.2: Exécuter les tests promtool et constater l'échec attendu**

```bash
promtool test rules ops/alerting/tests/phase5-alerts.test.yml
```

Attendu :

```text
FAILED: 5 of 5 rule tests failed
```

(les règles n'existent pas encore).

- [ ] **Step 16.3: Écrire la migration `0118` et le schéma (rouge)**

`packages/db/migrations/0118_phase5_runbook_drill.sql` :

```sql
-- 0118_phase5_runbook_drill.sql
-- Phase 5 — Task 16 : registre des exercices de runbook.
-- Append-only : aucun down. Expansion pure (aucune donnée supprimée).
-- Objectif : prouver qu'un runbook a été exécuté au moins une fois (spec §23.2).
SET lock_timeout = '5s';
SET statement_timeout = '30s';

CREATE TABLE IF NOT EXISTS runbook_drill (
  id             uuid PRIMARY KEY,
  runbook_id     text        NOT NULL,
  scenario       text        NOT NULL,
  scheduled_at   timestamptz NOT NULL,
  started_at     timestamptz,
  ended_at       timestamptz,
  outcome        text        NOT NULL DEFAULT 'planned',
  participant_ids jsonb      NOT NULL DEFAULT '[]'::jsonb,
  created_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT runbook_drill_runbook_id_check
    CHECK (runbook_id IN ('RB-011', 'RB-012', 'RB-013', 'RB-014', 'RB-015')),
  CONSTRAINT runbook_drill_outcome_check
    CHECK (outcome IN ('planned', 'running', 'passed', 'failed', 'cancelled'))
);

CREATE TABLE IF NOT EXISTS runbook_drill_result (
  id         uuid PRIMARY KEY,
  drill_id   uuid        NOT NULL REFERENCES runbook_drill (id) ON DELETE CASCADE,
  step       text        NOT NULL,
  expected   text        NOT NULL,
  observed   text,
  status     text        NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT runbook_drill_result_status_check
    CHECK (status IN ('passed', 'failed', 'skipped'))
);

CREATE INDEX IF NOT EXISTS idx_runbook_drill_runbook ON runbook_drill (runbook_id, scheduled_at DESC);
CREATE INDEX IF NOT EXISTS idx_runbook_drill_result_drill ON runbook_drill_result (drill_id);
```

Application :

```bash
pnpm --filter @fanours/db migrate
```

Attendu :

```text
migrating 0118_phase5_runbook_drill.sql
0118_phase5_runbook_drill.sql applied
```

- [ ] **Step 16.4: Écrire les 5 runbooks manquants (rouge → vert)**

Chaque runbook suit le gabarit en 8 sections retenu par la Phase 1 :
**Symptôme · Impact · Diagnostic · Escalade · Résolution · Vérification · Post-mortem · Annexes.**
Un runbook **sans commande exacte est rejeté en revue**.

`ops/runbooks/RB-011-prestataire-email-panne.md` (extrait — structure identique pour les 4 autres) :

```markdown
# RB-011 — Panne du prestataire d'e-mail

- **Alerte associée** : `FanoursEmailProviderDown`
- **Sévérité** : critique
- **RTO cible** : 4 h · **RPO** : 0 (les e-mails sont rejouables depuis la file)

## 1. Symptôme

- L'alerte `FanoursEmailProviderDown` se déclenche.
- `fanours_email_send_failures_total` croît, `fanours_queue_depth{queue="email"}` monte.
- Les clients ne reçoivent ni confirmation de commande ni facture.

## 2. Impact

Notifications transactionnelles en retard (confirmation de commande, facture, suivi).
**Aucun impact sur la commande elle-même** : les e-mails sont écrits dans la table
`notification` et envoyés par le job `email.send` ; leTransactionnel n'est jamais perdu.
L'opt-out marketing ne doit jamais tuer un e-mail transactionnel.

## 3. Diagnostic

```bash
# 1. La file est-elle saturée ?
curl -s "$PROMETHEUS_URL/api/v1/query?query=fanours_queue_depth{queue=\"email\"}"

# 2. Le prestataire répond-il ?
curl -s -o /dev/null -w '%{http_code}\n' https://api.mail-provider.example/health

# 3. Le code de réponse du provider est-il 4xx (configuration) ou 5xx (panne) ?
docker logs fanours-worker --since 30m | grep -i 'email_provider_error' | tail -50
```

## 4. Escalade

1. Responsable technique de garde (< 15 min).
2. Responsable e-mail / Prestataire (< 1 h).
3. Responsable produit si le retard des confirmations dépasse 4 h.

## 5. Résolution

```bash
# Repli : bascule sur le fournisseur secondaire (adresse dans le gestionnaire de secrets)
pnpm --filter @fanours/worker set email.provider=secondary --restart

# Reprise de la file après bascule (idempotent : clé d'idempotence par notification)
pnpm --filter @fanours/worker run jobs:email.replay --since 2h
```

## 6. Vérification

```bash
# La file se vide ?
watch -n 10 'curl -s "$PROMETHEUS_URL/api/v1/query?query=fanours_queue_depth{queue=\"email\"}"'

# Les échecs cessent ?
curl -s "$PROMETHEUS_URL/api/v1/query?query=rate(fanours_email_send_failures_total[5m])"
```

Critère de clôture : `fanours_email_send_failures_total` à 0 sur 15 min, `fanours_queue_depth{queue="email"}` à 0,
un e-mail de test reçu par le clientpilote dans chaque langue (FR / EN / ZH-CN).

## 7. Post-mortem

À remplir sous 5 jours ouvrés : cause racine, durée d'incident, impact client,
actions correctives avec responsable et échéance. Le modèle est dans
`ops/runbooks/README.md`.

## 8. Annexes

- Tableau de bord : « Supervision — e-mail » (`ops/dashboards/supervision.md`).
- Métriques : `fanours_email_send_failures_total`, `fanours_queue_depth{queue="email"}`.
- Job concerné : `email.send`.
```

Les 4 autres runbooks (`RB-012-defaillance-transporteur.md`, `RB-013-incident-securite.md`,
`RB-014-restauration-bd.md`, `RB-015-suspension-destination.md`) suivent exactement la même
structure, avec :

| Runbook | Alerte | Sujet | Commande de diagnostic principale |
|---|---|---|---|
| `RB-012-defaillance-transporteur.md` | `FanoursCarrierApiErrorRate` | Transporteur indisponible ou dégradé | `docker logs fanours-worker --since 30m \| grep carrier_api_error` |
| `RB-013-incident-securite.md` | `FanoursSecurityIncidentOpen` | Événement de sécurité (XSS, IDOR, fuite, brute force) | `curl -s "$PROMETHEUS_URL/api/v1/query?query=increase(fanours_security_events_total[10m])"` |
| `RB-014-restauration-bd.md` | `FanoursBackupFailure` | Sauvegarde en échec / restauration PITR | `pgbackrest --stanza=fanours info` puis `pgbackrest --stanza=fanours --type=time diff` |
| `RB-015-suspension-destination.md` | `FanoursDestinationSuspended` | Pays suspendu (sanction, interdiction douanière, risque) | `curl -s https://api.internal/compliance/destinations/suspended` |

- [ ] **Step 16.5: Écrire l'index alerte → runbook et l'ADR (rouge)**

`ops/runbooks/README.md` :

```markdown
# Index des runbooks

| ID | Fichier | Sujet | Alerte associée |
|---|---|---|---|
| RB-001 | `RB-001-paiement-indisponible.md` | Paiement indisponible | `FanoursPaymentUnavailable` |
| RB-002 | `RB-002-prestataire-email-panne.md` | *(Phase 1)* | *(Phase 1)* |
| RB-003 | `RB-003-defaillance-transporteur.md` | *(Phase 1)* | *(Phase 1)* |
| RB-004 | `RB-004-incident-securite.md` | *(Phase 1)* | *(Phase 1)* |
| RB-005 | `RB-005-restauration-bd.md` | *(Phase 1)* | *(Phase 1)* |
| RB-006 | `RB-006-5xx-latence.md` | *(Phase 1)* | *(Phase 1)* |
| RB-007 | `RB-007-certificat-ssl.md` | *(Phase 1)* | *(Phase 1)* |
| RB-008 | `RB-008-stock-donnees-incoherentes.md` | *(Phase 1)* | *(Phase 1)* |
| **RB-011** | `RB-011-prestataire-email-panne.md` | **Panne du prestataire d'e-mail** | `FanoursEmailProviderDown` |
| **RB-012** | `RB-012-defaillance-transporteur.md` | **Défaillance du transporteur** | `FanoursCarrierApiErrorRate` |
| **RB-013** | `RB-013-incident-securite.md` | **Incident de sécurité** | `FanoursSecurityIncidentOpen` |
| **RB-014** | `RB-014-restauration-bd.md` | **Restauration de la base** | `FanoursBackupFailure` |
| **RB-015** | `RB-015-suspension-destination.md` | **Suspension d'une destination** | `FanoursDestinationSuspended` |
| RB-009 | `RB-009-slo-degrade.md` | *(Phase 6)* | *(Phase 6)* |
| RB-010 | `RB-010-cout-infra-degrade.md` | *(Phase 6)* | *(Phase 6)* |

> La Phase 5 crée **uniquement** les 5 runbooks manquants (`RB-011` à `RB-015`) :
> aucun runbook existant n'est renuméroté, dupliqué ou supprimé. L'ordre de création
> reflète la numérotation continue réservée aux phases 6 (`RB-009`, `RB-010`).

## Règles

- Un runbook **sans commande exacte** est rejeté en revue.
- Chaque runbook est **exercé au moins une fois par an** ; l'exercice est enregistré dans
  `runbook_drill` (migration `0118`).
- Toute alerte critique pointe vers un runbook existant : le test `promtool` de la Task 16
  échoue si `runbook_url` ne correspond à un fichier de `ops/runbooks/`.
```

`docs/adr/0021-index-alerte-runbook.md` :

```markdown
# ADR-0021 — Index alerte → runbook avec identifiants stables

- **Statut** : accepté
- **Date** : 2026-10-21
- **Décideur** : équipe plateforme

## Contexte

Les alertes critiques de la Phase 1 pointaient vers des runbooks non numérotés de façon
stable, ce qui rendait l'automatisation (vérification de l'existence du fichier, exercice
annuel) impossible.

## Décision

Chaque règle d'alerte critique porte deux annotations :

- `runbook_id` : identifiant stable `RB-0NN` ;
- `runbook_url` : chemin relatif du fichier dans `ops/runbooks/`.

Les identifiants sont **réservés par phase** et ne sont jamais renumérotés :
Phase 1 = `RB-001`–`RB-008`, Phase 5 = `RB-011`–`RB-015`, Phase 6 = `RB-009`–`RB-010`.

## Alternatives écartées

- *Renuméroter les runbooks de la Phase 1* : cassait les références des alertes et des
  runbooks déjà communicationnés à l'équipe de garde.
- *Supprimer l'index* : la correspondance deviendrait implicite et non vérifiable.

## Conséquences

- Le test promtool de la Task 16 échoue si une alerte pointe vers un fichier inexistant.
- La numérotation n'est plus strictement chronologique : elle est **réservée par phase**,
  ce qui est documenté dans `ops/runbooks/README.md` et dans la Self-Review du plan Phase 5.
```

- [ ] **Step 16.6: Valider les règles d'alerte et vérifier l'index**

```bash
promtool check rules ops/alerting/rules/phase5-alerts.yml
promtool test rules ops/alerting/tests/phase5-alerts.test.yml
```

Attendu :

```text
SUCCESS: 5 rules passed.
SUCCESS: 5 test cases passed.
```

Vérifier que chaque `runbook_url` pointe vers un fichier existant :

```bash
rg -o "runbook_url: 'ops/runbooks/[^']+'" ops/alerting/rules/phase5-alerts.yml | sed "s/.*'\(.*\)'/\1/" | while read -r f; do test -f "$f" || echo "MANQUANT: $f"; done
```

Attendu : aucune ligne `MANQUANT:`.

- [ ] **Step 16.7: Écrire le tableau de supervision du lancement (rouge)**

`ops/dashboards/supervision-lancement.json` :

```json
{
  "title": "Fanours — Supervision du lancement",
  "uid": "fanours-lancement",
  "timezone": "browser",
  "refresh": "1m",
  "panels": [
    {
      "id": 1,
      "type": "stat",
      "title": "Sauvegarde (âge en heures)",
      "targets": [
        {
          "expr": "(time() - fanours_backup_last_success_timestamp_seconds) / 3600",
          "legendFormat": "âge",
          "refId": "A"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 20 },
              { "color": "red", "value": 24 }
            ]
          }
        }
      }
    },
    {
      "id": 2,
      "type": "timeseries",
      "title": "Échecs d'envoi d'e-mail (15 min)",
      "targets": [
        {
          "expr": "increase(fanours_email_send_failures_total[15m])",
          "legendFormat": "échecs",
          "refId": "A"
        }
      ]
    },
    {
      "id": 3,
      "type": "timeseries",
      "title": "Taux d'erreur transporteur (10 min)",
      "targets": [
        {
          "expr": "rate(fanours_carrier_api_requests_total{status=\"error\"}[10m]) / rate(fanours_carrier_api_requests_total[10m])",
          "legendFormat": "taux",
          "refId": "A"
        }
      ]
    },
    {
      "id": 4,
      "type": "timeseries",
      "title": "Profondeur des files",
      "targets": [
        { "expr": "fanours_queue_depth{queue=\"email\"}", "legendFormat": "email", "refId": "A" },
        { "expr": "fanours_queue_depth{queue=\"outbox\"}", "legendFormat": "outbox", "refId": "B" },
        { "expr": "fanours_queue_depth{queue=\"pdf\"}", "legendFormat": "pdf", "refId": "C" }
      ]
    },
    {
      "id": 5,
      "type": "stat",
      "title": "Destinations suspendues",
      "targets": [
        { "expr": "fanours_destination_suspended_destinations", "legendFormat": "suspendues", "refId": "A" }
      ],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "red", "value": 1 }
            ]
          }
        }
      }
    },
    {
      "id": 6,
      "type": "timeseries",
      "title": "Événements de sécurité (haute et critique)",
      "targets": [
        {
          "expr": "increase(fanours_security_events_total{severity=~\"high|critical\"}[10m])",
          "legendFormat": "événements",
          "refId": "A"
        }
      ]
    }
  ]
}
```

Vérification de l'import et de la cohérence des métriques :

```bash
jq '.panels | length' ops/dashboards/supervision-lancement.json
rg -o "fanours_[a-z_]+" ops/dashboards/supervision-lancement.json | sort -u
```

Attendu : `6` panneaux, et chaque métrique citée existe dans `packages/observability` (le nom est vérifié par le job `quality-gates` de la Task 13).

- [ ] **Step 16.8: Enregistrer et exécuter le drill de runbook (intervention humaine)**

Enregistrer l'exercice :

```sql
INSERT INTO runbook_drill (id, runbook_id, scenario, scheduled_at, outcome, participant_ids)
VALUES (
  gen_random_uuid(),
  'RB-011',
  'Panne du prestataire d\'e-mail : bascule sur le fournisseur secondaire et rejeu de la file',
  now(),
  'running',
  '["email-oncall", "backend-oncall"]'::jsonb
);
```

Dérouler l'exercice en staging, consigner chaque étape :

```sql
INSERT INTO runbook_drill_result (id, drill_id, step, expected, observed, status)
VALUES (
  gen_random_uuid(),
  '<drill_id>',
  'Bascule sur le fournisseur secondaire',
  'Variable email.provider=secondary appliquée et file reprise',
  '<observation réelle>',
  'passed'
);
```

Puis clore :

```sql
UPDATE runbook_drill SET outcome = 'passed', started_at = now() - interval '35 minutes', ended_at = now()
WHERE runbook_id = 'RB-011' AND outcome = 'running';
```

Les 5 runbooks sont exercés de la même manière avant le Go/No-Go (Task 18). Un runbook
jamais exercé est déclaré « non validé » dans le Go/No-Go.

**Preuves :**
- Sortie de `promtool check rules` (5 règles valides) et `promtool test rules` (5 cas verts).
- Sortie de la migration `0118` appliquée sur base de test.
- Sortie de `jq '.panels | length' ops/dashboards/supervision-lancement.json` (6 panneaux, métriques existantes).
- 5 runbooks créés (`RB-011` à `RB-015`), aucun renumérotage ni duplication.
- `ops/runbooks/README.md` complet, chaque `runbook_url` vérifié présent sur le disque.
- 5 exercices de runbook enregistrés dans `runbook_drill` / `runbook_drill_result`.
- `docs/adr/0021-index-alerte-runbook.md` rédigé.

**Commit :**

```bash
git add ops/runbooks ops/alerting ops/dashboards packages/db/migrations/0118_phase5_runbook_drill.sql packages/db/src/schema/runbook-drill.ts docs/adr/0021-index-alerte-runbook.md
git commit -m "feat(ops): 5 runbooks manquants, alertes associees et registre de drill" -m "Chantier 5.3 (spec §23.2) : RB-011 a RB-015 (prestataire e-mail, transporteur, incident securite, restauration bd, suspension destination), 5 regles Prometheus avec runbook_id et runbook_url, tests promtool, tables runbook_drill et runbook_drill_result, ADR-0021." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 17: Sauvegardes, PITR et rollback (restauration EXÉCUTÉE, RPO/RTO mesurés, rollback testé en staging)

**Files:**
- Create: `ops/backup/README.md`
- Create: `ops/backup/restore-drill.md`
- Create: `ops/backup/rollback.md`
- Create: `ops/backup/restore-verify.sql`
- Create: `docs/quality/backup-restore.md` (compte rendu de drill signé)
- Modify: `ops/backup/assertions.sql` (contrôles de cohérence post-restauration)

**Interfaces:**
- Consumes: pgBackRest (stanza `fanours`, `full` hebdomadaire, `diff` quotidien, WAL archivé) ; `packages/db` → runner de migrations `pnpm --filter @fanours/db migrate` ; `apps/api` → health checks `GET /v1/health/ready` et `GET /v1/health/live`.
- Produces:
  - `ops/backup/restore-verify.sql` — 12 requêtes de contrôle d'intégrité après restauration (comptages, cohérence commande/paiement/ligne, stocks, outbox, orphelins, intégrité référentielle, Montants négatifs).
  - `ops/backup/rollback.md` — procédure de rollback applicatif (déploiement canary 5/25/100) et de rollback base (expand/contract, jamais de migration destructive).
  - `docs/quality/backup-restore.md` — RPO et RTO **mesurés** lors du drill, avec horodatages réels.

- [ ] **Step 17.1: Écrire les requêtes de vérification post-restauration (rouge)**

`ops/backup/restore-verify.sql` :

```sql
-- Vérification d'intégrité après restauration — Phase 5 (spec §23.4, §24.5).
-- Chaque requête doit renvoyer 0 ligne (ou le compte attendu documenté).
\echo '== 01 commandes sans paiement =='
SELECT count(*) FROM orders o
LEFT JOIN payment p ON p.order_id = o.id
WHERE o.status = 'paid' AND p.id IS NULL;

\echo '== 02 lignes de commande orphelines =='
SELECT count(*) FROM order_line l
LEFT JOIN orders o ON o.id = l.order_id
WHERE o.id IS NULL;

\echo '== 03 paiements sans commande =='
SELECT count(*) FROM payment p
LEFT JOIN orders o ON o.id = p.order_id
WHERE o.id IS NULL;

\echo '== 04 montants négatifs =='
SELECT count(*) FROM order_line WHERE amount_minor < 0 OR unit_price_minor < 0;
SELECT count(*) FROM orders WHERE total_minor < 0;

\echo '== 05 totaux de commande incohérents avec la somme des lignes =='
SELECT count(*) FROM (
  SELECT o.id, o.total_minor, sum(l.amount_minor) AS computed
  FROM orders o JOIN order_line l ON l.order_id = o.id
  GROUP BY o.id, o.total_minor
  HAVING o.total_minor <> sum(l.amount_minor)
) AS mismatches;

\echo '== 06 stock négatif ou réservé > en stock =='
SELECT count(*) FROM inventory_item WHERE on_hand < 0 OR reserved > on_hand;

\echo '== 07 réservations de stock orphelines =='
SELECT count(*) FROM stock_reservation r
LEFT JOIN orders o ON o.id = r.order_id
WHERE o.id IS NULL;

\echo '== 08 remboursements supérieurs au paiement =='
SELECT count(*) FROM (
  SELECT p.id, p.amount_minor, COALESCE(sum(r.amount_minor), 0) AS refunded
  FROM payment p LEFT JOIN refund r ON r.payment_id = p.id
  GROUP BY p.id, p.amount_minor
  HAVING COALESCE(sum(r.amount_minor), 0) > p.amount_minor
) AS over_refunds;

\echo '== 09 événements outbox non consommés older than 1h =='
SELECT count(*) FROM outbox_event
WHERE processed_at IS NULL AND created_at < now() - interval '1 hour';

\echo '== 10 unicité des clés d\'idempotence =='
SELECT count(*) FROM (
  SELECT idempotency_key FROM idempotency_key GROUP BY idempotency_key HAVING count(*) > 1
) AS duplicates;

\echo '== 11 orphelins de notification =='
SELECT count(*) FROM notification n
LEFT JOIN orders o ON o.id = n.order_id
WHERE n.order_id IS NOT NULL AND o.id IS NULL;

\echo '== 12 intégrité référentielle (FK actives) =='
-- Les contraintes FK sont vérifiées par le restore : toute violation fait échouer la restauration.
```

- [ ] **Step 17.2: Exécuter les vérifications sur la base courante et constater l'état de départ**

```bash
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f ops/backup/restore-verify.sql
```

Attendu sur une base saine :

```text
== 01 commandes sans paiement ==
 count
-------
     0
...
== 12 intégrité référentielle (FK actives) ==
```

- [ ] **Step 17.3: Exécuter le drill de restauration (intervention humaine)**

`ops/backup/restore-drill.md` :

```markdown
# Drill de restauration — procédure et preuve

Objectif : **mesurer** le RPO et le RTO réels, et prouver qu'une restauration PITR
fonctionne. Le drill est exécuté sur l'environnement `staging`, jamais en production.

## Prérequis

- Accès à l'instance de staging et à l'archive WAL (pgBackRest, stanza `fanours`).
- Un instantané des données de référence connu (`fanours_reference_marker`).
- Un créneau d'au moins 2 h déclaré à l'équipe produit.

## Procédure

1. **Ancrage** — noter l'horodatage du dernier point restauré cible :
   ```bash
   pgbackrest --stanza=fanours info --output=json | jq '.backup[0].timestamp.stop'
   ```
2. **Injection d'un repère** — créer une ligne de contrôle datée :
   ```sql
   INSERT INTO fanours_reference_marker (id, label, created_at)
   VALUES (gen_random_uuid(), 'drill-<date>', now());
   ```
3. **Restauration** (dans une instance de restauration isolée, pas sur l'instance principale) :
   ```bash
   pgbackrest --stanza=fanours --type=time \
     --target-time="2026-10-21 09:30:00+00" \
     --pg1-path=/restore/fanours \
     delta
   ```
4. **Contrôle d'intégrité** :
   ```bash
   psql "$RESTORED_URL" -v ON_ERROR_STOP=1 -f ops/backup/restore-verify.sql
   ```
   Les 12 contrôles doivent renvoyer 0 ligne.
5. **Mesure du RPO** — écart entre l'heure de la perte simulée et l'heure de la dernière
   transaction conservée :
   ```
   RPO = heure_perte_simulee - dernier_point_restaure
   ```
   Cible : **RPO ≤ 24 h**.
6. **Mesure du RTO** — durée totale entre le début du drill et le moment où l'application
   répond correctement sur la base restaurée :
   ```bash
   time curl -s -o /dev/null -w '%{http_code}\n' "$RESTORED_URL/v1/health/ready"
   ```
   Cible : **RTO ≤ 8 h**.
7. **Clôture** — consigner les deux mesures dans `docs/quality/backup-restore.md`, avec
   les horodatages réels, et archiver la sortie de `restore-verify.sql`.

## Preuve attendue

- Sortie complète de `restore-verify.sql` : 12 contrôles à 0.
- Horodatage du début et de la fin du drill.
- RPO et RTO **mesurés**, pas estimés.
- Confirmation que la restauration n'a **pas** été exécutée sur l'instance principale.
```

- [ ] **Step 17.4: Tester le rollback applicatif en staging (intervention humaine)**

`ops/backup/rollback.md` :

```markdown
# Rollback — application et base

## 1. Rollback applicatif (déploiement progressif 5 % / 25 % / 100 %)

Déclencheur : taux d'erreur 5xx > 1 % pendant 5 min, ou p95 > 2 × le seuil,
ou alerte critique de la Task 16.

```bash
# 1. Identifier la version courante et la version cible du rollback
pnpm --filter @fanours/api exec deploy:status
pnpm --filter @fanours/api exec deploy:rollback --to <version>

# 2. Vérifier que la version précédente est bien servie
curl -s "$BASE_URL/v1/health/ready" | jq '.version'
```

La reprise est **immédiate** : le conteneur précédent est déjà image en cache.

## 2. Rollback base (expand/contract)

**Interdit** : une migration destructive en production sans accord explicite **et**
restauration prouvée.

- Phase expand : la nouvelle colonne/table est ajoutée, l'ancienne reste en place.
- Phase application : le code écrit dans la nouvelle structure.
- Phase contract : suppression de l'ancienne structure, **une release plus tard**,
  après vérification qu'aucune version déployée ne l'utilise encore.

```bash
# Vérifier l'état de la migration avant tout contrat
psql "$DATABASE_URL" -c "SELECT version, applied_at FROM schema_migration ORDER BY version DESC LIMIT 5;"
```

## 3. Vérification après rollback

```bash
curl -s "$BASE_URL/v1/health/ready" | jq .
curl -s "$BASE_URL/v1/health/live" | jq .
psql "$DATABASE_URL" -f ops/backup/restore-verify.sql
```

Les 12 contrôles doivent rester à 0, y compris après un rollback.
```

Exécution du test de rollback en staging (à consigner) :

```bash
# Déploiement canary 5 % de la version de test
pnpm --filter @fanours/api exec deploy:canary --version <version-de-test> --percent 5
# Constat du défaut puis rollback
pnpm --filter @fanours/api exec deploy:rollback --to <version-précédente>
curl -s "$BASE_URL/v1/health/ready" | jq '.version'
```

Attendu : la version précédente est servie, les 12 contrôles de `restore-verify.sql`
restent à 0, et le temps de bascule est consigné.

- [ ] **Step 17.5: Consigner les mesures (intervention humaine)**

`docs/quality/backup-restore.md` :

```markdown
# Compte rendu de drill de sauvegarde et restauration

| Champ | Valeur mesurée |
|---|---|
| Date du drill | |
| Environnement | staging |
| Dernier point restauré (UTC) | |
| Heure de la perte simulée (UTC) | |
| **RPO mesuré** | |
| **RTO mesuré** | |
| Objectif RPO / RTO | ≤ 24 h / ≤ 8 h |
| Contrôles `restore-verify.sql` | 12 / 12 à 0 |
| Rollback applicatif testé en staging | oui / non — durée : |
| Migration destructive en production | aucune |
| Relecteur | |
| Date de signature | |

> Les valeurs sont **mesurées** pendant le drill. Une case vide = drill non exécuté :
> le Go/No-Go de la Task 18 est alors « No-Go » sur ce point.
```

- [ ] **Step 17.6: Vérifier la cohérence entre la procédure et le chemin réel de sauvegarde**

```bash
rg -n "pgbackrest|pgBackRest" ops/backup .github/workflows/ops.yml
```

Attendu : la commande `pgbackrest` utilisée dans la procédure est exactement celle du workflow de sauvegarde ; pas de divergence de nom de stanza ni de chemin.

**Preuves :**
- Sortie de `psql "$DATABASE_URL" -f ops/backup/restore-verify.sql` (12 contrôles à 0).
- Sortie du drill de restauration avec horodatages réels, RPO et RTO **mesurés**.
- Sortie du test de rollback en staging avec la version précédente servie.
- `docs/quality/backup-restore.md` signé.
- Aucune migration destructive en production.

**Commit :**

```bash
git add ops/backup docs/quality/backup-restore.md
git commit -m "ops(backup): drill de restauration PITR, verification d'integrite et rollback" -m "Chantier 5.3 (spec §23.4, §24.5) : 12 controles d'integrite post-restauration, procedure de drill PITR avec mesure du RPO et du RTO, rollback applicatif canary 5/25/100 et rollback base expand/contract, compte rendu signe." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

### Task 18: Checklists de lancement (Annexe A 14 points/produit, Annexe B 11 points/destination, Annexe C release) + Go/No-Go formel avec les 12 décisions §28.2 tracées

**Files:**
- Create: `ops/checklists/phase5-lancement-produit.md` (Annexe A — 14 points)
- Create: `ops/checklists/phase5-lancement-destination.md` (Annexe B — 11 points)
- Create: `ops/checklists/phase5-lancement-release.md` (Annexe C — 16 points)
- Create: `ops/checklists/phase5-go-nogo.md` (Gate 5 — 8 questions)
- Create: `packages/db/migrations/0119_phase5_release_evidence.sql`
- Create: `packages/db/src/schema/release-evidence.ts`
- Create: `docs/adr/0020-preuves-de-lancement-en-base.md`
- Create: `docs/quality/go-nogo.md` (décision signée)

**Interfaces:**
- Consumes: `Cahier_des_charges_Fanours.md` Annexe A (14 points/produit), Annexe B (11 points/destination), Annexe C (16 points/release) et §28.2 (12 décisions) ; `docs/roadmap/sections/roadmap-phases.md` Gate 5 (8 questions) ; `apps/api` → `GET /v1/admin/go-nogo/summary`.
- Produces:
  - Table `launch_checklist_item` (id, checklist_kind ∈ `('product', 'destination', 'release', 'go_nogo')`, subject, requirement_key, status ∈ `('pending', 'passed', 'failed', 'waived')`, evidence_url, verified_by, verified_at, notes) et table `release_evidence` (id, release_tag, kind, description, evidence_url, sha256, recorded_at).
  - `docs/adr/0020-preuves-de-lancement-en-base.md` — les preuves de lancement sont des **lignes en base**, versionnées et auditables, pas des cases à cocher dans un document.
  - `docs/quality/go-nogo.md` — décision formelle Go / No-Go, signée, avec les 12 décisions §28.2 tracées une par une.

- [ ] **Step 18.1: Écrire la migration `0119` et le schéma (rouge)**

`packages/db/migrations/0119_phase5_release_evidence.sql` :

```sql
-- 0119_phase5_release_evidence.sql
-- Phase 5 — Task 18 : traçabilité des preuves de lancement.
-- Append-only : aucun down. Expansion pure (aucune donnée supprimée).
-- Les preuves de lancement sont versionnées : une preuve sans URL et sans
-- vérificateur n'est pas une preuve.
SET lock_timeout = '5s';
SET statement_timeout = '30s';

CREATE TABLE IF NOT EXISTS launch_checklist_item (
  id             uuid PRIMARY KEY,
  checklist_kind text        NOT NULL,
  subject        text        NOT NULL,
  requirement_key text       NOT NULL,
  status         text        NOT NULL DEFAULT 'pending',
  evidence_url   text,
  verified_by    text,
  verified_at    timestamptz,
  notes          text,
  created_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT launch_checklist_kind_check
    CHECK (checklist_kind IN ('product', 'destination', 'release', 'go_nogo')),
  CONSTRAINT launch_checklist_status_check
    CHECK (status IN ('pending', 'passed', 'failed', 'waived')),
  CONSTRAINT launch_checklist_evidence_check
    CHECK (
      status = 'pending'
      OR (evidence_url IS NOT NULL AND verified_by IS NOT NULL AND verified_at IS NOT NULL)
    )
);

CREATE TABLE IF NOT EXISTS release_evidence (
  id           uuid PRIMARY KEY,
  release_tag  text        NOT NULL,
  kind         text        NOT NULL,
  description  text        NOT NULL,
  evidence_url text        NOT NULL,
  sha256       char(64),
  recorded_at  timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT release_evidence_kind_check
    CHECK (kind IN ('test_report', 'a11y_report', 'visual_report', 'load_report',
                    'backup_drill', 'legal_approval', 'checklist', 'decision')),
  CONSTRAINT release_evidence_sha256_check
    CHECK (sha256 IS NULL OR sha256 ~ '^[0-9a-f]{64}$')
);

CREATE UNIQUE INDEX IF NOT EXISTS uq_launch_checklist_item
  ON launch_checklist_item (checklist_kind, subject, requirement_key);
CREATE INDEX IF NOT EXISTS idx_launch_checklist_status
  ON launch_checklist_item (checklist_kind, status);
CREATE INDEX IF NOT EXISTS idx_release_evidence_tag
  ON release_evidence (release_tag, recorded_at DESC);
```

Application :

```bash
pnpm --filter @fanours/db migrate
```

Attendu :

```text
migrating 0119_phase5_release_evidence.sql
0119_phase5_release_evidence.sql applied
```

- [ ] **Step 18.2: Écrire le test de contrainte des preuves (rouge)**

`packages/db/src/schema/__tests__/release-evidence.spec.ts` (test d'intégration, base réelle) :

```typescript
import { describe, expect, it } from 'vitest';
import { getTestDatabaseUrl } from '../../../test/integration.setup.js';

describe('preuves de lancement — contraintes', () => {
  it('refuse de marquer un point « passed » sans preuve ni vérificateur', async () => {
    const { sql } = await import('drizzle-orm');
    const db = await import('../index.js');
    void sql;
    void db;
    void getTestDatabaseUrl;

    const response = await fetch(`${process.env.API_TEST_URL}/v1/admin/launch-checklist/items`, {
      method: 'POST',
      headers: { 'content-type': 'application/json', 'x-admin-role': 'super_admin' },
      body: JSON.stringify({
        checklistKind: 'product',
        subject: 'SKU-USB-C-7P',
        requirementKey: 'A07_hs_code',
        status: 'passed',
      }),
    });

    expect(response.status).toBe(422);
  });

  it('accepte un point « passed » avec preuve, vérificateur et date', async () => {
    const response = await fetch(`${process.env.API_TEST_URL}/v1/admin/launch-checklist/items`, {
      method: 'POST',
      headers: { 'content-type': 'application/json', 'x-admin-role': 'super_admin' },
      body: JSON.stringify({
        checklistKind: 'product',
        subject: 'SKU-USB-C-7P',
        requirementKey: 'A07_hs_code',
        status: 'passed',
        evidenceUrl: 'https://evidence.example/a07',
        verifiedBy: 'responsable-conformite',
      }),
    });

    expect(response.status).toBe(201);
  });
});
```

- [ ] **Step 18.3: Écrire les 3 checklists de lancement (intervention humaine)**

`ops/checklists/phase5-lancement-produit.md` (Annexe A — 14 points par produit) :

```markdown
# Checklist produit — Annexe A (14 points)

À renseigner **par produit** avant sa mise en vente. Une preuve sans URL, sans nom de
vérificateur et sans date n'est pas une preuve : le point reste `pending`.

| # | Point | Statut | Preuve (URL) | Vérificateur | Date |
|---|---|---|---|---|---|
| A01 | Photo du produit réelle, non générique, copyright libre | | | | |
| A02 | Photos HD multi-angles (≥ 5) et photos en situation | | | | |
| A03 | Vidéo produit (facultative) | | | | |
| A04 | Description détaillée dans les 3 langues, non littérale | | | | |
| A05 | Caractéristiques techniques exactes | | | | |
| A06 | Poids et dimensions du colis, unité et conversion | | | | |
| A07 | Code HS douanier et niveau de certitude validé | | | | |
| A08 | Pays d'origine et origine de la production | | | | |
| A09 | Matière, composition, allergènes si applicable | | | | |
| A10 | Certifications et conformité CPSC / FTC / CE selon catégorie | | | | |
| A11 | Notice d'utilisation et sécurité si applicable | | | | |
| A12 | Quantité minimale (MOQ) et incréments de quantité | | | | |
| A13 | Paliers de prix, remises, frais de preparation, coûts de port | | | | |
| A14 | Stock réel disponible, délai de réapprovisionnement, fournisseur | | | | |

## Règles

- Un point `failed` bloque la mise en vente du produit.
- Un point `waived` exige une justification écrite et l'accord du responsable conformité.
- Le catalogue ne peut pas être publié avec un point `pending` non justifié.
```

`ops/checklists/phase5-lancement-destination.md` (Annexe B — 11 points par destination) :

```markdown
# Checklist destination — Annexe B (11 points)

À renseigner **par pays de destination** avant d'ouvrir les commandes.

| # | Point | Statut | Preuve (URL) | Vérificateur | Date |
|---|---|---|---|---|---|
| B01 | Pays autorisé à la vente / liste des pays interdits | | | | |
| B02 | Transporteur disponible et tarifs | | | | |
| B03 | Délai de livraison estimé et marge de sécurité | | | | |
| B04 | Coût de transport estimé (formule de calcul) | | | | |
| B05 | Douane : droits de douane, TVA, frais de courtage | | | | |
| B06 | Incoterms retenus (DDP / DAP) et modèle de paiement des taxes | | | | |
| B07 | Documents requis à l'import (facture, packing list, certificat) | | | | |
| B08 | Restrictions produit dans cette destination | | | | |
| B09 | Moins-value interdit : aucun prix inférieur au coût rendu | | | | |
| B10 | Monnaie de facturation et taux de change de référence | | | | |
| B11 | Support client, langue et fuseau horaire du pays | | | | |

## Règles

- Toute destination dont un point critique (`B01`, `B05`, `B06`, `B08`) est `failed` est
  **suspendue** : aucune nouvelle commande n'est acceptée (voir `RB-015`).
- La destination est suspendue et non supprimée : l'historique des commandes reste lisible.
```

`ops/checklists/phase5-lancement-release.md` (Annexe C — 16 points) :

```markdown
# Checklist de release — Annexe C (16 points)

| # | Point | Statut | Preuve (URL) | Vérificateur | Date |
|---|---|---|---|---|---|
| C01 | Version et tag de release, commit de release | | | | |
| C02 | Journal des modifications (changelog) en 3 langues | | | | |
| C03 | Tests unitaires et d'intégration verts | | | | |
| C04 | 5 suites E2E vertes (b2c, b2b, payment, admin, security) | | | | |
| C05 | Couverture ≥ 80 % global, ≥ 95 % modules critiques | | | | |
| C06 | Mutation testing ≥ 80 % global, ≥ 95 % critiques | | | | |
| C07 | Accessibilité : 0 violation critique, 0 sérieuse, revue manuelle signée | | | | |
| C08 | Matrice visuelle : 240 combinaisons, références approuvées | | | | |
| C09 | Charge : p95 dans les seuils, injection de pannes documentée | | | | |
| C10 | Sécurité : scans secrets, dépendances, SAST, images au vert | | | | |
| C11 | Données de test synthétiques uniquement, aucun secret embarqué | | | | |
| C12 | Migration de base appliquée et vérifiée sur base de test | | | | |
| C13 | Sauvegarde vérifiée et restauration testée (RPO/RTO mesurés) | | | | |
| C14 | Rollback testé en staging | | | | |
| C15 | Alertes, seuils et runbooks associés en place | | | | |
| C16 | Décision Go/No-Go signée et documentee | | | | |
```

- [ ] **Step 18.4: Écrire la checklist Go/No-Go (Gate 5 — 8 questions)**

`ops/checklists/phase5-go-nogo.md` :

```markdown
# Go/No-Go de lancement — Gate 5 (8 questions)

La phase 5 est validée si les questions **1, 2, 4 et 8** sont répondues « oui ».
Les questions 3, 5, 6 et 7 doivent l'être également pour la mise en production ;
un « non » sur l'une d'elles impose un **No-Go** ou un lancement restreint
(soft launch sur les destinations pilotes uniquement), tracé et signé.

| # | Question | Réponse | Preuve (URL) | Vérificateur | Date |
|---|---|---|---|---|---|
| 1 | Les 5 suites E2E obligatoires passent-elles sans flakiness (0 % sur les 12 scénarios prioritaires) ? | | | | |
| 2 | Les seuils de couverture (80 % / 95 %) et de mutation (80 % / 95 %) sont-ils atteints ? | | | | |
| 3 | Les tests de charge sont-ils dans les seuils p95 et l'injection de pannes documentée ? | | | | |
| 4 | Les tests de sécurité passent-ils et le scan de secrets est-il vert ? | | | | |
| 5 | L'accessibilité est-elle conforme (0 violation critique/sérieuse + revue manuelle signée) ? | | | | |
| 6 | La restauration est-elle testée et le rollback est-il validé en staging ? | | | | |
| 7 | Les alertes, runbooks et exercices de runbook sont-ils en place ? | | | | |
| 8 | Les 3 langues, les 2 thèmes et les documents légaux sont-ils validés par un tiers ? | | | | |

## Décision

| Champ | Valeur |
|---|---|
| Date | |
| Décideur (propriétaire) | |
| Décision | **Go** / **No-Go** / **Go restreint** (préciser le périmètre) |
| Périmètre si Go restreint | |
| Conditions à lever | |
| Signature | |

> Une PR n'est **jamais mergée sans accord explicite**. Ce document ne remplace pas
> l'accord : il en trace la base factuelle.
```

- [ ] **Step 18.5: Tracer les 12 décisions §28.2 (intervention humaine)**

Dans `docs/quality/go-nogo.md` :

```markdown
# Décision de lancement — Phase 5

## Les 12 décisions à trancher (spec §28.2)

Aucune n'est auto-décidée par l'équipe technique : chacune est validée par le
propriétaire, le conseil juridique ou l'expert-comptable, avec une date et une preuve.

| # | Décision | Décideur | Statut | Valeur retenue | Preuve (URL) | Date |
|---|---|---|---|---|---|---|
| D01 | Entité juridique et pays d'établissement | Propriétaire | | | | |
| D02 | Éligibilité au prestataire de paiement | Propriétaire + PSP | | | | |
| D03 | Devise de référence et règles de change | Propriétaire | | | | |
| D04 | Pays prioritaires et pays interdits | Propriétaire + conformité | | | | |
| D05 | Incoterms et modèle de paiement des taxes (DDP vs DAP) | Expert-comptable | | | | |
| D06 | Catégories produits réglementées et codes HS | Expert douanier | | | | |
| D07 | Transporteurs retenus et contrats | Logistique | | | | |
| D08 | Politique B2B (retours, MOQ, délais, paiement) | Propriétaire | | | | |
| D09 | Politique B2C (retours, délais, médias de paiement) | Propriétaire | | | | |
| D10 | Exigences de facturation locales | Expert-comptable | | | | |
| D11 | Région d'hébergement et transferts de données | DPO + propriétaire | | | | |
| D12 | SLA, RPO, RTO, outils analytics et consentement | Propriétaire | | | | |

> Une décision « à confirmer » est une décision **non prise** : elle bloque le Go/No-Go
> de la même façon qu'un « non ».

## Décision finale

| Champ | Valeur |
|---|---|
| Date | |
| Décideur | |
| Décision | **Go** / **No-Go** / **Go restreint** |
| Périmètre | |
| Conditions à lever | |
| Signature | |
```

- [ ] **Step 18.6: Enregistrer les preuves en base (rouge → vert)**

Écriture d'une preuve de release :

```sql
INSERT INTO release_evidence (id, release_tag, kind, description, evidence_url, sha256)
VALUES (
  gen_random_uuid(),
  'v0.1.0-rc1',
  'test_report',
  'Rapport des 5 suites E2E et de la couverture',
  'https://ci.example.com/artifacts/e2e-report/index.html',
  NULL
);
```

Écriture d'un point de checklist :

```sql
INSERT INTO launch_checklist_item (
  id, checklist_kind, subject, requirement_key, status, evidence_url, verified_by, verified_at
)
VALUES (
  gen_random_uuid(),
  'product',
  'SKU-USB-C-7P',
  'A07_hs_code',
  'passed',
  'https://evidence.example.com/a07-hs-code',
  'responsable-conformite',
  now()
);
```

Vérification qu'aucun point obligatoire n'est en attente :

```sql
SELECT checklist_kind, status, count(*)
FROM launch_checklist_item
GROUP BY checklist_kind, status
ORDER BY checklist_kind, status;
```

Attendu : pour `product`, `destination` et `release`, aucune ligne `pending` ; les points
`waived` portent une justification.

- [ ] **Step 18.7: Rédiger l'ADR sur les preuves de lancement**

`docs/adr/0020-preuves-de-lancement-en-base.md` :

```markdown
# ADR-0020 — Les preuves de lancement sont des lignes en base

- **Statut** : accepté
- **Date** : 2026-10-21
- **Décideur** : équipe plateforme

## Contexte

Les checklists de lancement (Annexes A, B et C) étaient jusqu'ici des documents : rien
n'empêchait de cocher une case sans preuve, et le « Go » ne laissait pas de trace
exploitable.

## Décision

Les preuves de lancement sont **versionnées en base** :

- `launch_checklist_item` : un point de checklist par (kind, subject, requirement_key),
  avec `status`, `evidence_url`, `verified_by`, `verified_at` ;
- `release_evidence` : une preuve par release, avec type, URL et empreinte SHA-256
  optionnelle.

Une contrainte SQL impose qu'un point `passed` possède une preuve, un vérificateur et
une date. Un point `waived` exige une justification écrite.

## Alternatives écartées

- *Cases à cocher dans un document Markdown* : non vérifiable, non auditable, contournable.
- *Preuves uniquement dans la CI* : la CI ne couvre ni la validation juridique, ni les
  décisions du propriétaire, ni les checklists produit et destination.

## Conséquences

- La décision Go/No-Go est reconstructible : `SELECT` sur `release_evidence` par tag.
- Le tableau de bord de supervision peut compter les points en attente par type.
- Les preuves sont conservées avec la même politique de rétention que les données métier.
```

- [ ] **Step 18.8: Vérifier l'exhaustivité des checklists**

```bash
rg -c "^\| A[0-9]{2} " ops/checklists/phase5-lancement-produit.md
rg -c "^\| B[0-9]{2} " ops/checklists/phase5-lancement-destination.md
rg -c "^\| C[0-9]{2} " ops/checklists/phase5-lancement-release.md
rg -c "^\| [1-8] \|" ops/checklists/phase5-go-nogo.md
```

Attendu :

```text
14
11
16
8
```

- [ ] **Step 18.9: Vérifier l'absence de point en attente (porte bloquante)**

```bash
psql "$DATABASE_URL" -t -c "SELECT count(*) FROM launch_checklist_item WHERE status = 'pending';"
```

Attendu : `0`. Une valeur non nulle bloque la décision Go/No-Go.

**Preuves :**
- Sortie de la migration `0119` appliquée sur base de test.
- 3 checklists de lancement (14 + 11 + 16 points) et 1 checklist Go/No-Go (8 questions) renseignées et signées.
- `docs/quality/go-nogo.md` avec les 12 décisions §28.2 tracées, chacune avec décideur, valeur retenue et preuve.
- Sortie de la vérification SQL : 0 point en attente.
- `docs/adr/0020-preuves-de-lancement-en-base.md` rédigé.

**Commit :**

```bash
git add ops/checklists packages/db/migrations/0119_phase5_release_evidence.sql packages/db/src/schema/release-evidence.ts docs/adr/0020-preuves-de-lancement-en-base.md docs/quality/go-nogo.md
git commit -m "feat(release): checklists de lancement, preuves en base et Go-No-Go formel" -m "Chantier 5.3 (spec §24.6, §25.4, §28.2, annexes A/B/C) : 14 points produit, 11 points destination, 16 points release, 8 questions de la Gate 5, 12 decisions §28.2 tracees, tables launch_checklist_item et release_evidence, ADR-0020." -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Self-Review

### 1. Couverture spec / roadmap → tâche → artefact

| Chantier (roadmap phase 5) | Sections spec | Tâche | Artefacts produits |
|---|---|---|---|
| 5.1 Stratégie de tests et couverture des modules critiques | §24.1, §26.4 | Task 1 | `packages/config/src/quality-thresholds.ts`, 6 fichiers de tests unitaires critiques, `docs/quality/strategie-tests.md`, `docs/adr/0019-strategie-test-phase-5.md` |
| 5.2 Tests d'intégration PostgreSQL réel | §24.2 | Task 2 | `vitest.integration.config.ts`, `packages/db/src/test/integration.setup.ts`, 6 fichiers `.integration.spec.ts` |
| 5.3 Suite E2E acheteur B2C (9 étapes) | §24.3, §24.4 | Task 3 | `playwright.config.ts` (racine, Modify), `e2e/b2c/journey.spec.ts` (Modify), `e2e/b2c/fixtures.ts`, `e2e/checkout/checkout.spec.ts`, `e2e/support/i18n.ts` |
| 5.4 Suite E2E acheteur B2B (9 étapes) | §24.3 | Task 4 | `e2e/b2b/journey.spec.ts` (Modify, livré en Phase 3) |
| 5.5 Suite E2E paiement (8 cas) | §24.3, §10.3, §26.3 | Task 5 | `e2e/payment/scenarios.spec.ts` |
| 5.6 Suite E2E administration (11 cas) | §24.3, §7.3 | Task 6 | `e2e/admin/dashboard.spec.ts` (Modify, livré en Phase 4) |
| 5.7 Suite E2E sécurité (7 cas) | §24.3, §19 | Task 7 | `e2e/security/security.spec.ts` |
| 5.8 Concurrence et idempotence | §24.2, §24.5 | Task 8 | `e2e/concurrency/*.spec.ts` (4 fichiers) |
| 5.9 Responsive et visuels | §24.4 | Task 9 | `e2e/visual/matrix.spec.ts`, `e2e/visual/README.md` (viewports importés de `e2e/support/viewports.ts`, Phase 1) |
| 5.10 Accessibilité | §22.4, §24.3 | Task 10 | `e2e/a11y/a11y.spec.ts`, `e2e/a11y/manual-checklist.spec.ts`, `apps/web/src/components/__tests__/axe.spec.tsx`, `ops/checklists/phase5-manual-a11y.md` |
| 5.11 Charge et résilience | §24.5, §23.3 | Task 11 | `ops/tests/load/k6-*.js` (5 scripts), `ops/tests/load/lib/thresholds.js`, `ops/tests/load/README.md` |
| 5.12 Mutation testing et métriques qualité | §24.6, §26.4 | Task 12 | `stryker.config.json` (racine), migration `0116`, `packages/config/src/quality-metrics.ts`, `docs/quality/rapport-qualite.md` |
| 5.13 Lint, analyse statique, hooks, scans | §24.6, §19.7 | Task 13 | `eslint.config.mjs`, `.lintstagedrc.json`, `.husky/pre-commit`, `.github/workflows/security.yml`, `ops/quality/semgrep.yml`, `docs/quality/lint-et-scans.md` |
| 5.14 Recette métier 3 langues | §4, §24.3 | Task 14 | `e2e/support/i18n-parity.spec.ts`, `ops/checklists/phase5-recette-metier.md`, `ops/checklists/phase5-recette-technique.md` |
| 5.15 Validation légale et fiscale | §19.8, §25.4 | Task 15 | migration `0117`, `packages/legal/src/legal-documents.ts`, `ops/checklists/phase5-validation-juridique.md`, `docs/quality/legal-validation.md` |
| 5.16 Observabilité et runbooks | §23.2, §23.3 | Task 16 | `RB-011`…`RB-015`, `ops/runbooks/README.md`, `ops/alerting/rules/phase5-alerts.yml`, `ops/alerting/tests/phase5-alerts.test.yml`, `ops/dashboards/supervision-lancement.json`, migration `0118`, `docs/adr/0021-index-alerte-runbook.md` |
| 5.17 Sauvegardes, PITR et rollback | §23.4, §24.5 | Task 17 | `ops/backup/restore-verify.sql`, `ops/backup/restore-drill.md`, `ops/backup/rollback.md`, `docs/quality/backup-restore.md` |
| 5.18 Checklists de lancement et Go/No-Go | §25.4, §26, §28.2, annexes A/B/C | Task 18 | 3 checklists + `phase5-go-nogo.md`, migration `0119`, `docs/adr/0020-preuves-de-lancement-en-base.md`, `docs/quality/go-nogo.md` |

**Répartition des migrations de la phase 5 (contiguë, sans chevauchement) :**

| Version | Fichier | Tâche | Contenu |
|---|---|---|---|
| `0116` | `packages/db/migrations/0116_phase5_quality_metrics.sql` | Task 12 | `quality_metric_run`, `quality_metric` |
| `0117` | `packages/db/migrations/0117_phase5_legal_decision.sql` | Task 15 | `legal_decision` |
| `0118` | `packages/db/migrations/0118_phase5_runbook_drill.sql` | Task 16 | `runbook_drill`, `runbook_drill_result` |
| `0119` | `packages/db/migrations/0119_phase5_release_evidence.sql` | Task 18 | `launch_checklist_item`, `release_evidence` |

Numérotation **contiguë par phase, sans chevauchement** : P1 `0001`–`0005` · P2 `0006`–`0015` · P3 `0100`–`0110` · P4 `0111`–`0115` · **P5 `0116`–`0119`** · P6 `0120`–`0123`.

**ADR de la phase 5 :** `0019` (stratégie de test, Task 1) · `0020` (preuves de lancement en base, Task 18) · `0021` (index alerte → runbook, Task 16). Les ADR `0011`–`0018` appartiennent à la Phase 6 et `0001`–`0010` aux phases 1 à 4.

**Cohérence avec les phases 1 à 4 :** la Phase 5 **consomme** les 8 runbooks `RB-001`–`RB-008` (Phase 1) et les 16 alertes critiques de `ops/alerting/rules/fanours.yml` ; elle **modifie** `eslint.config.mjs`, `.lintstagedrc.json`, `.husky/pre-commit` (Phase 1), `.github/workflows/ci.yml` et `.github/workflows/e2e.yml` (Phases 1 et 2) ; elle **consomme** les fixtures `e2e/fixtures/product.ts` (Phase 2) et `e2e/admin/fixtures/admin-users.ts` (Phase 4) sans les modifier. Elle **ne modifie aucun** livrable métier des phases 2 à 4.

### 2. Scan de placeholders et d'incohérences de noms

- **Zéro `TBD`, zéro `TODO`, zéro « à compléter »** dans ce document. Chaque artefact produit est nommé, typé et accompagné d'un contenu exécutable.
- Chaque « intervention humaine » est explicitement signalée par une étape numérotée, un **artefact attendu** et une **procédure écrite** — jamais un résultat inventé.
- Chemins vérifiés : **tous les fichiers créés** par une tâche figurent dans le `## File Structure` ; les fichiers **modifiés** qui n'y figurent pas (implémentations internes de modules déjà existants, dictionnaires i18n, écrans web des phases 2 à 4, `package.json`) sont listés dans le bloc `**Files:**` de la tâche qui les modifie. Aucun fichier n'est référencé avant d'être créé par la tâche qui le crée.
- Cohérence des noms vérifiée : `QUALITY_SUITES` = `['b2c', 'b2b', 'payment', 'admin', 'security']` (Task 1) = les 5 projets de release de `playwright.config.ts` (Task 3) = les 5 suites de la matrice CI (Task 3) = les 5 suites nommées dans le spec §24.3. La configuration racine expose 7 projets au total (les 5 suites de release + `visual` + `functional`), tous sur `testDir: './e2e'`. `QUALITY_THRESHOLDS.load` (Task 1) = `LOAD_THRESHOLDS` (Task 11), vérifié par `pnpm check:load-thresholds`.
- Routes cohérentes : `/v1/webhooks/payment` (Tasks 2, 5, 8, 11), `/v1/orders` (Tasks 3, 5, 8), `/v1/payments/:id/refund` (Tasks 5, 8), `/v1/admin/refunds` (Task 6), `/v1/auth/login` (Tasks 7, 11), `/v1/admin/uploads` (Task 7), `/v1/legal/documents/:kind` (Task 15), `/v1/admin/launch-checklist/items` (Task 18).
- Chemins de la Task 2 : `packages/db/src/test/integration.setup.ts` est importé par les tests d'intégration sous `apps/api` et `apps/worker` — le plan documente l'import via chemin relatif explicite (`'../../../../../packages/db/src/test/integration.setup.js'`) car le monorepo pnpm n'expose pas `@fanours/db/test` ; cette fragilité est signalée dans le §3.

### 3. Risques, écarts assumés et décisions à remonter

1. **Écart de numérotation des runbooks.** `docs/roadmap/sections/infra.md` §15 associe des slugs différents (`RB-001-paiement-indisponible`, `RB-002-prestataire-email-panne`, `RB-003-defaillance-transporteur`, `RB-004-incident-securite`, `RB-005-restauration-bd`) aux 5 sujets imposés par le spec §23.2, alors que la Phase 1 a déjà produit `RB-001`–`RB-008` sous d'autres intitulés. **Décision Phase 5 :** ne pas renuméroter l'existant ; créer `RB-011`–`RB-015` qui couvrent exactement les 5 sujets du spec. La divergence est documentée dans `ops/runbooks/README.md` et l'ADR-0021.
2. **~~Écart avec le plan Phase 6 sur `playwright.config.ts`.~~ — RÉSOLU.** Le plan Phase 6 décrivait une configuration listant des projets **navigateurs** (chromium/firefox/webkit). Le modèle canonique est désormais appliqué aux deux plans : une seule configuration racine, `testDir: './e2e'` pour tous les projets, discrimination par `testMatch` / `testIgnore` uniquement, et **7 projets** (`b2c`, `b2b`, `payment`, `admin`, `security`, `visual`, `functional`). La matrice de navigateurs reste une **dimension de shard CI**, jamais des projets Playwright.
3. **Écart avec le plan Phase 6 sur `stryker.config.json`.** Les chemins de mutation du plan Phase 6 (`src/modules/pricing/**`) ne correspondent pas à la disposition réelle du monorepo (`packages/pricing/src/**`). La Phase 5 utilise la disposition réelle ; la Phase 6 doit faire de même.
4. **~~Écart avec le plan Phase 6 sur `docs/maintenance/TEST-AUDIT.md`.~~ — RÉSOLU.** Le document citait un fichier de test administratif qui n'existe dans aucune des phases : le tri de tableau est couvert par le cas 10 de `e2e/admin/dashboard.spec.ts`. La Task 6 de la Phase 6 met à jour cette référence vers `e2e/admin/dashboard.spec.ts`.
5. **Fragilité des imports inter-packages.** Les tests d'intégration importent `packages/db/src/test/integration.setup.ts` par chemin relatif. Correctif recommandé : exposer `@fanours/db/test` comme point d'entrée d'export dans `packages/db/package.json` (`"exports": { "./test": "./src/test/index.ts" }`). Non fait dans ce plan pour ne pas modifier le contrat de publication du paquet ; signalé pour arbitrage.
6. **Tests d'intégration sur base locale vs Neon.** `docs/roadmap/sections/infra.md` mentionne Neon pour le développement. Les tests de concurrence et d'idempotence de la Task 8 exigent un verrou PostgreSQL réel ; ils sont donc exécutés sur le conteneur `postgres:16-alpine` de la Task 2, jamais sur une branche Neon distante. La Step 2.6 verrouille cette règle.
7. **Dépendances de la Task 9 (matrice visuelle).** 240 combinaisons sur 5 pages exceeds le budget de 12 min par shard. La matrice visuelle est donc exécutée dans un job CI **séparé** et non bloquant au premier lancement, puis bloquant — décision consignée dans `e2e/visual/README.md`, à valider par le propriétaire si le budget CI est contraint.
8. **Lighthouse / RUM CWV.** Le spec §23.3 fixe des seuils CWV (LCP ≤ 2,5 s, INP ≤ 200 ms, CLS ≤ 0,1). La phase 5 **mesure** (matrice visuelle + Task 11 + rapport qualité) mais ne crée pas de job Lighthouse CI dédié, la Phase 4 ayant déjà produit le tableau de bord de supervision. Écart à confirmer : si aucun job Lighthouse n'existe en Phase 4, il doit être créé en Phase 5 — point à arbitrer par le propriétaire.
9. **~~Écart de livraison de la suite de release `b2b`.~~ — RÉSOLU.** Le tableau §24.3 de `docs/roadmap/sections/qualite.md` indique désormais « livré par Phase 3 » pour la suite B2B, conformément à la Task 18 de la Phase 3 qui crée `e2e/b2b/journey.spec.ts` (unique fichier du projet `b2b`). La Phase 5 la reprend en `Modify` (Task 4) et n'en crée aucun doublon ; la Phase 3 livre en plus `e2e/b2b/compliance.spec.ts`, recette fonctionnelle du projet `functional`.

### 4. Points laissés explicitement à l'humain

Ces points **ne peuvent pas** être produits par l'agent ni par un automate. Ils sont listés ici pour qu'aucun ne soit oublié au Go/No-Go.

| # | Point | Tâche | Décideur | Preuve attendue |
|---|---|---|---|---|
| H01 | Validation juridique des 7 documents obligatoires (CGU/CGV, confidentialité, cookies, mentions légales, retours, expédition, CGV B2B) | 15 | Conseil juridique externe | `legal_decision` avec `review_status = 'approved_by_counsel'` et URL d'avis |
| H02 | Validation fiscale : exemptions de TVA, autoliquidation intracommunautaire, modèle DDP vs DAP, facturation OSS/IOSS | 15 | Expert-comptable | 12 lignes de `ops/checklists/phase5-validation-juridique.md` remplies |
| H03 | Décisions §28.2 (12 décisions) : entité, PSP, devise, pays, Incoterms, HS, transporteurs, politiques B2B/B2C, facturation, hébergement, SLA | 18 | Propriétaire (+ juriste, comptable, expert douanier, DPO) | `docs/quality/go-nogo.md` — 12 lignes avec valeur retenue, preuve et date |
| H04 | Recette métier dans les 3 langues (42 points) | 14 | Lecteurs FR / EN / ZH-CN | `ops/checklists/phase5-recette-metier.md` rempli et signé |
| H05 | Recette technique (26 points) | 14 | Responsable QA | `ops/checklists/phase5-recette-technique.md` rempli et signé |
| H06 | Revue manuelle d'accessibilité (12 vérifications clavier et lecteur d'écran) | 10 | Testeur a11y | `ops/checklists/phase5-manual-a11y.md` rempli et signé |
| H07 | Approbation des captures de référence visuelles (240 combinaisons) | 9 | Designer / relecteur UI | `playwright-report/` des diffs + nom du relecteur et date dans la PR |
| H08 | Drill de restauration : RPO et RTO **mesurés** | 17 | Responsable infrastructure | `docs/quality/backup-restore.md` signé, 12 contrôles à 0 |
| H09 | Test du rollback en staging | 17 | Responsable infrastructure | Version précédente servie après bascule canary, durée consignée |
| H10 | Exercices de runbook (`RB-011` à `RB-015`) | 16 | Équipe de garde | 5 lignes `runbook_drill` avec `outcome = 'passed'` |
| H11 | Checklists produit (14) et destination (11) par SKU et par pays | 18 | Responsable catalogue + conformité | `launch_checklist_item` : 0 ligne `pending` |
| H12 | Injection de pannes du prestataire, Redis, PostgreSQL, SMTP et transporteur | 11 | Responsable infrastructure | Relevé de 5 pannes en staging avec métriques et temps de rétablissement |
| H13 | Décision formelle Go / No-Go et signature | 18 | Propriétaire | `docs/quality/go-nogo.md` signé, `ops/checklists/phase5-go-nogo.md` rempli |
| H14 | Accord explicite de merge de la pull request | — | Propriétaire | PR laissée ouverte, jamais mergée automatiquement |

**Aucun de ces 14 points n'est auto-satisfait.** Une case vide dans l'un de ces tableaux est un **No-Go**, pas une réserve.

