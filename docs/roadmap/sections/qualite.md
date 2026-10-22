## Qualite, tests et process de qualite

### 24.1 Philosophie de tests

**Regle fondatrice.** Une fonctionnalite n'est pas finie parce qu'elle marche a la main (spec §24, §24.6). Une fonctionnalite est finie quand un test automatise reproduit son comportement attendu, qu'un reviewer independant a lu ce test et qu'il est vert en CI. Le spec le dit explicitement : « Les tests sont une exigence de livraison » (§24).

**Ce que « fini » veut dire.** Chaque branche de code livré est couverte par au moins un test automatise qui echouerait si la branche etait cassee. Une tache sans test est une tache non finie, pas une tache « a tester plus tard ». Le test est ecrit avant ou avec l'implementation (TDD sur la logique metier), jamais apres pour documenter a posteriori.

**Anti-patterns interdits (revue de code rejette la PR).**

| Anti-pattern | Pourquoi c'est un bug | Ce qu'on fait a la place |
|---|---|---|
| Tester l'implementation (assertion sur un mock interne, sur le nombre d'appels a un mock) | Refactor legitime casse le test sans changer le comportement | Tester le comportement observable : sortie, effet de bord persistant, code HTTP |
| E2E pour tout | 40 scenarios navigateur = 12 min/run, instables, impossible a diagnostiquer | Descendre au plus bas niveau de la pyramide ce qui est prouvable sans navigateur |
| Mocker ce qu'on ne maitrise pas | Un mock fige la believede l'API : le test passe, la prod casse (webhook, Stripe, transporteur) | Mocker uniquement nos dependances internes. Prestataires = sandbox ou faux serveur local (wire-level), jamais une fixture figee |
| `sleep(3000)` / `waitForTimeout` | Course conditionnelle : le test est vert sur une machine rapide, rouge en CI | Attendre une **condition** : etat d'element, reponse reseau, changement d'URL. `expect(locator).toBeVisible()` attend deja |
| `test.only` commit | Le reste de la suite ne tourne plus en CI | Interdit par ESLint + hook pre-commit |
| Snapshot approuve sans relecture | Le diff de snapshot montre la regression, la relecture montre l'intention | Chaque snapshot approuve est valide par un reviewer (voir §24.6) |
| Couverture haute, tests vides | 95 % de lignes couvertes par des `expect(true)` | Mutation testing sur la logique critique (§24.9) |

**Pyramide appliquee a Fanours.** Le spec est un catalogue B2B international : le risque n'est pas « le bouton marche », il est « le client a paye 12 000 EUR au lieu de 1 200 » ou « deux clients ont achete la derniere unite ». Donc :

```text
              /\                 E2E Playwright   ~5 % des tests, 12-20 scenarios bloquants
             /  \                parcours nominaux de bout en bout, 3 langues x 2 themes x 8 viewports
            /    \               Integrations       ~25 %, API + PostgreSQL reel + worker + outbox
           /      \              webhooks, outbox, jobs, PDF, import/export, concurrence
          /        \             Unitaires          ~70 %, logique pure : prix, douane, etats, authz
         /__________\            Vitest, sans base, sans reseau, < 2 s la suite entiere
```

**Pourquoi cette repartition.** Trois modules concentrent 80 % du risque financier et juridique et justifient integration + E2E :

- **Pricing** (§7.2, §8.4) : paliers, priorites, arrondis. Unitaire exhaustif + mutation testing (95 % de mutation score). Le calcul pur est parfait pour l'unitaire : pas de base, pas de reseau.
- **Trade compliance / tax** (§11.2, §11.3) : moteur de regles versionne. Unitaire pour la selection de regle (date, priorite, expiration) + integration pour la persistance des versions et l'alerte de revalidation. E2E uniquement pour l'affichage des incertitudes dans le checkout.
- **Payment / order state** (§9.3, §9.5, §10) : machine a etats, idempotence, webhooks. Unitaire pour les transitions, integration pour la transaction SQL et l'outbox, E2E pour le parcours nominal de paiement.

**Regle de placement (arbitrage en 30 secondes).** « Est-ce que ce test necessite un navigateur ? » Si non, il n'a rien a faire dans `e2e/`. « Est-ce que ce test necessite une vraie base ? » Si non, il appartient a l'unitaire avec une factory en memoire. Exception assumee : tout ce qui touche au **rendez-vous visuel** (spec §22.1, §24.4) et au **comportement multi-onglets / multi-cookies** (panier invite + fusion apres connexion, §9.1) reste en E2E meme si un equivalent existe ailleurs.

**Matrice de correspondance spec -> type de test.**

| Spec | Module | Unitaire | Integration | E2E |
|---|---|---|---|---|
| §7.2 paliers de prix | `pricing/price-tiers` | oui | snapshot en commande | affichage palier actif |
| §7.3 stock & reservations | `inventory/reservation` | expiration | decrement atomique | - |
| §9.3 etats de commande | `orders/state-machine` | oui (graphe complet) | transition + audit | parcours nominal |
| §9.5 idempotence | `idempotency` | dedoublonnage | cle unique SQL | double soumission |
| §10.3 webhooks | `payments/webhook` | signature | rejeu, ordre | paiement simule |
| §10.4 remboursements | `payments/refund` | transition | double remboursement | - |
| §11.2 regles douanieres | `compliance/rule-engine` | selection | versionnage + alerte | affichage incertitude |
| §11.3 calcul | `compliance/calculator` | 6 cas chiffres | - | recapitulatif checkout |
| §13 auth / 2FA | `auth` | matrice RBAC | session, MFA | login, MFA admin |
| §19.2 controle d'acces | `authz` | matrice action x role | IDOR reel | route admin refusee |
| §22.4 accessibilite | - | - | - | axe + manuel |
| §26 criteres d'acceptation | - | - | - | scenarios dedies |

---

### 24.2 Tests unitaires

**Outils.** Vitest 3 (moteur Vite, compatible TS/ESM, `expect` style Chai-like mais avec assertions Jest-compatible), `@vitest/coverage-v8`, `vitest-dom` pour le DOM, `@testing-library/react` + `@testing-library/user-event` pour les composants. Vitest est choisi sur Jest parce que la stack est Next.js/Vite-native (spec §5.1) et que le meme runner sert les tests de composants et de logique, ce qui evite deux runners a maintenir.

**Organisation.** Un fichier de test colocé : `src/modules/<module>/__tests__/<fichier>.spec.ts`. Les tests de composants : `src/components/<x>/__tests__/<x>.spec.tsx`. Aucun dossier `tests/` global qui disperse la logique.

```ts
// vitest.config.ts — runner unique, seuils globaux
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',              // 'jsdom' uniquement via @vitest-environment
    include: ['src/**/*.spec.ts', 'src/**/*.spec.tsx'],
    exclude: ['**/e2e/**', '**/node_modules/**', '**/.next/**'],
    setupFiles: ['src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text-summary', 'lcov', 'html'],
      thresholds: { lines: 80, functions: 80, branches: 80, statements: 80 },
      include: ['src/modules/**', 'src/lib/**'],
      exclude: ['**/*.d.ts', '**/index.ts', '**/types.ts'],
    },
  },
});
```

Un second fichier `vitest.integration.config.ts` isole le runner d'integration (DB, workers) : `environment: 'node'`, `setupFiles: ['src/test/integration.setup.ts']`, `include: ['src/**/__tests__/**/*.integration.spec.ts']`, `pool: 'forks'`, `poolOptions: { forks: { singleFork: false } }`, `fileParallelism: false` si la base est partagee (voir §24.3).

**Style d'ecriture.** `describe` = unite de comportement, `it` = un cas observable. Commentaire `// given / // when / // then` systematique sur les cas non triviaux. Pas de `expect` sur des valeurs intermediaires non contractuelles. Arrange/act/assert en 3 blocs separes par une ligne vide.

**Modules a couvrir en unitaire, cas par cas.**

1. **MOQ et multiples de commande** (`src/modules/catalog/moq.ts`) — `refuse une quantite inferieure au MOQ global`, `accepte une quantite exactement egale au MOQ`, `refuse une quantite non multiple de l increment`, `accepte une quantite multiple exacte`, `applique le MOQ de variante quand il existe`, `le MOQ global s applique quand la variante n en a pas`, `refuse la quantite zero et les quantites negatives`, `refuse une quantite au-dessus du plafond configure`, `signale un MOQ different entre canal B2B et B2C`.

2. **Paliers de prix** (`src/modules/pricing/price-tiers.ts`) — `applique le palier 1-9 pour la quantite 9 (borne haute inclusive)`, `bascule au palier 10 pour la quantite 10 (borne basse inclusive)`, `applique le dernier palier ouvert au-dela de 99`, `refuse un palier dont les bornes se chevauchent`, `refuse un palier dont les bornes laissent un trou`, `recalcule immediatement le palier quand la quantite change`, `applique le palier au niveau SKU et pas au niveau variante`, `applique le palier au niveau variante quand le SKU est configure ainsi`, `applique le palier au niveau ensemble de lignes (panier) quand le produit le declare`, `refuse un prix negatif ou nul`, `refuse un palier dont la devise differe de la devise de reference`, `ignore un palier dont la date de validite n est pas atteinte`, `ignore un palier expire a la date de la commande`.

3. **Priorite des prix** (`src/modules/pricing/price-resolution.ts`) — `le prix client specifique prime sur le prix de groupe`, `le prix de groupe prime sur le prix standard`, `le prix standard prime sur la promotion`, `la promotion ne s applique pas au-dela de sa date d expiration`, `le prix client expire est ignore et le prix groupe est applique`, `un prix client pour un autre produit ne s applique pas`, `une promotion limitee a un pays ne s applique pas a une autre destination`, `le prix resolu conserve la source et la regle appliquee (tracabilite §7.2)`.

4. **Arrondis monetaires** (`src/modules/money/money.ts`) — cf. code ci-dessous. Cas obligatoire qui casse avec un flottant : `19.99 + 19.99 + 19.99` vaut `59.97000000000001` en IEEE-754, et `Math.round(0.145 * 100) / 100` vaut `0.14` au lieu de `0.15`. En unites mineures entieres, aucune de ces deux expressions n'existe.

5. **Machine a etats des commandes** (`src/modules/orders/state-machine.ts`) — `autorise draft -> pending_payment`, `autorise pending_payment -> paid via webhook`, `autorise pending_payment -> payment_failed`, `refuse paid -> draft (regression metier interdite)`, `refuse paid -> cancelled sans workflow de remboursement`, `refuse shipped -> cancelled (spec §9.4)`, `refuse shipped sans Shipment associe (spec §9.3)`, `refuse refunded -> paid`, `autorise confirmed -> processing -> partially_fulfilled -> shipped`, `autorise shipped -> in_transit -> customs_action_required -> delivered -> completed`, `refuse deletion d une commande payee (spec §9.3)`, `refuse une transition inconnue non listee dans le graphe`, `enregistre un OrderStatusEvent immuable a chaque transition`.

6. **Droits et TVA par pays** (`src/modules/compliance/calculator.ts`) — les 6 cas chiffres de la table ci-dessous.

7. **Selection de regle douaniere** (`src/modules/compliance/rule-engine.ts`) — `selectionne la regle specifique au produit plutot que la regle generique du pays`, `selectionne la regle par priorite quand plusieurs regles du meme niveau matchent`, `utilise la version de regle en vigueur a la date de la commande et pas la derniere version`, `ignore une regle expiree et retourne un resultat incomplet`, `ignore une regle dont la date de debut n est pas atteinte`, `bascule sur une regle de remplacement lors d un changement de date d effet`, `retourne REVUE_MANUELLE si le code HS du produit n est pas valide (spec §11.5)`, `ne confond pas pays de destination et pays d origine`, `leve une alerte de revalidation quand la date de revue d une regle est depassee (spec §11.2)`.

8. **Validation des entrees (zod)** (`src/modules/*/schemas.ts`) — `refuse un email invalide`, `refuse une quantite non entiere ou negative`, `refuse un code devise inconnu`, `refuse une adresse de pays non livre`, `refuse une surcharge de proprietes non declarees (strip unknown)`, `produit un message d erreur localise par locale`, `accepte un payload minimal valide`.

9. **Autorisations par role** (`src/modules/authz/permissions.ts`) — matrice action x role : `support_agent peut lire une commande mais pas rembourser`, `finance_manager peut rembourser`, `catalog_manager peut modifier un prix mais pas un code HS valide`, `compliance_manager seul peut valider un code HS`, `analyst_readonly ne peut rien ecrire`, `super_admin peut tout`, `un utilisateur ne peut pas elever son propre role`, `la verification de propriete est obligatoire (IDOR, spec §19.2)`.

10. **Idempotence** (`src/modules/idempotency/`) — `dedoublonne une cle repetee`, `retourne le resultat memorise pour la meme cle`, `rejette une cle reutilisee avec un payload different`, `liberer la cle apres expiration`, `refuse une requete sans cle sur une operation sensible (spec §9.5)`.

11. **Localisation** (`src/modules/i18n/`) — `formate un montant selon la locale FR (1 234,56 EUR)`, `formate un montant selon la locale EN (1,234.56 EUR)`, `formate un montant selon la locale ZH-CN`, `convertit un timestamp UTC vers le fuseau du lieu logistique`, `n affiche jamais l heure de l utilisateur en UTC non explicite`, `respecte la langue de la commande pour la facture (spec §4.2)`, `signale une cle de traduction manquante au lieu de renvoyer la cle brute`.

**Code reel — paliers de prix.**

```ts
// src/modules/pricing/__tests__/price-tiers.spec.ts
import { describe, it, expect } from 'vitest';
import { resolvePriceTier, PriceTierError } from '../price-tiers';
import type { PriceTier } from '../types';

describe('resolvePriceTier — spec §7.2', () => {
  const tiers: PriceTier[] = [
    { minQuantity: 1,   maxQuantity: 9,   unitPriceMinor: 1500, currency: 'EUR' },
    { minQuantity: 10,  maxQuantity: 49,  unitPriceMinor: 1200, currency: 'EUR' },
    { minQuantity: 50,  maxQuantity: 99,  unitPriceMinor: 1050, currency: 'EUR' },
    { minQuantity: 100, maxQuantity: null, unitPriceMinor: 900,  currency: 'EUR' },
  ];

  it('applique le palier 1-9 pour une quantite de 9 (borne haute inclusive)', () => {
    // given
    const quantity = 9;
    // when
    const result = resolvePriceTier(tiers, quantity, { scope: 'sku' });
    // then
    expect(result.unitPriceMinor).toBe(1500);
    expect(result.tier.minQuantity).toBe(1);
    expect(result.tier.maxQuantity).toBe(9);
  });

  it('bascule au palier 10 pour une quantite de 10 (borne basse inclusive)', () => {
    const result = resolvePriceTier(tiers, 10, { scope: 'sku' });
    expect(result.unitPriceMinor).toBe(1200);
  });

  it('applique le dernier palier ouvert au-dela de 99', () => {
    const result = resolvePriceTier(tiers, 250, { scope: 'sku' });
    expect(result.unitPriceMinor).toBe(900);
    expect(result.tier.maxQuantity).toBeNull();
  });

  it('refuse un palier dont les bornes se chevauchent', () => {
    const overlapping = [
      { minQuantity: 1,  maxQuantity: 10, unitPriceMinor: 1500, currency: 'EUR' },
      { minQuantity: 10, maxQuantity: 20, unitPriceMinor: 1200, currency: 'EUR' },
    ] as PriceTier[];
    expect(() => resolvePriceTier(overlapping, 10, { scope: 'sku' })).toThrow(PriceTierError);
  });

  it('ignore un palier expire a la date de la commande', () => {
    const withValidity = [
      { minQuantity: 1, maxQuantity: null, unitPriceMinor: 1500, currency: 'EUR',
        validFrom: '2026-01-01', validUntil: '2026-06-30' },
    ] as PriceTier[];
    expect(() =>
      resolvePriceTier(withValidity, 1, { scope: 'sku', at: '2026-07-15' }),
    ).toThrow(PriceTierError);
  });
});
```

**Code reel — transition de commande.**

```ts
// src/modules/orders/__tests__/state-machine.spec.ts
import { describe, it, expect } from 'vitest';
import { canTransition, assertTransition, InvalidTransitionError } from '../state-machine';
import type { OrderStatus } from '../types';

describe('machine a etats des commandes — spec §9.3', () => {
  it('autorise la transition pending_payment -> paid', () => {
    expect(canTransition('pending_payment', 'paid')).toBe(true);
  });

  it('refuse la transition paid -> draft', () => {
    expect(canTransition('paid', 'draft')).toBe(false);
    expect(() => assertTransition('paid', 'draft')).toThrow(InvalidTransitionError);
  });

  it('refuse shipped -> cancelled (spec §9.4)', () => {
    expect(() => assertTransition('shipped', 'cancelled')).toThrow(InvalidTransitionError);
  });

  it('refuse shipped sans Shipment associe (spec §9.3)', () => {
    const ctx = { status: 'shipped' as OrderStatus, shipment: null };
    expect(() => assertTransition('shipped', 'in_transit', ctx)).toThrow(/shipment/i);
  });

  it('autorise la chaine complete confirmed -> ... -> completed', () => {
    const chain: OrderStatus[] = [
      'confirmed', 'processing', 'partially_fulfilled', 'shipped',
      'in_transit', 'customs_action_required', 'delivered', 'completed',
    ];
    for (let i = 0; i < chain.length - 1; i++) {
      expect(canTransition(chain[i], chain[i + 1])).toBe(true);
    }
  });

  it('refuse la suppression d une commande payee (spec §9.3)', () => {
    expect(() => assertTransition('paid', 'deleted')).toThrow(InvalidTransitionError);
  });
});
```

**Cas chiffres — droits et TVA par pays (spec §11.3, 6 cas).**

Modele : `baseTaxable = merchandise - discount + (shipping ? shipping : 0) + (insurance ? insurance : 0)`, `vat = roundHalfUp(baseTaxable * rateBps / 10000)` en centimes, `duty = roundHalfUp(goodsValue * dutyRateBps / 10000)`. Les frais de courtage sont **hors base** et **hors facture Fanours** (payes au transporteur).

| # | Pays / regle | Donnees | Base taxable | TVA | Droits | Courtage | Total facture Fanours | Total a l arrivee |
|---|---|---|---|---|---|---|---|---|
| 1 | FR, seuil de deduplication 150 EUR, TVA 20 % | marchandises 120,00 / port 30,00 | — (sous seuil) | 0 | 0 | 0 | 150,00 | 0,00 (mention « a confirmer ») |
| 2 | FR, TVA 20 %, base marchandises + port | marchandises 200,00 / port 30,00 | 230,00 | 46,00 | 0 | 0 | 276,00 | 0,00 |
| 3 | FR, TVA 20 %, assurance incluse dans la base (§11.3.3) | marchandises 200,00 / port 30,00 / assurance 20,00 | 250,00 | 50,00 | 0 | 0 | 300,00 | 0,00 |
| 4 | US, duty 5 %, pas de TVA collectee, courtage forfaitaire 12,50 | marchandises 500,00 / port 40,00 | 500,00 | 0 | 25,00 | 12,50 | 540,00 | 77,50 |
| 5 | Regle expiree (validUntil < date commande) | marchandises 500,00 / port 40,00 | — | — | — | — | 540,00 | « a confirmer par le transporteur » + alerte conformite |
| 6 | FR -> DE, B2B intracommunautaire, autoliquidation | marchandises 300,00 / port 25,00 | 325,00 | 0,00 (autoliquidation) | 0 | 0 | 325,00 | 0,00 — mention « TVA a recuperer par le client » |

Regle transverse testee : `ne double-compte jamais une taxe deja collectee` — si une regle prevoit la collecte au checkout ET que le pays la reclame a l arrivee, le calcul leve un conflit explicite et le checkout affiche les deux lignes separement (spec §11.3, §11.4).

---

### 24.3 Tests d'integration

**Outils.** Vitest (runner d'integration) + **PostgreSQL reel** + Redis reel + worker de jobs. Le spec exige que la logique financiere soit verifiee contre le vrai moteur (§24.2, §24.6, §26.2) : un SQLite ou un mock de l'ORM (Drizzle) ne prouve rien sur les contraintes, les verrous et les `CHECK`.

**Choix de la base de test : tranche et justification.**

| Option | Verdict | Raisonnement |
|---|---|---|
| **Testcontainers (PostgreSQL 16)** | **Retenu pour la CI** | Version identique a la prod, `CHECK`/`UNIQUE`/`SERIALIZABLE` reellement exerces, isolation par conteneur jetable, aucun state partage, pas de cout. Indisponible nativement sous Windows sans Docker Desktop : le developpeur local utilise le service CI ou une instance Docker locale. |
| **Neon branch (PITR)** | Retenu pour les tests de non-regression de schema et le staging | Branche jetable + point-in-time-restore = isolation forte, pas de conteneur a demarrer. Mais : cout par branche, latence reseau (inacceptable en unitaires), et le schema derive peut diverger de la prod si la branche n est pas recreee. **Interdit** pour les tests de concurrence (latence = faux positifs). |
| **PostgreSQL local partage (un dev, N bases)** | Retenu en developpement | Rapide, gratuit. Chaque worker a sa base `fanours_test_<workerId>`, schemas distincts dans la meme instance. |
| **PgBouncer / pool partage** | Ecarte | Le pooling casse l'isolation transactionnelle si deux tests partagent une connexion. |

Regle : **jamais de base de developpement pour un test d'integration** (§25.1 — « Aucune donnee reelle de client ne doit etre copiee en developpement »).

**Strategie de nettoyage.** Deux mecanismes, choisis selon le test :

- **Rollback par test** (defaut) : le test s'execute dans une transaction ouverte, `afterEach` fait `ROLLBACK`. Rapide, isolation parfaite. Incompatible avec les tests qui doivent valider la visibilite d'une transaction ailleurs (outbox, worker, webhook) : ceux-la utilisent le mode **truncate**.
- **Truncate par test** (mode `isolation: 'truncate'`) : la transaction est commitee, puis toutes les tables applicatives sont tronquees et les sequences reinitialisees. Utilise quand le comportement a tester implique une autre connexion (worker qui lit l'outbox, webhook qui arrive dans un autre process, test de concurrence avec `Promise.all` sur deux connexions).

```ts
// src/test/integration.setup.ts
import { afterEach, beforeAll, afterAll } from 'vitest';
import { db } from '@/lib/db';

beforeAll(async () => { await db.$connect(); });
afterAll(async () => { await db.$disconnect(); });

afterEach(async () => {
  // rollback automatique : Vitest execute chaque fichier dans une transaction si
  // le helper `withTx` est utilise. Sinon truncate.
  await db.$executeRawUnsafe('TRUNCATE TABLE "OrderLine","Order","StockReservation","InventoryItem","PriceTier","Product","OutboxEvent","AuditLog","WebhookEvent","BackgroundJob" RESTART IDENTITY CASCADE');
});
```

**Factory pattern.** Une factory par entite, avec des valeurs par defaut saines et une surcharge partielle. Les factories sont versionnees avec le schema : un changement de colonne casse le type de la factory, pas seulement les tests.

```ts
// src/test/factories/product.factory.ts
import { faker } from '@faker-js/faker';
import { db } from '@/lib/db';

type ProductOverrides = Partial<{
  moq: number; orderIncrement: number; stockAvailable: number;
}>;

export async function createProduct(overrides: ProductOverrides = {}) {
  const sku = `SKU-${faker.string.alphanumeric(10).toUpperCase()}`;
  return db.product.create({
    data: {
      sku,
      slug: faker.helpers.slug('product'),
      moq: overrides.moq ?? 12,
      orderIncrement: overrides.orderIncrement ?? 6,
      stockAvailable: overrides.stockAvailable ?? 100,
      salesModes: ['wholesale', 'retail'],
      priceTiers: {
        create: [
          { minQuantity: 1,  maxQuantity: 11,  unitPriceMinor: 1500, currency: 'EUR' },
          { minQuantity: 12, maxQuantity: 47,  unitPriceMinor: 1200, currency: 'EUR' },
          { minQuantity: 48, maxQuantity: null, unitPriceMinor: 900, currency: 'EUR' },
        ],
      },
    },
  });
}
```

**Cas d'integration obligatoires (spec §24.2).**

| Domaine | Cas | Assertion cle |
|---|---|---|
| API + DB | creation de commande | 201, `Order` + `OrderLine` + snapshot `priceSnapshot`/`taxSnapshot`/`conditionsSnapshot` (spec §20.2), ligne `AuditLog` ecrite |
| API + DB | recalcul checkout apres changement d'adresse (spec §24.2) | les `ShippingQuote`/`TaxEstimate` de la commande **precedente** ne sont pas modifies ; le nouveau calcul ecrase le recapitulatif affiche |
| Transactions | stock jamais negatif (spec §26.2) | `CHECK (stock_available >= 0)` + exception metier |
| Transactions | quantite > 0, montant > 0 | contrainte `CHECK` SQL, pas seulement la validation applicative |
| Webhook paiement | signature valide | commande `paid`, evenement `payment.succeeded` publie via outbox |
| Webhook paiement | signature invalide | 400, **aucun** changement d'etat, trace d'echec |
| Webhook paiement | webhook duplique | second appel = no-op (200), un seul `PaymentEvent`, une seule facture |
| Webhook paiement | webhook hors ordre (`payment.succeeded` recu apres `payment.failed` tardif) | l'etat le plus avance l'emporte ; l'ancien evenement est rejete et trace |
| Webhook paiement | horodatage hors tolerance (tolerance 5 min) | rejete, alerte integrite |
| Outbox | publication d'evenement | `OutboxEvent` ecrit **dans la meme transaction** que la mutation, relaye par le worker |
| Outbox | reprise apres echec | message en echec -> backoff -> retry -> livree une seule fois (idempotence du consumer) |
| File de jobs | job idempotent | rejeu du meme job (cle `idempotencyKey`) = pas de doublon d'email ni de PDF |
| Facture PDF | generation en FR / EN / ZH-CN | 3 fichiers, polices CJK embarquees, montants aux formats de chaque locale, texte extractible (pas une image) |
| Import/export catalogue | schema invalide | rapport d'erreur **par ligne** (numero de ligne + champ + message), 0 ecriture |
| Import/export catalogue | import partiel valide | lignes valides importees, lignes invalides rejetees, rapport produit |
| Import/export catalogue | annulation | import annule tant qu'aucune consequence irreversible ; champs de conformite et prix valides jamais ecrases silencieusement (spec §7.4) |
| Reservation | expiration | reservation `expired`, stock libere, mouvement d'inventaire trace |
| Reservation | double expiration | second passage du job = no-op, pas de double liberation |
| Concurrence | 2 clients, derniere unite | 1 succes, 1 conflit, stock final 0 (voir §24.5) |

**Exemple de code reel 1 — creation de commande + snapshots + audit.**

```ts
// src/modules/orders/__tests__/create-order.integration.spec.ts
import { describe, it, expect, beforeEach } from 'vitest';
import request from 'supertest';
import { app } from '@/app';
import { db } from '@/lib/db';
import { createProduct } from '@/test/factories/product.factory';

describe('POST /api/orders — spec §9.2, §20.2, §19.7', () => {
  beforeEach(async () => {
    await db.webhookEvent.deleteMany();
    await db.outboxEvent.deleteMany();
    await db.auditLog.deleteMany();
    await db.order.deleteMany();
  });

  it('persiste la commande avec snapshot prix/taxes/conditions et ecrit l audit', async () => {
    // given
    const product = await createProduct({ moq: 12, orderIncrement: 6, stockAvailable: 10 });
    const payload = {
      items: [{ productId: product.id, quantity: 12 }],
      shippingAddress: { country: 'FR', city: 'Lyon', postalCode: '69001', line1: '12 rue Test' },
    };
    const key = 'idem-key-001';

    // when
    const res = await request(app).post('/api/orders').set('Idempotency-Key', key).send(payload);

    // then
    expect(res.status).toBe(201);
    const order = await db.order.findUniqueOrThrow({ where: { id: res.body.id } });
    expect(order.priceSnapshot).toMatchObject({ unitPriceMinor: 1200, tierMin: 12, currency: 'EUR' });
    expect(order.taxSnapshot).toMatchObject({ country: 'FR' });
    expect(order.conditionsSnapshot).toHaveProperty('acceptedAt');
    const audit = await db.auditLog.findFirst({ where: { entityId: order.id, action: 'order.created' } });
    expect(audit).not.toBeNull();
  });

  it('dedoublonne la double soumission avec la meme cle d idempotence', async () => {
    const product = await createProduct({ stockAvailable: 50 });
    const payload = { items: [{ productId: product.id, quantity: 12 }], shippingAddress: { country: 'FR' } };
    const key = 'idem-key-002';

    const [a, b] = await Promise.all([
      request(app).post('/api/orders').set('Idempotency-Key', key).send(payload),
      request(app).post('/api/orders').set('Idempotency-Key', key).send(payload),
    ]);

    expect([a.status, b.status].sort()).toEqual([201, 201]);
    expect(a.body.id).toBe(b.body.id);
    expect(await db.order.count({ where: { idempotencyKey: key } })).toBe(1);
  });
});
```

**Exemple de code reel 2 — webhook de paiement (signature, doublon, hors ordre).**

```ts
// src/modules/payments/__tests__/webhook.integration.spec.ts
import { describe, it, expect } from 'vitest';
import crypto from 'node:crypto';
import request from 'supertest';
import { app } from '@/app';
import { db } from '@/lib/db';

const SECRET = 'whsec_test_secret';

function sign(payload: string, timestamp: number): string {
  return crypto.createHmac('sha256', SECRET).update(`${timestamp}.${payload}`).digest('hex');
}

describe('POST /api/webhooks/payment — spec §10.3, §9.5, §24.2', () => {
  it('rejette une signature invalide sans modifier la commande', async () => {
    const order = await createPendingOrder();
    const body = JSON.stringify({ type: 'payment.succeeded', data: { orderId: order.id, amountMinor: 120000, currency: 'EUR' } });
    const res = await request(app).post('/api/webhooks/payment')
      .set('x-webhook-timestamp', '1700000000').set('x-webhook-signature', 'deadbeef').send(body);

    expect(res.status).toBe(400);
    const after = await db.order.findUniqueOrThrow({ where: { id: order.id } });
    expect(after.status).toBe('pending_payment');
  });

  it('accepte une signature valide et fait passer la commande a paid une seule fois', async () => {
    const order = await createPendingOrder();
    const body = JSON.stringify({ type: 'payment.succeeded', data: { orderId: order.id, amountMinor: 120000, currency: 'EUR' } });
    const ts = Math.floor(Date.now() / 1000);

    const first = await request(app).post('/api/webhooks/payment')
      .set('x-webhook-timestamp', String(ts)).set('x-webhook-signature', sign(body, ts)).send(body);
    const second = await request(app).post('/api/webhooks/payment')
      .set('x-webhook-timestamp', String(ts)).set('x-webhook-signature', sign(body, ts)).send(body);

    expect(first.status).toBe(200);
    expect(second.status).toBe(200);
    expect(await db.paymentEvent.count({ where: { orderId: order.id } })).toBe(1);
    const after = await db.order.findUniqueOrThrow({ where: { id: order.id } });
    expect(after.status).toBe('paid');
  });

  it('ignore un evenement hors ordre (echec tardif apres succes)', async () => {
    const order = await createPendingOrder();
    const success = JSON.stringify({ type: 'payment.succeeded', data: { orderId: order.id } });
    const failure = JSON.stringify({ type: 'payment.failed',  data: { orderId: order.id } });
    const ts = Math.floor(Date.now() / 1000);

    await request(app).post('/api/webhooks/payment')
      .set('x-webhook-timestamp', String(ts)).set('x-webhook-signature', sign(success, ts)).send(success);
    const late = await request(app).post('/api/webhooks/payment')
      .set('x-webhook-timestamp', String(ts)).set('x-webhook-signature', sign(failure, ts)).send(failure);

    expect(late.status).toBe(200);
    const after = await db.order.findUniqueOrThrow({ where: { id: order.id } });
    expect(after.status).toBe('paid');
  });
});
```

**Exemple de code reel 3 — expiration de reservation et liberation de stock.**

```ts
// src/modules/inventory/__tests__/reservation-expiry.integration.spec.ts
import { describe, it, expect } from 'vitest';
import { db } from '@/lib/db';
import { expireReservations } from '../reservation';

describe('expireReservations — spec §7.3, §24.2', () => {
  it('libere le stock et journalise le mouvement a expiration', async () => {
    // given
    const item = await db.inventoryItem.create({ data: { sku: 'SKU-X', stockAvailable: 5, stockReserved: 0 } });
    await db.stockReservation.create({
      data: { inventoryItemId: item.id, quantity: 3, status: 'active', expiresAt: new Date(Date.now() - 1000) },
    });

    // when
    const result = await expireReservations();

    // then
    expect(result.expired).toBe(1);
    const after = await db.inventoryItem.findUniqueOrThrow({ where: { id: item.id } });
    expect(after.stockReserved).toBe(0);
    expect(after.stockAvailable).toBe(5);
    const movement = await db.inventoryMovement.findFirst({ where: { inventoryItemId: item.id, reason: 'reservation_expired' } });
    expect(movement).not.toBeNull();
  });

  it('est idempotent : un second passage ne libere pas deux fois', async () => {
    const item = await db.inventoryItem.create({ data: { sku: 'SKU-Y', stockAvailable: 5, stockReserved: 0 } });
    await db.stockReservation.create({
      data: { inventoryItemId: item.id, quantity: 3, status: 'active', expiresAt: new Date(Date.now() - 1000) },
    });

    await expireReservations();
    const second = await expireReservations();

    expect(second.expired).toBe(0);
    const after = await db.inventoryItem.findUniqueOrThrow({ where: { id: item.id } });
    expect(after.stockAvailable).toBe(5);
  });
});
```

---

### 24.4 Tests E2E Playwright

**Outils.** Playwright (`@playwright/test`), `axe-core` + `@axe-core/playwright`, `pixelmatch` (non requis si on utilise `toHaveScreenshot`). Les tests E2E sont dans `e2e/`, jamais dans `src/`.

**Configuration et discipline de parallelisation.**

```ts
// playwright.config.ts  (RACINE du depot -- un seul fichier de configuration, cree en Phase 1,
// modifie ensuite par les Phases 2, 4, 5 et 6 ; aucun config dans apps/ ni dans e2e/)
import { defineConfig } from '@playwright/test';

const isCI = !!process.env.CI;

/** Les 5 suites de release du spec §24.3. Chaque suite = UN fichier, un projet. */
const RELEASE_SUITES = {
  b2c: '**/b2c/journey.spec.ts',
  b2b: '**/b2b/journey.spec.ts',
  payment: '**/payment/scenarios.spec.ts',
  admin: '**/admin/dashboard.spec.ts',
  security: '**/security/security.spec.ts',
} as const;

export default defineConfig({
  testDir: './e2e',                           // UN SEUL testDir pour tous les projets
  fullyParallel: true,                       // chaque fichier est un worker independant
  forbidOnly: isCI,                          // test.only interdit en CI
  retries: isCI ? 2 : 0,                     // 2 retries en CI = signal de flake, pas un traitement
  workers: isCI ? 4 : undefined,
  timeout: 45_000,                           // par test ; l'API est rapide, 45 s suffit
  globalTimeout: 4 * 60 * 60 * 1000,         // plafond de l'execution complete
  expect: { timeout: 10_000 },
  reporter: [['html', { open: 'never' }], ['github'], ['junit', { outputFile: 'test-results/junit.xml' }]],
  outputDir: 'test-results/artifacts',
  use: {
    baseURL: process.env.BASE_URL ?? 'http://127.0.0.1:3000',
    trace: 'on-first-retry',                 // trace conservee au 1er retry : l'artefact de debug n°1
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    actionTimeout: 15_000,
    navigationTimeout: 20_000,
  },
  projects: [
    // 5 suites de release (spec §24.3). Les fichiers apparaissent au fil des phases :
    // un projet sans fichier correspondant ne fait tourner que la matrice `functional`.
    { name: 'b2c',     testMatch: RELEASE_SUITES.b2c },
    { name: 'b2b',     testMatch: RELEASE_SUITES.b2b },
    { name: 'payment', testMatch: RELEASE_SUITES.payment },
    { name: 'admin',   testMatch: RELEASE_SUITES.admin },
    { name: 'security',testMatch: RELEASE_SUITES.security },
    // Matrice visuelle (8 viewports x 3 langues x 2 themes)
    { name: 'visual',  testMatch: ['**/visual/**/*.spec.ts', '**/visual-matrix.spec.ts'] },
    // Recettes fonctionnelles de developpement : tout le reste, une fois les 6 exclus
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
```

**Regles structurantes du modele E2E (valables pour les 6 phases).**

1. **Une seule configuration** : `playwright.config.ts` a la racine. Elle est **creee en Phase 1** et **modifiee** (jamais recreee) par les Phases 2, 4, 5 et 6. Aucun `playwright.config.ts` dans `apps/web/`, `apps/api/` ou `e2e/`. Toutes les commandes Playwright sont lancees **depuis la racine** (`pnpm exec playwright test ...`) ; le paquet `e2e/` ne porte que les sources de test, son `tsconfig.json` et son lint.
2. **Un seul `testDir`** (`.` = `e2e/`) pour tous les projets. Les projets se distinguent par `testMatch` / `testIgnore` uniquement : aucun projet ne reference un repertoire qui n'existe pas encore a la phase courante.
3. **Sept projets, un fichier = un projet.** Les 5 premiers sont les **suites de release** du spec §24.3 ; `visual` porte la matrice visuelle ; `functional` porte les recettes fonctionnelles de developpement de chaque phase. Un fichier appartient a **un seul** projet : pas de doublon, pas de comptage de tests ambigu.
4. **Une seule suite de release par cas du spec §24.3.** Le fichier est livre par la phase qui implemente la fonctionnalite, puis **repris** par la Phase 5 qui le porte au niveau release (donnees staging, isolation, 3 langues, tracabilite §24.3). La Phase 5 ne cree aucun doublon de suite.
5. **Les viewports, les langues et les themes ne sont pas des projets Playwright.** Ils sont parametres **dans les fichiers de test** (`test.use({ viewport })` dans un `test.describe` iteratif) a partir de la liste canonique exportee par `e2e/support/viewports.ts` (creee en Phase 1, importee et **jamais re-declaree** dans les Phases 2, 4, 5 et 6) : `320x640, 360x740, 390x844, 430x932, 768x1024, 1024x768, 1440x900, 1920x1080`. Ces 8 largeurs couvrent les 5 breakpoints du spec §2.5.
6. **Invocations** : recette de phase -> `pnpm exec playwright test <fichier> --project=functional` ; release -> `pnpm exec playwright test --project=<suite>` (`b2c|b2b|payment|admin|security`) ; matrice visuelle -> `pnpm exec playwright test --project=visual`.
7. **Navigateur** : mono-navigateur (Chromium) au lancement. Un second navigateur, si necessaire plus tard, s'ajoute comme **dimension de shard en CI**, pas comme un projet Playwright (le projet definit la *suite*, pas le moteur).

**Regles de parallelisation et d'isolation.**
- **Un worker = un contexte navigateur isole.** `test.use({ storageState })` par fichier pour l'authentification, un `storageState` par role, jamais partage entre workers.
- **Sharding** : `pnpm exec playwright test --project=<suite> --shard=${{ matrix.shard }}`. Le nombre de shards est une **dimension de CI**, pas une decision du modele de test : la CI lance 5 shards (un par suite de release) en Phase 5, puis la matrice `shard x projet x moteur` en Phase 6. Le repartitionnement est par **fichier** (par defaut) pour equilibrer la duree.
- **Isolation des donnees** : chaque test cree ses propres donnees via la factory/seed (§ ci-dessous). Aucun test ne depend du resultat d'un autre, aucun test ne depend d'un jeu de seed partage. Regle non negociable : `seed` ne doit jamais creer d'etat « attendu par un autre test ».
- **Timeouts** : 45 s par test, 10 s pour une assertion. Un test qui depasse est un test bloque (boucle, requete pendante) : on diagnostique, on n'augmente pas le timeout.
- **Retries** : `retries: 2` en CI, uniquement pour **mesurer** la flake (un test qui passe au 2e ou 3e essai = 1 flake). Objectif : 0 retry a terme. Un test qui ne passe qu'avec retry ne merge pas.
- **Artefacts sur echec** : trace + screenshot + video, archives dans l'artefact CI `test-results/`. Diagnostic : `npx playwright show-trace test-results/artifacts/<test>/trace.zip`.

**Fixture de page objet.**

```ts
// e2e/fixtures/pages.ts
import { test as base, expect, type Page, type Locator } from '@playwright/test';

export class CheckoutPage {
  readonly total: Locator;
  readonly placeOrderButton: Locator;

  constructor(private readonly page: Page) {
    this.total = page.getByTestId('checkout-total');
    this.placeOrderButton = page.getByRole('button', { name: /place order|passer commande|提交订单/i });
  }

  async fillShippingAddress(a: { country: string; line1: string; city: string; postalCode: string }) {
    await this.page.getByLabel(/country|pays|国家/i).selectOption(a.country);
    await this.page.getByLabel(/address|adresse|地址/i).fill(a.line1);
    await this.page.getByLabel(/city|ville|城市/i).fill(a.city);
    await this.page.getByLabel(/postal code|code postal|邮编/i).fill(a.postalCode);
    // on attend la fin du recalcul serveur : condition, jamais un sleep
    await expect(this.page.getByTestId('checkout-loading')).toBeHidden();
  }

  async expectTotalMinor(minor: number, currency: string) {
    await expect(this.total).toHaveText(new RegExp(`${minor / 100}[.,]${String(minor % 100).padStart(2, '0')}\\s*${currency}`));
  }

  async placeOrder() {
    await this.placeOrderButton.click();
    await this.page.waitForURL(/\/orders\//);
  }
}

export const test = base.extend<{ checkout: CheckoutPage; seeded: SeededData }>({
  seeded: async ({}, use) => { await use(await seedTestData()); },
  checkout: async ({ page }, use) => { await use(new CheckoutPage(page)); },
});

export { expect };
```

> Le pattern page-objet ci-dessus est **illustratif** : il montre la discipline attendue, pas un fichier impose. Les fixtures livrees par les plans d'execution sont `e2e/fixtures/test.ts` (Phase 1, test Playwright etendu + fixture `api`), `e2e/fixtures/totp.ts` (Phase 1, calcul TOTP RFC 6238), `e2e/fixtures/product.ts` (Phase 2), `e2e/b2c/fixtures.ts` (Phase 5), `e2e/admin/fixtures/admin-users.ts` (Phase 4) et `e2e/support/i18n.ts` (Phase 5, dictionnaires source de verite des assertions). Toute nouvelle fixture de page-objet est declaree dans le `**Files:**` de la tache qui la cree.

**Seed de donnees de test (deterministe, isole).**

```ts
// e2e/fixtures/seed.ts
import { db } from '@/lib/db';
import { faker } from '@faker-js/faker';

export type SeededData = {
  productId: string; sku: string; unitPriceMinor: number; moq: number; stockAvailable: number;
};

export async function seedTestData(): Promise<SeededData> {
  const sku = `E2E-${faker.string.alphanumeric(8).toUpperCase()}`;
  const product = await db.product.create({
    data: {
      sku,
      slug: `e2e-${faker.string.alphanumeric(8).toLowerCase()}`,
      moq: 12,
      orderIncrement: 6,
      stockAvailable: 100,
      salesModes: ['wholesale', 'retail'],
      name: { create: [
        { locale: 'fr-FR', name: 'Produit E2E' },
        { locale: 'en',    name: 'E2E Product' },
        { locale: 'zh-CN', name: 'E2E 产品' },
      ] },
      priceTiers: { create: [
        { minQuantity: 1,  maxQuantity: 11,  unitPriceMinor: 1500, currency: 'EUR' },
        { minQuantity: 12, maxQuantity: 47,  unitPriceMinor: 1200, currency: 'EUR' },
      ] },
    },
  });
  return { productId: product.id, sku, unitPriceMinor: 1200, moq: 12, stockAvailable: 100 };
}
```

Le seed ecrit directement en base de test (rapide, deterministe, aucun endpoint de production). Si un endpoint `e2e/seed` est utilise en CI, il est **desactive en production** par une verification d'environnement au demarrage, et journalise chaque appel.

**Scenario detaille (1 sur 20) — E2E-B2B-05/06 : devis -> commande, parcours B2B (spec §24.3, §8.3).**

```ts
// e2e/b2b/journey.spec.ts  (suite de release B2B -- fichier unique, 9 etapes)
import { test, expect } from '../fixtures';

test('E2E-B2B-05/06 : un devis accepte se convertit en commande sans ressaisie', async ({ page, seeded }) => {
  // given — un compte B2B approuve avec un prix contractuel
  const business = await createApprovedBusinessAccount({ priceRule: { unitPriceMinor: 980, currency: 'EUR' } });
  await page.context().addCookies(await sessionCookiesFor(business.userId));

  await page.goto(`/fr/products/${seeded.sku}`);

  // when — demande de devis
  await page.getByRole('button', { name: /demander un devis|request a quote/i }).click();
  await page.getByLabel(/quantit/i).fill('48');                       // palier 3 du seed
  await page.getByLabel(/destination|pays/i).selectOption('FR');
  await page.getByRole('button', { name: /envoyer la demande/i }).click();

  // then — le devis est cree et porte les conditions de l'instant
  await expect(page.getByRole('heading', { name: /devis|quote/i })).toBeVisible();
  const quoteNumber = await page.getByTestId('quote-number').innerText();
  expect(quoteNumber).toMatch(/^Q-2026-\d{4,}$/);
  await expect(page.getByTestId('quote-valid-until')).toBeVisible();

  // when — l'admin accepte le devis (session admin separee)
  const admin = await page.context().newPage();
  await admin.context().addCookies(await sessionCookiesFor(await createAdminUser()));
  await admin.goto(`/admin/quotes?search=${quoteNumber}`);
  await admin.getByRole('button', { name: /accept/i }).click();
  await admin.getByRole('button', { name: /confirm accept/i }).click();
  await expect(admin.getByTestId('quote-status')).toHaveText(/accepted|accepte/i);

  // when — le client accepte et convertit
  await page.reload();
  await page.getByRole('button', { name: /accepter le devis|accept quote/i }).click();
  await page.getByRole('button', { name: /convertir en commande|convert to order/i }).click();
  await page.waitForURL(/\/orders\//);

  // then — l'instantane du devis est conserve, pas un recalcul depuis le prix liste
  const orderId = page.url().split('/').pop();
  const order = await fetchOrderFromApi(orderId);
  expect(order.priceSnapshot.unitPriceMinor).toBe(980);              // prix du devis, pas 1200
  expect(order.quoteId).toBe(quoteNumber);
  expect(order.conditionsSnapshot.acceptedAt).toBeTruthy();
  await expect(page.getByTestId('order-total')).toContainText('47,04');  // 48 x 0,98 EUR
});
```

**Les 20 scenarios E2E prioritaires.**

| ID | Scenario | Spec | Famille |
|---|---|---|---|
| E2E-B2C-01 | Visite anonyme, bascule langue FR/EN/ZH-CN + theme clair/sombre, persistance apres rechargement | §24.3, §22.1 | B2C |
| E2E-B2C-02 | Recherche, filtres (categorie, MOQ, disponibilite), URL partageable, etat vide | §6.3, §18.1 | B2C |
| E2E-B2C-03 | Fiche produit : paliers affiches, MOQ, multiple refuse (11 -> message), stock, delai | §6.4, §7.2 | B2C |
| E2E-B2C-04 | Ajout au panier en mode retail, recalcul serveur du total, le palier actif est bien celui de la quantite | §9.1 | B2C |
| E2E-B2C-05 | Checkout : adresse internationale, frais calcules, **mentions de limite** de l'estimation douaniere | §11.4 | B2C |
| E2E-B2C-06 | Paiement simule reussi -> commande `confirmed`, snapshot prix/conditions present | §9.3, §26.2 | B2C |
| E2E-B2C-07 | Facture/recu telechargeable + notification e-mail recue (boite mail de test) | §14.5, §26.3 | B2C |
| E2E-B2C-08 | Suivi de commande depuis le compte : statut, evenements, colis | §12.2, §15.3 | B2C |
| E2E-B2B-01 | Inscription, verification e-mail par lien, connexion, session persistee | §13.1 | B2B |
| E2E-B2B-02 | Demande d'activation professionnelle : formulaire, depot de document, etat « en attente » | §8.2 | B2B |
| E2E-B2B-03 | Approbation par un role autorise (MFA admin) -> tarifs de gros et MOQ visibles, audit ecrit | §16.6, §19.2 | B2B |
| E2E-B2B-04 | Creation de devis -> negociation -> acceptation par un utilisateur autorise | §8.3 | B2B |
| E2E-B2B-05/06 | Conversion devis -> commande sans ressaisie, instantane conserve | §8.3, §26.2 | B2B |
| E2E-B2B-07 | Telechargement de la facture (3 langues) + suivi d'expedition | §14.5, §12.2 | B2B |
| E2E-PAY-01 | Paiement refuse et paiement en attente : messages localises, aucun statut « paye » | §10.2, §26.3 | Paiement |
| E2E-PAY-02 | Webhook recu deux fois : un seul effet visible (une facture, un stock, un email) | §9.5, §24.3 | Paiement |
| E2E-PAY-03 | Retour navigateur sans webhook : la commande n'est **pas** « payee », un bandeau « paiement en attente » est affiche | §26.3, §6.5 | Paiement |
| E2E-ADMIN-01 | Un role trop faible est refuse sur `/admin/*` (403/404), l'audit le trace | §16.12, §24.3 | Secu |
| E2E-SEC-01 | IDOR : remplacer l'ID d'commande dans l'URL ne donne pas acces a la commande d'un autre | §19.2, §24.3 | Secu |
| E2E-SEC-02 | Double soumission du checkout (double clic + replay) : une seule commande, un seul paiement | §9.5, §26.3 | Secu |

**Les 11 cas du parcours administration (§24.3) traduits en cas de test.**

Le spec enonce 10 scenario pour le parcours administration ; la qualite en exige **11**. Le dixieme scenario du spec est eclate en deux cas distincts (publication d'un produit d'une part, modification de MOQ et de prix d'autre part) parce que les deux exercent des RBAC differents et deux contraintes d'integrite differentes. Le onzieme cas est propre a la phase 4 : la navigation admin elle-meme est limitee au role, et une route non declaree dans la matrice est refusee au demarrage. Le tableau ci-dessous est la reference : un cas non vert bloque la livraison.

| # | Scenario CDC (§24.3) | Assertion executable | Fichier |
|---|---|---|---|
| 1 | Connexion avec MFA | `loginAsAdmin(page, 'order_manager')` rejoue le challenge TOTP ; session persistante en base ; `audit_log` porte une entree `admin.login` avec `actor_role` | `e2e/admin/dashboard.spec.ts` |
| 2 | Refus d'acces pour un role trop faible | `support_agent` sur `GET /v1/admin/refunds` -> **403** ; la reponse ne revele pas l'existence de la ressource ; entree d'audit `result = 'denied'` | `e2e/admin/dashboard.spec.ts` |
| 3 | Creation et publication d'un produit traduit en trois langues | creation en `catalog_manager`, publication exige `product:update` ; les 3 locales FR/EN/ZH-CN sont presentes ; le produit apparait dans le sitemap de chaque locale ; refus de publication si une traduction manque | `e2e/admin/dashboard.spec.ts` |
| 4 | Modification de MOQ et de prix | `price_tier` modifie -> la modification passe par **four-eyes** (second role requis) ; l'ancien palier reste dans l'historique ; une commande deja passee **relit son snapshot** et n'est pas reevaluee | `e2e/admin/dashboard.spec.ts` |
| 5 | Gestion d'un stock bas | stock sous le seuil -> alerte `stock.low` ; reapprovisionnement enregistre comme mouvement `inventory_movement` ; l'ecart d'inventaire laisse une trace d'ecart non resolue | `e2e/admin/dashboard.spec.ts` |
| 6 | Traitement d'une commande et creation d'une expedition | transition de commande autorisee pour le role ; expedition creee avec `tracking_number` ; l'evenement de tracking est append-only et resistant au desordre | `e2e/admin/dashboard.spec.ts` |
| 7 | Refus d'expedition si les donnees douanieres obligatoires manquent | expedition bloquee -> **409** `customs_data_incomplete` ; la regle de douane applicable est citee (source + date de consultation) ; aucun numero de suivi cree | `e2e/admin/dashboard.spec.ts` |
| 8 | Mise a jour d'une regle, historique et alerte de revision | creation d'une `trade_rule_version` -> la version precedente reste lisible ; une regle dont la date de fin est depassee declenche une alerte visible du compliance manager et **n'est pas appliquee silencieusement** | `e2e/admin/dashboard.spec.ts` |
| 9 | Export restreint selon le role | `export_financial` par `analyst_readonly` -> **403** ; par `finance_manager` -> demande d'approbation, export genere, **journalise** (`audit_log` action `export`), URL signee a duree courte, contenu restreint aux colonnes autorisees | `e2e/admin/dashboard.spec.ts` |
| 10 | Journal d'audit pour action financiere sensible | remboursement : demande + approbation par un second role ; auto-approbation refusee en base **et** en code ; chainage de hash verifie (`prev_hash` / `entry_hash`) ; la suite `admin_action_approval` est tracee | `e2e/admin/dashboard.spec.ts` |
| 11 | Navigation admin limitee au role, route non declaree refusee (cas propre a la phase 4) | le menu n'affiche que les entrees autorisees pour le role connecte ; une route non declaree dans la matrice est refusee au demarrage ; le refus est journalise | `e2e/admin/dashboard.spec.ts` |

> La suite de release `admin` est **un** fichier : `e2e/admin/dashboard.spec.ts`, qui porte ces 11 cas. C'est le seul fichier capture par le projet Playwright `admin` (les autres specs `e2e/admin/*` — `a11y.spec.ts` (axe-core sur les 2 dashboards), `security-regression.spec.ts` (14 actions sensibles), `exports.spec.ts` — sont des recettes fonctionnelles de phase, capturees par le projet `functional`). La Phase 4 livre les 11 cas ; la Phase 5 **reprend** le fichier pour le niveau release et rejoue la suite complete sur staging. Un cas non vert = Gate bloque.

**Les 5 suites de release et leur fichier unique (regle 4 ci-dessus).**

| Suite du spec §24.3 | Fichier unique de la suite | Livre par | Repris (Modify) en |
|---|---|---|---|
| B2C — 9 etapes | `e2e/b2c/journey.spec.ts` | Phase 2 | Phase 5 |
| B2B — 9 etapes | `e2e/b2b/journey.spec.ts` | Phase 3 | Phase 5 |
| Paiement — 8 cas | `e2e/payment/scenarios.spec.ts` | Phase 2 | Phase 5 |
| Administration — 11 cas | `e2e/admin/dashboard.spec.ts` | Phase 4 | Phase 5 |
| Securite — 7 cas | `e2e/security/security.spec.ts` | Phase 5 | — |

Les recette fonctionnelles qui ne sont pas des suites de release restent hors de ces 5 fichiers : `e2e/specs/*` (Phase 1, dont `e2e/specs/security-headers.spec.ts` qui couvre les 8 en-tetes de securite), `e2e/visual-matrix.spec.ts` (Phase 2), `e2e/b2b/admin-views.spec.ts` et `e2e/b2b/compliance.spec.ts` (Phase 3), `e2e/admin/*` hors `dashboard.spec.ts`, `e2e/account/*` et `e2e/seo/content.spec.ts` (Phase 4), `e2e/concurrency/*`, `e2e/a11y/*`, `e2e/checkout/*` et `e2e/support/i18n-parity.spec.ts` (Phase 5), `e2e/seo/*` hors `content.spec.ts`, `e2e/perf/*` et `e2e/contracts/*` (Phase 6). Toutes relèvent du projet `functional` (ou `visual` pour la matrice).

---

### 24.5 Tests de concurrence et d'idempotence

**Scenario critique — deux clients achetent la derniere unite.** C'est le test qui justifie a lui seul toute la section : le spec exige « Le stock ne passe pas sous zero dans un test concurrent concu pour le detecter » (§26.2) et « Utiliser une transaction ou un verrou adapte pour empecher les surventes concurrentes » (§7.3).

```ts
// src/modules/orders/__tests__/last-unit.integration.spec.ts
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import { app } from '@/app';
import { db } from '@/lib/db';
import { createProduct } from '@/test/factories/product.factory';

describe('concurrence — derniere unite (spec §7.3, §26.2)', () => {
  it('une seule commande aboutit, la seconde est refusee, le stock final vaut 0', async () => {
    // given
    const product = await createProduct({ moq: 1, orderIncrement: 1, stockAvailable: 1 });
    const payload = {
      items: [{ productId: product.id, quantity: 1 }],
      shippingAddress: { country: 'FR' },
    };
    const headers = { 'Idempotency-Key': (k: string) => ({ 'Idempotency-Key': k }) };

    // when — deux clients distincts, en parallele, sur deux connexions
    const [clientA, clientB] = await Promise.all([
      request(app).post('/api/orders').set(headers['Idempotency-Key']('client-a')).send(payload),
      request(app).post('/api/orders').set(headers['Idempotency-Key']('client-b')).send(payload),
    ]);

    // then — exactement un succes
    const statuses = [clientA.status, clientB.status].sort();
    expect(statuses).toEqual([201, 409]);              // 409 = conflit de stock
    expect(await db.order.count({ where: { 'items.productId': product.id } })).toBe(1);

    const item = await db.inventoryItem.findFirstOrThrow({ where: { productId: product.id } });
    expect(item.stockAvailable).toBe(0);
    expect(item.stockAvailable).toBeGreaterThanOrEqual(0);
  });
});
```

Le mecanisme correct n'est pas dans le test mais dans l'implementation : `UPDATE inventory_item SET stock_available = stock_available - $1 WHERE id = $2 AND stock_available >= $1` avec `RETURNING` — si 0 ligne retournee, conflit. Plus une contrainte `CHECK (stock_available >= 0)` en base, qui est la vraie protection. Le test **verifie le comportement**, il ne verifie pas le SQL.

**Matrice d'idempotence — 4 doubles traitements a tester systematiquement.**

| Operation | Declencheur du double | attendu | Test |
|---|---|---|---|
| Creation de commande | double clic / replay reseau / retry de job | une seule commande, meme reponse HTTP | `dedoublonne la double soumission avec la meme cle` (integration) + E2E-SEC-02 |
| Webhook paiement | provider rejoue, retry reseau, double envoi | un seul `PaymentEvent`, une facture, un email | `accepte une signature valide ... une seule fois` (integration) + E2E-PAY-02 |
| Remboursement (spec §10.4) | double clic admin, retry job, webhook provider rejoue | un seul remboursement, `refundedAmount` non ecrase, montant d origine intact | `refuse un second remboursement au-dela du montant restant` |
| Expiration de reservation | job de relance + job d'expiration | une seule liberation de stock | `est idempotent : un second passage ne libere pas deux fois` (integration) |

**Regles d'implementation testees (spec §9.5).** Cle d'idempotence unique en base sur `(scope, key)` ; le resultat memorise (code HTTP + payload) est rejoue a l'identique ; une cle reutilisee avec un payload different est refusee (409) et tracee ; le verrou/constraint unique protege la confirmation ; les consommateurs d'evenements sont idempotents (cle de deduplication sur la table de consommation).

**Test de concurrence supplementaires (a executer en CI, hors suite par defaut, tag `concurrency`).**

- 10 clients simultanes sur 5 unites -> 5 succes, 5 conflits, stock 0.
- 20 reservations simultanees sur 1 unite -> 1 active, 19 echecs.
- Double confirmation d'expedition simultanee -> une seule `Shipment`.
- Deux admins refundent le meme paiement en parallele -> un seul remboursement effectif.

---

### 24.6 Tests responsive et visuels

**Matrice de viewports (spec §22.2, §24.4).** Huit largeurs representatives : **320, 360, 390, 430, 768, 1024, 1440, 1920**. 320 est la largeur plancher du spec (§26.1) : si 320 deborde, la livraison est bloquee.

**Combinaisons = 8 viewports x 3 langues x 2 themes = 48.** Ce n'est pas 48 parcours E2E complets : c'est une **matrice de non-regression** appliquee a un nombre restreint de vues critiques. Budget : 48 combinaisons x 4 vues cles = 192 captures, executees en une job Playwright dediee (`visual` project), non bloquante au merge mais bloquante a la release (spec §26.1).

```ts
// e2e/visual/matrix.spec.ts
import { test, expect } from '@playwright/test';
import { VIEWPORTS } from '../support/viewports.js';   // liste canonique (regle 5) -- jamais re-declaree

const LOCALES = ['fr-FR', 'en', 'zh-CN'] as const;
const THEMES = ['light', 'dark'] as const;
const VIEWS = ['/fr', '/fr/products', '/fr/cart', '/fr/checkout'] as const;

for (const vp of VIEWPORTS) {
  for (const locale of LOCALES) {
    for (const theme of THEMES) {
      test(`visuel ${vp.width}px ${locale} ${theme}`, async ({ page }) => {
        await page.setViewportSize({ width: vp.width, height: vp.height });
        await page.emulateMedia({ colorScheme: theme });
        await page.goto(VIEWS[0]);
        await page.evaluate((t) => localStorage.setItem('theme', t), theme);
        await page.addInitScript((l) => localStorage.setItem('locale', l), locale);
        for (const view of VIEWS) await page.goto(view);

        // pas de debordement horizontal (spec §24.4, §22.1)
        const overflow = await page.evaluate(() =>
          document.documentElement.scrollWidth - document.documentElement.clientWidth);
        expect(overflow, `debordement horizontal de ${overflow}px sur ${vp.width}px`).toBeLessThanOrEqual(0);

        await expect(page).toHaveScreenshot(`${vp.width}-${locale}-${theme}.png`, { maxDiffPixelRatio: 0.01 });
      });
    }
  }
}
```

**Les 3 langues doivent etre reellement traduites.** Regle : le test ne cherche **jamais** un texte anglais en dur. Chaque assertion utilise soit un role + un nom issu du **dictionnaire de traduction charge dans le test**, soit un `data-testid`. Interdit : `expect(page.getByText('Add to cart')).toBeVisible()` en dehors de la locale `en`.

```ts
// e2e/support/i18n.ts — les fixtures de traduction sont la source de verite des assertions
import fr from '@/i18n/locales/fr-FR.json';
import en from '@/i18n/locales/en.json';
import zh from '@/i18n/locales/zh-CN.json';

export const dictionaries = { 'fr-FR': fr, en, 'zh-CN': zh } as const;
export type Locale = keyof typeof dictionaries;

export function t(locale: Locale, key: string): string {
  const value = key.split('.').reduce<unknown>((acc, k) => (acc as any)?.[k], dictionaries[locale]);
  if (typeof value !== 'string') throw new Error(`traduction manquante: ${locale}.${key}`);
  return value;
}
```

**Test de debordement horizontal explicite.** Au-dela du `scrollWidth`, on detecte les elements qui depassent :

```ts
const offenders = await page.evaluate(() => {
  const limit = document.documentElement.clientWidth;
  return [...document.querySelectorAll('body *')]
    .filter((el) => el.getBoundingClientRect().right > limit + 1)
    .slice(0, 10)
    .map((el) => `${el.tagName}.${(el as HTMLElement).className}`.slice(0, 120));
});
expect(offenders).toEqual([]);
```

**Tests visuels de regression.** Playwright `toHaveSnapshot` integre au depart (pas de service externe a payer). Percy/Chromatic sont une option si l'equipe veut un diffing multi-navigateurs et une revue en PR — **non retenus au lancement** : cout + dependance externe pour un gain marginal sur une suite mono-navigateur (Chrome). Si un second navigateur est necessaire plus tard, il s'ajoute comme **dimension de shard en CI** (moteur), pas comme un projet Playwright (suite) et pas comme un service.

**Vues cles soumises a snapshot** (12 references) : accueil, catalogue (grille + filtres), fiche produit (paliers), panier, checkout (etapes 1 et 7), page de connexion, page de commande confirmee, dashboard client (mobile), dashboard admin (dense), page 404, banner multi-langue, etat vide de recherche.

**Regle de revue des snapshots.** Aucun snapshot n'est approuve automatiquement. Le process : PR avec le diff d'images en piece jointe -> le reviewer regarde le diff et le commit du snapshot **dans la meme PR** -> un snapshot approuve sans diff commente est rejete. `-update-snapshots` en local puis commit du binaire ; la CI ne met **jamais** a jour de snapshot (sinon la regression devient la reference).

---

### 24.7 Tests d'accessibilite

**Objectif : WCAG 2.2 AA** sur les parcours critiques (spec §22.4).

**Outils.** `axe-core` + `jest-axe` (composants, Vitest) et `@axe-core/playwright` (E2E, sur page reelle). L'auto-scan ne prouve rien sur le clavier, le focus, les lecteurs d'ecran ni le contraste reel du design system : il complete la revue manuelle, il ne la remplace pas (§24.6).

```ts
// e2e/a11y/a11y.spec.ts — un scan par parcours critique, en plus des scenarios dedies
import AxeBuilder from '@axe-core/playwright';
import { test, expect } from '../fixtures';

const CRITICAL_PATHS = ['/fr', '/fr/products', '/fr/cart', '/fr/checkout', '/fr/login', '/fr/account/orders'];

for (const path of CRITICAL_PATHS) {
  test(`a11y ${path} — zero violation critique ou serieuse`, async ({ page }) => {
    await page.goto(path);
    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa'])
      .analyze();
    const blocking = results.violations.filter((v) =>
      v.impact === 'critical' || v.impact === 'serious');
    expect(blocking, JSON.stringify(blocking, null, 2)).toHaveLength(0);
  });
}
```

**Tolerances.**
- **Zero** violation `critical` et `serious` sur les parcours critiques (checkout, login, paiement, dashboard admin). Blocant au merge.
- Les violations `moderate` et `minor` sont listees dans un rapport non bloquant, avec une issue creee automatiquement ; plus de 30 jours = blocage.
- En composant : `expect(await axe(html)).toHaveNoViolations()` sur les composants partages (Button, Input, Modal, Table, Toast) — c'est la que se concentrent les regressions.
- Axe ne couvre pas : le focus visible, l'ordre de tabulation, les lecteurs d'ecran, le contraste des images, les cibles tactiles reelles, les animations. Ces points sont dans la revue manuelle ci-dessous.

**Les 12 verifications manuelles obligatoires** (spec §22.4 : « Tests manuels avec clavier et lecteur d'ecran sur checkout, login, dashboard et gestion des commandes »).

| # | Verification | Methode | Preuve |
|---|---|---|---|
| 1 | Navigation clavier complete du checkout (de la 1re etape a la confirmation, sans souris) | `Tab` / `Shift+Tab` depuis le haut de la page ; aucune tabulation perdue dans un overlay | Recording video + notes |
| 2 | Focus trap en modale (panneau de paiement, confirmation de suppression) | `Tab` en boucle : le focus ne sort jamais de la modale ; `Esc` ferme | Notes + capture du focus |
| 3 | Focus visible sur tous les elements interactifs, dans les **deux** themes | Navigation clavier + inspection du ring focus (contraste >= 3:1, spec §3.3) | Captures clair/sombre |
| 4 | Contraste verifie sur les deux themes avec le design system reellement rendu | Tokens de §3.3 et §3.4 appliques a des paires reel ; pas de contraste nominal | Tableau des paires (texte/fond, ratio, verdict) |
| 5 | Annonce lecteur d'ecran du recalcul de total (changement de quantite -> nouveau total) | NVDA (Windows) + VoiceOver (macOS/iOS) : le total est annonce (`aria-live="polite"`, `role="status"`) | Script des annonces annoncees |
| 6 | Navigation clavier sur la page de connexion + libelle des erreurs | NVDA : erreurs annoncees et associees au champ (`aria-describedby`) | Script |
| 7 | Formulaires : labels persistants, erreurs associees, autocompletion correcte | Inspection + NVDA | Notes |
| 8 | Tableau de commandes sur mobile : informations accessibles sans tableau trop large | VoiceOver iOS + zoom 200 % | Notes + capture |
| 9 | Cibles tactiles >= 44x44 px (spec §3.7) sur les parcours mobiles | Mesure au navigateur (bounding boxes) + test physique | Export CSV des tailles |
| 10 | `prefers-reduced-motion` respecte : aucune animation non essentielle | Emulation `reducedMotion: 'reduce'` + inspection | Notes |
| 11 | Lecteur d'ecran sur le dashboard admin (donnees denses, statuts non uniquement colores, spec §3.4) | NVDA : statuts annonces avec texte + icone, pas seulement la couleur | Script |
| 12 | Zoom texte 200 % et redimensionnement : aucune perte de contenu ni de functionalite (WCAG 1.4.4/1.4.10) | `Ctrl+ +` jusqu'a 200 % sur checkout et fiche produit | Captures |

Regle de cloture : ces 12 verifications sont rejouees a chaque release candidate et dans la Phase 5 (§27). Un echec bloque la release, pas le merge.

---

### 24.8 Tests de charge et de resilience

**Outils.** **k6** (script JS, scenarios, thresholds declaratifs, integration native avec les checks HTTP) — retenu sur Artillery (besoin d'un runner Node de plus, scripting moins expressif pour les seuils). Les tests de charge ne font **jamais** partie du merge : ils sont joues sur staging, sur un jeu de donnees iso, et publient un rapport.

**Cibles (spec §24.5)** : pages catalogue, recherche, login, panier, checkout, webhooks.

**Seuils (CI, sur staging, a ajuster apres premiere mesure — la premiere execution sert de reference).**

| Endpoint | p95 | p99 | Taux d'erreur | RPS cibles |
|---|---|---|---|---|
| `GET /fr/products` | < 600 ms | < 1200 ms | < 0,5 % | 50 |
| `GET /fr/search?q=` | < 800 ms | < 1500 ms | < 0,5 % | 30 |
| `POST /api/auth/login` | < 900 ms | < 1500 ms | < 1 % (dont 429 attendus) | 20 |
| `POST /api/cart/items` | < 400 ms | < 800 ms | < 0,5 % | 50 |
| `POST /api/checkout/quote` | < 1200 ms | < 2000 ms | < 1 % | 15 |
| `POST /api/webhooks/payment` | < 500 ms | < 1000 ms | < 0,1 % | 100 |

**Scenario k6 1 — charge nominale (montee douce, 10 min).**

```js
// perf/nominal.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const errorRate = new Rate('functional_errors');
const checkoutLatency = new Trend('checkout_latency', true);

export const options = {
  scenarios: {
    nominal: {
      executor: 'ramping-arrival-rate',
      startRate: 5, timeUnit: '1s', preAllocatedVUs: 50, maxVUs: 200,
      stages: [
        { target: 25, duration: '2m' },   // montee
        { target: 50, duration: '5m' },   // plateau
        { target: 0,  duration: '3m' },   // drainage
      ],
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.01'],
    'http_req_duration{endpoint:products}': ['p(95)<600'],
    'http_req_duration{endpoint:checkout}': ['p(95)<1200'],
    functional_errors: ['rate<0.005'],
    checks: ['rate>0.995'],
  },
};

const BASE = __ENV.BASE_URL;

export default function () {
  const catalog = http.get(`${BASE}/fr/products`, { tags: { endpoint: 'products' } });
  check(catalog, { 'catalogue 200': (r) => r.status === 200 });
  errorRate.add(catalog.status !== 200);

  const res = http.get(`${BASE}/fr/search?q=ceramic`, { tags: { endpoint: 'search' } });
  check(res, { 'recherche 200': (r) => r.status === 200 });

  sleep(Math.random() * 2);
}
```

**Scenario k6 2 — pic (achat flash, 2 min a 150 RPS, dont 60 % sur le checkout).**

```js
// perf/peak.js
export const options = {
  scenarios: {
    flash_sale: {
      executor: 'constant-arrival-rate',
      rate: 150, timeUnit: '1s', duration: '2m', preAllocatedVUs: 150, maxVUs: 400,
      exec: 'checkoutFlow', tags: { scenario: 'peak' },
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.02'],
    'http_req_duration{endpoint:checkout}': ['p(95)<2000', 'p(99)<3500'],
    functional_errors: ['rate<0.01'],
  },
};

export function checkoutFlow() {
  const login = http.post(`${BASE}/api/auth/login`, JSON.stringify({ email: `load+${__VU}@test.local`, password: 'x' }),
    { headers: { 'Content-Type': 'application/json' }, tags: { endpoint: 'login' } });
  check(login, { 'login 200': (r) => r.status === 200 });
  if (login.status !== 200) return;

  const quote = http.post(`${BASE}/api/checkout/quote`, JSON.stringify({ items: [{ sku: 'LOAD-SKU-1', quantity: 12 }] }),
    { headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${login.json('token')}` }, tags: { endpoint: 'checkout' } });
  check(quote, { 'quote 200': (r) => r.status === 200, 'quote non vide': (r) => r.body.length > 0 });
}
```

**Scenario k6 3 — tenue dans la duree (soak, 2 h a 30 RPS).** But : detecter les fuites (memoire, pool de connexions, table qui grossit, worker qui s'accumule). On mesure `http_req_duration` mais aussi la **derive** : la p95 de la derniere minute ne doit pas depasser de 25 % la p95 de la premiere minute, et le nombre de connexions DB actives doit rester stable.

```js
// perf/soak.js
export const options = {
  scenarios: {
    soak: { executor: 'constant-arrival-rate', rate: 30, timeUnit: '1s', duration: '2h', preAllocatedVUs: 60, maxVUs: 120 },
  },
  thresholds: {
    http_req_failed: ['rate<0.005'],
    'http_req_duration{endpoint:products}': ['p(95)<700'],
    dropped_iterations: ['count==0'],
  },
};
```

**Test de resilience — indisponibilite d'un prestataire.** Le spec exige « Tester les retries reseau et l'indisponibilite temporaire d'un prestataire » (§24.5) et « Tester le comportement lorsque les e-mails sont retardes ou en echec » (§24.5). On simule la panne par un **fault injection controllee** (un fake serveur local qui renvoie 503, ou un mode degrade du connecteur activable par variable d'environnement en staging uniquement).

| Prestataire tombe | Comportement attendu | Verification |
|---|---|---|
| **Paiement** | La commande reste `pending_payment`, jamais `paid` (§26.3). L'UI affiche « paiement indisponible, reessayez », le CTA reste disponible, aucun numero de carte perdu, aucun double debit. | E2E-PAY-01 + test d'integration du fallback du connecteur + alerte `IntegrationHealth` |
| **Email** | La commande est confirmee et les documents sont generes ; l'email part en file, retry avec backoff, dead-letter au-dela de N tentatives, notification dans le centre de notifications de l'utilisateur comme filet de securite. | Test d'integration du worker + verification de la DLQ |
| **Transporteur (labels, tracking)** | Le checkout fonctionne, la commande est confirmee, l'expedition passe en `customs_action_required` / action manuelle cote admin, et le client est informe du retard. | Test d'integration du connecteur + scenario d'expedition en etat degrade |
| **Base Redis / cache** | Repli en lecture seule sur PostgreSQL, degradation de la page (pas d'ecrasement), pas d'erreur 500, alerte critique. | Test de resilience (arret du service Redis en staging) |
| **Service de calcul de transport** | Le checkout montre « transport a confirmer », bloque la confirmation si le total est inconnu, ne proceed pas avec un total faux. | Test d'integration du mode degrade |
| **Service de conversion de devise** | Affichage du montant de reference avec la mention « indicatif », pas de conversion fantaisiste. | Test unitaire du fallback |

**Test de restauration de sauvegarde (spec §24.5, §19.8, §26.5).** Procedure trimestrielle, executee par l'equipe Ops, avec un compte rendu signe :

1. Identifier la sauvegarde a restaurer (PITR si disponible, sinon dump complet chiffre).
2. Restaurer dans une **base isolee** (`fanours_restore_check`), jamais en production.
3. Verifier l'integrite : `SELECT count(*)` par table critique, comparaison avec les controles de la sauvegarde.
4. Rejouer les **tests d'integration de smoke** (creation de commande, webhook, PDF) contre la base restauree.
5. Verifier le **RTO** : horodatage debut -> base prete a servir. Seuil : **RTO <= 8 h** (spec §19.8).
6. Verifier le **RPO** : perte de donnees acceptable **<= 24 h** (spec §19.8). Si l'ecart est plus grand, on le declare comme ecart au lieu de le maquiller.
7. Verifier la **coherence applicative** : la version du schema de la base restauree est compatible avec le code de production actuel ; sinon la restauration est « restauree mais non exploitable » et le test est en echec.
8. Documenter dans `docs/runbooks/restore.md` : date, duree, ecarts, actions. Un RTO/RPO non documente = restauration non prouvee.

---

### 24.9 Mutation testing et couverture

**Stryker** sur la logique critique uniquement : `src/modules/pricing/**`, `src/modules/compliance/**`, `src/modules/orders/state-machine.ts`, `src/modules/money/**`. Mutation testing repond a la seule question qui compte : « mes tests detectent-ils vraiment un changement de comportement ? ».

```json
// stryker.config.json
{
  "$schema": "./node_modules/stryker/schema.json",
  "testRunner": "vitest",
  "reporters": ["html", "clear-text", "json"],
  "mutate": [
    "src/modules/pricing/**/*.ts",
    "src/modules/compliance/**/*.ts",
    "src/modules/orders/state-machine.ts",
    "src/modules/money/**/*.ts"
  ],
  "thresholds": { "high": 90, "low": 80, "break": 80 },
  "coverageAnalysis": "perTest"
}
```

**Regle de seuil.** Mutation score **>= 95 %** sur les modules ci-dessus, execute en CI sur la branche (job dedie, non bloqueant au premier trimestre puis bloquant). Un mutant survivant sur la logique de prix ou de douane est traite comme un bug de test : soit on ecrit le test manquant, soit on ajoute `/* Stryker disable */` **avec un commentaire justifiant pourquoi le mutant est equivalent** (mutation equivalente = comportement identique, pas un mutant qu'on ne sait pas tester).

**Couverture (indicateur, pas preuve — spec §24.6).**
- Global : **80 %** des lignes / branches / fonctions / statements (V8), echec du build sous ce seuil.
- Modules critiques (`pricing`, `compliance`, `orders`, `payments`, `authz`) : **95 %**.
- Une PR qui **fait baisser** la couverture globale de plus de 0,5 point est rejetee en revue, meme si le seuil global reste passe (le seuil global masque la regression locale).
- La couverture ne dispense d'aucun des autres livrables : une ligne couverte par un test qui n'assert rien est du bruit. C'est precisement ce que la mutation testing mesure.

---

### 24.10 Lint, format, analyse statique et hooks pre-commit

**ESLint flat config** (ESLint 9, `eslint.config.mjs`), TypeScript strict, regles de type-check.

```js
// eslint.config.mjs
import tseslint from 'typescript-eslint';
import reactHooks from 'eslint-plugin-react-hooks';
import jsxA11y from 'eslint-plugin-jsx-a11y';
import importPlugin from 'eslint-plugin-import';
import { noRestrictedImports } from './eslint/no-restricted-imports.mjs';

export default tseslint.config(
  { ignores: ['.next/**', 'node_modules/**', 'test-results/**', 'coverage/**', 'playwright-report/**'] },
  ...tseslint.configs.strictTypeChecked,
  {
    files: ['**/*.{ts,tsx}'],
    languageOptions: { parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname } },
    plugins: { 'react-hooks': reactHooks, 'jsx-a11y': jsxA11y, import: importPlugin },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-hooks/exhaustive-deps': 'error',
      'jsx-a11y/alt-text': 'error',
      'jsx-a11y/anchor-is-valid': 'error',
      'jsx-a11y/label-has-associated-control': 'error',
      'jsx-a11y/no-autofocus': 'warn',
      '@typescript-eslint/no-floating-promises': 'error',
      '@typescript-eslint/no-misused-promises': 'error',
      '@typescript-eslint/require-await': 'error',
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/consistent-type-imports': ['error', { prefer: 'type-imports' }],
      'import/order': ['error', { groups: ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'], 'newlines-between': 'always' }],
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      eqeqeq: ['error', 'always', { null: 'ignore' }],
      'prefer-const': 'error',
      ...noRestrictedImports(),
    },
  },
  // la couche backend a le droit d'acceder a la base et a Redis
  { files: ['src/server/**', 'src/worker/**', 'packages/data/**', 'src/test/**'], rules: { 'no-restricted-imports': 'off' } },
);
```

```js
// eslint/no-restricted-imports.mjs
/**
 * Regle stricte : aucun acces direct a la base, a Redis ou a l'ORM hors couche backend.
 * Le frontend est un client HTTP (spec §5.4, §19.2). Cette regle est la premiere
 * barriere contre un prix, un droit ou un acces lu cote client.
 */
export function noRestrictedImports() {
  return {
    'no-restricted-imports': ['error',
      {
        paths: [
          { name: 'drizzle-orm', message: 'Acces direct a l ORM interdit hors couche backend (spec §5.4). Passer par un service metier.' },
          { name: 'pg',               message: 'Acces direct a PostgreSQL interdit hors couche backend (spec §5.4).' },
          { name: 'ioredis',          message: 'Acces direct a Redis interdit hors couche backend (spec §5.4).' },
          { name: 'redis',            message: 'Acces direct a Redis interdit hors couche backend (spec §5.4).' },
        ],
        patterns: [
          { group: ['@/server/db/**', '@/server/redis/**', '@/db/**'], message: 'Utiliser un repository expose par la couche data.' },
          { group: ['**/node_modules/drizzle-orm/**'], message: 'Acces direct a l ORM interdit hors couche backend (spec §5.4).' },
        ],
      },
    ],
  };
}
```

Le meme principe s'applique a l'interdiction de `process.env` cote client : `NEXT_PUBLIC_` est la seule variable accessible au bundle, et la liste blanche des variables publiques est explicitement listee dans `next.config.ts` (aucun secret ne peut y atterrir par accident). Regle complementaire : un test unitaire verifie que `import.meta.env` / `process.env` n'expose pas de cle contenant `SECRET`, `KEY`, `TOKEN`, `PASSWORD` dans le bundle de production.

**Autres outils.**

| Outil | Role | Commande | Blocant |
|---|---|---|---|
| **TypeScript strict** | `strict: true`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` | `tsc --noEmit` | oui |
| **Prettier** | formatage | `prettier --check .` | oui (hook) |
| **ESLint** | lint type-aware | `eslint . --max-warnings 0` | oui (hook + CI) |
| **Knip** | imports/exports/exports morts | `knip --no-progress` | oui |
| **Semgrep** | regles de securite maison (validation, SQL concat, `dangerouslySetInnerHTML`, secrets) | `semgrep --config auto` | oui sur les findings `ERROR` |
| **CodeQL / Trivy / `npm audit`** | dependances vulnérables (§19.3) | en CI, rapport dans la PR | oui au-dela de `moderate` |
| **Gitleaks** | secrets dans le depot (§19.5) | pre-commit + CI | oui |
| **tsc-strict sur les types de routes** | le contrat API est type (spec §21.1) | TypeScript + zod inferes | oui |

**Hooks pre-commit — Husky + lint-staged** (choisi plutot que Lefthook : Husky + lint-staged est le standard le plus documente, et l'equipe a deja ce vocabulaire ; Lefthook est equivalent et reste une option si la CI passe sous container).

```json
// .husky/pre-commit
npx lint-staged
```

```json
// package.json (extrait)
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix --max-warnings 0", "prettier --write"],
    "*.{json,md,css}": ["prettier --write"]
  },
  "scripts": {
    "lint": "eslint . --max-warnings 0 && prettier --check .",
    "typecheck": "tsc --noEmit",
    "test:unit": "vitest run --config vitest.config.ts",
    "test:integration": "vitest run --config vitest.integration.config.ts",
    "test:e2e": "playwright test",
    "test:e2e:functional": "playwright test --project=functional",
    "test:e2e:visual": "playwright test --project=visual",
    "test:e2e:b2c": "playwright test --project=b2c",
    "test:e2e:b2b": "playwright test --project=b2b",
    "test:e2e:payment": "playwright test --project=payment",
    "test:e2e:admin": "playwright test --project=admin",
    "test:e2e:security": "playwright test --project=security",
    "test:a11y": "playwright test e2e/a11y --project=functional",
    "mutation": "stryker run"
  }
}
```

Le pre-commit execute **uniquement** les checks sur les fichiers stages (rapides : < 10 s). Les tests complets (unitaires, integration, E2E) sont **uniquement en CI** — un pre-commit qui lance 300 tests fait contourner le hook. En cas de contournement (`--no-verify`), la CI rattrape tout.

**Commitlint** (conventional commits) : `feat|fix|docs|style|refactor|perf|test|build|ci|chore(scope): description`. Le type `test` est obligatoire pour une PR qui n'ajoute que des tests. `commitlint.config.cjs` + hook `commit-msg`. Un commit de merge squash suit la meme regle.

---

### 24.11 Definition of Done

Une tache n'est closee que si **les 25 points** sont coches. Chaque point est verifie automatiquement quand c'est possible (colonne « Verification »).

**Code**
1. La tache respecte les criteres d'acceptation de l'issue, sans ecart non documente. — *revue*
2. TypeScript strict passe (`tsc --noEmit`, zero erreur et zero `@ts-ignore` ajoute). — *automatique*
3. ESLint `--max-warnings 0` et Prettier passent. — *automatique*
4. Aucun import interdit (acces direct DB/Redis hors backend), aucun `console.log` residuel, aucun `TODO`/`FIXME` non lie a une issue. — *automatique*
5. Aucun secret, cle, token ou URL interne en dur ; les variables d'environnement sont nommees et documentees. — *automatique (Gitleaks)*

**Tests**
6. Chaque branche de la logique ajoutee est couverte par un test qui echouerait si la branche etait cassee. — *revue + mutation score*
7. Les tests unitaires passent (`vitest run`) et la couverture des modules concernes ne baisse pas. — *automatique*
8. Les tests d'integration passent contre la base de test reelle, avec isolation verifiee. — *automatique*
9. Le scenario E2E correspondant existe et passe (ou l'absence est justifiee par le niveau de la pyramide). — *automatique*
10. Le nouveau test est **deterministe** : aucun `waitForTimeout`, aucun `test.only`, aucun ordre implicite, re-executable 10 fois (`--repeat-each=10`) sans echec. — *automatique + revue*
11. Les cas limites et d'echec sont testes (etat vide, erreur reseau, permissions insuffisantes, stock insuffisant). — *revue*

**Securite**
12. Les entrees sont validees cote serveur par schema zod ; aucune requete SQL concatenee ; pas de `dangerouslySetInnerHTML` sur donnee non fiable. — *automatique (Semgrep) + revue*
13. Les autorisations sont verifiees cote serveur (RBAC + propriete de la ressource) ; le test d'IDOR de la ressource modifiee passe. — *automatique*
14. Les actions sensibles (paiement, remboursement, changement de role, export) sont journalisees dans l'audit log avec acteur, objet, resultat et correlation id. — *automatique*
15. Aucun secret ni donnee de carte dans les logs ; `npm audit` / Dependabot sans vulnerabilite critique non resolue. — *automatique*

**Performance**
16. Les listes sont paginees ou chargees de facon incremntale ; aucune requete N+1 introduite sur un parcours chaud (panier, checkout, catalogue). — *revue + `EXPLAIN` sur les 3 requetes principales*
17. Les images ont des dimensions explicites et un format adaptatif ; aucune requete bloquante tierce introduite. — *revue + Lighthouse CI*
18. Le parcours affecte reste dans les budgets CWV (LCP <= 2,5 s, INP <= 200 ms, CLS <= 0,1 au p75, spec §23.1) — mesurage avant/apres sur staging. — *automatique (Lighthouse CI)*

**Accessibilite**
19. Axe ne signale aucune violation critique ou serieuse sur les composants et pages modifies. — *automatique*
20. Le focus, l'ordre de tabulation, le focus trap et les annonces lecteur d'ecran ont ete verifies sur le parcours affecte. — *revue manuelle*
21. Le contraste des nouvelles paires couleur/fond est verifie dans les deux themes (spec §3.3, §3.4). — *revue*

**i18n**
22. Aucune chaine utilisateur en dur : tout provient des dictionnaires FR/EN/ZH-CN ; la cle existe dans les 3 locales. — *automatique (test de parite des cles de traduction)*
23. Les montants, dates et nombres sont formates par locale ; les chaines chinoises ont ete relues par une personne competente (spec §4.2) pour tout contenu juridique ou contractuel. — *automatique (formats) + revue humaine*

**Docs et git**
24. La documentation impactee est mise a jour (guide operateur, runbook, reference API/OpenAPI, changelog si visible client), et le commit suit la convention conventional commits. — *revue + automatique (commitlint)*
25. La PR est complete : titre, contexte, changements, **preuves** (sorties de tests, captures avant/apres), limites connues, lien vers l'issue ; **moins de 800 lignes de diff** (au-dela : scinder). — *revue*

---

### 24.12 Process de revue

**Ce que le reviewer verifie, dans cet ordre.**

1. **Bug** : le changement fait-il ce qu'il dit ? Cherche le cas limite non couvert : quantite 0, quantite enorme, devise etrangere, produit sans stock, pays non livre, webhook duplique, requete concurrente. Le reviewer reproduit mentalement un scenario qui casse.
2. **Regression** : le diff touche-t-il un comportement existant ? Y a-t-il un test qui verrouille l'ancien comportement ? Les seuils, la pagination, les valeurs par defaut ont-ils change ?
3. **Ecart vs criteres d'acceptation** : chaque point des §26 applicable a la modification est-il satisfait ? Les "criteres d'acceptation" de l'issue sont-ils tous couverts par un test ?
4. **Securite** : validation des entrees, autorisations, IDOR, XSS, CSRF, secrets, exposition de donnees, rate limiting, audit log, idempotence des operations sensibles. Le reviewer security-auditor passe en priority sur les chemins finances/admin/conformite.
5. **Performance** : N+1, requetes non indexees, requete par element de liste, rendu cote client de donnees volumineuses, images non optimisees, bundle qui grossit.
6. **Qualite du test** : le test teste-t-il le comportement ou l'implementation ? Echouerait-il si la logique etait cassee ? (verifier par mutation mental ou en local). Un test qui passe toujours ne prouve rien.
7. **Perimetre** : le diff fait-il plus que l'issue ? Y a-t-il du code mort, du refactor non annonce, du TODO ?

**Regle d'approbation.**

| Criticite du chemin modifie | Approbations requises |
|---|---|
| Faible (front, contenu, style, docs) | **1 reviewer** |
| Moyenne (metier standard : panier, catalogue, dashboard client) | **1 reviewer metier** |
| Critique (paiement, remboursement, **tarification**, **douane/TVA**, **authentification/2FA**, **RBAC**, **migrations**, **donnees personnelles**, **exports**) | **1 reviewer metier + 1 revue securite** (security-auditor), les deux obligatoires avant merge |

Aucun auto-approbation. Le merge n'est jamais fait par l'auteur de la PR. Un reviewer qui constate un bloqueur **n'approuve pas et le dit explicitement** ; l'absence de commentaire n'est pas une approbation.

**Modele de PR.**

```markdown
## Titre
feat(checkout): recalcul des frais de douane sur changement d'adresse (#412)

## Contexte
Probleme : le recapitulatif de checkout conservait l'estimation de la commande
precedente apres un changement de pays de destination (spec §24.2, §9.2).
Repro : checkout FR -> US, le total conserve le droit FR.

## Changements
- separation du calcul de quote (ephemere) du snapshot de commande (persiste)
- ajout de TradeCompliance.recalculateForAddress()
- regle de test: le snapshot de commande n'est jamais reecrit

## Preuves
- tests: `npm run test:integration -- checkout` (14 passed), `npm run test:unit -- tax` (63 passed)
- concurrency: `npm run test:integration -- concurrency` (stock final = 0, 1 succes / 1 conflit)
- mutation: stryker pricing+compliance -> 96.2% (seuil 95)
- a11y: axe sur /fr/checkout -> 0 violation critique/serieuse
- avant/apres: captures dans le ticket #412
- perf: p95 checkout 840 ms -> 910 ms (budget 1200 ms)

## Limites connues
- les droits US relies sur une regle a expiration 2026-12-31, a revoir avant cette date
- pas de test E2E pour le changement d'adresse en ZH-CN (ajoute en Phase 5)

## Checklist
- [x] Definition of Done (25/25)
- [x] Revue securite requise (chemin critique) — @security-auditor
- [x] Mutation score >= 95% sur pricing/compliance
- [ ] Revue metier — @commerce-lead

## Lie
- Issue #412
- Spec §9.2, §11.3, §24.2
```

Regles de forme : **titre au format conventional commits**, description en 4 blocs (contexte / changements / preuves / limites), **preuves obligatoires** (sorties de tests, pas « les tests passent »), captures avant/apres pour tout changement visuel, limites connues explicites (un PR sans limite connue et sans interaction graphique est suspect), lien vers l'issue.

**Regle des 800 lignes.** Une PR au-dela de **800 lignes de diff** (hors generated : lockfiles, snapshots) est scindee en PRs empilees sequentielles, chacune verifiable et mergable. Justification : une revue de 2000 lignes n'est pas une revue. Exception documentee : un changement purement mecanique (migration generatee, mise a jour massive de tokens de design) peut depasser, a condition d'etre valide par un outillage dedie (migration rollback-teste, diff de tokens revu ligne par ligne).

**Delais de revue.** Premiere reponse sous 24 h ouvrees. Une PR sans reponse au bout de 48 h est relancee par le lead. Une PR en attente de revue **bloque le merge** mais pas les autres PR : on ne bloque pas l'equipe sur une PR dormante.

---

### 24.13 Metriques de qualite

Suivies mensuellement, revues en rituel qualite (mensuel) et affichees dans le dashboard technique (§23.2).

| Metrique | Definition | Seuil | Alerte |
|---|---|---|---|
| **Escaped defects** | Defauts decouverts en production (ou par un client) traces a une PR | **<= 2 / sprint** ; **0** sur les parcours paiement, douane, auth | au-dela : revue de la phase concernee |
| **Change failure rate** | Part des deploiements necessitant un rollback ou un hotfix | **<= 10 %** | au-dela : pause sur les deploiements, analyse |
| **Lead time for change** | Delai entre premiere commit de la PR et mise en production | **<= 3 jours ouvres** (median) | suivi de tendance, pas de seuil dur |
| **Flaky test rate** | Part des executions qui reussissent au 2e essai (pass-on-retry) | **< 1 %** ; **0 %** sur les 12 scenarios E2E prioritaires | au-dela de 1 % : le test est desactive et ticket de cause racine cree sous 24 h |
| **Couverture** | Lignes / branches, globale et par module critique | globale **>= 80 %**, critique **>= 95 %** (mutation score) | build rouge |
| **Temps de suite** | Duree totale de la CI (unitaire + integration + E2E) et duree par shard | **unitaire + integration <= 6 min**, **E2E <= 12 min** par shard, **total <= 18 min** | au-dela : sharding a revoir, tests lents a diagnostiquer |
| **Taux de rollback** | Rollbacks / deploiements, avec cause | **<= 5 %**, tend vers 0 | revue d'incident |
| **Incidents** | Incidents de production, par severite | **0 sev-1**, **<= 1 sev-2 / mois** | postmortem obligatoire |
| **Ecart spec** | Points du §26 non satisfaits ou acceptes comme limitation | **0 non documente** | bloquant la release |
| **Temps de correction** | Delai entre detection d'un bug et merge du correctif | **sev-1 : < 4 h**, **sev-2 : < 2 jours** | suivi |
| **Couverture des 3 langues** | % des parcours E2E executes en FR/EN/ZH-CN | **100 %** des 20 scenarios prioritaires en 3 langues sur la phase 5 | non atteint = non conforme au §22.4/§26.1 |

**Regles de gouvernance associees.**

- **Chaque metrique a une definition ecrite** (pas de calcul flou) et un proprietaire, comme l'exige le spec pour les KPI metier (§16.2).
- **Un test flaky ne vit pas plus de 24 h dans la suite bloquante.** Il est soit corrige, soit desactive et place dans une file de triage avec un ticket de cause racine. Supprimer un test sans diagnostic = supprimer un rapport de bug.
- **Les retries ne sont pas un traitement.** `retries: 2` en CI sert a **mesurer** la flake (pass-on-retry = signal). La cible est `retries: 0` sur la suite merge-bloquante.
- **Le taux de rollback et le CFR sont les deux metriques les plus correllees a la qualite du process.** Une hausse de l'un declenche une revue retrospective du cycle, pas un reproche individuel.
- **Une regression visual, responsive ou a11y est un bloqueur de release** au meme titre qu'un echec de test fonctionnel (spec §26.1, §26.6).
- **Revue mensuelle qualite (30 min)** : 4 metriques cles (CFR, escaped defects, flaky rate, temps de suite), 3 top causes racines, 3 actions avec owner et date. La revue produit un compte rendu dans le dashboard et alimente la roadmap.
