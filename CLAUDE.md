# Fanours — Instructions projet

Fichier relu à **chaque prompt**. Source de vérité = `Cahier_des_charges_Fanours.md` (29 sections + annexes A/B/C).
Roadmap = `ROADMAP.md`. Plans phase = `docs/superpowers/plans/`.

**Règle de lecture :** en cas de conflit, `Cahier_des_charges_Fanours.md` > `ROADMAP.md` > ce fichier > conventions habituelles.
Ce fichier contient les **pratiques** ; il ne remplace pas le spec. Si le spec est plus exigeant, le spec gagne — signale l'écart.

---

## 0. Rituels de session (avant toute action)

1. `git status` + branche courante. Jamais de dev/commit/push sur `main`. Branche : `feat|fix|refactor|test|docs|chore/<slug>`.
2. Lire `ROADMAP.md` → phase courante + critères d'acceptation de la tâche.
3. `gh pr status` si une PR est ouverte.
4. Charger les skills utiles (voir §9) **avant** de coder, pas après.

**Interdits absolus :** merge sans accord explicite · force-push · secret en fichier/mémoire/log/commit · migration destructive en prod sans accord + restauration prouvée · données de carte réelles en dev · bypass de test/protection · « terminé » sans preuve.

---

## 1. Stack (figée — spec §5.1)

| Couche | Choix | Contrainte |
|---|---|---|
| Front | Next.js 15 App Router + TypeScript strict + Tailwind | Tokens CSS centralisés, aucune couleur en dur hors tokens |
| Back | TypeScript (modular monolith) | Logique métier / accès données / présentation séparés |
| Données | PostgreSQL + **Drizzle** (choisi) | Un seul ORM, pas de mix Prisma/Drizzle |
| Cache/queue | Redis (BullMQ) | Idempotence + rate limit + files |
| Objets | S3-compatible + CDN | Médias et documents, jamais en bundle web |
| Paiement | Interface `PaymentProvider` + implémentation Stripe | Prestataire interchangeable, jamais de PAN stocké |
| Docs PDF | Générateur headless + polices CJK embarquées | PDF jamais dans le bundle public |
| Tests | Vitest + Testing Library + Playwright | Framework réel d'abord |
| Déploiement | Conteneurs (Cloud Run / ECS) — **pas** Vercel seul | Worker long-running + webhooks stables + coût maîtrisé |

**Front n'est jamais source de vérité** pour prix, taxe, frais, stock, autorisation. Recalcul serveur obligatoire (§7.2, §8.2).

---

## 2. Direction design & UI/UX (spec §3, §6, §15, §16, §22)

### 2.1 Identité
« Un peu de rose. Beaucoup de style. Je craque. » Rose = signature, **pas**dobber de fond. Chaque page a une hiérarchie claire. Le B2B exige crédibilité : le rose vit dans les accents (CTA, badges, focus), pas dans le bruit.

**Interdit (rejet auto UI générique) :** hero sans identité · gradient partout · glassmorphism sur toute surface · emoji comme icône · badge `rounded-full` systématisé · ombre portée décorative sans hiérarchie · coins incohérents · espace non Token.

### 2.2 Tokens — source unique
Tout vient de `--token`. Jamais de hex en dur dans un composant. Bloqué par ESLint (règle `no-restricted-syntax` sur les couleurs hex hors `tokens.css`).

- Clair : fond `#FFF5FA` · surface `#FFFFFF` · texte `#321526` · primaire `#D81B75` · accent `#FFD6EA` · surface2 `#F7DCE9`
- Sombre : fond `#180D15` · surface `#261420` · texte `#FFF0F7` · primaire `#FF70B7` · accent `#4B1F39` · surface élevée `#35202D`
- Fonctionnels : succès `#16794B`/`#4ADE80` · attention `#9A5B00`/`#FBBF24` · erreur `#B42336`/`#FF6B7A` · info `#2458B8`/`#7CA7FF` · neutre `#667085`/`#B8A8B3` · bordure `#F0DCE6`/`#49303E`
- Typo : Plus Jakarta Sans (titres 500–800) · Inter (corps 400–700) · Noto Sans SC (zh-CN). `font-display: swap`. CSS vars `--font-heading` / `--font-body`.
- Rythme : espacements échelle 4px · rayons `--radius-sm/md/lg/xl` · ombres `--shadow-1..3` · z-index nommés.

Le fichier de tokens est la source : `packages/ui/src/styles/tokens.css` (light + dark via `:root:not([data-theme="light"])` et `:root[data-theme="dark"]`).

### 2.3 Thème
Clair / sombre / auto. Persistance locale immédiate + synchro compte. Anti-flash : script inline avant hydration. Le checkout et la page de paiement hebergée gardent un contraste minimal imposé (le prestataire n'accepte que les personnalisations autorisées).

### 2.4 Glassmorphism — budget serré
Autorisé : header, barre de panier, modale, drawer. **Interdit** : zones paiement, connexion, MFA, tableaux opérationnels, checkout (le spec l'exclut). `backdrop-filter: blur(12px)`–`blur(20px)`, opacité 70–90 %, bordure 1px translucide, **fallback opaque** si `blur` non supporté. Jamais au-dessus d'un texte critique. Max 2 couches floutées simultanées sur mobile (risque perf explicitement au §28.1).

### 2.5 Règles responsive
Mobile-first. Breakpoints : ≤360 · 361–767 · 768–1023 · 1024–1439 · ≥1440.
Nav basse 4 onglets (Accueil / Catalogue / Panier / Commandes-Compte). En-tête compact. **Zéro débordement horizontal** (assertion E2E `document.scrollWidth <= viewport`). Cibles tactiles ≥44px. Tableaux → cartes en mobile. Upload document possible depuis mobile.

### 2.6 États obligatoires
Chaque vue implémente : loading (skeleton) · vide · erreur · accès refusé · session expirée · réseau indisponible · ressource indisponible · conflit de stock · quantité invalide · paiement en attente · interruption prestataire.
Message **localisé FR/EN/ZH**, icône, action de sortie. **Jamais de donnée technique exposée** au client.

### 2.7 Accessibilité (WCAG 2.2 AA) — spec §22.4
Focus visible (anneau `--focus-ring`, offset 2px) · ordre de focus logique · piège de focus en modale · landmarks ARIA · live regions sur recalcul de total et changement de stock · annonces SR · target size ≥24×24 (2.5.8) · `prefers-reduced-motion` respecté · 200% zoom sans perte de contenu · jamais couleur seule · erreurs liées au champ et annoncées.

**L'audit auto ne prouve rien** : passage manuel clavier + lecteur d'écran sur checkout, login, dashboard, gestion de commande est exigé (voir §8, ROADMAP §Qualité).

### 2.8 i18n dans l'UI (spec §4)
Aucune chaîne en dur — tout vient des dictionnaires `fr-FR` / `en` / `zh-CN`. Locale de repli explicite. Expansion de layout (EN ≈ +30 % vs FR) : conteneurs `min-w-0`, `truncate`, pas de largeur fixe sur du texte traduit. CJK : pas d'italique, interligne +10 %. Montants/date : format locale via `Intl`, jamais de formatage manuel. Distinguer **devise d'affichage** vs **devise de transaction** (§4.4) et marquer « indicatif » quand converti.

---

## 3. Base de données (spec §20)

### 3.1 Règles
- Un seul ORM : **Drizzle**. Migrations SQL versionnées à la main, revues, appliquées via runner dédié. Pas de `db push` en prod.
- IDs stables (UUID v7). FKs explicites. Contraintes `UNIQUE` sur les clés métier (email normalisé, SKU, n° de commande, n° de document).
- `CHECK` sur invariants : `quantity > 0`, `price >= 0`, `amount >= 0`, pas de ligne orpheline, stock ≥ 0.
- Index sur : SKU, email normalisé, n° commande, statut, dates de recherche, `sku + min_qty + valid_from`.
- Montants : **jamais de float binaire**. `NUMERIC` en base + entier de minor units en code. Devises en codes ISO 4217.
- Snapshot dans la commande : adresse, prix, règles de taxe, conditions de vente, ID de version de règle. La commande ne relit jamais le catalogue courant.
- Transactions atomiques pour : paiement, réservation de stock, création de commande. `SELECT ... FOR UPDATE` ou `UPDATE ... WHERE stock >= n` pour éviter l'oversell.
- Audit : toute modification de prix, stock, statut, règle douanière, rôle est historisée (append-only).

### 3.2 Tables clés à créer
`User` · `Session` · `MfaMethod` · `RecoveryCode` · `Product` · `ProductVariant` · `PriceTier` · `CustomerPriceRule` · `StockItem` · `StockReservation` · `Cart`/`CartLine` · `Quote`/`QuoteRevision` · `Order`/`OrderLine`/`OrderStatusEvent` · `Payment`/`PaymentEvent`/`Refund` · `Shipment`/`TrackingEvent` · `TradeRule`/`TradeRuleVersion` · `AuditLog` · `OutboxEvent` · `Document` · `Notification` · `IdempotencyKey` · `AdminActionApproval`.

### 3.3 Moteur de règles douanières
Règles = **données**, jamais du code. `TradeRule` (pays/territoire, catégories/produits, code HS, type, mode de calcul, base taxable, seuils, niveau de certitude, propriétaire de validation, date de dernière revue) + `TradeRuleVersion` (dates début/fin, source officielle + URL + date de consultation, historique).
- Résolution : pays + produit + **date de la commande** → règle applicable la plus spécifique et la plus récente.
- Règle expirée → **alerte au compliance manager**, jamais d'application silencieuse.
- Niveau de certitude (`validé` / `estimatif` / `incomplet` / `spécialiste`) affiché dans l'admin et exigé avant publication.
- Interdit : sous-valoirisation, fausse catégorie/origine, fractionnement de colis.

### 3.4 Migrations
Expand/contract : ajout d'abord, application, retrait différé d'une release. Aucune migration destructive sans **preuve de restauration testée** + plan de recovery. Testée sur base de test en CI. Backup vérifié avant toute migration en prod.

---

## 4. Backend, API & architecture (spec §5, §21)

- **Monolithe modulaire**, 15 modules métier. Pas de microservices prématurés. Front ≠ source de vérité.
- REST + OpenAPI. Validation schéma **en entrée et en sortie** (zod). Versioning `/v1`.
- Erreurs : **code stable** + message UI localisé + `correlation_id`. Jamais de stack trace ni de message SQL au client.
- Pagination + filtres contrôlés. Limites de taille requête/réponse. Champs internes jamais exposés en public.
- `authN` + `authZ` sur **chaque** endpoint privé. Les routes `/admin/*` ne sont jamais protégées par l'obscurité d'URL seule.
- **Idempotence** sur toute opération sensible (création de commande, paiement, remboursement, webhook) — clé unique en base.
- **Outbox pattern** pour les événements : 16+ événements métier publiés de façon fiable, consommateurs idempotents.
- Chaque intégration externe a : propriétaire, clé de test, procédure staging, politique de retry, alertes, procédure de rotation de secret, repli opérationnel.
- Les prix/taxes/frais sont recalculés **côté serveur** à chaque checkout, et le snapshot est persisté.

---

## 5. Infrastructure & exploitation (spec §23, §25)

### 5.1 Environnements
`local` (données factices, sandbox) · `test/CI` (synthétique) · `staging` (prod-like, clés sandbox, UAT) · `production` (accès restreint, secrets dédiés, alerting, backups, rollback).
**Aucune donnée réelle de carte ni document client sensible en dev** sans base légale + masquage + procédure approuvée. Une seule procédure de migration d'environnement, versionnée, sans secret en clair.

### 5.2 Pipeline CI — aucun déploiement si un job bloquant échoue
format/lint → typecheck strict → unit → integration (Postgres réel) → scan secrets + dépendances + SAST → build reproductible + SBOM → E2E sur staging → revue sécurité + validation métier sur changement sensible → déploiement progressif avec rollback → smoke tests + métriques.

### 5.3 Observabilité
- Logs JSON structurés : `timestamp, level, service, env, correlation_id, user_id (hashé), route, status, duration_ms`. Redaction automatique des PII et secrets. Aucune donnée de carte. Rétention définie.
- `correlation_id` propagé de bout en bout (HTTP → job → intégration) en `traceparent` W3C.
- Métriques : latence p50/p95/p99 par route, taux d'erreur, profondeur de file, durée de job, échecs webhooks, doublons, échecs paiement, échecs réservation, emails en échec, DLQ. Alertes sur seuils.
- Alertes : paiement indisponible · webhook en échec répété · file email saturée · stock critique · 5xx au-dessus du seuil · p95 > seuil · certificat proche de l'expiration · backup échoué.

### 5.4 Fiabilité
Disponibilité cible 99,5 % mensuel. Health checks. **RPO ≤ 24 h / RTO ≤ 8 h** (valeurs de départ, à revoir avec le propriétaire). Backups chiffrés, 3-2-1, **test de restauration mensuel**. Runbooks obligatoires : paiement HS · e-mail HS · transporteur HS · incident sécurité · restauration.

### 5.5 Jobs
Retry : backoff exponentiel + jitter, attempts plafonnés, DLQ, **idempotence par job**, clé d'idempotence par événement. Types : email transactionnel · webhook paiement · génération PDF/facture · estimation transport · sync tracking · réconciliation paiement · import catalogue · export · analytics · alerte stock · revue de règle · sitemap · notifications · traitement images.

### 5.6 E-mail & PDF
SPF, DKIM (rotation), DMARC (`p=none` → `quarantine` → `reject`), bounce et complaint tracking, DLQ + renvoi autorisé. Transactionnel ≠ promotionnel : **l'opt-out marketing ne doit jamais tuer un email transactionnel**. PDF : polices CJK embarquées, cache, stockage S3 + URL signée courte durée, jamais en bundle public.

### 5.7 PWA
Cache : shell en precache, assets publics en runtime, pages catalogue en NetworkFirst, **panier/checkout/compte en NetworkOnly**. Interdiction absolue de mettre en cache factures/commandes/données privées. Le paiement exige le réseau **et** la validation serveur — jamais « payé » déclaré en local. Service worker versionné, nettoyage des anciens caches.

---

## 6. SEO & contenu (spec §17)

**S'applique** : pages publiques FR/EN/ZH-CN indexables. **Ne s'applique pas** : API, admin, compte, panier, checkout, devis, factures, suivi de commande, recherche interne → `noindex` + exclusion sitemap.

- URLs : `/fr/`, `/en/`, `/zh-cn/`. `canonical` absolu. `hreflang` complet + `x-default`. **Langue ≠ pays ≠ devise** : trois dimensions distinctes, jamais confondues.
- **Pas de redirection forcée par IP** (interdit). Détection par `Accept-Language` seulement, avec switcher visible.
- `robots.txt` : liste blanche explicite. Aucun mot de passe.
- Sitemaps : index par langue, séparation produits / pages / éditorial. Exclure non publié, rupture de stock, pages filtrées.
- JSON-LD : `Organization`, `WebSite`+`SearchAction`, `BreadcrumbList`, `Product` avec `offers` (dont `eligibleQuantity` pour le MOQ). `aggregateRating` **uniquement** avec avis réellement vérifiés. `Product` doit refléter le contenu visible.
- Titres/meta uniques par page et par locale. H1 unique. Hiérarchie H2/H3. Images : AVIF/WebP, dimensions explicites, `alt` utile. Vrais 404, redirections contrôlées. **Pas de pages dupliquées par facette/tri/pagination.**
- Paliers de prix : une seule page produit indexable, paliers dans le contenu principal (pas en onglets JS), pas de page par palier. Le prix minimum affiché doit rester cohérent avec le prix réel.
- CWV : LCP ≤ 2,5 s · INP ≤ 200 ms · CLS ≤ 0,1 au p75.
- Contenu : clusters FR/EN/ZH autour de l'achat en gros depuis la Chine (MOQ/paliers, frais de douane par pays, contrôle qualité, délais, Incoterms DDP/DAP, devis B2B, codes HS).
- **Interdits marketing** : avis/comptes/badges de rareté inventés · urgency artificielle · dark patterns · masquage du coût total. Ne jamais promettre un classement.

---

## 7. Sécurité (spec §19)

### 7.1 Principes
Privilège minimal · défense en profondeur · secure by default · **modèle de menace avant la prod** · inventaire des données. OWASP ASVS niveau 2 minimum, niveau 3 sur paiement / rôles / exports.

### 7.2 AuthN / sessions / MFA
- Cookie de session : `HttpOnly`, `Secure`, `SameSite=Lax` (ou `Strict` sur mutation), `Path=/`, `__Host-` prefix.
- **Rotation de l'identifiant de session** au login et à l'élévation de privilèges. Expiration inactive + expiration absolue. Révocation multi-appareils. Changement de mot de passe → révocation.
- TOTP (RFC 6238) : secret chiffré (envelope key, KMS), fenêtre de tolérance bornée, **compteur anti-rejeu**. **MFA obligatoire** admin/finance/compliance, fortement recommandé B2B.
- Codes de récupération : Argon2id, usage unique, affichés une seule fois.
- Ré-authentification pour : changement d'e-mail/mot de passe, désactivation MFA, remboursement sensible.
- Google Sign-In : validation **serveur** du jeton, `state` + `nonce`, redirect URIs exacts, aucun secret OAuth côté front, **pas de fusion de compte par similarité de nom**.
- CSRF : token synchronisé sur mutation + vérification `Origin` + `SameSite`. CORS : allowlist exacte, jamais `*` avec credentials.

### 7.3 Autorisation
RBAC serveur, 9 rôles : `super_admin`, `catalog_manager`, `order_manager`, `logistics_operator`, `finance_manager`, `support_agent`, `compliance_manager`, `marketing_manager`, `analyst_readonly`.
- Permission par **action + ressource**, `deny by default`.
- **Vérification de propriété sur la ressource** (anti-IDOR/BOLA) : filtre scoping obligatoire dans la requête, jamais en post-filtrage en mémoire.
- **Double contrôle (four-eyes)** pour les actions financières : table `AdminActionApproval`, approbation d'un second rôle, trace dans l'audit.
- Journal d'audit immuable : acteur, action, objet, horodatage, résultat, `correlation_id`. Événements : connexion admin, échec MFA, création/modification produit, changement de prix, changement de règle douanière, variation de stock, décision B2B, remboursement, annulation, changement de rôle, export de données, modification de config sensible. **Aucun secret dedans.**

### 7.4 Paiement
Aucune donnée de carte stockée (éléments/Checkout hébergé + tokenisation). Interdiction de logger le corps de requête de paiement. Vérification de signature webhook (HMAC + tolérance d'horodatage + anti-rejeu). Séparation initiation / confirmation. Idempotence. Réconciliation quotidienne. Remboursement : rôles finance/admin seulement, confirmé par le prestataire, **double remboursement impossible**. Procédure de preuve de chargeback. 3DS selon le risque.

### 7.5 Entrées, XSS, SQLi, SSRF, headers
Validation zod en entrée **et** en sortie. Requêtes paramétrées (ORM). React échappe par défaut : `dangerouslySetInnerHTML` interdit, `href` utilisateur assaini (protocole `http/https/mailto` uniquement), style dynamique via tokens uniquement. SSRF : allowlist d'hôtes pour les transporteurs, pas d'URL utilisateur, timeout, re-résolution DNS. CSP stricte (nonce/hash, pas de `unsafe-inline` script, `connect-src` restreint aux intégrations), HSTS, `X-Content-Type-Options`, `frame-ancestors`, `Referrer-Policy`, `Permissions-Policy`.

### 7.6 Uploads
MIME réel (magic bytes, pas l'extension), taille max, nom régénéré (UUID), stockage hors racine web, scan antivirus, jamais exécuté. Documents de vérification B2B : chiffrés, accès restreint, URL signée courte durée, **jamais d'URL publique**.

### 7.7 Rate limiting & secrets
Limites : login 5/15 min (IP+email) · signup 3/h · reset 3/h · MFA 5/10 min · recherche 60/min · API 600/min. Sliding window Redis, clé composite, `429` + `Retry-After`. Anti-énumération de comptes (réponse identique). Secrets : gestionnaire dédié (pas GitHub Secrets comme coffre applicatif), jamais en Git/build/logs/bundle, rotation + procédure d'urgence. Scan de secrets en pre-commit et en CI (gitleaks/trufflehog). En cas de fuite : **rotation d'abord**, purge d'historique ensuite.

### 7.8 Conformité
RGPD/UK GDPR (bases légales, registre, DPA, transferts, DSAR, rétention) · PIPL chinois (données clients CN) · obligations acheteur (CPSC, FTC, English Colée) · registre des sous-traitants. Points à faire valider par un juriste — **ne jamais s'auto-certifier**.

---

## 8. Qualité & tests (spec §24, §26)

- Critères d'acceptation → tests → revue sécurité ciblée → revue diff/régressions → corriger → documenter. **Pas de « terminé » sans preuve.**
- Framework réel du projet d'abord. E2E sur les parcours affectés, avec erreurs console et réseau — **un screenshot seul ne prouve rien**.
- Bug corrigé → test de non-régression.
- 5 suites E2E Playwright obligatoires : acheteur B2C (9 étapes) · acheteur B2B (9 étapes) · paiement (8 cas) · admin (11 cas) · sécurité (7 cas).
- Responsive/visuel : 8 viewports × 3 langues × 2 thèmes. Tests visuels de régression avec **snapshots approuvés après revue** (jamais auto-approuvés).
- A11y : axe-core + jest-axe en composant, @axe-core/playwright en E2E. Zéro violation critique/sérieuse sur les parcours critiques. Revue manuelle obligatoire.
- Concurrence : 2 clients achètent la dernière unité → une seule commande réussit. Idempotence : double soumission, double webhook, double remboursement.
- Charge : k6/Artillery sur catalogue, recherche, login, checkout, webhook. Seuils p95 et taux d'erreur.
- Mutation testing (Stryker) sur pricing/tax/order-state. Couverture : indicateur, **pas preuve**.
- Lint : ESLint flat strict + `jsx-a11y` + **interdiction d'import direct de la base depuis le front**. Prettier. Husky + lint-staged. commitlint.
- **PR > 800 lignes → à scinder.** 2 approbations avant merge ; 1 approbation + revue sécurité sur chemin sensible.

---

## 9. Workflow Git / GitHub

- Branches : `feat|fix|refactor|test|docs|chore/<slug>`. Cycle : branche → PR → revue → squash-merge → suppression.
- Commits **Conventional Commits 1.0**, scopes alignés sur modules métier (`catalog`, `pricing`, `cart`, `checkout`, `order`, `payment`, `tax`, `customs`, `shipping`, `auth`, `account`, `b2b`, `quote`, `admin`, `content`, `i18n`, `infra`). Titre ≤72 car, corps ligne ≤72 car. Footer : `BREAKING CHANGE`, `Co-Authored-By`.
- Jamais de commit mixte. Jamais de commit sur `main`. Jamais de force-push. Jamais de merge commit sur `main` (squash-merge).
- PR : liée à une issue · checklist · preuves (tests, captures) · description en français. **PR laissée ouverte, jamais mergée sans accord explicite.**
- `main` protégé : required status checks, require review, dismiss stale, block force push, block deletion, signed commits, branches up to date.
- Dependabot (npm + actions) · code scanning (CodeQL/semgrep) · dependency review · secret scanning + push protection · `GITHUB_TOKEN` permissions minimales · actions pinnées par SHA · GitHub Environments avec secrets d'environnement et required reviewers en production.
- Secrets : scopes repository / environment / organization. **GitHub Secrets n'est pas le coffre applicatif final.**

---

## 10. Skills à invoiter (auto — sans attendre `/nom`)

| Domaine | Skills |
|---|---|
| Systématique | `caveman` (actif) · `brainstorming` (avant toute feature) · `writing-plans` / `executing-plans` · `test-driven-development` · `verification-before-completion` · `requesting-code-review` · `finishing-a-development-branch` · `using-git-worktrees` · `dispatching-parallel-agents` · `subagent-driven-development` · `surgical-patch` · `safe-refactor` · `caveman-commit` · `caveman-review` · `code-review` · `security-review` |
| Design/UI | `frontend-design` · `taste-skill` · `ui-ux-pro-max` · `web-design-guidelines` · `brand-guidelines` · `theme-factory` · `emil-design-eng` · `apple-design` · `composition-patterns` · `react-best-practices` · `pick-ui-library` · `prototype` · `motion-design` · `find-animation-opportunities` · `improve-animations` |
| SEO / contenu | `seo-audit` · `schema` · `ai-seo` · `site-architecture` · `content-strategy` · `copywriting` · `copy-editing` · `launch` · `analytics` |
| Dev / platform | `api-platform-engineer` · `database-optimizer` · `database-reliability-engineer` · `payments-engineering` · `devops` · `migration` · `lean-build` |
| QA / perf / a11y | `test-driven-development` · `webapp-testing` · `a11y` · `wcag` · `screenshot` · `image` |

**Agents** : `Explore`, `Plan`, `general-purpose`, `Code Reviewer`, `security-auditor`, `qa-e2e`, `seo-auditor`, `ui-ux-checker`, `Software Architect`, `Database Optimizer`, `DevOps Automator`, `Senior SecOps Engineer`, `Test Automation Engineer`, `Payments & Billing Engineer`, `UI Designer`, `SEO Specialist`, `Legal Compliance Checker`, `Tax Strategist`, `Reality Checker`, `Evidence Collector`, `Senior Project Manager`.

---

## 11. Checklist avant de dire « terminé »

- [ ] Critères d'acceptation du spec couverts, pas approximés
- [ ] Tests ajoutés (unit/int/E2E selon le risque) et **passants**, sortie collée
- [ ] Revue sécurité ciblée faite (authz, injection, XSS, secrets, uploads, rate limit)
- [ ] Diff relu : pas de debug, pas de TODO, pas de secret, pas de code mort
- [ ] Les 3 langues à jour pour toute nouvelle chaîne ; les 2 thèmes pour tout nouveau composant
- [ ] États loading/vide/erreur implémentés
- [ ] A11y : clavier + focus + contraste + annonces ; Revue manuelle si parcours critique
- [ ] Perf : pas de régression mesurable (bundle, requêtes, images)
- [ ] Migrations testées sur base de test ; destructive = accord + restauration prouvée
- [ ] SEO si page publique : title/meta/canonical/hreflang/JSON-LD/robots vérifiés
- [ ] Docs mises à jour (README, runbook, .env en **noms de variables uniquement**)
- [ ] Commits conventionnels, branche correcte, **pas** sur `main`
- [ ] PR ouverte avec preuves — **jamais mergée sans accord explicite**

---

## 12. Ce qui est à faire valider par un humain (jamais auto-décidé)

Entité juridique et pays d'établissement · éligibilité au prestataire de paiement · devise de référence et règles FX · pays prioritaires et pays interdits · Incoterms et modèle de paiement des taxes (DDP vs DAP) · catégories produits réglementées et codes HS · politique B2B et B2C (retours, MOQ, délais) · exigences de facturation locales · région d'hébergement et transferts de données · SLA / RPO / RTO · outils analytics et consentement · tout contenu juridique,douanier ou fiscal publié.
