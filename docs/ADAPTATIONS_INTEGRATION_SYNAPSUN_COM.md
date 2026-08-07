# Adaptations du projet Barometer pour une intégration fluide dans synapsun.com

> Analyse du 2026-07-17 — croisement du pipeline Barometer (`C:\Claude\Synapsun\Barometer`) avec la cartographie du portail (`synapsun_com/docs/ARCHITECTURE_SYNAPSUN_COM.md`), la fiche d'intégration (`INTEGRATION_PROJETS.md` §1) et le guide « code + specs » (`GUIDE_DEV_INTEGRATION.md`). Code du portail vérifié dans le clone lecture seule `synapsun_com/data/portal-repo/`.

---

## 1. État des lieux — les deux mondes à réconcilier

### Ce que le portail synapsun.com possède déjà

| Élément | Détail |
|---|---|
| Page publique | `front_barometer` — routes `fr: /barometre` · `en: /barometer` (`HomeController.php:30`), **dans le sitemap**, meta SEO dédiées |
| Contenu actuel | Entité `Barometer` (`src/Domain/Barometer/Entity/ORM/Barometer.php`) : **1 image (≤ 2 Mo) + 1 PDF (≤ 4 Mo) par période mensuelle** (VichUploader), meta title/keywords/description, date |
| Publication | **Manuelle** — un humain uploade image + PDF chaque mois via EasyAdmin (rôle `ROLE_BAROMETER`) |
| Téléchargement | `front_barometer_download` (`/barometre/{id}/telecharger/pdf`) + variante `_unique` |
| Fallback | `barometer_missing.html.twig` si aucune donnée |
| Leads | `ZohoLeadApiWriter` (Lead_Source = portal.synapsun.com) + workflow rules Zoho « Portal - New Lead » actives (restriction UE incluse) ; domaine `Newsletter` existant dont le formulaire mentionne déjà le baromètre |
| Lien Zoho | **Aucun** pour l'entité Barometer (pas de module CRM) |
| Conventions | Symfony 6.4/7.2, Twig + Bootstrap 5 + **Stimulus (Turbo désactivé)**, Webpack Encore, i18n Loco (zéro texte en dur), contrôleurs single-action, ViewModels `final readonly`, feature flags, PHPStan 9, tests Application/Integration |
| Mode de livraison | **« code + specs » uniquement** — interdiction de pousser dans `synapsun/portal` ; livrable remis au prestataire (branche Jira D26 → PR vers `dev`) |

### Ce que le pipeline Barometer produit aujourd'hui

| Élément | Détail |
|---|---|
| Pipeline | Scraper TaiyangNews → Claude Vision → Google Sheets, GitHub Actions vendredi 08:00 UTC + health check quotidien 07:00 UTC |
| Rendu | Dashboard HTML monolithique (~86 Ko, JS/CSS inline) sur GitHub Pages (`synapsun-dev.github.io/barometer-graph-gsheet`) |
| Dépendances **côté navigateur du visiteur** | 1. CSV gviz Google Sheets (données prix) · 2. `sheets.googleapis.com` v4 · 3. API BCE (FX USD/CNY→EUR) · 4. `open.er-api.com` (FX fallback) · 5. `currency-api` jsdelivr + pages.dev (argent XAG) · 6. **2 iframes Zoho Analytics** · 7. Chart.js via CDN jsdelivr · 8. flagcdn.com (drapeaux) |
| i18n | Toggle FR/EN maison en JS, textes en dur dans le HTML |
| SEO | schema.org + canonicals déjà orientés synapsun.com… mais vers des **URLs inexistantes** (`/fr/barometre-prix-modules-photovoltaiques`, `/en/pv-module-price-barometer` — la route réelle est `/barometre` · `/barometer`) |
| Roadmap v2 | Email hebdo → leads Zoho, commentaire de marché auto, simulateur DDP, alertes internes, exports |

### Le diagnostic en une phrase

Le portail a déjà **la page, la route SEO, l'admin, le rôle et le canal leads** ; le pipeline Barometer a déjà **les données fraîches et l'automatisation**. Ce qui manque entre les deux : un **contrat de données stable**, un front **conforme aux conventions du portail**, et la suppression des **dépendances tierces côté client** incompatibles avec un site corporate (CSP, RGPD, fiabilité).

---

## 2. Adaptations Phase 0 — immédiates, dans Barometer, indépendantes de l'arbitrage

Ces quatre chantiers se font entièrement dans le repo Barometer, sans toucher au portail, et gardent toute leur valeur quel que soit l'arbitrage hébergement (portail vs GitHub Pages).

### 2.1 Contrat de données JSON versionné — la pièce maîtresse

Le portail ne doit **jamais** parser le CSV gviz de Google (format non contractuel, quota, dépendance Google en frontal). Ajouter au run hebdo une étape d'export qui publie sur GitHub Pages un fichier stable, par exemple `data/barometer.json` :

```json
{
  "schema_version": 1,
  "generated_at": "2026-07-17T08:12:00Z",
  "last_week": "W28-2026",
  "source": "TaiyangNews",
  "fx": { "usd_eur": 0.86, "cny_eur": 0.12, "xag_usd_oz": 38.2, "fixed_at": "2026-07-17" },
  "market_comment": { "fr": null, "en": null },
  "categories": [
    { "name": "Module", "products": [
      { "name": "TOPCon Module", "unit": "USD/W", "show_in_barometer": true,
        "series": [ { "week": "W27-2026", "value": 0.087 } ],
        "wow_change_pct": -1.1, "trend_4w": "down" }
    ]}
  ]
}
```

- Nouveau script `export_json.py` (ou étape dans `taiyangnews_pv_scraper.py`) + step dans `pv_price_weekly.yml`.
- **Les taux FX et le prix de l'argent sont figés à la génération** : c'est ce qui permettra de supprimer les 4 appels d'APIs tierces du navigateur (§3.3).
- `schema_version` obligatoire : le portail (ou tout autre consommateur) se cale dessus ; toute rupture = incrément.
- Champ `market_comment` FR/EN prévu dès maintenant (vide) pour accueillir le Lot 1 de la roadmap v2 sans casser le schéma.

### 2.2 Génération automatique image + PDF au format admin du portail

C'est le « court terme, zéro dev portail » recommandé par la fiche d'intégration : l'humain `ROLE_BAROMETER` ne fait plus que valider l'upload mensuel.

- Rendu headless (Playwright/Chromium dans le runner GitHub Actions) du dashboard → **PNG ≤ 2 Mo** + **PDF ≤ 4 Mo** (contraintes `Assert\Image(maxSize: '2048k')` / `Assert\File(maxSize: '4096k')` de l'entité `Barometer`).
- Généré au dernier run du mois, avec **meta title/description proposées** (l'entité les porte) et la date de période.
- Dépôt en artifact de run + email — l'upload EasyAdmin reste le seul geste humain.

### 2.3 Correction des canonicals / SEO du dashboard

Les canonicals actuels pointent vers des 404 — les signaux SEO envoyés depuis GitHub Pages sont perdus. Deux options :

- **Option simple (recommandée)** : aligner les canonicals + hreflang du dashboard sur les routes réelles `https://synapsun.com/barometre` · `https://synapsun.com/barometer`.
- Option ambitieuse : conserver les URLs longues descriptives, mais alors livrer au prestataire une spec de renommage de la route `front_barometer` (nouveaux paths FR/EN + redirections 301 des anciennes) — à ne faire que si la valeur SEO du slug long est jugée décisive.

### 2.4 Basculer le dashboard GitHub Pages sur le JSON

Le dashboard consomme son propre `barometer.json` (le CSV gviz reste en fallback). Bénéfices : le dashboard devient le **client de référence du contrat de données** (il le valide chaque semaine), et la page GitHub Pages servira de préversion/environnement de dev une fois la page portail en place.

---

## 3. Adaptations Phase 1 — refonte « portail-compatible » du front (si arbitrage = portail)

> EN ATTENTE de l'arbitrage hébergement (question bloquante posée dans `synapsun_com`). Recommandation maintenue : **portail** — SEO sur le domaine synapsun.com, capture de leads native, page/rôle/admin déjà existants.

Tout ce qui suit se livre au format **« code + specs »** (template §4 du guide), ticket Jira D26, jamais de push direct.

### 3.1 Exploser le monolithe HTML selon les conventions du portail

| Aujourd'hui (Barometer) | Cible (portail) |
|---|---|
| 1 fichier HTML 86 Ko, JS/CSS inline | Template `templates/visitor/barometer/show.html.twig` étendant `visitor/layout.html.twig` |
| JS inline | Contrôleurs **Stimulus** `assets/controllers/barometer_*.js` (Turbo désactivé — aucune autre forme d'interactivité) |
| CSS inline | `assets/scss/components/_barometer.scss` importé dans `main.scss` |
| Chart.js CDN jsdelivr | **Chart.js via npm/Webpack Encore** (déjà utilisé côté admin du portail) — pas de CDN (CSP à venir, audit A-M6) |
| Toggle FR/EN maison | Routes i18n par tableau de paths + clés **Loco** `page.barometer.*` FR/EN — zéro texte en dur ; hreflang géré par le layout |
| schema.org inline | Conservé, alimenté par le ViewModel |

Architecture : contrôleur single-action + ViewModel `final readonly` (factory `fromEntity()`/`fromJson()`), sous `src/Business/Barometer/` (pattern de référence `Business/Sales/`) ou en refonte encadrée du legacy `Domain/Barometer`. L'entité actuelle (image + PDF mensuels) **reste** pour l'historique et le PDF téléchargeable — la page v2 s'y superpose.

### 3.2 Acheminement des données : pull (recommandé) ou push

| Option | Mécanisme | Adaptation côté Barometer | Avis |
|---|---|---|---|
| **(a) Pull — cron d'import** | Commande `app:import:barometer` (pattern cookbook `cli/import_command.md`) qui lit `barometer.json` sur GitHub Pages, upsert en base, **Sentry Monitor** comme les autres crons | Aucune (le JSON de la Phase 0 suffit) | **Recommandé** : zéro secret partagé, zéro couplage des runs, résilient (le portail sert la dernière version connue si GitHub Pages est indisponible) |
| (b) Push — webhook | `POST /api/webhook/barometer-update` en fin de run Actions (pattern upsert async, auth token) | Étape supplémentaire dans `pv_price_weekly.yml` + secret GitHub ↔ variable d'env portail (`.env` template + `.gpg`) | Fraîcheur immédiate, mais gestion de secret et couplage en plus — à réserver si le délai du cron est jugé insuffisant |

Dans les deux cas, la page ne fait **aucun appel externe au moment de la visite** : elle sert des données déjà en base (ou en cache serveur).

### 3.3 Supprimer les dépendances tierces côté navigateur

| Dépendance actuelle | Traitement |
|---|---|
| CSV gviz + sheets.googleapis | Remplacés par le JSON (Phase 0) importé côté serveur |
| API BCE, open.er-api, currency-api (FX + argent) | Valeurs **figées dans le JSON** à la génération hebdo ; si un cours « live » est vraiment souhaité, proxy côté serveur avec cache — jamais depuis le navigateur du visiteur |
| flagcdn.com | Drapeaux en assets locaux (ou toggle de langue natif du layout, qui rend le composant inutile) |
| Chart.js CDN | Bundle Encore local |
| **2 iframes Zoho Analytics** | À remplacer par des graphes Chart.js natifs alimentés par le même JSON. À défaut (transitoire) : lazy-load + fallback, mais cookies tiers/CSP/RGPD en font un candidat prioritaire à la suppression |

### 3.4 Filet de sécurité et déploiement progressif

- **Feature flag `barometer_v2`** (`feature_toggle.yaml` + `#[BehindFeatureGate]`) : la page interactive s'active progressivement ; rollback = désactiver le flag.
- Fallback : si le JSON est absent/périmé → afficher le dernier contenu image + PDF (mécanique `barometer_missing.html.twig` déjà en place).
- Livrable complet conforme au guide : tests `Application` (WebTestCase, assertions `data-test-id`), clés Loco FR/EN, schéma d'entité (le prestataire génère la migration), entrée cron + Sentry Monitor, plan de rollback.

---

## 4. Adaptations Phase 2 — réaligner la roadmap v2 sur le portail

| Item roadmap v2 (Barometer) | Réalignement |
|---|---|
| **Abonnement email hebdo** (Lot 1) | Ne PAS construire de formulaire maison sur GitHub Pages. Formulaire sur la page portail → canal **`ZohoLeadApiWriter`** existant (Lead_Source = portal.synapsun.com ; workflow rules « Portal - New Lead » déjà actives, restriction UE comprise) ou domaine Newsletter. L'étape d'envoi hebdo du pipeline consomme la liste Zoho — pas de liste parallèle |
| **Commentaire de marché auto** (Lot 1) | Généré en **FR + EN** par le pipeline, transporté par le champ `market_comment` du JSON, affiché en tête de la page portail = contenu frais SEO sur synapsun.com (au lieu de GitHub Pages) |
| **Alertes prix internes** (Lot 1) | Restent 100 % dans le pipeline (`health_check.py`) — rien à porter côté portail |
| **Simulateur DDP** (Lot 2) | Contrôleur Stimulus dédié sur la page portail ; briques FOB/fret/FX transportées par le JSON ; CTA « Obtenir un prix ferme » → formulaire → lead Zoho (bien plus de valeur sur le portail : visiteur potentiellement connecté et identifié) |
| **Annotations d'événements** (Lot 2) | Onglet dédié du Google Sheet → section `annotations` du JSON (extension de schéma) |
| **Export PNG/CSV + permaliens** (Lot 3) | Export par graphique en Stimulus ; permaliens = query params sur `/barometre` (catégorie + période) |
| **Vue interne spot vs négocié** (Lot 3) | Ne va PAS sur la page publique — candidat naturel : page **connectée** `ROLE_SYNAPSUN` (checklist B du guide), données via un onglet privé du Sheet ou import dédié |
| **Health check** | Une fois la bascule faite : vérifier la fraîcheur de la **page portail** (lag entre `last_week` du JSON et l'affichage) en plus/au lieu de GitHub Pages |

---

## 5. Récapitulatif des impacts fichier par fichier (repo Barometer)

| Fichier | Adaptation | Phase |
|---|---|---|
| `export_json.py` (nouveau) | Génération du contrat `barometer.json` (schema_version, FX/XAG figés, market_comment) | 0 |
| `pv_price_weekly.yml` | + step export JSON ; + step rendu image/PDF fin de mois ; (+ step webhook si option push) | 0 (webhook : 1) |
| `render_snapshot.py` (nouveau) | Playwright headless → PNG ≤ 2 Mo + PDF ≤ 4 Mo + meta proposées | 0 |
| `livrables/barometre-synapsun.html` | Canonicals/hreflang → `/barometre` · `/barometer` ; consommation du JSON (CSV en fallback) | 0 |
| `health_check.py` | + check fraîcheur JSON ; plus tard + check page portail | 0 puis 2 |
| `taiyangnews_pv_scraper.py` | Inchangé (ou appel d'`export_json.py` en fin de run) | 0 |
| Livrable « code + specs » (nouveau dossier, projet `synapsun_com`) | Page Twig/Stimulus/SCSS + import cron + tests + spec template §4 | 1 |
| Roadmap v2 (PROJECT.md) | Lot 1 « email hebdo » repointé sur le canal leads du portail | 2 |

## 6. Décisions à trancher (gouvernance)

1. **Arbitrage hébergement** (question bloquante déjà posée dans `synapsun_com`) : page interactive **sur le portail** (recommandé) vs maintien GitHub Pages avec lien. Toute la Phase 1 en dépend ; la Phase 0 non.
2. **Pull vs push** pour l'acheminement des données (§3.2) — recommandation : (a) cron pull.
3. **Sort des iframes Zoho Analytics** : remplacement natif (recommandé) vs maintien transitoire en lazy-load.
4. **Slug SEO** : conserver `/barometre` (simple) vs renommage en URL longue descriptive avec 301 (spec prestataire supplémentaire).

---

*Références : `synapsun_com/docs/ARCHITECTURE_SYNAPSUN_COM.md` · `synapsun_com/docs/INTEGRATION_PROJETS.md` §1 · `synapsun_com/docs/GUIDE_DEV_INTEGRATION.md` · clone lecture seule `synapsun_com/data/portal-repo/` (routes et entité vérifiées le 2026-07-17).*
