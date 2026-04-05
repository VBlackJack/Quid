---
description: Sécurité des GPO — surface d'attaque, ACL audit, GPO Hijacking, SYSVOL hardening, monitoring des modifications non autorisées
tags: [gpo-admins, securite-gpo, gpo-hijacking, acl, sysvol, smb-signing, event-5136]
---
# Sécurité des GPO elles-mêmes

!!! abstract "Ce que vous allez apprendre"
    - Identifier les **trois vecteurs d'attaque** ciblant les GPO (modification directe, SYSVOL tampering, gPCFileSysPath hijacking) et comprendre pourquoi les permissions GPO sont une préoccupation de Tier-0
    - Auditer exhaustivement les **ACL sur les objets GPO** dans AD et générer un rapport CSV des éditeurs autorisés — avec alerte sur les comptes inattendus
    - Vérifier et corriger les **permissions NTFS sur SYSVOL** pour les dossiers `{GUID}` de chaque GPO
    - Détecter une attaque **GPO Hijacking** via l'événement 5136 filtré sur la modification de l'attribut `gPCFileSysPath`
    - Implémenter le **SYSVOL hardening** (SMB signing, chemins UNC qualifiés) et centraliser les alertes sur les événements 5136 et 4663

!!! tip "Si vous ne retenez qu'une chose"
    Un attaquant qui obtient le droit **Edit** sur une seule GPO liée à une OU de production peut exécuter du code sur toutes les machines de cette OU au prochain redémarrage. Ce n'est pas une hypothèse théorique — c'est la mécanique exacte des mouvements latéraux documentés dans les incidents Active Directory réels. Les permissions GPO ne sont pas une question de gouvernance. C'est du Tier-0.

---

## Contexte de production

Dans la majorité des environnements, les permissions GPO n'ont jamais été auditées depuis leur création. Les groupes autorisés à modifier les GPO critiques ont accumulé des membres au fil des années, parfois des comptes de service oubliés, parfois des comptes d'anciens administrateurs non désactivés.

Cette situation est radicalement différente d'une simple négligence de gouvernance. Une GPO modifiée de manière malveillante peut déployer un script de démarrage, affaiblir la politique de pare-feu ou installer une tâche planifiée sur l'ensemble d'une OU — sans déclencher aucune alerte si la surveillance n'est pas en place.


!!! quote "En résumé"
    - Dans la majorité des environnements, les permissions GPO n'ont jamais été auditées depuis leur création.
    - Cette situation est radicalement différente d'une simple négligence de gouvernance.
    - Le contexte de production fixe les contraintes réelles de réseau, de portée et d’exploitation qui gouvernent tout le chapitre.
    - Retenez les hypothèses opérationnelles avant de choisir un modèle de liaison ou de déploiement.
---

## Surface d'attaque des GPO

### :material-target: Pourquoi les GPO sont un objectif Tier-0

Une GPO en production est l'équivalent fonctionnel d'un accès administrateur local sur chaque machine de l'OU cible. Elle peut configurer les scripts de démarrage (exécutés en contexte SYSTEM), les paramètres de sécurité locaux, les droits utilisateurs, et les tâches planifiées.

Pour un attaquant ayant obtenu un compte dans le groupe `Group Policy Creator Owners` ou ayant reçu le droit `GpoEdit` sur une GPO stratégique, la surface d'action est considérable. Il peut compromettre un parc entier sans jamais toucher au contrôleur de domaine directement.

### :material-vector-triangle: Les trois vecteurs d'attaque

**Vecteur 1 — Modification directe de la GPO.**
L'attaquant dispose d'un compte avec le droit `GpoEdit` sur une GPO liée à une OU de machines ou d'utilisateurs. Il modifie un script de démarrage ou une tâche planifiée. Au prochain redémarrage des machines, le code s'exécute en contexte SYSTEM.

**Vecteur 2 — SYSVOL tampering.**
L'attaquant accède directement au partage SYSVOL (via SMB) et modifie les fichiers de la GPO dans `\\domain\SYSVOL\domain\Policies\{GUID}\`. Si les permissions NTFS sur ces dossiers sont trop permissives, la modification contourne entièrement les ACL de l'objet AD.

**Vecteur 3 — gPCFileSysPath hijacking.**
L'attaquant modifie l'attribut LDAP `gPCFileSysPath` d'un objet `groupPolicyContainer` pour pointer vers un partage qu'il contrôle. Les machines continuent d'appliquer la "GPO" officielle — mais récupèrent les paramètres depuis un chemin attaquant. C'est le vecteur le plus discret et le plus difficile à détecter sans surveillance spécifique.

### :material-shield-alert: Pourquoi ce n'est pas couvert par la séparation des rôles seule

La délégation par rôles (voir [Chapitre 02 — Gouvernance](02-gouvernance.md)) réduit la surface d'attaque en limitant qui peut modifier quoi. Mais elle ne suffit pas : un compte délégué compromis, un service account mal cloisonné, ou un ancien administrateur non retiré des groupes de délégation rendent la séparation des rôles caduque.

La sécurité des GPO repose sur **trois piliers simultanés** : ACL correctes sur les objets AD, permissions NTFS correctes sur SYSVOL, et surveillance active des modifications.

!!! quote "En résumé"
    - Les GPO sont des vecteurs d'exécution de code à l'échelle d'une OU — traiter leurs permissions comme du Tier-0
    - Les trois vecteurs d'attaque sont : modification directe (droit GpoEdit), SYSVOL tampering (ACL NTFS permissives), et gPCFileSysPath hijacking (attribut LDAP modifié)
    - La séparation des rôles est nécessaire mais insuffisante sans surveillance active

---

## Audit des ACL sur les objets GPO

### :material-eye-check: Lister tous les éditeurs de GPO

La commande `Get-GPPermissions` avec le flag `-All` retourne l'ensemble des permissions sur une GPO donnée. Combinée à `Get-GPO -All`, elle permet de construire une vue complète des droits d'édition sur l'ensemble du parc GPO.

```powershell title="Get-GPOEditors.ps1"
# List all accounts with GPO edit rights across all GPOs in the domain
Import-Module GroupPolicy

$editRights = @("GpoEdit", "GpoEditDeleteModifySecurity")

Get-GPO -All | ForEach-Object {
    $gpo = $_
    Get-GPPermissions -Guid $gpo.Id -All | Where-Object {
        $_.Permission -in $editRights
    } | ForEach-Object {
        [PSCustomObject]@{
            GPOName    = $gpo.DisplayName
            GUID       = $gpo.Id
            Trustee    = $_.Trustee.Name
            TrusteeType = $_.Trustee.SidType
            Permission = $_.Permission
            Inherited  = $_.Inherited
        }
    }
} | Sort-Object GPOName, Trustee
```

```title="Résultat attendu (extrait)"
GPOName                  GUID                                   Trustee                Permission
-------                  ----                                   -------                ----------
Default Domain Policy    {31B2F340-016D-11D2-945F-00C04FB984F9} CONTOSO\Domain Admins  GpoEditDeleteModifySecurity
SEC-Postes-Baseline      {A1B2C3D4-...}                         CONTOSO\GPO-Editors    GpoEdit
SEC-Postes-Baseline      {A1B2C3D4-...}                         CONTOSO\svc-old-tool   GpoEdit
WSUS-Configuration       {B2C3D4E5-...}                         CONTOSO\GPO-Editors    GpoEdit
```

La ligne `svc-old-tool` avec le droit `GpoEdit` sur une GPO de sécurité est exactement ce qu'il faut détecter et corriger.

### :material-file-export: Rapport CSV complet des permissions GPO

```powershell title="Export-GPOPermissionReport.ps1"
# Generate a full GPO permission report and export to CSV
Import-Module GroupPolicy

$reportPath = "C:\Temp\GPO-ACL-Report-$(Get-Date -Format 'yyyyMMdd-HHmm').csv"
$allPermissions = @()

$allGPOs = Get-GPO -All

foreach ($gpo in $allGPOs) {
    $perms = Get-GPPermissions -Guid $gpo.Id -All
    foreach ($perm in $perms) {
        $allPermissions += [PSCustomObject]@{
            GPOName      = $gpo.DisplayName
            GUID         = $gpo.Id.ToString()
            LinkedOU     = ($gpo | Get-GPOLinks | Select-Object -ExpandProperty Target -EA SilentlyContinue) -join "; "
            Trustee      = $perm.Trustee.Name
            TrusteeSID   = $perm.Trustee.Sid
            TrusteeType  = $perm.Trustee.SidType
            Permission   = $perm.Permission.ToString()
            Inherited    = $perm.Inherited
            LastModified = $gpo.ModificationTime.ToString("yyyy-MM-dd HH:mm")
        }
    }
}

$allPermissions | Export-Csv -Path $reportPath -NoTypeInformation -Encoding UTF8
Write-Host "Permission report exported: $reportPath ($($allPermissions.Count) entries)"
```

```title="Résultat attendu"
Permission report exported: C:\Temp\GPO-ACL-Report-20260405-0914.csv (312 entries)
```

Ce rapport CSV est le document de référence pour un audit formel. Il doit être produit avant et après chaque modification de délégation, et archivé avec la date.

### :material-alert-decagram: Détecter les éditeurs inattendus

Après avoir établi une liste blanche des groupes autorisés à éditer les GPO, ce script identifie tous les comptes qui ne devraient pas avoir ce droit.

```powershell title="Find-UnexpectedGPOEditors.ps1"
# Alert on any account with GPO edit rights outside the approved list
Import-Module GroupPolicy

# Adjust this list to match your environment's approved delegation groups
$approvedTrustees = @(
    "CONTOSO\Domain Admins",
    "CONTOSO\GPO-Editors",
    "CONTOSO\GPO-Admins",
    "CONTOSO\Enterprise Admins"
)

$editRights = @("GpoEdit", "GpoEditDeleteModifySecurity")

$unexpected = Get-GPO -All | ForEach-Object {
    $gpo = $_
    Get-GPPermissions -Guid $gpo.Id -All | Where-Object {
        $_.Permission -in $editRights -and
        $_.Trustee.Name -notin $approvedTrustees
    } | ForEach-Object {
        [PSCustomObject]@{
            GPOName    = $gpo.DisplayName
            Trustee    = $_.Trustee.Name
            TrusteeType = $_.Trustee.SidType
            Permission = $_.Permission
        }
    }
}

if ($unexpected) {
    Write-Warning "UNEXPECTED GPO EDITORS DETECTED: $($unexpected.Count) entries"
    $unexpected | Format-Table -AutoSize
} else {
    Write-Host "All GPO editors are within the approved list."
}
```

```title="Résultat attendu (alerte)"
WARNING: UNEXPECTED GPO EDITORS DETECTED: 2 entries

GPOName                 Trustee               TrusteeType  Permission
-------                 -------               -----------  ----------
SEC-Postes-Baseline     CONTOSO\svc-old-tool  User         GpoEdit
WSUS-Configuration      CONTOSO\jdupont       User         GpoEdit
```

### :material-broom: Nettoyer CREATOR OWNER

En production, l'entrée `CREATOR OWNER` sur les GPO est une anomalie à corriger. Elle accorde des permissions implicites au compte qui a créé la GPO — un comportement acceptable en test, dangereux en production si le compte créateur n'est plus en activité ou a été compromis.

```powershell title="Remove-CreatorOwnerGPOPermissions.ps1"
# Remove CREATOR OWNER permissions from all GPOs (production hardening)
Import-Module GroupPolicy

Get-GPO -All | ForEach-Object {
    $gpo = $_
    $creatorOwnerPerms = Get-GPPermissions -Guid $gpo.Id -All |
        Where-Object { $_.Trustee.Name -eq "CREATOR OWNER" }

    if ($creatorOwnerPerms) {
        Write-Host "Removing CREATOR OWNER from: $($gpo.DisplayName)"
        Set-GPPermissions -Guid $gpo.Id `
            -TargetName "CREATOR OWNER" `
            -TargetType "Group" `
            -PermissionLevel "None"
    }
}

Write-Host "CREATOR OWNER cleanup complete."
```

!!! danger "Production — Ne pas exécuter sans sauvegarde"
    Avant de supprimer `CREATOR OWNER` ou tout autre trustee sur une GPO, effectuez une sauvegarde complète avec `Backup-GPO -All`. La suppression de permissions est irréversible sans restauration. Vérifiez également que les GPO critiques (`Default Domain Policy`, `Default Domain Controllers Policy`) ne perdent pas d'accès légitime.

!!! quote "En résumé"
    - `Get-GPPermissions -All | Where-Object { $_.Permission -in "GpoEdit","GpoEditDeleteModifySecurity" }` est la commande centrale de l'audit ACL
    - Exportez en CSV et archivez — c'est votre référence avant/après pour tout audit
    - `CREATOR OWNER` n'a aucune place en production sur une GPO critique
    - Maintenez une liste blanche des groupes autorisés et alertez sur tout écart

---

## ACL sur SYSVOL : le deuxième périmètre

### :material-folder-lock: Structure attendue des permissions NTFS

Chaque GPO possède un dossier dans SYSVOL sous `\\domain\SYSVOL\domain\Policies\{GUID}\`. Ce dossier contient les fichiers de configuration réels (Registry.pol, GptTmpl.inf, scripts).

Les permissions NTFS attendues sur ce dossier sont strictes.

| Identité | Permissions NTFS requises | Justification |
|---|---|---|
| `SYSTEM` | Contrôle total | Traitement GPO par le système |
| `Domain Admins` | Contrôle total | Administration |
| `Enterprise Admins` | Contrôle total | Administration domaine |
| `Authenticated Users` | Lecture + Exécution | Application de la GPO aux machines/utilisateurs |
| `CREATOR OWNER` | Aucune permission | Doit être supprimé en production |

Tout groupe ou compte supplémentaire avec des permissions **d'écriture** sur ce dossier est une anomalie.

### :material-magnify: Audit des permissions SYSVOL par GPO

```powershell title="Audit-SYSVOLPermissions.ps1"
# Audit NTFS permissions on SYSVOL GPO folders and flag unexpected write access
$domain     = $env:USERDNSDOMAIN
$sysvolBase = "\\$domain\SYSVOL\$domain\Policies"

# Identities that are allowed write access
$allowedWriters = @(
    "NT AUTHORITY\SYSTEM",
    "BUILTIN\Administrators",
    "CONTOSO\Domain Admins",
    "CONTOSO\Enterprise Admins"
)

$report = Get-GPO -All | ForEach-Object {
    $gpo     = $_
    $gpoPath = "$sysvolBase\{$($gpo.Id)}"

    if (-not (Test-Path $gpoPath)) {
        Write-Warning "SYSVOL folder missing for GPO: $($gpo.DisplayName) [{$($gpo.Id)}]"
        return
    }

    $acl = Get-Acl -Path $gpoPath
    foreach ($ace in $acl.Access) {
        $isWrite = $ace.FileSystemRights -band [System.Security.AccessControl.FileSystemRights]::Write
        if ($isWrite -and $ace.IdentityReference.Value -notin $allowedWriters) {
            [PSCustomObject]@{
                GPOName    = $gpo.DisplayName
                GUID       = $gpo.Id
                Path       = $gpoPath
                Identity   = $ace.IdentityReference.Value
                Rights     = $ace.FileSystemRights.ToString()
                AccessType = $ace.AccessControlType.ToString()
                Inherited  = $ace.IsInherited
            }
        }
    }
}

if ($report) {
    Write-Warning "Unexpected SYSVOL write permissions detected: $($report.Count) entries"
    $report | Format-Table -AutoSize
    $report | Export-Csv "C:\Temp\SYSVOL-ACL-Issues-$(Get-Date -Format 'yyyyMMdd').csv" -NoTypeInformation
} else {
    Write-Host "SYSVOL permissions are clean — no unexpected write access found."
}
```

```title="Résultat attendu (anomalie détectée)"
WARNING: Unexpected SYSVOL write permissions detected: 1 entries

GPOName               GUID         Identity               Rights        AccessType  Inherited
-------               ----         --------               ------        ----------  ---------
SEC-Postes-Baseline   {A1B2C3...}  CONTOSO\svc-deploy     Modify        Allow       False
```

### :material-wrench: Corriger les permissions avec icacls

Quand une anomalie est détectée, la correction se fait avec `icacls` en ligne de commande ou `Set-Acl` en PowerShell.

```powershell title="Fix-SYSVOLPermission.ps1"
# Remove unexpected write access from a SYSVOL GPO folder
# Run as Domain Admin on a DC or from an admin workstation with DC connectivity

$domain  = $env:USERDNSDOMAIN
$guid    = "{A1B2C3D4-xxxx-xxxx-xxxx-xxxxxxxxxxxx}"  # replace with actual GUID
$gpoPath = "\\$domain\SYSVOL\$domain\Policies\$guid"
$account = "CONTOSO\svc-deploy"

# Remove all permissions for the account
icacls $gpoPath /remove:g "$account" /T /C

Write-Host "Removed permissions for $account on $gpoPath"

# Verify the change
$acl = Get-Acl -Path $gpoPath
$acl.Access | Where-Object { $_.IdentityReference.Value -eq $account } |
    Format-Table IdentityReference, FileSystemRights, AccessControlType
```

```title="Résultat attendu"
Removed permissions for CONTOSO\svc-deploy on \\contoso.local\SYSVOL\...
(no output = permission successfully removed)
```

### :material-alert: L'événement 1058 : accès SYSVOL refusé

Quand les permissions SYSVOL sont incorrectes — trop restrictives cette fois — les machines ne peuvent plus lire les fichiers de la GPO. L'événement **1058** dans le journal `System` en est la signature.

| Événement | Journal | Signification |
|---|---|---|
| **1058** | System | Impossible d'accéder aux fichiers de la GPO sur SYSVOL — permissions insuffisantes ou chemin inaccessible |
| **1030** | System | Traitement GPO échoué — souvent couplé à 1058 |
| **1129** | System | Impossible de contacter le DC — problème réseau ou DNS, peut masquer un 1058 |

!!! danger "Production — 1058 après une modification de permissions"
    Si vous observez des événements 1058 après avoir modifié les ACL SYSVOL, vous avez probablement retiré les permissions de lecture pour `Authenticated Users`. Cette entrée est indispensable — sans elle, les machines et utilisateurs ne peuvent pas lire les paramètres de la GPO. Restaurez immédiatement avec `icacls $path /grant "NT AUTHORITY\Authenticated Users:(OI)(CI)RX"`.

!!! quote "En résumé"
    - Le dossier `{GUID}` dans SYSVOL doit être accessible en lecture à `Authenticated Users` et en écriture uniquement aux administrateurs et à SYSTEM
    - L'événement 1058 indique une GPO inaccessible sur SYSVOL — vérifiez les permissions NTFS immédiatement
    - Auditez SYSVOL avec `Get-Acl` et alertez sur tout compte non administrateur avec des droits d'écriture

---

## Scénario d'attaque : GPO Hijacking

### :material-skull: Mécanique de l'attaque

Le GPO Hijacking exploite l'attribut LDAP `gPCFileSysPath` de l'objet `groupPolicyContainer` dans Active Directory. Cet attribut contient le chemin UNC vers le dossier SYSVOL de la GPO, par exemple :

```
\\contoso.local\SYSVOL\contoso.local\Policies\{A1B2C3D4-...}
```

!!! danger "Scénario d'attaque complet"

    **Étape 1 — Reconnaissance.**
    L'attaquant énumère les GPO liées aux OUs de machines critiques avec `Get-GPInheritance` ou via une requête LDAP sur `CN=Policies,CN=System,DC=...`.

    **Étape 2 — Préparation du payload.**
    L'attaquant crée un partage SMB sur un serveur qu'il contrôle et y place une arborescence GPO minimale avec un script de démarrage malveillant dans `Machine\Scripts\Startup\`.

    **Étape 3 — Modification de gPCFileSysPath.**
    En utilisant un compte ayant le droit `GpoEditDeleteModifySecurity` sur la GPO cible (ou en abusant de `CREATOR OWNER`), l'attaquant modifie l'attribut `gPCFileSysPath` pour pointer vers son partage.

    **Étape 4 — Exécution silencieuse.**
    Au prochain traitement GPO (redémarrage ou `gpupdate /force`), les machines de l'OU cible récupèrent les paramètres depuis le partage attaquant. Le script s'exécute en contexte SYSTEM.

    **Pourquoi c'est discret :** la GPO dans GPMC semble normale — son nom, sa description et son statut n'ont pas changé. Seul l'attribut `gPCFileSysPath` a été modifié, et cela ne génère aucune alerte sans surveillance AD spécifique.

### :material-magnify-scan: Détection via l'événement 5136

La modification de `gPCFileSysPath` génère un événement **5136** dans le journal Security du PDC Emulator. Le filtre ci-dessous cible précisément cet attribut.

```powershell title="Detect-GPCFileSysPathChange.ps1"
# Detect gPCFileSysPath modifications — primary GPO Hijacking indicator
$dcPDC     = (Get-ADDomain).PDCEmulator
$startDate = (Get-Date).AddDays(-7)

$events = Get-WinEvent -ComputerName $dcPDC -FilterHashtable @{
    LogName   = 'Security'
    Id        = 5136
    StartTime = $startDate
} -ErrorAction SilentlyContinue

$hijackAttempts = $events | Where-Object {
    $_.Message -match "groupPolicyContainer" -and
    $_.Message -match "gPCFileSysPath"
}

if ($hijackAttempts) {
    Write-Warning "CRITICAL: gPCFileSysPath modification detected — possible GPO Hijacking"

    foreach ($evt in $hijackAttempts) {
        # Extract relevant fields from the event message
        $lines      = $evt.Message -split "`n"
        $objectName = ($lines | Where-Object { $_ -match "Object Name:" })     -replace ".*Object Name:\s*", ""
        $newValue   = ($lines | Where-Object { $_ -match "New Value:" })       -replace ".*New Value:\s*", ""
        $account    = ($lines | Where-Object { $_ -match "Account Name:" })[0] -replace ".*Account Name:\s*", ""

        Write-Host ("-" * 60)
        Write-Host "Time      : $($evt.TimeCreated)"
        Write-Host "Account   : $account"
        Write-Host "GPO object: $objectName"
        Write-Host "New path  : $newValue"
    }
} else {
    Write-Host "No gPCFileSysPath modifications found in the last 7 days."
}
```

```title="Résultat attendu (alerte critique)"
CRITICAL: gPCFileSysPath modification detected — possible GPO Hijacking
------------------------------------------------------------
Time      : 2026-04-04 22:14:37
Account   : CONTOSO\svc-compromised
GPO object: CN={A1B2C3D4-...},CN=Policies,CN=System,DC=contoso,DC=local
New path  : \\attacker-srv\C$\FakeGPO\{A1B2C3D4-...}
```

### :material-shield-check: Vérification de l'intégrité de gPCFileSysPath

En parallèle de la surveillance en temps réel, ce script vérifie périodiquement que tous les `gPCFileSysPath` pointent bien vers SYSVOL et non vers un chemin externe.

```powershell title="Verify-GPCFileSysPath.ps1"
# Verify all gPCFileSysPath values point to the legitimate SYSVOL path
$domain      = $env:USERDNSDOMAIN
$sysvolBase  = "\\$domain\SYSVOL\$domain\Policies"

$gpoObjects = Get-ADObject -Filter { objectClass -eq "groupPolicyContainer" } `
    -Properties gPCFileSysPath, displayName

$anomalies = $gpoObjects | Where-Object {
    $_.gPCFileSysPath -and
    -not $_.gPCFileSysPath.StartsWith($sysvolBase, [System.StringComparison]::OrdinalIgnoreCase)
}

if ($anomalies) {
    Write-Warning "ANOMALY: gPCFileSysPath points outside SYSVOL for $($anomalies.Count) GPO(s)"
    $anomalies | Select-Object DisplayName, gPCFileSysPath | Format-Table -AutoSize
} else {
    Write-Host "All gPCFileSysPath values point to the legitimate SYSVOL path."
}
```

```title="Résultat attendu (environnement sain)"
All gPCFileSysPath values point to the legitimate SYSVOL path.
```

!!! danger "Production — gPCFileSysPath hors SYSVOL = incident actif"
    Si ce script retourne une anomalie, traitez-la comme un incident de sécurité immédiat. Isolez les machines de l'OU cible du réseau le temps de l'investigation, restaurez `gPCFileSysPath` à sa valeur légitime, et analysez les logs des machines qui ont appliqué la GPO corrompue.

!!! quote "En résumé"
    - Le GPO Hijacking modifie `gPCFileSysPath` pour rediriger les machines vers un payload attaquant
    - L'événement 5136 filtré sur `gPCFileSysPath` + `groupPolicyContainer` est le seul moyen de détecter cette attaque en temps réel
    - Vérifiez périodiquement que tous les `gPCFileSysPath` pointent vers SYSVOL légitime
    - Un `gPCFileSysPath` hors SYSVOL est un indicateur de compromission — pas une hypothèse de travail

---

## Hardening SYSVOL

### :material-lock-check: SMB Signing obligatoire sur les DCs

Le partage SYSVOL est un partage SMB. Sans signature SMB (`SMB Signing`), un attaquant en position Man-in-the-Middle peut intercepter et modifier le trafic SYSVOL entre les machines et les contrôleurs de domaine — compromettant le contenu des GPO en transit.

La signature SMB doit être **obligatoire** (`RequireSecuritySignature = 1`) sur tous les contrôleurs de domaine, côté serveur et côté client.

```powershell title="Set-SMBSigningRequired.ps1"
# Enforce SMB signing on all Domain Controllers via GPO registry values
# These values are set by GPO under:
# Computer Configuration > Windows Settings > Security Settings > Local Policies > Security Options

# Registry path for SMB server signing
$serverPath = "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters"

# Registry path for SMB client signing
$clientPath = "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanWorkstation\Parameters"

# Run on each DC — or deploy via GPO
Set-ItemProperty -Path $serverPath -Name "RequireSecuritySignature" -Value 1 -Type DWord
Set-ItemProperty -Path $serverPath -Name "EnableSecuritySignature"  -Value 1 -Type DWord
Set-ItemProperty -Path $clientPath -Name "RequireSecuritySignature" -Value 1 -Type DWord
Set-ItemProperty -Path $clientPath -Name "EnableSecuritySignature"  -Value 1 -Type DWord

Write-Host "SMB signing enforced on $env:COMPUTERNAME"
```

### :material-check-all: Vérifier l'état du SMB Signing sur tous les DCs

```powershell title="Verify-SMBSigning.ps1"
# Verify SMB signing status on all Domain Controllers
$dcs = (Get-ADDomainController -Filter *).Name

foreach ($dc in $dcs) {
    $smbConfig = Get-SmbServerConfiguration -CimSession $dc -ErrorAction SilentlyContinue

    [PSCustomObject]@{
        DomainController         = $dc
        RequireSecuritySignature = $smbConfig.RequireSecuritySignature
        EnableSecuritySignature  = $smbConfig.EnableSecuritySignature
        Status                   = if ($smbConfig.RequireSecuritySignature) { "HARDENED" } else { "VULNERABLE" }
    }
} | Format-Table -AutoSize
```

```title="Résultat attendu"
DomainController  RequireSecuritySignature  EnableSecuritySignature  Status
----------------  ------------------------  -----------------------  ------
DC01              True                      True                     HARDENED
DC02              True                      True                     HARDENED
DC03              False                     True                     VULNERABLE
```

DC03 avec `RequireSecuritySignature = False` est une surface d'attaque. Corrigez immédiatement.

### :material-link-variant-off: Éviter les chemins UNC non qualifiés dans les GPOs

Les chemins UNC non qualifiés dans les paramètres GPO (exemple : `\\serveur\partage`) sont vulnérables à une attaque d'injection de chemin UNC. Un attaquant capable de faire résoudre ce nom vers un serveur qu'il contrôle intercepte le trafic.

Utilisez systématiquement des chemins **FQDN qualifiés** :

| Format | Risque | À utiliser |
|---|---|---|
| `\\serveur\partage` | Résolution NetBIOS injectable | Non |
| `\\serveur.contoso.local\partage` | Résolution DNS, plus difficile à détourner | Oui |
| `\\contoso.local\SYSVOL\...` | Chemin SYSVOL standard — FQDN | Oui |

### :material-table-lock: Tableau des mesures de hardening SYSVOL

| Mesure | Commande / Paramètre | Priorité |
|---|---|---|
| SMB Signing obligatoire | `RequireSecuritySignature = 1` côté serveur et client | Critique |
| Chemins UNC FQDN dans les GPO | Remplacer `\\server\` par `\\server.domain.local\` | Haute |
| Permissions NTFS dossiers `{GUID}` | `Authenticated Users` en lecture seule | Critique |
| Suppression de `CREATOR OWNER` sur SYSVOL | `icacls /remove:g "CREATOR OWNER"` | Haute |
| Audit SACL sur les dossiers `{GUID}` | Activer `Audit Object Access` pour détecter 4663 | Haute |

!!! warning "SMB Signing et performances"
    L'activation du SMB Signing obligatoire sur les serveurs de fichiers (pas uniquement les DCs) peut avoir un impact mesurable sur les performances des transferts de gros volumes. Sur les DCs, cet impact est négligeable et le risque de le ne pas l'activer est inacceptable. Activez-le inconditionnellement sur les DCs, et évaluez au cas par cas sur les serveurs de fichiers.

!!! quote "En résumé"
    - SMB Signing obligatoire (`RequireSecuritySignature = 1`) sur tous les DCs est non négociable
    - Vérifiez l'état de chaque DC avec `Get-SmbServerConfiguration` — un seul DC vulnérable suffit
    - Utilisez des chemins UNC FQDN dans toutes les GPO — les chemins NetBIOS courts sont injectables
    - Les SACL sur les dossiers `{GUID}` permettent de générer l'événement 4663 pour surveiller les modifications SYSVOL

---

## Monitoring et alertes

### :material-eye-outline: Les deux événements fondamentaux

La surveillance des GPO repose sur deux événements complémentaires qui couvrent les deux surfaces d'attaque.

| Événement | Journal | Déclencheur | Surface couverte |
|---|---|---|---|
| **5136** | Security (DC) | Modification d'un attribut d'un objet AD | Modification de l'objet GPO dans AD (gPCFileSysPath, ACL, versionNumber) |
| **4663** | Security (DC/Serveur) | Accès à un objet fichier avec SACL | Modification directe des fichiers dans SYSVOL |
| **5137** | Security (DC) | Création d'un objet AD | Nouvelle GPO créée |
| **5141** | Security (DC) | Suppression d'un objet AD | GPO supprimée |
| **4670** | Security (DC) | Modification des permissions d'un objet AD | ACL d'une GPO modifiée |

L'événement 5136 nécessite l'activation de **"Audit Directory Service Changes"** (voir [Chapitre 06 — Audit](06-audit-conformite.md)).

L'événement 4663 nécessite l'activation de **"Audit Object Access"** et la pose d'une SACL sur les dossiers SYSVOL concernés.

### :material-shield-sword: Activer les SACL sur les dossiers SYSVOL

Sans SACL sur les dossiers `{GUID}`, l'événement 4663 n'est jamais généré — même si "Audit Object Access" est activé dans la politique d'audit.

```powershell title="Set-SYSVOLAuditSACL.ps1"
# Add audit SACL to all SYSVOL GPO folders to enable event 4663 on write operations
$domain     = $env:USERDNSDOMAIN
$sysvolBase = "\\$domain\SYSVOL\$domain\Policies"

Get-GPO -All | ForEach-Object {
    $gpoPath = "$sysvolBase\{$($_.Id)}"

    if (-not (Test-Path $gpoPath)) { return }

    $acl = Get-Acl -Path $gpoPath -Audit

    # Create audit rule: audit write access by Everyone (catches all unexpected writers)
    $auditRule = New-Object System.Security.AccessControl.FileSystemAuditRule(
        "Everyone",
        [System.Security.AccessControl.FileSystemRights]::Write,
        [System.Security.AccessControl.InheritanceFlags]"ContainerInherit,ObjectInherit",
        [System.Security.AccessControl.PropagationFlags]::None,
        [System.Security.AccessControl.AuditFlags]::Success
    )

    $acl.AddAuditRule($auditRule)
    Set-Acl -Path $gpoPath -AclObject $acl

    Write-Host "SACL set on: $gpoPath"
}

Write-Host "Audit SACL deployment complete."
```

### :material-bell-ring: Script d'alerte unifié 5136 + 4663

```powershell title="Monitor-GPOModifications.ps1"
# Unified GPO modification alert — monitors both AD object changes (5136) and SYSVOL file changes (4663)
param(
    [int]$LookbackMinutes  = 60,
    [string]$AlertEmail    = "soc@contoso.local",
    [string]$SmtpServer    = "smtp.contoso.local",
    [string]$FromAddress   = "gpo-monitor@contoso.local"
)

$dcPDC    = (Get-ADDomain).PDCEmulator
$cutoff   = (Get-Date).AddMinutes(-$LookbackMinutes)
$alerts   = @()

# --- Event 5136: AD object modifications on groupPolicyContainer ---
$adEvents = Get-WinEvent -ComputerName $dcPDC -FilterHashtable @{
    LogName   = 'Security'
    Id        = 5136
    StartTime = $cutoff
} -ErrorAction SilentlyContinue | Where-Object {
    $_.Message -match "groupPolicyContainer"
}

foreach ($evt in $adEvents) {
    $lines = $evt.Message -split "`n"
    $alerts += [PSCustomObject]@{
        Type      = "AD-Object-5136"
        Time      = $evt.TimeCreated
        Account   = ($lines | Where-Object { $_ -match "Account Name:" })[0] -replace ".*Account Name:\s*", ""
        Object    = ($lines | Where-Object { $_ -match "Object Name:" })     -replace ".*Object Name:\s*", ""
        Attribute = ($lines | Where-Object { $_ -match "LDAP Display Name:" }) -replace ".*LDAP Display Name:\s*", ""
        Source    = $dcPDC
    }
}

# --- Event 4663: SYSVOL file write operations ---
$fileEvents = Get-WinEvent -ComputerName $dcPDC -FilterHashtable @{
    LogName   = 'Security'
    Id        = 4663
    StartTime = $cutoff
} -ErrorAction SilentlyContinue | Where-Object {
    $_.Message -match "SYSVOL" -and
    $_.Message -match "Policies"
}

foreach ($evt in $fileEvents) {
    $lines = $evt.Message -split "`n"
    $alerts += [PSCustomObject]@{
        Type      = "SYSVOL-File-4663"
        Time      = $evt.TimeCreated
        Account   = ($lines | Where-Object { $_ -match "Account Name:" })[0] -replace ".*Account Name:\s*", ""
        Object    = ($lines | Where-Object { $_ -match "Object Name:" })     -replace ".*Object Name:\s*", ""
        Attribute = "File write"
        Source    = $dcPDC
    }
}

# --- Output and optional email alert ---
if ($alerts) {
    Write-Warning "GPO modifications detected: $($alerts.Count) event(s) in the last $LookbackMinutes minutes"
    $alerts | Format-Table -AutoSize

    if ($AlertEmail -and $SmtpServer) {
        $body = $alerts | Format-Table -AutoSize | Out-String
        Send-MailMessage `
            -To      $AlertEmail `
            -From    $FromAddress `
            -Subject "[GPO-ALERT] $($alerts.Count) modification(s) on $($dcPDC) — $(Get-Date -Format 'yyyy-MM-dd HH:mm')" `
            -Body    $body `
            -SmtpServer $SmtpServer
        Write-Host "Alert email sent to $AlertEmail"
    }
} else {
    Write-Host "No GPO modifications detected in the last $LookbackMinutes minutes."
}
```

```title="Résultat attendu (alerte)"
WARNING: GPO modifications detected: 3 event(s) in the last 60 minutes

Type             Time                   Account              Object                         Attribute
----             ----                   -------              ------                         ---------
AD-Object-5136   2026-04-04 22:14:37    CONTOSO\svc-comp     CN={A1B2C3...},CN=Policies...  gPCFileSysPath
AD-Object-5136   2026-04-04 22:14:38    CONTOSO\svc-comp     CN={A1B2C3...},CN=Policies...  versionNumber
SYSVOL-File-4663 2026-04-04 22:16:11    CONTOSO\svc-comp     \\...\SYSVOL\...\startup.cmd   File write
Alert email sent to soc@contoso.local
```

### :material-server-network: Centralisation WEC pour les événements GPO

Pour centraliser les événements 5136 et 4663 vers un collecteur WEC sans agent, déployez cette souscription sur le serveur WEC.

```xml
<!-- WEF Subscription: GPO Security Events (5136 + 4663 + 4670)        -->
<!-- Deploy via: wecutil cs GPO-Security-Subscription.xml               -->
<Subscription>
  <SubscriptionType>SourceInitiated</SubscriptionType>
  <Description>GPO security events: AD modifications and SYSVOL file access</Description>
  <SubscriptionId>GPO-Security-Events</SubscriptionId>
  <Enabled>true</Enabled>
  <Uri>http://schemas.microsoft.com/wbem/wsman/1/windows/EventLog</Uri>
  <ConfigurationMode>Custom</ConfigurationMode>
  <Delivery Mode="Push">
    <Batching>
      <MaxLatencyTime>300000</MaxLatencyTime>
    </Batching>
    <PushSettings>
      <Heartbeat Interval="900000"/>
    </PushSettings>
  </Delivery>
  <Query>
    <![CDATA[
      <QueryList>
        <Query Id="0">
          <Select Path="Security">
            *[System[(EventID=5136 or EventID=5137 or EventID=5141 or EventID=4670)]]
          </Select>
        </Query>
        <Query Id="1">
          <Select Path="Security">
            *[System[EventID=4663]]
            and
            *[EventData[Data[@Name='ObjectName'] and contains(Data,'SYSVOL') and contains(Data,'Policies')]]
          </Select>
        </Query>
      </QueryList>
    ]]>
  </Query>
  <ReadExistingEvents>false</ReadExistingEvents>
  <TransportSecurity>HTTPS</TransportSecurity>
  <ContentFormat>RenderedText</ContentFormat>
</Subscription>
```

!!! quote "En résumé"
    - Les événements 5136 (objet AD) et 4663 (fichier SYSVOL) couvrent les deux surfaces d'attaque GPO
    - 4663 nécessite une SACL sur les dossiers `{GUID}` — sans elle, aucun événement n'est généré
    - Centralisez via WEC pour préserver les événements même si un DC est compromis
    - L'alerte unifiée 5136 + 4663 est votre filet de détection minimal en production

---

## Moindre privilège pour la gestion des GPO

### :material-account-group: Séparer les quatre rôles opérationnels

Une gestion GPO sécurisée repose sur la séparation des quatre rôles définis dans le [Chapitre 02 — Gouvernance](02-gouvernance.md). La différence avec le contexte sécurité est ici la granularité : chaque rôle ne doit s'exercer que sur le périmètre strictement nécessaire.

| Rôle | Permission GPO | Portée recommandée |
|---|---|---|
| **GPO Creator** | Membre de `Group Policy Creator Owners` | Équipe infrastructure — liste nominative |
| **GPO Editor** | `GpoEdit` sur les GPO spécifiques | Par GPO — jamais sur toutes les GPO du domaine |
| **GPO Linker** | Permission "Link GPOs" sur les OU cibles | Par OU — pas au niveau du domaine |
| **GPO Approver** | `GpoEditDeleteModifySecurity` avec AGPM | Lead technique ou responsable sécurité uniquement |

### :material-key-variant: Déléguer le droit d'édition sur une GPO spécifique

```powershell title="Grant-GPOEditPermission.ps1"
# Grant GpoEdit permission to a specific group on a specific GPO
# Replace GPOName and GroupName with actual values

$gpoName   = "SEC-Postes-Baseline"
$groupName = "CONTOSO\GPO-Editors-Securite"

Set-GPPermissions -Name $gpoName `
    -TargetName $groupName `
    -TargetType "Group" `
    -PermissionLevel "GpoEdit"

Write-Host "GpoEdit granted to $groupName on GPO: $gpoName"

# Verify
Get-GPPermissions -Name $gpoName -All |
    Where-Object { $_.Trustee.Name -eq $groupName } |
    Format-Table Trustee, Permission
```

```title="Résultat attendu"
GpoEdit granted to CONTOSO\GPO-Editors-Securite on GPO: SEC-Postes-Baseline

Trustee                           Permission
-------                           ----------
CONTOSO\GPO-Editors-Securite      GpoEdit
```

### :material-link: Déléguer la liaison de GPO sur une OU

La liaison d'une GPO à une OU est une permission distincte, accordée via `Set-GPPermissions` sur l'OU dans Active Directory, pas sur la GPO elle-même.

```powershell title="Grant-GPOLinkPermission.ps1"
# Delegate GPO link permission on a specific OU to a group
# This permission is set on the OU, not on the GPO

$ouPath    = "OU=Workstations,OU=Paris,DC=contoso,DC=local"
$groupName = "CONTOSO\GPO-Linkers"

# Get the OU object and modify its permissions via AD module
$ou  = Get-ADOrganizationalUnit -Filter { DistinguishedName -eq $ouPath }
$acl = Get-Acl -Path "AD:$($ou.DistinguishedName)"

# The GUID for "Link GPO" right (msDS-GPLink attribute)
$linkGPORight = [System.Guid]::New("f30e3bc2-9ff0-11d1-b603-0000f80367c1")
$adRight      = [System.DirectoryServices.ActiveDirectoryRights]::WriteProperty
$type         = [System.Security.AccessControl.AccessControlType]::Allow

$sid  = (Get-ADGroup $groupName).SID
$rule = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $sid, $adRight, $type, $linkGPORight
)

$acl.AddAccessRule($rule)
Set-Acl -Path "AD:$($ou.DistinguishedName)" -AclObject $acl

Write-Host "GPO link permission granted to $groupName on OU: $ouPath"
```

### :material-delete-alert: Révoquer un droit d'édition

```powershell title="Revoke-GPOEditPermission.ps1"
# Revoke all GPO edit permissions for a specific account (e.g., on departure or compromise)
$accountToRevoke = "CONTOSO\svc-old-tool"

Get-GPO -All | ForEach-Object {
    $gpo    = $_
    $hasPerm = Get-GPPermissions -Guid $gpo.Id -All |
        Where-Object { $_.Trustee.Name -eq $accountToRevoke -and
                       $_.Permission -in @("GpoEdit","GpoEditDeleteModifySecurity") }

    if ($hasPerm) {
        Write-Host "Revoking permissions from $accountToRevoke on: $($gpo.DisplayName)"
        Set-GPPermissions -Guid $gpo.Id `
            -TargetName $accountToRevoke `
            -TargetType "User" `
            -PermissionLevel "None"
    }
}

Write-Host "Permission revocation complete for $accountToRevoke"
```

```title="Résultat attendu"
Revoking permissions from CONTOSO\svc-old-tool on: SEC-Postes-Baseline
Revoking permissions from CONTOSO\svc-old-tool on: WSUS-Configuration
Permission revocation complete for CONTOSO\svc-old-tool
```

!!! danger "Production — Comptes de service avec droits GPO"
    Les comptes de service (`svc-*`) ne doivent **jamais** avoir le droit `GpoEdit` en production. Si un outil de déploiement ou de monitoring nécessite d'accéder aux paramètres GPO, accordez uniquement `GpoRead` (lecture). Si la modification automatisée est impérative, isolez cette action dans un pipeline CI/CD avec un compte dédié audité (voir [Chapitre 23 — Automatisation CI/CD](23-automation-cicd.md)).

!!! quote "En résumé"
    - Déléguer `GpoEdit` par GPO spécifique, jamais sur toutes les GPO du domaine
    - Le droit de liaison de GPO se configure sur l'OU, pas sur la GPO — c'est une permission distincte
    - Révoquez immédiatement les droits d'édition des comptes qui quittent l'équipe ou sont suspectés de compromission
    - Les comptes de service n'ont pas leur place parmi les éditeurs de GPO

---

## Vérification post-déploiement

Après toute modification des permissions GPO ou du hardening SYSVOL, exécutez ce bloc de vérification complet.

```powershell title="Verify-GPOSecurityPosture.ps1"
# Full GPO security posture check — run after any permission or hardening change
Import-Module GroupPolicy, ActiveDirectory

$domain  = $env:USERDNSDOMAIN
$dcPDC   = (Get-ADDomain).PDCEmulator
$results = @()

Write-Host "=== GPO Security Posture Check ===" -ForegroundColor Cyan
Write-Host "Domain : $domain"
Write-Host "PDC    : $dcPDC"
Write-Host ""

# 1. SMB Signing status on all DCs
Write-Host "--- SMB Signing ---"
(Get-ADDomainController -Filter *).Name | ForEach-Object {
    $smb = Get-SmbServerConfiguration -CimSession $_ -EA SilentlyContinue
    $status = if ($smb.RequireSecuritySignature) { "OK" } else { "FAIL" }
    Write-Host "  $_ : RequireSecuritySignature = $($smb.RequireSecuritySignature) [$status]"
}

# 2. CREATOR OWNER check on GPOs
Write-Host ""
Write-Host "--- CREATOR OWNER on GPOs ---"
$creatorOwnerFound = Get-GPO -All | ForEach-Object {
    Get-GPPermissions -Guid $_.Id -All |
        Where-Object { $_.Trustee.Name -eq "CREATOR OWNER" } |
        ForEach-Object { $gpo = $_; Write-Output $gpo }
}
if ($creatorOwnerFound) {
    Write-Warning "  CREATOR OWNER found on $($creatorOwnerFound.Count) GPO permission entries"
} else {
    Write-Host "  OK — No CREATOR OWNER entries found"
}

# 3. Audit Directory Service Changes
Write-Host ""
Write-Host "--- Audit Directory Service Changes ---"
$auditStatus = Invoke-Command -ComputerName $dcPDC -ScriptBlock {
    auditpol /get /subcategory:"Directory Service Changes"
}
Write-Host ($auditStatus | Select-String "Directory Service Changes")

# 4. gPCFileSysPath integrity
Write-Host ""
Write-Host "--- gPCFileSysPath Integrity ---"
$sysvolBase = "\\$domain\SYSVOL\$domain\Policies"
$anomalies  = Get-ADObject -Filter { objectClass -eq "groupPolicyContainer" } `
    -Properties gPCFileSysPath, displayName |
    Where-Object {
        $_.gPCFileSysPath -and
        -not $_.gPCFileSysPath.StartsWith($sysvolBase, [System.StringComparison]::OrdinalIgnoreCase)
    }

if ($anomalies) {
    Write-Warning "  ANOMALY: $($anomalies.Count) gPCFileSysPath outside SYSVOL"
    $anomalies | Select-Object DisplayName, gPCFileSysPath | Format-Table -AutoSize
} else {
    Write-Host "  OK — All gPCFileSysPath values point to SYSVOL"
}

Write-Host ""
Write-Host "=== Check complete ===" -ForegroundColor Cyan
```

```title="Résultat attendu (environnement correctement durci)"
=== GPO Security Posture Check ===
Domain : contoso.local
PDC    : DC01.contoso.local

--- SMB Signing ---
  DC01 : RequireSecuritySignature = True [OK]
  DC02 : RequireSecuritySignature = True [OK]

--- CREATOR OWNER on GPOs ---
  OK — No CREATOR OWNER entries found

--- Audit Directory Service Changes ---
  Directory Service Changes               Success and Failure

--- gPCFileSysPath Integrity ---
  OK — All gPCFileSysPath values point to SYSVOL

=== Check complete ===
```


!!! quote "En résumé"
    - Après toute modification des permissions GPO ou du hardening SYSVOL, exécutez ce bloc de vérification complet.
    - Validez toujours le résultat sur un poste ou un utilisateur réellement dans le périmètre avant d’élargir.
    - Conservez les commandes et résultats de contrôle comme preuve de conformité post-déploiement.
---

## Référence : tableau des mesures de sécurité GPO

| Mesure | Priorité | Vérification | Référence |
|---|---|---|---|
| SMB Signing obligatoire sur tous les DCs | Critique | `Get-SmbServerConfiguration` | Section SYSVOL hardening |
| Audit Directory Service Changes activé | Critique | `auditpol /get /subcategory:"..."` | [Ch. 06 — Audit](06-audit-conformite.md) |
| ACL GPO auditées et liste blanche maintenue | Haute | `Get-GPPermissions -All` | Section ACL GPO |
| CREATOR OWNER absent des GPO de production | Haute | Script de nettoyage | Section ACL GPO |
| SACL sur les dossiers SYSVOL `{GUID}` | Haute | `Get-Acl -Audit` | Section monitoring |
| gPCFileSysPath vérifié périodiquement | Critique | Script d'intégrité | Section GPO Hijacking |
| Chemins UNC FQDN dans les GPO | Haute | Revue manuelle des scripts GPO | Section SYSVOL hardening |
| Comptes de service exclus des éditeurs GPO | Haute | Script Find-UnexpectedGPOEditors | Section moindre privilège |


!!! quote "En résumé"
    - SMB Signing obligatoire sur tous les DCs : Section SYSVOL hardening.
    - Audit Directory Service Changes activé : Ch. 06 — Audit.
    - ACL GPO auditées et liste blanche maintenue : Section ACL GPO.
    - CREATOR OWNER absent des GPO de production : Section ACL GPO.
    - Cette synthèse condense référence : tableau des mesures de sécurité gpo en aide de décision rapide.
---

## Cross-références

| Sujet | Référence |
|---|---|
| Modèle de délégation à quatre rôles et groupes AD | [Ch. 02 — Gouvernance](02-gouvernance.md) |
| Audit GPO avec événements 5136 et 4719 | [Ch. 06 — Audit et conformité](06-audit-conformite.md) |
| Filtrage GPO et permissions sur les objets GPO | [Bible GPO — Ch. 09 — Filtrage](../bible-gpo/09-filtrage.md) |
| Intégration dans un contexte Zero Trust | [Ch. 25 — Zero Trust](25-zero-trust.md) |
| Automatisation CI/CD — pipeline de validation sécurité | [Ch. 23 — Automatisation CI/CD](23-automation-cicd.md) |

!!! quote "En résumé"
    - À relire : Modèle de délégation à quatre rôles et groupes AD → Ch. 02 — Gouvernance.
    - À relire : Audit GPO avec événements 5136 et 4719 → Ch. 06 — Audit et conformité.
    - À relire : Filtrage GPO et permissions sur les objets GPO → Bible GPO — Ch. 09 — Filtrage.
    - À relire : Intégration dans un contexte Zero Trust → Ch. 25 — Zero Trust.
    - À relire : Ch. 02 — Gouvernance.
