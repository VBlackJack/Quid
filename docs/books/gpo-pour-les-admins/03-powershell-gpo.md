---
description: "Automatiser et auditer les GPO avec le module PowerShell GroupPolicy : cmdlets essentiels, scripts de production, inventaire complet et pieges a eviter."
tags:
  - gpo
  - administration
  - powershell
  - automatisation
---
# PowerShell GroupPolicy module

!!! abstract "Ce que vous allez pouvoir faire"
    - Installer le module `GroupPolicy` sur un poste Windows 10/11 ou Windows Server et verifier sa disponibilite en 30 secondes
    - Creer, lier, configurer et supprimer des GPO entierement depuis la ligne de commande, sans ouvrir GPMC une seule fois
    - Auditer un parc de GPO existant et exporter le rapport complet en CSV, pret pour Excel
    - Detecter les GPO avec des parametres de registre "invisibles" dans GPMC (Extra Registry Settings) avant qu'un audit de conformite les signale
    - Comparer les parametres de deux GPO et identifier les differences en une commande


!!! tip "Si vous ne retenez qu'une chose"
    PowerShell devient rentable dès que vous devez inventorier, comparer ou corriger des GPO à grande échelle de façon répétable.

---

## Installation et prerequis

### Ou se trouve le module

Le module `GroupPolicy` est fourni avec les **Remote Server Administration Tools (RSAT)**. Il n'est pas installe par defaut sur les postes de travail.

Sur Windows Server, il est installe automatiquement avec la fonctionnalite GPMC. Sur Windows 10 et 11, il faut l'ajouter via les fonctionnalites a la demande.

### Installation sur Windows 10 et 11

```powershell
# Install the GroupPolicy management RSAT component
Add-WindowsCapability -Online -Name Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0
```

L'installation necessite une connexion internet (ou un point de distribution WSUS/WU configuré). Pas de redemarrage requis.

```powershell
# Verify the installation succeeded
Get-WindowsCapability -Online -Name Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0 |
    Select-Object Name, State
```

```title="Resultat attendu"
Name  : Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0
State : Installed
```

### Installation sur Windows Server

```powershell
# Install GPMC feature on Windows Server (including Server Core)
Install-WindowsFeature -Name GPMC

# Verify
Get-WindowsFeature -Name GPMC | Select-Object Name, InstallState
```

```title="Resultat attendu"
Name  InstallState
----  ------------
GPMC  Installed
```

:material-information-outline: Le module fonctionne egalement sur Server Core. L'absence d'interface graphique n'est pas un obstacle — c'est meme l'usage prevu pour les automatisations.

### Verifier et importer le module

```powershell
# Check if the module is available (without loading it)
Get-Module -Name GroupPolicy -ListAvailable
```

```title="Resultat attendu"
ModuleType Version    PreRelease Name                                ExportedCommands
---------- -------    ---------- ----                                ----------------
Manifest   1.0.0.0               GroupPolicy                         {Backup-GPO, Copy-GPO...}
```

```powershell
# Import explicitly if not auto-loaded (rare but can be needed in some environments)
Import-Module GroupPolicy

# List all available cmdlets organized by Noun then Verb
Get-Command -Module GroupPolicy | Sort-Object Noun, Verb | Format-Table -AutoSize
```

!!! warning "A surveiller — Contexte de domaine"
    Tous les cmdlets du module `GroupPolicy` necessitent un acces au controleur de domaine. Ils s'executent dans le contexte de la session Windows courante. Si vous etes sur un poste en workgroup ou non joint au domaine cible, utilisez le parametre `-Domain` et `-Server` sur chaque cmdlet, ou etablissez d'abord une session `Enter-PSSession` vers un controleur du domaine.

!!! quote "En resume"
    Sur Windows 10/11 : `Add-WindowsCapability`. Sur Windows Server : `Install-WindowsFeature -Name GPMC`. Verifiez avec `Get-Module -Name GroupPolicy -ListAvailable`. Le module fonctionne sur Server Core. Toute commande necessite un acces DC actif.

---

## :material-table-of-contents: Les 25 cmdlets essentiels : reference groupee

Les cmdlets du module `GroupPolicy` sont ici organises par scenario d'usage, pas par ordre alphabetique. L'ordre alphabetique est utile pour la reference. L'ordre par scenario est utile pour le travail quotidien.

### Creation et configuration

| Cmdlet | Usage typique |
|--------|---------------|
| `New-GPO` | Creer une GPO vide avec nom, commentaire et domaine |
| `New-GPLink` | Lier une GPO a une OU, un site ou le domaine |
| `Set-GPLink` | Modifier un lien existant : activer, desactiver, enforced, ordre |
| `Remove-GPLink` | Supprimer un lien sans supprimer la GPO |
| `Remove-GPO` | Supprimer une GPO definitvement (irreversible) |
| `Set-GPRegistryValue` | Ecrire une valeur directement dans `registry.pol` |
| `Remove-GPRegistryValue` | Supprimer une valeur de `registry.pol` |
| `Set-GPPermissions` | Definir les permissions (lecture, modification, application) |

### Lecture et rapport

| Cmdlet | Usage typique |
|--------|---------------|
| `Get-GPO` | Recuperer une GPO par nom ou GUID, ou lister toutes les GPO |
| `Get-GPOReport` | Generer un rapport XML ou HTML complet d'une GPO |
| `Get-GPRegistryValue` | Lire une valeur specifique dans `registry.pol` |
| `Get-GPPermissions` | Lire les permissions d'une GPO |
| `Get-GPInheritance` | Voir les GPO heritees et les liens actifs sur une OU |
| `Get-GPResultantSetOfPolicy` | Calculer le RSoP simule pour un ordinateur ou un utilisateur |

### Sauvegarde et restauration

| Cmdlet | Usage typique |
|--------|---------------|
| `Backup-GPO` | Sauvegarder une ou toutes les GPO vers un dossier |
| `Restore-GPO` | Restaurer une GPO depuis une sauvegarde |
| `Copy-GPO` | Copier une GPO dans le meme domaine |
| `Import-GPO` | Importer les parametres depuis une sauvegarde (inter-domaine possible) |
| `New-GPStarterGPO` | Creer un modele de demarrage (Starter GPO) |
| `Get-GPStarterGPO` | Lister les modeles de demarrage disponibles |

### Diagnostic et rafraichissement

| Cmdlet | Usage typique |
|--------|---------------|
| `Invoke-GPUpdate` | Forcer `gpupdate /force` sur un ou plusieurs postes distants |
| `Get-GPResultantSetOfPolicy` | Simuler le RSoP sans appliquer les parametres |

:material-lightbulb-outline: `Rename-GPO` n'est pas dans le module `GroupPolicy`. Le renommage passe par `Set-GPO -Name "ancien" -TargetName "nouveau"` — ou directement via `Rename-GPO` si vous avez la version Server 2019+ du module.

!!! quote "En resume"
    25 cmdlets couvrent la quasi-totalite des operations GPO. Les groupes "creation", "lecture" et "sauvegarde" representent 90 % des usages quotidiens. `Invoke-GPUpdate` et `Get-GPResultantSetOfPolicy` sont les outils de diagnostic immediat.

---

## :material-plus-box-outline: Patterns courants : creation et liaison

### Creer une GPO et la lier en une seule sequence

L'operation la plus courante en production : creer une GPO nommee, commentee, et la lier immediatement a l'OU cible.

Le commentaire est essentiel. Il doit contenir au minimum : l'auteur, la date, et l'objet.

```powershell
# Create a new GPO with a descriptive comment, then link it immediately
$gpo = New-GPO -Name "CFG-Postes-EnvironnementBureau" `
               -Comment "Bureau et ecran de veille — cree par J.Dupont le $(Get-Date -Format 'yyyy-MM-dd')"

New-GPLink -Guid $gpo.Id `
           -Target "OU=Postes-Standard,DC=contoso,DC=local" `
           -LinkEnabled Yes `
           -Order 3

Write-Host "GPO creee et liee : $($gpo.DisplayName)" -ForegroundColor Green
```

```title="Resultat attendu"
GPO creee et liee : CFG-Postes-EnvironnementBureau
```

### Activer, desactiver ou enforcer un lien existant

```powershell
# Disable a link without removing it (useful during testing)
Set-GPLink -Name "CFG-Postes-EnvironnementBureau" `
           -Target "OU=Postes-Standard,DC=contoso,DC=local" `
           -LinkEnabled No

# Re-enable it
Set-GPLink -Name "CFG-Postes-EnvironnementBureau" `
           -Target "OU=Postes-Standard,DC=contoso,DC=local" `
           -LinkEnabled Yes

# Mark the link as Enforced (use sparingly — see Chapter 01)
Set-GPLink -Name "SEC-Postes-Baseline" `
           -Target "OU=Postes-Standard,DC=contoso,DC=local" `
           -Enforced Yes
```

### Ecrire une valeur de registre dans une GPO

`Set-GPRegistryValue` ecrit directement dans le fichier `registry.pol` de la GPO. Pas besoin d'ouvrir GPMC, pas besoin d'un template ADMX.

```powershell
# Write a registry value into the Computer Configuration side of a GPO
Set-GPRegistryValue -Name "CFG-Postes-EnvironnementBureau" `
    -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows\Personalization" `
    -ValueName "LockScreenImage" `
    -Type String `
    -Value "\\serveur\images\lockscreen.jpg"
```

```title="Resultat attendu"
KeyPath     : SOFTWARE\Policies\Microsoft\Windows\Personalization
FullKeyPath : HKLM\SOFTWARE\Policies\Microsoft\Windows\Personalization
ValueName   : LockScreenImage
Value       : \\serveur\images\lockscreen.jpg
Type        : String
HasValue    : True
```

Le prefixe de la cle determine si le parametre s'applique cote ordinateur ou cote utilisateur :

- `HKLM\...` → `Computer Configuration`
- `HKCU\...` → `User Configuration`

### Desactiver la section inutilisee

Une GPO qui ne configure que les ordinateurs ne doit pas avoir la section utilisateur active. Desactiver la section vide reduit les requetes SYSVOL a chaque logon.

```powershell
# Disable the User Configuration section (GPO only has Computer settings)
Set-GPO -Name "CFG-Postes-EnvironnementBureau" -GpoStatus ComputerSettingsEnabled

# Or disable the Computer Configuration section
Set-GPO -Name "USR-Standard-Bureau" -GpoStatus UserSettingsEnabled

# Check the current status
Get-GPO -Name "CFG-Postes-EnvironnementBureau" | Select-Object DisplayName, GpoStatus
```

```title="Resultat attendu"
DisplayName                    GpoStatus
-----------                    ---------
CFG-Postes-EnvironnementBureau ComputerSettingsEnabled
```

!!! warning "A surveiller — GpoStatus vs LinkEnabled"
    `GpoStatus = AllSettingsDisabled` desactive la GPO entiere, pour tous ses liens. `Set-GPLink -LinkEnabled No` desactive uniquement le lien vers une OU specifique. Ce sont deux niveaux de desactivation differents, avec des impacts tres differents. Confondre les deux est une erreur classique.

!!! quote "En resume"
    Creez avec `New-GPO`, liez avec `New-GPLink`, configurez le registre avec `Set-GPRegistryValue`. Desactivez toujours la section inutilisee avec `Set-GPO -GpoStatus`. Ne confondez pas `GpoStatus = AllSettingsDisabled` et `Set-GPLink -LinkEnabled No`.

---

## :material-magnify: Patterns courants : lecture et audit

### Recuperer une GPO specifique

```powershell
# Get a GPO by name
Get-GPO -Name "CFG-Postes-EnvironnementBureau"

# Get a GPO by GUID
Get-GPO -Guid "{a1b2c3d4-e5f6-7890-abcd-ef1234567890}"

# Get all GPOs in the domain
Get-GPO -All | Select-Object DisplayName, GpoStatus, ModificationTime | Sort-Object DisplayName
```

### Lire une valeur de registre dans une GPO

```powershell
# Read a specific registry value from a GPO's registry.pol
Get-GPRegistryValue -Name "CFG-Postes-EnvironnementBureau" `
    -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows\Personalization" `
    -ValueName "LockScreenImage"
```

```title="Resultat attendu"
KeyPath     : SOFTWARE\Policies\Microsoft\Windows\Personalization
FullKeyPath : HKLM\SOFTWARE\Policies\Microsoft\Windows\Personalization
ValueName   : LockScreenImage
Value       : \\serveur\images\lockscreen.jpg
Type        : String
HasValue    : True
```

### Inventaire complet avec liens et versions

Ce script produit un tableau exploitable en 10 secondes sur un parc de 200 GPO. C'est le point de depart de tout audit.

```powershell
# Full GPO inventory with link targets, versions and modification dates
Get-GPO -All | ForEach-Object {
    $gpo = $_
    $xml  = [xml](Get-GPOReport -Guid $gpo.Id -ReportType Xml)
    $links = $xml.GPO.LinksTo | ForEach-Object { $_.SOMPath }

    [PSCustomObject]@{
        Name        = $gpo.DisplayName
        Status      = $gpo.GpoStatus
        Modified    = $gpo.ModificationTime.ToString("yyyy-MM-dd")
        ComputerVer = $gpo.Computer.DSVersion
        UserVer     = $gpo.User.DSVersion
        Links       = if ($links) { ($links -join " | ") } else { "UNLINKED" }
        Description = $gpo.Description
    }
} | Sort-Object Name | Format-Table -AutoSize -Wrap
```

:material-information-outline: `DSVersion` est le numero de version de la GPO cote SYSVOL. Si `DSVersion` est 0, la section correspondante n'a jamais ete configuree.

### Lire les permissions d'une GPO

```powershell
# List all permissions on a specific GPO
Get-GPPermissions -Name "SEC-Postes-Baseline" -All |
    Select-Object @{N="Trustee"; E={$_.Trustee.Name}},
                  @{N="Type";    E={$_.Trustee.SidType}},
                  Permission |
    Format-Table -AutoSize
```

```title="Resultat attendu"
Trustee                  Type  Permission
-------                  ----  ----------
Domain Admins            Group GpoEditDeleteModifySecurity
Enterprise Admins        Group GpoEditDeleteModifySecurity
Authenticated Users      Group GpoApply
SYSTEM                   WellKnownGroup GpoEdit
```

### Voir l'heritage des GPO sur une OU

```powershell
# Show all GPOs inherited by an OU (from domain root down)
Get-GPInheritance -Target "OU=Postes-Standard,DC=contoso,DC=local" |
    Select-Object -ExpandProperty GpoLinks |
    Select-Object DisplayName, Enabled, Enforced, Order |
    Sort-Object Order | Format-Table -AutoSize
```

```title="Resultat attendu"
DisplayName                    Enabled Enforced Order
-----------                    ------- -------- -----
SEC-Postes-Baseline            True    False    1
CFG-Postes-WUfB-Ring2          True    False    2
CFG-Postes-EnvironnementBureau True    False    3
APP-Edge-ConfigEntreprise      True    False    4
```

### Generer un rapport HTML pour une GPO

```powershell
# Generate a standalone HTML report for a GPO
Get-GPOReport -Name "SEC-Postes-Baseline" `
              -ReportType Html `
              -Path "C:\Temp\SEC-Postes-Baseline-$(Get-Date -Format 'yyyyMMdd').html"

Write-Host "Report generated: C:\Temp\SEC-Postes-Baseline-$(Get-Date -Format 'yyyyMMdd').html"
```

Le rapport HTML peut etre ouvert dans n'importe quel navigateur. Il est identique a ce qu'affiche GPMC dans l'onglet "Settings". Pratique pour envoyer a un responsable securite sans lui donner acces a GPMC.

!!! quote "En resume"
    `Get-GPO -All` est la commande de depart de tout audit. `Get-GPOReport -ReportType Xml` donne acces a tous les parametres configures via XPath. `Get-GPInheritance` montre exactement ce qui s'applique a une OU. `Get-GPRegistryValue` lit le contenu de `registry.pol` sans ouvrir la GPO.

---

## :material-content-copy: Patterns courants : sauvegarde et restauration

### Sauvegarder toutes les GPO

La sauvegarde inclut les parametres, les permissions, les filtres WMI et les liens — mais pas les cibles de liens (les OU elles-memes ne sont pas sauvegardees).

```powershell
# Backup all GPOs to a timestamped folder
$backupPath = "\\serveur-backup\GPO-Backups\$(Get-Date -Format 'yyyy-MM-dd')"
New-Item -ItemType Directory -Path $backupPath -Force | Out-Null

Backup-GPO -All -Path $backupPath

Write-Host "Backup completed to: $backupPath" -ForegroundColor Green
```

```title="Resultat attendu"
DisplayName                : CFG-Postes-EnvironnementBureau
GpoId                      : a1b2c3d4-e5f6-7890-abcd-ef1234567890
Id                         : {c8d9e0f1-a2b3-4c5d-6e7f-890abc123456}
BackupDirectory            : \\serveur-backup\GPO-Backups\2026-04-05
CreationTime               : 05/04/2026 09:00:00
DomainName                 : contoso.local
Comment                    :
```

Un sous-dossier GUID est cree par GPO dans le dossier de sauvegarde. La structure est compatible avec `Restore-GPO` et `Import-GPO`.

### Sauvegarder une GPO specifique avant modification

Bonne pratique : toujours sauvegarder une GPO avant de la modifier, meme pour un changement mineur.

```powershell
# Backup a single GPO before making changes
$gpoName    = "SEC-Postes-Baseline"
$backupPath = "C:\Temp\GPO-Backup-Pre-Change"
New-Item -ItemType Directory -Path $backupPath -Force | Out-Null

$backup = Backup-GPO -Name $gpoName -Path $backupPath
Write-Host "Backup ID: $($backup.Id) — Path: $backupPath" -ForegroundColor Yellow
```

### Restaurer une GPO depuis une sauvegarde

```powershell
# Restore a GPO from its latest backup (replaces current settings)
Restore-GPO -Name "SEC-Postes-Baseline" -Path "\\serveur-backup\GPO-Backups\2026-04-05"
```

```title="Resultat attendu"
DisplayName      : SEC-Postes-Baseline
DomainName       : contoso.local
Owner            : CONTOSO\Domain Admins
Id               : {a1b2c3d4-...}
GpoStatus        : AllSettingsEnabled
ModificationTime : 05/04/2026 09:15:00
```

### Importer des parametres depuis une sauvegarde (sans ecraser les metadonnees)

`Import-GPO` est different de `Restore-GPO`. Il importe uniquement les parametres de configuration, sans ecraser les permissions ni les liens de la GPO cible.

Utile pour : appliquer les parametres d'une GPO de reference a une GPO existante, sans perdre les ACL ni les liaisons.

```powershell
# Import settings from a backup into an existing GPO (preserves links and permissions)
Import-GPO -BackupId "{c8d9e0f1-a2b3-4c5d-6e7f-890abc123456}" `
           -TargetName "CFG-Postes-EnvironnementBureau" `
           -Path "\\serveur-backup\GPO-Backups\2026-04-05"
```

### Copier une GPO dans le meme domaine

```powershell
# Copy a GPO — creates a new GPO with the same settings but no links
Copy-GPO -SourceName "SEC-Postes-Baseline" `
         -TargetName "SEC-Postes-Baseline-TEST" `
         -CopyAcls
```

`-CopyAcls` copie egalement les permissions. Sans ce parametre, la GPO copiee herite des permissions par defaut du domaine.

!!! warning "A surveiller — Copy-GPO ne copie pas les liens"
    `Copy-GPO` cree une nouvelle GPO avec les memes parametres, mais sans aucune liaison. Vous devez lier la GPO manuellement avec `New-GPLink` apres la copie. C'est voulu : une GPO TEST ne doit pas s'appliquer automatiquement aux memes OU que la GPO de production.

!!! quote "En resume"
    `Backup-GPO` avant toute modification est une regle non-negociable. `Restore-GPO` ecrase tout (parametres, permissions). `Import-GPO` importe uniquement les parametres. `Copy-GPO` cree une nouvelle GPO sans liaisons — a lier manuellement.

---

## :material-code-braces: Patterns avances : comparer deux GPO

### Cas d'usage

Avant de pousser une modification en production, il faut verifier que la GPO de test a exactement les memes parametres ADMX que la GPO de reference. La comparaison manuelle dans GPMC est fastidieuse et sujette aux erreurs d'omission.

La fonction suivante compare deux GPO via le rapport XML et retourne les differences.

```powershell
# Compare ADMX-based settings between two GPOs using XPath on the XML report
function Compare-GPOSettings {
    param(
        [Parameter(Mandatory)][string]$SourceGPO,
        [Parameter(Mandatory)][string]$TargetGPO
    )

    $nsManager = @{ q2 = "http://www.microsoft.com/GroupPolicy/Settings/Registry" }

    $sourceXml = [xml](Get-GPOReport -Name $SourceGPO -ReportType Xml)
    $targetXml = [xml](Get-GPOReport -Name $TargetGPO -ReportType Xml)

    $sourceSettings = $sourceXml | Select-Xml -XPath "//q2:Policy" -Namespace $nsManager |
        ForEach-Object { "$($_.Node.Name) = $($_.Node.State)" }

    $targetSettings = $targetXml | Select-Xml -XPath "//q2:Policy" -Namespace $nsManager |
        ForEach-Object { "$($_.Node.Name) = $($_.Node.State)" }

    $diff = Compare-Object $sourceSettings $targetSettings

    if ($diff) {
        Write-Host "Differences found between '$SourceGPO' and '$TargetGPO':" -ForegroundColor Yellow
        $diff | ForEach-Object {
            [PSCustomObject]@{
                Setting = $_.InputObject
                OnlyIn  = if ($_.SideIndicator -eq "<=") { $SourceGPO } else { $TargetGPO }
            }
        } | Format-Table -AutoSize
    }
    else {
        Write-Host "GPOs are identical in ADMX-based settings." -ForegroundColor Green
    }
}

# Usage
Compare-GPOSettings -SourceGPO "SEC-Postes-Baseline-PROD" -TargetGPO "SEC-Postes-Baseline-TEST"
```

```title="Resultat attendu — si differences"
Differences found between 'SEC-Postes-Baseline-PROD' and 'SEC-Postes-Baseline-TEST':

Setting                                        OnlyIn
-------                                        ------
Turn off Windows Defender SmartScreen = Enabled SEC-Postes-Baseline-PROD
Configure SmartScreen Filter = Disabled        SEC-Postes-Baseline-TEST
```

```title="Resultat attendu — si identiques"
GPOs are identical in ADMX-based settings.
```

### Limitation de la comparaison XPath

Cette fonction compare uniquement les parametres **bases sur des ADMX** (les `<Policy>` dans le rapport XML). Elle ne couvre pas :

- Les valeurs de registre ecrites directement avec `Set-GPRegistryValue` (apparaissent sous `<RegistrySettings>`)
- Les parametres de securite (comptes, droits utilisateur, options de securite)
- Les scripts de demarrage/arret ou de logon/logoff
- Les preferences GPP

Pour une comparaison complete incluant les valeurs de registre brutes, ajoutez un second XPath :

```powershell
# Also compare raw registry values written via Set-GPRegistryValue
$nsReg = @{ q1 = "http://www.microsoft.com/GroupPolicy/Settings/Registry" }

$sourceReg = $sourceXml | Select-Xml `
    -XPath "//q1:RegistrySettings/q1:Registry" -Namespace $nsReg |
    ForEach-Object { "$($_.Node.Properties.keyName)\$($_.Node.Properties.valueName) = $($_.Node.Properties.value)" }

$targetReg = $targetXml | Select-Xml `
    -XPath "//q1:RegistrySettings/q1:Registry" -Namespace $nsReg |
    ForEach-Object { "$($_.Node.Properties.keyName)\$($_.Node.Properties.valueName) = $($_.Node.Properties.value)" }

Compare-Object $sourceReg $targetReg | Format-Table -AutoSize
```

!!! quote "En resume"
    La comparaison via `Get-GPOReport -ReportType Xml` et XPath est la methode la plus fiable pour valider qu'une GPO de test est identique a sa reference de production. Elle ne couvre pas les parametres non-ADMX — prevoyez un second XPath pour les valeurs de registre brutes.

---

## :material-file-import-outline: Migration table : import inter-domaine

### Le probleme de l'import inter-domaine

Quand vous exportez une GPO d'un domaine `contoso.local` et que vous l'importez dans `fabrikam.local`, deux categories de references deviennent invalides :

1. **Les chemins UNC** : `\\contoso-dc\SYSVOL\...` ne pointe plus vers rien dans le domaine cible
2. **Les SID** : les groupes de securite references dans les filtres ou les droits sont specifiques au domaine source

Une table de migration (`*.migtable`) fait correspondre les references source aux references cibles. Sans elle, `Import-GPO` abandonne les references inconnues — ce qui peut resulter en des parametres de securite silencieusement absents.

### Creer et utiliser une table de migration

La creation de la table de migration elle-meme n'a pas de cmdlet PowerShell direct. Elle se cree via GPMC (clic droit sur une sauvegarde → "Open Migration Table Editor") ou via l'API .NET `Microsoft.GroupPolicy.GPMigrationTable`.

L'import avec la table se fait en PowerShell :

```powershell
# Import a GPO from a backup using a migration table for cross-domain references
Import-GPO -BackupId "{c8d9e0f1-a2b3-4c5d-6e7f-890abc123456}" `
           -TargetName "SEC-Postes-Baseline" `
           -Path "\\backup-server\GPO-Backups\2026-04-05" `
           -MigrationTable "C:\Temp\migration-table.migtable" `
           -CreateIfNeeded
```

Le parametre `-CreateIfNeeded` cree la GPO cible si elle n'existe pas encore dans le domaine de destination.

### Verifier le resultat de l'import

Apres un import inter-domaine, verifiez systematiquement :

```powershell
# Verify the imported GPO exists and has a non-zero version
Get-GPO -Name "SEC-Postes-Baseline" |
    Select-Object DisplayName, Id, GpoStatus,
                  @{N="ComputerVer"; E={$_.Computer.DSVersion}},
                  @{N="UserVer";     E={$_.User.DSVersion}}
```

```title="Resultat attendu"
DisplayName     : SEC-Postes-Baseline
Id              : {e1f2a3b4-c5d6-7890-ef12-345678901234}
GpoStatus       : AllSettingsEnabled
ComputerVer     : 4
UserVer         : 0
```

Un `DSVersion` de 0 sur une section signifie qu'aucun parametre n'a ete importe pour cette section. Ce n'est pas toujours un probleme — mais si vous attendiez des parametres cote ordinateur et que `ComputerVer = 0`, l'import a echoue silencieusement.

!!! danger "Production — CreateIfNeeded en domaine cible"
    `-CreateIfNeeded` avec `Import-GPO` cree la GPO dans le domaine courant de la session PowerShell. Verifiez toujours le domaine actif avec `(Get-GPO -All)[0].DomainName` avant d'executer un import inter-domaine. Un import dans le mauvais domaine n'a pas de confirmation — il s'execute sans avertissement.

!!! quote "En resume"
    L'import inter-domaine necessite une table de migration pour les chemins UNC et les SID. Sans table, les references inconnues sont silencieusement ignorees. Verifiez `DSVersion` apres import pour confirmer que les parametres ont bien ete importes. Controllez le domaine actif avant tout `Import-GPO -CreateIfNeeded`.

---

## :material-alert-outline: Piege production : Set-GPRegistryValue et les ADMX absents

### Le comportement

`Set-GPRegistryValue` ecrit directement dans `registry.pol` sans verifier si un template ADMX correspondant existe dans le Central Store. Le parametre s'applique correctement sur les clients Windows. Mais dans GPMC, il n'apparait pas dans la vue "Settings" habituelle.

Il apparait dans une section cachee appelee **Extra Registry Settings** (sous le noeud brut de la cle de registre), completement invisible si vous ne savez pas ou chercher.

### Consequence pratique

Un audit de conformite base sur GPMC peut conclure qu'une GPO "ne contient aucun parametre" alors qu'elle en contient plusieurs — ecrits via `Set-GPRegistryValue`.

Ce n'est pas un bug. C'est le comportement documente. Mais ca surprend systematiquement les equipes qui decouvrent ce comportement pour la premiere fois lors d'un audit externe.

### Detecter les GPO avec Extra Registry Settings

```powershell
# Find all GPOs that contain raw registry values (no ADMX backing)
# These appear as "Extra Registry Settings" in GPMC
Get-GPO -All | ForEach-Object {
    $gpo    = $_
    $report = Get-GPOReport -Guid $gpo.Id -ReportType Xml

    if ($report -match "ExtraRegistrySettings") {
        [PSCustomObject]@{
            GPO   = $gpo.DisplayName
            GUID  = $gpo.Id
            Issue = "Has Extra Registry Settings — verify Central Store ADMX coverage"
        }
    }
} | Format-Table -AutoSize
```

```title="Resultat attendu"
GPO                            GUID       Issue
---                            ----       -----
CFG-Postes-EnvironnementBureau {a1b2...}  Has Extra Registry Settings — verify Central Store ADMX coverage
APP-Edge-ConfigEntreprise      {7c2b...}  Has Extra Registry Settings — verify Central Store ADMX coverage
```

### Lister toutes les valeurs Extra Registry Settings d'une GPO

```powershell
# Extract all raw registry values from a GPO's registry.pol
# These are the values written via Set-GPRegistryValue
$gpoName = "CFG-Postes-EnvironnementBureau"
$xml     = [xml](Get-GPOReport -Name $gpoName -ReportType Xml)

# XPath to find raw registry entries (not backed by ADMX)
$ns = @{ q1 = "http://www.microsoft.com/GroupPolicy/Settings/Registry" }

$xml | Select-Xml -XPath "//q1:RegistrySettings/q1:Registry" -Namespace $ns |
    ForEach-Object {
        $props = $_.Node.Properties
        [PSCustomObject]@{
            KeyPath   = $props.keyName
            ValueName = $props.valueName
            Type      = $props.type
            Value     = $props.value
        }
    } | Format-Table -AutoSize
```

```title="Resultat attendu"
KeyPath                                                        ValueName       Type   Value
-------                                                        ---------       ----   -----
SOFTWARE\Policies\Microsoft\Windows\Personalization            LockScreenImage REG_SZ \\serveur\images\lockscreen.jpg
SOFTWARE\Policies\Microsoft\Windows\System                     DisableCMD      REG_DWORD 2
```

!!! danger "Production — Extra Registry Settings et les audits de securite"
    Les audits de conformite bases sur des outils qui parsent uniquement les templates ADMX (certains outils SCAP, certaines versions de Microsoft Security Compliance Toolkit) ne voient pas les Extra Registry Settings. Un parametre critique ecrit via `Set-GPRegistryValue` peut passer invisible a l'audit. Si votre politique de securite repose sur des valeurs sans ADMX, documentez-le explicitement et ajoutez un ADMX correspondant au Central Store si possible.

!!! quote "En resume"
    `Set-GPRegistryValue` ecrit sans verifier si un ADMX existe. Le parametre s'applique sur les clients mais est invisible dans la vue normale de GPMC. Le script ci-dessus detecte ces GPO. Si votre Central Store est complet, ajoutez les ADMX manquants pour rendre ces parametres visibles.

---

## :material-refresh: Forcer le rafraichissement sur des postes distants

### Invoke-GPUpdate : gpupdate a distance

`Invoke-GPUpdate` execute `gpupdate /force` sur un ou plusieurs postes distants via WMI. Pas besoin de PSRemoting — il utilise le service `Remote Registry` et WMI.

```powershell
# Force a gpupdate on a single remote computer
Invoke-GPUpdate -Computer "PC-JULIEN-01" -Force -RandomDelayInMinutes 0
```

```title="Resultat attendu"
ComputerName  : PC-JULIEN-01
Status        : Successful
Timestamp     : 05/04/2026 09:22:14
```

### Forcer le rafraichissement sur une OU entiere

```powershell
# Force gpupdate on all computers in an OU
# Get computer accounts from the OU, then invoke gpupdate on each
$ou = "OU=Postes-Pilote,DC=contoso,DC=local"

$computers = Get-ADComputer -Filter * -SearchBase $ou |
             Where-Object { $_.Enabled -eq $true } |
             Select-Object -ExpandProperty Name

Invoke-GPUpdate -Computer $computers -Force -RandomDelayInMinutes 0 |
    Select-Object ComputerName, Status | Format-Table -AutoSize
```

```title="Resultat attendu"
ComputerName    Status
------------    ------
PC-PILOTE-01    Successful
PC-PILOTE-02    Successful
PC-PILOTE-03    AccessDenied
PC-PILOTE-04    Successful
```

Un status `AccessDenied` indique que le compte qui execute le script n'a pas les droits WMI sur ce poste, ou que le pare-feu bloque le trafic WMI (port 135 + ports dynamiques).

!!! warning "A surveiller — RandomDelayInMinutes = 0 en production"
    En production, deployer `Invoke-GPUpdate` avec `-RandomDelayInMinutes 0` sur 500 postes simultanement peut creer une tempete de requetes SYSVOL et surcharger les controleurs de domaine. Utilisez un delai aleatoire (`-RandomDelayInMinutes 30`) pour etaler la charge sur une fenetre de 30 minutes.

### Simuler le RSoP sans appliquer

```powershell
# Simulate RSoP for a specific user on a specific computer (planning mode)
Get-GPResultantSetOfPolicy -Computer "PC-JULIEN-01" `
                           -User "CONTOSO\j.dupont" `
                           -ReportType Html `
                           -Path "C:\Temp\rsop-j.dupont-PC-JULIEN-01.html"
```

Le rapport HTML genere est identique au rapport RSoP de planification de GPMC. Il montre quelles GPO s'appliquent, dans quel ordre, et quels parametres sont effectifs.

!!! quote "En resume"
    `Invoke-GPUpdate` passe par WMI — il ne necessite pas PSRemoting. Limitez `-RandomDelayInMinutes 0` aux environnements de test ou aux interventions ciblees sur peu de postes. En production a grande echelle, utilisez un delai aleatoire pour eviter la surcharge SYSVOL. `Get-GPResultantSetOfPolicy` produit un rapport HTML de planification sans appliquer quoi que ce soit.

---

## :material-file-document-outline: Script d'inventaire complet : production-ready

Ce script est concu pour etre execute en production sur n'importe quel domaine. Il exporte un CSV exploitable dans Excel ou Power BI, avec une ligne par GPO et toutes les metadonnees utiles.

```powershell
# GPO Inventory Script — production-ready, exports to timestamped CSV
# Run from any workstation with RSAT GroupPolicy module installed

$exportPath = "C:\Temp\gpo-inventory-$(Get-Date -Format 'yyyyMMdd-HHmm').csv"

$report = Get-GPO -All | ForEach-Object {
    $gpo = $_
    $xml = [xml](Get-GPOReport -Guid $gpo.Id -ReportType Xml)

    # Count configured extension sections (not individual settings)
    $computerExtCount = ($xml | Select-Xml "//Computer/ExtensionData").Count
    $userExtCount     = ($xml | Select-Xml "//User/ExtensionData").Count

    # Check for raw registry values (no ADMX backing)
    $hasExtraRegistry = ($xml.OuterXml -match "ExtraRegistrySettings")

    # Get linked OUs with their enabled/enforced state
    $links = $xml.GPO.LinksTo | ForEach-Object {
        "$($_.SOMPath) [enabled=$($_.Enabled), enforced=$($_.NoOverride)]"
    }

    [PSCustomObject]@{
        DisplayName      = $gpo.DisplayName
        Id               = $gpo.Id
        Status           = $gpo.GpoStatus
        Created          = $gpo.CreationTime.ToString("yyyy-MM-dd")
        Modified         = $gpo.ModificationTime.ToString("yyyy-MM-dd")
        ComputerVersion  = $gpo.Computer.DSVersion
        UserVersion      = $gpo.User.DSVersion
        ComputerExtCount = $computerExtCount
        UserExtCount     = $userExtCount
        HasExtraRegistry = $hasExtraRegistry
        LinkCount        = @($xml.GPO.LinksTo).Count
        Links            = if ($links) { ($links -join " || ") } else { "UNLINKED" }
        Description      = ($gpo.Description -replace "`n", " ").Trim()
        Owner            = $gpo.Owner
    }
}

$report | Export-Csv -Path $exportPath -NoTypeInformation -Encoding UTF8
Write-Host "Exported $($report.Count) GPOs to: $exportPath" -ForegroundColor Green

# Quick summary to the console
$unlinked     = ($report | Where-Object { $_.Links -eq "UNLINKED" }).Count
$extraReg     = ($report | Where-Object { $_.HasExtraRegistry }).Count
$bothDisabled = ($report | Where-Object { $_.Status -eq "AllSettingsDisabled" }).Count

Write-Host ""
Write-Host "Summary:" -ForegroundColor Cyan
Write-Host "  Total GPOs       : $($report.Count)"
Write-Host "  Unlinked GPOs    : $unlinked"
Write-Host "  Extra Registry   : $extraReg"
Write-Host "  Fully disabled   : $bothDisabled"
```

```title="Resultat attendu"
Exported 47 GPOs to: C:\Temp\gpo-inventory-20260405-0922.csv

Summary:
  Total GPOs       : 47
  Unlinked GPOs    : 3
  Extra Registry   : 5
  Fully disabled   : 2
```

### Colonnes exportees et usage

| Colonne | Usage |
|---------|-------|
| `DisplayName` | Nom de la GPO — verifier le respect de la convention de nommage |
| `Status` | `AllSettingsEnabled`, `ComputerSettingsEnabled`, `AllSettingsDisabled`, etc. |
| `ComputerVersion` / `UserVersion` | 0 = section jamais configuree — candidat a desactivation |
| `HasExtraRegistry` | True = verifier le Central Store ADMX |
| `LinkCount` | 0 = GPO non liee — candidat a suppression |
| `Links` | Detail des OU liees avec statut enabled/enforced |
| `Owner` | Verifier que l'owner est un groupe (Domain Admins), pas un individu |

!!! quote "En resume"
    Ce script est le seul outil necessaire pour le premier audit d'un environnement GPO inconnu. Le CSV resultant donne une vision complete en moins de 5 minutes. Les colonnes `HasExtraRegistry` et `LinkCount = 0` sont les deux premiers filtres a appliquer.

---

## :material-speedometer: Performances : ce qui ralentit les scripts GPO

### Le goulot d'etranglement : Get-GPOReport

`Get-GPOReport` fait un appel LDAP + un acces SYSVOL par GPO. Sur un domaine de 200 GPO, c'est 200 appels sequentiels. Un script naif peut prendre 3 a 5 minutes sur un reseau LAN.

Solution : paralleliser avec `ForEach-Object -Parallel` (PowerShell 7+) ou `Start-Job`.

```powershell
# Parallel GPO report generation (PowerShell 7+ only)
$allGPOs = Get-GPO -All

$results = $allGPOs | ForEach-Object -Parallel {
    $gpo = $_
    $xml = [xml](Get-GPOReport -Guid $gpo.Id -ReportType Xml)
    $links = $xml.GPO.LinksTo | ForEach-Object { $_.SOMPath }

    [PSCustomObject]@{
        Name     = $gpo.DisplayName
        Links    = if ($links) { $links -join " | " } else { "UNLINKED" }
        Modified = $gpo.ModificationTime.ToString("yyyy-MM-dd")
    }
} -ThrottleLimit 10  # Max 10 concurrent calls to avoid overloading the DC

$results | Sort-Object Name | Format-Table -AutoSize
```

:material-information-outline: Le parametre `-ThrottleLimit` est important. Trop de requetes simultanees vers le DC ou SYSVOL peut causer des timeouts. 10 est une valeur conservative et fiable.

### Mise en cache du rapport XML

Si vous utilisez le rapport XML de la meme GPO plusieurs fois dans le meme script, stockez-le dans une variable. Ne rappelez pas `Get-GPOReport` deux fois pour la meme GPO.

```powershell
# Cache the XML report to avoid double network calls
$xml = [xml](Get-GPOReport -Name "SEC-Postes-Baseline" -ReportType Xml)

# Use $xml multiple times without additional network calls
$computerSettings = $xml | Select-Xml "//Computer/ExtensionData"
$links            = $xml.GPO.LinksTo
$hasExtraReg      = $xml.OuterXml -match "ExtraRegistrySettings"
```

!!! quote "En resume"
    `Get-GPOReport` est le seul cmdlet GPO lent. Sur un grand domaine, parallelisez avec `-Parallel -ThrottleLimit 10` (PowerShell 7+). Mettez toujours le resultat XML en cache dans une variable si vous l'utilisez plusieurs fois dans le meme script.

---

## :material-book-open-outline: References croisees

| Sujet | Reference |
|-------|-----------|
| Gouvernance — qui peut creer et modifier | Ch. 02 — Gouvernance et delegation |
| Sauvegarde et restauration PowerShell | Ch. 05 — Sauvegarde et migration |
| Automatisation CI/CD pour les GPO | Ch. 23 — Automatisation CI/CD |
| registry.pol — ce qu'ecrit Set-GPRegistryValue | La Bible GPO — Ch. 06 — registry.pol |
| Central Store et gestion des ADMX | Ch. 04 — Central Store et ADMX |
| Architecture OU et principes de liaison | Ch. 01 — Architecture GPO d'entreprise |
| Audit et conformite | Ch. 06 — Audit et conformite des GPO |

!!! quote "En résumé"
    - À relire : Gouvernance — qui peut creer et modifier → Ch. 02 — Gouvernance et delegation.
    - À relire : Sauvegarde et restauration PowerShell → Ch. 05 — Sauvegarde et migration.
    - À relire : Automatisation CI/CD pour les GPO → Ch. 23 — Automatisation CI/CD.
    - À relire : registry.pol — ce qu'ecrit Set-GPRegistryValue → La Bible GPO — Ch. 06 — registry.pol.
    - Ces renvois prolongent le chapitre avec des mécanismes complémentaires ou des cas d’usage voisins.
