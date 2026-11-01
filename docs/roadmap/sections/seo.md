## SEO, contenu et marketing technique

Périmètre : visibilité publique de Fanours (spec §17.1, §17.2, §17.3), cohérence technique des pages indexables, garde-fous marketing. S'appliquent aussi §4 (langues/devises), §6 (pages publiques), §7 (catalogue et paliers), §16.10 (CMS/SEO/marketing en admin), §23.1 (performance), §24.4 (tests multi-langue), §26 (critères d'acceptation).

Trois principes non négociables : **(1)** langue (URL, indexée) ≠ pays de destination (cookie, non indexé) ≠ devise (affichage, non indexée) — le spec §17.2 interdit de les confondre ; **(2)** aucune promesse de classement, le SEO est une hypothèse mesurée ; **(3)** aucune donnée inventée (avis, compteurs, badges, certifications, délais, prix) — §16.10, §18.3.

---

### 17.1 Architecture des URLs et i18n SEO

**Schéma d'URL** — locales exactes `fr`, `en`, `zh-cn` (minuscules, jamais de suffixe pays).

| Type | Format | Exemple |
|---|---|---|
| Accueil | `/{locale}/` | `/fr/`, `/en/`, `/zh-cn/` |
| Catégorie | `/{locale}/categorie/{slug-cat}/{slug-sous-cat}/` | `/fr/categorie/cuisine/gobelets-isothermes/` |
| Fiche produit | `/{locale}/produits/{slug-produit}/` | `/fr/produits/gobelet-isotherme-500-ml/` |
| Page éditoriale | `/{locale}/guides/{slug}/` | `/fr/guides/acheter-en-gros-depuis-la-chine/` |
| FAQ / aide | `/{locale}/aide/{slug}/` | `/fr/aide/moq-et-paliers-de-prix/` |
| Politique | `/{locale}/politiques/{slug}/` | `/fr/politiques/livraison/` |
| Pays (page éditoriale) | `/{locale}/livraison/{pays}/` | `/fr/livraison/france/` |
| Racine | `/` | 302 vers la locale détectée |

- Slugs : ASCII minuscules, tirets, 3-60 caractères, sans date, sans ID, sans paramètre marketing. Un slug est **définitif** après publication ; tout changement déclenche une 301 unique et tracée (§17.1 « redirections contrôlées »).
- Slug par locale stocké dans `ProductTranslation.slug` / `CategoryTranslation` (§20.1) : FR = slug français, EN = anglais, ZH-CN = pinyin ASCII. Le H1 et le contenu restent dans la langue de la locale ; le slug n'est qu'un identifiant d'URL.
- Pages authentifiées (compte, commandes, devis, factures) existantes **sous les 3 préfixes** pour éviter toute négociation de langue en zone privée.

**Les trois dimensions, jamais confondues**

| Dimension | Où vit-elle | Indexée ? | Effet |
|---|---|---|---|
| Langue | Segment d'URL `/fr/ /en/ /zh-cn/` | Oui (hreflang + sitemap par locale) | Contenu, H1, métadonnées, `lang` de la page |
| Pays de destination | Cookie `fanours_country` + champ checkout (§9.2) | Non | Transport, taxes, droits estimés, Incoterm, moyens de paiement |
| Devise | Cookie `fanours_currency` (affichage) + devise de transaction (§4.3) | Non | Affichage des montants, mention « indicatif » si converti |

Conséquences obligatoires : le pays de destination **n'apparaît jamais dans l'URL** d'une fiche produit indexable (une page produit reste identique quelle que soit la destination simulée — sinon dédoublement de contenu quasi identique et cloaky) ; le changement de pays ou de devise **ne provoque jamais de redirection** ni de modification d'URL, il se produit côté client avec rechargement des estimations (§11.4) ; seules les pages pays éditoriales portent un pays dans l'URL, à contenu réel rédigé et traduit, pas généré ; la devise de transaction et la devise d'affichage restent distinctes partout (§4.3) et `priceCurrency` porte la devise de référence du catalogue, jamais celle choisie par l'utilisateur.

**Canonical** — absolu, auto-référentiel, sur chaque page indexable (`https://fanours.com/{locale}/{chemin}`) ; jamais de canonical vers une autre locale (chaque locale a le sien, articulé par hreflang) ; le canonical retire les paramètres de tracking et de filtre (`?utm_*`, `?page=`, `?qty=`, `?sort=`, `?filter=`) ; pagination `?page=N` → canonical vers `?page=1` + `noindex,follow` ; doublon de fiche (variante d'attribut non différenciante) → canonical vers la variante par défaut ; migration de slug → 301 unique, ancien slug hors sitemap, canonical sur la destination, journal de redirection en admin (§16.10).

**hreflang** — réciprocité obligatoire : chaque URL déclare **toutes** les variantes de son groupe, y compris elle-même ; sans réciprocité, Google ignore l'ensemble du set.

| Type de page | x-default | fr | en | zh-cn |
|---|---|---|---|---|
| Accueil | `/en/` | `/fr/` | `/en/` | `/zh-cn/` |
| Catégorie | `/en/category/{slug}/` | `/fr/categorie/{slug}/` | `/en/category/{slug}/` | `/zh-cn/category/{slug}/` |
| Fiche produit | `/en/products/{slug}/` | `/fr/produits/{slug}/` | `/en/products/{slug}/` | `/zh-cn/products/{slug}/` |
| Page éditoriale | `/en/guides/{slug}/` | `/fr/guides/{slug}/` | `/en/guides/{slug}/` | `/zh-cn/guides/{slug}/` |
| FAQ / aide | `/en/help/{slug}/` | `/fr/aide/{slug}/` | `/en/help/{slug}/` | `/zh-cn/help/{slug}/` |
| Politique | `/en/policies/{slug}/` | `/fr/politiques/{slug}/` | `/en/policies/{slug}/` | `/zh-cn/policies/{slug}/` |
| Page pays | `/en/shipping/{country}/` | `/fr/livraison/{country}/` | `/en/shipping/{country}/` | `/zh-cn/shipping/{country}/` |

Implémentation : `x-default` pointe vers la variante **anglaise** (sauf contenu réservé au marché francophone → FR) ; `hreflang` décrit la **langue de la page**, jamais le pays de destination, aucune valeur `fr-FR` pour du contenu international ; `<html lang="fr">` / `lang="en"` / `lang="zh-CN"` en plus du hreflang ; `og:locale` = `fr_FR` / `en_US` / `zh_CN` + `og:locale:alternate` ; une locale dont la traduction **manque** est servie en `noindex,follow`, canonical vers l'URL réellement traduite, **exclue** du hreflang de cette locale (jamais de doublon indexable) ; les pages de politique doivent exister dans les 3 locales avant publication (politique monolingue = noindex + hors sitemap).

**Détection de langue — sans redirection forcée** (le spec §17.2 l'interdit explicitement). Priorité 1 : cookie `fanours_locale` (choix explicite, 12 mois). Priorité 2 : en-tête `Accept-Language`. Priorité 3 : on reste sur la page demandée.

À la racine `/` uniquement, **une seule fois** : cookie présent → 302 vers `/{cookie}/` ; sinon `Accept-Language` concordant → 302 vers `/{locale}/` avec switcher visible (§6.2.1) et mention « vous pouvez changer de pays et de langue à tout moment » ; sinon → 302 vers la locale par défaut. Règles : redirection **302** et jamais 301 (une détection automatique est temporaire, elle ne doit pas encoder une préférence dans l'index) ; **aucun** usage de l'IP pour rediriger — l'IP peut seulement *suggérer* un pays dans un bandeau non bloquant ; aucune boucle ; `Accept-Language` invalide = aucune erreur. Test E2E (§24.4) : `Accept-Language: zh-CN` → 302 vers `/zh-cn/`, puis `Accept-Language: fr` → **aucune** redirection (cookie posé).

---

### 17.2 robots.txt et directives d'indexation

**Contenu exact de `https://fanours.com/robots.txt`**

```
# Fanours — https://fanours.com
# Politique : https://fanours.com/fr/politiques/robots/
# Mise a jour : 2026-10-21
#
# AVERTISSEMENT : robots.txt controle l'EXPLORATION, pas l'ACCES.
# Toutes les routes listees ci-dessous sont en plus protegees cote serveur
# (authentification, RBAC, verification de propriete) — cf. spec 19.2, 21.2.
# Une page bloquee ici n'est de toute facon pas indexable.

User-agent: *
Allow: /robots.txt
Allow: /sitemap.xml
Allow: /sitemaps/
Allow: /assets/
Allow: /media/
Allow: /_next/static/
Allow: /fr/
Allow: /en/
Allow: /zh-cn/

# --- Espaces prives : les 3 locales (spec 6.1, 17.1) ---
Disallow: /fr/compte/
Disallow: /en/account/
Disallow: /zh-cn/account/
Disallow: /fr/panier/
Disallow: /en/cart/
Disallow: /zh-cn/cart/
Disallow: /fr/checkout/
Disallow: /en/checkout/
Disallow: /zh-cn/checkout/
Disallow: /fr/devis/
Disallow: /en/quotes/
Disallow: /zh-cn/quotes/
Disallow: /fr/commandes/
Disallow: /en/orders/
Disallow: /zh-cn/orders/
Disallow: /fr/facturation/
Disallow: /en/invoices/
Disallow: /zh-cn/invoices/
Disallow: /fr/suivi/
Disallow: /en/tracking/
Disallow: /zh-cn/tracking/
Disallow: /fr/retours/
Disallow: /en/returns/
Disallow: /zh-cn/returns/
Disallow: /fr/favoris/
Disallow: /en/wishlist/
Disallow: /zh-cn/wishlist/
Disallow: /fr/recherche/
Disallow: /en/search/
Disallow: /zh-cn/search/

# --- Recherche interne et parametres : aucune valeur d'index ---
Disallow: /fr/*?*
Disallow: /en/*?*
Disallow: /zh-cn/*?*
Disallow: /*?*

# --- Administration, API, systeme ---
Disallow: /admin/
Disallow: /api/
Disallow: /_next/
Disallow: /staging/
Disallow: /preview/
Disallow: /graphql/
Disallow: /internal/
Disallow: /.well-known/
Disallow: /sitemap_index.xml
# Note : "Allow: /_next/static/" est plus specifique que "Disallow: /_next/"
# et l'emporte ; les assets statiques restent donc explorables.

User-agent: GPTBot
Allow: /

User-agent: Googlebot
Allow: /

User-agent: Bingbot
Allow: /

Sitemap: https://fanours.com/sitemap.xml
```

Points de conception : `Disallow: /*?*` neutralise d'un bloc tous les URLs paramétrés (filtres, tris, pagination, `?qty=`) — aucune URL paramétrée n'est indexable (§17.2), le budget d'exploration n'est donc pas gaspillé ; contrepartie assumée : Google ne voit plus les signaux `rel=canonical`/`rel=next` portés par ces URLs, sans impact puisque ces listings sont non indexables. Les motifs `/fr/*?*`, `/en/*?*`, `/zh-cn/*?*` sont redondants mais rendent l'intention explicite à la relecture. `Allow: /fr/` etc. + `Disallow` privés = effet « liste blanche » attendu. Un `Disallow` seul **ne suffit pas** (Google peut afficher l'URL en « résultat sans description ») : chaque page privée porte aussi `<meta name="robots" content="noindex, nofollow, noarchive">`.

**Liste exhaustive des pages à `noindex`** — doivent porter `<meta name="robots" content="noindex, nofollow">` **et** être absentes du sitemap :

| Catégorie | Routes (par locale) | Réf. spec |
|---|---|---|
| Panier | `/{locale}/panier/`, `/{locale}/cart/` | §6.1, §9.1 |
| Checkout (toutes étapes) | `/{locale}/checkout/**` | §6.1, §9.2 |
| Connexion / inscription / MFA / reset | `/{locale}/connexion/`, `/{locale}/inscription/`, `/{locale}/mfa/`, `/{locale}/mot-de-passe/**` | §13.1, §13.3 |
| Compte client / dashboard | `/{locale}/compte/**`, `/{locale}/account/**` | §15, §6.1 |
| Devis B2B | `/{locale}/devis/**`, `/{locale}/quotes/**` | §8.3, §17.1 |
| Commandes et détail de commande | `/{locale}/commandes/**`, `/{locale}/orders/**` | §9.3, §17.1 |
| Factures, avoirs, documents commerciaux | `/{locale}/facturation/**`, `/{locale}/invoices/**` | §14.5, §17.1 |
| Suivi de commande | `/{locale}/suivi/**`, `/{locale}/tracking/**` | §6.1, §12.2 |
| Retours et litiges | `/{locale}/retours/**`, `/{locale}/returns/**` | §12.4, §12.5 |
| Favoris | `/{locale}/favoris/**` | — |
| Recherche interne et suggestions | `/{locale}/recherche/**`, `/{locale}/search/**` | §18.1 |
| Administration | `/admin/**` | §16 |
| API, webhooks, jobs | `/api/**`, `/webhooks/**` | §21.1 |
| Erreurs | `/404`, `/500`, `/maintenance`, `/403` | §6.1 |

**Pages à faible valeur** — non « noindex » par défaut mais **canonicalisées vers la version propre**, exclues du sitemap, sans maillage interne : facettes (`?couleur=rose&taille=m`) → canonical URL nue + `noindex,follow` ; tris (`?sort=prix-asc`) idem ; pagination (`?page=3`) → canonical `?page=1` + `noindex,follow` ; quantité/palier (`?qty=500`) → canonical URL nue + `noindex,follow` ; recherche (`?q=gobelet`) → `noindex,follow` + `Disallow` robots ; comparaison (`?compare=1,2,3`) → `noindex,follow` ; langue en query (`?lang=zh`) → 301 vers `/{locale}/` une seule fois ; devise (`?currency=EUR`) et pays (`?country=FR`) → canonical URL nue, jamais dans l'URL.

Règle de maillage : **aucun** lien interne (nav, footer, fil d'Ariane, maillage éditorial) vers une URL paramétrée ou une page privée. Seul lien vers une page privée : « Mes commandes » dans l'en-tête, `rel="nofollow"`.

---

### 17.3 Sitemaps

```
/sitemap.xml                                    → sitemapindex (racine, 1 seul)
  └── /sitemaps/sitemap-index.xml               → sitemapindex par langue
        ├── /sitemaps/sitemap-fr.xml            → sitemapindex FR
        │     ├── /sitemaps/sitemap-fr-products.xml     → urlset
        │     ├── /sitemaps/sitemap-fr-pages.xml        → urlset
        │     └── /sitemaps/sitemap-fr-editorial.xml    → urlset
        ├── /sitemaps/sitemap-en.xml            → sitemapindex EN (idem)
        └── /sitemaps/sitemap-zh-cn.xml         → sitemapindex ZH (idem)
```

Index racine unique déclaré en `robots.txt`. Hiérarchie à 3 niveaux : racine → index par langue → index de type → `urlset` ; chaque `sitemapindex` référence ses enfants par `<sitemap><loc>` + `<lastmod>`. Génération par job planifié (BullMQ, §5.5 du CLAUDE.md projet) : catalogue = quotidien, pages/éditorial = à chaque publication, incrémental. Limites Google : 50 000 URLs et 50 Mo (non compressé) par fichier ; au-delà, segmentation par tranche de dates ou sous-catégorie.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:xhtml="http://www.w3.org/1999/xhtml">
  <url>
    <loc>https://fanours.com/fr/produits/gobelet-isotherme-500-ml/</loc>
    <lastmod>2026-10-18T08:00:00+00:00</lastmod>
    <changefreq>weekly</changefreq>
    <priority>0.8</priority>
    <xhtml:link rel="alternate" hreflang="fr" href="https://fanours.com/fr/produits/gobelet-isotherme-500-ml/"/>
    <xhtml:link rel="alternate" hreflang="en" href="https://fanours.com/en/products/insulated-tumbler-500ml/"/>
    <xhtml:link rel="alternate" hreflang="zh" href="https://fanours.com/zh-cn/products/bao-wen-bei-500ml/"/>
    <xhtml:link rel="alternate" hreflang="x-default" href="https://fanours.com/en/products/insulated-tumbler-500ml/"/>
  </url>
</urlset>
```

Règles de contenu : uniquement des URL **canoniques absolues** en 200 (aucune URL paramétrée, redirection ou 404) ; liens hreflang `xhtml:link` répétés dans le sitemap pour rester synchronisés avec le HTML ; `changefreq` et `priority` sont des hints ignorés par Google, conservés comme indicateur interne admin, pas comme outil d'optimisation.

**Exclusions** — un produit est exclu du sitemap si l'un de ces critères est vrai : (1) statut ≠ `published` (brouillon, en revue, suspendu, archivé) §7.1 ; (2) stock disponible = 0 sans précommande activée §7.3 (la réapparition à chaque réassort crée du churn) ; (3) traduction manquante pour la locale du sitemap ; (4) champs obligatoires incomplets (nom, description, image principale, SKU, MOQ, ≥ 1 palier valide, origine, pays d'expédition) ; (5) prix sans devise, palier incohérent ou prix de référence non résolu §7.2 ; (6) `noindex` posé par l'opérateur ; (7) page en maintenance ou brouillon CMS ; (8) canonical pointant ailleurs (doublon) ; (9) toute page privée ou à faible valeur (§17.2). Pages éditoriales et pages pays : même logique, exclues tant que le contenu n'est pas publié **et** traduit dans la locale du sitemap.

**Règles de `lastmod`** — doit refléter une **modification de contenu réelle**, jamais l'horodatage technique d'un build. Produit : `max(mise à jour fiche, dernière mise à jour traduction, modification des spécifications, modification des paliers)`. Le **stock** ne modifie **jamais** `lastmod` (sinon ré-exploration several fois par jour) ; un changement de palier est throttlé à une mise à jour par 24 h. Page éditoriale : publication puis dernière révision éditoriale réelle. Catégorie : mise à jour de description ou ajout/suppression d'un produit publié. Page pays / politique : dernière validation métier ou juridique. Format : ISO 8601 avec fuseau, en UTC (§4.4).

---

### 17.4 Données structurées JSON-LD

JSON-LD injecté dans le HTML rendu, un seul bloc `@graph` par page, sans microdata, **jamais** de contenu structuré absent de la page visible. Exemples valides, à coller tels quels dans les constantes de l'app.

**Organization + WebSite** (toutes pages publiques)

```json
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "Organization",
      "@id": "https://fanours.com/#organization",
      "name": "Fanours",
      "url": "https://fanours.com/",
      "description": "Plateforme de vente en gros depuis la Chine : MOQ explicites, paliers de prix, devis professionnels et livraison internationale.",
      "logo": { "@type": "ImageObject", "url": "https://fanours.com/assets/logo-fanours-512.png", "width": 512, "height": 512 },
      "image": "https://fanours.com/assets/og-default-1200x630.jpg",
      "foundingDate": "2024-01-01",
      "address": { "@type": "PostalAddress", "streetAddress": "…", "addressLocality": "…", "addressRegion": "…", "postalCode": "…", "addressCountry": "CN" },
      "contactPoint": [
        { "@type": "ContactPoint", "contactType": "sales", "email": "sales@fanours.com", "telephone": "+86-…", "availableLanguage": ["fr-FR", "en", "zh-CN"],
          "areaServed": [
            { "@type": "Country", "name": "France" },
            { "@type": "Country", "name": "United Kingdom" },
            { "@type": "Country", "name": "United States" },
            { "@type": "Country", "name": "Canada" },
            { "@type": "Country", "name": "Australia" }
          ] },
        { "@type": "ContactPoint", "contactType": "customer support", "email": "support@fanours.com", "availableLanguage": ["fr-FR", "en", "zh-CN"] }
      ],
      "sameAs": [
        "https://www.linkedin.com/company/fanours",
        "https://www.instagram.com/fanours",
        "https://www.youtube.com/@fanours"
      ]
    },
    {
      "@type": "WebSite",
      "@id": "https://fanours.com/#website",
      "url": "https://fanours.com/",
      "name": "Fanours",
      "inLanguage": "fr-FR",
      "publisher": { "@id": "https://fanours.com/#organization" },
      "potentialAction": {
        "@type": "SearchAction",
        "target": { "@type": "EntryPoint", "urlTemplate": "https://fanours.com/fr/recherche?q={search_term_string}" },
        "query-input": "required name=search_term_string"
      }
    }
  ]
}
```

`sameAs` ne liste que des profils **réels et vérifiés** ; `areaServed` ne cite que des marchés effectivement desservis et activés (§11.7), jamais « World » ; `foundingDate` seulement si réelle ; pas de `numberOfEmployees` ni de note interne non vérifiée ; `inLanguage` suit la locale de la page (`en` sur la version EN).

**BreadcrumbList** (pages de profondeur > 1) — le fil d'Ariane visible doit reproduire exactement les mêmes libellés et les mêmes positions, sans saut ni niveau décoratif.

```json
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "@id": "https://fanours.com/fr/produits/gobelet-isotherme-500-ml/#breadcrumb",
  "itemListElement": [
    { "@type": "ListItem", "position": 1, "name": "Accueil", "item": "https://fanours.com/fr/" },
    { "@type": "ListItem", "position": 2, "name": "Cuisine", "item": "https://fanours.com/fr/categorie/cuisine/" },
    { "@type": "ListItem", "position": 3, "name": "Gobelets isothermes", "item": "https://fanours.com/fr/categorie/cuisine/gobelets-isothermes/" },
    { "@type": "ListItem", "position": 4, "name": "Gobelet isotherme 500 ml", "item": "https://fanours.com/fr/produits/gobelet-isotherme-500-ml/" }
  ]
}
```

**Product B2B complet** (fiche produit)

```json
{
  "@context": "https://schema.org",
  "@graph": [
    { "@type": "Organization", "@id": "https://fanours.com/#organization" },
    { "@type": "WebSite", "@id": "https://fanours.com/#website" },
    {
      "@type": "BreadcrumbList",
      "@id": "https://fanours.com/fr/produits/gobelet-isotherme-500-ml/#breadcrumb",
      "itemListElement": [
        { "@type": "ListItem", "position": 1, "name": "Accueil", "item": "https://fanours.com/fr/" },
        { "@type": "ListItem", "position": 2, "name": "Cuisine", "item": "https://fanours.com/fr/categorie/cuisine/" },
        { "@type": "ListItem", "position": 3, "name": "Gobelets isothermes", "item": "https://fanours.com/fr/categorie/cuisine/gobelets-isothermes/" },
        { "@type": "ListItem", "position": 4, "name": "Gobelet isotherme 500 ml", "item": "https://fanours.com/fr/produits/gobelet-isotherme-500-ml/" }
      ]
    },
    {
      "@type": "Product",
      "@id": "https://fanours.com/fr/produits/gobelet-isotherme-500-ml/#product",
      "name": "Gobelet isotherme 500 ml — édition export",
      "description": "Gobelet isotherme en acier inoxydable 304, 500 ml, garde la température 12 h. MOQ 50 unités, incrément 5, quatre paliers de prix, origine Chine, code HS 9617.00.",
      "sku": "FN-GOB-500-BLK",
      "mpn": "FN-GOB-500-BLK",
      "image": [
        "https://fanours.com/media/products/fn-gob-500-blk/01.avif",
        "https://fanours.com/media/products/fn-gob-500-blk/02.avif",
        "https://fanours.com/media/products/fn-gob-500-blk/03.avif"
      ],
      "brand": { "@type": "Brand", "name": "Fanours" },
      "manufacturer": { "@id": "https://fanours.com/#organization" },
      "category": "Cuisine > Accessoires de boisson",
      "color": "Noir mat",
      "material": "Acier inoxydable 304",
      "weight": { "@type": "QuantitativeValue", "value": 0.32, "unitCode": "KGM" },
      "additionalProperty": [
        { "@type": "PropertyValue", "name": "Capacité", "value": "500 ml" },
        { "@type": "PropertyValue", "name": "MOQ", "value": "50 unités" },
        { "@type": "PropertyValue", "name": "Incrément de commande", "value": "5 unités" },
        { "@type": "PropertyValue", "name": "Pays d'origine", "value": "Chine" },
        { "@type": "PropertyValue", "name": "Code HS", "value": "9617.00" }
      ],
      "offers": {
        "@type": "Offer",
        "@id": "https://fanours.com/fr/produits/gobelet-isotherme-500-ml/#offer",
        "url": "https://fanours.com/fr/produits/gobelet-isotherme-500-ml/",
        "price": "12.00",
        "priceCurrency": "EUR",
        "priceValidUntil": "2027-01-31",
        "itemCondition": "https://schema.org/NewCondition",
        "availability": "https://schema.org/InStock",
        "seller": { "@id": "https://fanours.com/#organization" },
        "eligibleQuantity": { "@type": "QuantitativeValue", "minValue": 50, "maxValue": 100000, "unitCode": "C62" },
        "priceSpecification": [
          { "@type": "UnitPriceSpecification", "price": "12.00", "priceCurrency": "EUR", "minQuantity": 50, "maxQuantity": 99, "valueAddedTaxIncluded": false, "unitCode": "C62" },
          { "@type": "UnitPriceSpecification", "price": "10.50", "priceCurrency": "EUR", "minQuantity": 100, "maxQuantity": 249, "valueAddedTaxIncluded": false, "unitCode": "C62" },
          { "@type": "UnitPriceSpecification", "price": "9.50", "priceCurrency": "EUR", "minQuantity": 250, "maxQuantity": 499, "valueAddedTaxIncluded": false, "unitCode": "C62" },
          { "@type": "UnitPriceSpecification", "price": "9.00", "priceCurrency": "EUR", "minQuantity": 500, "unitCode": "C62" }
        ],
        "shippingDetails": {
          "@type": "OfferShippingDetails",
          "shippingDestination": { "@type": "DefinedRegion", "addressCountry": "FR" },
          "deliveryTime": {
            "@type": "ShippingDeliveryTime",
            "handlingTime": { "@type": "QuantitativeValue", "minValue": 3, "maxValue": 7, "unitCode": "DAY" },
            "transitTime": { "@type": "QuantitativeValue", "minValue": 18, "maxValue": 35, "unitCode": "DAY" }
          },
          "shippingRate": {
            "@type": "ShippingRate",
            "name": "Standard international",
            "value": "24.90",
            "currency": "EUR",
            "shippingDestination": { "@type": "DefinedRegion", "addressCountry": "FR" }
          }
        },
        "hasMerchantReturnPolicy": {
          "@type": "MerchantReturnPolicy",
          "applicableCountry": [
            { "@type": "Country", "name": "France" },
            { "@type": "Country", "name": "United Kingdom" },
            { "@type": "Country", "name": "United States" }
          ],
          "returnPolicyCategory": "https://schema.org/MerchantReturnFiniteReturnWindow",
          "merchantReturnDays": 30,
          "returnMethod": "https://schema.org/ReturnByMail",
          "returnFees": "https://schema.org/ReturnFeesCustomerResponsibility"
        }
      }
    }
  ]
}
```

Pièges critiques : **`price` = prix du palier au MOQ** (ici 12,00 € pour 50 unités), **exactement** le prix affiché en haut de la page — Google compare la valeur structurée au prix visible, toute divergence est un signal de structured data non fiable ; le prix le plus bas de la grille (9,00 €) ne doit **jamais** devenir le `price` principal. **`priceSpecification`** porte la grille complète en `UnitPriceSpecification` (mécanisme prévu pour les prix dégressifs). **`eligibleQuantity`** (`QuantitativeValue`, `unitCode: "C62"`) porte le MOQ : sans lui, l'acheteur B2B lit un prix unitaire hors de portée. **`priceValidUntil`** = date **réelle** d'expiration du palier (`PriceTier.valid_to`, §20.1) ; si le palier n'a pas de fin, **omettre** la propriété (une date inventée invalide l'offre). **`availability`** synchronisé avec le stock réel §7.3 : `InStock` / `OutOfStock` / `PreOrder` (produit en rupture = hors sitemap mais fiche accessible en `OutOfStock`). **`itemCondition`** : `NewCondition` (vente neuve par défaut). **`seller`** : Fanours, pas un tiers. **`AggregateOffer`** possible (`lowPrice`/`highPrice`/`offerCount`) mais **déconseillé par défaut en B2B** : le `lowPrice` n'est pas commandable en l'état et se prête à la lecture trompeuse ; décision à documenter en admin et appliquée de façon homogène. **`shippingDetails`** : le type concret est `OfferShippingDetails` (sous-type de `ShippingDetails` depuis la révision 2023 de schema.org) ; `deliveryTime` distingue le délai de préparation du délai de transport (§6.4). **`hasMerchantReturnPolicy`** : ne déclarer que ce qui est contractuellement réel (§12.4) — pas de « retour gratuit 30 jours » non appliqué ; alternative : déclaration au niveau `Organization` si la politique est uniforme. **`aggregateRating`** : voir §17.9, jamais ajouté par défaut. `priceCurrency` = devise de référence du catalogue (§4.3), jamais celle choisie par l'utilisateur.

**Bloc `aggregateRating`** (à n'ajouter que sous conditions)

```json
{ "@type": "AggregateRating", "ratingValue": "4.6", "reviewCount": 23, "bestRating": "5", "worstRating": "1" }
```

Conditions cumulatives : au moins un avis **vérifié** existe ; la note et le nombre affichés sur la page sont **identiques** au JSON ; la note est calculée côté serveur à partir des seuls avis publiés et non modérés ; l'avis est visible sur la page. Sinon : **ne pas inclure la propriété** (pas de propriété vide, pas de valeur par défaut). Ne jamais poser `aggregateRating` sur `Organization` : c'est un avis auto-évaluatif de l'entreprise, exclu des résultats enrichis par Google.

**Autres types à prévoir** : `FAQPage` sur la page FAQ/aide (information utile : depuis août 2023, Google n'accorde plus de résultat enrichi FAQ aux sites non institutionnels ; le balisage reste valide et sert aux moteurs IA ; le contenu FAQ doit rester visible et rédigé) ; `Article` / `BlogPosting` sur les pages éditoriales (`headline`, `datePublished`, `dateModified`, `author` réel, `publisher`, `inLanguage`) ; `ItemList` sur les pages catégorie avec la **liste visible** des produits, jamais un balisage fantôme de tout le catalogue.

---

### 17.5 On-page par type de page

**Fiche produit** — `title` = `{Nom produit} + {qualificatif B2B} — {promesse} | Fanours` (FR 50-62 car · EN 50-60 car · ZH ≤ 70 caractères CJK) ; `meta description` = {MOQ} + {prix dégressifs} + {délai} + {frais d'import} + {CTA} (FR 145-160 car · EN 130-160 car · ZH 40-70 car CJK) ; H1 unique ≤ 70 car = nom commercial + attribut distinctif ; H2 : Paliers de prix et MOQ · Caractéristiques · Expédition et douane · Conditionnement et incrément · Documents et conformité · Frais d'importation · Questions fréquentes (5 à 7 H2) ; H3 : dimensions, matériaux, délais par destination, options de paiement (2-4 par H2, jamais plus de 3 niveaux) ; slug `{nom}-{attribut-clé}` ASCII par locale ≤ 60 car ; densité : description longue 150-250 mots + tableau de spécifications + tableau de paliers + FAQ courte, ≥ 400 mots de contenu original ; maillage : → catégorie parente (ancre = nom de catégorie), → 2 guides du cluster §17.8, → 1 page pays si applicable, → produits comparables (ancre = nom produit), 4-8 liens contextuels ; CTA « Ajouter au panier » (si MOQ atteint) + « Demander un devis » (B2B) + « Contacter l'équipe », toujours visible sans scroll en mobile.

**Accueil** — `title` = `Achat en gros depuis la Chine — MOQ, prix dégressifs | Fanours` (≤ 60 car) ; meta = promesse B2B (gros volumes, MOQ clair, délais et frais d'import explicités, 150-160 car) ; H1 = « Un peu de rose. Beaucoup de style. Je craque. » + sous-titre B2B en H2 ; H2 : Catégories · Produits populaires · Prix dégressifs (exemple transparent) · Livraison internationale et frais d'import · Engagements vérifiables · Guide d'achat grossiste en 4 étapes · Avis authentiques · FAQ (7-9 H2) ; maillage → 4-6 catégories + 2-3 guides + page pays si détecté (8-12 liens) ; CTA « Explorer le catalogue » (primaire) + « Acheter en gros » (secondaire, §6.2.3) ; **interdit** : tout avis, compteur, badge ou urgence non vérifié (§6.2.8, §6.2.10).

**Catégorie** — `title` = `{Catégorie} en gros — MOQ et prix dégressifs | Fanours` (50-60 car) ; meta = contenu + MOQ médiane + livraison internationale (145-160 car) ; H1 = nom de catégorie (ou sous-catégorie) ; H2 : introduction de catégorie (80-150 mots utiles) · Guide d'achat de la catégorie · FAQ catégorie (3-4 H2) ; maillage → sous-catégories (ancre = nom) + guide de cluster + 2 catégories sœurs (6-10 liens) ; grille produits **puis** un bloc de texte de 150-300 mots original **sous** la grille (pas seulement au-dessus) ; pagination `?page=N` → canonical page 1 + `noindex,follow`.

**Page éditoriale / guide** — `title` = `{Sujet} : guide complet {année} | Fanours` (50-60 car) ; meta = 2 bénéfices concrets + 1 élément de preuve chiffré réel (145-160 car) ; H1 = question ou promesse éditoriale ; plan en 6-9 H2 avec sous-parties et réponses PAA en H3 (2 niveaux max) ; longueur 900-1 800 mots, ≥ 800 ; auteur nommé + « relu par » + date de publication et de mise à jour ; 3-8 liens vers sources officielles (douane, transporteur, Incoterms) ; maillage → 2-3 fiches produit du cluster + 1 autre guide + page pays (5-8 liens) ; CTA bandeau discret « Demander un devis » / « Voir le catalogue ».

**FAQ / aide** — `title` = `{Sujet} : questions fréquentes | Fanours` ; H1 = sujet ; 10-20 questions en H2 avec réponse de 40-90 mots en première phrase (format extractible) ; balisage `FAQPage` conforme au contenu visible ; maillage vers 2-3 pages produit et 1 guide.

**Politiques** (livraison, retours, confidentialité, cookies, CGV B2B/B2C, taxes et douane) — `title` = `{Politique} — {pays ou marché} | Fanours` si la politique diffère par marché, sinon `{Politique} | Fanours` ; versions FR/EN/ZH obligatoires avant indexation ; H1 = titre de la politique ; sections en H2 ; **jamais** de contenu masqué ni de clause inégale entre versions linguistiques ; date de dernière révision affichée.

**Page pays** — URL `/fr/livraison/{pays}/` ; `title` = `Livraison en {pays} depuis la Chine — délais, douane, Incoterms` ; H1 = `Livraison Fanours vers {pays}` ; contenu réellement différent par pays (transporteurs, Incoterm, restrictions, délais, documents requis) — sinon la page n'est pas publiée ; aucune donnée de droits ou de seuil non sourcée (§11.2, §11.7).

**Exemples rédigés — fiche produit (FR / EN / ZH)**

*Titre* — FR (61 car.) : `Gobelet isotherme 500 ml en gros — prix dégressifs | Fanours` · EN (60 car.) : `Wholesale 500ml insulated tumbler — tiered pricing | Fanours` · ZH (28 car. CJK) : `500ml 保温杯批发 · 阶梯价格 | Fanours`

*Meta description* — FR (154 car.) : `Commandez par lots dès 50 unités. Prix dégressifs, MOQ explicite, délais d'expédition depuis la Chine et estimation des frais d'import. Devis B2B sous 24 h.` · EN (137 car.) : `Order from 50 units with tiered pricing, a clear MOQ, shipping times from China and estimated import costs. B2B quote within 24 hours.` · ZH (46 car. CJK) : `50件起订，阶梯价格，MOQ清晰透明，中国发货时效与进口费用预估，24小时内提供B2B报价。`

Règle de cohérence : `title` et `meta description` doivent refléter des faits **présents sur la page**. Si le MOQ de la fiche est 100 et non 50, la meta dit 100. Un title qui promet un palier inexistant est une pratique trompeuse, pas une optimisation.

**Unicité des mots-clés primaires (anti-cannibalisation)** — une page = un rôle = un mot-clé primaire. Un audit croisé (GSC, dimensions `page` + `query`) est lancé **avant** toute modification de title, H1 ou meta ; en son absence, on raisonne sur le sitemap + le plan de contenu (méthode sans GSC).

| Mot-clé primaire | URL propriétaire | Pages qui ne doivent PAS le cibler |
|---|---|---|
| `achat en gros en Chine` | `/fr/` | Guides, catégories, pages pays |
| `MOQ et paliers de prix` | `/fr/guides/moq-et-paliers-de-prix/` | Fiches produit, accueil |
| `frais d'importation et douane` | `/fr/guides/frais-importation-douane/` | Pages pays, catégories |
| `Incoterms DDP vs DAP` | `/fr/guides/ddp-vs-dap/` | Accueil, pages pays |
| `frais de douane en France` | `/fr/livraison/france/` | Guide générique, accueil |
| `délais de livraison depuis la Chine` | `/fr/guides/delais-expedition-chine/` | Catégories, accueil |
| `contrôle qualité / inspection` | `/fr/guides/controle-qualite/` | Fiches produit |
| `devis B2B / paiement professionnel` | `/fr/devis/` + `/fr/guides/paiement-b2b/` | Accueil, catégories |
| `codes HS` | `/fr/guides/codes-hs/` | Fiches produit |
| `catégorie X` | `/fr/categorie/x/` | Accueil, guides, pages pays |

---

### 17.6 CWV et performance SEO

Cibles au p75, données de terrain quand disponibles (§23.1) : **LCP ≤ 2,5 s · INP ≤ 200 ms · CLS ≤ 0,1**.

**Budgets par route (données compressées)**

| Route | JS first-load | JS total | CSS | Images | Polices |
|---|---|---|---|---|---|
| Accueil `/{locale}/` | ≤ 170 Ko | ≤ 280 Ko | ≤ 35 Ko | ≤ 1 200 Ko (LCP ≤ 150 Ko) | ≤ 120 Ko |
| Catégorie | ≤ 160 Ko | ≤ 260 Ko | ≤ 35 Ko | ≤ 900 Ko | ≤ 120 Ko |
| Fiche produit | ≤ 190 Ko | ≤ 300 Ko | ≤ 35 Ko | ≤ 1 000 Ko | ≤ 120 Ko |
| Éditorial / guide | ≤ 120 Ko | ≤ 200 Ko | ≤ 30 Ko | ≤ 700 Ko | ≤ 120 Ko |
| FAQ / politique | ≤ 100 Ko | ≤ 160 Ko | ≤ 25 Ko | ≤ 400 Ko | ≤ 90 Ko |
| Checkout (hors SEO) | ≤ 220 Ko | ≤ 320 Ko | ≤ 35 Ko | ≤ 400 Ko | ≤ 90 Ko |

Polices : 2 familles × 2 graisses maximum préchargées, sous-ensemble `latin` + `latin-ext` ; CJK (`Noto Sans SC`) **uniquement** sur les pages `zh-cn`, sous-ensemble limité aux caractères réellement rendus, jamais en global. Images : LCP ≤ 150 Ko en AVIF, vignette de grille ≤ 45 Ko, `srcset` + `sizes` sur 4 breakpoints, dimensions **explicites** sur chaque `<img>`. Scripts tiers : **2 maximum** par page publique, ≤ 50 Ko au total, `lazyOnload` ou `afterInteractive`, jamais bloquants ; tout ajout de script tiers est audité avant intégration (risque direct de dégradation CWV). Pages 404 et d'erreur : aucun budget SEO (noindex).

**LCP** — pages publiques rendues côté serveur (SSG catalogue et éditorial, ISR pages pays et politiques, §23.1) ; image LCP en `<Image priority>` + `fetchpriority="high"` + `<link rel="preload" as="image" imagesrcset imagesizes>`, **jamais** `loading="lazy"`, AVIF puis WebP puis fallback ; texte LCP (titre de catégorie) dans le HTML, pas hydraté tardivement ; `preconnect` vers le CDN d'images, l'origine des polices et l'API de recherche (2-3 origines max).

**INP** — composants serveur par défaut, l'hydratation du catalogue ne porte que les interactions (filtres, quantité, ajout panier) ; aucun carousel, moteur de graphiques ou lecteur de PDF embarqué sur les pages publiques ; `content-visibility: auto` + `contain-intrinsic-size` sur les sections longues d'un guide ; analytics chargé sur `requestIdleCallback` après consentement, jamais de travail long synchrone au clic ; toute interaction (recalcul de palier, changement de quantité) sous 200 ms, calculs de prix côté serveur (§5.4 du CLAUDE.md projet).

**CLS** — `width`/`height` ou `aspect-ratio` sur **toute** image ; conteneur réservé pour le bloc de prix (le prix change de largeur selon devise et langue) ; hauteur réservée pour bannières et emplacements promo (la bannière ne pousse jamais le contenu) ; polices `font-display: swap` + fallback avec `size-adjust` / `ascent-override` pour que la substitution ne décale pas la ligne (§3.2) ; skeletons dimensionnés à l'identique de la structure finale (§3.7) ; bandeau de consentement en **overlay**, jamais inséré en tête de flux ; aucun contenu injecté au-dessus d'un contenu existant sans réservation d'espace.

**Mesure** — données de **terrain** : CrUX (API CrUX) par locale et par appareil, seule mesure alignée sur les CWV, cible p75 Vert sur les 3 locales ; **laboratoire** : PageSpeed Insights / Lighthouse en complément (diagnostic, pas verdict) ; **RUM** : `web-vitals` envoyé vers un endpoint Fanours (pas un tiers), attribution par locale, appareil et type de route ; **CI** : budgets Lighthouse CI bloquants sur les routes publiques, un dépassement bloque la release (pipeline §25.2).

---

### 17.7 Tables des prix B2B et SEO

Le prix dégressif (§7.2) est le cœur de la proposition B2B et la principale source de pièges.

**Architecture** — **une seule page produit indexable** par produit et par locale, le tableau de paliers vit sur cette page ; **aucune page par palier** (`/produit/tier-3/`, `/gobelet-500-unites/`) ; **aucune page par quantité** (`?qty=500` → canonical URL nue + `noindex,follow`) ; **aucun onglet JS** pour la grille de prix (tableau dans le HTML rendu, présent sans JavaScript, visible sans clic) ; pas de `details/summary` pour la grille (contenu masqué par défaut = contenu potentiellement non indexé) — réservé aux informations secondaires (variantes, notice d'entretien) ; un seul `Product` par page, pas d'`AggregateOffer` multi-produits sur une page n'affichant qu'un produit.

**Rendu du tableau**

```html
<table>
  <caption>Prix dégressifs par volume — Gobelet isotherme 500 ml (prix unitaires, hors transport et droits)</caption>
  <thead>
    <tr>
      <th scope="col">Quantité commandée</th>
      <th scope="col">Prix unitaire</th>
      <th scope="col">Montant de la ligne</th>
    </tr>
  </thead>
  <tbody>
    <tr><td>50 – 99 unités</td><td>12,00 €</td><td>à partir de 600,00 €</td></tr>
    <tr><td>100 – 249 unités</td><td>10,50 €</td><td>à partir de 1 050,00 €</td></tr>
    <tr><td>250 – 499 unités</td><td>9,50 €</td><td>à partir de 2 375,00 €</td></tr>
    <tr><td>500 unités et +</td><td>9,00 €</td><td>à partir de 4 500,00 €</td></tr>
  </tbody>
</table>
```

`<caption>` explicite (seul moyen d'expliquer au moteur de recherche de quoi parle le tableau) ; tableau **visible** dans le HTML initial, pas injecté après hydratation ; mention « hors transport et droits » obligatoire — le prix affiché ne doit jamais laisser croire qu'il inclut les frais d'import (§11.4, §9.2) ; valeurs affichées **exactement** égales aux `priceSpecification` du JSON-LD.

**Le piège « prix minimum affiché » vs prix réel** — risque principal d'une page B2B : afficher « dès 9,00 € » en accroche alors que le prix réel à l'achat est 12,00 €, ou l'inverse.

| Règle | Application |
|---|---|
| Prix mis en avant (carte de grille, title, meta) = **prix du palier MOQ** | Cohérence totale avec `Offer.price` |
| Le prix le plus bas n'est jamais présenté seul | « dès 9,00 € à partir de 500 unités », jamais « 9,00 € » |
| `lowPrice` / `AggregateOffer` évité par défaut | Un `lowPrice` non commandable en l'état = réclamation et signalement |
| Mention « indicatif » si le prix est converti | §4.3 : devise d'affichage ≠ devise de transaction |
| Inclusion/exclusion des frais toujours explicitée dans la même unité | « 12,00 € / unité, hors transport et droits » |
| Prix négocié jamais structuré | `CustomerPriceRule` est client-spécifique ; la fiche publique ne le reflète pas |

Un test automatisé compare, pour chaque fiche produit : premier prix du tableau, `Offer.price`, prix de la carte de grille, prix du `title`. Toute divergence = blocage de publication.

---

### 17.8 Contenu éditorial et stratégie

Promesse éditoriale : **« acheter en gros en Chine depuis l'international, sans mauvaise surprise »**. Les clusters couvrent les blocages réels de l'acheteur B2B : MOQ, coût rendu, délais, douane, Incoterms, contrôle qualité, paiement.

Règles de production : un article = un mot-clé primaire = une page, jamais deux clusters ; rédaction humaine, pas de traduction automatique non relue pour les sujets juridiques, douaniers ou de sécurité (§4.2) ; auteur nommé, date de publication et de mise à jour, sources officielles citées ; **traduction EN et ZH de chaque article publié** — un article publié seulement en FR reste `noindex` dans les autres locales, absent du hreflang de ces locales et du sitemap.

| # | Cluster | Intention | Type de page | Priorité |
|---|---|---|---|---|
| 1 | Acheter en gros depuis la Chine : méthode et critères | Info commerciale | Guide pilier + 3 articles | P0 |
| 2 | MOQ, paliers de prix, incréments de commande | Info / pré-achat | Guide + fiches produit | P0 |
| 3 | Frais d'importation, douane et TVA par pays | Info à forte intent | Guide + pages pays | P0 |
| 4 | Incoterms DDP vs DAP et responsabilités | Info / comparatif | Guide + article DDP | P0 |
| 5 | Contrôle qualité, inspection et certifications | Info | Guide + article | P1 |
| 6 | Délais d'expédition depuis la Chine et transporteurs | Info | Guide + article | P1 |
| 7 | Devis B2B, paiement et acompte | Transactionnelle | Page devis + guide | P1 |
| 8 | Codes HS, origine et dédouanement | Info | Guide | P2 |
| 9 | Conformité produit par marché (CPSC, CE…) | Info | Guide + pages pays | P2 |
| 10 | Rentabilité d'un achat en gros (coût rendu, ROI) | Info / calcul | Guide + article | P2 |

**Titres par cluster (FR / EN / ZH)**

1. *Acheter en gros depuis la Chine* — FR `Comment acheter en gros en Chine depuis l'étranger : la méthode en 7 étapes` · EN `How to buy wholesale in China from abroad: the 7-step method` · ZH `如何从海外在中国批发采购：七步完整指南`
2. *MOQ et paliers de prix* — FR `MOQ, paliers de prix et incréments : ce qu'un fournisseur chinois attend de vous` · EN `MOQ, price tiers and order increments: what a Chinese supplier expects from you` · ZH `MOQ、阶梯价格与起订倍数：中国供应商对采购商的真实要求`
3. *Frais d'importation et douane par pays* — FR `Frais de douane et TVA à l'importation : ce que vous payez vraiment à la réception` · EN `Import duties and VAT: what you really pay on delivery` · ZH `进口关税与增值税：到货时你真正需要支付的费用`
4. *Incoterms DDP vs DAP* — FR `DDP ou DAP : qui paie les droits de douane, et pourquoi le choix change tout` · EN `DDP vs DAP: who pays the import duties, and why the choice matters` · ZH `DDP 还是 DAP：关税由谁承担，为什么这个选择决定最终成本`
5. *Contrôle qualité et certifications* — FR `Contrôle qualité avant expédition : les 5 points à vérifier sur un lot` · EN `Pre-shipment quality inspection: the 5 checks that matter on a bulk order` · ZH `出货前质检：大货订单必须核对的五个关键点`
6. *Délais d'expédition et transporteurs* — FR `Délais d'expédition depuis la Chine : ce qui est réaliste, du port à votre porte` · EN `Shipping times from China: what is realistic from port to your door` · ZH `中国发货时效真实预期：从港口到收货地的完整时间线`
7. *Devis B2B et paiement* — FR `Demander un devis B2B à un fournisseur chinois : le modèle d'e-mail qui aboutit` · EN `Requesting a B2B quote from a Chinese supplier: the email template that works` · ZH `向中国供应商索取B2B报价：真正有效的邮件模板与流程`
8. *Codes HS et douane pratique* — FR `Code HS : comment le choisir, qui le valide et pourquoi il change vos droits` · EN `HS codes: how to choose them, who validates them, and why they change your duties` · ZH `HS 编码如何确定、谁来审核，以及它如何影响你的进口税费`
9. *Conformité produit par marché* — FR `Vendre en Europe, aux États-Unis ou au Canada : les exigences produit qui bloquent` · EN `Selling in Europe, the US or Canada: the product requirements that block shipments` · ZH `销往欧美与加拿大：哪些产品合规要求会导致清关被扣`
10. *Rentabilité d'un achat en gros* — FR `Calculer le coût rendu d'un lot : la méthode pour vérifier qu'une commande est rentable` · EN `Calculating landed cost for a bulk order: how to check a purchase is actually profitable` · ZH `大货订单到岸成本计算法：如何验证这笔采购真正划算`

**Maillage** — chaque article du cluster 1 pointe vers les clusters 2, 3, 6 et 7 (les quatre blocages du parcours d'achat) ; chaque fiche produit des clusters 2 et 3 pointe vers l'article du cluster avec une ancre descriptive (jamais « en savoir plus ») ; le guide pilier (cluster 1) est le hub, il reçoit les liens de tous les autres et les redistribue ; le hub est la seule page visant le mot-clé générique du cluster, les sous-articles visent des modificateurs précis (« MOQ », « DDP », « codes HS »).

---

### 17.9 Avis clients et signaux de confiance

**Interdits absolus (§16.10, §6.2.10, §17.3)** — aucun avis, témoignage, note, compteur ou badge de rareté **inventé**, simulé, importé d'un autre site ou généré par IA ; aucun « X personnes regardent ce produit », « 37 clients ont commandé cette semaine », « Plus que 3 en stock » si la donnée n'est pas calculée en temps réel sur des données réelles ; aucun compteur d'urgence (« plus que 2 aujourd'hui ») ni faux compte à rebours ; aucun avis d'un « expert » non identifié, aucun avis acheté masqué comme organique ; aucun avis sur un produit non livré, aucun avis commandité non signalé.

**Collecte des avis vérifiés** — conditions cumulatives : (1) commande au statut `delivered` ou `completed` (§9.3) ; (2) avis rattaché à une ligne de commande identifiée (`OrderLine`) et un produit précis ; (3) un avis par ligne, pas de réavis ; (4) soumission via compte authentifié **ou** lien signé à usage unique envoyé après livraison, valide 30 jours ; (5) modération **obligatoire** avant publication (file de modération admin §16.10, statut `pending` → `published`/`rejected`, motif de rejet conservé et journalisé) ; (6) données conservées : `order_id`, `order_line_id`, `buyer_id`, `product_id`, `rating`, `title`, `body`, `date`, `verified`, `moderation_state`, `locale`.

Publication : nom de l'auteur masqué par défaut (« A. B. **** »), nom complet affiché seulement si l'acheteur l'a explicitement autorisé ; pas de publication automatique (y compris pour les avis 5 étoiles) ; avis rejeté conservé en base et journalisé mais invisible ; le signalement d'un avis abusif par un utilisateur déclenche une file de modération, taux de signalement suivi.

**Balisage `Review` / `aggregateRating`** — `aggregateRating` sur la fiche produit autorisé **uniquement** si : avis vérifiés existants, note et nombre affichés sur la page **identiques** au JSON, calcul fait côté serveur à partir des seuls avis publiés, note visible sans JavaScript. Nombre minimal recommandé : **3 avis vérifiés** avant d'afficher une note globale (en dessous : avis sans note, plus honnête et sans signal de faible qualité). `reviewCount` = avis **réellement publiés** (pas soumis) ; `ratingValue` = moyenne exacte au dixième, recalculée à chaque publication. `Review` : `reviewRating`, `author` (Personne ou `Organization` identifiée), `datePublished`, `reviewBody` (apparu tel quel sur la page), `itemReviewed` (l'ID du Product). Le balisage d'avis n'est **jamais** posé sur `Organization` (avis auto-évaluatif de l'entreprise, non éligible aux résultats enrichis et risqué en politique). Le `Product` reste exempt d'`aggregateRating` tant que la première condition n'est pas remplie : pas de propriété vide, pas de valeur par défaut.

**Signaux de confiance réels (vérifiables, sans invention)** — mentions légales réelles (raison sociale, pays d'établissement, immatriculation, identifiant fiscal lorsqu'il existe, adresse, support, §6.1 et §19.6) ; politiques complètes et cohérentes avec le checkout (retours §12.4, Incoterms §11.4, délais §12.1, frais d'import §11.3) ; origine réelle, code HS, poids et dimensions issus des données validées (§7.1, §11.5) ; distinction explicite entre frais **prépayés par Fanours** et frais **potentiellement collectés à l'arrivée** (§26.4) ; aucune promesse de « dédouanement garanti » — les estimations affichent leur niveau de certitude (§11.3, §11.10) ; auteur et relecture identifiés sur le contenu éditorial (§17.8).

---

### 17.10 Marketing et promotions

**Codes promo** (§17.3, §9.1, §20.1) — modèle `Promotion` : `code` (unique, insensible à la casse, non ambigu : pas de `0/O`, `1/I`) · `type` (`percentage` ou `fixed_amount`) · `value` (pourcentage borné 5-30 % ou montant) · `currency` (obligatoire si `fixed_amount`) · `min_basket_amount` (montant minimum, devise de transaction) · `max_discount_amount` (plafond de remise, devise de transaction) · `applies_to` (produits ou catégories inclus/exclus) · `segment` (`b2b`/`b2c`/`both`) · `countries` (pays de destination autorisés, liste vide = tous) · `starts_at`/`ends_at` (UTC, contrôlées **côté serveur**) · `max_uses` · `max_uses_per_customer` · `stackable` (`false` par défaut) · `exclusive` (`true` exclut tout autre code et toute remise automatique).

Implémentation : validation **serveur** à chaque ajout au panier et à chaque checkout (§5.4), le client ne peut pas forcer un code invalide (§26.2) ; le code ne s'applique pas si le panier n'atteint pas `min_basket_amount`, si le pays n'est pas dans `countries`, si le segment ne correspond pas, si la date est hors fenêtre, si le quota est atteint, ou si un produit est exclu ; remise calculée sur le sous-total **produits** (jamais sur le total frais et taxes) avec plafond appliqué ; journal d'utilisation (`PromotionRedemption` : code, commande, montant, horodatage, résultat) ; un code expiré ne disparaît pas silencieusement — le checkout affiche « code expiré » avec le motif et propose de le retirer ; la réduction ne modifie jamais la commande, le snapshot conserve prix d'origine **et** remise appliquée (§20.2).

**Bannières et campagnes** (§17.3, §16.10) — planifiées, ciblables par locale, pays, segment et type de page, créées en admin et validées avant publication ; contenu **identique** pour utilisateurs et robots, aucune variante « cachée » selon user-agent ou cookies (cloaking sanctionné) ; jamais de bannière recouvrant le prix, le MOQ, le total estimé ou le CTA d'achat ; hauteur d'emplacement réservée à l'avance (CLS), responsive, repli textuel si l'image ne charge pas ; pas d'animation perpétuelle ni de vidéo avec son, `prefers-reduced-motion` respecté (§22.4) ; les bannières de marque ne doivent **jamais masquer l'information de coût total** (§17.3, §9.2) — le récapitulatif complet reste visible au panier et au checkout.

**Newsletter et consentement** (§17.3, §6.2.12) — double opt-in recommandé (inscription effective après confirmation) ; consentement **enregistré** (horodatage, source, locale, version du texte, hash de l'IP — jamais l'IP en clair, identifiant du formulaire) ; case de consentement **non pré-cochée**, libellé expliquant finalité et fréquence, lien vers la politique de confidentialité ; désinscription en un clic dans chaque envoi et automatique en fin de campagne ; séparation stricte emails transactionnels / promotionnels — l'opt-out marketing ne doit jamais supprimer un email transactionnel (§14.3) ; aucun achat de listes, aucun envoi à des contacts collectés ailleurs, RGPD / CAN-SPAM / PIPL à faire valider par un juriste avant activation.

**Analytics respectueux de la confidentialité** (§17.3, §23.3, §19.6) — scripts non essentiels (analytics, pixels, heatmaps) **bloqués** tant que le consentement n'est pas donné, choix persisté et révocable ; aucune donnée personnelle dans les paramètres d'événement (ni e-mail, ni nom, ni adresse, ni contenu de commande, ni référence de devis) ; session replay **désactivé** sur checkout, paiement, connexion et toute zone de données personnelles (§19.6) ; paramètres d'URL contenant des PII non transmis, `utm_*` nettoyés avant envoi ; agrégation et anonymisation par défaut, rétention limitée et documentée, registre des sous-traitants tenu à jour ; préférence pour une solution first-party, le trafic organique est mesurable sans profiler l'individu.

**Dark patterns interdits** (§17.3) — case de consentement pré-cochée · bouton « Accepter tout » mis en avant par rapport à « Refuser » · *confirmshaming* (« Non, je veux payer plus cher ») · bouton de sortie caché ou disproportionné · pré-remplissage du panier sans action explicite · fausse urgence · fausse rareté · redirection forcée vers une offre. Chaque écran d'achat doit permettre refus, retour et abandon immédiat, avec un chemin visible vers le support.

---

### 17.11 Contrôle SEO pré-release

Checklist à exécuter **avant chaque release** (pipeline §25.2). « Bloquant » = empêche la mise en production. Automatisable en CI quand possible, sinon revue manuelle tracée.

| # | Point de contrôle | Méthode / outil | Bloquant |
|---|---|---|---|
| 1 | `robots.txt` présent, valide, sitemap déclaré | `curl -s` + validateur robots | Oui |
| 2 | Aucune route privée dans le sitemap | Script CI (sitemap ↔ liste noire §17.2) | Oui |
| 3 | Sitemap : uniquement des URL en 200 | `curl -w "%{http_code}"` échantillon 5 % + `Screaming Frog` | Oui |
| 4 | Aucune URL du sitemap avec paramètre | Script CI (inspection XML) | Oui |
| 5 | `lastmod` ≤ aujourd'hui et ≥ date de publication | Script CI sur les 3 index | Oui |
| 6 | Titre unique par page et par locale | `Screaming Frog` + script d'unicité | Oui |
| 7 | Description unique, longueur dans la plage cible | `Screaming Frog` | Oui |
| 8 | Un seul H1, non vide, unique | `Screaming Frog` | Oui |
| 9 | Hiérarchie H2/H3 sans saut de niveau | `Screaming Frog` (heading order) | Oui |
| 10 | Canonical absolu, auto-référentiel, unique | `Screaming Frog` + `curl` | Oui |
| 11 | Aucune canonical croisée entre locales | Script CI (canonical vs préfixe) | Oui |
| 12 | hreflang réciproque et complet (3 locales + x-default) | Script CI (graphe de réciprocité) | Oui |
| 13 | `hreflang` ↔ sitemap cohérents (mêmes slugs) | Script CI de correspondance | Oui |
| 14 | `<html lang>` correct sur les 3 locales | Playwright multi-locale | Oui |
| 15 | Aucun `noindex` accidentel sur locale publique | `curl -A "Googlebot"` sur 3 URLs/locale | Oui |
| 16 | `noindex` présent sur toutes les pages privées | Playwright sur les 14 routes privées | Oui |
| 17 | `x-robots-tag: noindex` absent des pages publiques | En-têtes HTTP sur échantillon | Oui |
| 18 | Statuts HTTP corrects (200/301/302/404) | `Screaming Frog` (response codes) | Oui |
| 19 | Vrais 404 sur 3 URL inexistantes/locale | `curl` | Oui |
| 20 | Aucune redirection en chaîne (max 1 saut) | `Screaming Frog` | Oui |
| 21 | Pas de boucle de redirection (racine ↔ locale) | Playwright : `/` → locale → `/` | Oui |
| 22 | JSON-LD valide (parsing) sur toutes pages publiques | `Rich Results Test` + script de parsing | Oui |
| 23 | `Product` : `price` = prix affiché, `priceCurrency`, `availability` valides | `Rich Results Test` + test de cohérence prix | Oui |
| 24 | `aggregateRating` présent uniquement si avis vérifiés et affichés | Script CI (compare aux avis publiés) | Oui |
| 25 | `Organization.sameAs` et `areaServed` réels | Revue manuelle | Oui |
| 26 | Pas de contenu structuré décrivant du contenu invisible | Revue manuelle (JSON-LD ↔ page) | Oui |
| 27 | `og:title`, `og:description`, `og:image`, `og:locale` présents | Playwright (balises meta) | Oui |
| 28 | Images : `alt` non vide, `width`/`height` déclarés | `Screaming Frog` + ESLint | Oui |
| 29 | AVIF/WebP, poids respectés, `priority` sur l'image LCP | Budget CI + revue | Oui |
| 30 | CWV terrain p75 dans les seuils | PageSpeed Insights API + CrUX API | Oui |
| 31 | Budgets de bundle respectés par route | Lighthouse CI (budgets) | Oui |
| 32 | Aucun script tiers bloquant, ≤ 2 scripts tiers | `Lighthouse` + revue | Oui |
| 33 | Polices : `font-display: swap`, ≤ 2 préchargements | Revue code + `Lighthouse` | Oui |
| 34 | Aucun lien interne vers URL paramétrée ou privée | `Screaming Frog` (crawl interne) | Oui |
| 35 | Pages orphelines = 0 | `Screaming Frog` (orphan pages) | Oui |
| 36 | Fil d'Ariane visible = fil d'Ariane JSON-LD | Script CI de correspondance | Oui |
| 37 | Traductions FR/EN/ZH-CN relues sur parcours critiques | Revue humaine (§26.1) | Oui |
| 38 | Aucun avis, compteur, badge ou urgence non vérifié | Revue humaine (§16.10) | Oui |
| 39 | Aucun contenu IA non vérifié en conformité/douane | Revue conformité | Oui |
| 40 | Aucun contenu dupliqué inter-locales | `Screaming Frog` (near duplicates) | Oui |
| 41 | Consentement analytics respecté, aucun script avant consentement | Revue code + test réseau Playwright | Oui |
| 42 | Logs : aucun accès anormal de crawler en zone privée | Revue logs (24 h) | Non |
| 43 | Pages indexées vs sitemap, dans les 3 locales | Search Console (couverture) | Non |
| 44 | Aucune erreur `noindex` signalée par GSC | Search Console (inspection) | Oui |
| 45 | Baseline positions/impressions enregistrée avant déploiement | Search Console export | Non |

Les points 1 à 41 sont automatisables ou vérifiables en une revue de 30 minutes : ils constituent la définition de « prêt pour le SEO » d'une release.

---

### 17.12 Mesure

| Famille | Indicateur | Source | Fréquence |
|---|---|---|---|
| Visibilité | Impressions, clics, CTR, position moyenne — **par locale** | Search Console (`country` + `page` + `query`) | Hebdo |
| Visibilité | Positions moyennes sur les 10 clusters (§17.8) | Search Console + suivi de positions | Hebdo |
| Visibilité | Part des impressions des 3 locales dans le total | Search Console | Mensuel |
| Indexation | Pages indexées vs présentes au sitemap, par locale | Search Console (couverture) | Hebdo |
| Indexation | Pages bloquées / dupliquées / exclues | Search Console + `Screaming Frog` | Mensuel |
| Indexation | Ratio `sitemap` → indexées | Search Console | Mensuel |
| Technique | CWV p75 par locale et par appareil | CrUX + RUM Fanours | Hebdo |
| Technique | Erreurs de balisage JSON-LD, taux de rich result valide | Search Console (rich results) + Rich Results Test | Mensuel |
| Trafic | Sessions organiques par locale, par pays de destination | Analytics (consent) | Hebdo |
| Trafic | Profondeur de session et clic vers fiche produit depuis l'éditeurial | Analytics | Mensuel |
| Conversion | Commandes et CA organiques, panier moyen, mix B2B/B2C | Analytics + données serveur | Hebdo |
| Conversion | Taux de conversion checkout, abandon par étape | Données serveur | Hebdo |
| Qualité | Requêtes de marque vs non-marque | Search Console | Mensuel |
| Qualité | Taux de rebond sur pages pays et guides | Analytics | Mensuel |
| Confiance | Avis vérifiés publiés, taux de modération | Admin | Mensuel |
| Confiance | Réclamations liées à des informations produit erronées | Tickets support | Mensuel |

**Outillage** — Search Console (référence indexation, hreflang, requêtes ; exports par locale et par cluster, jamais globaux) · Bing Webmaster Tools (hreflang croisé, indexation, sitemap) · CrUX API + Search Console CrUX (terrain, par locale et appareil) · PageSpeed Insights API (laboratoire, pré-release) · Screaming Frog (crawl complet, doublons, chaînes de redirection, orphelines, titles dupliqués ; hebdomadaire sur `fanours.com` + les 3 préfixes) · Search Console API (suivi des clusters et détection de cannibalisation, dimensions `page` + `query`) · RUM maison `web-vitals` → endpoint Fanours (attribution fine par route et locale, sans tiers).

**Lecture des résultats et règles de communication** — toute évolution est une **hypothèse** : un cluster qui progresse peut venir d'une mise à jour d'algorithme, de la saisonnalité, d'une promotion ou d'une variation de stock ; on documente les 2-3 causes concurrentes avant de conclure · un volume de données faible ne permet pas de conclure : sous 30 impressions sur 14 jours, la mesure n'est pas exploitable — on l'assume au lieu de forger une tendance · les délais sont longs (plusieurs semaines pour exploration, indexation, positionnement sur un cluster concurrentiel) : on mesure sur des fenêtres de 4 à 12 semaines · **interdiction absolue de promettre un classement**, un volume de trafic ou un délai d'indexation, en interne comme en externe — les rapports présentent des tendances observées, jamais des garanties ; la règle vaut pour les arguments commerciaux, les contrats et les réponses aux clients · le SEO est évalué sur le trafic qualifié et le CA qu'il génère, pas sur le nombre de mots-clés en position 1 · toute promesse commerciale sur un délai de livraison, un dédouanement ou un droit de douane est un sujet conformité (§11.10, §26.4), pas un sujet SEO : la page qui l'affiche doit présenter un niveau de certitude explicite.
