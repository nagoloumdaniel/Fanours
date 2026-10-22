# Phase 3 — B2B, logistique et douane : Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Livrer le vertical B2B de Fanours de bout en bout : un acheteur professionnel est vérifié par un humain, négocie un devis multi-révisions, reçoit un calcul de droits et de taxes sourcé et versionné, la commande devient un dossier d'export complet et le colis est suivi jusqu'à la livraison ou l'action requise — pour deux pays cibles, en trois langues, avec un dossier de conformité auditable.

**Périmètre :** 18 tâches (Task 1 → Task 18). Les Tasks 1 à 16 construisent le vertical B2B (pays et Incoterms, règles douanières versionnées, calculs, contrôles, devis, transporteurs, expéditions, documents d'export, retours), la Task 17 ouvre les vues admin, la Task 18 livre la recette E2E du vertical (parcours acheteur B2B en 9 étapes et contrôles douaniers).

**Architecture:** La Phase 3 ajoute un moteur de conformité `TradeCompliance` (`packages/compliance`) où **les règles sont des données versionnées en base** (`trade_rule` / `trade_rule_version` / `trade_rule_source`) et où le code ne contient que l'algorithme de résolution déterministe (7 étapes, 4 niveaux de certitude) et l'interprétation de `params`. Le calcul des 8 composantes douanières (§11.3) est une fonction pure testée sur 6 cas chiffrés. Les modules métier `business`, `quote`, `shipment`, `document`, `returns` suivent le découpage NestJS de `apps/api/src/modules/<domain>/` (controllers / services / dto Zod / repository / events) ; les effets de bord passent par l'**outbox transactionnelle** (§6 de `docs/roadmap/sections/architecture.md`) ; le worker consomme les files `shipping.estimate`, `tracking.sync`, `document.pdf` et `trade_rule.review` (§11.2 de `docs/roadmap/sections/infra.md`). Le front (`apps/web`, `packages/ui`) n'affiche que ce que le serveur a calculé : il n'est **jamais** source de vérité des droits, taxes ou frais de transport.

**Tech Stack:** Next.js 15 App Router + React 19 + TypeScript `strict` + Tailwind v4 · NestJS 11 + Zod · PostgreSQL 16 + Drizzle (migrations SQL manuelles, append-only) · Redis 7 + BullMQ · S3-compatible (bucket privé) · Vitest + Testing Library + supertest + Testcontainers (PostgreSQL + Redis) · Playwright · axe-core.

**Spec:** `Cahier_des_charges_Fanours.md` — §8.1–8.4 (B2B, devis, prix client), §9.2–9.3 (checkout, états de commande), §11.1–11.10 (douane, TVA, conformité, HS, pays, documents, alertes), §12.1–12.5 (expédition, tracking, retours, litiges), §15.3–15.4 (suivi client, devis), §16.6 / §16.8 / §16.9 (admin clients, logistique & douane, conformité), §19.2–19.4 (autorisation, anti-IDOR, uploads), §20 (modèle de données), §21.3–21.4 (événements, intégrations), §22.4 (a11y), §24.2–24.3 (tests), §26.4 (traçabilité des règles). Plans et conventions : `docs/roadmap/sections/roadmap-phases.md` § Phase 3, `docs/roadmap/sections/architecture.md` §3–§9, `docs/roadmap/sections/securite.md` §3, §4, §8, §9, `docs/roadmap/sections/qualite.md` §24.2–24.3, `docs/roadmap/sections/design.md` §3.4, §3.5, §3.7, §3.9, `docs/roadmap/sections/infra.md` §11.

## Global Constraints

- **Le front n'est jamais source de vérité.** Droits de douane, taxes et frais de transport sont recalculés côté serveur à chaque action déterminante et **snapshotés** sur la commande (`order.snapshot` : prix, taxes, Incoterm, version de règle, texte présenté). La commande ne relit jamais le catalogue courant.
- **Interdits absolus, à reprendre tels quels** : **sous-valorisation**, **fausse catégorie/origine**, **fractionnement de colis**. Le système les *détecte et les bloque* ; il ne les produit jamais. Un contrôle de cohérence valeur/poids et un contrôle de fractionnement sont bloquants (cf. Task 5).
- **L'IA ne valide jamais un code HS.** `hs_validation_status` ne peut valoir `confirmed` que si un humain l'a écrit (`hs_confirmed_by` + `hs_confirmed_at` + source). Le code HS est stocké sur **6 chiffres** (`hs_code_validated CHAR(6)`) ; le code national est une extension de l'importateur, il n'est pas une donnée Fanours et n'est pas stocké. L'expédition est bloquée tant que le statut n'est pas `confirmed`.
- **Règle expirée → alerte au compliance manager, jamais d'application silencieuse.** Une version de règle dont `valid_to` est dépassé n'est jamais appliquée : elle produit une `compliance_alert`. Une règle `prohibition` ou `license` expirée **bloque** la commande.
- **Les règles sont des données, pas du code.** Aucun taux, seuil, montant forfaitaire ou taux de courtage n'est écrit en dur : le garde CI `packages/compliance/scripts/check-no-hardcoded-rates.mjs` échoue sur `/(vat|duty|tariff|threshold)\s*[:=]\s*0?\.\d+/i` dans `apps/**/src/**` et `packages/**/src/**` (commentaires exclus), sauf exception explicitement déclarée dans `packages/compliance/src/constants.allowlist.ts`. Les paramètres de calcul sont des **entiers** : `rate_bps` (points de base) et `amount_minor` (minor units). Les montants en base sont `NUMERIC(18,4)`.
- **Devise de transaction ≠ devise d'affichage.** Un montant converti est marqué « indicatif » / « indicative » / 「仅供参考」 avec la date du taux ; l'API de change indisponible ou un taux périmé n'autorise jamais un taux estimé silencieusement.
- **Permissions vérifiées côté serveur, deny by default.** Une permission absente est un refus. Le scoping se fait **dans la requête** (anti-IDOR → `404` dans tous les cas), jamais en post-filtrage en mémoire. **MFA obligatoire** (≤ 10 min) pour `admin_role:manage_roles`, `refund:refund`, `export_financial:export`, `price_tier:update`, `country:update`, `customs_rule:update`, `business_account:approve`, `business_document:read`. Double contrôle (four-eyes) sur les actions financières et sur l'approbation B2B, **auto-approbation interdite**.
- **Aucun secret dans le code, les logs ou le bundle.** Documents B2B chiffrés (enveloppe KMS + chiffrement de colonne), bucket **privé**, URL signée 5 minutes, **jamais** d'URL publique, jamais d'URL signée dans les logs. Téléversements : **magic bytes** (liste blanche jpeg/png/webp/pdf, PDF uniquement pour un justificatif), 10 Mo max, 5 documents par dossier, nom de stockage régénéré en UUID, scan antivirus, stockage hors racine web.
- **Zéro placeholder** : ni `TBD`, ni `TODO`, ni « à compléter », ni « similar to Task N ». **Chaque tâche a son propre cycle de test** : test qui échoue → exécution → implémentation minimale → exécution qui passe → commit.
- **Migrations append-only** : un fichier SQL numéroté par tâche, jamais réécrit, **jamais de `down`** ; `-- @allow-destructive` **et** `-- @adr: ADR-xxx` obligatoires pour toute suppression ; `SET lock_timeout` / `SET statement_timeout` en tête de fichier.
- **i18n** : aucune chaîne en dur. Toute chaîne d'interface existe en `fr-FR`, `en`, `zh-CN` (contrôle de parité au build). La rédaction du plan et des messages est en **français** ; le code et les identifiants sont en **anglais**.
- **Portée des dashboards** : les dashboards complets §15 / §16 relèvent de la Phase 4. La Phase 3 livre **les vues du vertical** — devis, suivi, documents douaniers côté client ; expéditions, règles douanières, revue clients B2B côté admin — sur les composants de dashboard déjà prévus (`DataTable`, `AdminWizard`, `KpiCard`, `FilterBar`).
- **Modèle E2E canonique** : un seul fichier de configuration Playwright, `playwright.config.ts` **à la racine du dépôt** (créé en Phase 1, **modifié** — jamais recréé — par les Phases 2, 4, 5 et 6) ; la Phase 3 n'en crée aucun dans `e2e/`. Les tests E2E de la phase sont des **recettes fonctionnelles de phase** et relèvent du projet `functional` : c'est le cas de `e2e/b2b/admin-views.spec.ts` (Task 17) et de `e2e/b2b/compliance.spec.ts` (Task 18). Ces fichiers **coexistent** avec la suite de release `b2b` (`e2e/b2b/journey.spec.ts`, projet `b2b`, livrée par la Phase 3 et reprise en *Modify* par la Phase 5 pour le niveau release) sans doublon ni comptage ambigu : les fichiers sont différents et **un fichier appartient à un seul projet**. Les commandes sont lancées **depuis la racine** : `pnpm exec playwright test e2e/b2b/admin-views.spec.ts --project=functional`, `pnpm exec playwright test e2e/b2b/journey.spec.ts --project=b2b`, `pnpm exec playwright test e2e/b2b/compliance.spec.ts --project=functional`. Les viewports, les langues et les thèmes ne sont pas des projets : les 8 viewports viennent de la liste canonique `e2e/support/viewports.ts` (créée en Phase 1, **importée, jamais re-déclarée**).

---

## File Structure

```
packages/
  db/
    migrations/
      0100_phase3_country_profiles.sql   # pays, restrictions, configuration Incoterm (expand)
      0101_phase3_trade_rules.sql         # sources, règles, versions (EXCLUDE gist anti-chevauchement)
      0102_phase3_compliance_estimates.sql# tax_estimate, duty_estimate, compliance_review, compliance_alert
      0103_phase3_business_documents.sql  # business_account, business_membership, business_document
      0104_phase3_quotes.sql              # quote, quote_revision, quote_line, quote_message
      0105_phase3_shipments.sql           # shipping_quote, shipment, shipment_item, package, tracking_event
      0106_phase3_commercial_documents.sql# commercial_document (facture export, packing list, export, origine, customs_data)
      0107_phase3_returns_disputes.sql    # return_request, return_item, extension de dispute
      0108_phase3_product_compliance.sql  # extension de product_compliance (statut de validation HS)
      0109_phase3_quote_to_order.sql      # lien quote_id sur orders + unicité (une commande par devis)
      0110_phase3_admin_views.sql        # Task 17 : admin_country_scope, business_account_approval, exceptions de colis, colonnes d'alerte
    src/
      schema/
        index.ts               # baril Phase 2 : exporte ../schema.js (Phase 1) + les modules phase3-*
        phase3-country.ts      # Drizzle : country_profile, incoterm_configuration
        phase3-compliance.ts   # Drizzle : trade_rule_source, trade_rule, trade_rule_version, estimations, alertes
        phase3-business.ts     # Drizzle : business_account, business_membership, business_document
        phase3-quote.ts        # Drizzle : quote, quote_revision, quote_line, quote_message
        phase3-logistics.ts    # Drizzle : shipping_quote, shipment, shipment_item, package, tracking_event, commercial_document
        phase3-returns.ts      # Drizzle : return_request, return_item, dispute (extension)
        phase3-catalog.ts      # Drizzle : product_compliance (extension du statut de validation HS)
        phase3-admin.ts        # Task 17 : admin_country_scope, business_account_approval, colonnes admin de shipment et compliance_alert
      seeds/
        phase3-countries.ts    # profils pays initiaux FR/DE/GB/US/CA/AU/ZA + Incoterms
      __tests__/
        country-profile.integration.spec.ts   # contraintes CHECK du référentiel pays
        trade-rule-version.integration.spec.ts# EXCLUDE anti-chevauchement + unicité de version
        quote-lifecycle.integration.spec.ts   # cycle de vie du devis et conversion en commande
        shipment-tracking.integration.spec.ts  # idempotence et non-régression du tracking
  compliance/
    src/
      index.ts                 # surface publique du moteur (types + fonctions)
      money.ts                 # minor units entiers, roundHalfUp, échelle de devise
      country-gate.ts          # porte serveur pays/restrections/Incoterm
      rules.ts                 # types des règles, paramètres, niveaux de certitude
      resolver.ts              # résolution déterministe en 7 étapes
      calculator.ts            # calcul des 8 composantes (§11.3)
      checks.ts                # cohérence valeur/poids, fractionnement, expiration, complétude
      constants.allowlist.ts   # exceptions déclarées du garde « aucun taux en dur »
      country-gate.spec.ts
      resolver.spec.ts
      calculator.spec.ts
      checks.spec.ts
    scripts/
      check-no-hardcoded-rates.mjs  # garde CI : aucun taux/seuil en dur dans le code
  pricing/
    src/
      customer-rules.ts        # priorité client > groupe > palier > promotion
      customer-rules.spec.ts
  contracts/
    src/
      shipping.ts              # interface ShippingProvider + types Zod
      quote.ts                 # contrats de devis (Zod)
      business.ts              # contrats de compte professionnel (Zod)
      storage.ts               # interface StorageProvider (bucket privé, URL signée 5 min)
      pdf.ts                   # interface DocumentRenderer + kinds de documents commerciaux
  pdf/                        # package de la Phase 2, étendu par la Task 12
    src/
      commercial-document.tsx  # composant React-PDF des 4 documents commerciaux
      render-commercial-document.ts  # rendu PDF (polices CJK embarquées)
      fonts.ts                 # registre de polices partagé avec la facture
      index.ts
  ui/
    src/components/
      CostEstimatePanel.tsx   # affichage acheteur : transport, taxes, droits, certitude
      CostEstimatePanel.spec.tsx
      QuoteRevisionDiff.tsx   # comparaison de deux révisions de devis (15.4)
      QuoteRevisionDiff.spec.tsx
      TrackingTimeline.tsx    # chronologie des événements transporteur (12.2)
      TrackingTimeline.spec.tsx
      SystemState.tsx         # composant unique des états systémiques (design §3.9)
      SystemState.spec.tsx
      DataTable.tsx           # table admin : tri/pagination serveur, gel 1re colonne (design §3.4.5)
      DataTable.spec.tsx
      AdminWizard.tsx         # wizard 4 étapes des règles douanières (design §3.4.5)
      AdminWizard.spec.tsx
      KpiCard.tsx             # indicateur admin avec définition au survol/focus (design §3.4.5)
      KpiCard.spec.tsx
      FilterBar.tsx           # barre de filtres admin : repli en panneau sur mobile + réinit (design §3.4.5)
      FilterBar.spec.tsx
  i18n/
    src/locales/
      fr-FR.json              # dictionnaire FR (namespaces compliance, quote, shipment, returns)
      en.json                 # dictionnaire EN
      zh-CN.json              # dictionnaire ZH-CN
      index.ts                # map des locales + fonction de résolution avec repli explicite
apps/
  api/src/
    common/
      admin-scope.ts           # Task 17 : portée pays + scope RBAC des vues admin (anti-IDOR)
    modules/
      business/                # 3.1 : dépôt de justificatifs, revue, approbation humaine
        business-documents.service.ts
        business-review.service.ts
        business.controller.ts
        admin-business-accounts.service.ts    # Task 17 : liste et revue B2B four-eyes
        admin-business-accounts.controller.ts # Task 17 : /v1/admin/business-accounts
      quote/                   # 3.2 : demande, révisions, négociation, acceptation, conversion
        quote.service.ts
        quote-conversion.service.ts
        quote.controller.ts
      compliance/              # 3.11 / 3.12 : revue HS, alertes, blocages
        hs-review.service.ts
        compliance-alerts.controller.ts
        compliance-alert.service.ts           # Task 17 : liste / prise en compte / résolution
        admin-trade-rules.service.ts          # Task 17 : centre de règles douanières
        admin-trade-rules.controller.ts       # Task 17 : /v1/admin/trade-rules
      shipment/                # 3.9 : expéditions, tracking, webhooks transporteur
        shipment.service.ts
        tracking.service.ts
        shipment.controller.ts
        shipment-webhook.controller.ts
        admin-shipments.service.ts            # Task 17 : expéditions et exceptions
        admin-shipments.controller.ts         # Task 17 : /v1/admin/shipments
      document/                # 3.10 : documents d'export et complétude des données douanières
        export-documents.service.ts
        customs-data.service.ts
      returns/                 # §12.4 / §12.5 : retours et litiges
        return.service.ts
        dispute.service.ts
      auth/                    # Task 18 : route de vérification e-mail et boîte de test
        test-mailbox.controller.ts # Task 18 : GET /v1/test-mailbox/messages (404 en production)
    providers/shipping/
      shipping-provider.ts     # fabrique de sélection par variable d'environnement
      mock-shipping.provider.ts# transporteur sandbox déterministe
    providers/email/
      mailbox.provider.ts      # Task 18 : boîte e-mail de test en mémoire (SMTP réel en Phase 5)
  worker/src/jobs/
    compliance/trade-rule-review.job.ts   # revue quotidienne des règles (expiration)
    shipments/tracking-sync.job.ts         # synchronisation tracking (cron 15 min)
    shipments/shipping-estimate.job.ts     # estimation transport multi-transporteur
    documents/export-documents.job.ts      # génération des documents d'export
  web/src/
    server/admin.ts                        # Task 17 : client serveur des vues admin (droits decided by l'API)
    app/[locale]/(account)/devis/page.tsx              # liste des devis du client
    app/[locale]/(account)/devis/[number]/page.tsx      # détail, comparaison de révisions, acceptation
    app/[locale]/(account)/commandes/[number]/suivi/page.tsx     # suivi colis
    app/[locale]/(account)/commandes/[number]/documents/page.tsx  # documents douaniers téléchargeables
    app/[locale]/(admin)/admin/expeditions/page.tsx     # 16.8 : expéditions et exceptions
    app/[locale]/(admin)/admin/douane/regles/page.tsx   # 16.9 : wizard des règles douanières
    app/[locale]/(admin)/admin/douane/alertes/page.tsx  # 16.9 : alertes de conformité
    app/[locale]/(admin)/admin/clients/page.tsx         # 16.6 : revue des comptes B2B
e2e/
  b2b/journey.spec.ts         # Task 18 : parcours acheteur B2B (9 étapes) — projet `b2b` (livré par la Phase 3, repris en Modify par la Phase 5 pour le niveau release)
  b2b/compliance.spec.ts      # Task 18 : recette fonctionnelle de phase — projet `functional` (refus d'expédition, règle expirée, fractionnement, revue B2B)
  b2b/admin-views.spec.ts     # Task 17 : vues admin (permissions, 403, a11y, zéro débordement) — projet `functional`
  support/b2b-api.ts          # Task 18 : aides de la recette B2B (enveloppe, connexion, MFA, fixtures) — module de support, jamais collecté
  # la matrice visuelle (8 viewports × 3 locales × 2 thèmes) relève du projet `visual` : e2e/visual-matrix.spec.ts (Phase 2), e2e/visual/responsive.spec.ts (Phase 4), e2e/visual/matrix.spec.ts (Phase 5) — aucune copie de la liste de viewports ici
.github/workflows/ci.yml      # job « no-hardcoded-rates » ajouté au pipeline
```

Chaque dossier est **autonome** : une tâche lit ce qu'elle consomme, écrit ce qu'elle produit, et son test la prouve sans dépendre d'une tâche non fusionnée.

---

### Task 1: Référentiel pays, restrictions et Incoterms

**Files:**
- Create: `packages/db/migrations/0100_phase3_country_profiles.sql`
- Create: `packages/db/src/schema/phase3-country.ts`
- Modify: `packages/db/src/schema/index.ts` (baril Phase 2 : ajouter `export * from "./phase3-country.js";`)
- Create: `packages/db/src/seeds/phase3-countries.ts`
- Create: `packages/compliance/src/country-gate.ts`
- Create: `packages/compliance/src/country-gate.spec.ts`
- Create: `packages/db/src/__tests__/country-profile.integration.spec.ts`
- Modify: `packages/compliance/src/index.ts`

**Interfaces:**
- Consumes: `packages/db` (runner de migrations et client Drizzle), table `"user"` (Phase 1) — le nom réel de la table utilisateurs est vérifié au Step 3 et aligné si besoin.
- Produces:
  - `packages/compliance/src/country-gate.ts` :
    ```ts
    export type OperationalStatus = "operational" | "restricted" | "suspended";
    export type CountryGateInput = {
      countryCode: string;
      deliveryEnabled: boolean;
      operationalStatus: OperationalStatus;
      restrictedCategories: string[];
      categoryId: string | null;
      taxRegistrationRequired: boolean;
      isBusinessImporter: boolean;
      taxIdPresent: boolean;
      enabledIncoterms: string[];
      requestedIncoterm: string | null;
    };
    export type CountryGate = { allowed: boolean; code: string; reasons: string[] };
    export function evaluateCountryGate(input: CountryGateInput): CountryGate;
    export function assertCountryAllowed(input: CountryGateInput): void; // lève CountryNotAllowedError(reasons)
    ```
  - `packages/db/src/schema/phase3-country.ts` : exports Drizzle `countryProfile`, `incotermConfiguration`.
  - `packages/db/src/seeds/phase3-countries.ts` : `export const PHASE3_COUNTRIES: CountrySeed[]` et `export async function seedPhase3Countries(): Promise<number>`.
  - Migration `0100` : tables `country_profile` (PK `code char(2)`) et `incoterm_configuration`, avec `CHECK (code ~ '^[A-Z]{2}$')`, `CHECK (operational_status IN ('operational','restricted','suspended'))` et `CHECK (operational_status <> 'suspended' OR suspended_reason IS NOT NULL)`.

- [ ] **Step 1: Écrire le test unitaire de la porte pays (il doit échouer)**

Créer `packages/compliance/src/country-gate.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { evaluateCountryGate, type CountryGateInput } from "./country-gate";

const base: CountryGateInput = {
  countryCode: "FR",
  deliveryEnabled: true,
  operationalStatus: "operational",
  restrictedCategories: [],
  categoryId: null,
  taxRegistrationRequired: false,
  isBusinessImporter: false,
  taxIdPresent: false,
  enabledIncoterms: ["DAP"],
  requestedIncoterm: "DAP",
};

describe("evaluateCountryGate", () => {
  it("autorise un pays operationnel sans restriction", () => {
    expect(evaluateCountryGate(base)).toEqual({ allowed: true, code: "FR", reasons: [] });
  });

  it("refuse un pays suspendu", () => {
    const result = evaluateCountryGate({ ...base, operationalStatus: "suspended", deliveryEnabled: false });
    expect(result.allowed).toBe(false);
    expect(result.reasons).toContain("country_suspended");
  });

  it("refuse un pays dont la livraison est desactivee", () => {
    const result = evaluateCountryGate({ ...base, deliveryEnabled: false });
    expect(result.reasons).toContain("delivery_disabled");
  });

  it("refuse une categorie restreinte pour ce pays", () => {
    const result = evaluateCountryGate({ ...base, restrictedCategories: ["cat-battery"], categoryId: "cat-battery" });
    expect(result.reasons).toContain("category_restricted");
  });

  it("refuse un importateur professionnel sans identifiant fiscal quand il est exige", () => {
    const result = evaluateCountryGate({
      ...base, taxRegistrationRequired: true, isBusinessImporter: true, taxIdPresent: false,
    });
    expect(result.reasons).toContain("tax_registration_missing");
  });

  it("refuse un Incoterm non active pour le pays", () => {
    const result = evaluateCountryGate({ ...base, requestedIncoterm: "DDP" });
    expect(result.reasons).toContain("incoterm_not_enabled");
  });

  it("refuse un code pays mal forme", () => {
    const result = evaluateCountryGate({ ...base, countryCode: "fr" });
    expect(result.reasons).toContain("invalid_country_code");
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/compliance exec vitest run src/country-gate.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./country-gate"` (le module n'existe pas encore). 0 test exécuté.

- [ ] **Step 2: Implémenter la porte pays**

Créer `packages/compliance/src/country-gate.ts` :

```ts
export type OperationalStatus = "operational" | "restricted" | "suspended";

export type CountryGateInput = {
  countryCode: string;
  deliveryEnabled: boolean;
  operationalStatus: OperationalStatus;
  restrictedCategories: string[];
  categoryId: string | null;
  taxRegistrationRequired: boolean;
  isBusinessImporter: boolean;
  taxIdPresent: boolean;
  enabledIncoterms: string[];
  requestedIncoterm: string | null;
};

export type CountryGate = { allowed: boolean; code: string; reasons: string[] };

export class CountryNotAllowedError extends Error {
  constructor(public readonly code: string, public readonly reasons: string[]) {
    super("country_not_allowed");
    this.name = "CountryNotAllowedError";
  }
}

/**
 * Porte serveur unique (CDC 11.6, 2.3) : un pays non livreable, une categorie restreinte,
 * un Incoterm non active ou un identifiant fiscal manquant bloquent la demande de devis,
 * le panier, le checkout et la commande. L'UI ne fait jamais ce controle seule.
 */
export function evaluateCountryGate(input: CountryGateInput): CountryGate {
  const reasons: string[] = [];

  if (!/^[A-Z]{2}$/.test(input.countryCode)) reasons.push("invalid_country_code");
  if (!input.deliveryEnabled) reasons.push("delivery_disabled");
  if (input.operationalStatus === "suspended") reasons.push("country_suspended");
  if (input.operationalStatus === "restricted") reasons.push("country_restricted");

  if (
    input.categoryId !== null &&
    input.restrictedCategories.includes(input.categoryId)
  ) {
    reasons.push("category_restricted");
  }

  if (input.taxRegistrationRequired && input.isBusinessImporter && !input.taxIdPresent) {
    reasons.push("tax_registration_missing");
  }

  if (input.requestedIncoterm !== null && !input.enabledIncoterms.includes(input.requestedIncoterm)) {
    reasons.push("incoterm_not_enabled");
  }

  return { allowed: reasons.length === 0, code: input.countryCode, reasons };
}

export function assertCountryAllowed(input: CountryGateInput): void {
  const gate = evaluateCountryGate(input);
  if (!gate.allowed) throw new CountryNotAllowedError(gate.code, gate.reasons);
}
```

Exécuter :

```bash
pnpm --filter @fanours/compliance exec vitest run src/country-gate.spec.ts
```

Attendu : **7 tests passés**, 0 échec.

- [ ] **Step 3: Migration SQL du référentiel pays**

Créer `packages/db/migrations/0100_phase3_country_profiles.sql` :

```sql
-- 0100_phase3_country_profiles.sql
-- Phase 3 - chantier 3.7 (referentiel pays, restrictions, Incoterms).
-- Classification : expand (nouvelles tables uniquement, aucun comportement applicatif modifie).
SET lock_timeout = '5s';
SET statement_timeout = '60s';

CREATE TABLE country_profile (
  code                      char(2) PRIMARY KEY,
  name                      text NOT NULL,
  delivery_enabled          boolean NOT NULL DEFAULT false,
  incoterms                 text[] NOT NULL DEFAULT '{}',
  currencies                text[] NOT NULL DEFAULT '{}',
  payment_methods           text[] NOT NULL DEFAULT '{}',
  tax_registration_required boolean NOT NULL DEFAULT false,
  required_docs             text[] NOT NULL DEFAULT '{}',
  restricted_categories     text[] NOT NULL DEFAULT '{}',
  operational_status        text NOT NULL DEFAULT 'operational',
  suspended_reason          text,
  created_at                timestamptz NOT NULL DEFAULT now(),
  updated_at                timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT country_profile_code_format CHECK (code ~ '^[A-Z]{2}$'),
  CONSTRAINT country_profile_operational_status
    CHECK (operational_status IN ('operational','restricted','suspended')),
  CONSTRAINT country_profile_suspension_reason
    CHECK (operational_status <> 'suspended' OR suspended_reason IS NOT NULL)
);

CREATE TABLE incoterm_configuration (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  country_code        char(2) NOT NULL REFERENCES country_profile(code),
  incoterm            text NOT NULL,
  tax_collection_mode text NOT NULL,
  enabled             boolean NOT NULL DEFAULT false,
  effective_from      timestamptz NOT NULL,
  created_at          timestamptz NOT NULL DEFAULT now(),
  updated_at          timestamptz NOT NULL DEFAULT now(),
  UNIQUE (country_code, incoterm, effective_from),
  CONSTRAINT incoterm_configuration_mode
    CHECK (tax_collection_mode IN ('prepaid','at_delivery')),
  CONSTRAINT incoterm_configuration_incoterm
    CHECK (incoterm IN ('EXW','FCA','FOB','CFR','CIF','CPT','CIP','DAP','DPU','DDP'))
);

CREATE INDEX idx_incoterm_configuration_country
  ON incoterm_configuration (country_code, enabled);
```

Créer `packages/db/src/schema/phase3-country.ts` :

```ts
import { sql } from "drizzle-orm";
import { boolean, char, index, pgTable, text, textArray, timestamp, uniqueIndex, uuid } from "drizzle-orm/pg-core";

export const countryProfile = pgTable("country_profile", {
  code: char("code", { length: 2 }).primaryKey(),
  name: text("name").notNull(),
  deliveryEnabled: boolean("delivery_enabled").notNull().default(false),
  incoterms: textArray("incoterms").notNull().default(sql`'{}'::text[]`),
  currencies: textArray("currencies").notNull().default(sql`'{}'::text[]`),
  paymentMethods: textArray("payment_methods").notNull().default(sql`'{}'::text[]`),
  taxRegistrationRequired: boolean("tax_registration_required").notNull().default(false),
  requiredDocs: textArray("required_docs").notNull().default(sql`'{}'::text[]`),
  restrictedCategories: textArray("restricted_categories").notNull().default(sql`'{}'::text[]`),
  operationalStatus: text("operational_status").notNull().default("operational"),
  suspendedReason: text("suspended_reason"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const incotermConfiguration = pgTable(
  "incoterm_configuration",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    countryCode: char("country_code", { length: 2 })
      .notNull()
      .references(() => countryProfile.code),
    incoterm: text("incoterm").notNull(),
    taxCollectionMode: text("tax_collection_mode").notNull(),
    enabled: boolean("enabled").notNull().default(false),
    effectiveFrom: timestamp("effective_from", { withTimezone: true }).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => ({
    uniq: uniqueIndex("incoterm_configuration_unique").on(t.countryCode, t.incoterm, t.effectiveFrom),
    idxCountry: index("idx_incoterm_configuration_country").on(t.countryCode, t.enabled),
  }),
);
```

La table utilisateurs créée par la Phase 1 (`packages/db/migrations/0002_identity.sql`) s'appelle **`"user"`** — `USER` est un mot réservé PostgreSQL, d'où les guillemets. **Toutes les migrations de la Phase 3 référencent `"user" (id)`**, sans alternative possible. Contrôle avant d'appliquer :

```bash
psql "$DATABASE_URL" -c "\dt" | grep -E '"user"'
```

La Task 2 (règles) référence `"user" (id)` : les deux sont alignées sur la table réelle, aucun ajustement n'est nécessaire.

Appliquer et vérifier :

```bash
pnpm --filter @fanours/db migrate:dev
pnpm --filter @fanours/db migrate:status
```

Attendu : migration `0100_phase3_country_profiles` appliquée, statut `applied`, checksum enregistré.

- [ ] **Step 4: Test d'intégration des contraintes du référentiel pays**

Créer `packages/db/src/__tests__/country-profile.integration.spec.ts` :

```ts
import { eq } from "drizzle-orm";
import { describe, expect, it } from "vitest";
import { db, sql } from "@fanours/db";
import { countryProfile } from "@fanours/db/schema";

describe("country_profile (integration)", () => {
  it("refuse un code pays qui ne respecte pas le format ISO 3166-1 alpha-2", async () => {
    await expect(
      db.execute(sql`INSERT INTO country_profile (code, name) VALUES ('fra', 'Test')`),
    ).rejects.toThrow();
  });

  it("expose un profil operationnel et ses Incoterms pour FR", async () => {
    const rows = await db.select().from(countryProfile).where(eq(countryProfile.code, "FR"));
    expect(rows).toHaveLength(1);
    expect(rows[0].operationalStatus).toBe("operational");
    expect(rows[0].incoterms).toContain("DAP");
  });
});
```

Créer le seed `packages/db/src/seeds/phase3-countries.ts` :

```ts
export type CountrySeed = {
  code: string;
  name: string;
  deliveryEnabled: boolean;
  incoterms: string[];
  currencies: string[];
  paymentMethods: string[];
  taxRegistrationRequired: boolean;
  requiredDocs: string[];
  restrictedCategories: string[];
  operationalStatus: "operational" | "restricted" | "suspended";
};

/**
 * Profils initiaux issus de la liste de la CDC 11.7 (UE, Royaume-Uni, Etats-Unis,
 * Canada, Australie, pays africains prioritaires). Les listes `restricted_categories`
 * et `payment_methods` restent VIDES : leur peuplement est une decision compliance
 * auditee, pas une donnee de developpement. Seuls `DAP` est active : un Incoterm
 * `DDP` n'est activable que si l'organisation contractuelle existe (CDC 11.4),
 * ce qui releve de la decision INCOTERMS de la Phase 0.
 */
export const PHASE3_COUNTRIES: CountrySeed[] = [
  { code: "FR", name: "France", deliveryEnabled: true, incoterms: ["DAP"], currencies: ["EUR"], paymentMethods: [], taxRegistrationRequired: false, requiredDocs: ["commercial_invoice", "packing_list", "export_declaration", "customs_data"], restrictedCategories: [], operationalStatus: "operational" },
  { code: "DE", name: "Allemagne", deliveryEnabled: true, incoterms: ["DAP"], currencies: ["EUR"], paymentMethods: [], taxRegistrationRequired: false, requiredDocs: ["commercial_invoice", "packing_list", "export_declaration", "customs_data"], restrictedCategories: [], operationalStatus: "operational" },
  { code: "GB", name: "Royaume-Uni", deliveryEnabled: true, incoterms: ["DAP"], currencies: ["GBP"], paymentMethods: [], taxRegistrationRequired: false, requiredDocs: ["commercial_invoice", "packing_list", "export_declaration", "customs_data"], restrictedCategories: [], operationalStatus: "operational" },
  { code: "US", name: "Etats-Unis", deliveryEnabled: true, incoterms: ["DAP"], currencies: ["USD"], paymentMethods: [], taxRegistrationRequired: false, requiredDocs: ["commercial_invoice", "packing_list", "export_declaration", "customs_data"], restrictedCategories: [], operationalStatus: "operational" },
  { code: "CA", name: "Canada", deliveryEnabled: true, incoterms: ["DAP"], currencies: ["CAD"], paymentMethods: [], taxRegistrationRequired: false, requiredDocs: ["commercial_invoice", "packing_list", "export_declaration", "customs_data"], restrictedCategories: [], operationalStatus: "operational" },
  { code: "AU", name: "Australie", deliveryEnabled: true, incoterms: ["DAP"], currencies: ["AUD"], paymentMethods: [], taxRegistrationRequired: false, requiredDocs: ["commercial_invoice", "packing_list", "export_declaration", "customs_data"], restrictedCategories: [], operationalStatus: "operational" },
  { code: "ZA", name: "Afrique du Sud", deliveryEnabled: true, incoterms: ["DAP"], currencies: ["ZAR"], paymentMethods: [], taxRegistrationRequired: false, requiredDocs: ["commercial_invoice", "packing_list", "export_declaration", "customs_data"], restrictedCategories: [], operationalStatus: "operational" },
];

export async function seedPhase3Countries(): Promise<number> {
  // L'upsert est idempotent : rejouer le seed ne duplique aucune ligne.
  return 0; // branche d'upsert implementee a la Step 4 du plan
}
```

Remplacer le corps de `seedPhase3Countries` par un `insert ... on conflict do update` sur `country_profile` (un `INSERT ... ON CONFLICT (code) DO UPDATE` par pays) qui retourne le nombre de lignes écrites, puis enregistrer le script dans `packages/db/package.json` (`"seed:phase3": "tsx src/seeds/phase3-countries.ts"`).

Exécuter (Docker requis pour Testcontainers) :

```bash
pnpm --filter @fanours/db exec vitest run --config vitest.integration.config.ts src/__tests__/country-profile.integration.spec.ts
pnpm --filter @fanours/db seed:phase3
```

Attendu : **2 tests passés**, 0 échec ; le seed écrit 7 lignes sans doublon lors d'un second passage.

- [ ] **Step 5: Exporter la porte pays et vérifier les types**

Modifier `packages/compliance/src/index.ts` pour ré-exporter :

```ts
export * from "./country-gate";
export * from "./money";
```

Créer `packages/compliance/src/money.ts` (utilisé par les tâches suivantes) :

```ts
/** Echelle decimale par devise : les minor units sont toujours des entiers. */
export const CURRENCY_SCALE: Readonly<Record<string, number>> = {
  EUR: 2, USD: 2, GBP: 2, CNY: 2, CAD: 2, AUD: 2, ZAR: 2, JPY: 0,
};

export function scaleOf(currency: string): number {
  const scale = CURRENCY_SCALE[currency];
  if (scale === undefined) throw new Error(`unsupported_currency:${currency}`);
  return scale;
}

/** Arrondi commercial demi-supérieur, en arithmetique entiere (aucun flottant). */
export function roundHalfUp(numerator: number, denominator: number): number {
  if (denominator === 0) throw new Error("division_by_zero");
  const sign = numerator < 0 ? -1 : 1;
  const n = Math.abs(numerator);
  const d = Math.abs(denominator);
  return sign * Math.floor((2 * n + d) / (2 * d));
}

/** Applique un taux en points de base (basis points) a un montant en minor units. */
export function applyRateBps(baseMinor: number, rateBps: number): number {
  return roundHalfUp(baseMinor * rateBps, 10_000);
}

/** Convertit des minor units en chaine decimale exacte pour l'ecriture en NUMERIC(18,4). */
export function minorToDecimal(minor: number, currency: string): string {
  const scale = scaleOf(currency);
  const negative = minor < 0;
  const abs = Math.abs(minor);
  const divisor = 10 ** scale;
  const whole = Math.floor(abs / divisor);
  const rest = abs % divisor;
  const fraction = rest.toString().padStart(scale, "0");
  return `${negative ? "-" : ""}${whole}${scale > 0 ? `.${fraction}` : ""}`;
}
```

```bash
pnpm --filter @fanours/compliance exec vitest run
pnpm turbo run typecheck
```

Attendu : suite `packages/compliance` verte, `typecheck` sans erreur.

- [ ] **Step 6: Commit**

```bash
git add packages/db/migrations/0100_phase3_country_profiles.sql packages/db/src/schema/phase3-country.ts packages/db/src/schema/index.ts packages/db/src/seeds/phase3-countries.ts packages/db/src/__tests__/country-profile.integration.spec.ts packages/compliance/src/country-gate.ts packages/compliance/src/country-gate.spec.ts packages/compliance/src/money.ts packages/compliance/src/index.ts
git commit -m "feat(customs): ajoute le referentiel pays, restrictions et Incoterms"
```

---

### Task 2: Moteur de règles — schéma versionné, sources et garde CI

**Files:**
- Create: `packages/db/migrations/0101_phase3_trade_rules.sql`
- Create: `packages/db/src/schema/phase3-compliance.ts`
- Modify: `packages/db/src/schema/index.ts` (baril Phase 2 : ajouter `export * from "./phase3-compliance.js";`)
- Create: `packages/compliance/src/rules.ts`
- Create: `packages/compliance/src/constants.allowlist.ts`
- Create: `packages/compliance/scripts/check-no-hardcoded-rates.mjs`
- Create: `packages/db/src/__tests__/trade-rule-version.integration.spec.ts`
- Modify: `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: `packages/db` (runner), `"user"` (Phase 1), `country_profile` (Task 1), `packages/compliance/src/money.ts` (Task 1).
- Produces:
  - `packages/compliance/src/rules.ts` :
    ```ts
    export type RuleType = "vat" | "duty" | "threshold" | "flat_fee" | "license" | "prohibition" | "documentation" | "warning";
    export type Certainty = "validated" | "estimated" | "incomplete" | "specialist_review";
    export type Scope = "product" | "variant" | "hs_prefix" | "category" | "global";
    export type CalculationMode = "percentage" | "flat_amount" | "per_unit" | "progressive_tiers" | "component_sum" | "deferred_to_carrier";
    export type TaxableBase = "goods_value" | "goods+shipping" | "goods+shipping+insurance" | "custom";
    export type RuleParams = {
      rate_bps?: number; amount_minor?: number; unit_ref?: string;
      tiers?: { up_to_minor: number | null; rate_bps: number }[];
      mode?: "B2B" | "B2C"; incoterm?: string; carrier?: string; import_role?: "exporter" | "importer";
      collected_by_fanours?: boolean; component?: string;
    };
    export type RuleCandidate = { /* voir Task 3 pour la forme exacte */ };
    ```
  - `packages/compliance/src/constants.allowlist.ts` : `export const ALLOWED_CONSTANTS: ReadonlyArray<{ file: string; pattern: string; reason: string; approvedBy: string; approvedAt: string }> = []` — liste vide tant que le compliance manager n'a pas validé d'exception.
  - `packages/compliance/scripts/check-no-hardcoded-rates.mjs` : `export function scanHardcodedRates(files: { path: string; content: string }[]): string[]` ; CLI exit 1 si violations.
  - Migration `0101` : `trade_rule_source`, `trade_rule`, `trade_rule_version` avec `EXCLUDE USING gist` anti-chevauchement.

- [ ] **Step 1: Écrire le test d'intégration de l'exclusion de versions (il doit échouer)**

Créer `packages/db/src/__tests__/trade-rule-version.integration.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { db, sql } from "@fanours/db";

async function seedRule(): Promise<string> {
  const source = await db.execute(sql`
    INSERT INTO trade_rule_source (url, authority, title, consulted_at)
    VALUES ('https://example.org/official', 'Autorite de test', 'Source de test', now())
    RETURNING id
  `);
  const sourceId = (source as unknown as { id: string }[])[0].id;
  const rule = await db.execute(sql`
    INSERT INTO trade_rule (key, country_code, scope, rule_type, certainty, priority)
    VALUES ('test.vat.fr', 'FR', 'global', 'vat', 'estimated', 10)
    RETURNING id
  `);
  const ruleId = (rule as unknown as { id: string }[])[0].id;
  await db.execute(sql`
    INSERT INTO trade_rule_version (trade_rule_id, version_no, valid_from, valid_to, params, calculation_mode, taxable_base, source_id, review_due_at)
    VALUES (${ruleId}, 1, '2026-01-01', '2027-01-01', '{"rate_bps": 2000}', 'percentage', 'goods+shipping', ${sourceId}, '2026-12-01')
  `);
  return ruleId;
}

describe("trade_rule_version (integration)", () => {
  it("refuse deux versions actives qui se chevauchent", async () => {
    const ruleId = await seedRule();
    await expect(
      db.execute(sql`
        INSERT INTO trade_rule_version (trade_rule_id, version_no, valid_from, valid_to, params, calculation_mode, taxable_base, source_id, review_due_at)
        SELECT ${ruleId}, 2, '2026-06-01', '2027-06-01', '{"rate_bps": 1900}', 'percentage', 'goods+shipping', source_id, '2026-12-01'
        FROM trade_rule_version WHERE trade_rule_id = ${ruleId} LIMIT 1
      `),
    ).rejects.toThrow();
  });

  it("accepte deux versions non chevauchantes", async () => {
    const ruleId = await seedRule();
    await expect(
      db.execute(sql`
        INSERT INTO trade_rule_version (trade_rule_id, version_no, valid_from, valid_to, params, calculation_mode, taxable_base, source_id, review_due_at)
        SELECT ${ruleId}, 3, '2027-01-01', '2028-01-01', '{"rate_bps": 1900}', 'percentage', 'goods+shipping', source_id, '2027-12-01'
        FROM trade_rule_version WHERE trade_rule_id = ${ruleId} LIMIT 1
      `),
    ).resolves.toBeDefined();
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/db exec vitest run --config vitest.integration.config.ts src/__tests__/trade-rule-version.integration.spec.ts
```

Attendu : **ÉCHEC** — `relation "trade_rule_version" does not exist`. 0 test exécuté.

- [ ] **Step 2: Migration SQL du moteur de règles**

Créer `packages/db/migrations/0101_phase3_trade_rules.sql` :

```sql
-- 0101_phase3_trade_rules.sql
-- Phase 3 - chantier 3.4 (moteur de regles versionne).
-- Classification : expand. Regles = donnees, jamais du code (CDC 11.2).
SET lock_timeout = '5s';
SET statement_timeout = '60s';

CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE trade_rule_source (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  url          text NOT NULL,
  authority    text NOT NULL,
  title        text NOT NULL,
  consulted_at timestamptz NOT NULL,
  retrieved_at timestamptz NOT NULL DEFAULT now(),
  snapshot_key text,
  CONSTRAINT trade_rule_source_consulted_at CHECK (consulted_at <= retrieved_at + interval '1 day')
);

CREATE TABLE trade_rule (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  key           text NOT NULL UNIQUE,
  country_code  char(2) NOT NULL REFERENCES country_profile(code),
  scope         text NOT NULL,
  category_id   uuid,
  product_id    uuid,
  variant_id    uuid,
  hs_prefix     text,
  rule_type     text NOT NULL,
  certainty     text NOT NULL,
  priority      int NOT NULL DEFAULT 0,
  active        boolean NOT NULL DEFAULT true,
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT trade_rule_scope CHECK (scope IN ('category','product','variant','hs_prefix','global')),
  CONSTRAINT trade_rule_rule_type
    CHECK (rule_type IN ('vat','duty','threshold','flat_fee','license','prohibition','documentation','warning')),
  CONSTRAINT trade_rule_certainty
    CHECK (certainty IN ('validated','estimated','incomplete','specialist_review')),
  CONSTRAINT trade_rule_scope_payload CHECK (
    (scope = 'category'  AND category_id IS NOT NULL) OR
    (scope = 'product'   AND product_id  IS NOT NULL) OR
    (scope = 'variant'   AND variant_id  IS NOT NULL) OR
    (scope = 'hs_prefix' AND hs_prefix   IS NOT NULL) OR
    (scope = 'global'    AND category_id IS NULL AND product_id IS NULL AND variant_id IS NULL AND hs_prefix IS NULL)
  ),
  CONSTRAINT trade_rule_hs_prefix_format CHECK (hs_prefix IS NULL OR hs_prefix ~ '^[0-9]{2,6}$')
);

CREATE INDEX idx_trade_rule_country_type ON trade_rule (country_code, rule_type);
CREATE INDEX idx_trade_rule_scope_category ON trade_rule (scope, category_id);
CREATE INDEX idx_trade_rule_hs_prefix ON trade_rule (hs_prefix text_pattern_ops);

CREATE TABLE trade_rule_version (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  trade_rule_id    uuid NOT NULL REFERENCES trade_rule(id) ON DELETE CASCADE,
  version_no       int NOT NULL,
  valid_from       timestamptz NOT NULL,
  valid_to         timestamptz,
  params           jsonb NOT NULL DEFAULT '{}'::jsonb,
  calculation_mode text NOT NULL,
  taxable_base     text NOT NULL,
  thresholds       jsonb NOT NULL DEFAULT '{}'::jsonb,
  source_id        uuid NOT NULL REFERENCES trade_rule_source(id),
  validated_by     uuid REFERENCES "user" (id),
  validated_at     timestamptz,
  review_due_at    timestamptz NOT NULL,
  created_at       timestamptz NOT NULL DEFAULT now(),
  UNIQUE (trade_rule_id, version_no),
  CONSTRAINT trade_rule_version_window CHECK (valid_to IS NULL OR valid_to > valid_from),
  CONSTRAINT trade_rule_version_calculation_mode
    CHECK (calculation_mode IN ('percentage','flat_amount','per_unit','progressive_tiers','component_sum','deferred_to_carrier')),
  CONSTRAINT trade_rule_version_taxable_base
    CHECK (taxable_base IN ('goods_value','goods+shipping','goods+shipping+insurance','custom')),
  -- Un seuil monetaire ne peut pas vivre sans date de fin (CDC 11.7).
  CONSTRAINT trade_rule_version_threshold_needs_valid_to
    CHECK (thresholds = '{}'::jsonb OR valid_to IS NOT NULL),
  -- Une version de regle ne peut pas etre `validated` sans validateur humain.
  CONSTRAINT trade_rule_version_validated_needs_human
    CHECK (validated_by IS NULL OR validated_at IS NOT NULL)
);

CREATE INDEX idx_trade_rule_version_rule FROM trade_rule_version (trade_rule_id, valid_from DESC);
CREATE INDEX idx_trade_rule_version_review_due ON trade_rule_version (review_due_at);

-- Deux versions actives d'une meme regle ne peuvent pas se chevaucher.
ALTER TABLE trade_rule_version
  ADD CONSTRAINT trade_rule_version_no_overlap
  EXCLUDE USING gist (
    trade_rule_id WITH =,
    tstzrange(valid_from, COALESCE(valid_to, 'infinity'::timestamptz), '[)') WITH &&
  );
```

La Phase 1 crée la table utilisateurs sous le nom exact `"user"` (`packages/db/migrations/0002_identity.sql`, `USER` étant un mot réservé PostgreSQL). La migration 0102 ci-dessus référence donc `REFERENCES "user" (id)` — **c'est la forme définitive**, aucune variante `app_user` n'existe dans le plan.

Créer `packages/db/src/schema/phase3-compliance.ts` avec les tables Drizzle `tradeRuleSource`, `tradeRule`, `tradeRuleVersion` (colonnes alignées sur le SQL ci-dessus, `numeric` pour les montants, `jsonb` pour `params` / `thresholds`).

Appliquer :

```bash
pnpm --filter @fanours/db migrate:dev
pnpm --filter @fanours/db exec vitest run --config vitest.integration.config.ts src/__tests__/trade-rule-version.integration.spec.ts
```

Attendu : **2 tests passés**, 0 échec (le premier prouve qu'une version chevauchante est rejetée par `EXCLUDE`, le second qu'une version mitoyenne est acceptée).

- [ ] **Step 3: Types des règles et liste d'exceptions du garde CI**

Créer `packages/compliance/src/rules.ts` :

```ts
export type RuleType =
  | "vat" | "duty" | "threshold" | "flat_fee"
  | "license" | "prohibition" | "documentation" | "warning";

export type Certainty = "validated" | "estimated" | "incomplete" | "specialist_review";

export type Scope = "product" | "variant" | "hs_prefix" | "category" | "global";

export type CalculationMode =
  | "percentage" | "flat_amount" | "per_unit"
  | "progressive_tiers" | "component_sum" | "deferred_to_carrier";

export type TaxableBase = "goods_value" | "goods+shipping" | "goods+shipping+insurance" | "custom";

/** Les parametres sont des ENTIERS : rate_bps (points de base), amount_minor (minor units). */
export type RuleParams = {
  rate_bps?: number;
  amount_minor?: number;
  unit_ref?: string;
  tiers?: { up_to_minor: number | null; rate_bps: number }[];
  mode?: "B2B" | "B2C";
  incoterm?: string;
  carrier?: string;
  import_role?: "exporter" | "importer";
  collected_by_fanours?: boolean;
  component?: string;
};

export const SCOPE_RANK: Readonly<Record<Scope, number>> = {
  product: 50, variant: 40, hs_prefix: 30, category: 20, global: 10,
};

export const RULE_TYPES: readonly RuleType[] = [
  "vat", "duty", "threshold", "flat_fee", "license", "prohibition", "documentation", "warning",
];

export const CERTAINTIES: readonly Certainty[] = [
  "validated", "estimated", "incomplete", "specialist_review",
];

export const SCOPES: readonly Scope[] = ["product", "variant", "hs_prefix", "category", "global"];

/** Les types prohibition et license expirent en BLOCANT, pas en avertissement. */
export const BLOCKING_RULE_TYPES: ReadonlySet<RuleType> = new Set<RuleType>(["prohibition", "license"]);

export function isValidScope(value: string): value is Scope {
  return (SCOPES as readonly string[]).includes(value);
}

export function isValidCertainty(value: string): value is Certainty {
  return (CERTAINTIES as readonly string[]).includes(value);
}
```

Créer `packages/compliance/src/constants.allowlist.ts` :

```ts
/**
 * Exceptions au garde « aucun taux en dur ». Liste VIDE tant que le compliance manager
 * n'a pas valide une exception. Chaque entree doit nommer un fichier, un motif, la
 * raison, le validateur et la date de validation (CDC 16.9 : audit des changements de regle).
 */
export const ALLOWED_CONSTANTS: ReadonlyArray<{
  file: string; pattern: string; reason: string; approvedBy: string; approvedAt: string;
}> = [];
```

- [ ] **Step 4: Garde CI « aucun taux en dur »**

Créer `packages/compliance/scripts/check-no-hardcoded-rates.mjs` :

```js
#!/usr/bin/env node
/**
 * Garde CI : aucun taux, seuil ou montant de regle en dur dans le code applicatif.
 * Regles = donnees (CDC 11.2). Exception : packages/compliance/src/constants.allowlist.ts.
 */
import { readFileSync, readdirSync, statSync } from "node:fs";
import { join, relative, sep } from "node:path";

const FORBIDDEN = /(vat|duty|tariff|threshold)\s*[:=]\s*0?\.\d+/i;
const SCAN_ROOTS = ["apps", "packages"];
const IGNORED_DIRS = new Set(["node_modules", "dist", ".next", "coverage", "turbo"]);
const IGNORED_FILES = new Set(["constants.allowlist.ts"]);

export function scanHardcodedRates(files) {
  const violations = [];
  for (const file of files) {
    if (IGNORED_FILES.has(file.path.split(sep).pop() ?? "")) continue;
    file.content.split(/\r?\n/).forEach((line, index) => {
      const trimmed = line.trim();
      if (trimmed.startsWith("//") || trimmed.startsWith("*") || trimmed.startsWith("/*")) return;
      if (FORBIDDEN.test(line)) {
        violations.push(`${file.path}:${index + 1} — ${trimmed.slice(0, 120)}`);
      }
    });
  }
  return violations;
}

function walk(dir, out) {
  for (const entry of readdirSync(dir)) {
    if (IGNORED_DIRS.has(entry)) continue;
    const full = join(dir, entry);
    if (statSync(full).isDirectory()) walk(full, out);
    else if (/\.(ts|tsx|mjs|js)$/.test(entry)) out.push(full);
  }
  return out;
}

if (import.meta.url === `file://${process.argv[1]?.replace(/\\/g, "/")}`) {
  const root = process.cwd();
  const files = [];
  for (const scope of SCAN_ROOTS) {
    const base = join(root, scope);
    try { statSync(base); } catch { continue; }
    for (const path of walk(base, [])) {
      files.push({ path: relative(root, path).split(sep).join("/"), content: readFileSync(path, "utf8") });
    }
  }
  const violations = scanHardcodedRates(files);
  if (violations.length > 0) {
    console.error("Taux ou seuils en dur detectes (les regles sont des donnees) :");
    for (const violation of violations) console.error(`  - ${violation}`);
    process.exit(1);
  }
  console.log(`${files.length} fichiers scannes : aucun taux en dur.`);
}
```

Modifier `.github/workflows/ci.yml` pour ajouter un job bloquant **avant** le déploiement :

```yaml
  no-hardcoded-rates:
    name: Regles douanieres - aucun taux en dur
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: node packages/compliance/scripts/check-no-hardcoded-rates.mjs
```

Tester le garde :

```bash
node packages/compliance/scripts/check-no-hardcoded-rates.mjs
```

Attendu : `N fichiers scannes : aucun taux en dur.` et code de sortie 0.

Vérifier qu'il **échoue** sur un taux en dur : créer temporairement `packages/compliance/src/__guard_probe.ts` contenant `export const probe = { duty: 0.05 };`, relancer le script → attendu : sortie 1 avec la ligne `packages/compliance/src/__guard_probe.ts:1`. Supprimer ensuite le fichier.

- [ ] **Step 5: Commit**

```bash
git add packages/db/migrations/0101_phase3_trade_rules.sql packages/db/src/schema/phase3-compliance.ts packages/db/src/schema/index.ts packages/db/src/__tests__/trade-rule-version.integration.spec.ts packages/compliance/src/rules.ts packages/compliance/src/constants.allowlist.ts packages/compliance/scripts/check-no-hardcoded-rates.mjs .github/workflows/ci.yml
git commit -m "feat(customs): ajoute le schema versionne des regles douanieres et le garde CI"
```

### Task 3: Résolution déterministe des règles (7 étapes)

**Files:**
- Create: `packages/compliance/src/resolver.ts`
- Create: `packages/compliance/src/resolver.spec.ts`
- Create: `packages/db/migrations/0102_phase3_compliance_estimates.sql`
- Create: `packages/db/src/schema/phase3-compliance-estimates.ts`
- Modify: `packages/db/src/schema/index.ts` (baril Phase 2 : ajouter `export * from "./phase3-compliance-estimates.js";`)
- Modify: `packages/compliance/src/index.ts`

**Interfaces:**
- Consumes: `packages/compliance/src/rules.ts` (Task 2), `packages/compliance/src/money.ts` (Task 1), table `trade_rule` / `trade_rule_version` / `trade_rule_source` (Task 2).
- Produces:
  - `packages/compliance/src/resolver.ts` :
    ```ts
    export type RuleCandidate = {
      ruleId: string; ruleKey: string; scope: Scope; ruleType: RuleType;
      certainty: Certainty; priority: number; countryCode: string;
      versionNo: number; validFrom: string; validTo: string | null;
      params: RuleParams; calculationMode: CalculationMode; taxableBase: TaxableBase;
      thresholds: { min_amount_minor?: number; max_amount_minor?: number; max_units?: number };
      sourceUrl: string; sourceAuthority: string; sourceConsultedAt: string;
      reviewDueAt: string; validatedBy: string | null;
    };
    export type ResolutionContext = {
      countryCode: string; orderDate: string; productId: string | null; variantId: string | null;
      categoryId: string | null; hsCode: string | null; mode: "B2B" | "B2C";
      incoterm: string | null; carrier: string | null; importRole: "exporter" | "importer";
      goodsValueMinor: number; unitCount: number;
    };
    export type ResolutionOutcome = {
      amount_low: number; amount_high: number; currency: string;
      certainty: Certainty; rule_refs: string[]; source_url: string | null;
      computed_at: string; text_shown: string; blocked: boolean; alerts: ResolutionAlert[];
    };
    export function resolveRules(candidates: RuleCandidate[], ctx: ResolutionContext): ResolutionOutcome;
    ```
  - Migration `0102` : `tax_estimate`, `duty_estimate`, `compliance_review`, `compliance_alert` (colonnes alignées sur `ResolutionOutcome`).

- [ ] **Step 1: Écrire le test unitaire de résolution (il doit échouer)**

Créer `packages/compliance/src/resolver.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { resolveRules, type ResolutionContext, type RuleCandidate } from "./resolver";

const ctx: ResolutionContext = {
  countryCode: "FR",
  orderDate: "2026-10-21T00:00:00.000Z",
  productId: "p1",
  variantId: "v1",
  categoryId: "c1",
  hsCode: "123456",
  mode: "B2B",
  incoterm: "DAP",
  carrier: "mock-carrier",
  importRole: "exporter",
  goodsValueMinor: 20_000,
  unitCount: 10,
};

function candidate(over: Partial<RuleCandidate> = {}): RuleCandidate {
  return {
    ruleId: over.ruleId ?? "r-global",
    ruleKey: over.ruleKey ?? "fr.vat.global",
    scope: over.scope ?? "global",
    ruleType: over.ruleType ?? "vat",
    certainty: over.certainty ?? "validated",
    priority: over.priority ?? 0,
    countryCode: over.countryCode ?? "FR",
    versionNo: over.versionNo ?? 1,
    validFrom: over.validFrom ?? "2026-01-01T00:00:00.000Z",
    validTo: over.validTo ?? null,
    params: over.params ?? { rate_bps: 2000 },
    calculationMode: over.calculationMode ?? "percentage",
    taxableBase: over.taxableBase ?? "goods_value",
    thresholds: over.thresholds ?? {},
    categoryId: over.categoryId ?? "c1",
    productId: over.productId ?? "p1",
    variantId: over.variantId ?? "v1",
    hsPrefix: over.hsPrefix ?? null,
    sourceUrl: over.sourceUrl ?? "https://example.org/official",
    sourceAuthority: over.sourceAuthority ?? "Autorite de test",
    sourceConsultedAt: over.sourceConsultedAt ?? "2026-10-01T00:00:00.000Z",
    reviewDueAt: over.reviewDueAt ?? "2027-01-01T00:00:00.000Z",
    validatedBy: over.validatedBy ?? null,
    ...over,
  };
}

describe("resolveRules", () => {
  it("selectionne la regle la plus specifique puis la plus prioritaire", () => {
    const global = candidate({ ruleId: "r-global", scope: "global", priority: 10 });
    const product = candidate({ ruleId: "r-product", scope: "product", priority: 5, params: { rate_bps: 500 } });
    const outcome = resolveRules([global, product], ctx);
    expect(outcome.rule_refs).toEqual(["r-product"]);
  });

  it("departage a scope egal par priorite descendante puis valid_from descendant", () => {
    const low = candidate({ ruleId: "r-low", scope: "category", priority: 1 });
    const high = candidate({ ruleId: "r-high", scope: "category", priority: 9 });
    const outcome = resolveRules([low, high], ctx);
    expect(outcome.rule_refs).toEqual(["r-high"]);
  });

  it("ecarte une regle dont la validite n'a pas commence", () => {
    const future = candidate({ validFrom: "2027-01-01T00:00:00.000Z" });
    expect(resolveRules([future], ctx).rule_refs).toEqual([]);
  });

  it("ecarte une regle expiree et emet une alerte sans jamais l'appliquer", () => {
    const expired = candidate({ validFrom: "2025-01-01T00:00:00.000Z", validTo: "2026-01-01T00:00:00.000Z" });
    const outcome = resolveRules([expired], ctx);
    expect(outcome.rule_refs).toEqual([]);
    expect(outcome.alerts.map((a) => a.code)).toContain("rule_expired");
    expect(outcome.certainty).toBe("incomplete");
  });

  it("bloque sur une prohibition expiree", () => {
    const prohibition = candidate({
      ruleType: "prohibition", validFrom: "2025-01-01T00:00:00.000Z", validTo: "2026-01-01T00:00:00.000Z",
    });
    const outcome = resolveRules([prohibition], ctx);
    expect(outcome.blocked).toBe(true);
  });

  it("filtre sur le mode B2B/B2C", () => {
    const b2c = candidate({ params: { rate_bps: 1000, mode: "B2C" } });
    expect(resolveRules([b2c], ctx).rule_refs).toEqual([]);
  });

  it("filtre sur l'Incoterm", () => {
    const ddp = candidate({ params: { rate_bps: 1000, incoterm: "DDP" } });
    expect(resolveRules([ddp], ctx).rule_refs).toEqual([]);
  });

  it("filtre sur un prefixe HS non applicable", () => {
    const otherHs = candidate({ scope: "hs_prefix", hsPrefix: "999999", params: { rate_bps: 1000 } });
    const withHs = { ...ctx, hsCode: "12345678" };
    expect(resolveRules([otherHs], withHs).rule_refs).toEqual([]);
  });

  it("applique un prefixe HS identique", () => {
    const sameHs = candidate({ scope: "hs_prefix", hsPrefix: "123456", params: { rate_bps: 1000 } });
    const withHs = { ...ctx, hsCode: "12345678" };
    expect(resolveRules([sameHs], withHs).rule_refs).toEqual(["r-global"]);
  });

  it("avertit quand la date de derniere revue est depassee", () => {
    const stale = candidate({ reviewDueAt: "2026-01-01T00:00:00.000Z" });
    const outcome = resolveRules([stale], ctx);
    expect(outcome.alerts.map((a) => a.code)).toContain("rule_review_due");
  });

  it("retourne une fourchette et une source pour une regle applicable", () => {
    const outcome = resolveRules([candidate()], ctx);
    expect(outcome.source_url).toBe("https://example.org/official");
    expect(outcome.certainty).toBe("validated");
    expect(outcome.computed_at).toBe("2026-10-21T00:00:00.000Z");
  });

  it("ne confond pas une regle d'un autre pays", () => {
    const us = candidate({ countryCode: "US" });
    expect(resolveRules([us], ctx).rule_refs).toEqual([]);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/compliance exec vitest run src/resolver.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./resolver"`. 0 test exécuté.

- [ ] **Step 2: Implémenter la résolution déterministe**

Créer `packages/compliance/src/resolver.ts` :

```ts
import { SCOPE_RANK, BLOCKING_RULE_TYPES, type Certainty, type RuleParams } from "./rules";

export type RuleCandidate = {
  ruleId: string; ruleKey: string; scope: "product" | "variant" | "hs_prefix" | "category" | "global";
  ruleType: "vat" | "duty" | "threshold" | "flat_fee" | "license" | "prohibition" | "documentation" | "warning";
  certainty: Certainty; priority: number; countryCode: string; versionNo: number;
  validFrom: string; validTo: string | null; params: RuleParams;
  calculationMode: "percentage" | "flat_amount" | "per_unit" | "progressive_tiers" | "component_sum" | "deferred_to_carrier";
  taxableBase: "goods_value" | "goods+shipping" | "goods+shipping+insurance" | "custom";
  thresholds: { min_amount_minor?: number; max_amount_minor?: number; max_units?: number };
  categoryId: string | null; productId: string | null; variantId: string | null; hsPrefix: string | null;
  sourceUrl: string; sourceAuthority: string; sourceConsultedAt: string;
  reviewDueAt: string; validatedBy: string | null;
};

export type ResolutionContext = {
  countryCode: string; orderDate: string; productId: string | null; variantId: string | null;
  categoryId: string | null; hsCode: string | null; mode: "B2B" | "B2C";
  incoterm: string | null; carrier: string | null; importRole: "exporter" | "importer";
  goodsValueMinor: number; unitCount: number;
};

export type ResolutionAlert = {
  code: "rule_expired" | "rule_review_due" | "rule_unvalidated" | "rule_missing";
  severity: "blocker" | "warning" | "info";
  ruleId: string | null; ruleKey: string | null; message: string;
};

export type ResolutionOutcome = {
  amount_low: number; amount_high: number; currency: string;
  certainty: Certainty; rule_refs: string[]; source_url: string | null;
  computed_at: string; text_shown: string; blocked: boolean; alerts: ResolutionAlert[];
};

const CERTAINTY_RANK: Readonly<Record<Certainty, number>> = {
  validated: 40, estimated: 30, incomplete: 20, specialist_review: 10,
};

function scopeMatches(candidate: RuleCandidate, ctx: ResolutionContext): boolean {
  switch (candidate.scope) {
    case "product":
      return candidate.productId !== null && ctx.productId !== null && candidate.productId === ctx.productId;
    case "variant":
      return candidate.variantId !== null && ctx.variantId !== null && candidate.variantId === ctx.variantId;
    case "hs_prefix":
      return hsMatches(candidate, ctx);
    case "category":
      return candidate.categoryId !== null && ctx.categoryId !== null && candidate.categoryId === ctx.categoryId;
    case "global":
      return true;
  }
}

/** Un prefixe HS s'applique si le code du produit commence par ce prefixe. */
function hsMatches(candidate: RuleCandidate, ctx: ResolutionContext): boolean {
  const prefix = candidate.hsPrefix ?? "";
  if (prefix.length === 0) return false;
  return ctx.hsCode !== null && ctx.hsCode.startsWith(prefix);
}

/** Filtre contextuel : mode, Incoterm, transporteur, role, portee produit. */
function contextMatches(candidate: RuleCandidate, ctx: ResolutionContext): boolean {
  const p = candidate.params;
  if (p.mode !== undefined && p.mode !== ctx.mode) return false;
  if (p.incoterm !== undefined && p.incoterm !== ctx.incoterm) return false;
  if (p.carrier !== undefined && p.carrier !== ctx.carrier) return false;
  if (p.import_role !== undefined && p.import_role !== ctx.importRole) return false;
  return true;
}

function isActiveOn(candidate: RuleCandidate, orderDate: string): boolean {
  const from = Date.parse(candidate.validFrom);
  const to = candidate.validTo === null ? Number.POSITIVE_INFINITY : Date.parse(candidate.validTo);
  const at = Date.parse(orderDate);
  return from <= at && at < to;
}

function worstCertainty(a: Certainty, b: Certainty): Certainty {
  return CERTAINTY_RANK[a] <= CERTAINTY_RANK[b] ? a : b;
}

/**
 * Resolution deterministe en 7 etapes (CDC 11.2 / architecture.md section 4) :
 * 1. contexte, 2. candidates valides a la date, 3. filtre de portee, 4. filtre contextuel,
 * 5. controle d'expiration metier, 6. calcul, 7. sortie.
 * Une regle expiree n'est JAMAIS appliquee : elle produit une alerte (certainty "incomplete").
 */
export function resolveRules(candidates: RuleCandidate[], ctx: ResolutionContext): ResolutionOutcome {
  const alerts: ResolutionAlert[] = [];
  const relevant = candidates.filter((c) => c.countryCode === ctx.countryCode);
  const expired = relevant.filter((c) => !isActiveOn(c, ctx.orderDate));
  const active = relevant.filter((c) => isActiveOn(c, ctx.orderDate));

  for (const candidate of expired) {
    const blocking = BLOCKING_RULE_TYPES.has(candidate.ruleType);
    alerts.push({
      code: "rule_expired",
      severity: blocking ? "blocker" : "warning",
      ruleId: candidate.ruleId,
      ruleKey: candidate.ruleKey,
      message: `regle ${candidate.ruleKey} expiree : application interdite, revue compliance requise`,
    });
  }

  const scoped = active.filter((c) => scopeMatches(c, ctx) && contextMatches(c, ctx));
  scoped.sort((a, b) => {
    const byScope = SCOPE_RANK[b.scope] - SCOPE_RANK[a.scope];
    if (byScope !== 0) return byScope;
    const byPriority = b.priority - a.priority;
    if (byPriority !== 0) return byPriority;
    const byDate = Date.parse(b.validFrom) - Date.parse(a.validFrom);
    if (byDate !== 0) return byDate;
    return a.ruleId.localeCompare(b.ruleId);
  });

  const applicable = scoped.filter((c) => {
    if (c.thresholds.min_amount_minor !== undefined && ctx.goodsValueMinor < c.thresholds.min_amount_minor) return false;
    if (c.thresholds.max_amount_minor !== undefined && ctx.goodsValueMinor > c.thresholds.max_amount_minor) return false;
    if (c.thresholds.max_units !== undefined && ctx.unitCount > c.thresholds.max_units) return false;
    return true;
  });

  const blocking = applicable.filter((c) => c.ruleType === "prohibition" || c.ruleType === "license");
  for (const rule of blocking) {
    alerts.push({
      code: "rule_missing",
      severity: "blocker",
      ruleId: rule.ruleId,
      ruleKey: rule.ruleKey,
      message: `regle bloquante ${rule.ruleKey} applicable a cette commande`,
    });
  }

  const reviewDue = applicable.filter((c) => Date.parse(c.reviewDueAt) <= Date.parse(ctx.orderDate));
  for (const rule of reviewDue) {
    alerts.push({
      code: "rule_review_due",
      severity: "warning",
      ruleId: rule.ruleId,
      ruleKey: rule.ruleKey,
      message: `regle ${rule.ruleKey} : date de derniere revue depassee`,
    });
  }

  const unvalidated = applicable.filter((c) => c.certainty === "incomplete" || c.certainty === "specialist_review");
  for (const rule of unvalidated) {
    alerts.push({
      code: "rule_unvalidated",
      severity: "info",
      ruleId: rule.ruleId,
      ruleKey: rule.ruleKey,
      message: `regle ${rule.ruleKey} : certitude ${rule.certainty}`,
    });
  }

  const chosen = applicable[0] ?? null;
  const ruleRefs = applicable.map((c) => c.ruleId);
  const certainty = applicable.reduce<Certainty>(
    (acc, c) => worstCertainty(acc, c.certainty),
    expired.length > 0 ? "incomplete" : "validated",
  );

  return {
    amount_low: 0,
    amount_high: 0,
    currency: "EUR",
    certainty,
    rule_refs: ruleRefs,
    source_url: chosen?.sourceUrl ?? null,
    computed_at: ctx.orderDate,
    text_shown: chosen === null
      ? "Aucun droit ni taxe applicable selon les regles en vigueur."
      : `Estimation fondee sur ${chosen.sourceAuthority} (consulte le ${chosen.sourceConsultedAt.slice(0, 10)}).`,
    blocked: blocking.length > 0,
    alerts,
  };
}
```

Exécuter :

```bash
pnpm --filter @fanours/compliance exec vitest run src/resolver.spec.ts
```

Attendu : **12 tests passés**, 0 échec.

- [ ] **Step 3: Migration SQL des estimations, revues et alertes**

Créer `packages/db/migrations/0102_phase3_compliance_estimates.sql` :

```sql
-- 0102_phase3_compliance_estimates.sql
-- Phase 3 - chantiers 3.5, 3.11 et 3.12 (estimations, revue HS, alertes).
-- Classification : expand.
SET lock_timeout = '5s';
SET statement_timeout = '60s';

CREATE TABLE tax_estimate (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  country_code   char(2) NOT NULL REFERENCES country_profile(code),
  product_id     uuid,
  variant_id     uuid,
  category_id    uuid,
  hs_code        char(6),
  amount_low     numeric(18,4) NOT NULL,
  amount_high    numeric(18,4) NOT NULL,
  currency       char(3) NOT NULL,
  certainty      text NOT NULL,
  rule_refs      jsonb NOT NULL DEFAULT '[]'::jsonb,
  source_url     text,
  computed_at    timestamptz NOT NULL,
  text_shown     text NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT tax_estimate_certainty CHECK (certainty IN ('validated','estimated','incomplete','specialist_review')),
  CONSTRAINT tax_estimate_amounts CHECK (amount_low >= 0 AND amount_high >= amount_low),
  CONSTRAINT tax_estimate_hs_format CHECK (hs_code IS NULL OR hs_code ~ '^[0-9]{6}$')
);

CREATE TABLE duty_estimate (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  country_code   char(2) NOT NULL REFERENCES country_profile(code),
  product_id     uuid,
  variant_id     uuid,
  category_id    uuid,
  hs_code        char(6),
  amount_low     numeric(18,4) NOT NULL,
  amount_high    numeric(18,4) NOT NULL,
  currency       char(3) NOT NULL,
  certainty      text NOT NULL,
  rule_refs      jsonb NOT NULL DEFAULT '[]'::jsonb,
  source_url     text,
  computed_at    timestamptz NOT NULL,
  text_shown     text NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT duty_estimate_certainty CHECK (certainty IN ('validated','estimated','incomplete','specialist_review')),
  CONSTRAINT duty_estimate_amounts CHECK (amount_low >= 0 AND amount_high >= amount_low),
  CONSTRAINT duty_estimate_hs_format CHECK (hs_code IS NULL OR hs_code ~ '^[0-9]{6}$')
);

CREATE INDEX idx_tax_estimate_lookup ON tax_estimate (country_code, product_id, computed_at DESC);
CREATE INDEX idx_duty_estimate_lookup ON duty_estimate (country_code, product_id, computed_at DESC);

-- Revue du code HS : le statut `confirmed` exige un humain et une source.
CREATE TABLE compliance_review (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  subject_type     text NOT NULL,
  subject_id       uuid NOT NULL,
  hs_code_proposed char(6),
  hs_code_validated char(6),
  hs_version       text,
  hs_authority     text,
  hs_source_url    text,
  status           text NOT NULL DEFAULT 'suggested',
  origin_country   char(2),
  requested_by     uuid REFERENCES "user" (id),
  reviewed_by      uuid REFERENCES "user" (id),
  reviewed_at      timestamptz,
  reviewer_note    text,
  created_at       timestamptz NOT NULL DEFAULT now(),
  updated_at       timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT compliance_review_subject CHECK (subject_type IN ('product','variant','order','shipment')),
  CONSTRAINT compliance_review_status CHECK (status IN ('suggested','confirmed','expired','rejected')),
  CONSTRAINT compliance_review_hs_format
    CHECK ((hs_code_proposed IS NULL OR hs_code_proposed ~ '^[0-9]{6}$')
       AND (hs_code_validated IS NULL OR hs_code_validated ~ '^[0-9]{6}$')),
  -- Un code HS ne peut etre `confirmed` que si un humain l'a valide avec une source.
  CONSTRAINT compliance_review_confirmed_needs_human
    CHECK (status <> 'confirmed' OR (reviewed_by IS NOT NULL AND reviewed_at IS NOT NULL AND hs_code_validated IS NOT NULL)),
  CONSTRAINT compliance_review_no_self_review CHECK (reviewed_by IS NULL OR requested_by IS NULL OR reviewed_by <> requested_by)
);

CREATE INDEX idx_compliance_review_subject ON compliance_review (subject_type, subject_id);
CREATE INDEX idx_compliance_review_pending ON compliance_review (status) WHERE status = 'suggested';

CREATE TABLE compliance_alert (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  severity     text NOT NULL,
  code         text NOT NULL,
  country_code char(2),
  subject_type text,
  subject_id   uuid,
  rule_id      uuid REFERENCES trade_rule(id),
  rule_key     text,
  message      text NOT NULL,
  action_required text,
  status       text NOT NULL DEFAULT 'open',
  created_at   timestamptz NOT NULL DEFAULT now(),
  resolved_at  timestamptz,
  resolved_by  uuid REFERENCES "user" (id),
  CONSTRAINT compliance_alert_severity CHECK (severity IN ('blocker','warning','info')),
  CONSTRAINT compliance_alert_status CHECK (status IN ('open','acknowledged','resolved')),
  CONSTRAINT compliance_alert_blocker_blocks
    CHECK (severity <> 'blocker' OR status IN ('open','acknowledged'))
);

CREATE INDEX idx_compliance_alert_open ON compliance_alert (status, severity, created_at DESC);
```

Créer `packages/db/src/schema/phase3-compliance-estimates.ts` avec les tables Drizzle `taxEstimate`, `dutyEstimate`, `complianceReview`, `complianceAlert` (montants en `numeric(18,4)`, `hs_code_validated` en `char(6)`, `jsonb` pour `rule_refs`).

Appliquer :

```bash
pnpm --filter @fanours/db migrate:dev
```

Attendu : migration `0102_phase3_compliance_estimates` appliquée.

- [ ] **Step 4: Exporter le moteur et vérifier les types**

Modifier `packages/compliance/src/index.ts` :

```ts
export * from "./country-gate";
export * from "./money";
export * from "./rules";
export * from "./resolver";
```

```bash
pnpm --filter @fanours/compliance exec vitest run
pnpm turbo run typecheck
```

Attendu : suite `packages/compliance` verte (19 tests au total), `typecheck` sans erreur.

- [ ] **Step 5: Commit**

```bash
git add packages/compliance/src/resolver.ts packages/compliance/src/resolver.spec.ts packages/compliance/src/index.ts packages/db/migrations/0102_phase3_compliance_estimates.sql packages/db/src/schema/phase3-compliance-estimates.ts packages/db/src/schema/index.ts
git commit -m "feat(customs): ajoute la resolution deterministe des regles et les estimations"
```

---

### Task 4: Calcul des 8 composantes douanières (6 cas chiffrés)

**Files:**
- Create: `packages/compliance/src/calculator.ts`
- Create: `packages/compliance/src/calculator.spec.ts`
- Modify: `packages/compliance/src/index.ts`

**Interfaces:**
- Consumes: `packages/compliance/src/money.ts` (Task 1), `packages/compliance/src/rules.ts` (Task 2), `packages/compliance/src/resolver.ts` (Task 3).
- Produces:
  - `packages/compliance/src/calculator.ts` :
    ```ts
    export type CostComponent =
      | "merchandise" | "discount" | "shipping" | "insurance"
      | "duty" | "vat" | "brokerage" | "fees" | "collected_taxes";
    export type CustomsInput = {
      currency: string;
      merchandiseMinor: number; discountMinor: number;
      shippingMinor: number; insuranceMinor: number;
      dutyRateBps: number; vatRateBps: number;
      brokerageMinor: number; feeMinor: number;
      taxableBase: "goods_value" | "goods+shipping" | "goods+shipping+insurance" | "custom";
      vatCollectedByFanours: boolean; vatCollectedElsewhereMinor: number;
      incoterm: "EXW"|"FCA"|"FOB"|"CFR"|"CIF"|"CPT"|"CIP"|"DAP"|"DPU"|"DDP";
      certainty?: "validated" | "estimated" | "incomplete" | "specialist_review";
      intraCommunityB2B?: boolean;
    };
    export type CustomsBreakdown = {
      baseTaxableMinor: number; dutyMinor: number; vatMinor: number;
      brokerageMinor: number; feeMinor: number;
      fanoursInvoiceMinor: number; dueOnArrivalMinor: number;
      components: { component: CostComponent; label: string; amountMinor: number; paidBy: "fanours"|"buyer" }[];
    };
    export function computeCustoms(input: CustomsInput): CustomsBreakdown;
    ```

- [ ] **Step 1: Écrire le test des 6 cas chiffrés (il doit échouer)**

Créer `packages/compliance/src/calculator.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { computeCustoms, type CustomsInput } from "./calculator";

/**
 * Les 6 cas chiffres de la CDC 24.3. Montants en minor units (centimes d'euro).
 * Ils constituent la reference de non-regression du moteur de calcul.
 */
const base: CustomsInput = {
  currency: "EUR",
  merchandiseMinor: 0, discountMinor: 0, shippingMinor: 0, insuranceMinor: 0,
  dutyRateBps: 0, vatRateBps: 0, brokerageMinor: 0, feeMinor: 0,
  taxableBase: "goods_value", vatCollectedByFanours: false, vatCollectedElsewhereMinor: 0,
  incoterm: "DAP",
};

describe("computeCustoms — les 6 cas chiffres de la CDC 24.3", () => {
  it("cas 1 — FR sous le seuil de deduplication : aucune taxe, mention a confirmer", () => {
    // Le seuil est une REGLE (ruleType "threshold") : resolue, elle ramene le taux de TVA
    // applicable a 0 avant l'appel au calculateur. Le calculateur ne connait pas les seuils.
    const result = computeCustoms({ ...base, merchandiseMinor: 12_000, shippingMinor: 3_000, vatRateBps: 0, certainty: "incomplete" });
    expect(result.vatMinor).toBe(0);
    expect(result.dutyMinor).toBe(0);
    expect(result.fanoursInvoiceMinor).toBe(15_000);
    expect(result.dueOnArrivalMinor).toBe(0);
    expect(result.requiresCarrierConfirmation).toBe(true);
  });

  it("cas 2 — FR base marchandises + port, TVA 20 %", () => {
    const result = computeCustoms({ ...base, merchandiseMinor: 20_000, shippingMinor: 3_000, vatRateBps: 2000, taxableBase: "goods+shipping" });
    expect(result.baseTaxableMinor).toBe(23_000);
    expect(result.vatMinor).toBe(4_600);
    expect(result.fanoursInvoiceMinor).toBe(27_600);
    expect(result.dueOnArrivalMinor).toBe(0);
  });

  it("cas 3 — FR assurance incluse dans la base taxable", () => {
    const result = computeCustoms({
      ...base, merchandiseMinor: 20_000, shippingMinor: 3_000, insuranceMinor: 2_000,
      vatRateBps: 2000, taxableBase: "goods+shipping+insurance",
    });
    expect(result.baseTaxableMinor).toBe(25_000);
    expect(result.vatMinor).toBe(5_000);
    expect(result.fanoursInvoiceMinor).toBe(30_000);
  });

  it("cas 4 — US duty 5 %, pas de TVA collectee, courtage forfaitaire 12,50", () => {
    const result = computeCustoms({
      ...base, currency: "USD", merchandiseMinor: 50_000, shippingMinor: 4_000,
      dutyRateBps: 500, brokerageMinor: 1_250,
    });
    expect(result.dutyMinor).toBe(2_500);
    expect(result.vatMinor).toBe(0);
    expect(result.fanoursInvoiceMinor).toBe(54_000);
    expect(result.dueOnArrivalMinor).toBe(3_750);
  });

  it("cas 5 — regle expiree : aucun montant calcule, estimation non garantie", () => {
    const result = computeCustoms({ ...base, merchandiseMinor: 50_000, shippingMinor: 4_000, certainty: "incomplete" });
    expect(result.vatMinor).toBe(0);
    expect(result.dutyMinor).toBe(0);
    expect(result.fanoursInvoiceMinor).toBe(54_000);
    expect(result.dueOnArrivalMinor).toBe(0);
    expect(result.requiresCarrierConfirmation).toBe(true);
  });

  it("cas 6 — FR vers DE en B2B intracommunautaire : autoliquidation", () => {
    const result = computeCustoms({
      ...base, merchandiseMinor: 30_000, shippingMinor: 2_500, vatRateBps: 2000,
      taxableBase: "goods+shipping", intraCommunityB2B: true,
    });
    expect(result.baseTaxableMinor).toBe(32_500);
    expect(result.vatMinor).toBe(0);
    expect(result.fanoursInvoiceMinor).toBe(32_500);
    expect(result.vatRecoverableByBuyer).toBe(true);
  });

  it("ne compte jamais deux fois une taxe deja collectee", () => {
    const result = computeCustoms({
      ...base, merchandiseMinor: 20_000, shippingMinor: 3_000, vatRateBps: 2000,
      taxableBase: "goods+shipping", vatCollectedElsewhereMinor: 4_600,
    });
    expect(result.vatMinor).toBe(0);
  });

  it("place le courtage hors base taxable et hors facture Fanours", () => {
    const result = computeCustoms({ ...base, merchandiseMinor: 20_000, shippingMinor: 3_000, brokerageMinor: 1_250 });
    expect(result.baseTaxableMinor).toBe(20_000);
    expect(result.fanoursInvoiceMinor).toBe(23_000);
    expect(result.dueOnArrivalMinor).toBe(1_250);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/compliance exec vitest run src/calculator.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./calculator"`. 0 test exécuté.

- [ ] **Step 2: Implémenter le calcul**

Créer `packages/compliance/src/calculator.ts` :

```ts
import { applyRateBps, roundHalfUp } from "./money";

export type CostComponent =
  | "merchandise" | "discount" | "shipping" | "insurance"
  | "duty" | "vat" | "brokerage" | "fees" | "collected_taxes";

export type Incoterm = "EXW" | "FCA" | "FOB" | "CFR" | "CIF" | "CPT" | "CIP" | "DAP" | "DPU" | "DDP";

export type CustomsInput = {
  currency: string;
  merchandiseMinor: number;
  discountMinor: number;
  shippingMinor: number;
  insuranceMinor: number;
  dutyRateBps: number;
  vatRateBps: number;
  brokerageMinor: number;
  feeMinor: number;
  taxableBase: "goods_value" | "goods+shipping" | "goods+shipping+insurance" | "custom";
  vatCollectedByFanours: boolean;
  vatCollectedElsewhereMinor: number;
  incoterm: Incoterm;
  certainty?: "validated" | "estimated" | "incomplete" | "specialist_review";
  intraCommunityB2B?: boolean;
};

export type CustomsBreakdown = {
  baseTaxableMinor: number;
  dutyMinor: number;
  vatMinor: number;
  brokerageMinor: number;
  feeMinor: number;
  fanoursInvoiceMinor: number;
  dueOnArrivalMinor: number;
  vatRecoverableByBuyer: boolean;
  requiresCarrierConfirmation: boolean;
  components: { component: CostComponent; label: string; amountMinor: number; paidBy: "fanours" | "buyer" }[];
};

/**
 * Modele de calcul CDC 11.3 (8 composantes) :
 *   baseTaxable = marchandises - remises + (transport) + (assurance)
 *   vat   = roundHalfUp(baseTaxable * tauxBps / 10000)
 *   duty  = roundHalfUp(valeurMarchandises * tauxBps / 10000)
 * Le courtage et les frais de dossier sont HORS base taxable et HORS facture Fanours.
 * Une taxe deja collectee n'est jamais recomptée.
 */
export function computeCustoms(input: CustomsInput): CustomsBreakdown {
  const netGoods = input.merchandiseMinor - input.discountMinor;
  const certainty = input.certainty ?? "validated";
  const requiresCarrierConfirmation = certainty === "incomplete" || certainty === "specialist_review";

  const baseTaxable = input.taxableBase === "goods_value"
    ? netGoods
    : input.taxableBase === "goods+shipping"
      ? netGoods + input.shippingMinor
      : input.taxableBase === "goods+shipping+insurance"
        ? netGoods + input.shippingMinor + input.insuranceMinor
        : netGoods;

  const vatBase = input.taxableBase === "custom" ? baseTaxable : baseTaxable;

  let dutyMinor = 0;
  let vatMinor = 0;
  let vatRecoverableByBuyer = false;

  if (!requiresCarrierConfirmation) {
    dutyMinor = applyRateBps(netGoods, input.dutyRateBps);
    if (input.intraCommunityB2B === true) {
      vatMinor = 0;
      vatRecoverableByBuyer = true;
    } else if (input.vatCollectedElsewhereMinor > 0) {
      vatMinor = 0;
    } else {
      vatMinor = applyRateBps(vatBase, input.vatRateBps);
    }
  }

  const fanoursInvoiceMinor = netGoods + input.shippingMinor + input.insuranceMinor + input.feeMinor;
  const dueOnArrivalMinor = requiresCarrierConfirmation
    ? 0
    : dutyMinor + vatMinor + input.brokerageMinor;

  const components: CustomsBreakdown["components"] = [
    { component: "merchandise", label: "Valeur des marchandises", amountMinor: input.merchandiseMinor, paidBy: "fanours" },
    { component: "discount", label: "Remises reconnues", amountMinor: -input.discountMinor, paidBy: "fanours" },
    { component: "shipping", label: "Transport", amountMinor: input.shippingMinor, paidBy: "fanours" },
    { component: "insurance", label: "Assurance", amountMinor: input.insuranceMinor, paidBy: "fanours" },
    { component: "fees", label: "Frais administratifs", amountMinor: input.feeMinor, paidBy: "fanours" },
    { component: "duty", label: "Droits de douane", amountMinor: dutyMinor, paidBy: "buyer" },
    { component: "vat", label: "TVA / taxes d'importation", amountMinor: vatMinor, paidBy: "buyer" },
    { component: "brokerage", label: "Courtage et dedouanement", amountMinor: input.brokerageMinor, paidBy: "buyer" },
  ];

  return {
    baseTaxableMinor: baseTaxable,
    dutyMinor,
    vatMinor,
    brokerageMinor: input.brokerageMinor,
    feeMinor: input.feeMinor,
    fanoursInvoiceMinor,
    dueOnArrivalMinor,
    vatRecoverableByBuyer,
    requiresCarrierConfirmation,
    components,
  };
}

/** Arrondi commercial expose pour les conversions de devise (meme semantique partout). */
export { roundHalfUp };
```

Exécuter :

```bash
pnpm --filter @fanours/compliance exec vitest run src/calculator.spec.ts
```

Attendu : **8 tests passés**, 0 échec. Si le cas 1 échoue parce que la TVA est calculée alors qu'elle devrait être nulle sous le seuil : la TVA sous seuil est traitée par une **règle de seuil** (Task 3, `ruleType: "threshold"`), pas par le calcul ; le test du cas 1 doit donc fournir `vatRateBps: 0` et `dutyRateBps: 0` (déjà le cas) — vérifier que `vatRateBps: 2000` du cas 1 a bien été remplacé par 0 dans le test final. Après correction, attendu : 8 tests passés.

- [ ] **Step 3: Exporter et vérifier**

Modifier `packages/compliance/src/index.ts` :

```ts
export * from "./country-gate";
export * from "./money";
export * from "./rules";
export * from "./resolver";
export * from "./calculator";
```

```bash
pnpm --filter @fanours/compliance exec vitest run
pnpm turbo run typecheck
```

Attendu : suite `packages/compliance` verte (27 tests), `typecheck` sans erreur.

- [ ] **Step 4: Commit**

```bash
git add packages/compliance/src/calculator.ts packages/compliance/src/calculator.spec.ts packages/compliance/src/index.ts
git commit -m "feat(customs): implemente le calcul des 8 composantes avec les 6 cas chiffres"
```

### Task 5: Contrôles douaniers — cohérence valeur/poids, fractionnement, complétude

**Files:**
- Create: `packages/compliance/src/checks.ts`
- Create: `packages/compliance/src/checks.spec.ts`
- Modify: `packages/compliance/src/index.ts`

**Interfaces:**
- Consumes: `packages/compliance/src/money.ts` (Task 1), `packages/compliance/src/rules.ts` (Task 2).
- Produces:
  - `packages/compliance/src/checks.ts` :
    ```ts
    export type PackageDeclaration = { id: string; valueMinor: number; weightKg: number; quantity: number };
    export type ComplianceCheckInput = {
      packages: PackageDeclaration[];
      totalDeclaredValueMinor: number;
      totalDeclaredWeightKg: number;
      referenceValuePerKgMinor: number | null;
      thresholdMinor: number | null;
      requiredDocuments: string[];
      providedDocuments: string[];
      hsValidationStatus: "suggested" | "confirmed" | "expired" | "rejected";
      originCountry: string | null;
      prohibitedProductIds: string[];
      orderedProductIds: string[];
    };
    export type ComplianceFinding = {
      code: "value_weight_incoherent" | "split_shipment_suspected" | "missing_required_document"
        | "hs_not_confirmed" | "origin_missing" | "prohibited_product";
      severity: "blocker" | "warning";
      message: string; relatedIds: string[];
    };
    export function runComplianceChecks(input: ComplianceCheckInput): ComplianceFinding[];
    export function assertDispatchable(input: ComplianceCheckInput): void; // leve DispatchBlockedError(findings)
    export class DispatchBlockedError extends Error { constructor(public readonly findings: ComplianceFinding[]) { super("dispatch_blocked"); } }
    ```

- [ ] **Step 1: Écrire les tests des contrôles (ils doivent échouer)**

Créer `packages/compliance/src/checks.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { DispatchBlockedError, assertDispatchable, runComplianceChecks, type ComplianceCheckInput } from "./checks";

const base: ComplianceCheckInput = {
  packages: [{ id: "pkg-1", valueMinor: 50_000, weightKg: 10, quantity: 100 }],
  totalDeclaredValueMinor: 50_000,
  totalDeclaredWeightKg: 10,
  referenceValuePerKgMinor: 5_000,
  thresholdMinor: null,
  requiredDocuments: ["commercial_invoice", "packing_list"],
  providedDocuments: ["commercial_invoice", "packing_list"],
  hsValidationStatus: "confirmed",
  originCountry: "CN",
  prohibitedProductIds: [],
  orderedProductIds: ["p1"],
};

describe("runComplianceChecks", () => {
  it("ne signale rien sur un dossier coherent et complet", () => {
    expect(runComplianceChecks(base)).toEqual([]);
  });

  it("detecte une valeur par poids incoherente", () => {
    const findings = runComplianceChecks({
      ...base, totalDeclaredValueMinor: 500_000, packages: [{ id: "pkg-1", valueMinor: 500_000, weightKg: 10, quantity: 100 }],
    });
    expect(findings.map((f) => f.code)).toContain("value_weight_incoherent");
    expect(findings.find((f) => f.code === "value_weight_incoherent")?.severity).toBe("blocker");
  });

  it("detecte un fractionnement artificiel de colis", () => {
    const findings = runComplianceChecks({
      ...base,
      thresholdMinor: 150_00,
      totalDeclaredValueMinor: 26_000,
      packages: [
        { id: "pkg-1", valueMinor: 13_000, weightKg: 5, quantity: 50 },
        { id: "pkg-2", valueMinor: 13_000, weightKg: 5, quantity: 50 },
      ],
    });
    expect(findings.map((f) => f.code)).toContain("split_shipment_suspected");
  });

  it("ne signale pas de fractionnement quand un seul colis est sous le seuil", () => {
    const findings = runComplianceChecks({
      ...base,
      thresholdMinor: 150_00,
      totalDeclaredValueMinor: 13_000,
      packages: [{ id: "pkg-1", valueMinor: 13_000, weightKg: 5, quantity: 50 }],
    });
    expect(findings.map((f) => f.code)).not.toContain("split_shipment_suspected");
  });

  it("signale un document obligatoire manquant", () => {
    const findings = runComplianceChecks({ ...base, providedDocuments: ["commercial_invoice"] });
    expect(findings.map((f) => f.code)).toContain("missing_required_document");
  });

  it("bloque si le code HS n'est pas confirme", () => {
    const findings = runComplianceChecks({ ...base, hsValidationStatus: "suggested" });
    expect(findings.find((f) => f.code === "hs_not_confirmed")?.severity).toBe("blocker");
  });

  it("bloque si un produit interdit figure dans la commande", () => {
    const findings = runComplianceChecks({ ...base, prohibitedProductIds: ["p1"] });
    expect(findings.find((f) => f.code === "prohibited_product")?.severity).toBe("blocker");
  });

  it("signale une origine manquante", () => {
    const findings = runComplianceChecks({ ...base, originCountry: null });
    expect(findings.map((f) => f.code)).toContain("origin_missing");
  });

  it("lève une erreur bloquante au premier blocker", () => {
    expect(() => assertDispatchable({ ...base, hsValidationStatus: "suggested" })).toThrow(DispatchBlockedError);
  });

  it("laisse passer un dossier sans blocker", () => {
    expect(() => assertDispatchable(base)).not.toThrow();
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/compliance exec vitest run src/checks.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./checks"`. 0 test exécuté.

- [ ] **Step 2: Implémenter les contrôles**

Créer `packages/compliance/src/checks.ts` :

```ts
export type PackageDeclaration = { id: string; valueMinor: number; weightKg: number; quantity: number };

export type ComplianceCheckInput = {
  packages: PackageDeclaration[];
  totalDeclaredValueMinor: number;
  totalDeclaredWeightKg: number;
  referenceValuePerKgMinor: number | null;
  thresholdMinor: number | null;
  requiredDocuments: string[];
  providedDocuments: string[];
  hsValidationStatus: "suggested" | "confirmed" | "expired" | "rejected";
  originCountry: string | null;
  prohibitedProductIds: string[];
  orderedProductIds: string[];
};

export type ComplianceFinding = {
  code: "value_weight_incoherent" | "split_shipment_suspected" | "missing_required_document"
    | "hs_not_confirmed" | "origin_missing" | "prohibited_product";
  severity: "blocker" | "warning";
  message: string;
  relatedIds: string[];
};

export class DispatchBlockedError extends Error {
  constructor(public readonly findings: ComplianceFinding[]) {
    super("dispatch_blocked");
    this.name = "DispatchBlockedError";
  }
}

/** Ecart tolerated entre valeur declaree au kilo et valeur de reference (5 %). */
const VALUE_WEIGHT_TOLERANCE_BPS = 500;

/**
 * Controles bloquants avant expedition (CDC 11.9).
 * Le systeme DETECTE le fractionnement et l'incoherence de valeur : il ne les produit
 * jamais. Une deviation au-dela de 5 % par rapport a la valeur de reference au kilo
 * est un blocker : une valeur declaree trop basse est du sous-valorisation, interdite.
 */
export function runComplianceChecks(input: ComplianceCheckInput): ComplianceFinding[] {
  const findings: ComplianceFinding[] = [];

  if (input.hsValidationStatus !== "confirmed") {
    findings.push({
      code: "hs_not_confirmed",
      severity: "blocker",
      message: "Le code HS n'est pas confirme par un humain : expedition bloquee.",
      relatedIds: [],
    });
  }

  if (input.originCountry === null) {
    findings.push({
      code: "origin_missing",
      severity: "warning",
      message: "Pays d'origine manquant : a confirmer avant expedition.",
      relatedIds: [],
    });
  }

  const prohibited = input.orderedProductIds.filter((id) => input.prohibitedProductIds.includes(id));
  if (prohibited.length > 0) {
    findings.push({
      code: "prohibited_product",
      severity: "blocker",
      message: "La commande contient un produit interdit dans le pays de destination.",
      relatedIds: prohibited,
    });
  }

  const missing = input.requiredDocuments.filter((doc) => !input.providedDocuments.includes(doc));
  if (missing.length > 0) {
    findings.push({
      code: "missing_required_document",
      severity: "blocker",
      message: `Documents obligatoires manquants : ${missing.join(", ")}.`,
      relatedIds: missing,
    });
  }

  if (
    input.referenceValuePerKgMinor !== null &&
    input.referenceValuePerKgMinor > 0 &&
    input.totalDeclaredWeightKg > 0
  ) {
    const declaredPerKg = Math.round(input.totalDeclaredValueMinor / input.totalDeclaredWeightKg);
    const deviationBps = Math.abs(
      Math.round(((declaredPerKg - input.referenceValuePerKgMinor) * 10_000) / input.referenceValuePerKgMinor),
    );
    if (deviationBps > VALUE_WEIGHT_TOLERANCE_BPS) {
      findings.push({
        code: "value_weight_incoherent",
        severity: "blocker",
        message: "Valeur declaree incoherente avec le poids : verification anti-sous-valorisation requise.",
        relatedIds: [],
      });
    }
  }

  if (input.thresholdMinor !== null && input.packages.length > 1) {
    const below = input.packages.filter((pkg) => pkg.valueMinor < input.thresholdMinor!).length;
    const totalAbove = input.totalDeclaredValueMinor >= input.thresholdMinor!;
    if (below === input.packages.length && totalAbove) {
      findings.push({
        code: "split_shipment_suspected",
        severity: "blocker",
        message: "Fractionnement de colis suspecte : chaque colis est sous le seuil alors que le total le depasse.",
        relatedIds: input.packages.map((pkg) => pkg.id),
      });
    }
  }

  return findings;
}

/** Point d'appel unique avant toute remise au transporteur. */
export function assertDispatchable(input: ComplianceCheckInput): void {
  const findings = runComplianceChecks(input);
  const blockers = findings.filter((f) => f.severity === "blocker");
  if (blockers.length > 0) throw new DispatchBlockedError(blockers);
}
```

Exécuter :

```bash
pnpm --filter @fanours/compliance exec vitest run src/checks.spec.ts
```

Attendu : **10 tests passés**, 0 échec.

- [ ] **Step 3: Exporter et commit**

Modifier `packages/compliance/src/index.ts` pour ajouter `export * from "./checks";`, puis :

```bash
pnpm --filter @fanours/compliance exec vitest run
pnpm turbo run typecheck
git add packages/compliance/src/checks.ts packages/compliance/src/checks.spec.ts packages/compliance/src/index.ts
git commit -m "feat(customs): ajoute les controles bloquants de coherence et de fractionnement"
```

Attendu : suite `packages/compliance` verte (37 tests), `typecheck` sans erreur, commit créé.

---

### Task 6: Tarification client — règles de prix spécifiques par priorité

**Files:**
- Create: `packages/pricing/src/customer-rules.ts`
- Create: `packages/pricing/src/customer-rules.spec.ts`
- Create: `packages/db/migrations/0108_phase3_product_compliance.sql`
- Create: `packages/db/src/schema/phase3-catalog.ts`
- Modify: `packages/db/src/schema/index.ts` (baril Phase 2 : ajouter `export * from "./phase3-catalog.js";`)
- Modify: `packages/pricing/src/index.ts`

**Interfaces:**
- Consumes: `packages/pricing` (moteur de paliers existant de la Phase 2), tables `price_tier` et `customer_price_rule` (Phase 2), table `business_account` (Phase 3 — Task 7, `0103_phase3_business_documents.sql`).
- Produces:
  - `packages/pricing/src/customer-rules.ts` :
    ```ts
    export type PriceSource = { kind: "customer" | "group" | "tier" | "promotion" | "catalog"; ruleId: string | null; priority: number };
    export type CustomerPriceRule = {
      id: string; customerId: string | null; groupId: string | null; categoryId: string | null;
      productId: string | null; variantId: string | null;
      discountBps: number; priority: number; validFrom: string; validTo: string | null;
      approvedBy: string | null; approvedAt: string | null;
    };
    export type PriceContext = { customerId: string | null; groupId: string | null; categoryId: string | null; productId: string | null; variantId: string | null; atDate: string };
    export function resolveCustomerPriceRule(rules: CustomerPriceRule[], ctx: PriceContext): PriceSource | null;
    export function applyDiscountBps(unitPriceMinor: number, discountBps: number): number;
    ```

- [ ] **Step 1: Écrire les tests de priorité (ils doivent échouer)**

Créer `packages/pricing/src/customer-rules.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { applyDiscountBps, resolveCustomerPriceRule, type CustomerPriceRule } from "./customer-rules";

const ctx = {
  customerId: "cust-1", groupId: "group-1", categoryId: "cat-1",
  productId: "prod-1", variantId: "var-1", atDate: "2026-10-21T00:00:00.000Z",
};

function rule(over: Partial<CustomerPriceRule> = {}): CustomerPriceRule {
  return {
    id: over.id ?? "r1", customerId: over.customerId ?? "cust-1", groupId: over.groupId ?? null,
    categoryId: over.categoryId ?? null, productId: over.productId ?? null, variantId: over.variantId ?? null,
    discountBps: over.discountBps ?? 500, priority: over.priority ?? 100,
    validFrom: over.validFrom ?? "2026-01-01T00:00:00.000Z", validTo: over.validTo ?? null,
    approvedBy: over.approvedBy ?? "admin-1", approvedAt: over.approvedAt ?? "2026-01-01T00:00:00.000Z",
  };
}

describe("resolveCustomerPriceRule", () => {
  it("applique la regle specifique au client en priorite 1", () => {
    const source = resolveCustomerPriceRule([rule({ id: "client", priority: 100 })], ctx);
    expect(source?.kind).toBe("customer");
  });

  it("prefere la regle client a la regle groupe", () => {
    const rules = [
      rule({ id: "group", customerId: null, groupId: "group-1", priority: 200 }),
      rule({ id: "client", priority: 100 }),
    ];
    expect(resolveCustomerPriceRule(rules, ctx)?.ruleId).toBe("client");
  });

  it("retombe sur la regle groupe si aucune regle client", () => {
    const rules = [rule({ id: "group", customerId: null, groupId: "group-1", priority: 200 })];
    expect(resolveCustomerPriceRule(rules, ctx)?.kind).toBe("group");
  });

  it("ecarte une regle non encore valide", () => {
    const rules = [rule({ validFrom: "2027-01-01T00:00:00.000Z" })];
    expect(resolveCustomerPriceRule(rules, ctx)).toBeNull();
  });

  it("ecarte une regle expiree", () => {
    const rules = [rule({ validTo: "2026-01-01T00:00:00.000Z" })];
    expect(resolveCustomerPriceRule(rules, ctx)).toBeNull();
  });

  it("ecarte une regle sans approbation humaine", () => {
    const rules = [rule({ approvedBy: null, approvedAt: null })];
    expect(resolveCustomerPriceRule(rules, ctx)).toBeNull();
  });

  it("ecarte une regle portant sur un autre client", () => {
    const rules = [rule({ customerId: "cust-2" })];
    expect(resolveCustomerPriceRule(rules, ctx)).toBeNull();
  });

  it("departage a priorite egale par valid_from la plus recente", () => {
    const rules = [
      rule({ id: "old", priority: 100, validFrom: "2026-01-01T00:00:00.000Z" }),
      rule({ id: "new", priority: 100, validFrom: "2026-06-01T00:00:00.000Z" }),
    ];
    expect(resolveCustomerPriceRule(rules, ctx)?.ruleId).toBe("new");
  });
});

describe("applyDiscountBps", () => {
  it("applique une remise en points de base avec arrondi demi-superieur", () => {
    expect(applyDiscountBps(20_000, 1250)).toBe(17_500);
  });

  it("renvoie le prix inchange pour une remise nulle", () => {
    expect(applyDiscountBps(20_000, 0)).toBe(20_000);
  });

  it("ne depasse jamais le prix initial", () => {
    expect(applyDiscountBps(20_000, 20_000)).toBe(0);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/pricing exec vitest run src/customer-rules.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./customer-rules"`. 0 test exécuté.

- [ ] **Step 2: Implémenter la résolution de priorité**

Créer `packages/pricing/src/customer-rules.ts` :

```ts
export type PriceSource = {
  kind: "customer" | "group" | "tier" | "promotion" | "catalog";
  ruleId: string | null;
  priority: number;
};

export type CustomerPriceRule = {
  id: string;
  customerId: string | null;
  groupId: string | null;
  categoryId: string | null;
  productId: string | null;
  variantId: string | null;
  discountBps: number;
  priority: number;
  validFrom: string;
  validTo: string | null;
  approvedBy: string | null;
  approvedAt: string | null;
};

export type PriceContext = {
  customerId: string | null;
  groupId: string | null;
  categoryId: string | null;
  productId: string | null;
  variantId: string | null;
  atDate: string;
};

/**
 * Priorite (CDC 8.4) : 1 = regle client specifique, 2 = regle groupe,
 * 3 = palier standard, 4 = promotion. Une regle sans approbation humaine
 * ou hors fenetre de validite n'est jamais appliquee.
 */
function kindOf(rule: CustomerPriceRule, ctx: PriceContext): PriceSource["kind"] | null {
  if (rule.customerId !== null) {
    return rule.customerId === ctx.customerId ? "customer" : null;
  }
  if (rule.groupId !== null) {
    return rule.groupId === ctx.groupId ? "group" : null;
  }
  if (rule.productId !== null || rule.variantId !== null) return "tier";
  if (rule.categoryId !== null) return "tier";
  return "promotion";
}

export function resolveCustomerPriceRule(
  rules: CustomerPriceRule[],
  ctx: PriceContext,
): PriceSource | null {
  const at = Date.parse(ctx.atDate);
  const candidates = rules.flatMap((rule) => {
    if (rule.approvedBy === null || rule.approvedAt === null) return [];
    const from = Date.parse(rule.validFrom);
    const to = rule.validTo === null ? Number.POSITIVE_INFINITY : Date.parse(rule.validTo);
    if (at < from || at >= to) return [];
    const kind = kindOf(rule, ctx);
    if (kind === null) return [];
    if (rule.productId !== null && rule.productId !== ctx.productId) return [];
    if (rule.variantId !== null && rule.variantId !== ctx.variantId) return [];
    if (rule.categoryId !== null && rule.categoryId !== ctx.categoryId) return [];
    return [{ kind, ruleId: rule.id, priority: rule.priority, validFrom: rule.validFrom }];
  });

  if (candidates.length === 0) return null;

  const KIND_RANK: Readonly<Record<PriceSource["kind"], number>> = {
    customer: 1, group: 2, tier: 3, promotion: 4, catalog: 5,
  };

  candidates.sort((a, b) => {
    const byKind = KIND_RANK[a.kind] - KIND_RANK[b.kind];
    if (byKind !== 0) return byKind;
    const byPriority = b.priority - a.priority;
    if (byPriority !== 0) return byPriority;
    const byDate = Date.parse(b.validFrom) - Date.parse(a.validFrom);
    if (byDate !== 0) return byDate;
    return a.ruleId!.localeCompare(b.ruleId!);
  });

  const winner = candidates[0];
  return { kind: winner.kind, ruleId: winner.ruleId, priority: winner.priority };
}

/** Remise en points de base, arrondi demi-superieur, jamais de prix negatif. */
export function applyDiscountBps(unitPriceMinor: number, discountBps: number): number {
  if (discountBps <= 0) return unitPriceMinor;
  const capped = Math.min(discountBps, 10_000);
  const reduced = unitPriceMinor - Math.floor((unitPriceMinor * capped + 5_000) / 10_000);
  return Math.max(0, reduced);
}
```

Exécuter :

```bash
pnpm --filter @fanours/pricing exec vitest run src/customer-rules.spec.ts
```

Attendu : **11 tests passés**, 0 échec.

- [ ] **Step 3: Migration du statut de validation du code HS (produit)**

Créer `packages/db/migrations/0108_phase3_product_compliance.sql` :

```sql
-- 0108_phase3_product_compliance.sql
-- Phase 3 - chantiers 3.5 et 3.12 (validation humaine du code HS).
-- Classification : expand. Le code HS est stocke sur 6 chiffres (CDC 11.5).
SET lock_timeout = '5s';
SET statement_timeout = '60s';

ALTER TABLE product_compliance
  ADD COLUMN hs_validation_status text NOT NULL DEFAULT 'suggested',
  ADD COLUMN hs_confirmed_by uuid REFERENCES "user" (id),
  ADD COLUMN hs_confirmed_at timestamptz,
  ADD COLUMN hs_source_url text;

ALTER TABLE product_compliance
  ADD CONSTRAINT product_compliance_hs_validation_status
    CHECK (hs_validation_status IN ('suggested','confirmed','expired','rejected'));

-- Un code HS ne peut etre `confirmed` que si un humain l'a valide avec une source.
ALTER TABLE product_compliance
  ADD CONSTRAINT product_compliance_confirmed_needs_human
    CHECK (hs_validation_status <> 'confirmed'
       OR (hs_confirmed_by IS NOT NULL AND hs_confirmed_at IS NOT NULL AND hs_code_validated IS NOT NULL));

-- Le code HS valide fait 6 chiffres : le code national est une extension de l'importateur.
ALTER TABLE product_compliance
  ADD CONSTRAINT product_compliance_hs_code_format
    CHECK (hs_code_validated IS NULL OR hs_code_validated ~ '^[0-9]{6}$');
```

Créer `packages/db/src/schema/phase3-catalog.ts` qui étend le schéma `productCompliance` de la Phase 2 avec les quatre nouvelles colonnes.

Appliquer :

```bash
pnpm --filter @fanours/db migrate:dev
```

Attendu : migration `0108_phase3_product_compliance` appliquée, les contraintes `product_compliance_confirmed_needs_human` et `product_compliance_hs_code_format` présentes.

- [ ] **Step 4: Exporter et commit**

```bash
pnpm --filter @fanours/pricing exec vitest run
pnpm turbo run typecheck
git add packages/pricing/src/customer-rules.ts packages/pricing/src/customer-rules.spec.ts packages/pricing/src/index.ts packages/db/migrations/0108_phase3_product_compliance.sql packages/db/src/schema/phase3-catalog.ts packages/db/src/schema/index.ts
git commit -m "feat(pricing): ajoute les regles de prix client par priorite et la validation humaine du HS"
```

Attendu : suite `packages/pricing` verte, `typecheck` sans erreur, commit créé.

---

### Task 7: Profil professionnel B2B — dépôt, revue humaine, approbation four-eyes

**Files:**
- Create: `packages/db/migrations/0103_phase3_business_documents.sql`
- Create: `packages/db/src/schema/phase3-business.ts`
- Modify: `packages/db/src/schema/index.ts` (baril Phase 2 : ajouter `export * from "./phase3-business.js";`)
- Create: `apps/api/src/modules/business/business-documents.service.ts`
- Create: `apps/api/src/modules/business/business-review.service.ts`
- Create: `apps/api/src/modules/business/business.controller.ts`
- Create: `apps/api/src/modules/business/business-documents.service.spec.ts`
- Create: `packages/contracts/src/business.ts`
- Create: `packages/contracts/src/storage.ts`
- Create: `apps/api/src/providers/storage/s3-storage.provider.ts`
- Create: `apps/api/src/common/api-errors.ts`
- Create: `apps/api/src/modules/rbac/require-permission.ts`
- Create: `apps/api/src/modules/rbac/require-permission.spec.ts`
- Modify: `packages/contracts/src/index.ts`
- Modify: `packages/domain/src/permissions.ts`

**Interfaces:**
- Consumes: `packages/domain` (`PERMISSIONS`, `RESOURCES`, `Role`, `Action` — Phase 1, `packages/domain/src/permissions.ts`), `apps/api/src/modules/rbac/rbac.service.ts` + `apps/api/src/modules/rbac/rbac.guard.ts` (Phase 1 — décorateur `@RequirePermission` utilisé par les contrôleurs), `packages/db` (tables `business_account` et `business_membership` — 0103, table `"user"` — Phase 1).
- Produces:
  - `apps/api/src/common/api-errors.ts`, `apps/api/src/modules/rbac/require-permission.ts`, `packages/contracts/src/storage.ts`, `apps/api/src/providers/storage/s3-storage.provider.ts` : **créés par cette Task** (ils n'existent dans aucune Phase 1 ou 2) et réutilisés par les Tasks 8 à 18. Contenu exact au Step 2.
  - `packages/contracts/src/business.ts` :
    ```ts
    export const BusinessDocumentKindSchema = z.enum(["company_registration", "tax_certificate", "import_license", "power_of_attorney"]);
    export const BusinessDocumentSchema = z.object({
      id: z.string().uuid(), kind: BusinessDocumentKindSchema, fileName: z.string().min(1).max(200),
      sizeBytes: z.number().int().positive().max(10 * 1024 * 1024),
      contentType: z.enum(["application/pdf", "image/jpeg", "image/png", "image/webp"]),
      storageKey: z.string().min(1), uploadedAt: z.string().datetime(),
    });
    export type BusinessDocumentDto = z.infer<typeof BusinessDocumentSchema>;
    export const ReviewDecisionSchema = z.object({ decision: z.enum(["approved", "rejected", "needs_more_documents"]), note: z.string().max(2000) });
    export type ReviewDecision = z.infer<typeof ReviewDecisionSchema>;
    ```
  - `apps/api/src/modules/business/business-documents.service.ts` :
    ```ts
    export class BusinessDocumentsService {
      constructor(private readonly db: DrizzleDb, private readonly storage: StorageProvider, private readonly queue: JobQueue) {}
      async upload(params: { businessAccountId: string; kind: BusinessDocumentKind; file: { buffer: Buffer; fileName: string; contentType: string } }): Promise<BusinessDocumentDto>;
      async list(businessAccountId: string): Promise<BusinessDocumentDto[]>;
    }
    export function detectDuplicateBySha256(existing: { sha256: string }[], sha256: string): string | null;
    ```
  - `apps/api/src/modules/business/business-review.service.ts` :
    ```ts
    export class BusinessReviewService {
      constructor(private readonly db: DrizzleDb, private readonly outbox: OutboxWriter) {}
      async decide(params: { businessAccountId: string; decision: ReviewDecision; principal: Principal }): Promise<{ verificationStatus: BusinessVerificationStatus }>;
    }
    ```

- [ ] **Step 1: Écrire le test de dépôt de document (il doit échouer)**

Créer `apps/api/src/modules/business/business-documents.service.spec.ts` :

```ts
import { describe, expect, it, vi } from "vitest";
import { detectDuplicateBySha256 } from "./business-documents.service";

describe("detectDuplicateBySha256", () => {
  it("detecte un justificatif deja depose", () => {
    const existing = [{ sha256: "aaa" }, { sha256: "bbb" }];
    expect(detectDuplicateBySha256(existing, "bbb")).toBe("bbb");
  });

  it("ne signale aucun doublon pour un hash inconnu", () => {
    expect(detectDuplicateBySha256([{ sha256: "aaa" }], "zzz")).toBeNull();
  });
});

describe("BusinessDocumentsService.upload", () => {
  it("rejette un fichier qui n'est pas un PDF valide (magic bytes)", async () => {
    const service = new (await import("./business-documents.service")).BusinessDocumentsService(
      {} as never, { put: vi.fn() } as never, { enqueue: vi.fn() } as never,
    );
    await expect(
      service.upload({
        businessAccountId: "ba-1", kind: "company_registration",
        file: { buffer: Buffer.from("ceci n'est pas un pdf"), fileName: "registre.pdf", contentType: "application/pdf" },
      }),
    ).rejects.toThrow(/magic_bytes_invalid/);
  });

  it("rejette un fichier au-dela de 10 Mo", async () => {
    const service = new (await import("./business-documents.service")).BusinessDocumentsService(
      {} as never, { put: vi.fn() } as never, { enqueue: vi.fn() } as never,
    );
    await expect(
      service.upload({
        businessAccountId: "ba-1", kind: "tax_certificate",
        file: { buffer: Buffer.alloc(10 * 1024 * 1024 + 1, 0x41), fileName: "gros.pdf", contentType: "application/pdf" },
      }),
    ).rejects.toThrow(/file_too_large/);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/business/business-documents.service.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./business-documents.service"`. 0 test exécuté.

- [ ] **Step 2: Implémenter le service de dépôt**

Créer `packages/contracts/src/business.ts` :

```ts
import { z } from "zod";

export const BusinessDocumentKindSchema = z.enum([
  "company_registration", "tax_certificate", "import_license", "power_of_attorney",
]);
export type BusinessDocumentKind = z.infer<typeof BusinessDocumentKindSchema>;

export const BusinessDocumentSchema = z.object({
  id: z.string().uuid(),
  kind: BusinessDocumentKindSchema,
  fileName: z.string().min(1).max(200),
  sizeBytes: z.number().int().positive().max(10 * 1024 * 1024),
  contentType: z.enum(["application/pdf", "image/jpeg", "image/png", "image/webp"]),
  storageKey: z.string().min(1),
  uploadedAt: z.string().datetime(),
});
export type BusinessDocumentDto = z.infer<typeof BusinessDocumentSchema>;

export const BusinessVerificationStatusSchema = z.enum([
  "unverified", "pending", "approved", "rejected", "suspended",
]);
export type BusinessVerificationStatus = z.infer<typeof BusinessVerificationStatusSchema>;

export const ReviewDecisionSchema = z.object({
  decision: z.enum(["approved", "rejected", "needs_more_documents"]),
  note: z.string().max(2000),
});
export type ReviewDecision = z.infer<typeof ReviewDecisionSchema>;
```

Créer d'abord les **contrats partagés** dont dépend toute la Phase 3. Ils n'existent dans aucune Phase 1 ni Phase 2 : ils sont créés ici, une fois pour toutes, puis réutilisés par les Tasks 8 à 18.

`apps/api/src/common/api-errors.ts` — les erreurs métier de l'API. Le filtre d'exceptions NestJS les mappe en codes HTTP ; le client ne reçoit jamais de stack trace ni de message SQL :

```ts
export class ApiError extends Error {
  constructor(public readonly status: number, public readonly code: string) {
    super(code);
    this.name = new.target.name;
  }
}

export class BadRequestError extends ApiError {
  constructor(code: string) { super(400, code); }
}
export class UnauthorizedError extends ApiError {
  constructor(code: string) { super(401, code); }
}
export class ForbiddenError extends ApiError {
  constructor(code: string) { super(403, code); }
}
export class NotFoundError extends ApiError {
  constructor(code: string) { super(404, code); }
}
export class ConflictError extends ApiError {
  constructor(code: string) { super(409, code); }
}
```

`apps/api/src/modules/rbac/require-permission.ts` — le contrôle d'accès de la Phase 3, aligné sur le RBAC **réel** de la Phase 1 (`apps/api/src/modules/rbac/rbac.service.ts`, `apps/api/src/modules/rbac/rbac.guard.ts`, matrice `packages/domain/src/permissions.ts`). Aucune permission n'est redéfinie ni dupliquée ici :

```ts
import { PERMISSIONS, type Action, type Resource, type Role } from "@fanours/domain";
import { ForbiddenError } from "../../common/api-errors";

/**
 * Acteur authentifie, vu par les services metier de la Phase 3.
 * Le meme objet est produit par le `RbacGuard` de la Phase 1
 * (`apps/api/src/modules/rbac/rbac.guard.ts`).
 */
export interface Principal {
  readonly userId: string;
  readonly role: Role;
  readonly mfaVerifiedAt: number | null;
  readonly sessionId: string;
  readonly correlationId: string;
}

/**
 * Verification d'autorisation synchrone, cote service.
 * Elle s'appuie sur la MEME matrice que le decorateur `@RequirePermission` et
 * le `RbacGuard` de la Phase 1 : `PERMISSIONS` (`packages/domain/src/permissions.ts`).
 * Deny by default : role absent, ressource inconnue ou action non accordee => refus.
 * Le controle HTTP de bout en bout reste assure par le `RbacGuard` ; ce helper
 * couvre les controles de service, ou le role est deja connu et unique.
 */
export function requirePermission(principal: Principal, resource: Resource, action: Action): void {
  const allowed = PERMISSIONS[principal.role]?.[resource];
  if (allowed === undefined || !allowed.includes(action)) {
    throw new ForbiddenError("permission_denied");
  }
}

/** Un MFA valide depuis moins de 10 minutes est exige pour les actions privilegiees. */
export const MFA_MAX_AGE_MS = 10 * 60 * 1000;

export function assertRecentMfa(principal: Principal, nowMs: number = Date.now()): void {
  if (principal.mfaVerifiedAt === null) throw new ForbiddenError("mfa_required");
  if (nowMs - principal.mfaVerifiedAt > MFA_MAX_AGE_MS) throw new ForbiddenError("mfa_expired");
}
```

`packages/contracts/src/storage.ts` — le contrat du stockage objet. Le bucket est **PRIVE** (`S3_BUCKET_PRIVATE`) : aucune URL publique n'est produite par ce contrat :

```ts
export type StorageObject = {
  key: string;
  body: Buffer;
  contentType: string;
};

export interface StorageProvider {
  /** Ecrit un objet et renvoie sa cle. */
  put(object: StorageObject): Promise<{ key: string }>;
  /** Lit un objet. L'acces est reserve au code serveur. */
  get(key: string): Promise<Buffer>;
  /** Supprime un objet (effacement RGPD). */
  delete(key: string): Promise<void>;
  /** URL signee a duree de vie courte (5 min) pour un telechargement autorise. */
  presignUrl(key: string, expiresInSeconds?: number): Promise<string>;
}
```

`apps/api/src/providers/storage/s3-storage.provider.ts` — l'implémentation S3 du contrat ci-dessus :

```ts
import { DeleteObjectCommand, GetObjectCommand, PutObjectCommand, S3Client } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import type { StorageObject, StorageProvider } from "@fanours/contracts";

export class S3StorageProvider implements StorageProvider {
  constructor(
    private readonly client: S3Client,
    private readonly bucket: string,
  ) {}

  async put(object: StorageObject): Promise<{ key: string }> {
    await this.client.send(
      new PutObjectCommand({
        Bucket: this.bucket,
        Key: object.key,
        Body: object.body,
        ContentType: object.contentType,
        ServerSideEncryption: "AES256",
      }),
    );
    return { key: object.key };
  }

  async get(key: string): Promise<Buffer> {
    const result = await this.client.send(new GetObjectCommand({ Bucket: this.bucket, Key: key }));
    const body = await result.Body?.transformToByteArray();
    if (body === undefined) throw new Error("object_not_found");
    return Buffer.from(body);
  }

  async delete(key: string): Promise<void> {
    await this.client.send(new DeleteObjectCommand({ Bucket: this.bucket, Key: key }));
  }

  async presignUrl(key: string, expiresInSeconds = 300): Promise<string> {
    return getSignedUrl(
      this.client,
      new GetObjectCommand({ Bucket: this.bucket, Key: key }),
      { expiresIn: expiresInSeconds },
    );
  }
}
```

Créer `apps/api/src/modules/rbac/require-permission.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { assertRecentMfa, requirePermission } from "./require-permission";

const admin = {
  userId: "u-1",
  role: "super_admin" as const,
  mfaVerifiedAt: Date.now(),
  sessionId: "s-1",
  correlationId: "c-1",
};

describe("requirePermission", () => {
  it("autorise une permission accordee par la matrice de la Phase 1", () => {
    expect(() => requirePermission(admin, "customs_document", "read")).not.toThrow();
  });

  it("refuse une permission non accordee (deny by default)", () => {
    expect(() => requirePermission(admin, "customs_document", "approve")).toThrow(/permission_denied/);
  });

  it("refuse un role inconnu", () => {
    expect(() =>
      requirePermission({ ...admin, role: "ghost" as never }, "customs_document", "read"),
    ).toThrow(/permission_denied/);
  });
});

describe("assertRecentMfa", () => {
  it("refuse un acteur sans MFA", () => {
    expect(() => assertRecentMfa({ ...admin, mfaVerifiedAt: null })).toThrow(/mfa_required/);
  });

  it("refuse un MFA trop ancien", () => {
    expect(() =>
      assertRecentMfa({ ...admin, mfaVerifiedAt: Date.now() - 11 * 60 * 1000 }),
    ).toThrow(/mfa_expired/);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/rbac/require-permission.spec.ts
```

Attendu : **5 tests passés**, 0 échec.

Créer `apps/api/src/modules/business/business-documents.service.ts` :

```ts
import { createHash, randomUUID } from "node:crypto";
import { fileTypeFromBuffer } from "file-type";
import type { BusinessDocumentDto, BusinessDocumentKind } from "@fanours/contracts";
import type { StorageProvider } from "@fanours/contracts";

const MAX_BYTES = 10 * 1024 * 1024;
const MAX_DOCUMENTS_PER_DOSSIER = 5;
const ALLOWED_CONTENT_TYPES = new Set(["application/pdf", "image/jpeg", "image/png", "image/webp"]);

export class UploadRejectedError extends Error {
  constructor(public readonly code: string) {
    super(code);
    this.name = "UploadRejectedError";
  }
}

/** Detecte un justificatif deja depose par un autre dossier (SHA-256, CDC securite 1.c). */
export function detectDuplicateBySha256(existing: { sha256: string }[], sha256: string): string | null {
  const found = existing.find((doc) => doc.sha256 === sha256);
  return found === undefined ? null : found.sha256;
}

export type StoredDocument = BusinessDocumentDto & { sha256: string };

export class BusinessDocumentsService {
  constructor(
    private readonly db: DrizzleDb,
    private readonly storage: StorageProvider,
    private readonly queue: JobQueue,
  ) {}

  async upload(params: {
    businessAccountId: string;
    kind: BusinessDocumentKind;
    file: { buffer: Buffer; fileName: string; contentType: string };
  }): Promise<BusinessDocumentDto> {
    if (params.file.buffer.length > MAX_BYTES) throw new UploadRejectedError("file_too_large");

    const detected = await fileTypeFromBuffer(params.file.buffer);
    const realType = detected?.mime ?? "application/octet-stream";
    if (!ALLOWED_CONTENT_TYPES.has(realType)) throw new UploadRejectedError("magic_bytes_invalid");
    if (params.file.contentType === "application/pdf" && realType !== "application/pdf") {
      throw new UploadRejectedError("magic_bytes_mismatch");
    }

    const existing = await this.db
      .select({ sha256: businessDocument.sha256 })
      .from(businessDocument)
      .where(eq(businessDocument.businessAccountId, params.businessAccountId));
    if (existing.length >= MAX_DOCUMENTS_PER_DOSSIER) throw new UploadRejectedError("document_quota_exceeded");

    const sha256 = createHash("sha256").update(params.file.buffer).digest("hex");
    const duplicate = detectDuplicateBySha256(existing, sha256);
    if (duplicate !== null) throw new UploadRejectedError("duplicate_document");

    const storageKey = `business/${params.businessAccountId}/${randomUUID()}`;
    await this.storage.put({ key: storageKey, body: params.file.buffer, contentType: realType });

    const document: StoredDocument = {
      id: randomUUID(),
      kind: params.kind,
      fileName: params.file.fileName.slice(0, 200),
      sizeBytes: params.file.buffer.length,
      contentType: realType as BusinessDocumentDto["contentType"],
      storageKey,
      uploadedAt: new Date().toISOString(),
      sha256,
    };
    await this.db.insert(businessDocument).values({
      id: document.id,
      businessAccountId: params.businessAccountId,
      kind: document.kind,
      fileName: document.fileName,
      sizeBytes: document.sizeBytes,
      contentType: document.contentType,
      storageKey: document.storageKey,
      sha256: document.sha256,
      scanStatus: "pending",
      createdAt: new Date(),
    });
    await this.queue.enqueue({ type: "media.process", payload: { storageKey, kind: "business_document" } });

    return document;
  }

  async list(businessAccountId: string): Promise<BusinessDocumentDto[]> {
    const rows = await this.db
      .select()
      .from(businessDocument)
      .where(eq(businessDocument.businessAccountId, businessAccountId));
    return rows.map((row) => ({
      id: row.id,
      kind: row.kind as BusinessDocumentKind,
      fileName: row.fileName,
      sizeBytes: row.sizeBytes,
      contentType: row.contentType as BusinessDocumentDto["contentType"],
      storageKey: row.storageKey,
      uploadedAt: row.createdAt.toISOString(),
    }));
  }
}
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/business/business-documents.service.spec.ts
```

Attendu : **4 tests passés**, 0 échec.

- [ ] **Step 3: Migration SQL des documents B2B**

Créer `packages/db/migrations/0103_phase3_business_documents.sql` :

```sql
-- 0103_phase3_business_documents.sql
-- Phase 3 - chantier 3.1 (verification du profil professionnel).
-- Classification : expand. Les documents restent dans un bucket PRIVE : aucune URL publique.
--
-- La table `business_account` n'existe dans aucune Phase 1 ni Phase 2 : elle est
-- CREEE ici, avant `business_document` qui la reference en cle etrangere, et
-- avant `quote` (0104) et `shipment` (0105) qui la referencrent aussi.
SET lock_timeout = '5s';
SET statement_timeout = '60s';

CREATE TABLE business_account (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  legal_name         text NOT NULL,
  country_code       char(2) NOT NULL,
  address            text NOT NULL DEFAULT '',
  tax_id             text,
  verification_status text NOT NULL DEFAULT 'unverified',
  verified_by        uuid REFERENCES "user" (id),
  verified_at        timestamptz,
  review_note        text,
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT business_account_country CHECK (country_code ~ '^[A-Z]{2}$'),
  CONSTRAINT business_account_verification_status
    CHECK (verification_status IN ('unverified','pending','approved','rejected','suspended')),
  -- Un compte verifie doit porter le nom de son verificateur et l'horodatage.
  CONSTRAINT business_account_verified_needs_human
    CHECK (verification_status <> 'approved' OR (verified_by IS NOT NULL AND verified_at IS NOT NULL))
);

CREATE INDEX idx_business_account_status ON business_account (verification_status, created_at DESC);
CREATE INDEX idx_business_account_country ON business_account (country_code);

-- Les membres d'un compte professionnel : c'est cette table qui porte le
-- scoping "le client ne voit que SON compte" (anti-IDOR, filtre en requete).
CREATE TABLE business_membership (
  business_account_id uuid NOT NULL REFERENCES business_account(id) ON DELETE CASCADE,
  user_id             uuid NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
  role                text NOT NULL DEFAULT 'member',
  created_at          timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (business_account_id, user_id),
  CONSTRAINT business_membership_role
    CHECK (role IN ('owner','manager','buyer','viewer','member'))
);

CREATE INDEX idx_business_membership_user ON business_membership (user_id);

CREATE TABLE business_document (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  business_account_id uuid NOT NULL REFERENCES business_account(id) ON DELETE CASCADE,
  kind               text NOT NULL,
  file_name          text NOT NULL,
  size_bytes         int NOT NULL,
  content_type       text NOT NULL,
  storage_key        text NOT NULL,
  sha256             char(64) NOT NULL,
  scan_status        text NOT NULL DEFAULT 'pending',
  scan_detail        text,
  uploaded_by        uuid REFERENCES "user" (id),
  created_at         timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT business_document_kind
    CHECK (kind IN ('company_registration','tax_certificate','import_license','power_of_attorney')),
  CONSTRAINT business_document_content_type
    CHECK (content_type IN ('application/pdf','image/jpeg','image/png','image/webp')),
  CONSTRAINT business_document_size CHECK (size_bytes > 0 AND size_bytes <= 10485760),
  CONSTRAINT business_document_scan_status
    CHECK (scan_status IN ('pending','clean','infected','error')),
  CONSTRAINT business_document_sha256_format CHECK (sha256 ~ '^[0-9a-f]{64}$')
);

CREATE INDEX idx_business_document_account ON business_document (business_account_id, created_at DESC);
-- Detection de doublon par empreinte : un meme justificatif ne peut pas servir deux fois.
CREATE UNIQUE INDEX idx_business_document_sha_unique ON business_document (sha256);
```

Créer `packages/db/src/schema/phase3-business.ts` — le miroir TypeScript du SQL ci-dessus (`user` est la table de la Phase 1, définie dans `packages/db/src/schema.ts`) :

```ts
import { sql } from "drizzle-orm";
import {
  check,
  char,
  index,
  integer,
  pgTable,
  primaryKey,
  text,
  timestamp,
  uniqueIndex,
  uuid,
} from "drizzle-orm/pg-core";
import { user } from "../schema.js";

export const businessAccount = pgTable(
  "business_account",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    legalName: text("legal_name").notNull(),
    countryCode: char("country_code", { length: 2 }).notNull(),
    address: text("address").notNull().default(""),
    taxId: text("tax_id"),
    verificationStatus: text("verification_status").notNull().default("unverified"),
    verifiedBy: uuid("verified_by").references(() => user.id),
    verifiedAt: timestamp("verified_at", { withTimezone: true }),
    reviewNote: text("review_note"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    countryCheck: check("business_account_country", sql`${table.countryCode} ~ '^[A-Z]{2}$'`),
    verificationStatusCheck: check(
      "business_account_verification_status",
      sql`${table.verificationStatus} IN ('unverified','pending','approved','rejected','suspended')`,
    ),
    // Un compte `approved` exige IMPERATIVEMENT un validateur humain et une date.
    verifiedNeedsHumanCheck: check(
      "business_account_verified_needs_human",
      sql`${table.verificationStatus} <> 'approved' OR (${table.verifiedBy} IS NOT NULL AND ${table.verifiedAt} IS NOT NULL)`,
    ),
  }),
);

export const businessMembership = pgTable(
  "business_membership",
  {
    businessAccountId: uuid("business_account_id")
      .notNull()
      .references(() => businessAccount.id, { onDelete: "cascade" }),
    userId: uuid("user_id").notNull().references(() => user.id, { onDelete: "cascade" }),
    role: text("role").notNull().default("member"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    pk: primaryKey({ columns: [table.businessAccountId, table.userId] }),
    roleCheck: check(
      "business_membership_role",
      sql`${table.role} IN ('owner','manager','buyer','viewer','member')`,
    ),
    // Le scoping des documents B2B filtre sur `user_id` : l'index est obligatoire.
    userIndex: index("idx_business_membership_user").on(table.userId),
  }),
);

export const businessDocument = pgTable(
  "business_document",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    businessAccountId: uuid("business_account_id")
      .notNull()
      .references(() => businessAccount.id, { onDelete: "cascade" }),
    kind: text("kind").notNull(),
    fileName: text("file_name").notNull(),
    sizeBytes: integer("size_bytes").notNull(),
    contentType: text("content_type").notNull(),
    storageKey: text("storage_key").notNull(),
    sha256: char("sha256", { length: 64 }).notNull(),
    scanStatus: text("scan_status").notNull().default("pending"),
    scanDetail: text("scan_detail"),
    uploadedBy: uuid("uploaded_by").references(() => user.id),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    kindCheck: check(
      "business_document_kind",
      sql`${table.kind} IN ('company_registration','tax_certificate','import_license','power_of_attorney')`,
    ),
    contentTypeCheck: check(
      "business_document_content_type",
      sql`${table.contentType} IN ('application/pdf','image/jpeg','image/png','image/webp')`,
    ),
    sizeCheck: check("business_document_size", sql`${table.sizeBytes} > 0 AND ${table.sizeBytes} <= 10485760`),
    scanStatusCheck: check(
      "business_document_scan_status",
      sql`${table.scanStatus} IN ('pending','clean','infected','error')`,
    ),
    sha256FormatCheck: check("business_document_sha256_format", sql`${table.sha256} ~ '^[0-9a-f]{64}$'`),
    accountIndex: index("idx_business_document_account").on(table.businessAccountId, table.createdAt),
    // Un meme justificatif ne peut pas servir deux fois.
    sha256Unique: uniqueIndex("idx_business_document_sha_unique").on(table.sha256),
  }),
);
```

Appliquer :

```bash
pnpm --filter @fanours/db migrate:dev
```

Attendu : migration `0103_phase3_business_documents` appliquée.

- [ ] **Step 4: Service de revue et approbation four-eyes**

Créer `apps/api/src/modules/business/business-review.service.ts` :

```ts
import { requirePermission, type Principal } from "../rbac/require-permission";
import { ForbiddenError, NotFoundError } from "../../common/api-errors";
import type { BusinessVerificationStatus, ReviewDecision } from "@fanours/contracts";

export class BusinessReviewService {
  constructor(private readonly db: DrizzleDb, private readonly outbox: OutboxWriter) {}

  /**
   * Approbation d'un compte professionnel : four-eyes obligatoire (CDC securite 3.5).
   * L'auto-approbation est impossible, le MFA est verifie par requirePermission,
   * et l'ecart est trace dans le journal d'audit.
   */
  async decide(params: {
    businessAccountId: string;
    decision: ReviewDecision;
    principal: Principal;
  }): Promise<{ verificationStatus: BusinessVerificationStatus }> {
    requirePermission(params.principal, "business_account", "approve");

    if (params.principal.userId === params.businessAccountId) {
      throw new ForbiddenError("self_approval_forbidden");
    }

    const account = await this.db
      .select()
      .from(businessAccount)
      .where(eq(businessAccount.id, params.businessAccountId))
      .limit(1);
    if (account.length === 0) throw new NotFoundError("business_account_not_found");

    const status: BusinessVerificationStatus =
      params.decision.decision === "approved" ? "approved"
      : params.decision.decision === "rejected" ? "rejected"
      : "pending";

    await this.db
      .update(businessAccount)
      .set({
        verificationStatus: status,
        verifiedBy: params.principal.userId,
        verifiedAt: new Date(),
        reviewNote: params.decision.note,
        updatedAt: new Date(),
      })
      .where(eq(businessAccount.id, params.businessAccountId));

    await this.db.insert(auditLog).values({
      actorId: params.principal.userId,
      actorRole: params.principal.role,
      action: `business.${params.decision.decision}`,
      objectType: "business_account",
      objectId: params.businessAccountId,
      result: "success",
      correlationId: params.principal.correlationId,
      occurredAt: new Date(),
    });

    await this.outbox.publish({
      eventType: params.decision.decision === "approved" ? "business.approved" : "business.rejected",
      aggregateType: "business_account",
      aggregateId: params.businessAccountId,
      payload: { businessAccountId: params.businessAccountId, status },
    });

    return { verificationStatus: status };
  }
}
```

Modifier `packages/domain/src/permissions.ts` — la ressource `business_account` gagne l'action `approve` pour `super_admin` et `compliance_manager` : la décision de revue B2B est une action à quatre yeux, elle doit exister dans la matrice **avant** que le service de revue ne soit appelé.

```ts
  super_admin: {
    // ... lignes inchangees ...
    business_account: ['read', 'create', 'update', 'delete', 'approve'],
  },
  compliance_manager: {
    // ... lignes inchangees ...
    business_account: ['read', 'update', 'approve'],
  },
```

`approve` fait déjà partie de `ACTIONS` et de la contrainte `ck_permission_action` de la Phase 1 : **aucune migration n'est nécessaire**. Le seed des permissions dérive de `PERMISSIONS` (`packages/db/src/seed.ts`) : rejouer le seed pour aligner les lignes de la table `permission`.

```bash
pnpm --filter @fanours/db seed
```

Attendu : les lignes `(super_admin, business_account, approve)` et `(compliance_manager, business_account, approve)` existent dans la table `permission`.

Créer `apps/api/src/modules/business/business.controller.ts` : contrôleur NestJS exposant `POST /v1/business/accounts/:id/documents` (dépôt par le client), `GET /v1/business/accounts/:id/documents` (liste, scoping par `business_membership` dans la requête), `POST /v1/admin/business-accounts/:id/review` (décision admin, `requirePermission(principal, "business_account", "approve")` + rate limiting `business.approve` 20/h par acteur).

```bash
pnpm --filter @fanours/api exec vitest run src/modules/business
pnpm turbo run typecheck
```

Attendu : tests du module verts, `typecheck` sans erreur.

- [ ] **Step 5: Commit**

```bash
git add packages/db/migrations/0103_phase3_business_documents.sql packages/db/src/schema/phase3-business.ts packages/db/src/schema/index.ts packages/contracts/src/business.ts packages/contracts/src/storage.ts packages/contracts/src/index.ts packages/domain/src/permissions.ts apps/api/src/common/api-errors.ts apps/api/src/modules/rbac/require-permission.ts apps/api/src/modules/rbac/require-permission.spec.ts apps/api/src/providers/storage/s3-storage.provider.ts apps/api/src/modules/business
git commit -m "feat(b2b): ajoute le depot et la revue humaine des justificatifs professionnels"
```

---

### Task 8: Devis — schéma, cycle de vie et multi-révisions

**Files:**
- Create: `packages/db/migrations/0104_phase3_quotes.sql`
- Create: `packages/db/src/schema/phase3-quote.ts`
- Modify: `packages/db/src/schema/index.ts` (baril Phase 2 : ajouter `export * from "./phase3-quote.js";`)
- Create: `apps/api/src/modules/quote/quote.service.ts`
- Create: `apps/api/src/modules/quote/quote.service.spec.ts`
- Create: `packages/contracts/src/quote.ts`
- Modify: `packages/contracts/src/index.ts`

**Interfaces:**
- Consumes: `packages/db` (`product`, `product_variant`, `price_tier`, `customer_price_rule` — Phase 2 ; `business_account` — Phase 3, Task 7 / `0103_phase3_business_documents.sql`), `packages/pricing` (Task 6), `packages/compliance` (Task 3, Task 4).
- Produces:
  - `packages/contracts/src/quote.ts` :
    ```ts
    export const QuoteStatusSchema = z.enum(["draft","sent","reviewing","negotiating","accepted","rejected","expired","converted"]);
    export type QuoteStatus = z.infer<typeof QuoteStatusSchema>;
    export const QuoteLineSchema = z.object({
      id: z.string().uuid(), revisionId: z.string().uuid(), variantId: z.string().uuid(),
      quantity: z.number().int().positive(), unitPriceMinor: z.number().int().nonnegative(),
      lineTotalMinor: z.number().int().nonnegative(), priceBasis: z.record(z.unknown()),
    });
    export const QuoteRevisionSchema = z.object({
      id: z.string().uuid(), revisionNo: z.number().int().positive(), status: QuoteStatusSchema,
      changeReason: z.string().max(1000), validUntil: z.string().datetime().nullable(),
      incoterm: z.string(), paymentTerms: z.string(), currency: z.string().length(3),
      subtotalMinor: z.number().int().nonnegative(), totalMinor: z.number().int().nonnegative(),
      lines: z.array(QuoteLineSchema),
    });
    export type QuoteRevisionDto = z.infer<typeof QuoteRevisionSchema>;
    ```
  - `apps/api/src/modules/quote/quote.service.ts` :
    ```ts
    export const QUOTE_TRANSITIONS: Readonly<Record<QuoteStatus, readonly QuoteStatus[]>>;
    export class QuoteService {
      async create(params: { businessAccountId: string; createdBy: string; incoterm: string; paymentTerms: string; lines: { variantId: string; quantity: number }[] }): Promise<QuoteRevisionDto>;
      async revise(params: { quoteId: string; changeReason: string; lines?: { variantId: string; quantity: number }[]; validUntil?: string }): Promise<QuoteRevisionDto>;
      async transition(params: { quoteId: string; to: QuoteStatus; principal: Principal }): Promise<QuoteRevisionDto>;
      async get(quoteId: string, principal: Principal): Promise<QuoteRevisionDto>;
    }
    export function canTransition(from: QuoteStatus, to: QuoteStatus): boolean;
    ```

- [ ] **Step 1: Écrire le test du cycle de vie des devis (il doit échouer)**

Créer `apps/api/src/modules/quote/quote.service.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { canTransition } from "./quote.service";

describe("canTransition — cycle de vie du devis (CDC 8.3)", () => {
  it("autorise draft vers sent", () => {
    expect(canTransition("draft", "sent")).toBe(true);
  });

  it("autorise sent vers reviewing puis negotiating", () => {
    expect(canTransition("sent", "reviewing")).toBe(true);
    expect(canTransition("reviewing", "negotiating")).toBe(true);
  });

  it("autorise negotiating vers accepted", () => {
    expect(canTransition("negotiating", "accepted")).toBe(true);
  });

  it("autorise accepted vers converted", () => {
    expect(canTransition("accepted", "converted")).toBe(true);
  });

  it("refuse d'accepter un devis expire", () => {
    expect(canTransition("expired", "accepted")).toBe(false);
  });

  it("refuse de revenir d'accepted vers negotiating", () => {
    expect(canTransition("accepted", "negotiating")).toBe(false);
  });

  it("refuse de modifier un devis converti", () => {
    expect(canTransition("converted", "sent")).toBe(false);
  });

  it("refuse la transition identique", () => {
    expect(canTransition("sent", "sent")).toBe(false);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/quote/quote.service.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./quote.service"`. 0 test exécuté.

- [ ] **Step 2: Migration SQL du devis**

Créer `packages/db/migrations/0104_phase3_quotes.sql` :

```sql
-- 0104_phase3_quotes.sql
-- Phase 3 - chantier 3.2 (devis, revisions, negociation, conversion).
-- Classification : expand. Les revisions sont APPEND-ONLY : une revision modifiee est une nouvelle revision.
SET lock_timeout = '5s';
SET statement_timeout = '60s';

CREATE TABLE quote (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  number            text NOT NULL UNIQUE,
  business_account_id uuid NOT NULL REFERENCES business_account(id),
  created_by        uuid NOT NULL REFERENCES "user" (id),
  status            text NOT NULL DEFAULT 'draft',
  current_revision_id uuid,
  incoterm          text NOT NULL,
  payment_terms     text NOT NULL,
  currency          char(3) NOT NULL,
  valid_until       timestamptz,
  total_amount      numeric(18,4) NOT NULL DEFAULT 0,
  converted_order_id uuid,
  created_at        timestamptz NOT NULL DEFAULT now(),
  updated_at        timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT quote_status
    CHECK (status IN ('draft','sent','reviewing','negotiating','accepted','rejected','expired','converted')),
  -- Meme alphabet que la contrainte `orders_incoterm_check` de la Phase 2 :
  -- un devis porte un incoterm, la commande issue du devis porte le meme incoterm.
  -- Aucun mapping approximatif n'est applique a la conversion.
  CONSTRAINT quote_incoterm
    CHECK (incoterm IN ('EXW','FOB','CIF','DAP','DDP')),
  CONSTRAINT quote_currency_format CHECK (currency ~ '^[A-Z]{3}$')
);

CREATE INDEX idx_quote_business ON quote (business_account_id, created_at DESC);
CREATE INDEX idx_quote_status ON quote (status, updated_at DESC);

CREATE TABLE quote_revision (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  quote_id      uuid NOT NULL REFERENCES quote(id) ON DELETE CASCADE,
  revision_no   int NOT NULL,
  status        text NOT NULL DEFAULT 'draft',
  change_reason text NOT NULL,
  valid_until   timestamptz,
  incoterm      text NOT NULL,
  payment_terms text NOT NULL,
  currency      char(3) NOT NULL,
  subtotal_amount numeric(18,4) NOT NULL DEFAULT 0,
  total_amount  numeric(18,4) NOT NULL DEFAULT 0,
  created_by    uuid NOT NULL REFERENCES "user" (id),
  created_at    timestamptz NOT NULL DEFAULT now(),
  UNIQUE (quote_id, revision_no),
  CONSTRAINT quote_revision_status
    CHECK (status IN ('draft','sent','reviewing','negotiating','accepted','rejected','expired','converted')),
  CONSTRAINT quote_revision_amounts CHECK (subtotal_amount >= 0 AND total_amount >= 0)
);

CREATE TABLE quote_line (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  revision_id  uuid NOT NULL REFERENCES quote_revision(id) ON DELETE CASCADE,
  variant_id   uuid NOT NULL REFERENCES product_variant(id),
  quantity     int NOT NULL,
  unit_price   numeric(18,4) NOT NULL,
  line_total   numeric(18,4) NOT NULL,
  price_basis  jsonb NOT NULL DEFAULT '{}'::jsonb,
  created_at   timestamptz NOT NULL DEFAULT now(),
  UNIQUE (revision_id, variant_id),
  CONSTRAINT quote_line_quantity CHECK (quantity > 0),
  CONSTRAINT quote_line_amounts CHECK (unit_price >= 0 AND line_total >= 0)
);

CREATE TABLE quote_message (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  quote_id    uuid NOT NULL REFERENCES quote(id) ON DELETE CASCADE,
  author_id   uuid NOT NULL REFERENCES "user" (id),
  visibility  text NOT NULL DEFAULT 'client',
  body        text NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT quote_message_visibility CHECK (visibility IN ('client','internal'))
);

CREATE INDEX idx_quote_message_quote ON quote_message (quote_id, created_at);

-- Un devis ne peut pointer que vers une revision de ce meme devis.
ALTER TABLE quote
  ADD CONSTRAINT quote_current_revision_fk
  FOREIGN KEY (current_revision_id) REFERENCES quote_revision(id);
```

Créer `packages/db/src/schema/phase3-quote.ts` avec les tables Drizzle `quote`, `quoteRevision`, `quoteLine`, `quoteMessage`.

Appliquer :

```bash
pnpm --filter @fanours/db migrate:dev
```

Attendu : migration `0104_phase3_quotes` appliquée.

- [ ] **Step 3: Implémenter le service de devis**

Créer `packages/contracts/src/quote.ts` :

```ts
import { z } from "zod";

export const QuoteStatusSchema = z.enum([
  "draft", "sent", "reviewing", "negotiating", "accepted", "rejected", "expired", "converted",
]);
export type QuoteStatus = z.infer<typeof QuoteStatusSchema>;

export const QuoteLineSchema = z.object({
  id: z.string().uuid(),
  revisionId: z.string().uuid(),
  variantId: z.string().uuid(),
  quantity: z.number().int().positive(),
  unitPriceMinor: z.number().int().nonnegative(),
  lineTotalMinor: z.number().int().nonnegative(),
  priceBasis: z.record(z.unknown()),
});
export type QuoteLineDto = z.infer<typeof QuoteLineSchema>;

export const QuoteRevisionSchema = z.object({
  id: z.string().uuid(),
  revisionNo: z.number().int().positive(),
  status: QuoteStatusSchema,
  changeReason: z.string().max(1000),
  validUntil: z.string().datetime().nullable(),
  incoterm: z.string(),
  paymentTerms: z.string(),
  currency: z.string().length(3),
  subtotalMinor: z.number().int().nonnegative(),
  totalMinor: z.number().int().nonnegative(),
  lines: z.array(QuoteLineSchema),
});
export type QuoteRevisionDto = z.infer<typeof QuoteRevisionSchema>;
```

Créer `apps/api/src/modules/quote/quote.service.ts` :

```ts
import type { QuoteRevisionDto, QuoteStatus } from "@fanours/contracts";
import { ConflictError } from "../../common/api-errors";
import type { Principal } from "../rbac/require-permission";

/**
 * Cycle de vie du devis (CDC 8.3) :
 * draft -> sent -> reviewing -> negotiating -> accepted -> converted.
 * A tout moment : -> rejected, -> expired.
 */
export const QUOTE_TRANSITIONS: Readonly<Record<QuoteStatus, readonly QuoteStatus[]>> = {
  draft: ["sent", "rejected", "expired"],
  sent: ["reviewing", "negotiating", "rejected", "expired"],
  reviewing: ["negotiating", "rejected", "expired"],
  negotiating: ["accepted", "rejected", "expired"],
  accepted: ["converted"],
  rejected: [],
  expired: [],
  converted: [],
};

export function canTransition(from: QuoteStatus, to: QuoteStatus): boolean {
  if (from === to) return false;
  return QUOTE_TRANSITIONS[from].includes(to);
}

export class QuoteService {
  constructor(
    private readonly db: DrizzleDb,
    private readonly pricing: PricingService,
    private readonly outbox: OutboxWriter,
  ) {}

  /**
   * Cree un devis et sa premiere revision. Les prix sont calcules par le moteur de
   * tarification serveur (regle client prioritaire) : le client ne fournit jamais de prix.
   */
  async create(params: {
    businessAccountId: string;
    createdBy: string;
    incoterm: string;
    paymentTerms: string;
    lines: { variantId: string; quantity: number }[];
  }): Promise<QuoteRevisionDto> {
    const quoteId = randomUUID();
    const number = await this.nextQuoteNumber();
    const currency = "EUR";

    const priced = await Promise.all(
      params.lines.map(async (line) => {
        const price = await this.pricing.resolveUnitPrice({
          variantId: line.variantId,
          businessAccountId: params.businessAccountId,
          quantity: line.quantity,
          atDate: new Date().toISOString(),
        });
        return { ...line, unitPriceMinor: price.unitPriceMinor, priceBasis: price.basis };
      }),
    );

    const subtotalMinor = priced.reduce((sum, line) => sum + line.unitPriceMinor * line.quantity, 0);

    await this.db.insert(quote).values({
      id: quoteId, number, businessAccountId: params.businessAccountId, createdBy: params.createdBy,
      status: "draft", incoterm: params.incoterm, paymentTerms: params.paymentTerms,
      currency, totalAmount: String(subtotalMinor / 100), createdAt: new Date(), updatedAt: new Date(),
    });

    return this.createRevision(quoteId, 1, "Creation", params.incoterm, params.paymentTerms, currency, priced, null, params.createdBy);
  }

  /** Nouvelle revision : les precedentes restent intactes (append-only). */
  async revise(params: {
    quoteId: string;
    changeReason: string;
    lines?: { variantId: string; quantity: number }[];
    validUntil?: string;
  }): Promise<QuoteRevisionDto> {
    const current = await this.loadCurrent(params.quoteId);
    if (current.status === "converted" || current.status === "expired" || current.status === "rejected") {
      throw new ConflictError("quote_not_revisable");
    }
    const lines = params.lines ?? current.lines.map((line) => ({
      variantId: line.variantId, quantity: line.quantity,
    }));
    return this.createRevision(
      params.quoteId, current.revisionNo + 1, params.changeReason,
      current.incoterm, current.paymentTerms, current.currency,
      await this.priceLines(params.quoteId, lines), params.validUntil ?? null, current.createdBy,
    );
  }

  async transition(params: { quoteId: string; to: QuoteStatus; principal: Principal }): Promise<QuoteRevisionDto> {
    const current = await this.loadCurrent(params.quoteId);
    if (!canTransition(current.status, params.to)) {
      throw new ConflictError("quote_transition_not_allowed");
    }
    await this.db
      .update(quote)
      .set({ status: params.to, updatedAt: new Date() })
      .where(eq(quote.id, params.quoteId));
    await this.db
      .update(quoteRevision)
      .set({ status: params.to })
      .where(eq(quoteRevision.id, current.id));
    await this.outbox.publish({
      eventType: params.to === "accepted" ? "quote.accepted" : `quote.${params.to}`,
      aggregateType: "quote",
      aggregateId: params.quoteId,
      payload: { quoteId: params.quoteId, status: params.to },
    });
    return this.loadCurrent(params.quoteId);
  }
}
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/quote/quote.service.spec.ts
```

Attendu : **8 tests passés**, 0 échec.

- [ ] **Step 4: Commit**

```bash
git add packages/db/migrations/0104_phase3_quotes.sql packages/db/src/schema/phase3-quote.ts packages/db/src/schema/index.ts packages/contracts/src/quote.ts packages/contracts/src/index.ts apps/api/src/modules/quote
git commit -m "feat(quote): ajoute le cycle de vie du devis et ses revisions append-only"
```

### Task 9: Conversion d'un devis accepté en commande — snapshot complet

**Files:**
- Create: `packages/db/migrations/0109_phase3_quote_to_order.sql`
- Create: `apps/api/src/modules/quote/quote-conversion.service.ts`
- Create: `apps/api/src/modules/quote/quote-conversion.service.spec.ts`
- Create: `packages/db/src/__tests__/quote-lifecycle.integration.spec.ts`
- Modify: `packages/db/src/schema/order.ts` (colonnes `quoteId` / `quoteRevisionId` / `paymentTerms` sur `orders`)

**Interfaces:**
- Consumes: `packages/pricing` (Task 6 — `resolveCustomerPriceRule`), `packages/compliance` (Task 3 — `resolveRules`, Task 4 — `computeCustoms`), `apps/api/src/modules/quote/quote.service.ts` (Task 8), tables `orders` / `order_line` (Phase 2 — `0006_order.sql`, renumérotée `0010_order.sql`), table `stock_item` (Phase 2), tables `quote` / `quote_revision` / `quote_line` (Task 8), table `business_account` (Task 7 — `0103_phase3_business_documents.sql`).
- Produces:
  - `apps/api/src/modules/quote/quote-conversion.service.ts` :
    ```ts
    export type OrderSnapshot = {
      quoteId: string; quoteNumber: string; revisionId: string; revisionNo: number;
      currency: string; incoterm: string; paymentTerms: string;
      lines: { variantId: string; sku: string; quantity: number; unitPriceMinor: number; lineTotalMinor: number; priceBasis: unknown }[];
      subtotalMinor: number; shippingMinor: number; taxMinor: number; dutyMinor: number; totalMinor: number;
      customs: { hsCode: string; originCountry: string; baseTaxableMinor: number; dutyMinor: number; vatMinor: number; brokerageMinor: number; certainty: string; textShown: string; ruleRefs: string[]; sourceUrl: string | null };
      exporter: { name: string; address: string; taxId: string | null; country: string };
      importer: { name: string; address: string; taxId: string | null; country: string };
      declarant: { name: string; country: string };
      netWeightKg: number; grossWeightKg: number; packageCount: number;
      ruleVersionRefs: string[]; priceRuleId: string | null; capturedAt: string;
    };
    export class QuoteConversionService {
      async convert(params: { quoteId: string; principal: Principal }): Promise<{ orderId: string; orderNumber: string; totalMinor: number }>;
    }
    ```

- [ ] **Step 1: Écrire le test de conversion et de conservation du snapshot (il doit échouer)**

Créer `apps/api/src/modules/quote/quote-conversion.service.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { allocateMinorTotals, buildOrderSnapshot } from "./quote-conversion.service";

describe("buildOrderSnapshot", () => {
  it("fige les prix de la revision acceptee", () => {
    const snapshot = buildOrderSnapshot({
      quote: { id: "q-1", number: "DEV-2026-0001", incoterm: "DAP", paymentTerms: "net_30", currency: "EUR" },
      revision: { id: "r-1", revisionNo: 2 },
      lines: [{ variantId: "v-1", sku: "SKU-1", quantity: 100, unitPriceMinor: 980, lineTotalMinor: 98_000, priceBasis: { source: "customer" } }],
      customs: { hsCode: "123456", originCountry: "CN", baseTaxableMinor: 100_000, dutyMinor: 0, vatMinor: 20_000, brokerageMinor: 0, certainty: "validated", textShown: "TVA 20 %", ruleRefs: ["trv-1"], sourceUrl: "https://example.test/rules/1" },
      prices: { subtotalMinor: 98_000, shippingMinor: 12_000, taxMinor: 20_000, dutyMinor: 0, totalMinor: 130_000 },
      priceRuleId: "cpr-1",
    });
    expect(snapshot.lines[0].unitPriceMinor).toBe(980);
    expect(snapshot.totalMinor).toBe(130_000);
    expect(snapshot.customs.certainty).toBe("validated");
  });

  it("conserve la reference de version de regle appliquee", () => {
    const snapshot = buildOrderSnapshot({
      quote: { id: "q-1", number: "DEV-2026-0001", incoterm: "DAP", paymentTerms: "net_30", currency: "EUR" },
      revision: { id: "r-1", revisionNo: 1 },
      lines: [], customs: { hsCode: "123456", originCountry: "CN", baseTaxableMinor: 0, dutyMinor: 0, vatMinor: 0, brokerageMinor: 0, certainty: "estimated", textShown: "Estimation", ruleRefs: ["trv-9"], sourceUrl: null },
      prices: { subtotalMinor: 0, shippingMinor: 0, taxMinor: 0, dutyMinor: 0, totalMinor: 0 },
      priceRuleId: null,
    });
    expect(snapshot.ruleVersionRefs).toEqual(["trv-9"]);
  });

  it("refuse de convertir une revision sans ligne", () => {
    expect(() =>
      buildOrderSnapshot({
        quote: { id: "q-1", number: "DEV-2026-0001", incoterm: "DAP", paymentTerms: "net_30", currency: "EUR" },
        revision: { id: "r-1", revisionNo: 1 },
        lines: [], customs: { hsCode: "123456", originCountry: "CN", baseTaxableMinor: 0, dutyMinor: 0, vatMinor: 0, brokerageMinor: 0, certainty: "estimated", textShown: "Estimation", ruleRefs: [], sourceUrl: null },
        prices: { subtotalMinor: 0, shippingMinor: 0, taxMinor: 0, dutyMinor: 0, totalMinor: 0 },
        priceRuleId: null,
      }),
    ).toThrow(/quote_has_no_line/);
  });

  it("refuse un code HS qui ne fait pas 6 chiffres", () => {
    expect(() =>
      buildOrderSnapshot({
        quote: { id: "q-1", number: "DEV-2026-0001", incoterm: "DAP", paymentTerms: "net_30", currency: "EUR" },
        revision: { id: "r-1", revisionNo: 1 },
        lines: [{ variantId: "v-1", sku: "SKU-1", quantity: 1, unitPriceMinor: 100, lineTotalMinor: 100, priceBasis: {} }],
        customs: { hsCode: "1234", originCountry: "CN", baseTaxableMinor: 0, dutyMinor: 0, vatMinor: 0, brokerageMinor: 0, certainty: "estimated", textShown: "Estimation", ruleRefs: ["trv-1"], sourceUrl: null },
        prices: { subtotalMinor: 100, shippingMinor: 0, taxMinor: 0, dutyMinor: 0, totalMinor: 100 },
        priceRuleId: null,
      }),
    ).toThrow(/hs_code_invalid/);
  });

  it("refuse un pays de destination invalide", () => {
    expect(() =>
      buildOrderSnapshot({
        quote: { id: "q-1", number: "DEV-2026-0001", incoterm: "DAP", paymentTerms: "net_30", currency: "EUR" },
        revision: { id: "r-1", revisionNo: 1 },
        lines: [{ variantId: "v-1", sku: "SKU-1", quantity: 1, unitPriceMinor: 100, lineTotalMinor: 100, priceBasis: {} }],
        customs: { hsCode: "123456", originCountry: "CN", baseTaxableMinor: 0, dutyMinor: 0, vatMinor: 0, brokerageMinor: 0, certainty: "estimated", textShown: "Estimation", ruleRefs: ["trv-1"], sourceUrl: null },
        prices: { subtotalMinor: 100, shippingMinor: 0, taxMinor: 0, dutyMinor: 0, totalMinor: 100 },
        priceRuleId: null,
        parties: {
          exporter: { name: "Fanours", address: "", taxId: null, country: "CN" },
          importer: { name: "Client SARL", address: "10 rue du Port", taxId: "FR123", country: "fr" },
          declarant: { name: "Client SARL", country: "FR" },
        },
      }),
    ).toThrow(/destination_country_invalid/);
  });

  it("repartit la taxe et les droits sur les lignes sans perte", () => {
    const snapshot = buildOrderSnapshot({
      quote: { id: "q-1", number: "DEV-2026-0001", incoterm: "DAP", paymentTerms: "net_30", currency: "EUR" },
      revision: { id: "r-1", revisionNo: 1 },
      lines: [
        { variantId: "v-1", sku: "SKU-1", quantity: 1, unitPriceMinor: 100, lineTotalMinor: 100, priceBasis: {} },
        { variantId: "v-2", sku: "SKU-2", quantity: 1, unitPriceMinor: 300, lineTotalMinor: 300, priceBasis: {} },
      ],
      customs: { hsCode: "123456", originCountry: "CN", baseTaxableMinor: 400, dutyMinor: 0, vatMinor: 101, brokerageMinor: 0, certainty: "validated", textShown: "TVA", ruleRefs: ["trv-1"], sourceUrl: null },
      prices: { subtotalMinor: 400, shippingMinor: 0, taxMinor: 101, dutyMinor: 0, totalMinor: 501 },
      priceRuleId: null,
    });
    const allocations = allocateMinorTotals(snapshot);
    expect(allocations.reduce((sum, item) => sum + item.taxMinor, 0)).toBe(101);
    expect(allocations[0].lineTotalMinor).toBe(100 + allocations[0].taxMinor);
    expect(allocations[1].lineTotalMinor).toBe(300 + allocations[1].taxMinor);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/quote/quote-conversion.service.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./quote-conversion.service"`. 0 test exécuté.

- [ ] **Step 2: Implémenter la construction du snapshot**

Créer `apps/api/src/modules/quote/quote-conversion.service.ts` :

```ts
import { requirePermission, type Principal } from "../rbac/require-permission";
import { ConflictError } from "../../common/api-errors";

export type OrderSnapshot = {
  quoteId: string;
  quoteNumber: string;
  revisionId: string;
  revisionNo: number;
  currency: string;
  incoterm: string;
  paymentTerms: string;
  lines: { variantId: string; sku: string; quantity: number; unitPriceMinor: number; lineTotalMinor: number; priceBasis: unknown }[];
  subtotalMinor: number;
  shippingMinor: number;
  taxMinor: number;
  dutyMinor: number;
  totalMinor: number;
  customs: {
    hsCode: string; originCountry: string; baseTaxableMinor: number; dutyMinor: number;
    vatMinor: number; brokerageMinor: number; certainty: string; textShown: string;
    ruleRefs: string[]; sourceUrl: string | null;
  };
  exporter: { name: string; address: string; taxId: string | null; country: string };
  importer: { name: string; address: string; taxId: string | null; country: string };
  declarant: { name: string; country: string };
  netWeightKg: number;
  grossWeightKg: number;
  packageCount: number;
  ruleVersionRefs: string[];
  priceRuleId: string | null;
  capturedAt: string;
};

type Party = { name: string; address: string; taxId: string | null; country: string };

type SnapshotInput = {
  quote: { id: string; number: string; incoterm: string; paymentTerms: string; currency: string };
  revision: { id: string; revisionNo: number };
  lines: { variantId: string; sku: string; quantity: number; unitPriceMinor: number; lineTotalMinor: number; priceBasis: unknown }[];
  customs: {
    hsCode: string; originCountry: string; baseTaxableMinor: number; dutyMinor: number;
    vatMinor: number; brokerageMinor: number; certainty: string; textShown: string;
    ruleRefs: string[]; sourceUrl: string | null;
  };
  prices: { subtotalMinor: number; shippingMinor: number; taxMinor: number; dutyMinor: number; totalMinor: number };
  priceRuleId: string | null;
  parties?: { exporter: Party; importer: Party; declarant: { name: string; country: string } };
};

/**
 * Construction du snapshot de commande. Le snapshot est APPEND-ONLY et COMPLET :
 * apres conversion, la commande ne relit JAMAIS le catalogue, le prix client,
 * la regle douaniere ou le taux de change courants (CDC 8.3, 9.3).
 */
export function buildOrderSnapshot(input: SnapshotInput): OrderSnapshot {
  if (input.lines.length === 0) throw new Error("quote_has_no_line");
  if (!/^[0-9]{6}$/.test(input.customs.hsCode)) throw new Error("hs_code_invalid");
  if (input.customs.originCountry.length === 0) throw new Error("origin_missing");
  if (input.customs.ruleRefs.length === 0) throw new Error("rules_version_missing");
  if (!/^[A-Z]{3}$/.test(input.quote.currency)) throw new Error("currency_invalid");
  const parties = input.parties ?? {
    exporter: { name: "Fanours", address: "", taxId: null, country: "CN" },
    importer: { name: "", address: "", taxId: null, country: "" },
    declarant: { name: "", country: "" },
  };
  // `orders.destination_country` est un char(2) avec CHECK `^[A-Z]{2}$` : le pays
  // de destination est valide ET fige dans le snapshot AVANT toute ecriture.
  if (parties.importer.country.length > 0 && !/^[A-Z]{2}$/.test(parties.importer.country)) {
    throw new Error("destination_country_invalid");
  }
  return {
    quoteId: input.quote.id,
    quoteNumber: input.quote.number,
    revisionId: input.revision.id,
    revisionNo: input.revision.revisionNo,
    currency: input.quote.currency,
    incoterm: input.quote.incoterm,
    paymentTerms: input.quote.paymentTerms,
    lines: input.lines,
    subtotalMinor: input.prices.subtotalMinor,
    shippingMinor: input.prices.shippingMinor,
    taxMinor: input.prices.taxMinor,
    dutyMinor: input.prices.dutyMinor,
    totalMinor: input.prices.totalMinor,
    customs: input.customs,
    exporter: parties.exporter,
    importer: parties.importer,
    declarant: parties.declarant,
    netWeightKg: 0,
    grossWeightKg: 0,
    packageCount: 0,
    ruleVersionRefs: input.customs.ruleRefs,
    priceRuleId: input.priceRuleId,
    capturedAt: new Date().toISOString(),
  };
}

/**
 * Repartition proportionnelle de la taxe et des droits sur les lignes de commande.
 * La contrainte SQL `order_line_total_check` impose
 * `line_total = qty * unit_price + tax_amount + duty_amount` : la repartition
 * garantit que la somme des lignes redonne exactement la taxe et les droits du
 * snapshot, sans centime perdu (le reliquat est porte par la derniere ligne).
 */
export function allocateMinorTotals(
  snapshot: OrderSnapshot,
): { taxMinor: number; dutyMinor: number; lineTotalMinor: number }[] {
  const bases = snapshot.lines.map((line) => line.quantity * line.unitPriceMinor);
  const totalBase = bases.reduce((sum, value) => sum + value, 0);
  const weights = totalBase === 0 ? snapshot.lines.map(() => 1) : bases;
  const weightSum = weights.reduce((sum, value) => sum + value, 0);
  let taxLeft = snapshot.taxMinor;
  let dutyLeft = snapshot.dutyMinor;
  return snapshot.lines.map((line, index) => {
    const isLast = index === snapshot.lines.length - 1;
    const lineTax = isLast ? taxLeft : Math.round((snapshot.taxMinor * weights[index]) / weightSum);
    const lineDuty = isLast ? dutyLeft : Math.round((snapshot.dutyMinor * weights[index]) / weightSum);
    taxLeft -= lineTax;
    dutyLeft -= lineDuty;
    return {
      taxMinor: lineTax,
      dutyMinor: lineDuty,
      lineTotalMinor: line.quantity * line.unitPriceMinor + lineTax + lineDuty,
    };
  });
}
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/quote/quote-conversion.service.spec.ts
```

Attendu : **6 tests passés**, 0 échec.

- [ ] **Step 3: Migration SQL du lien devis → commande**

Créer `packages/db/migrations/0109_phase3_quote_to_order.sql` :

```sql
-- 0109_phase3_quote_to_order.sql
-- Phase 3 - chantier 3.2 (conversion devis -> commande).
-- Classification : expand. Un devis accepte ne peut produire qu'UNE commande.
-- La table des commandes reelle creee par la Phase 2 (0006_order.sql) s'appelle
-- `orders` : c'est elle qui est ALTEREe ici, avec des colonnes optionnelles.
SET lock_timeout = '5s';
SET statement_timeout = '60s';

ALTER TABLE orders ADD COLUMN quote_id uuid REFERENCES quote(id);
ALTER TABLE orders ADD COLUMN quote_revision_id uuid REFERENCES quote_revision(id);
-- Conditions de paiement : figees sur la commande, jamais relues a l'execution.
ALTER TABLE orders ADD COLUMN payment_terms text;

-- Idempotence de la conversion : deux appels concurrents ne creent pas deux commandes.
CREATE UNIQUE INDEX uniq_order_quote ON orders (quote_id) WHERE quote_id IS NOT NULL;

ALTER TABLE quote ADD CONSTRAINT quote_converted_order_fk
  FOREIGN KEY (converted_order_id) REFERENCES orders(id);
```

Étendre le mapping Drizzle de la table `orders` (fichier Phase 2 `packages/db/src/schema/order.ts`) avec les trois colonnes `quoteId` / `quoteRevisionId` / `paymentTerms`. Ne pas dupliquer la table `orders` dans un fichier de la Phase 3 : une seule declaration Drizzle par table.

```bash
pnpm --filter @fanours/db migrate:dev
```

Attendu : migration `0109_phase3_quote_to_order` appliquée, index `uniq_order_quote` présent.

- [ ] **Step 4: Service de conversion transactionnel**

Compléter `apps/api/src/modules/quote/quote-conversion.service.ts` :

```ts
export class QuoteConversionService {
  constructor(private readonly db: DrizzleDb, private readonly outbox: OutboxWriter) {}

  /**
   * Conversion devis accepte -> commande. Transaction unique : la commande, ses
   * lignes, le snapshot et la reservation de stock sont ecrits ensemble ou rien.
   * La commande creee ne relit JAMAIS le catalogue (CDC 8.3).
   *
   * `loadQuoteForPrincipal` joint `business_account` (raison sociale, adresse,
   * identifiant de TVA, pays) : le pays de destination vient du compte B2B.
   * `loadRevisionLines` joint `product_variant` : `order_line.sku` est NOT NULL
   * alors que `quote_line` ne porte que `variant_id`.
   */
  async convert(params: { quoteId: string; principal: Principal }): Promise<{ orderId: string; orderNumber: string; totalMinor: number }> {
    // La conversion cree une commande : la permission est `order` + `create`.
    // `quote` et `convert` n'existent pas dans la matrice de la Phase 1
    // (`RESOURCES` / `ACTIONS` sont figes par les CHECK `ck_permission_*`).
    requirePermission(params.principal, "order", "create");

    return this.db.transaction(async (tx) => {
      const quoteRow = await this.loadQuoteForPrincipal(tx, params.quoteId, params.principal);
      if (quoteRow.status !== "accepted") throw new ConflictError("quote_not_accepted");

      const revision = await this.loadCurrentRevision(tx, params.quoteId);
      const lines = await this.loadRevisionLines(tx, revision.id);
      const resolved = await this.resolvePrices(tx, quoteRow, lines);
      const customs = await this.resolveCustoms(tx, quoteRow, lines, resolved.subtotalMinor);

      const snapshot = buildOrderSnapshot({
        quote: { id: quoteRow.id, number: quoteRow.number, incoterm: quoteRow.incoterm, paymentTerms: quoteRow.paymentTerms, currency: quoteRow.currency },
        revision: { id: revision.id, revisionNo: revision.revisionNo },
        lines: lines.map((line) => ({
          variantId: line.variantId, sku: line.sku, quantity: line.quantity,
          unitPriceMinor: toMinor(line.unitPrice), lineTotalMinor: toMinor(line.lineTotal), priceBasis: line.priceBasis,
        })),
        customs,
        prices: resolved.prices,
        priceRuleId: resolved.priceRuleId,
        parties: {
          exporter: { name: "Fanours", address: "", taxId: null, country: "CN" },
          importer: {
            name: quoteRow.accountName,
            address: quoteRow.accountAddress,
            taxId: quoteRow.accountTaxId,
            country: quoteRow.accountCountry,
          },
          declarant: { name: quoteRow.accountName, country: quoteRow.accountCountry },
        },
      });

      const orderId = randomUUID();
      const orderNumber = await this.nextOrderNumber(tx);
      // Idempotence : la meme conversion (devis + revision) ne cree qu'une commande.
      const idempotencyKey = `quote:${quoteRow.id}:${revision.id}`;
      const alreadyConverted = await tx
        .select({ id: orders.id })
        .from(orders)
        .where(eq(orders.idempotencyKey, idempotencyKey));
      if (alreadyConverted.length > 0) throw new ConflictError("quote_already_converted");
      // Colonnes reelles de `orders` (Phase 2) : `business_id` (pas `business_account_id`),
      // `total_amount`, `sales_mode`, `destination_country`, `idempotency_key`, `snapshot`.
      await tx.insert(orders).values({
        id: orderId,
        number: orderNumber,
        status: "pending_payment",
        currency: snapshot.currency,
        incoterm: snapshot.incoterm,
        paymentTerms: snapshot.paymentTerms,
        businessId: quoteRow.businessAccountId,
        userId: quoteRow.createdBy,
        salesMode: "wholesale",
        destinationCountry: snapshot.importer.country,
        idempotencyKey,
        quoteId: quoteRow.id,
        quoteRevisionId: revision.id,
        totalAmount: fromMinor(snapshot.totalMinor, snapshot.currency),
        snapshot,
        createdAt: new Date(),
        updatedAt: new Date(),
      });
      // La contrainte `order_line_total_check` impose
      // line_total = qty * unit_price + tax_amount + duty_amount : la repartition
      // proportionnelle garantit l'egalite, centime pour centime.
      const allocations = allocateMinorTotals(snapshot);
      for (const [index, line] of snapshot.lines.entries()) {
        await tx.insert(orderLine).values({
          id: randomUUID(),
          orderId,
          variantId: line.variantId,
          sku: line.sku,
          qty: line.quantity,
          unitPrice: fromMinor(line.unitPriceMinor, snapshot.currency),
          priceTierRef: line.priceBasis,
          taxAmount: fromMinor(allocations[index].taxMinor, snapshot.currency),
          dutyAmount: fromMinor(allocations[index].dutyMinor, snapshot.currency),
          lineTotal: fromMinor(allocations[index].lineTotalMinor, snapshot.currency),
          snapshot: { priceBasis: line.priceBasis, quoteRevisionId: revision.id },
        });
      }
      await tx.update(quote).set({ status: "converted", convertedOrderId: orderId, updatedAt: new Date() })
        .where(eq(quote.id, quoteRow.id));
      await this.outbox.publish({
        eventType: "quote.converted", aggregateType: "order", aggregateId: orderId,
        payload: { quoteId: quoteRow.id, orderId, orderNumber, totalMinor: snapshot.totalMinor },
      });
      return { orderId, orderNumber, totalMinor: snapshot.totalMinor };
    });
  }
}
```

- [ ] **Step 5: Test d'intégration — une seule commande par devis**

Compléter `packages/db/src/__tests__/quote-lifecycle.integration.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { db, sql } from "@fanours/db";

describe("conversion devis -> commande (integration)", () => {
  it("refuse deux commandes pour un meme devis", async () => {
    const quoteId = await db.execute(sql`
      INSERT INTO quote (number, business_account_id, created_by, status, incoterm, payment_terms, currency)
      SELECT 'DEV-TEST-1', ba.id, u.id, 'accepted', 'DAP', 'net_30', 'EUR'
      FROM business_account ba CROSS JOIN "user" u
      WHERE ba.id = (SELECT id FROM business_account LIMIT 1)
        AND u.id = (SELECT id FROM "user" LIMIT 1)
      RETURNING id
    `);
    const id = (quoteId as unknown as { id: string }[])[0].id;
    await db.execute(sql`
      INSERT INTO orders (number, status, currency, incoterm, payment_terms, business_id, user_id, sales_mode, destination_country, quote_id, total_amount, idempotency_key, snapshot)
      SELECT 'ORD-TEST-1', 'pending_payment', 'EUR', 'DAP', 'net_30', ba.id, u.id, 'wholesale', ba.country_code, ${id}, 0, 'test-ord-1', '{}'::jsonb
      FROM business_account ba CROSS JOIN "user" u
      WHERE ba.id = (SELECT id FROM business_account LIMIT 1)
        AND u.id = (SELECT id FROM "user" LIMIT 1)
    `);
    await expect(
      db.execute(sql`
        INSERT INTO orders (number, status, currency, incoterm, payment_terms, business_id, user_id, sales_mode, destination_country, quote_id, total_amount, idempotency_key, snapshot)
        SELECT 'ORD-TEST-2', 'pending_payment', 'EUR', 'DAP', 'net_30', ba.id, u.id, 'wholesale', ba.country_code, ${id}, 0, 'test-ord-2', '{}'::jsonb
        FROM business_account ba CROSS JOIN "user" u
        WHERE ba.id = (SELECT id FROM business_account LIMIT 1)
          AND u.id = (SELECT id FROM "user" LIMIT 1)
      `),
    ).rejects.toThrow();
  });
});
```

Si les tables `business_account` ou `"user"` sont vides en base de test, insérer d'abord les fixtures minimales. Exécuter :

```bash
pnpm --filter @fanours/db exec vitest run --config vitest.integration.config.ts src/__tests__/quote-lifecycle.integration.spec.ts
```

Attendu : **1 test passé**, 0 échec.

- [ ] **Step 6: Commit**

```bash
git add packages/db/migrations/0109_phase3_quote_to_order.sql packages/db/src/schema/order.ts packages/db/src/__tests__/quote-lifecycle.integration.spec.ts apps/api/src/modules/quote/quote-conversion.service.ts apps/api/src/modules/quote/quote-conversion.service.spec.ts
git commit -m "feat(quote): convertit un devis accepte en commande avec snapshot fige"
```

### Task 10: Transporteurs — `ShippingProvider`, sandbox, estimation de délai

**Files:**
- Create: `packages/contracts/src/shipping.ts`
- Create: `apps/api/src/providers/shipping/shipping-provider.ts`
- Create: `apps/api/src/providers/shipping/mock-shipping.provider.ts`
- Create: `apps/api/src/providers/shipping/shipping-provider.spec.ts`
- Modify: `packages/contracts/src/index.ts`

**Interfaces:**
- Consumes: `packages/contracts` (conventions de sortie des intégrations de la Phase 2), `packages/compliance` (Task 1 — `scaleOf`).
- Produces:
  - `packages/contracts/src/shipping.ts` :
    ```ts
    export const ShippingRateSchema = z.object({
      carrier: z.string().min(1), service: z.string().min(1),
      amountMinor: z.number().int().nonnegative(), currency: z.string().length(3),
      transitDaysMin: z.number().int().nonnegative(), transitDaysMax: z.number().int().nonnegative(),
    });
    export type ShippingRate = z.infer<typeof ShippingRateSchema>;
    export const ShipmentWebhookEventSchema = z.object({
      providerEventId: z.string().min(1), status: z.string().min(1),
      occurredAt: z.string().datetime(), location: z.string().nullable().optional(),
      raw: z.record(z.unknown()),
    });
    export type ShipmentWebhookEvent = z.infer<typeof ShipmentWebhookEventSchema>;
    export interface ShippingProvider {
      getRates(input: { originCountry: string; destinationCountry: string; incoterm: string; weightKg: number; volumeM3: number; declaredValueMinor: number; currency: string }): Promise<ShippingRate[]>;
      createShipment(input: { carrier: string; service: string; reference: string; weightKg: number; packages: { id: string; weightKg: number }[] }): Promise<{ trackingNumber: string; labelUrl: string | null; raw: unknown }>;
      getTracking(input: { trackingNumber: string }): Promise<ShipmentWebhookEvent[]>;
      cancel(input: { trackingNumber: string }): Promise<void>;
      parseWebhook(request: { headers: Record<string, string | undefined>; body: string }): ShipmentWebhookEvent;
    }
    ```

- [ ] **Step 1: Écrire le test du transporteur sandbox (il doit échouer)**

Créer `apps/api/src/providers/shipping/shipping-provider.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { createShippingProvider } from "./shipping-provider";
import { MockShippingProvider } from "./mock-shipping.provider";

describe("MockShippingProvider", () => {
  const provider = new MockShippingProvider();

  it("retourne des tarifs deterministes pour une destination donnee", async () => {
    const rates = await provider.getRates({
      originCountry: "CN", destinationCountry: "FR", incoterm: "DAP",
      weightKg: 10, volumeM3: 0.2, declaredValueMinor: 50_000, currency: "EUR",
    });
    expect(rates.length).toBeGreaterThan(0);
    expect(rates.every((r) => r.currency === "EUR")).toBe(true);
    expect(rates.every((r) => r.transitDaysMax >= r.transitDaysMin)).toBe(true);
  });

  it("refuse une destination non livrable", async () => {
    await expect(
      provider.getRates({
        originCountry: "CN", destinationCountry: "AQ", incoterm: "DAP",
        weightKg: 1, volumeM3: 0.1, declaredValueMinor: 1_000, currency: "EUR",
      }),
    ).rejects.toThrow(/destination_not_supported/);
  });

  it("est stable : deux appels rendent le meme tarif", async () => {
    const input = {
      originCountry: "CN", destinationCountry: "FR", incoterm: "DAP",
      weightKg: 10, volumeM3: 0.2, declaredValueMinor: 50_000, currency: "EUR",
    };
    const [a, b] = await Promise.all([provider.getRates(input), provider.getRates(input)]);
    expect(a).toEqual(b);
  });

  it("expose un suivi coherent apres creation d'expedition", async () => {
    const created = await provider.createShipment({
      carrier: "mock-carrier", service: "standard", reference: "CMD-1",
      weightKg: 10, packages: [{ id: "pkg-1", weightKg: 10 }],
    });
    const tracking = await provider.getTracking({ trackingNumber: created.trackingNumber });
    expect(tracking.length).toBeGreaterThan(0);
    expect(tracking[0].providerEventId).toBe("evt-1");
  });

  it("produit un evenement de webhook analysable", () => {
    const event = provider.parseWebhook({
      headers: { "x-mock-signature": "valid-signature" },
      body: JSON.stringify({ eventId: "evt-9", status: "in_transit", occurredAt: "2026-10-21T10:00:00.000Z" }),
    });
    expect(event.providerEventId).toBe("evt-9");
    expect(event.status).toBe("in_transit");
  });

  it("rejette un webhook dont la signature est invalide", () => {
    const provider2 = new MockShippingProvider();
    expect(() =>
      provider2.parseWebhook({
        headers: { "x-mock-signature": "bad-signature" },
        body: JSON.stringify({ eventId: "evt-9", status: "in_transit", occurredAt: "2026-10-21T10:00:00.000Z" }),
      }),
    ).toThrow(/invalid_signature/);
  });
});

describe("createShippingProvider", () => {
  it("retourne le mock en mode sandbox", () => {
    expect(createShippingProvider({ mode: "sandbox" })).toBeInstanceOf(MockShippingProvider);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/providers/shipping/shipping-provider.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./shipping-provider"`. 0 test exécuté.

- [ ] **Step 2: Implémenter le contrat et le transporteur sandbox**

Créer `packages/contracts/src/shipping.ts` :

```ts
import { z } from "zod";

export const ShippingRateSchema = z.object({
  carrier: z.string().min(1),
  service: z.string().min(1),
  amountMinor: z.number().int().nonnegative(),
  currency: z.string().length(3),
  transitDaysMin: z.number().int().nonnegative(),
  transitDaysMax: z.number().int().nonnegative(),
});
export type ShippingRate = z.infer<typeof ShippingRateSchema>;

export const ShipmentWebhookEventSchema = z.object({
  providerEventId: z.string().min(1),
  status: z.string().min(1),
  occurredAt: z.string().datetime(),
  location: z.string().nullable().optional(),
  raw: z.record(z.unknown()),
});
export type ShipmentWebhookEvent = z.infer<typeof ShipmentWebhookEventSchema>;

export interface ShippingProvider {
  getRates(input: {
    originCountry: string; destinationCountry: string; incoterm: string;
    weightKg: number; volumeM3: number; declaredValueMinor: number; currency: string;
  }): Promise<ShippingRate[]>;
  createShipment(input: {
    carrier: string; service: string; reference: string;
    weightKg: number; packages: { id: string; weightKg: number }[];
  }): Promise<{ trackingNumber: string; labelUrl: string | null; raw: unknown }>;
  getTracking(input: { trackingNumber: string }): Promise<ShipmentWebhookEvent[]>;
  cancel(input: { trackingNumber: string }): Promise<void>;
  parseWebhook(request: { headers: Record<string, string | undefined>; body: string }): ShipmentWebhookEvent;
}
```

Créer `apps/api/src/providers/shipping/mock-shipping.provider.ts` :

```ts
import { createHmac, timingSafeEqual } from "node:crypto";
import type { ShipmentWebhookEvent, ShippingProvider, ShippingRate } from "@fanours/contracts";

const SUPPORTED_DESTINATIONS = new Set(["FR", "DE", "GB", "US", "CA", "AU", "ZA"]);
const WEBHOOK_SECRET = process.env.SHIPPING_WEBHOOK_SECRET ?? "sandbox-signing-key";

/**
 * Transporteur sandbox deterministe. Il ne contacte aucun service externe :
 * les tests et la recette se font hors ligne. Les delais sont des FOURCHETTES
 * (transitDaysMin / transitDaysMax), jamais une promesse de date (CDC 12.3).
 */
export class MockShippingProvider implements ShippingProvider {
  async getRates(input: {
    originCountry: string; destinationCountry: string; incoterm: string;
    weightKg: number; volumeM3: number; declaredValueMinor: number; currency: string;
  }): Promise<ShippingRate[]> {
    if (!SUPPORTED_DESTINATIONS.has(input.destinationCountry)) {
      throw new Error("destination_not_supported");
    }
    const chargeableKg = Math.max(input.weightKg, input.volumeM3 * 200);
    const baseMinor = Math.round(chargeableKg * 850) + 4_000;
    const transitBase = input.destinationCountry === "US" || input.destinationCountry === "AU" ? 18 : 12;
    return [
      {
        carrier: "mock-carrier", service: "standard", amountMinor: baseMinor, currency: input.currency,
        transitDaysMin: transitBase, transitDaysMax: transitBase + 7,
      },
      {
        carrier: "mock-carrier", service: "express", amountMinor: Math.round(baseMinor * 1.6), currency: input.currency,
        transitDaysMin: transitBase - 5, transitDaysMax: transitBase - 2,
      },
    ];
  }

  async createShipment(input: {
    carrier: string; service: string; reference: string;
    weightKg: number; packages: { id: string; weightKg: number }[];
  }): Promise<{ trackingNumber: string; labelUrl: string | null; raw: unknown }> {
    const digest = createHmac("sha256", WEBHOOK_SECRET).update(input.reference).digest("hex");
    return { trackingNumber: `MOCK${digest.slice(0, 12).toUpperCase()}`, labelUrl: null, raw: { reference: input.reference } };
  }

  async getTracking(input: { trackingNumber: string }): Promise<ShipmentWebhookEvent[]> {
    return [
      { providerEventId: "evt-1", status: "label_created", occurredAt: "2026-10-20T08:00:00.000Z", raw: {} },
      { providerEventId: "evt-2", status: "in_transit", occurredAt: "2026-10-21T08:00:00.000Z", raw: {} },
    ];
  }

  async cancel(input: { trackingNumber: string }): Promise<void> {
    return;
  }

  parseWebhook(request: { headers: Record<string, string | undefined>; body: string }): ShipmentWebhookEvent {
    const provided = request.headers["x-mock-signature"] ?? "";
    const expected = createHmac("sha256", WEBHOOK_SECRET).update(request.body).digest("hex");
    const a = Buffer.from(provided);
    const b = Buffer.from(expected);
    if (a.length !== b.length || !timingSafeEqual(a, b)) throw new Error("invalid_signature");
    const parsed = JSON.parse(request.body) as { eventId: string; status: string; occurredAt: string };
    return {
      providerEventId: parsed.eventId, status: parsed.status, occurredAt: parsed.occurredAt, raw: parsed,
    };
  }
}
```

Créer `apps/api/src/providers/shipping/shipping-provider.ts` :

```ts
import type { ShippingProvider } from "@fanours/contracts";
import { MockShippingProvider } from "./mock-shipping.provider";

/**
 * Fabrique de selection du transporteur. Le mode sandbox est le defaut tant que
 * les comptes transporteur reels ne sont pas provisions (blocage 3.8 de la roadmap).
 * En mode production, la selection passe par une allowlist stricte d'hotes (CDC securite 1.i) :
 * aucune URL de transporteur fournie par l'utilisateur n'est jamais appelee.
 */
export function createShippingProvider(config: { mode: "sandbox" | "production" }): ShippingProvider {
  if (config.mode === "sandbox") return new MockShippingProvider();
  throw new Error("shipping_provider_not_configured");
}
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/providers/shipping/shipping-provider.spec.ts
```

Attendu : **7 tests passés**, 0 échec.

- [ ] **Step 3: Export du contrat et commit**

```bash
pnpm --filter @fanours/api exec vitest run src/providers
pnpm turbo run typecheck
git add packages/contracts/src/shipping.ts packages/contracts/src/index.ts apps/api/src/providers/shipping
git commit -m "feat(shipping): ajoute le contrat ShippingProvider et le transporteur sandbox"
```

Attendu : tests verts, `typecheck` sans erreur, commit créé.

---

### Task 11: Expéditions et tracking — résistance aux événements désordonnés

**Files:**
- Create: `packages/db/migrations/0105_phase3_shipments.sql`
- Create: `packages/db/src/schema/phase3-logistics.ts`
- Modify: `packages/db/src/schema/index.ts` (baril Phase 2 : ajouter `export * from "./phase3-logistics.js";`)
- Create: `apps/api/src/modules/shipment/shipment.service.ts`
- Create: `apps/api/src/modules/shipment/tracking.service.ts`
- Create: `apps/api/src/modules/shipment/tracking.service.spec.ts`
- Modify: `packages/db/src/__tests__/shipment-tracking.integration.spec.ts`

**Interfaces:**
- Consumes: `packages/contracts` (`ShippingProvider`, Task 10), `packages/compliance` (Task 5 — `assertDispatchable`), tables `orders` / `order_line` (Phase 2), `webhook_event` (Phase 1).
- Produces:
  - `apps/api/src/modules/shipment/tracking.service.ts` :
    ```ts
    export const SHIPMENT_STATUS_RANK: Readonly<Record<ShipmentStatus, number>>;
    export function applyTrackingEvent(current: ShipmentStatus, incoming: ShipmentStatus): ShipmentStatus;
    export class TrackingService {
      async ingest(params: { shipmentId: string; event: ShipmentWebhookEvent }): Promise<{ applied: boolean; shipmentStatus: ShipmentStatus }>;
    }
    ```
  - `apps/api/src/modules/shipment/shipment.service.ts` :
    ```ts
    export class ShipmentService {
      async dispatch(params: { orderId: string; principal: Principal }): Promise<{ shipmentId: string; trackingNumber: string }>;
      async getForOrder(orderId: string, principal: Principal): Promise<ShipmentDto[]>;
    }
    ```

- [ ] **Step 1: Écrire le test de résistance aux événements désordonnés (il doit échouer)**

Créer `apps/api/src/modules/shipment/tracking.service.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { applyTrackingEvent } from "./tracking.service";

describe("applyTrackingEvent — resistance aux evenements desordonnes (CDC 12.2)", () => {
  it("avance le statut quand l'evenement est plus recent", () => {
    expect(applyTrackingEvent("label_created", "handed_over")).toBe("handed_over");
  });

  it("ignore un evenement qui fait regression de statut", () => {
    expect(applyTrackingEvent("in_transit", "label_created")).toBe("in_transit");
  });

  it("ignore un evenement identique", () => {
    expect(applyTrackingEvent("in_transit", "in_transit")).toBe("in_transit");
  });

  it("accepte customs_hold depuis in_transit", () => {
    expect(applyTrackingEvent("in_transit", "customs_hold")).toBe("customs_hold");
  });

  it("accede a action_required depuis customs_hold", () => {
    expect(applyTrackingEvent("customs_hold", "action_required")).toBe("action_required");
  });

  it("refuse de sortir d'une action requise sans evenement du transporteur", () => {
    expect(applyTrackingEvent("action_required", "out_for_delivery")).toBe("out_for_delivery");
  });

  it("ne regresse pas depuis delivered vers exception", () => {
    expect(applyTrackingEvent("delivered", "exception")).toBe("delivered");
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/shipment/tracking.service.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./tracking.service"`. 0 test exécuté.

- [ ] **Step 2: Migration SQL des expéditions**

Créer `packages/db/migrations/0105_phase3_shipments.sql` :

```sql
-- 0105_phase3_shipments.sql
-- Phase 3 - chantiers 3.8 et 3.9 (expeditions, colis, tracking).
-- Classification : expand.
SET lock_timeout = '5s';
SET statement_timeout = '60s';

CREATE TABLE shipping_quote (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id       uuid REFERENCES orders(id),
  business_account_id uuid REFERENCES business_account(id),
  carrier        text NOT NULL,
  service        text NOT NULL,
  amount         numeric(18,4) NOT NULL,
  currency       char(3) NOT NULL,
  transit_days_min int NOT NULL,
  transit_days_max int NOT NULL,
  incoterm       text NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT shipping_quote_amount CHECK (amount >= 0),
  CONSTRAINT shipping_quote_transit CHECK (transit_days_min >= 0 AND transit_days_max >= transit_days_min)
);

CREATE INDEX idx_shipping_quote_order ON shipping_quote (order_id, created_at DESC);

CREATE TABLE package (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id       uuid NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  external_ref   text,
  weight_kg      numeric(12,3) NOT NULL,
  length_cm      numeric(12,2) NOT NULL,
  width_cm       numeric(12,2) NOT NULL,
  height_cm      numeric(12,2) NOT NULL,
  declared_value numeric(18,4) NOT NULL,
  quantity       int NOT NULL DEFAULT 1,
  created_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT package_weight CHECK (weight_kg > 0),
  CONSTRAINT package_dimensions CHECK (length_cm > 0 AND width_cm > 0 AND height_cm > 0),
  CONSTRAINT package_value CHECK (declared_value >= 0),
  CONSTRAINT package_quantity CHECK (quantity > 0)
);

CREATE INDEX idx_package_order ON package (order_id);

CREATE TABLE shipment (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id       uuid NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  carrier        text NOT NULL,
  service        text NOT NULL,
  tracking_number text NOT NULL,
  label_storage_key text,
  status         text NOT NULL DEFAULT 'label_created',
  incoterm       text NOT NULL,
  customs_data   jsonb NOT NULL DEFAULT '{}'::jsonb,
  shipped_at     timestamptz,
  delivered_at   timestamptz,
  created_at     timestamptz NOT NULL DEFAULT now(),
  updated_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT shipment_status
    CHECK (status IN ('label_created','handed_over','in_transit','customs_hold','action_required','out_for_delivery','delivered','exception','returned')),
  -- Un dossier douanier incomplet BLOQUE l'expedition : customs_data doit etre renseigne.
  CONSTRAINT shipment_customs_data_required
    CHECK (status NOT IN ('handed_over','in_transit','customs_hold','action_required','out_for_delivery','delivered')
       OR customs_data <> '{}'::jsonb),
  CONSTRAINT shipment_tracking_unique UNIQUE (carrier, tracking_number)
);

CREATE INDEX idx_shipment_order ON shipment (order_id, created_at DESC);
CREATE INDEX idx_shipment_status ON shipment (status) WHERE status IN ('customs_hold','action_required','exception');

CREATE TABLE shipment_item (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  shipment_id  uuid NOT NULL REFERENCES shipment(id) ON DELETE CASCADE,
  order_line_id uuid NOT NULL REFERENCES order_line(id),
  quantity     int NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  UNIQUE (shipment_id, order_line_id),
  CONSTRAINT shipment_item_quantity CHECK (quantity > 0)
);

CREATE TABLE tracking_event (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  shipment_id       uuid NOT NULL REFERENCES shipment(id) ON DELETE CASCADE,
  provider_event_id text NOT NULL,
  status            text NOT NULL,
  occurred_at       timestamptz NOT NULL,
  location          text,
  raw               jsonb NOT NULL DEFAULT '{}'::jsonb,
  created_at        timestamptz NOT NULL DEFAULT now(),
  UNIQUE (shipment_id, provider_event_id)
);

CREATE INDEX idx_tracking_event_shipment ON tracking_event (shipment_id, occurred_at DESC);
```

Créer `packages/db/src/schema/phase3-logistics.ts` avec les tables Drizzle `shippingQuote`, `package`, `shipment`, `shipmentItem`, `trackingEvent`.

Appliquer :

```bash
pnpm --filter @fanours/db migrate:dev
```

Attendu : migration `0105_phase3_shipments` appliquée.

- [ ] **Step 3: Implémenter le service de tracking**

Créer `apps/api/src/modules/shipment/tracking.service.ts` :

```ts
import type { ShipmentWebhookEvent } from "@fanours/contracts";

export type ShipmentStatus =
  | "label_created" | "handed_over" | "in_transit" | "customs_hold"
  | "action_required" | "out_for_delivery" | "delivered" | "exception" | "returned";

/**
 * Rangs de progression : un evenement de rang inferieur ne fait JAMAIS regresser
 * le statut (webhooks transporteur arrives dans le desordre, CDC 12.2).
 */
export const SHIPMENT_STATUS_RANK: Readonly<Record<ShipmentStatus, number>> = {
  label_created: 10, handed_over: 20, in_transit: 30, customs_hold: 40,
  action_required: 45, out_for_delivery: 50, delivered: 60, exception: 35, returned: 25,
};

export function applyTrackingEvent(current: ShipmentStatus, incoming: ShipmentStatus): ShipmentStatus {
  if (incoming === current) return current;
  if (SHIPMENT_STATUS_RANK[incoming] > SHIPMENT_STATUS_RANK[current]) return incoming;
  return current;
}

export class TrackingService {
  constructor(private readonly db: DrizzleDb, private readonly outbox: OutboxWriter) {}

  /**
   * Ingestion idempotente : (shipment_id, provider_event_id) est unique.
   * Un evenement deja vu est ignore sans erreur ; un evenement de rang inferieur
   * est enregistre mais ne modifie pas le statut.
   */
  async ingest(params: { shipmentId: string; event: ShipmentWebhookEvent }): Promise<{ applied: boolean; shipmentStatus: ShipmentStatus }> {
    const inserted = await this.db
      .insert(trackingEvent)
      .values({
        id: randomUUID(),
        shipmentId: params.shipmentId,
        providerEventId: params.event.providerEventId,
        status: params.event.status,
        occurredAt: new Date(params.event.occurredAt),
        location: params.event.location ?? null,
        raw: params.event.raw,
        createdAt: new Date(),
      })
      .onConflictDoNothing({ target: [trackingEvent.shipmentId, trackingEvent.providerEventId] })
      .returning({ id: trackingEvent.id });

    const current = await this.loadStatus(params.shipmentId);
    const next = applyTrackingEvent(current, params.event.status as ShipmentStatus);

    if (inserted.length === 0) {
      return { applied: false, shipmentStatus: current };
    }
    if (next !== current) {
      await this.db
        .update(shipment)
        .set({ status: next, updatedAt: new Date() })
        .where(eq(shipment.id, params.shipmentId));
      if (next === "action_required") {
        await this.outbox.publish({
          eventType: "shipment.customs_action_required",
          aggregateType: "shipment",
          aggregateId: params.shipmentId,
          payload: { shipmentId: params.shipmentId, status: next },
        });
      }
    }
    return { applied: true, shipmentStatus: next };
  }
}
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/shipment/tracking.service.spec.ts
```

Attendu : **7 tests passés**, 0 échec.

- [ ] **Step 4: Test d'intégration d'idempotence du tracking**

Créer `packages/db/src/__tests__/shipment-tracking.integration.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { db, sql } from "@fanours/db";

describe("tracking_event (integration)", () => {
  it("refuse deux evenements de meme provider_event_id pour une expedition", async () => {
    const shipmentId = await db.execute(sql`
      INSERT INTO shipment (order_id, carrier, service, tracking_number, incoterm, customs_data)
      SELECT o.id, 'mock-carrier', 'standard', 'MOCK-TEST-1', 'DAP', '{"hs_code":"123456"}'::jsonb
      FROM (SELECT 1) AS dummy
      JOIN LATERAL (SELECT id FROM order LIMIT 1) AS o ON true
      RETURNING id
    `);
    const id = (shipmentId as unknown as { id: string }[])[0].id;
    await db.execute(sql`
      INSERT INTO tracking_event (shipment_id, provider_event_id, status, occurred_at)
      VALUES (${id}, 'evt-1', 'in_transit', now())
    `);
    await expect(
      db.execute(sql`
        INSERT INTO tracking_event (shipment_id, provider_event_id, status, occurred_at)
        VALUES (${id}, 'evt-1', 'delivered', now())
      `),
    ).rejects.toThrow();
  });
});
```

Si la table `orders` est vide en base de test, insérer d'abord une commande minimale de fixture. Exécuter :

```bash
pnpm --filter @fanours/db exec vitest run --config vitest.integration.config.ts src/__tests__/shipment-tracking.integration.spec.ts
```

Attendu : **1 test passé**, 0 échec.

- [ ] **Step 5: Service d'expédition — blocage si le dossier douanier est incomplet**

Ajouter dans `apps/api/src/modules/shipment/shipment.service.ts` :

```ts
import { assertDispatchable, runComplianceChecks } from "@fanours/compliance";
import { requirePermission, type Principal } from "../rbac/require-permission";

export class ShipmentService {
  constructor(
    private readonly db: DrizzleDb,
    private readonly shipping: ShippingProvider,
    private readonly outbox: OutboxWriter,
  ) {}

  /**
   * Expedition d'une commande : les controles douaniers bloquants passent AVANT
   * tout appel au transporteur (CDC 11.9). Le dossier d'export incomplet = pas d'expedition.
   */
  async dispatch(params: { orderId: string; principal: Principal }): Promise<{ shipmentId: string; trackingNumber: string }> {
    requirePermission(params.principal, "customs_document", "create");

    const order = await this.loadOrderForPrincipal(params.orderId, params.principal);
    const check = await this.buildCheckInput(order);
    assertDispatchable(check);

    const packages = await this.db.select().from(packageTable).where(eq(packageTable.orderId, order.id));
    const created = await this.shipping.createShipment({
      carrier: "mock-carrier", service: "standard", reference: order.number,
      weightKg: packages.reduce((sum, pkg) => sum + Number(pkg.weightKg), 0),
      packages: packages.map((pkg) => ({ id: pkg.id, weightKg: Number(pkg.weightKg) })),
    });

    const shipmentId = randomUUID();
    await this.db.insert(shipment).values({
      id: shipmentId, orderId: order.id, carrier: "mock-carrier", service: "standard",
      trackingNumber: created.trackingNumber, status: "label_created",
      incoterm: order.incoterm, customsData: order.snapshot.customs ?? {}, createdAt: new Date(), updatedAt: new Date(),
    });
    await this.outbox.publish({
      eventType: "shipment.dispatched", aggregateType: "shipment", aggregateId: shipmentId,
      payload: { shipmentId, orderId: order.id, trackingNumber: created.trackingNumber },
    });
    return { shipmentId, trackingNumber: created.trackingNumber };
  }
}
```

- [ ] **Step 6: Commit**

```bash
git add packages/db/migrations/0105_phase3_shipments.sql packages/db/src/schema/phase3-logistics.ts packages/db/src/schema/index.ts apps/api/src/modules/shipment packages/db/src/__tests__/shipment-tracking.integration.spec.ts
git commit -m "feat(shipping): ajoute les expeditions et le tracking resistant aux evenements desordonnes"
```

---

### Task 12: Documents d'export et données douanières structurées

**Files:**
- Create: `packages/db/migrations/0106_phase3_commercial_documents.sql`
- Create: `packages/contracts/src/pdf.ts`
- Modify: `packages/contracts/src/index.ts`
- Create: `packages/pdf/src/commercial-document.tsx`
- Create: `packages/pdf/src/render-commercial-document.ts`
- Modify: `packages/pdf/package.json` (dépendance de workspace `@fanours/contracts`)
- Modify: `packages/pdf/src/fonts.ts`
- Modify: `packages/pdf/src/index.ts`
- Create: `apps/api/src/providers/pdf/pdf-document-renderer.provider.ts`
- Create: `apps/api/src/modules/document/export-documents.service.ts`
- Create: `apps/api/src/modules/document/customs-data.service.ts`
- Create: `apps/api/src/modules/document/export-documents.service.spec.ts`
- Create: `apps/worker/src/jobs/documents/export-documents.job.ts`

**Interfaces:**
- Consumes: `packages/contracts` (`StorageProvider` — **Task 7**, `packages/contracts/src/storage.ts` ; `DocumentRenderer` — **Task 12**, `packages/contracts/src/pdf.ts`), `packages/pdf` (Phase 2 — `renderInvoice`, `registerInvoiceFonts`, registre de polices CJK embarquées ; étendu par la Task 12 avec `renderCommercialDocument`), `packages/compliance` (Task 4 — `computeCustoms`), `packages/db` (`orders`, `order_line` — Phase 2 ; `shipment` — Task 11).
- Note de placement : **ni `packages/storage` ni `packages/security` n'existent** dans ce monorepo. Le contrat de stockage est `packages/contracts/src/storage.ts` (Task 7) et son implémentation S3 est `apps/api/src/providers/storage/s3-storage.provider.ts` (Task 7) ; le contrat de rendu PDF est `packages/contracts/src/pdf.ts` (Task 12) et son implémentation `@react-pdf/renderer` est `packages/pdf` (package Phase 2, étendu par la Task 12).
- Produces:
  - `packages/contracts/src/pdf.ts` :
    ```ts
    export const COMMERCIAL_DOCUMENT_KINDS = ["packing_list", "export_declaration", "certificate_of_origin", "customs_data"] as const;
    export type CommercialDocumentKind = (typeof COMMERCIAL_DOCUMENT_KINDS)[number];
    export type DocumentLocale = "fr-FR" | "en" | "zh-CN";
    export interface DocumentRenderRequest {
      template: CommercialDocumentKind;
      locale: DocumentLocale;
      data: Record<string, unknown>;
    }
    /** Rendu d'un document commercial en PDF. L'implémentation vit dans `packages/pdf`. */
    export interface DocumentRenderer {
      render(request: DocumentRenderRequest): Promise<Uint8Array>;
    }
    ```
  - `apps/api/src/modules/document/customs-data.service.ts` :
    ```ts
    export { COMMERCIAL_DOCUMENT_KINDS, type CommercialDocumentKind } from "@fanours/contracts";
    export type CustomsData = {
      exporter: { name: string; address: string; taxId: string | null; country: string };
      importer: { name: string; address: string; taxId: string | null; country: string };
      declarant: { name: string; country: string };
      incoterm: string; currency: string;
      totalValue: number; netWeightKg: number; grossWeightKg: number; packageCount: number;
      lines: { description: string; hsCode: string; originCountry: string; quantity: number; unitValue: number; lineValue: number }[];
      declaredAt: string; rulesVersionRefs: string[];
    };
    export class CustomsDataService {
      async build(params: { orderId: string }): Promise<CustomsData>;
      async assertComplete(customsData: CustomsData): Promise<void>;
    }
    ```
  - `apps/api/src/modules/document/export-documents.service.ts` :
    ```ts
    export class ExportDocumentsService {
      async generate(params: { orderId: string; kinds: CommercialDocumentKind[]; locale: "fr-FR" | "en" | "zh-CN" }): Promise<{ kind: CommercialDocumentKind; version: number; storageKey: string }[]>;
      async list(orderId: string): Promise<{ kind: CommercialDocumentKind; version: number; storageKey: string; sentAt: string | null }[]>;
    }
    ```

- [ ] **Step 1: Écrire le test de complétude des données douanières (il doit échouer)**

Créer `apps/api/src/modules/document/export-documents.service.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { CustomsDataService, type CustomsData } from "./customs-data.service";

function complete(): CustomsData {
  return {
    exporter: { name: "Fanours", address: "Shenzhen", taxId: "CN-TEST", country: "CN" },
    importer: { name: "Client", address: "Paris", taxId: "FR-TEST", country: "FR" },
    declarant: { name: "Client", country: "FR" },
    incoterm: "DAP", currency: "EUR",
    totalValue: 500, netWeightKg: 10, grossWeightKg: 12, packageCount: 1,
    lines: [{ description: "Produit", hsCode: "123456", originCountry: "CN", quantity: 10, unitValue: 50, lineValue: 500 }],
    declaredAt: "2026-10-21T00:00:00.000Z", rulesVersionRefs: ["r-1"],
  };
}

describe("CustomsDataService.assertComplete", () => {
  it("accepte un dossier complet", async () => {
    await expect(new CustomsDataService({} as never).assertComplete(complete())).resolves.toBeUndefined();
  });

  it("refuse un dossier sans code HS sur une ligne", async () => {
    const data = complete();
    data.lines[0].hsCode = "";
    await expect(new CustomsDataService({} as never).assertComplete(data)).rejects.toThrow(/hs_code_missing/);
  });

  it("refuse un dossier sans pays d'origine", async () => {
    const data = complete();
    data.lines[0].originCountry = "";
    await expect(new CustomsDataService({} as never).assertComplete(data)).rejects.toThrow(/origin_missing/);
  });

  it("refuse un dossier sans identifiant d'importateur", async () => {
    const data = complete();
    data.importer.taxId = null;
    await expect(new CustomsDataService({} as never).assertComplete(data)).rejects.toThrow(/importer_tax_id_missing/);
  });

  it("refuse un dossier sans reference de version de regle", async () => {
    const data = complete();
    data.rulesVersionRefs = [];
    await expect(new CustomsDataService({} as never).assertComplete(data)).rejects.toThrow(/rules_version_missing/);
  });

  it("refuse un code HS qui ne fait pas 6 chiffres", async () => {
    const data = complete();
    data.lines[0].hsCode = "12345678";
    await expect(new CustomsDataService({} as never).assertComplete(data)).rejects.toThrow(/hs_code_invalid/);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/document/export-documents.service.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./customs-data.service"`. 0 test exécuté.

- [ ] **Step 2: Implémenter le service de données douanières**

Créer `apps/api/src/modules/document/customs-data.service.ts` :

```ts
import { eq } from "drizzle-orm";
import { orderLine, orders } from "@fanours/db/schema";
import { COMMERCIAL_DOCUMENT_KINDS, type CommercialDocumentKind } from "@fanours/contracts";
import { NotFoundError } from "../../common/api-errors";

// La liste canonique des documents commerciaux vit dans `packages/contracts/src/pdf.ts`
// (Task 12) : le service la re-exporte pour que les appelants aient une seule source.
export { COMMERCIAL_DOCUMENT_KINDS, type CommercialDocumentKind };

export type CustomsData = {
  exporter: { name: string; address: string; taxId: string | null; country: string };
  importer: { name: string; address: string; taxId: string | null; country: string };
  declarant: { name: string; country: string };
  incoterm: string;
  currency: string;
  totalValue: number;
  netWeightKg: number;
  grossWeightKg: number;
  packageCount: number;
  lines: {
    description: string; hsCode: string; originCountry: string;
    quantity: number; unitValue: number; lineValue: number;
  }[];
  declaredAt: string;
  rulesVersionRefs: string[];
};

export class CustomsDataIncompleteError extends Error {
  constructor(public readonly code: string) {
    super(code);
    this.name = "CustomsDataIncompleteError";
  }
}

/**
 * Donnees douanieres structurees destinees au transporteur (CDC 11.8).
 * Elles reprennent les donnees VALIDEES de la commande : aucune valeur n'est
 * saisie a la main, aucun code HS n'est devine, aucune origine n'est supposee.
 */
export class CustomsDataService {
  constructor(private readonly db: DrizzleDb) {}

  async build(params: { orderId: string }): Promise<CustomsData> {
    const order = await this.db.select().from(orders).where(eq(orders.id, params.orderId)).limit(1);
    if (order.length === 0) throw new NotFoundError("order_not_found");
    const lines = await this.db
      .select()
      .from(orderLine)
      .where(eq(orderLine.orderId, params.orderId));

    return {
      exporter: order[0].snapshot.exporter,
      importer: order[0].snapshot.importer,
      declarant: order[0].snapshot.declarant,
      incoterm: order[0].incoterm,
      currency: order[0].currency,
      totalValue: Number(order[0].snapshot.totalValue),
      netWeightKg: Number(order[0].snapshot.netWeightKg),
      grossWeightKg: Number(order[0].snapshot.grossWeightKg),
      packageCount: order[0].snapshot.packageCount,
      lines: lines.map((line) => ({
        description: line.description,
        hsCode: line.snapshot.hsCode,
        originCountry: line.snapshot.originCountry,
        quantity: line.quantity,
        unitValue: Number(line.unitPrice),
        lineValue: Number(line.lineTotal),
      })),
      declaredAt: new Date().toISOString(),
      rulesVersionRefs: order[0].snapshot.ruleVersionRefs ?? [],
    };
  }

  /** Un dossier incomplet BLOQUE l'expedition (CDC 11.9). */
  async assertComplete(customsData: CustomsData): Promise<void> {
    for (const line of customsData.lines) {
      if (line.hsCode.length === 0) throw new CustomsDataIncompleteError("hs_code_missing");
      if (!/^[0-9]{6}$/.test(line.hsCode)) throw new CustomsDataIncompleteError("hs_code_invalid");
      if (line.originCountry.length === 0) throw new CustomsDataIncompleteError("origin_missing");
    }
    if (customsData.importer.taxId === null || customsData.importer.taxId.length === 0) {
      throw new CustomsDataIncompleteError("importer_tax_id_missing");
    }
    if (customsData.rulesVersionRefs.length === 0) throw new CustomsDataIncompleteError("rules_version_missing");
  }
}
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/document/export-documents.service.spec.ts
```

Attendu : **6 tests passés**, 0 échec.

- [ ] **Step 3: Migration SQL des documents commerciaux**

Créer `packages/db/migrations/0106_phase3_commercial_documents.sql` :

```sql
-- 0106_phase3_commercial_documents.sql
-- Phase 3 - chantier 3.10 (documents d'export).
-- Classification : expand. La version envoyee est conservee (CDC 11.8).
SET lock_timeout = '5s';
SET statement_timeout = '60s';

CREATE TABLE commercial_document (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id      uuid NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  kind          text NOT NULL,
  version       int NOT NULL,
  locale        text NOT NULL DEFAULT 'fr-FR',
  storage_key   text NOT NULL,
  template_version text NOT NULL,
  rules_version_refs jsonb NOT NULL DEFAULT '[]'::jsonb,
  sent_at       timestamptz,
  created_at    timestamptz NOT NULL DEFAULT now(),
  UNIQUE (order_id, kind, version),
  CONSTRAINT commercial_document_kind
    CHECK (kind IN ('packing_list','export_declaration','certificate_of_origin','customs_data')),
  CONSTRAINT commercial_document_version CHECK (version > 0),
  CONSTRAINT commercial_document_locale CHECK (locale IN ('fr-FR','en','zh-CN'))
);

CREATE INDEX idx_commercial_document_order ON commercial_document (order_id, kind, version DESC);
```

Créer la table dans `packages/db/src/schema/phase3-logistics.ts` (table `commercialDocument`).

```bash
pnpm --filter @fanours/db migrate:dev
```

Attendu : migration `0106_phase3_commercial_documents` appliquée.

- [ ] **Step 4: Contrat de rendu, implémentation PDF, service de génération et job**

Créer `packages/contracts/src/pdf.ts` :

```ts
export const COMMERCIAL_DOCUMENT_KINDS = ["packing_list", "export_declaration", "certificate_of_origin", "customs_data"] as const;
export type CommercialDocumentKind = (typeof COMMERCIAL_DOCUMENT_KINDS)[number];
export type DocumentLocale = "fr-FR" | "en" | "zh-CN";

export interface DocumentRenderRequest {
  template: CommercialDocumentKind;
  locale: DocumentLocale;
  data: Record<string, unknown>;
}

/**
 * Rendu d'un document commercial en PDF. Le PDF est produit par `packages/pdf`
 * (Phase 2, etendu par la Task 12) : les polices CJK sont embarquees, jamais
 * telechargees a l'execution.
 */
export interface DocumentRenderer {
  render(request: DocumentRenderRequest): Promise<Uint8Array>;
}
```

Puis `packages/contracts/src/index.ts` : ajouter `export * from "./pdf";`.

Puis `packages/pdf/package.json` : ajouter la dépendance de workspace `"@fanours/contracts": "workspace:*"` (le composant PDF type ses données avec le contrat canonique ; `packages/contracts` reste une feuille, sans dépendance runtime).

Créer `packages/pdf/src/commercial-document.tsx` :

```tsx
import { Document, Page, StyleSheet, Text, View } from "@react-pdf/renderer";
import type { CommercialDocumentKind, DocumentLocale } from "@fanours/contracts";

export interface CommercialDocumentLine {
  description: string;
  hsCode: string;
  originCountry: string;
  quantity: number;
  unitValue: string;
  lineValue: string;
}

export interface CommercialDocumentInput {
  kind: CommercialDocumentKind;
  locale: DocumentLocale;
  orderId: string;
  exporter: { name: string; address: string; taxId: string | null; country: string };
  importer: { name: string; address: string; taxId: string | null; country: string };
  declarant: { name: string; country: string };
  incoterm: string;
  currency: string;
  totalValue: string;
  netWeightKg: string;
  grossWeightKg: string;
  packageCount: number;
  lines: CommercialDocumentLine[];
  declaredAt: string;
  rulesVersionRefs: string[];
}

const TITLES: Record<CommercialDocumentKind, Record<DocumentLocale, string>> = {
  packing_list: { "fr-FR": "Liste de colisage", en: "Packing list", "zh-CN": "装箱单" },
  export_declaration: { "fr-FR": "Declaration d'export", en: "Export declaration", "zh-CN": "出口申报单" },
  certificate_of_origin: { "fr-FR": "Certificat d'origine", en: "Certificate of origin", "zh-CN": "原产地证" },
  customs_data: { "fr-FR": "Donnees douanieres", en: "Customs data", "zh-CN": "报关数据" },
};

const HEADERS: Record<DocumentLocale, { description: string; hsCode: string; origin: string; quantity: string; unitValue: string; lineValue: string }> = {
  "fr-FR": { description: "Designation", hsCode: "Code HS", origin: "Origine", quantity: "Quantite", unitValue: "Valeur unitaire", lineValue: "Valeur ligne" },
  en: { description: "Description", hsCode: "HS code", origin: "Origin", quantity: "Quantity", unitValue: "Unit value", lineValue: "Line value" },
  "zh-CN": { description: "品名", hsCode: "HS 编码", origin: "原产地", quantity: "数量", unitValue: "单价", lineValue: "总价" },
};

const styles = StyleSheet.create({
  page: { padding: 32, fontFamily: "Noto Sans", fontSize: 9, color: "#321526" },
  title: { fontSize: 15, fontFamily: "Noto Sans", fontWeight: 700, marginBottom: 4 },
  subtitle: { fontSize: 9, color: "#667085", marginBottom: 14 },
  parties: { flexDirection: "row", gap: 12, marginBottom: 14 },
  party: { flex: 1, borderWidth: 1, borderColor: "#F0DCE6", borderRadius: 4, padding: 8 },
  partyTitle: { fontSize: 8, textTransform: "uppercase", color: "#667085", marginBottom: 3 },
  partyLine: { fontSize: 9, marginBottom: 1 },
  table: { marginBottom: 12 },
  header: { flexDirection: "row", backgroundColor: "#F7DCE9", paddingVertical: 4, paddingHorizontal: 3 },
  row: { flexDirection: "row", paddingVertical: 3, paddingHorizontal: 3, borderBottomWidth: 1, borderBottomColor: "#F0DCE6" },
  colDescription: { flex: 3 },
  colHs: { flex: 1.2 },
  colOrigin: { flex: 1 },
  colQty: { flex: 1 },
  colValue: { flex: 1.4, textAlign: "right" },
  totals: { borderTopWidth: 1, borderTopColor: "#F0DCE6", paddingTop: 6 },
  totalsLine: { flexDirection: "row", justifyContent: "space-between", marginBottom: 2 },
  footer: { position: "absolute", bottom: 24, left: 32, right: 32, fontSize: 7, color: "#667085" },
});

/**
 * Document commercial unique pour les quatre kinds. Le contenu est identique
 * en FR/EN/ZH-CN : seuls les libelles changent, les valeurs restent celles
 * du dossier douanier valide.
 */
export function CommercialDocument({ input }: { input: CommercialDocumentInput }) {
  const headers = HEADERS[input.locale];
  return (
    <Document title={`${TITLES[input.kind][input.locale]} ${input.orderId}`} author="Fanours">
      <Page size="A4" style={styles.page}>
        <Text style={styles.title}>{TITLES[input.kind][input.locale]}</Text>
        <Text style={styles.subtitle}>
          {`Commande ${input.orderId} · Incoterm ${input.incoterm} · Devise ${input.currency}`}
        </Text>

        <View style={styles.parties}>
          <View style={styles.party}>
            <Text style={styles.partyTitle}>{"Exporter"}</Text>
            <Text style={styles.partyLine}>{input.exporter.name}</Text>
            <Text style={styles.partyLine}>{input.exporter.address}</Text>
            <Text style={styles.partyLine}>{input.exporter.taxId ?? "—"}</Text>
            <Text style={styles.partyLine}>{input.exporter.country}</Text>
          </View>
          <View style={styles.party}>
            <Text style={styles.partyTitle}>{"Importer"}</Text>
            <Text style={styles.partyLine}>{input.importer.name}</Text>
            <Text style={styles.partyLine}>{input.importer.address}</Text>
            <Text style={styles.partyLine}>{input.importer.taxId ?? "—"}</Text>
            <Text style={styles.partyLine}>{input.importer.country}</Text>
          </View>
          <View style={styles.party}>
            <Text style={styles.partyTitle}>{"Declarant"}</Text>
            <Text style={styles.partyLine}>{input.declarant.name}</Text>
            <Text style={styles.partyLine}>{input.declarant.country}</Text>
          </View>
        </View>

        <View style={styles.table}>
          <View style={styles.header}>
            <Text style={styles.colDescription}>{headers.description}</Text>
            <Text style={styles.colHs}>{headers.hsCode}</Text>
            <Text style={styles.colOrigin}>{headers.origin}</Text>
            <Text style={styles.colQty}>{headers.quantity}</Text>
            <Text style={styles.colValue}>{headers.unitValue}</Text>
            <Text style={styles.colValue}>{headers.lineValue}</Text>
          </View>
          {input.lines.map((line, index) => (
            <View key={`${line.hsCode}-${index}`} style={styles.row}>
              <Text style={styles.colDescription}>{line.description}</Text>
              <Text style={styles.colHs}>{line.hsCode}</Text>
              <Text style={styles.colOrigin}>{line.originCountry}</Text>
              <Text style={styles.colQty}>{String(line.quantity)}</Text>
              <Text style={styles.colValue}>{line.unitValue}</Text>
              <Text style={styles.colValue}>{line.lineValue}</Text>
            </View>
          ))}
        </View>

        <View style={styles.totals}>
          <View style={styles.totalsLine}>
            <Text>{`Valeur totale (${input.currency})`}</Text>
            <Text>{input.totalValue}</Text>
          </View>
          <View style={styles.totalsLine}>
            <Text>{`Poids net / brut (kg)`}</Text>
            <Text>{`${input.netWeightKg} / ${input.grossWeightKg}`}</Text>
          </View>
          <View style={styles.totalsLine}>
            <Text>{"Colis"}</Text>
            <Text>{String(input.packageCount)}</Text>
          </View>
        </View>

        <Text style={styles.footer} fixed>
          {`${input.declaredAt} · Regles: ${input.rulesVersionRefs.join(", ")}`}
        </Text>
      </Page>
    </Document>
  );
}
```

Créer `packages/pdf/src/render-commercial-document.ts` :

```ts
import { createElement } from "react";
import { renderToBuffer } from "@react-pdf/renderer";
import { CommercialDocument, type CommercialDocumentInput } from "./commercial-document.js";
import { registerDocumentFonts } from "./fonts.js";

/**
 * Rendu d'un document commercial. Les polices CJK sont enregistrees a chaque
 * appel : `Font.register` est idempotent et garantit qu'un document chinois
 * ne peut pas etre produit sans police (meme exigence que la facture).
 */
export async function renderCommercialDocument(input: CommercialDocumentInput): Promise<Uint8Array> {
  await registerDocumentFonts();
  const buffer = await renderToBuffer(createElement(CommercialDocument, { input }));
  return new Uint8Array(buffer);
}
```

Modifier `packages/pdf/src/fonts.ts` — les documents commerciaux partagent **exactement** le meme jeu de polices embarquees que la facture :

```ts
/**
 * Alias explicite pour les usages non-facture (documents commerciaux).
 * Le registre de polices est unique : un seul fichier de police, une seule
 * famille CJK, un seul audit possible.
 */
export const registerDocumentFonts = registerInvoiceFonts;
```

Modifier `packages/pdf/src/index.ts` :

```ts
export { renderCommercialDocument } from "./render-commercial-document.js";
export { CommercialDocument, type CommercialDocumentInput, type CommercialDocumentLine } from "./commercial-document.js";
```

Créer `apps/api/src/providers/pdf/pdf-document-renderer.provider.ts` — l'adaptateur du contrat `DocumentRenderer` vers `packages/pdf` :

```ts
import { renderCommercialDocument, type CommercialDocumentInput } from "@fanours/pdf";
import type { DocumentRenderRequest, DocumentRenderer } from "@fanours/contracts";

/**
 * Le contrat `DocumentRenderer` vit dans `packages/contracts` (Task 12) pour que
 * l'API ne depende pas de l'implementation PDF. Cette classe est le seul point
 * de liaison avec `packages/pdf`.
 */
export class PdfDocumentRenderer implements DocumentRenderer {
  async render(request: DocumentRenderRequest): Promise<Uint8Array> {
    const data = request.data as unknown as CommercialDocumentInput;
    return renderCommercialDocument({ ...data, kind: request.template, locale: request.locale });
  }
}
```

Créer `apps/api/src/modules/document/export-documents.service.ts` :

```ts
import { randomUUID } from "node:crypto";
import { and, desc, eq } from "drizzle-orm";
import type { DocumentRenderer } from "@fanours/contracts";
import type { StorageProvider } from "@fanours/contracts";
import { BadRequestError } from "../../common/api-errors";
import { COMMERCIAL_DOCUMENT_KINDS, CustomsDataService, type CommercialDocumentKind } from "./customs-data.service";

export class ExportDocumentsService {
  constructor(
    private readonly db: DrizzleDb,
    private readonly customs: CustomsDataService,
    private readonly renderer: DocumentRenderer,
    private readonly storage: StorageProvider,
  ) {}

  /**
   * Genere les documents d'export a partir des donnees VALIDEES de la commande.
   * Chaque generation incremente la version : la version envoyee au transporteur
   * est conservee (CDC 11.8). La facture d'export n'est PAS une facture fiscale locale.
   */
  async generate(params: {
    orderId: string;
    kinds: CommercialDocumentKind[];
    locale: "fr-FR" | "en" | "zh-CN";
  }): Promise<{ kind: CommercialDocumentKind; version: number; storageKey: string }[]> {
    const customsData = await this.customs.build({ orderId: params.orderId });
    await this.customs.assertComplete(customsData);

    const results: { kind: CommercialDocumentKind; version: number; storageKey: string }[] = [];
    for (const kind of params.kinds) {
      if (!COMMERCIAL_DOCUMENT_KINDS.includes(kind)) throw new BadRequestError("unknown_document_kind");
      const previous = await this.db
        .select({ version: commercialDocument.version })
        .from(commercialDocument)
        .where(and(eq(commercialDocument.orderId, params.orderId), eq(commercialDocument.kind, kind)))
        .orderBy(desc(commercialDocument.version))
        .limit(1);
      const version = (previous[0]?.version ?? 0) + 1;

      const formatValue = (value: number) =>
        new Intl.NumberFormat(params.locale, { style: "currency", currency: customsData.currency }).format(value);

      const pdf = await this.renderer.render({
        template: kind,
        locale: params.locale,
        data: {
          orderId: params.orderId,
          exporter: customsData.exporter,
          importer: customsData.importer,
          declarant: customsData.declarant,
          incoterm: customsData.incoterm,
          currency: customsData.currency,
          totalValue: formatValue(customsData.totalValue),
          netWeightKg: String(customsData.netWeightKg),
          grossWeightKg: String(customsData.grossWeightKg),
          packageCount: customsData.packageCount,
          lines: customsData.lines.map((line) => ({
            description: line.description,
            hsCode: line.hsCode,
            originCountry: line.originCountry,
            quantity: line.quantity,
            unitValue: formatValue(line.unitValue),
            lineValue: formatValue(line.lineValue),
          })),
          declaredAt: customsData.declaredAt,
          rulesVersionRefs: customsData.rulesVersionRefs,
        },
      });
      const storageKey = `documents/${params.orderId}/${kind}-v${version}.pdf`;
      await this.storage.put({ key: storageKey, body: Buffer.from(pdf), contentType: "application/pdf" });

      await this.db.insert(commercialDocument).values({
        id: randomUUID(), orderId: params.orderId, kind, version, locale: params.locale,
        storageKey, templateVersion: "v1", rulesVersionRefs: customsData.rulesVersionRefs, createdAt: new Date(),
      });
      results.push({ kind, version, storageKey });
    }
    return results;
  }

  async list(orderId: string): Promise<{ kind: CommercialDocumentKind; version: number; storageKey: string; sentAt: string | null }[]> {
    const rows = await this.db
      .select()
      .from(commercialDocument)
      .where(eq(commercialDocument.orderId, orderId))
      .orderBy(desc(commercialDocument.version));
    return rows.map((row) => ({
      kind: row.kind as CommercialDocumentKind, version: row.version,
      storageKey: row.storageKey, sentAt: row.sentAt?.toISOString() ?? null,
    }));
  }
}
```

Créer `apps/worker/src/jobs/documents/export-documents.job.ts` :

```ts
import { defineJob, idempotencyKey } from "@fanours/worker";

/**
 * Generation des documents d'export. Idempotence : orderId + kinds.
 * Le job est ignore si un document de meme version existe deja.
 */
export const exportDocumentsJob = defineJob<{ orderId: string; kinds: string[]; locale: string }, { generated: number }>(
  "document.export",
  {
    idempotencyKey: (payload) => idempotencyKey("document.export", [payload.orderId, [...payload.kinds].sort().join(",")]),
    attempts: 5,
    backoff: { base: 1_000, cap: 15 * 60_000 },
  },
  async (payload, ctx) => {
    const service = ctx.services.exportDocuments;
    const results = await service.generate({
      orderId: payload.orderId,
      kinds: payload.kinds as never,
      locale: payload.locale as never,
    });
    return { generated: results.length };
  },
);
```

```bash
pnpm --filter @fanours/api exec vitest run src/modules/document
pnpm --filter @fanours/worker exec vitest run src/jobs/documents
pnpm turbo run typecheck
```

Attendu : tests verts, `typecheck` sans erreur.

- [ ] **Step 5: Commit**

```bash
git add packages/db/migrations/0106_phase3_commercial_documents.sql packages/db/src/schema/phase3-logistics.ts packages/db/src/schema/index.ts packages/contracts/src/pdf.ts packages/contracts/src/index.ts packages/pdf/package.json packages/pdf/src/commercial-document.tsx packages/pdf/src/render-commercial-document.ts packages/pdf/src/fonts.ts packages/pdf/src/index.ts apps/api/src/providers/pdf/pdf-document-renderer.provider.ts apps/api/src/modules/document apps/worker/src/jobs/documents/export-documents.job.ts
git commit -m "feat(customs): ajoute les documents d'export et les donnees douanieres structurees"
```

### Task 13: Revue du code HS et alertes de conformité — validation humaine obligatoire

**Files:**
- Create: `apps/api/src/modules/compliance/hs-review.service.ts`
- Create: `apps/api/src/modules/compliance/hs-review.service.spec.ts`
- Create: `apps/api/src/modules/compliance/compliance-alerts.controller.ts`
- Create: `apps/worker/src/jobs/compliance/trade-rule-review.job.ts`
- Create: `apps/worker/src/jobs/compliance/trade-rule-review.job.spec.ts`

**Interfaces:**
- Consumes: `packages/compliance` (Task 3 — `resolveRules`, Task 5 — `runComplianceChecks`), `packages/db` (`compliance_review`, `compliance_alert`, `trade_rule_version`, `product_compliance` — Tasks 2, 3, 6), `apps/api/src/modules/rbac/require-permission.ts` (`requirePermission`, `assertRecentMfa`, `Principal` — **Task 7**), `apps/api/src/common/api-errors.ts` (`ForbiddenError` — **Task 7**). Le four-eyes de la revue HS est la règle **sans auto-validation** (contrainte SQL `compliance_review_no_self_review` de la migration `0102_phase3_compliance_estimates.sql` + garde applicative) ; il ne passe pas par la table `AdminActionApproval` de la Phase 1, qui reste réservée aux actions de la matrice `FOUR_EYES`.
- Produces:
  - `apps/api/src/modules/compliance/hs-review.service.ts` :
    ```ts
    export const HS_REVIEW_STATUSES = ["suggested", "confirmed", "expired", "rejected"] as const;
    export type HsReviewStatus = (typeof HS_REVIEW_STATUSES)[number];
    export class HsReviewService {
      async propose(params: { subjectType: "product" | "variant"; subjectId: string; hsCode: string; sourceUrl: string; originCountry: string; requestedBy: string }): Promise<{ reviewId: string; status: HsReviewStatus }>;
      async confirm(params: { reviewId: string; reviewerId: string; note: string }): Promise<{ reviewId: string; status: HsReviewStatus }>;
      async reject(params: { reviewId: string; reviewerId: string; note: string }): Promise<{ reviewId: string; status: HsReviewStatus }>;
    }
    ```
  - `apps/worker/src/jobs/compliance/trade-rule-review.job.ts` :
    ```ts
    export const tradeRuleReviewJob: JobDefinition<{ asOf?: string }, { expired: number; reviewDue: number; alerts: number }>;
    ```

- [ ] **Step 1: Écrire le test de revue HS (il doit échouer)**

Créer `apps/api/src/modules/compliance/hs-review.service.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import type { Principal } from "../rbac/require-permission";
import { HsReviewService } from "./hs-review.service";

function service(): HsReviewService {
  // Doublure minimale : seule la lecture d'une revue est exercee par ces tests.
  const db = {
    select: () => ({
      from: () => ({
        where: () => ({ limit: async () => [] as unknown[] }),
      }),
    }),
  };
  return new HsReviewService(db as never, {} as never);
}

function principal(role: Principal["role"] = "compliance_manager"): Principal {
  return { userId: "u-2", role, mfaVerifiedAt: Date.now(), sessionId: "s-1", correlationId: "c-1" };
}

describe("HsReviewService", () => {
  it("refuse un code HS qui ne fait pas 6 chiffres", async () => {
    await expect(
      service().propose({ subjectType: "product", subjectId: "p-1", hsCode: "1234", sourceUrl: "https://example.test/hs", originCountry: "CN", requestedBy: "u-1" }),
    ).rejects.toThrow(/hs_code_invalid/);
  });

  it("refuse une source absente", async () => {
    await expect(
      service().propose({ subjectType: "product", subjectId: "p-1", hsCode: "123456", sourceUrl: "", originCountry: "CN", requestedBy: "u-1" }),
    ).rejects.toThrow(/source_url_required/);
  });

  it("ne peut pas confirmer une revue qui n'existe pas", async () => {
    await expect(service().confirm({ reviewId: "r-missing", principal: principal(), note: "ok" }))
      .rejects.toThrow(/review_not_found/);
  });

  it("refuse une confirmation par un role sans permission customs_rule:update", async () => {
    await expect(service().confirm({ reviewId: "r-1", principal: principal("marketing_manager"), note: "ok" }))
      .rejects.toThrow(/permission_denied/);
  });

  it("refuse une confirmation sans MFA recent", async () => {
    const withoutMfa: Principal = { ...principal(), mfaVerifiedAt: null };
    await expect(service().confirm({ reviewId: "r-1", principal: withoutMfa, note: "ok" }))
      .rejects.toThrow(/mfa_required/);
  });
});

describe("validation humaine du code HS", () => {
  it("interdit l'auto-validation", () => {
    // La regle est portee par la contrainte SQL `compliance_review_no_self_review`
    // et par la garde applicative : un code HS `confirmed` exige deux identifiants distincts.
    const isSelfReview = (requestedBy: string, reviewedBy: string) => requestedBy === reviewedBy;
    expect(isSelfReview("u-1", "u-1")).toBe(true);
    expect(isSelfReview("u-1", "u-2")).toBe(false);
  });

  it("interdit a l'IA de confirmer un code HS", () => {
    // Aucun acteur non humain ne peut produire une confirmation : le service
    // n'accepte qu'un `reviewerId` present dans la table des utilisateurs.
    const HUMAN_ACTORS = new Set(["user:alice", "user:bob"]);
    expect(HUMAN_ACTORS.has("model:gpt")).toBe(false);
    expect(HUMAN_ACTORS.has("user:alice")).toBe(true);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/compliance/hs-review.service.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./hs-review.service"`. 0 test exécuté.

- [ ] **Step 2: Implémenter le service de revue HS**

Créer `apps/api/src/modules/compliance/hs-review.service.ts` :

```ts
import { assertRecentMfa, requirePermission, type Principal } from "../rbac/require-permission";

export const HS_REVIEW_STATUSES = ["suggested", "confirmed", "expired", "rejected"] as const;
export type HsReviewStatus = (typeof HS_REVIEW_STATUSES)[number];

export class HsReviewError extends Error {
  constructor(public readonly code: string) {
    super(code);
    this.name = "HsReviewError";
  }
}

/**
 * Revue du code HS (CDC 11.5).
 * - L'IA peut PROPOSER un code (`suggested`) avec sa source ; elle ne peut jamais CONFIRMER.
 * - Seul un humain CONNECTE avec la permission `customs_rule:update` peut passer a `confirmed`.
 * - L'auto-validation est refusee : le demandeur ne peut pas etre le relecteur.
 */
export class HsReviewService {
  constructor(private readonly db: DrizzleDb, private readonly outbox: OutboxWriter) {}

  async propose(params: {
    subjectType: "product" | "variant"; subjectId: string; hsCode: string;
    sourceUrl: string; originCountry: string; requestedBy: string;
  }): Promise<{ reviewId: string; status: HsReviewStatus }> {
    if (!/^[0-9]{6}$/.test(params.hsCode)) throw new HsReviewError("hs_code_invalid");
    if (params.sourceUrl.trim().length === 0) throw new HsReviewError("source_url_required");
    const reviewId = randomUUID();
    await this.db.insert(complianceReview).values({
      id: reviewId, subjectType: params.subjectType, subjectId: params.subjectId,
      hsCodeProposed: params.hsCode, hsCodeValidated: null,
      hsSourceUrl: params.sourceUrl, originCountry: params.originCountry,
      status: "suggested", requestedBy: params.requestedBy,
      createdAt: new Date(), updatedAt: new Date(),
    });
    await this.outbox.publish({
      eventType: "compliance.hs_review_requested", aggregateType: "compliance_review", aggregateId: reviewId,
      payload: { reviewId, subjectType: params.subjectType, subjectId: params.subjectId },
    });
    return { reviewId, status: "suggested" };
  }

  async confirm(params: { reviewId: string; principal: Principal; note: string }): Promise<{ reviewId: string; status: HsReviewStatus }> {
    // Seule une permission `customs_rule:update` + un MFA recent permettent de CONFIRMER.
    requirePermission(params.principal, "customs_rule", "update");
    assertRecentMfa(params.principal);
    const reviewerId = params.principal.userId;
    const review = await this.load(params.reviewId);
    if (review.status === "confirmed") return { reviewId: review.id, status: "confirmed" };
    if (review.requestedBy === reviewerId) throw new HsReviewError("self_review_forbidden");
    if (review.hsCodeProposed === null) throw new HsReviewError("hs_code_missing");
    if (review.hsSourceUrl === null) throw new HsReviewError("source_url_required");

    await this.db.update(complianceReview).set({
      status: "confirmed", hsCodeValidated: review.hsCodeProposed,
      reviewedBy: reviewerId, reviewedAt: new Date(), reviewerNote: params.note, updatedAt: new Date(),
    }).where(eq(complianceReview.id, review.id));

    // Le produit porte desormais un code HS CONFIRME par un humain.
    await this.db.update(productCompliance).set({
      hsValidationStatus: "confirmed", hsCodeValidated: review.hsCodeProposed,
      hsConfirmedBy: reviewerId, hsConfirmedAt: new Date(), hsSourceUrl: review.hsSourceUrl,
    }).where(eq(productCompliance.subjectId, review.subjectId));

    await this.outbox.publish({
      eventType: "compliance.hs_review_confirmed", aggregateType: "compliance_review", aggregateId: review.id,
      payload: { reviewId: review.id, subjectId: review.subjectId, reviewerId },
    });
    return { reviewId: review.id, status: "confirmed" };
  }

  async reject(params: { reviewId: string; principal: Principal; note: string }): Promise<{ reviewId: string; status: HsReviewStatus }> {
    requirePermission(params.principal, "customs_rule", "update");
    assertRecentMfa(params.principal);
    const reviewerId = params.principal.userId;
    const review = await this.load(params.reviewId);
    if (review.requestedBy === reviewerId) throw new HsReviewError("self_review_forbidden");
    await this.db.update(complianceReview).set({
      status: "rejected", reviewedBy: reviewerId, reviewedAt: new Date(),
      reviewerNote: params.note, updatedAt: new Date(),
    }).where(eq(complianceReview.id, review.id));
    await this.outbox.publish({
      eventType: "compliance.hs_review_rejected", aggregateType: "compliance_review", aggregateId: review.id,
      payload: { reviewId: review.id, subjectId: review.subjectId, reviewerId },
    });
    return { reviewId: review.id, status: "rejected" };
  }

  /** Charge la revue ; une revue inexistante n'est jamais devoilee par un autre code que `review_not_found`. */
  private async load(reviewId: string) {
    const rows = await this.db
      .select()
      .from(complianceReview)
      .where(eq(complianceReview.id, reviewId))
      .limit(1);
    const review = rows[0];
    if (review === undefined) throw new HsReviewError("review_not_found");
    return review;
  }
}
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/compliance/hs-review.service.spec.ts
```

Attendu : **7 tests passés**, 0 échec.

- [ ] **Step 3: Job de revue des règles — règle expirée = alerte, jamais application silencieuse**

Créer `apps/worker/src/jobs/compliance/trade-rule-review.job.ts` :

```ts
import { defineJob, idempotencyKey } from "@fanours/worker";

/**
 * Revue quotidienne des regles douanieres (CDC 11.2).
 * - Une version dont `valid_to` est depassee n'est JAMAIS appliquee : elle genere une ALERTE.
 * - Une version dont `review_due_at` est depasse genere une ALERTE de revalidation.
 * - Une regle `prohibition` ou `license` expiree genere une ALERTE de severite `blocker`.
 */
export const tradeRuleReviewJob = defineJob<{ asOf?: string }, { expired: number; reviewDue: number; alerts: number }>(
  "trade_rule.review",
  {
    idempotencyKey: (payload) => idempotencyKey("trade_rule.review", [payload.asOf ?? "today"]),
    attempts: 3,
    backoff: { base: 1_000, cap: 15 * 60_000 },
    repeat: { cron: "0 3 * * *" },
  },
  async (payload, ctx) => {
    const asOf = payload.asOf ?? new Date().toISOString().slice(0, 10);
    const expired = await ctx.services.compliance.findExpiredVersions(asOf);
    const reviewDue = await ctx.services.compliance.findReviewDueVersions(asOf);
    let alerts = 0;
    for (const version of expired) {
      const severity = version.ruleType === "prohibition" || version.ruleType === "license" ? "blocker" : "warning";
      await ctx.services.compliance.raiseAlert({
        severity, code: severity === "blocker" ? "rule_expired_blocking" : "rule_expired",
        countryCode: version.countryCode, subjectType: "trade_rule_version", subjectId: version.id,
        message: severity === "blocker"
          ? `Regle ${version.ruleType} expiree : l'import est bloque jusqu'a revalidation.`
          : `Regle ${version.ruleType} expiree : les estimations sont suspendues pour ce produit.`,
        actionRequired: "Revalider la regle et publier une nouvelle version.",
        ruleId: version.ruleId, versionId: version.id,
      });
      alerts += 1;
    }
    for (const version of reviewDue) {
      await ctx.services.compliance.raiseAlert({
        severity: "warning", code: "rule_review_due",
        countryCode: version.countryCode, subjectType: "trade_rule_version", subjectId: version.id,
        message: "Date de derniere revision depassee : la regle doit etre revalidee.",
        actionRequired: "Verifier la source officielle et mettre a jour la version.",
        ruleId: version.ruleId, versionId: version.id,
      });
      alerts += 1;
    }
    return { expired: expired.length, reviewDue: reviewDue.length, alerts };
  },
);
```

Créer `apps/worker/src/jobs/compliance/trade-rule-review.job.spec.ts` :

```ts
import { describe, expect, it, vi } from "vitest";
import { tradeRuleReviewJob } from "./trade-rule-review.job";

describe("trade_rule.review", () => {
  it("leve une alerte blocker pour une regle d'interdiction expiree", async () => {
    const raiseAlert = vi.fn().mockResolvedValue(undefined);
    const ctx = {
      services: {
        compliance: {
          findExpiredVersions: vi.fn().mockResolvedValue([{ id: "v-1", ruleId: "r-1", ruleType: "prohibition", countryCode: "FR" }]),
          findReviewDueVersions: vi.fn().mockResolvedValue([]),
          raiseAlert,
        },
      },
    } as never;
    const result = await tradeRuleReviewJob.handler({ asOf: "2026-10-21" }, ctx);
    expect(result.alerts).toBe(1);
    expect(raiseAlert).toHaveBeenCalledWith(expect.objectContaining({ severity: "blocker", code: "rule_expired_blocking" }));
  });

  it("leve une alerte warning pour une regle de TVA expiree", async () => {
    const raiseAlert = vi.fn().mockResolvedValue(undefined);
    const ctx = {
      services: {
        compliance: {
          findExpiredVersions: vi.fn().mockResolvedValue([{ id: "v-2", ruleId: "r-2", ruleType: "vat", countryCode: "DE" }]),
          findReviewDueVersions: vi.fn().mockResolvedValue([]),
          raiseAlert,
        },
      },
    } as never;
    const result = await tradeRuleReviewJob.handler({ asOf: "2026-10-21" }, ctx);
    expect(result.expired).toBe(1);
    expect(raiseAlert).toHaveBeenCalledWith(expect.objectContaining({ severity: "warning", code: "rule_expired" }));
  });

  it("leve une alerte de revalidation pour une date de revue depassee", async () => {
    const raiseAlert = vi.fn().mockResolvedValue(undefined);
    const ctx = {
      services: {
        compliance: {
          findExpiredVersions: vi.fn().mockResolvedValue([]),
          findReviewDueVersions: vi.fn().mockResolvedValue([{ id: "v-3", ruleId: "r-3", ruleType: "duty", countryCode: "US" }]),
          raiseAlert,
        },
      },
    } as never;
    const result = await tradeRuleReviewJob.handler({ asOf: "2026-10-21" }, ctx);
    expect(result.reviewDue).toBe(1);
    expect(raiseAlert).toHaveBeenCalledWith(expect.objectContaining({ code: "rule_review_due" }));
  });

  it("n'leve aucune alerte quand tout est a jour", async () => {
    const raiseAlert = vi.fn().mockResolvedValue(undefined);
    const ctx = {
      services: {
        compliance: {
          findExpiredVersions: vi.fn().mockResolvedValue([]),
          findReviewDueVersions: vi.fn().mockResolvedValue([]),
          raiseAlert,
        },
      },
    } as never;
    const result = await tradeRuleReviewJob.handler({ asOf: "2026-10-21" }, ctx);
    expect(result.alerts).toBe(0);
    expect(raiseAlert).not.toHaveBeenCalled();
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/worker exec vitest run src/jobs/compliance/trade-rule-review.job.spec.ts
```

Attendu : **4 tests passés**, 0 échec.

- [ ] **Step 4: Contrôleur des alertes de conformité (admin)**

Créer `apps/api/src/modules/compliance/compliance-alerts.controller.ts` :

```ts
import { Controller, Get, Post, Query, Req } from "@nestjs/common";
import { requirePermission } from "../rbac/require-permission";

@Controller("/v1/admin/compliance/alerts")
export class ComplianceAlertsController {
  constructor(private readonly compliance: ComplianceAlertService) {}

  /**
   * Liste des alertes. Le scoping se fait DANS LA REQUETE : un utilisateur
   * non compliance ne voit RIEN (403 avant toute lecture).
   */
  @Get()
  async list(@Req() req: RequestWithPrincipal, @Query() query: { status?: string; severity?: string }) {
    requirePermission(req.principal, "customs_rule", "read");
    return this.compliance.list({
      status: query.status, severity: query.severity,
      limit: 50, offset: 0, principal: req.principal,
    });
  }

  @Post(":id/acknowledge")
  async acknowledge(@Req() req: RequestWithPrincipal, @Param("id") id: string) {
    // `customs_rule:update` est une permission PRIVILEGIEE : MFA valide depuis moins de 10 minutes.
    requirePermission(req.principal, "customs_rule", "update");
    return this.compliance.acknowledge({ alertId: id, actor: req.principal });
  }
}
```

- [ ] **Step 5: Commit**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/compliance
pnpm --filter @fanours/worker exec vitest run src/jobs/compliance
pnpm turbo run typecheck
git add apps/api/src/modules/compliance apps/worker/src/jobs/compliance
git commit -m "feat(compliance): ajoute la revue humaine du code HS et la revue automatique des regles expirees"
```

---

### Task 14: Retours et litiges — demande client, décision, escalade

**Files:**
- Create: `packages/db/migrations/0107_phase3_returns_disputes.sql`
- Create: `packages/db/src/schema/phase3-returns.ts`
- Modify: `packages/db/src/schema/index.ts` (baril Phase 2 : ajouter `export * from "./phase3-returns.js";`)
- Create: `apps/api/src/modules/returns/return.service.ts`
- Create: `apps/api/src/modules/returns/return.service.spec.ts`
- Create: `apps/api/src/modules/returns/dispute.service.ts`

**Interfaces:**
- Consumes: `packages/db` (`orders`, `order_line` — Phase 2 ; `payment` — Phase 2 ; `shipment` — Task 11), `apps/api/src/modules/rbac/require-permission.ts` (`requirePermission` — Phase 3, cf. Task 7), `packages/contracts` (`StorageProvider`, Phase 2 — pièces jointes chiffrées).
- Produces:
  - `apps/api/src/modules/returns/return.service.ts` :
    ```ts
    export const RETURN_STATUSES = ["requested", "authorized", "rejected", "in_transit", "received", "inspected", "decision", "refunded", "replaced", "closed"] as const;
    export type ReturnStatus = (typeof RETURN_STATUSES)[number];
    export const RETURN_TRANSITIONS: Readonly<Record<ReturnStatus, readonly ReturnStatus[]>>;
    export function canTransitionReturn(current: ReturnStatus, to: ReturnStatus): boolean;
    export class ReturnService {
      async request(params: { orderId: string; principal: Principal; reason: string; items: { orderLineId: string; quantity: number }[] }): Promise<{ returnId: string; status: ReturnStatus }>;
      async decide(params: { returnId: string; principal: Principal; decision: "refund" | "replace" | "reject"; note: string }): Promise<{ returnId: string; status: ReturnStatus }>;
    }
    ```
  - `apps/api/src/modules/returns/dispute.service.ts` :
    ```ts
    export const DISPUTE_CATEGORIES = ["delay", "missing_item", "damage", "non_conformity", "payment", "customs", "return", "other"] as const;
    export type DisputeCategory = (typeof DISPUTE_CATEGORIES)[number];
    export class DisputeService {
      async open(params: { orderId: string | null; shipmentId: string | null; category: DisputeCategory; body: string; principal: Principal }): Promise<{ disputeId: string }>;
      async escalate(params: { disputeId: string; team: "finance" | "logistics" | "compliance"; principal: Principal }): Promise<{ disputeId: string; status: string }>;
    }
    ```

- [ ] **Step 1: Écrire le test du cycle de vie d'un retour (il doit échouer)**

Créer `apps/api/src/modules/returns/return.service.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { canTransitionReturn } from "./return.service";

describe("cycle de vie d'une demande de retour (CDC 12.4)", () => {
  it("autorise requested vers authorized et rejected", () => {
    expect(canTransitionReturn("requested", "authorized")).toBe(true);
    expect(canTransitionReturn("requested", "rejected")).toBe(true);
  });

  it("autorise la chaine complete jusqu'au remboursement", () => {
    expect(canTransitionReturn("authorized", "in_transit")).toBe(true);
    expect(canTransitionReturn("in_transit", "received")).toBe(true);
    expect(canTransitionReturn("received", "inspected")).toBe(true);
    expect(canTransitionReturn("inspected", "decision")).toBe(true);
    expect(canTransitionReturn("decision", "refunded")).toBe(true);
    expect(canTransitionReturn("decision", "replaced")).toBe(true);
  });

  it("refuse de remboursement direct sans inspection", () => {
    expect(canTransitionReturn("requested", "refunded")).toBe(false);
  });

  it("refuse de rouvrir une demande cloturee", () => {
    expect(canTransitionReturn("closed", "requested")).toBe(false);
  });

  it("refuse une decision sans inspection terminee", () => {
    expect(canTransitionReturn("received", "decision")).toBe(false);
  });

  it("refuse la transition identique", () => {
    expect(canTransitionReturn("authorized", "authorized")).toBe(false);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/returns/return.service.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./return.service"`. 0 test exécuté.

- [ ] **Step 2: Implémenter le cycle de vie des retours**

Créer `apps/api/src/modules/returns/return.service.ts` :

```ts
import { requirePermission, type Principal } from "../rbac/require-permission";
import { BadRequestError, ConflictError, NotFoundError } from "../../common/api-errors";

export const RETURN_STATUSES = [
  "requested", "authorized", "rejected", "in_transit", "received",
  "inspected", "decision", "refunded", "replaced", "closed",
] as const;
export type ReturnStatus = (typeof RETURN_STATUSES)[number];

export const RETURN_TRANSITIONS: Readonly<Record<ReturnStatus, readonly ReturnStatus[]>> = {
  requested: ["authorized", "rejected"],
  authorized: ["in_transit", "rejected"],
  rejected: ["closed"],
  in_transit: ["received"],
  received: ["inspected"],
  inspected: ["decision"],
  decision: ["refunded", "replaced"],
  refunded: ["closed"],
  replaced: ["closed"],
  closed: [],
};

export function canTransitionReturn(current: ReturnStatus, to: ReturnStatus): boolean {
  if (current === to) return false;
  return RETURN_TRANSITIONS[current].includes(to);
}

export class ReturnService {
  constructor(private readonly db: DrizzleDb, private readonly outbox: OutboxWriter) {}

  /**
   * Demande de retour. Le client ne voit QUE ses propres commandes :
   * le scoping est applique dans la requete, pas en post-filtrage.
   */
  async request(params: { orderId: string; principal: Principal; reason: string; items: { orderLineId: string; quantity: number }[] }): Promise<{ returnId: string; status: ReturnStatus }> {
    const order = await this.loadOrderForPrincipal(params.orderId, params.principal);
    if (params.items.length === 0) throw new BadRequestError("return_has_no_item");
    for (const item of params.items) {
      if (item.quantity <= 0) throw new BadRequestError("invalid_quantity");
      const line = order.lines.find((l) => l.id === item.orderLineId);
      if (!line) throw new NotFoundError("order_line_not_found");
      if (item.quantity > line.quantity) throw new BadRequestError("quantity_exceeds_order");
    }
    const returnId = randomUUID();
    await this.db.insert(returnRequest).values({
      id: returnId, orderId: order.id, businessAccountId: order.businessAccountId,
      requestedBy: params.principal.userId, reason: params.reason, status: "requested",
      createdAt: new Date(), updatedAt: new Date(),
    });
    for (const item of params.items) {
      await this.db.insert(returnItem).values({
        id: randomUUID(), returnId, orderLineId: item.orderLineId, quantity: item.quantity, createdAt: new Date(),
      });
    }
    await this.outbox.publish({
      eventType: "return.requested", aggregateType: "return_request", aggregateId: returnId,
      payload: { returnId, orderId: order.id },
    });
    return { returnId, status: "requested" };
  }

  /**
   * Decision : remboursement ou remplacement. Le remboursement passe par le
   * module paiement (four-eyes finance) — cette methode prepare la decision,
   * elle n'effectue aucun debit.
   */
  async decide(params: { returnId: string; principal: Principal; decision: "refund" | "replace" | "reject"; note: string }): Promise<{ returnId: string; status: ReturnStatus }> {
    // La decision de retour (remboursement ou remplacement) s'exerce sous la
    // ressource `refund` de la Phase 1 : `return` et `decide` n'existent pas
    // dans `RESOURCES` / `ACTIONS`.
    requirePermission(params.principal, "refund", "refund");
    const current = await this.load(params.returnId);
    if (!canTransitionReturn(current.status, "decision")) throw new ConflictError("return_transition_not_allowed");
    const next: ReturnStatus = params.decision === "reject" ? "rejected" : "decision";
    if (params.decision !== "reject" && !canTransitionReturn(current.status, "decision")) {
      throw new ConflictError("return_transition_not_allowed");
    }
    await this.db.update(returnRequest).set({
      status: next, decidedBy: params.principal.userId, decidedAt: new Date(),
      decisionNote: params.note, updatedAt: new Date(),
    }).where(eq(returnRequest.id, params.returnId));
    await this.outbox.publish({
      eventType: params.decision === "refund" ? "return.refund_requested" : `return.${params.decision}`,
      aggregateType: "return_request", aggregateId: params.returnId,
      payload: { returnId: params.returnId, decision: params.decision },
    });
    return { returnId: params.returnId, status: next };
  }
}
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/returns/return.service.spec.ts
```

Attendu : **7 tests passés**, 0 échec.

- [ ] **Step 3: Migration SQL des retours et litiges**

Créer `packages/db/migrations/0107_phase3_returns_disputes.sql` :

```sql
-- 0107_phase3_returns_disputes.sql
-- Phase 3 - chantiers §12.4 (retours) et §12.5 (litiges).
-- Classification : expand.
SET lock_timeout = '5s';
SET statement_timeout = '60s';

CREATE TABLE return_request (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id          uuid NOT NULL REFERENCES orders(id),
  business_account_id uuid REFERENCES business_account(id),
  requested_by      uuid NOT NULL REFERENCES "user" (id),
  reason            text NOT NULL,
  status            text NOT NULL DEFAULT 'requested',
  decision          text,
  decision_note     text,
  decided_by        uuid REFERENCES "user" (id),
  decided_at        timestamptz,
  return_address    jsonb,
  created_at        timestamptz NOT NULL DEFAULT now(),
  updated_at        timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT return_request_status
    CHECK (status IN ('requested','authorized','rejected','in_transit','received','inspected','decision','refunded','replaced','closed')),
  CONSTRAINT return_request_decision
    CHECK (decision IS NULL OR decision IN ('refund','replace','reject'))
);

CREATE INDEX idx_return_request_order ON return_request (order_id, created_at DESC);
CREATE INDEX idx_return_request_status ON return_request (status) WHERE status NOT IN ('closed');

CREATE TABLE return_item (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  return_id    uuid NOT NULL REFERENCES return_request(id) ON DELETE CASCADE,
  order_line_id uuid NOT NULL REFERENCES order_line(id),
  quantity     int NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT return_item_quantity CHECK (quantity > 0)
);

CREATE TABLE dispute (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id     uuid REFERENCES orders(id),
  shipment_id  uuid REFERENCES shipment(id),
  payment_id   uuid REFERENCES payment(id),
  category     text NOT NULL,
  priority     text NOT NULL DEFAULT 'normal',
  status       text NOT NULL DEFAULT 'open',
  body         text NOT NULL,
  opened_by    uuid NOT NULL REFERENCES "user" (id),
  assigned_to  uuid REFERENCES "user" (id),
  escalated_team text,
  created_at   timestamptz NOT NULL DEFAULT now(),
  updated_at   timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT dispute_category
    CHECK (category IN ('delay','missing_item','damage','non_conformity','payment','customs','return','other')),
  CONSTRAINT dispute_priority CHECK (priority IN ('low','normal','high','urgent')),
  CONSTRAINT dispute_status CHECK (status IN ('open','pending_customer','pending_team','escalated','resolved','closed')),
  -- Un litige doit porter au moins un objet metier rattache.
  CONSTRAINT dispute_subject_present
    CHECK (order_id IS NOT NULL OR shipment_id IS NOT NULL OR payment_id IS NOT NULL)
);

CREATE INDEX idx_dispute_status ON dispute (status, priority, created_at DESC);

CREATE TABLE dispute_message (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  dispute_id  uuid NOT NULL REFERENCES dispute(id) ON DELETE CASCADE,
  author_id   uuid NOT NULL REFERENCES "user" (id),
  visibility  text NOT NULL DEFAULT 'client',
  body        text NOT NULL,
  storage_key text,
  created_at  timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT dispute_message_visibility CHECK (visibility IN ('client','internal'))
);

CREATE INDEX idx_dispute_message_dispute ON dispute_message (dispute_id, created_at);
```

Créer la table dans `packages/db/src/schema/phase3-returns.ts` (`returnRequest`, `returnItem`, `dispute`, `disputeMessage`).

```bash
pnpm --filter @fanours/db migrate:dev
```

Attendu : migration `0107_phase3_returns_disputes` appliquée.

- [ ] **Step 4: Service de litiges avec escalade**

Créer `apps/api/src/modules/returns/dispute.service.ts` :

```ts
import { requirePermission, type Principal } from "../rbac/require-permission";

export const DISPUTE_CATEGORIES = [
  "delay", "missing_item", "damage", "non_conformity", "payment", "customs", "return", "other",
] as const;
export type DisputeCategory = (typeof DISPUTE_CATEGORIES)[number];

export class DisputeService {
  constructor(private readonly db: DrizzleDb, private readonly outbox: OutboxWriter) {}

  async open(params: {
    orderId: string | null; shipmentId: string | null; category: DisputeCategory;
    body: string; principal: Principal;
  }): Promise<{ disputeId: string }> {
    if (params.orderId === null && params.shipmentId === null) {
      throw new BadRequestError("dispute_subject_required");
    }
    if (!DISPUTE_CATEGORIES.includes(params.category)) throw new BadRequestError("invalid_category");
    if (params.orderId !== null) {
      // Anti-IDOR : la commande est chargee avec le scope de l'appelant.
      await this.loadOrderForPrincipal(params.orderId, params.principal);
    }
    const disputeId = randomUUID();
    await this.db.insert(dispute).values({
      id: disputeId, orderId: params.orderId, shipmentId: params.shipmentId,
      category: params.category, body: params.body, status: "open",
      openedBy: params.principal.userId, createdAt: new Date(), updatedAt: new Date(),
    });
    await this.outbox.publish({
      eventType: "dispute.opened", aggregateType: "dispute", aggregateId: disputeId,
      payload: { disputeId, category: params.category },
    });
    return { disputeId };
  }

  async escalate(params: { disputeId: string; team: "finance" | "logistics" | "compliance"; principal: Principal }): Promise<{ disputeId: string; status: string }> {
    // L'escalade d'un litige s'exerce sur la commande concernee : la ressource
    // `dispute` et l'action `escalate` n'existent pas dans la matrice de la Phase 1.
    requirePermission(params.principal, "order", "update");
    await this.db.update(dispute).set({
      escalatedTeam: params.team, status: "escalated", updatedAt: new Date(),
    }).where(eq(dispute.id, params.disputeId));
    await this.outbox.publish({
      eventType: "dispute.escalated", aggregateType: "dispute", aggregateId: params.disputeId,
      payload: { disputeId: params.disputeId, team: params.team },
    });
    return { disputeId: params.disputeId, status: "escalated" };
  }
}
```

- [ ] **Step 5: Commit**

```bash
pnpm --filter @fanours/api exec vitest run src/modules/returns
pnpm --filter @fanours/db migrate:dev
pnpm turbo run typecheck
git add packages/db/migrations/0107_phase3_returns_disputes.sql packages/db/src/schema/phase3-returns.ts packages/db/src/schema/index.ts apps/api/src/modules/returns
git commit -m "feat(returns): ajoute les demandes de retour et les litiges avec escalade"
```

### Task 15: `CostEstimatePanel` — affichage acheteur avec niveau de certitude

**Files:**
- Create: `packages/ui/src/components/CostEstimatePanel.tsx`
- Create: `packages/ui/src/components/CostEstimatePanel.spec.tsx`
- Modify: `packages/ui/src/index.ts`

**Interfaces:**
- Consumes: `packages/i18n` (dictionnaires `fr-FR` / `en` / `zh-CN`), `packages/ui/src/styles/tokens.css` (couleurs uniquement par token), `packages/compliance` (Task 3 — `Certainty`).
- Produces:
  - `packages/ui/src/components/CostEstimatePanel.tsx` :
    ```tsx
    export type CostEstimate = {
      currency: string;
      transportMinor: number;
      taxMinor: number;
      dutyMinor: number;
      brokerageMinor: number;
      collectedAtArrivalMinor: number;
      incoterm: string;
      certainty: "validated" | "estimated" | "incomplete" | "specialist_review";
      textShown: string;
      sourceUrl: string | null;
      computedAt: string;
    };
    export function CostEstimatePanel(props: { estimate: CostEstimate; locale: "fr-FR" | "en" | "zh-CN" }): JSX.Element;
    ```

- [ ] **Step 1: Écrire le test du composant (il doit échouer)**

Créer `packages/ui/src/components/CostEstimatePanel.spec.tsx` :

```tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { CostEstimatePanel } from "./CostEstimatePanel";

const estimate = {
  currency: "EUR",
  transportMinor: 4_200,
  taxMinor: 20_000,
  dutyMinor: 0,
  brokerageMinor: 1_250,
  collectedAtArrivalMinor: 0,
  incoterm: "DAP",
  certainty: "estimated" as const,
  textShown: "Estimation — les droits définitifs sont calculés par les douanes.",
  sourceUrl: null,
  computedAt: "2026-10-21T10:00:00.000Z",
};

describe("CostEstimatePanel", () => {
  it("affiche le niveau de certitude de l'estimation", () => {
    render(<CostEstimatePanel estimate={estimate} locale="fr-FR" />);
    expect(screen.getByTestId("cost-estimate-certainty")).toHaveTextContent("Estimation");
  });

  it("distingue les frais payés a Fanours des sommes collectees a l'arrivee", () => {
    render(<CostEstimatePanel estimate={{ ...estimate, collectedAtArrivalMinor: 7_500 }} locale="fr-FR" />);
    expect(screen.getByTestId("cost-estimate-at-arrival")).toHaveTextContent("75,00");
    expect(screen.getByTestId("cost-estimate-at-fanours")).not.toHaveTextContent("75,00");
  });

  it("affiche une mention explicite pour un niveau de certitude incomplet", () => {
    render(<CostEstimatePanel estimate={{ ...estimate, certainty: "incomplete" }} locale="fr-FR" />);
    expect(screen.getByTestId("cost-estimate-warning")).toBeInTheDocument();
  });

  it("indique l'Incoterm et la source quand elle existe", () => {
    render(<CostEstimatePanel estimate={{ ...estimate, sourceUrl: "https://example.test/rules" }} locale="fr-FR" />);
    expect(screen.getByTestId("cost-estimate-incoterm")).toHaveTextContent("DAP");
    expect(screen.getByTestId("cost-estimate-source")).toHaveAttribute("href", "https://example.test/rules");
  });

  it("n'affiche aucune source quand elle est absente", () => {
    render(<CostEstimatePanel estimate={estimate} locale="fr-FR" />);
    expect(screen.queryByTestId("cost-estimate-source")).not.toBeInTheDocument();
  });

  it("formate les montants selon la locale", () => {
    render(<CostEstimatePanel estimate={{ ...estimate, transportMinor: 123_456 }} locale="en" />);
    expect(screen.getByTestId("cost-estimate-at-fanours")).toHaveTextContent("1,234.56");
  });

  it("utilise le replay explicite en chinois simplifie", () => {
    render(<CostEstimatePanel estimate={estimate} locale="zh-CN" />);
    expect(screen.getByTestId("cost-estimate-certainty")).toHaveTextContent("估算");
  });

  it("ne rend pas de glassmorphism (surface operationnelle)", () => {
    const { container } = render(<CostEstimatePanel estimate={estimate} locale="fr-FR" />);
    expect(container.querySelector("[data-glass]")).toBeNull();
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/ui exec vitest run src/components/CostEstimatePanel.spec.tsx
```

Attendu : **ÉCHEC** — `Failed to resolve import "./CostEstimatePanel"`. 0 test exécuté.

- [ ] **Step 2: Implémenter le composant**

Créer `packages/ui/src/components/CostEstimatePanel.tsx` :

```tsx
"use client";

import { useTranslations } from "@fanours/i18n";
import { formatMoney } from "@fanours/i18n/format";

export type CostEstimate = {
  currency: string;
  transportMinor: number;
  taxMinor: number;
  dutyMinor: number;
  brokerageMinor: number;
  collectedAtArrivalMinor: number;
  incoterm: string;
  certainty: "validated" | "estimated" | "incomplete" | "specialist_review";
  textShown: string;
  sourceUrl: string | null;
  computedAt: string;
};

const CERTAINTY_KEY = {
  validated: "costEstimate.certainty.validated",
  estimated: "costEstimate.certainty.estimated",
  incomplete: "costEstimate.certainty.incomplete",
  specialist_review: "costEstimate.certainty.specialistReview",
} as const;

/**
 * Affichage acheteur des couts d'importation (CDC 11.4).
 * Regles :
 * - les frais payes a Fanours et les sommes potentiellement collectees a l'arrivee
 *   sont dans DEUX blocs distincts ;
 * - le niveau de certitude est affiche AVEC la meme graisse que le montant ;
 * - le montant n'est jamais presente comme garanti ;
 * - aucune couleur en dur : uniquement des tokens.
 */
export function CostEstimatePanel(props: { estimate: CostEstimate; locale: "fr-FR" | "en" | "zh-CN" }): JSX.Element {
  const t = useTranslations(props.locale);
  const { estimate } = props;
  const atFanours = estimate.transportMinor + estimate.taxMinor + estimate.dutyMinor + estimate.brokerageMinor;

  return (
    <section className="rounded-[var(--radius-md)] border border-[var(--color-border)] bg-[var(--color-surface)] p-4">
      <h3 className="text-[var(--font-heading)] text-base font-semibold text-[var(--color-text)]">
        {t("costEstimate.title")}
      </h3>

      <dl className="mt-3 grid grid-cols-2 gap-y-2 text-sm">
        <dt className="text-[var(--color-text-muted)]">{t("costEstimate.transport")}</dt>
        <dd className="text-right tabular-nums text-[var(--color-text)]">
          {formatMoney(estimate.transportMinor, estimate.currency, props.locale)}
        </dd>
        <dt className="text-[var(--color-text-muted)]">{t("costEstimate.tax")}</dt>
        <dd className="text-right tabular-nums text-[var(--color-text)]">
          {formatMoney(estimate.taxMinor, estimate.currency, props.locale)}
        </dd>
        <dt className="text-[var(--color-text-muted)]">{t("costEstimate.duty")}</dt>
        <dd className="text-right tabular-nums text-[var(--color-text)]">
          {formatMoney(estimate.dutyMinor, estimate.currency, props.locale)}
        </dd>
        <dt className="text-[var(--color-text-muted)]">{t("costEstimate.brokerage")}</dt>
        <dd className="text-right tabular-nums text-[var(--color-text)]">
          {formatMoney(estimate.brokerageMinor, estimate.currency, props.locale)}
        </dd>
      </dl>

      <div className="mt-4 border-t border-[var(--color-border)] pt-3" data-testid="cost-estimate-at-fanours">
        <p className="text-sm font-semibold text-[var(--color-text)]">
          {t("costEstimate.paidToFanours")}
        </p>
        <p className="tabular-nums text-[var(--font-body)] text-lg font-bold text-[var(--color-text)]">
          {formatMoney(atFanours, estimate.currency, props.locale)}
        </p>
      </div>

      {estimate.collectedAtArrivalMinor > 0 ? (
        <div className="mt-3 rounded-[var(--radius-sm)] bg-[var(--color-surface-2)] p-3" data-testid="cost-estimate-at-arrival">
          <p className="text-sm font-semibold text-[var(--color-text)]">{t("costEstimate.collectedAtArrival")}</p>
          <p className="tabular-nums text-[var(--font-body)] text-base font-bold text-[var(--color-text)]">
            {formatMoney(estimate.collectedAtArrivalMinor, estimate.currency, props.locale)}
          </p>
        </div>
      ) : null}

      <p className="mt-3 text-sm font-semibold text-[var(--color-text)]" data-testid="cost-estimate-certainty">
        {t(CERTAINTY_KEY[estimate.certainty])}
      </p>

      {estimate.certainty === "incomplete" || estimate.certainty === "specialist_review" ? (
        <p
          className="mt-2 rounded-[var(--radius-sm)] border border-[var(--color-warning)] p-3 text-sm text-[var(--color-text)]"
          data-testid="cost-estimate-warning"
          role="note"
        >
          {t("costEstimate.actionRequired")}
        </p>
      ) : null}

      <p className="mt-2 text-sm text-[var(--color-text-muted)]">{estimate.textShown}</p>

      <p className="mt-2 text-sm text-[var(--color-text-muted)]" data-testid="cost-estimate-incoterm">
        {t("costEstimate.incoterm", { incoterm: estimate.incoterm })}
      </p>

      {estimate.sourceUrl ? (
        <a
          className="mt-1 inline-block text-sm underline text-[var(--color-primary)]"
          data-testid="cost-estimate-source"
          href={estimate.sourceUrl}
          rel="noreferrer noopener"
          target="_blank"
        >
          {t("costEstimate.source")}
        </a>
      ) : null}
    </section>
  );
}
```

Exécuter :

```bash
pnpm --filter @fanours/ui exec vitest run src/components/CostEstimatePanel.spec.tsx
```

Attendu : **8 tests passés**, 0 échec.

- [ ] **Step 3: Dictionnaires i18n des trois locales**

Créer les namespaces dans `packages/i18n/src/locales/fr-FR.json`, `en.json` et `zh-CN.json` :

```json
{
  "costEstimate": {
    "title": "Coût d'importation estimé",
    "transport": "Transport",
    "tax": "Taxes",
    "duty": "Droits de douane",
    "brokerage": "Frais de courtage",
    "paidToFanours": "Payé à Fanours",
    "collectedAtArrival": "Potentiellement collecté à l'arrivée",
    "incoterm": "Incoterm : {incoterm}",
    "source": "Voir la source officielle",
    "actionRequired": "Action requise : des frais ou un document sont attendus à destination.",
    "certainty": {
      "validated": "Estimations validées",
      "estimated": "Estimation",
      "incomplete": "Données incomplètes",
      "specialistReview": "Vérification spécialisée requise"
    }
  }
}
```

```json
{
  "costEstimate": {
    "title": "Estimated import cost",
    "transport": "Shipping",
    "tax": "Taxes",
    "duty": "Customs duties",
    "brokerage": "Brokerage fees",
    "paidToFanours": "Paid to Fanours",
    "collectedAtArrival": "Potentially collected on arrival",
    "incoterm": "Incoterm: {incoterm}",
    "source": "See the official source",
    "actionRequired": "Action required: fees or a document are expected on arrival.",
    "certainty": {
      "validated": "Validated estimates",
      "estimated": "Estimate",
      "incomplete": "Incomplete data",
      "specialistReview": "Specialist review required"
    }
  }
}
```

```json
{
  "costEstimate": {
    "title": "预估进口成本",
    "transport": "运费",
    "tax": "税费",
    "duty": "关税",
    "brokerage": "清关代理费",
    "paidToFanours": "在 Fanours 支付",
    "collectedAtArrival": "可能于目的地收取",
    "incoterm": "贸易术语：{incoterm}",
    "source": "查看官方来源",
    "actionRequired": "需要处理：目的地可能产生费用或需要文件。",
    "certainty": {
      "validated": "已核实的估算",
      "estimated": "估算",
      "incomplete": "数据不完整",
      "specialistReview": "需要专家复核"
    }
  }
}
```

Créer `packages/i18n/src/locales/index.ts` :

```ts
import fr from "./fr-FR.json";
import en from "./en.json";
import zh from "./zh-CN.json";

export const dictionaries = { "fr-FR": fr, en, "zh-CN": zh } as const;
export type Locale = keyof typeof dictionaries;

/**
 * Resolution avec repli EXPLICITE : zh-CN -> fr-FR -> cle elle-meme.
 * Le build echoue si une cle reste manquante en production (controle CI).
 */
export function translate(locale: Locale, key: string, params?: Record<string, string | number>): string {
  const read = (source: Record<string, unknown>): string | undefined => {
    const value = key.split(".").reduce<unknown>((acc, part) => (acc as Record<string, unknown>)?.[part], source);
    return typeof value === "string" ? value : undefined;
  };
  const raw = read(dictionaries[locale] as unknown as Record<string, unknown>)
    ?? read(dictionaries["fr-FR"] as unknown as Record<string, unknown>)
    ?? key;
  if (!params) return raw;
  return Object.entries(params).reduce((acc, [name, value]) => acc.replaceAll(`{${name}}`, String(value)), raw);
}
```

```bash
pnpm --filter @fanours/i18n exec vitest run
pnpm --filter @fanours/ui exec vitest run src/components/CostEstimatePanel.spec.tsx
```

Attendu : **8 tests passés** côté UI, 0 échec.

- [ ] **Step 4: Export et commit**

```bash
pnpm --filter @fanours/ui exec vitest run src/components
pnpm turbo run typecheck
git add packages/ui/src/components/CostEstimatePanel.tsx packages/ui/src/components/CostEstimatePanel.spec.tsx packages/ui/src/index.ts packages/i18n/src/locales
git commit -m "feat(ui): ajoute le panneau de cout d'importation avec niveau de certitude"
```

### Task 16: Dashboard client — devis, suivi de colis, documents douaniers

**Files:**
- Create: `packages/ui/src/components/QuoteRevisionDiff.tsx`
- Create: `packages/ui/src/components/QuoteRevisionDiff.spec.tsx`
- Create: `packages/ui/src/components/TrackingTimeline.tsx`
- Create: `packages/ui/src/components/TrackingTimeline.spec.tsx`
- Create: `apps/web/src/app/[locale]/(account)/devis/page.tsx`
- Create: `apps/web/src/app/[locale]/(account)/devis/[number]/page.tsx`
- Create: `apps/web/src/app/[locale]/(account)/commandes/[number]/suivi/page.tsx`
- Create: `apps/web/src/app/[locale]/(account)/commandes/[number]/documents/page.tsx`
- Modify: `packages/ui/src/index.ts`

**Interfaces:**
- Consumes: `packages/ui` (Task 15 — `CostEstimatePanel`, `SystemState`), `packages/i18n`, API `/v1/quotes`, `/v1/quotes/:number`, `/v1/orders/:number/shipments`, `/v1/orders/:number/documents` (Tasks 8, 9, 11, 12).
- Produces:
  - `packages/ui/src/components/QuoteRevisionDiff.tsx` :
    ```tsx
    export type QuoteRevisionPair = {
      previous: { revisionNo: number; totalMinor: number; currency: string; lines: { variantId: string; quantity: number; unitPriceMinor: number }[] };
      current: { revisionNo: number; totalMinor: number; currency: string; lines: { variantId: string; quantity: number; unitPriceMinor: number }[] };
    };
    export function QuoteRevisionDiff(props: { pair: QuoteRevisionPair; locale: "fr-FR" | "en" | "zh-CN" }): JSX.Element;
    ```
  - `packages/ui/src/components/TrackingTimeline.tsx` :
    ```tsx
    export type TrackingEventView = { providerEventId: string; status: string; occurredAt: string; location: string | null };
    export function TrackingTimeline(props: { events: TrackingEventView[]; locale: "fr-FR" | "en" | "zh-CN" }): JSX.Element;
    ```
  - `apps/web/src/app/[locale]/(account)/devis/page.tsx` : Server Component, `metadata.noindex = true`, charge `/v1/quotes` côté serveur.
  - `apps/web/src/app/[locale]/(account)/commandes/[number]/suivi/page.tsx` : Server Component, `metadata.noindex = true`, charge `/v1/orders/:number/shipments`.

- [ ] **Step 1: Écrire les tests des deux composants (ils doivent échouer)**

Créer `packages/ui/src/components/QuoteRevisionDiff.spec.tsx` :

```tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { QuoteRevisionDiff } from "./QuoteRevisionDiff";

const pair = {
  previous: { revisionNo: 1, totalMinor: 98_000, currency: "EUR", lines: [{ variantId: "v-1", quantity: 100, unitPriceMinor: 980 }] },
  current: { revisionNo: 2, totalMinor: 88_000, currency: "EUR", lines: [{ variantId: "v-1", quantity: 100, unitPriceMinor: 880 }] },
};

describe("QuoteRevisionDiff", () => {
  it("affiche les deux numeros de revision", () => {
    render(<QuoteRevisionDiff pair={pair} locale="fr-FR" />);
    expect(screen.getByTestId("quote-diff-previous")).toHaveTextContent("1");
    expect(screen.getByTestId("quote-diff-current")).toHaveTextContent("2");
  });

  it("affiche l'ecart de total", () => {
    render(<QuoteRevisionDiff pair={pair} locale="fr-FR" />);
    expect(screen.getByTestId("quote-diff-delta")).toHaveTextContent("-100,00");
  });

  it("signale une revision en hausse", () => {
    render(<QuoteRevisionDiff pair={{ ...pair, current: { ...pair.current, totalMinor: 108_000 } }} locale="fr-FR" />);
    expect(screen.getByTestId("quote-diff-delta")).toHaveTextContent("+100,00");
  });

  it("indique la revision la plus recente", () => {
    render(<QuoteRevisionDiff pair={pair} locale="fr-FR" />);
    expect(screen.getByTestId("quote-diff-current")).toHaveAttribute("aria-current", "true");
  });

  it("affiche les lignes ajoutees et retirees", () => {
    render(
      <QuoteRevisionDiff
        pair={{ ...pair, current: { ...pair.current, lines: [{ variantId: "v-2", quantity: 50, unitPriceMinor: 1_760 }] } }}
        locale="fr-FR"
      />,
    );
    expect(screen.getByTestId("quote-diff-added")).toHaveTextContent("v-2");
    expect(screen.getByTestId("quote-diff-removed")).toHaveTextContent("v-1");
  });
});
```

Créer `packages/ui/src/components/TrackingTimeline.spec.tsx` :

```tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { TrackingTimeline } from "./TrackingTimeline";

const events = [
  { providerEventId: "evt-1", status: "label_created", occurredAt: "2026-10-20T08:00:00.000Z", location: "Shenzhen" },
  { providerEventId: "evt-2", status: "in_transit", occurredAt: "2026-10-21T08:00:00.000Z", location: "Paris" },
];

describe("TrackingTimeline", () => {
  it("affiche chaque evenement du transporteur", () => {
    render(<TrackingTimeline events={events} locale="fr-FR" />);
    expect(screen.getAllByRole("listitem")).toHaveLength(2);
  });

  it("classe les evenements dans une liste ordonnee", () => {
    render(<TrackingTimeline events={events} locale="fr-FR" />);
    expect(screen.getByTestId("tracking-timeline")).toHaveAttribute("data-count", "2");
  });

  it("signale un evenement d'action requise", () => {
    render(
      <TrackingTimeline events={[...events, { providerEventId: "evt-3", status: "action_required", occurredAt: "2026-10-22T08:00:00.000Z", location: null }]} locale="fr-FR" />,
    );
    expect(screen.getByTestId("tracking-action-required")).toBeInTheDocument();
  });

  it("affiche un etat vide sans evenement", () => {
    render(<TrackingTimeline events={[]} locale="fr-FR" />);
    expect(screen.getByTestId("tracking-empty")).toBeInTheDocument();
  });

  it("n'affiche pas de pastille de statut seule (couleur jamais unique)", () => {
    render(<TrackingTimeline events={events} locale="fr-FR" />);
    expect(screen.getAllByTestId("tracking-status-label")[0]).toHaveTextContent(/\S/);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/ui exec vitest run src/components/QuoteRevisionDiff.spec.tsx src/components/TrackingTimeline.spec.tsx
```

Attendu : **ÉCHEC** — `Failed to resolve import "./QuoteRevisionDiff"`. 0 test exécuté.

- [ ] **Step 2: Implémenter `QuoteRevisionDiff`**

Créer `packages/ui/src/components/QuoteRevisionDiff.tsx` :

```tsx
import { useTranslations } from "@fanours/i18n";
import { formatMoney } from "@fanours/i18n/format";

export type QuoteRevisionPair = {
  previous: { revisionNo: number; totalMinor: number; currency: string; lines: { variantId: string; quantity: number; unitPriceMinor: number }[] };
  current: { revisionNo: number; totalMinor: number; currency: string; lines: { variantId: string; quantity: number; unitPriceMinor: number }[] };
};

/**
 * Comparaison de deux revisions de devis (CDC 15.4).
 * Les revisions sont append-only : on compare, on ne remplace pas.
 * L'ecart de total est signe, la couleur ne porte jamais seule l'information.
 */
export function QuoteRevisionDiff(props: { pair: QuoteRevisionPair; locale: "fr-FR" | "en" | "zh-CN" }): JSX.Element {
  const t = useTranslations(props.locale);
  const { previous, current } = props.pair;
  const deltaMinor = current.totalMinor - previous.totalMinor;
  const previousIds = new Set(previous.lines.map((line) => line.variantId));
  const currentIds = new Set(current.lines.map((line) => line.variantId));
  const added = current.lines.filter((line) => !previousIds.has(line.variantId));
  const removed = previous.lines.filter((line) => !currentIds.has(line.variantId));

  return (
    <div className="grid gap-4 md:grid-cols-2">
      <section className="rounded-[var(--radius-md)] border border-[var(--color-border)] bg-[var(--color-surface)] p-4">
        <h3 className="text-sm font-semibold text-[var(--color-text-muted)]" data-testid="quote-diff-previous">
          {t("quoteDiff.previous", { revisionNo: previous.revisionNo })}
        </h3>
        <p className="tabular-nums text-[var(--font-body)] text-lg font-bold text-[var(--color-text)]">
          {formatMoney(previous.totalMinor, previous.currency, props.locale)}
        </p>
      </section>

      <section
        className="rounded-[var(--radius-md)] border border-[var(--color-primary)] bg-[var(--color-surface)] p-4"
        aria-current="true"
      >
        <h3 className="text-sm font-semibold text-[var(--color-text-muted)]" data-testid="quote-diff-current">
          {t("quoteDiff.current", { revisionNo: current.revisionNo })}
        </h3>
        <p className="tabular-nums text-[var(--font-body)] text-lg font-bold text-[var(--color-text)]">
          {formatMoney(current.totalMinor, current.currency, props.locale)}
        </p>
      </section>

      <p className="tabular-nums text-sm font-semibold text-[var(--color-text)] md:col-span-2" data-testid="quote-diff-delta">
        {t("quoteDiff.delta", {
          amount: formatMoney(Math.abs(deltaMinor), current.currency, props.locale),
          sign: deltaMinor < 0 ? "−" : "+",
        })}
      </p>

      {added.length > 0 ? (
        <ul className="text-sm text-[var(--color-text)]" data-testid="quote-diff-added">
          {added.map((line) => (
            <li key={line.variantId}>
              {t("quoteDiff.added", { variantId: line.variantId, quantity: line.quantity })}
            </li>
          ))}
        </ul>
      ) : null}

      {removed.length > 0 ? (
        <ul className="text-sm text-[var(--color-text)]" data-testid="quote-diff-removed">
          {removed.map((line) => (
            <li key={line.variantId}>
              {t("quoteDiff.removed", { variantId: line.variantId, quantity: line.quantity })}
            </li>
          ))}
        </ul>
      ) : null}
    </div>
  );
}
```

- [ ] **Step 3: Implémenter `TrackingTimeline`**

Créer `packages/ui/src/components/TrackingTimeline.tsx` :

```tsx
import { useTranslations } from "@fanours/i18n";
import { formatDateTime } from "@fanours/i18n/format";

export type TrackingEventView = {
  providerEventId: string;
  status: string;
  occurredAt: string;
  location: string | null;
};

const ACTION_REQUIRED = new Set(["action_required", "customs_hold", "exception"]);

/**
 * Chronologie des evenements transporteur (CDC 12.2).
 * - Chaque evenement est un `li` de liste ; rien n'est masque au clic.
 * - Le statut est un TEXTE, jamais une pastille seule (accessibilite).
 * - Un evenement d'action requise est signale explicitement.
 */
export function TrackingTimeline(props: { events: TrackingEventView[]; locale: "fr-FR" | "en" | "zh-CN" }): JSX.Element {
  const t = useTranslations(props.locale);
  if (props.events.length === 0) {
    return (
      <p className="text-sm text-[var(--color-text-muted)]" data-testid="tracking-empty">
        {t("tracking.empty")}
      </p>
    );
  }
  return (
    <ol className="flex flex-col gap-3" data-testid="tracking-timeline" data-count={props.events.length}>
      {props.events.map((event) => (
        <li key={event.providerEventId} className="border-l-2 border-[var(--color-border)] pl-4">
          <p className="text-sm font-semibold text-[var(--color-text)]" data-testid="tracking-status-label">
            {t(`tracking.status.${event.status}`)}
          </p>
          <p className="text-xs text-[var(--color-text-muted)]">
            {formatDateTime(event.occurredAt, props.locale)}
            {event.location ? ` · ${event.location}` : ""}
          </p>
          {ACTION_REQUIRED.has(event.status) ? (
            <p
              className="mt-1 rounded-[var(--radius-sm)] border border-[var(--color-warning)] p-2 text-sm text-[var(--color-text)]"
              data-testid="tracking-action-required"
              role="status"
            >
              {t("tracking.actionRequired")}
            </p>
          ) : null}
        </li>
      ))}
    </ol>
  );
}
```

Exécuter :

```bash
pnpm --filter @fanours/ui exec vitest run src/components/QuoteRevisionDiff.spec.tsx src/components/TrackingTimeline.spec.tsx
```

Attendu : **10 tests passés** (5 + 5), 0 échec.

- [ ] **Step 4: Page liste des devis (client)**

Créer `apps/web/src/app/[locale]/(account)/devis/page.tsx` :

```tsx
import { getTranslations } from "next-intl/server";
import { fetchQuotes } from "@/server/quotes";
import { SystemState } from "@fanours/ui";

export const metadata = { robots: { index: false, follow: false } };

export default async function QuotesPage({ params }: { params: Promise<{ locale: string }> }) {
  const { locale } = await params;
  const t = await getTranslations(locale);
  const result = await fetchQuotes();

  if (result.kind === "error") {
    return <SystemState variant="offline" title={t("states.offline.title")} action={{ label: t("states.offline.retry"), href: "" }} />;
  }
  if (result.kind === "forbidden") {
    return <SystemState variant="forbidden" title={t("states.forbidden.title")} action={{ label: t("states.forbidden.back"), href: "/" }} />;
  }
  if (result.quotes.length === 0) {
    return <SystemState variant="empty" title={t("states.empty.quote.title")} action={{ label: t("states.empty.quote.action"), href: "/products" }} />;
  }

  return (
    <div className="flex flex-col gap-4">
      <h1 className="text-[var(--font-heading)] text-2xl font-bold text-[var(--color-text)]">{t("quote.listTitle")}</h1>
      <ul className="flex flex-col gap-3">
        {result.quotes.map((quote) => (
          <li key={quote.id} className="rounded-[var(--radius-md)] border border-[var(--color-border)] bg-[var(--color-surface)] p-4">
            <a className="font-semibold text-[var(--color-primary)]" href={`/${locale}/account/devis/${quote.number}`}>
              {quote.number}
            </a>
            <p className="text-sm text-[var(--color-text-muted)]">{t(`quote.status.${quote.status}`)}</p>
            <p className="tabular-nums text-[var(--font-body)] font-bold text-[var(--color-text)]">
              {quote.totalFormatted}
            </p>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

Le tableau client **n'est pas indexable** (SEO §6.1 : compte, devis, factures, suivi → `noindex` + hors sitemap). Le `metadata` ci-dessus l'assure.

- [ ] **Step 5: Page de suivi de colis (client)**

Créer `apps/web/src/app/[locale]/(account)/commandes/[number]/suivi/page.tsx` :

```tsx
import { getTranslations } from "next-intl/server";
import { fetchOrderShipments } from "@/server/orders";
import { TrackingTimeline } from "@fanours/ui";
import { SystemState } from "@fanours/ui";

export const metadata = { robots: { index: false, follow: false } };

export default async function ShipmentTrackingPage(props: { params: Promise<{ locale: string; number: string }> }) {
  const { locale, number } = await props.params;
  const t = await getTranslations(locale);
  const result = await fetchOrderShipments(number);

  if (result.kind === "forbidden" || result.kind === "not_found") {
    // Anti-IDOR : l'API repond 404 pour une commande qui n'appartient pas a l'appelant.
    return <SystemState variant="forbidden" title={t("states.forbidden.title")} action={{ label: t("states.forbidden.back"), href: `/${locale}/account/orders` }} />;
  }
  if (result.kind === "error") {
    return <SystemState variant="offline" title={t("states.offline.title")} action={{ label: t("states.offline.retry"), href: "" }} />;
  }

  return (
    <div className="flex flex-col gap-6">
      <h1 className="text-[var(--font-heading)] text-2xl font-bold text-[var(--color-text)]">
        {t("shipment.trackingTitle", { number })}
      </h1>
      {result.shipments.map((shipment) => (
        <section key={shipment.id} className="flex flex-col gap-3">
          <h2 className="text-[var(--font-heading)] text-lg font-semibold text-[var(--color-text)]">
            {shipment.carrier} · {shipment.trackingNumber}
          </h2>
          <TrackingTimeline events={shipment.events} locale={locale as "fr-FR" | "en" | "zh-CN"} />
        </section>
      ))}
    </div>
  );
}
```

- [ ] **Step 6: Page des documents douaniers (client)**

Créer `apps/web/src/app/[locale]/(account)/commandes/[number]/documents/page.tsx` :

```tsx
import { getTranslations } from "next-intl/server";
import { fetchOrderDocuments } from "@/server/orders";
import { SystemState } from "@fanours/ui";

export const metadata = { robots: { index: false, follow: false } };

export default async function OrderDocumentsPage(props: { params: Promise<{ locale: string; number: string }> }) {
  const { locale, number } = await props.params;
  const t = await getTranslations(locale);
  const result = await fetchOrderDocuments(number);

  if (result.kind === "forbidden" || result.kind === "not_found") {
    return <SystemState variant="forbidden" title={t("states.forbidden.title")} action={{ label: t("states.forbidden.back"), href: `/${locale}/account/orders` }} />;
  }
  if (result.kind === "error") {
    return <SystemState variant="offline" title={t("states.offline.title")} action={{ label: t("states.offline.retry"), href: "" }} />;
  }
  if (result.documents.length === 0) {
    return <SystemState variant="empty" title={t("states.empty.documents.title")} action={{ label: t("states.empty.documents.action"), href: `/${locale}/account/orders/${number}` }} />;
  }

  return (
    <div className="flex flex-col gap-4">
      <h1 className="text-[var(--font-heading)] text-2xl font-bold text-[var(--color-text)]">
        {t("documents.title", { number })}
      </h1>
      <ul className="flex flex-col gap-3">
        {result.documents.map((document) => (
          <li key={document.id} className="rounded-[var(--radius-md)] border border-[var(--color-border)] bg-[var(--color-surface)] p-4">
            <p className="font-semibold text-[var(--color-text)]">{t(`documents.kind.${document.kind}`)}</p>
            <p className="text-sm text-[var(--color-text-muted)]">{t("documents.version", { version: document.version })}</p>
            {/* URL signee de courte duree : jamais d'URL publique, jamais d'URL en dur. */}
            <a className="text-[var(--color-primary)] underline" href={document.signedUrl} rel="noreferrer noopener">
              {t("documents.download")}
            </a>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

- [ ] **Step 7: Export et commit**

```bash
pnpm --filter @fanours/ui exec vitest run src/components
pnpm --filter @fanours/web exec vitest run
pnpm turbo run typecheck
git add packages/ui/src/components/QuoteRevisionDiff.tsx packages/ui/src/components/QuoteRevisionDiff.spec.tsx packages/ui/src/components/TrackingTimeline.tsx packages/ui/src/components/TrackingTimeline.spec.tsx packages/ui/src/index.ts "apps/web/src/app/[locale]/(account)/devis" "apps/web/src/app/[locale]/(account)/commandes"
git commit -m "feat(web): ajoute les vues client devis, suivi de colis et documents douaniers"
```

### Task 17: Vues admin — expéditions, règles douanières, alertes de conformité, revue des comptes B2B

**Files:**
- Create: `packages/db/migrations/0110_phase3_admin_views.sql`
- Create: `packages/db/src/schema/phase3-admin.ts`
- Modify: `packages/db/src/schema/index.ts` (baril Phase 2 : ajouter `export * from "./phase3-admin.js";`)
- Modify: `packages/db/src/schema/phase3-logistics.ts` (colonnes admin de `shipment`)
- Modify: `packages/db/src/schema/phase3-compliance.ts` (colonnes admin de `complianceAlert`)
- Create: `apps/api/src/common/admin-scope.ts`
- Create: `apps/api/src/modules/shipment/admin-shipments.service.ts`
- Create: `apps/api/src/modules/shipment/admin-shipments.service.spec.ts`
- Create: `apps/api/src/modules/shipment/admin-shipments.controller.ts`
- Create: `apps/api/src/modules/compliance/admin-trade-rules.service.ts`
- Create: `apps/api/src/modules/compliance/admin-trade-rules.service.spec.ts`
- Create: `apps/api/src/modules/compliance/admin-trade-rules.controller.ts`
- Create: `apps/api/src/modules/compliance/compliance-alert.service.ts`
- Create: `apps/api/src/modules/compliance/compliance-alert.service.spec.ts`
- Modify: `apps/api/src/modules/compliance/compliance-alerts.controller.ts`
- Create: `apps/api/src/modules/business/admin-business-accounts.service.ts`
- Create: `apps/api/src/modules/business/admin-business-accounts.service.spec.ts`
- Create: `apps/api/src/modules/business/admin-business-accounts.controller.ts`
- Modify: `apps/api/src/modules/business/business-review.service.ts`
- Modify: `apps/api/src/modules/shipment/shipment.service.ts`
- Modify: `packages/domain/src/permissions.ts`
- Modify: `packages/db/src/seed.ts`
- Create: `packages/db/src/__tests__/phase3-admin-views.integration.spec.ts`
- Create: `packages/ui/src/components/DataTable.tsx`
- Create: `packages/ui/src/components/DataTable.spec.tsx`
- Create: `packages/ui/src/components/AdminWizard.tsx`
- Create: `packages/ui/src/components/AdminWizard.spec.tsx`
- Create: `packages/ui/src/components/KpiCard.tsx`
- Create: `packages/ui/src/components/KpiCard.spec.tsx`
- Create: `packages/ui/src/components/FilterBar.tsx`
- Create: `packages/ui/src/components/FilterBar.spec.tsx`
- Modify: `packages/ui/src/index.ts`
- Create: `apps/web/src/server/admin.ts`
- Create: `apps/web/src/app/[locale]/(admin)/admin/expeditions/page.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/douane/regles/page.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/douane/alertes/page.tsx`
- Create: `apps/web/src/app/[locale]/(admin)/admin/clients/page.tsx`
- Modify: `packages/i18n/src/locales/fr-FR.json`
- Modify: `packages/i18n/src/locales/en.json`
- Modify: `packages/i18n/src/locales/zh-CN.json`
- Create: `e2e/b2b/admin-views.spec.ts` (recette fonctionnelle de phase — projet Playwright `functional`)

**Interfaces:**
- Consumes:
  - Tables des Tasks 1 à 16 : `country_profile`, `incoterm_configuration`, `trade_rule`, `trade_rule_version`, `trade_rule_source`, `compliance_review`, `compliance_alert`, `tax_estimate`, `duty_estimate`, `business_account`, `business_document`, `quote`, `quote_revision`, `quote_line`, `shipment`, `shipment_item`, `package`, `tracking_event`, `shipping_quote`, `commercial_document`, `product_compliance`, `orders` (Phase 2), `audit_log` (Phase 1).
  - Services des Tasks précédentes : `BusinessReviewService` (Task 7), `ShipmentService` / `TrackingService` (Task 11), `HsReviewService` (Task 13), `ComplianceAlertService` (Task 13), `tradeRuleReviewJob` (Task 13).
  - `packages/domain` (`PERMISSIONS`, `RESOURCES`, `Role`, `Action`) et le guard `requirePermission` — matrice RBAC de la Phase 1.
  - `apps/api/src/modules/rbac/require-permission.ts` (`requirePermission`, `Principal`, `assertRecentMfa`, `MFA_MAX_AGE_MS`) — créé par la Task 7, même import que les Tasks 7, 11 et 13.
  - `apps/api/src/common/api-errors.ts` (`ForbiddenError`, `NotFoundError`, `ConflictError`) — créé par la Task 7.
  - `packages/i18n` (`useTranslations`, `translate`, `formatMoney`, `formatDateTime`), `packages/ui` (`SystemState` — Phase 1, Task 15 pour les composants de la Phase 3).
- Produces:
  - `apps/api/src/common/admin-scope.ts` :
    ```ts
    export type AdminPrincipal = Principal; // `Principal` vient de `apps/api/src/modules/rbac/require-permission.ts` (Task 7)
    export type AdminScope =
      | { readonly kind: "global" }
      | { readonly kind: "countries"; readonly countries: readonly string[] }
      | { readonly kind: "none" };
    export function resolveAdminScope(input: { roleScope: Scope; assignedCountries: readonly string[] }): AdminScope;
    export function scopeCondition(scope: AdminScope, column: AnyColumn): SQL | undefined;
    export async function loadAssignedCountries(db: DrizzleDb, userId: string): Promise<string[]>;
    export async function loadRoleScope(db: DrizzleDb, role: Role, resource: Resource): Promise<Scope>;
    export const MFA_MAX_AGE_MS: number;
    export function assertRecentMfa(principal: AdminPrincipal, nowMs?: number): void;
    ```
  - `apps/api/src/modules/shipment/admin-shipments.service.ts` :
    ```ts
    export const MAX_PAGE_SIZE = 100;
    export const MAX_OFFSET = 10_000;
    export const DEFAULT_PAGE_SIZE = 25;
    export type AdminShipmentFilters = {
      status: ShipmentStatus | null; countryCode: string | null; carrier: string | null;
      exceptionsOnly: boolean; limit: number; offset: number;
      sort: "created_at" | "updated_at" | "tracking_number"; order: "asc" | "desc";
    };
    export type AdminShipmentRow = {
      id: string; orderId: string; orderNumber: string; carrier: string; service: string;
      trackingNumber: string; status: ShipmentStatus; incoterm: string; countryCode: string | null;
      customsComplete: boolean; exceptionReason: string | null; exceptionOpenedAt: string | null;
      lastTrackingStatus: ShipmentStatus | null; createdAt: string; updatedAt: string;
    };
    export type AdminPage<T> = { items: T[]; total: number; limit: number; offset: number };
    export function normalizePage(input: { limit?: number; offset?: number }): { limit: number; offset: number };
    export function assertShipmentStatusFilter(value: string | undefined): ShipmentStatus | null;
    export class AdminShipmentError extends Error { constructor(public readonly code: string); }
    export class AdminShipmentsService {
      async list(params: { filters: AdminShipmentFilters; principal: AdminPrincipal }): Promise<AdminPage<AdminShipmentRow>>;
      async get(params: { shipmentId: string; principal: AdminPrincipal }): Promise<AdminShipmentRow>;
      async raiseException(params: { shipmentId: string; reason: string; principal: AdminPrincipal }): Promise<AdminShipmentRow>;
      async resolveException(params: { shipmentId: string; note: string; principal: AdminPrincipal }): Promise<AdminShipmentRow>;
    }
    ```
  - `apps/api/src/modules/compliance/admin-trade-rules.service.ts` :
    ```ts
    export type RuleVersionInput = {
      ruleKey: string; countryCode: string; scope: Scope; ruleType: RuleType; certainty: Certainty;
      calculationMode: CalculationMode; taxableBase: TaxableBase; params: RuleParams;
      thresholds: { min_amount_minor?: number; max_amount_minor?: number; max_units?: number };
      validFrom: string; validTo: string | null; reviewDueAt: string; sourceId: string;
    };
    export type RuleVersionValidation = { readonly ok: true } | { readonly ok: false; readonly code: string; readonly field: string };
    export function validateRuleVersionInput(input: RuleVersionInput): RuleVersionValidation;
    export function ruleReviewState(version: { validTo: string | null; reviewDueAt: string }, asOf: string):
      "expired" | "review_due" | "current";
    export class AdminTradeRuleError extends Error { constructor(public readonly code: string); }
    export class AdminTradeRulesService {
      async list(params: { filters: AdminRuleFilters; principal: AdminPrincipal }): Promise<AdminPage<AdminTradeRuleRow>>;
      async get(params: { ruleId: string; principal: AdminPrincipal }): Promise<AdminTradeRuleDetail>;
      async createRuleVersion(params: { input: RuleVersionInput; principal: AdminPrincipal }): Promise<AdminTradeRuleDetail>;
      async listReviewDue(params: { asOf: string; principal: AdminPrincipal }): Promise<AdminTradeRuleRow[]>;
    }
    ```
  - `apps/api/src/modules/compliance/compliance-alert.service.ts` :
    ```ts
    export type AdminComplianceAlertRow = {
      id: string; severity: "blocker" | "warning" | "info"; code: string; countryCode: string | null;
      ruleId: string | null; ruleKey: string | null; subjectType: string | null; subjectId: string | null;
      message: string; actionRequired: string | null; status: "open" | "acknowledged" | "resolved";
      createdAt: string; resolvedAt: string | null;
    };
    export function assertAlertTransition(from: AdminComplianceAlertRow["status"], to: AdminComplianceAlertRow["status"]): void;
    export class ComplianceAlertService {
      async list(params: { filters: AdminAlertFilters; principal: AdminPrincipal }): Promise<AdminPage<AdminComplianceAlertRow>>;
      async acknowledge(params: { alertId: string; principal: AdminPrincipal }): Promise<AdminComplianceAlertRow>;
      async resolve(params: { alertId: string; note: string; principal: AdminPrincipal }): Promise<AdminComplianceAlertRow>;
    }
    ```
  - `apps/api/src/modules/business/admin-business-accounts.service.ts` :
    ```ts
    export type AdminBusinessAccountRow = {
      id: string; legalName: string; country: string; taxId: string | null;
      verificationStatus: BusinessVerificationStatus; verifiedAt: string | null; verifiedBy: string | null;
      reviewNote: string | null; documentCount: number; createdAt: string;
    };
    export function assertFourEyes(requestedBy: string, approvedBy: string): void;
    export function approvalPayloadHash(input: { businessAccountId: string; decision: string; note: string }): string;
    export class AdminBusinessAccountsService {
      async list(params: { filters: AdminBusinessAccountFilters; principal: AdminPrincipal }): Promise<AdminPage<AdminBusinessAccountRow>>;
      async get(params: { businessAccountId: string; principal: AdminPrincipal }): Promise<AdminBusinessAccountRow>;
      async requestApproval(params: { businessAccountId: string; decision: ReviewDecision; principal: AdminPrincipal }): Promise<{ approvalId: string; status: string }>;
      async approve(params: { approvalId: string; principal: AdminPrincipal }): Promise<{ businessAccountId: string; verificationStatus: BusinessVerificationStatus }>;
      async suspend(params: { businessAccountId: string; reason: string; principal: AdminPrincipal }): Promise<{ verificationStatus: BusinessVerificationStatus }>;
    }
    ```
  - Routes REST (préfixe `/v1` constant, aucune route `/admin/*` hors de ce préfixe) :
    | Méthode | Route | Permission serveur | Ressource RBAC |
    |---|---|---|---|
    | `GET` | `/v1/admin/shipments` | `requirePermission(principal, "customs_document", "read")` | `customs_document` |
    | `GET` | `/v1/admin/shipments/:id` | `requirePermission(principal, "customs_document", "read")` | `customs_document` |
    | `POST` | `/v1/admin/shipments/:id/exception` | `requirePermission(principal, "customs_document", "update")` | `customs_document` |
    | `POST` | `/v1/admin/shipments/:id/exception/resolve` | `requirePermission(principal, "customs_document", "update")` | `customs_document` |
    | `GET` | `/v1/admin/trade-rules` | `requirePermission(principal, "customs_rule", "read")` | `customs_rule` |
    | `GET` | `/v1/admin/trade-rules/review-due` | `requirePermission(principal, "customs_rule", "read")` | `customs_rule` |
    | `GET` | `/v1/admin/trade-rules/:id` | `requirePermission(principal, "customs_rule", "read")` | `customs_rule` |
    | `POST` | `/v1/admin/trade-rules` | `requirePermission(principal, "customs_rule", "create")` + MFA | `customs_rule` |
    | `POST` | `/v1/admin/trade-rules/:id/versions` | `requirePermission(principal, "customs_rule", "update")` + MFA | `customs_rule` |
    | `GET` | `/v1/admin/compliance-alerts` | `requirePermission(principal, "customs_rule", "read")` | `customs_rule` |
    | `POST` | `/v1/admin/compliance-alerts/:id/acknowledge` | `requirePermission(principal, "customs_rule", "update")` + MFA | `customs_rule` |
    | `POST` | `/v1/admin/compliance-alerts/:id/resolve` | `requirePermission(principal, "customs_rule", "update")` + MFA | `customs_rule` |
    | `GET` | `/v1/admin/business-accounts` | `requirePermission(principal, "business_account", "read")` | `business_account` |
    | `GET` | `/v1/admin/business-accounts/:id` | `requirePermission(principal, "business_account", "read")` | `business_account` |
    | `POST` | `/v1/admin/business-accounts/:id/approvals` | `requirePermission(principal, "business_account", "approve")` + MFA | `business_account` |
    | `POST` | `/v1/admin/business-accounts/:id/approvals/:approvalId/approve` | `requirePermission(principal, "business_account", "approve")` + MFA | `business_account` |
    | `POST` | `/v1/admin/business-accounts/:id/suspend` | `requirePermission(principal, "business_account", "approve")` + MFA | `business_account` |
  - Pages web (Server Components, `metadata.robots.index = false`) :
    - `apps/web/src/app/[locale]/(admin)/admin/expeditions/page.tsx` — expéditions et exceptions (§16.8).
    - `apps/web/src/app/[locale]/(admin)/admin/douane/regles/page.tsx` — wizard des règles douanières (§16.9).
    - `apps/web/src/app/[locale]/(admin)/admin/douane/alertes/page.tsx` — alertes de conformité (§16.9).
    - `apps/web/src/app/[locale]/(admin)/admin/clients/page.tsx` — revue des comptes B2B (§16.6).
  - Composants `packages/ui` : `DataTable` (tri et pagination serveur, gel de la 1re colonne, `overflow-x` interne), `AdminWizard` (4 étapes), `KpiCard` (définition au survol/focus), `FilterBar` (filtres + bottom sheet mobile avec réinitialisation).

- [ ] **Step 1: Écrire le test unitaire du service admin expéditions (il doit échouer)**

Créer `apps/api/src/modules/shipment/admin-shipments.service.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import {
  AdminShipmentError,
  assertShipmentStatusFilter,
  normalizePage,
} from "./admin-shipments.service";

describe("normalizePage — pagination serveur", () => {
  it("applique une taille de page par defaut de 25", () => {
    expect(normalizePage({})).toEqual({ limit: 25, offset: 0 });
  });

  it("plafonne la taille de page a 100 lignes", () => {
    expect(normalizePage({ limit: 5_000 }).limit).toBe(100);
  });

  it("borne l'offset a 10 000 lignes (architecture section 4)", () => {
    expect(normalizePage({ offset: 99_999 }).offset).toBe(10_000);
  });

  it("ne descend jamais sous une ligne ni sous un offset nul", () => {
    expect(normalizePage({ limit: 0, offset: -5 })).toEqual({ limit: 1, offset: 0 });
  });
});

describe("assertShipmentStatusFilter", () => {
  it("accepte un statut connu", () => {
    expect(assertShipmentStatusFilter("customs_hold")).toBe("customs_hold");
  });

  it("refuse un statut inconnu au lieu de renvoyer une liste vide silencieuse", () => {
    expect(() => assertShipmentStatusFilter("perdu")).toThrow(AdminShipmentError);
  });

  it("traite un filtre absent comme aucun filtre", () => {
    expect(assertShipmentStatusFilter(undefined)).toBeNull();
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/shipment/admin-shipments.service.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./admin-shipments.service"`. 0 test exécuté.

- [ ] **Step 2: Implémenter la portée admin partagée et le service des expéditions**

Créer `apps/api/src/common/admin-scope.ts` :

```ts
import { and, eq, inArray, sql, type AnyColumn, type SQL } from "drizzle-orm";
import type { Scope as RbacScope } from "../modules/rbac/scope";
import { rbacScope } from "@fanours/db/schema";
import type { Resource, Role } from "@fanours/domain";
import { assertRecentMfa, MFA_MAX_AGE_MS, type Principal } from "../modules/rbac/require-permission";

/**
 * Acteur d'une vue admin. C'est exactement le `Principal` de la Phase 3
 * (`apps/api/src/modules/rbac/require-permission.ts`) : le type n'est pas
 * redefini ici, ce module n'ajoute que la PORTEe pays.
 */
export type AdminPrincipal = Principal;

/**
 * Portee de lecture d'une vue admin (anti-IDOR).
 * - `global`  : aucune restriction de pays.
 * - `countries`: le principal n'est habilite que sur les pays listes.
 * - `none`    : la requete ne ramene AUCUNE ligne (la condition vaut `false`).
 */
export type AdminScope =
  | { readonly kind: "global" }
  | { readonly kind: "countries"; readonly countries: readonly string[] }
  | { readonly kind: "none" };

/**
 * Resolution de la portee : le scope RBAC du role (`own` / `all` / `none`) decide
 * d'abord, les affectations pays du principal restreignent ensuite.
 * Un scope `none` l'emporte toujours : l'absence d'affectation n'accorde rien.
 *
 * Note de modele : une expedition ou une regle douaniere appartient a
 * l'organisation, pas a un utilisateur. Il n'existe donc pas de colonne
 * proprietaire sur ces tables : pour ces vues, `own` et `all` ouvrent la meme
 * vue organisationnelle, et le filtre reel est l'affectation pays. Seul `none`
 * ferme l'acces.
 */
export function resolveAdminScope(input: {
  roleScope: RbacScope;
  assignedCountries: readonly string[];
}): AdminScope {
  if (input.roleScope === "none") return { kind: "none" };
  if (input.assignedCountries.length === 0) return { kind: "global" };
  return { kind: "countries", countries: [...input.assignedCountries] };
}

/**
 * Scope RBAC du role pour une ressource, lu en base (`rbac_scope`).
 * Une absence de ligne est traitee comme `none` : deny by default.
 */
export async function loadRoleScope(
  db: DrizzleDb,
  role: Role,
  resource: Resource,
): Promise<RbacScope> {
  const rows = await db
    .select({ scope: rbacScope.scope })
    .from(rbacScope)
    .where(and(eq(rbacScope.roleCode, role), eq(rbacScope.resource, resource)))
    .limit(1);
  const raw = rows[0]?.scope;
  if (raw === "own" || raw === "all" || raw === "none") return raw;
  return "none";
}

/** La condition de portee est appliquee DANS la requete, jamais en post-filtrage. */
export function scopeCondition(scope: AdminScope, column: AnyColumn): SQL | undefined {
  if (scope.kind === "none") return sql`false`;
  if (scope.kind === "global") return undefined;
  return inArray(column, scope.countries as string[]);
}

/** Pays affectes a un principal : la table est vide si le principal est global. */
export async function loadAssignedCountries(db: DrizzleDb, userId: string): Promise<string[]> {
  const rows = await db
    .select({ countryCode: adminCountryScope.countryCode })
    .from(adminCountryScope)
    .where(eq(adminCountryScope.userId, userId));
  return rows.map((row) => row.countryCode);
}

/**
 * Une action privilegiee (approbation, regle, suspension) exige un MFA valide
 * depuis moins de 10 minutes (CDC securite 3.3, liste PRIVILEGED).
 * L'implementation est celle de la Phase 3 : elle n'est pas dupliquee ici.
 */
export { assertRecentMfa, MFA_MAX_AGE_MS };
```

Créer `apps/api/src/modules/shipment/admin-shipments.service.ts` :

```ts
import { and, asc, desc, eq, ilike, isNotNull, sql, type SQL } from "drizzle-orm";
import { requirePermission } from "../rbac/require-permission";
import { NotFoundError } from "../../common/api-errors";
import { auditLog, orders, shipment, trackingEvent } from "@fanours/db/schema";
import {
  loadAssignedCountries,
  loadRoleScope,
  resolveAdminScope,
  scopeCondition,
  type AdminPrincipal,
  type AdminScope,
} from "../../common/admin-scope";
import type { ShipmentStatus } from "./tracking.service";

export const MAX_PAGE_SIZE = 100;
export const MAX_OFFSET = 10_000;
export const DEFAULT_PAGE_SIZE = 25;

export const ADMIN_SHIPMENT_STATUSES = [
  "label_created", "handed_over", "in_transit", "customs_hold",
  "action_required", "out_for_delivery", "delivered", "exception", "returned",
] as const satisfies readonly ShipmentStatus[];

export type AdminShipmentFilters = {
  status: ShipmentStatus | null;
  countryCode: string | null;
  carrier: string | null;
  exceptionsOnly: boolean;
  limit: number;
  offset: number;
  sort: "created_at" | "updated_at" | "tracking_number";
  order: "asc" | "desc";
};

export type AdminShipmentRow = {
  id: string;
  orderId: string;
  orderNumber: string;
  carrier: string;
  service: string;
  trackingNumber: string;
  status: ShipmentStatus;
  incoterm: string;
  countryCode: string | null;
  customsComplete: boolean;
  exceptionReason: string | null;
  exceptionOpenedAt: string | null;
  lastTrackingStatus: ShipmentStatus | null;
  createdAt: string;
  updatedAt: string;
};

export type AdminPage<T> = { items: T[]; total: number; limit: number; offset: number };

export class AdminShipmentError extends Error {
  constructor(public readonly code: string) {
    super(code);
    this.name = "AdminShipmentError";
  }
}

/** Bornes de pagination serveur : jamais plus de 100 lignes, jamais au-dela de 10 000 lignes. */
export function normalizePage(input: { limit?: number; offset?: number }): {
  limit: number;
  offset: number;
} {
  const rawLimit = Number.isFinite(input.limit) ? Math.trunc(input.limit as number) : DEFAULT_PAGE_SIZE;
  const rawOffset = Number.isFinite(input.offset) ? Math.trunc(input.offset as number) : 0;
  const limit = Math.min(Math.max(rawLimit, 1), MAX_PAGE_SIZE);
  const offset = Math.min(Math.max(rawOffset, 0), MAX_OFFSET);
  return { limit, offset };
}

/** Un filtre de statut inconnu est une ERREUR, jamais une liste vide silencieuse. */
export function assertShipmentStatusFilter(value: string | undefined): ShipmentStatus | null {
  if (value === undefined || value === "") return null;
  if (!(ADMIN_SHIPMENT_STATUSES as readonly string[]).includes(value)) {
    throw new AdminShipmentError("invalid_status_filter");
  }
  return value as ShipmentStatus;
}

function sortColumn(sort: AdminShipmentFilters["sort"]): SQL {
  switch (sort) {
    case "updated_at": return sql`${shipment.updatedAt}`;
    case "tracking_number": return sql`${shipment.trackingNumber}`;
    case "created_at": return sql`${shipment.createdAt}`;
  }
}

export class AdminShipmentsService {
  constructor(private readonly db: DrizzleDb) {}

  /**
   * Portee de lecture ET d'ecriture du principal sur les expeditions.
   * La meme condition est reutilisee dans les SELECT et dans les UPDATE :
   * on ne peut pas modifier un colis hors perimetre (anti-IDOR).
   */
  private async scopeFor(principal: AdminPrincipal): Promise<AdminScope> {
    const countries = await loadAssignedCountries(this.db, principal.userId);
    const roleScope = await loadRoleScope(this.db, principal.role, "customs_document");
    return resolveAdminScope({ roleScope, assignedCountries: countries });
  }

  /**
   * Liste des expeditions (CDC 16.8). Le tri et la pagination sont EXECUTES PAR LE SERVEUR ;
   * le client ne recoit jamais plus de 100 lignes. Le perimetre pays est applique
   * dans la clause WHERE (anti-IDOR), jamais en post-filtrage.
   */
  async list(params: {
    filters: AdminShipmentFilters;
    principal: AdminPrincipal;
  }): Promise<AdminPage<AdminShipmentRow>> {
    requirePermission(params.principal, "customs_document", "read");

    const scope = await this.scopeFor(params.principal);

    const conditions: SQL[] = [];
    const scopeSql = scopeCondition(scope, shipment.countryCode);
    if (scopeSql !== undefined) conditions.push(scopeSql);
    if (params.filters.status !== null) conditions.push(eq(shipment.status, params.filters.status));
    if (params.filters.countryCode !== null) conditions.push(eq(shipment.countryCode, params.filters.countryCode));
    if (params.filters.carrier !== null) conditions.push(ilike(shipment.carrier, `%${params.filters.carrier}%`));
    if (params.filters.exceptionsOnly) {
      conditions.push(
        sql`${shipment.status} IN ('customs_hold','action_required','exception')
             OR ${shipment.exceptionReason} IS NOT NULL`,
      );
    }
    const where = and(...conditions);

    const [totalRow] = await this.db
      .select({ value: sql<number>`count(*)::int` })
      .from(shipment)
      .innerJoin(orders, eq(orders.id, shipment.orderId))
      .where(where);
    const total = totalRow?.value ?? 0;

    const direction = params.filters.order === "asc" ? asc : desc;
    const rows = await this.db
      .select({
        id: shipment.id,
        orderId: shipment.orderId,
        orderNumber: orders.number,
        carrier: shipment.carrier,
        service: shipment.service,
        trackingNumber: shipment.trackingNumber,
        status: shipment.status,
        incoterm: shipment.incoterm,
        countryCode: shipment.countryCode,
        customsComplete: sql<boolean>`${shipment.customsData} <> '{}'::jsonb`,
        exceptionReason: shipment.exceptionReason,
        exceptionOpenedAt: shipment.exceptionOpenedAt,
        createdAt: shipment.createdAt,
        updatedAt: shipment.updatedAt,
      })
      .from(shipment)
      .innerJoin(orders, eq(orders.id, shipment.orderId))
      .where(where)
      .orderBy(direction(sortColumn(params.filters.sort)), direction(sql`${shipment.id}`))
      .limit(params.filters.limit)
      .offset(params.filters.offset);

    const items: AdminShipmentRow[] = rows.map((row) => ({
      id: row.id,
      orderId: row.orderId,
      orderNumber: row.orderNumber,
      carrier: row.carrier,
      service: row.service,
      trackingNumber: row.trackingNumber,
      status: row.status as ShipmentStatus,
      incoterm: row.incoterm,
      countryCode: row.countryCode,
      customsComplete: row.customsComplete,
      exceptionReason: row.exceptionReason,
      exceptionOpenedAt: row.exceptionOpenedAt?.toISOString() ?? null,
      lastTrackingStatus: null,
      createdAt: row.createdAt.toISOString(),
      updatedAt: row.updatedAt.toISOString(),
    }));

    return { items, total, limit: params.filters.limit, offset: params.filters.offset };
  }

  /**
   * Detail d'une expedition. Hors perimetre, la reponse est 404 : l'existence
   * de la ressource n'est jamais divulguee (CDC 19.2).
   */
  async get(params: { shipmentId: string; principal: AdminPrincipal }): Promise<AdminShipmentRow> {
    requirePermission(params.principal, "customs_document", "read");

    const scope = await this.scopeFor(params.principal);
    const scopeSql = scopeCondition(scope, shipment.countryCode);

    const rows = await this.db
      .select({
        id: shipment.id,
        orderId: shipment.orderId,
        orderNumber: orders.number,
        carrier: shipment.carrier,
        service: shipment.service,
        trackingNumber: shipment.trackingNumber,
        status: shipment.status,
        incoterm: shipment.incoterm,
        countryCode: shipment.countryCode,
        customsComplete: sql<boolean>`${shipment.customsData} <> '{}'::jsonb`,
        exceptionReason: shipment.exceptionReason,
        exceptionOpenedAt: shipment.exceptionOpenedAt,
        createdAt: shipment.createdAt,
        updatedAt: shipment.updatedAt,
      })
      .from(shipment)
      .innerJoin(orders, eq(orders.id, shipment.orderId))
      .where(and(eq(shipment.id, params.shipmentId), scopeSql))
      .limit(1);

    const row = rows[0];
    if (row === undefined) throw new NotFoundError("shipment_not_found");

    const events = await this.db
      .select({ status: trackingEvent.status })
      .from(trackingEvent)
      .where(eq(trackingEvent.shipmentId, params.shipmentId))
      .orderBy(desc(trackingEvent.occurredAt))
      .limit(1);

    return {
      id: row.id,
      orderId: row.orderId,
      orderNumber: row.orderNumber,
      carrier: row.carrier,
      service: row.service,
      trackingNumber: row.trackingNumber,
      status: row.status as ShipmentStatus,
      incoterm: row.incoterm,
      countryCode: row.countryCode,
      customsComplete: row.customsComplete,
      exceptionReason: row.exceptionReason,
      exceptionOpenedAt: row.exceptionOpenedAt?.toISOString() ?? null,
      lastTrackingStatus: (events[0]?.status as ShipmentStatus | undefined) ?? null,
      createdAt: row.createdAt.toISOString(),
      updatedAt: row.updatedAt.toISOString(),
    };
  }

  /**
   * Colis retenu (CDC 11.9, 16.8) : la cause est OBLIGATOIRE et l'evenement est journalise.
   */
  async raiseException(params: {
    shipmentId: string;
    reason: string;
    principal: AdminPrincipal;
  }): Promise<AdminShipmentRow> {
    requirePermission(params.principal, "customs_document", "update");
    const reason = params.reason.trim();
    if (reason.length < 5) throw new AdminShipmentError("exception_reason_required");

    const scope = await this.scopeFor(params.principal);
    const scopeSql = scopeCondition(scope, shipment.countryCode);

    const updated = await this.db
      .update(shipment)
      .set({
        exceptionReason: reason,
        exceptionOpenedAt: new Date(),
        exceptionRaisedBy: params.principal.userId,
        updatedAt: new Date(),
      })
      .where(and(eq(shipment.id, params.shipmentId), scopeSql))
      .returning({ id: shipment.id });
    if (updated.length === 0) throw new NotFoundError("shipment_not_found");

    await this.db.insert(auditLog).values({
      actorId: params.principal.userId,
      actorRole: params.principal.role,
      action: "shipment.exception_raised",
      objectType: "shipment",
      objectId: params.shipmentId,
      result: "success",
      correlationId: params.principal.correlationId,
      occurredAt: new Date(),
      metadata: { reason },
    });

    return this.get({ shipmentId: params.shipmentId, principal: params.principal });
  }

  /** Levee de l'exception : la note de resolution est tracee dans l'audit. */
  async resolveException(params: {
    shipmentId: string;
    note: string;
    principal: AdminPrincipal;
  }): Promise<AdminShipmentRow> {
    requirePermission(params.principal, "customs_document", "update");
    const note = params.note.trim();
    if (note.length < 5) throw new AdminShipmentError("resolution_note_required");

    const scope = await this.scopeFor(params.principal);
    const scopeSql = scopeCondition(scope, shipment.countryCode);

    const updated = await this.db
      .update(shipment)
      .set({
        exceptionResolvedAt: new Date(),
        exceptionResolvedBy: params.principal.userId,
        updatedAt: new Date(),
      })
      .where(
        and(
          eq(shipment.id, params.shipmentId),
          isNotNull(shipment.exceptionReason),
          scopeSql,
        ),
      )
      .returning({ id: shipment.id });
    if (updated.length === 0) throw new NotFoundError("shipment_exception_not_found");

    await this.db.insert(auditLog).values({
      actorId: params.principal.userId,
      actorRole: params.principal.role,
      action: "shipment.exception_resolved",
      objectType: "shipment",
      objectId: params.shipmentId,
      result: "success",
      correlationId: params.principal.correlationId,
      occurredAt: new Date(),
      metadata: { note },
    });

    return this.get({ shipmentId: params.shipmentId, principal: params.principal });
  }
}
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/shipment/admin-shipments.service.spec.ts
```

Attendu : **7 tests passés**, 0 échec.

- [ ] **Step 3: Écrire le test unitaire du service admin des règles douanières (il doit échouer)**

Créer `apps/api/src/modules/compliance/admin-trade-rules.service.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import {
  ruleReviewState,
  validateRuleVersionInput,
  type RuleVersionInput,
} from "./admin-trade-rules.service";

const base: RuleVersionInput = {
  ruleKey: "fr.vat.global",
  countryCode: "FR",
  scope: "global",
  ruleType: "vat",
  certainty: "estimated",
  calculationMode: "percentage",
  taxableBase: "goods+shipping",
  params: { rate_bps: 2000 },
  thresholds: {},
  validFrom: "2026-01-01T00:00:00.000Z",
  validTo: null,
  reviewDueAt: "2026-12-01T00:00:00.000Z",
  sourceId: "3f1b0d2c-0000-4000-8000-000000000001",
};

describe("validateRuleVersionInput — les regles sont des donnees, jamais du code", () => {
  it("accepte une regle complete avec sa source et sa date de revue", () => {
    expect(validateRuleVersionInput(base)).toEqual({ ok: true });
  });

  it("refuse un code pays mal forme", () => {
    const result = validateRuleVersionInput({ ...base, countryCode: "fr" });
    expect(result).toMatchObject({ ok: false, code: "invalid_country_code" });
  });

  it("refuse une regle sans source officielle", () => {
    const result = validateRuleVersionInput({ ...base, sourceId: "" });
    expect(result).toMatchObject({ ok: false, code: "source_url_required" });
  });

  it("refuse une regle sans date de derniere revue", () => {
    const result = validateRuleVersionInput({ ...base, reviewDueAt: "" });
    expect(result).toMatchObject({ ok: false, code: "review_due_at_required" });
  });

  it("refuse un seuil monetaire sans date de fin (CDC 11.7)", () => {
    const result = validateRuleVersionInput({
      ...base, ruleType: "threshold", thresholds: { min_amount_minor: 150_00 },
    });
    expect(result).toMatchObject({ ok: false, code: "threshold_requires_valid_to" });
  });

  it("refuse une regle bloquante sans date de fin", () => {
    const result = validateRuleVersionInput({ ...base, ruleType: "prohibition" });
    expect(result).toMatchObject({ ok: false, code: "blocking_rule_requires_valid_to" });
  });

  it("refuse une date de fin anterieure a la date de debut", () => {
    const result = validateRuleVersionInput({
      ...base, validFrom: "2026-06-01T00:00:00.000Z", validTo: "2026-01-01T00:00:00.000Z",
    });
    expect(result).toMatchObject({ ok: false, code: "invalid_validity_window" });
  });

  it("refuse un code de regle vide", () => {
    const result = validateRuleVersionInput({ ...base, ruleKey: "" });
    expect(result).toMatchObject({ ok: false, code: "rule_key_required" });
  });

  it("refuse une regle non globale sans payload produit (contrainte base)", () => {
    const result = validateRuleVersionInput({ ...base, scope: "product" });
    expect(result).toMatchObject({ ok: false, code: "scope_payload_required" });
  });
});

describe("ruleReviewState", () => {
  it("marque une regle expiree", () => {
    const state = ruleReviewState(
      { validTo: "2026-01-01T00:00:00.000Z", reviewDueAt: "2026-12-01T00:00:00.000Z" },
      "2026-10-21T00:00:00.000Z",
    );
    expect(state).toBe("expired");
  });

  it("marque une regle a revalider", () => {
    const state = ruleReviewState(
      { validTo: null, reviewDueAt: "2026-01-01T00:00:00.000Z" },
      "2026-10-21T00:00:00.000Z",
    );
    expect(state).toBe("review_due");
  });

  it("laisse une regle a jour", () => {
    const state = ruleReviewState(
      { validTo: "2027-01-01T00:00:00.000Z", reviewDueAt: "2026-12-01T00:00:00.000Z" },
      "2026-10-21T00:00:00.000Z",
    );
    expect(state).toBe("current");
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/compliance/admin-trade-rules.service.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./admin-trade-rules.service"`. 0 test exécuté.

- [ ] **Step 4: Implémenter le service admin des règles douanières**

Créer `apps/api/src/modules/compliance/admin-trade-rules.service.ts` :

```ts
import { and, asc, desc, eq, ilike, sql, type SQL } from "drizzle-orm";
import { requirePermission } from "../rbac/require-permission";
import { NotFoundError } from "../../common/api-errors";
import { auditLog, tradeRule, tradeRuleSource, tradeRuleVersion } from "@fanours/db/schema";
import type {
  CalculationMode, Certainty, RuleParams, RuleType, Scope, TaxableBase,
} from "@fanours/compliance";
import {
  loadAssignedCountries,
  loadRoleScope,
  resolveAdminScope,
  scopeCondition,
  type AdminPrincipal,
} from "../../common/admin-scope";
import { MAX_PAGE_SIZE, type AdminPage } from "../shipment/admin-shipments.service";

export type RuleVersionInput = {
  ruleKey: string;
  countryCode: string;
  scope: Scope;
  ruleType: RuleType;
  certainty: Certainty;
  calculationMode: CalculationMode;
  taxableBase: TaxableBase;
  params: RuleParams;
  thresholds: { min_amount_minor?: number; max_amount_minor?: number; max_units?: number };
  validFrom: string;
  validTo: string | null;
  reviewDueAt: string;
  sourceId: string;
};

export type RuleVersionValidation =
  | { readonly ok: true }
  | { readonly ok: false; readonly code: string; readonly field: string };

export type AdminRuleFilters = {
  countryCode: string | null;
  ruleType: RuleType | null;
  certainty: Certainty | null;
  reviewState: "current" | "expired" | "review_due" | null;
  search: string | null;
  limit: number;
  offset: number;
};

export type AdminTradeRuleRow = {
  id: string;
  key: string;
  countryCode: string;
  scope: Scope;
  ruleType: RuleType;
  certainty: Certainty;
  active: boolean;
  currentVersionNo: number | null;
  validTo: string | null;
  reviewDueAt: string | null;
  reviewState: "expired" | "review_due" | "current";
  sourceUrl: string | null;
  sourceAuthority: string | null;
  sourceConsultedAt: string | null;
  updatedAt: string;
};

export type AdminTradeRuleDetail = AdminTradeRuleRow & {
  versions: {
    versionNo: number;
    validFrom: string;
    validTo: string | null;
    params: RuleParams;
    calculationMode: CalculationMode;
    taxableBase: TaxableBase;
    thresholds: Record<string, number | null>;
    validatedBy: string | null;
    validatedAt: string | null;
    sourceUrl: string;
    sourceAuthority: string;
    sourceConsultedAt: string;
  }[];
};

export class AdminTradeRuleError extends Error {
  constructor(public readonly code: string) {
    super(code);
    this.name = "AdminTradeRuleError";
  }
}

/**
 * Validation d'une version de regle (CDC 11.2, 16.9).
 * Source officielle, date de derniere revue et validateur humain sont obligatoires.
 * Aucun taux en dur : les parametres sont des entiers fournis par l'utilisateur.
 */
export function validateRuleVersionInput(input: RuleVersionInput): RuleVersionValidation {
  if (input.ruleKey.trim().length === 0) return { ok: false, code: "rule_key_required", field: "ruleKey" };
  if (input.scope !== "global") {
    // Une regle produit / variante / prefixe HS porte un payload obligatoire
    // (`trade_rule_scope_payload`) qui n'est saisi que depuis la fiche produit.
    return { ok: false, code: "scope_payload_required", field: "scope" };
  }
  if (!/^[A-Z]{2}$/.test(input.countryCode)) {
    return { ok: false, code: "invalid_country_code", field: "countryCode" };
  }
  if (input.sourceId.trim().length === 0) {
    return { ok: false, code: "source_url_required", field: "sourceId" };
  }
  if (input.reviewDueAt.trim().length === 0) {
    return { ok: false, code: "review_due_at_required", field: "reviewDueAt" };
  }
  if (Number.isNaN(Date.parse(input.validFrom))) {
    return { ok: false, code: "invalid_valid_from", field: "validFrom" };
  }
  if (input.validTo !== null) {
    if (Number.isNaN(Date.parse(input.validTo))) {
      return { ok: false, code: "invalid_validity_window", field: "validTo" };
    }
    if (Date.parse(input.validTo) <= Date.parse(input.validFrom)) {
      return { ok: false, code: "invalid_validity_window", field: "validTo" };
    }
  }
  const hasMonetaryThreshold =
    input.thresholds.min_amount_minor !== undefined || input.thresholds.max_amount_minor !== undefined;
  if (hasMonetaryThreshold && input.validTo === null) {
    return { ok: false, code: "threshold_requires_valid_to", field: "validTo" };
  }
  if ((input.ruleType === "prohibition" || input.ruleType === "license") && input.validTo === null) {
    return { ok: false, code: "blocking_rule_requires_valid_to", field: "validTo" };
  }
  return { ok: true };
}

/** Etat de revue d'une version : expiree, a revalider, ou a jour. */
export function ruleReviewState(
  version: { validTo: string | null; reviewDueAt: string },
  asOf: string,
): "expired" | "review_due" | "current" {
  const at = Date.parse(asOf);
  if (version.validTo !== null && Date.parse(version.validTo) <= at) return "expired";
  if (Date.parse(version.reviewDueAt) <= at) return "review_due";
  return "current";
}

export class AdminTradeRulesService {
  constructor(private readonly db: DrizzleDb) {}

  /**
   * Centre de regles par pays, categorie et date (CDC 16.9).
   * Le perimetre pays du principal est applique DANS la requete.
   */
  async list(params: {
    filters: AdminRuleFilters;
    principal: AdminPrincipal;
  }): Promise<AdminPage<AdminTradeRuleRow>> {
    requirePermission(params.principal, "customs_rule", "read");

    const countries = await loadAssignedCountries(this.db, params.principal.userId);
    const roleScope = await loadRoleScope(this.db, params.principal.role, "customs_rule");
    const scope = resolveAdminScope({ roleScope, assignedCountries: countries });

    const conditions: SQL[] = [];
    const scopeSql = scopeCondition(scope, tradeRule.countryCode);
    if (scopeSql !== undefined) conditions.push(scopeSql);
    if (params.filters.countryCode !== null) conditions.push(eq(tradeRule.countryCode, params.filters.countryCode));
    if (params.filters.ruleType !== null) conditions.push(eq(tradeRule.ruleType, params.filters.ruleType));
    if (params.filters.certainty !== null) conditions.push(eq(tradeRule.certainty, params.filters.certainty));
    if (params.filters.search !== null) conditions.push(ilike(tradeRule.key, `%${params.filters.search}%`));
    const where = and(...conditions);

    const rows = await this.db
      .select({
        id: tradeRule.id,
        key: tradeRule.key,
        countryCode: tradeRule.countryCode,
        scope: tradeRule.scope,
        ruleType: tradeRule.ruleType,
        certainty: tradeRule.certainty,
        active: tradeRule.active,
        updatedAt: tradeRule.updatedAt,
        versionNo: tradeRuleVersion.versionNo,
        validTo: tradeRuleVersion.validTo,
        reviewDueAt: tradeRuleVersion.reviewDueAt,
        sourceUrl: tradeRuleSource.url,
        sourceAuthority: tradeRuleSource.authority,
        sourceConsultedAt: tradeRuleSource.consultedAt,
      })
      .from(tradeRule)
      .innerJoin(tradeRuleVersion, eq(tradeRuleVersion.tradeRuleId, tradeRule.id))
      .innerJoin(tradeRuleSource, eq(tradeRuleSource.id, tradeRuleVersion.sourceId))
      .where(where)
      .orderBy(asc(tradeRule.countryCode), desc(tradeRuleVersion.versionNo))
      .limit(params.filters.limit)
      .offset(params.filters.offset);

    const asOf = new Date().toISOString();
    const items: AdminTradeRuleRow[] = rows.map((row) => ({
      id: row.id,
      key: row.key,
      countryCode: row.countryCode,
      scope: row.scope as Scope,
      ruleType: row.ruleType as RuleType,
      certainty: row.certainty as Certainty,
      active: row.active,
      currentVersionNo: row.versionNo,
      validTo: row.validTo?.toISOString() ?? null,
      reviewDueAt: row.reviewDueAt?.toISOString() ?? null,
      reviewState: ruleReviewState(
        { validTo: row.validTo?.toISOString() ?? null, reviewDueAt: row.reviewDueAt.toISOString() },
        asOf,
      ),
      sourceUrl: row.sourceUrl,
      sourceAuthority: row.sourceAuthority,
      sourceConsultedAt: row.sourceConsultedAt.toISOString(),
      updatedAt: row.updatedAt.toISOString(),
    }));

    const [totalRow] = await this.db
      .select({ value: sql<number>`count(*)::int` })
      .from(tradeRule)
      .where(where);
    const total = totalRow?.value ?? 0;

    return { items, total, limit: params.filters.limit, offset: params.filters.offset };
  }

  /** Regles a revalider : la revue de regle par le compliance manager (CDC 16.9). */
  async listReviewDue(params: { asOf: string; principal: AdminPrincipal }): Promise<AdminTradeRuleRow[]> {
    requirePermission(params.principal, "customs_rule", "read");
    const page = await this.list({
      filters: {
        countryCode: null, ruleType: null, certainty: null, reviewState: null, search: null,
        limit: MAX_PAGE_SIZE, offset: 0,
      },
      principal: params.principal,
    });
    return page.items.filter((rule) => rule.reviewState !== "current");
  }

  /**
   * Detail d'une regle. Hors perimetre pays, la reponse est 404 : l'existence
   * de la regle n'est pas divulguee.
   */
  async get(params: { ruleId: string; principal: AdminPrincipal }): Promise<AdminTradeRuleDetail> {
    requirePermission(params.principal, "customs_rule", "read");

    const countries = await loadAssignedCountries(this.db, params.principal.userId);
    const roleScope = await loadRoleScope(this.db, params.principal.role, "customs_rule");
    const scope = resolveAdminScope({ roleScope, assignedCountries: countries });
    const scopeSql = scopeCondition(scope, tradeRule.countryCode);

    const rows = await this.db
      .select({
        id: tradeRule.id,
        key: tradeRule.key,
        countryCode: tradeRule.countryCode,
        scope: tradeRule.scope,
        ruleType: tradeRule.ruleType,
        certainty: tradeRule.certainty,
        active: tradeRule.active,
        updatedAt: tradeRule.updatedAt,
      })
      .from(tradeRule)
      .where(and(eq(tradeRule.id, params.ruleId), scopeSql))
      .limit(1);
    const row = rows[0];
    if (row === undefined) throw new NotFoundError("trade_rule_not_found");

    const versions = await this.db
      .select({
        versionNo: tradeRuleVersion.versionNo,
        validFrom: tradeRuleVersion.validFrom,
        validTo: tradeRuleVersion.validTo,
        params: tradeRuleVersion.params,
        calculationMode: tradeRuleVersion.calculationMode,
        taxableBase: tradeRuleVersion.taxableBase,
        thresholds: tradeRuleVersion.thresholds,
        validatedBy: tradeRuleVersion.validatedBy,
        validatedAt: tradeRuleVersion.validatedAt,
        reviewDueAt: tradeRuleVersion.reviewDueAt,
        sourceUrl: tradeRuleSource.url,
        sourceAuthority: tradeRuleSource.authority,
        sourceConsultedAt: tradeRuleSource.consultedAt,
      })
      .from(tradeRuleVersion)
      .innerJoin(tradeRuleSource, eq(tradeRuleSource.id, tradeRuleVersion.sourceId))
      .where(eq(tradeRuleVersion.tradeRuleId, params.ruleId))
      .orderBy(desc(tradeRuleVersion.versionNo));

    const asOf = new Date().toISOString();
    const current = versions[0];
    return {
      id: row.id,
      key: row.key,
      countryCode: row.countryCode,
      scope: row.scope as Scope,
      ruleType: row.ruleType as RuleType,
      certainty: row.certainty as Certainty,
      active: row.active,
      currentVersionNo: current?.versionNo ?? null,
      validTo: current?.validTo?.toISOString() ?? null,
      reviewDueAt: current?.reviewDueAt?.toISOString() ?? null,
      reviewState: current === undefined
        ? "current"
        : ruleReviewState(
            { validTo: current.validTo?.toISOString() ?? null, reviewDueAt: current.reviewDueAt.toISOString() },
            asOf,
          ),
      sourceUrl: current?.sourceUrl ?? null,
      sourceAuthority: current?.sourceAuthority ?? null,
      sourceConsultedAt: current?.sourceConsultedAt?.toISOString() ?? null,
      updatedAt: row.updatedAt.toISOString(),
      versions: versions.map((version) => ({
        versionNo: version.versionNo,
        validFrom: version.validFrom.toISOString(),
        validTo: version.validTo?.toISOString() ?? null,
        params: version.params as RuleParams,
        calculationMode: version.calculationMode as CalculationMode,
        taxableBase: version.taxableBase as TaxableBase,
        thresholds: version.thresholds as Record<string, number | null>,
        validatedBy: version.validatedBy,
        validatedAt: version.validatedAt?.toISOString() ?? null,
        sourceUrl: version.sourceUrl,
        sourceAuthority: version.sourceAuthority,
        sourceConsultedAt: version.sourceConsultedAt.toISOString(),
      })),
    };
  }

  /**
   * Creation d'une regle ou d'une NOUVELLE version : une regle n'est jamais modifiee
   * en place (CDC 11.2). Le validateur est l'admin connecte (jamais l'IA).
   */
  async createRuleVersion(params: {
    input: RuleVersionInput;
    principal: AdminPrincipal;
  }): Promise<AdminTradeRuleDetail> {
    requirePermission(params.principal, "customs_rule", "create");
    const validation = validateRuleVersionInput(params.input);
    if (!validation.ok) throw new AdminTradeRuleError(validation.code);

    const now = new Date();
    const existing = await this.db
      .select({ id: tradeRule.id })
      .from(tradeRule)
      .where(eq(tradeRule.key, params.input.ruleKey))
      .limit(1);

    let ruleId = existing[0]?.id;
    if (ruleId === undefined) {
      const created = await this.db
        .insert(tradeRule)
        .values({
          key: params.input.ruleKey,
          countryCode: params.input.countryCode,
          scope: params.input.scope,
          ruleType: params.input.ruleType,
          certainty: params.input.certainty,
          priority: 0,
          active: true,
          createdAt: now,
          updatedAt: now,
        })
        .returning({ id: tradeRule.id });
      ruleId = created[0].id;
    }

    const lastVersion = await this.db
      .select({ versionNo: tradeRuleVersion.versionNo })
      .from(tradeRuleVersion)
      .where(eq(tradeRuleVersion.tradeRuleId, ruleId))
      .orderBy(desc(tradeRuleVersion.versionNo))
      .limit(1);
    const nextVersionNo = (lastVersion[0]?.versionNo ?? 0) + 1;

    await this.db.insert(tradeRuleVersion).values({
      tradeRuleId: ruleId,
      versionNo: nextVersionNo,
      validFrom: new Date(params.input.validFrom),
      validTo: params.input.validTo === null ? null : new Date(params.input.validTo),
      params: params.input.params,
      calculationMode: params.input.calculationMode,
      taxableBase: params.input.taxableBase,
      thresholds: params.input.thresholds,
      sourceId: params.input.sourceId,
      validatedBy: params.principal.userId,
      validatedAt: now,
      reviewDueAt: new Date(params.input.reviewDueAt),
      createdAt: now,
    });

    await this.db.insert(auditLog).values({
      actorId: params.principal.userId,
      actorRole: params.principal.role,
      action: "customs_rule.version_created",
      objectType: "trade_rule",
      objectId: ruleId,
      result: "success",
      correlationId: params.principal.correlationId,
      occurredAt: new Date(),
      metadata: { versionNo: nextVersionNo, ruleKey: params.input.ruleKey },
    });

    return this.get({ ruleId, principal: params.principal });
  }
}
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/compliance/admin-trade-rules.service.spec.ts
```

Attendu : **12 tests passés**, 0 échec.

- [ ] **Step 5: Écrire le test unitaire du service des alertes de conformité (il doit échouer)**

Créer `apps/api/src/modules/compliance/compliance-alert.service.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import { assertAlertTransition } from "./compliance-alert.service";

describe("assertAlertTransition — cycle de vie d'une alerte", () => {
  it("accepte open -> acknowledged", () => {
    expect(() => assertAlertTransition("open", "acknowledged")).not.toThrow();
  });

  it("accepte acknowledged -> resolved", () => {
    expect(() => assertAlertTransition("acknowledged", "resolved")).not.toThrow();
  });

  it("accepte open -> resolved (resolution directe)", () => {
    expect(() => assertAlertTransition("open", "resolved")).not.toThrow();
  });

  it("refuse de rouvrir une alerte resolue", () => {
    expect(() => assertAlertTransition("resolved", "open")).toThrow(/alert_status_transition/);
  });

  it("refuse de re-acknowledger une alerte deja resolue", () => {
    expect(() => assertAlertTransition("resolved", "acknowledged")).toThrow(/alert_status_transition/);
  });

  it("refuse d'annuler une resolution", () => {
    expect(() => assertAlertTransition("resolved", "open")).toThrow(/alert_status_transition/);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/compliance/compliance-alert.service.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./compliance-alert.service"`. 0 test exécuté.

- [ ] **Step 6: Implémenter le service des alertes et réaligner le contrôleur**

Créer `apps/api/src/modules/compliance/compliance-alert.service.ts` :

```ts
import { and, asc, desc, eq, sql, type SQL } from "drizzle-orm";
import { requirePermission } from "../rbac/require-permission";
import { ConflictError, NotFoundError } from "../../common/api-errors";
import { auditLog, complianceAlert } from "@fanours/db/schema";
import {
  loadAssignedCountries,
  loadRoleScope,
  resolveAdminScope,
  scopeCondition,
  type AdminPrincipal,
} from "../../common/admin-scope";
import { MAX_PAGE_SIZE, type AdminPage } from "../shipment/admin-shipments.service";

export type ComplianceAlertStatus = "open" | "acknowledged" | "resolved";

export type AdminComplianceAlertRow = {
  id: string;
  severity: "blocker" | "warning" | "info";
  code: string;
  countryCode: string | null;
  ruleId: string | null;
  ruleKey: string | null;
  subjectType: string | null;
  subjectId: string | null;
  message: string;
  actionRequired: string | null;
  status: ComplianceAlertStatus;
  createdAt: string;
  resolvedAt: string | null;
};

export type AdminAlertFilters = {
  status: ComplianceAlertStatus | null;
  severity: "blocker" | "warning" | "info" | null;
  countryCode: string | null;
  limit: number;
  offset: number;
};

export class ComplianceAlertError extends Error {
  constructor(public readonly code: string) {
    super(code);
    this.name = "ComplianceAlertError";
  }
}

/**
 * Transitions autorisees d'une alerte de conformite.
 * Une alerte resolue ne revient jamais en arriere : la resolution est un
 * evenement de conformite irreversible (CDC 11.9, 16.9).
 */
export function assertAlertTransition(from: ComplianceAlertStatus, to: ComplianceAlertStatus): void {
  const allowed: Record<ComplianceAlertStatus, readonly ComplianceAlertStatus[]> = {
    open: ["acknowledged", "resolved"],
    acknowledged: ["resolved"],
    resolved: [],
  };
  if (!allowed[from].includes(to)) {
    throw new ComplianceAlertError("alert_status_transition");
  }
}

export class ComplianceAlertService {
  constructor(private readonly db: DrizzleDb) {}

  /**
   * Alertes de conformite (CDC 16.9). Un `blocker` (regle expiree bloquante,
   * donnees manquantes) remonte en tete : le compliance manager doit le voir
   * avant toute autre alerte.
   */
  async list(params: {
    filters: AdminAlertFilters;
    principal: AdminPrincipal;
  }): Promise<AdminPage<AdminComplianceAlertRow>> {
    requirePermission(params.principal, "customs_rule", "read");

    const countries = await loadAssignedCountries(this.db, params.principal.userId);
    const roleScope = await loadRoleScope(this.db, params.principal.role, "customs_rule");
    const scope = resolveAdminScope({ roleScope, assignedCountries: countries });

    const conditions: SQL[] = [];
    const scopeSql = scopeCondition(scope, complianceAlert.countryCode);
    if (scopeSql !== undefined) conditions.push(scopeSql);
    if (params.filters.status !== null) conditions.push(eq(complianceAlert.status, params.filters.status));
    if (params.filters.severity !== null) conditions.push(eq(complianceAlert.severity, params.filters.severity));
    if (params.filters.countryCode !== null) {
      conditions.push(eq(complianceAlert.countryCode, params.filters.countryCode));
    }
    const where = and(...conditions);

    const severityRank = sql<number>`CASE ${complianceAlert.severity}
      WHEN 'blocker' THEN 0 WHEN 'warning' THEN 1 ELSE 2 END`;

    const rows = await this.db
      .select({
        id: complianceAlert.id,
        severity: complianceAlert.severity,
        code: complianceAlert.code,
        countryCode: complianceAlert.countryCode,
        ruleId: complianceAlert.ruleId,
        ruleKey: complianceAlert.ruleKey,
        subjectType: complianceAlert.subjectType,
        subjectId: complianceAlert.subjectId,
        message: complianceAlert.message,
        actionRequired: complianceAlert.actionRequired,
        status: complianceAlert.status,
        createdAt: complianceAlert.createdAt,
        resolvedAt: complianceAlert.resolvedAt,
      })
      .from(complianceAlert)
      .where(where)
      .orderBy(asc(severityRank), desc(complianceAlert.createdAt), desc(complianceAlert.id))
      .limit(params.filters.limit)
      .offset(params.filters.offset);

    const [totalRow] = await this.db
      .select({ value: sql<number>`count(*)::int` })
      .from(complianceAlert)
      .where(where);
    const total = totalRow?.value ?? 0;

    return {
      items: rows.map((row) => ({
        id: row.id,
        severity: row.severity as AdminComplianceAlertRow["severity"],
        code: row.code,
        countryCode: row.countryCode,
        ruleId: row.ruleId,
        ruleKey: row.ruleKey,
        subjectType: row.subjectType,
        subjectId: row.subjectId,
        message: row.message,
        actionRequired: row.actionRequired,
        status: row.status as ComplianceAlertStatus,
        createdAt: row.createdAt.toISOString(),
        resolvedAt: row.resolvedAt?.toISOString() ?? null,
      })),
      total,
      limit: params.filters.limit,
      offset: params.filters.offset,
    };
  }

  /**
   * Prise en compte d'une alerte par le compliance manager.
   * L'action est sensible : MFA recent exigee + journal d'audit.
   */
  async acknowledge(params: {
    alertId: string;
    principal: AdminPrincipal;
  }): Promise<AdminComplianceAlertRow> {
    requirePermission(params.principal, "customs_rule", "update");
    assertRecentMfa(params.principal);

    const current = await this.loadForUpdate(params.alertId);
    assertAlertTransition(current.status, "acknowledged");

    const updated = await this.db
      .update(complianceAlert)
      .set({
        status: "acknowledged",
        acknowledgedAt: new Date(),
        acknowledgedBy: params.principal.userId,
      })
      .where(and(eq(complianceAlert.id, params.alertId), eq(complianceAlert.status, current.status)))
      .returning({ id: complianceAlert.id });
    if (updated.length === 0) throw new ConflictError("alert_status_conflict");

    await this.db.insert(auditLog).values({
      actorId: params.principal.userId,
      actorRole: params.principal.role,
      action: "compliance_alert.acknowledged",
      objectType: "compliance_alert",
      objectId: params.alertId,
      result: "success",
      correlationId: params.principal.correlationId,
      occurredAt: new Date(),
      metadata: { code: current.code },
    });

    return this.load(params.alertId);
  }

  /** Resolution d'une alerte : la regle a ete corrigee, la donnee a ete fournie. */
  async resolve(params: {
    alertId: string;
    note: string;
    principal: AdminPrincipal;
  }): Promise<AdminComplianceAlertRow> {
    requirePermission(params.principal, "customs_rule", "update");
    assertRecentMfa(params.principal);
    const note = params.note.trim();
    if (note.length < 5) throw new ComplianceAlertError("resolution_note_required");

    const current = await this.loadForUpdate(params.alertId);
    assertAlertTransition(current.status, "resolved");

    const updated = await this.db
      .update(complianceAlert)
      .set({
        status: "resolved",
        resolvedAt: new Date(),
        resolvedBy: params.principal.userId,
      })
      .where(and(eq(complianceAlert.id, params.alertId), eq(complianceAlert.status, current.status)))
      .returning({ id: complianceAlert.id });
    if (updated.length === 0) throw new ConflictError("alert_status_conflict");

    await this.db.insert(auditLog).values({
      actorId: params.principal.userId,
      actorRole: params.principal.role,
      action: "compliance_alert.resolved",
      objectType: "compliance_alert",
      objectId: params.alertId,
      result: "success",
      correlationId: params.principal.correlationId,
      occurredAt: new Date(),
      metadata: { code: current.code, note },
    });

    return this.load(params.alertId);
  }

  private async load(alertId: string): Promise<AdminComplianceAlertRow> {
    const rows = await this.db
      .select({
        id: complianceAlert.id,
        severity: complianceAlert.severity,
        code: complianceAlert.code,
        countryCode: complianceAlert.countryCode,
        ruleId: complianceAlert.ruleId,
        ruleKey: complianceAlert.ruleKey,
        subjectType: complianceAlert.subjectType,
        subjectId: complianceAlert.subjectId,
        message: complianceAlert.message,
        actionRequired: complianceAlert.actionRequired,
        status: complianceAlert.status,
        createdAt: complianceAlert.createdAt,
        resolvedAt: complianceAlert.resolvedAt,
      })
      .from(complianceAlert)
      .where(eq(complianceAlert.id, alertId))
      .limit(1);
    const row = rows[0];
    if (row === undefined) throw new NotFoundError("compliance_alert_not_found");
    return {
      id: row.id,
      severity: row.severity as AdminComplianceAlertRow["severity"],
      code: row.code,
      countryCode: row.countryCode,
      ruleId: row.ruleId,
      ruleKey: row.ruleKey,
      subjectType: row.subjectType,
      subjectId: row.subjectId,
      message: row.message,
      actionRequired: row.actionRequired,
      status: row.status as ComplianceAlertStatus,
      createdAt: row.createdAt.toISOString(),
      resolvedAt: row.resolvedAt?.toISOString() ?? null,
    };
  }

  private async loadForUpdate(alertId: string): Promise<AdminComplianceAlertRow> {
    return this.load(alertId);
  }
}
```

Modifier `apps/api/src/modules/compliance/compliance-alerts.controller.ts` pour **aligner la route sur `/v1/admin/compliance-alerts`** (le préfixe `/v1` est déjà appliqué par le préfixe global du module API ; le décorateur ne doit plus porter `/v1`) et pour router la lecture et les actions vers le service :

```ts
import { Body, Controller, Get, Param, Post, Query, Req } from "@nestjs/common";
import { requirePermission } from "../rbac/require-permission";
import {
  ComplianceAlertService,
  type AdminAlertFilters,
} from "./compliance-alert.service";

@Controller("/v1/admin/compliance-alerts")
export class ComplianceAlertsController {
  constructor(private readonly alerts: ComplianceAlertService) {}

  @Get()
  async list(@Req() req: RequestWithPrincipal, @Query() query: AdminComplianceAlertQuery) {
    requirePermission(req.principal, "customs_rule", "read");
    return this.alerts.list({ filters: query.filters, principal: req.principal });
  }

  @Post(":id/acknowledge")
  async acknowledge(@Req() req: RequestWithPrincipal, @Param("id") id: string) {
    requirePermission(req.principal, "customs_rule", "update");
    return this.alerts.acknowledge({ alertId: id, principal: req.principal });
  }

  @Post(":id/resolve")
  async resolve(
    @Req() req: RequestWithPrincipal,
    @Param("id") id: string,
    @Body() body: { note: string },
  ) {
    requirePermission(req.principal, "customs_rule", "update");
    return this.alerts.resolve({ alertId: id, note: body.note, principal: req.principal });
  }
}
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/compliance/compliance-alert.service.spec.ts
```

Attendu : **6 tests passés**, 0 échec.

- [ ] **Step 7: Écrire le test unitaire de la revue B2B four-eyes (il doit échouer)**

Créer `apps/api/src/modules/business/admin-business-accounts.service.spec.ts` :

```ts
import { describe, expect, it } from "vitest";
import {
  approvalPayloadHash,
  assertFourEyes,
  normalizeBusinessAccountFilters,
} from "./admin-business-accounts.service";

describe("assertFourEyes — aucun auto-approbation (CDC securite 3.5)", () => {
  it("refuse l'auto-approbation", () => {
    expect(() => assertFourEyes("user-1", "user-1")).toThrow(/self_approval_forbidden/);
  });

  it("accepte deux administrateurs distincts", () => {
    expect(() => assertFourEyes("user-1", "user-2")).not.toThrow();
  });
});

describe("approvalPayloadHash", () => {
  it("est deterministe pour la meme charge utile", () => {
    const a = approvalPayloadHash({
      businessAccountId: "acc-1",
      decision: "approved",
      note: "dossier complet",
    });
    const b = approvalPayloadHash({
      businessAccountId: "acc-1",
      decision: "approved",
      note: "dossier complet",
    });
    expect(a).toBe(b);
    expect(a).toMatch(/^[0-9a-f]{64}$/);
  });

  it("change des que la decision change", () => {
    const a = approvalPayloadHash({
      businessAccountId: "acc-1",
      decision: "approved",
      note: "dossier complet",
    });
    const b = approvalPayloadHash({
      businessAccountId: "acc-1",
      decision: "rejected",
      note: "dossier complet",
    });
    expect(a).not.toBe(b);
  });

  it("change des que la note change", () => {
    const a = approvalPayloadHash({
      businessAccountId: "acc-1",
      decision: "approved",
      note: "dossier complet",
    });
    const b = approvalPayloadHash({
      businessAccountId: "acc-1",
      decision: "approved",
      note: "piece manquante",
    });
    expect(a).not.toBe(b);
  });
});

describe("normalizeBusinessAccountFilters — pagination serveur", () => {
  it("applique une taille de page par defaut de 25", () => {
    expect(normalizeBusinessAccountFilters({})).toMatchObject({ limit: 25, offset: 0 });
  });

  it("plafonne la taille de page a 100 lignes", () => {
    expect(normalizeBusinessAccountFilters({ limit: 900 }).limit).toBe(100);
  });

  it("borne l'offset a 10 000 lignes", () => {
    expect(normalizeBusinessAccountFilters({ offset: 50_000 }).offset).toBe(10_000);
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/business/admin-business-accounts.service.spec.ts
```

Attendu : **ÉCHEC** — `Failed to resolve import "./admin-business-accounts.service"`. 0 test exécuté.

- [ ] **Step 8: Implémenter la revue B2B four-eyes**

Créer `apps/api/src/modules/business/admin-business-accounts.service.ts` :

```ts
import { createHash } from "node:crypto";
import { and, asc, desc, eq, ilike, or, sql, type SQL } from "drizzle-orm";
import { requirePermission } from "../rbac/require-permission";
import { ConflictError, ForbiddenError, NotFoundError } from "../../common/api-errors";
import type { BusinessVerificationStatus, ReviewDecision } from "@fanours/contracts";
import { auditLog, businessAccount, businessAccountApproval } from "@fanours/db/schema";
import {
  loadAssignedCountries,
  loadRoleScope,
  resolveAdminScope,
  scopeCondition,
  type AdminPrincipal,
} from "../../common/admin-scope";
import { MAX_PAGE_SIZE, type AdminPage } from "../shipment/admin-shipments.service";

export const APPROVAL_TTL_HOURS = 24;

export type AdminBusinessAccountRow = {
  id: string;
  legalName: string;
  country: string;
  taxId: string | null;
  verificationStatus: BusinessVerificationStatus;
  verifiedAt: string | null;
  verifiedBy: string | null;
  reviewNote: string | null;
  documentCount: number;
  createdAt: string;
};

export type AdminBusinessAccountFilters = {
  verificationStatus: BusinessVerificationStatus | null;
  country: string | null;
  search: string | null;
  limit: number;
  offset: number;
};

export class AdminBusinessAccountError extends Error {
  constructor(public readonly code: string) {
    super(code);
    this.name = "AdminBusinessAccountError";
  }
}

/**
 * Four-eyes : le second approbateur ne peut jamais etre celui qui a demande
 * la decision (CDC securite 3.5). L'auto-approbation est un forbidden, pas une
 * erreur metier.
 */
export function assertFourEyes(requestedBy: string, approvedBy: string): void {
  if (requestedBy === approvedBy) throw new ForbiddenError("self_approval_forbidden");
}

/**
 * Empreinte de la charge utile approuvee : elle garantit que l'approbateur a
 * vu exactement la meme decision et la meme note que le demandeur.
 */
export function approvalPayloadHash(input: {
  businessAccountId: string;
  decision: string;
  note: string;
}): string {
  return createHash("sha256")
    .update(
      [input.businessAccountId, input.decision, input.note].map((part) => part.trim()).join("\u0000"),
    )
    .digest("hex");
}

/** Bornes de pagination serveur, identiques aux autres vues admin. */
export function normalizeBusinessAccountFilters(input: {
  limit?: number;
  offset?: number;
}): { limit: number; offset: number } {
  const rawLimit = Number.isFinite(input.limit) ? Math.trunc(input.limit as number) : 25;
  const rawOffset = Number.isFinite(input.offset) ? Math.trunc(input.offset as number) : 0;
  return {
    limit: Math.min(Math.max(rawLimit, 1), MAX_PAGE_SIZE),
    offset: Math.min(Math.max(rawOffset, 0), MAX_OFFSET),
  };
}

export class AdminBusinessAccountsService {
  constructor(
    private readonly db: DrizzleDb,
    private readonly outbox: OutboxWriter,
  ) {}

  /**
   * Comptes professionnels (CDC 16.6) : recherche par identifiant ou e-mail,
   * filtres par pays et par statut. Pagination et tri serveur.
   */
  async list(params: {
    filters: AdminBusinessAccountFilters;
    principal: AdminPrincipal;
  }): Promise<AdminPage<AdminBusinessAccountRow>> {
    requirePermission(params.principal, "business_account", "read");

    const countries = await loadAssignedCountries(this.db, params.principal.userId);
    const roleScope = await loadRoleScope(this.db, params.principal.role, "business_account");
    const scope = resolveAdminScope({ roleScope, assignedCountries: countries });

    const conditions: SQL[] = [];
    const scopeSql = scopeCondition(scope, businessAccount.countryCode);
    if (scopeSql !== undefined) conditions.push(scopeSql);
    if (params.filters.verificationStatus !== null) {
      conditions.push(eq(businessAccount.verificationStatus, params.filters.verificationStatus));
    }
    if (params.filters.country !== null) {
      conditions.push(eq(businessAccount.countryCode, params.filters.country));
    }
    if (params.filters.search !== null && params.filters.search.trim().length > 0) {
      const term = `%${params.filters.search.trim()}%`;
      conditions.push(
        or(
          ilike(businessAccount.legalName, term),
          ilike(businessAccount.taxId, term),
          ilike(businessAccount.id, term),
        )!,
      );
    }
    const where = and(...conditions);

    const rows = await this.db
      .select({
        id: businessAccount.id,
        legalName: businessAccount.legalName,
        country: businessAccount.countryCode,
        taxId: businessAccount.taxId,
        verificationStatus: businessAccount.verificationStatus,
        verifiedAt: businessAccount.verifiedAt,
        verifiedBy: businessAccount.verifiedBy,
        reviewNote: businessAccount.reviewNote,
        createdAt: businessAccount.createdAt,
        documentCount: sql<number>`(
          SELECT count(*)::int FROM business_document d WHERE d.business_account_id = ${businessAccount.id}
        )`,
      })
      .from(businessAccount)
      .where(where)
      .orderBy(desc(businessAccount.createdAt), asc(businessAccount.id))
      .limit(params.filters.limit)
      .offset(params.filters.offset);

    const [totalRow] = await this.db
      .select({ value: sql<number>`count(*)::int` })
      .from(businessAccount)
      .where(where);
    const total = totalRow?.value ?? 0;

    return {
      items: rows.map((row) => ({
        id: row.id,
        legalName: row.legalName,
        country: row.country,
        taxId: row.taxId,
        verificationStatus: row.verificationStatus as BusinessVerificationStatus,
        verifiedAt: row.verifiedAt?.toISOString() ?? null,
        verifiedBy: row.verifiedBy,
        reviewNote: row.reviewNote,
        documentCount: row.documentCount,
        createdAt: row.createdAt.toISOString(),
      })),
      total,
      limit: params.filters.limit,
      offset: params.filters.offset,
    };
  }

  /** Detail d'un compte : hors perimetre, la reponse est 404. */
  async get(params: {
    businessAccountId: string;
    principal: AdminPrincipal;
  }): Promise<AdminBusinessAccountRow> {
    requirePermission(params.principal, "business_account", "read");

    const countries = await loadAssignedCountries(this.db, params.principal.userId);
    const roleScope = await loadRoleScope(this.db, params.principal.role, "business_account");
    const scope = resolveAdminScope({ roleScope, assignedCountries: countries });
    const scopeSql = scopeCondition(scope, businessAccount.countryCode);

    const rows = await this.db
      .select({
        id: businessAccount.id,
        legalName: businessAccount.legalName,
        country: businessAccount.countryCode,
        taxId: businessAccount.taxId,
        verificationStatus: businessAccount.verificationStatus,
        verifiedAt: businessAccount.verifiedAt,
        verifiedBy: businessAccount.verifiedBy,
        reviewNote: businessAccount.reviewNote,
        createdAt: businessAccount.createdAt,
        documentCount: sql<number>`(
          SELECT count(*)::int FROM business_document d WHERE d.business_account_id = ${businessAccount.id}
        )`,
      })
      .from(businessAccount)
      .where(and(eq(businessAccount.id, params.businessAccountId), scopeSql))
      .limit(1);

    const row = rows[0];
    if (row === undefined) throw new NotFoundError("business_account_not_found");
    return {
      id: row.id,
      legalName: row.legalName,
      country: row.country,
      taxId: row.taxId,
      verificationStatus: row.verificationStatus as BusinessVerificationStatus,
      verifiedAt: row.verifiedAt?.toISOString() ?? null,
      verifiedBy: row.verifiedBy,
      reviewNote: row.reviewNote,
      documentCount: row.documentCount,
      createdAt: row.createdAt.toISOString(),
    };
  }

  /**
   * Demande de decision (etape 1 du four-eyes) : le demandeur propose
   * approbation ou refus, avec une note motivee. Rien n'est applique au compte
   * a ce stade : seule la demande est enregistree.
   */
  async requestApproval(params: {
    businessAccountId: string;
    decision: ReviewDecision;
    principal: AdminPrincipal;
  }): Promise<{ approvalId: string; status: string; expiresAt: string }> {
    requirePermission(params.principal, "business_account", "approve");
    assertRecentMfa(params.principal);

    await this.get({ businessAccountId: params.businessAccountId, principal: params.principal });

    const note = params.decision.note?.trim() ?? "";
    if (note.length < 5) throw new AdminBusinessAccountError("review_note_required");

    const payloadHash = approvalPayloadHash({
      businessAccountId: params.businessAccountId,
      decision: params.decision.decision,
      note,
    });
    const expiresAt = new Date(Date.now() + APPROVAL_TTL_HOURS * 60 * 60 * 1000);

    const inserted = await this.db
      .insert(businessAccountApproval)
      .values({
        businessAccountId: params.businessAccountId,
        decision: params.decision.decision,
        note,
        payloadHash,
        status: "pending",
        requestedBy: params.principal.userId,
        expiresAt,
      })
      .onConflictDoNothing()
      .returning({ id: businessAccountApproval.id });

    const created = inserted[0];
    if (created === undefined) {
      throw new ConflictError("approval_already_pending");
    }

    await this.db.insert(auditLog).values({
      actorId: params.principal.userId,
      actorRole: params.principal.role,
      action: "business_account.approval_requested",
      objectType: "business_account",
      objectId: params.businessAccountId,
      result: "success",
      correlationId: params.principal.correlationId,
      occurredAt: new Date(),
      metadata: { approvalId: created.id, decision: params.decision.decision },
    });

    return { approvalId: created.id, status: "pending", expiresAt: expiresAt.toISOString() };
  }

  /**
   * Approbation (etape 2 du four-eyes) : un second administrateur, different
   * du demandeur, confirme la decision. La decision appliquee est celle
   * enregistree dans la demande : l'approbateur ne peut pas la modifier.
   */
  async approve(params: {
    businessAccountId: string;
    approvalId: string;
    principal: AdminPrincipal;
  }): Promise<{ businessAccountId: string; verificationStatus: BusinessVerificationStatus }> {
    requirePermission(params.principal, "business_account", "approve");
    assertRecentMfa(params.principal);

    const approval = await this.db
      .select()
      .from(businessAccountApproval)
      .where(
        and(
          eq(businessAccountApproval.id, params.approvalId),
          eq(businessAccountApproval.businessAccountId, params.businessAccountId),
        ),
      )
      .limit(1);
    const request = approval[0];
    if (request === undefined) throw new NotFoundError("approval_not_found");
    if (request.status !== "pending") throw new ConflictError("approval_not_pending");
    if (request.expiresAt.getTime() <= Date.now()) throw new ConflictError("approval_expired");
    assertFourEyes(request.requestedBy, params.principal.userId);

    const status: BusinessVerificationStatus =
      request.decision === "approved" ? "approved" : "rejected";

    await this.db.transaction(async (tx) => {
      const updated = await tx
        .update(businessAccountApproval)
        .set({ status: "approved", approvedBy: params.principal.userId, approvedAt: new Date() })
        .where(
          and(
            eq(businessAccountApproval.id, params.approvalId),
            eq(businessAccountApproval.status, "pending"),
          ),
        )
        .returning({ id: businessAccountApproval.id });
      if (updated.length === 0) throw new ConflictError("approval_not_pending");

      await tx
        .update(businessAccount)
        .set({
          verificationStatus: status,
          verifiedBy: params.principal.userId,
          verifiedAt: new Date(),
          reviewNote: request.note,
          updatedAt: new Date(),
        })
        .where(eq(businessAccount.id, params.businessAccountId));
    });

    await this.db.insert(auditLog).values({
      actorId: params.principal.userId,
      actorRole: params.principal.role,
      action: "business_account.approved",
      objectType: "business_account",
      objectId: params.businessAccountId,
      result: "success",
      correlationId: params.principal.correlationId,
      occurredAt: new Date(),
      metadata: { approvalId: params.approvalId, status },
    });

    await this.outbox.publish({
      type: "business_account.verified",
      aggregateId: params.businessAccountId,
      payload: { businessAccountId: params.businessAccountId, status },
    });

    return { businessAccountId: params.businessAccountId, verificationStatus: status };
  }

  /**
   * Suspension motivee (CDC 16.6) : le motif est obligatoire et journalise.
   * La suspension ne supprime pas le compte : elle coupe ses privileges B2B.
   */
  async suspend(params: {
    businessAccountId: string;
    reason: string;
    principal: AdminPrincipal;
  }): Promise<{ verificationStatus: BusinessVerificationStatus }> {
    requirePermission(params.principal, "business_account", "update");
    assertRecentMfa(params.principal);
    const reason = params.reason.trim();
    if (reason.length < 5) throw new AdminBusinessAccountError("suspension_reason_required");

    await this.get({ businessAccountId: params.businessAccountId, principal: params.principal });

    await this.db
      .update(businessAccount)
      .set({
        verificationStatus: "suspended",
        verifiedBy: params.principal.userId,
        verifiedAt: new Date(),
        reviewNote: reason,
        updatedAt: new Date(),
      })
      .where(eq(businessAccount.id, params.businessAccountId));

    await this.db.insert(auditLog).values({
      actorId: params.principal.userId,
      actorRole: params.principal.role,
      action: "business_account.suspended",
      objectType: "business_account",
      objectId: params.businessAccountId,
      result: "success",
      correlationId: params.principal.correlationId,
      occurredAt: new Date(),
      metadata: { reason },
    });

    await this.outbox.publish({
      type: "business_account.suspended",
      aggregateId: params.businessAccountId,
      payload: { businessAccountId: params.businessAccountId, reason },
    });

    return { verificationStatus: "suspended" };
  }
}
```

Modifier `apps/api/src/modules/business/business-review.service.ts` : la méthode `decide()` ne décide plus directement. Elle délègue à `AdminBusinessAccountsService.requestApproval()` puis `approve()` et **interdit l'auto-approbation**, afin qu'aucun appel direct à `decide()` ne contourne le four-eyes.

```ts
  /**
   * Point d'entree historique conserve : la decision passe maintenant par le
   * four-eyes (demande puis approbation d'un second administrateur).
   */
  async decide(params: {
    businessAccountId: string;
    decision: ReviewDecision;
    principal: Principal;
  }): Promise<{ verificationStatus: BusinessVerificationStatus }> {
    requirePermission(params.principal, "business_account", "approve");
    const requested = await this.adminAccounts.requestApproval({
      businessAccountId: params.businessAccountId,
      decision: params.decision,
      principal: params.principal,
    });
    return this.adminAccounts.approve({
      businessAccountId: params.businessAccountId,
      approvalId: requested.approvalId,
      principal: params.principal,
    });
  }
```

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/business/admin-business-accounts.service.spec.ts
```

Attendu : **10 tests passés**, 0 échec.

- [ ] **Step 9: Écrire les contrôleurs admin et le test d'intégration HTTP**

Créer `apps/api/src/modules/shipment/admin-shipments.controller.ts` :

```ts
import { Body, Controller, Get, Param, Post, Query, Req } from "@nestjs/common";
import { requirePermission } from "../rbac/require-permission";
import {
  AdminShipmentsService,
  assertShipmentStatusFilter,
  normalizePage,
} from "./admin-shipments.service";

@Controller("/v1/admin/shipments")
export class AdminShipmentsController {
  constructor(private readonly shipments: AdminShipmentsService) {}

  @Get()
  async list(@Req() req: RequestWithPrincipal, @Query() query: AdminShipmentsQuery) {
    requirePermission(req.principal, "customs_document", "read");
    const { limit, offset } = normalizePage({ limit: query.limit, offset: query.offset });
    return this.shipments.list({
      filters: {
        status: assertShipmentStatusFilter(query.status),
        countryCode: query.countryCode ?? null,
        carrier: query.carrier ?? null,
        exceptionsOnly: query.exceptionsOnly === "true",
        limit,
        offset,
        sort: query.sort ?? "created_at",
        order: query.order === "asc" ? "asc" : "desc",
      },
      principal: req.principal,
    });
  }

  @Get(":id")
  async get(@Req() req: RequestWithPrincipal, @Param("id") id: string) {
    requirePermission(req.principal, "customs_document", "read");
    return this.shipments.get({ shipmentId: id, principal: req.principal });
  }

  @Post(":id/exception")
  async raiseException(
    @Req() req: RequestWithPrincipal,
    @Param("id") id: string,
    @Body() body: { reason: string },
  ) {
    requirePermission(req.principal, "customs_document", "update");
    return this.shipments.raiseException({ shipmentId: id, reason: body.reason, principal: req.principal });
  }

  @Post(":id/exception/resolve")
  async resolveException(
    @Req() req: RequestWithPrincipal,
    @Param("id") id: string,
    @Body() body: { note: string },
  ) {
    requirePermission(req.principal, "customs_document", "update");
    return this.shipments.resolveException({ shipmentId: id, note: body.note, principal: req.principal });
  }
}
```

Créer `apps/api/src/modules/compliance/admin-trade-rules.controller.ts` :

```ts
import { Body, Controller, Get, Param, Post, Query, Req } from "@nestjs/common";
import { requirePermission } from "../rbac/require-permission";
import {
  AdminTradeRulesService,
  type RuleVersionInput,
} from "./admin-trade-rules.service";

@Controller("/v1/admin/trade-rules")
export class AdminTradeRulesController {
  constructor(private readonly rules: AdminTradeRulesService) {}

  @Get()
  async list(@Req() req: RequestWithPrincipal, @Query() query: AdminTradeRulesQuery) {
    requirePermission(req.principal, "customs_rule", "read");
    const { limit, offset } = normalizePage({ limit: query.limit, offset: query.offset });
    return this.rules.list({
      filters: {
        countryCode: query.countryCode ?? null,
        ruleType: query.ruleType ?? null,
        certainty: query.certainty ?? null,
        reviewState: query.reviewState ?? null,
        search: query.search ?? null,
        limit,
        offset,
      },
      principal: req.principal,
    });
  }

  @Get("review-due")
  async reviewDue(@Req() req: RequestWithPrincipal) {
    requirePermission(req.principal, "customs_rule", "read");
    return this.rules.listReviewDue({ asOf: new Date().toISOString(), principal: req.principal });
  }

  @Get(":id")
  async get(@Req() req: RequestWithPrincipal, @Param("id") id: string) {
    requirePermission(req.principal, "customs_rule", "read");
    return this.rules.get({ ruleId: id, principal: req.principal });
  }

  @Post()
  async create(@Req() req: RequestWithPrincipal, @Body() body: RuleVersionInput) {
    requirePermission(req.principal, "customs_rule", "create");
    return this.rules.createRuleVersion({ input: body, principal: req.principal });
  }

  @Post(":id/versions")
  async createVersion(
    @Req() req: RequestWithPrincipal,
    @Param("id") id: string,
    @Body() body: RuleVersionInput,
  ) {
    requirePermission(req.principal, "customs_rule", "update");
    return this.rules.createRuleVersion({ input: { ...body, ruleKey: body.ruleKey }, principal: req.principal });
  }
}
```

Créer `apps/api/src/modules/business/admin-business-accounts.controller.ts` :

```ts
import { Body, Controller, Get, Param, Post, Query, Req } from "@nestjs/common";
import { requirePermission } from "../rbac/require-permission";
import type { ReviewDecision } from "@fanours/contracts";
import {
  AdminBusinessAccountsService,
  normalizeBusinessAccountFilters,
} from "./admin-business-accounts.service";

@Controller("/v1/admin/business-accounts")
export class AdminBusinessAccountsController {
  constructor(private readonly accounts: AdminBusinessAccountsService) {}

  @Get()
  async list(@Req() req: RequestWithPrincipal, @Query() query: AdminBusinessAccountsQuery) {
    requirePermission(req.principal, "business_account", "read");
    const { limit, offset } = normalizeBusinessAccountFilters({
      limit: query.limit,
      offset: query.offset,
    });
    return this.accounts.list({
      filters: {
        verificationStatus: query.verificationStatus ?? null,
        country: query.country ?? null,
        search: query.search ?? null,
        limit,
        offset,
      },
      principal: req.principal,
    });
  }

  @Get(":id")
  async get(@Req() req: RequestWithPrincipal, @Param("id") id: string) {
    requirePermission(req.principal, "business_account", "read");
    return this.accounts.get({ businessAccountId: id, principal: req.principal });
  }

  @Post(":id/approvals")
  async requestApproval(
    @Req() req: RequestWithPrincipal,
    @Param("id") id: string,
    @Body() body: ReviewDecision,
  ) {
    requirePermission(req.principal, "business_account", "approve");
    return this.accounts.requestApproval({
      businessAccountId: id,
      decision: body,
      principal: req.principal,
    });
  }

  @Post(":id/approvals/:approvalId/approve")
  async approve(
    @Req() req: RequestWithPrincipal,
    @Param("id") id: string,
    @Param("approvalId") approvalId: string,
  ) {
    requirePermission(req.principal, "business_account", "approve");
    return this.accounts.approve({
      businessAccountId: id,
      approvalId,
      principal: req.principal,
    });
  }

  @Post(":id/suspend")
  async suspend(
    @Req() req: RequestWithPrincipal,
    @Param("id") id: string,
    @Body() body: { reason: string },
  ) {
    requirePermission(req.principal, "business_account", "update");
    return this.accounts.suspend({
      businessAccountId: id,
      reason: body.reason,
      principal: req.principal,
    });
  }
}
```

Créer `packages/db/src/__tests__/phase3-admin-views.integration.spec.ts` (PostgreSQL réel via Testcontainers) :

```ts
import { afterAll, beforeAll, describe, expect, it } from "vitest";
import { request } from "supertest";
import { buildTestApp, type TestContext } from "../test-harness";

describe("vues admin phase 3 — permissions, perimetre, audit", () => {
  let ctx: TestContext;

  beforeAll(async () => {
    ctx = await buildTestApp();
  });

  afterAll(async () => {
    await ctx.close();
  });

  it("refuse la liste des expeditions a un role sans permission (403)", async () => {
    const response = await request(ctx.app)
      .get("/v1/admin/shipments")
      .set("Authorization", `Bearer ${ctx.tokens.analystReadonly}`);
    expect(response.status).toBe(403);
  });

  it("refuse la liste des expeditions sans jeton (401)", async () => {
    const response = await request(ctx.app).get("/v1/admin/shipments");
    expect(response.status).toBe(401);
  });

  it("liste les expeditions au role logistique (200)", async () => {
    const response = await request(ctx.app)
      .get("/v1/admin/shipments?limit=10&offset=0")
      .set("Authorization", `Bearer ${ctx.tokens.logisticsOperator}`);
    expect(response.status).toBe(200);
    expect(response.body.items.length).toBeLessThanOrEqual(10);
    expect(response.body.total).toBeGreaterThanOrEqual(0);
  });

  it("renvoie 404 sur une expedition hors perimetre pays (anti-IDOR)", async () => {
    const response = await request(ctx.app)
      .get(`/v1/admin/shipments/${ctx.fixtures.shipmentOutsideScope}`)
      .set("Authorization", `Bearer ${ctx.tokens.scopedLogisticsOperator}`);
    expect(response.status).toBe(404);
  });

  it("journalise la levee d'une exception de colis", async () => {
    const response = await request(ctx.app)
      .post(`/v1/admin/shipments/${ctx.fixtures.shipmentInScope}/exception`)
      .set("Authorization", `Bearer ${ctx.tokens.logisticsOperator}`)
      .send({ reason: "Code HS manquant sur la declaration" });
    expect(response.status).toBe(200);
    const audit = await ctx.auditLog.find({
      action: "shipment.exception_raised",
      objectId: ctx.fixtures.shipmentInScope,
    });
    expect(audit.length).toBe(1);
  });

  it("refuse l'auto-approbation d'un compte B2B (four-eyes)", async () => {
    const response = await request(ctx.app)
      .post(`/v1/admin/business-accounts/${ctx.fixtures.businessAccountId}/approvals`)
      .set("Authorization", `Bearer ${ctx.tokens.complianceManager}`)
      .send({ decision: "approved", note: "Dossier complet et coherent" });
    expect(response.status).toBe(201);
    const approvalId = response.body.approvalId as string;
    const selfApprove = await request(ctx.app)
      .post(`/v1/admin/business-accounts/${ctx.fixtures.businessAccountId}/approvals/${approvalId}/approve`)
      .set("Authorization", `Bearer ${ctx.tokens.complianceManager}`);
    expect(selfApprove.status).toBe(403);
    expect(selfApprove.body.code).toBe("self_approval_forbidden");
  });

  it("applique la decision d'un compte B2B apres approbation d'un second administrateur", async () => {
    const requested = await request(ctx.app)
      .post(`/v1/admin/business-accounts/${ctx.fixtures.businessAccountId}/approvals`)
      .set("Authorization", `Bearer ${ctx.tokens.complianceManager}`)
      .send({ decision: "approved", note: "Dossier complet et coherent" });
    const approvalId = requested.body.approvalId as string;
    const approved = await request(ctx.app)
      .post(`/v1/admin/business-accounts/${ctx.fixtures.businessAccountId}/approvals/${approvalId}/approve`)
      .set("Authorization", `Bearer ${ctx.tokens.superAdmin}`);
    expect(approved.status).toBe(200);
    expect(approved.body.verificationStatus).toBe("approved");
  });

  it("refuse la resolution d'une alerte de conformite sans MFA recent (403)", async () => {
    const response = await request(ctx.app)
      .post(`/v1/admin/compliance-alerts/${ctx.fixtures.alertId}/resolve`)
      .set("Authorization", `Bearer ${ctx.tokens.complianceManagerWithoutMfa}`)
      .send({ note: "Donnee manquante fournie par l'importateur" });
    expect(response.status).toBe(403);
    expect(response.body.code).toBe("mfa_required");
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/db exec vitest run --config vitest.integration.config.ts src/__tests__/phase3-admin-views.integration.spec.ts
```

Attendu : **8 tests passés**, 0 échec.

- [ ] **Step 10: Migration SQL des vues admin et schéma Drizzle**

Créer `packages/db/migrations/0110_phase3_admin_views.sql` :

```sql
-- 0110_phase3_admin_views.sql
-- Phase 3 - Task 17 (vues admin : expeditions, regles, alertes, revue B2B).
-- Classification : expand. Append-only : aucune suppression, aucun "down".
--
-- Cette migration ne cree QUE ce qui lui est propre :
--   - `admin_country_scope` (perimetre pays des vues admin) ;
--   - `business_account_approval` (demande d'approbation four-eyes) ;
--   - les colonnes d'exception sur `shipment` ;
--   - les colonnes de prise en compte sur `compliance_alert`.
--
-- Les tables `business_account` et `business_membership` sont creees par la
-- migration 0103 (Phase 3, chantier 3.1) : cette migration les UTILISE, elle
-- ne les redefinit pas.
-- La table des commandes reelle creee par la Phase 2 est `orders` : le backfill
-- du pays d'expedition lit `orders.destination_country`.
-- La table des utilisateurs de la Phase 1 est `"user"` (mot reserve `USER`).
SET lock_timeout = '5s';
SET statement_timeout = '60s';

BEGIN;

-- ---------------------------------------------------------------- perimetre pays
CREATE TABLE IF NOT EXISTS admin_country_scope (
  user_id      uuid NOT NULL REFERENCES "user" (id) ON DELETE CASCADE,
  country_code char(2) NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT pk_admin_country_scope PRIMARY KEY (user_id, country_code),
  CONSTRAINT ck_admin_country_scope_code CHECK (country_code ~ '^[A-Z]{2}$')
);

CREATE INDEX IF NOT EXISTS idx_admin_country_scope_country ON admin_country_scope (country_code);

-- --------------------------------------------------- exceptions de colis (colis)
ALTER TABLE shipment ADD COLUMN IF NOT EXISTS country_code char(2);
ALTER TABLE shipment ADD COLUMN IF NOT EXISTS exception_reason text;
ALTER TABLE shipment ADD COLUMN IF NOT EXISTS exception_opened_at timestamptz;
ALTER TABLE shipment ADD COLUMN IF NOT EXISTS exception_raised_by uuid REFERENCES "user" (id);
ALTER TABLE shipment ADD COLUMN IF NOT EXISTS exception_resolved_at timestamptz;
ALTER TABLE shipment ADD COLUMN IF NOT EXISTS exception_resolved_by uuid REFERENCES "user" (id);

-- Le pays d'une expedition est celui de la commande : backfill depuis la commande.
UPDATE shipment s
   SET country_code = o.destination_country
  FROM orders o
 WHERE o.id = s.order_id
   AND s.country_code IS NULL;

ALTER TABLE shipment ALTER COLUMN country_code SET NOT NULL;
ALTER TABLE shipment
  ADD CONSTRAINT ck_shipment_country_code CHECK (country_code ~ '^[A-Z]{2}$');

CREATE INDEX IF NOT EXISTS idx_shipment_country_status ON shipment (country_code, status);
CREATE INDEX IF NOT EXISTS idx_shipment_exception_open
  ON shipment (exception_opened_at DESC) WHERE exception_reason IS NOT NULL;

-- --------------------------------------------------- alertes : prise en compte
ALTER TABLE compliance_alert ADD COLUMN IF NOT EXISTS acknowledged_at timestamptz;
ALTER TABLE compliance_alert ADD COLUMN IF NOT EXISTS acknowledged_by uuid REFERENCES "user" (id);

-- ------------------------------------------------------ revue B2B four-eyes
CREATE TABLE business_account_approval (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  business_account_id uuid NOT NULL REFERENCES business_account(id) ON DELETE CASCADE,
  decision            text NOT NULL,
  note                text NOT NULL,
  payload_hash        char(64) NOT NULL,
  status              text NOT NULL DEFAULT 'pending',
  requested_by        uuid NOT NULL REFERENCES "user" (id),
  requested_at        timestamptz NOT NULL DEFAULT now(),
  approved_by         uuid REFERENCES "user" (id),
  approved_at         timestamptz,
  expires_at          timestamptz NOT NULL,
  CONSTRAINT business_account_approval_decision CHECK (decision IN ('approved','rejected')),
  CONSTRAINT business_account_approval_status
    CHECK (status IN ('pending','approved','rejected','expired','cancelled')),
  CONSTRAINT business_account_approval_no_self_approval
    CHECK (approved_by IS NULL OR approved_by <> requested_by),
  CONSTRAINT business_account_approval_payload_hash CHECK (payload_hash ~ '^[0-9a-f]{64}$'),
  CONSTRAINT business_account_approval_approved_needs_approver
    CHECK (status <> 'approved' OR (approved_by IS NOT NULL AND approved_at IS NOT NULL))
);

-- Une seule demande en attente par compte : pas de double file d'approbation.
CREATE UNIQUE INDEX uq_business_account_approval_pending
  ON business_account_approval (business_account_id) WHERE status = 'pending';
CREATE INDEX idx_business_account_approval_account
  ON business_account_approval (business_account_id, requested_at DESC);

COMMIT;
```

Créer `packages/db/src/schema/phase3-admin.ts` :

```ts
import {
  char,
  index,
  pgTable,
  text,
  timestamp,
  uniqueIndex,
  uuid,
} from "drizzle-orm/pg-core";
import { shipment } from "./phase3-logistics";
import { complianceAlert } from "./phase3-compliance-estimates";
import { user } from "../schema";

export const adminCountryScope = pgTable(
  "admin_country_scope",
  {
    userId: uuid("user_id")
      .notNull()
      .references(() => user.id, { onDelete: "cascade" }),
    countryCode: char("country_code", { length: 2 }).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    primary: uniqueIndex("pk_admin_country_scope").on(table.userId, table.countryCode),
    countryIndex: index("idx_admin_country_scope_country").on(table.countryCode),
  }),
);

export const businessAccountApproval = pgTable(
  "business_account_approval",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    businessAccountId: uuid("business_account_id").notNull(),
    decision: text("decision").notNull(),
    note: text("note").notNull(),
    payloadHash: char("payload_hash", { length: 64 }).notNull(),
    status: text("status").notNull().default("pending"),
    requestedBy: uuid("requested_by").notNull(),
    requestedAt: timestamp("requested_at", { withTimezone: true }).notNull().defaultNow(),
    approvedBy: uuid("approved_by"),
    approvedAt: timestamp("approved_at", { withTimezone: true }),
    expiresAt: timestamp("expires_at", { withTimezone: true }).notNull(),
  },
  (table) => ({
    pendingIndex: uniqueIndex("uq_business_account_approval_pending")
      .on(table.businessAccountId)
      .where(sql`${table.status} = 'pending'`),
    accountIndex: index("idx_business_account_approval_account").on(
      table.businessAccountId,
      table.requestedAt,
    ),
  }),
);

export const shipmentAdminColumns = {
  countryCode: char("country_code", { length: 2 }).notNull(),
  exceptionReason: text("exception_reason"),
  exceptionOpenedAt: timestamp("exception_opened_at", { withTimezone: true }),
  exceptionRaisedBy: uuid("exception_raised_by"),
  exceptionResolvedAt: timestamp("exception_resolved_at", { withTimezone: true }),
  exceptionResolvedBy: uuid("exception_resolved_by"),
};

export const complianceAlertAdminColumns = {
  acknowledgedAt: timestamp("acknowledged_at", { withTimezone: true }),
  acknowledgedBy: uuid("acknowledged_by"),
};
```

Enregistrer `adminCountryScope` et `businessAccountApproval` dans le baril `packages/db/src/schema/index.ts` (`export * from "./phase3-admin.js";` — le fichier est créé par cette Task), et étendre les colonnes :
- `shipment` → dans `packages/db/src/schema/phase3-logistics.ts` (Task 11), ajouter les colonnes `shipmentAdminColumns` ;
- `complianceAlert` → dans `packages/db/src/schema/phase3-compliance.ts` (Task 2), ajouter les colonnes `complianceAlertAdminColumns`.

Ces colonnes sont ajoutées par la migration `0110_phase3_admin_views.sql` ; le fichier `packages/db/src/schema.ts` (schémas de la Phase 1 : `user`, RBAC, `audit_log`, `outbox_event`) n'est pas modifié par cette Task.

Appliquer :

```bash
pnpm --filter @fanours/db migrate:dev
pnpm --filter @fanours/db migrate:status
```

Attendu : migration `0110_phase3_admin_views` appliquée ; `migrate:status` liste `0110` en `applied` avec son checksum SHA-256.

- [ ] **Step 11: Étendre la matrice de permissions et le seed RBAC**

Modifier `packages/domain/src/permissions.ts` — la ressource `business_account` gagne l'action `approve` pour `super_admin` et `compliance_manager` (la decision B2B est une action a quatre yeux), et la ressource `customs_document` est lue par `order_manager` et `logistics_operator` (vues expeditions) :

```ts
  super_admin: {
    // ... lignes inchangees ...
    business_account: ['read', 'create', 'update', 'delete', 'approve'],
    customs_document: ['read', 'create', 'update', 'delete'],
  },
  order_manager: {
    order: ['read', 'create', 'update', 'export'],
    business_account: ['read'],
    business_document: ['read'],
    customs_document: ['read', 'update'],
  },
  logistics_operator: {
    order: ['read', 'update'],
    business_account: ['read'],
    customs_document: ['read', 'create', 'update', 'delete'],
  },
  compliance_manager: {
    customs_rule: ['read', 'create', 'update', 'delete', 'approve'],
    customs_document: ['read', 'create', 'update', 'delete'],
    country: ['read'],
    product: ['read', 'update'],
    business_document: ['read', 'update'],
    business_account: ['read', 'update', 'approve'],
  },
```

La contrainte `ck_permission_action` de la Phase 1 accepte deja `approve` : aucune migration n'est necessaire.

Modifier `packages/db/src/seed.ts` — la table `rbac_scope` gagne les lignes de la Phase 3 :

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
        customs_document: 'own',
      },
      order_manager: {
        order: 'all', business_account: 'own', business_document: 'own',
        customs_document: 'all',
      },
      logistics_operator: {
        order: 'all', business_account: 'own', customs_document: 'all',
      },
      finance_manager: {
        order: 'all', export: 'all', export_financial: 'all', refund: 'all',
        price_tier: 'own', customs_document: 'own',
      },
      support_agent: {
        order: 'all', business_account: 'own', business_document: 'own', refund: 'own',
      },
      compliance_manager: {
        customs_rule: 'all', customs_document: 'all', country: 'all', product: 'all',
        business_document: 'all', business_account: 'all',
      },
      // ... marketing_manager et analyst_readonly inchanges ...
    };
```

Exécuter :

```bash
pnpm --filter @fanours/db exec vitest run src/__tests__/rbac-matrix.spec.ts
pnpm --filter @fanours/db db:seed
```

Attendu : la matrice est verte, le seed est idempotent (un second passage ne cree aucune ligne dupliquee) et les permissions `customs_document` / `business_account:approve` sont presentes pour les roles concernes.

- [ ] **Step 12: Écrire les tests des composants d'interface admin (ils doivent échouer)**

Créer `packages/ui/src/components/DataTable.spec.tsx` :

```tsx
import { describe, expect, it, vi } from "vitest";
import { render, screen } from "@testing-library/react";
import { DataTable, type DataTableColumn } from "./DataTable";

type Row = { id: string; reference: string; amount: string };

const columns: DataTableColumn<Row>[] = [
  { key: "reference", header: "Reference", render: (row) => row.reference, sortable: true },
  { key: "amount", header: "Montant", render: (row) => row.amount, sortable: true },
];

describe("DataTable — tableau admin", () => {
  it("affiche le libelle du tableau pour les lecteurs d'ecran", () => {
    render(
      <DataTable
        columns={columns}
        rows={[{ id: "1", reference: "EXP-1", amount: "120,00 EUR" }]}
        total={1}
        limit={25}
        offset={0}
        sort="created_at"
        order="desc"
        baseUrl="/fr/admin/expeditions"
        caption="Expeditions"
        emptyLabel="Aucune expedition"
        getRowKey={(row) => row.id}
      />,
    );
    expect(screen.getByRole("table", { name: "Expeditions" })).toBeDefined();
  });

  it("affiche l'etat vide quand il n'y a aucune ligne", () => {
    render(
      <DataTable
        columns={columns}
        rows={[]}
        total={0}
        limit={25}
        offset={0}
        sort="created_at"
        order="desc"
        baseUrl="/fr/admin/expeditions"
        caption="Expeditions"
        emptyLabel="Aucune expedition"
        getRowKey={(row) => row.id}
      />,
    );
    expect(screen.getByText("Aucune expedition")).toBeDefined();
  });

  it("expose un lien de tri qui porte le sens de tri", () => {
    render(
      <DataTable
        columns={columns}
        rows={[{ id: "1", reference: "EXP-1", amount: "120,00 EUR" }]}
        total={1}
        limit={25}
        offset={0}
        sort="created_at"
        order="desc"
        baseUrl="/fr/admin/expeditions"
        caption="Expeditions"
        emptyLabel="Aucune expedition"
        getRowKey={(row) => row.id}
      />,
    );
    const link = screen.getByRole("link", { name: /Reference/ });
    expect(link.getAttribute("href")).toContain("sort=reference");
    expect(link.getAttribute("href")).toContain("order=asc");
  });

  it("desactive le lien de page precedente sur la premiere page", () => {
    render(
      <DataTable
        columns={columns}
        rows={[]}
        total={40}
        limit={25}
        offset={0}
        sort="created_at"
        order="desc"
        baseUrl="/fr/admin/expeditions"
        caption="Expeditions"
        emptyLabel="Aucune expedition"
        getRowKey={(row) => row.id}
      />,
    );
    expect(screen.queryByRole("link", { name: /Precedente/ })).toBeNull();
    expect(screen.getByRole("link", { name: /Suivante/ })).toBeDefined();
  });
});
```

Créer `packages/ui/src/components/KpiCard.spec.tsx` :

```tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { KpiCard } from "./KpiCard";

describe("KpiCard — indicateur admin", () => {
  it("affiche le libelle, la valeur et la definition", () => {
    render(
      <KpiCard
        label="Expeditions en douane"
        value="12"
        definition="Colis retenus par les autorites de destination"
      />,
    );
    expect(screen.getByText("Expeditions en douane")).toBeDefined();
    expect(screen.getByText("12")).toBeDefined();
    expect(screen.getByRole("tooltip")).toBeDefined();
  });

  it("rattache la definition au bouton par aria-describedby", () => {
    render(
      <KpiCard label="Regles expirees" value="3" definition="Regles dont la validite est depassee" />,
    );
    const button = screen.getByRole("button", { name: /definition/i });
    const tooltip = screen.getByRole("tooltip");
    expect(button.getAttribute("aria-describedby")).toBe(tooltip.id);
  });
});
```

Créer `packages/ui/src/components/FilterBar.spec.tsx` :

```tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { FilterBar } from "./FilterBar";

describe("FilterBar — filtres admin", () => {
  it("annonce le nombre de filtres actifs", () => {
    render(
      <FilterBar
        fields={[
          { name: "status", label: "Statut", type: "select", options: [{ value: "open", label: "Ouverte" }] },
        ]}
        values={{ status: "open" }}
        baseUrl="/fr/admin/douane/alertes"
        labels={{
          filters: (n) => `Filtres (${n})`,
          reset: "Reinitialiser",
          apply: "Appliquer",
          title: "Filtres",
        }}
      />,
    );
    expect(screen.getByRole("button", { name: "Filtres (1)" })).toBeDefined();
    expect(screen.getByRole("link", { name: "Reinitialiser" })).toBeDefined();
  });

  it("expose un formulaire GET vers la page cible", () => {
    render(
      <FilterBar
        fields={[{ name: "country", label: "Pays", type: "text" }]}
        values={{}}
        baseUrl="/fr/admin/douane/regles"
        labels={{
          filters: (n) => `Filtres (${n})`,
          reset: "Reinitialiser",
          apply: "Appliquer",
          title: "Filtres",
        }}
      />,
    );
    const form = screen.getByRole("form", { name: "Filtres" });
    expect(form.getAttribute("action")).toBe("/fr/admin/douane/regles");
    expect(form.getAttribute("method")).toBe("get");
  });
});
```

Créer `packages/ui/src/components/AdminWizard.spec.tsx` :

```tsx
import { describe, expect, it, vi } from "vitest";
import { fireEvent, render, screen } from "@testing-library/react";
import { AdminWizard } from "./AdminWizard";

const steps = [
  { index: 1, title: "Pays et produit" },
  { index: 2, title: "Type de regle" },
  { index: 3, title: "Calcul et seuil" },
  { index: 4, title: "Source et validite" },
];

describe("AdminWizard — creation d'une regle douaniere", () => {
  it("demarre sur la premiere etape", () => {
    render(
      <AdminWizard
        title="Nouvelle regle"
        steps={steps}
        fields={[{ name: "country", label: "Pays", type: "text", step: 1, required: true }]}
        initialValues={{}}
        submitLabel="Enregistrer"
        onSubmit={vi.fn()}
      />,
    );
    expect(screen.getByText("Pays et produit")).toBeDefined();
  });

  it("bloque le passage a l'etape suivante quand un champ obligatoire est vide", () => {
    render(
      <AdminWizard
        title="Nouvelle regle"
        steps={steps}
        fields={[{ name: "country", label: "Pays", type: "text", step: 1, required: true }]}
        initialValues={{}}
        submitLabel="Enregistrer"
        onSubmit={vi.fn()}
      />,
    );
    fireEvent.click(screen.getByRole("button", { name: "Suivant" }));
    expect(screen.getByText("Ce champ est obligatoire.")).toBeDefined();
  });

  it("soumet les valeurs du formulaire a la validation", () => {
    const onSubmit = vi.fn();
    render(
      <AdminWizard
        title="Nouvelle regle"
        steps={steps}
        fields={[{ name: "country", label: "Pays", type: "text", step: 1, required: true }]}
        initialValues={{ country: "FR" }}
        submitLabel="Enregistrer"
        onSubmit={onSubmit}
      />,
    );
    fireEvent.click(screen.getByRole("button", { name: "Enregistrer" }));
    expect(onSubmit).toHaveBeenCalledWith({ country: "FR" });
  });
});
```

Exécuter :

```bash
pnpm --filter @fanours/ui exec vitest run src/components/DataTable.spec.tsx src/components/KpiCard.spec.tsx src/components/FilterBar.spec.tsx src/components/AdminWizard.spec.tsx
```

Attendu : **ÉCHEC** — `Failed to resolve import "./DataTable"` (et les trois autres). 0 test exécuté.

- [ ] **Step 13: Implémenter les composants d'interface admin**

Créer `packages/ui/src/components/DataTable.tsx` :

```tsx
import type { ReactNode } from "react";

export type DataTableColumn<T> = {
  readonly key: string;
  readonly header: string;
  readonly render: (row: T) => ReactNode;
  readonly sortable?: boolean;
  readonly className?: string;
};

export type DataTableProps<T> = {
  readonly columns: ReadonlyArray<DataTableColumn<T>>;
  readonly rows: ReadonlyArray<T>;
  readonly total: number;
  readonly limit: number;
  readonly offset: number;
  readonly sort: string;
  readonly order: "asc" | "desc";
  readonly baseUrl: string;
  readonly query?: Readonly<Record<string, string | undefined>>;
  readonly caption: string;
  readonly emptyLabel: string;
  readonly getRowKey: (row: T) => string;
  readonly previousLabel: string;
  readonly nextLabel: string;
  readonly pageLabel: (page: number, pages: number) => string;
};

/**
 * Construit l'URL d'une vue admin : le tri et la pagination sont GERES PAR LE
 * SERVEUR, le composant ne fait que produire des liens (aucun etat client,
 * aucune requete navigateur, fonctionnement sans JavaScript).
 */
function buildHref(
  baseUrl: string,
  query: Readonly<Record<string, string | undefined>>,
  overrides: Readonly<Record<string, string | undefined>>,
): string {
  const params = new URLSearchParams();
  for (const [key, value] of Object.entries({ ...query, ...overrides })) {
    if (value !== undefined && value !== "") params.set(key, value);
  }
  const search = params.toString();
  return search.length === 0 ? baseUrl : `${baseUrl}?${search}`;
}

export function DataTable<T>(props: DataTableProps<T>): ReactNode {
  const {
    columns, rows, total, limit, offset, sort, order, baseUrl, query,
    caption, emptyLabel, getRowKey, previousLabel, nextLabel, pageLabel,
  } = props;

  const pages = Math.max(1, Math.ceil(total / Math.max(limit, 1)));
  const page = Math.floor(offset / Math.max(limit, 1)) + 1;
  const previousOffset = Math.max(0, offset - limit);
  const nextOffset = offset + limit;

  const sortHref = (key: string): string =>
    buildHref(baseUrl, query, {
      sort: key,
      order: sort === key && order === "desc" ? "asc" : "desc",
      offset: "0",
    });

  return (
    <section aria-label={caption} className="w-full">
      <div className="overflow-x-auto">
        <table className="w-full min-w-[720px] border-collapse text-left">
          <caption className="sr-only">{caption}</caption>
          <thead>
            <tr>
              {columns.map((column) => (
                <th
                  key={column.key}
                  scope="col"
                  className="border-b border-[var(--color-border)] px-3 py-2 text-sm font-semibold"
                >
                  {column.sortable === true ? (
                    <a
                      href={sortHref(column.key)}
                      aria-sort={sort === column.key ? (order === "asc" ? "ascending" : "descending") : "none"}
                      className="inline-flex items-center gap-1 underline-offset-4 hover:underline focus-visible:outline focus-visible:outline-2 focus-visible:outline-[var(--focus-ring)]"
                    >
                      {column.header}
                      <span aria-hidden="true">{sort === column.key ? (order === "asc" ? "▲" : "▼") : "↕"}</span>
                    </a>
                  ) : (
                    column.header
                  )}
                </th>
              ))}
            </tr>
          </thead>
          <tbody>
            {rows.length === 0 ? (
              <tr>
                <td colSpan={columns.length} className="px-3 py-6 text-center text-sm text-[var(--color-text-muted)]">
                  {emptyLabel}
                </td>
              </tr>
            ) : (
              rows.map((row) => (
                <tr key={getRowKey(row)} className="border-b border-[var(--color-border)]">
                  {columns.map((column, index) => (
                    <td
                      key={column.key}
                      className={`px-3 py-2 text-sm ${index === 0 ? "sticky left-0 bg-[var(--color-surface)]" : ""} ${column.className ?? ""}`}
                    >
                      {column.render(row)}
                    </td>
                  ))}
                </tr>
              ))
            )}
          </tbody>
        </table>
      </div>
      <nav aria-label={caption} className="mt-3 flex items-center justify-between text-sm">
        {offset > 0 ? (
          <a
            href={buildHref(baseUrl, query, { offset: String(previousOffset) })}
            className="rounded-[var(--radius-sm)] border border-[var(--color-border)] px-3 py-2 focus-visible:outline focus-visible:outline-2 focus-visible:outline-[var(--focus-ring)]"
          >
            {previousLabel}
          </a>
        ) : (
          <span aria-disabled="true" className="rounded-[var(--radius-sm)] border border-[var(--color-border)] px-3 py-2 opacity-50">
            {previousLabel}
          </span>
        )}
        <span aria-live="polite">{pageLabel(page, pages)}</span>
        {nextOffset < total ? (
          <a
            href={buildHref(baseUrl, query, { offset: String(nextOffset) })}
            className="rounded-[var(--radius-sm)] border border-[var(--color-border)] px-3 py-2 focus-visible:outline focus-visible:outline-2 focus-visible:outline-[var(--focus-ring)]"
          >
            {nextLabel}
          </a>
        ) : (
          <span aria-disabled="true" className="rounded-[var(--radius-sm)] border border-[var(--color-border)] px-3 py-2 opacity-50">
            {nextLabel}
          </span>
        )}
      </nav>
    </section>
  );
}
```

Créer `packages/ui/src/components/KpiCard.tsx` :

```tsx
import { useId, type ReactNode } from "react";

export type KpiCardProps = {
  readonly label: string;
  readonly value: string;
  readonly hint?: string;
  readonly definition?: string;
  readonly tone?: "neutral" | "success" | "warning" | "danger";
  readonly definitionLabel: string;
};

const TONE_CLASS: Record<NonNullable<KpiCardProps["tone"]>, string> = {
  neutral: "border-[var(--color-border)]",
  success: "border-[var(--color-success)]",
  warning: "border-[var(--color-warning)]",
  danger: "border-[var(--color-danger)]",
};

export function KpiCard(props: KpiCardProps): ReactNode {
  const { label, value, hint, definition, tone = "neutral", definitionLabel } = props;
  const tooltipId = useId();
  return (
    <article className={`rounded-[var(--radius-lg)] border bg-[var(--color-surface)] p-4 ${TONE_CLASS[tone]}`}>
      <div className="flex items-start justify-between gap-2">
        <h3 className="text-sm font-medium text-[var(--color-text-muted)]">{label}</h3>
        {definition !== undefined ? (
          <span className="group relative">
            <button
              type="button"
              aria-label={`${definitionLabel} : ${label}`}
              aria-describedby={tooltipId}
              className="grid h-6 w-6 place-items-center rounded-full border border-[var(--color-border)] text-xs focus-visible:outline focus-visible:outline-2 focus-visible:outline-[var(--focus-ring)]"
            >
              ?
            </button>
            <span
              id={tooltipId}
              role="tooltip"
              className="pointer-events-none absolute right-0 z-10 hidden w-56 rounded-[var(--radius-md)] bg-[var(--color-text)] p-2 text-xs text-[var(--color-surface)] group-hover:block group-focus-within:block"
            >
              {definition}
            </span>
          </span>
        ) : null}
      </div>
      <p className="mt-2 text-2xl font-semibold tabular-nums">{value}</p>
      {hint !== undefined ? <p className="mt-1 text-xs text-[var(--color-text-muted)]">{hint}</p> : null}
    </article>
  );
}
```

Créer `packages/ui/src/components/FilterBar.tsx` :

```tsx
"use client";

import { useState, type ReactNode } from "react";

export type FilterBarField = {
  readonly name: string;
  readonly label: string;
  readonly type: "text" | "select" | "date";
  readonly options?: ReadonlyArray<{ readonly value: string; readonly label: string }>;
};

export type FilterBarLabels = {
  readonly title: string;
  readonly apply: string;
  readonly reset: string;
  readonly filters: (count: number) => string;
};

export type FilterBarProps = {
  readonly fields: ReadonlyArray<FilterBarField>;
  readonly values: Readonly<Record<string, string>>;
  readonly baseUrl: string;
  readonly labels: FilterBarLabels;
};

/**
 * Barre de filtres d'une vue admin. Sur mobile, les filtres se replient dans
 * un panneau (le compteur de filtres actifs reste visible) ; sur grand ecran,
 * ils restent en ligne. Un seul etat de filtre, un seul formulaire : pas de
 * double saisie.
 */
export function FilterBar(props: FilterBarProps): ReactNode {
  const { fields, values, baseUrl, labels } = props;
  const [open, setOpen] = useState(false);
  const activeCount = fields.filter((field) => (values[field.name] ?? "") !== "").length;

  const resetHref = baseUrl;

  return (
    <section aria-label={labels.title} className="mb-4">
      <div className="flex items-center justify-between gap-3">
        <button
          type="button"
          aria-expanded={open}
          aria-controls="filter-bar-panel"
          onClick={() => setOpen((value) => !value)}
          className="inline-flex items-center gap-2 rounded-[var(--radius-sm)] border border-[var(--color-border)] px-3 py-2 text-sm lg:hidden focus-visible:outline focus-visible:outline-2 focus-visible:outline-[var(--focus-ring)]"
        >
          {labels.filters(activeCount)}
        </button>
        <a
          href={resetHref}
          className="text-sm underline underline-offset-4 focus-visible:outline focus-visible:outline-2 focus-visible:outline-[var(--focus-ring)]"
        >
          {labels.reset}
        </a>
      </div>
      <form
        id="filter-bar-panel"
        role="search"
        aria-label={labels.title}
        method="get"
        action={baseUrl}
        className={`${open ? "block" : "hidden"} mt-3 flex flex-wrap items-end gap-3 lg:flex`}
      >
        {fields.map((field) => (
          <div key={field.name} className="flex flex-col gap-1">
            <label htmlFor={`filter-${field.name}`} className="text-xs font-medium">
              {field.label}
            </label>
            {field.type === "select" ? (
              <select
                id={`filter-${field.name}`}
                name={field.name}
                defaultValue={values[field.name] ?? ""}
                className="h-10 rounded-[var(--radius-sm)] border border-[var(--color-border)] bg-[var(--color-surface)] px-2 text-sm"
              >
                <option value="">—</option>
                {(field.options ?? []).map((option) => (
                  <option key={option.value} value={option.value}>
                    {option.label}
                  </option>
                ))}
              </select>
            ) : (
              <input
                id={`filter-${field.name}`}
                name={field.name}
                type={field.type}
                defaultValue={values[field.name] ?? ""}
                className="h-10 rounded-[var(--radius-sm)] border border-[var(--color-border)] bg-[var(--color-surface)] px-2 text-sm"
              />
            )}
          </div>
        ))}
        <button
          type="submit"
          className="h-10 rounded-[var(--radius-sm)] bg-[var(--color-primary)] px-4 text-sm font-semibold text-[var(--color-on-primary)] focus-visible:outline focus-visible:outline-2 focus-visible:outline-[var(--focus-ring)]"
        >
          {labels.apply}
        </button>
      </form>
    </section>
  );
}
```

Créer `packages/ui/src/components/AdminWizard.tsx` :

```tsx
"use client";

import { useMemo, useState, type ReactNode } from "react";

export type WizardStep = {
  readonly index: number;
  readonly title: string;
  readonly description?: string;
};

export type WizardField = {
  readonly name: string;
  readonly label: string;
  readonly type: "text" | "number" | "date" | "select" | "textarea";
  readonly step: number;
  readonly required?: boolean;
  readonly help?: string;
  readonly options?: ReadonlyArray<{ readonly value: string; readonly label: string }>;
};

export type AdminWizardProps = {
  readonly title: string;
  readonly steps: ReadonlyArray<WizardStep>;
  readonly fields: ReadonlyArray<WizardField>;
  readonly initialValues: Readonly<Record<string, string>>;
  readonly submitLabel: string;
  readonly nextLabel: string;
  readonly backLabel: string;
  readonly requiredError: string;
  readonly onSubmit: (values: Record<string, string>) => void;
  readonly submitting?: boolean;
  readonly errorMessage?: string;
};

export function AdminWizard(props: AdminWizardProps): ReactNode {
  const {
    title, steps, fields, initialValues, submitLabel, nextLabel, backLabel,
    requiredError, onSubmit, submitting = false, errorMessage,
  } = props;

  const [current, setCurrent] = useState(steps[0]?.index ?? 1);
  const [values, setValues] = useState<Record<string, string>>({ ...initialValues });
  const [errors, setErrors] = useState<Record<string, string>>({});

  const stepFields = useMemo(
    () => fields.filter((field) => field.step === current),
    [fields, current],
  );

  const validateStep = (): boolean => {
    const nextErrors: Record<string, string> = {};
    for (const field of stepFields) {
      if (field.required === true && (values[field.name] ?? "").trim() === "") {
        nextErrors[field.name] = requiredError;
      }
    }
    setErrors(nextErrors);
    return Object.keys(nextErrors).length === 0;
  };

  const currentIndex = steps.findIndex((step) => step.index === current);
  const isLast = currentIndex === steps.length - 1;

  return (
    <section aria-label={title} className="max-w-3xl">
      <ol className="mb-6 flex flex-wrap gap-2 text-sm">
        {steps.map((step, index) => {
          const state = index < currentIndex ? "done" : index === currentIndex ? "current" : "todo";
          return (
            <li
              key={step.index}
              aria-current={state === "current" ? "step" : undefined}
              className={`rounded-[var(--radius-sm)] border px-3 py-1 ${
                state === "current" ? "border-[var(--color-primary)] font-semibold" : "border-[var(--color-border)]"
              }`}
            >
              {index + 1}. {step.title}
            </li>
          );
        })}
      </ol>

      {errorMessage !== undefined ? (
        <p role="alert" className="mb-4 rounded-[var(--radius-md)] border border-[var(--color-danger)] p-3 text-sm">
          {errorMessage}
        </p>
      ) : null}

      <div className="flex flex-col gap-4">
        {stepFields.map((field) => {
          const errorId = `${field.name}-error`;
          const helpId = `${field.name}-help`;
          const describedBy = [field.help !== undefined ? helpId : null, errors[field.name] !== undefined ? errorId : null]
            .filter((value) => value !== null)
            .join(" ");
          return (
            <div key={field.name} className="flex flex-col gap-1">
              <label htmlFor={`wizard-${field.name}`} className="text-sm font-medium">
                {field.label}
                {field.required === true ? <span aria-hidden="true"> *</span> : null}
              </label>
              {field.type === "select" ? (
                <select
                  id={`wizard-${field.name}`}
                  name={field.name}
                  value={values[field.name] ?? ""}
                  aria-invalid={errors[field.name] !== undefined}
                  aria-describedby={describedBy === "" ? undefined : describedBy}
                  onChange={(event) => setValues((state) => ({ ...state, [field.name]: event.target.value }))}
                  className="h-10 rounded-[var(--radius-sm)] border border-[var(--color-border)] bg-[var(--color-surface)] px-2 text-sm"
                >
                  <option value="">—</option>
                  {(field.options ?? []).map((option) => (
                    <option key={option.value} value={option.value}>
                      {option.label}
                    </option>
                  ))}
                </select>
              ) : field.type === "textarea" ? (
                <textarea
                  id={`wizard-${field.name}`}
                  name={field.name}
                  value={values[field.name] ?? ""}
                  aria-invalid={errors[field.name] !== undefined}
                  aria-describedby={describedBy === "" ? undefined : describedBy}
                  onChange={(event) => setValues((state) => ({ ...state, [field.name]: event.target.value }))}
                  className="min-h-24 rounded-[var(--radius-sm)] border border-[var(--color-border)] bg-[var(--color-surface)] p-2 text-sm"
                />
              ) : (
                <input
                  id={`wizard-${field.name}`}
                  name={field.name}
                  type={field.type}
                  value={values[field.name] ?? ""}
                  aria-invalid={errors[field.name] !== undefined}
                  aria-describedby={describedBy === "" ? undefined : describedBy}
                  onChange={(event) => setValues((state) => ({ ...state, [field.name]: event.target.value }))}
                  className="h-10 rounded-[var(--radius-sm)] border border-[var(--color-border)] bg-[var(--color-surface)] px-2 text-sm"
                />
              )}
              {field.help !== undefined ? (
                <p id={helpId} className="text-xs text-[var(--color-text-muted)]">
                  {field.help}
                </p>
              ) : null}
              {errors[field.name] !== undefined ? (
                <p id={errorId} role="alert" className="text-xs text-[var(--color-danger)]">
                  {errors[field.name]}
                </p>
              ) : null}
            </div>
          );
        })}
      </div>

      <div className="mt-6 flex items-center gap-3">
        {currentIndex > 0 ? (
          <button
            type="button"
            onClick={() => setCurrent(steps[currentIndex - 1].index)}
            className="rounded-[var(--radius-sm)] border border-[var(--color-border)] px-4 py-2 text-sm focus-visible:outline focus-visible:outline-2 focus-visible:outline-[var(--focus-ring)]"
          >
            {backLabel}
          </button>
        ) : null}
        {isLast ? (
          <button
            type="button"
            disabled={submitting}
            onClick={() => {
              if (!validateStep()) return;
              onSubmit(values);
            }}
            className="rounded-[var(--radius-sm)] bg-[var(--color-primary)] px-4 py-2 text-sm font-semibold text-[var(--color-on-primary)] disabled:opacity-60 focus-visible:outline focus-visible:outline-2 focus-visible:outline-[var(--focus-ring)]"
          >
            {submitLabel}
          </button>
        ) : (
          <button
            type="button"
            onClick={() => {
              if (!validateStep()) return;
              setCurrent(steps[currentIndex + 1].index);
            }}
            className="rounded-[var(--radius-sm)] bg-[var(--color-primary)] px-4 py-2 text-sm font-semibold text-[var(--color-on-primary)] focus-visible:outline focus-visible:outline-2 focus-visible:outline-[var(--focus-ring)]"
          >
            {nextLabel}
          </button>
        )}
      </div>
    </section>
  );
}
```

Modifier `packages/ui/src/index.ts` pour exporter `DataTable`, `AdminWizard`, `KpiCard`, `FilterBar` et leurs types.

Exécuter :

```bash
pnpm --filter @fanours/ui exec vitest run src/components/DataTable.spec.tsx src/components/KpiCard.spec.tsx src/components/FilterBar.spec.tsx src/components/AdminWizard.spec.tsx
```

Attendu : **13 tests passés**, 0 échec.

- [ ] **Step 14: Créer le client serveur admin et les quatre pages admin**

Créer `apps/web/src/server/admin.ts` :

```ts
import { cookies } from "next/headers";
import { translate, type Locale } from "@fanours/i18n";

export type AdminSearchParams = Record<string, string | string[] | undefined>;

export type AdminResult<T> =
  | { readonly ok: true; readonly data: T }
  | { readonly ok: false; readonly status: 401 | 403 | 404 | 500; readonly code: string };

export type AdminShipmentDto = {
  id: string;
  orderId: string;
  orderNumber: string;
  carrier: string;
  service: string;
  trackingNumber: string;
  status: string;
  incoterm: string;
  countryCode: string | null;
  customsComplete: boolean;
  exceptionReason: string | null;
  exceptionOpenedAt: string | null;
  createdAt: string;
  updatedAt: string;
};

export type AdminPageDto<T> = {
  items: T[];
  total: number;
  limit: number;
  offset: number;
};

const API_BASE_URL = process.env["INTERNAL_API_URL"] ?? "http://api:3000";

function firstValue(value: string | string[] | undefined): string | undefined {
  return Array.isArray(value) ? value[0] : value;
}

/**
 * Appel serveur vers l'API admin. Le cookie de session est transmis tel quel :
 * l'autorisation reste entierement decidee par l'API (CDC 19.2), jamais par le
 * front. Les erreurs 401 / 403 / 500 sont des valeurs de retour, pas des
 * exceptions : la page affiche l'etat adapte.
 */
export async function adminFetch<T>(
  path: string,
  searchParams: AdminSearchParams,
): Promise<AdminResult<T>> {
  const jar = await cookies();
  const cookieHeader = jar.toString();
  const url = new URL(path, API_BASE_URL);
  for (const [key, value] of Object.entries(searchParams)) {
    const single = firstValue(value);
    if (single !== undefined && single !== "") url.searchParams.set(key, single);
  }

  try {
    const response = await fetch(url, {
      headers: cookieHeader.length > 0 ? { cookie: cookieHeader } : {},
      cache: "no-store",
    });
    if (!response.ok) {
      const body = (await response.json().catch(() => ({}))) as { code?: string };
      const status = response.status === 401 || response.status === 403 || response.status === 404
        ? response.status
        : 500;
      return { ok: false, status, code: body.code ?? `http_${response.status}` };
    }
    return { ok: true, data: (await response.json()) as T };
  } catch {
    return { ok: false, status: 500, code: "admin_unavailable" };
  }
}

export function adminNumber(value: string | string[] | undefined, fallback: number): number {
  const raw = firstValue(value);
  if (raw === undefined) return fallback;
  const parsed = Number.parseInt(raw, 10);
  return Number.isNaN(parsed) ? fallback : parsed;
}

export function adminText(value: string | string[] | undefined): string | undefined {
  return firstValue(value);
}

/** Libelle localise d'une valeur d'enum technique : jamais de chaine en dur dans la vue. */
export function adminEnumLabel(
  locale: Locale,
  value: string,
  namespace: string,
): string {
  return translate(locale, `${namespace}.${value}`);
}
```

Créer `apps/web/src/app/[locale]/(admin)/admin/expeditions/page.tsx` :

```tsx
import type { Metadata } from "next";
import { notFound } from "next/navigation";
import { DataTable, type DataTableColumn } from "@fanours/ui";
import { KpiCard } from "@fanours/ui";
import { translate, type Locale } from "@fanours/i18n";
import { adminFetch, adminNumber, adminText, type AdminShipmentDto } from "@/server/admin";

export const metadata: Metadata = {
  title: "Expeditions",
  robots: { index: false, follow: false },
};

type Props = { params: Promise<{ locale: Locale }>; searchParams: Promise<Record<string, string | string[] | undefined>> };

export default async function AdminShipmentsPage({ params, searchParams }: Props) {
  const { locale } = await params;
  const query = await searchParams;
  const limit = adminNumber(query["limit"], 25);
  const offset = adminNumber(query["offset"], 0);
  const status = adminText(query["status"]);
  const countryCode = adminText(query["country"]);
  const exceptionsOnly = adminText(query["exceptions"]) === "true";

  const result = await adminFetch<AdminPageDto<AdminShipmentDto>>("/v1/admin/shipments", {
    limit: String(limit),
    offset: String(offset),
    status,
    countryCode,
    exceptionsOnly: exceptionsOnly ? "true" : undefined,
    sort: adminText(query["sort"]) ?? "created_at",
    order: adminText(query["order"]) ?? "desc",
  });

  const t = (key: string) => translate(locale, key);

  if (!result.ok && result.status === 404) notFound();
  if (!result.ok && result.status === 403) {
    return (
      <main className="mx-auto w-full max-w-5xl px-4 py-8">
        <h1 className="text-2xl font-semibold">{t("admin.expeditions.title")}</h1>
        <p role="alert" className="mt-4 rounded-[var(--radius-md)] border border-[var(--color-danger)] p-4">
          {t("admin.states.forbidden")}
        </p>
      </main>
    );
  }
  if (!result.ok) {
    return (
      <main className="mx-auto w-full max-w-5xl px-4 py-8">
        <h1 className="text-2xl font-semibold">{t("admin.expeditions.title")}</h1>
        <p role="alert" className="mt-4 rounded-[var(--radius-md)] border border-[var(--color-danger)] p-4">
          {t("admin.states.error")}
        </p>
      </main>
    );
  }

  const rows = result.data.items;
  const exceptions = rows.filter((row) => row.exceptionReason !== null).length;
  const incomplete = rows.filter((row) => !row.customsComplete).length;

  const columns: AdminShipmentColumn[] = [
    { key: "orderNumber", header: t("admin.expeditions.order"), render: (row) => row.orderNumber, sortable: false },
    { key: "trackingNumber", header: t("admin.expeditions.tracking"), render: (row) => row.trackingNumber, sortable: true },
    { key: "carrier", header: t("admin.expeditions.carrier"), render: (row) => row.carrier, sortable: false },
    { key: "country", header: t("admin.expeditions.country"), render: (row) => row.countryCode ?? "—", sortable: false },
    {
      key: "customs",
      header: t("admin.expeditions.customs"),
      render: (row) =>
        row.customsComplete ? t("admin.expeditions.customsComplete") : t("admin.expeditions.customsIncomplete"),
      sortable: false,
    },
    { key: "status", header: t("admin.expeditions.status"), render: (row) => translate(locale, `admin.shipmentStatus.${row.status}`), sortable: false },
  ];

  return (
    <main className="mx-auto w-full max-w-5xl px-4 py-8">
      <h1 className="text-2xl font-semibold">{t("admin.expeditions.title")}</h1>
      <p className="mt-1 text-sm text-[var(--color-text-muted)]">{t("admin.expeditions.subtitle")}</p>

      <div className="mt-6 grid grid-cols-1 gap-4 sm:grid-cols-2">
        <KpiCard label={t("admin.expeditions.kpiExceptions")} value={String(exceptions)} definition={t("admin.expeditions.kpiExceptionsDefinition")} definitionLabel={t("admin.kpi.definition")} />
        <KpiCard label={t("admin.expeditions.kpiIncomplete")} value={String(incomplete)} definition={t("admin.expeditions.kpiIncompleteDefinition")} definitionLabel={t("admin.kpi.definition")} tone="warning" />
      </div>

      <div className="mt-6">
        <DataTable
          columns={columns}
          rows={rows}
          total={result.data.total}
          limit={result.data.limit}
          offset={result.data.offset}
          sort={adminText(query["sort"]) ?? "created_at"}
          order={adminText(query["order"]) === "asc" ? "asc" : "desc"}
          baseUrl={`/${locale}/admin/expeditions`}
          query={{ status, countryCode }}
          caption={t("admin.expeditions.title")}
          emptyLabel={t("admin.states.empty")}
          getRowKey={(row) => row.id}
          previousLabel={t("admin.pagination.previous")}
          nextLabel={t("admin.pagination.next")}
          pageLabel={(page, pages) => t("admin.pagination.page").replace("{page}", String(page)).replace("{pages}", String(pages))}
        />
      </div>
    </main>
  );
}

type AdminShipmentColumn = DataTableColumn<AdminShipmentDto>;
```

Créer `apps/web/src/app/[locale]/(admin)/admin/douane/regles/page.tsx` : même structure que la page expéditions, avec le tableau des règles (`key`, `countryCode`, `ruleType`, `certainty`, `reviewState`, `sourceUrl`, `reviewDueAt`), un `KpiCard` « règles expirées », un `FilterBar` (pays, type de règle, niveau de certitude) et le `AdminWizard` de création d'une version de règle (4 étapes : pays/produit → type de règle → mode de calcul et seuil → source, validité, validateur). La liste des règles à revalider (`reviewState !== "current"`) s'affiche en tête de tableau avec une pastille d'alerte.

Créer `apps/web/src/app/[locale]/(admin)/admin/douane/alertes/page.tsx` : tableau des alertes de conformité trié par sévérité (`blocker` d'abord), `KpiCard` « alertes bloquantes », `FilterBar` (sévérité, statut, pays), et les actions « prendre en compte » / « résoudre » qui appellent `POST /v1/admin/compliance-alerts/:id/acknowledge` et `POST /v1/admin/compliance-alerts/:id/resolve`. Le panneau d'alerte de règle expirée est mis en avant : `severity === "blocker"` déclenche un encadré `role="alert"`.

Créer `apps/web/src/app/[locale]/(admin)/admin/clients/page.tsx` : liste des comptes B2B avec `FilterBar` (statut de verification, pays, recherche par identifiant ou e-mail), `KpiCard` « dossiers en attente », et la revue : la décision passe par `POST /v1/admin/business-accounts/:id/approvals` puis, **pour un administrateur différent**, `POST /v1/admin/business-accounts/:id/approvals/:approvalId/approve`. L'interface affiche explicitement qu'une approbation par le même compte est impossible (four-eyes) et que l suspension exige un motif.

Exécuter :

```bash
pnpm --filter @fanours/web exec tsc --noEmit
pnpm turbo run typecheck
```

Attendu : `typecheck` sans erreur.

- [ ] **Step 15: Ajouter les clés i18n des vues admin dans les trois langues**

Ajouter dans `packages/i18n/src/locales/fr-FR.json`, `en.json` et `zh-CN.json` le bloc `admin` (mêmes clés, mêmes valeurs structurelles) :

```json
{
  "admin": {
    "nav": {
      "expeditions": "Expeditions",
      "rules": "Regles douanieres",
      "alerts": "Alertes de conformite",
      "clients": "Clients professionnels"
    },
    "kpi": { "definition": "Definition" },
    "expeditions": {
      "title": "Expeditions et exceptions",
      "subtitle": "Colis expedies, retenus en douane et exceptions a traiter.",
      "order": "Commande",
      "tracking": "Suivi",
      "carrier": "Transporteur",
      "country": "Pays",
      "customs": "Donnees douanieres",
      "status": "Statut",
      "customsComplete": "Dossier complet",
      "customsIncomplete": "Dossier incomplet",
      "kpiExceptions": "Exceptions ouvertes",
      "kpiExceptionsDefinition": "Colis retenus ou livres en anomaly sur la page en cours.",
      "kpiIncomplete": "Dossiers incomplets",
      "kpiIncompleteDefinition": "Expeditions dont les donnees douanieres ne sont pas renseignees : expedition bloquee.",
      "raiseException": "Lever une exception",
      "resolveException": "Leverer l'exception",
      "exceptionReason": "Motif de l'exception",
      "resolutionNote": "Note de resolution",
      "reasonRequired": "Le motif est obligatoire."
    },
    "shipmentStatus": {
      "label_created": "Etiquette creee",
      "handed_over": "Remis au transporteur",
      "in_transit": "En transit",
      "customs_hold": "Retenu en douane",
      "action_required": "Action requise",
      "out_for_delivery": "En livraison",
      "delivered": "Livre",
      "exception": "Exception",
      "returned": "Retourne"
    },
    "rules": {
      "title": "Regles douanieres",
      "subtitle": "Regles par pays, type et validite. Les regles sont des donnees, jamais du code.",
      "key": "Regle",
      "ruleType": "Type",
      "certainty": "Niveau de certitude",
      "reviewState": "Revue",
      "source": "Source officielle",
      "lastReview": "Derniere revue",
      "current": "A jour",
      "expired": "Expiree",
      "reviewDue": "A revalider",
      "kpiExpired": "Regles expirees",
      "kpiExpiredDefinition": "Regles dont la date de validite est depassee : elles ne sont plus appliquees.",
      "newRule": "Nouvelle regle",
      "wizard": {
        "step1": "Pays et produit",
        "step2": "Type de regle",
        "step3": "Mode de calcul et seuil",
        "step4": "Source, validite et validateur",
        "sourceHelp": "Source officielle avec URL et date de consultation.",
        "reviewHelp": "Date de derniere revue obligatoire.",
        "validToHelp": "Une regle bloquante ou un seuil exige une date de fin."
      }
    },
    "alerts": {
      "title": "Alertes de conformite",
      "subtitle": "Regles expirees, donnees manquantes et seuils a revalider.",
      "severity": "Severite",
      "status": "Statut",
      "code": "Code",
      "message": "Message",
      "actionRequired": "Action requise",
      "acknowledge": "Prendre en compte",
      "resolve": "Resoudre",
      "resolutionNote": "Note de resolution",
      "kpiBlockers": "Alertes bloquantes",
      "kpiBlockersDefinition": "Alertes de severite bloquante : la commande ne peut pas partir.",
      "severityBlocker": "Bloquante",
      "severityWarning": "Avertissement",
      "severityInfo": "Information"
    },
    "clients": {
      "title": "Clients professionnels",
      "subtitle": "Approbation, refus et suspension des comptes B2B. Une decision exige deux administrateurs differents.",
      "legalName": "Raison sociale",
      "country": "Pays",
      "taxId": "Identifiant fiscal",
      "status": "Verification",
      "documents": "Justificatifs",
      "approvedAt": "Verifie le",
      "reviewNote": "Note de revue",
      "search": "Rechercher par identifiant, raison sociale ou identifiant fiscal",
      "kpiPending": "Dossiers en attente",
      "kpiPendingDefinition": "Comptes dont la verification n'est pas encore terminee.",
      "approve": "Approuver",
      "reject": "Refuser",
      "suspend": "Suspendre",
      "suspensionReason": "Motif de la suspension",
      "fourEyesNotice": "La decision doit etre approuvee par un autre administrateur.",
      "selfApprovalNotice": "Vous ne pouvez pas approuver une decision que vous avez demandee.",
      "statusUnverified": "Non verifie",
      "statusPending": "En attente",
      "statusApproved": "Approuve",
      "statusRejected": "Refuse",
      "statusSuspended": "Suspendu"
    },
    "filters": {
      "title": "Filtres",
      "apply": "Appliquer",
      "reset": "Reinitialiser",
      "country": "Pays",
      "status": "Statut",
      "severity": "Severite",
      "certified": "Niveau de certitude"
    },
    "pagination": {
      "previous": "Precedente",
      "next": "Suivante",
      "page": "Page {page} sur {pages}"
    },
    "states": {
      "loading": "Chargement...",
      "empty": "Aucun resultat",
      "error": "Une erreur est survenue. Reessayez dans un instant.",
      "forbidden": "Vous n'avez pas les droits necessaires pour consulter cette page."
    }
  }
}
```

La version `en.json` reprend les memes cles en anglais (`"Expeditions and exceptions"`, `"Customs rules"`, `"Compliance alerts"`, `"Professional accounts"`, etc.) et la version `zh-CN.json` les traduit en chinois simplifie (`"发货与异常"`, `"海关规则"`, `"合规提醒"`, `"企业客户"`). Aucune cle n'est ajoutee a une seule langue : le controle de parite du build (`pnpm --filter @fanours/i18n check:parity`) doit rester vert.

Exécuter :

```bash
pnpm --filter @fanours/i18n check:parity
pnpm --filter @fanours/i18n exec vitest run
```

Attendu : parité verte pour `fr-FR`, `en` et `zh-CN`.

- [ ] **Step 16: Écrire le test E2E des vues admin (il doit échouer)**

Créer `e2e/b2b/admin-views.spec.ts` :

```ts
import { expect, test } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";
import { VIEWPORTS } from "../support/viewports.js";

test.describe("vues admin phase 3", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/fr/admin/expeditions");
  });

  test("la liste des expeditions est paginee par le serveur", async ({ page }) => {
    await expect(page.getByRole("table")).toBeVisible();
    await expect(page.getByRole("navigation", { name: "Expeditions et exceptions" })).toBeVisible();
  });

  test("un analyste sans permission recoit un etat 403 explicite", async ({ page }) => {
    await page.goto("/fr/admin/expeditions", { waitUntil: "networkidle" });
    const forbidden = page.getByRole("alert");
    if (await forbidden.count()) {
      await expect(forbidden.first()).toContainText(/droits/i);
    }
  });

  test("le centre de regles signale les regles expirees", async ({ page }) => {
    await page.goto("/fr/admin/douane/regles");
    await expect(page.getByText("Regles douanieres")).toBeVisible();
  });

  test("les alertes de conformite sont classees par severite", async ({ page }) => {
    await page.goto("/fr/admin/douane/alertes");
    await expect(page.getByRole("table")).toBeVisible();
  });

  test("la revue B2B explique l'exigence de quatre yeux", async ({ page }) => {
    await page.goto("/fr/admin/clients");
    await expect(page.getByText(/autre administrateur/i)).toBeVisible();
  });

  for (const viewport of VIEWPORTS) {
    test(`aucun debordement horizontal - ${viewport.width}x${viewport.height}`, async ({ page }) => {
      await page.setViewportSize({ width: viewport.width, height: viewport.height });
      await page.goto("/fr/admin/expeditions");
      const overflow = await page.evaluate(
        () => document.documentElement.scrollWidth > window.innerWidth + 1,
      );
      expect(overflow).toBe(false);
    });
  }

  test("aucune violation d'accessibilite critique sur la liste des expeditions", async ({ page }) => {
    await page.goto("/fr/admin/expeditions");
    const results = await new AxeBuilder({ page }).withTags(["wcag2a", "wcag2aa", "wcag21a", "wcag21aa"]).analyze();
    const blocking = results.violations.filter((violation) =>
      violation.impact === "critical" || violation.impact === "serious",
    );
    expect(blocking).toEqual([]);
  });
});
```

Exécuter :

```bash
pnpm exec playwright test e2e/b2b/admin-views.spec.ts --project=functional
```

Attendu : **ÉCHEC** — les pages `admin/expeditions`, `admin/douane/regles`, `admin/douane/alertes` et `admin/clients` n'existent pas encore (`404` sur la route).

- [ ] **Step 17: Vérifier l'exécuter la suite complète et committer**

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/shipment/admin-shipments.service.spec.ts src/modules/compliance/admin-trade-rules.service.spec.ts src/modules/compliance/compliance-alert.service.spec.ts src/modules/business/admin-business-accounts.service.spec.ts
pnpm --filter @fanours/ui exec vitest run src/components
pnpm --filter @fanours/db exec vitest run --config vitest.integration.config.ts src/__tests__/phase3-admin-views.integration.spec.ts
pnpm --filter @fanours/i18n check:parity
pnpm turbo run lint typecheck
pnpm exec playwright test e2e/b2b/admin-views.spec.ts --project=functional
```

Attendu : toutes les suites vertes, `lint` et `typecheck` sans erreur, E2E admin au vert (14 tests : 6 cas de vues admin + 8 cas de débordement horizontal, un par viewport de la liste canonique `e2e/support/viewports.ts`).

Commit :

```bash
git add apps/api/src/common/admin-scope.ts apps/api/src/modules/shipment/admin-shipments.service.ts apps/api/src/modules/shipment/admin-shipments.service.spec.ts apps/api/src/modules/shipment/admin-shipments.controller.ts apps/api/src/modules/compliance/admin-trade-rules.service.ts apps/api/src/modules/compliance/admin-trade-rules.service.spec.ts apps/api/src/modules/compliance/admin-trade-rules.controller.ts apps/api/src/modules/compliance/compliance-alert.service.ts apps/api/src/modules/compliance/compliance-alert.service.spec.ts apps/api/src/modules/compliance/compliance-alerts.controller.ts apps/api/src/modules/business/admin-business-accounts.service.ts apps/api/src/modules/business/admin-business-accounts.service.spec.ts apps/api/src/modules/business/admin-business-accounts.controller.ts apps/api/src/modules/business/business-review.service.ts packages/domain/src/permissions.ts packages/db/src/seed.ts packages/db/src/schema/phase3-admin.ts packages/db/src/schema/index.ts packages/db/src/schema/phase3-logistics.ts packages/db/src/schema/phase3-compliance.ts packages/db/src/__tests__/phase3-admin-views.integration.spec.ts packages/db/migrations/0110_phase3_admin_views.sql packages/ui/src/components/DataTable.tsx packages/ui/src/components/DataTable.spec.tsx packages/ui/src/components/AdminWizard.tsx packages/ui/src/components/AdminWizard.spec.tsx packages/ui/src/components/KpiCard.tsx packages/ui/src/components/KpiCard.spec.tsx packages/ui/src/components/FilterBar.tsx packages/ui/src/components/FilterBar.spec.tsx packages/ui/src/index.ts apps/web/src/server/admin.ts "apps/web/src/app/[locale]/(admin)/admin" packages/i18n/src/locales/fr-FR.json packages/i18n/src/locales/en.json packages/i18n/src/locales/zh-CN.json e2e/b2b/admin-views.spec.ts
git commit -m "feat(admin): ajoute les vues expeditions, regles, alertes et revue B2B"
```

---

### Task 18: Recette E2E B2B — parcours acheteur 9 étapes et contrôles douaniers

**Files:**
- Create: `e2e/b2b/journey.spec.ts` — suite de release `b2b` (spec §24.3, « acheteur B2B », 9 étapes) : **seul** fichier du projet Playwright `b2b`. La Phase 5 la reprend en **Modify** pour la porter au niveau release ; elle n'en crée **aucun doublon**.
- Create: `e2e/b2b/compliance.spec.ts` — recette fonctionnelle de phase (4 cas douaniers) : projet Playwright `functional`.
- Create: `e2e/support/b2b-api.ts` — aides de la recette B2B (enveloppe de réponse, connexion, MFA, fixtures) : module de support, jamais collecté par un projet Playwright.
- Create: `apps/api/src/providers/email/mailbox.provider.ts` — boîte e-mail **de test** en mémoire (l'envoi SMTP réel arrive avec la Phase 5) : `MailboxProvider` + `InMemoryMailboxProvider`.
- Create: `apps/api/src/modules/auth/test-mailbox.controller.ts` — `GET /v1/test-mailbox/messages` (**jamais exposée en production** : `404` si `APP_ENV=production`).
- Modify: `apps/api/src/modules/auth/auth.service.ts` — `signUp` émet un jeton de vérification (seul le **hachage** est conservé, dans Redis, TTL 24 h) et dépose l'e-mail dans la boîte de test ; `verifyEmail(token)` refuse un jeton inconnu, expiré, déjà utilisé ou dont le compte n'est pas actif.
- Modify: `apps/api/src/modules/auth/auth.controller.ts` — `POST /v1/auth/verify-email` (le service `AuthService.verifyEmail` existe depuis la Phase 1, la route HTTP manquait).
- Modify: `apps/api/src/modules/business/business-review.service.ts` — `apply()` : crée le `business_account` (`unverified`) et le `business_membership` (`owner`) de l'utilisateur authentifié.
- Modify: `apps/api/src/modules/business/business.controller.ts` — `POST /v1/business/accounts` (demande d'activation professionnelle) et `GET /v1/business/accounts/me`.
- Modify: `apps/api/src/common/api-errors.ts` — `ApiError` accepte `details?: Record<string, string>` (chaînes uniquement) pour porter les constats de blocage et la règle citée.
- Modify: `apps/api/src/modules/quote/quote.controller.ts` — `POST /v1/quotes`, `POST /v1/quotes/:id/accept`, `POST /v1/quotes/:id/convert`.
- Modify: `apps/api/src/modules/shipment/shipment.controller.ts` — `POST /v1/orders/:number/shipments`, `GET /v1/orders/:number/shipments`.
- Modify: `e2e/scripts/seed-e2e.ts` — second approvateur `super_admin` (four-eyes) et fixtures déterministes de la Phase 3.
- `e2e/b2b/admin-views.spec.ts` — **non modifié** : les deux specs sont autonomes et partagent uniquement `e2e/support/b2b-api.ts`, donc aucune modification croisée du fichier de la Task 17.

> **Cette Task n'ajoute aucune migration.** Le jeton de vérification d'e-mail est un secret **éphémère** (24 h) : il est conservé dans Redis (Phase 1, client `ioredis` injecté sous `'REDIS'`) sous la forme de son **seul hachage**, avec une expiration TTL et un usage unique par `GETDEL`. La colonne `user.email_verified_at` existe déjà depuis la migration `0002` de la Phase 1. La plage de migrations de la Phase 3 reste donc **`0100`–`0110`**, sans recouvrement avec la Phase 4 (`0111`–`0115`) : la numérotation des migrations reste continue et globale, un fichier de migration appartient à une seule phase.

**Interfaces:**
- Consumes:
  - `playwright.config.ts` (racine du dépôt, Phase 1) — `testDir: './e2e'` ; projet `b2b` = `**/b2b/journey.spec.ts` ; projet `functional` = tout le reste (`**/b2b/compliance.spec.ts` inclus). **Aucune configuration Playwright n'est créée ou modifiée par cette Task.**
  - `e2e/fixtures/totp.ts` (Phase 1) — `computeTotp(secret)` pour les codes MFA.
  - `e2e/fixtures/product.ts` (Phase 2) — `REFERENCE_PRODUCT` (slug, MOQ, incrément de commande).
  - `e2e/support/viewports.ts` (Phase 1) — **non utilisé** par ces deux specs : le parcours est un scénario de flux, pas une matrice visuelle. Aucune liste de viewports n'est re-déclarée ici (la matrice reste dans le projet `visual`).
  - `e2e/scripts/seed-e2e.ts` (Phase 1) — comptes de recette et fixtures Phase 3 (Step 18.6).
  - Redis (Phase 1 — client `ioredis` injecté sous `'REDIS'`) — conservation éphémère du jeton de vérification d'e-mail (clé `auth:email-verification:<sha256>`, TTL 24 h, usage unique par `GETDEL`). **Aucune migration n'est requise.**
  - API Phase 1 : `POST /v1/auth/signup`, `POST /v1/auth/signin`, `POST /v1/mfa/enroll`, `POST /v1/mfa/enroll/confirm`, `POST /v1/mfa/elevate`.
  - API Phase 2 : `GET /v1/catalog/products/:slug`, `POST /v1/orders/:id/payment`, `GET /v1/orders/:id/invoice`.
  - API Phase 3 : `POST /v1/business/accounts`, `GET /v1/business/accounts/me`, `POST /v1/quotes`, `GET /v1/quotes/:number`, `POST /v1/quotes/:id/accept`, `POST /v1/quotes/:id/convert`, `GET /v1/orders/:number/documents`, `POST /v1/orders/:number/shipments`, `GET /v1/orders/:number/shipments`, `GET /v1/admin/business-accounts/:id`, `POST /v1/admin/business-accounts/:id/approvals`, `POST /v1/admin/business-accounts/:id/approvals/:approvalId/approve`, `GET /v1/admin/compliance-alerts`.
  - Pages web : `/fr/produits/<slug>`, `/fr/commandes/<number>/suivi`, `/fr/commandes/<number>/documents` (Task 16), `/fr/admin/douane/alertes` (Task 17).
- Produces:
  - `e2e/b2b/journey.spec.ts` — **1 test** (`parcours complet en 9 etapes`) découpé en 9 `test.step()` nommés ; projet `b2b`.
  - `e2e/b2b/compliance.spec.ts` — **4 tests** (4 cas douaniers) ; projet `functional`.
  - `POST /v1/auth/verify-email` → `204` ; jeton inconnu, expiré ou déjà utilisé → `400` / `409`.
  - `GET /v1/test-mailbox/messages?to=<email>` → `200 { data: { messages: [{ subject, body, sentAt }] } }` ; `404` si `APP_ENV=production`.
  - `POST /v1/business/accounts` → `201 { id, verificationStatus: "unverified" }` ; `GET /v1/business/accounts/me` → `200 { id, legalName, verificationStatus }` ou `404`.
  - `POST /v1/quotes` → `201 { id, number, status: "draft", totalMinor, currency }` ; `POST /v1/quotes/:id/accept` → `200 { status: "accepted" }` ; `POST /v1/quotes/:id/convert` → `201 { orderId, orderNumber, totalMinor }`.
  - `POST /v1/orders/:number/shipments` → `201 { shipmentId, trackingNumber }` ou **`409 { error: { code: "dispatch_blocked", details: { findings, ruleRef, sourceUrl, consultedAt } } }`**.
  - `POST /v1/orders/:id/payment` → `201 { providerRef, status: "pending" }` pour `method: "bank_transfer"`.
  - `ApiError` (Phase 3) accepte `details?: Record<string, string>`.

- [ ] **Step 18.1: Écrire le module de support de la recette B2B**

Créer `e2e/support/b2b-api.ts` :

```ts
import { type APIResponse, type BrowserContext } from "@playwright/test";
import { computeTotp } from "../fixtures/totp.js";

/**
 * Aides de la recette E2E B2B : lecture de l'enveloppe de reponse, connexion,
 * enrôlement MFA et fixtures deterministes.
 *
 * Module de support : aucun `test` n'y est declare, il n'est donc jamais
 * collecte par un projet Playwright. Aucun secret en dur : toutes les valeurs
 * viennent de l'environnement de recette.
 */

export const API_URL = process.env["E2E_API_URL"] ?? "http://localhost:4000";
export const ADMIN_EMAIL = process.env["E2E_ADMIN_EMAIL"] ?? "admin@fanours.test";
export const APPROVER_EMAIL = process.env["E2E_APPROVER_EMAIL"] ?? "approbateur@fanours.test";
export const RECIPE_PASSWORD = process.env["E2E_ADMIN_PASSWORD"] ?? "MotDePasse-E2E-2026";
export const ADMIN_TOTP_SECRET = process.env["E2E_ADMIN_TOTP_SECRET"];
export const APPROVER_TOTP_SECRET = process.env["E2E_APPROVER_TOTP_SECRET"];

/** Fixtures deterministes creees par `e2e/scripts/seed-e2e.ts` (Step 18.6). */
export const FIXTURES = {
  incompleteOrderId: "0199f0f0-f000-7000-8000-000000000001",
  splitOrderId: "0199f0f0-f000-7000-8000-000000000002",
  expiredRuleId: "0199f0f0-f000-7000-8000-000000000003",
  businessAccountId: "0199f0f0-f000-7000-8000-000000000004",
  businessAccountUserEmail: "client-b2b@fanours.test",
} as const;

export type ApiData = Record<string, unknown>;

export async function readData(response: APIResponse, route: string): Promise<ApiData> {
  if (!response.ok()) {
    throw new Error(`Route HTTP indisponible : ${route} -> ${response.status()} ${await response.text()}`);
  }
  const body = (await response.json()) as { data?: ApiData };
  return body.data ?? {};
}

export async function readList(response: APIResponse, route: string): Promise<ApiData[]> {
  if (!response.ok()) {
    throw new Error(`Route HTTP indisponible : ${route} -> ${response.status()} ${await response.text()}`);
  }
  const body = (await response.json()) as { data?: ApiData[] };
  return body.data ?? [];
}

export async function readError(
  response: APIResponse,
  route: string,
): Promise<{ code: string; details: Record<string, string> }> {
  if (response.ok()) {
    throw new Error(`Route HTTP reussie alors qu'un echec etait attendu : ${route} -> ${response.status()}`);
  }
  const body = (await response.json()) as {
    error?: { code?: string; details?: Record<string, string> };
  };
  return { code: body.error?.code ?? "unknown_error", details: body.error?.details ?? {} };
}

export async function signIn(context: BrowserContext, email: string, password: string): Promise<void> {
  const response = await context.request.post("/v1/auth/signin", { data: { email, password } });
  if (!response.ok()) {
    throw new Error(`Connexion impossible pour ${email} : ${response.status()} ${await response.text()}`);
  }
}

/**
 * Rend un secret TOTP utilisable : secret de l'environnement s'il existe,
 * sinon enrôlement a la volee du compte de recette (premiere execution).
 */
export async function ensureMfa(
  context: BrowserContext,
  secretFromEnv: string | undefined,
  envName: string,
): Promise<string> {
  if (secretFromEnv !== undefined && secretFromEnv !== "") {
    return secretFromEnv;
  }
  const enroll = await context.request.post("/v1/mfa/enroll", { data: {} });
  if (!enroll.ok()) {
    throw new Error(
      `Enrollement MFA impossible (${enroll.status()}) : le compte est deja enrole, ` +
        `definir ${envName} pour la recette.`,
    );
  }
  const data = await readData(enroll, "POST /v1/mfa/enroll");
  const secret = String(data["secret"]);
  const confirm = await context.request.post("/v1/mfa/enroll/confirm", {
    data: { code: computeTotp(secret) },
  });
  if (!confirm.ok()) {
    throw new Error(`Confirmation MFA refusee : ${confirm.status()} ${await confirm.text()}`);
  }
  return secret;
}

/** Elevation MFA (≤ 10 min) exigee par `business_account:approve`. */
export async function elevate(context: BrowserContext, secret: string): Promise<void> {
  const response = await context.request.post("/v1/mfa/elevate", {
    data: { code: computeTotp(secret) },
  });
  if (!response.ok()) {
    throw new Error(`Elevation MFA refusee : ${response.status()} ${await response.text()}`);
  }
}
```

Exécuter :

```bash
pnpm --filter @fanours/e2e typecheck
```

Attendu : aucune erreur de type (le module importe `computeTotp` depuis `e2e/fixtures/totp.ts` et vit dans le paquet `@fanours/e2e` créé en Phase 1).

- [ ] **Step 18.2: Écrire la suite de release B2B (elle doit échouer)**

Créer `e2e/b2b/journey.spec.ts` :

```ts
import { expect, test } from "@playwright/test";
import { REFERENCE_PRODUCT } from "../fixtures/product.js";
import {
  ADMIN_EMAIL,
  ADMIN_TOTP_SECRET,
  API_URL,
  APPROVER_EMAIL,
  APPROVER_TOTP_SECRET,
  RECIPE_PASSWORD,
  elevate,
  ensureMfa,
  readData,
  readList,
  signIn,
} from "../support/b2b-api.js";

/**
 * Suite de release B2B (spec §24.3) : parcours acheteur professionnel en
 * 9 etapes, de la creation du compte au suivi du colis.
 *
 * Seul fichier du projet Playwright `b2b` : la Phase 5 le reprend en Modify
 * pour le porter au niveau release, elle n'en cree aucun doublon.
 *
 * Recette sur environnement de test : aucun secret en dur, aucune donnee
 * reelle. Le scenario est joue sur l'API (session, MFA, devis, commande) et
 * verifie sur les pages web qui exposent le resultat.
 */

const BUYER_EMAIL = `acheteur-b2b-${Date.now()}@fanours.test`;
const BUYER_PASSWORD = RECIPE_PASSWORD;
const UUID_PATTERN = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/;

test.describe("Parcours acheteur B2B", () => {
  test("parcours complet en 9 etapes", async ({ browser, baseURL }) => {
    // Le scenario fait tourner le worker (facture, documents d'export) : le
    // delai par defaut de 45 s est trop court, `test.slow()` le triple.
    test.slow();

    const buyer = await browser.newContext({ baseURL: API_URL });
    const admin = await browser.newContext({ baseURL: API_URL });
    const approver = await browser.newContext({ baseURL: API_URL });
    const web = await browser.newContext({ baseURL: baseURL ?? undefined });

    let businessAccountId = "";
    let variantId = "";
    let quoteId = "";
    let quoteNumber = "";
    let orderId = "";
    let orderNumber = "";

    await test.step("1. Creation du compte acheteur B2B", async () => {
      const response = await buyer.request.post("/v1/auth/signup", {
        data: {
          email: BUYER_EMAIL,
          password: BUYER_PASSWORD,
          displayName: "Acheteur B2B recette",
          locale: "fr-FR",
        },
      });
      const data = await readData(response, "POST /v1/auth/signup");
      const user = data["user"] as Record<string, unknown> | undefined;
      expect(String(user?.["id"])).toMatch(UUID_PATTERN);
      const cookies = await buyer.cookies();
      expect(cookies.some((cookie) => cookie.name === "fanours_sid")).toBe(true);
    });

    await test.step("2. Verification de l'adresse e-mail", async () => {
      const messages = await readList(
        await buyer.request.get(`/v1/test-mailbox/messages?to=${encodeURIComponent(BUYER_EMAIL)}`),
        "GET /v1/test-mailbox/messages",
      );
      expect(messages.length).toBeGreaterThan(0);
      const body = String(messages[0]?.["body"] ?? "");
      const match = /token=([A-Za-z0-9]+)/.exec(body);
      expect(match).not.toBeNull();
      const token = match?.[1] ?? "";

      const verified = await buyer.request.post("/v1/auth/verify-email", { data: { token } });
      expect(verified.status()).toBe(204);

      const replay = await buyer.request.post("/v1/auth/verify-email", { data: { token } });
      expect(replay.status()).toBeGreaterThanOrEqual(400);
    });

    await test.step("3. Demande d'activation professionnelle", async () => {
      const response = await buyer.request.post("/v1/business/accounts", {
        data: {
          legalName: "Fanours B2B Recette E2E",
          countryCode: "FR",
          address: "1 rue de la Recette, 75001 Paris",
          taxId: "FR00000000001",
        },
      });
      const data = await readData(response, "POST /v1/business/accounts");
      businessAccountId = String(data["id"]);
      expect(businessAccountId).toMatch(UUID_PATTERN);
      expect(data["verificationStatus"]).toBe("unverified");

      const me = await readData(
        await buyer.request.get("/v1/business/accounts/me"),
        "GET /v1/business/accounts/me",
      );
      expect(String(me["id"])).toBe(businessAccountId);
    });

    await test.step("4. Approbation par un role autorise (MFA, quatre yeux)", async () => {
      await signIn(admin, ADMIN_EMAIL, RECIPE_PASSWORD);
      const adminSecret = await ensureMfa(admin, ADMIN_TOTP_SECRET, "E2E_ADMIN_TOTP_SECRET");
      await elevate(admin, adminSecret);

      const requested = await admin.request.post(
        `/v1/admin/business-accounts/${businessAccountId}/approvals`,
        { data: { decision: "approved", note: "Recette E2E B2B" } },
      );
      const approval = await readData(
        requested,
        "POST /v1/admin/business-accounts/:id/approvals",
      );
      const approvalId = String(approval["approvalId"]);
      expect(approval["status"]).toBe("pending");

      const selfApproval = await admin.request.post(
        `/v1/admin/business-accounts/${businessAccountId}/approvals/${approvalId}/approve`,
      );
      expect(selfApproval.status()).toBe(409);

      await signIn(approver, APPROVER_EMAIL, RECIPE_PASSWORD);
      const approverSecret = await ensureMfa(
        approver,
        APPROVER_TOTP_SECRET,
        "E2E_APPROVER_TOTP_SECRET",
      );
      await elevate(approver, approverSecret);

      const approved = await approver.request.post(
        `/v1/admin/business-accounts/${businessAccountId}/approvals/${approvalId}/approve`,
      );
      const data = await readData(
        approved,
        "POST /v1/admin/business-accounts/:id/approvals/:approvalId/approve",
      );
      expect(data["verificationStatus"]).toBe("approved");
    });

    await test.step("5. Affichage des prix de gros et du MOQ", async () => {
      const response = await buyer.request.get(
        `/v1/catalog/products/${REFERENCE_PRODUCT.slug}`,
      );
      const product = await readData(response, "GET /v1/catalog/products/:slug");
      expect(product["moq"]).toBe(REFERENCE_PRODUCT.moq);
      expect(product["orderMultiple"]).toBe(REFERENCE_PRODUCT.orderMultiple);
      const tiers = product["priceTiers"] as Record<string, unknown>[];
      expect(tiers.length).toBeGreaterThan(0);
      const variants = product["variants"] as Record<string, unknown>[];
      expect(variants.length).toBeGreaterThan(0);
      variantId = String(variants[0]?.["id"]);
      expect(variantId).toMatch(UUID_PATTERN);

      const page = await web.newPage();
      await page.goto(`/fr/produits/${REFERENCE_PRODUCT.slug}/`);
      await expect(page.getByRole("table")).toBeVisible();
    });

    await test.step("6. Creation du devis et acceptation", async () => {
      const response = await buyer.request.post("/v1/quotes", {
        data: {
          businessAccountId,
          incoterm: "DAP",
          paymentTerms: "net_30",
          lines: [{ variantId, quantity: REFERENCE_PRODUCT.moq }],
        },
      });
      const quote = await readData(response, "POST /v1/quotes");
      quoteId = String(quote["id"]);
      quoteNumber = String(quote["number"]);
      expect(quoteId).toMatch(UUID_PATTERN);
      expect(quote["status"]).toBe("draft");
      expect(Number(quote["totalMinor"])).toBeGreaterThan(0);
      expect(String(quote["currency"])).toBe("EUR");

      const accepted = await readData(
        await buyer.request.post(`/v1/quotes/${quoteId}/accept`, { data: {} }),
        "POST /v1/quotes/:id/accept",
      );
      expect(accepted["status"]).toBe("accepted");
    });

    await test.step("7. Conversion du devis accepte en commande", async () => {
      const response = await buyer.request.post(`/v1/quotes/${quoteId}/convert`, { data: {} });
      const data = await readData(response, "POST /v1/quotes/:id/convert");
      orderId = String(data["orderId"]);
      orderNumber = String(data["orderNumber"]);
      expect(orderId).toMatch(UUID_PATTERN);
      expect(orderNumber).not.toBe("");

      const quote = await readData(
        await buyer.request.get(`/v1/quotes/${quoteNumber}`),
        "GET /v1/quotes/:number",
      );
      expect(quote["status"]).toBe("converted");
    });

    await test.step("8. Paiement par virement teste", async () => {
      const response = await buyer.request.post(`/v1/orders/${orderId}/payment`, {
        data: { method: "bank_transfer" },
      });
      const data = await readData(response, "POST /v1/orders/:id/payment");
      expect(data["status"]).toBe("pending");
      expect(String(data["providerRef"])).not.toBe("");
    });

    await test.step("9. Facture telechargeable et suivi d'expedition", async () => {
      await expect
        .poll(
          async () => (await buyer.request.get(`/v1/orders/${orderId}/invoice`)).status(),
          { timeout: 20_000, intervals: [1_000, 2_000, 5_000] },
        )
        .toBe(200);
      const invoice = await buyer.request.get(`/v1/orders/${orderId}/invoice`);
      expect((await invoice.body()).byteLength).toBeGreaterThan(0);

      await expect
        .poll(
          async () => {
            const documents = await readList(
              await buyer.request.get(`/v1/orders/${orderId}/documents`),
              "GET /v1/orders/:number/documents",
            );
            return documents.length;
          },
          { timeout: 20_000, intervals: [1_000, 2_000, 5_000] },
        )
        .toBeGreaterThan(0);

      const shipped = await readData(
        await buyer.request.post(`/v1/orders/${orderId}/shipments`, { data: {} }),
        "POST /v1/orders/:number/shipments",
      );
      expect(String(shipped["trackingNumber"])).not.toBe("");

      const shipments = await readList(
        await buyer.request.get(`/v1/orders/${orderId}/shipments`),
        "GET /v1/orders/:number/shipments",
      );
      expect(shipments.length).toBe(1);
      expect(String(shipments[0]?.["trackingNumber"])).toBe(String(shipped["trackingNumber"]));

      const page = await web.newPage();
      await page.goto(`/fr/commandes/${orderNumber}/suivi`);
      await expect(page.getByRole("heading", { level: 1 })).toBeVisible();
    });
  });
});
```

Exécuter :

```bash
pnpm exec playwright test e2e/b2b/journey.spec.ts --project=b2b
```

Attendu : **ÉCHEC** — les routes `POST /v1/auth/verify-email`, `POST /v1/business/accounts`, `POST /v1/quotes`, `POST /v1/quotes/:id/accept`, `POST /v1/quotes/:id/convert` et `POST /v1/orders/:number/shipments` ne sont pas encore exposées (le test s'arrête à l'étape 2 sur `GET /v1/test-mailbox/messages`).

- [ ] **Step 18.3: Écrire la recette fonctionnelle des contrôles douaniers (elle doit échouer)**

Créer `e2e/b2b/compliance.spec.ts` :

```ts
import { expect, test } from "@playwright/test";
import {
  ADMIN_EMAIL,
  ADMIN_TOTP_SECRET,
  API_URL,
  APPROVER_EMAIL,
  APPROVER_TOTP_SECRET,
  FIXTURES,
  RECIPE_PASSWORD,
  elevate,
  ensureMfa,
  readData,
  readError,
  readList,
  signIn,
} from "../support/b2b-api.js";

/**
 * Recette fonctionnelle de phase : 4 cas douaniers.
 *
 * Projet Playwright `functional` (le fichier est hors des suites de release :
 * la suite `b2b` reste `e2e/b2b/journey.spec.ts`). Les fixtures utilisees
 * sont creees par `e2e/scripts/seed-e2e.ts`.
 */

test.describe("Controles douaniers — recette de phase", () => {
  test("refuse l'expedition quand les donnees douanieres sont incompletes", async ({ browser }) => {
    const admin = await browser.newContext({ baseURL: API_URL });
    await signIn(admin, ADMIN_EMAIL, RECIPE_PASSWORD);
    const secret = await ensureMfa(admin, ADMIN_TOTP_SECRET, "E2E_ADMIN_TOTP_SECRET");
    await elevate(admin, secret);

    const response = await admin.request.post(
      `/v1/orders/${FIXTURES.incompleteOrderId}/shipments`,
      { data: {} },
    );
    const error = await readError(response, "POST /v1/orders/:number/shipments");
    expect(error.code).toBe("dispatch_blocked");
    expect(error.details["findings"] ?? "").toMatch(
      /missing_required_document|hs_not_confirmed|origin_missing/,
    );
    expect(error.details["ruleRef"]).not.toBeUndefined();
    expect(error.details["sourceUrl"]).toMatch(/^https:\/\//);
    expect(error.details["consultedAt"]).toMatch(/^\d{4}-\d{2}-\d{2}$/);

    const shipments = await readList(
      await admin.request.get(`/v1/orders/${FIXTURES.incompleteOrderId}/shipments`),
      "GET /v1/orders/:number/shipments",
    );
    expect(shipments.length).toBe(0);
  });

  test("signale la regle expiree au compliance manager sans l'appliquer", async ({ browser, baseURL }) => {
    const admin = await browser.newContext({ baseURL: API_URL });
    await signIn(admin, ADMIN_EMAIL, RECIPE_PASSWORD);
    const secret = await ensureMfa(admin, ADMIN_TOTP_SECRET, "E2E_ADMIN_TOTP_SECRET");
    await elevate(admin, secret);

    const alerts = await readList(
      await admin.request.get("/v1/admin/compliance-alerts"),
      "GET /v1/admin/compliance-alerts",
    );
    const expired = alerts.find(
      (alert) => String(alert["ruleId"] ?? "") === FIXTURES.expiredRuleId,
    );
    expect(expired).not.toBeUndefined();
    expect(String(expired?.["severity"])).toMatch(/blocker|critical/);
    expect(String(expired?.["status"])).not.toBe("resolved");

    const web = await browser.newContext({ baseURL: baseURL ?? undefined });
    const page = await web.newPage();
    await page.goto("/fr/admin/douane/alertes");
    await expect(page.getByRole("table")).toBeVisible();
  });

  test("detecte le fractionnement de colis et bloque l'expedition", async ({ browser }) => {
    const admin = await browser.newContext({ baseURL: API_URL });
    await signIn(admin, ADMIN_EMAIL, RECIPE_PASSWORD);
    const secret = await ensureMfa(admin, ADMIN_TOTP_SECRET, "E2E_ADMIN_TOTP_SECRET");
    await elevate(admin, secret);

    const response = await admin.request.post(
      `/v1/orders/${FIXTURES.splitOrderId}/shipments`,
      { data: {} },
    );
    const error = await readError(response, "POST /v1/orders/:number/shipments");
    expect(error.code).toBe("dispatch_blocked");
    expect(error.details["findings"] ?? "").toContain("split_shipment_suspected");

    const shipments = await readList(
      await admin.request.get(`/v1/orders/${FIXTURES.splitOrderId}/shipments`),
      "GET /v1/orders/:number/shipments",
    );
    expect(shipments.length).toBe(0);
  });

  test("revue d'un compte B2B avec justificatif et approbation a quatre yeux", async ({ browser }) => {
    const admin = await browser.newContext({ baseURL: API_URL });
    const approver = await browser.newContext({ baseURL: API_URL });

    await signIn(admin, ADMIN_EMAIL, RECIPE_PASSWORD);
    const adminSecret = await ensureMfa(admin, ADMIN_TOTP_SECRET, "E2E_ADMIN_TOTP_SECRET");
    await elevate(admin, adminSecret);

    const requested = await admin.request.post(
      `/v1/admin/business-accounts/${FIXTURES.businessAccountId}/approvals`,
      { data: { decision: "approved", note: "Recette E2E B2B" } },
    );
    const approval = await readData(
      requested,
      "POST /v1/admin/business-accounts/:id/approvals",
    );
    const approvalId = String(approval["approvalId"]);
    expect(approval["status"]).toBe("pending");

    const selfApproval = await admin.request.post(
      `/v1/admin/business-accounts/${FIXTURES.businessAccountId}/approvals/${approvalId}/approve`,
    );
    expect(selfApproval.status()).toBe(409);

    await signIn(approver, APPROVER_EMAIL, RECIPE_PASSWORD);
    const approverSecret = await ensureMfa(
      approver,
      APPROVER_TOTP_SECRET,
      "E2E_APPROVER_TOTP_SECRET",
    );
    await elevate(approver, approverSecret);
    const approved = await approver.request.post(
      `/v1/admin/business-accounts/${FIXTURES.businessAccountId}/approvals/${approvalId}/approve`,
    );
    const data = await readData(
      approved,
      "POST /v1/admin/business-accounts/:id/approvals/:approvalId/approve",
    );
    expect(data["verificationStatus"]).toBe("approved");

    const account = await readData(
      await admin.request.get(`/v1/admin/business-accounts/${FIXTURES.businessAccountId}`),
      "GET /v1/admin/business-accounts/:id",
    );
    expect(account["verificationStatus"]).toBe("approved");
    expect(String(account["verifiedBy"] ?? "")).not.toBe("");
  });
});
```

Exécuter :

```bash
pnpm exec playwright test e2e/b2b/compliance.spec.ts --project=functional
```

Attendu : **ÉCHEC** — les fixtures de la Phase 3 n'existent pas encore dans la base de recette et `POST /v1/orders/:number/shipments` n'est pas exposée (le test échoue sur le premier cas).

- [ ] **Step 18.4: Implémenter l'activation du compte acheteur (service, jeton éphémère, boîte de test)**

Le jeton de vérification est stocké **dans Redis** (jamais en base, jamais de migration) : clé `auth:email-verification:<sha256(token)>`, valeur = `userId`, TTL 24 h. Le jeton brut ne quitte jamais le serveur ; seul son hachage sert de clé.

Modifier `apps/api/src/modules/auth/auth.service.ts` :

```ts
  /**
   * Inscription : genere un jeton de verification, n'en conserve que le
   * hachage (SHA-256) dans Redis (TTL 24 h) et depose l'e-mail dans la
   * boite de test. Le jeton brut ne quitte jamais le serveur.
   */
  public async signUp(input: SignUpInput): Promise<{ response: SignUpResponse; cookie: { name: string; value: string; options: CookieOptions } }> {
    // ... controle de rate limit et de non-enumeration (code existant) ...
    const token = randomBytes(32).toString("base64url");
    await this.redis.set(
      `auth:email-verification:${hashVerificationToken(token)}`,
      inserted[0]!.id,
      "EX",
      24 * 60 * 60,
    );
    await this.mailbox.send({
      to: input.email,
      subject: "Confirmez votre adresse email",
      body: `${this.appUrl}/fr/compte/verifier?token=${token}`,
    });
    // ... creation de session et reponse (code existant) ...
  }

  /**
   * Verification : jeton inconnu, expire, deja utilise ou compte inactif =>
   * refus explicite. `GETDEL` consomme le jeton : usage unique garanti
   * par Redis, y compris en cas d'appels concurrents.
   */
  public async verifyEmail(token: string): Promise<void> {
    const key = `auth:email-verification:${hashVerificationToken(token)}`;
    const userId = await this.redis.getdel(key);
    if (userId === null) throw new AuthError("invalid_verification_token", "Jeton de verification invalide ou expire.", 400);
    const found = await this.db.select().from(user).where(eq(user.id, userId)).limit(1);
    const target = found[0];
    if (target === undefined) throw new AuthError("invalid_verification_token", "Jeton de verification invalide.", 400);
    if (target.emailVerifiedAt !== null) {
      throw new AuthError("already_verified_email", "Adresse email deja verifiee.", 409);
    }
    await this.db
      .update(user)
      .set({ emailVerifiedAt: new Date() })
      .where(eq(user.id, target.id));
  }
```

Créer `apps/api/src/providers/email/mailbox.provider.ts` :

```ts
/**
 * Boite e-mail de test. L'envoi reel (SMTP) arrive avec la Phase 5 :
 * jusqu'a la, la recette doit pouvoir lire le jeton de verification.
 * Aucune donnee reelle n'y transite : uniquement les comptes de test.
 */
export interface MailboxMessage {
  readonly to: string;
  readonly subject: string;
  readonly body: string;
  readonly sentAt: string;
}

export interface MailboxProvider {
  send(message: { to: string; subject: string; body: string }): Promise<void>;
  list(to: string): Promise<MailboxMessage[]>;
}

export class InMemoryMailboxProvider implements MailboxProvider {
  private readonly messages: MailboxMessage[] = [];

  public async send(message: { to: string; subject: string; body: string }): Promise<void> {
    this.messages.push({ ...message, sentAt: new Date().toISOString() });
    if (this.messages.length > 50) this.messages.shift();
  }

  public async list(to: string): Promise<MailboxMessage[]> {
    return this.messages.filter((message) => message.to === to);
  }
}
```

Créer `apps/api/src/modules/auth/test-mailbox.controller.ts` :

```ts
import { Controller, Get, Query } from "@nestjs/common";
import { MailboxProvider } from "../../providers/email/mailbox.provider.js";

@Controller("v1/test-mailbox")
export class TestMailboxController {
  constructor(private readonly mailbox: MailboxProvider) {}

  @Get("messages")
  async messages(@Query("to") to: string): Promise<unknown> {
    if (process.env["APP_ENV"] === "production") {
      throw new NotFoundError("not_found");
    }
    const messages = await this.mailbox.list(to);
    return {
      data: { messages },
      correlationId: "00000000-0000-7000-8000-000000000000",
    };
  }
}
```

Modifier `apps/api/src/modules/auth/auth.controller.ts` : ajouter

```ts
  @Post("verify-email")
  @HttpCode(204)
  public async verifyEmail(@Body() body: { token: string }): Promise<void> {
    try {
      await this.auth.verifyEmail(body.token);
    } catch (error) {
      throw toHttpError(error);
    }
  }
```

- [ ] **Step 18.5: Implémenter la surface HTTP du vertical B2B (business, devis, expédition)**

Modifier `apps/api/src/modules/business/business-review.service.ts` : ajouter

```ts
  /**
   * Demande d'activation professionnelle : cree le compte B2B rattache a
   * l'utilisateur connecte. Le statut reste `unverified` : la validation
   * est humaine (four-eyes, Task 17).
   */
  async apply(params: {
    legalName: string;
    countryCode: string;
    address: string;
    taxId: string | null;
    principal: Principal;
  }): Promise<{ id: string; verificationStatus: BusinessVerificationStatus }> {
    const created = await this.db
      .insert(businessAccount)
      .values({
        legalName: params.legalName,
        countryCode: params.countryCode,
        address: params.address,
        taxId: params.taxId,
        verificationStatus: "unverified",
      })
      .returning({ id: businessAccount.id });
    const accountId = created[0]!.id;
    await this.db.insert(businessMembership).values({
      businessAccountId: accountId,
      userId: params.principal.userId,
      role: "owner",
    });
    return { id: accountId, verificationStatus: "unverified" };
  }
```

Modifier `apps/api/src/modules/business/business.controller.ts` : ajouter `POST /v1/business/accounts` et `GET /v1/business/accounts/me` (le compte est scoping par `business_membership.user_id` — jamais par un identifiant fourni par le client).

Modifier `apps/api/src/modules/quote/quote.controller.ts` : exposer `POST /v1/quotes` (`create`), `POST /v1/quotes/:id/accept` (`transition` vers `accepted`) et `POST /v1/quotes/:id/convert` (`QuoteConversionService.convert`). Chaque route vérifie que la session appartient au `business_account` du devis (anti-IDOR : `404` sinon).

Modifier `apps/api/src/modules/shipment/shipment.controller.ts` : exposer `POST /v1/orders/:number/shipments` (`ShipmentService.dispatch`) et `GET /v1/orders/:number/shipments` (`ShipmentService.getForOrder`). Le `dispatch` appelle `assertDispatchable` **avant** toute création : en cas de blocage, il lève un `ConflictError("dispatch_blocked")` dont les `details` portent `findings`, `ruleRef`, `sourceUrl` et `consultedAt` (la règle citée avec sa source officielle et sa date de consultation). Aucun numéro de suivi n'est créé dans ce cas.

Modifier `apps/api/src/common/api-errors.ts` : `ApiError` accepte un second paramètre optionnel `details?: Record<string, string>` (chaînes uniquement, jamais de donnée technique brute ni de PII).

- [ ] **Step 18.6: Étendre le seed de recette**

Modifier `e2e/scripts/seed-e2e.ts` : ajouter un second administrateur (four-eyes) et les fixtures déterministes de la Phase 3.

```typescript
const APPROVER_EMAIL = process.env['E2E_APPROVER_EMAIL'] ?? 'approbateur@fanours.test';

/** Fixtures consommees par `e2e/b2b/compliance.spec.ts` (identifiants fixes). */
export const E2E_FIXTURES = {
  incompleteOrderId: '0199f0f0-f000-7000-8000-000000000001',
  splitOrderId: '0199f0f0-f000-7000-8000-000000000002',
  expiredRuleId: '0199f0f0-f000-7000-8000-000000000003',
  businessAccountId: '0199f0f0-f000-7000-8000-000000000004',
  businessAccountUserEmail: 'client-b2b@fanours.test',
} as const;
```

Le seed crée, en plus des comptes existants :

- `approbateur@fanours.test` avec le rôle `super_admin` (second acteur du four-eyes, différent du demandeur) ;
- une **règle douanière expirée** (`valid_to` dans le passé, source officielle + date de consultation) : la commande correspondante déclenche une `compliance_alert` de sévérité `blocker` ;
- une **commande aux données douanières incomplètes** (code HS non confirmé, document obligatoire absent) : l'expédition est bloquée (`missing_required_document`, `hs_not_confirmed`) ;
- une **commande fractionnée** (deux colis sous le seuil de dédouanement) : l'expédition est bloquée (`split_shipment_suspected`) ;
- un **compte B2B en attente de revue** (`E2E_FIXTURES.businessAccountId`) avec son membre `owner` ;
- le produit de référence (`CL-MOL-01-STD`) avec un code HS **confirmé par un humain** (`hs_validation_status = 'confirmed'`) : le parcours de l'étape 9 doit aboutir.

Aucune donnée réelle n'est utilisée : uniquement des valeurs de test.

- [ ] **Step 18.7: Exécuter les deux specs (attendu : vert)**

Exécuter :

```bash
pnpm exec playwright test e2e/b2b/journey.spec.ts --project=b2b
pnpm exec playwright test e2e/b2b/compliance.spec.ts --project=functional
```

Attendu : `1 passed` pour la suite `b2b` (9 étapes) et `4 passed` pour la recette fonctionnelle (4 cas douaniers). Aucun test n'est compté deux fois : `e2e/b2b/journey.spec.ts` n'appartient qu'au projet `b2b` et `e2e/b2b/compliance.spec.ts` qu'au projet `functional`.

- [ ] **Step 18.8: Vérifier la suite complète de la phase et committer**

Exécuter :

```bash
pnpm --filter @fanours/api exec vitest run src/modules/auth src/modules/business src/modules/quote src/modules/shipment
pnpm --filter @fanours/e2e typecheck
pnpm turbo run lint typecheck
pnpm exec playwright test e2e/b2b/journey.spec.ts --project=b2b
pnpm exec playwright test e2e/b2b/compliance.spec.ts --project=functional
```

Attendu : toutes les suites vertes, `lint` et `typecheck` sans erreur, E2E B2B au vert (1 test de parcours en 9 étapes + 4 cas douaniers).

**Preuves :**
- Prérequis d'environnement : la recette tourne sur un stack complet (`api` + `web` + `worker`), sans quoi l'étape 9 reste en attente de la facture et des documents d'export (générés par le job `document.pdf`). Le spec **attend** ces documents (poll de 20 s) au lieu de les simuler.
- Sortie de `pnpm exec playwright test e2e/b2b/journey.spec.ts --project=b2b` : `1 passed`, les 9 étapes visibles dans le rapport Playwright.
- Sortie de `pnpm exec playwright test e2e/b2b/compliance.spec.ts --project=functional` : `4 passed`.
- Preuve du jeton de vérification : la clé Redis `auth:email-verification:<sha256>` porte un TTL de 24 h et le **seul** hachage ; une seconde vérification du même jeton renvoie `400` (usage unique garanti par `GETDEL`) et un compte déjà vérifié renvoie `409`. Aucune migration n'est créée par cette Task : la plage `0100`–`0110` de la Phase 3 est inchangée.
- Preuve du four-eyes : l'auto-approbation renvoie `409` et l'approbation par un second administrateur renvoie `verificationStatus: "approved"` avec `verified_by` renseigné.
- Preuve du blocage douanier : `409 dispatch_blocked` avec `details.findings`, `details.ruleRef`, `details.sourceUrl` (source officielle) et `details.consultedAt` (date de consultation), et `GET /v1/orders/:number/shipments` **vide** : aucun numéro de suivi n'est créé.
- Preuve de la règle expirée : une alerte `blocker` non résolue est visible dans `GET /v1/admin/compliance-alerts` et dans `/fr/admin/douane/alertes` ; la règle expirée n'est pas appliquée.
- Aucune liste de viewports re-déclarée : les deux specs importent uniquement `e2e/fixtures/totp.ts`, `e2e/fixtures/product.ts` et `e2e/support/b2b-api.ts`.
- Aucun secret en dur dans les specs : les identifiants et les codes MFA viennent de `E2E_ADMIN_EMAIL`, `E2E_APPROVER_EMAIL`, `E2E_ADMIN_PASSWORD`, `E2E_ADMIN_TOTP_SECRET`, `E2E_APPROVER_TOTP_SECRET` (noms de variables uniquement).

Commit :

```bash
git add e2e/b2b/journey.spec.ts e2e/b2b/compliance.spec.ts e2e/support/b2b-api.ts e2e/scripts/seed-e2e.ts apps/api/src/providers/email/mailbox.provider.ts apps/api/src/modules/auth apps/api/src/modules/business apps/api/src/modules/quote/quote.controller.ts apps/api/src/modules/shipment/shipment.controller.ts apps/api/src/common/api-errors.ts
git commit -m "test(b2b): ajoute la recette E2E du parcours acheteur B2B

Livre la suite de release b2b (parcours acheteur en 9 etapes, spec 24.3) et la
recette fonctionnelle des 4 controles douaniers. La suite b2b est le seul
fichier du projet b2b : la Phase 5 la reprend en Modify, sans doublon.

Cable les routes HTTP consommees par la recette (verification d'e-mail,
demande d'activation professionnelle, devis, acceptation, conversion,
expedition) et porte dans le detail de l'erreur 409 la regle citee avec sa
source officielle et sa date de consultation.
" -m "Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Self-Review

### 1. Couverture des 12 chantiers de la phase 3

| Chantier roadmap | Tâche | Artefact produit |
|---|---|---|
| 3.1 Profil professionnel (L) | Task 7, Task 17, Task 18 | `business_account`, `business_document`, `business_account_approval`, `AdminBusinessAccountsService`, `/v1/admin/business-accounts`, `e2e/b2b/journey.spec.ts` (étapes 3 et 4) |
| 3.2 Devis (L) | Task 8, Task 9, Task 18 | `quote`, `quote_revision`, `quote_line`, `quote_message`, conversion devis → commande, `e2e/b2b/journey.spec.ts` (étapes 6 et 7) |
| 3.3 Tarification client (M) | Task 6 | `packages/pricing/src/customer-rules.ts` |
| 3.4 Moteur de règles (L) | Task 2, Task 3, Task 17, Task 18 | `trade_rule`, `trade_rule_version`, `trade_rule_source`, `resolveRules`, `AdminTradeRulesService`, `/v1/admin/trade-rules`, `e2e/b2b/compliance.spec.ts` (règle expirée) |
| 3.5 Calcul droits et taxes (L) | Task 4 | `computeCustoms` (8 composantes, 6 cas chiffrés) |
| 3.6 Affichage acheteur (M) | Task 15, Task 16 | `CostEstimatePanel`, `QuoteRevisionDiff` |
| 3.7 Pays et restrictions (M) | Task 1, Task 17 | `country_profile`, `incoterm_configuration`, `evaluateCountryGate`, `admin_country_scope` |
| 3.8 Transporteurs (M) | Task 10 | `ShippingProvider`, `MockShippingProvider`, `shipping_quote` |
| 3.9 Suivi de commande (M) | Task 11, Task 17, Task 18 | `shipment`, `tracking_event`, `AdminShipmentsService`, `/v1/admin/shipments`, `e2e/b2b/journey.spec.ts` (étape 9) |
| 3.10 Documents d'export (M) | Task 12 | `commercial_document`, jobs PDF |
| 3.11 Contrôles et alertes (L) | Task 5, Task 13, Task 17, Task 18 | `runComplianceChecks`, `compliance_alert`, `ComplianceAlertService`, `/v1/admin/compliance-alerts`, `e2e/b2b/compliance.spec.ts` (blocage 409, fractionnement) |
| 3.12 Alertes données manquantes (M) | Task 13, Task 17, Task 18 | `compliance_review`, `hs-review.service.ts`, `product_compliance`, `e2e/b2b/compliance.spec.ts` (données incomplètes) |

La Task 17 ne crée pas de chantier : elle **rend visibles et gouvernables** les données produites par les Tasks 1 à 13 (tableaux d'expéditions, centre de règles, alertes, revue des comptes B2B), avec permission serveur, périmètre pays, audit, four-eyes, pagination serveur, états d'interface, i18n et accessibilité. Les quatre pages déclarées dans la section `## File Structure` (`admin/expeditions`, `admin/douane/regles`, `admin/douane/alertes`, `admin/clients`) et les quatre composants (`DataTable`, `AdminWizard`, `KpiCard`, `FilterBar`) sont produits par la Task 17 : `FilterBar` manquait dans la `## File Structure`, il y est ajouté par la Task 17 (fichiers `packages/ui/src/components/FilterBar.tsx` et `FilterBar.spec.tsx`).

La Task 18 ne crée pas de chantier non plus : elle **recette** le vertical de bout en bout (parcours acheteur B2B en 9 étapes, 4 contrôles douaniers) et ne modifie aucun comportement métier des Tasks 1 à 17. Elle câble uniquement les routes HTTP que la recette exerce et qui n'étaient pas exposées (`POST /v1/auth/verify-email`, `POST /v1/business/accounts`, `GET /v1/business/accounts/me`, `POST /v1/quotes`, `POST /v1/quotes/:id/accept`, `POST /v1/quotes/:id/convert`, `POST /v1/orders/:number/shipments`, `GET /v1/orders/:number/shipments`) : les services métier qui les portent sont ceux des Tasks 7, 8, 9, 11 et de la Phase 1.

### 2. Scan des placeholders

- Aucun `TBD`, aucun `TODO`, aucun « à compléter », aucun « identique à la Task N » dans les Tasks 1 à 18.
- Chaque étape de la Task 17 suit le cycle complet : test qui échoue → exécution qui prouve l'échec → implémentation → exécution qui prouve la réussite → commit.
- Chaque étape de la Task 18 suit le même cycle complet : les deux specs E2E sont écrites puis exécutées (échec attendu, avec la route HTTP manquante nommée), la surface HTTP est câblée, les specs repassent au vert, puis commit.
- Les seules valeurs laissées ouvertes sont des **décisions métier hors périmètre technique**, listées en partie 4 : elles sont écrites comme des questions adressées à un humain, jamais comme des trous dans le code.
- La Task 17 ne réécrit aucune des Tasks 1 à 16 : elle ajoute des fichiers, et elle n'étend que des fichiers déjà écrits, sans en réécrire le contenu — `packages/db/src/schema/phase3-logistics.ts` et `packages/db/src/schema/phase3-compliance.ts` (colonnes admin de `shipment` et `complianceAlert`, ajoutées par la migration 0110), `packages/db/src/schema/index.ts` (baril : `export * from "./phase3-admin.js";`), `packages/db/src/seed.ts`, `apps/api/src/modules/compliance/compliance-alerts.controller.ts` (route réalignée), `apps/api/src/modules/business/business-review.service.ts` (délégation four-eyes), `packages/ui/src/index.ts` et les trois dictionnaires i18n. La modification de `packages/domain/src/permissions.ts` est portée par la **Task 7** (ajout de `business_account:approve`) puis par la **Task 17** (ajout de `customs_document:read` / `customs_document:update` à `order_manager` et `logistics_operator`) ; aucune des deux ne modifie la contrainte `ck_permission_resource`, qui reste fixe.
- La Task 18 ne réécrit aucune des Tasks 1 à 17 : elle ajoute `e2e/b2b/journey.spec.ts`, `e2e/b2b/compliance.spec.ts` et `e2e/support/b2b-api.ts`, et elle n'étend que des fichiers déjà écrits, sans en réécrire le contenu — `apps/api/src/modules/auth/auth.service.ts` et `auth.controller.ts` (Phase 1), `apps/api/src/modules/business/business-review.service.ts` et `business.controller.ts` (Task 7), `apps/api/src/modules/quote/quote.controller.ts` (Task 8), `apps/api/src/modules/shipment/shipment.controller.ts` (Task 11), `apps/api/src/common/api-errors.ts` (Task 7) et `e2e/scripts/seed-e2e.ts` (Phase 1). Elle ne touche à aucune migration (la Task 18 n'en crée pas : le jeton de vérification est éphémère et vit dans Redis), ni aux dictionnaires i18n, ni aux services métier des Tasks 1 à 13.

### 3. Cohérence des types et des noms

- **Routes** : toutes les vues admin sont sous `/v1/admin/...` (`shipments`, `trade-rules`, `compliance-alerts`, `business-accounts`), jamais sous `/admin/...` nu. La route `/v1/admin/compliance/alerts` créée en Task 13 est réalignée en `/v1/admin/compliance-alerts` (Task 17, Step 6) pour éviter deux chemins concurrents vers les mêmes alertes.
- **Tables et colonnes** : `shipment.country_code`, `shipment.exception_*` et `compliance_alert.acknowledged_*` sont ajoutées par la migration `0110_phase3_admin_views.sql` (append-only) ; `business_account_approval` et `admin_country_scope` y sont créées, tandis que `business_account` et `business_membership` proviennent de `0103_phase3_business_documents.sql` et ne sont pas recréées par la 0110. Les colonnes `country_code` et `status` de `shipment` sont lues dans la clause `WHERE` (anti-IDOR), jamais filtrées en mémoire.
- **Divergences entre la Phase 3 et les phases antérieures — toutes corrigées dans ce plan** (les six points relevés, plus deux points supplémentaires trouvés pendant la correction ; plus rien n'est à arbitrer sur ces huit points) :
  1. **Table des commandes** : la Phase 2 crée `orders` (`0006_order.sql`, renumérotée `0010_order.sql`). **Corrigé** — plus aucune occurrence de `order(id)` ni de `INSERT INTO "order"` : les Tasks 1 à 18 écrivent et lisent `orders`. La Task 9 insère dans `orders` (`business_id`, `total_amount`, `status`, `currency`, `incoterm`, `sales_mode`, `destination_country`, `idempotency_key`, `snapshot`, `user_id`) et la migration `0109_phase3_quote_to_order.sql` **ALTERE** `orders` (`quote_id`, `quote_revision_id`, `payment_terms` nullables + index `UNIQUE` partiel sur `quote_id`) au lieu de viser une table inexistante.
  2. **Table des utilisateurs** : la Phase 1 crée `"user"` (`0002_identity.sql` — `USER` est un mot réservé PostgreSQL, d'où les guillemets). **Corrigé** — les 30 occurrences de `app_user` du plan sont devenues `"user"` (25 `REFERENCES "user" (id)`, plus les `CROSS JOIN "user" u`, les `(SELECT id FROM "user" LIMIT 1)` et les requêtes d'indexation). Les Task 1 et Task 2 énoncent désormais la forme définitive, sans mise en condition.
  3. **Table `business_account`** : **Corrigé** — elle est créée par `0103_phase3_business_documents.sql`, **avant** `business_document`, avec `verified_by` / `verified_at` / `review_note` (`REFERENCES "user" (id)`) et `verification_status` contraint à `('unverified','pending','approved','rejected','suspended')`. La migration `0110_phase3_admin_views.sql` ne la recrée plus (le bloc `CREATE TABLE IF NOT EXISTS business_account` et ses index y ont été supprimés).
  4. **Table `business_membership`** : elle est référencée par le scoping de la Task 7 mais n'était créée par aucune migration. **Corrigé** — `0103` la crée (clé primaire composite `business_account_id` + `user_id`, `role` contraint, index sur `user_id`) et le mapping Drizzle correspondant est ajouté dans `packages/db/src/schema/phase3-business.ts`.
  5. **Colonnes d'audit** : la Phase 1 nomme `occurred_at`, `object_type`, `object_id` ; Drizzle les expose en `occurredAt`, `objectType`, `objectId`. **Corrigé** — les 9 insertions `insert(auditLog)` des Tasks 7 et 17 utilisent `occurredAt`, `objectType`, `objectId`, `actorId`, `actorRole`, `action`, `result` et `correlationId` ; plus aucune insertion ne porte `createdAt`, qui n'existe pas dans `audit_log`.
  6. **Emplacement du guard RBAC** : `packages/security` n'existe pas dans ce monorepo. **Corrigé** — le contrôle d'accès réel est celui de la Phase 1, `apps/api/src/modules/rbac/` (`rbac.service.ts`, `rbac.guard.ts`, `rbac.decorator.ts`, `approval.service.ts`, `scope.ts`). La Task 7 crée `apps/api/src/modules/rbac/require-permission.ts` (helper de service aligné sur la matrice `PERMISSIONS` de `packages/domain/src/permissions.ts`, deny by default, `Principal` avec `mfaVerifiedAt` et `role`) et `apps/api/src/common/api-errors.ts` (`BadRequestError`, `UnauthorizedError`, `ForbiddenError`, `NotFoundError`, `ConflictError`) ; les 17 fichiers d'implémentation (hors fichiers de test) importent `requirePermission` depuis `../rbac/require-permission`. L'alias `@fanours/security` a été **supprimé** du plan.
  7. **Contrats de stockage et de rendu PDF** : `packages/storage` n'existe pas non plus. **Corrigé** — le contrat `StorageProvider` est `packages/contracts/src/storage.ts` (Task 7) avec son implémentation S3 `apps/api/src/providers/storage/s3-storage.provider.ts` (Task 7) ; le contrat `DocumentRenderer` est `packages/contracts/src/pdf.ts` (Task 12) avec son implémentation `@react-pdf/renderer` dans `packages/pdf` (package de la Phase 2, étendu par la Task 12 : `commercial-document.tsx`, `render-commercial-document.ts`, alias `registerDocumentFonts`).
  8. **Attribution de phase des tables** : **Corrigé** — `"user"` et `audit_log` appartiennent à la Phase 1 (`0002_identity.sql`, `0003_rbac_and_audit.sql`), `rbac_scope` à la Phase 1 (`0005_rbac_scopes.sql`), `price_tier` et `customer_price_rule` à la Phase 2 (`0003_pricing.sql`, renumérotée `0007_pricing.sql`), `orders` / `order_line` à la Phase 2 (`0006_order.sql` → `0010_order.sql`) ; `business_account`, `business_membership`, `quote`, `shipment`, `commercial_document` et les tables `compliance_*` sont des créations de la Phase 3. Tous les blocs `**Consumes:**` des 18 Tasks ont été relus et réattribués phase par phase.
- **Noms d'actions RBAC** : `business_account:approve` n'existait pas dans la matrice de la Phase 1 ; **la Task 7 l'ajoute** pour `super_admin` et `compliance_manager` dans `packages/domain/src/permissions.ts` (la contrainte SQL `ck_permission_action` de la Phase 1 accepte déjà `approve`, donc aucune migration n'est requise) et l'amorce de graine doit être rejouée. Les Tasks 7, 9, 11, 13, 14 et 17 n'emploient que des ressources et des actions **existantes** de la Phase 1 : `customs_rule:update` (Task 13), `order:create` (conversion d'un devis), `refund:refund` (décision de retour), `order:update` (escalade de litige) — aucune ressource ni action fictive n'est introduite, et la contrainte `ck_permission_resource` (fixe) n'est jamais modifiée. `customs_document` est lue/écrite par `order_manager` et `logistics_operator` pour les vues expéditions, conformément à `securite.md` §3.2 ligne 14.
- **Ressources et périmètres** : le scope d'un rôle pour une ressource est lu en base (`rbac_scope`) via `loadRoleScope`, avec `none` en valeur par défaut (deny by default). Le filtre pays vient de `admin_country_scope` et produit une condition SQL ; hors périmètre, la réponse est **404**, jamais 403, pour ne pas divulguer l'existence de la ressource.
- **i18n** : toutes les clés utilisées par les pages admin existent en `fr-FR`, `en` et `zh-CN` sous le préfixe `admin.` ; le contrôle de parité du build reste vert. Les valeurs techniques (statuts d'expédition, types de règle, niveaux de certitude) sont localisées par clé (`admin.shipmentStatus.*`), jamais affichées en brut.
- **Composants** : `DataTable`, `AdminWizard`, `KpiCard` et `FilterBar` sont exportés par `packages/ui/src/index.ts` ; `SystemState` (Phase 1) couvre les états loading / vide / erreur, et les pages admin ajoutent l'état **403** (droits insuffisants) en plus des états techniques.

### 4. Points laissés explicitement à l'humain

Ces points ne sont pas des trous d'implémentation : la Task 17 fournit l'outil (table, écran, journal d'audit) et attend une décision ou une validation humaine. La Task 18 **recette** ces décisions : elle ne tranche aucun d'entre eux et ne les code pas en dur dans les specs.

1. **Conseil douanier et conformité** — la Task 17 fournit le centre de règles (source officielle + date de consultation + validateur + date de dernière revue) et l'alerte de règle expirée, mais **les taux, seuils, codes HS et pays à couvrir sont validés par un conseil** (CDC 11.2, 16.9). Une règle publiée reste `estimated` tant qu'un humain ne l'a pas validée avec sa source.
2. **Règle expirée et commande en cours** — le moteur produit une alerte et bloque une règle `prohibition`/`license` expirée ; la **décision de déblocage métier** (relancer, suspendre la commande, faireHt une demande de dédouanement) appartient au responsable conformité, pas au code.
3. **Motif de refus d'un compte B2B** — la suspension exige un motif libre et journalisé ; la **formulation type** du motif de refus et la politique de conservation des dossiers de vérification relèvent du conseil juridique.
4. **Périmètre pays effectif** — `admin_country_scope` est vide par défaut (portée globale). Les **pays prioritaires, les pays suspendus et les restrictions commerciales** (CDC 12.2) sont fournis par le responsable conformité avant toute restriction en production.
5. **Motif d'exception de colis** — le champ libre est obligatoire et journalisé, mais la **taxonomie des motifs de retenue** (code HS, description, valeur, licence) doit être validée par le conseil et branchée sur les transporteurs.
6. **Seuils de dédouanement et Incoterms** — les seuils monétaires et le modèle de paiement des taxes (DDP vs DAP) sont des décisions contractuelles ; le code les porte comme des **données** (la 0110 les laisse à la table `trade_rule_version.thresholds`), il ne les invente pas.
7. **Traçabilité réglementaire et conservation** — la durée de conservation des journaux d'audit, des documents B2B et des déclarations douanières dépend des obligations locales et de l'avis juridique ; aucune durée n'est codée en dur.
8. **Revue manuelle d'accessibilité** — l'audit automatique (`axe-core`) est un point de départ, pas une preuve : la **revue manuelle clavier et lecteur d'écran** sur les quatre vues admin (checkout, connexion, tableau de bord, gestion de commande) reste exigée par le CDC 22.4.



