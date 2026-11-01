## Roadmap — phases, livrables et jalons

### Principe de séquence

On livre un **vertical complet** (un parcours de bout en bout qui fonctionne réellement), pas des couches horizontales. Un socle technique n'est « fait » que s'il a traversé un parcours réel.

**Condition maîtresse de la Phase 0 (spec §27) :** *aucun parcours de paiement ou d'expédition ne doit dépendre d'une hypothèse réglementaire non validée.* Tant que l'entité, le prestataire de paiement, les Incoterms et le modèle de taxes ne sont pas confirmés par un tiers compétent, on ne code que ce qui ne dépend pas d'eux — et toute décision prise est **tranchée par défaut** avec un point de sortie explicite.

**Deux accélérateurs d'ordre** : le paiement et la douane. Le catalogue, l'i18n, le design system et la CI ne dépendent d'aucune décision Business et peuvent avancer en parallèle dès la Phase 1.

---

### Phase 0 — Cadrage et conformité

**Objectif :** transformer les hypothèses du cahier des charges en décisions confirmées par des tiers, et verrouiller le périmètre de lancement.

**Pré-requis :** aucun. C'est la phase d'entrée.

| # | Chantier | Livrable concret | Spec | Taille |
|---|---|---|---|---|
| 0.1 | Entité juridique | Note de décision : pays d'enregistrement, forme, compte bancaire, prestataire de paiement admissible | §28.2.1 | S — bloquant |
| 0.2 | Éligibilité paiement | Attestation/accusé du prestataire de paiement, liste des moyen de paiement par pays | §10.1, §10.2 | S — bloquant |
| 0.3 | Marchés prioritaires | Liste pays prioritaires / pays interdits au lancement, avec justification | §11.6, §28.2.3 | S |
| 0.4 | Incoterms & taxes | Table Incoterms utilisables (DDP/DAP/EXW…), qui paie quoi, modèle de TVA (prépayée vs à l'arrivée) | §12.1, §28.2.4 | M — bloquant |
| 0.5 | Transporteurs | 2–3 transporteurs retenus, couverture, disponibilité API, mode de suivi | §12.1, §12.2, §28.2.5 | M |
| 0.6 | Inventaire produits | CSV : SKU, catégorie, origine, poids, dimensions, conditionnement, code HS, statut de validation HS | §11.5, §28.2.2 | L — bloquant |
| 0.7 | Données personnelles & rétention | Registre des données traitées : champ, finalité, base légale, durée de conservation | §19.3, §28.2.12 | M |
| 0.8 | Politique B2B / B2C | MOQ, acomptes, devis, retours, garanties, annulations par marché | §8.4, §12.4, §28.2.7–8 | M |
| 0.9 | Devise & change | Devise de référence, devises d'encaissement, source et heure des taux, arrondis | §4.3, §28.2.6 | S |
| 0.10 | Hébergement & transferts | Région d'hébergement, localisation des données, transferts internationaux | §28.2.10 | S |
| 0.11 | SLA & support | Disponibilité cible, RPO/RTO confirmés, calendrier de support, escalade | §23.3, §28.2.11 | S |
| 0.12 | Modèle de menace | Document STRIDE sur les 12 flux sensibles + plan de réduction | §19.1 | M |
| 0.13 | Taxonomie des permissions | Matrice 9 rôles × permissions × ressources, validée par métier | §16.12, §2.4 | M |
| 0.14 | Maquettes UX | Wireframes des parcours B2C, B2B, checkout, dashboard client, dashboard admin | §6, §15, §16 | L |
| 0.15 | Feuille de route conformité | Points à valider par juriste / fiscaliste / conseil douanier, avec owner et date | §11.10, §19.10 | M |

**Skills :** `brainstorming` (structurer les décisions) · `caveman` (notes) · agents `Legal Compliance Checker`, `Tax Strategist`, `Software Architect`, `UI Designer`.

**Preuves de clôture :** note de décision signée pour chaque point bloquant (0.1, 0.2, 0.4, 0.6) · modèle de menace rédigé · matrice permissions approuvée · wireframes validés · liste des 12 décisions §28.2 avec statut (tranchée / en attente / non applicable).

**Critère de sortie :** les points 0.1, 0.2, 0.4 et 0.6 sont tranchés **par un tiers**, pas par supposition. Tout point bloquant non tranché verrouille la Phase 3 (pas la Phase 1).

**Dépendances / blocages :** 0.1 et 0.2 dépendent du propriétaire (entité, banque). 0.6 dépend de l'accès aux fiches fournisseur. Le fournisseur de conseil fiscal est un **dépôt à payer** — à budgéter avant la Phase 1.

---

### Phase 1 — Fondations et design system

**Objectif :** un socle exécutable de bout en bout : un utilisateur s'inscrit, se connecte, navigue une page publique localisée, et un admin se connecte avec MFA.

**Pré-requis :** Phase 0 points 0.9, 0.12, 0.13, 0.14 (le reste peut attendre).

| # | Chantier | Livrable concret | Spec | Taille |
|---|---|---|---|---|
| 1.1 | Monorepo | pnpm + Turbo : `apps/web`, `apps/api`, `apps/worker`, `packages/{db,domain,contracts,compliance,pricing,i18n,ui,observability,config}` | §5.1 | M |
| 1.2 | Schéma DB initial | Migrations Drizzle : user, session, role, permission, outbox, audit_log | §20 | M |
| 1.3 | Tokens & thèmes | `tokens.css` complet (light + dark), typographies, gradients, glassmorphism borné | §3.3–3.6 | M |
| 1.4 | Composants de base | Button, Input, Select, Dialog/BottomSheet, Badge, Skeleton, Toast, Table, Tabs, Pagination | §3.7 | L |
| 1.5 | i18n | Dictionnaires fr-FR / en / zh-CN, ICU, switcher, fallback explicite, formats `Intl` | §4.1–4.4 | M |
| 1.6 | Devise & montants | Types d'unité mineure, formatage par locale, distinction référence/transaction | §4.3–4.4 | S |
| 1.7 | Auth | Inscription, e-mail vérifié, connexion, cookie de session, rotation, révocation | §13.1, §13.4 | L |
| 1.8 | MFA | TOTP + codes de récupération, application sur rôles admin | §13.3 | M |
| 1.9 | RBAC | Guard serveur, 9 rôles, matrice permission × ressource, four-eyes | §2.4, §16.12 | M |
| 1.10 | Pages publiques | Accueil, catalogue (liste vide), conditions, contact, page 404 | §6.1 | M |
| 1.11 | CI | format/lint · typecheck · unit · integration · scans secrets+deps+SAST · build | §24.6, §25 | M |
| 1.12 | Environnements | local / test / staging / prod, secrets hors Git, migrations outillées | §25.1 | M |
| 1.13 | Observabilité | Logs JSON + correlation_id, métriques de base, tracking d'erreurs | §23.4 | M |
| 1.14 | PWA socle | manifest, service worker de base (NetworkOnly sur privé), installabilité | §22.3 | M |
| 1.15 | SEO technique | URLs `/fr/ /en/ /zh-cn/`, canonical, hreflang, robots.txt, sitemaps, JSON-LD Org/WebSite | §17.1–17.2 | M |

**Skills :** `frontend-design` · `taste-skill` · `theme-factory` · `composition-patterns` · `react-best-practices` · `web-design-guidelines` · `ui-ux-pro-max` · `api-platform-engineer` · `database-optimizer` · `devops` · `test-driven-development`.

**Agents :** `Software Architect` (1.1–1.2) · `UI Designer` (1.3–1.4) · `Backend Architect` (1.7–1.9) · `DevOps Automator` (1.11–1.13) · `SEO Specialist` (1.15) · `Test Automation Engineer` (1.11).

**Preuves :** pipeline CI vert de bout en bout sur PR · inscription → confirmation e-mail → connexion → page publique en 3 langues · admin se connecte avec MFA et est refusé sur une route non autorisée · `npm run build` sans erreur TypeScript strict · audit axe-core sans violation critique sur les pages publiques.

**Critère de sortie :** un utilisateur réel peut s'inscrire et parcourir le site en 3 langues dans 2 thèmes, et un admin peut se connecter en MFA. **Aucune feature métier livrée.**

**Blocages :** 1.11–1.12 exigent les comptes fournisseurs (GitHub, hébergement, registre de secrets). 1.15 exige le nom de domaine définitif.

---

### Phase 2 — Catalogue et commerce

**Objectif :** le premier vertical complet — un acheteur trouve un produit, respecte le MOQ, commande, paie en sandbox, reçoit confirmation et facture.

**Pré-requis :** Phase 1 complète · Phase 0.1 et 0.2 tranchées (sinon paiement simulé sans encaissement réel).

| # | Chantier | Livrable concret | Spec | Taille |
|---|---|---|---|---|
| 2.1 | Catalogue | Entités Product/ProductVariant, images S3, traductions produit, variantes | §7.1 | L |
| 2.2 | Prix dégressifs | PriceTier, bornes non chevauchantes (contrainte d'exclusion en base), priorité des règles | §7.2, §8.4 | L |
| 2.3 | MOQ & incréments | MOQ global + par variante, incrément (6/12/24), modes B2B/B2C par produit/pays | §2.2–2.3 | M |
| 2.4 | Stock & réservations | StockItem, StockReservation, TTL, verrou transactionnel anti-oversell | §7.3 | L |
| 2.5 | Recherche | PostgreSQL d'abord (tsvector + GIN), facettes, pagination keyset | §18.1 | M |
| 2.6 | Fiche produit | Galerie, tableau de paliers en contenu principal, MOQ, délai, indications pays | §6.4, §7.2 | M |
| 2.7 | Panier | Panier serveur, persistance, fusion, validation MOQ/incrément en temps réel | §9.1 | M |
| 2.8 | Checkout | Adresses, transport, recalcul serveur des prix/taxes/frais, snapshot | §9.2 | L |
| 2.9 | Commandes | Création, machine à états, snapshots, protection double commande | §9.3, §9.5 | L |
| 2.10 | Paiement | `PaymentProvider` + implémentation sandbox, éléments hébergés, 3DS si requis | §10.2–10.3 | L |
| 2.11 | Webhooks | Signature HMAC, horodatage, anti-rejeu, idempotence, outbox | §10.3, §21.3 | M |
| 2.12 | E-mails transactionnels | SPF/DKIM/DMARC, templates 3 langues, files + retry + DLQ | §14.3–14.4 | M |
| 2.13 | Factures & reçus | Numérotation conforme, PDF localisé + polices CJK, archivage, URL signée | §14.5 | M |
| 2.14 | Notifications in-app | Centre de notifications, badges, préférences | §14.1–14.2 | M |

**Skills :** `test-driven-development` (obligatoire sur 2.2–2.4, 2.9) · `payments-engineering` (2.10–2.11) · `api-platform-engineer` (2.8) · `database-optimizer` (2.5) · `frontend-design` + `composition-patterns` (2.6–2.7) · `seo-audit` (2.6) · `emil-design-eng` (polish) · `web-design-guidelines`.

**Agents :** `Backend Architect` · `Payments & Billing Engineer` · `Database Optimizer` · `Frontend Developer` · `Test Automation Engineer` · `SEO Specialist`.

**Preuves :** E2E Playwright « acheteur B2C » 9 étapes verte · E2E « paiement » 8 cas (dont signature invalide, doublon, hors ordre) verte · test de concurrence « 2 clients, dernière unité » : une seule commande · PDF de facture vérifié visuellement en 3 langues (CJK inclus) · SPF/DKIM/DMARC validés par un checker externe.

**Critère de sortie :** une commande sandbox complète traverse catalogue → panier → checkout → paiement → confirmation → facture PDF, en 3 langues, et le stock est décrémenté de façon atomique.

**Blocages :** 2.10 exige le compte de test du prestataire de paiement. 2.13 exige un moteur PDF capable d'embarquer les polices CJK.

---

### Phase 3 — B2B, logistique et douane

**Objectif :** le second vertical — un acheteur professionnel est vérifié, négocie un devis, estExpansionné par pays avec taxes et droits fiables, et suit son colis.

**Pré-requis :** Phase 2 complète · **Phase 0 points 0.3, 0.4, 0.5, 0.6, 0.8 tranchés.** Sans eux, cette phase ne démarre pas.

| # | Chantier | Livrable concret | Spec | Taille |
|---|---|---|---|---|
| 3.1 | Profil professionnel | Entités entreprise, vérification (upload de justificatifs), statuts, revue admin | §8.1–8.2 | L |
| 3.2 | Devis | Quote/QuoteRevision, cycle demande → négociation → acceptation → conversion en commande | §8.3, §15.4 | L |
| 3.3 | Tarification client | Règles prix spécifiques client avec priorité et dates | §8.4, §7.2 | M |
| 3.4 | Moteur de règles | TradeRule/TradeRuleVersion, résolution déterministe, 4 niveaux de certitude | §11.2, §11.5 | L |
| 3.5 | Calcul droits/TVA | Modèle 8 composants, base taxable, seuils, exemptions, frais de courtage | §11.3 | L |
| 3.6 | Affichage acheteur | Devis de frais avec mention d'incertitude selon niveau de certitude | §11.4 | M |
| 3.7 | Pays & restrictions | Activation pays, produits interdits/soumis à approbation, blocage serveur | §2.2, §11.6 | M |
| 3.8 | Transporteurs | `ShippingProvider`, intégration API, calcul de délai, fallback sandbox | §12.1, §12.3 | L |
| 3.9 | Tracking | Shipment/TrackingEvent, webhooks transporteur, resistance aux événements désordonnés | §12.2 | M |
| 3.10 | Documents export | Facture commerciale, packing list, déclaration export, label, certificat origine, conformité | §11.8 | L |
| 3.11 | Contrôles & alertes | Cohérence valeur/poids, détection de fractionnement, alerte règle expirée, blocage expédition incomplète | §11.9 | M |
| 3.12 | Alertes données manquantes | Blocage ciblé produit, revue HS, revue de règle par le compliance manager | §11.9, §16.9 | M |

**Skills :** `database-optimizer` (3.4) · `api-platform-engineer` (3.8) · `test-driven-development` (3.5) · `payments-engineering` (3.2) · `security-review` (3.10–3.11) · `pdf` / `markdown-to-pdf` (3.10) · agents `Legal Compliance Checker`, `Tax Strategist`.

**Agents :** `Backend Architect` · `Database Optimizer` · `Payments & Billing Engineer` · `Senior SecOps Engineer` (revue 3.10–3.11) · `Legal Compliance Checker`.

**Preuves :** E2E « acheteur B2B » 9 étapes verte · suite de 6 cas chiffrés de calcul de droits (base taxable, seuil, exemption, courtage) verte · test de résolution de règle vérifiant l'exclusion d'une règle expirée · PDF de facture commerciale et packing list vérifiés visuellement en 3 langues · revue de conformité signée sur les 12 documents.

**Critère de sortie :** un devis B2B devient une commande payée et expédiable vers 2 pays, avec un dossier export complet et un calcul de droits sourcé.

**Blocages :** 3.8 exige des comptes transporteur sandbox. 3.10 exige la validation des documents par un conseil douanier — délai externe non contrôlable.

---

### Phase 4 — Dashboards et opérations

**Objectif :** le pilotage — l'acheteur suit ses commandes, l'administre opère au quotidien, la finance rapproche, le support traite.

**Pré-requis :** Phase 3 complète.

| # | Chantier | Livrable concret | Spec | Taille |
|---|---|---|---|---|
| 4.1 | Dashboard client | Vue d'ensemble, commandes, devis, factures/paiements, adresses, notifications, préférences | §15.1–15.8 | L |
| 4.2 | Admin — commandes | Vue globale + gestion par rôle, transitions autorisées, actions groupées | §16.2–16.3 | L |
| 4.3 | Admin — catalogue | CRUD, import/export CSV/XLSX, validation, paliers, MOQ | §16.4, §7.4 | L |
| 4.4 | Admin — stock | Mouvements, seuils, réapprovisionnement, réservations | §16.5 | M |
| 4.5 | Admin — clients & B2B | Comptes professionnels, revue, approbation, suspension | §16.6 | M |
| 4.6 | Admin — finance | Rapprochement paiements/remboursements, exports, double contrôle | §10.5, §16.7 | L |
| 4.7 | Admin — logistique & douane | Expéditions, tracking, documents, règles, alertes | §16.8–16.9 | L |
| 4.8 | Admin — contenu & SEO | CMS, meta, slugs, redirections, previews | §16.10, §17 | M |
| 4.9 | Support | Tickets, litiges, remboursement depuis le support | §16.11, §12.5 | M |
| 4.10 | Audit & journal | Journal immuable, recherche, export autorisé, traçabilité des 12+ événements | §19.7 | M |
| 4.11 | Analytics | KPIs documentés (définition, provenance, périmètre) — jamais mélanger CA compta / commandes / encaissements | §16.2 | M |
| 4.12 | Notifications finalisées | ~17 déclencheurs, push opt-in, SMS optionnel, préférences par canal | §14.1–14.2 | M |

**Skills :** `frontend-design` + `composition-patterns` (4.1–4.9) · `ui-ux-pro-max` · `dataviz` (4.11) · `react-best-practices` · `performance` (dashboard ne charge que la vue courante) · `web-design-guidelines` · `webapp-testing`.

**Agents :** `Frontend Developer` · `UI Designer` · `Data Visualization Engineer` · `Code Reviewer` · `security-auditor` (admin, exports).

**Preuves :** E2E « admin » 11 cas verte · audit axe-core sans violation critique sur les 2 dashboards · test non-régression sur chaque guard RBAC des 12 actions sensibles · revue de sécurité ciblée sur exports et rapprochement financier · vérification manuelle clavier + lecteur d'écran sur les 2 dashboards.

**Critère de sortie :** l'ensemble des 9 rôles admin est exercé de bout en bout, chaque action sensible est scopée et journalisée, le rapprochement quotidien est reproductible.

**Blocages :** 4.6 exige des données de paiement sandbox représentatives. 4.12 exige un service SMS si retenu — coût externe.

---

### Phase 5 — Assurance qualité et lancement

**Objectif :** prouver que le produit tient ses promesses, puis ouvrir.

**Pré-requis :** Phases 1–4 complètes.

| # | Chantier | Livrable concret | Spec | Taille |
|---|---|---|---|---|
| 5.1 | Tests unitaires | Couverture complète des modules critiques (prix, taxes, machine à états, authz, règles) | §24.1 | L |
| 5.2 | Tests d'intégration | Base réelle, transactions, webhooks, outbox, jobs, PDF, import/export | §24.2 | L |
| 5.3 | E2E Playwright | Les 5 suites : B2C 9 étapes · B2B 9 étapes · paiement 8 cas · admin 11 cas · sécurité 7 cas | §24.3 | L |
| 5.4 | Responsive & visuel | 8 viewports × 3 langues × 2 thèmes ; snapshots de référence approuvés | §24.4 | M |
| 5.5 | Accessibilité | axe-core + jest-axe + Playwright + revue manuelle clavier/SR | §22.4 | M |
| 5.6 | Charge & résilience | k6 catalogue/recherche/checkout/webhook ; injection de pannes prestataire ; test de restauration | §24.5 | M |
| 5.7 | Sécurité | Revue OWASP ASVS, scan deps/secrets/SAST, test de concurrence, audit des accès | §19 | L |
| 5.8 | Recette métier 3 langues | Relecture humaine FR / EN / ZH-CN des chaines, e-mails, factures, CGU, pages légales | §4.2 | M |
| 5.9 | Validation légale & fiscale | Décisions signées : CGU/CGV, confidentialité, cookies, exemptions TV | §19.10, §28.2 | M |
| 5.10 | Observabilité & runbooks | Alertes, seuils, 5 runbooks,仪表 de supervision | §23.4–23.5 | M |
| 5.11 | Sauvegardes & rollback | PITR, snapshots, **test de restauration exécuté**, procédure de rollback | §25 | M |
| 5.12 | Documentation | Guide opérateur admin, guide contributeur, runbooks, ADR | §26.6 | M |
| 5.13 | Checklists lancement | Annexe A (14 points/produit) et Annexe B (11 points/destination) passées | Annexes A/B | M |
| 5.14 | Go/No-Go | Revue finale avec les 12 décisions §28.2 tracées | §28.2 | S |

**Skills :** `verification-before-completion` (obligatoire) · `security-review` · `webapp-testing` · `a11y` / `wcag` · `screenshot` · `seo-audit` · `copy-editing` · `finishing-a-development-branch` · `caveman-commit`.

**Agents :** `Reality Checker` (gate final) · `Evidence Collector` (captures) · `security-auditor` · `qa-e2e` · `seo-auditor` · `ui-ux-checker` · `Legal Compliance Checker` · `Tax Strategist`.

**Preuves :** les 5 suites E2E vertes · CWV p75 conformes · **test de restauration exécuté avec RPO/RTO mesurés** · revue de sécurité sans finding critique ouvert · recette 3 langues signée · checklist §27 complète.

**Critère de sortie :** Go/No-Go au vert. Aucun point ouvert sur les parcours paiement, expédition, conformité ou sécurité.

---

### Phase 6 — Améliorations post-lancement

**Objectif :** optimiser après mesure réelle. Rien n'est engagé avant d'avoir 30 jours de données d'usage.

PWA complète et notifications push · tarifs contractuels avancés et espace multi-utilisateur entreprise · intégrations ERP/WMS/comptabilité · moteur de recherche dédié et recommandations · IA assistive (support, traduction, complétude catalogue) · rapports multi-entrepôts et prévision de stock.

**Garde-fou IA (spec §18.4) :** l'IA est **assistive** uniquement. Elle ne fixe aucun prix final, ne valide aucun code HS seule, ne confirme aucune importabilité, ne déclenche ni remboursement ni suspension, n'invente ni certification ni avis ni stock ni délai. Tout	output touchant conformité, sécurité, taxe ou contrat exige vérification humaine avant publication.

**Skills :** `cro` · `pricing` · `seo-audit` · `programmatic-seo` · `analytics` · `content-strategy` · `ab-testing`.

---

### Matrice de traçabilité spec → phase

| Spec | Phase | Spec | Phase | Spec | Phase |
|---|---|---|---|---|---|
| §1 Vision, objectifs, KPI | 0 | §10 Paiements, réconciliation | 2 (0.1) | §19 Sécurité & données | 0,1,3,5 |
| §2 Publics, B2B/B2C, autorisation | 0,1,2 | §11 Douanes, TVA, conformité | 0,3 | §20 Modèle de données | 1,2 |
| §3 Identité visuelle, design system | 1 | §12 Livraison, retours, litiges | 3,4 | §21 API, événements, intégrations | 1,2,3 |
| §4 Langues, devises | 1 | §13 Auth, comptes, 2FA | 1 | §22 Responsive, PWA, a11y | 1,5 |
| §5 Architecture, modules | 1 | §14 Notifications, e-mails, factures | 2 | §23 Perf, dispo, observabilité | 1,5 |
| §6 Navigation, parcours | 1,2 | §15 Dashboard client | 4 | §24 Tests | 5 |
| §7 Catalogue, prix, stock | 2 | §16 Dashboard admin | 4 | §25 Déploiement | 1,5 |
| §8 B2B, devis, prix client | 3 | §17 Contenus, SEO, marketing | 1,4 | §26 Critères d'acceptation | 5 |
| §9 Panier, checkout, commandes | 2 | §18 Recherche, IA assistive | 2,6 | §27 Phasage | 0–6 |
| | | | | §28 Risques, décisions | 0,5 |
| | | | | §29 Sources officielles | 0,5 |
| | | | | Annexe A (produit) | 5 |
| | | | | Annexe B (destination) | 5 |
| | | | | Annexe C (release) | 5 |

---

### Vagues de la Phase 1 (2 semaines chacune)

| Vague | Objectif | Livrables | Pile de tests qui doit passer |
|---|---|---|---|
| 1.1 | Socle repo | monorepo, TS strict, ESLint flat, Prettier, Husky, commitlint, `.gitignore`, `LICENSE`, `CONTRIBUTING` | `typecheck` + `lint` verts sur CI |
| 1.2 | DB + outbox | schéma Drizzle initial, migration outillée, runner, outbox + audit_log + trigger append-only | integration : migration apply/teardown sur Postgres réel |
| 1.3 | Tokens + composants | `tokens.css` (2 thèmes), Storybook, Button/Input/Badge/Skeleton/Dialog | Storybook build vert + axe-core 0 violation critique |
| 1.4 | i18n + devises | dictionnaires 3 langues, `Intl` formatters, switcher, middleware langue | test : formatage FR/EN/ZH + non-régression des clés |
| 1.5 | Auth cœur | inscription, e-mail vérifié, Argon2id, cookie session, rotation, révocation | unit (hash/rotation) + integration (session lifecycle) |
| 1.6 | MFA | TOTP, secret chiffré, anti-rejeu, codes de récupération, application admin | unit (RFC 6238 vecteurs) + integration (activation/désactivation) |
| 1.7 | RBAC | 9 rôles, guard, matrice, four-eyes | unit (matrice complète) + integration (routes protégées) |
| 1.8 | Pages publiques + CI + observabilité | accueil, catalogue vide, 404, robots/sitemap/hreflang, pipeline CI complet, logs JSON | E2E 1 parcours par langue + CI verte de bout en bout |

**Vagues de la Phase 2**

| Vague | Objectif | Livrables | Pile de tests |
|---|---|---|---|
| 2.1 | Catalogue | Product/Variant, traductions, images S3, import CSV | integration CRUD + import |
| 2.2 | Prix & MOQ | PriceTier (exclusion), règles client, MOQ/incréments, priorité | unit exhaustif + test float monetary |
| 2.3 | Stock | StockItem, réservation TTL, verrou anti-oversell | integration + **test de concurrence** |
| 2.4 | Recherche & fiche produit | tsvector+GIN, facettes, page produit, paliers en contenu | integration requête + E2E navigation |
| 2.5 | Panier | panier serveur, validation MOQ temps réel | E2E panier + unit validation |
| 2.6 | Checkout | adresses, transport, recalcul serveur, snapshot | integration recalcul + E2E checkout |
| 2.7 | Commandes | machine à états, snapshots, protection double commande | unit transitions (valides + interdites) |
| 2.8 | Paiement + webhook | provider sandbox, éléments hébergés, signature, idempotence, outbox | E2E paiement 8 cas |
| 2.9 | E-mails + factures | templates 3 langues, SPF/DKIM/DMARC, PDF CJK, archivage | integration email queue + PDF 3 langues |
| 2.10 | Notifications | centre in-app, badges, préférences | E2E notification |

---

### Jalons de validation (gate reviews) — un par phase

**Gate 0 — Conformité**
1. Entité juridique et compte bancaire confirmés par écrit ?
2. Prestataire de paiement a confirmé l'éligibilité du pays marchand ?
3. Les 12 décisions §28.2 ont-elles toutes un statut ?
4. Chaque décision bloquante a-t-elle un propriétaire nommé ?
5. Le modèle de menace couvre-t-il les 12 flux sensibles ?
6. La matrice 9 rôles est-elle validée par le métier ?
7. Les wireframes couvrent-ils B2C, B2B, checkout et les 2 dashboards ?
8. L'inventaire produit a-t-il SKU, poids, origine et code HS pour 100 % des références ?

> Passage si 1, 2 et 7 sont « oui ». Sinon retour en Phase 0.

**Gate 1 — Socle**
1. CI verte de bout en bout sur une PR ?
2. `typecheck` strict sans erreur, build prod reproductible ?
3. Inscription → connexion → page publique en 3 langues, 2 thèmes ?
4. Admin se connecte avec MFA, refusé hors permission ?
5. Un secret a-t-il été commité ? (scan vert)
6. Le `robots.txt` et les sitemaps sont-ils en place ?
7. Axe-core : 0 violation critique sur les pages publiques ?
8. Le journal d'audit écrit-il déjà les 3 événements de contrôle ?

> Passage si 1, 3, 4 sont « oui ».

**Gate 2 — Commerce**
1. Parcours B2C 9 étapes vert de bout en bout ?
2. Les 8 cas de paiement verts (dont doublon et signature invalide) ?
3. Le test « 2 clients / dernière unité » prouve-t-il une seule commande ?
4. Le PDF de facture est-il lisible en CJK ?
5. SPF/DKIM/DMARC validés par un outil externe ?
6. Le recalcul serveur du checkout est-il覆盖é par un test de non-régression ?
7. Le stock décrémenté est-il atomique et traçable ?

> Passage si 1, 2, 3 sont « oui ».

**Gate 3 — B2B & douane**
1. Parcours B2B 9 étapes vert ?
2. Les 6 cas chiffrés de droits/TVA sont-ils verts ?
3. Une règle expirée est-elle rejetée par le moteur ?
4. La détection de fractionnement bloque-t-elle un colisfractionné ?
5. Le dossier export complet est-il validé par un conseil ?
6. La documentation CN est-elle validée par un fournisseur de transport réel ?

> Passage si 1, 3, 4 sont « oui » et si 5 est « oui ou planifié avec date ».

**Gate 4 — Opérations**
1. Les 11 cas admin E2E sont-ils verts ?
2. Les 12 actions sensibles sont-elles scopées et testées ?
3. Chaque action financière sensible passe-t-elle par le double contrôle ?
4. Le rapprochement quotidien est-il reproductible depuis l'UI ?
5. Le journal d'audit est-il immuable (trigger SQL actif) ?
6. Les 2 dashboards passent-ils la revue clavier + lecteur d'écran ?

> Passage si 1, 2, 3 sont « oui ».

**Gate 5 — Lancement**
1. Les 5 suites E2E sont-elles vertes ?
2. Le test de restauration a-t-il été **exécuté** (RPO/RTO mesurés) ?
3. CWV p75 conformes aux seuils ?
4. La revue de sécurité a-t-elle zéro finding critique ouvert ?
5. La recette 3 langues est-elle signée ?
6. Les checklists Annexe A et B sont-elles passées ?
7. Les 5 runbooks existent-ils et ont-ils été lus ?
8. Le rollback a-t-il été testé en staging ?

> Passage si 1, 2, 4, 8 sont « oui ». Go/No-Go formel ensuite.

**Gate 6 — Post-lancement**
1. 30 jours de données d'usage disponibles ?
2. Les métriques de qualité sont-elles dans les seuils ?
3. Aucun finding critique non traité depuis le lancement ?

---

### Definition of Ready (une tâche est prête quand)

- [ ] Les critères d'acceptation sont **écrits** et rattachés à une section du spec
- [ ] Les dépendances sont faites ou explicitement mockées dans le test
- [ ] Le périmètre est borné : ce qui est **exclu** est écrit
- [ ] L'impact sur la DB est connu (migration ? contrainte ? index ?)
- [ ] L'impact i18n est connu (nouvelles chaînes ? FR/EN/ZH ?)
- [ ] Le comportement en cas d'échec de l'intégration est défini
- [ ] La tâche a une taille ≤ 1 journée ; au-delà, elle est découpée

### Definition of Done (une tâche est terminée quand)

- [ ] Le code est livré sur une branche `feat|fix|refactor|test/<slug>` — **jamais sur `main`**
- [ ] Les tests sont écrits et **passants** — la sortie est collée dans la PR
- [ ] Un test de non-régression existe si c'est une correction de bug
- [ ] La revue sécurité ciblée est faite (authz, injection, XSS, secrets, uploads, rate limit)
- [ ] Le diff est relu : pas de debug, pas de TODO, pas de secret, pas de code mort
- [ ] Les 3 langues et les 2 thèmes sont à jour si l'UI est concernée
- [ ] Les états loading / vide / erreur sont implémentés
- [ ] L'accessibilité clavier + focus + contraste est respectée
- [ ] Aucune régression perf mesurable (bundle, requêtes, images)
- [ ] La migration est testée sur base de test ; si destructive : accord + restauration prouvée
- [ ] La documentation est à jour (README, runbook, `.env` en **noms de variables seulement**)
- [ ] Le commit est conventionnel, la PR est ouverte avec ses preuves
- [ ] **La PR n'est pas mergée** sans accord explicite du propriétaire

---

### Chemin critique

```
Phase 0 (entité + paiement + Incoterms + HS)
   └─> Phase 2.8/2.9 (checkout + paiement sandbox + webhooks)
          └─> Phase 2.10 (e-mails + factures)
                 └─> Phase 3 (expédition + douane + documents)
                        └─> Phase 5 (recette 3 langues + validation légale)
                               └─> Go/No-Go
```

C'est le **seul** chemin strictement séquentiel de 4 à 6 mois. Tout le reste (design system, i18n, CI, observabilité, catalogue, recherche, dashboard) est parallélisable.

**Jalons de valeur livrables au plus tôt :**

| Jalon | Contenu | Condition |
|---|---|---|
| **MVP gris** | Catalogue FR/EN, panier, checkout, paiement sandbox, commande, e-mail — sans douane ni B2B | Phase 2 + Phase 1.5 (auth) |
| **B2B** | Devis, compte pro vérifié, prix négociés, dashboard client | Phase 3.1–3.3 + Phase 4.1 |
| **International** | Taxes/droits par pays, documents export, transporteurs, suivi | Phase 3 complète |

---

### Matrice de risques

| Risque (§28.1) | P | I | Signal précoce | Owner | Plan A | Plan B (déclencheur) |
|---|---|---|---|---|---|---|
| Prestataire de paiement non éligible | M | **Critique** | Refus ou silence du prestataire | Propriétaire | Compte dans une juridiction éligible | Prestataire alternatif derrière `PaymentProvider` (**déclencheur : refus écrit**) |
| Règles douanières obsolètes | Élevée | Élevée | Règle dont `valid_to` < aujourd'hui | Compliance manager | Versionnement + source officielle + revue mensuelle | Blocage de l'expédition + alerte (**déclencheur : règle expirée non revue**) |
| Données produit incomplètes | Élevée | Moyenne | Produits bloqués à l'import | Catalog manager | Champs obligatoires + validation import | Produit non publiable (**déclencheur : >2 % des SKU incomplets**) |
| Mauvaise estimation transport | Moyenne | Élevée | Marge négative sur commande close | Logistics operator | API transporteur + revalidation | Affichage d'incertitude + alerte (**déclencheur : marge < seuil**) |
| Fraude au paiement | Moyenne | Élevée | Pic de refus/chargebacks | Finance manager | 3DS + heuristiques + revue | 3DS forcé sur zone à risque (**déclencheur : taux > seuil**) |
| Double traitement webhook | Faible | Élevée | Commande payée 2× | Backend | Idempotence + outbox + test de duplication | Rejeu manuel depuis la DLQ (**déclencheur : occurrence confirmée**) |
| Fuite de données de compte | Faible | **Critique** | Alerte anomalie d'accès | Security owner | MFA admin + RBAC + chiffrement + monitoring | Révocation de sessions + rotation (**déclencheur : incident**) |
| Traduction imprécise | Élevée | Moyenne | Relecture humans en discrepancy | Content owner | Relecture humaine des termes critiques | Publication bloquée (**déclencheur : terme juridique non relu**) |
| Rupture de stock concurrente | Moyenne | Élevée | Sur-réservation détectée | Backend | Réservation transactionnelle + test de concurrence | Reconciliation de stock (**déclencheur : écart > 0**) |
| Surcoût glassmorphism | Moyenne | Faible | LCP > 2,5 s sur mobile réel | Frontend | Usage parcimonieux + test sur vrais appareils | Repli opaque + `prefers-reduced-motion` (**déclencheur : CWV non conforme**) |

**Risques de projet (ajoutés) :**

| Risque | P | I | Plan |
|---|---|---|---|
| Propriétaire indisponible pour les décisions bloquantes (Phase 0) | Élevée | **Critique** | Créer un **dossier de décision** avec options + recommandation, à valider en lot. Ne jamais trancher seul une décision juridique/financière |
| Dépendance à des tiers (conseil fiscal, transporteurs, hébergeur) | Élevée | Élevée | Démander les accès en Phase 0, pas en Phase 3. Prévoir 2 transporteurs |
| Perte de contexte entre sessions | Élevée | Moyenne | `CLAUDE.md` + `ROADMAP.md` + plans par phase + commits conventionnels. Reprise : `git status` → roadmap → PR active |
| Effort sous-estimé sur la conformité | Moyenne | Élevée | Le moteur de règles est la pièce la plus coûteuse. Ne pas le construire avant d'avoir les données HS validées |
| Traduction zh-CN bâclée (marché prioritaire en Chine) | Moyenne | Moyenne | Relecture humaine obligatoire par un locuteur natif, pas par la machine |
| Ajout de périmètre après lancement | Élevée | Moyenne | Phase 6 existe pour ça. Toute nouvelle demande en Phase ≤ 4 passe par un gate |

---

### Charge et capacité

| Phase | j-h estimés |
|---|---|
| 0 — Cadrage | 60 (beaucoup hors dev) |
| 1 — Fondations | 220 |
| 2 — Catalogue & commerce | 340 |
| 3 — B2B, logistique, douane | 380 |
| 4 — Dashboards | 300 |
| 5 — QA & lancement | 240 |
| 6 — Post-lancement | variable |
| **Total** | **~1 540 j-h** |

| Scénario | Équipe | Calendrier indicatif | Conséquence |
|---|---|---|---|
| **Solo + IA** | 1 dev | 14–18 mois | Tenable sur le périmètre. **Priorité :** B2C d'abord, un seul transporteur, 2 pays, 1 devise de paiement. Douane et B2B négocié en Phase 6. Risque principal : le propriétaire est aussi le seul decidedeur → geler la Phase 0 par un dossier de décision |
| **Équipe de 2** | 1 full-stack + 1 back/DevOps | 9–11 mois | Doux dollé. Paralleliser Phase 2 (catalogue ↔ paiement) et Phase 4 (dashboard client ↔ admin). La conformité reste le goulot |
| **Équipe de 3–4** | 2 dev + 1 back/DevOps + 0,5 design/QA | 6–8 mois | Tout le périmètre est realistic. Le goulot devient la **revue** : imposer la règle des 800 lignes/PR et 2 approbations |

**Réduction de périmètre (si retard) :** sortir dans l'ordre — (1) B2C détail désactivé, (2) pays au-delà de 2, (3) au-delà d'1 devise d'encaissement, (4) SMS, (5) push, (6) recherche avancée. **Ne jamais sortir** : MFA admin, double contrôle financier, journal d'audit, moteur de règles douanières, idempotence des paiements.

---

### Coûts à valider avant la Phase 1

| Poste | Détail | Pourquoi maintenant |
|---|---|---|
| Entité juridique + conseil | Société, banque, accountant, conseil fiscal/douanier | **Bloquant Phase 0.** Délai externe le plus long |
| Compte prestataire de paiement | frais de transaction + eventual onboarding fee | Dépend de l'entité |
| Nom de domaine + DNS | 2–3 domaines (boutique, transactionnel e-mail, suivi) | Requis avant l'email et le SEO |
| Service e-mail transactionnel | envoi + délivrabilité | Requis avant la Phase 2 |
| Stockage objet + CDN | images produits, documents, PDF | Requis dès la Phase 2 |
| Hébergement | conteneurs web/api/worker + PostgreSQL managé + Redis | Requis avant la Phase 1.12 |
| Registre de secrets | coffre dédié (pas GitHub Secrets comme coffre applicatif) | Requis avant la Phase 1.12 |
| Monitoring / tracking d'erreurs | logs, métriques, alertes | Requis avant la Phase 1.13 |
| Transporteurs | comptes + API (2 fournisseurs minimum) | Phase 0.5 |
| Outils de test | E2E, charge, a11y | Phase 1.11 |
| **Budget mensuel d'infra** | faible / moyen / fort trafic | À estimer avant la Phase 1.12 |

---

### Ce qui n'est **jamais** négocié

MFA sur les rôles admin/finance/compliance · double contrôle sur action financière · journal d'audit immuable · idempotence des paiements et webhooks · recalcul serveur des prix et taxes · moteur de règles douanières versionné avec source et date de revue · verrou de non-sous-évaluation et de non-fractionnement · validation humaine des textes juridiques en 3 langues · CWV au seuil p75 · test de restauration exécuté.
