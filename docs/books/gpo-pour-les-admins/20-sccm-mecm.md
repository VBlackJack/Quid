---
description: GPO et SCCM/MECM en coexistence — périmètres, conflits WSUS, Client Settings, Managed Installer, reporting
tags: [gpo-admins, sccm, mecm, wsus, co-management, managed-installer, configuration-manager]
---

# GPO et SCCM/MECM en coexistence

!!! abstract "Ce que vous allez apprendre"
    - Délimiter avec précision les **périmètres GPO et SCCM** pour éviter que les deux systèmes ne se marchent dessus, avec un tableau de chevauchements opérationnels
    - Détecter et résoudre les **conflits WSUS** entre la GPO `WUServer` et le Software Update Point SCCM — le piège le plus fréquent en environnement mixte
    - Comprendre les **SCCM Client Settings** qui écrivent dans le registre et identifier lesquels peuvent entrer en conflit avec une GPO active
    - Configurer **Managed Installer** via GPO + WDAC pour que les applications déployées par SCCM soient automatiquement autorisées sans liste blanche manuelle
    - Construire une **Configuration Baseline SCCM** qui vérifie la présence des valeurs de registre imposées par vos GPO de sécurité

!!! tip "Si vous ne retenez qu'une chose"
    Quand SCCM et une GPO définissent la même valeur de registre, **la GPO gagne toujours** — sauf sous `HKLM\SOFTWARE\Microsoft\CCM\` qui est la zone propriétaire du client SCCM. Un conflit silencieux sur `WUServer` peut laisser la moitié de votre parc sans patch pendant des semaines sans qu'aucune alerte ne remonte. Auditez cette clé en premier dès qu'un poste ne se met plus à jour.

---

## Contexte de production

Dans la majorité des organisations on-prem ou hybrides, SCCM/MECM et les GPO AD coexistent depuis des années dans un équilibre souvent fragile. Les deux systèmes ont des périmètres complémentaires mais des zones de recouvrement réelles — déploiement logiciel, mises à jour Windows, conformité.

Les incidents les plus courants ne viennent pas d'une mauvaise configuration d'un seul système, mais d'une méconnaissance des interactions entre les deux. Ce chapitre traite ces interactions comme des sujets opérationnels concrets, pas comme des cas théoriques.

---

## Périmètres respectifs et zones de chevauchement

### :material-map-check: Ce que chaque système est censé gérer

SCCM et GPO ont des philosophies différentes. GPO agit sur la configuration du système d'exploitation via la base de registre et les templates ADMX. SCCM agit sur le cycle de vie des machines : déploiement, inventaire, mises à jour, conformité applicative.

**GPO gère :**

- La configuration OS : restrictions, sécurité, pare-feu, audit, Defender
- Les paramètres utilisateur : profil itinérant, redirection de dossiers, logon scripts
- Les certificats machine et les politiques IPsec/réseau
- AppLocker, WDAC, politiques de contrôle d'application

**SCCM gère :**

- Le déploiement d'applications (MSI, MSIX, scripts, packages complexes)
- Les mises à jour Windows via le Software Update Point (SUP), surcouche WSUS
- L'inventaire matériel et logiciel (WMI, fichiers, registre)
- L'OS Deployment (OSD) via task sequences
- La conformité via Configuration Baselines (CI/CB)
- Le client health et les remontées télémétrie

### :material-table: Tableau des chevauchements opérationnels

| Fonction | Via GPO | Via SCCM | Recommandation |
|---|---|---|---|
| Déploiement MSI | GPO Software Installation (`\Software Settings`) | Application / Package SCCM | **SCCM préféré** — lifecycle managé, reporting natif |
| Mises à jour Windows | WUfB (`WUServer`, `DeferQualityUpdates`) | Software Update Point (SUP) | **Un seul système** — voir section suivante |
| Scripts de démarrage | GPO Logon/Startup scripts | SCCM Programs / Scripts | Cohabitation possible si périmètres distincts |
| Conformité registre | GPO (écriture directe) | Configuration Baseline (lecture/rapport) | GPO pour forcer, CB pour auditer |
| Inventaire logiciel | Add/Remove Programs via WMI (lecture seule) | SCCM Hardware/Software Inventory | **SCCM** pour l'inventaire — plus complet |
| Certificats machine | GPO Certificate AutoEnrollment | SCCM Certificate Profiles | GPO si PKI AD CS, SCCM si PKI externe |
| Pare-feu Windows | GPO Windows Firewall ADMX | SCCM Endpoint Protection Policies | **GPO préféré** — appliqué sans dépendance réseau |
| Contrôle d'application | AppLocker / WDAC via GPO | Managed Installer (SCCM taggé) | Combinaison optimale — voir section Managed Installer |

!!! warning "À surveiller"
    La colonne "Recommandation" reflète les meilleures pratiques opérationnelles, pas des règles absolues. Dans certains environnements, SCCM Endpoint Protection est déjà en place pour le pare-feu, et le basculer vers GPO introduit plus de risques que de bénéfices. L'essentiel est que **les deux ne configurent pas la même valeur en même temps**.

!!! quote "En résumé"
    - GPO = configuration OS statique, sécurité, restrictions. SCCM = cycle de vie applicatif, mises à jour, inventaire.
    - Les chevauchements réels sont : déploiement MSI, Windows Update, conformité. Choisissez un maître par fonction.
    - Ne laissez jamais GPO et SCCM définir la même valeur de registre sans avoir documenté laquelle prend la priorité.

---

## Conflits WSUS : la source principale d'incidents

### :material-alert-circle: Anatomie du conflit

Le conflit le plus fréquent — et le plus silencieux — entre GPO et SCCM concerne le canal de mise à jour Windows.

SCCM utilise un **Software Update Point (SUP)**, qui est un rôle WSUS exposé par le serveur de site. Quand un client SCCM reçoit sa politique de mise à jour, le serveur de site lui communique l'URL du SUP via la politique CCM — sans passer par GPO.

Le problème survient quand une GPO définit simultanément `WUServer` sous :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate
Valeur : WUServer  (REG_SZ)
```

Si `WUServer` est définie par GPO avec une URL différente de l'URL du SUP SCCM, le client Windows Update pointe vers le mauvais WSUS. Les mises à jour SCCM n'arrivent plus. L'agent CCM continue de reporter "compliant" dans la console SCCM — parce que la politique CCM est bien reçue — mais les mises à jour ne s'installent pas.

!!! danger "Production"
    Ce conflit est **invisible dans la console SCCM**. Le client reste en état "Last Update Success" jusqu'à expiration du cache. Le seul indicateur fiable est l'âge de la dernière mise à jour installée sur le poste, visible dans `WindowsUpdate.log` ou via le rapport SCCM "Software Updates - Computers Not Receiving Updates".

### :material-powershell: Script de détection des conflits WSUS

Ce script détecte les postes où `WUServer` GPO est défini différemment de l'URL attendue du SUP.

```powershell title="Detect-WSUSConflict.ps1"
# Detect WSUS GPO conflict on local machine or remote endpoints
# Returns: GPO WUServer value, CCM SUP URL, conflict status
param(
    [string[]]$ComputerName = @($env:COMPUTERNAME),
    [string]$ExpectedSUPUrl = "http://sccm-sup.contoso.local:8530"  # Adapt to your environment
)

function Get-WSUSConflictStatus {
    param([string]$Computer)

    $result = Invoke-Command -ComputerName $Computer -ScriptBlock {
        param($ExpectedSUP)

        # GPO-defined WUServer
        $wuPath   = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"
        $gpoWU    = (Get-ItemProperty -Path $wuPath -ErrorAction SilentlyContinue).WUServer
        $useWU    = (Get-ItemProperty -Path $wuPath -ErrorAction SilentlyContinue).UseWUServer
        $gpoWUAU  = (Get-ItemProperty -Path "$wuPath\AU" -ErrorAction SilentlyContinue).UseWUServer

        # CCM-defined SUP (from WMI)
        $ccmSUP = $null
        try {
            $ccmSUP = (Get-WmiObject -Namespace "root\ccm\SoftwareUpdates\WUAHandler" `
                -Class CCM_UpdateSource -ErrorAction SilentlyContinue |
                Where-Object { $_.ContentLocation -like "http*" } |
                Select-Object -First 1).ContentLocation
        } catch { }

        # Conflict detection
        $hasGpoWU   = -not [string]::IsNullOrEmpty($gpoWU)
        $conflict   = $hasGpoWU -and ($gpoWU -ne $ExpectedSUP)

        [PSCustomObject]@{
            Computer        = $env:COMPUTERNAME
            GPO_WUServer    = if ($gpoWU) { $gpoWU } else { "(not set)" }
            GPO_UseWUServer = $useWU
            CCM_SUP         = if ($ccmSUP) { $ccmSUP } else { "(WMI unavailable)" }
            Conflict        = $conflict
            Resolution      = if ($conflict) { "Remove WUServer GPO or align to SUP URL" }
                              elseif ($hasGpoWU) { "OK - GPO matches expected SUP" }
                              else { "OK - No GPO WUServer override" }
        }
    } -ArgumentList $ExpectedSUP -ErrorAction SilentlyContinue

    return $result
}

foreach ($computer in $ComputerName) {
    Get-WSUSConflictStatus -Computer $computer
}
```

```powershell title="Résultat attendu"
Computer        : WS-001
GPO_WUServer    : http://old-wsus.contoso.local:8530
GPO_UseWUServer : 1
CCM_SUP         : http://sccm-sup.contoso.local:8530
Conflict        : True
Resolution      : Remove WUServer GPO or align to SUP URL

Computer        : WS-002
GPO_WUServer    : (not set)
GPO_UseWUServer :
CCM_SUP         : http://sccm-sup.contoso.local:8530
Conflict        : False
Resolution      : OK - No GPO WUServer override
```

### :material-wrench: Résolution du conflit

Il existe deux approches selon votre contexte :

**Option A — Supprimer la GPO WUServer (recommandé si SCCM SUP est maître)**

Utilisez une GPO Preference pour supprimer explicitement la valeur :

```
Configuration ordinateur > Préférences > Paramètres Windows > Registre
Action : Delete
Chemin : HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate
Nom de valeur : WUServer
```

Répétez pour `WUStatusServer` et `UseWUServer = 0` dans la clé `AU`.

**Option B — Aligner la GPO sur l'URL du SUP (si GPO est la source de vérité)**

Mettez à jour la valeur GPO `WUServer` avec l'URL exacte du SUP SCCM. Les deux systèmes pointent alors vers la même infrastructure, mais la GPO garde la main — ce qui peut créer un nouveau problème si le SUP change de serveur.

!!! warning "À surveiller"
    L'Option B crée une dépendance : chaque migration de serveur SUP nécessite une mise à jour de GPO **avant** la migration, sinon les clients perdent le WSUS. L'Option A est plus robuste en environnement SCCM managé.

### :material-table: Règles de priorité WSUS/WUfB

| Situation | Comportement Windows Update |
|---|---|
| `WUServer` GPO défini + `UseWUServer = 1` | WSUS GPO prioritaire — WUfB ignoré |
| `WUServer` GPO non défini, SCCM SUP actif | SCCM SUP (via CCM policy) utilisé |
| `WUServer` GPO défini + SCCM SUP actif | Conflit — GPO WSUS gagne, CCM reporte incorrectement |
| `WUServer` GPO vide (`""`) + `UseWUServer = 1` | Windows Update bloqué — aucun serveur valide |
| Ni GPO ni SCCM | Windows Update direct (Microsoft Update) |

!!! quote "En résumé"
    - `WUServer` GPO défini = WSUS GPO prioritaire, toujours. SCCM SUP perd silencieusement.
    - Pour un parc SCCM, supprimez explicitement `WUServer` en GPO ou ne la définissez pas.
    - La suppression doit être **active** (GPO Delete) — une GPO qui "ne configure pas" ne supprime pas une valeur écrite par une GPO précédente.
    - Utilisez `Detect-WSUSConflict.ps1` lors de chaque audit de conformité ou migration WSUS.

---

## SCCM Client Settings vs GPO : qui écrit quoi

### :material-cog: Les deux zones de registre du client SCCM

Le client SCCM (ccmexec.exe) écrit sa configuration dans deux zones distinctes :

```
HKLM\SOFTWARE\Microsoft\CCM\           ← zone propriétaire CCM (jamais toucher en GPO)
HKLM\SOFTWARE\Microsoft\SMS\           ← configuration héritée (lectures/compatibilité)
HKLM\SOFTWARE\Policies\Microsoft\      ← zone GPO — SCCM peut y lire, jamais y écrire
```

La zone `CCM\` est **propriétaire du client**. Écrire dans `HKLM\SOFTWARE\Microsoft\CCM\` via GPO Preference peut corrompre la configuration du client ou générer des comportements non documentés.

!!! danger "Production"
    Ne jamais définir de valeurs sous `HKLM\SOFTWARE\Microsoft\CCM\` via GPO Preferences. Cette zone est gérée exclusivement par le service ccmexec.exe et son système de politiques internes. Une valeur GPO qui entre en conflit avec une valeur CCM peut bloquer le client health check — le client se considère en mauvaise santé et se réinstalle en boucle.

### :material-table: 6 Client Settings courants et leurs interactions GPO

| Client Setting SCCM | Chemin registre CCM équivalent | GPO pouvant interférer | Risque de conflit |
|---|---|---|---|
| Hardware Inventory cycle | `HKLM\SOFTWARE\Microsoft\CCM\CcmEval\` | Aucune GPO standard | Faible |
| Software Update Scan cycle | `HKLM\SOFTWARE\Microsoft\CCM\Policy\...\UpdateScanPolicy` | GPO `WUServer` / `UseWUServer` | **Élevé** — voir section WSUS |
| Client Cache Size | `HKLM\SOFTWARE\Microsoft\CCM\CcmCache\Size` | Aucune GPO standard | Faible |
| Power Management | `HKLM\SOFTWARE\Microsoft\CCM\Policy\...\PowerPolicy` | GPO `Configuration ordinateur > Paramètres Windows > Sécurité > Options de sécurité > Mise en veille` | Moyen — la GPO prend le dessus |
| Delivery Optimization | `HKLM\SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization` | GPO `DeliveryOptimization` ADMX | **Élevé** — même zone registre |
| Remote Control | `HKLM\SOFTWARE\Microsoft\CCM\RemoteControl\` | GPO `Remote Desktop` / `Remote Assistance` | Moyen — fonctions distinctes mais ports partagés |

### :material-powershell: Lire la configuration CCM en live pour débogage

```powershell title="Get-CCMDebugConfig.ps1"
# Read live CCM client configuration for conflict debugging
# Run locally on the target machine or via Invoke-Command

param([string]$ComputerName = $env:COMPUTERNAME)

Invoke-Command -ComputerName $ComputerName -ScriptBlock {
    # CCM assigned site and management point
    $ccmBase = "HKLM:\SOFTWARE\Microsoft\CCM"
    $ccmSetup = "HKLM:\SOFTWARE\Microsoft\SMS\Client\Configuration\Client Properties"

    $assignedSite = (Get-ItemProperty -Path $ccmBase -EA SilentlyContinue).AssignedSiteCode
    $mp           = (Get-ItemProperty -Path $ccmBase -EA SilentlyContinue).CurrentManagementPoint
    $cacheSize    = (Get-ItemProperty -Path "$ccmBase\CcmCache" -EA SilentlyContinue).Size
    $doGroupId    = (Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization" `
                        -EA SilentlyContinue).DOGroupId

    # Software Update Point from CCM WMI
    $sup = $null
    try {
        $sup = (Get-WmiObject -Namespace "root\ccm" -Class SMS_Authority -EA SilentlyContinue |
                Select-Object -ExpandProperty CurrentManagementPoint)
    } catch { }

    # Check for GPO override on Delivery Optimization
    $doGPODownloadMode = (Get-ItemProperty -Path `
        "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization" `
        -EA SilentlyContinue).DODownloadMode

    [PSCustomObject]@{
        Computer            = $env:COMPUTERNAME
        AssignedSite        = $assignedSite
        ManagementPoint     = $mp
        CacheSize_MB        = $cacheSize
        DO_GroupID          = $doGroupId
        DO_GPO_DownloadMode = if ($null -ne $doGPODownloadMode) { $doGPODownloadMode } else { "(not set by GPO)" }
        CCM_SUP             = if ($sup) { $sup } else { "(WMI query failed)" }
    }
}
```

```powershell title="Résultat attendu"
Computer            : WS-001
AssignedSite        : P01
ManagementPoint     : sccm-mp.contoso.local
CacheSize_MB        : 5120
DO_GroupID          : {a1b2c3d4-0000-0000-0000-000000000001}
DO_GPO_DownloadMode : 2
CCM_SUP             : sccm-mp.contoso.local
```

!!! quote "En résumé"
    - La zone `HKLM\SOFTWARE\Microsoft\CCM\` est propriétaire du client — ne jamais y écrire via GPO.
    - Les conflits réels se produisent dans `HKLM\SOFTWARE\Policies\Microsoft\Windows\` (Delivery Optimization, WindowsUpdate).
    - Utilisez le script de débogage CCM pour identifier rapidement si un GPO override interfère avec un client setting SCCM.

---

## Managed Installer SCCM pour WDAC

### :material-shield-lock: Le problème sans Managed Installer

Windows Defender Application Control (WDAC) fonctionne en mode liste blanche. Sans Managed Installer configuré, chaque application déployée par SCCM doit être explicitement autorisée dans la politique WDAC — soit par hash de fichier, soit par règle de signature.

Un parc SCCM actif peut déployer des dizaines d'applications par mois. Maintenir les hash à jour manuellement est non viable en production. C'est exactement le problème que Managed Installer résout.

### :material-information: Principe de Managed Installer

Managed Installer est un mécanisme de confiance dynamique : tout exécutable installé par un processus **taggé comme Managed Installer** est automatiquement autorisé par WDAC, sans modification de la politique WDAC ni mise à jour de hash.

Le tag s'applique à `ccmexec.exe` et à tous les processus enfants qu'il lance. En pratique, toute application déployée par SCCM — MSI, script, MSIX — hérite de la confiance Managed Installer.

Le mécanisme repose sur deux composants :

1. **AppLocker Managed Installer rule** — identifie `ccmexec.exe` comme Managed Installer (règle AppLocker de type Script, pas enforcement)
2. **Politique WDAC** — activée avec l'option `Enabled:Managed Installer` qui consume les tags AppLocker

!!! warning "À surveiller"
    Managed Installer ne fonctionne que si **AppLocker est actif** sur le poste, même en mode "Audit only" ou avec une seule règle. Si AppLocker est complètement désactivé, le tagging des exécutables ne se produit pas et WDAC ne reconnaîtra pas les installations SCCM.

### :material-powershell: Configuration via GPO — AppLocker Managed Installer rule

Cette GPO définit `ccmexec.exe` comme Managed Installer via une règle AppLocker. Elle ne bloque rien — elle pose uniquement le tag.

```powershell title="Set-SCCMManagedInstallerRule.ps1"
# Configure SCCM ccmexec.exe as AppLocker Managed Installer via GPO registry
# This script writes the AppLocker Managed Installer rule to a GPO
# Requires: RSAT GroupPolicy module, GroupPolicy module

#Requires -Modules GroupPolicy

param(
    [Parameter(Mandatory)]
    [string]$GPOName,

    [string]$Domain = (Get-ADDomain).DNSRoot
)

# AppLocker Managed Installer rule XML for ccmexec.exe
# RuleCollection type="ManagedInstaller" is the key — not Exe/Script/etc.
$managedInstallerXml = @'
<AppLockerPolicy Version="1">
  <RuleCollection Type="ManagedInstaller" EnforcementMode="AuditOnly">
    <FilePublisherRule
        Id="6cc9b840-b4ab-4e6d-97f4-12a8a9b3c4e7"
        Name="SCCM CCMExec - Managed Installer"
        Description="Tag all executables installed by SCCM client as trusted"
        UserOrGroupSid="S-1-1-0"
        Action="Allow">
      <Conditions>
        <FilePublisherCondition
            PublisherName="O=MICROSOFT CORPORATION, L=REDMOND, S=WASHINGTON, C=US"
            ProductName="SYSTEM CENTER CONFIGURATION MANAGER"
            BinaryName="CCMEXEC.EXE">
          <BinaryVersionRange LowSection="*" HighSection="*" />
        </FilePublisherCondition>
      </Conditions>
    </FilePublisherRule>
  </RuleCollection>
</AppLockerPolicy>
'@

# Write rule to target GPO via Set-GPRegistryValue (AppLocker XML via registry)
$gpo = Get-GPO -Name $GPOName -Domain $Domain -ErrorAction Stop

# AppLocker stores its policy as binary XML in registry
$xmlBytes = [System.Text.Encoding]::Unicode.GetBytes($managedInstallerXml)

Set-GPRegistryValue -Guid $gpo.Id -Domain $Domain `
    -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows\SrpV2\ManagedInstaller" `
    -ValueName "Policy" `
    -Type Binary `
    -Value $xmlBytes

Write-Output "Managed Installer rule written to GPO: $GPOName"
Write-Output "Next step: Enable 'Enabled:Managed Installer' option in your WDAC policy XML."
```

### :material-file-document-check: Activer l'option dans la politique WDAC

Dans votre fichier de politique WDAC XML, ajoutez l'option `Enabled:Managed Installer` dans la section `<PolicyTypeOptions>` :

```xml title="WDAC-Policy-ManagedInstaller.xml (extrait)"
<Rules>
  <!-- Existing WDAC rules -->
  <Rule>
    <Option>Enabled:Managed Installer</Option>
  </Rule>
  <Rule>
    <Option>Enabled:Audit Mode</Option>   <!-- Remove when ready for enforcement -->
  </Rule>
</Rules>
```

Distribuez la politique WDAC via GPO :

```
Configuration ordinateur > Stratégies > Modèles d'administration
> Système > Device Guard
> Déployer Windows Defender Application Control
Valeur : chemin UNC vers le fichier .p7b compilé
```

### :material-magnify: Vérification — Event Log WDAC

```powershell title="Verify-WDACManagedInstaller.ps1"
# Verify WDAC is correctly tagging SCCM-deployed executables as Managed Installer
# Events to look for:
#   3076 (Audit block — would have been blocked without MI tag)
#   3077 (Enforce block — blocked by WDAC)
#   3089 (Allowed by Managed Installer tag)

param(
    [string]$ComputerName = $env:COMPUTERNAME,
    [int]$LastHours = 24
)

Invoke-Command -ComputerName $ComputerName -ScriptBlock {
    param($Hours)

    $since  = (Get-Date).AddHours(-$Hours)
    $logName = "Microsoft-Windows-CodeIntegrity/Operational"

    $events = Get-WinEvent -LogName $logName -ErrorAction SilentlyContinue |
        Where-Object { $_.TimeCreated -ge $since } |
        Where-Object { $_.Id -in @(3076, 3077, 3089) } |
        Select-Object TimeCreated, Id,
            @{N="EventType"; E={
                switch ($_.Id) {
                    3076 { "Audit Block (MI tag bypassed)" }
                    3077 { "ENFORCE Block (WDAC blocked)" }
                    3089 { "Allowed by Managed Installer" }
                }
            }},
            @{N="FilePath"; E={ $_.Properties[1].Value }},
            @{N="SignerInfo"; E={ $_.Properties[4].Value }}

    if (-not $events) {
        Write-Output "No WDAC events in the last $Hours hours. If SCCM deployed software, Managed Installer is working silently (no block events = OK)."
    } else {
        $events | Sort-Object TimeCreated | Format-Table -AutoSize
    }
} -ArgumentList $LastHours
```

```powershell title="Résultat attendu — configuration correcte"
No WDAC events in the last 24 hours.
If SCCM deployed software, Managed Installer is working silently (no block events = OK).
```

Si vous voyez des événements `3077` (Enforce Block) sur des exécutables déployés par SCCM, le tag Managed Installer ne s'applique pas — vérifiez qu'AppLocker est actif (même en Audit) et que la GPO Managed Installer est bien appliquée.

!!! quote "En résumé"
    - Managed Installer évite de maintenir des hash WDAC pour chaque application SCCM déployée.
    - Deux composants requis : règle AppLocker `ManagedInstaller` sur `ccmexec.exe` + option `Enabled:Managed Installer` dans la politique WDAC.
    - AppLocker doit être actif — même en mode Audit Only — pour que le tagging fonctionne.
    - L'absence d'événements `3077` après un déploiement SCCM confirme que Managed Installer fonctionne.

---

## Configuration Baseline SCCM pour le reporting de conformité GPO

### :material-clipboard-check: Pourquoi auditer les GPO depuis SCCM

Les GPO forcent des valeurs de registre. Mais une GPO peut être bloquée, désactivée, ou supplantée par une GPO locale. Un poste peut sembler conforme dans GPMC et avoir une valeur différente en réalité.

La Configuration Baseline (CB) SCCM permet de vérifier la présence effective des valeurs dans le registre — indépendamment du système de politique qui les a écrites. C'est un audit de résultat, pas d'intention.

### :material-powershell: Script de création d'une CB avec 5 vérifications critiques

Ce script crée une Configuration Baseline qui vérifie 5 valeurs de registre couramment définies par GPO de sécurité.

```powershell title="New-GPOComplianceBaseline.ps1"
# Create SCCM Configuration Baseline to verify GPO-set registry values
# Requires: ConfigurationManager PowerShell module (on SCCM Primary Site server)
# Adapt $SiteCode and $SiteServer to your environment

#Requires -Modules ConfigurationManager

param(
    [string]$SiteCode    = "P01",
    [string]$SiteServer  = "sccm-primary.contoso.local",
    [string]$BaselineName = "GPO Security Baseline Audit"
)

# Connect to SCCM site
$psDrive = Get-PSDrive -Name $SiteCode -ErrorAction SilentlyContinue
if (-not $psDrive) {
    $null = New-PSDrive -Name $SiteCode -PSProvider CMSite -Root $SiteServer
}
Set-Location "$SiteCode`:\"

# Configuration Items to create
# Each CI checks one registry value that a security GPO should have set
$ciDefinitions = @(
    @{
        Name        = "CI-GPO-LMCompatibilityLevel"
        Description = "Verify LAN Manager authentication level >= 3 (NTLMv2 only)"
        RegHive     = "LocalMachine"
        RegKey      = "SYSTEM\CurrentControlSet\Control\Lsa"
        RegValue    = "LmCompatibilityLevel"
        RegType     = "Integer"
        ExpectedVal = 3
        Operator    = "GreaterEquals"
    },
    @{
        Name        = "CI-GPO-SMBSigning-Client"
        Description = "Verify SMB client signing is required (GPO: Microsoft network client - Digitally sign communications always)"
        RegHive     = "LocalMachine"
        RegKey      = "SYSTEM\CurrentControlSet\Services\LanmanWorkstation\Parameters"
        RegValue    = "RequireSecuritySignature"
        RegType     = "Integer"
        ExpectedVal = 1
        Operator    = "Equals"
    },
    @{
        Name        = "CI-GPO-SMBSigning-Server"
        Description = "Verify SMB server signing is required"
        RegHive     = "LocalMachine"
        RegKey      = "SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters"
        RegValue    = "RequireSecuritySignature"
        RegType     = "Integer"
        ExpectedVal = 1
        Operator    = "Equals"
    },
    @{
        Name        = "CI-GPO-WUServer-NotSet"
        Description = "Verify WUServer is NOT overriding SCCM SUP (key should be absent or empty)"
        RegHive     = "LocalMachine"
        RegKey      = "SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"
        RegValue    = "WUServer"
        RegType     = "String"
        ExpectedVal = ""
        Operator    = "Equals"
    },
    @{
        Name        = "CI-GPO-CredentialGuard"
        Description = "Verify Device Guard / Credential Guard is enabled (GPO: Virtualization Based Security)"
        RegHive     = "LocalMachine"
        RegKey      = "SYSTEM\CurrentControlSet\Control\DeviceGuard"
        RegValue    = "EnableVirtualizationBasedSecurity"
        RegType     = "Integer"
        ExpectedVal = 1
        Operator    = "Equals"
    }
)

$createdCIs = @()

foreach ($ci in $ciDefinitions) {
    Write-Host "Creating CI: $($ci.Name)..." -ForegroundColor Cyan

    $newCI = New-CMConfigurationItem `
        -Name        $ci.Name `
        -Description $ci.Description `
        -CreationType WindowsOS

    # Add registry setting rule to CI
    $registrySetting = New-CMRegistryAuditSetting `
        -Hive        $ci.RegHive `
        -Key         $ci.RegKey `
        -ValueName   $ci.RegValue `
        -DataType    $ci.RegType

    $complianceRule = New-CMComplianceRuleValue `
        -InputObject $registrySetting `
        -ExpectedValue $ci.ExpectedVal `
        -RuleOperator  $ci.Operator `
        -NoncomplianceSeverity Warning

    Set-CMConfigurationItem -InputObject $newCI -AddRule $complianceRule | Out-Null

    $createdCIs += $newCI
    Write-Host "  Created: $($ci.Name)" -ForegroundColor Green
}

# Create baseline and add all CIs
Write-Host "`nCreating Configuration Baseline: $BaselineName..." -ForegroundColor Cyan

$baseline = New-CMBaseline `
    -Name        $BaselineName `
    -Description "Audits effective presence of GPO security settings in registry. Does not modify values."

foreach ($ci in $createdCIs) {
    Set-CMBaseline -InputObject $baseline -AddOSConfigurationItem $ci | Out-Null
}

Write-Host "Baseline created. Deploy to a device collection to start compliance reporting." -ForegroundColor Green
Write-Host "Path in console: Assets and Compliance > Compliance Settings > Configuration Baselines > $BaselineName"
```

### :material-table: Les 5 vérifications et leur origine GPO

| Configuration Item | Valeur de registre vérifiée | GPO d'origine | Non-conformité = |
|---|---|---|---|
| `CI-GPO-LMCompatibilityLevel` | `LmCompatibilityLevel >= 3` | Sécurité réseau : niveau d'authentification LAN Manager | Authentification NTLM faible possible |
| `CI-GPO-SMBSigning-Client` | `RequireSecuritySignature = 1` (client SMB) | Client réseau Microsoft : communications signées toujours | NTLM relay possible |
| `CI-GPO-SMBSigning-Server` | `RequireSecuritySignature = 1` (serveur SMB) | Serveur réseau Microsoft : communications signées toujours | NTLM relay possible |
| `CI-GPO-WUServer-NotSet` | `WUServer = ""` | Absence de GPO WSUS concurrente | Conflit WSUS SCCM (voir section dédiée) |
| `CI-GPO-CredentialGuard` | `EnableVirtualizationBasedSecurity = 1` | Device Guard : sécurité basée sur la virtualisation | Credential Guard inactif |

!!! tip "Adapter au contexte"
    La CI `CI-GPO-WUServer-NotSet` vérifie que `WUServer` est vide — ce qui est correct si SCCM est maître. Si votre architecture utilise GPO WSUS comme source de vérité, inversez : vérifiez que `WUServer` est défini avec l'URL attendue.

### :material-powershell: Requête de reporting PowerShell (sans console SCCM)

```powershell title="Get-BaselineComplianceReport.ps1"
# Pull GPO compliance baseline results from SCCM SQL reporting
# Requires: SQL access to SCCM database, read-only is sufficient

param(
    [string]$SccmSqlServer  = "sccm-sql.contoso.local",
    [string]$SccmDatabase   = "CM_P01",
    [string]$BaselineName   = "GPO Security Baseline Audit",
    [int]$TopNonCompliant   = 20
)

$query = @"
SELECT TOP $TopNonCompliant
    v.Name0                      AS ComputerName,
    v.Operating_System_Name_and0 AS OS,
    bc.DisplayName               AS Baseline,
    ci.DisplayName               AS ConfigItem,
    cr.ComplianceStateName       AS Status,
    cr.LastEvalTime              AS LastChecked
FROM v_R_System v
JOIN v_CICurrentComplianceStatus cr ON cr.ResourceID = v.ResourceID
JOIN v_ConfigurationItems ci        ON ci.CI_ID = cr.CI_ID
JOIN v_BaselineTargetedConfigs btc  ON btc.CI_ID = cr.CI_ID
JOIN v_ConfigurationItems bc        ON bc.CI_ID = btc.BaselineCI_ID
WHERE bc.DisplayName = '$BaselineName'
  AND cr.ComplianceStateName != 'Compliant'
ORDER BY cr.LastEvalTime DESC
"@

$connection = New-Object System.Data.SqlClient.SqlConnection
$connection.ConnectionString = "Server=$SccmSqlServer;Database=$SccmDatabase;Integrated Security=True;"
$connection.Open()

$command = $connection.CreateCommand()
$command.CommandText = $query

$adapter = New-Object System.Data.SqlClient.SqlDataAdapter $command
$dataset = New-Object System.Data.DataSet
$adapter.Fill($dataset) | Out-Null
$connection.Close()

$results = $dataset.Tables[0]
if ($results.Rows.Count -eq 0) {
    Write-Output "All machines compliant on baseline: $BaselineName"
} else {
    Write-Output "Non-compliant machines (top $TopNonCompliant):"
    $results | Format-Table -AutoSize
}
```

!!! quote "En résumé"
    - Une Configuration Baseline SCCM audite le résultat réel dans le registre — indépendamment de la GPO qui l'a écrit.
    - Les 5 CI couvrent les vecteurs d'attaque les plus courants (NTLM, SMB signing, Credential Guard, WSUS conflict).
    - Le reporting SQL permet d'intégrer les résultats dans un dashboard opérationnel sans passer par la console SCCM.
    - Une CI non-conforme sur `WUServer` est un signal d'alarme immédiat pour le conflit WSUS.

---

## Le piège du déploiement du client SCCM via GPO Software Installation

### :material-alert-decagram: Le scénario "ça marche, mais..."

Il est tentant de déployer le client SCCM (ccmsetup.msi) via GPO Software Installation. C'est fonctionnel — le MSI s'installe, le client s'enrôle, tout semble opérationnel.

Le problème apparaît six mois plus tard, lors de la mise à jour du client SCCM.

### :material-timeline: Pourquoi c'est problématique

Quand Microsoft publie une nouvelle version du client SCCM, le serveur de site peut mettre à jour automatiquement les clients via le mécanisme **Client Push** ou **Automatic Client Upgrade**. Ce mécanisme fonctionne parfaitement — mais si le client a été initialement déployé par GPO Software Installation, la GPO conserve une règle "ce package doit être installé".

Après la mise à jour du client, le MSI d'origine dans le partage GPO est une version ancienne. Windows Installer détecte que la version actuelle (mise à jour par SCCM) diffère de celle attendue par la GPO, et selon la configuration, peut :

- Tenter de réinstaller la version ancienne (downgrade)
- Générer des erreurs MSI 1603 répétées dans le journal Application
- Bloquer les mises à jour logicielles SCCM suivantes

!!! danger "Production"
    Le symptôme classique : après un Automatic Client Upgrade SCCM, une partie du parc génère des événements MSI 1603 ou 1618 à chaque logon. L'investigation pointe vers le partage GPO Software Installation qui tente de réinstaller l'ancienne version du client. La résolution nécessite de supprimer la règle GPO ET de forcer un gpupdate — pendant les heures ouvrées, sur des milliers de postes.

### :material-check-circle: L'approche correcte : bootstrapping dédié

SCCM fournit ses propres mécanismes de déploiement initial du client. Utilisez l'un d'eux selon votre contexte :

| Méthode | Quand l'utiliser | Prérequis |
|---|---|---|
| **Client Push Installation** | Déploiement initial sur machines AD existantes | Compte admin local ou domaine, ports 135/445 ouverts |
| **Software Update Point** | Clients avec WSUS déjà actif | SUP configuré, KB SCCM Client publiée |
| **Logon Script (CCMSetup.exe)** | Environnements sans push réseau | Partage DFS avec `ccmsetup.exe`, script GPO léger |
| **Task Sequence OSD** | Nouvelle installation OS | SCCM OSD configuré, PXE ou media |
| **Intune Co-Management Bootstrap** | Migration vers co-management | Hybrid Azure AD Join, Intune licence |

La méthode **Logon Script** est la plus proche de l'approche GPO, mais avec une différence critique : le script appelle `ccmsetup.exe` et vérifie d'abord si le client est déjà installé. Il ne crée pas de règle MSI persistante.

```powershell title="Bootstrap-SCCMClient.ps1 — Logon Script GPO"
# Bootstrap SCCM client via GPO logon script
# Safe: checks for existing installation before running, no persistent MSI rule
# Deploy via: GPO > Computer Configuration > Windows Settings > Scripts > Startup

$ccmSetupPath = "\\sccm-primary.contoso.local\SMS_P01\Client\ccmsetup.exe"
$ccmInstalled = Test-Path "C:\Windows\CCM\CcmExec.exe"

if ($ccmInstalled) {
    # Client already present — nothing to do
    # The SCCM Automatic Client Upgrade handles version updates
    exit 0
}

if (-not (Test-Path $ccmSetupPath)) {
    Write-EventLog -LogName Application -Source "SCCM-Bootstrap" `
        -EventId 9001 -EntryType Error `
        -Message "SCCM client setup not found at: $ccmSetupPath"
    exit 1
}

# Install client — SMSSITECODE and MP parameters for your site
Start-Process -FilePath $ccmSetupPath `
    -ArgumentList "/mp:sccm-mp.contoso.local SMSSITECODE=P01 SMSSLP=sccm-primary.contoso.local" `
    -Wait

Write-EventLog -LogName Application -Source "SCCM-Bootstrap" `
    -EventId 9002 -EntryType Information `
    -Message "SCCM client bootstrap initiated on $env:COMPUTERNAME"
```

!!! tip "Enregistrer la source d'événement"
    Le script écrit dans le journal Application avec la source "SCCM-Bootstrap". Créez cette source une fois avec `New-EventLog -LogName Application -Source "SCCM-Bootstrap"` sur les machines cibles (via GPO Preferences Registry ou une autre GPO Startup Script dédiée).

### :material-magnify: Vérification post-déploiement client

```powershell title="Verify-SCCMClientHealth.ps1"
# Verify SCCM client installation and health on target machines
param(
    [string[]]$ComputerName = @($env:COMPUTERNAME)
)

foreach ($computer in $ComputerName) {
    $result = Invoke-Command -ComputerName $computer -ScriptBlock {
        $ccmExe   = Test-Path "C:\Windows\CCM\CcmExec.exe"
        $service  = Get-Service -Name CcmExec -ErrorAction SilentlyContinue
        $version  = if ($ccmExe) {
            (Get-Item "C:\Windows\CCM\CcmExec.exe").VersionInfo.FileVersion
        } else { "Not installed" }

        # Check for MSI error events (1603/1618) in last 24h
        $msiErrors = Get-WinEvent -LogName Application -MaxEvents 200 `
                        -ErrorAction SilentlyContinue |
                     Where-Object { $_.Id -in @(1603, 1618) -and
                                    $_.Message -like "*ccmsetup*" } |
                     Measure-Object | Select-Object -ExpandProperty Count

        [PSCustomObject]@{
            Computer         = $env:COMPUTERNAME
            CCMExePresent    = $ccmExe
            ServiceStatus    = if ($service) { $service.Status } else { "Not found" }
            ClientVersion    = $version
            MSIErrors_24h    = $msiErrors
            GPO_MSI_Conflict = ($msiErrors -gt 0)
        }
    } -ErrorAction SilentlyContinue

    $result
}
```

```powershell title="Résultat attendu"
Computer         : WS-001
CCMExePresent    : True
ServiceStatus    : Running
ClientVersion    : 5.00.9096.1000
MSIErrors_24h    : 0
GPO_MSI_Conflict : False
```

Un `MSIErrors_24h > 0` avec `GPO_MSI_Conflict : True` indique le problème de GPO Software Installation décrit plus haut. La résolution passe par la suppression du package dans la GPO et un `gpupdate /force`.

!!! quote "En résumé"
    - Déployer le client SCCM via GPO Software Installation fonctionne initialement — mais le lifecycle est non gérable.
    - Après chaque Automatic Client Upgrade SCCM, la GPO tente de réinstaller l'ancienne version (MSI 1603).
    - L'approche correcte est un script de bootstrap qui vérifie l'existence du client avant d'agir — pas de règle MSI persistante.
    - Le script `Verify-SCCMClientHealth.ps1` détecte immédiatement les conflits MSI post-upgrade.

---

## Vérification globale post-déploiement

### :material-check-all: Script de synthèse opérationnelle

Ce script consolide les vérifications de tous les points couverts dans ce chapitre en un seul rapport.

```powershell title="Invoke-GPOSCCMHealthCheck.ps1"
# Consolidated GPO/SCCM coexistence health check
# Covers: WSUS conflict, CCM client health, Managed Installer, MSI errors
param(
    [string[]]$ComputerName,
    [string]$ExpectedSUPUrl   = "http://sccm-sup.contoso.local:8530",
    [string]$ExpectedSiteCode = "P01"
)

# Default to all domain computers if not specified
if (-not $ComputerName) {
    $ComputerName = (Get-ADComputer -Filter { OperatingSystem -like "*Windows 10*" -or
                                              OperatingSystem -like "*Windows 11*" } |
                     Select-Object -ExpandProperty Name)
}

$report = foreach ($computer in $ComputerName) {
    Invoke-Command -ComputerName $computer -ScriptBlock {
        param($ExpSUP, $ExpSite)

        # 1. WSUS conflict check
        $wuPath  = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"
        $gpoWU   = (Get-ItemProperty $wuPath -EA SilentlyContinue).WUServer
        $useWU   = (Get-ItemProperty $wuPath -EA SilentlyContinue).UseWUServer
        $wsusConflict = (-not [string]::IsNullOrEmpty($gpoWU)) -and ($gpoWU -ne $ExpSUP)

        # 2. SCCM client health
        $ccmSvc    = Get-Service CcmExec -EA SilentlyContinue
        $ccmVer    = if (Test-Path "C:\Windows\CCM\CcmExec.exe") {
                         (Get-Item "C:\Windows\CCM\CcmExec.exe").VersionInfo.FileVersion
                     } else { $null }
        $assignedSite = (Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\CCM" -EA SilentlyContinue).AssignedSiteCode

        # 3. MSI errors (GPO Software Installation conflict indicator)
        $msiErrors = (Get-WinEvent -LogName Application -MaxEvents 300 -EA SilentlyContinue |
                      Where-Object { $_.Id -in @(1603, 1618) -and
                                     $_.Message -like "*ccm*" } |
                      Measure-Object).Count

        # 4. Managed Installer (AppLocker rule check)
        $alPolicy  = Get-AppLockerPolicy -Effective -EA SilentlyContinue
        $hasMI     = $false
        if ($alPolicy) {
            $hasMI = ($alPolicy.RuleCollections | Where-Object { $_.GetType().Name -like "*Managed*" }).Count -gt 0
        }

        [PSCustomObject]@{
            Computer          = $env:COMPUTERNAME
            WSUS_Conflict     = $wsusConflict
            GPO_WUServer      = if ($gpoWU) { $gpoWU } else { "(none)" }
            CCM_Service       = if ($ccmSvc) { $ccmSvc.Status } else { "Missing" }
            CCM_Version       = if ($ccmVer) { $ccmVer } else { "Not installed" }
            CCM_AssignedSite  = if ($assignedSite) { $assignedSite } else { "Unknown" }
            SiteCode_OK       = ($assignedSite -eq $ExpSite)
            MSI_Errors_24h    = $msiErrors
            ManagedInstaller  = $hasMI
        }
    } -ArgumentList $ExpectedSUPUrl, $ExpectedSiteCode -ErrorAction SilentlyContinue
}

# Summary
$conflicts   = ($report | Where-Object { $_.WSUS_Conflict }).Count
$msiIssues   = ($report | Where-Object { $_.MSI_Errors_24h -gt 0 }).Count
$missingCCM  = ($report | Where-Object { $_.CCM_Service -eq "Missing" }).Count
$noMI        = ($report | Where-Object { -not $_.ManagedInstaller }).Count

Write-Host "`n=== GPO/SCCM Coexistence Health Report ===" -ForegroundColor Cyan
Write-Host "Total machines checked : $($report.Count)"
Write-Host "WSUS conflicts         : $conflicts" -ForegroundColor $(if ($conflicts) { "Red" } else { "Green" })
Write-Host "MSI install errors     : $msiIssues" -ForegroundColor $(if ($msiIssues) { "Yellow" } else { "Green" })
Write-Host "Missing CCM client     : $missingCCM" -ForegroundColor $(if ($missingCCM) { "Red" } else { "Green" })
Write-Host "No Managed Installer   : $noMI" -ForegroundColor $(if ($noMI) { "Yellow" } else { "Green" })
Write-Host ""

$report | Sort-Object WSUS_Conflict, CCM_Service | Format-Table -AutoSize
```

---

## Références croisées

- [`../bible-gpo/17-deploiement-msi.md`](../bible-gpo/17-deploiement-msi.md) — Déploiement MSI via GPO Software Installation : périmètre exact et limitations du lifecycle
- [`../bible-gpo/15-applocker-wdac.md`](../bible-gpo/15-applocker-wdac.md) — AppLocker et WDAC via GPO : configuration complète de la politique, règles de type Managed Installer
- [`19-migration-intune.md`](19-migration-intune.md) — Migration de SCCM vers Intune : co-management, workloads, et impact sur les GPO existantes
