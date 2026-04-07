---
description: Sécurité endpoint via GPO — Defender, ASR, Credential Guard, Exploit Protection, Windows Hello for Business
tags: [gpo-admins, defender, asr, credential-guard, endpoint-security, whfb]
---
# Sécurité endpoint via GPO

!!! abstract "Ce que vous allez apprendre"
    - Déployer et vérifier la configuration de **Windows Defender** sur l'ensemble du parc via `WindowsDefender.admx`, avec les 8 paramètres critiques et leurs chemins de registre
    - Activer les **règles Attack Surface Reduction (ASR)** en mode audit d'abord, puis passer en bloc progressivement — les 16 règles, leurs GUIDs et les event IDs à surveiller
    - Activer **Credential Guard** via GPO avec les prérequis matériels (UEFI, TPM 2.0, Hyper-V) et valider l'activation sans ouvrir l'interface graphique
    - Déployer **Exploit Protection** (successeur d'EMET) avec une configuration XML par processus distribuée par GPO
    - Configurer **Windows Hello for Business** en mode clé ou certificat selon l'infrastructure PKI disponible

!!! tip "Si vous ne retenez qu'une chose"
    Activez toujours les règles ASR en mode **Audit (2)** sur 100 % du parc pendant au minimum deux semaines avant de passer en **Block (1)**. Un passage direct en bloc sans audit préalable génère des blocages applicatifs invisibles jusqu'au lendemain matin — et un afflux de tickets support que vous n'aviez pas planifié.

---

## Contexte de production

La sécurité endpoint pilotée par GPO reste le socle opérationnel de la majorité des organisations qui n'ont pas encore migré vers Intune ou Defender for Endpoint en mode full-cloud.

Même avec une stratégie hybride, les GPO restent pertinentes : elles s'appliquent dès l'authentification AD, sans dépendance à Internet, et couvrent les machines en zone isolée ou hors VPN. Maîtriser ce périmètre, c'est éviter que la sécurité ne dépende du bon vouloir des connexions réseau.


!!! quote "En résumé"
    - Maîtriser ce périmètre, c'est éviter que la sécurité ne dépende du bon vouloir des connexions réseau.
    - Le contexte de production fixe les contraintes réelles de réseau, de portée et d’exploitation qui gouvernent tout le chapitre.
    - Retenez les hypothèses opérationnelles avant de choisir un modèle de liaison ou de déploiement.
---

## :material-shield-check: Windows Defender via GPO

### Le modèle ADMX WindowsDefender

Windows Defender est piloté par le modèle **`WindowsDefender.admx`**, présent nativement dans `C:\Windows\PolicyDefinitions` sur tout poste Windows 10/11 et Server 2016+.

Si vous utilisez un Central Store (`SYSVOL\Policies\PolicyDefinitions`), copiez-y `WindowsDefender.admx` et son fichier de langue `WindowsDefender.adml` depuis un poste à jour. Le modèle inclut plus de 150 paramètres — les sections critiques sont listées ci-dessous.

### Chemin de registre de base

Tous les paramètres Defender déployés par GPO écrivent sous :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows Defender\
```

Les sous-clés principales sont `Real-Time Protection`, `Spynet`, `MpEngine`, `Exclusions`, et `Scan`.

!!! warning "À surveiller — La clé `DisableAntiSpyware`"
    La valeur `HKLM\SOFTWARE\Policies\Microsoft\Windows Defender\DisableAntiSpyware` à `1` désactive **entièrement** Defender. Elle était utilisée pour co-existence avec des AV tiers. Si vous déployez un AV tiers qui gère lui-même cette clé, ne la définissez pas en GPO — vous risquez un conflit qui laisse la machine sans protection pendant les redémarrages.

### :material-table: Les 8 paramètres critiques

| Paramètre GPMC | Chemin GPMC complet | Valeur registre | Recommandation |
|---|---|---|---|
| Turn off real-time protection | `Config. ordi. > Modèles d'admin. > Composants Windows > Antivirus Microsoft Defender > Protection en temps réel` | `DisableRealtimeMonitoring` = 0 | **Désactivé** (protection active) |
| Turn on behavior monitoring | Même nœud | `DisableBehaviorMonitoring` = 0 | **Désactivé** (monitoring actif) |
| Configure cloud-delivered protection level | `… > MAPS` | `MAPSReporting` = 2 (Advanced) | **Activé — Advanced** |
| Join Microsoft MAPS | `… > MAPS` | `SpynetReporting` = 2 | **Activé** |
| Configure extended cloud check | `… > MpEngine` | `MpCloudBlockLevel` = 2 | **2 (High)** |
| Scan removable drives during full scan | `… > Analyse` | `DisableRemovableDriveScanning` = 0 | **Désactivé** (scan actif) |
| Configure monitoring for incoming/outgoing file activity | `… > Protection en temps réel` | `RealtimeScanDirection` = 0 | **0 (Both directions)** |
| Turn on process scanning whenever real-time protection is enabled | `… > Protection en temps réel` | `DisableScanOnRealtimeEnable` = 0 | **Désactivé** (scan actif) |

### :material-alert: Les exclusions : piège classique

Les exclusions Defender se configurent dans `… > Exclusions`. Chaque type a sa propre sous-clé registre :

- `Paths` → exclusions par chemin de dossier ou fichier
- `Extensions` → exclusions par extension (`.log`, `.tmp`)
- `Processes` → exclusions par nom de processus

!!! danger "Production — Exclusions trop larges"
    Exclure `C:\Windows\Temp` ou `*.exe` entièrement revient à désactiver Defender sur une zone critique. Les exclusions doivent être aussi précises que possible : chemin complet avec nom de fichier, ou processus métier identifié par chemin absolu (`C:\App\Metier\service.exe`), jamais par nom seul.

    Une exclusion trop large est souvent héritée d'un ticket "l'AV bloque notre appli" résolu en production dans l'urgence — et jamais révisée.

### :material-powershell: Vérification post-déploiement

```powershell title="Verify-DefenderGPO.ps1"
# Verify Defender GPO settings are applied on remote endpoints
$computers = Get-ADComputer -Filter { OperatingSystem -like "*Windows*" } |
    Select-Object -ExpandProperty Name

foreach ($computer in $computers) {
    $result = Invoke-Command -ComputerName $computer -ScriptBlock {
        $defenderPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows Defender"
        $rtPath       = "$defenderPath\Real-Time Protection"

        [PSCustomObject]@{
            Computer              = $env:COMPUTERNAME
            DisableAntiSpyware    = (Get-ItemProperty $defenderPath -EA SilentlyContinue).DisableAntiSpyware
            DisableRealtimeMonitor = (Get-ItemProperty $rtPath -EA SilentlyContinue).DisableRealtimeMonitoring
            DisableBehaviorMonitor = (Get-ItemProperty $rtPath -EA SilentlyContinue).DisableBehaviorMonitoring
            MAPSReporting         = (Get-ItemProperty "$defenderPath\Spynet" -EA SilentlyContinue).MAPSReporting
            DefenderStatus        = (Get-MpComputerStatus).AntivirusEnabled
        }
    } -ErrorAction SilentlyContinue

    $result
}
```

```title="Résultat attendu"
Computer               : WKS-001
DisableAntiSpyware     :         # doit être vide ou 0
DisableRealtimeMonitor : 0
DisableBehaviorMonitor : 0
MAPSReporting          : 2
DefenderStatus         : True
```

!!! quote "En résumé"
    - Chemin registre de base : `HKLM\SOFTWARE\Policies\Microsoft\Windows Defender\`
    - Modèle ADMX : `WindowsDefender.admx` — à synchroniser dans le Central Store
    - Vérifiez `DisableAntiSpyware` : toute valeur `1` en GPO désactive entièrement la protection
    - Exclusions : chemin absolu complet uniquement, jamais par extension générique
    - Validez avec `Get-MpComputerStatus` sur un échantillon après application de la GPO

---

## :material-wall: Attack Surface Reduction (ASR) via GPO

### Principe et modes de fonctionnement

Les règles ASR sont des contrôles de sécurité ciblés intégrés à Windows Defender Exploit Guard. Chaque règle bloque un vecteur d'attaque précis — macro Office, script PowerShell obfusqué, injection de processus — indépendamment de la détection de signature.

Chaque règle est identifiée par un GUID et accepte quatre modes :

| Valeur | Mode | Comportement |
|--------|------|-------------|
| `0` | Off | Règle désactivée |
| `1` | Block | Bloque l'action et génère l'événement 1121 |
| `2` | Audit | Journalise sans bloquer, génère l'événement 1122 |
| `6` | Warn | Avertit l'utilisateur, peut être contourné |

### :material-table: Les 16 règles ASR avec leurs GUIDs

| GUID | Nom de la règle | Vecteur ciblé |
|------|----------------|---------------|
| `BE9BA2D9-53EA-4CDC-84E5-9B1EEEE46550` | Block executable content from email client and webmail | Pièces jointes email |
| `D4F940AB-401B-4EFC-AADC-AD5F3C50688A` | Block all Office apps from creating child processes | Macro Office → shell |
| `3B576869-A4EC-4529-8536-B80A7769E899` | Block Office apps from creating executable content | Macro → .exe |
| `75668C1F-73B5-4CF0-BB93-3ECF5CB7CC84` | Block Office apps from injecting code into other processes | Process injection |
| `D3E037E1-3EB8-44C8-A917-57927947596D` | Block JavaScript or VBScript from launching downloaded executable content | JS/VBS → exe |
| `5BEB7EFE-FD9A-4556-801D-275E5FFC04CC` | Block execution of potentially obfuscated scripts | Scripts obfusqués |
| `92E97FA1-2EDF-4476-BDD6-9DD0B4DDDC7B` | Block Win32 API calls from Office macros | Macro → Win32 API |
| `01443614-CD74-433A-B99E-2ECDC07BFC25` | Block executable files from running unless they meet a prevalence, age, or trusted list criterion | Binaires inconnus |
| `C1DB55AB-C21A-4637-BB3F-A12568109D35` | Use advanced protection against ransomware | Ransomware |
| `9E6C4E1F-7D60-472F-BA1A-A39EF669E4B0` | Block credential stealing from the Windows local security authority subsystem (lsass.exe) | LSASS dump |
| `D1E49AAC-8F56-4280-B9BA-993A6D77406C` | Block process creations originating from PSExec and WMI commands | Lateral movement |
| `B2B3F03D-6A65-4F7B-A9C7-1C7EF74A9BA4` | Block untrusted and unsigned processes that run from USB | Clés USB |
| `26190899-1602-49E8-8B27-EB1D0A1CE869` | Block Office communication applications from creating child processes | Outlook → shell |
| `7674BA52-37EB-4A4F-A9A1-F0F9A1619A2C` | Block Adobe Reader from creating child processes | PDF → shell |
| `E6DB77E5-3DF2-4CF1-B95A-636979351E5B` | Block persistence through WMI event subscription | WMI persistence |
| `56A863A9-875E-4185-98A7-B882C64B5CE5` | Block abuse of exploited vulnerable signed drivers | Driver vulnérable |

### :material-rocket-launch: Déploiement progressif : la méthode sûre

Le déploiement ASR se fait en deux phases distinctes. Ne sautez pas la phase d'audit — c'est elle qui révèle les faux positifs avant qu'ils n'impactent la production.

**Phase 1 — Audit sur 100 % du parc (minimum 2 semaines)**

Créez une GPO `ASR-Audit-All` et liez-la au domaine ou à l'OU racine des postes.

```powershell title="Deploy-ASR-AuditMode.ps1"
# Deploy all 16 ASR rules in Audit mode (2) via GPO registry settings
$gpoName = "ASR-Audit-All"
$domain  = (Get-ADDomain).DNSRoot

# Create GPO if not exists
if (-not (Get-GPO -Name $gpoName -ErrorAction SilentlyContinue)) {
    New-GPO -Name $gpoName -Domain $domain -Comment "ASR rules — Audit phase"
}

# Registry path for ASR rules
$regPath = "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender\Windows Defender Exploit Guard\ASR\Rules"

# All 16 ASR rule GUIDs
$asrRules = @(
    "BE9BA2D9-53EA-4CDC-84E5-9B1EEEE46550",
    "D4F940AB-401B-4EFC-AADC-AD5F3C50688A",
    "3B576869-A4EC-4529-8536-B80A7769E899",
    "75668C1F-73B5-4CF0-BB93-3ECF5CB7CC84",
    "D3E037E1-3EB8-44C8-A917-57927947596D",
    "5BEB7EFE-FD9A-4556-801D-275E5FFC04CC",
    "92E97FA1-2EDF-4476-BDD6-9DD0B4DDDC7B",
    "01443614-CD74-433A-B99E-2ECDC07BFC25",
    "C1DB55AB-C21A-4637-BB3F-A12568109D35",
    "9E6C4E1F-7D60-472F-BA1A-A39EF669E4B0",
    "D1E49AAC-8F56-4280-B9BA-993A6D77406C",
    "B2B3F03D-6A65-4F7B-A9C7-1C7EF74A9BA4",
    "26190899-1602-49E8-8B27-EB1D0A1CE869",
    "7674BA52-37EB-4A4F-A9A1-F0F9A1619A2C",
    "E6DB77E5-3DF2-4CF1-B95A-636979351E5B",
    "56A863A9-875E-4185-98A7-B882C64B5CE5"
)

# Enable ASR feature via GPO
Set-GPRegistryValue -Name $gpoName -Key $regPath -ValueName "." `
    -Type String -Value "" | Out-Null

# Set parent key to enable ASR
Set-GPRegistryValue -Name $gpoName `
    -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender\Windows Defender Exploit Guard\ASR" `
    -ValueName "ExploitGuard_ASR_Rules" -Type DWord -Value 1

# Set all rules to Audit mode (2)
foreach ($guid in $asrRules) {
    Set-GPRegistryValue -Name $gpoName -Key $regPath `
        -ValueName $guid -Type String -Value "2"
    Write-Host "Set $guid -> Audit (2)"
}
```

**Phase 2 — Passage en Block pour les règles stables**

Après l'audit, analysez les événements 1122 dans l'Event Viewer. Passez en Block uniquement les règles qui n'ont produit aucun faux positif sur votre parc.

```powershell title="Switch-ASR-ToBlock.ps1"
# Switch specific stable ASR rules to Block mode (1)
# Only rules validated during audit phase — adjust the list per your audit findings
$gpoName = "ASR-Block-Stable"
$regPath = "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender\Windows Defender Exploit Guard\ASR\Rules"

# Rules confirmed stable in your environment — example subset
$stableRules = @(
    "9E6C4E1F-7D60-472F-BA1A-A39EF669E4B0",  # Block LSASS credential stealing
    "C1DB55AB-C21A-4637-BB3F-A12568109D35",  # Advanced ransomware protection
    "56A863A9-875E-4185-98A7-B882C64B5CE5"   # Block vulnerable signed drivers
)

foreach ($guid in $stableRules) {
    Set-GPRegistryValue -Name $gpoName -Key $regPath `
        -ValueName $guid -Type String -Value "1"
    Write-Host "Set $guid -> Block (1)"
}
```

### :material-magnify: Surveillance des événements ASR

Les événements ASR sont journalisés dans :

```
Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational
```

| Event ID | Mode | Signification |
|----------|------|--------------|
| `1121` | Block | Action bloquée par une règle ASR |
| `1122` | Audit | Action détectée mais non bloquée |
| `1129` | — | Règle ASR en mode Warn — utilisateur a ignoré |

```powershell title="Get-ASREvents.ps1"
# Collect ASR events from the last 7 days across remote endpoints
$computers = Get-ADComputer -Filter { OperatingSystem -like "*Windows*" } |
    Select-Object -ExpandProperty Name

$report = foreach ($computer in $computers) {
    Invoke-Command -ComputerName $computer -ErrorAction SilentlyContinue -ScriptBlock {
        Get-WinEvent -LogName "Microsoft-Windows-Windows Defender/Operational" `
            -FilterXPath "*[System[(EventID=1121 or EventID=1122) and
                TimeCreated[timediff(@SystemTime) <= 604800000]]]" `
            -ErrorAction SilentlyContinue |
        ForEach-Object {
            $xml = [xml]$_.ToXml()
            $data = $xml.Event.EventData.Data
            [PSCustomObject]@{
                Computer  = $env:COMPUTERNAME
                Time      = $_.TimeCreated
                EventID   = $_.Id
                RuleGUID  = ($data | Where-Object { $_.Name -eq "ID" }).'#text'
                File      = ($data | Where-Object { $_.Name -eq "Path" }).'#text'
                ProcessID = ($data | Where-Object { $_.Name -eq "ProcessID" }).'#text'
            }
        }
    }
}

$report | Sort-Object Time -Descending | Format-Table -AutoSize
```

!!! danger "Production — Passage en Block sans audit préalable"
    Si vous déployez les règles ASR directement en mode Block (1) sans phase d'audit, vous bloquez potentiellement des applications métier légitimes. Les règles les plus agressives (`01443614`, `D1E49AAC`) génèrent régulièrement des faux positifs sur les ERP, les outils de déploiement et les scripts RMM.

    **Procédure de récupération d'urgence** — passage en Audit à distance :

    ```powershell title="Emergency-ASR-Rollback.ps1"
    # Emergency: switch all ASR rules back to Audit mode on live machines
    # Run from a DC or jump server with admin rights on targets
    $affectedComputers = @("WKS-001", "WKS-002", "WKS-003")  # or Get-ADComputer query
    $regPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows Defender\Windows Defender Exploit Guard\ASR\Rules"

    Invoke-Command -ComputerName $affectedComputers -ScriptBlock {
        # Force audit mode locally while GPO is being corrected
        $rules = Get-Item $using:regPath -ErrorAction SilentlyContinue
        if ($rules) {
            $rules.GetValueNames() | ForEach-Object {
                Set-ItemProperty -Path $using:regPath -Name $_ -Value "2"
            }
        }
        # Force GPO refresh after updating the GPO on the DC
        & gpupdate /force /wait:0
        Write-Output "$env:COMPUTERNAME — ASR set to Audit, gpupdate launched"
    }
    ```

    Après exécution, corrigez la GPO source sur le DC (remplacez les valeurs `1` par `2`), puis laissez la propagation normale terminer la remédiation.

!!! quote "En résumé"
    - 16 règles ASR, chacune identifiée par un GUID, quatre modes : 0/1/2/6
    - Phase d'audit obligatoire (min. 2 semaines) avant tout passage en Block
    - Event 1121 = bloqué, Event 1122 = détecté sans blocage — à surveiller en phase d'audit
    - Déployez via `Set-GPRegistryValue` dans `HKLM\…\ASR\Rules`
    - En cas de blocages inattendus : rollback en Audit via Invoke-Command + gpupdate /force

---

## :material-lock-check: Credential Guard via GPO

### Prérequis matériels et logiciels

Credential Guard isole les secrets LSASS dans une enclave protégée par la virtualisation (VBS — Virtualization Based Security). Avant tout déploiement, chaque machine doit satisfaire ces prérequis :

| Prérequis | Détail |
|-----------|--------|
| UEFI Secure Boot | Activé dans le firmware, mode UEFI natif (pas Legacy/CSM) |
| TPM 2.0 | Puce TPM version 2.0 présente et activée |
| Hyper-V | Fonctionnalité Windows activée (même sur les postes de travail) |
| 64 bits | Windows 10/11 64 bits ou Windows Server 2016+ |
| Pas de credential providers tiers | Certains SSO tiers incompatibles avec VBS |

!!! warning "À surveiller — Machines virtuelles"
    Credential Guard fonctionne sur les VM Hyper-V (nested virtualization), mais nécessite une configuration explicite de l'hôte. Sur VMware ou d'autres hyperviseurs, la prise en charge VBS varie — vérifiez la compatibilité avant le déploiement en production.

### :material-cog: Activation via GPO

La GPO se configure dans :

```
Configuration ordinateur > Modèles d'administration > Système > Device Guard
```

Paramètre : **Turn On Virtualization Based Security**

Les sous-paramètres critiques à configurer dans cette même entrée :

- **Select Platform Security Level** → `Secure Boot and DMA Protection`
- **Virtualization Based Protection of Code Integrity** → `Enabled with UEFI lock` (empêche la désactivation sans accès physique au firmware)
- **Credential Guard Configuration** → `Enabled with UEFI lock`

Chemin registre correspondant :

```
HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\
  EnableVirtualizationBasedSecurity = 1
  RequirePlatformSecurityFeatures   = 3  (Secure Boot + DMA)
  HypervisorEnforcedCodeIntegrity   = 1
  LsaCfgFlags                       = 1  (Credential Guard avec UEFI lock)
```

### :material-check-all: Vérification de l'activation

```powershell title="Verify-CredentialGuard.ps1"
# Check Credential Guard status on local or remote machines
$computers = @("WKS-001", "WKS-002")  # or Get-ADComputer query

Invoke-Command -ComputerName $computers -ScriptBlock {
    $info = Get-ComputerInfo -Property `
        DeviceGuardVirtualizationBasedSecurityStatus,
        DeviceGuardCredentialGuardStatus,
        DeviceGuardSecurityServicesRunning

    [PSCustomObject]@{
        Computer         = $env:COMPUTERNAME
        VBS_Status       = $info.DeviceGuardVirtualizationBasedSecurityStatus
        CredGuard_Status = $info.DeviceGuardCredentialGuardStatus
        ServicesRunning  = $info.DeviceGuardSecurityServicesRunning -join ", "
    }
}
```

```title="Résultat attendu"
Computer         : WKS-001
VBS_Status       : Running
CredGuard_Status : Running
ServicesRunning  : CredentialGuard, HypervisorEnforcedCodeIntegrity
```

Vous pouvez également vérifier via `msinfo32.exe` : la section **System Summary** affiche `Virtualization-based security` et `Credential Guard`.

### :material-alert-circle: Impact sur l'authentification NTLM

Credential Guard protège les tickets Kerberos et les hash NTLMv2 en mémoire, mais impose une contrainte importante : certaines authentifications NTLM de niveau NTLMv1 échouent sur les machines protégées.

Applications typiquement impactées :

- Anciens serveurs NAS avec authentification NTLMv1
- Certains logiciels de supervision utilisant WMI avec délégation non contrainte
- Applications tierces qui stockent des credentials en clair dans la mémoire du processus

!!! danger "Production — Rollout progressif obligatoire"
    Ne déployez pas Credential Guard sur l'ensemble du parc en une seule GPO liée au domaine. Commencez par un groupe pilote de 50 machines, laissez tourner une semaine, collectez les remontées d'incidents, puis élargissez par vague de 20 % du parc.

    La désactivation de Credential Guard avec UEFI lock nécessite un accès au firmware de la machine — elle ne peut pas se faire à distance. Tester avant de déployer n'est pas optionnel.

!!! quote "En résumé"
    - Prérequis : UEFI Secure Boot + TPM 2.0 + Hyper-V activé
    - GPO : `Turn On Virtualization Based Security` dans le nœud Device Guard
    - Option UEFI lock : empêche la désactivation sans intervention physique — à utiliser en production
    - Validation : `Get-ComputerInfo -Property DeviceGuardCredentialGuardStatus`
    - Impact NTLM : testez les applications tierces et NAS avant déploiement large

---

## :material-shield-bug: Exploit Protection via GPO

### Positionnement : le successeur d'EMET

Exploit Protection intègre directement dans Windows les mitigations qu'EMET (Enhanced Mitigation Experience Toolkit) apportait sous Windows 7/8. Il est actif par défaut depuis Windows 10 1709, mais peut être configuré finement par GPO et par processus.

Deux niveaux de configuration coexistent :

- **System-wide** : mitigations appliquées à tous les processus sur la machine
- **Per-application** : configuration ciblée par processus (chemin ou nom d'exécutable)

### :material-xml: La configuration XML

Exploit Protection s'exporte et s'importe via un fichier XML. C'est ce fichier qui est distribué par GPO.

Structure minimale d'un fichier de configuration :

```xml title="ExploitProtection.xml"
<?xml version="1.0" encoding="UTF-8"?>
<MitigationPolicy>

  <!-- System-wide settings applied to all processes -->
  <SystemConfig>
    <Aslr ForceRelocateImages="true" RequireInfo="false" BottomUp="true" HighEntropy="true" />
    <Dep OverrideMitigationOptions="true" Enable="true" EmulateAtlThunks="false" />
    <Sehop Enable="true" TelemetryOnly="false" />
    <Heap TerminateOnError="true" />
  </SystemConfig>

  <!-- Per-application settings -->
  <AppConfig Executable="excel.exe">
    <DEP Override="true" Enable="true" EmulateAtlThunks="false" />
    <ASLR Override="true" ForceRelocateImages="true" RequireInfo="false" BottomUp="true" HighEntropy="true" />
    <ChildProcess Override="true" Disallow="true" />
  </AppConfig>

  <AppConfig Executable="winword.exe">
    <DEP Override="true" Enable="true" EmulateAtlThunks="false" />
    <ASLR Override="true" ForceRelocateImages="true" RequireInfo="false" BottomUp="true" HighEntropy="true" />
    <ChildProcess Override="true" Disallow="true" />
  </AppConfig>

</MitigationPolicy>
```

Les nœuds de mitigation disponibles dans `<AppConfig>` et `<SystemConfig>` :

| Nœud XML | Mitigation | Description |
|----------|-----------|-------------|
| `<DEP>` | Data Execution Prevention | Interdit l'exécution depuis des zones de données |
| `<ASLR>` | Address Space Layout Randomization | Randomise les adresses mémoire des modules |
| `<ChildProcess>` | Child Process Creation | Interdit la création de processus enfants |
| `<Heap>` | Heap Integrity | Termine le processus en cas de corruption de heap |
| `<Sehop>` | Structured Exception Handler Overwrite Protection | Protège la chaîne SEH |
| `<CFG>` | Control Flow Guard | Valide les cibles d'appels indirects |

### :material-powershell: Déploiement via GPO

**Option 1 — Via ADMX (recommandée)**

Le paramètre GPMC est dans :

```
Configuration ordinateur > Modèles d'administration > Composants Windows > Windows Defender Exploit Guard > Exploit Protection
```

Paramètre : **Use a common set of exploit protection settings**

Valeur : chemin UNC vers le fichier XML stocké sur un partage SYSVOL ou un partage réseau accessible par les machines :

```
\\contoso.com\SYSVOL\contoso.com\ExploitProtection\config.xml
```

**Option 2 — Via script de démarrage GPO**

```powershell title="Apply-ExploitProtection.ps1"
# Deploy Exploit Protection XML configuration — used as a GPO startup script
$xmlSource = "\\contoso.com\SYSVOL\contoso.com\ExploitProtection\config.xml"
$localPath  = "C:\Windows\Temp\ep-config.xml"

# Copy XML locally to avoid UNC dependency at runtime
Copy-Item -Path $xmlSource -Destination $localPath -Force

# Apply system-wide + per-app configuration
Set-ProcessMitigation -PolicyFilePath $localPath

Write-EventLog -LogName Application -Source "GPO-Security" -EventId 5100 `
    -EntryType Information -Message "Exploit Protection policy applied from $xmlSource"
```

### :material-export: Export de la configuration existante

Avant de modifier, exportez toujours l'état actuel d'une machine de référence :

```powershell title="Export-ExploitProtection.ps1"
# Export current Exploit Protection settings to XML
$outputPath = "C:\Temp\ep-baseline-$(Get-Date -Format 'yyyyMMdd').xml"
Get-ProcessMitigation -RegistryConfigFilePath $outputPath
Write-Host "Configuration exported to: $outputPath"
```

!!! warning "À surveiller — Applications incompatibles"
    Certaines applications 32 bits anciennes (notamment les applications de supervision industrielle et les connecteurs ERP legacy) sont incompatibles avec DEP forcé ou CFG. Testez le fichier XML sur un parc pilote avant déploiement large. En cas de crash applicatif, utilisez `Set-ProcessMitigation -Name <exe> -Disable DEP` en urgence, puis affinez le XML.

!!! quote "En résumé"
    - Exploit Protection remplace EMET — actif par défaut, configurable en détail
    - Configuration via fichier XML : system-wide + par processus
    - Déployez via le paramètre ADMX `Use a common set of exploit protection settings` avec chemin UNC SYSVOL
    - Exportez la config de référence avec `Get-ProcessMitigation -RegistryConfigFilePath`
    - Testez sur un parc pilote — certaines applis legacy sont incompatibles avec DEP forcé

---

## :material-fingerprint: Windows Hello for Business via GPO

### Prérequis selon le mode de déploiement

Windows Hello for Business (WHfB) remplace le mot de passe par une authentification forte (biométrie ou PIN) liée à un matériel (TPM). Deux modes d'infrastructure existent :

| Mode | Prérequis infrastructure | Authentification |
|------|-------------------------|-----------------|
| **Key Trust** | Azure AD ou Hybrid Join, pas de PKI requise | Clé cryptographique liée au TPM |
| **Certificate Trust** | PKI d'entreprise (CA interne), ADFS ou Azure AD | Certificat délivré par la CA |

Le mode Key Trust est plus simple à déployer mais nécessite au minimum un niveau fonctionnel de domaine Windows Server 2016. Le mode Certificate Trust offre une compatibilité plus large mais exige une infrastructure PKI opérationnelle.

### :material-cog: Configuration GPO

Le nœud GPMC pour WHfB est dans :

```
Configuration ordinateur > Modèles d'administration > Composants Windows > Windows Hello for Business
```

Chemin registre de base :

```
HKLM\SOFTWARE\Policies\Microsoft\PassportForWork\
```

### :material-table: Paramètres registre clés

| Paramètre GPMC | Valeur registre | Type | Recommandation |
|---|---|---|---|
| Use Windows Hello for Business | `Enabled` = 1 | DWORD | Activé — obligatoire |
| Use a hardware security device | `RequireSecurityDevice` = 1 | DWORD | Activé (TPM requis) |
| Use certificate for on-premises authentication | `UseCertificateForOnPremAuth` = 1/0 | DWORD | 1 pour Certificate Trust, 0 pour Key Trust |
| Minimum PIN length | `MinimumPINLength` = 6 | DWORD | 6 chiffres minimum |
| Maximum PIN length | `MaximumPINLength` = 127 | DWORD | Valeur par défaut suffisante |
| Expiration de PIN | `PINExpirationDays` = 0 | DWORD | 0 = pas d'expiration (recommandé) |
| Require digits in PIN | `DigitsRequired` = 1 | DWORD | Activé |
| Require special characters in PIN | `SpecialCharactersRequired` = 0 | DWORD | Désactivé — augmente la friction |

### :material-powershell: Vérification du provisionnement WHfB

```powershell title="Verify-WHfB.ps1"
# Check Windows Hello for Business provisioning status on endpoints
$computers = Get-ADComputer -Filter { OperatingSystem -like "*Windows 1*" } |
    Select-Object -ExpandProperty Name

$results = Invoke-Command -ComputerName $computers -ErrorAction SilentlyContinue -ScriptBlock {
    # Check registry policy
    $wHfBPath = "HKLM:\SOFTWARE\Policies\Microsoft\PassportForWork"
    $enabled  = (Get-ItemProperty $wHfBPath -EA SilentlyContinue).Enabled

    # Check NGC key provisioning (user-level, run as system may not see user keys)
    $ngcPath  = "C:\Windows\ServiceProfiles\LocalService\AppData\Local\Microsoft\NGC"
    $ngcExists = Test-Path $ngcPath

    [PSCustomObject]@{
        Computer    = $env:COMPUTERNAME
        PolicyEnabled = $enabled
        NgcFolder   = $ngcExists
        TPMPresent  = (Get-Tpm -ErrorAction SilentlyContinue).TpmPresent
        TPMReady    = (Get-Tpm -ErrorAction SilentlyContinue).TpmReady
    }
}

$results | Format-Table -AutoSize
```

```title="Résultat attendu"
Computer    PolicyEnabled  NgcFolder  TPMPresent  TPMReady
--------    -------------  ---------  ----------  --------
WKS-001     1              True       True        True
WKS-002     1              True       True        True
```

!!! warning "À surveiller — Appareils sans TPM ou en VM"
    WHfB avec TPM requis (`RequireSecurityDevice = 1`) bloque le provisionnement sur les machines sans puce TPM 2.0 — typiquement les VM legacy et certains postes anciens. Si votre parc est hétérogène, déployez d'abord avec `RequireSecurityDevice = 0` pour les VM, et `= 1` uniquement sur les postes physiques via un filtrageWMI ou une OU dédiée.

!!! quote "En résumé"
    - Deux modes : Key Trust (simple, Hybrid Join suffit) et Certificate Trust (PKI requise)
    - Chemin registre : `HKLM\SOFTWARE\Policies\Microsoft\PassportForWork\`
    - TPM 2.0 requis pour le mode `RequireSecurityDevice = 1`
    - Vérifiez le provisionnement avec le dossier NGC et `Get-Tpm`
    - Filtrez par OU ou WMI filter pour exclure les VM de l'exigence TPM

---

## :material-refresh: Config Refresh (24H2+)

Config Refresh reapplique periodiquement certaines politiques de securite meme si la version de la GPO n'a pas change. Le cas d'usage est simple : si un administrateur local ou un outil tiers modifie un reglage protege, Windows le remet automatiquement dans l'etat attendu au cycle suivant.

### Fonctionnement attendu

L'intervalle de reference reste proche de `gpupdate` : 90 minutes par defaut. La difference est que Config Refresh ne se contente pas de verifier si la GPO a change cote SYSVOL ; il reapplique le sous-ensemble de parametres de securite pris en charge.

| Point | `gpupdate` classique | Config Refresh |
|---|---|---|
| Declencheur | Changement de version GPO ou cycle standard | Cycle standard, meme sans changement detecte |
| Portee | GPO completes selon CSE | Sous-ensemble de parametres de securite |
| Preferences GPP | Oui, selon extension | Non |
| Scripts | Oui, selon policy | Non |

### Cles registre

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\System
```

| Valeur | Type | Donnees | Effet |
|---|---|---|---|
| `ConfigRefreshEnabled` | REG_DWORD | `1` | Active Config Refresh |
| `ConfigRefreshInterval` | REG_DWORD | `90` | Intervalle en minutes, minimum attendu : 30 |

**Chemin GPO** : `Configuration ordinateur > Modeles d'administration > Systeme > Strategie de groupe > Activer l'actualisation de la configuration`

!!! warning "Mapping a verifier"
    La disponibilite de Config Refresh et de ces valeurs depend des ADMX/Policy CSP livres avec la build 24H2 cible. Si le parametre n'apparait pas dans votre Central Store, ne forcez pas ces cles sans test sur un poste pilote.

### Impact terrain

Si une modification locale desactive Defender ou affaiblit un reglage de securite couvert, Config Refresh peut restaurer l'etat voulu sans attendre une modification de GPO. Ce n'est pas un remplacement de la surveillance de conformite : journalisez toujours les derives et cherchez leur origine.

!!! quote "En resume"
    - Config Refresh reapplique des politiques de securite meme sans changement de version de GPO.
    - L'intervalle vise reste proche de 90 minutes, avec un plancher a valider autour de 30 minutes.
    - Il ne remplace pas les preferences, scripts ou extensions GPO classiques.
    - Verifiez le mapping ADMX/Policy CSP de `ConfigRefreshEnabled` sur la build 24H2 cible.

---

## :material-shield-key: LSASS PPL par défaut

Sur les installations propres de Windows 11 22H2+ et 24H2 compatibles HVCI, LSASS peut tourner par defaut en Protected Process Light (PPL). Ce mode limite l'injection, la lecture memoire et le chargement de composants non autorises dans le processus `lsass.exe`.

### Registre et GPO

La cle historique de runtime reste utile pour l'audit :

```
HKLM\SYSTEM\CurrentControlSet\Control\Lsa
```

| Valeur | Type | Donnees | Effet |
|---|---|---|---|
| `RunAsPPL` | REG_DWORD | `1` | Active PPL avec verrouillage UEFI selon contexte |
| `RunAsPPL` | REG_DWORD | `2` | Active PPL sans verrouillage UEFI |

Le parametre de politique moderne est expose dans les ADMX `Local Security Authority` :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\System
```

**Chemin GPO** : `Configuration ordinateur > Modeles d'administration > Systeme > Local Security Authority > Configurer LSASS pour qu'il s'execute en tant que processus protege`

| Valeur de politique | Signification |
|---|---|
| `Disabled` | LSASS ne tourne pas en processus protege |
| `EnabledWithUEFILock` | PPL actif avec verrouillage UEFI |
| `EnabledWithoutUEFILock` | PPL actif sans verrouillage UEFI |

### Verification

```powershell title="Verifier LSASS PPL et Device Guard"
Get-CimInstance Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard |
    Select-Object LsaCfgFlags, SecurityServicesConfigured, SecurityServicesRunning

Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name RunAsPPL -ErrorAction SilentlyContinue
```

### Lien avec Credential Guard

Credential Guard et LSASS PPL sont complementaires, pas equivalents. PPL durcit le processus `lsass.exe`; Credential Guard isole certains secrets dans VBS via `LsaIso.exe`. Sur un parc moderne, les deux doivent etre evalues ensemble, avec un pilote applicatif pour les SSP/AP legacy.

!!! warning "Upgrades depuis versions anciennes"
    Les installations propres et les upgrades ne se comportent pas toujours pareil. Sur un poste migre depuis une version anterieure, ne supposez pas que PPL est actif : auditez `RunAsPPL`, la policy effective et les journaux Device Guard.

!!! quote "En resume"
    - LSASS PPL reduit la surface de vol de secrets dans `lsass.exe`.
    - `RunAsPPL=2` active PPL sans verrouillage UEFI ; le pilotage prefere reste la GPO `Local Security Authority`.
    - Credential Guard isole des secrets via VBS, mais ne remplace pas PPL.
    - Auditez les postes migres : PPL n'est pas garanti sur tous les upgrades.

---

## :material-test-tube: Vérification globale post-déploiement

Après déploiement de l'ensemble des politiques de ce chapitre, exécutez un audit consolidé sur un échantillon représentatif du parc :

```powershell title="Security-EndpointAudit.ps1"
# Consolidated endpoint security audit — Defender, ASR, Credential Guard, WHfB
$computers = Get-ADComputer -Filter { OperatingSystem -like "*Windows*" } |
    Select-Object -First 10 -ExpandProperty Name

$audit = Invoke-Command -ComputerName $computers -ErrorAction SilentlyContinue -ScriptBlock {

    # Defender status
    $mpStatus = Get-MpComputerStatus -ErrorAction SilentlyContinue

    # ASR rules count
    $asrPath  = "HKLM:\SOFTWARE\Policies\Microsoft\Windows Defender\Windows Defender Exploit Guard\ASR\Rules"
    $asrCount = (Get-Item $asrPath -EA SilentlyContinue)?.GetValueNames().Count

    # Credential Guard
    $cgStatus = (Get-ComputerInfo -Property DeviceGuardCredentialGuardStatus `
        -ErrorAction SilentlyContinue).DeviceGuardCredentialGuardStatus

    # WHfB policy
    $wHfBEnabled = (Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\PassportForWork" `
        -EA SilentlyContinue).Enabled

    [PSCustomObject]@{
        Computer         = $env:COMPUTERNAME
        DefenderEnabled  = $mpStatus.AntivirusEnabled
        RealTimeProtect  = $mpStatus.RealTimeProtectionEnabled
        ASRRulesDeployed = $asrCount
        CredentialGuard  = $cgStatus
        WHfBPolicy       = $wHfBEnabled
        TPMReady         = (Get-Tpm -EA SilentlyContinue).TpmReady
    }
}

$audit | Sort-Object Computer | Format-Table -AutoSize

# Export to CSV for compliance reporting
$audit | Export-Csv -Path ".\endpoint-security-audit-$(Get-Date -Format 'yyyyMMdd').csv" `
    -NoTypeInformation -Encoding UTF8
Write-Host "Audit exported to CSV."
```


!!! quote "En résumé"
    - Validez toujours le résultat sur un poste ou un utilisateur réellement dans le périmètre avant d’élargir.
    - Conservez les commandes et résultats de contrôle comme preuve de conformité post-déploiement.
    - Retenez surtout ce qui change la portée, l’ordre d’application ou le résultat final observé.
    - Ce résumé sert à vérifier que vous avez retenu le mécanisme, sa portée et sa conséquence pratique.
---

## Références croisées

- [Stratégies de sécurité natives](../bible-gpo/13-securite-strategies.md) — Paramètres de sécurité locaux, stratégies de comptes et d'audit
- [AppLocker et WDAC](../bible-gpo/15-applocker-wdac.md) — Contrôle d'exécution des applications
- [Sécuriser les GPO elles-mêmes](24-securite-gpo.md) — Délégation, filtrage de sécurité, protection SYSVOL

!!! quote "En résumé"
    - À relire : Stratégies de sécurité natives.
    - À relire : AppLocker et WDAC.
    - À relire : Sécuriser les GPO elles-mêmes.
    - Ces renvois prolongent le chapitre avec des mécanismes complémentaires ou des cas d’usage voisins.
    - Gardez ces chapitres sous la main pour le diagnostic ou la conception d’une GPO liée à ce thème.
