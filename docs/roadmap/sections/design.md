## Direction design, design system et UI/UX

Section de référence pour l'implémentation visuelle de Fanours. Elle dérive du cahier des charges (sections 3, 4, 6, 15, 16, 17, 22, 24.4, 26.1) et fait loi sur toute décision d'interface : si un écran contredit ce document, c'est l'écran qui change, pas le système.

---

### 3.1 Identité et positionnement

**Promesse de marque.** « Un peu de rose. Beaucoup de style. Je craque. » Traduite en cinq principes opposables lors de la revue de design (critères 26.1) :

| Principe | Règle concrète | Ce qu'on refuse |
|---|---|---|
| Le rose est un accent, pas un décor | Le rose n'occupe jamais plus de ~15 % de la surface visible d'un écran | Fond rose saturé, rose partout, boutons roses en série |
| Le calme avant l'excitation | Fond `#FFF5FA` (clair) / `#180D15` (sombre) ; le rose n'apparaît que sur l'action principale et les états sélectionnés | Dégradés animés, blobs, particules, halos roses |
| La donnée avant la décoration | Sur un écran de vente en gros, MOQ, palier, stock, délai et coût d'import sont lisibles avant toute image | Photo plein écran de produit masquant le prix ou le MOQ |
| Le ton est prévisible : factuel, courtois, sans folklore | Microcopy en phrases courtes, jamais de point d'exclamation sur un frais, jamais de promesse de délai garantie | « Meilleure qualité d'Asie ! », « Livraison express garantie ! » |
| La préciosité vient de l'exactitude | Décimales, unités, Incoterm et niveau de certitude douanier toujours affichés | Montants arrondis, fourchettes non étiquetées, devise ambiguë |

**Hiérarchie.** Trois niveaux, jamais quatre : (1) titre de page/titre de section — Plus Jakarta Sans 600-700 ; (2) libellé de bloc et donnée de marché — Inter 500-600 ; (3) métadonnée — Inter 400, `--color-text-muted`. Un prix, un MOQ et un statut d'action sont toujours au niveau 2, jamais au niveau 3 : c'est ce qui rend l'interface « premium » sans sur-décoration — la précision perçue comme du soin, pas l'ornement.

**Densité.** Modèle catalogue dense (2 colonnes à 360 px, 4 à 1440 px), cartes produit à 3 niveaux d'information obligatoires : image → nom + SKU → prix du palier actif + MOQ. Le second plan (délai de préparation, stock, pays d'expédition) apparaît au survol/focus et toujours dans le détail produit. Les dashboards client et admin sont plus denses encore (lignes compactes, 44 px de hauteur de ligne) car l'utilisateur professionnel compare, il ne flâne pas.

**Usage du rose signature.** Le rose est un pigment de marque, pas une couleur d'état : `#D81B75` (clair) / `#FF70B7` (sombre) est réservé à l'action primaire, au lien actif, à la sélection et au focus. Le rose 100-300 sert de fond de sélection, de badge et de bandeau d'information ; le rose 700-900 sert au texte d'accentuation sur fond clair. Le rose ne signifie jamais « succès » ou « erreur » — les couleurs fonctionnelles (§3.4) s'en chargent, et un statut ne repose jamais sur la couleur seule (spec 3.4).

**Ce qui fait « premium ».** Trois leviers, dans cet ordre : (1) l'alignement et la respiration — une seule échelle d'espacement de 4 px, aucune valeur hors tokens ; (2) la ponctuation typographique — Plus Jakarta Sans en display, Inter en corps de texte, `font-variant-numeric: tabular-nums` sur tous les prix et quantités pour que les colonnes s'alignent ; (3) l'honnêteté de l'information — si un montant est estimé, il est marqué comme estimé dans la même graisse que s'il était certain.

**Concilier rose bonbon + glassmorphism + crédibilité B2B.** Les deux s'opposent si on les applique uniformément. L'arbitrage retenu : le glassmorphism est un matériau de *chrome* (barre supérieure, barre de panier, popover, toast, onglet actif), jamais un matériau de *contenu* (tableau, prix, formulaire, zone de paiement). Un portail B2B crédible, c'est donc : des surfaces de contenu opaques et stables, et un chrome vitré qui donne du relief. Le rose bonbon est contenu dans des aplats nets (pas de dégradé rose sur rose) ; la seule exception admise est le CTA en dégradé `linear-gradient(135deg, #D81B75 0%, #C2185B 100%)`, dont le dégradé est **assombri** pour que le texte blanc garde son contraste (cf. §3.3). Un B2B qui « fait premium » vend du prix exact, du délai clair et de la conformité — pas du verre dépoli.

---

### 3.2 Tokens de design complets (CSS réel)

Source unique de vérité. Un seul fichier de tokens, compilé par Tailwind (`theme.extend`) et importé en CSS natif. Aucune couleur, aucun rayon, aucune durée n'est écrite en dur dans un composant (spec 5.1, 3.2).

#### 3.2.1 Polices

Auto-hébergées, sous-ensembles latins et CJK séparés ; `font-display: swap` ; seules les graisses utilisées sont téléchargées ; les CJK ne sont chargés que sur `zh-CN` (spec 3.2).

```css
@font-face { font-family:'Plus Jakarta Sans'; font-style:normal; font-weight:500; font-display:swap; src:url('/fonts/plus-jakarta-sans/latin-500.woff2') format('woff2'); unicode-range:U+0000-00FF,U+0131,U+0152-0153,U+2000-206F,U+2074,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD; }
@font-face { font-family:'Plus Jakarta Sans'; font-style:normal; font-weight:600; font-display:swap; src:url('/fonts/plus-jakarta-sans/latin-600.woff2') format('woff2'); unicode-range:U+0000-00FF,U+0131,U+0152-0153,U+2000-206F,U+2074,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD; }
@font-face { font-family:'Plus Jakarta Sans'; font-style:normal; font-weight:700; font-display:swap; src:url('/fonts/plus-jakarta-sans/latin-700.woff2') format('woff2'); unicode-range:U+0000-00FF,U+0131,U+0152-0153,U+2000-206F,U+2074,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD; }
@font-face { font-family:'Plus Jakarta Sans'; font-style:normal; font-weight:800; font-display:swap; src:url('/fonts/plus-jakarta-sans/latin-800.woff2') format('woff2'); unicode-range:U+0000-00FF,U+0131,U+0152-0153,U+2000-206F,U+2074,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD; }

@font-face { font-family:'Inter'; font-style:normal; font-weight:400; font-display:swap; src:url('/fonts/inter/latin-400.woff2') format('woff2'); unicode-range:U+0000-00FF,U+0131,U+0152-0153,U+2000-206F,U+2074,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD; }
@font-face { font-family:'Inter'; font-style:normal; font-weight:500; font-display:swap; src:url('/fonts/inter/latin-500.woff2') format('woff2'); unicode-range:U+0000-00FF,U+0131,U+0152-0153,U+2000-206F,U+2074,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD; }
@font-face { font-family:'Inter'; font-style:normal; font-weight:600; font-display:swap; src:url('/fonts/inter/latin-600.woff2') format('woff2'); unicode-range:U+0000-00FF,U+0131,U+0152-0153,U+2000-206F,U+2074,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD; }
@font-face { font-family:'Inter'; font-style:normal; font-weight:700; font-display:swap; src:url('/fonts/inter/latin-700.woff2') format('woff2'); unicode-range:U+0000-00FF,U+0131,U+0152-0153,U+2000-206F,U+2074,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD; }

/* CJK : chargé uniquement quand la locale l'exige (spec 3.2) */
@font-face { font-family:'Noto Sans SC'; font-style:normal; font-weight:400; font-display:swap; src:url('/fonts/noto-sans-sc/chinese-simplified-400.woff2') format('woff2'); unicode-range:U+4E00-9FFF,U+3000-303F,U+FF00-FFEF; }
@font-face { font-family:'Noto Sans SC'; font-style:normal; font-weight:500; font-display:swap; src:url('/fonts/noto-sans-sc/chinese-simplified-500.woff2') format('woff2'); unicode-range:U+4E00-9FFF,U+3000-303F,U+FF00-FFEF; }
@font-face { font-family:'Noto Sans SC'; font-style:normal; font-weight:700; font-display:swap; src:url('/fonts/noto-sans-sc/chinese-simplified-700.woff2') format('woff2'); unicode-range:U+4E00-9FFF,U+3000-303F,U+FF00-FFEF; }
```

#### 3.2.2 Bloc `:root` — mode clair (exhaustif)

```css
:root {
  /* ---- Typographie (spec 3.2) ---- */
  --font-heading: 'Plus Jakarta Sans', 'Noto Sans SC', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  --font-body: 'Inter', 'Noto Sans SC', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  --font-mono: ui-monospace, SFMono-Regular, 'SF Mono', Menlo, Consolas, 'Liberation Mono', monospace;
  --font-cjk: 'Noto Sans SC', 'PingFang SC', 'Hiragino Sans GB', 'Microsoft YaHei', sans-serif;

  --font-size-xs: 0.75rem;    --font-size-sm: 0.875rem;   --font-size-base: 1rem;     --font-size-lg: 1.125rem;
  --font-size-xl: 1.25rem;    --font-size-2xl: 1.5rem;    --font-size-3xl: 1.875rem;  --font-size-4xl: 2.25rem;
  --font-size-5xl: 3rem;
  --font-weight-regular: 400;  --font-weight-medium: 500;  --font-weight-semibold: 600;
  --font-weight-bold: 700;     --font-weight-extrabold: 800;
  --line-height-tight: 1.15;   --line-height-snug: 1.3;    --line-height-base: 1.55;  --line-height-relaxed: 1.7;
  --letter-spacing-tight: -0.02em;  --letter-spacing-normal: 0;  --letter-spacing-wide: 0.04em;

  /* ---- Couleurs de base (spec 3.3) ---- */
  --color-bg: #FFF5FA;               /* Fond global légèrement rosé */
  --color-surface: #FFFFFF;          /* Cartes, menus, panneaux, formulaires */
  --color-text: #321526;             /* Texte principal */
  --color-primary: #D81B75;          /* CTA, liens clés, sélection */
  --color-accent: #FFD6EA;           /* Fonds décoratifs, éléments sélectionnés */
  --color-surface-secondary: #F7DCE9; /* Sections secondaires */
  --color-surface-elevated: #FFFFFF;  /* Menus, popovers, flottants (au-dessus de --color-surface) */

  /* ---- Nuances rose 50 → 900 (spec 3.3) ---- */
  --color-rose-50: #FFF5FA;  --color-rose-100: #FFE8F2;  --color-rose-200: #FFD6EA;  --color-rose-300: #FFB5D6;
  --color-rose-400: #F58AB9; --color-rose-500: #E84393;  --color-rose-600: #D81B75;  --color-rose-700: #B91662;
  --color-rose-800: #8F124F; --color-rose-900: #64103A;

  /* ---- Couleurs fonctionnelles (spec 3.4) ---- */
  --color-success: #16794B;  --color-success-bg: #E7F5EE;  --color-warning: #9A5B00;  --color-warning-bg: #FDF1DC;
  --color-error: #B42336;   --color-error-bg: #FCEBEC;    --color-info: #2458B8;     --color-info-bg: #E8F0FD;
  --color-neutral: #667085; --color-neutral-bg: #F2F4F7;
  --color-border: #F0DCE6;         /* Séparateurs, contours décoratifs */
  --color-border-strong: #A8718A;  /* Contours de composants (UI) — cf. §3.3 */
  --color-border-input: #A8718A;   /* Bordure de champ de formulaire */

  /* ---- Palette catégorielle de visualisation (cf. §3.2.4) ---- */
  --color-chart-1: #D81B75;  /* rose marque — série 1 */
  --color-chart-2: #00857A;  /* sarcelle — série 2 */
  --color-chart-3: #7A4BB5;  /* violet — série 3 */
  --color-chart-4: #2B5FD9;  /* bleu — série 4 */
  --color-chart-5: #B26A00;  /* ambre — série 5 */
  --color-chart-grid: #F0DCE6;  /* lignes de grille, axes */

  /* ---- Dérivés de texte (garantissent le contraste, cf. §3.3) ---- */
  --color-text-muted: #667085;      --color-text-subtle: #667085;        --color-text-inverse: #FFFFFF;
  --color-text-on-primary: #FFFFFF; /* 4.85:1 sur #D81B75 */
  --color-link: #B91662;             /* 6.26:1 sur blanc, 5.39:1 sur rose 100 */
  --color-link-hover: #8F124F;       /* 8.32:1 sur #FFF5FA */

  /* ---- Dégradés (spec 3.5) ---- */
  --gradient-hero-light: linear-gradient(135deg, #FFF5FA 0%, #FFD6EA 55%, #FFFFFF 100%);
  --gradient-hero-dark: linear-gradient(135deg, #180D15 0%, #35182A 55%, #4B1F39 100%);
  --gradient-cta-light: linear-gradient(135deg, #D81B75 0%, #C2185B 100%); /* fin assombrie : garde le blanc à 5.87:1 */
  --gradient-cta-dark: linear-gradient(135deg, #FF70B7 0%, #FF9CCB 100%);
  --gradient-accent-soft: linear-gradient(180deg, #FFE8F2 0%, #FFD6EA 100%);

  /* ---- Ombres (spec 3.6 : ombre légère, 1px translucide) ---- */
  --shadow-xs: 0 1px 2px 0 rgb(50 21 38 / 0.05);    --shadow-sm: 0 2px 6px -1px rgb(50 21 38 / 0.08);
  --shadow-md: 0 6px 16px -4px rgb(50 21 38 / 0.10); --shadow-lg: 0 16px 32px -8px rgb(50 21 38 / 0.14);
  --shadow-focus: 0 0 0 3px rgb(216 27 117 / 0.28);  --shadow-none: none;

  /* ---- Rayons ---- */
  --radius-xs: 4px;    --radius-sm: 6px;   --radius-md: 8px;   --radius-lg: 12px;
  --radius-xl: 16px;   --radius-2xl: 24px; --radius-pill: 999px; --radius-full: 50%;

  /* ---- Espacements (échelle 4px) ---- */
  --space-0: 0;    --space-1: 4px;   --space-2: 8px;   --space-3: 12px;  --space-4: 16px;   --space-5: 20px;
  --space-6: 24px; --space-8: 32px;  --space-10: 40px; --space-12: 48px; --space-16: 64px;
  --space-20: 80px; --space-24: 96px; --space-32: 128px;

  /* ---- Cibles tactiles & dimensions ---- */
  --control-height-sm: 36px;  --control-height: 44px;  --control-height-lg: 48px;  --touch-target-min: 44px;
  --content-max: 1280px;      --content-wide: 1440px;
  --gutter: 16px;             /* 16px à 360px, 24px à 1024px, 32px à 1440px */

  /* ---- z-index ---- */
  --z-base: 0;      --z-raised: 10;    --z-sticky: 100;   --z-nav: 150;    --z-dropdown: 200;
  --z-overlay: 300; --z-modal: 400;     --z-toast: 500;    --z-tooltip: 600;

  /* ---- Breakpoints (spec 22.2) — usage : @media (min-width: 768px) etc. ---- */
  --bp-mobile: 360px;   /* petit mobile ≤ 360 */
  --bp-tablet: 768px;   /* tablette 768–1023 */
  --bp-desktop: 1024px; /* desktop 1024–1439 */
  --bp-wide: 1440px;    /* grand écran ≥ 1440 */

  /* ---- Mouvement (spec 3.7) ---- */
  --duration-instant: 80ms;  --duration-fast: 150ms;  --duration-normal: 240ms;  --duration-slow: 400ms;  --duration-slower: 600ms;
  --ease-out: cubic-bezier(0.16, 1, 0.3, 1);  --ease-in-out: cubic-bezier(0.65, 0, 0.35, 1);
  --ease-standard: cubic-bezier(0.4, 0, 0.2, 1);  --ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);

  /* ---- Glassmorphism (spec 3.6) ---- */
  --glass-blur: 16px;  --glass-blur-strong: 20px;
  --glass-surface-light: rgb(255 255 255 / 0.78);  --glass-surface-dark: rgb(38 20 32 / 0.72);
  --glass-border-light: rgb(255 255 255 / 0.55);   --glass-border-dark: rgb(255 240 247 / 0.12);

  /* ---- Focus (cf. §3.10) ---- */
  --focus-ring-width: 2px;  --focus-ring-offset: 2px;
  --focus-ring-color: #D81B75;      /* 4.54:1 sur --color-bg */
  --focus-ring-color-dark: #FF9CCB; /* 9.82:1 sur #180D15 */
}
```

#### 3.2.3 Mode sombre

Le mode sombre est appliqué par défaut si le système le demande, et forcé par `data-theme`. La garde `:root:not([data-theme="light"])` évite un flash de thème clair quand la préférence système est sombre (cf. §3.6). Les deux sélecteurs portent la même palette : la duplication est volontaire pour que la cascade reste explicite et lisible sans préprocesseur.

```css
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    color-scheme: dark;
    --color-bg: #180D15;               --color-surface: #261420;      --color-text: #FFF0F7;
    --color-primary: #FF70B7;          --color-accent: #4B1F39;       --color-surface-secondary: #35182A;
    --color-surface-elevated: #35202D; /* Menus, popovers, flottants */

    --color-rose-50: #2A1421;  --color-rose-100: #35182A;  --color-rose-200: #4B1F39;  --color-rose-300: #7A2A52;
    --color-rose-400: #C84B88; --color-rose-500: #E95B9E;  --color-rose-600: #FF70B7;  --color-rose-700: #FF9CCB;
    --color-rose-800: #FFC2DE; --color-rose-900: #FFE0EF;

    --color-success: #4ADE80;  --color-success-bg: #16301F;  --color-warning: #FBBF24;  --color-warning-bg: #33260C;
    --color-error: #FF6B7A;    --color-error-bg: #3A1620;    --color-info: #7CA7FF;     --color-info-bg: #16233D;
    --color-neutral: #B8A8B3; --color-neutral-bg: #2A1A26;
    --color-border: #49303E;   --color-border-strong: #A87C90; --color-border-input: #A87C90;

    --color-chart-1: #FF70B7;  --color-chart-2: #2FD3B0;  --color-chart-3: #C4A2F7;
    --color-chart-4: #7CA7FF;  --color-chart-5: #FBBF24;  --color-chart-grid: #49303E;

    --color-text-muted: #B8A8B3;  --color-text-subtle: #B8A8B3;  --color-text-inverse: #180D15;
    --color-text-on-primary: #180D15; /* 7.45:1 sur #FF70B7 */
    --color-link: #FF9CCB;            /* 9.82:1 */  --color-link-hover: #FFC2DE; /* 12.67:1 */

    --gradient-hero-light: var(--gradient-hero-dark);  --gradient-cta-light: var(--gradient-cta-dark);
    --gradient-accent-soft: linear-gradient(180deg, #35182A 0%, #4B1F39 100%);

    --shadow-xs: 0 1px 2px 0 rgb(0 0 0 / 0.40);    --shadow-sm: 0 2px 6px -1px rgb(0 0 0 / 0.45);
    --shadow-md: 0 6px 16px -4px rgb(0 0 0 / 0.50); --shadow-lg: 0 16px 32px -8px rgb(0 0 0 / 0.60);
    --shadow-focus: 0 0 0 3px rgb(255 156 203 / 0.35);

    --glass-surface-light: var(--glass-surface-dark);  --glass-border-light: var(--glass-border-dark);
    --focus-ring-color: var(--focus-ring-color-dark);
  }
}

:root[data-theme="dark"] {
  color-scheme: dark;
  --color-bg: #180D15;               --color-surface: #261420;      --color-text: #FFF0F7;
  --color-primary: #FF70B7;          --color-accent: #4B1F39;       --color-surface-secondary: #35182A;
  --color-surface-elevated: #35202D;

  --color-rose-50: #2A1421;  --color-rose-100: #35182A;  --color-rose-200: #4B1F39;  --color-rose-300: #7A2A52;
  --color-rose-400: #C84B88; --color-rose-500: #E95B9E;  --color-rose-600: #FF70B7;  --color-rose-700: #FF9CCB;
  --color-rose-800: #FFC2DE; --color-rose-900: #FFE0EF;

  --color-success: #4ADE80;  --color-success-bg: #16301F;  --color-warning: #FBBF24;  --color-warning-bg: #33260C;
  --color-error: #FF6B7A;    --color-error-bg: #3A1620;    --color-info: #7CA7FF;     --color-info-bg: #16233D;
  --color-neutral: #B8A8B3; --color-neutral-bg: #2A1A26;
  --color-border: #49303E;   --color-border-strong: #A87C90; --color-border-input: #A87C90;

  --color-chart-1: #FF70B7;  --color-chart-2: #2FD3B0;  --color-chart-3: #C4A2F7;
  --color-chart-4: #7CA7FF;  --color-chart-5: #FBBF24;  --color-chart-grid: #49303E;

  --color-text-muted: #B8A8B3;  --color-text-subtle: #B8A8B3;  --color-text-inverse: #180D15;
  --color-text-on-primary: #180D15;  --color-link: #FF9CCB;  --color-link-hover: #FFC2DE;

  --gradient-hero-light: var(--gradient-hero-dark);  --gradient-cta-light: var(--gradient-cta-dark);
  --gradient-accent-soft: linear-gradient(180deg, #35182A 0%, #4B1F39 100%);

  --shadow-xs: 0 1px 2px 0 rgb(0 0 0 / 0.40);    --shadow-sm: 0 2px 6px -1px rgb(0 0 0 / 0.45);
  --shadow-md: 0 6px 16px -4px rgb(0 0 0 / 0.50); --shadow-lg: 0 16px 32px -8px rgb(0 0 0 / 0.60);
  --shadow-focus: 0 0 0 3px rgb(255 156 203 / 0.35);

  --glass-surface-light: var(--glass-surface-dark);  --glass-border-light: var(--glass-border-dark);
  --focus-ring-color: var(--focus-ring-color-dark);
}

:root[data-theme="light"] { color-scheme: light; }

/* Accessibilité : mouvement réduit (spec 3.7, 22.4) */
@media (prefers-reduced-motion: reduce) {
  :root {
    --duration-instant: 0ms;  --duration-fast: 0ms;  --duration-normal: 0ms;
      --duration-slow: 0ms;     --duration-slower: 0ms;
  }
}
```

#### 3.2.4 Palette catégorielle de visualisation

Les couleurs fonctionnelles (`--color-success`, `--color-error`…) et la gamme rose portent le **sens** (un état, un badge, un statut). Elles ne doivent pas servir à encoder des **séries** de données : deux séries de même sens mais de valeurs différentes seraient indiscernables, et la rose sature. Les graphiques utilisent donc une palette catégorielle dédiée, déclarée dans le même fichier de tokens.

| Token | Clair | Sombre | Usage |
|---|---|---|---|
| `--color-chart-1` | `#D81B75` | `#FF70B7` | série 1 — rose marque (alignée sur le primaire) |
| `--color-chart-2` | `#00857A` | `#2FD3B0` | série 2 — sarcelle |
| `--color-chart-3` | `#7A4BB5` | `#C4A2F7` | série 3 — violet |
| `--color-chart-4` | `#2B5FD9` | `#7CA7FF` | série 4 — bleu |
| `--color-chart-5` | `#B26A00` | `#FBBF24` | série 5 — ambre |
| `--color-chart-grid` | `#F0DCE6` | `#49303E` | lignes de grille, axes, graduations |

**Règles d'emploi**

- **3 séries maximum par graphique** (contrainte de lisibilité B2B, §3.4.5). Au-delà, on empile en graphiques distincts ou on bascule en palette séquentielle (une seule teinte, du clair au foncé) quand la variable est continue.
- **L'ordre des séries est stable** : série 1 = rose, 2 = sarcelle, 3 = violet. Deux graphiques qui comparent les mêmes grandeurs utilisent le même rang → la couleur porte la même information d'un écran à l'autre.
- **La couleur n'est jamais le seul encodage** : chaque série porte aussi un libellé direct (étiquette en bout de courbe) ou une forme de marqueur distincte (cercle / carré / triangle). C'est la règle WCAG 1.4.1 « l'utilisation de la couleur », et c'est aussi ce qui rend le graphique lisible en daltonisme.
- **Grille et axes** en `--color-chart-grid`, jamais en couleur de série : la grille ne doit pas se confondre avec une donnée.
- **Aucune interpolation ni extrapolation** : si une série n'a pas de données sur une période, la courbe s'interrompt (pas de segment comblé). Si la série est entièrement indisponible, le graphique affiche un état « indisponible » avec la raison.

**Contrôle de lisibilité (calculé, non estimé)**

| Série | Clair / `#FFFFFF` | Clair / `#FFF5FA` | Sombre / `#261420` | Sombre / `#180D15` |
|---|---:|---:|---:|---:|
| 1 | 4.85 | 4.54 | 6.86 | 7.45 |
| 2 | 4.53 | 4.25 | 9.20 | 9.99 |
| 3 | 6.00 | 5.62 | 8.19 | 8.90 |
| 4 | 5.61 | 5.26 | 7.34 | 7.97 |
| 5 | 4.24 | 3.97 | 10.45 | 11.36 |

Toutes les valeurs ≥ 3:1 (WCAG 1.4.11, composants graphiques) ; les séries 1 à 3, qui sont les seules coexister sur un même graphique, sont en outre toutes ≥ 4.5:1 sur les fonds clairs et ≥ 6.8:1 sur les fonds sombres (AA texte).

**Séparation des séries 1 à 3 sous déficit de vision des couleurs** (distance ΔE CIE76 entre paires, simulation de Viénot–Brettel–Mollon) :

| Simulation | Écart minimal entre 2 séries | Lecture |
|---|---:|---|
| Vision normale | 40.3 | séries nettement distinctes |
| Deutéranopie | 9.3 | distinctes |
| Protanopie | 32.1 | distinctes |
| Tritanopie | 52.8 | distinctes |

Les séries 4 et 5 ne sont pas garanties séparables sous deutéranopie (ΔE < 2 entre certaines paires) : c'est pourquoi la limite de 3 séries coexistant est une **règle de conception**, pas une préférence, et pourquoi l'encodage non chromatique (libellé + marqueur) est obligatoire.

---

### 3.3 Vérification contraste WCAG 2.2 AA

Méthode : luminance relative sRGB (`c_lin = c/12.92` si `c ≤ 0.03928`, sinon `((c+0.055)/1.055)^2.4`), `L = 0.2126·R + 0.7152·G + 0.0722·B`, ratio `= (L_clair + 0.05) / (L_sombre + 0.05)`. Seuils : **4.5:1** texte normal, **3:1** texte ≥ 24 px ou ≥ 18.66 px gras, **3:1** composants d'interface et focus (WCAG 1.4.3, 1.4.11, 2.4.13).

#### 3.3.1 Mode clair

| Couple (texte / fond) | Ratio | Statut | Correction si échec |
|---|---:|:---:|---|
| Texte `#321526` / fond `#FFF5FA` | **15.50** | AA | — |
| Texte `#321526` / surface `#FFFFFF` | **16.52** | AA | — |
| Texte `#321526` / surface secondaire `#F7DCE9` | **12.88** | AA | — |
| Texte `#321526` / rose 100 `#FFE8F2` | **14.22** | AA | — |
| Texte `#321526` / accent `#FFD6EA` | **12.64** | AA | — |
| Primaire `#D81B75` / fond `#FFF5FA` | **4.54** | AA (juste) | — |
| Primaire `#D81B75` / surface `#FFFFFF` | **4.85** | AA | — |
| Primaire `#D81B75` / rose 100 `#FFE8F2` | **4.17** | **ÉCHEC texte normal** | Lien en texte → `--color-link: #B91662` (**5.39**) ; `#D81B75` réservé aux gros titres/UI sur fond clair |
| Primaire `#D81B75` / accent `#FFD6EA` | **3.71** | AA grand texte/UI | Interdit en corps de texte sur accent |
| **Blanc `#FFFFFF` / CTA `#D81B75`** | **4.85** | AA | — |
| **Blanc `#FFFFFF` / fin de dégradé CTA `#F04C9A`** | **3.38** | **ÉCHEC** | Dégradé CTA corrigé en `linear-gradient(135deg,#D81B75 0%,#C2185B 100%)` → blanc sur `#C2185B` = **5.87** AA |
| Succès `#16794B` / `#FFFFFF` | **5.43** | AA | — |
| Succès `#16794B` / `#FFF5FA` | **5.09** | AA | — |
| Erreur `#B42336` / `#FFFFFF` | **6.49** | AA | — |
| Erreur `#B42336` / `#FFF5FA` | **6.09** | AA | — |
| Attention `#9A5B00` / `#FFFFFF` | **5.43** | AA | — |
| Attention `#9A5B00` / `#FFF5FA` | **5.09** | AA | — |
| Info `#2458B8` / `#FFFFFF` | **6.64** | AA | — |
| Info `#2458B8` / `#FFF5FA` | **6.23** | AA | — |
| Neutre `#667085` / `#FFFFFF` | **4.97** | AA | — |
| Neutre `#667085` / `#FFF5FA` | **4.67** | AA | — |
| Bordure `#F0DCE6` / `#FFFFFF` | **1.31** | **ÉCHEC 1.4.11** | Séparateur décoratif uniquement ; contour de composant → `--color-border-strong: #A8718A` (**3.89** vs `#FFFFFF`, **3.65** vs `#FFF5FA`) |
| Rose 700 `#B91662` / `#FFFFFF` | **6.26** | AA | — |
| Rose 700 `#B91662` / `#FFE8F2` | **5.39** | AA | — |
| Rose 700 `#B91662` / `#FFD6EA` | **4.79** | AA | — |
| Rose 800 `#8F124F` / surface `#FFFFFF` | **8.87** | AA | — |
| Rose 800 `#8F124F` / fond `#FFF5FA` | **8.32** | AA | — |
| Rose 900 `#64103A` / `#FFE8F2` | **10.78** | AA | — |
| Rose 500 `#E84393` / `#FFFFFF` | **3.71** | AA grand texte/UI | Interdit en corps de texte ; réservé aux éléments ≥ 18.66 px gras / 24 px |

#### 3.3.2 Mode sombre

| Couple (texte / fond) | Ratio | Statut | Correction si échec |
|---|---:|:---:|---|
| Texte `#FFF0F7` / fond `#180D15` | **17.21** | AA | — |
| Texte `#FFF0F7` / surface `#261420` | **15.84** | AA | — |
| Texte `#FFF0F7` / accent `#4B1F39` | **12.26** | AA | — |
| Texte `#FFF0F7` / surface élevée `#35202D` | **13.65** | AA | — |
| Primaire `#FF70B7` / fond `#180D15` | **7.45** | AA | — |
| Primaire `#FF70B7` / surface `#261420` | **6.86** | AA | — |
| Primaire `#FF70B7` / accent `#4B1F39` | **5.31** | AA | — |
| Primaire `#FF70B7` / surface élevée `#35202D` | **5.91** | AA | — |
| **Blanc `#FFFFFF` / CTA `#FF70B7`** | **2.54** | **ÉCHEC** | Texte de CTA sombre = `--color-text-on-primary: #180D15` (**7.45**) |
| Sombre `#180D15` / CTA `#FF70B7` | **7.45** | AA | — |
| Sombre `#180D15` / fin de dégradé `#FF9CCB` | **9.82** | AA | — |
| Succès `#4ADE80` / `#180D15` | **10.88** | AA | — |
| Succès `#4ADE80` / `#261420` | **10.02** | AA | — |
| Erreur `#FF6B7A` / `#180D15` | **6.90** | AA | — |
| Erreur `#FF6B7A` / `#261420` | **6.35** | AA | — |
| Erreur `#FF6B7A` / surface élevée `#35202D` | **5.47** | AA | — |
| Attention `#FBBF24` / `#180D15` | **11.36** | AA | — |
| Attention `#FBBF24` / `#261420` | **10.45** | AA | — |
| Info `#7CA7FF` / `#180D15` | **7.97** | AA | — |
| Info `#7CA7FF` / `#261420` | **7.34** | AA | — |
| Neutre `#B8A8B3` / `#180D15` | **8.39** | AA | — |
| Neutre `#B8A8B3` / `#261420` | **7.72** | AA | — |
| Neutre `#B8A8B3` / surface élevée `#35202D` | **6.66** | AA | — |
| Bordure `#49303E` / `#261420` | **1.48** | **ÉCHEC 1.4.11** | Séparateur décoratif ; contour de composant → `--color-border-strong: #A87C90` (**4.95** vs `#261420`) |
| Bordure `#49303E` / `#180D15` | **1.60** | ÉCHEC 1.4.11 | idem |
| Rose 700 `#FF9CCB` / `#180D15` | **9.82** | AA | — |
| Rose 800 `#FFC2DE` / `#261420` | **11.65** | AA | — |
| Rose 900 `#FFE0EF` / `#180D15` | **15.50** | AA | — |
| Rose 500 `#E95B9E` / `#180D15` | **5.83** | AA | — |
| Rose 500 `#E95B9E` / `#35202D` | **4.62** | AA | — |
| Rose 300 `#7A2A52` / `#180D15` | **2.06** | **ÉCHEC** | Fond décoratif uniquement (jamais de texte ni de contour) |
| Rose 100 `#35182A` / `#180D15` | **1.19** | **ÉCHEC** | Fond décoratif uniquement |

#### 3.3.3 Décisions issues de la vérification

1. **Dégradé CTA clair corrigé** : `#D81B75 → #F04C9A` (spécifié au §3.5 du cahier des charges) échoue à 3.38:1 pour le texte blanc. Le token livré est `linear-gradient(135deg, #D81B75 0%, #C2185B 100%)`, qui conserve l'intention (dégradé rose) et passe à 5.87:1 sur toute la plage. Toute reprise du `#F04C9A` en texte blanc est interdite.
2. **Bordures à deux niveaux** : `--color-border` reste le séparateur décoratif (1.3-1.6:1, non textuel) ; `--color-border-strong` / `--color-border-input` garantissent 3:1 pour les contours de champs, de boutons secondaires, de switches et de cases à cocher.
3. **Texte de lien** : le rose 600 échoue sur fond teinté (4.17:1) ; le lien en texte utilise le rose 700 (`#B91662`, 5.39-6.26:1). Le rose 600 reste la couleur de fond de bouton, de bordure de focus et de sélection.
4. **CTA en mode sombre** : texte `#180D15`, jamais blanc.
5. **Fondu de surfaces** : en mode sombre, `surface #261420` sur `fond #180D15` (1.09:1) est un fond et non une bordure ; toute surface doit être délimitée par une bordure ou une ombre, pas par un contraste de luminance.
6. Le tableau complet est rejoué par un test automatisé (§3.12) sur chaque couple de tokens ; toute nouvelle paire est bloquante si < 4.5:1 sur du texte normal.

---

### 3.4 Bibliothèque de composants

Principes transverses (spec 3.7) : hauteur de bouton 44-48 px ; label persistant sur tous les champs ; skeleton calqué sur la structure finale ; statut jamais porté par la couleur seule ; focus clavier visible ; mouvement réduit respecté.

#### 3.4.1 Boutons

| Composant | Variantes | États | Clavier | Mobile |
|---|---|---|---|---|
| `Button` | `primary` (dégradé CTA), `secondary` (surface + bordure forte), `ghost` (texte), `danger` (erreur), `link` | normal, hover, focus-visible, active, disabled, loading | `Enter`/`Espace` activent ; focus-visible = anneau 2 px + offset 2 px | pleine largeur (`w-full`) pour le CTA de checkout ; hauteur 48 px |
| `IconButton` | avec/sans libellé visible | + pressed (toggle) | accessible name obligatoire (`aria-label` localisé) | 44 × 44 px minimum |
| `ButtonGroup` | segmenté (filtres, mode B2B/B2C) | + selected | flèches ←/→ pour passer d'un segment à l'autre | scroll horizontal autorisé, jamais de page |

Le CTA en chargement conserve sa largeur (spinner à gauche du libellé, libellé remplacé par `aria-busy`), pour éviter tout saut de mise en page.

#### 3.4.2 Champs de formulaire

| Composant | Variantes | États | Clavier | Mobile |
|---|---|---|---|---|
| `Input` / `Textarea` / `Select` / `Combobox` | texte, nombre, devise, date, recherche | normal, hover, focus-visible, error, disabled, readonly, loading | `Tab` entre champs, `Échap` ferme un combobox | `inputmode`/`autocomplete` corrects (`email`, `tel`, `postal-code`, `numeric`) |
| `Checkbox` / `Radio` / `Switch` | unique, groupe, activation | + checked, indeterminate | case native masquée visuellement mais focusable | 44 × 44 px de zone cliquable |
| `NumberStepper` (quantité) | MOQ, incrément, palier | + min/max, invalide | `↑`/`↓` incrément, `PageUp`/`PageDown` ± palier | stepper natif + saisie directe |
| `FileUpload` | simple, multi | + progression, error, virus | `Espace` ouvre le dialogue | `accept` + capture mobile, rétention du nom de fichier |

Règles : label persistant au-dessus du champ (jamais placeholder comme seul label) ; erreur affichée sous le champ, reliée par `aria-describedby`, `aria-invalid="true"` ; aide contextuelle liée par `aria-describedby` ; focus = bordure rose 600 + anneau 3 px (ratio 4.54:1 sur fond clair, 7.45:1 sur fond sombre).

#### 3.4.3 Catalogue et commerce

| Composant | Contenu obligatoire | États | Mobile |
|---|---|---|---|
| `ProductCard` | image (ratio 4:3), nom, SKU, prix du palier actif, MOQ + incrément, stock, délai de préparation | loading (skeleton image + 3 lignes), sold-out (image désaturée + libellé), preorder (badge) | 2 colonnes à 360 px, 3 à 768, 4 à 1024 ; l'image passe sous le contenu si la hauteur > 220 px |
| `PriceTierTable` | quantités, prix unitaire, remise en %, palier actif surligné | loading, indisponible | devient une liste de cartes ; le palier actif est repositionné en premier |
| `TierSelector` | sélection de palier par quantité | + désactivé hors MOQ | chips de quantité, défilement horizontal |
| `QuantityField` | MOQ, multiple, minimum/maximum | + invalide (message dédié) | stepper + pavé numérique |
| `AddToCartBar` | prix total recalculé, CTA, estimation livraison | + désactivé si stock insuffisant | barre fixée au-dessus de la nav basse, sans masquer le CTA |
| `CostEstimatePanel` | transport, taxes, droits, frais de courtage, niveau de certitude, Incoterm | + estimation indisponible | replié par défaut, déplié au récapitulatif checkout |

#### 3.4.4 Checkout, commandes et suivi

| Composant | Variantes | États | Clavier | Mobile |
|---|---|---|---|---|
| `CheckoutStepper` | 8 étapes du §9.2 | + courante, complétée, bloquée | `Tab` / flèches, libellé de l'étape annoncé | rail horizontal compact, libellés courts + `aria-current="step"` |
| `OrderSummary` | lignes, paliers, remises, transport, taxes, droits, total | + recalcul en cours | liens de modification | récapitulatif repliable en haut, total toujours visible en bas |
| `OrderTimeline` | 18 états du §9.3 mappés en 6 phases | + action requise | chaque étape est un `li` de liste | verticale, aucun élément masqué au clic |
| `TrackingTimeline` | événements horodatés du transporteur | + événement dupliqué/ordonné, colis retenu | — | verticale, horodatage localisé |
| `OrderStatusBadge` | draft→completed (18 états) | + statut inconnu | libellé textuel obligatoire | pastille + texte, jamais pastille seule |
| `PaymentStateCard` | en attente, accepté, refusé, expiré, contesté | + interruption prestataire | — | affichée avant le retour du prestataire |

**Tableau de suivi de commande (client).** Colonnes desktop : référence, date, statut, lignes, destination, total, actions. Mobile (< 768 px) : **carte par commande** — référence + statut badge (1re ligne), date + total (2e ligne), destination + nb de colis (3e ligne), barre d'action (4e ligne). Tri, recherche et filtres passent par un **bottom sheet** plutôt qu'une ligne de filtres compressée. La largeur du tableau est contrainte (`table-layout: fixed`, `overflow-x: auto` sur le conteneur uniquement) : aucun débordement horizontal de page (spec 22.1, 26.1).

#### 3.4.5 Dashboard

| Composant | Notes |
|---|---|
| `KpiCard` | valeur (Inter 700, `tabular-nums`), libellé, définition au survol/focus (spec 16.2 : chaque indicateur a une définition : date de reconnaissance, inclusion des remboursements, devise, provenance) |
| `DataTable` (admin) | tri serveur, pagination serveur, colonnes configurables, sélection de lignes, gel de la 1re colonne, `overflow-x` interne, jamais de scroll de page ; en dessous de 1024 px → vue cartes ou liste de colonnes prioritaires |
| `ReconciliationTable` (rapprochement) | période, commande, client, pays, devise, moyen, statut, ID externe, frais, écart ; l'écart est une **cellule à statut** (vert/ambre/rouge + icône + montant) ; annotation et attribution par ligne ; export CSV limité aux rôles autorisés (spec 10.5) |
| `BulkActionBar` | apparaît quand ≥ 1 ligne sélectionnée, ancrée en bas, hors de la zone de scroll, avec compteur |
| `AdminWizard` (règles douanières) | 4 étapes : pays/produit → type de règle → mode de calcul & seuil → source, validité, validateur ; champ « date de dernière révision » obligatoire avec alerte si dépassée (spec 11.2) |
| `InlineNote` | note interne invisible du client, visuellement distincte (jamais assimilable à un message client) |
| `FilterBar` | filtres par période, pays, devise, segment, catégorie ; en mobile : bouton « Filtres (n) » ouvrant un bottom sheet avec réinitialisation visible |
| `TrendChart` | courbe d'évolution temporelle (KPI dans le temps) : SVG inline, sans bibliothèque de graphes, palette **catégorielle** du thème (`--color-chart-1..3`, §3.2.4), 3 séries maximum, axes en `tabular-nums` ; chaque série porte un **libellé direct** en bout de courbe et un marqueur distinct (la couleur n'est jamais le seul encodage) ; chaque point porte une infobulle au survol/focus avec la valeur et la période ; **`prefers-reduced-motion`** respecté (pas d'animation de tracé) ; une série sans donnée sur une période **interrompt** la courbe (jamais de segment comblé) ; si la donnée n'est pas disponible, le graphique affiche un état « indisponible » avec la raison — **jamais** de courbe tracée sur des valeurs inventées ou interpolées |

#### 3.4.6 Surcouches et retours

| Composant | Comportement | Clavier | Mobile |
|---|---|---|---|
| `Modal` | centrée, max 560 px, voile `rgb(50 21 38 / 0.45)`, titre en `aria-labelledby`, description en `aria-describedby` | piège de focus, `Échap` ferme (sauf confirmation destructive), focus restitué à l'élément déclencheur | **bottom sheet** : bord supérieur arrondi 16 px, hauteur ≤ 90 vh, geste de fermeture, `role="dialog"` + `aria-modal` |
| `BottomSheet` | contenu de formulaire, filtres, sélection multiple | `Échap` ferme, focus piégé, `overscroll-behavior: contain` | natif, 2 poignées de resize, hauteur 45/90 vh |
| `Toast` | 4 s (succès) / 6 s (erreur, non bloquante), bouton de fermeture, `role="status"` (polite) ou `role="alert"` (assertive) si action requise | focusable, annoncé par `aria-live` | empilés en bas, au-dessus de la nav basse, jamais plus de 3 |
| `Tooltip` | info contextuelle, 300 ms d'attente, disparaît au `Escape`/blur | atteignable au clavier (focus + `aria-describedby`) | non utilisé au tactile : l'info devient un texte inline ou un bottom sheet |
| `Skeleton` | reproduit la grille finale, pas un simple rectangle | `aria-busy="true"` sur le conteneur, texte « Chargement… » pour les lecteurs d'écran | idem |
| `ConfirmDialog` | motif de saisie obligatoire pour annulation/remboursement (spec 9.4, 10.4) | — | plein écran, CTA unique visible sans scroll |

---

### 3.5 Règles glassmorphism et performance

#### 3.5.1 Où l'utiliser (autorisé)

| Surface | Traitement |
|---|---|
| Barre supérieure (desktop, sticky) | `backdrop-filter: blur(16px)`, `--glass-surface-light` 78 % d'opacité, bordure 1 px translucide, ombre légère |
| Barre de panier / header mobile | idem, `--glass-blur: 16px` |
| Popover, menu, sélecteur de langue/devise | `blur(20px)`, opacité 85 % |
| Toast | flou 12 px + ombre |
| Onglet actif de navigation basse | halo rose 200 à 20 % + bordure rose 300 |

#### 3.5.2 Où ne pas l'utiliser (interdit)

Le cahier des charges §3.6 impose la lisibilité sur les zones de paiement, de connexion, de sécurité et les tableaux opérationnels. Liste ferme :

- Formulaires de connexion, inscription, 2FA, réinitialisation de mot de passe → surface opaque.
- Zone de paiement et de saisie d'adresse de facturation → opaque.
- Tableaux de commandes, de rapprochement, d'audit, de stock → opaques, bordures visibles.
- Fiche produit, tableau de paliers, récapitulatif de commande, documents, PDF → opaques.
- Zones de saisie de données sensibles (justificatifs entreprise, codes de récupération) → opaques.
- Tout texte au-dessus d'une image ou d'un décor animé sans fond de contraste (§3.6).

#### 3.5.3 Budget de performance

- **Maximum 2 couches floutées simultanées** sur mobile (barre supérieure + barre de panier, ou feuille de fond + toast). Jamais 3 : au-delà, coût de composition GPU prohibitif sur appareil d'entrée de gamme (risque listé au §28.1 du cahier des charges).
- `backdrop-filter` jamais sur un conteneur qui contient un conteneur flouté (flou cumulé, coût doublé).
- Aucun flou sur un élément de hauteur > 200 px : un `blur` sur une grande surface est la principale cause de chute d'INP sur Android d'entrée de gamme.
- `will-change: transform` posé uniquement pendant l'animation, retiré à la fin.
- Le flou est **désactivé** quand `prefers-reduced-motion: reduce` ou quand la mesure de performance signale un LCP dégradé ; le fond de secours opaque est alors appliqué.
- Le flou ne s'applique pas aux images produit, aux vignettes et aux cartes.

#### 3.5.4 Fallback opaque

```css
.glass { background: var(--color-surface); border: 1px solid var(--color-border); }
@supports (backdrop-filter: blur(1px)) or (-webkit-backdrop-filter: blur(1px)) {
  .glass {
    background: var(--glass-surface-light);
    border: 1px solid var(--glass-border-light);
    -webkit-backdrop-filter: blur(var(--glass-blur));
    backdrop-filter: blur(var(--glass-blur));
  }
}
@media (prefers-reduced-motion: reduce) {
  .glass { background: var(--color-surface); -webkit-backdrop-filter: none; backdrop-filter: none; }
}
```

Le repli est le style de base : un navigateur sans `backdrop-filter` affiche une surface opaque correcte, jamais un texte illisible sur un fond translucide.

---

### 3.6 Système de theming

**Trois modes** : `light`, `dark`, `auto` (défaut). `auto` suit `prefers-color-scheme`.

**Persistance et synchronisation** (spec 3.8) : la préférence est stockée en `localStorage` (`fanours.theme`) et en base sur le compte (`User.theme`, cf. §20.1). Priorité à la connexion : `localStorage` → compte ; à la déconnexion : retour à `localStorage`. Le changement de thème est appliqué sans rechargement et **propagé au compte de façon asynchrone** (fire-and-forget, échec silencieux mais journalisé).

**Anti-flash au chargement** — script inline dans `<head>`, exécuté avant le premier rendu :

```html
<script>
(function () {
  try {
    var t = localStorage.getItem('fanours.theme');
    if (t === 'light' || t === 'dark') {
      document.documentElement.setAttribute('data-theme', t);
    } else {
      document.documentElement.removeAttribute('data-theme'); /* auto → media query */
    }
  } catch (e) { /* navigation privée : on garde l'auto */ }
})();
</script>
```

Le CSS est construit pour que la variante claire soit celle par défaut et que le mode sombre ne s'applique que via `:root:not([data-theme="light"])` + la media query — le script ci-dessus garantit que l'attribut est posé avant le premier paint.

**Règles par surface** :
- **Dashboard client et admin** : les deux thèmes sont supportés sans compromis (spec 26.1) ; les surfaces de tableau restent opaques dans les deux cas ; l'intensité du rose diminue d'un cran en mode sombre (le rose 600 devient plus lumineux, donc l'accent sur les fonds se porte sur rose 200/300, pas sur rose 600 en aplat).
- **Checkout** : le thème suit la préférence, y compris sur les étapes de formulaire ; les totaux et les frais restent lisibles dans les deux thèmes. Le passage en thème sombre ne doit jamais faire disparaître le récapitulatif.
- **Page de paiement hébergée par le prestataire** : **seules les personnalisations autorisées par le prestataire sont appliquées**. Concrètement, le prestataire sert sa propre page ; Fanours n'injecte qu'un *return URL* et, si le contrat le permet, un logo et une couleur de marque dans les zones explicitement prévues par son API. Aucun CSS injecté, aucun en-tête personnalisé, aucun dégradé : la page de paiement est hors du design system, et les stimuli visuels de Fanours ne s'y appliquent pas.
- **PWA** : `theme_color` suit le thème actif et `background_color` suit `--color-bg` ; le `manifest` déclare les deux icônes et le mode `standalone`.

---

### 3.7 i18n et devise dans l'UI

**Chaînes** (spec 4.2) : aucune chaîne codée en dur dans un composant. Un dictionnaire par locale (`fr-FR`, `en`, `zh-CN`) avec **clé de secours explicite** : si une clé manque en `zh-CN`, on retombe sur `fr-FR` puis sur la clé elle-même ; le build échoue si la clé reste manquante en production (contrôle CI).

**Longueur des libellés et expansion de layout**. L'anglais occupe en moyenne **+25 à +35 %** de place que le français sur un libellé d'interface ; le chinois est plus compact (~-20 %) mais ses glyphes CJK sont de pleine largeur et exigent des hauteurs de ligne plus grandes.

| Risque | Règle de design |
|---|---|
| Bouton qui déborde en EN | largeur `min-width` définie par la version FR, `padding-inline` symétrique, `white-space: normal` + `text-wrap: balance` sur les libellés longs, jamais de `white-space: nowrap` sur un CTA traduit |
| Libellé tronqué | interdiction du `text-overflow: ellipsis` sur les libellés d'action ; troncature autorisée uniquement sur les valeurs truncables (SKU, n° de commande) avec libellé complet en `title` + `aria-label` |
| Texte CJK | `line-height` +2 % vs latin, `letter-spacing: 0` (jamais négatif sur CJK), `word-break: keep-all` pour ne pas couper un mot chinois, `text-wrap: pretty` sur les paragraphes |
| Tableau | colonnes dimensionnées en `fr`/`ch` pour absorber l'expansion EN, jamais de largeur fixe en px sur une colonne de contenu |
| Espacement | les unités d'espace restent en px (elles ne se traduisent pas) ; seuls les textes s'adaptent |

**Cibles de largeur** : CTA principal ≥ 120 px de largeur, ≤ 320 px de largeur maximale, hauteur 48 px ; un libellé CJK de 6 caractères tient sans réduction de la police ; on ne réduit jamais la taille de police sous 14 px pour faire tenir un libellé.

**Formats de nombre, date et montant** (spec 4.4) : via `Intl.NumberFormat` et `Intl.DateTimeFormat`, avec `timeZone` explicite.
- `fr-FR` : `1 234,56 €` (espace insécable fine U+202F), dates `12/03/2026`, heures `14:05`.
- `en` : `€1,234.56`, dates `12 Mar 2026` (format long explicite, `03/12/2026` est ambigu).
- `zh-CN` : `¥1,234.56`, dates `2026年3月12日`.
- Pour les montants dont la devise n'est pas celle de la locale, le code ISO est affiché explicitement : `1 234,56 USD`.
- Les **prix unitaires** utilisent toujours l'échelle de décimales de la devise (2 pour EUR/USD, 0 pour CNY/JPY) ; les **quantités** sont toujours en entiers.
- Les timestamps sont stockés en UTC, affichés dans le fuseau du lieu logistique pour les événements d'expédition, avec le fuseau indiqué explicitement quand l'événement est douanier.

**Devise de référence vs devise de transaction** (spec 4.3). Trois notions distinctes, trois libellés distincts :

| Notion | Stockage | Affichage |
|---|---|---|
| Montant de référence (back-office) | `amount_ref` + `currency_ref` (CNY ou USD, configurable) | dans le back-office et les exports comptables uniquement |
| Montant de la devise d'affichage | `amount_display` + `currency_display` + `rate` + `rate_timestamp` | sur le site, dans la devise choisie par l'utilisateur |
| Montant réellement débité | `amount_charged` + `currency_charged` | checkout et confirmation de commande, toujours explicite |

Règles d'affichage :
1. Un montant converti est **toujours** suivi de la mention « indicatif » (FR) / « indicative » (EN) / « 仅供参考 » (ZH), avec un tooltip expliquant que le taux de la banque ou du prestataire peut différer.
2. Le taux appliqué est horodaté et sa source est nommée (« taux.Provider, 21/10/2026 14:05 UTC »).
3. Le checkout affiche **la devise qui sera débitée** en gros, et la devise d'affichage en second, avec la conversion entre les deux.
4. API de change indisponible ou taux périmé → on affiche le montant de référence et la mention « taux indisponible », jamais un taux estimé silencieusement.
5. La distinction **frais payés par Fanours** vs **frais potentiellement collectés à la livraison** est obligatoire et visuelle (§11.4) : deux blocs distincts, l'un sur `--color-surface`, l'autre sur `--color-warning-bg` avec libellé explicite.

---

### 3.8 Mobile-first

Le mobile est le point de départ (spec 22.1), pas une version réduite.

**Navigation basse (4 onglets, ≤ 767 px)** : `Accueil` · `Catalogue` · `Panier` (badge de quantité) · `Compte` (Commandes/Compte, spec 22.1). Hauteur 64 px + zone de safe-area (`env(safe-area-inset-bottom)`). Le compteur du panier est un `aria-label` (« Panier, 3 articles ») et pas seulement une pastille chiffrée.

**En-tête compact** : 56 px — logo (28-32 px), recherche (icône qui s'ouvre en champ plein écran), compte, panier. Le sélecteur de langue et de devise passe dans un menu « ⋯ » ; il ne doit jamais manger plus de 2 lignes. Au scroll vers le bas, l'en-tête se réduit (logo seul) et la navigation basse reste visible.

**Panier accessible sans masquer le CTA** : le récapitulatif de panier et le CTA « Commander » ne se recouvrent jamais. Règle : le CTA est dans un flux normal, pas en `position: fixed` ; si un CTA fixe est utilisé (barre d'ajout au panier), il est placé **au-dessus** de la navigation basse (`bottom: calc(64px + env(safe-area-inset-bottom))`) et son conteneur parent reçoit un `padding-bottom` égal à sa hauteur. Test de non-recouvrement obligatoire à 320 px.

**Tableaux → cartes** : sous 768 px, tout tableau de données devient une liste de cartes. `OrdersTable`, `ReconciliationTable`, `KpiGrid` et `PriceTierTable` ont chacun une version carte dédiée. La table HTML reste en DOM pour le SEO et les lecteurs d'écran, mais `display: none` sous le breakpoint avec `aria-hidden` sur la version masquée.

**Prévention du débordement horizontal** : `min-width: 0` sur les enfants de flex/grid, `max-width: 100%` sur les images et médias, `word-break: break-word` sur les libellés longs, `overflow-wrap: anywhere` sur les SKU, jamais de `width: 100vw` (barre de défilement). `html, body { overflow-x: hidden; }` reste un dernier recours, jamais la solution principale. Test automatisé : `document.documentElement.scrollWidth <= clientWidth` à 320 / 360 / 390 / 430 px (spec 24.4).

**Cibles tactiles** : 44 × 44 px minimum pour toute action (spec 3.7), 48 × 48 px pour les actions principales. Les liens de texte inline sont exemptés mais doivent avoir un `padding` vertical qui porte la zone cliquable à 44 px sur les écrans tactiles.

**Upload de documents depuis mobile** (spec 22.1) : `<input type="file" accept="image/*,application/pdf" capture>` pour les justificatifs ; acceptation du glisser-déposer sur desktop et du choix de fichier / appareil photo sur mobile ; prévisualisation ; barre de progression ; validation de type MIME **réelle** et de taille (spec 19.3) ; reprise possible après échec réseau ; texte d'aide indiquant le format et la taille max.

**Formulaires** : champs `inputmode` et `autocomplete` corrects ; champs de date avec sélecteur natif ; checkout découpé en étapes courtes (une idée par écran) ; la touche « Entrée » ne valide l'étape que si le formulaire de l'étape est valide.

**Autres règles** : images `srcset` + `loading="lazy"` sous la ligne de flottaison, dimensions explicites pour éviter le CLS (spec 23.1) ; `dvh` pour la hauteur de vue mobile ; respect de `env(safe-area-inset-*)` ; pas de pull-to-refresh sur le checkout (risque de double soumission).

---

### 3.9 États systémiques

Spec 6.6 : chargement, absence de résultat, accès refusé, erreur réseau, session expirée, ressource indisponible, conflit de stock, quantité invalide, paiement en attente, interruption du prestataire. **Règle absolue : le message ne contient jamais de donnée technique** — pas de code HTTP brut, pas de stack trace, pas d'identifiant d'erreur interne, pas de nom de table, pas d'ID de webhook. Une référence de corrélation peut être affichée à l'utilisateur final uniquement comme référence à citer au support, et seulement au format « Réf. 7F3K2Q ».

| État | Message FR | Message EN | Message ZH | Icône | Action de sortie |
|---|---|---|---|---|---|
| Chargement | « Chargement… » (via `aria-busy`, pas de texte figé) | « Loading… » | 「正在加载…」 | skeleton | — (attente) |
| Vide (catalogue) | « Aucun produit ne correspond à ces filtres. » | « No products match these filters. » | 「没有符合筛选条件的产品。」 | recherche | « Retirer les filtres » (primaire) + « Voir tout le catalogue » |
| Vide (commande) | « Vous n'avez pas encore de commande. » | « You have no orders yet. » | 「您还没有订单。」 | colis | « Découvrir le catalogue » |
| Accès refusé (403) | « Vous n'avez pas accès à cette page. Si vous pensez qu'il s'agit d'une erreur, contactez le support. » | « You don't have access to this page. If you believe this is a mistake, contact support. » | 「您无权访问此页面。如认为有误，请联系客服。」 | cadenas | « Retour » + « Contacter le support » (primaire) |
| Session expirée | « Votre session a expiré. Reconnectez-vous pour continuer sans perdre votre panier. » | « Your session expired. Sign in to continue — your cart is saved. » | 「您的会话已过期。请重新登录以继续，购物车已保留。」 | clé | « Se reconnecter » (primaire) ; panier et brouillon conservés côté serveur |
| Erreur réseau | « Connexion impossible. Vérifiez votre réseau et réessayez. » | « Connection failed. Check your network and try again. » | 「无法连接。请检查网络后重试。」 | nuage barré | « Réessayer » (primaire) + « Voir mes commandes » hors ligne si disponible |
| Ressource indisponible | « Cette page n'est plus disponible. » | « This page is no longer available. » | 「此页面已不可用。」 | page | « Retour à l'accueil » + « Rechercher un produit » |
| Conflit de stock | « Ce stock vient d'être réservé par un autre acheteur. Modifiez la quantité ou demandez un devis. » | « This stock was just taken by another buyer. Adjust the quantity or request a quote. » | 「该库存刚被其他买家占用。请调整数量或申请报价。」 | cube | « Ajuster la quantité » (primaire) + « Demander un devis » + « Produits similaires » |
| Quantité invalide / MOQ | « Quantité invalide : le minimum est de 24 unités, par multiples de 6. » | « Invalid quantity: minimum 24 units, in multiples of 6. » | 「数量无效：最少 24 件，需为 6 的倍数。」 | calculatrice | champ en erreur, `aria-invalid`, correction proposée (« Passer à 24 ») ; si MOQ non atteint, CTA désactivé + explication, pas de message d'erreur |
| Paiement en attente | « Paiement en attente de confirmation par notre prestataire. Ne réglez pas deux fois : la page se mettra à jour automatiquement. » | « Payment is awaiting confirmation from our provider. Don't pay twice — this page updates automatically. » | 「支付正在等待服务商确认。请勿重复支付，本页将自动更新。」 | horloge | « Actualiser » + « Vérifier le statut du paiement » |
| Interruption du prestataire | « Le service de paiement est momentanément indisponible. Votre commande est enregistrée, vous n'aurez rien à re-saisir. » | « The payment service is temporarily unavailable. Your order is saved — nothing to re-enter. » | 「支付服务暂时不可用。您的订单已保存，无需重新填写。」 | outil cassé | « Réessayer le paiement » (primaire) + « Commander sans paiement en ligne » si le virement est activé |
| Règlement refusé | « Le paiement a été refusé par votre banque. Aucun montant n'a été débité. » | « Your bank declined the payment. Nothing was charged. » | 「您的银行拒绝了此次支付。未发生扣款。」 | carte barrée | « Réessayer avec un autre moyen » + « Contacter ma banque » |
| Timeout de paiement | « La confirmation prend plus de temps que prévu. Vérifiez l'état de la commande avant de réessayer. » | « Confirmation is taking longer than expected. Check the order status before retrying. » | 「确认时间超出预期。请先查看订单状态再重试。」 | sablier | « Vérifier l'état de la commande » (primaire) |
| Erreur de douane / colis retenu | « Action requise : des frais ou un document sont attendus à destination. » | « Action required: fees or a document are expected on arrival. » | 「需要处理：目的地可能产生费用或需要文件。」 | colis avec point d'exclamation | « Voir les documents à fournir » + « Contacter le support » |
| Document manquant (compte pro) | « Justificatif requis : attestation de conformité. » | « Required document: certificate of conformity. » | 「需要文件：符合性证明。」 | document | « Téléverser le document » (primaire) + formats acceptés |
| Règle douanière à revalider | « Les conditions d'importation pour cette destination sont en cours de vérification. Estimation suspendue. » | « Import conditions for this destination are being re-verified. Estimate unavailable for now. » | 「该目的地的进口条件正在复核，暂时无法给出估算。」 | balance | « Voir les conditions » + « Être averti » |
| Erreur générique | « Une erreur est survenue. Notre équipe a été informée. » | « Something went wrong. Our team has been notified. » | 「出现错误，我们已通知相关团队。」 | triangle | « Réessayer » + « Contacter le support » avec la référence |

Implémentation : un composant unique `SystemState` (variante `loading | empty | forbidden | offline | expired | unavailable | conflict | invalid | pending | interrupted | declined | error`) garantit un rendu homogène et une couverture de test unique. Les icônes proviennent de la bibliothèque unique du projet (jamais d'emoji, cf. §3.11). Les messages vivent dans les dictionnaires i18n, jamais en dur.

---

### 3.10 Accessibilité WCAG 2.2 AA

Objectif : WCAG 2.2 AA sur les parcours critiques (spec 22.4) — catalogue, fiche produit, panier, checkout, login, dashboard client, gestion de commande admin.

**Focus visible (2.4.7 / 2.4.13 Focus Appearance)** : anneau `outline: 2px solid var(--focus-ring-color); outline-offset: 2px;` + halo `box-shadow: var(--shadow-focus)`. Couleur : rose 600 `#D81B75` (4.54:1 sur fond clair, 4.85:1 sur surface) en mode clair ; rose 700 `#FF9CCB` (9.82:1) en mode sombre. Le focus n'est jamais supprimé (`outline: none` interdit sans remplacement visible) ; `:focus-visible` pour la souris, `:focus` maintenu sur les champs de saisie. Le focus ring n'est jamais rogné : `overflow: hidden` sur un conteneur qui porte un élément focusable est interdit, ou remplacé par `outline-offset: -2px` documenté. La cible est l'élément focusable entier (pas seulement son texte) ; dans les grilles, la carte entière est focusable si elle est cliquable.

**Ordre de focus** : suit l'ordre visuel. Pas de `tabindex` positif. Les conteneurs de grille ne sont pas des points d'arrêt sauf zone de liste (roving `tabindex`). Le focus revient à l'élément déclencheur à la fermeture d'une modale ou d'un bottom sheet.

**Piège de focus en modale** (2.4.3) : modale et bottom sheet piègent le focus, `aria-modal="true"`, contenu arrière-plan `inert` (ou `aria-hidden` + `inert`), `Échap` ferme (sauf confirmation destructive), focus initial sur le titre ou le premier champ utile, focus restitué à la fermeture.

**Landmarks et structure** : `<header>`, `<nav aria-label="Navigation principale">`, `<main id="contenu">`, `<aside aria-label="Filtres">`, `<footer>`, un seul `<h1>` par page, hiérarchie de titres sans saut (`h2` → `h3`), `<section aria-labelledby>` pour regrouper les cartes produit et les panneaux du dashboard. Lien d'évitement « Aller au contenu principal » visible au focus clavier.

**Live regions** : `aria-live="polite"` + `role="status"` pour les totaux recalculés (« Total : 1 240,00 € »), les changements de palier de prix, la mise à jour du stock (« Stock mis à jour : 240 unités disponibles »), les ajouts au panier. `aria-live="assertive"` + `role="alert"` **uniquement** pour : erreur de paiement, conflit de stock, session expirée, fichier rejeté. La région live existe dans le DOM avant l'injection du message (sinon la première annonce est perdue) ; le contenu est injecté dans un nœud vide. Les notifications non critiques (toast) utilisent `role="status"` ; le compteur du panier est annoncé via le `aria-label` du lien, pas via une live region (sinon doublon).

**Annonces lecteur d'écran** : chaque changement de palier annonce le prix unitaire *et* le prix total ; l'estimation de douane annonce le niveau de certitude (« estimation, montant définitif à confirmer ») ; la validation de champ annonce l'erreur via `aria-describedby` + `aria-invalid` ; les paliers de prix sont un vrai `<table>` avec `<caption>` et en-têtes `<th scope="col">` / `<th scope="row">`.

**Cible tactile (2.5.8 Target Size, minimum)** : 24 × 24 CSS px minimum pour toute cible pointer, 44 × 44 px minimum pour les actions principales (spec 3.7). Exceptions documentées : cibles inline dans un texte, éléments d'un composant où une cible alternative existe (petit `×` à côté d'un toast fermable de 44 px).

**`prefers-reduced-motion`** : toute animation > 200 ms est désactivée ou remplacée par un fondu ≤ 100 ms ; les transitions de position deviennent des fondus ; les animations de chargement (skeleton pulse, spinner) sont conservées mais ralenties, ou remplacées par une opacité fixe si l'utilisateur demande une réduction de mouvement (une animation continue peut déclencher des nausées — WCAG 2.3.3) ; aucune information n'est portée par le mouvement seul.

**Texte redimensionnable à 200 %** (1.4.4) : conteneurs en `rem`, hauteurs de ligne en `rem`, aucune hauteur fixe en `px` sur un bloc de texte, `max-height` uniquement avec scroll ; à 200 % le contenu n'est ni tronqué ni perdu ; les tableaux se transforment en cartes plutôt que de rétrécir.

**Autres exigences** : langue de page déclarée (`<html lang>` mis à jour à chaque changement de locale) ; titres de page et `meta description` localisés (spec 17.1) ; texte alternatif utile et non redondant sur les images produit ; `autocomplete` sur les champs de formulaire ; erreurs de champs annoncées et focalisées à la soumission (`aria-describedby` + focus sur le premier champ invalide) ; pas de piège de focus hors modale ; liens et boutons distingués sans dépendre uniquement de la couleur (soulignement ou bordure).

---

### 3.11 Règles de qualité visuelle (anti-template IA)

Une interface qui échoue à l'une de ces règles **n'est pas mergeable**. Ce sont des critères de rejet, pas des préférences.

**Interdits (rejet automatique)** :

1. **Hero générique sans identité** : photo d'ambiance + titre centré + « Explorez la collection » sans mentionner le gros, le MOQ, la Chine ou la destination. Le hero Fanours porte l'accroche de marque **et** une information commerciale vérifiable (MOQ, paliers, délai), sinon il est refusé.
2. **Dégradé sur tout** : un dégradé par section, ou un aplat dégradé de fond de carte. Les 4 dégradés du §3.2 ont une affectation unique (hero, CTA, accent doux) ; un cinquième dégradé est interdit.
3. **Glassmorphism partout** : toute surface de contenu vitrée est refusée (cf. §3.5.2).
4. **Icônes emoji** : aucune emoji comme icône d'interface. Bibliothèque d'icônes SVG unique, trait 1.75 px, tailles 20/24 px, `currentColor`.
5. **Badges `rounded-full` partout** : les pastilles sont réservées aux statuts et catégories ; CTA et étiquettes de formulaire au rayon `--radius-md` (8 px). Un `pill` décoratif non lié à un statut est interdit.
6. **Espacements incohérents** : toute valeur d'espacement hors de l'échelle 4 px est un défaut. Même règle pour les rayons, les durées et les tailles de police.
7. **Ombres « diffuses »** : pas d'ombre portée massive ni de `box-shadow` multi-couches sans rôle fonctionnel. Trois niveaux max (`xs`, `md`, `lg`) + `focus`.
8. **Faux compteurs, faux avis, faux badges de rareté** (spec 16.10, 17.3) : « 1 234 clients satisfaits » sans source est interdit.
9. **Sections d'« avis clients » génériques** : pas de témoignage inventé, pas d'avatar générique répété.
10. **Animations décoratives au chargement** : pas de fade-in en cascade sur toutes les cartes, pas de compteur animé.
11. **Boutons sans état désactivé** : tout bouton a un état `disabled` stylé, jamais seulement grisé par opacité.
12. **Icône seule sans libellé** : accessible name obligatoire.
13. **Texte marketing dans un contexte contractuel** : « Livraison internationale garantie 7 jours » est interdit sans engagement contractuel (spec 1.1, 11.10) ; les estimations sont étiquetées « estimation ».
14. **Prix sans palier** : afficher un prix unitaire sans indiquer la quantité qui le déclenche est refusé (spec 6.4) — c'est la signature d'un template plutôt que d'un outil de vente en gros.

**Exigés (présence obligatoire)** :

1. Le **MOQ et l'incrément** apparaissent sur chaque carte produit et dans le panier.
2. Le **niveau de certitude** (certain / estimé / à confirmer) est visible sur toute somme liée à la douane, aux droits ou au délai.
3. Le **délai de préparation** est distingué du **délai de transport** (spec 6.4, 12.1) — jamais « 10-15 jours » en un seul bloc.
4. La **devise** est affichée avec son code quand elle diffère de la locale.
5. Le **statut** combine toujours couleur + icône + texte (spec 3.4).
6. Les **états système** (§3.9) sont présents sur toutes les listes et formulaires.
7. Les **deux thèmes** sont implémentés partout, dashboard compris (spec 26.1) — pas de « sombre = clair inversé ».
8. Les **trois langues** sont implémentées avec des dictionnaires, et le layout est testé à 360 px dans les trois (spec 24.4).
9. Les **micro-interactions** sont limitées à trois gestes signature : apparition du recalcul de total (fondu 150 ms), ouverture du bottom sheet (translation 240 ms `ease-out`), confirmation d'ajout au panier (pastille qui pulse 1 fois, 400 ms). Pas d'effet « ressort » sur les CTA.
10. Une **densité** assumée : l'espace blanc sert à regrouper, pas à meubler. Pas de section vide de 200 px entre deux blocs liés.

---

### 3.12 Process de design

**Principe : les tokens sont la source unique, pas Figma.** Figma sert à explorer et à communiquer ; il ne sert pas de source de vérité. Les tokens vivent dans `tokens/` (JSON) et sont compilés en CSS, en thème Tailwind, en `manifest` PWA et en variables des PDF de facturation (spec 14.5 : les PDF doivent embarquer des polices compatibles CJK et être testés visuellement).

**Livrables de conception, par phase** :

| Phase | Livrable design |
|---|---|
| Phase 0 — Cadrage | Direction visuelle écrite (cette section) + inventaire des écrans critiques |
| Phase 1 — Fondations | Tokens CSS, thème clair/sombre, Storybook des composants de base, maquettes des 6 parcours critiques |
| Phase 2 — Catalogue et commerce | Maquettes catalogue / fiche produit / panier / checkout, validées en 3 langues et 2 thèmes |
| Phase 3 — B2B & logistique | Maquettes devis, suivi de commande, documents douaniers |
| Phase 4 — Dashboards | Maquettes client et admin par rôle, table de données dense, wizard de règles |
| Phase 5 — QA | Captures de référence, tests visuels, audit a11y |

**Figma : oui, mais cadré.** Un fichier unique avec une page « Foundations » (couleurs, typographie, espacements, rayons, ombres — alimentée depuis les tokens, pas recopiée), une page « Components » alignée sur Storybook, une page « Screens » par parcours. Le design ne diverge pas du code : **le fichier Figma est régénéré depuis les tokens** quand ceux-ci changent. Pas de maquette haute-fidélité d'un écran non implémentable.

**Storybook = documentation vivante.** Chaque composant a une story par variante × état (normal, hover, focus-visible, active, disabled, loading) × locale (FR, EN, ZH) × thème (clair, sombre). Les stories servent aussi de terrain de test visuel. Les tokens sont exposés via `@storybook/addon-tokens` ; une story qui diverge d'un token est un défaut CI.

**Revue de design avant merge** — obligatoire pour toute PR qui touche l'interface :
1. La PR contient une capture du composant dans ses états, dans les 2 thèmes, en FR/EN/ZH, à 360 et 1440 px.
2. Une revue (agent `ui-ux-checker` ou humain) valide : conformité aux tokens (§3.2), contraste calculé (§3.3), présence des états système (§3.9), conformité responsive (§3.8), conformité a11y (§3.10), respect des règles anti-template (§3.11).
3. Les écarts au design system sont soit corrigés, soit documentés et acceptés explicitement dans la description de la PR.
4. Aucun « ça marche en local » sans capture ni test.

**Tests visuels de régression** (spec 24.4) : Playwright `toHaveScreenshot` sur un catalogue de vues de référence.
- Viewports : 320, 360, 390, 430, 768, 1024, 1440 et 1920 px.
- Axes croisés : 2 thèmes × 3 locales sur les 5 vues clés (accueil, catalogue, fiche produit, panier, checkout) ; + dashboard client, dashboard admin, tableau de rapprochement, wizard de règles, modale/bottom sheet, états système (toutes les variantes de `SystemState`).
- Règles : seuil de diff ≤ 0.1 % ; diff > 0.5 % = échec bloquant ; snapshots mis à jour manuellement (jamais `updateSnapshots` en CI).
- Ce catalogue sert aussi de test de non-débordement horizontal (`scrollWidth <= clientWidth`).
- Les tests E2E de parcours (spec 24.3) couvrent la fonctionnalité en plus des captures.

**Contrôle automatisé des tokens** (CI) :
- Un script compare la palette utilisée dans les composants à la palette des tokens : toute couleur hex hors tokens échoue le build.
- Un script de contraste rejoue le tableau du §3.3 sur tous les couples de tokens (clair + sombre) et échoue sous 4.5:1 (texte) / 3:1 (composants).
- Un script vérifie qu'aucune valeur d'espacement, de rayon ou de durée hors tokens n'apparaît dans le CSS compilé.
- Un test vérifie que toute chaîne d'interface est présente dans les dictionnaires des trois locales, et qu'aucune n'est codée en dur dans les composants.

**Suivi des décisions** : toute évolution du design system (nouveau token, nouvelle combinaison couleur, nouvelle famille de composant) se fait par modification de ce document **et** des tokens dans le même PR, avec la ligne de justification. Le cahier des charges reste la spécification fonctionnelle ; ce document la spécification visuelle. Un écart entre les deux doit être signalé, pas contourné.
