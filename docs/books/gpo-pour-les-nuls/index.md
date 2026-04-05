---
description: "Guide debutant pour comprendre et utiliser les strategies de groupe Windows sans prerequis technique. gpedit, GPMC, securite, diagnostic et mini-projets."
tags:
  - gpo
  - debutant
  - guide
---

# Les GPO pour les Nuls

Comprendre les GPO sans jargon, sans prerequis et sans casser son poste.
Vous partez de zero, puis vous montez progressivement jusqu'au premier vrai diagnostic.

---

## A qui s'adresse ce livre ?

Ce livre s'adresse aux debutants complets, aux techniciens en formation et aux curieux qui voient passer le mot "GPO" sans savoir par ou commencer. Aucun prerequis Active Directory n'est necessaire : les outils, le vocabulaire et les bons reflexes sont introduits pas a pas.

## Ce que vous allez maitriser

- Reconnaitre le role d'une GPO locale ou de domaine dans Windows
- Ouvrir et utiliser `gpedit.msc` et `gpmc.msc` sans se perdre
- Creer, lier et tester une premiere GPO en environnement AD
- Comprendre l'heritage, la priorite et les effets de bord les plus courants
- Appliquer des reglages concrets sur la securite, le bureau et Windows Update
- Sauvegarder avant modification et revenir en arriere proprement
- Lire les premiers indices de diagnostic avec `gpresult` et l'Observateur d'evenements

## Parcours de lecture suggere

### Debutant complet

Chapitres recommandes dans l'ordre :

1. [01. C'est quoi une strategie de groupe ?](01-cest-quoi.md)
2. [02. L'editeur de strategies locales (gpedit.msc)](02-gpedit.md)
3. [04. La console GPMC : premiers pas](04-gpmc-premiers-pas.md)
4. [05. Creer et lier sa premiere GPO](05-premiere-gpo.md)
5. [07. Les parametres de securite essentiels](07-securite-base.md)
6. [11. Sauvegarder avant de toucher](11-sauvegarde.md)
7. [12. Mon premier diagnostic GPO](12-diagnostic.md)

### Lecture ciblee par besoin

| Je veux... | Aller au chapitre |
|------------|-------------------|
| Comprendre enfin ce qu'est une GPO | [01 - C'est quoi une strategie de groupe ?](01-cest-quoi.md) |
| Modifier un poste local sans domaine AD | [02 - L'editeur de strategies locales](02-gpedit.md) |
| Faire mes premiers reglages utiles | [03 - 10 modifications utiles avec gpedit](03-modifications-utiles.md) |
| Creer ma premiere GPO de domaine | [05 - Creer et lier sa premiere GPO](05-premiere-gpo.md) |
| Comprendre qui gagne entre plusieurs GPO | [06 - Comprendre l'heritage et l'ordre d'application](06-heritage.md) |
| Durcir rapidement un poste Windows | [07 - Les parametres de securite essentiels](07-securite-base.md) |
| Configurer Windows Update proprement | [09 - Configurer Windows Update par GPO](09-windows-update.md) |
| Savoir pourquoi une GPO ne s'applique pas | [12 - Mon premier diagnostic GPO](12-diagnostic.md) |

## Tous les chapitres

| # | Chapitre | Theme |
|---|----------|-------|
| 01 | [C'est quoi une strategie de groupe ?](01-cest-quoi.md) | Fondamentaux |
| 02 | [L'editeur de strategies locales (gpedit.msc)](02-gpedit.md) | Outils locaux |
| 03 | [10 modifications utiles avec gpedit](03-modifications-utiles.md) | Premiers usages |
| 04 | [La console GPMC : premiers pas](04-gpmc-premiers-pas.md) | Console AD |
| 05 | [Creer et lier sa premiere GPO](05-premiere-gpo.md) | Deploiement |
| 06 | [Comprendre l'heritage et l'ordre d'application](06-heritage.md) | Priorite |
| 07 | [Les parametres de securite essentiels](07-securite-base.md) | Securite |
| 08 | [Gerer le bureau et le menu Demarrer](08-bureau-demarrer.md) | Experience utilisateur |
| 09 | [Configurer Windows Update par GPO](09-windows-update.md) | Maintenance |
| 10 | [Les preferences de registre par GPO](10-preferences.md) | Registre |
| 11 | [Sauvegarder avant de toucher](11-sauvegarde.md) | Sauvegarde |
| 12 | [Mon premier diagnostic GPO](12-diagnostic.md) | Diagnostic |
| 13 | [Les erreurs classiques a eviter](13-erreurs.md) | Hygiene admin |
| 14 | [GPO et Windows 11](14-windows11.md) | Compatibilite |
| 15 | [Mini-projets GPO : mettre en pratique](15-mini-projets.md) | Pratique |

!!! tip "Par ou commencer ?"
    Lisez les chapitres 1 a 5 d'une traite pour construire le socle mental, gardez le chapitre 11 comme filet de securite, puis revenez au chapitre 12 des qu'un parametre refuse de s'appliquer.
