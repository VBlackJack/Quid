---
description: "Boite a outils des consoles et commandes Windows utiles autour des GPO et du registre."
tags:
  - outils
  - gpo
  - registre
  - windows
---

# Boite a outils - Outils Windows essentiels

| Outil | Chemin / commande d'acces | Usage principal | Flags utiles | Disponibilite |
|------|----------------------------|-----------------|--------------|---------------|
| `gpedit.msc` | `gpedit.msc` | Editer les strategies locales | Aucun flag notable | Windows Pro / Enterprise / Education, pas Home |
| `gpmc.msc` | `gpmc.msc` | Gerer les GPO de domaine | Aucun flag notable | Windows Server, ou client avec RSAT |
| `rsop.msc` | `rsop.msc` | Visualiser le Resultant Set of Policy local | Aucun flag notable | Client et Server joints au domaine ou avec politique locale |
| `gpupdate` | `gpupdate [/force] [/target:{computer|user}]` | Rafraichir les strategies | `/force`, `/wait:0`, `/boot`, `/logoff` | Tous Windows |
| `gpresult` | `gpresult /r` | Voir les GPO appliquees / refusees | `/scope`, `/h`, `/z` | Tous Windows |
| `secedit` | `secedit /analyze /db <db>` | Comparer, exporter ou reappliquer la politique de securite | `/analyze`, `/configure`, `/export` | Tous Windows |
| `regedit` | `regedit.exe` | Explorer et modifier le registre en GUI | `/m` (nouvelle instance) | Tous Windows |
| `reg.exe` | `reg query ...` | Lire / ecrire le registre en CLI | `query`, `add`, `delete`, `export`, `load` | Tous Windows |
| `regedt32` | `regedt32.exe` | Alias historique de Regedit | Aucun flag utile moderne | Tous Windows, compatibilite legacy |
| `auditpol` | `auditpol /get /category:*` | Lire / definir l'audit avance | `/get`, `/set`, `/backup`, `/restore` | Tous Windows modernes |
| `secpol.msc` | `secpol.msc` | Administrer la politique de securite locale | Aucun flag notable | Pro / Enterprise / Education, Server |
| `lgpo.exe` | `LGPO.exe /parse /m <Registry.pol>` | Import / export / analyse de politique locale | `/parse`, `/b`, `/g`, `/r`, `/w` | Security Compliance Toolkit, pas natif |
| `manage-bde` | `manage-bde -status` | Gerer BitLocker en ligne de commande | `-status`, `-protectors`, `-on`, `-off` | Editions avec BitLocker |
| `certutil` | `certutil -store my` | PKI, certificats, chaines et enrollement | `-store`, `-verify`, `-pulse`, `-dump` | Tous Windows |
| `dfsrdiag` | `dfsrdiag backlog /rgname:<RG>` | Diagnostiquer DFS-R et SYSVOL | `backlog`, `pollad`, `replstate` | Windows Server |
| `netsh advfirewall` | `netsh advfirewall firewall show rule name=all` | Lire / modifier Windows Firewall | `show`, `set`, `add rule`, `delete rule` | Tous Windows |
| `dsregcmd` | `dsregcmd /status` | Diagnostiquer Entra join / Hybrid join | `/status`, `/debug` | Windows 10/11, Server recent |
| `mdmdiagnosticstool` | `mdmdiagnosticstool.exe -area DeviceEnrollment;DeviceProvisioning` | Collecter les traces MDM | `-area`, `-cab`, `-zip` | Windows 10/11 modernes |

## Repere rapide

- GUI policy : `gpedit.msc`, `gpmc.msc`, `rsop.msc`, `secpol.msc`
- CLI GPO : `gpupdate`, `gpresult`, `secedit`, `lgpo.exe`
- CLI registre : `reg.exe`, PowerShell
- Securite : `auditpol`, `manage-bde`, `certutil`, `netsh advfirewall`
- Cloud / hybrid : `dsregcmd`, `mdmdiagnosticstool`

!!! tip "Choisir vite"
    Si vous devez prouver un etat, preferez `gpresult`, `Get-WinEvent`, `auditpol` et `reg query`. Si vous devez modifier, passez par GPMC, `gpedit.msc`, `reg.exe` ou PowerShell selon le perimetre.
