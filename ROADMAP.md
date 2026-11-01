# Fanours — Roadmap complète

> Document de pilotage du projet Fanours, dérivé du cahier des charges `Cahier_des_charges_Fanours.md` (v1.0, 29 sections + annexes A/B/C).
>
> **Règle de lecture :** en cas de conflit, `Cahier_des_charges_Fanours.md` > ce document > `CLAUDE.md` > conventions habituelles. Si le spec est plus exigeant que la roadmap, le spec gagne — signaler l'écart.
>
> **Ce document ne remplace pas le spec.** Il opérationnalise : il nomme les décisions, les tâches, les preuves attendues et les skills à invoquer.

---

## Table des matières

| # | Section | Contenu | Source |
|---|---|---|---|
| 0 | [Comment lire cette roadmap](#0-comment-lire-cette-roadmap) | Méthode, artefacts, conventions | — |
| 1 | [Synthèse exécutable](#1-synthèse-exécutable) | Stack figée, décisions non négociables, chiffres clés | spec §1–§6 |
| 2 | [Roadmap — phases, livrables et jalons](#2-roadmap--phases-livrables-et-jalons) | Phases 0→6, gates, vague, chemin critique, risques, charge | spec §27, §28 |
| 3 | [Architecture, stack et modèle de données](#3-architecture-stack-et-modèle-de-données) | Stack figée, arborescence, modèle de données, moteur douanier, machine à états, outbox, migrations, intégrations, perf | spec §5, §7, §9, §11, §20, §21 |
| 4 | [Direction design, design system et UI/UX](#4-direction-design-design-system-et-uiux) | Identité, tokens, contraste WCAG, composants, glassmorphism, theming, i18n UI, responsive, états, a11y, anti-template | spec §3, §6, §15, §16, §22 |
| 5 | [SEO, contenu et marketing technique](#5-seo-contenu-et-marketing-technique) | URLs/i18n, robots, sitemaps, JSON-LD, on-page, CWV, paliers B2B, clusters, avis, promotions, contrôle pré-release, KPI | spec §17 |
| 6 | [Sécurité, conformité et données](#6-sécurité-conformité-et-données) | Menace STRIDE, OWASP Top 10, RBAC/anti-IDOR, authN/MFA, paiements, secrets/chiffrement, validation/XSS/SSRF/CSRF, uploads, rate limiting | spec §19 |
| 7 | [Qualité, tests et process de qualité](#7-qualité-tests-et-process-de-qualité) | Pyramide, unitaires, intégration, E2E, concurrence, responsive, a11y, charge, mutation, lint, DoD, revue, métriques | spec §24, §26 |
| 8 | [Infrastructure, CI/CD, observabilité et exploitation](#8-infrastructure-cicd-observabilité-et-exploitation) | Environnements, déploiement, topologie, pipeline CI, canary/rollback, migrations, observabilité, alertes, sauvegardes, perf, jobs, e-mail/PDF, PWA, flags, runbooks, coûts | spec §23, §25 |
| 9 | [Workflow Git et GitHub](#9-workflow-git-et-github) | Branches, commits, hooks, PR, revue, protection `main`, Actions, supply chain, releases, ADR, historique, anti-patterns | spec §24.6, §25.2, §26.6 |
| 10 | [Matrice skills × phases](#10-matrice-skills--phases) | Quel skill pour quelle phase, quels agents | CLAUDE.md §10 |
| 11 | [Preuves de livraison](#11-preuves-de-livraison) | Ce qui compte comme « fait » | CLAUDE.md §11 |

---

## 0. Comment lire cette roadmap

**Artefacts du projet**

| Fichier | Rôle | Mis à jour |
|---|---|---|
| `Cahier_des_charges_Fanours.md` | Source de vérité. 29 sections + annexes. | Rarement (décision produit) |
| `CLAUDE.md` | Instructions relues à **chaque prompt** : pratiques, interdits, stack, skills, checklist | À chaque nouvelle pratique |
| `ROADMAP.md` | Ce document. Vue d'ensemble + renvois aux sections détaillées | À chaque gate de phase |
| `docs/roadmap/sections/*.md` | Détail par domaine. Fichiers de travail, modifiables sans toucher au reste | En continu |
| `docs/superpowers/plans/YYYY-MM-DD-<phase>.md` | Plan d'exécution d'une phase, tâche par tâche | Au début de chaque phase |

**Méthode de travail**

1. Lire le spec de la section concernée avant toute implémentation.
2. Retrouver la phase et le jalon dans la section [2](#2-roadmap--phases-livrables-et-jalons).
3. Ouvrir le plan de phase correspondant dans `docs/superpowers/plans/`.
4. Exécuter tâche par tâche : test d'abord, commit par commit, PR par jalon.
5. Prouver : sortie de tests, capture, diff relu, revue sécurité.
6. Ne jamais merger sans accord explicite du propriétaire.

**Convention de lecture des sections**

Chaque section deepened suit la même trame : périmètre et références spec → décision et raison → alternative écartée → contrainte induite → procédure ou code concret → preuve attendue. Les valeurs chiffrées (seuils, quotas, durées, taux) sont reprises du spec ou explicitement signalées comme « à valider ».

---

## 1. Synthèse exécutable

### 1.1 Ce que le produit est

Plateforme B2B **et** B2C d'achat de produits en gros (MOQ, paliers de prix dégressifs), avec conformité douanière multi-pays, catalogue multilingue (FR / EN / ZH-CN), paiements, logistique suivie, dashboards client et admin.

### 1.2 Stack figée

| Couche | Choix | Alternative écartée | Raison |
|---|---|---|---|
| Front | Next.js 15 App Router + React 19 + TypeScript `strict` | SPA Vite | RSC, ISR natif, CWV |
| UI | Tailwind v4 (CSS-first) + tokens CSS | CSS Modules, shadcn/ui en socle | Les tokens du spec sont des variables CSS ; design propriétaire exigé |
| API | NestJS 11 + Zod | Express/Fastify maison | Modules, DI, guards, OpenAPI |
| Données | PostgreSQL 16 + **Drizzle** | Prisma | Contraintes `EXCLUDE`/`CHECK`/`tstzrange` = SQL natif versionné |
| Cache/queue | Redis 7 + **BullMQ** | SQS/Temporal | Jobs retardés requis ; DLQ + retry ; pas de service managé en plus |
| Recherche | PostgreSQL (`tsvector` + `pg_trgm` + GIN) | Elasticsearch | Catalogue < 100k SKU ; interface `SearchProvider` pour la bascule |
| Objets | S3-compatible + CDN | Disque local | Médias et documents privés hors bundle |
| Paiement | Interface `PaymentProvider` | Prestataire en dur | Éligibilité pays non tranchée (spec §28.2) |
| Observabilité | Logs JSON + OpenTelemetry + métriques + tracking d'erreurs | Logs texte | `correlation_id` de bout en bout |
| Tests | Vitest + Testing Library + Playwright + k6 + Stryker | Jest | ESM natif, compatible pnpm workspaces |
| Déploiement | **Conteneurs multi-services** (web / api / worker) | Vercel | Worker long-running, webhooks stables, 1 fournisseur, coût maîtrisé |

### 1.3 Les 10 décisions non négociables

1. Le **front n'est jamais source de vérité** pour prix, taxe, frais, stock ou autorisation. Recalcul serveur obligatoire à chaque checkout, snapshot persisté dans la commande.
2. **Aucune donnée de carte** stockée, loggée ou transitée par l'API. Éléments hébergés + tokenisation.
3. **Aucun taux, seuil ou tarif douanier en dur** dans le code. Règles = données versionnées avec source officielle + date de consultation. Un contrôle CI refuse les constantes.
4. **Aucune IA ne confirme seule** un code HS, un prix final, une importabilité ou un remboursement.
5. **Jamais de sous-valorisation**, de fausse catégorie/origine ou de fractionnement de colis.
6. **MFA obligatoire** sur admin / finance / compliance. **Double contrôle** sur action financière sensible.
7. **Journal d'audit immuable** (append-only, trigger SQL bloquant UPDATE/DELETE), sans secret dedans.
8. **Idempotence** sur toute opération sensible : commande, paiement, remboursement, webhook, email, job.
9. **Migration destructive en prod** = accord explicite + restauration **prouvée par exécution**, pas par intention.
10. **Merge d'une PR = accord humain explicite.** Jamais de force-push, jamais de dev sur `main`.

### 1.4 Chiffres clés du spec

| Domaine | Valeur |
|---|---|
| Langues | 3 (`fr-FR`, `en`, `zh-CN`) |
| Rôles admin | 9 |
| États de commande | 20, avec 34 transitions explicites |
| Événements métier | 16 (spec §21.3) + 19 supplémentaires |
| Niveaux de certitude douanière | 4 (`validé`, `estimatif`, `incomplet`, `spécialiste`) |
| Environnements | 4 (local, test, staging, production) |
| Étapes du pipeline CI | 10 |
| Suites E2E obligatoires | 5 (B2C 9 étapes, B2B 9 étapes, paiement 8 cas, admin 11 cas, sécurité 7 cas) |
| Combinaisons responsive | 8 viewports × 3 langues × 2 thèmes |
| Cible CWV (p75) | LCP ≤ 2,5 s · INP ≤ 200 ms · CLS ≤ 0,1 |
| Fiabilité | RPO ≤ 24 h · RTO ≤ 8 h · disponibilité cible 99,5 % |
| Phases | 7 (0 → 6) |
| Risques identifiés | 10 (spec §28.1) |
| Décisions à confirmer | 12 (spec §28.2) |

### 1.5 Les 12 décisions qui appartiennent au propriétaire

Entité juridique et pays d'établissement · éligibilité au prestataire de paiement · devise de référence et règles FX · pays prioritaires et pays interdits · Incoterms et modèle de paiement des taxes (DDP vs DAP) · catégories réglementées et codes HS · politique B2B (retours, MOQ, délais) · politique B2C · exigences de facturation locales · région d'hébergement et transferts de données · SLA / RPO / RTO / support · outils analytics, consentement et durées de conservation.

> Ces points ne sont **jamais** tranchés par l'IA. Ils sont transformés en **dossier de décision** (options, impacts, recommandation) et attendent une validation humaine écrite. Le détail est en [section 12 de `CLAUDE.md`](CLAUDE.md) et en Phase 0 de la [section 2](#2-roadmap--phases-livrables-et-jalons).

---

## 2. Roadmap — phases, livrables et jalons

→ **[Détail complet : `docs/roadmap/sections/roadmap-phases.md`](docs/roadmap/sections/roadmap-phases.md)**

Contient : le principe de séquence, les 7 phases (0→6) avec pour chacune objectif, prérequis, chantiers, livrable concret, référence spec, taille, skills, agents, preuves, critère de sortie et blocages ; la matrice de traçabilité spec→phase (les 29 sections + annexes A/B/C) ; les vagues de la Phase 1 ; les 7 jalons de validation (Gate 0→6) ; la Definition of Ready ; la Definition of Done ; le chemin critique ; la matrice de risques (10 risques spec + 6 risques projet, avec P/I, signal d'alerte, propriétaire, Plan A, Plan B et déclencheur) ; l'estimation de charge ; les coûts à valider ; la liste de ce qui n'est **jamais** négocié.

| Phase | Objetif en une phrase | Sortie |
|---|---|---|
| **0 — Cadrage et conformité** | Trancher les 12 décisions du propriétaire avant d'écrire du code métier | Dossier de décision signé, registre de conformité initial |
| **1 — Fondations et design system** | Monorepo, tokens, auth, RBAC, outbox, observabilité, CI | Socle déployable, design system gelé, pipeline vert |
| **2 — Catalogue et commerce** | Catalogue, paliers, panier, checkout, paiement | Parcours B2C/B2B fonctionnels de bout en bout |
| **3 — B2B, logistique et douane** | Devis, règle TradeCompliance, expéditions, suivi | Moteur de règles branché sur les commandes réelles |
| **4 — Dashboards et opérations** | Espace client et console admin | Opérations pilotables sans intervention technique |
| **5 — Assurance qualité et lancement** | 5 suites E2E, charge, a11y, runbooks, go/no-go | Dossier de preuves de lancement |
| **6 — Améliorations post-lancement** | CRO, contenu, SEO, itérations | Backlog de croissance priorisé |

### 2.1 Plans d'exécution par phase

Chaque phase a un plan d'exécution détaillé dans `docs/superpowers/plans/`, au format fichier / interfaces / étapes `- [ ]` en cycle TDD (test échoue → sortie collée → implémentation → sortie verte → commit).

| Phase | Plan | Tâches | Migrations |
|---|---|---|---|
| 0 — Cadrage et conformité | [`2026-10-21-phase-0-cadrage-conformite.md`](docs/superpowers/plans/2026-10-21-phase-0-cadrage-conformite.md) | 13 | — (documents, pas de SQL) |
| 1 — Fondations et design system | [`2026-10-21-phase-1-fondations-et-design-system.md`](docs/superpowers/plans/2026-10-21-phase-1-fondations-et-design-system.md) | 15 | `0001`–`0005` |
| 2 — Catalogue et commerce | [`2026-10-21-phase-2-catalogue-et-commerce.md`](docs/superpowers/plans/2026-10-21-phase-2-catalogue-et-commerce.md) | 19 | `0006`–`0015` |
| 3 — B2B, logistique et douane | [`2026-10-21-phase-3-b2b-logistique-douane.md`](docs/superpowers/plans/2026-10-21-phase-3-b2b-logistique-douane.md) | 18 | `0100`–`0110` |
| 4 — Dashboards et opérations | [`2026-10-21-phase-4-dashboards-et-operations.md`](docs/superpowers/plans/2026-10-21-phase-4-dashboards-et-operations.md) | 14 | `0111`–`0115` |
| 5 — Assurance qualité et lancement | [`2026-10-21-phase-5-assurance-qualite-et-lancement.md`](docs/superpowers/plans/2026-10-21-phase-5-assurance-qualite-et-lancement.md) | 18 | `0116`–`0119` |
| 6 — Améliorations post-lancement | [`2026-10-21-phase-6-ameliorations-post-lancement.md`](docs/superpowers/plans/2026-10-21-phase-6-ameliorations-post-lancement.md) | 10 | `0120`–`0123` |

> Chaque plan se termine par une **Self-Review** en 4 parties (conformité au spec, couverture des critères, risques et limites, checklist de vérification). Les migrations sont **append-only** : jamais de `down`, checksum SHA-256 vérifié, `SET lock_timeout` / `statement_timeout`.

---

## 3. Architecture, stack et modèle de données

→ **[Détail complet : `docs/roadmap/sections/architecture.md`](docs/roadmap/sections/architecture.md)**

Contient : les décisions de stack figées et leurs alternatives écartées (Drizzle vs Prisma, conteneurs multi-services vs Vercel, BullMQ) ; l'arborescence du monorepo pnpm+Turbo et les règles de dépendance entre packages ; le modèle de données table par table (colonnes, index, invariants SQL) ; le moteur TradeCompliance et sa résolution en 7 étapes ; la machine à états de commande (20 états, 34 transitions) ; le pattern outbox (schéma, publication, consommation idempotente) ; la stratégie de tests par couche ; la stratégie de migration expand/contract avec ses 4 verrous ; les 8 interfaces d'intégration externe ; le budget de performance.

---

## 4. Direction design, design system et UI/UX

→ **[Détail complet : `docs/roadmap/sections/design.md`](docs/roadmap/sections/design.md)**

Contient : l'identité visuelle (rose en accent, crédibilité B2B) ; les tokens CSS complets (clair et sombre) ; le **recalcul de contraste WCAG 2.2 AA sur 61 paires** avec les 4 échecs corrigés ; les 6 familles de composants ; le budget glassmorphism et ses zones interdites ; le theming clair/sombre/auto avec anti-flash ; l'i18n UI (3 langues, 3 devises) ; le responsive mobile-first ; les 16 états systémiques FR/EN/ZH ; l'accessibilité opérationnelle ; les 14 interdits anti-template ; le processus de design.

---

## 5. SEO, contenu et marketing technique

→ **[Détail complet : `docs/roadmap/sections/seo.md`](docs/roadmap/sections/seo.md)**

Contient : l'architecture d'URL par locale et la table hreflang ; le `robots.txt` (36 chemins privés sur 3 locales) ; l'arborescence de sitemaps et les 9 critères d'exclusion produit ; le JSON-LD complet (Organization, WebSite+SearchAction, BreadcrumbList, Product B2B avec `eligibleQuantity` MOQ) ; les formules title/meta par type de page avec exemples FR/EN/ZH ; les budgets CWV par route ; la page produit unique indexable avec paliers dans le HTML rendu ; les 10 clusters de contenu ; les 6 conditions d'un avis vérifié ; le modèle promotionnel ; la checklist pré-release de 45 points ; les 17 KPI ; l'interdiction formelle de promettre un classement.

---

## 6. Sécurité, conformité et données

→ **[Détail complet : `docs/roadmap/sections/securite.md`](docs/roadmap/sections/securite.md)**

Contient : le modèle de menace STRIDE par flux (12 flux) ; la matrice OWASP Top 10 × modules Fanours ; le contrôle d'accès (9 rôles, 14 actions sensibles, schéma de permission type-chèque, anti-IDOR en cascade, double contrôle quatre-yeux) ; l'authentification (cookie opaque, TOTP anti-rejeu, codes de récupération Argon2id, Google Sign-In, reauthentification) ; la sécurité des paiements (aucune donnée de carte, HMAC webhook, idempotence, rapprochement, anti-double-remboursement, chargebacks) ; les données, secrets et chiffrement (inventaire, chiffrement en colonne par enveloppe, audit immuable à chaîne de hachage) ; la validation, XSS, injections, SSRF, CSRF, CORS, CSP et en-têtes ; les téléversements (magic bytes, quarantaine, antivirus) ; la limitation de débit et l'anti-abus ; la conformité réglementaire (RGPD, PIPL, obligations acheteurs) ; la douane et l'anti-fraude (cohérence valeur/quantité/poids, détection d'écartement de colis, origine réelle, statut des codes HS, verrou d'expédition, suspension d'urgence) ; la chaîne de build et le durcissement d'infrastructure ; les tests de sécurité et la **checklist de 25 items bloquants avant merge**.

---

## 7. Qualité, tests et process de qualité

→ **[Détail complet : `docs/roadmap/sections/qualite.md`](docs/roadmap/sections/qualite.md)**

Contient : la philosophie de tests ; les tests unitaires (MOQ, paliers, priorité des remises, arrondis, transitions, réservations, autorisation par rôle, versionnement des règles, paiements) ; les tests d'intégration (PostgreSQL réel via Testcontainers) ; les **5 suites E2E Playwright obligatoires** (B2C 9 étapes, B2B 9 étapes, paiement 8 cas, admin 11 cas, sécurité 7 cas) et le **modèle E2E canonique** (une seule configuration `playwright.config.ts` à la racine, 7 projets, un fichier = un projet, une seule suite de release par cas du spec §24.3, viewports importés de `e2e/support/viewports.ts` et jamais redéclarés) ; les tests de concurrence et d'idempotence ; les tests responsive et visuels (8 viewports × 3 langues × 2 thèmes) ; l'accessibilité (axe-core, jest-axe, @axe-core/playwright) ; les tests de charge et de résilience (k6) ; la mutation testing (Stryker) et la couverture ; le lint, le format, l'analyse statique et les hooks pre-commit ; la Definition of Done ; le process de revue (approbations selon la criticité) ; les métriques de qualité.

---

## 8. Infrastructure, CI/CD, observabilité et exploitation

→ **[Détail complet : `docs/roadmap/sections/infra.md`](docs/roadmap/sections/infra.md)**

Contient : les 4 environnements et la matrice de ce qui change entre eux ; la stratégie de déploiement (comparatif, recommandation conteneurs, plan de sortie si invalidée) ; la topologie de production ; le pipeline CI en 10 jobs avec ses conditions ; le déploiement progressif canary et le rollback automatique ; les migrations en production et la restauration ; l'observabilité (logs JSON, redaction PII, 25 métriques, traces `correlation_id`) ; les alertes et l'on-call ; les sauvegardes et le test de restauration mensuel ; la performance et les CWV en production ; les 15 types de jobs BullMQ et le pattern idempotent ; l'e-mail et le PDF ; la PWA et le service worker ; les feature flags ; les runbooks ; les coûts et le budget d'infrastructure.

---

## 9. Workflow Git et GitHub

→ **[Détail complet : `docs/roadmap/sections/git.md`](docs/roadmap/sections/git.md)**

Contient : la stratégie de branches (trunk-based, regex de nom, durée de vie) ; les Conventional Commits 1.0 et `commitlint.config.ts` ; Husky + lint-staged avec le hook pre-commit réel ; la politique de contournement de hooks (déclaration obligatoire dans la PR) ; la structure de PR et le **template réel** ; le process de revue ; la protection de `main` (ruleset) ; les workflows GitHub Actions et les scopes de secrets ; Dependabot et la politique de lockfile ; les versions et la promotion **par digest** ; les ADR ; les règles d'historique ; la checklist de fin de phase ; le tableau des 10 anti-patterns.

---

## 10. Matrice skills × phases

### 10.1 Par phase

| Phase | Skills à invoquer | Agents à mobiliser |
|---|---|---|
| **0 — Cadrage** | `brainstorming` · `caveman` · `copy-editing` | `Legal Compliance Checker` · `Tax Strategist` · `Software Architect` · `UI Designer` |
| **1 — Fondations** | `frontend-design` · `taste-skill` · `theme-factory` · `composition-patterns` · `react-best-practices` · `web-design-guidelines` · `ui-ux-pro-max` · `api-platform-engineer` · `database-optimizer` · `devops` · `test-driven-development` | `Software Architect` · `UI Designer` · `Backend Architect` · `DevOps Automator` · `SEO Specialist` · `Test Automation Engineer` |
| **2 — Catalogue & commerce** | `test-driven-development` · `payments-engineering` · `api-platform-engineer` · `database-optimizer` · `frontend-design` · `composition-patterns` · `seo-audit` · `emil-design-eng` · `web-design-guidelines` | `Backend Architect` · `Payments & Billing Engineer` · `Database Optimizer` · `Frontend Developer` · `Test Automation Engineer` · `SEO Specialist` |
| **3 — B2B, logistique, douane** | `database-optimizer` · `api-platform-engineer` · `test-driven-development` · `payments-engineering` · `security-review` · `pdf` | `Backend Architect` · `Database Optimizer` · `Payments & Billing Engineer` · `Senior SecOps Engineer` · `Legal Compliance Checker` · `Tax Strategist` |
| **4 — Dashboards & opérations** | `frontend-design` · `composition-patterns` · `ui-ux-pro-max` · `dataviz` · `react-best-practices` · `performance` · `web-design-guidelines` | `Frontend Developer` · `UI Designer` · `Data Visualization Engineer` · `Code Reviewer` · `security-auditor` |
| **5 — QA & lancement** | `verification-before-completion` · `security-review` · `webapp-testing` · `a11y` · `screenshot` · `seo-audit` · `copy-editing` · `finishing-a-development-branch` · `caveman-commit` | `Reality Checker` · `Evidence Collector` · `security-auditor` · `qa-e2e` · `seo-auditor` · `ui-ux-checker` · `Legal Compliance Checker` |
| **6 — Post-lancement** | `cro` · `pricing` · `seo-audit` · `programmatic-seo` · `analytics` · `content-strategy` · `ab-testing` | `Growth Hacker` · `SEO Specialist` · `Content Creator` |

### 10.2 Systématiques (à chaque phase)

`caveman` (permanent) · `brainstorming` avant toute feature · `writing-plans` avant une phase · `test-driven-development` sur toute logique métier · `verification-before-completion` avant de dire « terminé » · `requesting-code-review` avant merge · `finishing-a-development-branch` en fin de phase · `surgical-patch` / `safe-refactor` selon le type d'édition · `caveman-commit` pour les messages · `code-review` / `security-review` sur le diff.

### 10.3 Par domaine

| Domaine touched | Skill | Agent |
|---|---|---|
| UI, design system, tokens | `frontend-design` · `taste-skill` · `theme-factory` · `emil-design-eng` · `apple-design` | `UI Designer` · `ui-ux-checker` |
| React / Next.js | `react-best-practices` · `composition-patterns` | `Frontend Developer` |
| Accessibilité | `web-design-guidelines` · `a11y` · `wcag` | `ui-ux-checker` |
| Base de données | `database-optimizer` · `database-reliability-engineer` · `migration` | `Database Optimizer` |
| API | `api-platform-engineer` | `Backend Architect` · `API Platform Engineer` |
| Paiement | `payments-engineering` | `Payments & Billing Engineer` |
| Infra / CI | `devops` · `lean-build` | `DevOps Automator` · `Infrastructure Maintainer` |
| SEO | `seo-audit` · `schema` · `ai-seo` · `site-architecture` | `SEO Specialist` · `seo-auditor` |
| Contenu | `content-strategy` · `copywriting` · `copy-editing` | `Content Creator` |
| Sécurité | `security-review` | `security-auditor` · `Senior SecOps Engineer` |
| Conformité / douane | `copy-editing` (relecture juridique) | `Legal Compliance Checker` · `Tax Strategist` |
| Tests | `test-driven-development` · `webapp-testing` | `Test Automation Engineer` · `qa-e2e` |
| Git / PR | `caveman-commit` · `caveman-review` · `requesting-code-review` | `Code Reviewer` · `reviewer` |

---

## 11. Preuves de livraison

**Aucun « terminé » sans preuve.** Une phase n'est close que si les items ci-dessous sont produits et collés dans la PR ou dans le dossier de phase.

| # | Preuve | Où |
|---|---|---|
| 1 | Sortie de tests (unit, intégration, E2E) avec chiffres | corps de PR |
| 2 | Test de non-régression si bug corrigé | corps de PR |
| 3 | Revue sécurité ciblée (authz, injection, XSS, secrets, uploads, rate limit) | commentaire de revue |
| 4 | Diff relu : pas de debug, TODO, secret, code mort | relecture auteur + reviewer |
| 5 | 3 langues + 2 thèmes à jour si UI touchée | `pnpm i18n:check` + captures |
| 6 | États loading / vide / erreur implémentés | E2E + revue UI |
| 7 | A11y : clavier, focus, contraste, annonces ; revue manuelle si parcours critique | rapport axe + notes manuelles |
| 8 | Perf : pas de régression mesurable (bundle, requêtes, images) | rapport Lighthouse / k6 |
| 9 | Migration testée sur base de test ; destructive = accord + restauration prouvée | sortie du job de restauration |
| 10 | SEO si page publique : title, meta, canonical, hreflang, JSON-LD, robots | rapport `seo-audit` |
| 11 | Docs à jour (README, runbook, `.env` en **noms de variables uniquement**) | diff |
| 12 | Commits conventionnels, branche correcte, **pas** sur `main` | `git log` + `git status` |
| 13 | PR ouverte avec preuves — **jamais mergée sans accord explicite** | lien PR |

**Porte de sortie de phase** = checklist du gate correspondant dans la [section 2](#2-roadmap--phases-livrables-et-jalons), plus la Definition of Done (25 points) de la [section 7](#7-qualité-tests-et-process-de-qualité).

---

*Roadmap générée à partir de `Cahier_des_charges_Fanours.md` v1.0. Toute divergence entre ce document et le spec est un défaut de la roadmap, pas du spec.*
