---
description: Azure AD Hybrid Join et GPO cloud — AAD Connect, Cloud Policy, co-management Intune, Conditional Access
tags: [gpo-admins, azure-ad, hybrid-join, intune, co-management, cloud-policy, conditional-access]
---

# Azure AD Hybrid Join et GPO cloud

!!! abstract "Ce que vous allez apprendre"
    - Comprendre ce qu'est le Hybrid Join : deux identités sur une même machine, et pourquoi `gpsvc` continue de fonctionner normalement
    - Configurer les GPO qui déclenchent et sécurisent l'enrôlement Azure AD sans toucher au domaine on-prem existant
    - Distinguer Cloud Policy (Microsoft 365 Apps) des GPO traditionnelles et gérer leur coexistence sans conflit silencieux
    - Mettre en place le co-management Intune et basculer les workloads en production avec un risque maîtrisé
    - Identifier et résoudre le deadlock Hybrid Join / Conditional Access avant qu'il bloque vos utilisateurs en production

!!! tip "Si vous ne retenez qu'une chose"
    Le Hybrid Join n'est pas une migration : c'est une **double appartenance**. La machine reste jointe au domaine AD — `gpsvc` applique les GPO normalement — ET elle est enregistrée dans Azure AD. Les deux mondes coexistent. Le danger est ailleurs : une GPO qui désactive l'enrôlement MDM (`DisableMDMEnrollment = 1`) rend le co-management silencieusement impossible. Vérifiez cette clé avant tout déploiement Intune.

---

## Contexte de production

Les environnements hybrides Azure AD sont aujourd'hui la norme dans les entreprises en transition vers le cloud. L'Active Directory on-prem reste en place pour les applications legacy, les ressources fichiers et les scripts existants. Azure AD arrive en parallèle pour le SSO M365, le Conditional Access et la gestion moderne des postes.

La tentation est de traiter Hybrid Join comme un projet autonome. C'est une erreur. Hybrid Join touche simultanément à l'infrastructure AD, à Azure AD Connect, aux politiques de sécurité et à la gestion des postes. Un seul paramètre mal configuré — un SCP absent, une GPO MDM bloquante, un tenant mal ciblé — et l'enrôlement échoue sans message d'erreur évident.

Ce chapitre documente chaque étape avec les chemins exacts, les clés de registre impliquées et les scripts de vérification.

---

## Hybrid Join : deux identités, une seule machine

### :material-identifier: Ce que signifie concrètement "Hybrid Join"

Un poste Hybrid Joined possède **deux identités distinctes** :

- **Identité locale** : SID généré par le domaine AD on-prem, stocké dans SAM
- **Identité cloud** : Azure AD Device Object, identifié par un `DeviceId` GUID et un certificat machine (`MS-Organization-Access`)

Ces deux identités sont liées par Azure AD Connect. Quand AAD Connect synchronise le compte machine depuis AD, Azure AD crée l'objet Device correspondant.

La machine est toujours membre du domaine. `gpsvc` (Group Policy Client Service) fonctionne exactement comme avant : au démarrage, au login, toutes les 90 minutes. Rien ne change côté GPO.

### :material-check-network: Prérequis à vérifier avant de commencer

| Prérequis | Vérification |
|-----------|-------------|
| Azure AD Connect >= 1.1.819 | `Import-Module ADSync; Get-ADSyncScheduler` |
| Sync des comptes ordinateurs activée | Dans AAD Connect : « Computer objects » coché dans la sélection OU |
| SCP (Service Connection Point) dans AD | `Get-ADObject -Filter {objectClass -eq "serviceConnectionPoint"} -SearchBase "CN=Configuration,..."` |
| OS Windows 10 1607+ ou Windows 11 | `winver` ou `(Get-WmiObject Win32_OperatingSystem).BuildNumber` |
| Accès HTTPS sortant vers `login.microsoftonline.com` | Test depuis les machines cibles |
| Tenant Azure AD Premium P1 minimum (pour CA) | Portail Azure AD > Licences |

### :material-cog-sync: Le SCP : point de départ de tout enrôlement

Le SCP (Service Connection Point) est un objet AD qui indique aux machines quel tenant Azure AD cibler pour l'enrôlement.

Sans SCP, les machines ne savent pas où s'enregistrer. L'enrôlement échoue silencieusement.

```powershell title="Créer ou vérifier le SCP via PowerShell"
# Import the AAD Connect module (run on AAD Connect server or with AzureAD module)
Import-Module AzureAD

# Get your tenant ID
$tenantId = (Get-AzureADTenantDetail).ObjectId

# Build the AD configuration partition path
$configNC = (Get-ADRootDSE).configurationNamingContext

# Check if SCP already exists
$scp = Get-ADObject `
    -Filter { objectClass -eq "serviceConnectionPoint" -and name -eq "62a0ff2e-97b9-4513-943f-0d221bd30080" } `
    -SearchBase "CN=Device Registration Configuration,CN=Services,$configNC" `
    -Properties keywords -ErrorAction SilentlyContinue

if ($scp) {
    Write-Host "SCP found. Current keywords:" -ForegroundColor Cyan
    $scp.keywords
} else {
    Write-Host "SCP not found — creating..." -ForegroundColor Yellow

    # Create the SCP
    $scpDN = "CN=62a0ff2e-97b9-4513-943f-0d221bd30080,CN=Device Registration Configuration,CN=Services,$configNC"
    New-ADObject -Name "62a0ff2e-97b9-4513-943f-0d221bd30080" `
        -Type "serviceConnectionPoint" `
        -Path "CN=Device Registration Configuration,CN=Services,$configNC" `
        -OtherAttributes @{
            keywords = @("azureADName:$((Get-AzureADDomain | Where-Object {$_.IsDefault}).Name)", "azureADId:$tenantId")
        }
    Write-Host "SCP created for tenant: $tenantId" -ForegroundColor Green
}
```

```powershell title="Résultat attendu"
SCP found. Current keywords:
azureADName:contoso.onmicrosoft.com
azureADId:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### :material-monitor-check: Vérifier le statut Hybrid Join d'un poste

```powershell title="Vérification complète dsregcmd"
# Run locally on the target machine (elevated not required for status check)
dsregcmd /status
```

```text title="Résultat attendu — poste correctement Hybrid Joined"
+----------------------------------------------------------------------+
| Device State                                                         |
+----------------------------------------------------------------------+
             AzureAdJoined : YES
          EnterpriseJoined : NO
              DomainJoined : YES
                DomainName : CONTOSO
+----------------------------------------------------------------------+
| Device Details                                                       |
+----------------------------------------------------------------------+
                  DeviceId : xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
                Thumbprint : XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
        DeviceCertValidity : [ 2024-01-01 00:00:00.000 UTC -- 2026-01-01 00:00:00.000 UTC ]
+----------------------------------------------------------------------+
| SSO State                                                            |
+----------------------------------------------------------------------+
                AzureAdPrt : YES
```

Les trois champs clés sont `AzureAdJoined: YES`, `DomainJoined: YES` et `AzureAdPrt: YES`. Un `AzureAdPrt: NO` indique un problème de renouvellement de token — souvent lié à un accès réseau vers Azure AD bloqué.

```powershell title="Script de rapport Hybrid Join sur un parc de machines"
# Run from a management workstation with RSAT and AzureAD module
$computers = Get-ADComputer -Filter {OperatingSystem -like "Windows 10*" -or OperatingSystem -like "Windows 11*"} `
    -Properties OperatingSystem, LastLogonDate |
    Where-Object { $_.LastLogonDate -gt (Get-Date).AddDays(-30) }

$results = foreach ($computer in $computers) {
    $session = New-PSSession -ComputerName $computer.Name -ErrorAction SilentlyContinue
    if ($session) {
        $status = Invoke-Command -Session $session -ScriptBlock {
            $raw = dsregcmd /status
            [PSCustomObject]@{
                AzureAdJoined = ($raw | Select-String "AzureAdJoined\s+:\s+(\w+)").Matches[0].Groups[1].Value
                DomainJoined  = ($raw | Select-String "DomainJoined\s+:\s+(\w+)").Matches[0].Groups[1].Value
                PRT           = ($raw | Select-String "AzureAdPrt\s+:\s+(\w+)").Matches[0].Groups[1].Value
            }
        }
        Remove-PSSession $session
        [PSCustomObject]@{
            Computer      = $computer.Name
            OS            = $computer.OperatingSystem
            HybridJoined  = $status.AzureAdJoined -eq "YES" -and $status.DomainJoined -eq "YES"
            PRTValid      = $status.PRT -eq "YES"
            LastLogon     = $computer.LastLogonDate
        }
    } else {
        [PSCustomObject]@{
            Computer     = $computer.Name
            OS           = $computer.OperatingSystem
            HybridJoined = "UNREACHABLE"
            PRTValid     = "UNREACHABLE"
            LastLogon    = $computer.LastLogonDate
        }
    }
}

$results | Sort-Object HybridJoined | Format-Table -AutoSize
```

!!! quote "En résumé"
    - Hybrid Join = SID on-prem + Azure AD Device Object. Les GPO continuent de s'appliquer normalement via `gpsvc`.
    - Le SCP dans la partition Configuration d'AD est le prérequis numéro un. Vérifiez-le avant tout.
    - `dsregcmd /status` est la commande de référence pour diagnostiquer l'état d'un poste. `AzureAdJoined: YES` + `DomainJoined: YES` + `AzureAdPrt: YES` = état sain.
    - Un `AzureAdPrt: NO` n'est pas un problème de GPO mais un problème réseau ou de certificat machine.

---

## Cloud Policy : GPO-like pour Microsoft 365

### :material-cloud-check: Ce qu'est Cloud Policy (et ce qu'il n'est pas)

Cloud Policy (Microsoft 365 Apps admin center, anciennement Office Cloud Policy Service) permet d'appliquer des paramètres de configuration aux applications M365 (Word, Excel, Outlook, Teams, etc.) **sans que la machine soit jointe à un domaine**.

L'application se fait via le token AAD de l'utilisateur. Quand l'utilisateur ouvre une application M365, celle-ci vérifie si des politiques Cloud Policy s'appliquent à son compte, et les télécharge.

Cloud Policy ne s'appuie pas sur `gpsvc`. Ce n'est pas une GPO au sens technique du terme.

### :material-scale-balance: Comparaison Cloud Policy vs GPO traditionnelle

| Critère | GPO traditionnelle | Cloud Policy |
|---------|-------------------|-------------|
| Scope | Ordinateurs ou utilisateurs du domaine | Utilisateurs AAD (licence M365) |
| Prérequis machine | Domain join | Aucun (ni domain join, ni AAD join) |
| Déclenchement | `gpsvc` (démarrage, login, 90 min) | À l'ouverture de l'application M365 |
| Stockage | SYSVOL → registre local | Téléchargement AAD → registre local |
| Périmètre de settings | Tout Windows + toutes les apps | Applications M365 uniquement |
| Reporting | Résultats dans GPMC (RSoP, Modeling) | Microsoft 365 Apps admin center |
| Priorité en cas de conflit | GPO l'emporte sur Cloud Policy | Cloud Policy cède aux GPO |
| Offline | Oui (cache SYSVOL) | Délai de grâce 90 jours puis bloqué |
| Granularité ciblage | OU, Groupe de sécurité, WMI filter | Groupe AAD |
| Prix | Inclus AD DS | Inclus M365 Business/Enterprise |

### :material-alert-circle: La règle de coexistence : GPO gagne sur les settings registry-backed

Quand GPO et Cloud Policy configurent le même paramètre, la GPO l'emporte. Ce comportement est documenté par Microsoft mais rarement visible en pratique, car les deux systèmes écrivent au même endroit dans le registre :

```
HKCU\SOFTWARE\Policies\Microsoft\Office\<version>\<app>\
```

Lors du traitement, `gpsvc` écrase la valeur avec celle de la GPO. Cloud Policy, qui s'applique plus tard (à l'ouverture de l'app), trouve la valeur déjà présente et la considère comme une GPO existante — elle ne la remplace pas.

!!! warning "À surveiller"
    Si vous déployez Cloud Policy sur des postes qui ont aussi des GPO ADMX M365, vous pouvez vous retrouver avec des paramètres Cloud Policy **qui ne s'appliquent jamais** sans aucun message d'erreur. Auditez les GPO ADMX Office avant de déployer Cloud Policy. La section "Applied policies" dans le portail M365 Apps admin center indique si une politique est en conflit.

### :material-cog: Configurer Cloud Policy depuis le portail

**Chemin d'accès :** `config.office.com > Personnalisation > Gestion des stratégies`

Les étapes de création d'une politique Cloud Policy :

1. Créer une nouvelle politique, lui donner un nom descriptif
2. Choisir la priorité (entier — plus petit = plus prioritaire en cas de conflit Cloud Policy vs Cloud Policy)
3. Sélectionner un groupe AAD comme scope (utilisateurs ou "Tous les utilisateurs")
4. Chercher et configurer les paramètres ADMX disponibles (plus de 2 300 paramètres)
5. Publier — la politique s'applique au prochain lancement de l'app M365 concernée

```powershell title="Vérifier les paramètres Cloud Policy appliqués sur un poste"
# Check registry keys written by Cloud Policy (HKCU, per user)
$officePolicies = Get-ChildItem "HKCU:\SOFTWARE\Policies\Microsoft\Office" -Recurse -ErrorAction SilentlyContinue |
    Get-ItemProperty |
    Select-Object PSPath, * -ExcludeProperty PSPath, PSParentPath, PSChildName, PSDrive, PSProvider

$officePolicies | Where-Object { $_.PSPath -match "Office" } | Format-List
```

!!! quote "En résumé"
    - Cloud Policy cible les utilisateurs AAD, pas les machines — pas besoin de domain join.
    - GPO ADMX traditionnelle gagne sur Cloud Policy en cas de conflit de registre.
    - Le portail `config.office.com` est l'interface de gestion — aucun outil on-prem nécessaire.
    - Auditez les GPO ADMX Office existantes avant de déployer Cloud Policy pour éviter les paramètres qui ne s'appliquent jamais.

---

## Co-management Intune : qui gère quoi

### :material-merge: Le principe du co-management

Le co-management MECM + Intune consiste à ce qu'un poste soit géré **simultanément** par MECM (Configuration Manager) et par Intune. C'est un état de transition, pas une architecture cible permanente.

Le co-management se déclenche via l'enrôlement MDM automatique du poste Hybrid Joined. Cet enrôlement peut être forcé par GPO.

**Prérequis :**
- Poste Hybrid Joined (Azure AD + Domain joined)
- MECM en place avec le site configuré pour le co-management
- Licence Intune (incluse dans EMS E3/E5 ou M365 E3/E5)

### :material-table-arrow-right: Tableau des workloads co-management

| Workload | Géré par MECM (défaut) | Géré par Intune (après bascule) |
|----------|------------------------|--------------------------------|
| Compliance Policies | Non disponible MECM | Règles de conformité Intune |
| Device Configuration | GPO + MECM DCM | Profils de configuration Intune |
| Windows Update | SUP MECM / WSUS | WUfB policies Intune |
| Endpoint Protection | MECM Endpoint Protection | Microsoft Defender Intune |
| Resource Access Policies | Certificats MECM | Profils SCEP/PKCS Intune |
| Client Apps | MECM Software Center | Portail d'entreprise Intune |
| Office Click-to-Run | ODT via MECM | Microsoft 365 Apps via Intune |

La bascule d'un workload est granulaire : vous pouvez basculer "Compliance" vers Intune tout en gardant "Device Configuration" sous MECM. C'est le principe du curseur de workload dans la console MECM.

### :material-cog-play: Forcer l'enrôlement MDM via GPO

**Chemin GPMC :**
`Configuration ordinateur > Stratégies > Modèles d'administration > Windows Components > MDM`

**Paramètre :** `Enable automatic MDM enrollment using default Azure AD credentials`

Ou directement via registre (pour les environnements sans ADMX MDM) :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM
Valeur : AutoEnrollMDM (REG_DWORD) = 1
Valeur : UseAADCredentialType (REG_DWORD) = 1
```

```powershell title="Déclencher manuellement l'enrôlement MDM (test)"
# Trigger MDM enrollment manually — useful after GPO push to validate
$enrollmentPath = "HKLM:\SOFTWARE\Microsoft\Enrollments"
$mdmTask = Get-ScheduledTask -TaskPath "\Microsoft\Windows\EnterpriseMgmt\*" |
    Where-Object { $_.TaskName -like "*Schedule*" } |
    Select-Object -First 1

if ($mdmTask) {
    Start-ScheduledTask -TaskPath $mdmTask.TaskPath -TaskName $mdmTask.TaskName
    Write-Host "MDM enrollment task triggered: $($mdmTask.TaskName)" -ForegroundColor Green
} else {
    Write-Host "No MDM enrollment scheduled task found — enrollment may not be configured" -ForegroundColor Yellow
    # Alternative: use DeviceEnroller directly
    & "$env:windir\system32\DeviceEnroller.exe" /c /AutoEnrollMDM
}
```

### :material-check-circle: Vérifier le statut d'enrôlement Intune

```powershell title="Vérifier l'enrôlement MDM complet"
# Check enrollment status from the local machine
function Get-MDMEnrollmentStatus {
    $enrollmentKey = "HKLM:\SOFTWARE\Microsoft\Enrollments"
    $enrollments = Get-ChildItem $enrollmentKey -ErrorAction SilentlyContinue |
        Get-ItemProperty |
        Where-Object { $_.EnrollmentState -eq 1 } # 1 = enrolled

    if ($enrollments) {
        foreach ($e in $enrollments) {
            [PSCustomObject]@{
                EnrollmentId      = $e.PSChildName
                ProviderID        = $e.ProviderID
                UPN               = $e.UPN
                AADResourceID     = $e.AADResourceID
                EnrollmentState   = "Enrolled"
                LastSyncAttempt   = $e.LastSyncAttemptTime
            }
        }
    } else {
        Write-Warning "No active MDM enrollment found on this machine"
        Write-Host "Check: HKLM\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM\DisableMDMEnrollment" -ForegroundColor Cyan
    }
}

Get-MDMEnrollmentStatus
```

```text title="Résultat attendu — machine enrôlée"
EnrollmentId    : xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
ProviderID      : MS DM Server
UPN             : user@contoso.com
AADResourceID   : https://manage.microsoft.com
EnrollmentState : Enrolled
LastSyncAttempt : 04/05/2026 09:12:33
```

```powershell title="Forcer une synchronisation Intune"
# Force an immediate MDM sync (useful after policy change)
$session = New-CimSession
Invoke-CimMethod -Namespace "root\cimv2\mdm\dmmap" `
    -ClassName "MDM_Client" `
    -MethodName "StartMDMEnrollmentCheckForUpdates" `
    -CimSession $session -ErrorAction Stop
Write-Host "MDM sync triggered" -ForegroundColor Green
```

!!! quote "En résumé"
    - Le co-management MECM + Intune repose sur l'enrôlement MDM automatique du poste Hybrid Joined.
    - La bascule des workloads est granulaire — chaque workload peut être basculé indépendamment.
    - L'enrôlement MDM se force via GPO (`AutoEnrollMDM = 1`) ou via la tâche planifiée `DeviceEnroller`.
    - Vérifiez `HKLM\SOFTWARE\Microsoft\Enrollments\*` pour confirmer qu'un enrôlement actif existe (`EnrollmentState = 1`).

---

## Conditional Access et GPO : dépendances croisées

### :material-shield-lock: Le rôle des GPO dans le Conditional Access

Le Conditional Access (CA) d'Azure AD bloque ou autorise l'accès aux ressources cloud selon des conditions : type de device, conformité Intune, Hybrid Join, MFA, etc.

La condition **"Hybrid Azure AD Joined"** est l'une des plus utilisées. Elle garantit que seules les machines du domaine approuvé accèdent aux ressources M365.

**La GPO intervient en amont du CA** : elle configure le poste pour être Hybrid Joined, ce qui satisfait la condition CA. Sans GPO correctement configurée, le Hybrid Join n'aboutit pas, et la condition CA échoue.

### :material-alert: Le deadlock Hybrid Join / Conditional Access

C'est le scénario le plus dangereux de ce chapitre.

**Situation :** Une politique CA exige que le device soit Hybrid Joined pour accéder à Azure AD. Mais le device n'est pas encore Hybrid Joined. Pour compléter le Hybrid Join, la machine doit s'authentifier auprès d'Azure AD. Azure AD bloque l'authentification car le device n'est pas Hybrid Joined.

**Résultat :** La machine tourne en rond. Elle ne peut pas devenir Hybrid Joined, donc elle ne peut jamais satisfaire la condition CA.

```
Machine non Hybrid Joined
        ↓
Tentative d'enrôlement AAD
        ↓
Azure AD exige Hybrid Join (policy CA)
        ↓
Enrôlement bloqué
        ↓
Machine reste non Hybrid Joined
        ↑___________/
```

!!! danger "Production"
    Ce deadlock est **réel et fréquent** lors des premiers déploiements CA. Il touche typiquement les postes neufs qui rejoignent le domaine et n'ont jamais eu de session Azure AD, ou les postes réinstallés. N'activez jamais une politique CA "Require Hybrid Joined Device" en mode `Block` (vs `Report-only`) avant d'avoir vérifié que 100% du parc ciblé est déjà Hybrid Joined.

### :material-wrench: Détection et résolution du deadlock

```powershell title="Détection — identifier les postes AD non encore Hybrid Joined"
# Run from a management workstation with RSAT + AzureAD module
# Identify AD computers with no corresponding AAD Device object

Import-Module AzureAD
Connect-AzureAD

# Get all AD computers (workstations, last logged on within 60 days)
$adComputers = Get-ADComputer -Filter {
    (OperatingSystem -like "Windows 10*" -or OperatingSystem -like "Windows 11*")
} -Properties LastLogonDate |
    Where-Object { $_.LastLogonDate -gt (Get-Date).AddDays(-60) }

# Get all AAD devices
$aadDevices = Get-AzureADDevice -All $true |
    Where-Object { $_.DeviceTrustType -eq "ServerAd" } # ServerAd = Hybrid Joined

$aadNames = $aadDevices.DisplayName | ForEach-Object { $_.ToLower() }

# Find AD computers not present in AAD
$notHybridJoined = $adComputers |
    Where-Object { $_.Name.ToLower() -notin $aadNames }

Write-Host "Machines in AD but NOT Hybrid Joined in AAD: $($notHybridJoined.Count)" -ForegroundColor Yellow
$notHybridJoined | Select-Object Name, OperatingSystem, LastLogonDate | Sort-Object LastLogonDate -Descending
```

**Procédure de résolution du deadlock :**

1. **Mode Report-only d'abord** — Basculer la politique CA en `Report-only` pour observer sans bloquer. Cela brise le deadlock immédiatement.
2. **Exclure un groupe de grâce** — Créer un groupe AAD `CA-Exclusion-HybridJoin-Grace` et exclure ce groupe de la politique CA. Ajouter les postes bloqués à ce groupe via leur Device Object ou via les comptes utilisateurs concernés.
3. **Forcer le Hybrid Join** — Sur les postes bloqués, exécuter `dsregcmd /join` (nécessite une connectivité réseau interne ou VPN pour atteindre AD + Internet pour atteindre Azure AD).
4. **Vérifier** — `dsregcmd /status` confirme `AzureAdJoined: YES`.
5. **Retirer du groupe de grâce** — Une fois Hybrid Joined, retirer le poste (ou le compte utilisateur) du groupe d'exclusion.
6. **Repasser en Block** — Une fois le parc assaini, repasser la politique CA en mode `Grant : Require Hybrid Azure AD Joined`.

```powershell title="Forcer le Hybrid Join sur un poste (run as SYSTEM or elevated)"
# Force device registration — run on the affected machine
# Requires network access to both on-prem AD and Azure AD endpoints
dsregcmd /join

# Wait a few seconds, then verify
Start-Sleep -Seconds 10
dsregcmd /status | Select-String "(AzureAdJoined|DomainJoined|AzureAdPrt)"
```

!!! quote "En résumé"
    - GPO prépare le Hybrid Join, qui est la condition pour satisfaire les politiques CA "Require Hybrid Joined".
    - Le deadlock CA survient quand CA bloque l'authentification AAD avant que le poste soit Hybrid Joined.
    - Toujours déployer les politiques CA en mode `Report-only` d'abord. Identifier et traiter les postes non encore Hybrid Joined avant de passer en `Block`.
    - Le groupe d'exclusion de grâce est le seul moyen de débloquer des machines sans désactiver la politique CA.

---

## GPO qui préparent le cloud

### :material-cloud-upload: Forcer le Workplace Join Azure AD via GPO

Pour les environnements où Hybrid Join complet n'est pas encore déployé, le **Workplace Join** (aussi appelé Azure AD Registration) est une étape intermédiaire. Il enregistre l'appareil dans Azure AD sans en faire un device entièrement géré.

**Chemin GPMC :**
`Configuration ordinateur > Stratégies > Modèles d'administration > Windows Components > Workplace Join`

**Paramètre :** `Automatically workplace join computers`

Ou via registre :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\WorkplaceJoin
Valeur : autoWorkplaceJoin (REG_DWORD) = 1
```

### :material-key-variant: Préconfigurer le SSO via la clé CDJ

La clé `CDJ` (Client Device Join) dans le registre pré-configure les paramètres d'enrôlement Azure AD. Elle accélère et fiabilise l'enrôlement sur les réseaux où le SCP AD n'est pas accessible (ex. machines hors domaine VPN).

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\CDJ\AAD
Valeur : TenantId   (REG_SZ) = <your-tenant-GUID>
Valeur : TenantName (REG_SZ) = <your-tenant-domain.onmicrosoft.com>
```

Cette clé peut être déployée via GPO Preferences > Registry, ce qui la rend disponible avant même que la machine ait pu consulter le SCP.

```powershell title="Déployer la clé CDJ via GPO Preferences (script de référence)"
# This script sets CDJ registry values directly — use as reference for GPO Preferences configuration
# In GPMC: Computer Config > Preferences > Windows Settings > Registry

$tenantId   = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"  # Replace with your tenant GUID
$tenantName = "contoso.onmicrosoft.com"               # Replace with your tenant domain

$cdjPath = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\CDJ\AAD"

if (-not (Test-Path $cdjPath)) {
    New-Item -Path $cdjPath -Force | Out-Null
}

Set-ItemProperty -Path $cdjPath -Name "TenantId"   -Value $tenantId   -Type String
Set-ItemProperty -Path $cdjPath -Name "TenantName" -Value $tenantName -Type String

Write-Host "CDJ registry keys configured for tenant: $tenantName ($tenantId)" -ForegroundColor Green
```

### :material-sync: Configurer le SCP via PowerShell (au lieu d'AAD Connect)

Dans certains environnements, AAD Connect n'est pas disponible sur le serveur où vous travaillez. Le SCP peut être créé directement via PowerShell AD.

```powershell title="Créer le SCP sans AAD Connect"
# Run on a Domain Controller or machine with RSAT AD DS Tools
# Requires write access to the Configuration partition

$tenantId   = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$tenantName = "contoso.onmicrosoft.com"

$configNC = (Get-ADRootDSE).configurationNamingContext
$scpBaseDN = "CN=Device Registration Configuration,CN=Services,$configNC"

# Ensure the parent container exists
try {
    Get-ADObject -Identity $scpBaseDN | Out-Null
} catch {
    New-ADObject -Name "Device Registration Configuration" `
        -Type "container" `
        -Path "CN=Services,$configNC"
    Write-Host "Created parent container" -ForegroundColor Yellow
}

# Create the SCP object
$scpName = "62a0ff2e-97b9-4513-943f-0d221bd30080"
$existing = Get-ADObject -Filter { name -eq $scpName } -SearchBase $scpBaseDN -ErrorAction SilentlyContinue

if (-not $existing) {
    New-ADObject -Name $scpName `
        -Type "serviceConnectionPoint" `
        -Path $scpBaseDN `
        -OtherAttributes @{
            keywords = @("azureADName:$tenantName", "azureADId:$tenantId")
        }
    Write-Host "SCP created successfully" -ForegroundColor Green
} else {
    Set-ADObject -Identity $existing.DistinguishedName `
        -Replace @{ keywords = @("azureADName:$tenantName", "azureADId:$tenantId") }
    Write-Host "SCP updated for tenant: $tenantName" -ForegroundColor Cyan
}
```

```powershell title="Résultat attendu"
SCP created successfully
```

### :material-check-all: Checklist GPO "cloud-ready"

| Action | Chemin GPO ou Clé registre | Valeur |
|--------|---------------------------|--------|
| Workplace Join automatique | `HKLM\SOFTWARE\Policies\Microsoft\Windows\WorkplaceJoin\autoWorkplaceJoin` | 1 |
| CDJ TenantId | `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\CDJ\AAD\TenantId` | GUID tenant |
| CDJ TenantName | `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\CDJ\AAD\TenantName` | domain.onmicrosoft.com |
| Enrôlement MDM auto | `HKLM\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM\AutoEnrollMDM` | 1 |
| UseAADCredentialType | `HKLM\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM\UseAADCredentialType` | 1 |
| DisableMDMEnrollment absent ou à 0 | `HKLM\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM\DisableMDMEnrollment` | 0 ou absent |

!!! quote "En résumé"
    - La clé CDJ déploie le tenant ID avant que le SCP AD soit consultable — utile pour les machines hors réseau interne.
    - Le Workplace Join est un enrôlement partiel : il précède ou complète le Hybrid Join dans les scénarios progressifs.
    - Le SCP peut être créé directement via PowerShell AD sans passer par l'assistant AAD Connect.
    - La checklist ci-dessus couvre tous les paramètres registry nécessaires — déployez-les via GPO Preferences > Registry.

---

## Piège de production : DisableMDMEnrollment bloque le co-management

### :material-bomb: Le scénario

C'est l'une des erreurs les plus fréquentes en déploiement hybride. Un administrateur, lors d'un audit de sécurité ou d'un durcissement de poste, a activé la clé suivante pour bloquer l'enrôlement MDM non autorisé :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM
Valeur : DisableMDMEnrollment (REG_DWORD) = 1
```

Cette clé est parfaitement légitime dans un environnement purement on-prem. Mais elle bloque **silencieusement** l'enrôlement Intune sur les postes Hybrid Joined destinés au co-management.

Aucun message d'erreur dans les logs Windows standard. La machine semble normale, elle est Hybrid Joined, mais elle ne s'enrôle jamais dans Intune.

!!! danger "Production"
    La valeur `DisableMDMEnrollment = 1` est parfois héritée d'une GPO de sécurité ancienne, d'un benchmark CIS ou d'un template STIG. Elle peut s'appliquer via une GPO de niveau domaine ou de site, et être masquée par des GPO locales. Cherchez-la en priorité quand l'enrôlement Intune n'aboutit pas sur des postes Hybrid Joined apparemment sains.

### :material-magnify: Script de détection

```powershell title="Détecter DisableMDMEnrollment sur le parc"
# Run from management workstation — scans all Windows 10/11 machines
# Requires WinRM access to target machines

$computers = Get-ADComputer -Filter {
    OperatingSystem -like "Windows 10*" -or OperatingSystem -like "Windows 11*"
} -Properties LastLogonDate |
    Where-Object { $_.LastLogonDate -gt (Get-Date).AddDays(-30) }

$mdmKey    = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM"
$valueName = "DisableMDMEnrollment"

$results = foreach ($computer in $computers) {
    $session = New-PSSession -ComputerName $computer.Name -ErrorAction SilentlyContinue
    if ($session) {
        $status = Invoke-Command -Session $session -ScriptBlock {
            param($key, $val)
            $v = Get-ItemProperty -Path $key -Name $val -ErrorAction SilentlyContinue
            [PSCustomObject]@{
                KeyExists   = (Test-Path $key)
                ValueExists = ($null -ne $v)
                Value       = if ($v) { $v.$val } else { "NOT SET" }
                Blocked     = ($v -and $v.$val -eq 1)
            }
        } -ArgumentList $mdmKey, $valueName
        Remove-PSSession $session

        [PSCustomObject]@{
            Computer    = $computer.Name
            OS          = $computer.OperatingSystem
            KeyPresent  = $status.KeyExists
            ValueSet    = $status.ValueExists
            CurrentValue = $status.Value
            MDMBlocked  = $status.Blocked
        }
    } else {
        [PSCustomObject]@{
            Computer    = $computer.Name
            OS          = $computer.OperatingSystem
            KeyPresent  = "UNREACHABLE"
            ValueSet    = "UNREACHABLE"
            CurrentValue = "UNREACHABLE"
            MDMBlocked  = "UNREACHABLE"
        }
    }
}

# Report
$blocked = $results | Where-Object { $_.MDMBlocked -eq $true }
Write-Host "`nMachines with MDM enrollment BLOCKED: $($blocked.Count)" -ForegroundColor Red
$blocked | Format-Table Computer, OS, CurrentValue -AutoSize

$results | Export-Csv -Path "$env:TEMP\MDMEnrollmentAudit_$(Get-Date -Format yyyyMMdd).csv" -NoTypeInformation -Encoding UTF8
Write-Host "Full report exported to: $env:TEMP\MDMEnrollmentAudit_$(Get-Date -Format yyyyMMdd).csv" -ForegroundColor Cyan
```

```text title="Résultat attendu"
Machines with MDM enrollment BLOCKED: 3

Computer       OS                   CurrentValue
--------       --                   ------------
DESKTOP-AAA    Windows 10 Pro       1
LAPTOP-BBB     Windows 11 Pro       1
WORKSTATION-C  Windows 10 Enterprise 1

Full report exported to: C:\Users\Admin\AppData\Local\Temp\MDMEnrollmentAudit_20260405.csv
```

### :material-wrench-cog: Remédiation via GPO

**Option 1 : Supprimer la valeur via GPO Preferences (méthode recommandée)**

Chemin GPMC : `Configuration ordinateur > Préférences > Paramètres Windows > Registre`

- Action : **Delete**
- Clé : `HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM`
- Valeur : `DisableMDMEnrollment`

Cela supprime la valeur sur les machines qui la reçoivent, permettant à l'enrôlement MDM de reprendre.

**Option 2 : Forcer à 0 via GPO Preferences**

Si vous ne pouvez pas supprimer la valeur (conflit de GPO), forcez-la à 0 :

- Action : **Replace**
- Type : `REG_DWORD`
- Valeur : `0`

```powershell title="Remédiation locale — désactiver le blocage MDM (test/urgence)"
# Run elevated on the affected machine
# Use this for immediate unblocking — then deploy the fix via GPO

$mdmKeyPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM"

if (Test-Path $mdmKeyPath) {
    $currentValue = Get-ItemProperty -Path $mdmKeyPath -Name "DisableMDMEnrollment" -ErrorAction SilentlyContinue

    if ($currentValue -and $currentValue.DisableMDMEnrollment -eq 1) {
        Set-ItemProperty -Path $mdmKeyPath -Name "DisableMDMEnrollment" -Value 0 -Type DWord
        Write-Host "DisableMDMEnrollment set to 0 — MDM enrollment re-enabled" -ForegroundColor Green

        # Trigger enrollment
        Write-Host "Triggering MDM enrollment..." -ForegroundColor Cyan
        & "$env:windir\system32\DeviceEnroller.exe" /c /AutoEnrollMDM
    } else {
        Write-Host "DisableMDMEnrollment is not set to 1 — no action needed" -ForegroundColor Green
    }
} else {
    Write-Host "MDM policy key does not exist — enrollment should be possible" -ForegroundColor Green
}
```

### :material-check-all: Vérification post-remédiation

```powershell title="Vérification complète post-remédiation"
Write-Host "=== MDM ENROLLMENT VERIFICATION ===" -ForegroundColor Cyan

# 1. Check registry
$mdmKey = Get-ItemProperty `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM" `
    -ErrorAction SilentlyContinue
Write-Host "`n[1] DisableMDMEnrollment:" -ForegroundColor Yellow
if ($mdmKey -and $mdmKey.DisableMDMEnrollment -eq 1) {
    Write-Host "    BLOCKED (value = 1)" -ForegroundColor Red
} else {
    Write-Host "    OK (value = 0 or not set)" -ForegroundColor Green
}

# 2. Check Hybrid Join status
Write-Host "`n[2] Hybrid Join status:" -ForegroundColor Yellow
$dsreg = dsregcmd /status
$azJoined = ($dsreg | Select-String "AzureAdJoined\s+:\s+(\w+)").Matches[0].Groups[1].Value
$domJoined = ($dsreg | Select-String "DomainJoined\s+:\s+(\w+)").Matches[0].Groups[1].Value
Write-Host "    AzureAdJoined : $azJoined" -ForegroundColor $(if ($azJoined -eq "YES") {"Green"} else {"Red"})
Write-Host "    DomainJoined  : $domJoined" -ForegroundColor $(if ($domJoined -eq "YES") {"Green"} else {"Red"})

# 3. Check MDM enrollment
Write-Host "`n[3] Active MDM enrollments:" -ForegroundColor Yellow
$enrollments = Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Enrollments" -ErrorAction SilentlyContinue |
    Get-ItemProperty | Where-Object { $_.EnrollmentState -eq 1 }
if ($enrollments) {
    $enrollments | ForEach-Object {
        Write-Host "    Enrolled — Provider: $($_.ProviderID), UPN: $($_.UPN)" -ForegroundColor Green
    }
} else {
    Write-Host "    No active enrollment found" -ForegroundColor Red
    Write-Host "    Wait 5-10 min after enabling enrollment, then re-run this check" -ForegroundColor Yellow
}

Write-Host "`n=== END VERIFICATION ===" -ForegroundColor Cyan
```

!!! quote "En résumé"
    - `DisableMDMEnrollment = 1` bloque silencieusement l'enrôlement Intune — aucun log d'erreur visible.
    - Cette valeur provient souvent d'une ancienne GPO de durcissement (CIS, STIG) appliquée avant le projet cloud.
    - La remédiation est simple : supprimer ou mettre à 0 via GPO Preferences, puis déclencher `DeviceEnroller.exe /c /AutoEnrollMDM`.
    - Le script d'audit permet de cartographier en 10 minutes tous les postes bloqués sur le parc.

---

## Vérification globale post-déploiement

### :material-clipboard-check: Checklist de validation

Après avoir déployé les GPO Hybrid Join et les paramètres co-management, exécutez ce script de validation globale sur un échantillon de postes pilotes avant la production large.

```powershell title="Validation complète Hybrid Join + MDM sur un poste"
function Invoke-HybridJoinValidation {
    param(
        [string]$ComputerName = $env:COMPUTERNAME
    )

    Write-Host "`n===== HYBRID JOIN VALIDATION : $ComputerName =====" -ForegroundColor Cyan
    $score = 0
    $maxScore = 7

    # Test 1: Domain joined
    $domain = (Get-WmiObject Win32_ComputerSystem).Domain
    $isDomainJoined = $domain -ne "WORKGROUP" -and $domain -ne $env:COMPUTERNAME
    Write-Host "[$(if($isDomainJoined){'PASS'}else{'FAIL'})] Domain joined: $domain" `
        -ForegroundColor $(if($isDomainJoined){'Green'}else{'Red'})
    if ($isDomainJoined) { $score++ }

    # Test 2: Azure AD joined
    $dsreg = dsregcmd /status
    $aadJoined = ($dsreg | Select-String "AzureAdJoined\s+:\s+YES").Count -gt 0
    Write-Host "[$(if($aadJoined){'PASS'}else{'FAIL'})] Azure AD Joined" `
        -ForegroundColor $(if($aadJoined){'Green'}else{'Red'})
    if ($aadJoined) { $score++ }

    # Test 3: PRT valid
    $prtValid = ($dsreg | Select-String "AzureAdPrt\s+:\s+YES").Count -gt 0
    Write-Host "[$(if($prtValid){'PASS'}else{'FAIL'})] Primary Refresh Token valid" `
        -ForegroundColor $(if($prtValid){'Green'}else{'Red'})
    if ($prtValid) { $score++ }

    # Test 4: MDM enrollment not blocked
    $mdmKey = Get-ItemProperty `
        -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM" `
        -ErrorAction SilentlyContinue
    $mdmNotBlocked = -not ($mdmKey -and $mdmKey.DisableMDMEnrollment -eq 1)
    Write-Host "[$(if($mdmNotBlocked){'PASS'}else{'FAIL'})] MDM enrollment not blocked" `
        -ForegroundColor $(if($mdmNotBlocked){'Green'}else{'Red'})
    if ($mdmNotBlocked) { $score++ }

    # Test 5: MDM enrolled
    $enrolled = Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Enrollments" -ErrorAction SilentlyContinue |
        Get-ItemProperty | Where-Object { $_.EnrollmentState -eq 1 }
    $isEnrolled = $null -ne $enrolled
    Write-Host "[$(if($isEnrolled){'PASS'}else{'WARN'})] Active MDM enrollment: $(if($isEnrolled){$enrolled.ProviderID}else{'None'})" `
        -ForegroundColor $(if($isEnrolled){'Green'}else{'Yellow'})
    if ($isEnrolled) { $score++ }

    # Test 6: CDJ keys present
    $cdjPath = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\CDJ\AAD"
    $cdjPresent = Test-Path $cdjPath
    Write-Host "[$(if($cdjPresent){'PASS'}else{'WARN'})] CDJ registry keys present" `
        -ForegroundColor $(if($cdjPresent){'Green'}else{'Yellow'})
    if ($cdjPresent) { $score++ }

    # Test 7: Azure AD reachable
    try {
        $null = Invoke-WebRequest "https://login.microsoftonline.com" -UseBasicParsing -TimeoutSec 5
        $aadReachable = $true
    } catch { $aadReachable = $false }
    Write-Host "[$(if($aadReachable){'PASS'}else{'FAIL'})] Azure AD endpoints reachable" `
        -ForegroundColor $(if($aadReachable){'Green'}else{'Red'})
    if ($aadReachable) { $score++ }

    # Summary
    Write-Host "`nScore: $score / $maxScore" -ForegroundColor $(if($score -ge 6){'Green'}elseif($score -ge 4){'Yellow'}else{'Red'})
    Write-Host "==============================`n" -ForegroundColor Cyan
}

Invoke-HybridJoinValidation
```

```text title="Résultat attendu — poste en état nominal"
===== HYBRID JOIN VALIDATION : DESKTOP-PROD01 =====
[PASS] Domain joined: CONTOSO
[PASS] Azure AD Joined
[PASS] Primary Refresh Token valid
[PASS] MDM enrollment not blocked
[PASS] Active MDM enrollment: MS DM Server
[PASS] CDJ registry keys present
[PASS] Azure AD endpoints reachable

Score: 7 / 7
==============================
```

---

## Références croisées

- [GPO/MDM convergence — théorie et positionnement](../bible-gpo/25-mdm-convergence.md) : comprendre pourquoi GPO et MDM coexistent et les limites de chaque approche
- [Migration vers Intune](19-migration-intune.md) : après Hybrid Join, la migration complète vers Intune full-cloud
- [Windows Update for Business et co-management](10-windows-update.md) : workload WUfB dans le contexte co-management MECM/Intune
