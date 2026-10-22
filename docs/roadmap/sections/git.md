## Workflow Git et GitHub

Périmètre : branches, commits, hooks, pull requests, revue, protection de `main`, CI, supply chain, versions, ADR, règles d'historique. Cette section applique les exigences §24.6 (qualité, sécurité, automatisation CI), §25.2 (pipeline de livraison), §25.3 (migrations et retour arrière), §26.6 (qualité technique) et §19.5 (secrets) du cahier des charges (CDC), et les décisions figées dans `architecture.md` (monorepo pnpm + Turbo, conteneur multi-services, Drizzle, canary par digest).

---

### 1. Stratégie de branches

**Modèle retenu : trunk-based development.** `main` est la branche de production, toujours déployable, jamais porteuse de travail en cours. Chaque changement passe par une branche courte, une pull request, une revue, un squash-merge. Il n'existe pas de `develop` ni de branches de release longues : le versionnement est porté par les tags (section 10).

```
main      ──●──────●──────●──────●──────●───►  (toujours vert, toujours déployable)
           ╲      ╱ ╲    ╱       ╲    ╱
            ●────●   ●──●         ●──●        feat/*, fix/*, refactor/*, test/*
```

**Nommage exact.** Regex appliquée par le hook local `git-guard` et vérifiée par le workflow `pr-size.yml` :

```
^(feat|fix|refactor|test)/[a-z0-9]+(?:-[a-z0-9]+)*$
```

Préfixes : `feat` (nouvelle fonctionnalité), `fix` (correction de bug), `refactor` (changement interne sans changement de contrat), `test` (tests uniquement). Le slug est en minuscules, chiffres et tirets. `docs/` et `chore/` sont acceptés par le hook (CLAUDE.md projet §0.1) mais ne sont pas porteurs de changement fonctionnel : ils n'ouvrent pas de cycle de release. Valides : `feat/checkout-address-recalc`, `fix/webhook-idempotency-guard`. Invalides : `feat/Checkout_Address`, `feature/checkout`, `checkout-recalc`.

| Règle de cycle de vie | Valeur | Raison |
|---|---|---|
| Durée de vie maximale | 3 jours ouvrés | au-delà, la branche diverge de `main` et la revue devient un contrôle de forme |
| Taille d'une branche | ≤ 800 lignes de diff (hors généré) | une revue de 2 000 lignes n'est pas une revue (section 5) |
| Branches ouvertes par auteur | 2 | réduit le travail parallèle non relu |
| Branches obsolètes | supprimées à la fusion | `delete_branch_on_merge` du dépôt (section 2) |

Une branche qui dépasse 3 jours est découpée : les commits déjà relus passent dans des PRs empilées séquentielles, le reste repart d'un `main` à jour. On ne réécrit jamais l'historique d'une branche partagée.

```bash
git fetch origin --prune
git switch main && git pull --ff-only origin main
git switch --no-track -c feat/checkout-address-recalc origin/main
git push -u origin feat/checkout-address-recalc
```

`--no-track` empêche la nouvelle branche d'hériter de `origin/main` comme upstream : après le premier push, l'upstream est `origin/feat/checkout-address-recalc`. C'est la protection mécanique contre un `git push` sans argument qui pousserait sur `main`.

**Pourquoi `main` est protégée.** `main` contient l'état déployé en production ; une régression y est visible immédiatement par des clients réels : un prix recalculé côté client (interdit par le CDC §5.4), un checkout cassé, une règle douanière expirée appliquée en silence (§26.4). Trois couches indépendantes : (1) locale — hook `~/.claude/hooks/git-guard.js` bloquant `commit`, `push`, `merge` et `reset` sur `main` ; (2) serveur — ruleset GitHub refusant tout push direct, tout force-push et toute suppression de `main` (section 2) ; (3) processus — squash-merge obligatoire, jamais de commit de merge sur `main` (CLAUDE.md projet §9).

**Configuration Git locale.**

```bash
git config --local init.defaultBranch main && git config --local push.default current
git config --local pull.rebase true && git config --local fetch.prune true
git config --local merge.ff only && git config --local core.autocrlf false
git config --local commit.gpgsign true && git config --local tag.gpgsign true
```

Lecture : branche par défaut `main` · `push.default current` évite le push sans argument vers `main` · `pull.rebase` évite un commit de merge créé par un pull · `fetch.prune` supprime les branches distantes disparues · `merge.ff only` interdit le commit de merge accidentel · `core.autocrlf false` préserve les fins de ligne LF · commits et tags signés.

---

### 2. Ruleset de protection de `main`

Le ruleset est appliqué via l'API GitHub (format `repository rulesets`), versionné dans `infra/github/main-ruleset.json` et appliqué par un script idempotent (`infra/github/apply-ruleset.sh`). Un ruleset est préféré à la branch protection classique parce qu'il porte le bypass traçable et la règle de résolution des conversations, absents de l'API classique.

```json
{
  "name": "main-protection", "target": "branch", "enforcement": "active",
  "bypass_actors": [{ "actor_id": 5, "actor_type": "RepositoryRole", "bypass_mode": "always" }],
  "conditions": { "ref_name": { "include": ["refs/heads/main"], "exclude": [] } },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "pull_request", "parameters": {
        "required_approving_review_count": 2, "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true, "require_last_push_approval": true,
        "required_review_thread_resolution": true,
        "allowed_merge_methods": ["squash", "rebase"] } },
    { "type": "required_status_checks", "parameters": {
        "strict_required_status_checks": true,
        "required_status_checks": [
          { "context": "pr-size" }, { "context": "lint" }, { "context": "format" }, { "context": "typecheck" },
          { "context": "unit-tests" }, { "context": "integration-tests" }, { "context": "secret-scan" },
          { "context": "dependency-scan" }, { "context": "codeql" }, { "context": "e2e-critical" }, { "context": "build" }
        ] } }
  ]
}
```

Lecture du ruleset :

- `deletion` : `main` ne peut pas être supprimée. `non_fast_forward` : force-push interdit, y compris par un administrateur.
- `required_approving_review_count: 2` : deux approbations par défaut. Une PR à risque faible (`risk/low` validé par le lead dans la PR) n'obtient qu'une approbation, **tracée** dans l'issue de la PR : le ruleset reste à 2, l'exception est documentée, jamais un contournement silencieux.
- `dismiss_stale_reviews_on_push` : tout nouveau push invalide les approbations déjà données. `require_code_owner_review` : un `CODEOWNERS` doit approuver — `CODEOWNERS` est la source de vérité de la matrice de rôles (section 6). `require_last_push_approval` : la dernière approbation doit venir de quelqu'un qui n'a pas poussé depuis.
- `required_review_thread_resolution` : toutes les conversations de revue doivent être résolues avant merge.
- `allowed_merge_methods: ["squash", "rebase"]` : **historique linéaire obligatoire**. Le commit de merge est interdit sur `main` ; chaque commit de `main` est un commit conventionnel autonome, ce qui rend `git bisect` et `git log --oneline` utilisables en production.
- `strict_required_status_checks` : la branche doit être à jour avec `main` avant merge. `required_status_checks` : liste identique à la section 7 ; toute modification se fait dans les deux endroits en même temps.

Le push direct sur `main` est refusé : la règle `pull_request` est obligatoire, donc un push est rejeté même avec les permissions d'écriture. La suppression automatique de la branche après fusion est un paramètre de dépôt, appliqué par le même script.

```bash
./infra/github/apply-ruleset.sh        # applique main-ruleset.json (idempotent) + delete_branch_on_merge
gh api "repos/${GH_REPO}/rulesets" --jq '.[] | {name, enforcement, rules: [.rules[].type]}'
gh api "repos/${GH_REPO}/branches/main/protection" --jq '{checks: .required_status_checks.contexts, force_push: .allow_force_pushes.enabled}'
```

---

### 3. Conventional Commits appliqué

Format : `<type>(<scope>): <sujet>` — sujet en minuscules, sans point final, ≤ 72 caractères. Corps indenté, lignes ≤ 72 caractères. Pied de page pour `BREAKING CHANGE:` et `Co-Authored-By:`.

Types autorisés : `feat` (nouvelle fonctionnalité ou contrat d'API), `fix` (correction de bug, comportement observable), `docs` (documentation seule), `style` (formatage, sans changement de comportement), `refactor` (restructuration interne, contrat public inchangé), `perf` (gain de performance mesuré), `test` (tests uniquement), `build` (toolchain, dépendances, Dockerfiles, lockfile), `ci` (pipelines GitHub Actions), `chore` (maintenance sans impact fonctionnel), `revert` (annulation d'un commit, corps = `This reverts commit <sha>.`).

**Scopes du monorepo** — alignés sur l'arborescence de `architecture.md` : `web` (`apps/web`), `api` (`apps/api`), `worker` (`apps/worker`), `db` (`packages/db`), `domain` (`packages/domain`), `contracts` (`packages/contracts`), `compliance` (`packages/compliance`), `pricing` (`packages/pricing`), `i18n` (`packages/i18n`), `ui` (`packages/ui`), `observability` (`packages/observability`), `config` (`packages/config`), `e2e` (`e2e/`), `infra` (`infra/`), `ops` (`ops/`), `docs` (`docs/`), `ci` (`.github/`), `deps` (mise à jour de dépendances). Un commit qui touche plusieurs packages porte le scope du **composant métier principal** ; le détail des autres va dans le corps. Exemples : `feat(web): add sticky cart bar on mobile`, `fix(api): recalculate duties when destination country changes`, `feat(db): add exclusion constraint on overlapping price tiers`.

**Commits réels de ce projet** (périmètre §7.2, §10.3, §3.3, §25.3 du CDC) :

```text
feat(db): add exclusion constraint on overlapping price tiers

A plain UNIQUE on (sku, min_qty) allowed the ranges [1,50] and
[40,100] to coexist, so the resolved price depended on row order.
Adds an EXCLUDE constraint on numrange(min_qty, max_qty) with
btree_gist. The migration is marked expand.

Refs: #412
```

```text
fix(api): make the payment webhook idempotent on event id

The PSP delivers payment.succeeded twice in the retry window: the
handler wrote the order status twice and emitted two outbox events,
producing two confirmation emails. The handler now inserts the
provider event id into payment_event under a unique constraint and
returns 200 on conflict: the second delivery is a no-op.

Refs: #418
```

Deux autres commits du même périmètre : `style(ui): adjust the surface-2 token in the dark theme` (le token est le seul changement, aucun composant touché, contraste WCAG 2.2 AA inchangé, Refs: #401) et `feat(db): add order_snapshot table to persist the checkout snapshot` (adresse, prix unitaires, règles de taxe et conditions de vente figées dans une colonne JSONB écrite dans la même transaction que les lignes de commande, colonne ajoutée nullable puis backfillée par un job worker, Refs: #433).

**`commitlint.config.ts` réel.**

```ts
import type { UserConfig } from 'commitlint';

const types = ['feat', 'fix', 'docs', 'style', 'refactor', 'perf',
  'test', 'build', 'ci', 'chore', 'revert'] as const;
const scopes = ['web', 'api', 'worker', 'db', 'domain', 'contracts', 'compliance',
  'pricing', 'i18n', 'ui', 'observability', 'config', 'e2e', 'infra',
  'ops', 'docs', 'ci', 'deps'] as const;

export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [...types]], 'scope-enum': [2, 'always', [...scopes]],
    'scope-case': [2, 'always', 'kebab-case'], 'subject-case': [2, 'always', 'lower-case'],
    'subject-empty': [2, 'never'], 'subject-full-stop': [2, 'never', '.'],
    'type-empty': [2, 'never'], 'scope-empty': [2, 'never'],
    'header-max-length': [2, 'always', 72], 'body-max-line-length': [2, 'always', 72],
    'footer-max-line-length': [2, 'always', 72],
    'body-leading-blank': [2, 'always'], 'footer-leading-blank': [2, 'always'],
  },
} satisfies UserConfig;
```

Le fichier est `commitlint.config.ts` (TypeScript) et non `.cjs` : commitlint v19 charge la configuration TypeScript nativement, sans étape de build. Script associé : `"commitlint": "commitlint --edit"`, exécuté par le hook `commit-msg` (section 4).

---

### 4. Husky et lint-staged

Husky v9 est retenu parce que son format de hook est un script shell simple, lisible et débogable, et parce que `lint-staged` n'exécute que sur les fichiers indexés. Un pre-commit qui lance 300 tests fait contourner le hook ; la CI est le lieu des tests complets.

**`.husky/pre-commit`**

```sh
#!/usr/bin/env sh
set -euo pipefail
if [ "${HUSKY:-0}" = "0" ]; then echo "git-guard: hooks désactivés (HUSKY=0) — à déclarer dans la PR"; exit 0; fi
pnpm exec lint-staged                                               # format + lint + tests unitaires ciblés
pnpm exec gitleaks git --staged --redact --no-banner --exit-code 1 # scan de secrets sur le diff indexé
pnpm exec node scripts/git-guard.mjs pre-commit                     # refuse commit/push/merge sur main
```

**`.husky/commit-msg`**

```sh
#!/usr/bin/env sh
set -euo pipefail
if [ "${HUSKY:-0}" = "0" ]; then echo "git-guard: hooks désactivés (HUSKY=0) — à déclarer dans la PR"; exit 0; fi
pnpm exec commitlint --edit "$1"
```

**Configuration `lint-staged`** (dans le `package.json` racine) :

```json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix --max-warnings 0", "prettier --write", "vitest related --run --passWithNoTests"],
    "*.{json,md,css,yml,yaml}": ["prettier --write"],
    "*.sql": ["node scripts/check-migration-safety.mjs"],
    ".env*": ["node scripts/check-env-file.mjs"],
    "*.{png,jpg,jpeg,gif,webp,avif,zip,pdf,mp4,woff,woff2}": [
      "bash -c 'echo \"Fichier binaire $1 : il ne doit pas être versionné (section 12).\" >&2; exit 1' --"
    ]
  }
}
```

Chaque étape est bloquante : `prettier --write` met le format au standard du dépôt ; `eslint --fix --max-warnings 0` remonte les erreurs de type, les règles a11y et les imports interdits (accès direct base/Redis hors backend) ; `vitest related --run` rejoue les tests correspondant aux fichiers indexés ; `gitleaks git --staged` bloque un secret ou un PAN introduit dans le commit ; `check-migration-safety.mjs` refuse `DROP TABLE`, `DROP COLUMN`, `TRUNCATE`, `RENAME`, `ALTER COLUMN ... TYPE` et `DELETE FROM` sans `WHERE`, sauf marqueur `-- @allow-destructive` + `-- @adr: ADR-xxxx` ; `check-env-file.mjs` refuse toute valeur dans `.env*` et n'autorise que `.env.example` en mode « noms de variables uniquement » ; `commitlint --edit` contrôle type, scope, longueur et casse.

Les tests d'intégration, E2E, la charge (k6) et le mutation testing (Stryker) ne sont **pas** dans le pre-commit : ils s'exécutent en CI. Le mutation score est un gate de CI sur `packages/pricing` et `packages/compliance` (seuil 95), pas un hook local.

```bash
pnpm add -D husky lint-staged @commitlint/cli @commitlint/config-conventional
pnpm exec husky init
pnpm exec husky add .husky/pre-commit .husky/commit-msg
git config --local core.hooksPath .husky
```

**Politique de contournement.** `HUSKY=0` désactive les hooks pour une seule commande : `HUSKY=0 git commit -m "feat(api): hotfix payment webhook timeout"`. Autorisé **uniquement** en cas d'urgence documentée (incident de production en cours, bogue bloquant le pipeline, dépendance réseau indisponible). Jamais pour éviter un lint ou un scan de secrets. Chaque contournement doit être **déclaré dans le corps de la pull request** :

```markdown
## Contournement de hooks

- Hooks contournés : `HUSKY=0` sur le commit `abc1234`
- Motif : le scan gitleaks bloquait sur une fixture de test contenant une clé Stripe de test factice (`sk_test_...`) — le fichier a été déplacé dans `e2e/fixtures/` et le motif est allowlisté dans `.gitleaks.toml`.
- Contrôle équivalent exécuté manuellement : `pnpm exec gitleaks git --staged` et `pnpm exec eslint . --max-warnings 0` — sorties jointes.
```

Un contournement non déclaré dans la PR fait échouer la revue. La CI rattrape de toute façon les contrôles contournés localement : le hook local est un confort, la CI est la garantie.

---

### 5. Structure d'une pull request

**Modèle réel `.github/pull_request_template.md` :**

```markdown
## Contexte
Problème : · Reproduction : · Impact métier (qui est affecté, quelle perte, quel risque) :

## Ce qui change
- Fichiers structurants touchés : `apps/api`, `packages/db`

## Spec de référence
- CDC §  ·  ADR : ADR-  ·  Issue : #

## Tests et preuves
- Tests unitaires / intégration / E2E : `commande` → N passés, N échoués
- Mutation (si `packages/pricing`, `packages/compliance`) : score =
- Sécurité : `commande` → résultat  ·  Accessibilité : `commande` → résultat
- Performance : mesure avant / après, seuil
- Sorties de commande collées dans la PR (pas « les tests passent »)

## Risques et rollback
- Risque introduit :   ·   Plan de rollback (image précédente par digest, feature flag, ou `git revert`) :
- Ce qui déclenche un rollback (seuil observable) :

## Captures
- Avant : <lien image>   ·   Après : <lien image>
- (obligatoire si changement visuel ; sinon « sans changement visuel »)

## Checklist
- [ ] Définition of Done (qualite.md §24.11) cochée
- [ ] Aucun secret, aucun fichier binaire, aucun `.env` commité
- [ ] Les 3 langues à jour si nouvelle chaîne ; les 2 thèmes si nouveau composant
- [ ] États loading / vide / erreur implémentés   ·   Revue(s) obtenues (section 6)
- [ ] Diff ≤ 800 lignes hors généré

## Limites connues
-
```

**Règles de forme.** Le titre est un message de commit conventionnel : `feat(api): add order snapshot to the checkout flow (#412)`. Les quatre blocs obligatoires sont **Contexte, Ce qui change, Tests et preuves, Risques et rollback** : une PR sans « Preuves » (sorties de commande, pas affirmation) est rejetée à la revue. Les captures sont obligatoires pour tout changement visuel, en avant/après ; un screenshot seul ne prouve pas un comportement. « Limites connues » ne peut pas rester vide sans justification écrite (`qualite.md` §24.12). Le corps de la PR est rédigé en français ; le code, les identifiants et les messages de commit en anglais.

**Règle de taille.** Au-delà de **800 lignes de diff** (hors fichiers générés : lockfile, snapshots, migrations générées, SBOM), la PR est scindée en PRs empilées séquentielles, chacune relue et mergable séparément. Le workflow `pr-size.yml` compte les lignes ajoutées et supprimées et échoue au-delà du seuil, sauf si la PR porte le label `size/exempt` et une justification dans le corps (cas légitimes : mise à jour mécanique de tokens de design, migration générée, mise à jour massive de dépendances). La liste des chemins exclus est versionnée dans `infra/github/pr-size-exclude.txt`.

**Sections obligatoires conditionnelles.** Le contrôle est automatique : `pr-size.yml` compare les chemins modifiés aux zones sensibles et vérifie que la section correspondante est présente et non vide. Une section manquante fait échouer le check.

| Zone touchée | Bloc obligatoire | Contenu exigé |
|---|---|---|
| `packages/db/migrations/**` | « Si la PR touche des migrations » | fichier(s) de migration, phase expand/migrate/contract, plan de rollback, preuve de restauration (snapshot PITR + test exécuté le), marqueur `-- @allow-destructive` et ADR référencé |
| `packages/compliance/**`, `apps/api/src/modules/trade-compliance/**` | « Si la PR touche le moteur de conformité » | version de règle, source officielle (URL), date de consultation de la source, niveau de certitude, propriétaire de la validation, date de prochaine revue |
| `apps/api/src/modules/auth/**`, gardes RBAC, exports | « Si la PR touche l'authentification ou les permissions » | delta de matrice d'autorisation (rôle / action / ressource / avant / après), vérification de propriété anti-IDOR, rotation d'identifiant de session concernée ou non, MFA concerné ou non |
| `apps/web/**`, `packages/ui/**` | « Captures » | avant / après, sinon « sans changement visuel » |
| `packages/i18n/**`, toute nouvelle chaîne | preuve de parité des clés | parité FR / EN / ZH-CN vérifiée, textes chinois relus |

Motifs : une migration destructive sans plan de rollback ni preuve de restauration est interdite (CDC §25.3) ; une règle sans source officielle, ni date de consultation, ni propriétaire n'est pas publiable (CDC §26.4) ; une modification d'autorisation sans delta explicite est une régression silencieuse (CDC §19.2).

---

### 6. Process de revue

**Nombre d'approbations selon la zone touchée.** Le ruleset impose 2 approbations par défaut (section 2) ; la matrice ci-dessous est le contrat métier : 1 approbation sur les zones à risque faible, 1 + 1 revue spécialisée sur les zones sensibles.

| Zone touchée | Approbations | Reviewers obligatoires |
|---|---|---|
| `apps/web/**`, `packages/ui/**`, `docs/**`, contenus statiques | **1** | `reviewer` (Code Reviewer) |
| `apps/api/**` hors chemins sensibles, `packages/contracts/**`, `packages/i18n/**`, `e2e/**` | **1** | `reviewer` métier |
| `packages/pricing/**`, `packages/domain/**` (tarification, machine à états) | **1** | `reviewer` métier + **1** revue sécurité |
| `apps/api/src/modules/payments/**`, `apps/api/src/modules/refunds/**` | **1** | `reviewer` métier + **1** revue sécurité (obligatoires) |
| `apps/api/src/modules/trade-compliance/**`, `packages/compliance/**` | **1** | `reviewer` métier + **1** revue conformité (Tax Strategist) |
| `apps/api/src/modules/auth/**`, gardes RBAC, export de données | **1** | `reviewer` métier + **1** revue sécurité (obligatoires) |
| `packages/db/migrations/**` | **1** | `reviewer` + **1** second mainteneur (les deux requis si destructive) |
| Pages publiques indexables, `sitemap`, `robots.txt`, JSON-LD | **1** | `reviewer` + `seo-auditor` si la page est indexable |
| Parcours E2E, seuils de performance, configuration CI | **1** | `reviewer` + `qa-e2e` si le parcours est critique |

Aucune auto-approbation. Le merge n'est jamais effectué par l'auteur de la PR. Un reviewer qui identifie un bloqueur n'approuve pas et le dit explicitement : l'absence de commentaire n'est pas une approbation.

**Matrice des rôles de revue** (source : `.github/CODEOWNERS`) : `apps/api/**` → `@fanours/backend-reviewers` ; `apps/web/**`, `packages/ui/**` → `@fanours/frontend-reviewers` ; `packages/pricing/**`, `packages/domain/**` → `@fanours/commerce-reviewers` ; `packages/compliance/**` et `apps/api/src/modules/trade-compliance/**` → `@fanours/compliance-reviewers` ; `apps/api/src/modules/auth/**`, `**/guards/**` → `@fanours/security-reviewers` ; `packages/db/migrations/**` → `@fanours/database-reviewers` ; `e2e/**`, `**/*.spec.ts` → `@fanours/qa-reviewers` ; `apps/web/src/app/**/page.tsx`, `sitemap.ts`, `robots.ts` → `@fanours/seo-reviewers` ; `infra/**`, `.github/**`, `ops/**` → `@fanours/platform-reviewers` ; `docs/adr/**` → `@fanours/architecture-reviewers`.

**Checklist de revue** (ordre obligatoire, identique à `qualite.md` §24.12).

1. **Bug** — le changement fait-il ce qu'il annonce ? Cas limite non couvert : quantité 0, quantité maximale, devise étrangère, produit sans stock, pays non livré, webhook dupliqué, requêtes concurrentes.
2. **Régression** — le diff touche-t-il un comportement existant ? Un test verrouille-t-il l'ancien comportement ? Seuils, pagination, valeurs par défaut ont-ils changé ?
3. **Écart vs critères d'acceptation** — chaque point du CDC §26 applicable est-il satisfait ? Les critères de l'issue sont-ils tous couverts par un test ?
4. **Sécurité** — validation des entrées, autorisations serveur, IDOR/BOLA, XSS, CSRF, secrets, exposition de données, rate limiting, audit log, idempotence des opérations sensibles.
5. **Performance** — N+1, requêtes non indexées, rendu client de données volumineuses, images non optimisées, taille du bundle.
6. **Qualité du test** — le test vérifie-t-il le comportement observable ? Échouerait-il si la logique était cassée ?
7. **Périmètre** — le diff fait-il plus que l'issue ? Y a-t-il du code mort, un refactor non annoncé, un marqueur de travail oublié ?
8. **i18n et a11y** — les 3 locales sont-elles à jour ? Focus, contraste, annonces lecteur d'écran, `prefers-reduced-motion` respectés ?

**Définition de « approuvé »** — une approbation est une revue GitHub soumise avec l'état `Approved`, où : le reviewer a parcouru la checklist ci-dessus ; aucune conversation de revue n'est ouverte sur le diff ; les questions posées ont reçu une réponse écrite dans la conversation (une réponse non lue n'est pas une réponse) ; le reviewer a lu les preuves (sorties de tests), pas seulement le résumé. Un commentaire sans objection (comment, suggestion, `nit`) **n'est pas** une approbation. Un `Approve` laissé sur une PR dont les conversations ne sont pas résolues ne compte pas : le ruleset `required_review_thread_resolution` bloque le merge.

**Comportement face à un commentaire bloquant.** Correctif simple : l'auteur corrige et pousse ; `dismiss_stale_reviews_on_push` invalide l'approbation, la revue repart de zéro sur le nouveau diff. Désaccord d'architecture : discussion dans la PR ; si un ADR est nécessaire, il est créé dans `docs/adr/` et référencé ; le désaccord non résolu bloque la fusion. Bloquant non résolu à la date limite : l'auteur **ferme la PR** et consigne la justification écrite dans l'issue d'origine ; une PR close n'est jamais mergée par contournement. Sur un chemin non sensible : la discussion continue dans la PR, qui reste ouverte ; la fusion reste bloquée tant que la conversation est ouverte.

**Délais de revue.** Première réponse d'un reviewer (commentaire, question ou approbation) : **24 h ouvrées** ; relance par le lead à 48 h : **48 h ouvrées** ; revue de sécurité sur chemin critique : **48 h ouvrées** ; PR bloquant une phase : revue de phase convoquée sous 24 h ouvrées. Une PR en attente de revue **bloque son propre merge** mais pas les autres PR : on ne bloque pas l'équipe sur une PR dormante. Une PR sans activité depuis 10 jours ouvrés est fermée automatiquement, sauf commentaire du lead la maintenant ouverte.

---

### 7. Protection de `main` et de la production

**Angle déploiement.** Le ruleset de la section 2 protège le **code**. La protection de la **production** est une seconde barrière, dans l'environnement GitHub et dans le pipeline.

**Checks requis = jobs CI de la section 8.**

| Check requis | Job | Commande sous-jacente |
|---|---|---|
| `pr-size` | `pr-size.yml` | comptage des lignes de diff, sections obligatoires, nom de branche |
| `lint` · `format` · `typecheck` | `ci.yml` (matrice `static`) | `pnpm exec eslint . --max-warnings 0` · `pnpm run format:check` · `pnpm run typecheck` |
| `unit-tests` | `ci.yml` | `pnpm vitest run --coverage` |
| `integration-tests` | `ci.yml` | `pnpm vitest run --project integration` |
| `secret-scan` | `ci.yml` | `gitleaks git` + détection de PAN |
| `dependency-scan` | `ci.yml` | `pnpm audit` + `osv-scanner` + `dependency-review` |
| `codeql` | `codeql.yml` | analyse statique Semgrep/CodeQL |
| `e2e-critical` | `e2e.yml` | parcours critiques Playwright |
| `build` | `ci.yml` | build reproductible + SBOM + digest |

**Historique linéaire.** Décision : **historique linéaire obligatoire sur `main`**, réalisé par `allowed_merge_methods: ["squash", "rebase"]`. Le commit de merge est interdit. Chaque commit de `main` est un commit conventionnel autonome, ce qui rend `git bisect` et `git log --oneline` utilisables sur la production. Le squash-merge est le mode par défaut ; le rebase n'est accepté que pour les PRs à un seul commit.

**Qui peut contourner le ruleset, et traçabilité des contournements.**

| Acteur | Mode de bypass | Conditions |
|---|---|---|
| `admin` (rôle dépôt, `actor_id: 5`) | `always` | uniquement pour une correction de sécurité critique ou une restauration de service ; chaque bypass est journalisé par GitHub dans l'audit log du dépôt |
| `maintainer` du ruleset | `exempt` (pré-approbation) | uniquement pour ajouter un check d'urgence documenté |

Le bypass **n'est jamais** utilisé pour contourner un check CI en échec sur une PR ordinaire. Un bypass implique l'ouverture d'une issue d'incident (`incident/<date>-<sujet>`) décrivant le check contourné, la raison, la durée, l'action corrective ; l'issue est rattachée à la PR concernée et clôturée après vérification que la CI est verte sur le nouveau diff. La liste des bypasss du mois est revue dans le rituel de phase (section 13).

**Déploiement en production.**

1. Seul `main`, après squash-merge, déclenche un déploiement. Aucun tag, aucune branche, aucun push direct ne produit un déploiement production.
2. Le pipeline construit **une seule fois** l'image et la publie dans le registre par digest (`sha256:...`). Le digest est l'artefact.
3. Le même digest est promu de `staging` à `production` : la production ne rebuild jamais. Un digest différent entre les deux environnements est un défaut de pipeline, détecté par `post-deploy` qui compare le digest déployé au digest du run de build.
4. Le déploiement production passe par l'environnement GitHub `production`, qui exige une approbation manuelle (section 8).
5. Le rollout est un **canary** 5 % → 25 % → 100 % piloté par le reverse proxy, avec rollback automatique sur seuil de garde dépassé (seuils dans `docs/ROADMAP_INFRA.md` §5.3).
6. Les migrations sont appliquées **avant** le code, en mode expand/contract, et la version précédente du code reste fonctionnelle pendant toute la fenêtre.
7. L'image de la version précédente est conservée **30 jours** : le rollback est un changement de pool, pas un rebuild.
8. Après déploiement : smoke tests, lecture des métriques de garde, et `post-deploy` qui vérifie que le digest en production est celui attendu.

```bash
# Promotion par digest — la seule forme autorisée
./ops/deploy/deploy.sh production sha256:<digest>
# Rollback vers le pool précédent
./ops/deploy/deploy.sh rollback
```

---

### 8. GitHub Actions

**Fichiers sous `.github/workflows/` :**

| Fichier | Déclencheur | Rôle |
|---|---|---|
| `ci.yml` | `pull_request`, `push` sur `main`, `workflow_dispatch` | lint, format, typecheck, tests, scans, build de l'image |
| `e2e.yml` | `pull_request`, `push` sur `main`, `schedule` (nuit) | Playwright sharded, 5 parcours obligatoires |
| `security.yml` | `pull_request`, `schedule` (quotidien) | Semgrep, config scan, scan de secrets complet |
| `codeql.yml` | `pull_request`, `push` sur `main`, `schedule` | analyse statique de sécurité (TypeScript, JavaScript) |
| `release.yml` | `push` sur tag `v*`, `workflow_dispatch` | changelog, build reproductible, SBOM, signature, digest |
| `deploy-staging.yml` | `push` sur `main` | promotion du digest en staging, migrations, canary staging |
| `deploy-prod.yml` | `workflow_dispatch` (approbation manuelle `production`) | promotion du digest en production, canary 5/25/100, smoke tests |
| `dependabot.yml` | dépôt → PR automatique | mises à jour npm, GitHub Actions, Docker |
| `pr-size.yml` | `pull_request` | contrôle de taille de diff, sections obligatoires, nommage de branche |

**Secrets GitHub nécessaires — noms uniquement, jamais de valeurs.**

GitHub Secrets est un **mécanisme de livraison**, pas le coffre applicatif. Les secrets applicatifs de production vivent dans Vault/KMS et sont injectés au runtime par la plateforme d'hébergement ; ils ne transitent jamais par GitHub Secrets. GitHub Secrets ne contient que : (a) le bootstrap de livraison (accès registre, clé SSH, confiance OIDC), (b) des jetons de courte durée, (c) les secrets propres à la CI (bases de test, clés sandbox de prestataires).

| Fichier | Secrets / variables GitHub | Portée |
|---|---|---|
| `ci.yml` | `REGISTRY_USERNAME`, `REGISTRY_TOKEN`, `TEST_DATABASE_URL`, `TEST_REDIS_URL`, `SLACK_WEBHOOK_URL` | dépôt ; registre par OIDC (`id-token: write`), base éphémère recréée à chaque run |
| `e2e.yml` | `E2E_BASE_URL`, `E2E_TEST_USER_EMAIL`, `E2E_TEST_USER_PASSWORD`, `E2E_ADMIN_EMAIL`, `E2E_ADMIN_PASSWORD`, `E2E_TOTP_SECRET`, `STAGING_DATABASE_URL`, `STAGING_REDIS_URL`, `STAGING_S3_BUCKET` | environnement `ci` pour les comptes de test, `staging` pour les bases ; valeurs sandbox |
| `security.yml` | `SEMGREP_APP_TOKEN`, `SNYK_TOKEN`, `GITLEAKS_LICENSE_KEY` | dépôt ; hors de ces clés, l'analyse tourne en local sans envoi de code |
| `codeql.yml` | aucun secret requis | le token GitHub natif suffit (`security-events: write`) |
| `release.yml` | `GPG_PRIVATE_KEY`, `GPG_PASSPHRASE`, `COSIGN_PRIVATE_KEY`, `COSIGN_PASSWORD`, `NPM_TOKEN` | environnement `release` ; signature des tags, des images et publication des paquets internes |
| `deploy-staging.yml` | `STAGING_DATABASE_URL`, `STAGING_REDIS_URL`, `STAGING_S3_BUCKET`, `STAGING_S3_ACCESS_KEY_ID`, `STAGING_S3_SECRET_ACCESS_KEY`, `STAGING_EMAIL_API_KEY`, `STAGING_PAYMENT_SECRET_KEY`, `STAGING_PAYMENT_WEBHOOK_SECRET`, `SSH_DEPLOY_KEY`, `SSH_HOST_KEY` | environnement `staging` ; clés sandbox |
| `deploy-prod.yml` | `PROD_DATABASE_URL`, `PROD_REDIS_URL`, `PROD_S3_BUCKET`, `PROD_S3_ACCESS_KEY_ID`, `PROD_S3_SECRET_ACCESS_KEY`, `PROD_EMAIL_API_KEY`, `PROD_PAYMENT_SECRET_KEY`, `PROD_PAYMENT_WEBHOOK_SECRET`, `SSH_DEPLOY_KEY`, `SSH_HOST_KEY`, `SLACK_WEBHOOK_URL` | environnement `production` ; **noms de variables injectés au runtime, valeurs dans Vault/KMS** |
| `dependabot.yml` | `NPM_TOKEN` (registre privé éventuel) | dépôt |

**Règles transverses à tous les workflows** : `permissions: contents: read` par défaut ; `id-token: write` uniquement dans le job de build/push (OIDC vers le registre) ; `security-events: write` uniquement dans le job CodeQL ; `timeout-minutes` sur **chaque** job ; `defaults.run.shell: bash` ; `concurrency` avec `cancel-in-progress: true` sur les branches ; installation en `pnpm install --frozen-lockfile` ; aucun secret imprimé (`::add-mask::` sur les valeurs injectées). Actions pinnées par tag (`actions/checkout@v4`, `actions/setup-node@v4`, `docker/build-push-action@v6`, `actions/upload-artifact@v4`), les SHA étant figés par Dependabot dans une PR dédiée.

**`ci.yml`** — le build de l'image est factorisé dans le workflow réutilisable `_build-image.yml`, appelé aussi par `release.yml` pour garantir qu'une version et `main` partagent la même chaîne de build.

```yaml
name: CI
on: { pull_request: {}, push: { branches: [main] }, workflow_dispatch: {} }
permissions: { contents: read }
concurrency: { group: "ci-${{ github.ref }}", cancel-in-progress: true }
env: { NODE_VERSION: "22.x", PNPM_VERSION: "9.15.0" }
defaults: { run: { shell: bash } }
jobs:
  static: { name: "${{ matrix.check }}", runs-on: ubuntu-latest, timeout-minutes: 10,
    strategy: { fail-fast: false, matrix: { check: [lint, format, typecheck] } }, steps: [
      { uses: "actions/checkout@v4" }, { uses: "./.github/actions/setup" },
      { run: "pnpm run check:${{ matrix.check }}" } ] }
  unit-tests: { name: unit-tests, runs-on: ubuntu-latest, timeout-minutes: 15,
    strategy: { fail-fast: false, matrix: { shard: [1, 2] } }, steps: [
      { uses: "actions/checkout@v4" }, { uses: "./.github/actions/setup" },
      { run: "pnpm vitest run --coverage --shard=${{ matrix.shard }}/2" },
      { if: "always()", uses: "actions/upload-artifact@v4", with: { name: "coverage-${{ matrix.shard }}", path: "coverage/", retention-days: 14 } } ] }
  integration-tests: { name: integration-tests, runs-on: ubuntu-latest, timeout-minutes: 20,
    services: {
      postgres: { image: "postgres:16-alpine", env: { POSTGRES_USER: postgres, POSTGRES_PASSWORD: postgres, POSTGRES_DB: fanours_test },
        ports: ["5432:5432"], options: "--health-cmd 'pg_isready -U postgres' --health-interval 10s --health-retries 10" },
      redis: { image: "redis:7-alpine", ports: ["6379:6379"], options: "--health-cmd 'redis-cli ping' --health-interval 10s --health-retries 10" } },
    steps: [
      { uses: "actions/checkout@v4" }, { uses: "./.github/actions/setup" },
      { run: "pnpm db:migrate:deploy && pnpm db:seed:synthetic" },
      { run: "pnpm vitest run --project integration --coverage=false" } ] }
  secret-scan: { name: secret-scan, runs-on: ubuntu-latest, timeout-minutes: 5, steps: [
    { uses: "actions/checkout@v4", with: { fetch-depth: 0 } },
    { uses: "gitleaks/gitleaks-action@v2", env: { GITLEAKS_ENABLE_UPLOAD_ARTIFACT: "true" } },
    { name: "Detect card numbers in fixtures", run: "! rg -n --pcre2 '\\b(?:4\\d{3}|5[1-5]\\d{2}|3[47]\\d{2}|6011)[ -]?\\d{4}[ -]?\\d{4}[ -]?\\d{4}\\b' --glob '!pnpm-lock.yaml' ." } ] }
  dependency-scan: { name: dependency-scan, runs-on: ubuntu-latest, timeout-minutes: 10,
    permissions: { contents: read, pull-requests: read }, steps: [
      { uses: "actions/checkout@v4" }, { uses: "./.github/actions/setup" },
      { run: "pnpm audit --audit-level high" },
      { uses: "google/osv-scanner-action/osv-scanner-action@v2", with: { scan-args: "--lockfile=pnpm-lock.yaml" } },
      { uses: "actions/dependency-review-action@v4", with: { fail-on-severity: high } } ] }
  build: { name: build, needs: [static, unit-tests, integration-tests, secret-scan, dependency-scan],
    uses: "./.github/workflows/_build-image.yml", permissions: { contents: read, id-token: write } }
```

**`_build-image.yml`** (workflow réutilisable, appelé par `ci.yml` et `release.yml` — c'est lui qui produit le digest promu en staging puis en production) :

```yaml
name: Build image
on: { workflow_call: { outputs: { digest: { value: "${{ jobs.build.outputs.digest }}" } } } }
permissions: { contents: read, id-token: write }
jobs:
  build: { name: build, runs-on: ubuntu-latest, timeout-minutes: 20,
    outputs: { digest: "${{ steps.push.outputs.digest }}" }, steps: [
      { uses: "actions/checkout@v4" }, { uses: "./.github/actions/setup" },
      { uses: "docker/setup-buildx-action@v3" },
      { uses: "docker/build-push-action@v6", with: { context: ".", push: true, provenance: { mode: max }, sbom: true,
        outputs: "type=image,name=registry.fanours.com/fanours,push-by-digest=true,name-canonical=true" } },
      { id: push, run: "digest=$(docker buildx imagetools inspect registry.fanours.com/fanours:latest --format '{{json .Manifest.Digest}}'); echo \"digest=$digest\" >> \"$GITHUB_OUTPUT\"" },
      { uses: "actions/upload-artifact@v4", with: { name: sbom, path: "**/*.sbom.json", retention-days: 90 } } ] }
```

**`e2e.yml`.**

```yaml
name: E2E
on: { pull_request: {}, push: { branches: [main] }, schedule: [{ cron: "17 3 * * *" }] }
permissions: { contents: read }
concurrency: { group: "e2e-${{ github.ref }}", cancel-in-progress: true }
jobs:
  e2e-shards: { name: e2e-shards, runs-on: ubuntu-latest, timeout-minutes: 12,
    strategy: { fail-fast: false, matrix: { shard: [1, 2, 3, 4, 5] } },
    steps: [
      { uses: "actions/checkout@v4" }, { uses: "./.github/actions/setup" },
      { uses: "actions/cache@v4", with: { path: "~/.cache/ms-playwright", key: "playwright-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}" } },
      { run: "pnpm exec playwright install --with-deps chromium" },
      { run: "pnpm db:migrate:deploy && pnpm db:seed:synthetic" },
      { run: "pnpm exec playwright test --shard=${{ matrix.shard }}/5" },
      { if: "failure()", uses: "actions/upload-artifact@v4", with: { name: "playwright-shard-${{ matrix.shard }}", path: "playwright-report/", retention-days: 14 } } ] }
  e2e-critical: { name: e2e-critical, needs: [e2e-shards], runs-on: ubuntu-latest, timeout-minutes: 20,
    steps: [
      { uses: "actions/checkout@v4" }, { uses: "./.github/actions/setup" },
      { run: "pnpm exec playwright test --project=b2c --project=b2b --project=payment --project=admin --project=security --reporter=line" },
      { name: "Fail on unexpected console errors", run: "pnpm exec playwright test --project=functional --reporter=line" } ] }
```

Les 5 parcours obligatoires du CDC §24.3 (B2C, B2B, paiement, administration, sécurité) sont répartis sur **5 shards** : chaque shard exécute `1/5` de l'intégralité du `testDir` (`./e2e`), donc **tous** les projets du modèle E2E canonique — `b2c`, `b2b`, `payment`, `admin`, `security`, `visual` et `functional` (§24.4) — sont couverts une fois, sans trou. Le **navigateur est une dimension de CI**, jamais un projet Playwright : la Phase 5 est mono-navigateur (Chromium) ; la Phase 6 ajoute la dimension `browser` à la matrice de shard. Le plafond de 12 min par shard est un **budget de job CI** (`timeout-minutes`), pas un timeout Playwright : le timeout par test reste de 45 s. Le run nocturne exécute la suite complète avec `retries: 2` pour détecter le flakiness.

**`security.yml` et `codeql.yml`.**

```yaml
name: Security
on: { pull_request: {}, schedule: [{ cron: "23 4 * * *" }] }
permissions: { contents: read, security-events: write }
jobs:
  semgrep: { name: semgrep, runs-on: ubuntu-latest, timeout-minutes: 20, steps: [
    { uses: "actions/checkout@v4" }, { uses: "./.github/actions/setup" },
    { uses: "returntocorp/semgrep-action@v1", with: { config: ".semgrep/fanours.yml" } } ] }
  config-scan: { name: config-scan, runs-on: ubuntu-latest, timeout-minutes: 15, steps: [
    { uses: "actions/checkout@v4" }, { uses: "aquasecurity/trivy-action@master", with: { scan-type: config, scan-ref: ".", severity: "HIGH,CRITICAL" } },
    { uses: "docker://rhysd/actionlint:latest" } ] }
---
name: CodeQL
on: { pull_request: {}, push: { branches: [main] }, schedule: [{ cron: "31 2 * * 1" }] }
permissions: { contents: read, security-events: write }
jobs:
  analyze: { name: codeql, runs-on: ubuntu-latest, timeout-minutes: 20, steps: [
    { uses: "actions/checkout@v4" }, { uses: "github/codeql-action/init@v3", with: { languages: "typescript,javascript", queries: "security-extended" } },
    { uses: "github/codeql-action/autobuild@v3" }, { uses: "github/codeql-action/analyze@v3" } ] }
```

**`release.yml`.**

```yaml
name: Release
on:
  push: { tags: ["v*"] }
  workflow_dispatch:
    inputs:
      version: { description: "Version to release, without the v prefix (e.g. 1.4.0)", required: true, type: string }
      dry_run: { description: "Build, sign and publish the digest without publishing packages or tagging", required: false, default: true, type: boolean }
      channel: { description: "Release channel", required: false, default: stable, type: choice, options: [stable, patch] }
permissions: { contents: read, id-token: write }
jobs:
  changelog: { name: changelog, runs-on: ubuntu-latest, timeout-minutes: 5, steps: [
    { uses: "actions/checkout@v4", with: { fetch-depth: 0 } }, { uses: "./.github/actions/setup" },
    { run: "git-cliff --config cliff.toml --output CHANGELOG.md --tag ${{ inputs.version || github.ref_name }}" } ] }
  release: { name: release, needs: [changelog], environment: release,
    if: "github.event_name == 'push' || inputs.dry_run == false", uses: "./.github/workflows/_build-image.yml" }
  publish: { name: publish, needs: [release], runs-on: ubuntu-latest, timeout-minutes: 10,
    if: "github.event_name == 'push' || inputs.dry_run == false", steps: [
      { uses: "actions/checkout@v4", with: { fetch-depth: 0 } },
      { uses: "actions/attest-build-provenance@v2", with: { subject-name: "registry.fanours.com/fanours", subject-digest: "${{ needs.release.outputs.digest }}", push-to-registry: true } },
      { uses: "softprops/action-gh-release@v2", with: { tag_name: "v${{ inputs.version || github.ref_name }}", body_path: "CHANGELOG.md" } } ] }
```

**`deploy-staging.yml`, `deploy-prod.yml`, `pr-size.yml`.** Même structure pour les deux déploiements ; deux différences : l'environnement GitHub et l'approbation. `deploy-staging` consomme le digest construit par `_build-image.yml` sur `main` ; `deploy-prod` consomme **le même digest**, saisi en entrée et vérifié en staging avant promotion.

```yaml
name: Deploy staging
on: { push: { branches: [main] }, workflow_dispatch: {} }
permissions: { contents: read, id-token: write }
jobs:
  build: { uses: "./.github/workflows/_build-image.yml" }
  deploy: { name: deploy, needs: [build], environment: staging, runs-on: ubuntu-latest, timeout-minutes: 20, steps: [
    { uses: "actions/checkout@v4" },
    { run: "./ops/deploy/deploy.sh staging '${{ needs.build.outputs.digest }}'" } ] }
---
name: Deploy production
on: { workflow_dispatch: { inputs: { digest: { description: "Digest of the image promoted from staging (sha256:...)", required: true, type: string } } } }
permissions: { contents: read, id-token: write }
jobs:
  deploy: { name: deploy, environment: { name: production, url: "https://fanours.com" },
    runs-on: ubuntu-latest, timeout-minutes: 30, steps: [
      { uses: "actions/checkout@v4" },
      { run: "node scripts/verify-digest-on-staging.mjs '${{ inputs.digest }}'" },
      { run: "./ops/deploy/deploy.sh production '${{ inputs.digest }}'" },
      { run: "./ops/deploy/smoke-production.sh" } ] }
---
name: PR size
on: { pull_request: { types: [opened, synchronize, reopened, edited] } }
permissions: { contents: read, pull-requests: read }
jobs:
  pr-size: { name: pr-size, runs-on: ubuntu-latest, timeout-minutes: 5, steps: [
    { uses: "actions/checkout@v4", with: { fetch-depth: 0 } },
    { name: "Check branch name, diff size and mandatory sections", uses: "actions/github-script@v7", with: { script: |
        const { default: check } = await import('./scripts/pr-size-check.mjs');
        const { readFileSync } = await import('fs');
        const excludeList = readFileSync('infra/github/pr-size-exclude.txt', 'utf8').split('\n').filter(Boolean);
        const result = await check({ context, excludeList });
        if (!result.ok) { core.setFailed(result.errors.join('\n')); } else { core.info(result.summary); } } } ] }
```

Le déploiement production **exige une approbation manuelle** : c'est la protection d'environnement GitHub, pas une étape du script. `scripts/pr-size-check.mjs` vérifie le nom de la branche par rapport à la regex, le nombre de lignes ajoutées + supprimées hors fichiers exclus, la présence des sections conditionnelles obligatoires quand les chemins concernés sont modifiés, et la présence non vide de `## Tests et preuves` et `## Risques et rollback`.

**Protection des environnements GitHub.**

| Environnement | Reviewers requis | Attente | Règles de branche de déploiement | Secrets autorisés | Commentaire obligatoire |
|---|---|---|---|---|---|
| `ci` | 0 | 0 | toutes les branches | secrets de CI uniquement | non |
| `staging` | 0 | 0 | `main` | secrets `STAGING_*` | non |
| `release` | 1 (mainteneur) | 5 min | `main` et tags `v*` | `GPG_*`, `COSIGN_*`, `NPM_TOKEN` | oui |
| `production` | 2 (dont un lead) | 15 min | `main` uniquement | secrets `PROD_*` | oui, avec le digest promoté |

```bash
gh api --method PUT "repos/${GH_REPO}/environments/production" --input infra/github/environment-production.json
gh api --method PUT "repos/${GH_REPO}/environments/staging" --input infra/github/environment-staging.json
```

Le fichier `infra/github/environment-production.json` décrit l'environnement `production` : `deployment_branch_policy` en `custom_branch_policies` (branche `main` uniquement), `reviewers` = 1 `User` (lead) + 1 `Team` (`@fanours/release-managers`), `wait_timer: 15`, `prevent_self_review: true`, `deployment_protection_rule` de type `WaitTimer`. Les identifiants de reviewers et d'équipe sont résolus par le script qui applique le ruleset de la section 2 ; un environnement dont les ids ne sont pas résolus est refusé par l'API, ce qui empêche un déploiement sur un environnement mal configuré.

---

### 9. Supply chain et dependabot

**`.github/dependabot.yml` réel.**

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: '/'
    schedule: { interval: weekly, day: monday, time: '06:00', timezone: Europe/Paris }
    open-pull-requests-limit: 10
    groups: {
      typescript: { patterns: ['typescript', '@types/*'] }, react: { patterns: ['react', 'react-dom', '@types/react', '@types/react-dom'] },
      nestjs: { patterns: ['@nestjs/*'] }, drizzle: { patterns: ['drizzle-orm', 'drizzle-kit'] }, testing: { patterns: ['vitest', '@vitest/*', 'playwright', '@playwright/*', '@testing-library/*'] },
      eslint: { patterns: ['eslint', 'eslint-*', '@eslint/*', 'typescript-eslint'] }, build: { patterns: ['turbo', 'prettier', 'husky', 'lint-staged'] },
      security: { patterns: ['zod', 'argon2', 'jose'] }, prod-dependencies: { dependency-type: production, patterns: ['*'] } }
  - package-ecosystem: github-actions
    directory: '/'
    schedule: { interval: weekly, day: monday, time: '06:30', timezone: Europe/Paris }
    open-pull-requests-limit: 5
    groups:
      actions: { patterns: ['*'] }
  - package-ecosystem: docker
    directory: '/infra/docker'
    schedule: { interval: weekly, day: monday, time: '07:00', timezone: Europe/Paris }
    open-pull-requests-limit: 5
    groups:
      base-images: { patterns: ['node', 'postgres', 'redis'] }
```

**Politique de version majeure (SemVer).**

| Type de mise à jour | Traitement | Détail |
|---|---|---|
| `patch` (`1.4.3` → `1.4.4`) | PR Dependabot automatique, merge après CI verte | aucune action manuelle si le diff est limité au lockfile |
| `minor` (`1.4.3` → `1.5.0`) | PR Dependabot, revue standard (1 approbation) | merge après CI verte et revue du changelog de la dépendance |
| `major` (`1.4.3` → `2.0.0`) | **Jamais auto-mergé.** PR dédiée, suite complète, **soak de deux semaines** | la PR reste ouverte 14 jours ; le soak démarre au merge en `staging` |

Le soak de deux semaines d'une mise à jour majeure signifie : la dépendance est déployée en `staging` dès son merge, elle y reste **14 jours minimum** sans incident, et elle n'est pas promue en production qu'après ce délai et revue explicite. Ce n'est pas une attente passive : pendant ces 14 jours, les métriques de garde (5xx, p95, taux d'échec paiement, profondeur de file) sont comparées à la baseline des 24 h précédentes. Une régression arrête la promotion. La procédure est outillée par `infra/github/dependabot-major-policy.md` (PR dédiée → CI complète → merge en staging → soak 14 jours → promotion) et la PR porte le label `semver-major`.

**Politique de lockfile.** `pnpm-lock.yaml` est **commité** à la racine : c'est la spécification reproductible de l'arbre de dépendances. L'installation en CI se fait **exclusivement** en `pnpm install --frozen-lockfile` — si le lockfile n'est pas cohérent avec les manifestes, l'installation échoue au lieu de résoudre silencieusement de nouvelles versions. Le job `dependency-scan` compare le diff du lockfile et signale les changements non attendus. Le lockfile n'est jamais réécrit en masse (`pnpm update` global) : une mise à jour massive est une PR dédiée, hors Dependabot, revue ligne par ligne sur le diff des versions résolues. Les versions résolues sont épinglées (`pnpm.overrides` dans le `package.json` racine) quand une version transitive crée une vulnérabilité ou un conflit ; l'override est justifié par un commentaire.

---

### 10. Gestion des versions et des releases

**SemVer appliqué à une plateforme de commerce interne.**

| Composant | Cadence | Règle |
|---|---|---|
| `packages/*` et `apps/*` (monorepo) | version unique pour le dépôt | le monorepo est versionné globalement : le tag vaut pour l'ensemble de l'image déployée |
| Version de l'image Docker | même version que le dépôt | l'image est identifiée par son digest, le tag est une commodité de lisibilité |
| Version d'une migration SQL | **jamais** versionnée sémantiquement | numérotation `NNNN_nom.sql`, appliquée via le registre `schema_migrations` avec checksum |
| Version d'un contrat d'API | versionnée dans le chemin (`/v1/...`) | un changement cassant ouvre `/v2/` ; `/v1/` reste maintenu jusqu'à la fin de vie documentée |
| Version d'une règle douanière | dates de début / fin dans `trade_rule_version` | pas de SemVer : la règle est datée, pas versionnée |

**Ce qui compte comme changement cassant** dans ce dépôt : un changement du schéma d'un contrat Zod partagé (`packages/contracts`) qui supprime ou renomme un champ, ou resserre une validation ; la suppression ou le déplacement d'une route API, ou un changement de code HTTP pour un cas documenté ; la modification d'une colonne existante qui change la sémantique d'une valeur (pas l'ajout d'une colonne nullable) ; la modification d'une machine à états d'ordre (retrait d'une transition) ; tout changement du calcul d'un prix, d'une taxe ou d'un frais d'import ; un changement de format de montant ou de devise ; un changement de la stratégie de résolution d'une règle douanière. Ne sont **pas** cassants : l'ajout d'un champ optionnel, l'ajout d'une route, l'ajout d'un palier de prix, un changement de libellé, l'ajout d'une locale à un dictionnaire non mis en service.

**Format de tag.** `vX.Y.Z`, tag signé (annotated, `git tag -s`) : `v1.0.0`, `v1.0.1`, `v1.1.0`, `v2.0.0`. Aucun préfixe de type (`release-`, `prod-`), pas de date dans le tag. Le tag pointe sur le commit de `main` qui a été mergé, jamais sur un commit de branche.

**Génération du changelog.** `git-cliff` produit `CHANGELOG.md` à partir des messages conventionnels, configuration versionnée dans `cliff.toml` :

```toml
[changelog]
header = "# Changelog\n\n"
body = "## {{version}} — {{date}}\n\n{{description}}\n\n{{grouped_commits}}\n"
trim = true
footer = ""
[git]
conventional_commits = true
filter_unconventional = true
split_commits = false
```

Le changelog est généré au moment du tag (`release.yml`), publié dans la release GitHub, et `CHANGELOG.md` est mis à jour sur `main` par une PR automatique de synchronisation. Il est rédigé en français, avec les types regroupés par domaine fonctionnel (Fonctionnalités, Corrections, Performance, Sécurité, Interne).

**Chemin de promotion.**

```
main ──merge──► build (une seule fois) ──► digest sha256:abc…
                                                  │
                    ┌─────────────────────────────┴─────────────────────────────┐
                    ▼                                                           ▼
          staging (canary 5/25/100)                                  tag vX.Y.Z (release)
                    │                                                           │
                    └──────────────────► même digest ◄──────────────────────┘
                                                  │
                                                  ▼
                                    production (canary 5/25/100)
```

Le build est fait **une seule fois**, dans `ci.yml` (ou `release.yml` pour un tag), et l'image est publiée **par digest**, jamais par tag mutable. `staging` reçoit ce digest ; `production` reçoit **le même digest**, promu par `deploy-prod.yml` avec l'input `digest`. Un rebuild entre staging et production est un défaut de pipeline : `scripts/verify-digest-on-staging.mjs` compare le digest promu au digest effectivement déployé en staging et échoue si différent. L'image précédente est conservée **30 jours** : le rollback est un changement de pool. Le tag `vX.Y.Z` est posé sur le commit de `main` correspondant, **après** le déploiement staging réussi.

**Entrées de `release.yml`.**

| Entrée | Type | Défaut | Rôle |
|---|---|---|---|
| `version` | `string` | requis en dispatch, lu depuis le tag en push | version `X.Y.Z` sans le `v` ; détermine le nom du tag et de la release |
| `dry_run` | `boolean` | `true` | construit, signe et publie le digest et le SBOM, mais ne publie ni paquet npm, ni tag, ni release |
| `channel` | `choice: stable \| patch` | `stable` | `stable` suit `main` ; `patch` ne reçoit que des backports de sécurité et suit une branche de maintenance dédiée |

---

### 11. Architecture Decision Records

**Emplacement et nommage.** `docs/adr/NNNN-slug.md`, numéro à 4 chiffres, incrémental, jamais réutilisé. Le slug est en minuscules, tirets, décrivant la décision et non le problème : `0001-drizzle-over-prisma.md`.

**Statuts.** `Proposed` : rédigée, en attente de décision, le code ne doit pas encore s'y conformer. `Accepted` : décidée et appliquée, contraignant. `Superseded by ADR-XXXX` : remplacée par une ADR ultérieure, conservée dans l'historique. `Deprecated` : plus appliquée mais toujours valable pour le code existant. `Rejected` : décidée négativement, conservée pour expliquer pourquoi l'alternative a été écartée.

Passer une ADR de `Proposed` à `Accepted` se fait par merge sur `main` après approbation d'un membre de `@fanours/architecture-reviewers`. Une ADR `Accepted` ne se supprime pas : elle passe à `Superseded`.

**Modèle `docs/adr/0000-model.md`.**

```markdown
# ADR-0000 — Modèle d'ADR

- Statut : Accepted · Date : 2026-10-21 · Décideurs : @fanours/architecture-reviewers
- Remplace : — · Remplacée par : —

## Contexte
Quel est le problème, quelles contraintes s'appliquent, quelles options ont été examinées.

## Décision
Ce qui est décidé, en une phrase affirmative et sans ambiguïté.

## Alternatives écartées
| Option | Pourquoi écartée |
|---|---|

## Conséquences
**Positives** : - · **Négatives** : - · **Suivi** : ce qui doit être surveillé, revu ou re-décidé, et à quelle échéance.
```

**ADR à écrire pour ce projet.**

| ADR | Décision | Raison |
|---|---|---|
| ADR-0001 | Drizzle plutôt que Prisma | le CDC exige `CHECK`, `EXCLUDE`, `numrange` et des migrations expand/contract : ces objets sont du SQL natif versionné et reluable ; le moteur de migration de Prisma prend un lock DDL et exige une shadow DB, ce qui casse la garantie « schéma = source de vérité versionnée » |
| ADR-0002 | Hébergement conteneurisé multi-services plutôt que Vercel | le worker BullMQ est un processus long-lived et les webhooks de paiement exigent une durée non contrainte ; un conteneur unique donne un pipeline unique, un `correlation_id` natif et un rollback par digest |
| ADR-0003 | Monolithe modulaire plutôt que microservices | 15 domaines métier et une équipe réduite : le découpage en services apporte des transactions distribuées et un coût opérationnel disproportionné ; l'isolation par module permet une extraction ultérieure sans refonte |
| ADR-0004 | Pattern outbox transactionnel pour la publication d'événements | la mise à jour de la commande et la publication de `payment.succeeded` doivent être atomiques ; l'outbox garantit la livraison sans double émission, avec des consommateurs idempotents |
| ADR-0005 | Monorepo pnpm workspaces + Turbo | les règles de tarification et le moteur de conformité doivent s'exécuter à l'identique sur `web`, `api` et `worker` ; un monorepo donne une version unique, un commit atomique et un pipeline unique |
| ADR-0006 | i18n par dictionnaires typés FR / EN / ZH-CN avec fallback explicite | le CDC §4.1 impose les trois langues dès la Phase 1 ; les dictionnaires sont typés pour que la parité des clés soit vérifiée en CI et non à la relecture manuelle |
| ADR-0007 | Aucun code HS classé par IA seule | le CDC interdit l'automatisation autonome des décisions de classification ; l'IA propose, le compliance manager valide, et la règle publiée porte un niveau de certitude explicite |
| ADR-0008 | Canary par reverse proxy plutôt que blue/green strict | blue/green double la capacité en permanence et masque les régressions lentes ; le canary 5/25/100 avec soak mesuré détecte la régression avant qu'elle ne touche tout le trafic |
| ADR-0009 | Migrations en expand / migrate / contract | le CDC §25.3 interdit la migration destructive sans preuve de restauration ; le cycle en trois déploiements garde l'ancienne version du code fonctionnelle pendant la fenêtre de bascule |
| ADR-0010 | Rollback roll-forward plutôt que `down` SQL | une migration appliquée ne revient pas en arrière par SQL ; le retour arrière se fait par un redéploiement de l'image précédente compatible avec le nouveau schéma, le `down` n'étant écrit que lorsque la réversibilité est prouvée |

---

### 12. Règles de commit et d'historique

**Secrets.** Aucun secret, clé d'API, mot de passe, token, PAN ou URL interne signée n'entre dans le dépôt, un bundle, un log ou un message de commit. Les secrets vivent dans Vault/KMS par environnement ; le dépôt ne contient que des **noms** de variables. Le scan est automatique (hook local + CI `secret-scan`) et la correction d'une fuite suit un ordre strict : **rotation d'abord, purge d'historique ensuite** (CDC §19.5).

**`.gitignore` (extrait structurant).**

```gitignore
node_modules/  .pnpm-store/                              # Dépendances
.next/ out/ dist/ build/  *.tsbuildinfo  .turbo/         # Builds
coverage/ test-results/ playwright-report/ blob-report/   # Tests
.env  .env.*  !.env.example   *.pem  *.key  *.log   .DS_Store  Thumbs.db
```

**`.env.example` commité, valeurs jamais présentes.** Le fichier liste les noms de variables, leur type, si elles sont obligatoires, et une description. Les valeurs par défaut locales sont dans `.env.local`, ignoré, généré par `pnpm secrets:gen` avec des valeurs factices. Extrait réel :

```dotenv
# Connexion PostgreSQL et cache
DATABASE_URL=postgresql://<user>:<password>@<host>:5432/<database>
REDIS_URL=redis://:<password>@<host>:6379/0
# Prestataire de paiement
PAYMENT_PROVIDER=  PAYMENT_SECRET_KEY=  PAYMENT_WEBHOOK_SECRET=
# Stockage objet
S3_BUCKET=  S3_ACCESS_KEY_ID=  S3_SECRET_ACCESS_KEY=  S3_REGION=
# E-mail transactionnel et observabilité
EMAIL_API_KEY=  EMAIL_FROM_ADDRESS=  LOG_LEVEL=info  OTEL_EXPORTER_OTLP_ENDPOINT=  SENTRY_DSN=
```

Le hook `scripts/check-env-file.mjs` refuse toute valeur ressemblant à un secret dans `.env*`, à l'exception de `.env.example` où seule la présence d'une valeur littérale non vide est signalée.

**Fichiers binaires et volumineux.** Aucun binaire dans Git : les médias et documents clients vont vers le stockage objet S3 avec CDN, servis par URL signée à durée courte, jamais dans le dépôt. Artefacts de build, `node_modules` et rapports de test sont ignorés. Pour un fichier réellement nécessaire au dépôt et volumineux (fontes CJK embarquées pour les PDF), Git LFS est utilisé : `packages/documents/assets/fonts/*.otf filter=lfs diff=lfs merge=lfs -text` (idem `*.ttf`), dans `.gitattributes`. Le hook `lint-staged` refuse tout binaire ajouté au commit (section 4).

**Merge et historique.** Les branches de fonctionnalité sont mergées en **squash-merge** : `main` reçoit un commit unique, au message conventionnel, dont le corps reprend les messages des commits squashés. Le commit de merge est interdit. Un commit direct sur `main` est refusé par le hook local et par le ruleset.

**Récupération d'un mauvais commit.** Sur une branche partagée, on ne réécrit jamais l'historique : on annule.

```bash
# Annulation d'un commit déjà mergé sur main : revert, jamais réécriture
git revert <sha>
# Annulation d'une série de commits
git revert --no-commit <sha_1> <sha_2> <sha_3>
git commit -m "revert: undo the price tier rollout"
# Réécriture autorisée uniquement sur une branche de feature personnelle, jamais poussée
git rebase -i origin/main
```

Après un `git revert`, le commit d'annulation est poussé sur `main` par une PR normale, avec les tests qui prouvent que le comportement précédent est revenu. Un incident en production se traite par un `git revert` mergé en urgence, pas par un `git reset --hard` sur `main`.

**Désindexer un fichier sans perdre le travail.**

```bash
# Retirer un ou plusieurs fichiers de l'index, les conserver dans le working tree
git restore --staged <chemin1> <chemin2>
# Annuler les modifications non indexées d'un seul fichier (destructif : le contenu est perdu)
git restore <chemin>
```

`git restore --staged` est la commande sûre : elle sort le fichier de l'index sans toucher au fichier sur le disque. `git reset --hard` et `git checkout -- <chemin>` **écrasent** le travail local et ne sont jamais utilisés pour « ranger » un fichier ; s'ils sont indispensables, l'état courant est d'abord sauvegardé (`git stash push -m "save"`, ou copie hors du dépôt).

**Nettoyage des branches.** `git branch --merged main` liste les branches mergées supprimables ; `git branch -d <branche>` supprime une branche locale mergée ; `git push origin --delete <branche>` supprime une branche distante, la PR devant être mergée au préalable. La suppression automatique après fusion est portée par `delete_branch_on_merge` du dépôt (section 2).

---

### 13. Checklist de fin de phase

Vérifiée **avant** l'ouverture de la PR de clôture de phase. Chaque item est coché par son propriétaire, avec la preuve en pièce jointe.

**Code et commits** — la branche de clôture respecte la regex de nommage et n'a pas dépassé 3 jours sans découpage ; le diff est ≤ 800 lignes, ou la PR est scindée en PRs empilées justifiées ; tous les commits suivent la convention (commitlint vert) et aucun ne mélange refactor et fonctionnalité ; aucun commit sur `main` dans l'historique de la phase ; aucun `force-push` sur une branche partagée.

**Tests et preuves** — suite unitaire verte, couverture ≥ 70 % global et ≥ 80 % sur `packages/pricing`, `packages/compliance`, `apps/api/src/modules/payments` (CDC §24.6) ; mutation score ≥ 95 % sur `packages/pricing` et `packages/compliance` ; tests d'intégration verts sur PostgreSQL réel (service container), migrations réellement appliquées ; les 5 suites E2E Playwright obligatoires vertes (B2C, B2B, paiement, administration, sécurité) ; aucune erreur console majeure sur les parcours affectés (job `console-guard`) ; axe : zéro violation critique ou sérieuse sur les parcours modifiés, avec revue manuelle clavier et lecteur d'écran consignée ; tests de charge exécutés sur les parcours chauds de la phase (catalogue, recherche, login, panier, checkout, webhooks), seuils p95 respectés ; test de restauration de sauvegarde exécuté et horodaté (RPO/RTO mesurés, CDC §19.8).

**Sécurité** — revue sécurité effectuée sur les chemins financiers, d'autorisation et de conformité, sans finding critique ouvert ; `secret-scan` et `dependency-scan` verts, aucune vulnérabilité critique non résolue sans acceptation écrite ; CodeQL et Semgrep sans finding critique ; matrice d'autorisation revue, delta documenté dans la PR et test IDOR de la ressource modifiée vert ; aucun secret, aucune donnée de carte, aucun PAN dans le dépôt, les bundles et les logs.

**Livraison et infrastructure** — migrations de la phase testées sur une base répliquée de la production anonymisée, preuve de restauration horodatée ; aucune migration destructive sans `-- @allow-destructive` + référence d'ADR ; build reproductible (deux builds successifs produisent le même digest), SBOM généré et archivé, image signée ; déploiement staging effectué, E2E critiques passés sur staging, canary observé sur toute la durée du soak (CDC §25.2) ; smoke tests de production verts après déploiement, alertes configurées et testées pour les événements critiques de la phase.

**Contenu, i18n et SEO** — les 3 locales sont à jour, la parité des clés est vérifiée, les textes chinois relus par une personne compétente ; les 2 thèmes sont couverts pour tout nouveau composant, contraste vérifié ; pour toute page publique indexable : title, meta, canonical, hreflang + `x-default`, JSON-LD, `robots.txt`, sitemap vérifiés ; aucun avis, note, badge ou donnée de synthèse inventé.

**Processus et traçabilité** — toutes les décisions structurantes de la phase sont écrites dans `docs/adr/` ; les runbooks de la phase existent et ont été suivis une fois de bout en bout par une personne qui ne les avait pas écrits ; le changelog de la phase est rédigé et la version est taguée (`vX.Y.Z`, tag signé) ; la PR de clôture est ouverte avec les preuves, les limites connues et le rollback documenté — **jamais mergée sans accord explicite** ; les décisions listées au §12 du CLAUDE.md projet (entité, prestataires, devises, pays, Incoterms, SLA) qui concernaient la phase sont tranchées et tracées, ou explicitement reportées avec un motif.

---

### 14. Anti-patterns Git

| # | Anti-pattern | Symptôme | Correction |
|---|---|---|---|
| 1 | **Committer sur `main`** | commits directs dans la branche de production, cherry-pick manuels, conflits au merge | créer une branche `feat/…` ou `fix/…` depuis `origin/main` ; le hook `git-guard` bloque le commit, le ruleset bloque le push |
| 2 | **Force-push sur une branche partagée** | l'historique distant est réécrit, les reviews déjà données invalidées silencieusement, les worktrees des collègues cassés | ne jamais réécrire une branche poussée : `--force-with-lease` seulement sur une branche personnelle jamais partagée ; sinon `git revert` |
| 3 | **Mélanger refactor et fonctionnalité** | 2 000 lignes de diff, revue impossible, retour arrière impossible car les deux changements sont couplés | deux PRs : d'abord le refactor sans changement de comportement (tests existants verts), puis la fonctionnalité |
| 4 | **Commiter des fichiers générés** | `.next/`, `coverage/`, `dist/`, `.turbo/` ou snapshots Playwright regénérés dans le diff | corriger `.gitignore` ; ne commiter un snapshot que s'il est relu et approuvé explicitement (le diff montre la régression, la relecture montre l'intention) |
| 5 | **Commits `fixup!` sur une branche partagée** | l'historique devient illisible, `git rebase --autosquash` chez un collègue réécrit l'historique des autres | `git commit --fixup` uniquement sur une branche personnelle ; sur une branche partagée, commiter normalement avec un scope distinct ou amender **avant** la revue |
| 6 | **PR de 2 000 lignes** | la revue devient un contrôle de forme, les bloqueurs passent inaperçus | scinder en PRs empilées séquentielles, chacune ≤ 800 lignes ; exception documentée pour un changement purement mécanique, validé par un outillage dédié |
| 7 | **Secret dans le diff** | clé d'API, mot de passe ou PAN commité, parfois poussé avant relecture | **rotation d'abord**, purge d'historique ensuite ; `.env` ignoré, `.env.example` avec des noms de variables uniquement ; `gitleaks` en hook et en CI |
| 8 | **Désactiver la CI pour débloquer** | `continue-on-error: true`, `if: false`, check renommé, workflow neutralisé | corriger la cause : la CI signale un vrai défaut ; un contournement non déclaré dans la PR fait échouer la revue et l'audit de phase |
| 9 | **Réécrire l'historique de `main`** | `git rebase -i main`, `git reset --hard`, `git commit --amend` puis push sur `main` | l'historique de `main` est public et signé : il ne se réécrit pas. `git revert` produit un commit d'annulation, mergé par PR normale |
| 10 | **Merger sans revue** | squash-merge direct depuis l'onglet Files changed, sans lecture du diff ni des conversations | le ruleset impose PR, checks requis, approbations, résolution des conversations et approbation du dernier push ; l'auteur ne fusionne jamais sa propre PR |
