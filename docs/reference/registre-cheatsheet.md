---
description: "Reference rapide registre Windows : ruches, types, reg.exe, PowerShell, cles sensibles et format .reg."
tags:
  - registre
  - cheatsheet
  - reference
---

# Registre Windows - Aide-memoire de reference rapide

## Ruches principales

| Ruche | Fichier physique | Role |
|-------|------------------|------|
| `HKLM` | `%SystemRoot%\System32\Config\*` (`SYSTEM`, `SOFTWARE`, `SAM`, `SECURITY`, `DEFAULT`) | Configuration machine globale |
| `HKCU` | `%USERPROFILE%\NTUSER.DAT` + `%LOCALAPPDATA%\Microsoft\Windows\UsrClass.dat` | Configuration du profil utilisateur courant |
| `HKCR` | Vue fusionnee de `HKLM\Software\Classes` et `HKCU\Software\Classes` | Associations de fichiers, COM et classes |
| `HKU` | Profils charges a partir des `NTUSER.DAT` de chaque SID | Vue de tous les profils utilisateurs charges |
| `HKCC` | Alias runtime de `HKLM\SYSTEM\CurrentControlSet\Hardware Profiles\Current` | Profil materiel actif |

## Types de valeurs

| Type | Taille | Usage typique |
|------|--------|---------------|
| `REG_SZ` | Texte variable | Chaine simple, chemin fixe, nom affichable |
| `REG_DWORD` | 32 bits | Flags, booleens, compteurs, codes d'etat |
| `REG_BINARY` | Binaire variable | Donnees opaques, blobs applicatifs |
| `REG_MULTI_SZ` | Liste de chaines | Plusieurs lignes ou chemins dans une seule valeur |
| `REG_EXPAND_SZ` | Texte variable | Chaine avec variables d'environnement (`%SystemRoot%`) |
| `REG_QWORD` | 64 bits | Compteurs et tailles 64 bits |

## Commandes `reg.exe`

| Commande | Syntaxe minimale | Exemple |
|----------|------------------|---------|
| `query` | `reg query <cle> [/v Valeur]` | `reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v ProductName` |
| `add` | `reg add <cle> /v <nom> /t <type> /d <donnee> /f` | `reg add "HKCU\Software\MonApp" /v Enabled /t REG_DWORD /d 1 /f` |
| `delete` | `reg delete <cle> [/v Valeur] /f` | `reg delete "HKCU\Software\MonApp" /v Enabled /f` |
| `copy` | `reg copy <source> <destination> /s /f` | `reg copy "HKLM\SOFTWARE\Source" "HKLM\SOFTWARE\Dest" /s /f` |
| `compare` | `reg compare <cle1> <cle2> [/s]` | `reg compare "HKLM\SOFTWARE\AppA" "HKLM\SOFTWARE\AppB" /s` |
| `export` | `reg export <cle> <fichier.reg> /y` | `reg export "HKCU\Software\MonApp" C:\Temp\monapp.reg /y` |
| `import` | `reg import <fichier.reg>` | `reg import C:\Temp\baseline.reg` |
| `save` | `reg save <cle> <fichier.hiv> /y` | `reg save HKLM\SYSTEM C:\Temp\SYSTEM.hiv /y` |
| `restore` | `reg restore <cle> <fichier.hiv>` | `reg restore HKLM\TempHive C:\Temp\SOFTWARE.hiv` |
| `load` / `unload` | `reg load HKLM\<Nom> <fichier.hiv>` | `reg load HKLM\OfflineSOFT C:\Temp\SOFTWARE` |

## Cmdlets PowerShell registre

| Cmdlet | Usage en une ligne |
|--------|--------------------|
| `Get-ChildItem` | Liste les sous-cles d'une branche registre |
| `Get-Item` | Lit une cle et ses metadonnees |
| `Get-ItemProperty` | Lit toutes les valeurs d'une cle |
| `Get-ItemPropertyValue` | Lit une valeur precise |
| `Set-ItemProperty` | Modifie une valeur existante |
| `New-ItemProperty` | Cree une nouvelle valeur |
| `Remove-ItemProperty` | Supprime une valeur |
| `New-Item` | Cree une cle |
| `Remove-Item` | Supprime une cle |
| `Rename-ItemProperty` | Renomme une valeur |
| `Copy-Item` | Copie une branche registre |

## Cles de demarrage

| Zone | Chemin | Portee | Ordre / remarque |
|------|--------|--------|------------------|
| Run machine | `HKLM\Software\Microsoft\Windows\CurrentVersion\Run` | Tous les utilisateurs | Lance a chaque ouverture de session |
| Run utilisateur | `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` | Utilisateur courant | Lance a chaque ouverture de session de l'utilisateur |
| RunOnce machine | `HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce` | Tous les utilisateurs | Lance une seule fois puis supprime la valeur |
| RunOnce utilisateur | `HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce` | Utilisateur courant | Lance une fois a la prochaine session |
| Services | `HKLM\SYSTEM\CurrentControlSet\Services\<NomService>` | Machine | Demarrage selon `Start` (`2` auto, `3` manuel, `4` desactive) |
| Drivers | `HKLM\SYSTEM\CurrentControlSet\Services\<NomDriver>` | Machine | Charges tres tot selon `Type`, `Group` et `Start` |

## Cles de securite importantes

Valeurs recommandees hors exception metier explicite :

| Cle complete | Valeur recommandee | Pourquoi |
|--------------|--------------------|----------|
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\EnableLUA` | `1` | Garde l'UAC active |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\PromptOnSecureDesktop` | `1` | Isole les prompts UAC sur le bureau securise |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\DisableCAD` | `0` | Exige `Ctrl+Alt+Suppr` avant ouverture de session |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\LocalAccountTokenFilterPolicy` | `0` | Evite le contournement UAC en admin local distant |
| `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\LmCompatibilityLevel` | `5` | Force NTLMv2 et refuse LM/NTLM faibles |
| `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\NoLMHash` | `1` | Empeche le stockage du hash LM |
| `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\RestrictAnonymous` | `1` | Reduit l'enumeration anonyme |
| `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\RestrictAnonymousSAM` | `1` | Protege l'enumeration SAM anonyme |
| `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\RunAsPPL` | `1` | Protege `lsass.exe` quand supporte |
| `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\SCENoApplyLegacyAuditPolicy` | `1` | Force l'audit avance coherent avec `auditpol` |
| `HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest\UseLogonCredential` | `0` | Evite le stockage en clair pour WDigest |
| `HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated` | `0` | Evite l'elevation MSI dangereuse |
| `HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated` | `0` | Meme protection cote utilisateur |
| `HKLM\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters\SMB1` | `0` | Coupe SMBv1 sauf exception legacy documentee |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer\NoDriveTypeAutoRun` | `255` | Desactive AutoRun sur tous les medias |
| `HKLM\SYSTEM\CurrentControlSet\Control\Print\RpcAuthnLevelPrivacyEnabled` | `1` | Renforce RPC sur la pile d'impression |

!!! warning "Toujours valider dans le contexte"
    Une valeur "recommandee" n'est utile que si elle a ete testee sur le role cible. Sur un serveur legacy, une ligne SMB, impression ou LSA peut casser un composant vieux mais critique.

## Format de fichier `.reg`

### En-tete obligatoire

```reg
Windows Registry Editor Version 5.00
```

### Ajouter ou modifier une valeur

```reg
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\Contoso\App]
"ServerUrl"="https://intra.contoso.local"
"Enabled"=dword:00000001
"InstallPath"=hex(2):25,00,50,00,72,00,6f,00,67,00,72,00,61,00,6d,00,46,00,69,00,6c,00,65,00,73,00,25,00,5c,00,43,00,6f,00,6e,00,74,00,6f,00,73,00,6f,00,00,00
"Servers"=hex(7):73,00,72,00,76,00,31,00,00,00,73,00,72,00,76,00,32,00,00,00,00,00
"Counter64"=hex(b):01,00,00,00,00,00,00,00
```

### Supprimer une valeur

```reg
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Software\Contoso\App]
"DeprecatedValue"=-
```

### Supprimer une cle complete

```reg
Windows Registry Editor Version 5.00

[-HKEY_CURRENT_USER\Software\Contoso\OldApp]
```

### Rappel des prefixes de type

| Prefixe `.reg` | Type cible |
|----------------|-----------|
| `"Texte"` | `REG_SZ` |
| `dword:00000001` | `REG_DWORD` |
| `hex:` | `REG_BINARY` |
| `hex(2):` | `REG_EXPAND_SZ` |
| `hex(7):` | `REG_MULTI_SZ` |
| `hex(b):` | `REG_QWORD` |
