---
description: Migration GPO vers Intune — Group Policy Analytics, mapping CSP, gaps MDM, workload switching, validation
tags: [gpo-admins, intune, migration, gpa, csp, mdm, workload-switching, admx-backed]
---
# Migration GPO vers Intune

!!! abstract "Ce que vous allez apprendre"
    - Utiliser **Group Policy Analytics** dans le portail Intune pour analyser automatiquement vos GPO existantes et obtenir un rapport de couverture CSP avec pourcentage de prise en charge
    - Exporter toutes vos GPO en XML via PowerShell pour une analyse en lot, identifier les paramètres sans équivalent MDM et classer les gaps par catégorie
    - Suivre un **workflow en 7 étapes** de migration par workload : de l'analyse initiale jusqu'à la suppression sécurisée de la GPO, avec une fenêtre d'observation obligatoire de deux semaines
    - Gérer la coexistence GPO/Intune sur les machines **Hybrid Join** et comprendre le mécanisme de workload switching pour éviter les conflits silencieux
    - Importer des ADMX tierces (Chrome, Firefox, Office) dans Intune et recréer les profils équivalents à vos GPO existantes

!!! tip "Si vous ne retenez qu'une chose"
    Ne supprimez jamais une GPO le jour même où vous activez la politique Intune correspondante. Un appareil hors ligne peut ne pas recevoir la politique MDM pendant plusieurs heures. La **fenêtre de conformité vide** entre la suppression de la GPO et l'application réelle de la politique Intune est le piège numéro un des migrations. Deux semaines d'observation minimum, point non négociable.

---

## Contexte de production

La migration des GPO vers Intune n'est pas un événement ponctuel. C'est un processus progressif, workload par workload, sur plusieurs mois.

La réalité terrain : la majorité des organisations qui commencent cette migration ont entre 200 et 800 GPO actives, dont 30 à 40 % ont été créées pour résoudre un problème précis et jamais documentées. Avant de migrer quoi que ce soit, vous devez savoir ce que vous avez.

Ce chapitre part du principe que vos machines sont en **Hybrid Join** (jointes au domaine AD et enregistrées dans Azure AD) et que vous avez un tenant Intune actif. Le co-management MECM est traité dans le chapitre suivant.


!!! quote "En résumé"
    - La migration des GPO vers Intune n'est pas un événement ponctuel.
    - C'est un processus progressif, workload par workload, sur plusieurs mois.
    - Avant de migrer quoi que ce soit, vous devez savoir ce que vous avez.
    - Le contexte de production fixe les contraintes réelles de réseau, de portée et d’exploitation qui gouvernent tout le chapitre.
    - Retenez les hypothèses opérationnelles avant de choisir un modèle de liaison ou de déploiement.
---

## :material-chart-bar: Group Policy Analytics : analyser avant de migrer

### Ce qu'est Group Policy Analytics

**Group Policy Analytics (GPA)** est un outil intégré au portail Intune (`Devices > Group Policy Analytics`). Il prend en entrée un fichier XML d'export de GPO et produit un rapport indiquant, pour chaque paramètre, s'il est :

- **Supported** : un équivalent CSP (Configuration Service Provider) existe dans Intune
- **Not supported** : aucun équivalent MDM connu
- **Deprecated** : le paramètre GPO existe mais Microsoft recommande de ne plus l'utiliser

Le rapport affiche également un **pourcentage de couverture** global pour chaque GPO importée.

### Exporter une GPO en XML

L'export s'effectue avec `Get-GPOReport`, cmdlet native du module RSAT `GroupPolicy`.

```powershell title="Export-SingleGPO.ps1"
# Export a single GPO to XML for Group Policy Analytics import
Import-Module GroupPolicy

$gpoName    = "Baseline-Securite-Postes"
$outputPath = "C:\GPO_Export\$gpoName.xml"

Get-GPOReport -Name $gpoName -ReportType XML -Path $outputPath

Write-Host "Exported: $outputPath"
```

### Exporter toutes les GPO en lot

En production, vous n'allez pas exporter 200 GPO à la main. Ce script parcourt le domaine et exporte chaque GPO dans un dossier dédié.

```powershell title="Export-AllGPOs.ps1"
# Batch export all GPOs to XML for Group Policy Analytics bulk import
Import-Module GroupPolicy

$exportRoot = "C:\GPO_Export\$(Get-Date -Format 'yyyy-MM-dd')"
New-Item -ItemType Directory -Path $exportRoot -Force | Out-Null

$allGPOs = Get-GPO -All
$total   = $allGPOs.Count
$current = 0

foreach ($gpo in $allGPOs) {
    $current++
    # Sanitize GPO display name for use as filename
    $safeName   = $gpo.DisplayName -replace '[\\/:*?"<>|]', '_'
    $outputFile = Join-Path $exportRoot "$safeName.xml"

    try {
        Get-GPOReport -Guid $gpo.Id -ReportType XML -Path $outputFile
        Write-Host "[$current/$total] OK : $($gpo.DisplayName)"
    }
    catch {
        Write-Warning "[$current/$total] FAILED : $($gpo.DisplayName) — $($_.Exception.Message)"
    }
}

Write-Host "`nExport complete. Files saved to: $exportRoot"
```

```powershell title="Résultat attendu"
[1/247] OK : Baseline-Securite-Postes
[2/247] OK : Config-Chrome-Entreprise
[3/247] OK : Redirect-Dossiers-Utilisateurs
...
[247/247] OK : Legacy-IE-Compat-Intranet

Export complete. Files saved to: C:\GPO_Export\2025-06-12
```

### Importer dans Group Policy Analytics

Dans le portail Intune (`Devices > Group Policy Analytics`), cliquez sur **Import** et sélectionnez les fichiers XML exportés. L'analyse prend quelques secondes par fichier.

Une fois l'analyse terminée, chaque GPO affiche :

| Colonne rapport GPA | Signification |
|---|---|
| **MDM Support** | Pourcentage de paramètres avec équivalent CSP |
| **Supported count** | Nombre de paramètres migrables |
| **Not supported count** | Nombre de paramètres sans équivalent MDM |
| **Deprecated count** | Paramètres GPO obsolètes |
| **Unknown count** | Paramètres que GPA ne reconnaît pas (modèles ADMX tiers non analysés) |

Une GPO avec 85 % de couverture ou plus est une bonne candidate pour une migration directe. En dessous de 60 %, anticipez des workarounds.

!!! warning "À surveiller — Les paramètres 'Unknown'"
    Les paramètres classés **Unknown** dans GPA sont souvent des ADMX tierces (Chrome, Firefox, applications métier). GPA ne les analyse pas automatiquement. Ce n'est pas qu'ils n'ont pas d'équivalent Intune — c'est que GPA ne sait pas. Analysez-les manuellement avant de conclure à un gap réel.

!!! quote "En résumé"
    - GPA s'alimente de fichiers XML produits par `Get-GPOReport -ReportType XML`
    - Le rapport classe chaque paramètre : Supported / Not supported / Deprecated / Unknown
    - Un pourcentage de couverture >= 85 % = GPO candidate à migration directe
    - Les ADMX tierces génèrent des "Unknown" — ce n'est pas forcément un gap MDM

---

## :material-map-marker-path: Processus de migration en 7 étapes

### Vue d'ensemble du workflow

La migration d'une GPO vers Intune ne se fait pas en un clic. Chaque workload doit passer par sept étapes distinctes, avec validation formelle avant de passer à la suivante.

| Étape | Action | Responsable | Outil | Validation |
|---|---|---|---|---|
| **1. Analyse** | Importer la GPO dans GPA, identifier les paramètres sans CSP | Admin IT | Portail Intune — GPA | Rapport GPA exporté |
| **2. Priorisation** | Classer les paramètres : sécurité d'abord, confort ensuite | Architecte / Admin | Excel / CSV | Backlog priorisé validé |
| **3. Création du profil Intune** | Créer le profil de configuration Intune équivalent | Admin IT | Portail Intune | Profil créé, assigné au groupe test |
| **4. Test pilote** | Déployer sur un groupe test (10–20 machines), observer pendant 48 h | Admin IT | Portail Intune + `Get-MDMDeviceDetail` | 0 erreur dans le rapport de conformité |
| **5. Désactivation de la GPO** | **Désactiver** (pas supprimer) la GPO sur le groupe pilote | Admin AD | GPMC | `gpresult /H` ne montre plus la GPO |
| **6. Observation** | Surveiller les tickets, les événements et la conformité pendant **2 semaines** | Admin IT + Support | Intune Compliance + helpdesk | Aucun incident lié au paramètre |
| **7. Suppression de la GPO** | Supprimer la GPO après la fenêtre d'observation | Admin AD | GPMC | GPO absente de `Get-GPO -All` |

### Étape 1 — Identifier les gaps avant de commencer

Avant de créer un seul profil Intune, exportez le rapport GPA en CSV depuis le portail.

```powershell title="Parse-GPAReport.ps1"
# Parse exported GPA CSV to identify not-supported settings by GPO
$gpaCsvPath = "C:\GPO_Export\GPA_Report_Export.csv"

$report = Import-Csv -Path $gpaCsvPath

# Group by GPO name and count supported vs not-supported
$summary = $report | Group-Object -Property 'GPO Name' | ForEach-Object {
    $settings        = $_.Group
    $supported       = ($settings | Where-Object { $_.MDMSupport -eq 'Supported' }).Count
    $notSupported    = ($settings | Where-Object { $_.MDMSupport -eq 'Not supported' }).Count
    $deprecated      = ($settings | Where-Object { $_.MDMSupport -eq 'Deprecated' }).Count
    $total           = $settings.Count
    $coveragePct     = if ($total -gt 0) { [math]::Round(($supported / $total) * 100, 1) } else { 0 }

    [PSCustomObject]@{
        GPOName      = $_.Name
        Total        = $total
        Supported    = $supported
        NotSupported = $notSupported
        Deprecated   = $deprecated
        Coverage_Pct = $coveragePct
    }
}

$summary | Sort-Object Coverage_Pct -Descending | Format-Table -AutoSize
```

### Étape 4 — Valider l'application MDM sur les machines pilotes

```powershell title="Verify-MDMPolicyApplied.ps1"
# Verify that an MDM policy has been applied on a list of pilot machines
# Requires WinRM access and the MDM Diagnostics module

$pilotMachines = @("PC-PILOT-001", "PC-PILOT-002", "PC-PILOT-003")
# CSP path to check — adapt to the policy being migrated
$cspPath       = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"
$valueName     = "WUServer"

foreach ($computer in $pilotMachines) {
    try {
        $result = Invoke-Command -ComputerName $computer -ScriptBlock {
            param($path, $val)
            $reg = Get-ItemProperty -Path $path -Name $val -ErrorAction SilentlyContinue
            [PSCustomObject]@{
                Computer   = $env:COMPUTERNAME
                ValueFound = ($null -ne $reg)
                Value      = $reg.$val
                # Check if value was set by MDM (written under SOFTWARE\Policies by MDM bridge)
                Source     = if (Test-Path "HKLM:\SOFTWARE\Microsoft\Enrollments") { "MDM-enrolled" } else { "Not enrolled" }
            }
        } -ArgumentList $cspPath, $valueName
        $result
    }
    catch {
        Write-Warning "Cannot reach $computer : $($_.Exception.Message)"
    }
}
```

```powershell title="Résultat attendu"
Computer      ValueFound Value                         Source
--------      ---------- -----                         ------
PC-PILOT-001  True       https://wsus.corp.local:8530  MDM-enrolled
PC-PILOT-002  True       https://wsus.corp.local:8530  MDM-enrolled
PC-PILOT-003  True       https://wsus.corp.local:8530  MDM-enrolled
```

!!! danger "Production — Ne désactivez la GPO qu'après validation des 3 machines"
    L'étape 5 (désactivation de la GPO) ne doit intervenir que si **100 % des machines pilotes** affichent `ValueFound = True` via MDM. Une machine qui n'a pas encore reçu la politique Intune est en état non conforme dès la désactivation de la GPO. Sur un réseau avec des appareils nomades, ce délai peut atteindre 8 heures.

!!! quote "En résumé"
    - 7 étapes séquentielles : Analytics → Priorisation → Profil → Pilote → Désactivation GPO → Observation → Suppression
    - La sécurité passe en premier dans la priorisation
    - La fenêtre d'observation de 2 semaines n'est pas optionnelle
    - `Get-MDMDeviceDetail` et l'inspection du registre valident l'application réelle de la politique MDM

---

## :material-alert-circle-outline: Paramètres sans équivalent MDM

### Les catégories à risque

Certains paramètres GPO n'ont tout simplement pas d'équivalent CSP dans Intune. Ce n'est pas un oubli de Microsoft — c'est souvent parce que la fonctionnalité sous-jacente n'existe pas dans le monde Modern Management.

Les catégories les plus fréquentes en production :

| Catégorie GPO | Pourquoi pas de CSP | Workaround recommandé |
|---|---|---|
| Scripts Startup/Shutdown complexes | Les scripts MDM sont exécutés en contexte différent (pas SYSTEM au boot) | Intune Remediations (PowerShell) en contexte SYSTEM |
| Folder Redirection (chemins UNC complexes) | Known Folders Move dans Intune ne supporte pas les conditions avancées | Maintenir la GPO pour ce paramètre spécifique |
| Restrictions Internet Explorer avancées | IE n'existe plus en tant que browser manageable par CSP | Supprimer si IE est désinstallé ; migrer vers Edge |
| Logon scripts utilisateur complexes (>5 min) | Timeout MDM plus court que les scripts GPO | Intune Custom Scripts (attention aux limites de durée) |
| Préférences GPO — Raccourcis bureau par utilisateur | GPP User Preferences n'ont pas d'équivalent MDM direct | Intune Win32 App ou script de remédiation |
| Préférences GPO — Sources de données ODBC | Très spécifique à l'infrastructure legacy | Script PowerShell déployé via Remediations |
| Mappage lecteurs réseau conditionnel | GPP Drive Maps avec filtrage WMI | Logon script via Intune Custom Scripts |
| Stratégies locales avancées (User Rights) | Partiellement supporté selon la version d'Intune | CSP `Policy/Config/UserRights` — vérifier la liste |
| AppLocker (règles héritées complexes) | WDAC (Windows Defender Application Control) remplace AppLocker | Migration WDAC via Intune ou maintien GPO transitoire |
| Configuration imprimantes via GPP (Port 9100 custom) | Printer CSP limité aux imprimantes partagées standard | Intune Win32 App avec pilote + script port |

### Stratégies de workaround détaillées

**Option A — Intune Remediations**

Les Remediations (anciennement Proactive Remediations) permettent d'exécuter un script PowerShell de détection et un script de correction. C'est le remplacement naturel des scripts Startup/Shutdown.

```powershell title="Remediation-Detection-StartupScript.ps1"
# Detection script: check if the registry value expected from the legacy startup script is present
$expectedKey   = "HKLM:\SOFTWARE\Corp\StartupConfig"
$expectedValue = "Configured"
$expectedData  = "v2.1"

try {
    $actual = Get-ItemPropertyValue -Path $expectedKey -Name $expectedValue -ErrorAction Stop
    if ($actual -eq $expectedData) {
        # Compliant — no remediation needed
        exit 0
    }
    else {
        # Value exists but wrong data
        exit 1
    }
}
catch {
    # Key or value missing
    exit 1
}
```

```powershell title="Remediation-Remediate-StartupScript.ps1"
# Remediation script: apply what the legacy GPO startup script used to do
$targetKey   = "HKLM:\SOFTWARE\Corp\StartupConfig"
$targetValue = "Configured"
$targetData  = "v2.1"

try {
    if (-not (Test-Path $targetKey)) {
        New-Item -Path $targetKey -Force | Out-Null
    }
    Set-ItemProperty -Path $targetKey -Name $targetValue -Value $targetData -Type String
    Write-Host "Remediation applied successfully."
    exit 0
}
catch {
    Write-Error "Remediation failed: $($_.Exception.Message)"
    exit 1
}
```

**Option B — Maintenir la GPO pour les paramètres orphelins**

La migration n'est pas tout-ou-rien. Il est parfaitement valide de migrer 95 % d'une GPO vers Intune et de conserver une GPO légère pour les 5 % sans équivalent.

Créez une GPO nommée `LEGACY-NoMDM-[nom-original]` qui ne contient **que** les paramètres sans CSP. Cette GPO reste sur les machines Hybrid Join. Elle n'interfère pas avec Intune tant que les paramètres ne se chevauchent pas.

!!! warning "À surveiller — Conflits entre GPO résiduelle et profil Intune"
    Si un paramètre est à la fois dans la GPO résiduelle et dans le profil Intune, le comportement dépend du paramètre. Pour les valeurs de registre sous `SOFTWARE\Policies`, la **dernière écriture gagne**. GPO écrit lors du prochain cycle (90 min). MDM écrit lors de la prochaine synchronisation (8 h max). Résultat : flipping de configuration. Éliminez tout chevauchement avant la mise en production.

!!! quote "En résumé"
    - Les scripts Startup/Shutdown complexes migrent vers les **Intune Remediations**
    - Folder Redirection avancé, ODBC, et lecteurs réseau conditionnels restent sur GPO résiduelle
    - AppLocker legacy migre vers WDAC (projet distinct, pas une migration directe)
    - Une GPO `LEGACY-NoMDM-*` dédiée aux orphelins est préférable à une GPO mixte

---

## :material-laptop-account: Hybrid Join pendant la migration

### Pourquoi les GPO continuent de s'appliquer

Sur une machine **Hybrid Joined**, `gpsvc` (Group Policy Client Service) est toujours actif. La machine est membre du domaine AD — elle applique les GPO normalement, toutes les 90 minutes, exactement comme avant l'enrôlement Intune.

L'enrôlement Intune ajoute un canal de gestion **supplémentaire**. Il ne remplace pas les GPO automatiquement. Les deux canaux coexistent.

Le risque réel n'est pas le conflit évident — c'est le conflit silencieux. Quand GPO et MDM définissent la même valeur de registre à des valeurs différentes, le comportement observé dépend de l'ordre d'écriture et du type de paramètre.

### Le workload switching

Dans le portail Intune, sous `Tenant administration > Cloud attach > Co-management`, vous pouvez basculer des **workloads** de MECM/GPO vers Intune, un par un.

Un workload basculé vers Intune signifie que c'est Intune qui a l'autorité pour ce domaine fonctionnel. La GPO correspondante doit être désactivée sur les machines concernées, sinon les deux canaux écrivent en parallèle.

| Workload | Géré par défaut | Après basculement vers Intune | Ce que "basculer" implique |
|---|---|---|---|
| **Compliance policies** | MECM / GPO | Intune | Les règles de conformité (BitLocker, version OS) viennent d'Intune |
| **Device configuration** | GPO | Intune | Les profils de configuration remplacent les GPO de config poste |
| **Endpoint Protection** | GPO / MECM Endpoint | Intune + Defender | Les politiques Defender, ASR, Firewall viennent d'Intune |
| **Resource access policies** | GPO | Intune | Certificats, Wi-Fi, VPN — profils Intune actifs |
| **Client apps** | MECM Software Center | Intune Company Portal | Déploiements applicatifs pilotés par Intune |
| **Office Click-to-Run apps** | GPO ou MECM | Intune (ODT via Intune) | Configuration M365 Apps gérée par Intune |
| **Windows Update policies** | GPO / WSUS | Windows Update for Business via Intune | Les anneaux WUfB remplacent les GPO WSUS |

!!! danger "Production — Ne basculez jamais deux workloads majeurs le même jour"
    Basculer Device Configuration et Endpoint Protection simultanément en production, c'est deux sources de changement silencieux en parallèle. Si un incident survient, vous ne savez pas lequel des deux workloads en est responsable. Espacez les bascules d'au moins une semaine, avec une fenêtre d'observation entre les deux.

### Détecter les conflits GPO / MDM sur les machines Hybrid

```powershell title="Detect-GPOMDMConflict.ps1"
# Compare GPO-managed vs MDM-managed registry values on a set of machines
# to detect potential write conflicts during co-management
$machines = @("PC-001", "PC-002", "PC-003", "PC-004", "PC-005",
              "PC-006", "PC-007", "PC-008", "PC-009", "PC-010")

# Define key paths to check — adapt to your workload
$checks = @(
    @{ Path = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"; Value = "WUServer" },
    @{ Path = "HKLM:\SOFTWARE\Policies\Microsoft\Windows Defender";      Value = "DisableAntiSpyware" },
    @{ Path = "HKLM:\SOFTWARE\Policies\Microsoft\Edge";                  Value = "HomepageLocation" }
)

$results = foreach ($machine in $machines) {
    foreach ($check in $checks) {
        try {
            $data = Invoke-Command -ComputerName $machine -ScriptBlock {
                param($p, $v)
                $val = Get-ItemProperty -Path $p -Name $v -ErrorAction SilentlyContinue
                # Check MDM bridge enrollment
                $mdmEnrolled = (Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Enrollments" -ErrorAction SilentlyContinue).Count -gt 0

                # GPO last write time from RSOP (approx — check GPO policy source)
                $gpResult = (& gpresult /scope computer /fo csv 2>$null) -join "`n"

                [PSCustomObject]@{
                    Machine    = $env:COMPUTERNAME
                    RegPath    = $p
                    RegValue   = $v
                    Data       = $val.$v
                    MDMActive  = $mdmEnrolled
                }
            } -ArgumentList $check.Path, $check.Value
            $data
        }
        catch {
            [PSCustomObject]@{
                Machine   = $machine
                RegPath   = $check.Path
                RegValue  = $check.Value
                Data      = "UNREACHABLE"
                MDMActive = $false
            }
        }
    }
}

$results | Format-Table -AutoSize
$results | Export-Csv -Path "C:\GPO_Export\ConflictCheck_$(Get-Date -Format 'yyyyMMdd').csv" -NoTypeInformation
```

### Matrice de décision : GPP vs Intune CSP vs GPO

:material-compare-horizontal:

Le choix entre GPP, GPO et Intune CSP dépend du contexte de jonction et de l'objectif de gestion.

| Critère | GPO classique | GPP (Group Policy Preferences) | Intune / CSP |
|---|---|---|---|
| Jonction requise | AD on-prem | AD on-prem | Azure AD / Hybrid |
| Révocation au désalignement | Oui (tattoo = Non) | Non (tattoo = Oui) | Oui |
| Scope | Ordi ou User | Ordi ou User | User ou Device |
| Ciblage granulaire | WMI Filter | Item-Level Targeting | Assignments + Filters |
| Settings complexes (XML, JSON) | Limité | Limité | Oui (OMA-URI) |
| Reporting centralisé | Non (gpresult local) | Non | Oui (Intune portal) |
| Hors réseau AD | Non | Non | Oui |
| Coexistence possible | — | Oui avec GPO | Oui avec GPO (ADMX-backed) |

!!! info "Règle de priorité en cas de conflit"
    En Hybrid Join, si un paramètre est défini à la fois par GPO et par Intune CSP,
    c'est le **dernier écrit** qui gagne — il n'y a pas de hiérarchie formelle entre
    les deux systèmes. La solution : ne pas dupliquer les paramètres entre GPO et Intune.
    Définir une frontière claire : GPO pour l'on-prem, Intune pour le cloud-native.

!!! tip "Migrer GPP → Intune"
    Pour migrer des GPP vers Intune, utiliser **Group Policy Analytics** dans le
    portail Intune (`Devices > Group Policy Analytics`) : il analyse les GPO existantes
    et indique quels paramètres ont un équivalent CSP direct.

!!! quote "En résumé"
    - Sur Hybrid Join, `gpsvc` continue d'appliquer les GPO normalement — l'enrôlement MDM ne les désactive pas
    - Le workload switching bascule l'autorité domaine par domaine — ce n'est pas tout ou rien
    - Ne basculez jamais deux workloads majeurs le même jour en production
    - Détectez les conflits de registre avant de basculer, pas après

---

## :material-file-import: ADMX-backed policies dans Intune

### Ce que sont les ADMX-backed policies

Dans Intune, les **Administrative Templates** natifs couvrent les paramètres Microsoft (Edge, Office, Windows). Pour les ADMX tierces — Chrome, Firefox, Zoom, applications métier — vous devez importer les fichiers ADMX manuellement.

La fonctionnalité s'appelle **Imported Administrative Templates** dans Intune. Elle est disponible depuis `Devices > Configuration profiles > Create > Windows 10 and later > Imported Administrative Templates`.

Une fois importée, une ADMX apparaît comme n'importe quel Administrative Template natif. Vous créez un profil, vous configurez les paramètres, vous l'assignez à des groupes.

### Importer une ADMX tierce dans Intune

Voici la procédure complète pour Chrome, applicable à n'importe quelle ADMX tierce.

**Étape 1 — Télécharger les fichiers ADMX**

```powershell title="Download-ChromeADMX.ps1"
# Download the latest Chrome ADMX bundle from Google
# Adapt the URL to the version you need — check https://chromeenterprise.google/browser/download/
$chromeAdmxUrl = "https://dl.google.com/dl/edgedl/chrome/policy/policy_templates.zip"
$destZip       = "C:\ADMX_Import\chrome_policy_templates.zip"
$destExtract   = "C:\ADMX_Import\chrome"

New-Item -ItemType Directory -Path "C:\ADMX_Import" -Force | Out-Null

Invoke-WebRequest -Uri $chromeAdmxUrl -OutFile $destZip
Expand-Archive -Path $destZip -DestinationPath $destExtract -Force

# ADMX files are in the windows\admx subfolder
$admxFiles = Get-ChildItem -Path "$destExtract\windows\admx" -Filter "*.admx"
Write-Host "Found ADMX files:"
$admxFiles | Select-Object Name
```

**Étape 2 — Importer dans Intune via le portail**

1. Ouvrez `Devices > Configuration profiles`
2. Cliquez sur **Import ADMX** (bouton en haut de la liste)
3. Uploadez le fichier `.admx` (ex : `chrome.admx`)
4. Uploadez le fichier `.adml` correspondant (ex : `chrome.adml` depuis le dossier `en-US`)
5. Attendez le statut **Available** (quelques secondes à quelques minutes)

**Étape 3 — Créer un profil basé sur l'ADMX importée**

1. `Devices > Configuration profiles > Create profile`
2. Platform : **Windows 10 and later**
3. Profile type : **Imported Administrative Templates**
4. Sélectionnez l'ADMX importée (ex : `Google Chrome`)
5. Configurez les paramètres comme dans GPMC

### Limitations connues des ADMX-backed policies Intune

| Limitation | Détail | Impact |
|---|---|---|
| **User scope partiel** | Certains paramètres ADMX sont marqués "User scope only" — Intune les déploie uniquement si l'appareil est configuré en User Enrollment | Les paramètres User ne s'appliquent pas en Device Enrollment standard |
| **Pas de filtrage WMI** | Les ADMX Intune ne supportent pas les filtres WMI (contrairement aux GPO) | Les conditions complexes doivent être gérées par groupes Azure AD |
| **Pas de GPP** | Les Group Policy Preferences (raccourcis, sources ODBC, etc.) n'ont pas d'équivalent ADMX Intune | Voir section "Paramètres sans équivalent MDM" |
| **Conflits de namespace** | Deux ADMX avec le même namespace provoquent une erreur d'import | Vérifier les namespaces ADMX avant import (`categories` dans le XML) |
| **Taille max** | Limite de 1 Mo par fichier ADMX | Les ADMX volumineuses (Office, certaines suites) doivent être divisées |

### Vérifier qu'une ADMX-backed policy Intune est appliquée

```powershell title="Verify-ADMXBackedPolicy.ps1"
# Verify that an ADMX-backed Intune policy has written the expected registry value
# Run locally on the target machine or via Invoke-Command

$checks = @(
    @{
        Description = "Chrome Homepage via Intune ADMX"
        Path        = "HKLM:\SOFTWARE\Policies\Google\Chrome"
        Value       = "HomepageLocation"
        Expected    = "https://intranet.corp.local"
    },
    @{
        Description = "Firefox default browser check disabled"
        Path        = "HKLM:\SOFTWARE\Policies\Mozilla\Firefox"
        Value       = "DontCheckDefaultBrowser"
        Expected    = 1
    }
)

foreach ($check in $checks) {
    $actual = Get-ItemProperty -Path $check.Path -Name $check.Value -ErrorAction SilentlyContinue
    $status = if ($actual -and ($actual.($check.Value) -eq $check.Expected)) { "OK" } else { "MISSING/WRONG" }
    Write-Host "[$status] $($check.Description) — Expected: $($check.Expected) / Got: $($actual.($check.Value))"
}
```

```powershell title="Résultat attendu"
[OK] Chrome Homepage via Intune ADMX — Expected: https://intranet.corp.local / Got: https://intranet.corp.local
[OK] Firefox default browser check disabled — Expected: 1 / Got: 1
```

!!! quote "En résumé"
    - Les ADMX tierces s'importent dans Intune via **Imported Administrative Templates**
    - Chrome, Firefox, Zoom, Office ADMX suivent tous la même procédure d'import
    - Les paramètres **User scope** ne s'appliquent pas en Device Enrollment standard
    - Pas de filtrage WMI dans Intune — remplacez les filtres par des groupes Azure AD dynamiques

---

## :material-clock-alert: Le piège de la fenêtre de conformité vide

### Ce qui se passe quand vous supprimez la GPO trop tôt

Voici le scénario exact qui se produit en production plusieurs fois par an :

1. L'admin crée le profil Intune, l'assigne au groupe. La politique s'applique sur les machines en ligne.
2. L'admin supprime la GPO. À ce moment, toutes les machines qui ont reçu la politique Intune sont conformes.
3. Deux heures plus tard, cinq laptops se reconnectent depuis l'extérieur. Ils n'ont pas encore reçu la politique Intune (cycle MDM toutes les 8 h maximum).
4. Ces cinq machines n'ont ni la GPO (supprimée), ni la politique Intune (pas encore reçue). Elles sont dans un **état non conforme non détecté**.

La fenêtre de conformité vide dure jusqu'à ce que chaque machine ait effectué au moins un cycle MDM complet depuis l'enrôlement du profil. En pratique : entre 20 minutes (machine en ligne constante) et 8 heures (machine nomade qui se connecte une fois par jour).

### La règle des deux semaines

Deux semaines représentent suffisamment de cycles pour que :

- Tous les appareils nomades aient eu au moins un cycle MDM
- Les éventuels incidents utilisateurs aient remonté au support
- Les machines en maintenance ou en stock aient été reconnectées

Ne pas confondre "la politique est verte dans Intune" (= assignée) avec "la politique est appliquée sur tous les appareils" (= vérifiée par inspection).

### Script de détection avant suppression de GPO

Ce script compare l'état du registre sur un échantillon de machines et confirme que la politique MDM a bien écrit les valeurs attendues avant que vous supprimiez la GPO.

```powershell title="Pre-GPODelete-Validation.ps1"
# Validate MDM policy application on N machines before deleting the source GPO
# Run this script 24h after enabling the Intune profile, then again after 2 weeks

param(
    [string[]] $Computers   = @("PC-001","PC-002","PC-003","PC-004","PC-005",
                                "PC-006","PC-007","PC-008","PC-009","PC-010"),
    [string]   $RegPath     = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate",
    [string]   $RegValue    = "WUServer",
    [string]   $ExpectedVal = "https://wsus.corp.local:8530",
    [string]   $GPOName     = "Config-WSUS-Postes"
)

$results = foreach ($computer in $Computers) {
    try {
        Invoke-Command -ComputerName $computer -ScriptBlock {
            param($path, $val, $expected, $gpoName)

            # Read current registry value
            $current = (Get-ItemProperty -Path $path -Name $val -ErrorAction SilentlyContinue).$val

            # Check MDM enrollment status
            $enrollments = Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Enrollments" -ErrorAction SilentlyContinue
            $mdmEnrolled = $enrollments.Count -gt 0

            # Check last MDM sync time
            $syncLog = Get-WinEvent -LogName "Microsoft-Windows-DeviceManagement-Enterprise-Diagnostics-Provider/Admin" `
                -MaxEvents 1 -ErrorAction SilentlyContinue
            $lastSync = if ($syncLog) { $syncLog.TimeCreated } else { "Unknown" }

            [PSCustomObject]@{
                Computer    = $env:COMPUTERNAME
                RegValue    = $current
                ValueMatch  = ($current -eq $expected)
                MDMEnrolled = $mdmEnrolled
                LastMDMSync = $lastSync
                ReadyForGPODelete = ($current -eq $expected) -and $mdmEnrolled
            }
        } -ArgumentList $RegPath, $RegValue, $ExpectedVal, $GPOName
    }
    catch {
        [PSCustomObject]@{
            Computer          = $computer
            RegValue          = "UNREACHABLE"
            ValueMatch        = $false
            MDMEnrolled       = $false
            LastMDMSync       = "N/A"
            ReadyForGPODelete = $false
        }
    }
}

$results | Format-Table -AutoSize

$notReady = $results | Where-Object { -not $_.ReadyForGPODelete }
if ($notReady.Count -gt 0) {
    Write-Warning "$($notReady.Count) machine(s) NOT ready. Do NOT delete GPO '$GPOName' yet."
    $notReady | Select-Object Computer, RegValue, MDMEnrolled, LastMDMSync | Format-Table -AutoSize
}
else {
    Write-Host "All $($results.Count) machines validated. Safe to proceed with GPO deletion." -ForegroundColor Green
}
```

```powershell title="Résultat attendu — Migration sûre"
Computer    RegValue                      ValueMatch MDMEnrolled LastMDMSync           ReadyForGPODelete
--------    --------                      ---------- ----------- -----------           -----------------
PC-001      https://wsus.corp.local:8530  True       True        04/06/2025 08:32:11   True
PC-002      https://wsus.corp.local:8530  True       True        04/06/2025 09:14:55   True
...
PC-010      https://wsus.corp.local:8530  True       True        04/06/2025 07:58:02   True

All 10 machines validated. Safe to proceed with GPO deletion.
```

```powershell title="Résultat attendu — Migration bloquée (ne pas supprimer la GPO)"
PC-007      UNREACHABLE                   False      False       N/A                   False
PC-009      https://old-wsus:8530         False      True        03/06/2025 22:11:40   False

WARNING: 2 machine(s) NOT ready. Do NOT delete GPO 'Config-WSUS-Postes' yet.
```

!!! danger "Production — Supprimer une GPO est irréversible sans backup"
    Avant toute suppression de GPO, exécutez `Backup-GPO -Name "NomGPO" -Path "C:\GPO_Backups\"`. La restauration via `Restore-GPO` depuis un backup prend moins de 2 minutes. La recréation manuelle d'une GPO complexe peut prendre des heures.

    Voir le [chapitre 05 — Backup et migration](05-backup-migration.md) pour la stratégie de backup GPO complète.

!!! quote "En résumé"
    - La fenêtre de conformité vide = temps entre suppression de la GPO et réception de la politique MDM par tous les appareils
    - Cette fenêtre peut atteindre 8 heures pour les appareils nomades
    - Deux semaines d'observation minimum avant toute suppression
    - Le script `Pre-GPODelete-Validation.ps1` doit retourner `ReadyForGPODelete = True` sur 100 % des machines échantillonnées avant d'agir

---

## :material-check-all: Vérification post-migration

### Valider l'état global d'une migration

Une fois la migration d'un workload terminée (GPO supprimée, politique Intune active), ces commandes confirment l'état final.

```powershell title="Post-Migration-Audit.ps1"
# Post-migration audit: verify Intune policy is active and no GPO remnants remain
# Run from a management machine with WinRM access to targets

param(
    [string[]] $Computers = @("PC-001","PC-002","PC-003"),
    [string]   $DeletedGPOName = "Config-WSUS-Postes"
)

foreach ($computer in $Computers) {
    Write-Host "`n=== $computer ===" -ForegroundColor Cyan
    try {
        Invoke-Command -ComputerName $computer -ScriptBlock {
            param($gpoName)

            # 1. Check GPO is no longer in applied policies
            $rsopApplied = & gpresult /scope computer /fo csv 2>$null |
                ConvertFrom-Csv |
                Where-Object { $_.'Applied GPOs' -like "*$gpoName*" }

            if ($rsopApplied) {
                Write-Warning "  GPO '$gpoName' still appears in gpresult output!"
            }
            else {
                Write-Host "  [OK] GPO not found in applied policies"
            }

            # 2. Check MDM enrollment
            $mdmEnrolled = (Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Enrollments" -ErrorAction SilentlyContinue).Count -gt 0
            Write-Host "  [$(if ($mdmEnrolled) { 'OK' } else { 'FAIL' })] MDM enrollment: $mdmEnrolled"

            # 3. Check last MDM sync
            $lastSync = (Get-WinEvent -LogName "Microsoft-Windows-DeviceManagement-Enterprise-Diagnostics-Provider/Admin" `
                -MaxEvents 1 -ErrorAction SilentlyContinue)?.TimeCreated
            Write-Host "  [INFO] Last MDM sync: $lastSync"

            # 4. Check compliance status via MDM bridge
            $complianceKey = "HKLM:\SOFTWARE\Microsoft\PolicyManager\current\device"
            $complianceExists = Test-Path $complianceKey
            Write-Host "  [$(if ($complianceExists) { 'OK' } else { 'WARN' })] MDM PolicyManager active: $complianceExists"

        } -ArgumentList $DeletedGPOName
    }
    catch {
        Write-Warning "  Cannot reach $computer"
    }
}
```

### Checklist de validation finale

| Vérification | Commande / Action | Résultat attendu |
|---|---|---|
| GPO absente du domaine | `Get-GPO -Name "NomGPO"` → doit retourner une erreur | `GpoWithNameNotFound` |
| GPO absente de `gpresult` | `gpresult /scope computer /fo csv` sur 5 machines | Nom de la GPO absent |
| Politique Intune = "Succeeded" | Portail Intune > Devices > [device] > Device configuration | Status: Succeeded |
| Valeur registre correcte | `Get-ItemProperty -Path ... -Name ...` | Valeur attendue présente |
| Aucun ticket support lié | Helpdesk — filtrer les tickets des 2 dernières semaines | 0 incident lié au workload migré |
| MDM enrolled | `dsregcmd /status` → `MDMEnrolled : YES` | YES sur toutes les machines |


!!! quote "En résumé"
    - Une fois la migration d'un workload terminée (GPO supprimée, politique Intune active), ces commandes confirment l'état final.
    - Validez toujours le résultat sur un poste ou un utilisateur réellement dans le périmètre avant d’élargir.
    - Conservez les commandes et résultats de contrôle comme preuve de conformité post-déploiement.
---

## Références croisées

- [Chapitre 18 — Azure AD Hybrid Join](18-azure-ad-hybrid.md) : prérequis et configuration du Hybrid Join, co-management initial
- [Chapitre 20 — SCCM/MECM](20-sccm-mecm.md) : co-management MECM/Intune et workload switching depuis MECM
- [Bible GPO — Chapitre 25 : Convergence GPO/MDM](../bible-gpo/25-mdm-convergence.md) : théorie de la convergence, architecture CSP, comparaison des modèles de gestion

!!! quote "En résumé"
    - À relire : Chapitre 18 — Azure AD Hybrid Join.
    - À relire : Chapitre 20 — SCCM/MECM.
    - À relire : Bible GPO — Chapitre 25 : Convergence GPO/MDM.
    - Ces renvois prolongent le chapitre avec des mécanismes complémentaires ou des cas d’usage voisins.
    - Gardez ces chapitres sous la main pour le diagnostic ou la conception d’une GPO liée à ce thème.
