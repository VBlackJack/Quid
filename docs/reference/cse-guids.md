---
description: "Table complete des GUIDs des Client-Side Extensions (CSE) pour les strategies de groupe Windows."
tags:
  - gpo
  - cse
  - reference
---

# GUIDs des Client-Side Extensions (CSE)

| GUID | Nom | DLL | Description | Chapitres |
|------|-----|-----|-------------|-----------|
| `{35378EAC-683F-11D2-A89A-00C04FBBCFA2}` | Registry | `userenv.dll` | Administrative Templates, registre via `registry.pol` | [Bible GPO ch.03](../books/bible-gpo/03-cse.md), [Bible Registre ch.03](../books/bible-registre-windows/03-ruches.md) |
| `{827D319E-6EAC-11D2-A4EA-00C04F79F83A}` | Security | `scecli.dll` | Strategies de securite, `GptTmpl.inf` | [Bible GPO ch.13](../books/bible-gpo/13-securite-strategies.md) |
| `{42B5FAAE-6536-11D2-AE5A-0000F87571E3}` | Scripts | `gpscript.dll` | Scripts de demarrage/arret et logon/logoff | [Bible GPO ch.18](../books/bible-gpo/18-scripts.md) |
| `{25537BA6-77A8-11D2-9B6C-0000F8080861}` | Folder Redirection | `fdeploy.dll` | Redirection de dossiers | [Bible GPO ch.19](../books/bible-gpo/19-redirection-profils.md) |
| `{C6DC5466-785A-11D2-84D0-00C04FB169F7}` | Software Installation (Machine) | `appmgmts.dll` | Deploiement MSI cote machine | [Bible GPO ch.17](../books/bible-gpo/17-deploiement-msi.md) |
| `{3610EDA5-77EF-11D2-8DC5-00C04FA31A66}` | Software Installation (User) | `appmgmts.dll` | Deploiement MSI cote utilisateur | [Bible GPO ch.17](../books/bible-gpo/17-deploiement-msi.md) |
| `{0ACDD3F5-75AC-47AB-BAA0-BF6DE7E7FE63}` | 802.11 Wireless | `gptext.dll` | Politiques Wi-Fi | [GPO Admins ch.16](../books/gpo-pour-les-admins/16-wifi-vpn.md) |
| `{B587E2B1-4D59-4E7E-AED9-22B9DF11D053}` | 802.3 Wired | `dot3gpclnt.dll` | Politiques filaires 802.1X | [GPO Admins ch.16](../books/gpo-pour-les-admins/16-wifi-vpn.md) |
| `{0F6B957D-509E-11D1-A7CC-0000F87571E3}` | EFS Recovery | `gptext.dll` | Recuperation EFS | [Bible GPO ch.13](../books/bible-gpo/13-securite-strategies.md) |
| `{25537523-E2C2-11D2-8DC5-00C04FA31A66}` | Microsoft Disk Quota | `dskquota.dll` | Quotas disque | [Bible GPO ch.03](../books/bible-gpo/03-cse.md) |
| `{426031C0-0B47-4852-B0CA-AC3D37BFCB39}` | QoS Packet Scheduler | `gptext.dll` | QoS basee sur strategie | [Bible GPO ch.03](../books/bible-gpo/03-cse.md) |
| `{4CFB60C1-FAA6-47F1-89AA-0B18730C9FD3}` | Internet Explorer Maintenance | `gptext.dll` | Parametres IE legacy, obsolete | [Bible GPO ch.24](../books/bible-gpo/24-versions.md) |
| `{A2E30F80-D7DE-11D2-BBDE-00C04F86AE3B}` | Audit Policy | `auditcse.dll` | Strategies d'audit classiques | [Bible GPO ch.13](../books/bible-gpo/13-securite-strategies.md) |
| `{FC491EF1-C4AA-4CE1-B329-414B101DB823}` | Advanced Audit Policy | `auditcse.dll` | Audit avance et SACL | [Hardening ch.09](../books/hardening-windows/09-audit-eventlog.md) |
| `{0E28E245-9368-4853-AD84-6DA3BA35BB75}` | Group Policy Preferences | `gpprefcl.dll` | Preferences de strategie de groupe | [GPO Nuls ch.10](../books/gpo-pour-les-nuls/10-preferences.md) |
| `{169EBF44-942F-4C43-87CE-13C93996EBBE}` | Group Policy Preferences - Computer | `gpprefcl.dll` | Preferences cote machine | [GPO Nuls ch.10](../books/gpo-pour-les-nuls/10-preferences.md) |
| `{AADCED64-746C-4633-A97C-D61349046527}` | Group Policy Preferences - User | `gpprefcl.dll` | Preferences cote utilisateur | [GPO Nuls ch.10](../books/gpo-pour-les-nuls/10-preferences.md) |
| `{F3CCC681-B74C-4060-9F26-CD84525DCA2A}` | Folder Redirection (legacy) | `fdeploy.dll` | Redirection de dossiers legacy | [Bible GPO ch.19](../books/bible-gpo/19-redirection-profils.md) |
| `{C631DF4C-088F-4156-B058-4375F0DB8C21}` | Microsoft Offline Files | `cscobj.dll` | Fichiers hors connexion | [Bible GPO ch.19](../books/bible-gpo/19-redirection-profils.md) |
| `{3060E8CE-7020-11D2-842D-00C04FA372D4}` | Remote Installation Services | `rigpsnap.dll` | RIS, obsolete | [Bible GPO ch.24](../books/bible-gpo/24-versions.md) |
| `{E437BC1C-AA7D-11D2-A382-00C04F991E27}` | IP Security | `polstore.dll` | Politiques IPSec | [Bible GPO ch.13](../books/bible-gpo/13-securite-strategies.md) |

!!! tip "Ou les voir ?"
    Les GUIDs CSE apparaissent dans les attributs `gPCMachineExtensionNames` et `gPCUserExtensionNames` de l'objet GPO dans AD. Consultez [la Bible GPO - CSE](../books/bible-gpo/03-cse.md) pour le detail du fonctionnement.

!!! warning "Windows LAPS"
    Windows LAPS moderne s'appuie sur ses composants Windows et ses journaux dedies ; ne confondez pas ses parametres avec les GUIDs Group Policy Preferences ci-dessus.
