---
description: Windows Update for Business via GPO — anneaux de deploiement, WUfB vs WSUS, reports, redemarrages
tags: [gpo-admins, windows-update, wufb, wsus, mise-a-jour, rings]
---

# Windows Update for Business via GPO

!!! abstract "Ce que vous allez apprendre"
    - Choisir entre WUfB et WSUS en fonction de votre architecture (on-prem, hybride, full-cloud) avec une matrice de decision claire
    - Configurer les parametres GPO cles de WUfB avec leurs chemins GPMC exacts et les cles de registre correspondantes
    - Deployer quatre anneaux de mise a jour (Pilot, Early Adopters, Production, Critical) avec des GPO distinctes et un filtrage par groupe de securite
    - Satisfaire l'exigence telemetrie pour que les rapports cloud WUfB soient fonctionnels
    - Implementer un script de surveillance qui alerte quand une machine est en retard de plus de 30 jours sur les mises a jour de qualite

!!! tip "Si vous ne retenez qu'une chose"
    Si la cle `HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\WUServer` est definie sur un poste, **WSUS prend le dessus et les parametres WUfB sont ignores** — meme si vous avez configure des reports dans la meme GPO. Une GPO WUfB mal nettoyee apres une migration WSUS est silencieuse et destructrice. Verifiez cette cle en premier quand WUfB ne semble pas fonctionner.

---

## Contexte de production

La gestion des mises a jour Windows est le sujet qui cristallise le plus de tensions entre les equipes securite et les equipes operationnelles. D'un cote, les equipes securite exigent des delais de correction de plus en plus courts sur les CVE critiques. De l'autre, les equipes ops craignent les regressions introduites par des Patch Tuesdays precipites.

WUfB (Windows Update for Business) repond a cette tension en fournissant un mecanisme natif de report par anneau, sans infrastructure supplementaire. Il est complementaire a WSUS mais ne le remplace pas dans tous les scenarios.

---

## WUfB vs WSUS : tableau comparatif et matrice de decision

### :material-scale-balance: Comparaison directe

| Critere | WSUS (on-prem) | WUfB (cloud) | Mode mixte |
|---------|---------------|-------------|-----------|
| Infrastructure requise | Serveur WSUS + SQL | Aucune | WSUS existant |
| Peering (DO/BITS) | Non natif (BranchCache) | Oui (Delivery Optimization) | Partiel |
| Reporting | WSUS Console, SCCM | Update Compliance (Log Analytics) | Depend du canal |
| Prerequis Azure | Aucun | Azure AD Join ou Hybrid Join | Hybrid Join |
| Granularite des reports | Par update individuelle | Par anneau | Par anneau |
| Support Windows 10/11 | Oui | Oui (10 1607+) | Oui |
| Support Server | Oui | Limite (pas de Feature Updates) | Oui |
| Controle bande passante LAN | BranchCache | Delivery Optimization | DO prioritaire |
| Approbation manuelle par update | Oui | Non | Oui (WSUS) |
| Couts operationnels | Eleves (maintenance WSUS) | Faibles | Moyens |

### :material-alert-circle: La regle de coexistence critique

Quand les deux systemes sont configures en meme temps sur un poste, **WSUS gagne systematiquement**.

La priorite est determinee par la presence de la cle :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate
Valeur : WUServer  (REG_SZ)
```

Si `WUServer` est definie (meme avec une valeur vide ou invalide), Windows Update ignore tous les parametres WUfB — `DeferFeatureUpdatesPeriodInDays`, `DeferQualityUpdatesPeriodInDays`, `BranchReadinessLevel` — sans aucun message d'erreur ou d'avertissement dans les logs.

Ce comportement est intentionnel : Microsoft considere que la presence d'un WSUS signifie que l'organisation veut un controle centralise complet.

!!! danger "Production"
    En migration de WSUS vers WUfB, la suppression de la cle `WUServer` doit etre explicite. Une GPO qui "ne configure pas" WUServer ne la supprime pas si elle a ete definie par une GPO precedente. Utilisez `Preference > Registry > Delete` ou une GPO qui ecrit `WUServer` vide et `UseWUServer = 0` pour desactiver WSUS proprement avant d'activer les parametres WUfB.

### :material-table-arrow-right: Matrice de decision : quoi choisir ?

| Scenario | Recommandation |
|----------|---------------|
| Domaine AD on-prem pur, pas d'Azure AD | WSUS seul |
| Hybrid Join + licences E3/E5, reporting cloud | WUfB + Update Compliance |
| WSUS existant, migration progressive vers WUfB | Mode mixte : WSUS pour approbation, WUfB pour reports |
| Serveurs Windows (2016+) | WSUS (WUfB ne gere pas les Feature Updates Server) |
| Postes Windows 10/11, aucun WSUS, besoin simple | WUfB seul |
| SCCM/MECM en place | SUP SCCM (surcouche WSUS) — WUfB deconseille |
| Intune co-management | Intune WUfB policies (hors perimetre GPO) |

!!! quote "En resume"
    - WSUS prend la priorite si `WUServer` est definie — sans exception et sans message d'erreur.
    - WUfB est ideal pour les environnements hybrides Azure AD avec reporting cloud via Update Compliance.
    - En environnement SCCM ou serveurs Windows, restez sur WSUS/SUP.
    - La matrice ci-dessus couvre 90% des cas ; le 10% restant necessite une analyse de co-management Intune (voir chapitre 18).

---

## Parametres GPO cles de WUfB

### :material-cog: Reference complete des parametres

Tous les parametres WUfB se trouvent sous le meme noeud GPMC :

`Configuration ordinateur > Stratégies > Modèles d'administration > Composants Windows > Windows Update > Windows Update for Business`

Le tableau ci-dessous recense les parametres operationnels essentiels.

| Parametre GPO | Chemin GPMC (relatif a Windows Update for Business) | Cle de registre | Valeurs admises |
|--------------|---------------------------------------------------|----------------|----------------|
| `DeferFeatureUpdatesPeriodInDays` | Selectionner le moment ou les mises a jour des fonctionnalites sont recues | `HKLM\...\WindowsUpdate\AU\DeferFeatureUpdatesPeriodInDays` (REG_DWORD) | 0 – 365 |
| `DeferQualityUpdatesPeriodInDays` | Selectionner le moment ou les mises a jour de qualite sont recues | `HKLM\...\WindowsUpdate\AU\DeferQualityUpdatesPeriodInDays` (REG_DWORD) | 0 – 30 |
| `PauseFeatureUpdates` | Suspendre les mises a jour des fonctionnalites | `HKLM\...\WindowsUpdate\PauseFeatureUpdatesStartTime` (REG_SZ) | Date ISO ou vide |
| `PauseQualityUpdates` | Suspendre les mises a jour de qualite | `HKLM\...\WindowsUpdate\PauseQualityUpdatesStartTime` (REG_SZ) | Date ISO ou vide |
| `BranchReadinessLevel` | Selectionner le canal de preparation des fonctionnalites | `HKLM\...\WindowsUpdate\BranchReadinessLevel` (REG_DWORD) | 2 (Beta), 8 (Release Preview), 16 (GA) |
| `ManagePreviewBuilds` | Gerer les builds de la version d'evaluation | `HKLM\...\WindowsUpdate\ManagePreviewBuilds` (REG_DWORD) | 0 (desactive), 1 (activer mises a jour preview), 2 (forcer preview) |

Le chemin de registre complet pour tous ces parametres est :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate
```

### :material-flag: BranchReadinessLevel : les trois canaux

| Valeur | Canal | Public cible |
|--------|-------|-------------|
| `2` | Beta Channel | Testeurs internes volontaires — builds instables |
| `8` | Release Preview Channel | Pilotes elargis — builds proches de la GA mais non finales |
| `16` | General Availability Channel (GA) | Production — **valeur par defaut recommandee** |

En production, tous les anneaux doivent etre sur `16` (GA). Les canaux `2` et `8` sont reserves aux environnements de test volontaires — les machines avec `BranchReadinessLevel = 2` recevront des builds Insider qui peuvent introduire des regressions majeures.

### :material-clock-alert: DeferFeatureUpdates et DeferQualityUpdates : limites absolues

`DeferFeatureUpdatesPeriodInDays` accepte 0 a 365 jours. Au-dela, la valeur est ignoree et le comportement revient a 0.

`DeferQualityUpdatesPeriodInDays` accepte 0 a 30 jours maximum. Une valeur de 31 est silencieusement tronquee a 30. Pour les anneaux qui ont besoin de plus de 30 jours sur les quality updates, la seule option est de passer par WSUS avec approbation manuelle.

!!! warning "A surveiller"
    `PauseFeatureUpdates` et `PauseQualityUpdates` fonctionnent en ecrivant une date de debut de pause. La pause dure automatiquement **35 jours** apres cette date, puis est levee par Windows — meme si la GPO est toujours configuree. Microsoft a concu ce comportement pour eviter des machines bloquees indefiniment. Si vous pausez pour une urgence operationnelle, planifiez un rappel a J+30 pour verifier l'etat.

!!! quote "En resume"
    - `DeferQualityUpdates` est limite a 30 jours — pas plus, meme si vous ecrivez 60.
    - `DeferFeatureUpdates` peut aller jusqu'a 365 jours — utile pour geler une version pendant la validation applicative.
    - `BranchReadinessLevel = 16` (GA) est la seule valeur acceptable en production.
    - Les pauses GPO ont une duree de vie de 35 jours max — Windows les leve automatiquement.

---

## Anneaux de deploiement : architecture a quatre niveaux

### :material-layers: La logique des anneaux

Un anneau de deploiement est un groupe de machines qui recoit les mises a jour avec un report specifique. L'objectif est simple : si une mise a jour casse quelque chose, le probleme est detecte sur le petit groupe pilote avant d'atteindre la production.

Chaque anneau est implementé via une GPO distincte, liee a une OU ou filtree par groupe de securite.

### :material-table: Architecture des quatre anneaux

| Anneau | Report qualite | Report fonctionnalites | Taille typique | Profil |
|--------|---------------|----------------------|---------------|--------|
| Pilot | 0 jour | 0 jour | 1-5% du parc | Equipe IT, testeurs volontaires |
| Early Adopters | 7 jours | 30 jours | 10-15% du parc | Utilisateurs mobiles, early adopters |
| Production | 21 jours | 90 jours | 70-80% du parc | Ensemble des postes standard |
| Critical | 35 jours | 180 jours | 5-10% du parc | Serveurs, postes de trading, salles de reunion |

!!! warning "A surveiller"
    30 jours est le maximum pour `DeferQualityUpdates`. Les anneaux Production (21j) et Critical (35j) semblent contradictoires : Critical est limite a 30j en pratique pour les quality updates. Pour forcer un report supplementaire sur Critical, combinez `DeferQualityUpdates = 30` avec une approbation manuelle WSUS ou un `PauseQualityUpdates` a renouveler manuellement.

### :material-powershell: Script de creation des quatre GPO WUfB

Ce script cree les quatre GPO avec les parametres corrects et les lie aux OU correspondantes. Adaptez les noms d'OU a votre annuaire.

```powershell title="Create-WUfBRings.ps1"
# Create four WUfB deployment ring GPOs with correct deferral settings
# Requires: RSAT GroupPolicy module, Domain Admin or GPO Creator Owner rights

#Requires -Modules GroupPolicy, ActiveDirectory

$domain    = (Get-ADDomain).DNSRoot
$domainDN  = (Get-ADDomain).DistinguishedName

# Ring definitions: Name, QualityDeferDays, FeatureDeferDays, BranchReadiness, OUPath
$rings = @(
    @{
        Name            = "WUfB-Ring-Pilot"
        QualityDefer    = 0
        FeatureDefer    = 0
        BranchReadiness = 16     # GA Channel
        OUPath          = "OU=Pilot,OU=Computers,$domainDN"
        Description     = "WUfB Ring 0 - Pilot: IT team and volunteers. No deferral."
    },
    @{
        Name            = "WUfB-Ring-EarlyAdopters"
        QualityDefer    = 7
        FeatureDefer    = 30
        BranchReadiness = 16
        OUPath          = "OU=EarlyAdopters,OU=Computers,$domainDN"
        Description     = "WUfB Ring 1 - Early Adopters: 7d quality / 30d feature deferral."
    },
    @{
        Name            = "WUfB-Ring-Production"
        QualityDefer    = 21
        FeatureDefer    = 90
        BranchReadiness = 16
        OUPath          = "OU=Production,OU=Computers,$domainDN"
        Description     = "WUfB Ring 2 - Production: 21d quality / 90d feature deferral."
    },
    @{
        Name            = "WUfB-Ring-Critical"
        QualityDefer    = 30     # Maximum enforced by Windows Update
        FeatureDefer    = 180
        BranchReadiness = 16
        OUPath          = "OU=Critical,OU=Computers,$domainDN"
        Description     = "WUfB Ring 3 - Critical: 30d quality (max) / 180d feature deferral."
    }
)

foreach ($ring in $rings) {
    Write-Host "Creating GPO: $($ring.Name)..."

    # Create GPO
    $gpo = New-GPO -Name $ring.Name -Comment $ring.Description
    $gpoId = $gpo.Id

    # --- Quality Updates deferral ---
    Set-GPRegistryValue -Guid $gpoId `
        -Key  "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
        -ValueName "DeferQualityUpdates" `
        -Type DWord -Value 1

    Set-GPRegistryValue -Guid $gpoId `
        -Key  "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
        -ValueName "DeferQualityUpdatesPeriodInDays" `
        -Type DWord -Value $ring.QualityDefer

    # --- Feature Updates deferral ---
    Set-GPRegistryValue -Guid $gpoId `
        -Key  "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
        -ValueName "DeferFeatureUpdates" `
        -Type DWord -Value 1

    Set-GPRegistryValue -Guid $gpoId `
        -Key  "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
        -ValueName "DeferFeatureUpdatesPeriodInDays" `
        -Type DWord -Value $ring.FeatureDefer

    # --- Branch readiness (GA Channel = 16) ---
    Set-GPRegistryValue -Guid $gpoId `
        -Key  "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
        -ValueName "BranchReadinessLevel" `
        -Type DWord -Value $ring.BranchReadiness

    # --- Explicitly disable WSUS pointer to avoid coexistence conflict ---
    Set-GPRegistryValue -Guid $gpoId `
        -Key  "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" `
        -ValueName "UseWUServer" `
        -Type DWord -Value 0

    # Link GPO to the target OU
    if (Get-ADOrganizationalUnit -Filter "DistinguishedName -eq '$($ring.OUPath)'" -ErrorAction SilentlyContinue) {
        New-GPLink -Guid $gpoId -Target $ring.OUPath -LinkEnabled Yes
        Write-Host "  Linked to $($ring.OUPath)" -ForegroundColor Green
    } else {
        Write-Warning "  OU not found: $($ring.OUPath) — GPO created but NOT linked."
    }
}

Write-Host ""
Write-Host "Done. Created $($rings.Count) WUfB ring GPOs." -ForegroundColor Cyan
Write-Host "Verify with: Get-GPO -All | Where-Object Name -like 'WUfB-Ring-*' | Select-Object DisplayName, GpoStatus"
```

```title="Resultat attendu"
Creating GPO: WUfB-Ring-Pilot...
  Linked to OU=Pilot,OU=Computers,DC=contoso,DC=local
Creating GPO: WUfB-Ring-EarlyAdopters...
  Linked to OU=EarlyAdopters,OU=Computers,DC=contoso,DC=local
Creating GPO: WUfB-Ring-Production...
  Linked to OU=Production,OU=Computers,DC=contoso,DC=local
Creating GPO: WUfB-Ring-Critical...
  Linked to OU=Critical,OU=Computers,DC=contoso,DC=local

Done. Created 4 WUfB ring GPOs.
```

### :material-shield-account: Alternative : filtrage par groupe de securite

Si votre structure d'OU ne correspond pas aux anneaux (cas frequent dans les domaines historiques), utilisez le filtrage de securite GPO plutot que le lien sur OU.

La GPO est liee au niveau du domaine ou d'une OU parente. Le filtrage s'effectue via les permissions Apply Group Policy.

```powershell title="Set-GPOSecurityFiltering.ps1"
# Replace default security filtering (Authenticated Users) with a specific group
# This approach is used when OU structure does not map cleanly to rings

param(
    [string]$GPOName,
    [string]$AllowedGroup    # e.g. "GRP-WUfB-EarlyAdopters"
)

# Remove the default "Authenticated Users" apply permission
Set-GPPermission -Name $GPOName `
    -TargetName "Authenticated Users" `
    -TargetType Group `
    -PermissionLevel GpoRead

# Grant the target group both Read and Apply
Set-GPPermission -Name $GPOName `
    -TargetName $AllowedGroup `
    -TargetType Group `
    -PermissionLevel GpoApply

Write-Host "GPO '$GPOName' will now apply only to members of '$AllowedGroup'"
```

!!! warning "A surveiller"
    Quand vous retirez "Authenticated Users" du filtrage, les ordinateurs hors du groupe cible ne peuvent plus **lire** la GPO du tout. C'est correct pour Apply, mais si vous utilisez la GPO comme source de settings Read-only pour d'autres outils (comme des scripts de compliance), ajoutez "Authenticated Users" en GpoRead separement, sans GpoApply.

!!! quote "En resume"
    - Quatre anneaux : Pilot (0j), Early Adopters (7j), Production (21j), Critical (30j max pour quality).
    - Privilegiez le lien sur OU si votre structure AD le permet — c'est plus simple a maintenir.
    - Le filtrage par groupe de securite est la solution quand la structure d'OU est heterogene.
    - `UseWUServer = 0` dans chaque GPO WUfB est obligatoire si WSUS a ete utilise auparavant sur le meme parc.

---

## Telemetrie : prerequis pour les rapports WUfB cloud

### :material-chart-bar: Pourquoi la telemetrie est obligatoire

Windows Update for Business est une technologie de pilotage de mises a jour. Ses reports et deferrals fonctionnent meme sans telemetrie.

Mais les **rapports cloud** (Update Compliance dans Log Analytics, puis Windows Update for Business Reports dans le portail Azure) dependent du flux de telemetrie Windows. Sans ce flux, le portail affiche vos machines comme "non reportantes".

### :material-cog: La cle AllowTelemetry

| Valeur | Niveau | Description | Suffisant pour WUfB Reports |
|--------|--------|-------------|---------------------------|
| `0` | Security (off) | Aucune donnee envoyee — Enterprise uniquement | Non |
| `1` | Basic / Required | Donnees de diagnostic minimales | **Oui — niveau minimum requis** |
| `2` | Enhanced | Donnees supplementaires (deprecie dans Win11) | Oui |
| `3` | Full / Optional | Toutes les donnees de diagnostic | Oui |

La valeur minimale pour que les rapports WUfB fonctionnent est `1` (Basic / Required Diagnostic Data).

Chemin de registre :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection
Valeur : AllowTelemetry  (REG_DWORD)
Valeur minimale : 1
```

Chemin GPMC :

`Configuration ordinateur > Stratégies > Modèles d'administration > Composants Windows > Collecte des données et builds Preview > Autoriser la télémétrie`

### :material-powershell: Configurer AllowTelemetry via GPO PowerShell

```powershell title="Set-WUfBTelemetry.ps1"
# Set AllowTelemetry to Required (1) on all WUfB ring GPOs
# This is the minimum level for WUfB cloud reporting to function

$wufbGPOs = Get-GPO -All | Where-Object { $_.DisplayName -like "WUfB-Ring-*" }

foreach ($gpo in $wufbGPOs) {
    Set-GPRegistryValue -Guid $gpo.Id `
        -Key       "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" `
        -ValueName "AllowTelemetry" `
        -Type      DWord `
        -Value     1

    Write-Host "AllowTelemetry = 1 set on: $($gpo.DisplayName)"
}
```

```title="Resultat attendu"
AllowTelemetry = 1 set on: WUfB-Ring-Pilot
AllowTelemetry = 1 set on: WUfB-Ring-EarlyAdopters
AllowTelemetry = 1 set on: WUfB-Ring-Production
AllowTelemetry = 1 set on: WUfB-Ring-Critical
```

### :material-magnify: Verifier la telemetrie sur un poste cible

```powershell title="Verify-Telemetry.ps1"
# Check AllowTelemetry registry value on a remote machine
param(
    [string]$ComputerName = $env:COMPUTERNAME
)

$result = Invoke-Command -ComputerName $ComputerName -ScriptBlock {
    $key = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection"
    $val = (Get-ItemProperty -Path $key -Name AllowTelemetry -ErrorAction SilentlyContinue).AllowTelemetry

    [PSCustomObject]@{
        ComputerName    = $env:COMPUTERNAME
        AllowTelemetry  = if ($null -eq $val) { "NOT SET (defaults to 3 on non-Enterprise)" } else { $val }
        SufficientForWUfB = ($null -eq $val -or $val -ge 1)
    }
}

$result | Format-Table -AutoSize
```

```title="Resultat attendu"
ComputerName   AllowTelemetry  SufficientForWUfB
------------   --------------  -----------------
WS-PILOT-01    1               True
```

!!! danger "Production"
    Les editions **Windows 10/11 Enterprise** et **Education** acceptent `AllowTelemetry = 0` (Security), ce qui desactive entierement la telemetrie. C'est une politique de securite valide pour certains secteurs (defense, sante). Mais si vous activez cette politique sur vos postes **et** que vous utilisez WUfB Reports, le portail Azure ne recevra aucune donnee et affichera toutes vos machines comme "Unknown". Les deux choix sont valides — mais pas simultanement.

!!! quote "En resume"
    - `AllowTelemetry >= 1` est le prerequis pour que WUfB Reports fonctionne dans Azure.
    - La configuration se fait via GPMC ou directement avec `Set-GPRegistryValue`.
    - Si votre politique de securite impose `AllowTelemetry = 0`, le reporting cloud WUfB est inutilisable — utilisez WSUS Reports ou SCCM a la place.

---

## Parametres de redemarrage : equilibre securite / experience utilisateur

### :material-restart: Les quatre parametres cles

Le plus grand point de friction entre equipes IT et utilisateurs est le redemarrage force. WUfB offre plusieurs parametres pour calibrer ce comportement.

| Parametre GPO | Cle de registre | Plage de valeurs | Effet |
|--------------|----------------|-----------------|-------|
| `ActiveHoursStart` / `ActiveHoursEnd` | `HKLM\...\WindowsUpdate\AU\ActiveHoursStart` et `ActiveHoursEnd` | 0-23 (heure) | Definit la fenetre ou Windows ne redemarrera pas automatiquement |
| `AutoRestartNotificationSchedule` | `HKLM\...\WindowsUpdate\AutoRestartNotificationSchedule` | 15, 30, 60, 120, 240 (minutes) | Frequence des notifications de redemarrage |
| `EngagedRestartDeadline` | `HKLM\...\WindowsUpdate\EngagedRestartDeadline` | 2-30 (jours) | Delai max avant que Windows force le redemarrage |
| `EngagedRestartSnoozeSchedule` | `HKLM\...\WindowsUpdate\EngagedRestartSnoozeSchedule` | 1-3 (jours) | Duree de report quand l'utilisateur clique "Rappeler" |

Chemin GPMC pour tous les parametres de redemarrage :

`Configuration ordinateur > Stratégies > Modèles d'administration > Composants Windows > Windows Update > Gestion de l'experience utilisateur`

### :material-clock-time-eight: Valeurs recommandees pour un environnement enterprise

Ces valeurs representent un equilibre valide pour la majorite des entreprises avec des journees de travail 8h-18h.

```powershell title="Set-WUfBRestartPolicy.ps1"
# Configure WUfB restart behavior for enterprise workstations
# Target: minimize disruption while ensuring patches are applied within 7 days

param(
    [string]$GPOName = "WUfB-Ring-Production"
)

$gpoId = (Get-GPO -Name $GPOName).Id

# Active hours: protect 7:00 AM to 9:00 PM (21:00) — 14-hour window
Set-GPRegistryValue -Guid $gpoId `
    -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" `
    -ValueName "ActiveHoursStart" -Type DWord -Value 7

Set-GPRegistryValue -Guid $gpoId `
    -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" `
    -ValueName "ActiveHoursEnd" -Type DWord -Value 21

# Let users extend active hours but cap at 18 hours max
Set-GPRegistryValue -Guid $gpoId `
    -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" `
    -ValueName "AutoRestartRequiredNotificationDismissal" -Type DWord -Value 1

# Notification frequency: remind every 60 minutes once deadline approaches
Set-GPRegistryValue -Guid $gpoId `
    -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
    -ValueName "AutoRestartNotificationSchedule" -Type DWord -Value 60

# Engaged restart: 7-day deadline, 1-day snooze
Set-GPRegistryValue -Guid $gpoId `
    -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
    -ValueName "EngagedRestartDeadline" -Type DWord -Value 7

Set-GPRegistryValue -Guid $gpoId `
    -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
    -ValueName "EngagedRestartSnoozeSchedule" -Type DWord -Value 1

Write-Host "Restart policy configured on GPO '$GPOName'"
Write-Host "  Active hours   : 07:00 - 21:00"
Write-Host "  Restart deadline : 7 days after patch availability"
Write-Host "  Snooze duration  : 1 day"
```

```title="Resultat attendu"
Restart policy configured on GPO 'WUfB-Ring-Production'
  Active hours   : 07:00 - 21:00
  Restart deadline : 7 days after patch availability
  Snooze duration  : 1 day
```

### :material-table: Recommandations par anneau

| Anneau | Deadline redemarrage | Snooze | Horaires actifs |
|--------|---------------------|--------|----------------|
| Pilot | 2 jours | Aucun (force) | Heures bureau standard |
| Early Adopters | 3 jours | 1 jour | 07:00-21:00 |
| Production | 7 jours | 1 jour | 07:00-21:00 |
| Critical | 14 jours | 3 jours | Fenetres maintenance planifiees |

!!! danger "Production"
    Les postes de l'anneau Critical (serveurs, postes de trading, salles de conference) ne doivent **jamais** redemarrer automatiquement pendant les heures de production. Configurez `EngagedRestartDeadline = 14` et couplez ca a une procedure de redemarrage planifie via SCCM, une tache planifiee ou une notification au responsable de perimetre. Un redemarrage non planifie d'un poste de trading ou d'un serveur de production pendant les heures de marche est un incident P1.

!!! quote "En resume"
    - La fenetre Active Hours protege les utilisateurs des redemarrages pendant le travail — maximum 18 heures de protection par jour.
    - `EngagedRestartDeadline` est le filet de securite : apres ce delai, Windows redemarre meme si l'utilisateur refuse.
    - Pour les serveurs et postes critiques, desactivez les redemarrages automatiques et gerez-les manuellement via SCCM ou tache planifiee.

---

## Piege de production : masquage de Windows Update + report = machines oubliees

### :material-bomb: Le scenario

La configuration suivante est frequente dans les environnements qui veulent une experience utilisateur propre :

1. `SetDisableUXWUAccess = 1` — masque le panneau Windows Update dans Parametres. Les utilisateurs ne voient plus les mises a jour en attente.
2. `DeferQualityUpdatesPeriodInDays = 30` — report de 30 jours sur les quality updates.

En theorie, c'est propre : les utilisateurs ne sont pas distraits par les notifications de mise a jour, et l'administrateur controle le calendrier.

En pratique, c'est un piege.

### :material-alert-decagram: Ce qui se passe reellement

Si l'administrateur oublie de forcer les mises a jour apres les 30 jours de report, les machines restent indefiniment en retard. Les utilisateurs ne voient rien (panneau masque). Aucune alerte n'est generee par defaut.

Resultat : des machines qui n'ont pas recu de patch depuis 45, 60, 90 jours — sans que personne ne le sache.

!!! danger "Production"
    `SetDisableUXWUAccess = 1` supprime la visibilite utilisateur **ET** admin si vous n'avez pas de monitoring alternatif. En production, ne deployez jamais ce parametre sans un script de surveillance qui alerte sur les machines en retard. La conformite aux CVE critiques depend de cette vigilance.

### :material-powershell: Script de surveillance : alerter sur les machines en retard

Ce script interroge les machines du domaine pour identifier celles dont la derniere installation de quality update date de plus de 30 jours.

```powershell title="Watch-UpdateCompliance.ps1"
# Alert on machines that have not installed quality updates in more than N days
# Schedule this script weekly via Task Scheduler on a management server

param(
    [int]$AlertThresholdDays  = 30,
    [string]$TargetOU         = "",        # OU DN, or empty for all domain computers
    [string]$AlertEmail       = "it-ops@contoso.local",
    [string]$SmtpServer       = "smtp.contoso.local",
    [string]$ReportPath       = "C:\Temp\WUfB-Compliance-$(Get-Date -Format 'yyyyMMdd').csv"
)

# Build the computer list
$queryParams = @{ Filter = "Enabled -eq `$true" }
if ($TargetOU) { $queryParams.SearchBase = $TargetOU }
$computers = Get-ADComputer @queryParams | Select-Object -ExpandProperty Name

Write-Host "Checking $($computers.Count) computers for update compliance..."

$results = $computers | ForEach-Object -ThrottleLimit 20 -Parallel {
    $computer = $_
    $threshold = $using:AlertThresholdDays

    try {
        $session = New-CimSession -ComputerName $computer -OperationTimeoutSec 10 -ErrorAction Stop

        # Get the last successful quality update installation date
        $wu = Get-CimInstance -CimSession $session `
            -ClassName Win32_QuickFixEngineering `
            -ErrorAction SilentlyContinue |
            Sort-Object InstalledOn -Descending |
            Select-Object -First 1

        $lastPatch = if ($wu -and $wu.InstalledOn) {
            [datetime]$wu.InstalledOn
        } else {
            $null
        }

        $daysSinceUpdate = if ($lastPatch) {
            (Get-Date) - $lastPatch | Select-Object -ExpandProperty Days
        } else { 999 }

        # Also check WUServer registry to detect WSUS override
        $wuServer = Invoke-Command -Session (New-PSSession -ComputerName $computer -ErrorAction SilentlyContinue) -ScriptBlock {
            (Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
                -Name WUServer -ErrorAction SilentlyContinue).WUServer
        } -ErrorAction SilentlyContinue

        Remove-CimSession $session -ErrorAction SilentlyContinue

        [PSCustomObject]@{
            ComputerName    = $computer
            LastPatchDate   = if ($lastPatch) { $lastPatch.ToString("yyyy-MM-dd") } else { "Unknown" }
            DaysSinceUpdate = $daysSinceUpdate
            Compliant       = $daysSinceUpdate -le $threshold
            WUSERVERDefined = (-not [string]::IsNullOrEmpty($wuServer))
            Status          = if ($daysSinceUpdate -le $threshold) { "OK" }
                              elseif ($daysSinceUpdate -le $threshold + 7) { "WARNING" }
                              else { "CRITICAL" }
        }
    } catch {
        [PSCustomObject]@{
            ComputerName    = $computer
            LastPatchDate   = "Unreachable"
            DaysSinceUpdate = -1
            Compliant       = $false
            WUSERVERDefined = $false
            Status          = "UNREACHABLE"
        }
    }
}

# Export full report to CSV
$results | Export-Csv $ReportPath -NoTypeInformation
Write-Host "Report saved to $ReportPath"

# Summary
$critical = $results | Where-Object Status -eq "CRITICAL"
$warnings  = $results | Where-Object Status -eq "WARNING"
$ok        = $results | Where-Object Status -eq "OK"

Write-Host ""
Write-Host "=== Update Compliance Summary ==="
Write-Host "  OK         : $($ok.Count)"
Write-Host "  WARNING    : $($warnings.Count) (between $AlertThresholdDays and $($AlertThresholdDays + 7) days)"
Write-Host "  CRITICAL   : $($critical.Count) (more than $($AlertThresholdDays + 7) days behind)"
Write-Host "  UNREACHABLE: $(($results | Where-Object Status -eq 'UNREACHABLE').Count)"

# Send alert email if critical machines exist
if ($critical.Count -gt 0) {
    $emailBody = @"
Windows Update Compliance Alert — $(Get-Date -Format 'yyyy-MM-dd')

The following machines have not installed quality updates in more than $($AlertThresholdDays + 7) days:

$(($critical | Select-Object ComputerName, LastPatchDate, DaysSinceUpdate, WUSERVERDefined |
    Format-Table -AutoSize | Out-String))

Full report: $ReportPath

Immediate action required:
1. Verify the machine is reachable and the WUfB GPO is applied (gpresult /r).
2. Check if WUServer registry key overrides WUfB settings (WUSERVERDefined = True).
3. Force update check: usoclient StartScan; usoclient StartInstall
"@

    Send-MailMessage `
        -To         $AlertEmail `
        -From       "wu-monitor@contoso.local" `
        -Subject    "[ALERTE WUfB] $($critical.Count) machine(s) en retard critique — $(Get-Date -Format 'yyyy-MM-dd')" `
        -Body       $emailBody `
        -SmtpServer $SmtpServer

    Write-Host ""
    Write-Warning "Alert email sent to $AlertEmail — $($critical.Count) critical machine(s) detected."
}
```

```title="Resultat attendu"
Checking 247 computers for update compliance...
Report saved to C:\Temp\WUfB-Compliance-20260405.csv

=== Update Compliance Summary ===
  OK         : 231
  WARNING    : 9 (between 30 and 37 days)
  CRITICAL   : 4 (more than 37 days behind)
  UNREACHABLE: 3

[WARNING] Alert email sent to it-ops@contoso.local — 4 critical machine(s) detected.
```

### :material-check-all: Verification post-alerte : forcer la mise a jour sur un poste

Quand le script identifie une machine en retard, voici la sequence de verification et de remediation.

```powershell title="Force-UpdateOnMachine.ps1"
# Diagnose and force Windows Update on a non-compliant machine
param(
    [Parameter(Mandatory)]
    [string]$ComputerName
)

Write-Host "=== WUfB Diagnostics: $ComputerName ===" -ForegroundColor Cyan

Invoke-Command -ComputerName $ComputerName -ScriptBlock {
    # 1. Check GPO result for Windows Update keys
    Write-Host "--- Applied Update Policy ---"
    $wuKey = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"
    $auKey = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU"

    $props = @("WUServer","DeferQualityUpdates","DeferQualityUpdatesPeriodInDays",
               "DeferFeatureUpdates","DeferFeatureUpdatesPeriodInDays","BranchReadinessLevel")

    foreach ($p in $props) {
        $val = (Get-ItemProperty $wuKey -Name $p -ErrorAction SilentlyContinue).$p
        Write-Host "  $p = $(if ($null -eq $val) { '(not set)' } else { $val })"
    }

    $useWU = (Get-ItemProperty $auKey -Name "UseWUServer" -ErrorAction SilentlyContinue).UseWUServer
    Write-Host "  UseWUServer = $(if ($null -eq $useWU) { '(not set)' } else { $useWU })"

    # 2. Force an update scan and install
    Write-Host ""
    Write-Host "--- Forcing update scan and install ---"
    usoclient StartScan
    Start-Sleep -Seconds 5
    usoclient StartInstall
    Write-Host "Update scan and install initiated. Check Windows Update log for progress."
}
```

```title="Resultat attendu"
=== WUfB Diagnostics: WS-FINANCE-07 ===
--- Applied Update Policy ---
  WUServer = (not set)
  DeferQualityUpdates = 1
  DeferQualityUpdatesPeriodInDays = 21
  DeferFeatureUpdates = 1
  DeferFeatureUpdatesPeriodInDays = 90
  BranchReadinessLevel = 16
  UseWUServer = 0

--- Forcing update scan and install ---
Update scan and install initiated. Check Windows Update log for progress.
```

!!! quote "En resume"
    - La combinaison `SetDisableUXWUAccess = 1` + `DeferQualityUpdates = 30` est un piege classique : les machines peuvent rester des semaines sans mise a jour sans que personne ne le remarque.
    - Le script `Watch-UpdateCompliance.ps1` planifie une verification hebdomadaire et envoie une alerte par email quand des machines sont en retard critique.
    - En cas d'alerte, `Force-UpdateOnMachine.ps1` diagnostique l'etat GPO et force le scan immediatement.
    - `WUSERVERDefined = True` dans le rapport signale une potentielle coexistence WSUS/WUfB — a investiguer en priorite.

---

## Verification post-deploiement

### :material-check-decagram: Valider l'application des GPO WUfB

Apres deployment, verifiez que les parametres sont corrects sur un echantillon de machines de chaque anneau.

```powershell title="Verify-WUfBRing.ps1"
# Verify WUfB settings are correctly applied on a set of target machines
param(
    [string[]]$ComputerNames,
    [string]$ExpectedRing     # "Pilot", "EarlyAdopters", "Production", "Critical"
)

$expectedSettings = @{
    Pilot        = @{ QualityDefer = 0;  FeatureDefer = 0   }
    EarlyAdopters = @{ QualityDefer = 7;  FeatureDefer = 30  }
    Production   = @{ QualityDefer = 21; FeatureDefer = 90  }
    Critical     = @{ QualityDefer = 30; FeatureDefer = 180 }
}

$expected = $expectedSettings[$ExpectedRing]

$results = $ComputerNames | ForEach-Object {
    $computer = $_

    $check = Invoke-Command -ComputerName $computer -ScriptBlock {
        $wuKey = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"
        $auKey = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU"

        [PSCustomObject]@{
            QualityDefer    = (Get-ItemProperty $wuKey -Name DeferQualityUpdatesPeriodInDays -ErrorAction SilentlyContinue).DeferQualityUpdatesPeriodInDays
            FeatureDefer    = (Get-ItemProperty $wuKey -Name DeferFeatureUpdatesPeriodInDays -ErrorAction SilentlyContinue).DeferFeatureUpdatesPeriodInDays
            BranchReadiness = (Get-ItemProperty $wuKey -Name BranchReadinessLevel -ErrorAction SilentlyContinue).BranchReadinessLevel
            UseWUServer     = (Get-ItemProperty $auKey -Name UseWUServer -ErrorAction SilentlyContinue).UseWUServer
            Telemetry       = (Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" -Name AllowTelemetry -ErrorAction SilentlyContinue).AllowTelemetry
        }
    } -ErrorAction SilentlyContinue

    [PSCustomObject]@{
        ComputerName       = $computer
        QualityDefer_OK    = ($check.QualityDefer -eq $expected.QualityDefer)
        FeatureDefer_OK    = ($check.FeatureDefer -eq $expected.FeatureDefer)
        BranchReadiness_OK = ($check.BranchReadiness -eq 16)
        WSUSDisabled_OK    = ($check.UseWUServer -eq 0)
        Telemetry_OK       = ($null -ne $check.Telemetry -and $check.Telemetry -ge 1)
        AllOK              = (
            $check.QualityDefer  -eq $expected.QualityDefer -and
            $check.FeatureDefer  -eq $expected.FeatureDefer -and
            $check.BranchReadiness -eq 16 -and
            $check.UseWUServer   -eq 0 -and
            $check.Telemetry     -ge 1
        )
    }
}

$results | Format-Table -AutoSize

$failCount = ($results | Where-Object AllOK -eq $false).Count
if ($failCount -eq 0) {
    Write-Host "All $($ComputerNames.Count) machines are correctly configured for ring '$ExpectedRing'." -ForegroundColor Green
} else {
    Write-Warning "$failCount machine(s) have incorrect settings. Review the table above."
}
```

```title="Resultat attendu (anneau Production)"
ComputerName    QualityDefer_OK  FeatureDefer_OK  BranchReadiness_OK  WSUSDisabled_OK  Telemetry_OK  AllOK
------------    ---------------  ---------------  ------------------  ---------------  ------------  -----
WS-PROD-001     True             True             True                True             True          True
WS-PROD-002     True             True             True                True             True          True
WS-PROD-003     False            True             True                True             True          False

WARNING: 1 machine(s) have incorrect settings. Review the table above.
```

### :material-google-circles-group: Verifier l'appartenance aux anneaux

```powershell title="Get-MachineRing.ps1"
# Report which WUfB ring each machine belongs to based on its GPO application
param(
    [string[]]$ComputerNames
)

$ComputerNames | ForEach-Object {
    $computer = $_
    $info = Invoke-Command -ComputerName $computer -ScriptBlock {
        $wuKey = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"

        $quality = (Get-ItemProperty $wuKey -Name DeferQualityUpdatesPeriodInDays -ErrorAction SilentlyContinue).DeferQualityUpdatesPeriodInDays
        $feature = (Get-ItemProperty $wuKey -Name DeferFeatureUpdatesPeriodInDays -ErrorAction SilentlyContinue).DeferFeatureUpdatesPeriodInDays

        $ring = switch ($quality) {
            0       { "Pilot" }
            7       { "EarlyAdopters" }
            21      { "Production" }
            30      { "Critical" }
            default { "Unknown ($quality days)" }
        }

        [PSCustomObject]@{
            Ring         = $ring
            QualityDefer = $quality
            FeatureDefer = $feature
        }
    } -ErrorAction SilentlyContinue

    [PSCustomObject]@{
        ComputerName = $computer
        Ring         = if ($info) { $info.Ring } else { "Unreachable" }
        QualityDefer = if ($info) { $info.QualityDefer } else { "-" }
        FeatureDefer = if ($info) { $info.FeatureDefer } else { "-" }
    }
} | Format-Table -AutoSize
```

```title="Resultat attendu"
ComputerName   Ring          QualityDefer  FeatureDefer
------------   ----          ------------  ------------
WS-PILOT-01    Pilot         0             0
WS-EARLY-05    EarlyAdopters 7             30
WS-PROD-012    Production    21            90
WS-CRIT-03     Critical      30            180
```

!!! quote "En resume"
    - Verifiez systematiquement cinq cles apres deploiement : `DeferQualityUpdatesPeriodInDays`, `DeferFeatureUpdatesPeriodInDays`, `BranchReadinessLevel`, `UseWUServer`, et `AllowTelemetry`.
    - `Get-MachineRing.ps1` vous donne une vue immediate de quel anneau chaque machine applique reellement — utile pour detecter les machines qui n'ont pas recu la bonne GPO.
    - Un `AllOK = False` sur plusieurs machines du meme anneau signale generalement un probleme de lien GPO ou de filtrage de securite — verifiez avec `gpresult /h`.

---

## Cross-references

| Sujet | Reference |
|-------|-----------|
| Introduction WSUS et WUfB pour debutants | [La Bible GPO pour les Nuls — Ch. 09 Windows Update](../gpo-pour-les-nuls/09-windows-update.md) |
| WUfB avec Intune co-management et Azure AD Hybrid Join | [Ch. 18 — Azure AD Hybrid](18-azure-ad-hybrid.md) |
| Automatisation GPO et pipelines CI/CD | Ch. 23 — Automatisation CI/CD |
| Sauvegarde des GPO WUfB avant migration | Ch. 05 — Sauvegarde et migration |
| Audit de conformite des parametres WUfB | Ch. 06 — Audit et conformite |
