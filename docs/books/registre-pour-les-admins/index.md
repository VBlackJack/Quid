---
tags:
  - registre
  - administration
  - entreprise
---

# Le Registre pour les Administrateurs

> Cas concrets et cles de registre pour les roles serveur et applications d'entreprise.

## Presentation

Ce livre est un guide pratique destine aux administrateurs systemes en environnement professionnel. Chaque chapitre cible un role serveur ou une application tierce specifique, avec les cles de registre reelles, les commandes PowerShell associees et des scenarios de depannage issus du terrain.

Pas de theorie abstraite : chaque section part d'un **cas concret** (deploiement, durcissement, depannage) et fournit la solution registre complete.

!!! info "En resume"
    - Guide 100% pratique oriente cas concrets pour administrateurs systemes
    - Structure en trois parties : fondamentaux admin, roles serveur, applications tierces

---

## Sommaire

### Partie 1 — Fondamentaux Admin

| Chapitre | Sujet |
|:--------:|-------|
| 1 | [PowerShell Remoting et le registre a distance](01-powershell-remoting.md) |
| 2 | [GPO et registre en environnement AD](02-gpo-registre-ad.md) |
| 3 | [Audit et conformite du registre](03-audit-conformite.md) |
| 4 | [Deploiement et industrialisation](04-deploiement.md) |

### Partie 2 — Roles serveur

| Chapitre | Sujet |
|:--------:|-------|
| 5 | [Active Directory Domain Services](05-ad-ds.md) |
| 6 | [DNS Server](06-dns.md) |
| 7 | [DHCP Server](07-dhcp.md) |
| 8 | [WSUS](08-wsus.md) |
| 9 | [IIS / Web Server](09-iis.md) |
| 10 | [Remote Desktop Services](10-rds.md) |
| 11 | [Hyper-V](11-hyper-v.md) |
| 12 | [File Server / DFS](12-file-server.md) |
| 13 | [Print Server](13-print-server.md) |
| 14 | [RADIUS / NPS](14-radius-nps.md) |

### Partie 3 — Applications tierces

| Chapitre | Sujet |
|:--------:|-------|
| 15 | [GlobalProtect (Palo Alto)](15-globalprotect.md) |
| 16 | [SCCM / MECM (ConfigMgr)](16-sccm-mecm.md) |
| 17 | [Microsoft Intune / Endpoint Manager](17-intune.md) |
| 18 | [Antivirus enterprise](18-antivirus.md) |
| 19 | [Citrix Virtual Apps & Desktops](19-citrix.md) |
| 20 | [VMware Horizon / Workspace ONE](20-vmware.md) |
| 21 | [Microsoft 365 Apps (Office)](21-m365-office.md) |
| 22 | [Navigateurs enterprise](22-navigateurs.md) |
| 23 | [VPN clients](23-vpn-clients.md) |
| 24 | [Backup & monitoring](24-backup-monitoring.md) |

### Partie 4 — Serveurs applicatifs et conformite

| Chapitre | Sujet |
|:--------:|-------|
| 25 | [Exchange Server](25-exchange.md) |
| 26 | [SQL Server](26-sql-server.md) |
| 27 | [SharePoint Server](27-sharepoint.md) |
| 28 | [Adobe et Java](28-adobe-java.md) |
| 29 | [Compliance (HIPAA, PCI-DSS, SOC 2)](29-compliance.md) |
| 30 | [Zero Trust et VDI](30-zero-trust.md) |

!!! info "En resume"
    - Les chapitres 1 a 4 posent les fondamentaux : PowerShell remoting, GPO, audit et deploiement industrialise
    - Les chapitres 5 a 14 couvrent les roles serveur Windows les plus courants en entreprise
    - Les chapitres 15 a 24 traitent des applications tierces et outils de gestion
    - Les chapitres 25 a 30 couvrent les serveurs applicatifs Microsoft, les applications tierces courantes et la conformite

---

## Public vise

Ce livre s'adresse aux administrateurs systemes, ingenieurs infrastructure et techniciens de support niveau 2/3 qui gerent des parcs Windows en entreprise. Une connaissance de base du registre est recommandee — pour les fondamentaux, consulter [La Bible de la Base de Registre](../bible-registre-windows/index.md) ou [La Base de Registre pour les Nuls](../registre-pour-les-nuls/index.md).

!!! info "En resume"
    - Destine aux admins systemes, ingenieurs infra et support N2/N3 en environnement entreprise
    - Prerequis : connaissance de base du registre Windows (voir les deux autres livres)
