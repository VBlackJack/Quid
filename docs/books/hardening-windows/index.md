---
description: "Hardening Windows par le registre et les GPO : baselines CIS/ANSSI, credential protection, ASR, AppLocker, WDAC, Firewall, BitLocker, LAPS et maintien dans le temps en 20 chapitres."
tags:
  - hardening
  - securite
  - windows
  - gpo
  - registre
---

# Hardening Windows — Registre & GPO

Durcir un poste Windows ne s'improvise pas.
Ce livre relie les clés de registre, les GPO et les baselines de sécurité en une approche cohérente, testée sur le terrain.

---

## A qui s'adresse ce livre ?

Administrateurs systèmes, ingénieurs sécurité et équipes SOC qui veulent aller au-delà de "cocher les cases CIS" et comprendre **pourquoi** chaque paramètre existe, où il s'écrit dans le registre et comment le déployer via GPO.

Prérequis utiles : connaître Active Directory, manipuler Regedit sans stress, avoir déjà créé une GPO.

## Ce que vous allez maîtriser

- Lire et appliquer une baseline CIS, STIG ou ANSSI sans se perdre
- Protéger les credentials (LSASS, WDigest, Credential Guard)
- Durcir SMB, RDP, le réseau et le firewall
- Déployer AppLocker, WDAC et les règles ASR
- Gérer LAPS, BitLocker et Windows Hello for Business par GPO
- Construire une checklist de durcissement par niveau (workstation / serveur / DC)
- Auditer la dérive de configuration dans le temps

## Parcours de lecture suggéré

### Démarrage rapide (posture de base)

1. [01. Philosophie du hardening](01-philosophie.md)
2. [02. Baselines](02-baselines.md)
3. [03. Credential protection](03-credential-protection.md)
4. [18. Checklist par niveau](18-checklist-niveaux.md)

### Lecture ciblée par besoin

| Je veux... | Aller au chapitre |
|------------|-------------------|
| Comprendre les baselines CIS/ANSSI/STIG | [02 - Baselines](02-baselines.md) |
| Protéger LSASS et les mots de passe | [03 - Credential protection](03-credential-protection.md) |
| Durcir SMB et les partages | [05 - SMB hardening](05-smb-hardening.md) |
| Sécuriser RDP | [06 - RDP hardening](06-rdp-hardening.md) |
| Contrôler les applications | [07 - AppLocker / WDAC](07-applocker-wdac.md) |
| Réduire la surface d'attaque Defender | [08 - ASR rules](08-asr-rules.md) |
| Configurer les audits et les logs | [09 - Audit & Event Log](09-audit-eventlog.md) |
| Gérer les comptes locaux et LAPS | [11 - LAPS & comptes locaux](11-laps-comptes-locaux.md) |
| Déployer BitLocker | [12 - BitLocker](12-bitlocker.md) |
| Avoir une checklist prête à l'emploi | [18 - Checklist par niveau](18-checklist-niveaux.md) |
| Surveiller la dérive dans le temps | [20 - Maintien dans le temps](20-maintien-temps.md) |

## Tous les chapitres

| # | Chapitre | Thème |
|---|----------|-------|
| 01 | [Philosophie du hardening](01-philosophie.md) | Fondations |
| 02 | [Baselines : CIS, STIG, ANSSI, Microsoft](02-baselines.md) | Référentiels |
| 03 | [Credential protection](03-credential-protection.md) | Identités |
| 04 | [UAC et privilèges](04-uac-privileges.md) | Élévation |
| 05 | [SMB hardening](05-smb-hardening.md) | Réseau |
| 06 | [RDP hardening](06-rdp-hardening.md) | Accès distant |
| 07 | [AppLocker et WDAC](07-applocker-wdac.md) | Contrôle applicatif |
| 08 | [Attack Surface Reduction](08-asr-rules.md) | Defender |
| 09 | [Audit policies et Event Log](09-audit-eventlog.md) | Visibilité |
| 10 | [PowerShell Constrained Language Mode](10-powershell-clm.md) | Scripts |
| 11 | [LAPS et comptes locaux](11-laps-comptes-locaux.md) | Comptes |
| 12 | [BitLocker via GPO/registre](12-bitlocker.md) | Chiffrement |
| 13 | [Windows Firewall](13-firewall.md) | Filtrage réseau |
| 14 | [Windows Hello for Business](14-whfb.md) | Authentification |
| 15 | [Secured-Core PC](15-secured-core.md) | Firmware |
| 16 | [Defender et antivirus via GPO](16-defender-antivirus.md) | EDR |
| 17 | [Réseau : LLMNR, NBT-NS, DoH](17-reseau-llmnr.md) | Protocoles |
| 18 | [Checklist hardening par niveau](18-checklist-niveaux.md) | Référence terrain |
| 19 | [Audit de conformité et RSoP](19-audit-conformite.md) | Conformité |
| 20 | [Maintien dans le temps](20-maintien-temps.md) | Durabilité |

!!! tip "Par où commencer ?"
    Lisez 01 et 02 pour cadrer la démarche, puis allez directement au chapitre qui correspond à votre risque prioritaire. La checklist du chapitre 18 est un bon point d'entrée si vous avez un audit imminent.
