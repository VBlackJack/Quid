---
description: "Reference exhaustive sur les strategies de groupe Windows : architecture, CSE, ADMX, SYSVOL, filtrage, loopback, GPP, securite, baselines et convergence MDM en 25 chapitres."
tags:
  - gpo
  - reference
  - architecture
---

# La Bible des Strategies de Groupe

La vue interne des GPO, depuis `gpsvc` et les CSE jusqu'a `registry.pol`, SYSVOL et MDM.
Ce livre se lit tres bien par probleme technique autant qu'en progression lineaire.

---

## A qui s'adresse ce livre ?

Ce livre vise les administrateurs systemes, ingenieurs poste de travail et architectes qui manipulent deja Active Directory au quotidien. Il vaut mieux arriver avec des bases AD, GPMC, PowerShell et un minimum d'aisance sur les journaux Windows pour tirer pleinement profit des chapitres les plus denses.

## Ce que vous allez maitriser

- Expliquer le role de `gpsvc`, des CSE et des attributs `gPC*ExtensionNames`
- Diagnostiquer les incidents SYSVOL, `registry.pol`, GPP et filtrage sans rester au niveau symptome
- Lire et decoder `gPLink`, LSDOU, loopback, WMI filters et les conflits de priorite
- Maitriser ADMX/ADML, Central Store, baselines et compatibilite entre versions Windows
- Savoir ou et comment une GPO ecrit dans le registre ou dans les fichiers de strategie
- Mesurer les performances d'application et isoler les CSE lentes
- Positionner GPO, LGPO, SCCM, Intune et MDM dans une meme architecture de configuration

## Parcours de lecture suggere

### Debutant complet

Chapitres recommandes dans l'ordre :

1. [01. Introduction et histoire des strategies de groupe](01-introduction.md)
2. [02. Architecture et composants internes](02-architecture.md)
3. [03. Client-Side Extensions (CSE) en profondeur](03-cse.md)
4. [07. Traitement des strategies : cycle et modes](07-traitement.md)
5. [08. Heritage et ordre d'application (LSDOU)](08-heritage-lsdou.md)
6. [09. Filtrage de securite et filtrage WMI](09-filtrage.md)
7. [20. RSoP et diagnostic des strategies de groupe](20-rsop-diagnostic.md)

### Lecture ciblee par besoin

| Je veux... | Aller au chapitre |
|------------|-------------------|
| Comprendre ce qui se passe sous le capot | [02 - Architecture et composants internes](02-architecture.md) |
| Identifier les GUID, DLL et timings des CSE | [03 - Client-Side Extensions (CSE) en profondeur](03-cse.md) |
| Depanner SYSVOL, GPT et replication | [04 - SYSVOL : replication et structure](04-sysvol.md) |
| Comprendre ADMX, ADML et le Central Store | [05 - Modeles d'administration (ADMX/ADML)](05-admx-adml.md) |
| Decoder `registry.pol` et les Extra Registry Settings | [06 - Le format registry.pol](06-registry-pol.md) |
| Comprendre LSDOU, Enforced et Block Inheritance | [08 - Heritage et ordre d'application (LSDOU)](08-heritage-lsdou.md) |
| Diagnostiquer un refus de filtrage securite ou WMI | [09 - Filtrage de securite et filtrage WMI](09-filtrage.md) |
| Arbitrer entre GPO et MDM/Intune | [25 - GPO et MDM : convergence et coexistence](25-mdm-convergence.md) |

## Tous les chapitres

| # | Chapitre | Theme |
|---|----------|-------|
| 01 | [Introduction et histoire des strategies de groupe](01-introduction.md) | Fondations |
| 02 | [Architecture et composants internes](02-architecture.md) | Moteur GPO |
| 03 | [Client-Side Extensions (CSE) en profondeur](03-cse.md) | CSE |
| 04 | [SYSVOL : replication et structure](04-sysvol.md) | SYSVOL |
| 05 | [Modeles d'administration (ADMX/ADML)](05-admx-adml.md) | Templates |
| 06 | [Le format registry.pol](06-registry-pol.md) | Stockage |
| 07 | [Traitement des strategies : cycle et modes](07-traitement.md) | Execution |
| 08 | [Heritage et ordre d'application (LSDOU)](08-heritage-lsdou.md) | Priorite |
| 09 | [Filtrage de securite et filtrage WMI](09-filtrage.md) | Ciblage |
| 10 | [Loopback Processing : modes et cas d'usage](10-loopback.md) | Session |
| 11 | [Preferences de strategie de groupe (GPP)](11-preferences-gpp.md) | Preferences |
| 12 | [Item-Level Targeting avance](12-ilt.md) | Ciblage fin |
| 13 | [Strategies de securite (mot de passe, audit, droits)](13-securite-strategies.md) | Hardening |
| 14 | [Windows Firewall with Advanced Security via GPO](14-firewall.md) | Pare-feu |
| 15 | [AppLocker et WDAC : controle d'application via GPO](15-applocker-wdac.md) | Application control |
| 16 | [BitLocker via GPO](16-bitlocker.md) | Chiffrement |
| 17 | [Deploiement de logiciels via GPO (MSI)](17-deploiement-msi.md) | Packaging |
| 18 | [Scripts et planification via GPO](18-scripts.md) | Automatisation |
| 19 | [Redirection de dossiers et profils via GPO](19-redirection-profils.md) | Profils |
| 20 | [RSoP et diagnostic des strategies de groupe](20-rsop-diagnostic.md) | Troubleshooting |
| 21 | [LGPO et strategies locales multiples](21-lgpo.md) | Local policy |
| 22 | [Baselines de securite (CIS, STIG, Microsoft)](22-baselines.md) | Conformite |
| 23 | [Performances et optimisation des GPO](23-performances.md) | Performance |
| 24 | [GPO et versions Windows : compatibilite, depreciations, parametres fantomes](24-versions.md) | Compatibilite |
| 25 | [GPO et MDM : convergence et coexistence](25-mdm-convergence.md) | Modern management |

!!! tip "Par ou commencer ?"
    Si vous avez deja deploye des GPO mais que vous manquez de profondeur, commencez par 02, 03, 07 et 20. Gardez 04, 06 et 09 a portee de main pendant vos depannages terrain : ce sont eux qui font gagner du temps.
