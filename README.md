# Fanours

Plateforme e-commerce **B2B / B2C** pour une boutique basée en Chine qui vend principalement **en gros à une clientèle internationale**, avec vente au détail activable selon le produit.

> **Statut : en cours — phase de cadrage et de spécification. Aucun code applicatif n'existe encore.**
> Le dépôt contient pour l'instant le cahier des charges, la roadmap et les plans d'exécution. Il n'y a ni application à lancer, ni démo, ni capture d'écran : aucune n'est publiée tant que l'interface n'existe pas.

## Objectif

Permettre à des acheteurs professionnels et particuliers de découvrir des produits, comparer les prix selon les volumes, acheter des lots, demander des devis pour les grosses commandes, et suivre leurs commandes. L'expérience vise la confiance : prix lisibles, MOQ (minimum de commande) explicite, stock, délais, frais et taxes estimés avant paiement.

## Fonctionnalités prévues (d'après le cahier des charges)

Ces éléments sont **planifiés, pas implémentés** :

- Catalogue public, prix dégressifs par paliers, MOQ, demandes de devis, tarifs spécifiques par client
- Comptes individuels et entreprises, vérification professionnelle, MFA/2FA
- Panier, checkout, paiement (Stripe prévu), remboursements, commandes
- Expéditions, suivi, documents douaniers, estimation des taxes et droits d'import (règles versionnées, présentées comme estimations)
- Factures et documents commerciaux localisés, notifications e-mail
- Trois langues (français, anglais, chinois simplifié) et plusieurs devises
- Dashboards client et administrateur, journal d'audit
- Thème clair / sombre, accessibilité WCAG 2.2 AA, SEO, PWA installable

Hors périmètre initial : application native iOS/Android, marketplace multi-vendeurs, stockage de numéros de carte.

## Stack cible

| Couche | Choix prévu |
|---|---|
| Front | Next.js (App Router), TypeScript strict, Tailwind CSS, tokens CSS |
| Back | TypeScript, monolithe modulaire |
| Données | PostgreSQL, Drizzle |
| Cache / files | Redis (BullMQ) |
| Fichiers | Stockage compatible S3 + CDN |
| Paiement | Interface `PaymentProvider`, implémentation Stripe |
| Tests | Vitest, Testing Library, Playwright |
| Déploiement | Conteneurs (Cloud Run / ECS) |

## Lancement

Sans objet pour le moment : il n'y a pas encore de code à installer ni à exécuter.

## Contenu du dépôt

Sur la branche `main`, le dépôt ne contient que ce README. Les documents de pilotage sont dans la PR ouverte [#1](https://github.com/nagoloumdaniel/Fanours/pull/1) (branche `feat/roadmap-and-project-instructions`) :

```
Cahier_des_charges_Fanours.md   Spécification fonctionnelle, technique et UX/UI (source de vérité)
ROADMAP.md                      Roadmap en 7 phases (0 à 6)
CLAUDE.md                       Instructions et conventions du projet
docs/roadmap/sections/          Détail par thème (architecture, design, SEO, sécurité, qualité, infra, git)
docs/superpowers/plans/         Plans d'exécution par phase
```

## Avancement

| Phase | Objectif | État |
|---|---|---|
| 0 | Cadrage et conformité | Plan rédigé, non exécuté |
| 1 | Fondations et design system | Non démarrée |
| 2 | Catalogue et commerce | Non démarrée |
| 3 | B2B, logistique et douane | Non démarrée |
| 4 | Dashboards et opérations | Non démarrée |
| 5 | Assurance qualité et lancement | Non démarrée |
| 6 | Améliorations post-lancement | Non démarrée |

## Licence

Aucune licence n'a été choisie à ce jour : **tous droits réservés par défaut**. Le code source étant public mais sans licence, il ne peut être réutilisé sans autorisation de l'auteur.
