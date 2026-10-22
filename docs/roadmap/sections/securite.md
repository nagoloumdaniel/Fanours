## Securite, conformite et donnees

Cadrage. Section normative : un ecart entre le code livre et cette section est un ecart au CDC Fanours v1.0 (`Cahier_des_charges_Fanours.md`). References citees : §2.3-§2.4 (activation, regles d'autorisation), §5.4 (le front n'est pas la source de verite), §6.3-§7.4 (catalogue, recherche, import), §8.1-§8.2 (B2B), §9.1-§9.5 (panier, checkout, etats, idempotence), §10.1-§10.5 (paiements), §11.1-§11.10 (douane/TVA), §13.1-§13.5 (auth, 2FA, sessions), §14.5 (factures), §15.5/§15.8 (securite client), §16.1-§16.13 (dashboard, roles), §18.1/§18.3 (recherche, IA), §19.1-§19.8 (securite applicative), §20.2 (integrite), §21.1-§21.4 (API/webhooks), §22.3 (PWA), §23.1-§23.3 (observabilite), §24.1-§24.6 (tests, CI), §25.1-§25.4 (environnements, pipeline), §26.1-§26.6 (criteres d'acceptation), §28.1-§28.2 (risques).

Niveaux de reference. OWASP **ASVS niveau 2 minimum** sur toute la plateforme ; **niveau 3** sur paiement (§10), roles et exports (§16.7, §16.12), documents B2B (§8.1) et moteur douanier (§11). OWASP Top 10 2021 (matrice §2) et OWASP API Security Top 10 sur `/api/*` et `/webhooks/*` (§21.1-§21.2). Regle de preuve : chaque controle declare porte un test automatise ou une procedure documentee ; un controle sans preuve est « non implemente ».

Trois invariants non negociables (§19.2, §19.4, §19.5) : (1) **aucune donnee de carte sensible chez Fanours** — le front ne voit qu'un jeton du prestataire ; (2) **deny by default** — tout acces non explicitement autorise est refuse cote serveur ; (3) **aucun secret dans le code, le build, les logs ni le bundle public**.

---

### 1. Modele de menace STRIDE par flux

« A AJOUTER » = ecart au CDC a livrer. Chaque ligne est un livrable de roadmap.

#### (a) Navigation catalogue et fiche produit (§6.3, §6.4, §7.1)

| STRIDE | Menace et impact | Controles CDC | A AJOUTER |
|---|---|---|---|
| Tampering | Prix, MOQ ou stock alteres via le payload panier : commande sous-payee | §9.1 recalcul serveur obligatoire ; §26.2 le backend recalcule | `enforceSalesMode` + `enforceMoq` sur **toutes** les routes panier/checkout, pas seulement l'UI ; test par mode |
| Elevation | Contournement B2C/B2B par appel API direct | §2.3, §2.4, §21.1 autorisation par endpoint | `role`/`price`/`discount` interdits dans les schemas d'entree ; `zod` `.strict()` sur tout payload public |
| Information disclosure | Fuite de prix de gros, stock ou marge vers le public | §7.2 paliers ; §16.2 separation des dashboards | Sortie publique limitee au palier eligible ; marge, cout et prix de gros jamais serialises |
| Repudiation | Impossible de prouver quel prix a ete affiche | §20.2 snapshot prix dans la commande | Journal `product.price.displayed` (produit, devise, prix, palier, horodatage), 180 j, sans PII |
| DoS / XSS | Recherche/IA non bornee ; HTML riche injecte | §18.1, §18.3, §23.1, §19.3 | Quota de recherche par IP + cache LRU, budget IA par session ; sanitisation serveur (DOMPurify + allowlist) avant persistance **et** rendu |

#### (b) Inscription, connexion, 2FA (§13.1-§13.5, §24.3)

| STRIDE | Menace et impact | Controles CDC | A AJOUTER |
|---|---|---|---|
| Spoofing | Prise de controle par rejeu d'un lien de reinitialisation | §13.1 lien court, usage unique, expiration | Jeton de reinitialisation hache (Argon2id), usage unique en base, invalidation de toutes les sessions |
| Spoofing | Usurpation via liaison Google a un compte existant | §13.2 pas de fusion par similitude de nom | `email_verified` + confirmation du compte lie avant fusion ; `nonce` + `state` en session |
| Elevation | Passage B2C vers B2B sans validation | §8.2 pas d'activation sur donnee falsifiable ; §16.6 | Decision reservee a `compliance_manager`/`super_admin`, tracee ; jamais d'auto-activation |
| Repudiation | Contestation d'une connexion ou d'un changement de mot de passe | §19.7 journal d'audit ; §13.5 historique | Journal immuable (§6.6) des evenements `auth.login`, `auth.login.failed`, `auth.mfa.*`, `auth.password.changed`, avec `correlation_id` |
| DoS / Disclosure | Brute force, credential stuffing ; enumeration de comptes | §13.1, §19.3 | Limites exactes §4.6 + lockout progressif ; reponse, statut et duree **identiques** que l'e-mail existe ou non (§9) |

#### (c) Verification de compte professionnel B2B (§8.1-§8.2, §16.6)

| STRIDE | Menace et impact | Controles CDC | A AJOUTER |
|---|---|---|---|
| Tampering | Falsification d'un justificatif (registre, licence) | §8.2 controles manuels ; pas d'activation definitive sur donnee falsifiable | Detection de document duplique (SHA-256 du fichier dechiffre), revue manuelle obligatoire, decision motivee |
| Information disclosure | Divulgation de documents d'entreprise a des roles non autorises | §8.1 fichiers sensibles chiffres, acces limite ; §19.6 | Chiffrement colonne en enveloppe KMS (§6.4), URL signee 5 min, **aucune** URL publique, journalisation de chaque lecture |
| Elevation | Validation accordee par un role non habilite ou en auto-validation | §16.6, §19.2 | `business.approve` exige `compliance_manager`/`super_admin` **et** MFA ; auto-approbation impossible (§3.5) |
| Repudiation / DoS | Approbation non tracee ; depot massif de justificatifs | §8.2 journaliser qui approuve ; §19.3 | Entree d'audit `business.review.*` (acteur, decision, motif, correlation_id) ; quota de documents, antivirus, taille max (§8) |

#### (d) Panier et checkout (§9.1-§9.2, §6.5)

| STRIDE | Menace et impact | Controles CDC | A AJOUTER |
|---|---|---|---|
| Tampering | Modification du total, des frais ou taxes entre panier et paiement | §9.1 recalcul serveur ; §9.2 verification complete | Snapshot `{prix, frais, taxes, devise, version_regle}` a l'entree en `pending_payment` ; ecart -> `checkout.stale` (409) et re-affichage |
| Tampering | Reduction abusive, cumul de codes promo | §9.1 conditions, expiration, journal | Grille de priorite unique et testee ; cumul interdit par defaut, cumulabilite en configuration ; rate limit par code (§9) |
| Elevation | Ajout au panier sans compte alors que le detail est reserve | §2.3, §2.4 | Regle serveur unique `resolveBuyerMode` appliquee a panier, devis, checkout et commande |
| Repudiation / Disclosure | Contestation du contenu de la commande ; PII dans les URL | §20.2 snapshot ; §21.1 erreurs a code stable | `order_snapshot` JSONB versionne, immuable apres `paid` ; aucune PII (CPF, e-mail complet, n° importateur) en query string ni dans les logs |

#### (e) Initiation et confirmation du paiement (§10.2-§10.3, §9.3)

| STRIDE | Menace et impact | Controles CDC | A AJOUTER |
|---|---|---|---|
| Spoofing | Faux retour navigateur simulant un paiement reussi | §6.5 etape 11 ; §26.3 aucun « paye » sur redirection | Le retour navigateur ne transite **jamais** a `paid` : seul le webhook signe l'ecrit |
| Tampering | Montant/devise modifies a la confirmation | §10.3 identifiant, devise, montant, statut ; §19.4 alertes | Coherence serveur commande ↔ PaymentIntent ↔ webhook ; ecart -> `payment_review` + alerte immediate (§5.5) |
| Elevation | Confirmation manuelle d'un `paid` depuis le back-office | §9.3 un remboursement n'est pas deduit du clic ; §16.7 | Aucune transition `→ paid` depuis l'admin : le back-office relance un rapprochement, il n'ecrit pas l'etat |
| Replay | Webhook rejoue : double facture, double stock, double notification | §9.5 cles d'idempotence, webhooks signes ; §28.1 | Table `webhook_event(provider, event_id)` unique + tolerance d'horodatage + rejeu = 409 journalise (§5.3) |
| DoS | Envoi massif de faux webhooks | §21.2 routes `/webhooks/*` | Limite stricte par IP et par provider ; schema de signature verifie **avant** toute lecture du corps metier |

#### (f) Webhook de paiement (§10.3, §21.2, §9.5)

| STRIDE | Menace et impact | Controles CDC | A AJOUTER |
|---|---|---|---|
| Spoofing / Tampering | Webhook forge ou corps modifie apres signature | §10.3 verifier les signatures ; §19.4 | HMAC-SHA256 sur `${timestamp}.${rawBody}`, secret du gestionnaire (§6.2), comparaison a temps constant, tolerance 300 s ; signature sur le **corps brut**, jamais sur le JSON reparse |
| Repudiation | Webhook recu mais non trace | §19.7 | Table `webhook_event` : provider, event_id, verdict, payload chiffre/redige, correlation_id |
| Information disclosure | Payload contenant PAN/CVV ou PII | §10.3 ne pas stocker PAN/CVV dans les logs | Redaction automatique (`card`, `pan`, `cvv`, `email`, `name` -> `[REDACTED]`) avant stockage |
| DoS | Rejeu massif du meme webhook | §9.5 | Idempotence applicative + rate limit par IP + dedup `event_id` |

#### (g) Creation de commande (§9.2-§9.3, §9.5, §20.2)

| STRIDE | Menace et impact | Controles CDC | A AJOUTER |
|---|---|---|---|
| Tampering | Double commande sur double clic ou retry reseau | §9.5 cles d'idempotence, verrou, contrainte unique | `idempotency_key` UNIQUE par utilisateur + route ; `SELECT ... FOR UPDATE` a la confirmation ; reponse identique au retry |
| Tampering | Stock negatif en concurrence | §7.3, §26.2 | `UPDATE stock SET qty = qty - $1 WHERE sku = $2 AND qty >= $1` + verification `rowCount = 1` ; test de concurrence (§13.2) |
| Elevation | Commande creee au nom d'un autre compte | §19.2 verification de propriete | `customer_id` derive de la session, jamais du body ; test IDOR sur `/orders/:id` (§3.4) |
| Repudiation / Integrity | Commande supprimee apres paiement ; montant commande ≠ montant capture | §9.3 une commande payee n'est pas supprimee ; §20.2 transactions atomiques | Suppression logique seule : `paid` -> aucun `DELETE` autorise, meme en base ; transaction unique commande + reservation stock + outbox + tentative de paiement, rollback integral |

#### (h) Administration catalogue et prix (§16.4, §7.1-§7.2, §17.1)

| STRIDE | Menace et impact | Controles CDC | A AJOUTER |
|---|---|---|---|
| Elevation | `catalog_manager` modifie un prix hors de son perimetre | §16.12 permissions par action et ressource | Matrice §3.2 : paliers = `super_admin` + `catalog_manager`, **4 yeux** au-dela d'un seuil ; `super_admin` ne peut pas supprimer un audit |
| Tampering | Modification de prix sans historique exploitable | §19.7 journaliser creation/modification et changement de prix | Diff JSON avant/apres en audit, hash du palier, raison obligatoire |
| Repudiation | Edition concurrente silencieuse | §20.2 | Optimistic locking (`version`) : `UPDATE ... WHERE id = $1 AND version = $2` -> 409 si conflit |
| Information disclosure | Fuite de la grille tarifaire interne | §7.2, §16.2 | Les paliers non applicables a l'acheteur ne sont jamais serialises en sortie publique |
| DoS | Import catalogue massif ou malveillant | §7.4 import/export catalogue | Import en job asynchrone, validation schema par ligne, taille limitee, rejeu atomique par lot |

#### (i) Moteur de regles douanieres (§11.2-§11.3, §11.5-§11.9, §16.9)

| STRIDE | Menace et impact | Controles CDC | A AJOUTER |
|---|---|---|---|
| Tampering | Regle modifiee pour sous-facturer | §11.2 regles versionnees ; §19.7 journal du changement | Toute regle diminuant les droits sans source officielle a jour est signalee et bloquee (§11.2) |
| Repudiation | Regle appliquee non identifiable apres coup | §11.2 version, date d'effet ; §16.9 sources, validateur, date de revision | Snapshot de la version de regle dans chaque commande et chaque document de douane |
| Integrity | Classement HS automatise sans validation humaine | §11.5 separer code suggere et confirme ; aucune IA ne peut confirmer seule | Statut `suggested|confirmed|expired` + `confirmed_by` + source ; expedition bloquee si `suggested` (etape 12) |
| Information disclosure / DoS | Fuite d'un dossier fournisseur (transporteur, fisc) ; resolution en cascade lente | §19.5, §21.4, §23.1 | Appels fournisseurs via proxy interne a allowlist (§7.4), secrets du gestionnaire ; memoisation des regles par (pays, categorie, date) et budget de calcul |

#### (j) Generation et telechargement de documents (§14.5, §11.8, §15.5, §16.7)

| STRIDE | Menace et impact | Controles CDC | A AJOUTER |
|---|---|---|---|
| Information disclosure | Facture ou document d'un autre client (IDOR) | §15.5, §19.2 | Chaque document telechargeable est lie a une ressource proprietaire ; test E2E IDOR sur `/documents/:id` (§3.4) |
| Spoofing | Document genere avec des donnees falsifiees (valeur, origine) | §11.8 documents d'expedition et de douane | Document genere **uniquement** depuis le snapshot de commande, jamais depuis des champs libres |
| Tampering / Repudiation | Document modifie apres edition ; preuve absente | §20.2, §19.7 | PDF/A signe, hash stocke, URL signee 5 min, version en audit ; journal `document.issued` (type, id commande, hash, destinataire, correlation_id) |

#### (k) Rapprochement financier et exports (§10.5, §16.7, §16.13)

| STRIDE | Menace et impact | Controles CDC | A AJOUTER |
|---|---|---|---|
| Information disclosure | Export massif de donnees clients ou comptables par un role trop permissif | §10.5 export CSV limite aux roles autorises ; §16.7 exports limites et journalises | Liste blanche de champs, lignes plafonnees, generation asynchrone, URL signee 24 h, audit `export.*` avec filtres et nombre de lignes |
| Tampering | Ecart de rapprochement masque ou ajuste a la main | §10.5 tout ecart annotable, attribue, suivi | Statut d'ecart avec historique de resolution ; aucun ecart resolu sans action tracee (§5.5) |
| Elevation | `analyst_readonly` exporte des donnees financieres | §16.12 | `export:financial` reserve a `finance_manager` + `super_admin`, 4 yeux (§3.2) ; test de non-regression du guard (§13.3) |
| DoS | Export de millions de lignes | §21.1 pagination et limites de taille | Export plafonne, asynchrone, notifie ; pas de telechargement direct massif |

#### (l) Dashboard admin et gestion des roles (§16.1-§16.13, §19.2, §19.7)

| STRIDE | Menace et impact | Controles CDC | A AJOUTER |
|---|---|---|---|
| Elevation | Acces admin sans MFA, ou role elargi par auto-attribution | §16.1 URL protegee, MFA obligatoire, RBAC, journalisation ; §19.2 | MFA impose **a chaque requete** `/admin/*` (pas seulement a la connexion) ; reauthentification pour actions sensibles (§4.5) |
| Tampering | Changement de role sans double controle | §19.2 double controle ; §19.7 journal du changement de role | `admin_role:manage_roles` exige 4 yeux, le demandeur ne peut pas etre l'approbateur (§3.5) |
| Repudiation / Disclosure | Action privilegiee non tracable ; profilage d'un administrateur | §19.7, §26.5, §23.2 | Audit immuable (§6.6) : acteur, action, objet, resultat, correlation_id, chaine de hash ; alerte sur export massif, acces a un document hors horaire, refus repetes |
| DoS | Requetes admin non bornees | §19.3 | Rate limit global admin plus strict que l'API publique, par action et par utilisateur (§4.6) |

**Regles transverses STRIDE.** (1) Le front n'est jamais la source de verite : prix, stock, role et devise sont re-evalues cote serveur (§5.4, §9.1). (2) `/admin/*` est protege par role **et** MFA **et** audit ; l'obscurite des URLs n'est jamais une protection. (3) Le journal d'audit (§19.7) est la source de verite forensique, non modifiable par aucun role metier. (4) Toute action financiere, de regle, de role ou d'export leve une alerte (§11.9, §14.2).

---

### 2. Matrice OWASP Top 10 2021 x modules Fanours

Chaque ligne nomme le controle applique (avec sa source CDC) **et** le test automatise qui le verifie. Un controle sans test dans la colonne de droite n'est pas livre.

| OWASP 2021 | Modules concernes | Controle specifique applique | Test automatise verifiant |
|---|---|---|---|
| **A01 Broken Access Control** | `/admin/*` (§21.2), commandes (§9.3), documents (§14.5), exports (§10.5), B2B (§8.2) | RBAC serveur + verification de propriete ; deny by default ; 4 yeux sur actions financieres et changement de role (§19.2, §16.12) | E2E `security/idor.spec.ts` : A lit `order_B` -> 404 ; `support_agent` sur `POST /api/admin/refunds` -> 403 ; `analyst_readonly` sur export financier -> 403 ; matrice 9 roles x 14 actions en test parametrique (§13.3) |
| **A02 Cryptographic Failures** | Sessions (§13.4), documents B2B (§8.1), secrets TOTP (§13.3), sauvegardes (§19.8) | Cookies `HttpOnly`/`Secure`/`SameSite` ; TLS 1.3 ; chiffrement au repos managé + chiffrement colonne en enveloppe KMS sur documents et secrets TOTP (§6.4) ; Argon2id pour mots de passe et codes de recuperation | `crypto/column-encryption.spec.ts` : la colonne `business_document.blob` est illisible sans la cle KMS ; `auth/session-cookie.spec.ts` verifie `HttpOnly`, `Secure`, `SameSite`, `Path` ; `npm audit --audit-level=high` sans CVE crypto bloquante |
| **A03 Injection** | Toutes entrees : recherche (§18.1), produits, commandes, import catalogue (§7.4) | Validation zod stricte en entree **et** en sortie (§21.1) ; requetes parametrees via ORM ; echappement React ; CSP sans `unsafe-inline` sur `script` (§7.5) | `validation/schema-fuzz.spec.ts` (fast-check, 500 cas/endpoint) : payload invalide -> 400, aucun fragment de payload dans la requete SQL ; `xss/reflected.spec.ts` : `<script>` et `"><img onerror>` reflectes -> 200 sans execution ; lint interdisant `.raw(` avec template literal |
| **A04 Insecure Design** | Concurrence stock (§7.3), idempotence commande (§9.5), transitions d'etat (§9.3), regles (§11.2), 4 yeux (§3.5) | Modele STRIDE par flux (§1) ; transitions codifiees ; cles d'idempotence + contrainte unique ; regles versionnees avec date d'effet | `concurrency/stock-oversell.spec.ts` : 50 requetes concurrentes sur stock 10 -> 10 succes, 40 refus, stock final 0 ; `payments/double-webhook.spec.ts` : 2 webhooks identiques -> 1 facture, 1 stock, 1 e-mail ; `state-machine/order-transitions.spec.ts` : transition non listee au §9.3 refusee |
| **A05 Security Misconfiguration** | Environnements (§25.1), CI (§24.6), en-tetes, CORS, debug | En-tetes de securite (§7.7) ; CORS allowlist stricte, jamais `*` avec credentials ; debug off en production ; DB non publique (§19.5) | `e2e/specs/security-headers.spec.ts` : 8 en-tetes verifies sur `/`, `/admin`, `/api` ; `integration/cors.spec.ts` : origine hors allowlist -> pas d'`Access-Control-Allow-Origin` ; pre-commit `gitleaks` + `trufflehog` bloquants ; scan d'image : non-root, pas de shell, ports minimaux (§12.2) |
| **A06 Vulnerable and Outdated Components** | Toute la dependance (framework, ORM, runner de paiement, SDK transporteur) | Dependabot/Renovate ; versions epinglees + lockfile ; SBOM par release ; scan de dependances bloquant (§24.6) | Job CI `dependency-scan` : `npm audit --audit-level=high` et Trivy sur l'image sortent 0 ; SBOM CycloneDX attache a l'artefact ; test de non-regression du scanner (dependance volontairement vulnerable detectee) |
| **A07 Identification and Authentication Failures** | Auth (§13.1-§13.3), sessions (§13.4), Google Sign-In (§13.2), MFA admin (§16.1) | Argon2id ; reponse identique a l'inscription et a la connexion ; TOTP RFC 6238 anti-replay par compteur ; codes de recuperation Argon2id usage unique ; rotation d'ID a la connexion et a l'elevation ; expiration inactive + absolue ; revocabilite multi-appareils ; limites §4.6 | `auth/mfa-replay.spec.ts` : meme code rejoue -> 401 ; `auth/recovery-code.spec.ts` : code deja consomme -> 401 ; `auth/session-rotation.spec.ts` : ID change apres login et apres elevation, l'ancien invalide ; `auth/rate-limit.spec.ts` : 6e login en 15 min -> 429 + `Retry-After` ; `auth/enumeration.spec.ts` : corps, statut et duree identiques |
| **A08 Software and Data Integrity Failures** | Webhooks (§10.3, §21.2), imports (§7.4), artefacts de build, sauvegardes (§19.8) | Signature HMAC + tolerance d'horodatage + anti-rejeu (§5.3) ; schema + signature valides sur import ; build reproductible et artefact signe (§12.3) ; sauvegarde chiffree et testee | `webhooks/signature.spec.ts` : signature invalide -> 401, corps non lu, aucun ecrit ; signature sur corps reparse -> 401 ; timestamp hors tolerance -> 401 ; `ci/build-reproducible.spec.ts` : deux builds du meme commit -> meme digest (cache off) ; signature d'artefact verifiee au deploiement |
| **A09 Security Logging and Monitoring Failures** | Toutes surfaces ; admin (§16.1), finance (§16.7), auth (§13.5) | Journal d'audit immuable (§6.6) ; `correlation_id` de bout en bout (§23.2) ; redaction des secrets et PII ; alertes sur actions sensibles ; runbook incident (§12.4) | `audit/immutability.spec.ts` : `UPDATE`/`DELETE` sur `audit_log` -> exception SQL (declencheur §6.6) ; `audit/redaction.spec.ts` : aucun `token`, `password`, `pan`, `cvv`, `cookie` dans les logs ; `audit/coverage.spec.ts` : les 13 evenements du §19.7 sont produits par un scenario E2E |
| **A10 SSRF** | Appels transporteur, calcul de transport, webhooks sortants, import URL, apercus de liens | Allowlist stricte des hotes sortants (§7.4) ; pas de fetch d'URL utilisateur ; resolution DNS re-verifiee a la connexion (anti-rebinding) ; timeout + taille max ; redirections limitees a 2 et revalidees | `unit/ssrf-guard.spec.ts` : `169.254.169.254`, `localhost:6379`, `[::1]`, `metadata.google.internal`, `2130706433` (decimal), `0x7f000001`, DNS rebinding -> tous rejetes ; `10.0.0.5` rejete ; un hote de l'allowlist passe |

**OWASP API Security Top 10 (complements).** API1 (BOLA) : cf. A01, test parametrique sur les 14 actions. API2 (authz) : pas de JWT utilisateur, session opaque verifiee a chaque endpoint prive (§21.1). API3 (proprietes d'objet) : `zod` `.strict()` interdit tout champ non declare (`role`, `is_admin`, `price`, `userId`). API4 (ressources) : pagination, `limit` max, rate limit par IP et par utilisateur, timeouts. API5 (niveau admin) : `/admin/*` exige role + MFA, jamais une cle secrete partagee. API6 : integrite metier = A04. API7 : cf. A10. API8 : cf. A05. API9 : OpenAPI diffuse, chaque route declaree. API10 : quotas IA (§18.3), recherche (§18.1), export (§10.5), webhook.

---

### 3. Controle d'acces, RBAC et anti-IDOR

#### 3.1 Les 9 roles (§16.12)

`super_admin`, `catalog_manager`, `order_manager`, `logistics_operator`, `finance_manager`, `support_agent`, `compliance_manager`, `marketing_manager`, `analyst_readonly`.

Regles de derivation (§16.12, §19.2) : `super_admin` n'est pas « tout permis » — ensemble **typé** et explicite, ce qui n'est pas liste est refuse, il ne peut ni supprimer une entree d'audit (§19.7) ni s'attribuer `compliance_manager` sans 4 yeux ; `analyst_readonly` : aucune ecriture et pas d'export financier (§16.7) ; `support_agent` consulte une commande mais ne rembourse pas, `catalog_manager` modifie une description mais ni les donnees bancaires ni l'audit (§16.12) ; separation des taches : `catalog_manager` ne modifie pas un prix de gros sans 4 yeux, `finance_manager` ne valide pas un compte B2B, `compliance_manager` n'accorde pas les acces aux regles ; le superadmin est un role **unique et nominatif**, jamais partage (§16.1).

#### 3.2 Matrice permission x ressource x action (14 actions sensibles)

Legende : `R` lecture, `W` ecriture, `-` interdit, `4E` quatre yeux requis, `M` MFA obligatoire sur l'action.

| # | Action | Ressource | super_admin | catalog_manager | order_manager | logistics_operator | finance_manager | support_agent | compliance_manager | marketing_manager | analyst_readonly | Regle |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Lire une commande | `order` | R | R | R | R | R | R | R | - | R | `support_agent` en lecture limitee au necessaire (§16.11) |
| 2 | Modifier une commande | `order` | W | - | W | - | - | - | - | - | - | `order_manager` ; modification de montant reservee au mecanisme de paiement (§9.4) |
| 3 | Creer un produit | `product` | W | W | - | - | - | - | - | - | - | `catalog_manager` ; publication en 3 langues soumise a relecture (§24.3) |
| 4 | Modifier un prix / palier | `price_tier` | W 4E | W 4E | - | - | - | - | - | - | - | 4E au-dela du seuil ; version optimiste + diff en audit |
| 5 | Export de donnees clients | `export` | W | - | - | - | R | R (limite) | R | R (limite) | - | Liste blanche de champs + lignes plafonnees + URL signee expiree |
| 6 | Export financier | `export_financial` | W 4E | - | - | - | W 4E | - | - | - | - | `finance_manager` + `super_admin`, 4E, journalise (§16.7) |
| 7 | Rembourser | `refund` | W 4E | - | - | - | W 4E | - | - | - | - | Motif obligatoire, confirmation prestataire (§10.4), **jamais** de double remboursement (§5.6) |
| 8 | Gerer les roles | `admin_role` | W 4E | - | - | - | - | - | - | - | - | 4E obligatoire, le demandeur ne peut pas etre l'approbateur (§19.2) |
| 9 | Exporter le journal d'audit | `audit_log` | R | - | - | - | R | - | R | - | - | Aucun role metier ne peut **modifier** le journal (§19.7) |
| 10 | Modifier une regle douaniere | `customs_rule` | W | - | - | - | - | - | W | - | - | Source + date de revision obligatoires (§11.2, §16.9) ; valeur suspecte -> alerte |
| 11 | Suspendre un pays | `country` | W 4E | - | - | - | - | - | W | - | - | Desactivation d'urgence avec justification et audit (§11.6) |
| 12 | Lire un document de verification | `business_document` | R | - | - | - | - | R (dossier) | R | - | - | Acces trace ligne par ligne ; URL signee 5 min ; aucun URL public |
| 13 | Valider un compte B2B | `business_account` | W | - | - | - | - | - | W | - | - | Decision motivee + notification (§8.2) ; jamais d'auto-activation |
| 14 | Generer/voir un document douanier | `customs_document` | R | R | R | W | R | - | W | - | - | Genere depuis le snapshot de commande, jamais depuis des champs libres |

Exclusions negatives codees dans le type, pas seulement dans l'UI : `compliance_manager` n'a pas l'export financier ; `marketing_manager` n'a pas le journal d'audit ; `analyst_readonly` n'a aucune ecriture.

#### 3.3 Schema et guard d'autorisation (deny by default, type-cheque)

```ts
// src/authz/permissions.ts
export const RESOURCES = ["order","product","price_tier","export","export_financial","refund",
  "admin_role","audit_log","customs_rule","country","business_document","business_account","customs_document"] as const;
export type Resource = (typeof RESOURCES)[number];
export type Role = "super_admin" | "catalog_manager" | "order_manager" | "logistics_operator"
  | "finance_manager" | "support_agent" | "compliance_manager" | "marketing_manager" | "analyst_readonly";
export type Action = "read" | "create" | "update" | "delete" | "export" | "approve" | "refund" | "manage_roles";

/** Deny by default : une permission absente est un refus, pas une hesitation. */
export type PermissionTable = { readonly [R in Role]?: { readonly [Res in Resource]?: readonly Action[] } };

// Generee depuis la matrice §3.2 et verifiee par test parametrique (§13.3) :
// toute divergence entre la matrice et le code est un echec de test, pas une doc obsolete.
export const PERMISSIONS: PermissionTable = {
  super_admin: { order:["read","update"], product:["read","create","update"], price_tier:["read","update"],
    export:["export"], export_financial:["export"], refund:["refund"], admin_role:["manage_roles"],
    audit_log:["read"], customs_rule:["read","update"], country:["update"], business_document:["read"],
    business_account:["read","approve"], customs_document:["read"] },
  catalog_manager: { order:["read"], product:["read","create","update"], price_tier:["read","update"],
    export:["export"], customs_document:["read"] },
  order_manager: { order:["read","update"], product:["read"], export:["export"], customs_document:["read"] },
  logistics_operator: { order:["read"], customs_document:["read","create","update"] },
  finance_manager: { order:["read"], export:["export"], export_financial:["export"], refund:["refund"],
    audit_log:["read"], customs_document:["read"] },
  support_agent: { order:["read"], export:["export"], business_document:["read"] },
  compliance_manager: { order:["read"], export:["export"], audit_log:["read"], customs_rule:["read","update"],
    business_document:["read"], business_account:["read","approve"], customs_document:["read"] },
  marketing_manager: { product:["read"], export:["export"] },
  analyst_readonly: { order:["read"], product:["read"] },
} as const;

const FOUR_EYES = new Set<string>([
  "super_admin:price_tier:update","catalog_manager:price_tier:update",
  "super_admin:export_financial:export","finance_manager:export_financial:export",
  "super_admin:refund:refund","finance_manager:refund:refund",
  "super_admin:admin_role:manage_roles","super_admin:country:update",
]);

export function decide(role: Role, resource: Resource, action: Action): "allow" | "four_eyes" | "deny" {
  if (!PERMISSIONS[role]?.[resource]?.includes(action)) return "deny"; // absence => refus
  return FOUR_EYES.has(`${role}:${resource}:${action}`) ? "four_eyes" : "allow";
}
```

```ts
// src/authz/guard.ts
import { decide, type Action, type Resource, type Role } from "./permissions";
import { ForbiddenError } from "../errors";

export type Principal = { userId: string; role: Role; mfaVerifiedAt: number | null; sessionId: string };
const MFA_MAX_AGE_MS = 10 * 60 * 1000;   // MFA revalide a chaque requete admin
const PRIVILEGED = new Set<string>(["admin_role:manage_roles","refund:refund","export_financial:export",
  "price_tier:update","country:update","customs_rule:update","business_account:approve","business_document:read"]);

export function requirePermission(principal: Principal, resource: Resource, action: Action,
  opts: { fourEyes?: { requesterId: string; approverId?: string } } = {}): void {
  const decision = decide(principal.role, resource, action);
  if (decision === "deny") throw new ForbiddenError("action_not_permitted"); // 403 sans divulguer l'existence
  if (PRIVILEGED.has(`${resource}:${action}`) &&
      (!principal.mfaVerifiedAt || Date.now() - principal.mfaVerifiedAt > MFA_MAX_AGE_MS)) {
    throw new ForbiddenError("mfa_required");
  }
  if (decision === "four_eyes" || opts.fourEyes) {
    if (!opts.fourEyes?.approverId) throw new ForbiddenError("approval_required");
    if (opts.fourEyes.approverId === opts.fourEyes.requesterId) throw new ForbiddenError("self_approval_forbidden");
  }
}
```

#### 3.4 Anti-IDOR : verification de propriete en cascade

Regle : l'identifiant de la ressource vient de l'URL, l'autorite vient de la session. La propriete est verifiee **au moment de la lecture** par un `where` compose, jamais par un controle applicatif separe (fenetre TOCTOU).

```ts
// src/orders/ownership.ts
import { db } from "../db";
import { NotFoundError } from "../errors";

const STAFF_ROLES: ReadonlySet<Role> = new Set<Role>(["super_admin","catalog_manager","order_manager",
  "logistics_operator","finance_manager","support_agent","compliance_manager","marketing_manager"]);

/** Anti-IDOR : la commande n'est renvoyee que si elle appartient au principal (ou si le
 *  principal a un droit metier de lecture). 404 dans les deux cas : l'existence n'est pas revelee. */
export async function loadOrderForPrincipal(principal: Principal, orderId: string) {
  const order = await db.order.findFirst({
    where: { id: orderId,
      ...(STAFF_ROLES.has(principal.role) ? {} : { customerId: principal.userId }) },
    select: { id: true, customerId: true, status: true, totalAmount: true, currency: true },
  });
  if (!order) throw new NotFoundError("order_not_found");
  return order;
}
```

Regles dures associees : identifiants publics opaques (UUIDv7 / CUID2), **jamais** des entiers sequentiels ; `customerId`, `role`, `price` ne sont **jamais** acceptes depuis le body — ils sont derives de la session et des snapshots serveur (§5.4, §20.2) ; la projection Drizzle est explicite (liste blanche de colonnes), pas de fuite de champs internes (§21.1) ; test E2E obligatoire : remplacer un ID dans l'URL ne donne jamais acces a la commande d'un autre utilisateur (§19.2, §24.3).

#### 3.5 Double controle (quatre yeux) sur actions financieres

Modele retenu : demande, approbation et execution sont trois actes distincts, traces separement. La demande fige le payload par son hash ; toute modification ulterieure est detectee.

```sql
-- packages/db/migrations/0003_rbac_and_audit.sql (Phase 1)
CREATE TYPE approval_status AS ENUM ('pending','approved','rejected','expired','executed','cancelled');
CREATE TYPE approval_kind AS ENUM ('refund','financial_export','price_tier_change','role_change','country_suspension');
CREATE TABLE admin_action_approval (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  kind             approval_kind   NOT NULL,
  payload          jsonb           NOT NULL,          -- action demandee, parametres figes
  payload_hash     text            NOT NULL,          -- SHA-256 du payload canonique
  amount           numeric(14,2),
  currency         char(3),
  status           approval_status NOT NULL DEFAULT 'pending',
  requested_by     uuid            NOT NULL REFERENCES app_user(id),
  requested_at     timestamptz     NOT NULL DEFAULT now(),
  approved_by      uuid            REFERENCES app_user(id),
  approved_at      timestamptz,
  rejection_reason text,
  expires_at       timestamptz     NOT NULL,          -- 24 h par defaut
  executed_at      timestamptz,
  execution_ref    text,                               -- reference externe (remboursement, export)
  correlation_id   uuid            NOT NULL,
  CONSTRAINT no_self_approval CHECK (approved_by IS NULL OR approved_by <> requested_by),
  CONSTRAINT executed_fields CHECK (status <> 'executed' OR (executed_at IS NOT NULL AND execution_ref IS NOT NULL))
);
CREATE UNIQUE INDEX uq_approval_pending ON admin_action_approval (kind, requested_by) WHERE status = 'pending';
CREATE INDEX ix_approval_kind_status ON admin_action_approval (kind, status);
```

```ts
// src/admin/approvals.ts
import { db } from "../db";
import { ForbiddenError, ConflictError } from "../errors";
import { audit } from "../audit";

type ApprovalRow = { id: string; kind: string; payload: unknown; payload_hash: string;
  amount: string | null; currency: string | null;
  status: "pending"|"approved"|"rejected"|"expired"|"executed"|"cancelled";
  requested_by: string; expires_at: Date; approved_by: string | null };

/** Demande d'action sensible : ne change rien au modele de domaine. */
export async function requestApproval(input: { kind: string; payload: unknown; amount?: number;
  currency?: string; requestedBy: string; correlationId: string }) {
  const payloadHash = sha256(canonicalJson(input.payload));
  const approval = await db.adminActionApproval.create({ data: { kind: input.kind, payload: input.payload,
    payload_hash: payloadHash, amount: input.amount ?? null, currency: input.currency ?? null,
    requested_by: input.requestedBy, correlation_id: input.correlationId,
    expires_at: new Date(Date.now() + 24 * 3600 * 1000) } });
  await audit.write({ actorId: input.requestedBy, action: `approval.requested.${input.kind}`,
    objectType: "admin_action_approval", objectId: approval.id, result: "success",
    correlationId: input.correlationId, metadata: { payload_hash: payloadHash } });
  return approval;
}

/** Approbation : l'approbateur ne peut pas etre le demandeur (contrainte SQL + garde applicative). */
export async function approve(input: { approvalId: string; approverId: string; correlationId: string }) {
  return db.$transaction(async (tx) => {
    const rows = await tx.$queryRaw<ApprovalRow[]>`SELECT * FROM admin_action_approval
      WHERE id = $1 AND status = 'pending' AND expires_at > now() FOR UPDATE`, [input.approvalId];
    if (rows.length === 0) throw new ConflictError("approval_not_pending_or_expired");
    if (rows[0].requested_by === input.approverId) throw new ForbiddenError("self_approval_forbidden");
    if (sha256(canonicalJson(rows[0].payload)) !== rows[0].payload_hash) throw new ConflictError("payload_tampered");
    const updated = await tx.adminActionApproval.update({ where: { id: input.approvalId },
      data: { status: "approved", approved_by: input.approverId, approved_at: new Date() } });
    await audit.write({ actorId: input.approverId, action: `approval.approved.${rows[0].kind}`,
      objectType: "admin_action_approval", objectId: input.approvalId, result: "success",
      correlationId: input.correlationId });
    return updated;
  });
}

/** Execution : reverifie que le payload n'a pas change entre approbation et execution. */
export async function execute(approvalId: string, actorId: string, correlationId: string) {
  return db.$transaction(async (tx) => {
    const rows = await tx.$queryRaw<ApprovalRow[]>`SELECT * FROM admin_action_approval
      WHERE id = $1 AND status = 'approved' FOR UPDATE`, [approvalId];
    if (rows.length === 0) throw new ConflictError("approval_not_approved");
    if (sha256(canonicalJson(rows[0].payload)) !== rows[0].payload_hash) throw new ConflictError("payload_tampered");
    const result = await executeApprovedAction(rows[0], actorId);   // effets de domaine
    await tx.adminActionApproval.update({ where: { id: approvalId },
      data: { status: "executed", executed_at: new Date(), execution_ref: result.reference } });
    await audit.write({ actorId, action: `approval.executed.${rows[0].kind}`,
      objectType: "admin_action_approval", objectId: approvalId, result: "success",
      correlationId, metadata: { execution_ref: result.reference } });
    return result;
  });
}
```

Gouvernance associee : un approbateur ne peut pas approuver une demande qu'il a lui-meme portee plus de N fois (anti-conflit d'interets) ; le seuil de 4 yeux est configurable par type d'action et par montant (§19.2), un montant faible pouvant passer en 2 yeux selon une regle documentee et auditee ; une approbation expire (24 h) et ne se rafraichit pas (nouvelle demande requise) ; l'approbation n'est jamais implicite (ni par delai, ni par absence de rejet).

---

### 4. Authentification, sessions et MFA

#### 4.1 Session : cookie opaque, pas de JWT

Choix : **identifiant de session opaque** en base (256 bits aleatoires, stocke hache), pas de JWT — le CDC impose la revocation de sessions (§13.4), la liste des appareils (§13.5) et la rotation d'identifiant (§13.4), ce qu'un JWT auto-contenu ne permet pas sans liste de deni. Un JWT **interne** peut exister pour la communication de service, mais ne porte jamais d'etat de session ni de role client (les roles viennent du serveur, §16.12).

| Attribut | Valeur | Raison |
|---|---|---|
| Nom | `fanours_sid` | Non predictible par convention, distinct de `fanours_rt` |
| `HttpOnly` | `true` | Inaccessible au JavaScript : bloque l'exfiltration via XSS (§13.4, §19.3) |
| `Secure` | `true` | HTTPS uniquement (§13.4) |
| `SameSite` | `Lax` | `Strict` casserait le retour apres paiement externe (redirection cross-site) ; `Lax` + double-submit CSRF (§7.5) + verification `Origin` couvrent le risque |
| `Path` | `/` | Couvre `/api` et les pages ; le cookie de refresh est scope a `/api/auth/refresh` |
| `Domain` | absent (host-only) | Evite la fuite a un sous-domaine tiers ; pas de `Domain=.fanours.com` |
| `Max-Age` | `900` (15 min) | Expiration inactive et absolue (§13.4) |

Politique d'expiration (§13.4) : **inactive** 30 min (client), 15 min (admin, MFA), 60 min (session B2B approuvee en attente de virement) ; **absolue** 12 h (client), 8 h (back-office), 30 min (session en elevation sensible) meme si l'utilisateur est actif. Le refresh est plafonne a 30 jours mais **revoque** a chaque changement de mot de passe, changement d'e-mail, suspicion de compromission et deconnexion globale (§13.4, §13.5).

```ts
// src/auth/session.ts
import { randomBytes, createHash } from "node:crypto";
import { db } from "../db";

const SESSION_IDLE_MS = 30 * 60 * 1000, SESSION_ABSOLUTE_MS = 12 * 60 * 60 * 1000;
const hash = (v: string) => createHash("sha256").update(v).digest("hex");

/** L'identifiant brut ne quitte jamais le serveur : on ne stocke que son SHA-256. */
export async function createSession(input: { userId: string; ip: string; userAgent: string; role: string; mfa?: boolean }) {
  const raw = randomBytes(32).toString("base64url"), now = new Date();
  await db.session.create({ data: { id_hash: hash(raw), user_id: input.userId, role: input.role,
    mfa_verified: input.mfa ?? false, mfa_verified_at: input.mfa ? now : null,
    ip_hash: hash(input.ip), user_agent: input.userAgent.slice(0, 512),
    created_at: now, last_seen_at: now,
    absolute_expires_at: new Date(now.getTime() + SESSION_ABSOLUTE_MS) } });
  return raw; // place dans le cookie HttpOnly
}

/** Rotation : l'ancien identifiant est invalide immediatement (fixation de session). */
export async function rotateSession(rawSid: string): Promise<string> {
  const next = randomBytes(32).toString("base64url");
  await db.$transaction(async (tx) => {
    const rows = await tx.$queryRaw<{ id: string }[]>`SELECT id FROM session WHERE id_hash = $1 FOR UPDATE`,
      [hash(rawSid)];
    if (rows.length === 0) throw new Error("session_not_found");
    const sess = await tx.session.findUniqueOrThrow({ where: { id: rows[0].id } });
    await tx.session.update({ where: { id: sess.id }, data: { revoked_at: new Date(), revoke_reason: "rotated" } });
    await tx.session.create({ data: { id_hash: hash(next), user_id: sess.user_id, role: sess.role,
      mfa_verified: sess.mfa_verified, mfa_verified_at: sess.mfa_verified_at, ip_hash: sess.ip_hash,
      user_agent: sess.user_agent, created_at: new Date(), last_seen_at: new Date(),
      absolute_expires_at: sess.absolute_expires_at } });
  });
  return next;
}
```

Regles de rotation (§13.4) : **a la connexion** (systematique, le cookie pre-connexion n'est jamais reutilise) et **a l'elevation** (passage admin, activation MFA, validation B2B, changement d'e-mail/mot de passe, remboursement sensible → §13.3). Le rejeu d'une ancienne requete donne `session_not_found` → 401, jamais 500.

#### 4.2 MFA : TOTP RFC 6238 avec anti-replay par compteur

Le CDC impose TOTP via application d'authentification, codes de recuperation generes une fois et stockes hache, MFA obligatoire admin et roles finance/conformite (§13.3), aucun code en clair (§13.3).

```ts
// src/auth/totp.ts
import { createHmac, timingSafeEqual } from "node:crypto";
import { db } from "../db";
import { decryptSecret } from "../crypto/envelope";

const TOTP_STEP = 30;         // secondes (RFC 6238)
const TOTP_DIGITS = 6;
const WINDOW = 1;             // tolerance +/- 1 pas de 30 s
const MAX_COUNTER_DRIFT = 3;  // anti-replay : nombre max de pas rejouables

export type TotpResult = { ok: true; counter: number } | { ok: false; reason: string };

/** RFC 6238 : T = floor(unixTime/30), code = HOTP(secret, T) sur 30 s.
 *  Anti-replay : le dernier compteur accepte est memorise par secret ; un code
 *  d'un compteur deja utilise est refuse, meme s'il est dans la fenetre. */
export async function verifyTotp(input: { userId: string; code: string; serverTime?: number }): Promise<TotpResult> {
  const nowSec = Math.floor(input.serverTime ?? Date.now() / 1000);
  const secret = await loadDecryptedTotpSecret(input.userId);
  if (!secret) return { ok: false, reason: "totp_not_enrolled" };
  const counter = Math.floor(nowSec / TOTP_STEP);
  const state = await db.mfaTotpState.findUnique({ where: { user_id: input.userId } });
  const minCounter = (state?.last_counter ?? -1) + 1;
  if (counter - minCounter > MAX_COUNTER_DRIFT) return { ok: false, reason: "totp_too_old" };
  for (let d = -WINDOW; d <= WINDOW; d++) {
    const c = counter + d;
    if (c < minCounter) continue;
    if (secureEquals(hotp(secret, c), input.code)) {
      await db.mfaTotpState.upsert({ where: { user_id: input.userId },
        create: { user_id: input.userId, last_counter: c }, update: { last_counter: c } });
      return { ok: true, counter: c };
    }
  }
  return { ok: false, reason: "totp_invalid" };
}

function hotp(secret: Buffer, counter: number): string {
  const buf = Buffer.alloc(8); buf.writeBigUInt64BE(BigInt(counter));
  const hmac = createHmac("sha1", secret).update(buf).digest();
  const offset = hmac[hmac.length - 1] & 0x0f;
  const code = ((hmac[offset] & 0x7f) << 24) | ((hmac[offset+1] & 0x0xff) << 16) |
               ((hmac[offset+2] & 0xff) << 8) | hmac[offset+3];
  return String(code % 10 ** TOTP_DIGITS).padStart(TOTP_DIGITS, "0");
}
function secureEquals(a: string, b: string): boolean {
  const ba = Buffer.from(a), bb = Buffer.from(b);
  return ba.length === bb.length && timingSafeEqual(ba, bb);
}
```

Le **secret TOTP** n'est jamais stocke en clair (§13.3) : chiffrement AES-256-GCM en enveloppe, cle dans le KMS (§6.4), version de cle stockee pour permettre la rotation. Algorithme et longueur sont fixes a l'enrollment (SHA-1 / 160 bits, conforme RFC 6238) et le secret n'est montre qu'une fois, sur une page dediee.

#### 4.3 Codes de recuperation : Argon2id, usage unique

```ts
// src/auth/recovery-codes.ts
import { randomBytes } from "node:crypto";
import argon2 from "argon2";
import { db } from "../db";

const N = 2 ** 16, r = 3, p = 1;                                   // Argon2id, params OWASP
const HASH_OPTS = { type: argon2.argon2id, memoryCost: N, timeCost: r, parallelism: p };
const normalize = (c: string) => c.replace(/[-\s]/g, "").toUpperCase();

export async function generateRecoveryCodes(userId: string): Promise<string[]> {
  const codes = Array.from({ length: 10 }, () =>
    randomBytes(5).toString("hex").toUpperCase().replace(/(.{5})/, "$1-"));
  await db.recoveryCode.createMany({ data: await Promise.all(codes.map(async (code) => ({
    user_id: userId, code_hash: await argon2.hash(normalize(code), HASH_OPTS), used_at: null }))) });
  return codes; // affiches une seule fois, jamais stockes en clair
}

export async function consumeRecoveryCode(userId: string, raw: string): Promise<boolean> {
  const candidates = await db.recoveryCode.findMany({ where: { user_id: userId, used_at: null } });
  for (const row of candidates) {
    if (await argon2.verify(row.code_hash, normalize(raw))) {
      // usage unique garanti par le filtre `used_at: null` sur l'UPDATE
      const res = await db.recoveryCode.updateMany({ where: { id: row.id, used_at: null },
        data: { used_at: new Date() } });
      if (res.count === 1) return true;   // un seul code peut etre consomme
    }
  }
  return false;
}
```

Regles : 10 codes, affichage unique, hachage Argon2id (§13.3), usage unique strict (le second usage echoue, meme depuis la session d'origine), invalidation de tous les codes a l'activation d'un nouveau TOTP, **deactivation de la session courante** apres usage (l'attaquant qui a vole un code ne garde pas la session).

#### 4.4 Google Sign-In : validation serveur, state/nonce, anti-liaison (§13.2)

```ts
// src/auth/google.ts
const GOOGLE_JWKS = "https://www.googleapis.com/oauth2/v3/certs";
const GOOGLE_ISSUERS = ["https://accounts.google.com", "accounts.google.com"];

/** Validation complete cote serveur : signature (JWKS), issuer, audience, exp, nonce. */
export async function verifyGoogleIdToken(args: { idToken: string; expectedNonce: string; clientId: string }):
  Promise<{ sub: string; email: string; emailVerified: boolean }> {
  const header = decodeSegment(args.idToken.split(".")[0]);
  if (header.alg !== "RS256") throw new Error("alg_not_allowed"); // jamais 'none', jamais HS256
  const jwk = await jwks.get(header.kid, GOOGLE_JWKS);
  const claims = await verifyRs256(args.idToken, jwk);
  if (!GOOGLE_ISSUERS.includes(claims.iss)) throw new Error("bad_issuer");
  if (claims.aud !== args.clientId) throw new Error("bad_audience");
  if (typeof claims.exp !== "number" || claims.exp * 1000 < Date.now()) throw new Error("expired");
  if (!claims.nonce || !timingSafeEqual(Buffer.from(claims.nonce), Buffer.from(args.expectedNonce)))
    throw new Error("nonce_mismatch");                            // anti-rejeu de l'ID token
  if (claims.email_verified !== true) throw new Error("email_not_verified");
  return { sub: claims.sub, email: String(claims.email).toLowerCase(), emailVerified: true };
}
```

Regles complementaires (§13.2) : le `state` est un jeton CSRF a usage unique lie a la session de depart, expire en 10 min ; le `nonce` est derive de la session (`HMAC(sessionId, serverPepper)`) et non reutilisable ; les redirect URIs sont exacts (pas de wildcard) ; le secret OAuth vit dans le gestionnaire de secrets (§6.2) et n'apparait jamais dans le bundle public. **Anti-liaison** : si un compte local existe avec la meme adresse verifiee, la liaison n'est automatique que si le compte local est deja verifie par e-mail **et** que l'email du fournisseur est verifie ; sinon reauthentification (mot de passe existant) ou creation d'un nouveau compte. Aucune fusion sur simple similitude de nom. La de-association est possible sans perdre l'acces.

#### 4.5 Reauthentification et elevation

Actions exigeant reauthentification recente (< 5 min) **et** MFA (§13.3) : changer l'e-mail, changer le mot de passe, desactiver le MFA, lancer un remboursement sensible, modifier un role, exporter des donnees financieres, modifier une regle douaniere, desactiver un pays.

#### 4.6 Limites de debit d'authentification (valeurs exactes)

| Endpoint | Limite | Cle | Fenetre | Raison |
|---|---|---|---|---|
| `POST /api/auth/login` | **5 tentatives / 15 min** | IP **et** e-mail normalise — les deux sous le seuil | 15 min | §13.1, §19.3 anti brute force et credential stuffing |
| `POST /api/auth/register` | **3 / heure** | IP | 1 h | Anti-spam et anti-enumeration |
| `POST /api/auth/password-reset/request` | **3 / heure** | IP **et** e-mail | 1 h | §13.1 abus de recuperation |
| `POST /api/auth/password-reset/confirm` | **5 / heure** | IP | 1 h | Anti brute force du lien |
| `POST /api/auth/mfa/verify` | **5 / 10 min** | IP + user_id | 10 min | Anti brute force du code TOTP |
| `GET /api/search` | **60 / min** | IP | 1 min | §19.3 abus de recherche (et cout moteur/IA, §18.1) |
| `GET/POST /api/*` | **600 / min** | user_id si authentifie, sinon IP | 1 min | §19.3 abus d'API |
| `POST /api/webhooks/*` | **120 / min** par IP, **600 / min** par provider | IP + provider | 1 min | §21.2 flood de faux webhooks |
| `POST /api/contact`, `POST /api/quotes` | **5 / heure** par IP, **3 / jour** par e-mail | IP + e-mail | 1 h / 24 h | Anti-spam |
| `POST /api/cart/promo` | **10 / 10 min** | IP + code promo | 10 min | Anti-abus sur codes promo |
| `/api/admin/*` | **300 / min** par utilisateur | user_id | 1 min | §19.3 |

```ts
// src/http/rate-limit.ts
import { Redis } from "ioredis";
import { randomUUID } from "node:crypto";

const redis = new Redis(process.env.REDIS_URL!); // fail-fast : pas de fallback in-memory
type Rule = { limit: number; windowSec: number; key: (req: Req) => string };

/** Fenetre glissante Redis (ZSET) : compteur exact, atomique, partage entre instances.
 *  Cle composite = action + IP (+ utilisateur) : le contournement par rotation d'IP
 *  ne passe pas la limite utilisateur, et un utilisateur derriere une IP partagee
 *  (NAT, VPN) n'est pas bloque a cause des autres. */
export async function consume(rule: Rule, req: Req): Promise<{ allowed: boolean; retryAfter: number }> {
  const now = Date.now(), member = `${now}:${randomUUID()}`, zkey = `rl:${rule.key(req)}`;
  const lua = `
    local key = KEYS[1] local now = tonumber(ARGV[1]) local window = tonumber(ARGV[2])
    local limit = tonumber(ARGV[3]) local member = ARGV[4]
    redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
    local count = redis.call('ZCARD', key)
    if count < limit then
      redis.call('ZADD', key, now, member) redis.call('PEXPIRE', key, window)
      return {1, limit - count - 1}
    end
    local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
    local retry = math.ceil((tonumber(oldest[2]) + window - now) / 1000)
    redis.call('PEXPIRE', key, window) return {0, retry}`;
  const res = (await redis.eval(lua, 1, zkey, now, rule.windowSec * 1000, rule.limit, member)) as [number, number];
  return res[0] === 1 ? { allowed: true, retryAfter: 0 } : { allowed: false, retryAfter: Math.max(1, res[1]) };
}

export const RULES: Record<string, Rule> = {
  "auth:login":    { limit: 5,   windowSec: 900, key: r => `login:${r.ip}:${r.email ?? ""}` },
  "auth:register": { limit: 3,   windowSec: 3600, key: r => `reg:${r.ip}` },
  "auth:reset":    { limit: 3,   windowSec: 3600, key: r => `reset:${r.ip}:${r.email ?? ""}` },
  "auth:mfa":      { limit: 5,   windowSec: 600,  key: r => `mfa:${r.ip}:${r.userId ?? ""}` },
  "search":        { limit: 60,  windowSec: 60,   key: r => `search:${r.ip}` },
  "api":           { limit: 600, windowSec: 60,   key: r => `api:${r.userId ?? r.ip}` },
  "webhook":       { limit: 120, windowSec: 60,   key: r => `wh:${r.ip}` },
  "admin":         { limit: 300, windowSec: 60,   key: r => `admin:${r.userId ?? r.ip}` },
};
```

Reponse en cas de depassement : **HTTP 429** avec `Retry-After: <secondes>` et en-tetes `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset`. Le corps ne revele pas si l'e-mail existe. Escalade anti-bruteforce : au-dela de 3 echecs sur login, CAPTCHA ou delai progressif (1 s, 5 s, 30 s) ; l'echec est journalise avec IP, e-mail hache et user-agent (§6.6).

---

### 5. Securite des paiements

#### 5.1 Regle « aucune donnee de carte chez Fanours » et son application

Le CDC est sans ambiguite (§10.3, §19.4, §26.5) : composants heberges/tokenises, aucun PAN, CVV ou donnee de carte dans les logs, la base ou l'analytics.

| Surface | Regle | Verification |
|---|---|---|
| Front (`checkout`) | Stripe Elements / Checkout heberge : le numero de carte ne transite **jamais** par le JS de Fanours ; le navigateur recoit un `client_secret` / `paymentMethodId` | Revue de code : grep `cardNumber\|cvc\|pan` dans le bundle client -> 0 resultat hors lib tierce ; test Playwright qui intercepte les requetes reseau du checkout -> aucun corps porte un numero de carte |
| Serveur | `POST /api/payments` ne recoit que `{ orderId, paymentMethodId }` ; le schema `.strict()` **rejette** tout champ non declare | `payments/no-card-data.spec.ts` : body `{ cardNumber: "4242..." }` -> 400, aucun log, aucune ecriture |
| Base | Aucune colonne de type carte : grep `card_number\|cvv\|cvc\|pan` sur les migrations -> 0 | Job CI `schema-card-guard` |
| Logs / APM / analytics | Redaction automatique (§12.2) : champs `card*`, `cvc`, `pan`, `number` -> `[REDACTED]` avant ecriture, appliquee au logger central | `logs/redaction.spec.ts` : un evenement contenant `cardNumber` sort `[REDACTED]` |
| Erreurs | Erreurs du prestataire mappees en codes stables localises (§21.1) ; le message brut n'est jamais renvoye ni journalise | Test : la creation de PaymentIntent echouee expose `payment_initiation_failed`, pas le payload du prestataire |

**Reduction du scope PCI DSS** : l'integration hebergee/tokenisee maintient Fanours hors du scope de gestion des donnees de carte ; le questionnaire **PCI SAQ A** est applicable. Condition de maintien : ne jamais activer un SDK mobile/JS qui gere le numero de carte chez Fanours, et re-evaluer si un moyen de paiement non heberge (virement integre, mobile money) est ajoute.

#### 5.2 Separation commande de paiement / confirmation finale, et machine a etats

Le CDC exige la separation « entre ordre de paiement et confirmation finale » (§19.4) et qu'aucun statut « paye » ne vienne d'une redirection navigateur (§26.3).

```
Commande:  draft -> pending_payment -> payment_processing -> paid -> confirmed -> ...
                                      |                            |
                                      v                            v
                               payment_failed        (refund_pending -> partially_refunded -> refunded)
```

```ts
// src/payments/state-machine.ts
export type PaymentState =
  | "created" | "processing" | "requires_action" | "succeeded"
  | "failed" | "expired" | "refunded" | "partially_refunded" | "disputed";

/** Transitions autorisees ; toute autre est refusee (§9.3 « les transitions autorisees sont codifiees »). */
const ALLOWED: Record<PaymentState, readonly PaymentState[]> = {
  created:           ["processing", "requires_action", "failed", "expired"],
  processing:        ["succeeded", "failed", "requires_action", "expired"],
  requires_action:   ["processing", "succeeded", "failed", "expired"],
  succeeded:         ["refunded", "partially_refunded", "disputed"],
  failed:            ["processing", "expired"],          // un nouvel essai est possible
  expired:           [],
  refunded:          [],
  partially_refunded:["refunded", "partially_refunded"],
  disputed:          ["refunded", "partially_refunded"],
};

export function assertTransition(from: PaymentState, to: PaymentState): void {
  if (!ALLOWED[from].includes(to)) {
    throw new InvalidStateTransition(`payment_transition_not_allowed:${from}->${to}`);
  }
}
```

Regles strictes : la transition `→ succeeded` (et la transition de commande `→ paid`) n'est ecrite **que** par le consommateur de webhook (§5.3) — jamais par le retour navigateur, jamais par le back-office, jamais par une tache planifiee. Le back-office peut relancer le rapprochement et demander un refund via le workflow 4 yeux (§3.5), pas ecrire l'etat. Une commande `paid` n'est jamais supprimee (§9.3) ; un refund cree un evenement et incremente `refunded_amount` **sans ecraser** le montant d'origine (§10.4).

#### 5.3 Verification de webhook : HMAC, tolerance d'horodatage, anti-rejeu

```ts
// src/webhooks/verify.ts
import { createHmac, timingSafeEqual } from "node:crypto";
import { db } from "../db";
import { audit } from "../audit";

const TOLERANCE_SEC = 300; // 5 minutes

export type WebhookVerdict = { ok: true; eventId: string; provider: string }
                        | { ok: false; reason: string; status: number };

/**
 * Verification en 4 temps :
 *  1) signature calculee sur le CORPS BRUT (rawBody), jamais sur le JSON reparse ;
 *  2) HMAC-SHA256 sur `${timestamp}.${rawBody}`, secret du gestionnaire de secrets ;
 *  3) comparaison a temps constant ;
 *  4) rejet si |now - timestamp| > 5 min ET si l'event_id a deja ete traite.
 */
export async function verifyWebhook(args: {
  provider: string; signature: string; timestamp: string; rawBody: Buffer; secret: string;
}): Promise<WebhookVerdict> {
  const ts = Number(args.timestamp);
  if (!Number.isFinite(ts)) return { ok: false, reason: "bad_timestamp", status: 401 };
  const skew = Math.abs(Math.floor(Date.now() / 1000) - ts);
  if (skew > TOLERANCE_SEC) {
    await audit.write({ actorId: "system", action: "webhook.rejected.timestamp",
      objectType: "webhook_event", objectId: args.provider, result: "failure",
      metadata: { skew, provider: args.provider } });
    return { ok: false, reason: "timestamp_out_of_tolerance", status: 401 };
  }

  const expected = createHmac("sha256", args.secret)
    .update(`${args.timestamp}.`).update(args.rawBody).digest("hex");
  const a = Buffer.from(expected, "hex");
  const b = Buffer.from(args.signature.replace(/^sha256=/, ""), "hex");
  if (a.length !== b.length || !timingSafeEqual(a, b)) {
    await audit.write({ actorId: "system", action: "webhook.rejected.signature",
      objectType: "webhook_event", objectId: args.provider, result: "failure" });
    return { ok: false, reason: "invalid_signature", status: 401 };
  }

  // Anti-rejeu : l'identifiant d'evenement du fournisseur est unique par evenement
  const eventId = extractEventId(args.rawBody);
  const existing = await db.webhookEvent.findUnique({
    where: { provider_event: { provider: args.provider, event_id: eventId } },
  });
  if (existing) {
    await audit.write({ actorId: "system", action: "webhook.duplicate",
      objectType: "webhook_event", objectId: eventId, result: "ignored" });
    return { ok: false, reason: "duplicate_event", status: 409 };
  }
  return { ok: true, eventId, provider: args.provider };
}
```

Le middleware qui capture le **corps brut** est imperatif : `express.raw({ type: "application/json" })` sur `/api/webhooks/*`, avant tout middleware JSON. La verification est faite avant toute lecture du payload metier, avant tout acces base, avant tout log du contenu.

```sql
-- Deduplication et tracabilite des webhooks
CREATE TABLE webhook_event (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  provider       text        NOT NULL,
  event_id       text        NOT NULL,
  event_type     text        NOT NULL,
  received_at    timestamptz NOT NULL DEFAULT now(),
  signature_ok   boolean     NOT NULL,
  processed_at   timestamptz,
  outcome        text,                 -- applied | ignored_duplicate | rejected | failed
  payload_redacted jsonb,              -- payload stocke avec redaction (§6.6)
  correlation_id uuid        NOT NULL,
  UNIQUE (provider, event_id)
);
```

Traitement idempotent : le consumer ecrit d'abord l'evenement (`INSERT ... ON CONFLICT DO NOTHING`) et n'applique les effets que si l'insertion a reussi. Tous les effets (facture, stock, notification, evenement metier) sont dans une transaction : un evenement rejoue ne duplique rien (§9.5, §26.3, §28.1).

#### 5.4 Idempotence : cle metier et contrainte unique

1. **Cle d'idempotence cote client** : `Idempotency-Key` (UUID) sur `POST /api/orders` et `POST /api/payments`. Le serveur la stocke avec le hash de la requete ; requete identique + meme cle -> reponse d'origine rejouee (200, pas de double creation) ; requete **differente** + meme cle -> 409 `idempotency_key_reuse`.
2. **Contrainte unique en base** : `UNIQUE (provider, provider_reference)` sur `payment_attempt`, `UNIQUE (customer_id, idempotency_key)` sur `order` — la garantie ne depend pas du code applicatif (§9.5, §20.2).
3. **Verrou transactionnel a la confirmation** : `SELECT ... FOR UPDATE` sur la commande avant `paid`, et `UPDATE stock SET qty = qty - $1 WHERE sku = $2 AND qty >= $1` verifie par `rowCount` : la concurrence ne peut pas faire passer le stock sous zero (§7.3, §26.2).

```sql
ALTER TABLE order ADD COLUMN idempotency_key uuid;
CREATE UNIQUE INDEX uq_order_idem ON order (customer_id, idempotency_key)
  WHERE idempotency_key IS NOT NULL;
CREATE UNIQUE INDEX uq_payment_provider_ref ON payment_attempt (provider, provider_reference);
```

#### 5.5 Rapprochement (§10.5, §16.7)

Rapprochement **quotidien** automatique : commandes payees ↔ transactions prestataire ↔ versements bancaires, par devise et par moyen. Tout ecart est cree comme **tache** (montant, devise, commission, commande orpheline, versement manquant) avec statut `open -> investigating -> resolved | written_off`, owner, SLA et historique (§10.5). Le rapprochement ne **corrige jamais** silencieusement : une correction manuelle est un evenement trace et audite, jamais un `UPDATE` du chiffre. Les exports sont limites aux roles finances (§3.2) et journalises (§6.6). Le retraitement quotidien est idempotent : relancer le job du jour ne duplique ni ecart ni notification.

#### 5.6 Detection de fraude et prevention du double remboursement

Heuristiques minimales, rejouables, explicables, journalisees :
- Incoherence montant / devise / commande (§19.4) -> blocage + alerte.
- Vitesse : N creations de commande ou N tentatives de paiement en fenetre courte depuis la meme IP ou le meme appareil -> 3DS force ou challenge.
- Geographie : pays de livraison different du pays de la carte, pays en liste de risque (maintenue par le `compliance_manager`, avec source et date, §11.6), premier achat a forte valeur.
- Velocite de session : panier puis paiement en moins de 5 s, ou abandon puis paiement repete sur la meme carte.
- Regle metier : MOQ depasse, B2B actif sans verification (§8.2) -> demande de verification plutot que paiement.

3-D Secure quand elle est applicable (§10.2) : le prestataire decide, Fanours ne contourne jamais un `requires_action`.

**Regle du double remboursement** (§16.7) :
- Un refund porte une **cle d'idempotence** dediee derivee de `(orderId, providerRefundId)` et une contrainte unique `UNIQUE (provider, provider_refund_id)`.
- Le workflow 4 yeux (§3.5) est requis : la demande porte le montant ; l'execution re-verifie `sum(refunds) + montant <= montant_paye_original`.
- Un refund n'est jamais marque `refunded` sur le seul retour du back-office : l'etat suit le **statut reel du prestataire** (§10.4), mis a jour par webhook.
- Un refund partiel puis un complement est possible ; le montant d'origine n'est jamais ecrase, `refunded_amount` est incremente, l'ecart est verifie en base (contrainte `CHECK`).
- Deux refunds partiels simultanes sont impossibles : verrou `FOR UPDATE` sur la commande + recalcul dans la meme transaction.

```sql
ALTER TABLE payment_attempt
  ADD CONSTRAINT ck_refund_not_exceeding_paid
  CHECK (refunded_amount_cents >= 0 AND refunded_amount_cents <= amount_cents);
```

#### 5.7 Chargebacks et production de preuves (§10.3)

Un `disputed` est un etat a part entiere : tache support avec SLA, alerte `compliance_manager` + `finance_manager`, dossier de preuve (commande, snapshot produits/prix, justificatif de livraison avec tracking et date signee par le transporteur, correspondance client, regle douaniere, documents d'expedition §11.8) assemble en PDF, hache et rattache a l'evenement d'audit. Un litige n'entraine **jamais** la modification ou la suppression d'une commande (§9.3) : la piece est ajoutee, pas remplacee. Le volume de litiges par prestataire est suivi au tableau de bord finance (§16.7) : au-dela d'un seuil, la cause racine (produit, description, delai de livraison) est traitee, pas seulement le litige.

---

### 6. Donnees, secrets et chiffrement

#### 6.1 Inventaire des secrets

Un secret = toute valeur qui donne un acces ou un pouvoir (cle API, secret de webhook, mot de passe de base, cle de chiffrement, secret OAuth, jeton interne). Regle : **aucune valeur en clair dans le depot, le build, les logs ou le bundle public** (§19.5, §26.5).

| Secret | Emplacement | Rotation | Consommateurs |
|---|---|---|---|
| `DATABASE_URL` | Gestionnaire de secrets | 90 j | Backend, worker, migrations |
| `REDIS_URL` | Gestionnaire de secrets | 90 j | Backend, worker |
| `SESSION_SECRET` (pepper sessions/nonces) | Gestionnaire de secrets | 90 j | Backend |
| `JWT_SIGNING_KEY` (si JWT de service) | KMS / gestionnaire | 90 j | Backend ↔ worker |
| `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` | Gestionnaire de secrets | 90 j / 180 j | Backend, worker |
| `STRIPE_PUBLISHABLE_KEY` | Build, variable publique | 90 j | Front (public par nature) |
| `OAUTH_GOOGLE_CLIENT_SECRET` | Gestionnaire de secrets | 180 j | Backend |
| `EMAIL_API_KEY` | Gestionnaire de secrets | 180 j | Worker e-mail |
| `CARRIER_API_KEY` (par transporteur) | Gestionnaire de secrets | 180 j | Worker logistique |
| `KMS_MASTER_KEY` (enveloppe) | KMS, non exportable si possible | 180 j + versionnage | Backend (documents, TOTP) |
| `DATABASE_ENCRYPTION_KEY` | KMS | 90 j | Backend |
| `WEBHOOK_SIGNING_SECRETS` (par provider) | Gestionnaire de secrets | 180 j | Worker |
| `SENTRY_DSN` | Build (public par nature) | — | Observabilite |

Nommage par environnement (§25.1) : prefixe d'environnement au deploiement (`FANOURS_PROD_DATABASE_URL`) ; les environnements sont separes et une valeur de staging n'est jamais acceptee en production — le demarrage echoue si une variable d'un autre environnement est detectee (§12.2).

#### 6.2 Detection de secrets dans le code et la CI

```yaml
# .github/workflows/security.yml (extrait) — §24.6, §25.2
name: security
on: [push, pull_request]
jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }          # historique complet : un secret supprime reste vole
      - name: Detect-secrets
        run: pipx run detect-secrets --baseline .secrets.baseline --all-files
      - name: Gitleaks
        run: gitleaks detect --source . --redact --no-banner --exit-code 1
      - name: TruffleHog (historique)
        run: docker run --rm -v "$PWD:/repo" trufflesecurity/trufflehog:latest git file:///repo --fail
      - name: Bandit
        run: bandit -r src -ll
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: semgrep ci --config p/owasp-top-ten --config p/security-audit --error
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm audit --audit-level=high
      - run: npx osv-scanner --lockfile=package-lock.json
      - run: trivy fs --severity HIGH,CRITICAL --exit-code 1 .
```

Le meme detecteur tourne en **pre-commit** (hook `detect-secrets`) et en CI. Un secret detecte dans l'historique est traite comme un incident : rotation immediate, purge de l'historique si possible, notification (§12.4).

#### 6.3 Chiffrement en transit et au repos

- **En transit** : TLS 1.3 impose (1.2 minimum pour les clients legacy documente), HSTS `max-age=31536000; includeSubDomains; preload` (§19.3). Appels internes en mTLS ou reseau prive chiffre ; redirections HTTP→HTTPS strictes ; aucune connexion en clair vers la base, le cache ou le stockage objet.
- **Au repos** : base managee avec chiffrement de disque et de volume ; sauvegardes chiffrees independamment (§19.8) ; stockage objet en SSE-KMS ; volumes de cache et fichiers temporaires chiffres ou jetables.
- **Transports tiers** : prestataire de paiement, transporteur et fournisseur d'e-mail en TLS verifie (jamais `verify=false` ni `rejectUnauthorized: false`).

#### 6.4 Chiffrement au niveau colonne et patron d'enveloppe

Cible (§8.1, §13.3) : documents de verification d'entreprise, secrets TOTP, donnees personnelles a protection renforcee. Patron : une **DEK** chiffre la donnee (AES-256-GCM) ; la DEK est elle-meme chiffree par une **KEK** detenue par le KMS. On ne stocke que ciphertext, nonce, tag et version de la KEK.

```ts
// src/crypto/envelope.ts
import { createCipheriv, createDecipheriv, randomBytes } from "node:crypto";
import { kms } from "../infra/kms";

type Envelope = { v: number; kid: string; alg: "A256GCM"; iv: string; ct: string; tag: string; wrapped: string };

/** Chiffrement d'enveloppe : DEK par donnee, chiffree par la KEK du KMS. */
export async function seal(plaintext: Buffer, aad: string): Promise<Envelope> {
  const kek = await kms.getKey(process.env.KMS_MASTER_KEY!); // jamais exportee si possible
  const dek = Buffer.from(await kms.generateDataKey(kek, "AES_256"));
  const iv = randomBytes(12);
  const cipher = createCipheriv("aes-256-gcm", dek.plaintext, iv);
  cipher.setAAD(Buffer.from(aad, "utf8")); // lie le ciphertext a sa ligne/table
  const ct = Buffer.concat([cipher.update(plaintext), cipher.final()]);
  const tag = cipher.getAuthTag();
  const wrapped = await kms.encrypt(kek, dek.plaintext);
  dek.plaintext.fill(0); // la DEK en clair est immediatement ecrasee en memoire
  return { v: 1, kid: kek.id, alg: "A256GCM", iv: iv.toString("base64"),
           ct: ct.toString("base64"), tag: tag.toString("base64"),
           wrapped: wrapped.toString("base64") } as Envelope;
}

export async function open(env: Envelope, aad: string): Promise<Buffer> {
  const kek = await kms.getKey(env.kid);
  const dek = await kms.decrypt(kek, Buffer.from(env.wrapped, "base64"));
  const decipher = createDecipheriv("aes-256-gcm", dek, Buffer.from(env.iv, "base64"));
  decipher.setAAD(Buffer.from(aad, "utf8"));
  decipher.setAuthTag(Buffer.from(env.tag, "base64"));
  return Buffer.concat([decipher.update(Buffer.from(env.ct, "base64")), decipher.final()]);
}
```

Rotation de cle (§19.5) : chaque ciphertext porte `kid` (version de la KEK), donc la rotation est **incrementale et sans downtime** — nouvelle version, dechiffrement + rechiffrement en job de fond par lots (reprise possible), bascule de la version d'ecriture, conservation de l'ancienne en lecture seule pendant la retention. Rotation d'urgence (compromission suspectee) : revocation de la version, re-chiffrement immediat, invalidation des sessions et des secrets derives (§12.4). Le `aad` lie le ciphertext a sa table et a sa ligne : un ciphertext de document de verification ne peut pas etre deplace vers une autre ligne ou table sans echec de l'AAD.

#### 6.5 Minimisation des donnees

Le CDC impose de ne collecter que le necessaire (§19.1) et d'evaluer la proportionnalite des justificatifs B2B (§8.1).

| Donnee | Regle |
|---|---|
| Numero de carte, CVC, code PIN bancaire | Jamais (§10.3, §19.4) |
| Mot de passe en clair | Jamais — Argon2id uniquement, jamais journalise |
| Documents d'identite non necessaires | Non collectes ; si un justificatif est exige, une seule piece, avec date de suppression |
| Coordonnees bancaires completes (IBAN + titulaire) | IBAN chiffre au niveau colonne, masque ailleurs (`****1234`), acces restreint a `finance_manager` |
| Numero de telephone si l'e-mail suffit | Optionnel ; pas de collecte par defaut |
| Geolocalisation precise | Jamais ; uniquement le pays (derniere adresse IP, stockee hachee) |
| Adresse IP complete en base | Version tronquee (IPv4 /24, IPv6 /48) ; la version complete vit 7 j max dans les logs techniques |
| Historique de navigation, cookies non necessaires | Refus par defaut ; consentement requis (§19.6) |

Regle de retention : chaque champ a une duree de vie declaree ; suppression automatique par job (soft-delete puis purge) ; documents transactionnels conserves selon les obligations legales (§20.2) ; documents de verification B2B supprimes a la fin de la relation + delai legal.

#### 6.6 Journal d'audit immuable

Le CDC fixe le contenu : acteur, action, objet, horodatage, resultat et **ID de correlation**, sans mot de passe, token, donnee de carte ni contenu personnel superflu ; les journaux ne sont pas modifiables par les roles metier ordinaires (§19.7).

```sql
-- Journal d'audit append-only, avec chaine de hachage
CREATE TABLE audit_log (
  id            bigserial PRIMARY KEY,
  occurred_at   timestamptz NOT NULL DEFAULT now(),
  actor_id      uuid,                       -- NULL = systeme
  actor_role    text,
  action        text        NOT NULL,       -- ex: order.refund.requested
  object_type   text        NOT NULL,
  object_id     text        NOT NULL,
  result        text        NOT NULL CHECK (result IN ('success','failure','denied','ignored')),
  correlation_id uuid       NOT NULL,
  ip_hash       text,
  metadata      jsonb       NOT NULL DEFAULT '{}'::jsonb,
  prev_hash     text        NOT NULL,
  entry_hash    text        NOT NULL UNIQUE
);
CREATE INDEX ix_audit_object ON audit_log (object_type, object_id, occurred_at DESC);
CREATE INDEX ix_audit_actor  ON audit_log (actor_id, occurred_at DESC);
CREATE INDEX ix_audit_corr   ON audit_log (correlation_id);
CREATE INDEX ix_audit_action ON audit_log (action, occurred_at DESC);

-- Chaine de hachage : chaque entree scelle la precedente (tamper-evident)
CREATE OR REPLACE FUNCTION audit_chain() RETURNS trigger AS $$
DECLARE last_hash text;
BEGIN
  IF TG_OP = 'INSERT' THEN
    SELECT entry_hash INTO last_hash FROM audit_log ORDER BY id DESC LIMIT 1 FOR UPDATE;
    NEW.prev_hash := COALESCE(last_hash, 'GENESIS');
    NEW.entry_hash := encode(sha256(
      NEW.prev_hash || '|' || NEW.occurred_at || '|' || COALESCE(NEW.actor_id::text,'') || '|' ||
      NEW.action || '|' || NEW.object_type || '|' || NEW.object_id || '|' || NEW.result || '|' ||
      NEW.correlation_id::text || '|' || NEW.metadata::text
    ), 'hex');
    RETURN NEW;
  END IF;
  RAISE EXCEPTION 'audit_log is append-only (attempted %)', TG_OP;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_chain BEFORE INSERT OR UPDATE OR DELETE ON audit_log
  FOR EACH ROW EXECUTE FUNCTION audit_chain();
```

Le declencheur **refuse** tout `UPDATE` et tout `DELETE`, y compris depuis un superutilisateur applicatif : seule l'insertion est possible. La chaine de hachage rend detectable toute insertion falsifiee (recalcul de la chaine). Complement : export quotidien de l'audit vers un stockage **en lecture seule** (object lock WORM), retention 7 ans, et revue mensuelle de l'integrite (recalcul de la chaine) par un role distinct de celui qui ecrit.

Redaction : la fonction d'ecriture applique une liste blanche de metadonnees et masque les cles sensibles (`token`, `password`, `secret`, `cookie`, `authorization`, `pan`, `cvv`, `card`, `email`, `cpf`, `iban`) a toute profondeur (§12.2) avant persistance.

**Evenements minimum a journaliser** (§19.7) : connexion admin, echec de MFA, creation/modification produit, changement de prix, changement de regle douaniere, modification de stock, decision B2B, remboursement, annulation, changement de role, export de donnees, modification de configuration sensible, exports financiers, acces aux documents de verification. Un test de couverture verifie qu'ils sont tous produits par un scenario E2E (§13.5).

---

### 7. Validation, XSS, injections, CSRF, CORS et en-tetes HTTP

#### 7.1 Validation des entrees et des sorties

Le CDC impose « validation stricte des entrees cote serveur » (§19.3) et « validation de schemas entree/sortie » (§21.1). Regle : **tout** ce qui entre dans le systeme passe par un schema `zod` ; **tout** ce qui sort vers le client passe par un schema de sortie. Le `.strict()` est obligatoire sur les schemas d'entree : un champ non declare est une erreur 400, pas un champ ignore silencieusement.

```ts
// src/http/validation.ts
import { z } from "zod";
import type { NextRequest } from "next/server";

/** Schemas d'entree : stricts, bornes, sans champ superflu. */
export const CreateOrderSchema = z.object({
  items: z.array(z.object({
    sku: z.string().regex(/^[A-Z0-9-]{3,32}$/),
    quantity: z.number().int().positive().max(10000),
  })).min(1).max(200),
  shippingAddressId: z.string().uuid(),
  billingAddressId: z.string().uuid().optional(),
  promoCode: z.string().regex(/^[A-Z0-9-]{3,24}$/).optional(),
  acceptTerms: z.literal(true),
}).strict(); // customerId, role, price, discount : interdits -> 400

export const UpdatePriceTierSchema = z.object({
  productId: z.string().uuid(),
  minQuantity: z.number().int().positive(),
  unitPrice: z.string().regex(/^\d{1,10}(\.\d{1,2})?$/), // Decimal en chaine, jamais un float
  currency: z.enum(["USD", "EUR", "GBP", "CNY"]),
  effectiveFrom: z.string().datetime({ offset: true }),
  reason: z.string().min(10).max(500),
}).strict();

/** Validation des sorties : le client ne recoit jamais un champ interne. */
export const PublicProductSchema = z.object({
  id: z.string().uuid(),
  sku: z.string(),
  name: z.string().min(1).max(200),
  price: z.object({ amount: z.string(), currency: z.string().length(3) }),
  tiers: z.array(z.object({ minQuantity: z.number().int(), unitPrice: z.string() })),
  moq: z.number().int().positive(),
  available: z.boolean(),
}).strict();

export function parseBody<T extends z.ZodTypeAny>(schema: T, body: unknown): z.infer<T> {
  const result = schema.safeParse(body);
  if (!result.success) {
    // 400 avec codes de champs stables ; jamais le detail interne zod brut
    throw new ValidationError(result.error.flatten().fieldErrors);
  }
  return result.data;
}
```

Regles de validation par type de donnee :
- **Montants** : `Decimal` en chaine, jamais `float` (les flottants perdent la precision sur les centimes et creent des ecarts de rapprochement, §10.5, §20.2).
- **Devises** : code ISO 4217 en `char(3)`, uppercase ; pas de nom libre ; pas de symbole.
- **Dates** : ISO 8601 avec fuseau, stockees en `timestamptz` ; les dates de regles ont une date d'effet (§11.2).
- **Langues** : enum `fr | en | zh-CN` (§4.1).
- **E-mails** : normalise (minuscules, `trim`) avant stockage et avant recherche — la cle d'unicite est l'e-mail normalise (§20.2).
- **Quantites** : entier positif, borne superieure metier (MOQ, multiples, §7.2).
- **Pays** : reference du referentiel §11.6 (pas de chaine libre).
- **Tailles de requete** : corps JSON limite a 64 Ko, nombre de parametres de requete limite, longueur de chaine limitee (§21.1).

Les schemas de sortie s'appliquent egalement aux **webhooks entrants** : le payload du prestataire est valide par schema avant tout traitement metier, sinon l'evenement est rejete et journalise (§5.3).

#### 7.2 XSS

Le CDC exige echappement et sanitation (§19.3). Application :
- React echappe par defaut : tout texte rendu via `{valeur}` est sur. `dangerouslySetInnerHTML` est **interdit** par lint dans le code applicatif ; la seule exception est un composant unique et dedie, dont le HTML a ete sanitise **cote serveur** a l'ecriture (allowlist DOMPurify : `p, br, strong, em, ul, ol, li, a[href], span, table...` ; interdits : `script`, `style`, `iframe`, `object`, `embed`, `form`, `input`, `on*`, `javascript:`).
- Les URL fournies par l'utilisateur (lien de site web de l'entreprise, avatar, redirection post-login) sont validees : schema `z.string().url()` **et** verification du protocole (`https:` ou `http:` uniquement ; `javascript:`, `data:`, `vbscript:` rejetes) et, si le lien est rendu dans un `href`, `rel="noopener noreferrer"` + `target="_blank"`.
- Les styles dynamiques (attribut `style`) sont construits a partir d'une **allowlist de proprietes**, jamais depuis une chaine libre ; le CSS est genere a la compilation (tokens §3.3), pas a l'execution.
- La generation de PDF (factures, documents douaniers) utilise un moteur qui **echappe** le texte ; aucun HTML brut n'est injecte dans le template de generation.
- Le CMS (§16.10) est la source de HTML riche : le sanitiseur est applique a l'ecriture **et** au rendu (double barriere).

```ts
// src/content/sanitize.ts
import DOMPurify from "isomorphic-dompurify";

const ALLOWED = {
  ALLOWED_TAGS: ["p","br","strong","em","ul","ol","li","span","a","h2","h3","h4","table","thead","tbody","tr","th","td"],
  ALLOWED_ATTR: ["href","title","rel","target"],
  ALLOWED_URI_REGEXP: /^(?:https?|mailto):/i,
};
export const sanitizeHtml = (dirty: string): string => DOMPurify.sanitize(dirty, ALLOWED);
```

#### 7.3 Injection SQL

- **Uniquement** des requetes parametrees via Drizzle : pas de concatenation, pas de template literal dans du SQL, pas de `sql.raw()` ni de `sql`` ` construit a partir d'une entree utilisateur. La regle est verifiee par lint (`eslint-plugin-security` / regle custom interdisant `sql.raw` et l'usage de `sql` hors de valeurs liees) et par un test qui echoue si une chaine de requete contient un fragment de payload.
- Les champs de tri et de filtre dynamiques passent par une **allowlist** de colonnes : `sort=createdAt` -> `["created_at"]` ; un nom inconnu est rejete, jamais injecte.
- La recherche plein texte (Postgres `tsvector`/`pg_trgm`, ou moteur dedie) utilise les parametres lies ; la syntaxe de recherche est un AST, pas une chaine concatenee.
- Les migrations sont versionnees et relues (§25.3) ; aucune migration destructive sans preuve de restauration (§25.3).

#### 7.4 SSRF

Le CDC ne mentionne pas explicitement le SSRF, mais §21.4 impose des integrations externes documentees et §19.5 une separation des secrets : les appels sortants sont une surface d'attaque. Regle : **aucune URL fournie par un utilisateur n'est jamais fetchee**.

```ts
// src/http/ssrf-guard.ts
import { lookup } from "node:dns/promises";
import { isIP } from "node:net";

const ALLOWED_HOSTS = new Set([
  "api.stripe.com", "api.transporteur-partenaire.com", "api.email-provider.com",
]); // allowlist stricte, versionnee dans la configuration

const BLOCKED_V4 = [/^127\./, /^10\./, /^192\.168\./, /^169\.254\./, /^172\.(1[6-9]|2\d|3[01])\./, /^0\./, /^100\.(6[4-9]|[7-9]\d|1[01]\d|12[0-7])\./];
const BLOCKED_V6 = [/^::1$/, /^fc00:/i, /^fe80:/i, /^::ffff:/i];

/**
 * Resolution DNS puis verification de l'IP reellement contactee.
 * La resolution est refaite au moment de la connexion pour bloquer le DNS rebinding :
 * le nom peut pointer sur une IP publique a l'inspection, puis sur 169.254.169.254 au moment du socket.
 */
export async function assertOutboundUrlAllowed(rawUrl: string): Promise<URL> {
  const url = new URL(rawUrl);
  if (url.protocol !== "https:") throw new SSRFError("https_only");
  if (!ALLOWED_HOSTS.has(url.hostname)) throw new SSRFError("host_not_allowlisted");

  const addrs = await lookup(url.hostname, { all: true });
  for (const { address } of addrs) {
    const v4 = isIP(address) === 4;
    if (v4 && BLOCKED_V4.some((re) => re.test(address))) throw new SSRFError("private_address");
    if (!v4 && BLOCKED_V6.some((re) => re.test(address))) throw new SSRFError("private_address");
  }
  return url;
}

/** A appeler immediatement avant la connexion : re-resoudre et comparer. */
export function assertPinnedIp(url: URL, pinnedIp: string): void {
  const v4 = isIP(pinnedIp) === 4;
  if (v4 && BLOCKED_V4.some((re) => re.test(pinnedIp))) throw new SSRFError("private_address");
  if (!v4 && BLOCKED_V6.some((re) => re.test(pinnedIp))) throw new SSRFError("private_address");
}
```

Regles complementaires : timeout de connexion et de reponse (5 s / 15 s), taille de reponse max (5 Mo), nombre de redirections limite a 2 **et** revalide (allowlist reappliquee a chaque saut), pas de proxy d'environnement en production (`HTTP_PROXY`/`HTTPS_PROXY` desactive), pas de resolution DNS arbitraire depuis l'input, et separateur : les appels a l'API transporteur passent par un **service interne** qui possede l'allowlist, jamais par le code qui recoit des saisies utilisateur.

#### 7.5 CSRF

Le CDC impose « protection CSRF adaptee aux mecanismes d'authentification » (§13.4, §19.3). Modele retenu : **cookie de session + double soumission + verification `Origin`** (le cookie `SameSite=Lax` est une defense, pas la defense).

```ts
// src/http/csrf.ts
import { randomBytes, timingSafeEqual } from "node:crypto";
import { db } from "../db";

const SAFE_METHODS = new Set(["GET", "HEAD", "OPTIONS"]);
const MUTATING = new Set(["POST", "PUT", "PATCH", "DELETE"]);

/**
 * Double soumission : un jeton non-secret dans un cookie lisible par le JS,
 * identique dans l'en-tete X-CSRF-Token. Un attaquant cross-site peut declencher
 * la requete mais ne peut pas LIRE le cookie : il ne peut donc pas produire l'en-tete.
 */
export async function assertCsrf(req: Request, res: Response): Promise<void> {
  if (SAFE_METHODS.has(req.method)) return;
  const cookieToken = readCookie(req, "fanours_csrf");
  const headerToken = req.headers.get("x-csrf-token");
  const origin = req.headers.get("origin");

  if (!cookieToken || !headerToken) throw new CsrfError("csrf_token_missing");
  if (!timingSafeEqual(Buffer.from(cookieToken), Buffer.from(headerToken))) {
    throw new CsrfError("csrf_token_mismatch");
  }
  // Verification Origin/Referer en plus, meme si le jeton est present
  if (origin && !ALLOWED_ORIGINS.has(origin)) throw new CsrfError("origin_not_allowed");
}
```

Le jeton CSRF est lie a la session (HMAC de l'ID de session) : un jeton d'une session ne vaut pas pour une autre. Le cookie CSRF n'est **pas** `HttpOnly` (il doit etre lisible par le JS pour l'envoyer en en-tete) mais ne porte aucune donnee sensible et n'est jamais rejoue seul (l'en-tete est indispensable).

#### 7.6 CORS

Regle stricte : **jamais** `Access-Control-Allow-Origin: *` sur un endpoint qui accepte des cookies ou un en-tete `Authorization` (§19.3, RULE CORS du standard). Allowlist explicite, derivee des origines de l'application.

```ts
// src/http/cors.ts
import cors from "cors";

const ALLOWED_ORIGINS = new Set([
  "https://www.fanours.com", "https://fanours.com", "https://app.fanours.com",
  ...(process.env.NODE_ENV !== "production" ? ["http://localhost:3000"] : []),
]);

export const corsOptions: cors.CorsOptions = {
  origin(origin, callback) {
    if (!origin) return callback(null, true);        // serveur a serveur, curl, mobile
    if (ALLOWED_ORIGINS.has(origin)) return callback(null, true);
    return callback(new Error("cors_origin_not_allowed"));
  },
  credentials: true,                                  // cookies de session
  methods: ["GET","POST","PUT","PATCH","DELETE","OPTIONS"],
  allowedHeaders: ["Content-Type","X-CSRF-Token","X-Request-ID","Idempotency-Key"],
  exposedHeaders: ["X-Request-ID","X-RateLimit-Remaining","X-RateLimit-Reset","Retry-After"],
  maxAge: 600,
};
```

Regles : `Access-Control-Allow-Credentials: true` **impose** une origine explicite (jamais `*`) ; le preflight est repondu sans toucher a la base ; une origine hors allowlist ne recoit aucun en-tete CORS (le navigateur bloque) et la requete ne doit pas avoir d'effet metier (pre-verification de l'origine sur les routes mutantes).

#### 7.7 CSP et en-tetes de securite

CSP stricte : pas de `unsafe-inline` sur `script-src` (nonce ou hash obligatoire), `connect-src` limite aux domaines d'integration, `object-src 'none'`, `base-uri 'none'`, `frame-ancestors 'none'`.

```
Content-Security-Policy: default-src 'self';
  script-src 'self' 'nonce-{RANDOM}' 'strict-dynamic' https://js.stripe.com https://www.googletagmanager.com;
  style-src 'self' 'nonce-{RANDOM}' https://fonts.googleapis.com https://fonts.gstatic.com;
  img-src 'self' data: blob: https://*.cdn.example.com;
  font-src 'self' data: https://fonts.gstatic.com;
  connect-src 'self' https://api.stripe.com https://api.transporteur-partenaire.com;
  frame-src https://js.stripe.com https://hooks.stripe.com;
  object-src 'none'; base-uri 'none'; form-action 'self'; frame-ancestors 'none';
  require-trusted-types-for 'script'; upgrade-insecure-requests
```

Les autres en-tetes (§19.3) :

| En-tete | Valeur | Motif |
|---|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` | Force HTTPS |
| `X-Content-Type-Options` | `nosniff` | Empeche l'interpretabilite de type MIME |
| `X-Frame-Options` | `DENY` | Defense clickjacking (en plus de `frame-ancestors`) |
| `Content-Security-Policy` | ci-dessus | XSS, injection de frames |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Evite la fuite d'URL porteuse d'identifiants |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=(), payment=(self)` | Minimise les API navigateur |
| `Cross-Origin-Opener-Policy` | `same-origin` | Isolement de fenetre |
| `Cross-Origin-Resource-Policy` | `same-origin` | Protection des ressources |
| `Cache-Control` (routes auth) | `no-store` | Pas de cache des pages authentifiees |
| `X-Permitted-Cross-Domain-Policies` | `none` | Ancien Flash, hygiene |
| `Server` / `X-Powered-By` | supprimes | Pas de fuite de version |

CSP en mode **report-only** d'abord, puis enforce avec rapport CSP recu par le pipeline d'observabilite ; toute violation CSP (meme en report-only) est une alerte avant le passage en enforce.

---

### 8. Televersements de fichiers

Le CDC pose les exigences (§19.3) : validation MIME reelle, taille maximale, noms surs, stockage separe, scan si possible. Application complete :

| Controle | Implementation |
|---|---|
| Type reel (magic bytes) | `file-type` / lecture des 4 premiers octets. L'extension et le `Content-Type` declare par le client ne sont **jamais** suffisants. Liste blanche : `image/jpeg`, `image/png`, `image/webp`, `application/pdf` pour les justificatifs B2B (§8.1) |
| Taille maximale | 10 Mo par document de verification, 5 Mo par image produit, limite globale par compte et par jour |
| Nombre de fichiers | 5 documents maximum par dossier de verification ; 20 images par produit (§7.4) |
| Renommage sur | Nom de stockage = UUIDv4 + extension deduite du type reel. Le nom d'origine est conserve dans un champ metadonnee (non utilise pour le stockage) : suppression du path traversal (`../../`), des null bytes, des caracteres de controle |
| Stockage hors racine web | Bucket objet **prive**, jamais servi par le serveur d'application ; CDN seulement pour les assets publics explicitement publies |
| Aucun execution | Les fichiers ne sont jamais executes ni servis avec un `Content-Disposition: inline` sur un type actif ; `Content-Disposition: attachment` + `X-Content-Type-Options: nosniff` ; les documents sont convertis en PDF/A non actif si possible ; les images sont re-encodees (suppression des metadonnees EXIF, des charges utiles) |
| Antivirus | ClamAV (daemon) sur le flux, avant promotion vers le bucket « sain » ; fichier infecte -> quarantaine + alerte + notification a l'operateur ; regle : **aucun fichier n'est diffuse avant d'etre scanne** (pipeline a 2 buckets : `quarantine/` -> `documents/`) |
| Chiffrement | Chiffrement au repos cote serveur (SSE-KMS) + chiffrement au niveau colonne du binaire (§6.4) pour les documents de verification |
| Acces | URL signee a duree de vie courte (5 min), scope a l'objet, lie a l'ID de session ; **aucune URL publique** ; aucune URL signee dans les logs (elles sont des capacites) |
| Journalisation | Chaque lecture d'un document de verification est journalisee (acteur, objet, raison, correlation_id) ; un audit des lectures detecte la collecte massive |
| Nettoyage | Job quotidien : supprime les uploads non valides (quarantaine > 24 h) et applique la retention (§6.5) |

```ts
// src/uploads/validate.ts
import { fileTypeFromBuffer } from "file-type";
import { randomUUID } from "node:crypto";

const ALLOWED = new Map([
  ["image/jpeg", ".jpg"], ["image/png", ".png"], ["image/webp", ".webp"],
  ["application/pdf", ".pdf"],
]);
const MAX_BYTES = 10 * 1024 * 1024;

export async function validateUpload(input: {
  buffer: Buffer; declaredMime: string; originalName: string; kind: "product_image" | "verification_document";
}) {
  if (input.buffer.length === 0) throw new UploadError("empty_file");
  if (input.buffer.length > MAX_BYTES) throw new UploadError("file_too_large");

  // 1. Type reel par magic bytes : l'extension et le Content-Type sont ignores
  const detected = await fileTypeFromBuffer(input.buffer);
  if (!detected || !ALLOWED.has(detected.mime)) throw new UploadError("mime_not_allowed");
  if (input.kind === "verification_document" && detected.mime !== "application/pdf") {
    throw new UploadError("document_type_not_allowed");
  }

  // 2. Nom sur : UUID + extension deduite du type reel
  const storageKey = `${input.kind}/${randomUUID()}${ALLOWED.get(detected.mime)!}`;

  // 3. Sanitation du nom d'origine conserve en metadonnee
  const safeOriginal = input.originalName.replace(/[\u0000-\u001f\u007f]/g, "").slice(0, 200);

  return { storageKey, mime: detected.mime, bytes: input.buffer.length, safeOriginal };
}
```

Politique specifique aux **documents de verification d'entreprise** (§8.1, §19.6) : chiffrement obligatoire, acces limite aux roles autorises (`compliance_manager`, `super_admin`, et `support_agent` uniquement pour repondre a un dossier en cours), suppression conforme a la politique de conservation, aucune reutilisation pour du marketing, et absence totale d'indexation : le bucket est prive, sans CDN, sans `robots.txt` implicite.

---

### 9. Limitation de debit, anti-abus et protection metier

Les limites exactes sont en §4.6. Ce paragraphe couvre la **strategie** et les protections metier specifiques.

**Mecanique.** Fenetre glissante Redis (ZSET) executee dans un script Lua atomique : le compteur est exact, partage entre toutes les instances, et sans course entre lecture et ecriture. Cles composites : `action + IP` **et** `action + utilisateur` quand l'utilisateur est connu. Le double comptage evite deux defauts opposes : un attaquant rotatif ne contourne pas la limite utilisateur, et un utilisateur derriere une IP d'entreprise (proxy, VPN, NAT) n'est pas bloque a cause des autres.

**Reponses.** Depassement → `429` + `Retry-After` (secondes) + `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset`. Le corps est identique a celui d'un email inexistant (§9 anti-enumeration). Un `429` ne doit jamais reveler si un compte existe.

**Protection anti-enumeration.** Pour `login`, `password-reset/request`, `register` et `mfa/verify` : meme statut HTTP (200 ou 202), meme corps, meme structure JSON, meme duree de traitement (le temps de reponse est normalise par un delai minimal), meme nombre de requetes de reseau sortantes. Concretement, la verification de mot de passe sur un compte inexistant execute un hachage factice (dummy hash) pour egaliser le temps.

```ts
// src/auth/enumeration-safe.ts
import argon2 from "argon2";
const DUMMY_HASH = "$argon2id$v=19$m=65536,t=3,p=1$..."; // hachage factice, cout equivalent

export async function verifyPasswordUniformly(storedHash: string | null, password: string): Promise<boolean> {
  const hash = storedHash ?? DUMMY_HASH;   // toujours un hachage : temps constant
  const ok = await argon2.verify(hash, password);
  return storedHash !== null && ok;
}
```

**Anti-bruteforce sur codes promo** : chaque code a un compteur d'echec ; au-dela de 5 essais par code et par IP sur 10 min, le code est mis en quarantaine temporaire et une alerte est envoyee. Les codes promo sont opaques (>= 8 caracteres aleatoires, pas de mot de passe devinable), a usage unique ou a quota, et leur tentative d'application est journalisee (§9.1).

**Anti-spam sur contact et demande de devis** : 5/h par IP et 3/jour par e-mail, plus un champ honeypot et une verification de temps de remplissage (formulaire rempli en moins de 2 s = bot). Les soumissions sont limitees a une file d'attente et ne declenchent pas d'email transactionnel immediat si le score de spam est eleve.

**Anti-abus moteur de recherche et IA** (§18.1, §18.3) : 60 requetes/min par IP, plus un quota quotidien par session ; les resultats IA sont mis en cache (TTL 15 min) ; le cout par session est plafonne ; les IP de serveurs connus (crawlers) sont traitees via robots.txt et un limites dediees, pas en contournant le budget.

**Abus de l'API publique** (§21.1) : pagination obligatoire, `limit` maximal, reponses de liste plafonnees, pas de recherche exhaustive ni de dump. Les cles API tiers ne sont jamais acceptees en query string (elles iraient dans les logs) : uniquement en en-tete `Authorization`.

**Protection du stock** (§7.3) : un barrage anti-bot cote acheteur (cookie + signature de session + debit) sur la creation de commande, plus la reservation transactionnelle : la reservation expire et le stock est libere (job de maintenance), et un acheteur ne peut pas detenir plus de N reservations simultanees.

**Protection de la messagerie** : le formulaire de contact et la demande de devis n'envoient jamais de piece jointe executable ; les liens de suivi des e-mails sont signes et a usage unique avec expiration, et ne transportent aucune donnee personnelle en clair dans l'URL.
---

### 10. Conformite reglementaire

Cadre CDC (§19.6) : politique de confidentialite localisee, consentements par juridiction, procedures DSAR, retention et suppression automatique, registre des prestataires et des transferts internationaux, plan de reponse aux incidents, et evaluation des obligations RGPD/UK GDPR, lois chinoises et regles des pays cibles **avec un conseil competent**. Cette section n'est pas cet avis : elle organise les controles techniques qui rendent l'avis tenable.

#### 10.1 RGPD et UK GDPR

| Exigence | Controle technique |
|---|---|
| Bases legales | Chaque traitement est declare dans le registre (finalite, base legale, categories, destinataires, duree) ; le service refuse de demarrer si un traitement actif n'a pas de base legale |
| Minimisation (§19.1) | Liste des champs non collectes (§6.5) + schema zod qui rend invalide tout champ non declare |
| Consentement | Bandeau avec refuser/accepter, granularite par finalite, aucun cookie non essentiel avant consentement (§19.6) ; les cookies de session, strictement necessaires, sont exemptes mais declares |
| Registre des traitements | Exportable, versionne, revu trimestriellement |
| DPA | Registre des sous-traitants (paiement, e-mail, transporteur, stockage, analytics) avec date, localisation, garanties ; la localisation du stockage est un parametre de configuration, pas une convention |
| Transferts hors UE/UK | Enregistrement systematique (pays, destination, garantie : SCC / adequation / derogation, analyse de risque). Regle technique : les donnees des clients UE/UK sont stockees en region UE/UK, sauf derogation documentee et approuvee |
| DSAR | Acces, rectification, export, suppression, limitation (§19.6) : export machine-lisible (JSON/CSV) sous 30 j, suppression ou anonymisation verifiable, tracabilite (qui, quoi, quand, delai) |
| Violation de donnees | Processus 72 h avec template, arbre de decision, responsables nommes, simulation annuelle ; correlation technique via `correlation_id` (§23.2) |
| Droits des personnes | Aucune decision automatisee ayant un effet juridique sans controle humain ; les recommandations IA (§18.3) sont informatives et ne profilent pas (§23.3) |

#### 10.2 PIPL (Chine)

Fanours est basee en Chine : les donnees de clients chinois relevent du PIPL.

- **Localisation** : donnees des residents chinois stockees et traitees en Chine ; aucun transfert hors de Chine sans mecanisme legal (evaluation de conformite, certification, contrat type) et sans information du client.
- **Consentement explicite et separe** : traitement sensible (identification, paiement, localisation) et transmission a des tiers.
- **Droits** : acces, rectification, suppression, portabilite, retrait du consentement, droit de ne pas faire l'objet d'une decision automatisee ; mecanismes identiques au DSAR, avec point de contact local.
- **Responsable de la protection des donnees** et, le cas echeant, representant en Chine.
- **Mesures techniques exigees** : chiffrement, controle d'acces, journalisation, sauvegardes, plan de reponse — deja couverts (§6, §13).
- **Annotations IA** : si une IA de generation est utilisee sur des donnees chinoises, obligations de marquage ; les fonctions IA du CDC (§18.3) sont internes et ne destinent pas reemettre de contenu tiers.

#### 10.3 Obligations des acheteurs et des pays cibles

A traiter avec le conseil ; Fanours est vendeur, certaines obligations pesent sur l'acheteur (importateur de fait) :

- **Etats-Unis** : CPSC (rapports d'accident, certification CPSIA enfants), FTC (publicite trompeuse, prix), regles douanieres americaines (Section 301, *de minimis* : les seuils evoluent, **aucun seuil code en dur**, §11.7), *Mail, Internet, or Telephone Order Merchandise Rule*.
- **Union europeenne** : obligations du vendeur a distance (retractation 14 j, garantie legale 2 ans, information precontractuelle, TVS), IOSS/OSS selon le modele retenu (§11.7), obligations d'accessibilite (European Accessibility Act) — la section accessibilite vise WCAG 2.2 AA, au-dela du minimum legal.
- **Royaume-Uni** : Consumer Contracts Regulations, Consumer Rights Act 2015, marquage UKCA/CE, distance selling post-Brexit.
- **Canada, Australie** : competition and consumer laws (ACCC, Competition Bureau), product safety, obligation d'indication d'origine — directement liee a §11.5.
- **Affichage** : fiche produit et checkout doivent permettre de comprendre qui est le vendeur, les prix TTC/HT selon la regle applicable, les delais, les droits de retour et les obligations d'import (§11.4, §11.10, §9.2).

#### 10.4 Controles a faire valider par un conseil (responsable nomme + jalon)

Aucune ligne n'est close par la seule lecture du code.

| # | Question a valider | Responsable | Jalon |
|---|---|---|---|
| 1 | Entite juridique, pays d'enregistrement, base legale PIPL | Direction + DPO | Phase 0 |
| 2 | Base legale et information de chaque traitement (cookies, analytics, e-mail marketing, IA) | DPO | Phase 1 |
| 3 | Transferts hors UE/UK (Chine, US) et garanties retenues | DPO | Phase 1 |
| 4 | Modele TVA/OSS/IOSS par destination et obligations d'enregistrement | Finance + fiscaliste | Phase 1 |
| 5 | Incoterms, responsable du dedouanement, TVA a la livraison, statut d'importateur de fait | Conformite | Phase 1 |
| 6 | CGU/CGV par segment (B2C/B2B) et par pays ; nullite des clauses limitant les droits imperatifs | Direction juridique | Phase 2 |
| 7 | Politique de retour et garantie (14 j, 2 ans) et exclusions licites | Support + juridique | Phase 2 |
| 8 | Mentions obligatoires sur les factures (entite, identifiants, TVA, devise, Incoterm) | Finance | Phase 2 |
| 9 | Classification et marquage des produits reglementes (CE/UKCA, CPSC, batteries, cosmetiques, REACH) | Conformite | Phase 3 |
| 10 | Regles de la plateforme de vente et obligations de l'acheteur professionnel (B2B) | Conformite | Phase 2 |
| 11 | Procedure de reclamation, mediation et litiges | Support | Phase 2 |
| 12 | Publication CGU/CGV, politique de confidentialite et cookies dans chaque langue (§4.1) | Produit | Phase 4 |
| 13 | DPA et transferts avec chaque prestataire | DPO | Phase 1 |
| 14 | Retention et suppression des pieces de verification B2B (§8.1) | DPO | Phase 2 |
| 15 | Information precontractuelle B2B et statut d'auto-entrepreneur | Juridique | Phase 2 |
| 16 | Regles douanieres : sources officielles retenues, date de revision, proprietaire (§11.2) | Conformite | Phase 1, puis trimestriel |

---

### 11. Douane, regles et anti-fraude

Interdits explicites du CDC (§11.5, §11.9, §11.10, §28.1) : pas de sous-valuation deliberee, pas de fausse categorie ou fausse origine, pas d'ecartement artificiel en colis multiples, pas d'IA confirmant seule un classement douanier. La section 11 transforme ces interdits en controles techniques.

#### 11.1 Coherence valeur / quantite / poids

| Controle | Regle | Seuil et exception |
|---|---|---|
| Valeur declaree vs prix catalogue | Valeur en douane = prix de vente (produits + transport interne), jamais un montant saisi | ecart > 0 % -> blocage |
| Valeur unitaire vs palier | Coherente avec le palier de quantite (§7.2) | ecart > 0 % -> blocage |
| Poids / quantite | Poids declare par variante coherent avec la quantite commandee, dans une bande de tolerance de l'emballage | hors bande -> revue manuelle |
| Seuil de valeur | Au-dela d'un seuil (fixe par pays, avec source et date), revue de conformite avant expedition | seuil configurable, versionne |
| Marge de tolerance | Aucun seuil « approximatif » en dur : source, date de validite et proprietaire sur chaque seuil (§16.9) | revue trimestrielle |

#### 11.2 Detection d'ecartement de colis (artificial parcel splitting)

Le fractionnement artificiel vise a contourner un seuil de valeur, un seuil de droits ou une restriction. Signaux : **S1** meme destinataire + meme adresse (normalisee) + colis distincts crees en < 24 h dont la somme des valeurs franchit un seuil ; **S2** meme compte / meme moyen de paiement / meme appareil, colis vers des pays differents ; **S3** meme produit et meme variante, quantites divisees en lots juste sous un seuil ; **S4** poids et valeur de chaque colis anormalement proches d'un seuil (« juste en dessous »).

Seuil de detection : **3 colis ou plus** du meme destinataire en 24 h, ou 2 colis dont la somme franchit un seuil de valeur, ou une valeur unitaire dans la bande **[90 %, 100 %]** d'un seuil sur 3 occurrences consecutives. Action : **blocage** de l'expedition des colis supplementaires, tache `compliance_manager`, notification au client si l'expedition est retenue (§11.9), journalisation (§19.7). Le blocage est reversible apres revue manuelle ; il n'est **jamais** leve automatiquement.

#### 11.3 Origine reelle vs pays d'expedition

- Deux champs distincts et obligatoires : `country_of_origin` (fabrication reelle, §11.5) et `country_of_shipment` (depart logistique). Un produit fabrique en Chine mais expedie depuis un hub reste `origin = CN`, `shipment = HK`.
- `country_of_origin` est un attribut du **produit**, herite dans le snapshot de commande (§20.2) et dans le document douanier (§11.8).
- Coherence verifiee : un produit avec origine declaree dans l'UE exporte depuis la Chine declenche une **alerte de revue** (preference ou etiquetage d'origine), pas un blocage automatique — la decision appartient au `compliance_manager`.
- Toute divergence entre origine et etiquette de l'expéditeur declenche une alerte (§11.9).

#### 11.4 Codes HS : statut, source, expiration

```sql
CREATE TABLE hs_code (
  code              text        NOT NULL,
  version           text        NOT NULL,          -- nomenclature (ex. HS2022) — §11.5
  authority         text        NOT NULL,          -- autorite / source officielle
  source_url        text        NOT NULL,
  description       text        NOT NULL,
  status            text        NOT NULL CHECK (status IN ('suggested','confirmed','expired','restricted')),
  confirmed_by      uuid        REFERENCES app_user(id),
  confirmed_at      timestamptz,
  valid_from        date        NOT NULL,
  valid_until       date,
  source_checked_at date        NOT NULL,          -- §16.9 « dernier controle »
  next_review_at    date        NOT NULL,
  review_owner      uuid        NOT NULL,
  UNIQUE (code, version)
);
```

Regles : le code d'un produit est un **lien** vers `hs_code`, pas une chaine libre. `suggested` (propose par l'IA ou un regleur) **!=** `confirmed` : un produit dont le code est `suggested` **ne peut pas** etre expedie (§11.5, §11.9) — le blocage est la regle par defaut. `confirmed` exige un `confirmed_by` humain identifie (`compliance_manager` ou `super_admin`), une source et une date de validite. **Aucune IA ne peut confirmer seule un classement douanier definitif** (§11.5) : la proposition de l'IA est un `suggested` avec trace du modele, du prompt et de la date ; la validation humaine est obligatoire et tracee dans l'audit. Une regle dont `valid_until` est depasse passe a `expired` : le moteur refuse de l'appliquer, alerte levee, produits concernes bloques selon la criticite (§26.4). Chaque regle de calcul (droits, TVA, frais) porte source, proprietaire et date de revision (§16.9, §26.4).

#### 11.5 Verrou sur expedition incomplete

Une expedition ne peut pas partir si : le code HS est absent, `suggested` ou `expired` ; `country_of_origin` manque ; la valeur en douane est incalculable (regle manquante, devise inconnue) ; un document obligatoire manque (§11.8) ; le produit est interdit dans la destination (§11.6) ; un controle anti-fraude est en revue (§11.2). Le controle est un **garde** execute dans la meme transaction que la creation de l'expedition ; le retour est un etat `customs_action_required` (§9.3), pas une erreur technique. Client notifie (§11.9), support voit la cause, audit conserve l'evenement.

```ts
// src/shipment/customs-gate.ts
export async function assertShippable(orderId: string, actorId: string): Promise<ShipmentReadiness> {
  const order = await loadOrderWithProducts(orderId);
  const blockers: Blocker[] = [];
  for (const line of order.lines) {
    const hs = line.hsCodeId ? await loadHsCode(line.hsCodeId) : null;
    if (!hs) blockers.push({ code: "hs_code_missing", lineId: line.id });
    else if (hs.status !== "confirmed") blockers.push({ code: `hs_code_${hs.status}`, lineId: line.id });
    if (!line.countryOfOrigin) blockers.push({ code: "country_of_origin_missing", lineId: line.id });
    if (await isProductRestricted(line.productId, order.country)) {
      blockers.push({ code: "product_restricted_in_destination", lineId: line.id });
    }
  }
  if (!order.customsValueCents || order.customsValueCents <= 0) {
    blockers.push({ code: "customs_value_uncomputable" });
  }
  if (await isSplitShipmentSuspect(order)) blockers.push({ code: "split_shipment_suspect" });
  if (blockers.length > 0) {
    await setOrderState(order.id, "customs_action_required");
    await audit.write({ actorId, action: "shipment.blocked.customs", objectType: "order",
      objectId: order.id, result: "denied", correlationId: order.correlationId, metadata: { blockers } });
    await notifySupportAndCustomer(order, blockers);
    return { shippable: false, blockers };
  }
  return { shippable: true, blockers: [] };
}
```

#### 11.6 Suspension d'urgence d'un pays ou d'une categorie

Le CDC impose de pouvoir desactiver un pays ou une categorie en urgence, avec justification et audit (§11.6). Technique : un **kill switch** par pays et par categorie, applique de facon atomique a la disponibilite du catalogue, du checkout, de la recherche et du moteur de regles. Effet : les produits ne sont plus achetable, les paniers existants passent en `awaiting_review` (pas de perte de commande silencieuse), les commandes en vol sont signalees, l'operation est tracee avec justification et auteur. Delai d'effet : **< 60 s** (cache invalide par evenement de domaine). Regle : la suspension est un **acte de conformite**, jamais une mesure de performance ; elle n'est jamais automatique.

#### 11.7 Tracabilite et alerting douaniere

Journal des incidents (§11.9) : colis retenus, frais supplementaires, motifs, actions, horodatage, montant, transporteur, reference. Alertes : `customs_rule.expiring`, `hs_code.unconfirmed`, `customs_value.incoherent`, `split_shipment.suspect`, `country.suspended`, `origin.mismatch`, `shipment.blocked` ; chacune a un destinataire (conformite, support), un delai de traitement et une escalade.

---

### 12. Chaine de build, dependances et securite d'infrastructure

#### 12.1 Dependances et chaine logicielle

- **Epinglage** : versions exactes + `package-lock.json` versionne ; `npm ci` en CI (jamais `npm install`).
- **Scan** : `npm audit --audit-level=high`, `osv-scanner` sur le lockfile, Trivy sur l'image ; **bloquants** au-dela du seuil haut (§24.6).
- **Mise a jour** : Dependabot ou Renovate, PR automatiques, revue humaine obligatoire ; pas de merge automatique sur une dependance critique (framework, ORM, SDK de paiement, agent d'auth).
- **SBOM** : genere a chaque release (CycloneDX ou SPDX), attache a l'artefact, conserve — inventaire de reference en cas d'incident de supply chain.
- **Provenance** : artefact signe (Sigstore/cosign), attestation de build, verification de la signature au deploiement.
- **Build reproductible** : memes entrees -> meme digest ; horodatages normalises, images base epinglees **par digest** (jamais `latest`).
- **Licences** : analyse de licences (incompatibilites GPL/AGPL dans un produit commercial), dependances transitives examinees.
- **Caches CI** : pas de cache partage non authentifie (risque de poison de cache) ; cache prive et lie au lockfile.
- **Contournement** : aucune dependance non publiee ni issue d'un fork sans revue ; toute dependance ajoutee exige une justification dans la PR.

#### 12.2 Durcissement de l'infrastructure

| Controle | Mise en oeuvre |
|---|---|
| Images conteneur | Base minimale (`distroless` ou `alpine`), **utilisateur non-root** (`USER app`, UID fixe, sans shell), pas de package manager dans l'image finale, `readOnlyRootFilesystem` quand possible, `cap_drop: ALL`, `no-new-privileges` |
| Ports exposes | Un seul port applicatif interne ; le reverse proxy expose 443 (et 80 pour la redirection). Ni base, ni cache, ni worker joignables depuis l'exterieur (§19.5) |
| Reseau | Base et cache sur un sous-reseau prive ; egress controle par allowlist ; pas d'IP publique sur les pods |
| Secrets en runtime | Injection par le gestionnaire de secrets au demarrage ; aucun secret dans l'image, l'historique de build ni les variables de build (le build est en reseau isolee, sans acces aux secrets de production) |
| WAF / CDN | WAF avec regles OWASP, rate-limit au bord, mitigation L7 DDoS ; le CDN ne sert que les assets publics ; le HTML n'est pas mis en cache sur `/account`, `/admin`, `/api` |
| TLS | Certificats ACME renouveles automatiquement, surveillance d'expiration, TLS 1.3, HSTS preload (§7.7) |
| Isolation des environnements | Production, staging, test et dev sur comptes, reseaux, cles et bases **distincts** (§19.5, §25.1) ; une erreur cross-environnement est detectee au demarrage (comparaison du nom d'environnement declare) |
| Donnees de developpement | Donnees synthetiques en dev et CI (§25.1) ; aucune copie de donnees reelles sans base juridique, masquage et procedure approuvee ; l'anonymisation est irreversible et testee |
| Acces production | Acces nominatif, MFA, elevation temporaire, sessions tracees, revue periodique des droits, moindre privilege (§19.5) |
| Agents de securite | EDR sur les noeuds ; images de base issues de registries de confiance, scannees avant promotion |
| Sauvegardes | Chiffrees, comptes de service dedies, copies hors site, **test de restauration periodique** (§19.8) ; RPO <= 24 h / RTO <= 8 h proposes, a confirmer |
| Observabilite | Logs structureles sans PII superflue (§23.2), `correlation_id` de bout en bout, alertes sur taux d'erreur, echec de paiement, file d'e-mails, webhooks, stock critique |

#### 12.3 Rotation des secrets

- Rotation programmee : 90 j pour les secrets applicatifs, 180 j pour les secrets fournisseurs ; le gestionnaire accepte deux versions simultanees (basculement sans downtime).
- Rotation d'urgence (< 1 h) : revocation, redelivraison, redemarrage des instances concernees, invalidation des sessions si le secret est lie a l'authentification.
- Un secret suppose vole (scan de l'historique, leak externe, signalement) suit le **runbook incident securite** (§12.4) : rotation immediate, evaluation d'impact, recherche d'usage dans les logs, notification des parties concernees (§19.6).

#### 12.4 Plan de reponse aux incidents et 5 runbooks

Runbooks exiges (§23.2) : **paiement indisponible**, **prestataire e-mail en panne**, **defaillance transporteur**, **incident securite**, **restauration**. Chaque runbook contient : declencheur, symptomes dans les metriques, severite, role de garde, procedure pas-a-pas, contournement, critere de reprise, communication, post-mortem. Les runbooks sont **testes** (jeu de role en staging au moins une fois par trimestre) : un runbook non teste est un document, pas un runbook.

**Runbook 1 — Paiement indisponible** (prestataire, 3DS ou passerelle) : symptomes = taux d'echec de paiement > 5 % sur 15 min, ou `payment_processing` > 10 min, ou file de webhooks bloquee. Actions : basculer le `PaymentProvider` via feature flag vers le prestataire secondaire (le fournisseur est encapsule derriere une interface interne, §10.1) ; proposer le « paiement differe / virement » si prevu (§10.2) ; **ne jamais simuler un succes** ; geler les commandes en `payment_processing` (elles ne doivent pas etre confirmees) ; communiquer sur le status page et dans le compte client ; reconcilier a la reprise.

**Runbook 2 — Prestataire e-mail en panne** : symptomes = file d'e-mails en erreur, taux d'echec > 10 %, bounce rate en hausse. Actions : basculement du provider d'email transactionnel (§21.4) ; passage en file d'attente avec backoff ; inventaire des envois non critiques (newsletter, campagnes) tout en maintenant les **transactionnels** (confirmation de commande, facture, notification de paiement, verification d'e-mail, reinitialisation de mot de passe) ; alerte ops ; re-envoi **idempotent** (cle d'idempotence par message, §9.5 : pas de double e-mail apres reprise).

**Runbook 3 — Defaillance transporteur** : symptomes = erreurs d'appel API transporteur, tracking non mis a jour > 30 %, retards. Actions : desactiver le transporteur concerne dans le referentiel (§16.8) ; basculer vers le transporteur secondaire ou le mode « retrait / retour depot » si configure ; ne pas promettre de delai non confirme (§12.1) ; informer les clients ; mettre a jour le tableau des exceptions ; rejouer les creations d'expedition de facon idempotente.

**Runbook 4 — Incident securite** (compte compromis, fuite de secret, webhook forge, upload malveillant, escalation de privileges) : declencheur = alerte securite ou signalement. Procedure : 1) **contenir** (revoquer les sessions, bloquer l'IP ou le compte, desactiver la cle compromise, suspendre le pays/categorie si pertinent) ; 2) **preserver** (snapshot des logs, export de l'audit en lecture seule, gel des purges) ; 3) **evaluer** (perimetre, donnees concernees, obligation de notification 72 h) ; 4) **remedier** (rotation des secrets, correctif, de-identification si possible) ; 5) **combler** (rotation de cle KMS, invalidation des URL signees, revocation des tokens) ; 6) **communiquer** (internes, clients, autorite si requis, §19.6) ; 7) **post-mortem** blameless avec actions datees. Le journal d'audit (§6.6) et le `correlation_id` sont les preuves de reference.

**Runbook 5 — Restauration** : declencheur = perte de donnees (corruption, suppression accidentelle, erreur d'infrastructure). Procedure : evaluer l'etat (PITR, snapshot, sauvegarde chiffree) ; restaurer dans un environnement isole **avant** la production ; verifier l'integrite (comptages, chaine de hachage de l'audit, coherence du rapprochement financier §10.5) ; communiquer l'impact ; basculer le trafic ; rejouer les traitements asynchrones de facon idempotente. Regle forte : **une restauration n'est annoncee reussie qu'apres verification** ; exercice trimestriel (§19.8).
---

### 13. Tests de securite

« Les tests sont une exigence de livraison » (§24) ; le parcours de securite E2E est explicitement listé (§24.3). La section 14 verifie que ces tests existent et sont verts ; cette section les concretise.

#### 13.1 Parcours de securite E2E (§24.3) traduit en cas de test

| # | Scenario CDC (§24.3) | Assertion executable |
|---|---|---|
| 1 | « Acces a la commande d'un autre utilisateur via modification d'ID refuse » | `GET /api/orders/{id_B}` avec la session de A -> **404** (pas 403 : l'existence n'est pas revelee) ; reponse identique (corps, statut) a un id inexistant ; 0 ligne lue en base |
| 2 | « Jeton expire ou invalide refuse » | cookie `fanours_sid` modifie (1 caractere) -> **401** ; session revoquee -> **401** ; session au-dela de l'expiration absolue -> **401** ; identifiant **pre-login** reutilise apres connexion -> **401** (rotation effective) |
| 3 | « Depassement du rate limit » | 6e `POST /api/auth/login` en 15 min depuis la meme IP -> **429** + `Retry-After` present + `X-RateLimit-Remaining: 0` ; la 7e ne modifie pas le compteur de verrouillage de compte |
| 4 | « CSRF/XSS et validation des entrees » | `POST` sans `X-CSRF-Token` ou avec un jeton d'une autre session -> **403** ; `Origin: https://evil.example` sur route mutante -> **403** ; `<script>alert(1)</script>` dans `name` -> **400** a l'ecriture et **echappe** a la lecture ; `javascript:alert(1)` dans une URL de profil -> **400** ; `"><img src=x onerror=...>` dans un commentaire -> **200** sans execution (verifie via `page.on('dialog')` et console) |
| 5 | « Televersement de fichier non autorise refuse » | `.exe` renomme `.pdf` -> **415** (magic bytes) ; `.pdf` de 12 Mo -> **413** ; fichier valide passe le scan antivirus -> stocke dans le bucket « documents » (jamais « quarantine ») ; URL du fichier en quarantaine inaccessible |
| 6 | « MFA incorrect et codes de recuperation deja consommes refuses » | code TOTP faux -> **401** ; **le meme** code rejoue 30 s apres -> **401** (anti-replay par compteur) ; code de recuperation deja utilise -> **401** ; le secret TOTP n'apparait dans aucune reponse ni aucun log |
| 7 | « Route administrative inaccessible sans role approprie » | `/api/admin/refunds` en `support_agent` -> **403** ; `/api/admin/exports/financial` en `analyst_readonly` -> **403** ; `/api/admin/roles` sans 4 yeux -> **409** `approval_required` ; requete admin sans session MFA recente -> **403** `mfa_required` |

#### 13.2 Tests de concurrence et d'etat (stock, double commande, double webhook)

```ts
// tests/concurrency/stock-oversell.spec.ts — §7.3, §24.5, §26.2, §28.1
it("n'autorise jamais plus que le stock disponible", async () => {
  await db.stock.create({ data: { sku: "SKU-1", qty: 10, reserved: 0 } });
  const results = await Promise.allSettled(
    Array.from({ length: 50 }, () => reserveStock({ sku: "SKU-1", quantity: 1, orderId: crypto.randomUUID() })));
  expect(results.filter((r) => r.status === "fulfilled").length).toBe(10);  // exactement le stock
  const stock = await db.stock.findUnique({ where: { sku: "SKU-1" } });
  expect(stock!.qty - stock!.reserved).toBe(0);                          // jamais negatif
});
```

```ts
// tests/payments/double-webhook.spec.ts — §9.5, §26.3, §28.1
it("traite deux fois le meme webhook sans dupliquer les effets", async () => {
  const body = buildSignedWebhook({ eventId: "evt_123", type: "payment.succeeded" });
  expect((await postWebhook(body)).status).toBe(200);
  expect((await postWebhook(body)).status).toBe(409);              // deja traite
  expect(await db.invoice.count({ where: { orderId } })).toBe(1);
  expect(await db.stockReservation.count({ where: { orderId } })).toBe(1);
  expect(await db.outboxEvent.count({ where: { aggregateId: orderId } })).toBe(1);
  expect(await db.notification.count({ where: { orderId } })).toBe(1);
});
it("refuse un webhook dont la signature est invalide et ne fait aucun effet", async () => {
  const body = buildSignedWebhook({ eventId: "evt_124", signature: "deadbeef" });
  expect((await postWebhook(body)).status).toBe(401);
  expect(await db.paymentAttempt.count({ where: { orderId } })).toBe(0);
});
it("refuse un webhook hors tolerance d'horodatage (rejeu a 10 min)", async () => {
  expect((await postWebhook(buildSignedWebhook({ eventId: "evt_125", timestamp: now() - 600 }))).status).toBe(401);
});
it("refuse un corps modifie apres signature (signature calculee sur le corps brut)", async () => {
  const signed = sign(JSON.stringify(payload));
  expect((await postWebhook(signed.replace(/"amount":100/, '"amount":1'))).status).toBe(401);
});
```

```ts
// tests/orders/idempotency.spec.ts — §9.5
it("ne cree qu'une commande pour la meme cle d'idempotence", async () => {
  const key = crypto.randomUUID();
  const body = { items: [{ sku: "SKU-1", quantity: 2 }], shippingAddressId, acceptTerms: true };
  const a = await postOrder(body, { "Idempotency-Key": key });
  const b = await postOrder(body, { "Idempotency-Key": key });
  expect(a.status).toBe(201);
  expect(b.status).toBe(200);                                     // reponse rejouee
  expect(b.body.orderId).toBe(a.body.orderId);
  expect(await db.order.count({ where: { idempotency_key: key } })).toBe(1);
});
it("refuse la reutilisation d'une cle d'idempotence avec un corps different", async () => {
  const key = crypto.randomUUID();
  await postOrder(orderBody(2), { "Idempotency-Key": key });
  expect((await postOrder(orderBody(9), { "Idempotency-Key": key })).status).toBe(409);
});
```

#### 13.3 Tests de non-regression du guard RBAC

La matrice §3.2 (9 roles x 14 actions) est executee comme test parametrique : c'est le test qui garantit qu'un role n'obtient jamais une action par accident (refactor, duplication de la table, ajout d'un role).

```ts
// tests/authz/guard.spec.ts
import { decide, PERMISSIONS, type Role, type Resource, type Action } from "@/authz/permissions";
const ROLES: Role[] = ["super_admin","catalog_manager","order_manager","logistics_operator",
  "finance_manager","support_agent","compliance_manager","marketing_manager","analyst_readonly"];
const RESOURCES: Resource[] = ["order","product","price_tier","export","export_financial","refund",
  "admin_role","audit_log","customs_rule","country","business_document","business_account","customs_document"];
const ACTIONS: Action[] = ["read","create","update","delete","export","approve","refund","manage_roles"];
const EXPECTED_DENY: Array<[Role, Resource, Action]> = [
  ["support_agent","refund","refund"], ["catalog_manager","audit_log","delete"],
  ["catalog_manager","export_financial","export"], ["analyst_readonly","order","update"],
  ["analyst_readonly","export_financial","export"], ["marketing_manager","audit_log","read"],
  ["order_manager","price_tier","update"], ["logistics_operator","business_document","read"],
  ["compliance_manager","export_financial","export"], ["finance_manager","business_account","approve"],
  ["support_agent","admin_role","manage_roles"], ["catalog_manager","business_account","approve"],
];
describe("guard RBAC — deny by default", () => {
  it.each(EXPECTED_DENY)("refuse %s sur %s:%s", (role, resource, action) => {
    expect(decide(role, resource, action)).toBe("deny");
  });
  it("refuse toute action non declaree pour tout role", () => {
    for (const role of ROLES) for (const resource of RESOURCES) for (const action of ACTIONS) {
      if (!PERMISSIONS[role]?.[resource]?.includes(action)) expect(decide(role, resource, action)).toBe("deny");
    }
  });
  it("interdit l'auto-approbation (4 yeux)", async () => {
    await expect(approve({ approvalId, approverId: requesterId })).rejects.toThrow(/self_approval_forbidden/);
  });
});
```

#### 13.4 Fuzzing des schemas de validation

```ts
// tests/validation/schema-fuzz.spec.ts
import fc from "fast-check";
import { CreateOrderSchema } from "@/http/validation";
it("rejette toute entree hors schema sans lever d'erreur non controlee", () => {
  fc.assert(fc.property(fc.anything(), (input) => {
    const res = CreateOrderSchema.safeParse(input);
    if (res.success) {
      expect(Object.keys(res.data).every((k) =>
        ["items","shippingAddressId","billingAddressId","promoCode","acceptTerms"].includes(k))).toBe(true);
    } else expect(res.error.issues.length).toBeGreaterThan(0);
  }), { numRuns: 500 });
});
it("rejette les champs d'elevation (role, prix, customerId) meme dans un payload valide", () => {
  const injected = { ...validOrder, role: "super_admin", price: "0.01", customerId: "autre-id" };
  expect(CreateOrderSchema.safeParse(injected).success).toBe(false);
});
```

Cible de fuzzing : `POST /api/orders`, `/api/checkout`, `/api/payments`, `/api/quotes`, `/api/auth/register`, `/api/contact`, `PUT /api/admin/products/:id/price-tiers`, `POST /api/admin/customs/rules`. 500 cas par endpoint en CI, 5 000 en pre-release.

#### 13.5 Tests d'integration securite supplementaires

| Theme | Test | Reference |
|---|---|---|
| Journal immuable | `UPDATE audit_log` / `DELETE FROM audit_log` -> exception SQL ; chaine de hachage reverifiee (recalcul de `entry_hash` sur les N dernieres entrees) | §6.6, §19.7 |
| Redaction des logs | Aucun `token`, `password`, `secret`, `cookie`, `authorization`, `pan`, `cvv` dans les logs de test (grep sur les fixtures) | §12.2, §19.7 |
| Chiffrement colonne | `business_document.blob` illisible sans la cle KMS ; un `aad` different fait echouer le dechiffrement | §6.4 |
| Codes de recuperation | Un code utilise deux fois -> refus ; le hachage en base n'est pas lisible (Argon2id) | §4.3 |
| Sessions | Rotation apres login et apres elevation ; ancien cookie invalide ; expiration inactive et absolute respectees | §4.1 |
| Google Sign-In | `state` reuse -> refus ; `nonce` incorrect -> refus ; `alg: none` ou `HS256` dans l'ID token -> refus | §4.4 |
| Anti-enumeration | Corps, statut et duree de reponse identiques pour un e-mail existant et inconnu (100 iterations, ecart < 5 ms) | §9 |
| SSRF | Liste de 15 cibles interdites rejetees (loopback, RFC1918, link-local, IPv6, DNS rebinding) | §7.4 |
| CORS | Origine hors allowlist -> pas d'en-tete ACAO ; `Allow-Credentials: true` sans origine explicite -> erreur de config detectee par un lint de configuration | §7.6 |
| En-tetes | 8 en-tetes presents sur `/`, `/admin`, `/api`, avec les valeurs attendues | §7.7 |
| Exports | `analyst_readonly` -> **403** sur export financier ; lignes plafonnees ; URL expiree apres 24 h ; export journalise | §3.2, §10.5 |
| Double controle | Demande -> approbation par un tiers -> execution ; auto-approbation refusee ; payload modifie apres approbation -> `payload_tampered` ; approbation expiree -> refus | §3.5 |
| Douane | Code HS `suggested` -> expedition bloquee ; origine manquante -> blocage ; signal d'ecartement de colis -> blocage + tache | §11 |
| Paiement | Montant du webhook different de la commande -> `payment_review` + alerte ; `paid` jamais atteint par le retour navigateur | §5.2, §5.3 |
| Rapprochement | Ecart de commission detecte, tache creee, resolution tracee | §5.5 |

#### 13.6 Analyse de dependances et scan de secrets en CI

- `npm audit --audit-level=high` et `osv-scanner` : **bloquants** ; une PR Dependabot ne peut pas etre mergee si le scan echoue.
- `gitleaks detect` + `trufflehog` sur l'historique complet : bloquants.
- `semgrep ci` (OWASP Top 10 + regles TypeScript) : bloquant sur severites elevees.
- Trivy sur l'image de production : bloquant sur CRITICAL/HIGH (vulns OS + libs).
- Test de non-regression du scanner : une dependance vulnerable de test (fixture) doit etre detectee, sinon le scanner est considere inoperant.
- Rapport de scan conserve en artefact de build, avec le SBOM.

#### 13.7 Tests d'acceptation de conformite (§26)

| Criter (§26) | Preuve |
|---|---|
| §26.2 « Le backend recalcule les prix » | Test du calcul de palier + E2E : payload du panier avec `price` modifie -> prix serveur |
| §26.2 « Le stock ne passe pas sous zero dans un test concurrent » | §13.2 |
| §26.3 « Aucun statut paye sur redirection navigateur » | E2E : retour navigateur sans webhook -> `payment_processing`, jamais `paid` |
| §26.3 « Webhooks verifies et idempotents » | §13.2 |
| §26.3 « Les e-mails de commande et de paiement sont envoyes sans doublon » | `notification.count === 1` (§13.2) |
| §26.4 « Toute regle comporte une source, un proprietaire et une date de revision » | Test de schema : creation sans `source_url`/`review_owner` -> 400 |
| §26.4 « Les regles expirees ou incompletes declenchent une alerte et un blocage » | Test du moteur : regle `valid_until` passee -> alerte + blocage |
| §26.5 « MFA administrateur activee » | E2E admin : connexion sans MFA -> refuse ; avec MFA -> acces |
| §26.5 « Permissions testees cote serveur » | §13.3 (matrice complete) |
| §26.5 « Aucun secret present dans le depot ou les bundles publics » | gitleaks + grep des patterns de secret dans `.next/static` |
| §26.5 « Scenarios d'acces non autorise refuses » | §13.1 |
| §26.5 « Sauvegarde et restauration testees » | Test de restauration automatise (§12.4 runbook 5) |
| §26.5 « Logs depourvus de secrets et de donnees de carte » | §13.5 |

---

### 14. Definition of Done — checklist securite avant merge

Chaque item est coche par un outil ou une methode identifiee. Une PR ne peut pas etre mergee avec un item critique en echec ; un item « non applicable » doit etre justifie dans la PR.

| # | Item | Verification / outil | Bloquant |
|---|---|---|---|
| 1 | Aucun secret dans le code, le build, les logs ou le bundle | `gitleaks detect` + `trufflehog` (historique complet) + grep des bundles `.next/static` | Oui |
| 2 | Toutes les entrees sortantes sont validees par un schema `zod` strict | Fuzzing (500 cas/endpoint) + revue du `.strict()` | Oui |
| 3 | Aucune requete SQL concatenee ; requetes parametrees uniquement | Lint interdisant `sql.raw` et l'usage de `sql` avec une entree utilisateur + grep `sql.raw` | Oui |
| 4 | Aucun `dangerouslySetInnerHTML` hors composant unique ; HTML sanitise cote serveur | Lint + `grep -R "dangerouslySetInnerHTML"` (1 occurrence autorisee, commentee) | Oui |
| 5 | CORS : allowlist explicite, pas de `*` avec credentials | Test d'integration CORS + revue de config | Oui |
| 6 | CSP sans `unsafe-inline` sur `script-src` ; `connect-src` limite | Test d'en-tetes + rapport CSP sans violation bloquante | Oui |
| 7 | En-tetes de securite presents (HSTS, nosniff, X-Frame-Options/frame-ancestors, Referrer-Policy, Permissions-Policy) | `e2e/specs/security-headers.spec.ts` (Phase 1) — verifie sur `/`, `/admin` et l'API | Oui |
| 8 | Session : cookie `HttpOnly` + `Secure` + `SameSite` + `Path` ; rotation apres login et elevation | Test E2E de session | Oui |
| 9 | Route `/admin/*` : role **et** MFA verifie cote serveur a chaque requete | Test parametrique de la matrice RBAC + test MFA admin | Oui |
| 10 | Actions financieres et changement de role : workflow 4-yeux actif, auto-approbation impossible | Test du workflow d'approbation | Oui |
| 11 | Anti-IDOR : la propriete de la ressource est verifiee dans la requete | E2E IDOR sur les 14 actions | Oui |
| 12 | Webhooks : signature HMAC sur corps brut, tolerance d'horodatage, anti-rejeu, idempotence | `double-webhook.spec.ts` + `webhooks/signature.spec.ts` | Oui |
| 13 | Aucun statut `paid` issu d'une redirection navigateur | E2E paiement (§26.3) | Oui |
| 14 | Idempotence sur creation de commande et paiement (cle + contrainte unique) | `orders/idempotency.spec.ts` | Oui |
| 15 | Aucune donnee de carte stockee, loguee ou analysee | Grep schema/migrations + test `no-card-data` + test de redaction des logs | Oui |
| 16 | Rate limiting actif sur toutes les routes d'auth, recherche et API | Test de depassement (429 + `Retry-After`) + revue des routes | Oui |
| 17 | Chiffrement au niveau colonne des documents B2B et des secrets TOTP (enveloppe KMS) | Test `crypto/column-encryption.spec.ts` | Oui |
| 18 | Journal d'audit immuable, chaine de hachage verifiee, 13 evenements couverts | Test d'immuabilite + test de couverture des evenements | Oui |
| 19 | Uploads : magic bytes, taille max, renommage UUID, antivirus, stockage hors racine web | Test d'upload (extension falsifiee, taille, quarantaine) | Oui |
| 20 | Dependances sans CVE haute/critique ; SBOM et artefact de build signe | `npm audit` + `osv-scanner` + Trivy + verification de signature en CI | Oui |
| 21 | Conteneur non-root, sans shell, ports minimaux, secrets hors image | Analyse d'image Trivy + revue du Dockerfile | Oui |
| 22 | Tests de securite verts (E2E §13.1, concurrence §13.2, RBAC §13.3, fuzzing §13.4) | Pipeline CI vert sur la branche | Oui |
| 23 | Aucun `TODO`/`FIXME` de securite ni de contournement temporaire en attente | Grep + revue | Oui |
| 24 | Revue securite par un pair pour toute modification touchant auth, paiement, roles, exports ou regles | PR approuvee par le role `security-auditor` / reviewer designe | Oui |
| 25 | Impact sur la vie privee et la retention evalue si des donnees personnelles sont ajoutees | Checklist RGPD/PIPL de la PR + journal de mise a jour du registre | Oui |

Regle de merge : les items 1 a 18 et 20 a 23 sont **bloquants** par defaut ; un item non applique doit etre coche avec une justification ecrite dans la PR. Le bypass d'un item bloquant impose l'accord explicite du proprietaire de la securite et laisse une trace dans le journal d'audit de la decision (§6.6).

Cette checklist ne couvre pas la conformite metier (tarifs, douane, litiges) : elle verifie que les controles techniques qui portent ces regles existent, sont actifs et sont testes. La validite des regles elles-memes (seuils, sources, Incoterms) releve du proprietaire metier et du conseil (§10.4).
