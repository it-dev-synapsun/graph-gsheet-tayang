# Runbook — Upload du snapshot baromètre juillet 2026 dans EasyAdmin

**Projet :** Barometer (Synapsun)
**Date :** 2026-07-21
**Action concernée :** Uploader `snapshots/2026-07/` dans l'admin EasyAdmin `/barometre` avec le rôle `ROLE_BAROMETER`

---

## Pourquoi cette action reste manuelle

La décision est déjà documentée dans `docs/ADAPTATIONS_INTEGRATION_SYNAPSUN_COM.md` :

> Publication : **Manuelle** — un humain uploade image + PDF chaque mois via EasyAdmin (rôle `ROLE_BAROMETER`)

C'est un choix assumé (« court terme, zéro dev portail ») en attendant l'arbitrage hébergement de la Tâche 6. **Claude Code n'a pas de session authentifiée sur l'admin de production `synapsun.com`** — le rôle `ROLE_BAROMETER` est réservé à un compte humain réel (login + éventuel 2FA). Cette étape ne peut donc pas être automatisée depuis cet environnement : ce document prépare tout le travail en amont pour que Franck n'ait plus qu'à cliquer.

## Pré-requis vérifiés (déjà faits)

Fichiers présents et conformes dans `C:\Claude\Synapsun\Barometer\snapshots\2026-07\` :

| Fichier | Taille | Limite serveur | Marge |
|---|---|---|---|
| `barometre-2026-07.png` | 1 749 472 octets (1.67 Mio) | 2 048 000 octets (`Assert\Image(maxSize: '2048k')`) | ✅ OK — 14,6 % de marge |
| `barometre-2026-07.pdf` | 1 302 164 octets (1.24 Mio) | 4 096 000 octets (`Assert\File(maxSize: '4096k')`) | ✅ OK — 68,2 % de marge |

Limites lues dans le code source : `synapsun_com/data/portal-repo/src/Domain/Barometer/Entity/ORM/Barometer.php`.

## Étapes à réaliser (Franck, ~2 minutes)

1. Se connecter à l'admin EasyAdmin de production (`https://synapsun.com/admin` ou l'URL admin habituelle si différente) avec un compte disposant du rôle `ROLE_BAROMETER`.
2. Menu **Baromètre** → bouton **« Ajouter un baromètre »** (route `admin_barometer_new`).
3. Remplir le formulaire avec ces valeurs exactes :

   | Champ | Valeur à saisir |
   |---|---|
   | Image (`imageFile`) | `C:\Claude\Synapsun\Barometer\snapshots\2026-07\barometre-2026-07.png` |
   | PDF (`pdfFile`) | `C:\Claude\Synapsun\Barometer\snapshots\2026-07\barometre-2026-07.pdf` |
   | Titre méta (`metaTitle`) | Baromètre des prix modules photovoltaïques — juillet 2026 \| Synapsun |
   | Mots-clés méta (`metaKeywords`) | prix modules photovoltaïques, baromètre PV, prix polysilicium, prix wafer, modules TOPCon, FOB Chine, juillet 2026, Synapsun |
   | Description méta (`metaDescription`) | Baromètre Synapsun juillet 2026 : suivi hebdomadaire des prix de la filière photovoltaïque — polysilicium, wafers, cellules TOPCon, modules FOB Chine, verre solaire, fret maritime et taux de change EUR/USD/CNY. |
   | Date (période, `date`) | 07/2026 (juillet 2026) |

4. Valider. Un message de succès doit apparaître : « Baromètre du 07/2026 mis à jour ».
5. Vérifier le rendu public : `https://synapsun.com/barometre` (et `/barometer` en EN) affiche bien la nouvelle image et le lien PDF de juillet 2026.

## Traçabilité des sources utilisées pour préparer ce runbook

- Contenu méta : `snapshots/2026-07/meta.txt`
- Formulaire/entité : `synapsun_com/data/portal-repo/src/Domain/Barometer/Form/Type/BarometerType.php`, `.../Entity/ORM/Barometer.php`
- Contrôleurs admin : `synapsun_com/data/portal-repo/src/Controller/Admin/BarometerController.php`, `BarometerFormController.php`
- Rôles : `synapsun_com/data/portal-repo/config/packages/security.yaml` (`ROLE_BAROMETER: ROLE_SYNAPSUN`)
- Décision « publication manuelle » : `docs/ADAPTATIONS_INTEGRATION_SYNAPSUN_COM.md`

## Statut

☐ **En attente de Franck** — upload à réaliser manuellement dans l'admin de production. Tout le reste (fichiers, textes, chemins) est prêt et vérifié.
