# Fiche de conception — Baromètre des prix PV v2 (page portail interactive)

> Fiche remplie le 2026-08-07 selon le template `Design_System/livrables/BRIEF_PACK_SYNAPSUN/FICHE_PAGE.md`.
> Accompagne le prototype `prototype_barometre_visitor.html` (même dossier).

---

## 0. Aiguillage

- [x] **Non, c'est une fonctionnalité/donnée applicative** → portail (pas Strapi). La page affiche des données de prix hebdomadaires issues d'un pipeline externe, avec dataviz interactive.

---

## 1. Identité

- **Nom de la page** : Baromètre des prix des modules photovoltaïques (v2 interactive)
- **Objectif** : le visiteur consulte l'évolution hebdomadaire des prix de toute la chaîne de valeur PV (polysilicium → wafer → cellule → module → verre, + taux de change figés) sous forme de graphiques et tableaux, télécharge l'édition mensuelle PDF, et peut contacter Synapsun.
- **Audience** :
  - [x] Public (visiteur non connecté)
- **URL souhaitée FR** : `/barometre` — **route existante** `front_barometer` (`HomeController`), déjà dans le sitemap. Pas de nouvelle route à créer.
- **URL souhaitée EN** : `/barometer` — idem, déjà déclarée.
- **Entrée de menu souhaitée** : catégorie « Ressources » (`menu_main()`), libellé « Baromètre des prix » — à ajouter via `MenuNavigationExtension.php` (jamais le template).

---

## 2. Données (bloquant)

- **Quelles données affiche la page ?**
  1. Séries de prix hebdo par produit (5 catégories, 13 séries `show_in_barometer=true`, ~110 semaines depuis W1-2024) ;
  2. Dernière valeur + variation semaine-1 + tendance 4 semaines par produit ;
  3. Taux de change figés à la génération (EUR/USD, EUR/CNY, argent XAG/oz) ;
  4. Commentaire de marché FR/EN (champ réservé, null en v1) ;
  5. Archive mensuelle image + PDF (entité `Barometer` existante).
- **Chaque donnée est-elle dans le catalogue (`08_CATALOGUE_DONNEES.md`) ?**
  - [x] **Cas tranché hors catalogue** : les données 1-4 viennent du contrat JSON versionné `barometer.json` (schema_version 1, produit chaque vendredi par le pipeline GitHub Actions du repo `barometer-graph-gsheet`). C'est le **pattern 12 « Ingestion de données externes »** de `07_PATTERNS_AVANCES.md`, décision Franck 2026-07-27 : hébergement portail retenu, pipeline externe = source de vérité (question L1 du questionnaire IT). Acheminement recommandé : **cron pull** `app:import:barometer` (cookbook `cli/import_command.md`, Sentry Monitor) ; alternative webhook entrant authentifié (modèle `WebhookController` Zoho). La donnée 5 est déjà en base (entité `Domain/Barometer`).
- **Fraîcheur requise** : hebdomadaire (vendredi ~09:00 UTC). Aucun appel externe au moment de la visite : la page sert des données déjà importées en base/cache serveur.
- **Volumétrie attendue** : 13 séries × ~110 points ; page unique, pas de pagination.

---

## 3. Écritures (bloquant)

- [x] **Non** — aucune écriture en v1. (Roadmap Lot 1 : formulaire d'abonnement hebdo → canal `ZohoLeadApiWriter` existant — hors périmètre de cette page v1, à spécifier séparément.)
- **Emails déclenchés** : aucun.

---

## 4. Sécurité / RGPD (bloquant)

- **Données personnelles affichées ?** [x] Non — uniquement des prix de marché agrégés.
- **Un client ne doit voir que SES propres données ?** [x] Sans objet (page publique, données identiques pour tous).
- **Upload/download de fichiers ?**
  - [x] Download → PDF mensuel via la route publique existante `front_barometer_download` (document public par nature, pas d'ownership requis — c'est déjà le comportement en production).
- **Parcours invité avec formulaire de soumission ?** [x] Non (pas de formulaire en v1 → pas de reCAPTCHA).

---

## 5. UX

- **Composants pressentis** : [x] Boutons · [x] Badges/Pills · [x] Cartes · [x] Tables · [x] Breadcrumb · dataviz Chart.js (pattern 7)
- **Pattern de page** : dashboard public (sections `section__header`/`section__content`, KPI cards, graphiques + tableaux)
- **États à prévoir** :
  - vide [x] : JSON absent/périmé → fallback sur le dernier contenu mensuel image + PDF (mécanique `barometer_missing.html.twig` existante) ; série sans aucune valeur → exclue côté serveur ;
  - erreur [x] : import en échec → la page sert la dernière version en base (résilience du cron pull) ;
  - chargement [x] : rendu serveur (pas de spinner ; les graphiques s'initialisent au chargement du bundle).
- **Dataviz nécessaire ?** [x] Oui → Chart.js via le contrôleur Stimulus `chart` (`@symfony/ux-chartjs`), jamais de CDN ni d'autre lib. Palette catégorielle validée (skill dataviz, validateur 6 checks, ordre fixe) : `#536cc6` (secondary-lighter) → `#e27928` (primary) → `#10b2bd` (lagoon) → `#a765de` (pink) sur fond blanc ; le contraste limite du lagoon est compensé par légende + vue tableau adjacente (relief exigé par le validateur).

---

## 6. SEO (page publique)

- **Meta title** : « Baromètre des prix des modules photovoltaïques - Synapsun » (56 car.)
- **Meta description** : « Suivez chaque semaine les prix du polysilicium, des wafers, cellules, modules et du verre solaire. Données TaiyangNews analysées par les experts Synapsun. » (155 car.)
- **Indexable ?** [x] Oui — route déjà dans le sitemap, canonicals/hreflang gérés automatiquement par le layout (paths FR/EN déclarés). Les canonicals du dashboard GitHub Pages pointent déjà vers `/barometre` · `/barometer` (Phase 0 du 2026-07-17) : la page portail devient la cible canonique.

---

## 7. Mesure

- **Événements analytics à suivre** (implémentation IT, question J1) : clic CTA « Nous contacter » depuis le baromètre ; clic « Télécharger le PDF » ; (v2 : soumission formulaire d'abonnement).
- **Définition du succès** : trafic organique sur `/barometre` (contenu frais hebdo) + leads contact attribués à la page.

---

## 8. Recette

- **Critères d'acceptation** :
  1. Étant donné un import `barometer.json` réussi (W28-2026), quand un visiteur ouvre `/barometre`, alors il voit la semaine de référence, 6 KPI, 5 sections de graphiques avec tableaux, et les taux figés au 2026-07-17 — sans aucun appel réseau tiers depuis son navigateur.
  2. Étant donné un JSON absent ou périmé (> 9 jours), quand un visiteur ouvre `/barometre`, alors la page sert la dernière version importée en base ; si aucune donnée n'a jamais été importée, le fallback image + PDF mensuel s'affiche (comportement `barometer_missing` actuel).
  3. Étant donné la page affichée en FR, quand le visiteur bascule en EN via le sélecteur de langue du header, alors tous les libellés (clés `page.barometer.*`) sont traduits et l'URL devient `/barometer`.
  4. Étant donné la page affichée à 375 px de large, quand le visiteur fait défiler, alors aucun débordement horizontal ; graphiques et tableaux restent lisibles.
- **Qui valide la recette métier** : Franck Catanese.
- **Feature flag souhaité ?** [x] Oui — nom pressenti : `barometer_v2` (`feature_toggle.yaml` + `#[BehindFeatureGate]`) ; rollback = désactiver le flag (retour au rendu image + PDF actuel).

---

*Fiche transmise avec le prototype `prototype_barometre_visitor.html` pour conversion selon `06_CONVERSION_TWIG.md` (paquet `spec_IT_barometre/`).*
