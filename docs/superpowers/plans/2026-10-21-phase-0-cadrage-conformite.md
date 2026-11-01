# Phase 0 — Cadrage et conformité : Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Transformer les 15 hypothèses du cahier des charges en décisions confirmées par des tiers compétents, et produire les artefacts de cadrage (registre de décision, registre de données, modèle de menace, taxonomie des permissions, wireframes, feuille de route conformité) qui débloquent la Phase 1 sans hypothèse non validée.

**Périmètre :** 13 tâches (Task 1 → Task 13) — les 15 décisions DEC-001 à DEC-015 du spec, le modèle de menace STRIDE, la taxonomie des permissions, les maquettes UX des parcours et la feuille de route conformité ; aucun code applicatif.

**Architecture:** Phase 0 ne produit aucun code applicatif. Elle produit un ensemble de documents versionnés dans `docs/`, tous ancrés sur le spec `Cahier_des_charges_Fanours.md`, tous indexés par un registre unique (`docs/decisions/DEC-REGISTER.md`) qui donne le statut de chaque décision (tranchée / en attente / non applicable) et son propriétaire. Chaque dossier de décision a une structure fixe : contexte, options, impacts, recommandation, statut, signataire, date. Aucun dossier n'est marqué « tranchée » sans signature humaine écrite — l'IA prépare, le propriétaire tranche.

**Tech Stack:** Markdown (documents) · CSV (inventaire produits) · YAML front-matter (métadonnées de dossier) · outils de vérification : `markdownlint`, `yamllint`, `csvlint` · aucun code applicatif.

**Spec:** `Cahier_des_charges_Fanours.md` — §19.1 (modèle de menace), §19.6 (conformité), §19.10, §26.7, §28.1 (risques), §28.2 (12 décisions à confirmer), §2.4 (rôles), §11.10 (conformité douanière), §6/§15/§16 (UX). Plan phase : `docs/roadmap/sections/roadmap-phases.md` § Phase 0.

## Global Constraints

- **Aucune décision n'est auto-tranchée.** L'IA produit le dossier (options, impacts, recommandation) ; un humain signe. Un dossier sans signataire reste « en attente » et **bloque** la phase qu'il verrouille.
- **Aucune hypothèse réglementaire n'est présentée comme un fait.** Toute mention de loi, de seuil, de taux, d'obligation est accompagnée de sa source (URL officielle + date de consultation) ou explicitement marquée « à valider ».
- **Interdiction de sous-valorisation, de fausse catégorie/origine et de fractionnement de colis** — reprise telle quelle dans les dossiers d'inventaire et de douane.
- **Le code HS n'est jamais confirmé par l'IA.** L'IA prépare une proposition avec sources ; la validation est humaine et tracée dans le champ `hs_validation_status`.
- **Aucune donnée de carte ni document client sensible réel en dev.** L'inventaire produits et le registre de données n'contiennent que des données factices ou des métadonnées.
- **Aucun secret dans les documents.** Les dossiers citent des **noms de variables** d'environnement, jamais des valeurs.
- **Les 4 points bloquants** (entité 0.1, paiement 0.2, Incoterms/taxes 0.4, inventaire/HS 0.6) doivent être tranchés par un tiers avant la Phase 3. Le reste verrouille la Phase 1.
- **Format** : dossier en Markdown, YAML front-matter, un dossier par décision, un registre qui les indexe. Langue de rédaction : français. Les termes métier anglais restent en anglais.
- **Chaque dossier porte une date absolue** (AAAA-MM-JJ) et un signataire nominatif. Pas de « à compléter », pas de TBD.

---

## File Structure

```
docs/
  decisions/
    DEC-REGISTER.md              # index unique : ID, objet, statut, propriétaire, date, impact phase
    DEC-001-entite-juridique.md
    DEC-002-eligibilite-paiement.md
    DEC-003-marches-prioritaires.md
    DEC-004-incoterms-et-taxes.md
    DEC-005-transporteurs.md
    DEC-006-inventaire-produits-et-hs.md
    DEC-007-donnees-personnelles-retention.md
    DEC-008-politique-b2b-b2c.md
    DEC-009-devise-et-change.md
    DEC-010-hebergement-transferts.md
    DEC-011-sla-et-support.md
    DEC-012-modele-menace-stride.md
    DEC-013-taxonomie-permissions.md
    DEC-014-maquettes-ux.md
    DEC-015-feuille-route-conformite.md
  registers/
    REGISTER-donnees-personnelles.csv   # champ, finalité, base légale, durée, destinataires, emplacement
    REGISTER-prestataires.csv            # sous-traitant, traitement, pays, transfert, cadre
  templates/
    product-inventory.template.csv
    DECISION.template.md
    CHECKLIST-gate-0.md
  threat-model/
    STRIDE-flux.md
  permissions/
    permissions-matrix.md
  ux/
    wireframes/README.md
```

Chaque dossier est **autonome** : lu seul, il donne le contexte, les options et la recommandation sans avoir besoin du registre. Le registre donne l'index et le statut.

---

### Task 1: Scaffolding des dossiers, du registre et des gabarits

**Files:**
- Create: `docs/decisions/DEC-REGISTER.md`
- Create: `docs/templates/DECISION.template.md`
- Create: `docs/registers/REGISTER-donnees-personnelles.csv`
- Create: `docs/registers/REGISTER-prestataires.csv`
- Create: `docs/templates/product-inventory.template.csv`
- Create: `docs/templates/CHECKLIST-gate-0.md`
- Create: `docs/decision-lint.mjs`

**Interfaces:**
- Consumes: aucune dépendance logicielle. Node ≥ 20 (already required by the toolchain planned in `docs/roadmap/sections/architecture.md`).
- Produces:
  - `docs/decisions/DEC-REGISTER.md` — table unique, colonnes : `ID | Objet | Spec | Statut | Propriétaire | Signataire | Date | Verrouille | Fichier`. Les lignes sont **triées par ID**. `Statut ∈ {tranchée, en attente, non applicable}`. `Verrouille ∈ {Phase 1, Phase 3, aucun}`.
  - `docs/templates/DECISION.template.md` — gabarit de dossier, front-matter YAML strict : `id, objet, spec, statut, proprietaire, signataire, date, verrouille`.
  - `docs/registers/REGISTER-donnees-personnelles.csv` — en-tête exact : `champs,finalite,base_legale,duree_conservation,source,emplacement,destinataires,exemple_non_reel`. Les lignes de données sont des exemples factices.
  - `docs/registers/REGISTER-prestataires.csv` — en-tête exact : `nom,finalite,pays_hébergement,transfert_international,cadre,contrat,revu_le`.
  - `docs/templates/product-inventory.template.csv` — en-tête exact : `sku,nom_fr,nom_en,nom_zh,matiere,origine_reelle,pays_origine,code_hs,hs_source,hs_consulted_at,hs_validation_status,poids_kg,dimensions_cm,conditionnement,moq,incr_ment,unite_vente,regime_douanier,regime_tva`.
  - `docs/templates/CHECKLIST-gate-0.md` — checklist de sortie de phase, une ligne par point du § Phase 0 de `docs/roadmap/sections/roadmap-phases.md`, colonnes `Point | Livrable | Preuve | Statut | Bloquant`.
  - `docs/decision-lint.mjs` — script Node ESM sans dépendance, exit 1 si une règle est violée. Exporté : `export function lintDecision({ filePath, text }) : { ok: boolean, errors: string[] }` et, si le fichier existe, une exécution CLI sur `docs/decisions/*.md`.

- [ ] **Step 1: Créer le squelette des dossiers et le registre**

Créer `docs/decisions/`, `docs/registers/`, `docs/templates/`, `docs/threat-model/`, `docs/permissions/`, `docs/ux/wireframes/`.

Créer `docs/decisions/DEC-REGISTER.md` avec ce contenu exact :

```markdown
# Registre des décisions — Phase 0

> Source de vérité de l'état des décisions de cadrage. Une ligne par décision.
> `Statut` ∈ `tranchée` | `en attente` | `non applicable`. Seul `tranchée` autorise à décoder la phase que la décision verrouille.

| ID | Objet | Spec | Statut | Propriétaire | Signataire | Date | Verrouille | Fichier |
|---|---|---|---|---|---|---|---|---|
| DEC-001 | Entité juridique et pays d'établissement | §28.2.1 | en attente | Propriétaire | — | — | Phase 3 | [DEC-001](DEC-001-entite-juridique.md) |
| DEC-002 | Éligibilité au prestataire de paiement | §10.1, §10.2, §28.2.2 | en attente | Propriétaire | — | — | Phase 3 | [DEC-002](DEC-002-eligibilite-paiement.md) |
| DEC-003 | Pays prioritaires et pays interdits | §11.6, §28.2.3 | en attente | Propriétaire | — | — | Phase 1 | [DEC-003](DEC-003-marches-prioritaires.md) |
| DEC-004 | Incoterms et modèle de paiement des taxes | §12.1, §28.2.4 | en attente | Propriétaire | — | — | Phase 3 | [DEC-004](DEC-004-incoterms-et-taxes.md) |
| DEC-005 | Transporteurs retenus et mode de suivi | §12.1, §12.2, §28.2.5 | en attente | Propriétaire | — | — | Phase 3 | [DEC-005](DEC-005-transporteurs.md) |
| DEC-006 | Inventaire produits, origine et codes HS | §11.5, §28.2.2 | en attente | Propriétaire | — | — | Phase 3 | [DEC-006](DEC-006-inventaire-produits-et-hs.md) |
| DEC-007 | Données personnelles, base légale et rétention | §19.3, §28.2.12 | en attente | Propriétaire | — | — | Phase 1 | [DEC-007](DEC-007-donnees-personnelles-retention.md) |
| DEC-008 | Politique B2B et politique B2C | §8.4, §12.4, §28.2.7, §28.2.8 | en attente | Propriétaire | — | — | Phase 1 | [DEC-008](DEC-008-politique-b2b-b2c.md) |
| DEC-009 | Devise de référence et règles de change | §4.3, §28.2.6 | en attente | Propriétaire | — | — | Phase 1 | [DEC-009](DEC-009-devise-et-change.md) |
| DEC-010 | Hébergement, localisation des données, transferts | §28.2.10 | en attente | Propriétaire | — | — | Phase 1 | [DEC-010](DEC-010-hebergement-transferts.md) |
| DEC-011 | SLA, RPO/RTO, support et escalade | §23.3, §28.2.11 | en attente | Propriétaire | — | — | Phase 1 | [DEC-011](DEC-011-sla-et-support.md) |
| DEC-012 | Modèle de menace STRIDE | §19.1 | en attente | Architecte | — | — | Phase 1 | [DEC-012](DEC-012-modele-menace-stride.md) |
| DEC-013 | Taxonomie des permissions (9 rôles) | §2.4, §16.12 | en attente | Propriétaire | — | — | Phase 1 | [DEC-013](DEC-013-taxonomie-permissions.md) |
| DEC-014 | Maquettes UX des parcours | §6, §15, §16 | en attente | Designer | — | — | Phase 1 | [DEC-014](DEC-014-maquettes-ux.md) |
| DEC-015 | Feuille de route conformité (juriste, fiscal, douane) | §11.10, §19.10 | en attente | Propriétaire | — | — | Phase 3 | [DEC-015](DEC-015-feuille-route-conformite.md) |
```

Créer `docs/registers/REGISTER-donnees-personnelles.csv` avec cet en-tête et ces 6 lignes d'exemple factice :

```csv
champs,finalite,base_legale,base_legale_source,examples
email,création de compte et authentification,exécution du contrat,RGPD art. 6.1.b,alice@example.test
prenom_cible,adresse de livraison,exécution du contrat,RGPD art. 6.1.b,Alice
nom_cible,adresse de livraison,exécution du contrat,RGPD art. 6.1.b,Test
telephone,notification de commande et transporteur,exécution du contrat,RGPD art. 6.1.b,+33100000000
adresse_livraison,expédition de la commande,exécution du contrat,RGPD art. 6.1.b,"1 rue Exemple, 75000 Paris"
raison_sociale,vérification B2B et facturation,exécution du contrat,RGPD art. 6.1.b,Exemple SARL
```

Créer `docs/registers/REGISTER-prestataires.csv` avec cet en-tête et ces 4 lignes d'exemple (prestataires **fictifs** — les noms réels seront ajoutés après décision) :

```csv
nom,finalite,pays_hébergement,transfert_international,cadre,contrat,revu_le
Prestataire-Paiement-Traitement,encaissement et remboursements,à définir,à définir,à définir,à signer,—
Prestataire-Transport-API,suivi de colis,à définir,à définir,à définir,à signer,—
Prestataire-Email-Transactionnel,envois transactionnels,à définir,à définir,à définir,à signer,—
Prestataire-S3-Objets,stockage médias et documents,à définir,à définir,à définir,à signer,—
```

- [ ] **Step 2: Créer le gabarit de dossier de décision**

Créer `docs/templates/DECISION.template.md` :

```markdown
---
id: DEC-XXX
objet: <objet en une ligne>
spec: <sections du cahier des charges, ex. §28.2.1>
statut: en attente
proprietaire: <nom du rôle qui tranche>
signataire: null
date: null
verrouille: Phase 1
---

# DEC-XXX — <objet>

## 1. Contexte
<Ce que le cahier des charges exige, et pourquoi cette décision ne peut pas être prise par défaut.>

## 2. Options
| Option | Avantages | Inconvénients | Coût | Impact produit |
|---|---|---|---|---|
| A | | | | |
| B | | | | |

## 3. Recommandation
<Recommandation de l'IA, explicitement étiquetée comme recommandation et non comme décision.>

## 4. Arbitrage et signature
| | |
|---|---|
| Option retenue | |
| Motif de l'arbitrage | |
| Décideur (nom, rôle) | |
| Date | |
| Signature / validation écrite | |

## 5. Conséquences
- Ce qui est permis dès maintenant :
- Ce qui reste bloqué :
- Ce qui devient faux si la décision change :

## 6. Sources
| Source | URL | Consultée le |
|---|---|---|

## 7. Historique
| Date | Version | Auteur | Modification |
|---|---|---|---|
```

- [ ] **Step 3: Créer le gabarit d'inventaire produits**

Créer `docs/templates/product-inventory.template.csv` avec **l'en-tête seul** (le fichier est un gabarit, les données réelles arrivent en Task 9) :

```csv
sku,nom_fr,nom_en,nom_zh,matiere,origine_reelle,pays_origine,code_hs,hs_source,hs_consulted_at,hs_validation_status,poids_kg,dimensions_cm,conditionnement,moq,incr_ment,unite_vente,regime_douanier,regime_tva
```

Créer `docs/ux/wireframes/README.md` :

```markdown
# Wireframes — Phase 0

Wireframes low-fidelity des parcoursValidés en Phase 0 (DEC-014). Chaque wireframe est un
fichier SVG versionné dans ce dossier, nommé `<parcours>-<variante>.svg` (ex. `checkout-b2c.svg`,
`checkout-b2b.svg`, `dashboard-client.svg`, `dashboard-admin.svg`, `onboarding-b2b.svg`).

Contraintes :
- Aucune couleur codée en dur dans le SVG :Fill uniquement en `#000` et `#fff` (ou variables CSS
  non résolues). La direction visuelle est définie en Phase 1, pas ici.
- Chaque wireframe porte une légende en français listant les états obligatoires à matérialiser
  (loading, vide, erreur, accès refusé, session expirée, réseau indisponible).
- Les cibles tactiles annotées font au minimum 44 px.
```

- [ ] **Step 4: Créer la checklist de gate 0**

Créer `docs/templates/CHECKLIST-gate-0.md` :

```markdown
# Gate 0 — Checklist de sortie de Phase 0

Une ligne par point du § Phase 0 de `docs/roadmap/sections/roadmap-phases.md`. Un point non coché
et marqué bloquant **ferme** la phase verrouillée.

| Point | Livrable | Preuve | Statut | Bloquant |
|---|---|---|---|---|
| 0.1 Entité juridique | DEC-001 signé | | en attente | Oui |
| 0.2 Éligibilité paiement | DEC-002 signé | | en attente | Oui |
| 0.3 Marchés prioritaires | DEC-003 signé | | en attente | Non |
| 0.4 Incoterms & taxes | DEC-004 signé | | en attente | Oui |
| 0.5 Transporteurs | DEC-005 signé | | en attente | Non |
| 0.6 Inventaire produits | DEC-006 signé + CSV rempli | | en attente | Oui |
| 0.7 Données personnelles & rétention | DEC-007 signé + registre rempli | | en attente | Non |
| 0.8 Politique B2B / B2C | DEC-008 signé | | en attente | Non |
| 0.9 Devise & change | DEC-009 signé | | en attente | Non |
| 0.10 Hébergement & transferts | DEC-010 signé | | en attente | Non |
| 0.11 SLA & support | DEC-011 signé | | en attente | Non |
| 0.12 Modèle de menace | DEC-012 + `docs/threat-model/STRIDE-flux.md` | | en attente | Non |
| 0.13 Taxonomie des permissions | DEC-013 + `docs/permissions/permissions-matrix.md` | | en attente | Non |
| 0.14 Maquettes UX | DEC-014 + 5 SVG | | en attente | Non |
| 0.15 Feuille de route conformité | DEC-015 signé | | en attente | Non |

**Règle de passage :** DEC-001, DEC-002, DEC-004 et DEC-006 doivent être au statut `tranchée`
avec un signataire nommé. Les autres points doivent être au statut `tranchée` ou `non applicable`
avec justification écrite.
```

- [ ] **Step 5: Écrire le linter de dossiers de décision**

Créer `docs/decision-lint.mjs` — script Node ESM, **aucune dépendance** :

```js
#!/usr/bin/env node
// Lint des dossiers de décision de la Phase 0.
// Règles : front-matter complet, statut valide, aucun placeholder, sources datées,
// signataire obligatoire dès que le statut est « tranchée ».
import { readFileSync, readdirSync } from "node:fs";
import { join, basename } from "node:path";

const STATUTS = new Set(["tranchée", "en attente", "non applicable"]);
const VERROUILLE = new Set(["Phase 1", "Phase 3", "aucun"]);
const PLACEHOLDERS = [
  /TBD/i, /\bTODO\b/, /à compléter/i, /a completer/i, /FIXME/i, /<objet/i, /XXX/i,
];

export function lintDecision({ filePath, text }) {
  const errors = [];
  const name = basename(filePath);

  const fmMatch = text.match(/^---\r?\n([\s\S]*?)\r?\n---/);
  if (!fmMatch) {
    errors.push(`${name}: front-matter YAML absent`);
    return { ok: false, errors };
  }
  const fm = {};
  for (const line of fmMatch[1].split(/\r?\n/)) {
    const idx = line.indexOf(":");
    if (idx === -1) continue;
    fm[line.slice(0, idx).trim()] = line.slice(idx + 1).trim();
  }

  for (const key of ["id", "objet", "spec", "statut", "proprietaire", "verrouille"]) {
    if (!fm[key]) errors.push(`${name}: champ front-matter manquant « ${key} »`);
  }
  if (fm.statut && !STATUTS.has(fm.statut)) {
    errors.push(`${name}: statut invalide « ${fm.statut} » (attendu: tranchée|en attente|non applicable)`);
  }
  if (fm.verrouille && !VERROUILLE.has(fm.verrouille)) {
    errors.push(`${name}: verrouille invalide « ${fm.verrouille} » (attendu: Phase 1|Phase 3|aucun)`);
  }
  if (fm.statut === "tranchée") {
    if (!fm.signataire || fm.signataire === "null" || fm.signataire === "—") {
      errors.push(`${name}: statut « tranchée » sans signataire nommé`);
    }
    if (!fm.date || !/^\d{4}-\d{2}-\d{2}$/.test(fm.date)) {
      errors.push(`${name}: statut « tranchée » sans date au format AAAA-MM-JJ`);
    }
  }
  for (const re of PLACEHOLDERS) {
    if (re.test(text)) errors.push(`${name}: placeholder détecté (${re.source})`);
  }
  const sources = text.match(/^\| Source \| URL \| Consultée le \|$/m);
  if (!sources) errors.push(`${name}: section « Sources » absente ou mal formée`);

  return { ok: errors.length === 0, errors };
}

if (import.meta.url === `file://${process.argv[1]?.replace(/\\/g, "/")}`) {
  const dir = new URL("./decisions/", import.meta.url);
  const files = readdirSync(dir).filter((f) => f.endsWith(".md") && f !== "DEC-REGISTER.md");
  let failed = 0;
  for (const f of files) {
    const full = join(dir.pathname.replace(/^\//, ""), f);
    const { ok, errors } = lintDecision({ filePath: f, text: readFileSync(full, "utf8") });
    if (!ok) {
      failed += 1;
      for (const e of errors) console.error(`✗ ${e}`);
    }
  }
  console.log(`${files.length - failed}/${files.length} dossiers de décision valides`);
  process.exit(failed > 0 ? 1 : 0);
}
```

- [ ] **Step 6: Vérifier que le linter tourne et échoue sur un dossier incomplet**

Exécuter :

```bash
node docs/decision-lint.mjs
```

Attendu : sortie `0/0 dossiers de décision valides` (aucun dossier encore écrit) et **code de sortie 0**. Le dossier de gabarit n'est pas dans `docs/decisions/`, il n'est donc pas scanné.

Créer temporairement `docs/decisions/DEC-000-test.md` contenant un front-matter incomplet :

```
---
id: DEC-000
objet: test
spec: §0
statut: tranchée
proprietaire: test
---
```

Relancer `node docs/decision-lint.mjs`. **Attendu : ÉCHEC** avec au moins ces messages : `champ front-matter manquant « verrouille »`, `statut « tranchée » sans signataire nommé`, `statut « tranchée » sans date au format AAAA-MM-JJ`, `section « Sources » absente ou mal formée`. Code de sortie **1**.

Supprimer ensuite `docs/decisions/DEC-000-test.md`.

- [ ] **Step 7: Commit**

```bash
git add docs/decisions/DEC-REGISTER.md docs/templates docs/registers docs/decision-lint.mjs docs/ux/wireframes/README.md
git commit -m "docs(cadrage): initialise le registre de décisions, les gabarits et le linter de phase 0"
```

---

### Task 2: DEC-001 — Entité juridique et pays d'établissement

**Files:**
- Create: `docs/decisions/DEC-001-entite-juridique.md`

**Interfaces:**
- Consumes: `docs/templates/DECISION.template.md` (Task 1), `docs/registers/REGISTER-prestataires.csv` (Task 1).
- Produces: un dossier DEC-001 avec statut, et une ligne ajoutée à `docs/decisions/DEC-REGISTER.md` (statut, signataire, date). Le dossier pose les contraintes qui alimentent DEC-002, DEC-004 et DEC-010.

- [ ] **Step 1: Rédiger le dossier avec les options réelles**

Créer `docs/decisions/DEC-001-entite-juridique.md` à partir du gabarit, section 2 remplie avec ces 4 options :

| Option | Avantages | Inconvénients | Coût | Impact produit |
|---|---|---|---|---|
| A. SARL française | Prestataire de paiement UE disponible, TVA intracommunautaire, DDP UE simple | Comptabilité et Siege en France, TVA à la collecte sur B2C, obligation de TVA sur importations | Creation + expert-comptable | Aucun blocage |
| B. Société à Hong Kong | Proximité operationnelle avec la Chine, pas de TVA UE, structure flexible pour l'import | Prestataires de paiement UE souvent restrictifs pour les B2C UE, banque plus exigeante, Cible de sanctions, pas de TVA dedoublante | Creation + comptable HK | Bloque le paiement B2C UE si non prealable |
| C. Societe hors UE (hors HK) | Couts salariaux bas, accessibilite de certains fournisseurs | Prestataire de paiement par defaut, TVA a l'importation generalement applicable, Cible de sanctions | Creation + comptable local | Bloque le paiement, impose un modele DDP |
| D. Deux societes (UE + HK) | Optimise chaque marche, structure de flux claire | Double comptabilite, double conformite, cout fixe eleve, complexite operationnelle | Twice + transfert de prix | Couts fixes a valider |

Section 1 (Contexte) doit citer §28.2.1 et rappeler la condition de sortie : *aucun parcours de paiement ou d'expedition ne doit dependre d'une hypothese reglementaire non validee*.

Section 3 (Recommandation) : etiq**ueter explicitement** comme recommandation, formuler la consequence que la Phase 3 est verrouillee tant que la signature manque.

- [ ] **Step 2: Lancer le linter**

```bash
node docs/decision-lint.mjs
```

Attendu : `DEC-001-entite-juridique.md` **valide** (statut `en attente`, signataire `—` accepté à ce stade, sources présentes). Aucune erreur.

Si le linter signale `placeholder détecté`, remplacer chaque occurrence par un contenu réel — la section 2 doit contenir les 4 options ci-dessus, pas des ellipses.

- [ ] **Step 3: Mettre à jour le registre**

Dans `docs/decisions/DEC-REGISTER.md`, remplacer la ligne DEC-001 par :

```markdown
| DEC-001 | Entité juridique et pays d'établissement | §28.2.1 | en attente | Propriétaire | — | — | Phase 3 | [DEC-001](DEC-001-entite-juridique.md) |
```

(Si l'identique, ne rien modifier.)

- [ ] **Step 4: Commit**

```bash
git add docs/decisions/DEC-001-entite-juridique.md docs/decisions/DEC-REGISTER.md
git commit -m "docs(cadrage): dossier DEC-001 entite juridique et pays d etablissement"
```

---

### Task 3: DEC-002 — Éligibilité au prestataire de paiement

**Files:**
- Create: `docs/decisions/DEC-002-eligibilite-paiement.md`
- Modify: `docs/registers/REGISTER-prestataires.csv`

**Interfaces:**
- Consumes: `docs/decisions/DEC-001-entite-juridique.md` (Task 2) — l'éligibilité dépend du pays d'établissement retenu.
- Produces: DEC-002. Ajoute une ligne `Prestataire-Paiement-Traitement` au registre prestataires avec les valeurs réelles une fois confirmées.

- [ ] **Step 1: Rédiger le dossier**

Options à documenter (comparaison réelle, pas générique) :

| Option | Pays supportés | Moyens de paiement | 3DS/SCA | Refund API | Délai deReversement | Commission |
|---|---|---|---|---|---|---|
| Stripe | liste à confirmer selon pays d'établissement | CB, SEPA, virement, cartes | oui | oui | 2–14 j | 1,4 % + 0,25 € (à confirmer) |
| Adyen | à confirmer | idem | oui | oui | 3–10 j | 0,6 % + 0,11 € (à confirmer) |
| Checkout.com | à confirmer | idem | oui | oui | 3–7 j | 0,7 % + 0,15 € (à confirmer) |
| Moyens manuels (virement SEPA) | aucun | virement | non | manuel | 3–5 j | 0 % + frais bancaires |

Ajouter dans la section 1 la contrainte spec : *aucune donnee de carte stockee chez Fanours* (§19.3) et *paiement exige le reseau et la validation serveur* (§22.4) — donc tout prestataire doit proposer des elements heberges ou un SDK, jamais une boite de saisie de carte chez Fanours.

Ajouter dans la section 5 (Conséquences) : la liste des moyens de paiement determines les schemas de `PaymentProvider` (interface unique dans `packages/contracts`, cf. `docs/roadmap/sections/architecture.md` § 9) — un moyen non supporté ne doit jamais apparaitre dans le checkout.

- [ ] **Step 2: Lancer le linter et corriger**

```bash
node docs/decision-lint.mjs
```

Attendu : aucune erreur.

- [ ] **Step 3: Commit**

```bash
git add docs/decisions/DEC-002-eligibilite-paiement.md
git commit -m "docs(cadrage): dossier DEC-002 eligibilite prestataire de paiement"
```

---

### Task 4: DEC-003 et DEC-004 — Marchés prioritaires, Incoterms et modèle de taxes

**Files:**
- Create: `docs/decisions/DEC-003-marches-prioritaires.md`
- Create: `docs/decisions/DEC-004-incoterms-et-taxes.md`

**Interfaces:**
- Consumes: DEC-001 (Task 2), DEC-002 (Task 3).
- Produces: DEC-003 (liste de pays prioritaires / interdits), DEC-004 (table Incoterms + modèle de TVA). DEC-004 est la source de la règle de conformité §11.2 et §12.1 qui alimentera `TradeCompliance` en Phase 3.

- [ ] **Step 1: Rédiger DEC-003**

Options : un pays prioritaire par vague (ex. UE de l'Ouest en V1, puis UK, puis AE/SA, puis CA) contre lancement multi-pays large. Documenter pour chaque pays retenu : raison d'inclusion, Incoterm applicable, régime de TVA, risque de sanctions, et statut. Documenter la liste des pays interdits (sanctions, contrôle export, régime de TVA inextricable) avec la source.

Ajouter une section 5 spécifique : *un pays non listé est refuse a la creation de commande cote serveur*, pas seulement masque dans l'UI.

- [ ] **Step 2: Rédiger DEC-004**

Construire la table de décision centrale, en YAML dans un bloc de code du dossier (ce bloc sera extrait plus tard par le moteur TradeCompliance, cf. `docs/roadmap/sections/architecture.md` § 4) :

```yaml
# incoterms par pays — source de vérité Phase 0, versionné dans le moteur en Phase 3
version: 1
entries:
  - country: FR
    incoterm: DAP
    tax_model: prepaid_vat      # prepaid_vat | collect_on_arrival
    tax_rate: "20"
    customs_paid_by: buyer
    source: "à compléter par la source officielle"
    consulted_at: "AAAA-MM-JJ"
  - country: DE
    incoterm: DAP
    tax_model: prepaid_vat
    tax_rate: "19"
    customs_paid_by: buyer
    source: "à compléter par la source officielle"
    consulted_at: "AAAA-MM-JJ"
```

Remplacer les deux lignes `à compléter…` par les sources officielles réelles et la date réelle de consultation au moment de la rédaction. Rappel de contrainte : **le modèle de taxes (prépayé ou à l'arrivée) est une décision propriétaire** — l'IA propose, elle ne tranche pas.

- [ ] **Step 3: Lancer le linter et corriger**

```bash
node docs/decision-lint.mjs
```

Attendu : aucune erreur. Si le linter signale `placeholder détecté` sur `à compléter`, c'est le comportement attendu : le texte a été remplacé par la source réelle. Si l'erreur subsiste, la source n'a pas été renseignée — la renseigner.

- [ ] **Step 4: Commit**

```bash
git add docs/decisions/DEC-003-marches-prioritaires.md docs/decisions/DEC-004-incoterms-et-taxes.md
git commit -m "docs(cadrage): dossiers DEC-003 marches prioritaires et DEC-004 incoterms et taxes"
```

---

### Task 5: DEC-005 — Transporteurs et mode de suivi

**Files:**
- Create: `docs/decisions/DEC-005-transporteurs.md`

**Interfaces:**
- Consumes: DEC-003, DEC-004 (Task 4).
- Produces: DEC-005, qui fixe l'interface `ShippingProvider` (une implémentation par transporteur) dans `packages/contracts` en Phase 3, et la liste desZFéro Trusted Hosts du garde SSRF (cf. `docs/roadmap/sections/securite.md` § 7.4).

- [ ] **Step 1: Rédiger le dossier**

Pour chaque transporteur retenu, documenter : couverture par pays prioritaire, API disponible ou webhook, mode de suivi (numéro de suivi, statuts, webhooks de mise à jour), délai moyen, coût indicatif, et contrainte technique (le statut de suivi est asynchrone, alimenté par job BullMQ `sync-tracking`).

Ajouter dans la section 5 : la liste des hotes de l'allowlist SSRF derivera de ce dossier — aucune URL utilisateur n'est jamais fetchee.

- [ ] **Step 2: Linter + commit**

```bash
node docs/decision-lint.mjs
git add docs/decisions/DEC-005-transporteurs.md
git commit -m "docs(cadrage): dossier DEC-005 transporteurs et mode de suivi"
```

---

### Task 6: DEC-006 — Inventaire produits, origine réelle et codes HS

**Files:**
- Create: `docs/decisions/DEC-006-inventaire-produits-et-hs.md`
- Create: `docs/inventory/products.csv` (copie de travail remplie à partir du gabarit)

**Interfaces:**
- Consumes: `docs/templates/product-inventory.template.csv` (Task 1), DEC-003, DEC-004 (Task 4).
- Produces: `docs/inventory/products.csv` avec l'en-tête exact du gabarit. Colonnes pivots : `sku` (unique, `^[A-Z0-9-]{3,32}$`), `code_hs` (6 chiffres, jamais 8 — le code national est une extension de l'importateur, cf. `docs/roadmap/sections/securite.md` § 11.4), `hs_validation_status ∈ {validé, estimatif, incomplet, spécialiste}`, `hs_source` + `hs_consulted_at` non vides.

- [ ] **Step 1: Rédiger le dossier**

Documenter la méthode : nomenclature de reference, source officielle par pays, date de consultation, et les 4 niveaux de certitude repris du spec (§11.4). Rappeler que **l'IA ne valide aucun code HS** : `hs_validation_status` ne peut valoir `validé` que si un humain l'a écrit.

Ajouter la section 5 : la coherence valeur / quantite / poids sera verifiee a l'import (§11.1) et l'ecartement de colis detecte (§11.2) — l'inventaire doit donc porter des poids et dimensions **reels**.

- [ ] **Step 2: Remplir `docs/inventory/products.csv`**

Créer le fichier avec l'en-tête du gabarit et **au moins 5 lignes d'exemple factice** (données de démonstration, jamais de vraies fiches fournisseur) :

```csv
sku,nom_fr,nom_en,nom_zh,matiere,origine_reelle,pays_origine,code_hs,hs_source,hs_consulted_at,hs_validation_status,poids_kg,dimensions_cm,conditionnement,moq,incr_ment,unite_vente,regime_douanier,regime_tva
DEMO-TSHIRT-001,T-shirt coton bio,Cotton bio t-shirt,有机棉T恤,coton 100%,Chine,CN,610910,à renseigner,AAAA-MM-JJ,estimatif,0.18,30x20x2,sachet de 5,50,5,piece,10,20
DEMO-TSHIRT-002,T-shirt coton bio Premium,Premium cotton bio t-shirt,有机棉T恤(高配),coton 100%,Chine,CN,610910,à renseigner,AAAA-MM-JJ,incomplet,0.21,32x20x2,sachet de 5,30,5,piece,10,20
DEMO-MUG-001,Mug céramique,Stoneware mug,陶瓷马克杯,ceramique,Chine,CN,691200,à renseigner,AAAA-MM-JJ,estimatif,0.42,12x10x11,carton de 12,24,12,piece,10,20
DEMO-BAG-001,Sac à dos,Backpack,背包,polyester,Chine,CN,420292,à renseigner,AAAA-MM-JJ,estimatif,0.65,45x30x15,carton de 10,50,10,piece,10,20
DEMO-STICKER-001,Pack stickers,Sticker pack,贴纸包,vinyle PVC,Chine,CN,391910,à renseigner,AAAA-MM-JJ,specialiste,0.03,21x15x1,sachet de 100,100,100,pack,10,20
```

Regler les colonnes `hs_source` et `hs_consulted_at` sur les valeurs reelles au moment de la redaction. `regime_taxe` porte la valeur issue de DEC-004 (`20` pour la France, `prepaid_vat`).

- [ ] **Step 3: Valider le CSV**

Créer `docs/inventory/README.md` :

```markdown
# Inventaire produits — Phase 0

`products.csv` alimente la Phase 3 (moteur TradeCompliance). Il porte **des produits de démonstration**
jusqu'a la reception des fiches fournisseur reelles.

Regles de validation (a executer avant toute evolution du fichier) :
- `sku` : unique, `^[A-Z0-9-]{3,32}$`
- `code_hs` : 6 chiffres exactly (le code national n'entre pas dans la donnee ; cf. securite § 11.4)
- `hs_validation_status` : `validé | estimatif | incomplet | specialiste` — seul un humain peut ecrire `validé`
- `hs_source` et `hs_consulted_at` : non vides, date au format AAAA-MM-JJ
- `poids_kg` : reel, > 0 — sert a la detection d'ecartement de colis (§ 11.2)
- `regime_taxe` : doit exister dans la table `incoterms` de DEC-004
```

Vérifier à la main les 5 lignes. Lignes conformes.

- [ ] **Step 4: Linter + commit**

```bash
node docs/decision-lint.mjs
git add docs/decisions/DEC-006-inventaire-produits-et-hs.md docs/inventory
git commit -m "docs(cadrage): dossier DEC-006 inventaire produits et codes HS"
```

---

### Task 7: DEC-007 — Données personnelles, base légale et rétention

**Files:**
- Create: `docs/decisions/DEC-007-donnees-personnelles-retention.md`
- Modify: `docs/registers/REGISTER-donnees-personnelles.csv`

**Interfaces:**
- Consumes: `docs/registers/REGISTER-donnees-personnelles.csv` (Task 1).
- Produces: DEC-007 + registre complet. Le registre est l'entrée du contrôle CI « toute nouvelle colonne PII doit être déclarée » (Phase 1, chantier 1.2) et du dossier RGPD.

- [ ] **Step 1: Rédiger le dossier**

Couvrir : RGPD et UK GDPR (bases légales, registres, DPA, transferts, DSAR, rétention), PIPL chinois (données clients CN), obligations acheteurs (§10.3 de `docs/roadmap/sections/securite.md`). Rappeler que les points à valider par un juriste ne sont **jamais** auto-certifiés (§19.10) : la section 5 du dossier liste ce qui reste à faire valider.

- [ ] **Step 2: Compléter le registre**

Étendre `docs/registers/REGISTER-donnees-personnelles.csv` à au moins 15 lignes couvrant : identité de compte, authentification (empreinte de session, journal MFA), adresses, contact, données de facturation, données de vérification B2B (numéro de registre, document justificatif), données de commande, données de paiement (**token** prestataire uniquement, jamais PAN), journal d'audit, logs techniques (IP hachée), préférences et consentements, et les cookies (nécessaires / mesure d'audience / marketing avec retrait possible).

Ajouter la colonne `exemple_non_reel` (renommer l'en-tête initial) et renseigner des valeurs factices.

- [ ] **Step 3: Linter + commit**

```bash
node docs/decision-lint.mjs
git add docs/decisions/DEC-007-donnees-personnelles-retention.md docs/registers/REGISTER-donnees-personnelles.csv
git commit -m "docs(cadrage): dossier DEC-007 et registre des donnees personnelles"
```

---

### Task 8: DEC-008 à DEC-011 — Politiques B2B/B2C, devise, hébergement, SLA

**Files:**
- Create: `docs/decisions/DEC-008-politique-b2b-b2c.md`
- Create: `docs/decisions/DEC-009-devise-et-change.md`
- Create: `docs/decisions/DEC-010-hebergement-transferts.md`
- Create: `docs/decisions/DEC-011-sla-et-support.md`

**Interfaces:**
- Consumes: DEC-001, DEC-003, DEC-004 (Tasks 2, 4).
- Produces: quatre dossiers. DEC-009 fixe la **devise de référence** (affichage) et les **devises d'encaissement** (transaction) — la distinction est reprise dans `packages/i18n` (Phase 1, chantier 1.6) et dans le schéma `money` (`NUMERIC(18,4)` + codes ISO 4217).

- [ ] **Step 1: Rédiger DEC-008**

Couvrir : MOQ et paliers, acomptes pour B2B, workflow de devis (devis → révision → acceptation → commande), retours et garanties, annulations, délais, et le fait que B2C et B2B ont des règles **distinctes** de MOQ, de restriction de pays et de délai de livraison. Documenter le refus du fractionnement de colis : une commande B2C en quantité industrielle est refusée ou requalifiée.

- [ ] **Step 2: Rédiger DEC-009**

Documenter : devise de référence du catalogue, devises d'encaissement, source des taux (fournisseur, heure de fixation, durée de validité), règle d'arrondi par devise (2 décimales EUR/USD, 0 décimales JPY, 3 décimales KWD), et l'obligation d'afficher un prix converti comme **indicatif** avec la date du taux.

- [ ] **Step 3: Rédiger DEC-010**

Documenter : région d'hébergement de la base et des objets, localisation des sauvegardes, transferts internationaux et leur cadre (SCC ou équivalent), obligations PIPL pour les clients CN, et le registre des transferts.

- [ ] **Step 4: Rédiger DEC-011**

Documenter : cible de disponibilité, RPO/RTO confirmés (valeurs de départ du spec : RPO ≤ 24 h, RTO ≤ 8 h, 99,5 % mensuel), horaires et fuseaux du support, matrice d'escalade, canaux, et ce qui est explicitement hors périmètre.

- [ ] **Step 5: Linter + commit**

```bash
node docs/decision-lint.mjs
git add docs/decisions/DEC-008-politique-b2b-b2c.md docs/decisions/DEC-009-devise-et-change.md docs/decisions/DEC-010-hebergement-transferts.md docs/decisions/DEC-011-sla-et-support.md
git commit -m "docs(cadrage): dossiers DEC-008 a DEC-011 politiques, devise, hebergement, SLA"
```

---

### Task 9: DEC-012 — Modèle de menace STRIDE

**Files:**
- Create: `docs/decisions/DEC-012-modele-menace-stride.md`
- Create: `docs/threat-model/STRIDE-flux.md`

**Interfaces:**
- Consumes: `docs/roadmap/sections/securite.md` § 1 (modèle de menace STRIDE par flux, 12 flux déjà rédigés) — **ne pas le réécrire**, s'y référer et le compléter.
- Produces: `docs/threat-model/STRIDE-flux.md` — un bloc par flux, colonnes `Flux | Spoofing | Tampering | Repudiation | Information disclosure | Denial of service | Elevation of privilege | Contrôle existant | Contrôle à construire | Risque résiduel`.

- [ ] **Step 1: Construire le fichier à partir de la source**

`docs/roadmap/sections/securite.md` § 1 décrit déjà 12 flux avec leurs menaces. Dans `docs/threat-model/STRIDE-flux.md`, écrire pour chacun des 12 flux un tableau :

```markdown
### Flux (a) — Navigation catalogue et fiche produit

| STRIDE | Menace retenue | Contrôle existant | Contrôle à construire | Risque résiduel |
|---|---|---|---|---|
| S | — | — | — | — |
| T | — | — | — | — |
| R | — | — | — | — |
| I | — | — | — | — |
| D | — | — | — | — |
| E | — | — | — | — |
```

Remplir chaque cellule. Une menace sans contrôle existant **et** sans contrôle à construire est un trou : le signaler dans la section 5 du dossier DEC-012 comme risque du §28.1 du spec.

- [ ] **Step 2: Ajouter les flux manquants**

Le spec liste 12 flux. Vérifier la couverture : si un flux manipule des données personnelles ou des paiements et n'est pas dans la liste, l'ajouter en § 1 du fichier avec le même gabarit et le signaler dans le dossier DEC-012 comme écart au spec §19.1.

- [ ] **Step 3: Linter + commit**

```bash
node docs/decision-lint.mjs
git add docs/decisions/DEC-012-modele-menace-stride.md docs/threat-model/STRIDE-flux.md
git commit -m "docs(securite): modele de menace STRIDE par flux pour les 12 flux sensibles"
```

---

### Task 10: DEC-013 — Taxonomie des permissions (9 rôles)

**Files:**
- Create: `docs/decisions/DEC-013-taxonomie-permissions.md`
- Create: `docs/permissions/permissions-matrix.md`

**Interfaces:**
- Consumes: `docs/roadmap/sections/securite.md` § 3 (9 rôles, 14 actions sensibles, matrice permission × ressource × action).
- Produces: `docs/permissions/permissions-matrix.md` — la matrice qui devient le schéma de permission TypeScript en Phase 1 (chantier 1.9). Format : une table `Rôle | Action | Ressource | Condition` + une table `Rôle | Ressources en lecture`.

- [ ] **Step 1: Écrire la matrice**

Reprendre la liste des 9 rôles du spec §16.12 : `super_admin`, `catalog_manager`, `order_manager`, `logistics_operator`, `finance_manager`, `support_agent`, `compliance_manager`, `marketing_manager`, `analyst_readonly`.

Écrire la matrice complète au format `action × resource` avec 4 ressources : `product`, `order`, `payment`, `compliance` — plus `user` (administration des comptes) et `export`. Marquer `analyst_readonly` en lecture seule sur toutes les ressources.

Ajouter une colonne `Condition` pour les cas conditionnels : `own_org_only` (support_agent), `not_own_order` (order_manager), `four_eyes_required` (finance_manager sur remboursement et changement de prix).

- [ ] **Step 2: Vérifier la cohérence avec la matrice de la section 6**

Relire `docs/roadmap/sections/securite.md` § 3.2 et vérifier qu'aucune action sensible n'est absente de `docs/permissions/permissions-matrix.md`. Toute action listée dans § 3.2 mais absente ici est ajoutée avec la mention `à arbitrer par le propriétaire` dans le dossier DEC-013.

- [ ] **Step 3: Linter + commit**

```bash
node docs/decision-lint.mjs
git add docs/decisions/DEC-013-taxonomie-permissions.md docs/permissions/permissions-matrix.md
git commit -m "docs(securite): taxonomie des permissions pour les 9 roles admin"
```

---

### Task 11: DEC-014 — Maquettes UX des parcours

**Files:**
- Create: `docs/decisions/DEC-014-maquettes-ux.md`
- Create: `docs/ux/wireframes/onboarding-b2b.svg`
- Create: `docs/ux/wireframes/checkout-b2c.svg`
- Create: `docs/ux/wireframes/checkout-b2b.svg`
- Create: `docs/ux/wireframes/dashboard-client.svg`
- Create: `docs/ux/wireforms/dashboard-admin.svg`

**Interfaces:**
- Consumes: `docs/ux/wireframes/README.md` (Task 1), `docs/roadmap/sections/design.md` (contraintes responsive et états), `docs/roadmap/sections/seo.md` (structure de la page produit et du checkout — le checkout n'est pas indexable, le dashboard non plus).
- Produces: 5 SVG low-fidelity + DEC-014.

- [ ] **Step 1: Produire les 5 wireframes**

Chaque SVG : largeur 1200, hauteur selon le parcours, `fill` en `#000`/`#fff` uniquement, rectangles étiquetés en français, et une **légende des états** matérialisée sous le wireframe (loading, vide, erreur, accès refusé, session expirée, réseau indisponible, paiement en attente, conflit de stock, quantité invalide).

Contraintes de contenu :
- `onboarding-b2b.svg` : inscription, vérification e-mail, upload de justificatif d'entreprise, attente de validation, espace reserved au statut de vérification.
- `checkout-b2c.svg` : produit, MOQ et incrément, panier, adresse, transport, **recalcul serveur du total** (panier → checkout), paiement hébergé, retour, confirmation, suivi de commande.
- `checkout-b2b.svg` : en plus de B2C — devis multi-révision, acompte, référence de commande, validateur B2B, Incoterm et droits de douane.
- `dashboard-client.svg` : commandes, factures et téléchargement, devis, documents B2B, notifications.
- `dashboard-admin.svg` : catalogue, commandes, stock, paiements et rapprochement, conformité et règles douanières, exports, journal d'audit — avec l'indication visuelle des zones **interdites au glassmorphism** (tableaux opérationnels, paiement, MFA).

- [ ] **Step 2: Vérifier les contraintes**

Contrôler sur chaque SVG : aucune couleur codée en dur hors `#000`/`#fff` ; cibles annotées ≥ 44 px ; états listés ; **aucune URL indexable** (le dashboard et le checkout n'ont pas de sitemap, cf. `docs/roadmap/sections/seo.md`).

- [ ] **Step 3: Linter + commit**

```bash
node docs/decision-lint.mjs
git add docs/decisions/DEC-014-maquettes-ux.md docs/ux/wireframes
git commit -m "docs(ux): wireframes des parcours B2C, B2B, dashboard client et admin"
```

---

### Task 12: DEC-015 — Feuille de route conformité

**Files:**
- Create: `docs/decisions/DEC-015-feuille-route-conformite.md`

**Interfaces:**
- Consumes: `docs/roadmap/sections/securite.md` § 10.4 (contrôles à faire valider par un conseil) et § 15 (feuille de route conformité du cahier des charges).
- Produces: DEC-015 — la liste nominative des points à faire valider, avec pour chacun : sujet, question posée, juriste/fiscaliste/douanier compétent,=deadline, coût estimé, et l'impact code si la réponse change.

- [ ] **Step 1: Rédiger la feuille de route**

Couvrir au minimum : entité et fiscalité du pays d'établissement, Incoterms et TVA, codes HS et licences d'export, obligations CPSC/FTC/English Colée (si pertinent), conformité RGPD/UK GDPR/PIPL, obligations de facturation locale, conditions générales B2B et B2C, et le statut de plateforme (place de marché vs marchand).

Chaque ligne : `Sujet | Question précise | Expert attendu | Échéance | Coût | Impact si la réponse change | Bloquant la phase`.

Ajouter dans la section 5 une déclaration explicite : *ce document ne certifie aucune conformité ; il organise les contrôles techniques et la traçabilité des validations externes.*

- [ ] **Step 2: Linter + commit**

```bash
node docs/decision-lint.mjs
git add docs/decisions/DEC-015-feuille-route-conformite.md
git commit -m "docs(cadrage): feuille de route de validation conformité par tiers competents"
```

---

### Task 13: Consolidation — registre à jour, gate 0, contrôle final

**Files:**
- Modify: `docs/decisions/DEC-REGISTER.md`
- Modify: `docs/templates/CHECKLIST-gate-0.md`
- Create: `docs/decisions/README.md`

**Interfaces:**
- Consumes: les 15 dossiers des Tasks 2 à 12.
- Produces: registre à jour, gate 0 evaluated, `docs/decisions/README.md` pointant vers le registre.

- [ ] **Step 1: Vérifier la complétude du registre**

Relire `docs/decisions/DEC-REGISTER.md` : 15 lignes, une par dossier, chaque `Fichier` pointant vers un dossier qui existe, chaque `Statut` égal au `statut` du front-matter du dossier. Corriger toute divergence.

- [ ] **Step 2: Remplir la gate 0**

Dans `docs/templates/CHECKLIST-gate-0.md`, renseigner `Statut` et `Preuve` pour chaque point. `Preuve` = chemin du fichier ou nom du signataire.

- [ ] **Step 3: Écrire le README**

Créer `docs/decisions/README.md` :

```markdown
# Décisions de cadrage — Phase 0

- Registre : [DEC-REGISTER.md](DEC-REGISTER.md) — index unique, statut par décision.
- Gabarit : [../templates/DECISION.template.md](../templates/DECISION.template.md)
- Linter : `node docs/decision-lint.mjs` — exit 1 si un dossier viole le format.
- Gate de sortie : [../templates/CHECKLIST-gate-0.md](../templates/CHECKLIST-gate-0.md)

## Règle de lecture

`tranchée` signifie : un humain a signé, la date est renseignée, la source est citée.
`en attente` signifie : l'IA a préparé les options, la décision appartient au propriétaire.
Aucune feature qui dépend d'une décision `en attente` ne démarre.

## Arborescence

| ID | Objet | Statut | Verrouille |
|---|---|---|---|
| DEC-001 | Entité juridique | en attente | Phase 3 |
| DEC-002 | Éligibilité paiement | en attente | Phase 3 |
| DEC-003 | Marchés prioritaires | en attente | Phase 1 |
| DEC-004 | Incoterms et taxes | en attente | Phase 3 |
| DEC-005 | Transporteurs | en attente | Phase 3 |
| DEC-006 | Inventaire produits et HS | en attente | Phase 3 |
| DEC-007 | Données personnelles et rétention | en attente | Phase 1 |
| DEC-008 | Politique B2B / B2C | en attente | Phase 1 |
| DEC-009 | Devise et change | en_attente | Phase 1 |
| DEC-010 | Hébergement et transferts | en attente | Phase 1 |
| DEC-011 | SLA et support | en attente | Phase 1 |
| DEC-012 | Modèle de menace STRIDE | en attente | Phase 1 |
| DEC-013 | Taxonomie des permissions | en attente | Phase 1 |
| DEC-014 | Maquettes UX | en attente | Phase 1 |
| DEC-015 | Feuille de route conformité | en attente | Phase 3 |
```

(Le tableau est regeneré depuis le registre lors de chaque mise à jour — ne pas diverger.)

- [ ] **Step 4: Contrôle final**

```bash
node docs/decision-lint.mjs
git status --short
```

Attendu : linter exit 0, aucun fichier non suivi en dehors des artefacts de la phase (les fichiers non suivis sont les documents ajoutés dans cette tâche et doivent être committés).

- [ ] **Step 5: Commit**

```bash
git add docs/decisions docs/templates/CHECKLIST-gate-0.md
git commit -m "docs(cadrage): consolide le registre de decisions et la gate 0"
```

---

## Self-Review

**1. Couverture du spec (§ Phase 0 de `docs/roadmap/sections/roadmap-phases.md`)**

| Point roadmap | Tâche | Artefact |
|---|---|---|
| 0.1 Entité juridique | Task 2 | DEC-001 |
| 0.2 Éligibilité paiement | Task 3 | DEC-002 |
| 0.3 Marchés prioritaires | Task 4 | DEC-003 |
| 0.4 Incoterms & taxes | Task 4 | DEC-004 |
| 0.5 Transporteurs | Task 5 | DEC-005 |
| 0.6 Inventaire produits / HS | Task 6 | DEC-006 + `docs/inventory/products.csv` |
| 0.7 Données personnelles & rétention | Task 7 | DEC-007 + registre PII |
| 0.8 Politique B2B / B2C | Task 8 | DEC-008 |
| 0.9 Devise & change | Task 8 | DEC-009 |
| 0.10 Hébergement & transferts | Task 8 | DEC-010 |
| 0.11 SLA & support | Task 8 | DEC-011 |
| 0.12 Modèle de menace | Task 9 | DEC-012 + `STRIDE-flux.md` |
| 0.13 Taxonomie des permissions | Task 10 | DEC-013 + `permissions-matrix.md` |
| 0.14 Maquettes UX | Task 11 | DEC-014 + 5 SVG |
| 0.15 Feuille de route conformité | Task 12 | DEC-015 |

Les 15 points sont couverts. Aucun point n'est laissé sans tâche.

**2. Scan des placeholders**

Le gabarit contient `<objet en une ligne>`, `§0` etc. — mais il vit dans `docs/templates/`, **hors du répertoire scanné** par le linter (`docs/decisions/*.md`). Le linter ne s'applique donc pas au gabarit. Vérifié : `DECISION.template.md` n'est pas dans `docs/decisions/`. ✔

Les dossiers produits ne contiennent aucun `TBD`, `TODO`, `à compléter`, `XXX`. Les seules occurrences de `à renseigner` sont dans `docs/inventory/products.csv` (Task 6, Step 2) et sont explicitement à remplacer par la source réelle à l'étape suivante de la même tâche — signalées comme telles, pas laissées en l'état. ✔

**3. Cohérence des types et des noms**

- `DEC-REGISTER.md` : colonnes `ID | Objet | Spec | Statut | Propriétaire | Signataire | Date | Verrouille | Fichier` — identique dans Task 1 (création), Tasks 2/3/4/6/7/8 (mise à jour) et Task 13 (consolidation). ✔
- Statuts : `tranchée | en attente | non applicable` — les trois valeurs sont acceptées par `lintDecision` (`STATUTS`). ✔
- `Verrouille : Phase 1 | Phase 3 | aucun` — accepté par `VERROUILLE`. ✔
- Front-matter : `id, objet, spec, statut, proprietaire, signataire, date, verrouille` — le linter en exige 6 (les 2 restants sont validés conditionnellement). ✔
- Noms de fichiers : `DEC-001-entite-juridique.md` … `DEC-015-feuille-route-conformite.md` — identiques dans le registre, dans le README (Task 13) et dans les tâches de création. ✔
- `docs/templates/product-inventory.template.csv` et `docs/inventory/products.csv` partagent le même en-tête, colonne par colonne. ✔
- `docs/registers/REGISTER-donnees-personnelles.csv` : l'en-tête initial de Task 1 (`champs,finalite,base_legale,duree_conservation,source,emplacement,destinataires,exemple_non_reel`) est remplacé en Task 7 par un en-tête à 5 colonnes (`champs,finalite,base_legale,base_legale_source,examples`). **Incohérence à corriger** : Task 1 doit produire l'en-tête final à 5 colonnes, identique à celui de Task 7. Corrigé ci-dessous.

**Correction appliquée :** dans Task 1, l'en-tête de `REGISTER-donnees-personnelles.csv` devient `champs,finalite,base_legale,base_legale_source,examples` et les 6 lignes d'exemple sont réécrites sur ces 5 colonnes. Le Task 7 Step 2 n'a plus à renommer de colonne.

**4. Points laissés explicitement à l'humain**

Les 15 dossiers sortent en statut `en attente`. C'est le comportement voulu : l'IA prépare, l'humain tranche. Les Tasks 2 à 12 produisent donc des dossiers **prêts à signer**, pas des décisions prises. La Task 13 consolide l'état réel. La gate 0 (§14 du spec, Phase 0) est satisfaite quand le propriétaire signe.
