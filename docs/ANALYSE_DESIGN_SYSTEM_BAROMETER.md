# Baromètre × Design System — modifications pour une intégration facile et stable sur synapsun.com

> Analyse du 2026-08-07 — croisement du projet Barometer avec le **Design System Synapsun**
> (`Design_System/livrables/BRIEF_PACK_SYNAPSUN/`, snapshot portail SHA `6e877ac7` du 2026-07-27).
> Prolonge `docs/ADAPTATIONS_INTEGRATION_SYNAPSUN_COM.md` (2026-07-17) : la Phase 0 (contrat JSON,
> canonicals, snapshot mensuel) était déjà livrée ; ce document couvre la **Phase 1 front**, désormais
> outillée par le design system.

---

## 1. Fait nouveau : l'arbitrage hébergement est tranché

La question bloquante « portail vs GitHub Pages » qui gelait la Tâche 6 du PROJECT.md **a été tranchée
le 2026-07-27** lors de la construction du design system : **hébergement portail retenu**, pipeline
externe conservé comme source de vérité (décision consignée dans `07_PATTERNS_AVANCES.md` §12
« Ingestion de données externes », cas réel = le Baromètre, question L1 du questionnaire IT).

Conséquences immédiates :
- la Tâche 6 (« EN ATTENTE de l'arbitrage ») est **débloquée** ;
- le chemin d'intégration officiel est le **workflow du BRIEF_PACK** : fiche de conception →
  prototype HTML autonome conforme au design system → conversion Twig (« code + specs », doc 06) ;
- le dashboard GitHub Pages actuel devient à terme la préversion/environnement de dev (Phase 0 §2.4).

---

## 2. Grille d'analyse — dashboard actuel vs exigences du design system

| # | Écart constaté (Barometer aujourd'hui) | Exigence design system | Traitement |
|---|---|---|---|
| M1 | Monolithe HTML 91 Ko, CSS/JS inline, structure maison (hero, TOC sticky, sections custom) | Page `visitor/layout.html.twig`, composants du catalogue uniquement (`section__*`, `card`, `table`, `badge`, `breadcrumb`), zéro CSS custom | ✅ **Appliqué** — prototype reconstruit sur `starter_visitor.html`, 0 `<style>` custom, 100 % composants catalogués |
| M2 | 8 dépendances tierces côté navigateur (CSV gviz, sheets.googleapis, API BCE, open.er-api, currency-api, flagcdn, CDN Chart.js, 2 iframes Zoho Analytics) | Aucun appel externe au moment de la visite ; FX/argent figés dans le contrat JSON ; Chart.js via bundle Encore (`@symfony/ux-chartjs`) ; pas de cookies tiers (CSP/RGPD) | ✅ **Appliqué au prototype** — données 100 % embarquées/serveur, FX figés affichés ; Chart.js CDN restant = démo uniquement, flagué pour conversion Encore |
| M3 | i18n maison (toggle FR/EN en JS, textes en dur, drapeaux flagcdn) | Clés Loco `page.barometer.*` (convention `{global}.{second}.{internal}.{component}.{atom}`), sélecteur de langue natif du layout, hreflang automatique | ✅ **Appliqué** — 41 clés `data-i18n` posées + tableau FR/EN complet ; toggle maison supprimé (layout du portail s'en charge) |
| M4 | Dataviz : couleurs non normées, iframes Zoho, graphiques sans contrat d'accessibilité | Chart.js uniquement (pattern 7), palette issue des tokens `01_TOKENS.md`, accessibilité dataviz | ✅ **Appliqué** — palette catégorielle **validée par le validateur 6-checks de la skill dataviz** : `#536cc6` (secondary-lighter) → `#e27928` (primary) → `#10b2bd` (lagoon) → `#a765de` (pink), ordre fixe ; légende + tooltips croisés + **vue tableau adjacente** (relief exigé, pastilles couleur liant tableau et séries) ; lignes 2 px, grille en retrait, `aria-label` par graphique |
| M5 | SEO : canonicals corrigés en Phase 0 vers `/barometre` · `/barometer` ; meta à la main | `meta_title`/`meta_desc` définis par le contrôleur, route déjà dans le sitemap, un seul `h1`, hiérarchie Hn | ✅ **Appliqué** — meta proposés (56/155 car.) dans la fiche §6 ; structure `h1` + `h2` de sections conforme aux mixins du portail |
| M6 | Données poussées vers le navigateur (fetch JSON GitHub Pages + fallback CSV) | Import côté serveur : **cron pull `app:import:barometer`** (cookbook `cli/import_command.md`, Sentry Monitor) recommandé ; alternative webhook authentifié (pattern 12) | 📋 **Consigné** — fiche §2 + INTEGRATION-NOTES (décision pull/push à confirmer à la conversion) |
| M7 | Pas de filet de déploiement | Feature flag `barometer_v2` + fallback `barometer_missing.html.twig` (image + PDF mensuels existants) | 📋 **Consigné** — fiche §8 + INTEGRATION-NOTES (rollback = désactiver le flag) |
| M8 | Page invisible dans la navigation du portail | Entrée de menu via `MenuNavigationExtension::getMenuMain()` (jamais le template) | 📋 **Consigné** — « Ressources → Baromètre des prix » proposé (clé `menu.barometer`), annoté dans le prototype |
| M9 | Aucun `data-test-id`, pas de tests Application | Convention `data-test-id` (kebab-case métier) + tests `ShowPublicActionTestInterface` niveau 2 | ✅ **Appliqué (niveau 1)** — 27 `data-test-id` posés, conservés tels quels à la conversion ; tests niveau 2 spécifiés par le paquet spec_IT |
| M10 | Sections argent/structure de coût/FOB/fret/DDP alimentées par iframes Zoho + APIs live | Toute donnée affichée doit venir du contrat JSON importé côté serveur | 📋 **Consigné** — **non repris en v1** (données absentes du schéma v1) ; extension `schema_version 2` candidate — voir §5 |

**Verdict d'ensemble** : aucun composant manquant au catalogue (zéro `<style data-integration>` nécessaire) —
la page Baromètre est entièrement réalisable avec le design system existant. Le seul « nouveau composant »
est fonctionnel, pas visuel : le contrôleur Stimulus de graphiques, déjà standardisé par `@symfony/ux-chartjs`.

---

## 3. Modifications appliquées ce jour (livrables)

1. **`livrables/PROTOTYPE_PORTAIL_BAROMETRE/prototype_barometre_visitor.html`** — prototype autonome
   niveau 1 de la page `/barometre` v2, conforme au BRIEF_PACK :
   - basé sur `starter_visitor.html` (header/footer réels du portail), assets de prod embarqués ;
   - 6 KPI cards, bloc « L'analyse Synapsun » (champ `market_comment`, contenu simulé, masqué si null),
     5 sections catégorie (graphique Chart.js + tableau), taux de change figés, édition mensuelle PDF
     (route `front_barometer_download` existante), méthodologie, CTA contact ;
   - données réelles W28-2026 embarquées depuis `data/barometer.json` (schema v1) ;
   - annotations de conversion complètes : `data-i18n` (41), `data-auth="public"`, `#route:` (43),
     `data-test-id` (27), tableau de traductions FR/EN, bloc INTEGRATION-NOTES.
2. **`livrables/PROTOTYPE_PORTAIL_BAROMETRE/FICHE_PAGE_BAROMETRE.md`** — fiche de conception remplie
   (sections bloquantes Données/Écritures/Sécurité incluses), prête à accompagner le prototype.
3. **Jeu de tests niveau 1 exécuté : 30/30 OK** (voir §6).
4. **PROJECT.md** — Tâche 6 débloquée et re-séquencée, question hébergement marquée résolue,
   nouvelles questions de validation consignées.

## 4. Rapport de test (protocole 05_JEU_DE_TESTS.md, niveau 1)

```
PLAN DE TEST — Prototype page portail /barometre v2
[x] A  Rendu 4 breakpoints (375/768/1200/1600) → aucun débordement ; nav mobile jusqu'à 1440px, desktop au-delà
[x] B  Conformité design system → 0 <style> custom ; toutes les couleurs ∈ tokens 01_TOKENS
[x] C  Éléments interactifs (présence / cursor+href / effet du clic) → CTA contact, download PDF, breadcrumb, hover tooltip graphique
[x] D  i18n → 41 clés data-i18n = tableau FR/EN, convention snake_case.dot respectée
[x] E  Annotations conversion → data-auth, #route: (43), data-test-id (27), INTEGRATION-NOTES
[x] F  Données/états → 5 graphiques (4/2/3/2/2 séries), 5 tableaux (13 lignes), W28-2026, KPI 0,68 RMB/W ; état vide market_comment documenté
---
RÉSULTATS : 30/30 OK (exécution Playwright/Chromium, 2026-08-07)
---
STATUT GLOBAL : VALIDÉ
```

Contrôle visuel effectué (captures) : rendu conforme au portail — titres `h2` à barre latérale orange,
cards, tableaux stylés, graphiques dans la palette validée avec légende et pastilles de liaison.

---

## 5. Ce qui reste à faire (consigné au PROJECT.md, Tâche 6)

| Étape | Contenu | Référence |
|---|---|---|
| Recette métier du prototype | Franck déroule les critères d'acceptation de la fiche §8 (ouvrir le prototype dans un navigateur) | `FICHE_PAGE_BAROMETRE.md` §8 |
| Conversion Twig « code + specs » | Session Claude Code avec clone `portal-repo`, prompt prêt à l'emploi §6 de `06_CONVERSION_TWIG.md` → paquet `spec_IT_barometre/` (contrôleur single-action, ViewModel, template visitor, contrôleur Stimulus chart, clés Loco, cron import + Sentry, feature flag, tests Application) | `06_CONVERSION_TWIG.md` |
| Acheminement des données | Trancher cron pull `app:import:barometer` (recommandé) vs webhook push | fiche §2, pattern 12 |
| Extension schéma v2 (optionnelle) | Rapatrier dans le contrat JSON les données aujourd'hui servies par les iframes Zoho / APIs live du dashboard GitHub Pages : FOB modules, fret maritime, DDP Europe, historique argent, structure de coût — pour élargir la page portail sans iframe ni cookie tiers | `export_json.py` (pipeline) |
| Lot 1 roadmap | Formulaire d'abonnement hebdo sur la page portail → canal `ZohoLeadApiWriter` existant ; commentaire de marché auto → champ `market_comment` déjà prévu | roadmap v2 PROJECT.md |

**Aucune modification appliquée aux dashboards GitHub Pages existants** (`index.html`,
`barometre-synapsun.html`) : ils restent le rendu de production actuel et l'entrée du snapshot mensuel
PNG/PDF (contraintes EasyAdmin ≤ 2 Mo / ≤ 4 Mo) — les restyler avant la bascule portail ferait courir un
risque au pipeline sans bénéfice (ils deviendront préversion une fois la page portail en ligne).

---

## 6. Questions à trancher

1. **Prototype validé pour conversion ?** — recette métier fiche §8 par Franck, puis lancement de la
   conversion Twig (paquet `spec_IT_barometre/`).
2. **Extension du contrat JSON (schema v2)** — faut-il rapatrier FOB/fret/DDP/argent/structure de coût
   dans le JSON pour une page portail complète, ou lancer la v1 avec les 5 catégories TaiyangNews + FX ?

---

*Références : `BRIEF_PACK_SYNAPSUN/` docs 00-09 · `docs/ADAPTATIONS_INTEGRATION_SYNAPSUN_COM.md` ·
`data/barometer.json` (schema v1, W28-2026) · tests : Playwright/Chromium local, 30/30 OK.*
