---
description: "Creer, maintenir et securiser le Central Store ADMX dans SYSVOL : guide complet pour les administrateurs d'entreprise, avec scripts PowerShell de production et gestion des modeles tiers."
tags:
  - gpo
  - administration
  - admx
  - central-store
---
# Central Store et gestion des ADMX

!!! abstract "Ce que vous allez pouvoir faire"
    - Creer le Central Store sur l'emulateur PDC en moins de cinq minutes avec un script PowerShell pret a l'emploi
    - Mettre a jour les modeles ADMX Windows sans ecraser les fichiers inchanges, grace a une comparaison de hash
    - Integrer les ADMX tiers (Edge, Chrome, Firefox, Office 365, OneDrive) dans le Central Store sans collision de namespace
    - Versionner le Central Store avec Git et deployer les changements via `robocopy` depuis un depot centralise
    - Detecter les GPO cassees apres une mise a jour ADMX et valider l'integrite du Central Store avant un gel de changement

!!! tip "Si vous ne retenez qu'une chose"
    Le Central Store est un dossier dans SYSVOL. Quand il existe, GPMC l'utilise a la place du magasin local. Pour le creer : copier `C:\Windows\PolicyDefinitions\` vers `\\domaine\SYSVOL\domaine\Policies\PolicyDefinitions\`.

---

## :material-folder-network-outline: Pourquoi le Central Store

### Le probleme sans Central Store

Sans Central Store, chaque machine administrateur utilise son propre `%SystemRoot%\PolicyDefinitions\`. Si deux administrateurs ont des versions de Windows differentes — ou des paquets ADMX differents installes — ils ne voient pas les memes parametres dans GPMC.

L'un voit "Configurer la mise en veille apres inactivite". L'autre voit "Extra Registry Settings" au meme endroit. Personne ne comprend ce qui s'est passe.

:material-alert-outline: Ce n'est pas un bug GPMC. C'est la consequence logique d'un referentiel ADMX non centralise. Chaque modification devient une source potentielle d'erreur silencieuse.

### Ce que le Central Store resout

Le Central Store est une source unique d'autorite pour tous les fichiers ADMX du domaine. Il est stocke dans SYSVOL — donc replique automatiquement sur tous les controleurs de domaine.

Quand GPMC ouvre une GPO, il cherche d'abord le Central Store. S'il existe, il l'utilise. Sinon, il retombe sur le magasin local de la machine.

:material-check-circle-outline: Tous les administrateurs voient exactement les memes parametres, depuis n'importe quelle machine jointe au domaine.

### Comment detecter l'absence de Central Store

Ouvrez GPMC, editez n'importe quelle GPO, et regardez le bas de la fenetre de l'editeur de strategie de groupe. Si le Central Store est absent, vous verrez :

```
Policy definitions (ADMX files) retrieved from the local computer.
```

Une fois le Central Store cree et GPMC relance, ce message disparait. Son absence confirme que le Central Store est actif.

!!! quote "En resume"
    Sans Central Store, chaque admin travaille avec ses propres ADMX locaux — c'est une source de divergence et d'erreurs. Le Central Store est un dossier SYSVOL unique qui sert de reference pour tous. Son activation est verifiable en un coup d'oeil dans GPMC.

---

## :material-folder-plus-outline: Creer le Central Store

### Sur quel serveur executer le script

Executez toujours ce script sur l'**emulateur PDC**. Toutes les ecritures GPMC passent par l'emulateur PDC — c'est lui qui a la version la plus a jour des outils Windows et, donc, les ADMX les plus recents.

```powershell
# Identify the PDC Emulator for the current domain
(Get-ADDomain).PDCEmulator
```

Connectez-vous directement sur ce serveur (ou via PSRemoting) pour la creation initiale.

### Script de creation

```powershell
# Create the Central Store from the PDC Emulator's local PolicyDefinitions
$domain = (Get-ADDomain).DNSRoot
$centralStore = "\\$domain\SYSVOL\$domain\Policies\PolicyDefinitions"
$localStore   = "$env:SystemRoot\PolicyDefinitions"

# Create the Central Store root directory
New-Item -ItemType Directory -Path $centralStore -Force | Out-Null
Write-Host "Central Store directory created." -ForegroundColor Cyan

# Copy all ADMX files
$admxFiles = Get-ChildItem $localStore -Filter "*.admx"
Copy-Item $admxFiles.FullName -Destination $centralStore -Force
Write-Host "Copied $($admxFiles.Count) ADMX files."

# Copy all language folders with their ADML files
Get-ChildItem $localStore -Directory | ForEach-Object {
    $langFolder  = $_
    $destFolder  = Join-Path $centralStore $langFolder.Name
    New-Item -ItemType Directory -Path $destFolder -Force | Out-Null

    $admlFiles = Get-ChildItem $langFolder.FullName -Filter "*.adml"
    Copy-Item $admlFiles.FullName -Destination $destFolder -Force
    Write-Host "Copied $($admlFiles.Count) ADML files for $($langFolder.Name)."
}

Write-Host "`nCentral Store created at: $centralStore" -ForegroundColor Green
```

```title="Resultat attendu"
Central Store directory created.
Copied 421 ADMX files.
Copied 421 ADML files for en-US.
Copied 421 ADML files for fr-FR.

Central Store created at: \\contoso.local\SYSVOL\contoso.local\Policies\PolicyDefinitions
```

### Verifier le resultat dans GPMC

Fermez et relancez GPMC. Ouvrez n'importe quelle GPO existante dans l'editeur. Le message "retrieved from the local computer" ne doit plus apparaitre.

:material-eye-check-outline: Si le message persiste, verifiez que le dossier `PolicyDefinitions` est bien cree directement sous `\\domaine\SYSVOL\domaine\Policies\` — pas dans un sous-dossier supplementaire.

!!! warning "Chemin exact — pas de sous-dossier supplementaire"
    Le chemin doit etre **exactement** `\\domaine\SYSVOL\domaine\Policies\PolicyDefinitions\`. Un dossier `PolicyDefinitions\PolicyDefinitions\` ne sera pas reconnu par GPMC. C'est l'erreur la plus frequente a la creation.

!!! quote "En resume"
    Executez le script sur l'emulateur PDC. Le Central Store se cree en moins de deux minutes. Validez immediatement dans GPMC : l'absence du message "local computer" confirme le succes. Le chemin exact est la seule chose qui compte — toute erreur de profondeur rend le Central Store invisible.

---

## :material-refresh: Mettre a jour le Central Store

### Quand mettre a jour

Microsoft publie de nouveaux modeles ADMX a chaque version majeure de Windows (23H2, 24H2, etc.) et pour certaines mises a jour fonctionnelles. Les applications tierces font de meme.

Ne mettez pas a jour le Central Store "en passant". Planifiez la mise a jour, testez en laboratoire, et deployez en dehors d'un gel de changement.

:material-download-outline: Telechargez les nouveaux modeles depuis le [Microsoft Download Center](https://www.microsoft.com/en-us/download/details.aspx?id=104677). Extrayez les fichiers dans un dossier temporaire — par exemple `C:\Temp\PolicyDefinitions`.

### Script de mise a jour avec comparaison de hash

Ce script ne copie que les fichiers modifies. Il evite d'ecraser des fichiers identiques et produit un rapport clair de ce qui a change.

```powershell
# Update Central Store with new ADMX templates (hash-based, only copies changed files)
param(
    [string]$NewAdmxSource = "C:\Temp\PolicyDefinitions",
    [string]$Domain        = (Get-ADDomain).DNSRoot
)

$centralStore = "\\$Domain\SYSVOL\$Domain\Policies\PolicyDefinitions"
$updated = 0
$added   = 0
$skipped = 0

# Process ADMX files at root level
Get-ChildItem $NewAdmxSource -Filter "*.admx" | ForEach-Object {
    $dest = Join-Path $centralStore $_.Name
    if (Test-Path $dest) {
        $srcHash = (Get-FileHash $_.FullName -Algorithm SHA256).Hash
        $dstHash = (Get-FileHash $dest      -Algorithm SHA256).Hash
        if ($srcHash -ne $dstHash) {
            Copy-Item $_.FullName $dest -Force
            Write-Host "  UPDATED : $($_.Name)"
            $updated++
        } else {
            $skipped++
        }
    } else {
        Copy-Item $_.FullName $dest -Force
        Write-Host "  ADDED   : $($_.Name)"
        $added++
    }
}

# Process language subfolders and their ADML files
Get-ChildItem $NewAdmxSource -Directory | ForEach-Object {
    $langFolder = $_
    $destLang   = Join-Path $centralStore $langFolder.Name
    New-Item -ItemType Directory -Path $destLang -Force | Out-Null

    Get-ChildItem $langFolder.FullName -Filter "*.adml" | ForEach-Object {
        $destAdml = Join-Path $destLang $_.Name
        Copy-Item $_.FullName $destAdml -Force
    }
    Write-Host "  LANG    : $($langFolder.Name) processed."
}

Write-Host "`nSummary — Updated: $updated | Added: $added | Skipped (identical): $skipped" -ForegroundColor Green
```

```title="Resultat attendu"
  UPDATED : Windows.admx
  UPDATED : WindowsUpdate.admx
  ADDED   : WindowsCopilot.admx
  LANG    : en-US processed.
  LANG    : fr-FR processed.

Summary — Updated: 2 | Added: 1 | Skipped (identical): 418
```

### Valider apres mise a jour

Apres chaque mise a jour du Central Store, verifiez que vos GPO existantes n'ont pas ete affectees.

```powershell
# Scan all GPOs for Extra Registry Settings (indicates missing ADMX after update)
Get-GPO -All | ForEach-Object {
    $gpo    = $_
    $report = Get-GPOReport -Guid $gpo.Id -ReportType Xml
    if ($report -match "ExtraRegistrySettings") {
        Write-Warning "GPO '$($gpo.DisplayName)' has Extra Registry Settings — possible missing ADMX."
    }
}
Write-Host "Scan complete." -ForegroundColor Green
```

```title="Resultat attendu (aucun probleme)"
Scan complete.
```

:material-alert-circle-outline: Si des avertissements apparaissent, notez les noms des GPO concernees. Ouvrez-les dans GPMC et identifiez les parametres marques "Extra Registry Settings". Cela signifie que l'ADMX correspondant n'est plus dans le Central Store.

!!! danger "Ne jamais mettre a jour pendant un gel de changement"
    Ajouter des ADMX pendant un gel rend immediatement visibles de nouveaux parametres dans GPMC. Un administrateur peut, sans s'en rendre compte, configurer un parametre non valide par le Change Advisory Board. Reservez les mises a jour ADMX a des fenetres de maintenance planifiees.

!!! quote "En resume"
    La mise a jour utilise une comparaison de hash SHA-256 pour ne copier que ce qui a change. Apres chaque mise a jour, lancez le scan des "Extra Registry Settings" pour detecter les GPO potentiellement affectees. Ne jamais mettre a jour pendant un gel de changement.

---

## :material-package-variant-plus: Gestion des ADMX tiers

### Principe general

Les ADMX tiers s'ajoutent au Central Store comme les ADMX Windows : fichiers `.admx` a la racine, fichiers `.adml` dans les sous-dossiers de langue.

La seule complication est la gestion des **dependances de namespace**. Certains ADMX referencent un namespace declare dans un autre ADMX. Si la dependance est absente, GPMC ignore le fichier sans avertissement.

:material-information-outline: Toujours verifier les dependances avant d'ajouter un ADMX tiers. Le tableau ci-dessous recense les cas les plus courants.

### Microsoft Edge

Telechargez les modeles depuis [Microsoft Edge for Business](https://www.microsoft.com/en-us/edge/business/download). Extrayez le package et copiez les fichiers vers le Central Store.

Fichiers principaux : `msedge.admx` et `msedgeupdate.admx`.

```powershell
# Verify Edge ADMX files are present in the Central Store
$centralStore = "\\$env:USERDNSDOMAIN\SYSVOL\$env:USERDNSDOMAIN\Policies\PolicyDefinitions"

$edgeFiles = @("msedge.admx", "msedgeupdate.admx")
$edgeFiles | ForEach-Object {
    $path   = Join-Path $centralStore $_
    $exists = Test-Path $path
    $status = if ($exists) { "OK" } else { "MISSING" }
    Write-Host "[$status] $_"
}
```

```title="Resultat attendu"
[OK] msedge.admx
[OK] msedgeupdate.admx
```

### Google Chrome

Telechargez le [Chrome Enterprise Bundle](https://chromeenterprise.google/intl/fr_fr/browser/download/). Extrayez et localisez le sous-dossier `Configuration\admx\`.

:material-alert-outline: Chrome a une dependance critique : `chrome.admx` requiert `google.admx` pour declarer le namespace parent `Google`. Si `google.admx` est absent, toutes les politiques Chrome disparaissent de GPMC sans erreur visible.

Fichiers requis : `google.admx`, `chrome.admx`, `GoogleUpdate.admx` (pour les mises a jour Chrome).

### Mozilla Firefox

Telechargez depuis le depot [mozilla/policy-templates](https://github.com/mozilla/policy-templates) sur GitHub. Les releases incluent un dossier `windows/` avec les fichiers ADMX.

Fichier principal : `firefox.admx`. Aucune dependance externe.

### Microsoft 365 Apps (Office)

Telechargez depuis le Microsoft Download Center — cherchez "Administrative Template files (ADMX/ADML) for Microsoft 365 Apps".

Le paquet est volumineux : `office16.admx` plus une quinzaine de fichiers supplementaires (`excel16.admx`, `word16.admx`, `outlook16.admx`, `teams16.admx`, etc.).

:material-information-outline: OneDrive a son propre paquet ADMX separe de celui d'Office. Telechargez-le independamment depuis le [portail Microsoft](https://learn.microsoft.com/en-us/sharepoint/use-group-policy).

### Tableau recapitulatif

| Application | Fichiers principaux | Namespace | Dependance requise |
|-------------|---------------------|-----------|-------------------|
| Edge | `msedge.admx` | `microsoft.policies.edge` | Aucune |
| Chrome | `chrome.admx` | `Google.Chrome` | `google.admx` |
| Firefox | `firefox.admx` | `Mozilla.Firefox` | Aucune |
| Office 365 | `office16.admx` + 15 autres | `office16` | Aucune |
| OneDrive | `OneDrive.admx` | `Microsoft.OneDrive` | Aucune |
| GoogleUpdate | `GoogleUpdate.admx` | `Google.Update` | `google.admx` |

!!! warning "Toujours inclure les fichiers ADML pour chaque langue"
    Un ADMX sans son ADML correspondant dans le dossier de langue de votre GPMC provoque des descriptions vides ou des erreurs a l'ouverture de la GPO. Si vous administrez en francais, verifiez que le dossier `fr-FR\` du Central Store contient bien les ADML de chaque ADMX tiers ajoute.

!!! quote "En resume"
    Chaque ADMX tiers suit le meme schema : fichier ADMX a la racine, ADML dans les sous-dossiers de langue. Les dependances de namespace (notamment `google.admx` pour Chrome) sont le point de defaillance le plus frequent. Utilisez le tableau ci-dessus comme checklist avant chaque ajout.

---

## :material-git: Versionner les ADMX avec Git

### Pourquoi versionner

Le Central Store est dans SYSVOL — c'est un dossier comme un autre. Mais SYSVOL n'est pas un systeme de versionnement. Personne ne sait qui a ajoute `chrome.admx` le 14 mars, ni pourquoi `office16.admx` a ete remplace sans mise a jour du CHANGELOG.

Versionner les ADMX avec Git donne une trace complete : qui, quoi, quand, et pourquoi.

:material-source-branch: Le depot Git devient la source d'autorite. Le Central Store n'est que la destination de deploiement.

### Structure recommandee du depot

```
admx-repo/
├── PolicyDefinitions/
│   ├── *.admx                  (Windows + third-party ADMX files)
│   ├── en-US/
│   │   └── *.adml
│   └── fr-FR/
│       └── *.adml
├── custom/
│   └── MonApp.admx             (in-house custom ADMX files)
├── CHANGELOG.md
└── deploy.ps1
```

Le dossier `custom/` contient les ADMX developpes en interne — par exemple pour gerer des parametres d'une application maison via le registre.

### Script de deploiement

Ce script deploie le contenu du depot vers le Central Store via `robocopy`. L'option `/MIR` synchronise : les fichiers supprimes du depot sont supprimes du Central Store.

```powershell
# deploy.ps1 — Synchronize ADMX repository to the Central Store
param(
    [string]$Domain = (Get-ADDomain).DNSRoot
)

$source  = Join-Path $PSScriptRoot "PolicyDefinitions"
$dest    = "\\$Domain\SYSVOL\$Domain\Policies\PolicyDefinitions"
$logFile = "C:\Temp\admx-deploy-$(Get-Date -Format 'yyyyMMdd-HHmm').log"

Write-Host "Deploying ADMX from: $source"
Write-Host "Destination        : $dest"
Write-Host "Log file           : $logFile"
Write-Host ""

# /MIR mirrors source to destination (adds, updates, deletes)
# /R:3  retries up to 3 times on failure
# /W:5  waits 5 seconds between retries
robocopy $source $dest /MIR /R:3 /W:5 /LOG:"$logFile" /TEE

if ($LASTEXITCODE -le 1) {
    Write-Host "`nDeployment complete. Exit code: $LASTEXITCODE (success)." -ForegroundColor Green
} else {
    Write-Host "`nDeployment finished with warnings or errors. Exit code: $LASTEXITCODE." -ForegroundColor Yellow
    Write-Host "Review the log file: $logFile"
}
```

```title="Resultat attendu"
Deploying ADMX from: C:\repos\admx-repo\PolicyDefinitions
Destination        : \\contoso.local\SYSVOL\contoso.local\Policies\PolicyDefinitions
Log file           : C:\Temp\admx-deploy-20250405-1430.log

-------------------------------------------------------------------------------
   ROBOCOPY     ::     Robust File Copy for Windows
-------------------------------------------------------------------------------
  Started : Saturday, April 05, 2025 2:30:00 PM
   Source : C:\repos\admx-repo\PolicyDefinitions\
     Dest : \\contoso.local\SYSVOL\contoso.local\Policies\PolicyDefinitions\

Deployment complete. Exit code: 1 (success).
```

:material-information-outline: Les codes de retour `robocopy` sont non conventionnels. `0` = aucun fichier copie. `1` = fichiers copies avec succes. `2` a `7` = copies avec avertissements. `8` et plus = erreurs reelles. Un code de `1` est un succes normal.

### Workflow Git recommande

Chaque modification du Central Store passe par une Pull Request ou une revue de code :

1. Branche de feature : `feat/add-chrome-126-admx`
2. Ajout des fichiers dans `PolicyDefinitions/`
3. Mise a jour de `CHANGELOG.md`
4. Revue et approbation
5. Merge sur `main`
6. Execution de `deploy.ps1` depuis le CI ou manuellement

:material-shield-check-outline: Ce workflow garantit une trace d'audit complete et permet de revenir en arriere en cas de probleme — `git revert` + `deploy.ps1`.

!!! quote "En resume"
    Git + `robocopy` transforme le Central Store en infrastructure versionnee. Le depot est la source de verite, SYSVOL est le resultat du deploiement. Chaque changement passe par une revue avant d'atterrir en production. Le retour en arriere est trivial.

---

## :material-stethoscope: Validation post-deploiement

### Verifier l'integrite des fichiers

Apres chaque deploiement, verifiez que les fichiers ADMX du Central Store sont syntaxiquement valides et lisibles.

```powershell
# Parse all ADMX files in Central Store and report any XML errors
$centralStore = "\\$env:USERDNSDOMAIN\SYSVOL\$env:USERDNSDOMAIN\Policies\PolicyDefinitions"
$errors  = 0
$checked = 0

Get-ChildItem $centralStore -Filter "*.admx" | ForEach-Object {
    $checked++
    try {
        $null = [xml](Get-Content $_.FullName -Encoding UTF8 -Raw)
    } catch {
        Write-Warning "XML parse error in '$($_.Name)': $_"
        $errors++
    }
}

if ($errors -eq 0) {
    Write-Host "All $checked ADMX files parsed successfully." -ForegroundColor Green
} else {
    Write-Host "$errors error(s) found in $checked ADMX files." -ForegroundColor Red
}
```

```title="Resultat attendu"
All 438 ADMX files parsed successfully.
```

### Verifier les GPO apres mise a jour

Apres une mise a jour du Central Store, scannez toutes les GPO du domaine pour detecter les parametres devenus orphelins.

```powershell
# Detect GPOs with Extra Registry Settings after ADMX update
$affected = @()

Get-GPO -All | ForEach-Object {
    $gpo    = $_
    $report = Get-GPOReport -Guid $gpo.Id -ReportType Xml
    if ($report -match "ExtraRegistrySettings") {
        $affected += $gpo.DisplayName
        Write-Warning "Affected GPO: $($gpo.DisplayName)"
    }
}

if ($affected.Count -eq 0) {
    Write-Host "No GPOs affected. Central Store update is clean." -ForegroundColor Green
} else {
    Write-Host "`n$($affected.Count) GPO(s) require attention:" -ForegroundColor Yellow
    $affected | ForEach-Object { Write-Host "  - $_" }
}
```

```title="Resultat attendu (aucun probleme)"
No GPOs affected. Central Store update is clean.
```

### Verifier manuellement les GPO critiques

Le scan PowerShell detecte les parametres "Extra Registry Settings" au niveau XML. Mais certains problemes ne sont visibles que dans GPMC.

:material-mouse-outline: Ouvrez manuellement les 5 a 10 GPO les plus critiques de votre parc (securite, restrictions, acces reseau). Verifiez que leurs parametres sont toujours affiches correctement et non remplaces par des entrees "Extra Registry Settings".

Cette verification manuelle prend dix minutes. Elle vaut mieux qu'un incident en production.

!!! quote "En resume"
    Trois niveaux de validation : (1) parsing XML de tous les ADMX, (2) scan automatique des GPO pour les Extra Registry Settings, (3) verification manuelle des GPO critiques. Ne pas negliger le niveau 3 — le scan automatique ne voit pas tout.

---

## :material-alert-octagon: Pieges en production

### Conflit de namespace

Si deux fichiers ADMX declarent le meme namespace, GPMC en ignore un silencieusement. Il n'y a aucun message d'erreur. Les parametres de l'un des deux fichiers disparaissent simplement.

Ce scenario arrive typiquement quand on ajoute une version plus recente d'un ADMX tiers sans supprimer l'ancienne, ou quand deux editeurs utilisent le meme namespace (rare mais possible).

```powershell
# Detect namespace conflicts in the Central Store
$centralStore = "\\$env:USERDNSDOMAIN\SYSVOL\$env:USERDNSDOMAIN\Policies\PolicyDefinitions"
$namespaces   = @{}
$conflicts    = 0

Get-ChildItem $centralStore -Filter "*.admx" | ForEach-Object {
    try {
        $xml = [xml](Get-Content $_.FullName -Encoding UTF8 -Raw)
        $ns  = $xml.policyDefinitions.policyNamespaces.target.namespace
        if ($namespaces.ContainsKey($ns)) {
            Write-Warning "CONFLICT: namespace '$ns' declared in '$($namespaces[$ns])' AND '$($_.Name)'"
            $conflicts++
        } else {
            $namespaces[$ns] = $_.Name
        }
    } catch {
        Write-Warning "Parse error in '$($_.Name)': $_"
    }
}

Write-Host "`nChecked $($namespaces.Count) unique namespaces. Conflicts found: $conflicts" -ForegroundColor $(if ($conflicts -eq 0) { 'Green' } else { 'Red' })
```

```title="Resultat attendu (aucun conflit)"
Checked 438 unique namespaces. Conflicts found: 0
```

:material-shield-alert-outline: Executez ce script **avant** chaque ajout d'ADMX tiers. Pas apres. Detecter un conflit avant le deploiement evite de devoir expliquer pourquoi des parametres Chrome ont disparu de GPMC.

!!! warning "GPMC ne signale pas les conflits de namespace"
    Un conflit de namespace provoque une disparition silencieuse de parametres. Aucune erreur dans les logs, aucun avertissement dans GPMC. Seule une verification proactive avec le script ci-dessus permet de les detecter.

### Mise a jour pendant un gel de changement

Ajouter des ADMX pendant un gel rend immediatement visibles de nouveaux parametres dans GPMC. Un administrateur peut, sans le savoir, configurer un parametre non valide par le Change Advisory Board.

:material-calendar-lock-outline: Planifiez systematiquement les mises a jour ADMX en dehors des gels de changement. Incluez-les dans votre calendrier de maintenance regulier — par exemple apres chaque publication de modeles par Microsoft.

!!! danger "Gel de changement : aucune modification du Central Store"
    Une mise a jour ADMX pendant un gel est une modification de l'environnement de production non approuvee. Meme si elle semble "inoffensive", elle cree un risque de derive de configuration. Documentez la politique et faites-la respecter.

### Le Central Store n'est pas sauvegarde par Backup-GPO

`Backup-GPO` sauvegarde les objets GPO (parametres, permissions, liens). Il ne sauvegarde pas le Central Store.

Si SYSVOL est corrumpu ou si le dossier `PolicyDefinitions` est accidentellement supprime, vous perdez tous vos ADMX tiers et personnalises. Les GPO existent toujours, mais leurs parametres deviennent illisibles dans GPMC.

:material-database-export-outline: Incluez le chemin `\\domaine\SYSVOL\domaine\Policies\PolicyDefinitions\` dans votre strategie de sauvegarde SYSVOL separement. Mieux encore : si vous utilisez un depot Git, il constitue lui-meme la sauvegarde.

!!! warning "Sauvegarde SYSVOL separee obligatoire"
    `Backup-GPO` ne sauvegarde pas le Central Store. Assurez-vous que votre strategie de sauvegarde couvre explicitement le chemin `PolicyDefinitions\` dans SYSVOL. Un depot Git joue ce role efficacement s'il est pousse vers un remote distant.

### Oublier les fichiers ADML

Un ADMX sans son ADML dans le dossier de langue de GPMC provoque des descriptions vides, des noms de parametres incomprehensibles, ou des erreurs a l'ouverture de la GPO.

:material-translate: Pour chaque ADMX ajoute au Central Store, verifiez que le fichier ADML correspondant est present dans **tous** les dossiers de langue utilises par vos administrateurs — typiquement `en-US\` et `fr-FR\`.

```powershell
# Verify every ADMX has a matching ADML in en-US and fr-FR
$centralStore   = "\\$env:USERDNSDOMAIN\SYSVOL\$env:USERDNSDOMAIN\Policies\PolicyDefinitions"
$languagesToCheck = @("en-US", "fr-FR")
$missing        = 0

Get-ChildItem $centralStore -Filter "*.admx" | ForEach-Object {
    $admxName = [System.IO.Path]::GetFileNameWithoutExtension($_.Name)
    foreach ($lang in $languagesToCheck) {
        $admlPath = Join-Path $centralStore "$lang\$admxName.adml"
        if (-not (Test-Path $admlPath)) {
            Write-Warning "Missing ADML: $lang\$admxName.adml (for $($_.Name))"
            $missing++
        }
    }
}

if ($missing -eq 0) {
    Write-Host "All ADMX files have matching ADML files for: $($languagesToCheck -join ', ')" -ForegroundColor Green
} else {
    Write-Host "$missing missing ADML file(s) detected." -ForegroundColor Red
}
```

```title="Resultat attendu"
All ADMX files have matching ADML files for: en-US, fr-FR
```

!!! quote "En resume"
    Quatre pieges a surveiller : (1) conflits de namespace — detectez-les avant le deploiement, (2) mises a jour pendant les gels — interdites, (3) sauvegarde SYSVOL — `Backup-GPO` ne suffit pas, (4) ADML manquants — verifiez chaque langue apres chaque ajout. Ces quatre points couvrent 90 % des incidents liees au Central Store en production.

---

## :material-table: Tableau de bord des ADMX tiers

Ce tableau est a maintenir dans votre `CHANGELOG.md` ou votre documentation interne. Il permet de savoir en un coup d'oeil ce qui est dans le Central Store, en quelle version, et par qui.

| Application | Version ADMX | Date ajout | Responsable | Dependances | Dossiers langue |
|-------------|-------------|------------|-------------|-------------|-----------------|
| Windows 11 24H2 | 10.0.26100 | 2025-01-15 | J.Dupont | Aucune | en-US, fr-FR |
| Microsoft Edge 130 | 130.0 | 2025-02-01 | J.Dupont | Aucune | en-US, fr-FR |
| Google Chrome 126 | 126.0 | 2025-03-10 | M.Martin | google.admx | en-US |
| Firefox 124 | 124.0 | 2025-03-10 | M.Martin | Aucune | en-US, fr-FR |
| Office 365 (16.0.x) | v5410 | 2025-01-20 | J.Dupont | Aucune | en-US, fr-FR |
| OneDrive | 24.x | 2025-01-20 | J.Dupont | Aucune | en-US, fr-FR |

:material-pencil-outline: Adaptez ce tableau a votre environnement. Le maintenir a jour prend deux minutes par mise a jour et evite des heures de recherche lors d'un incident.


!!! quote "En résumé"
    - Windows 11 24H2 : en-US, fr-FR.
    - Microsoft Edge 130 : en-US, fr-FR.
    - Google Chrome 126 : en-US.
    - Firefox 124 : en-US, fr-FR.
    - Ce tableau est a maintenir dans votre CHANGELOG.md ou votre documentation interne.
---

## :material-custom-admx: Creer ses propres ADMX

### Quand creer un ADMX personnalise

Un ADMX personnalise est utile quand vous devez gerer des cles de registre d'une application interne via GPO, sans passer par "Preferences > Windows Settings > Registry" (qui ne supporte pas les filtres WMI et n'a pas d'interface documentee).

Les ADMX personalises permettent de presenter vos parametres maison exactement comme les parametres Windows — avec description, documentation, et validation des valeurs.

### Structure minimale d'un ADMX

Un ADMX est un fichier XML avec une structure definie. Voici le squelette minimum pour un ADMX valide :

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- MonApp.admx — Administrative template for MonApp internal application -->
<policyDefinitions revision="1.0" schemaVersion="1.0"
  xmlns="http://schemas.microsoft.com/GroupPolicy/2006/07/PolicyDefinitions">

  <policyNamespaces>
    <!-- Define this ADMX's own namespace -->
    <target prefix="monapp" namespace="Contoso.MonApp" />
  </policyNamespaces>

  <resources minRequiredRevision="1.0" />

  <categories>
    <category name="MonApp" displayName="$(string.MonApp_Category)" />
  </categories>

  <policies>
    <policy name="MonApp_EnableFeatureX"
            class="Machine"
            displayName="$(string.MonApp_EnableFeatureX_Name)"
            explainText="$(string.MonApp_EnableFeatureX_Explain)"
            key="SOFTWARE\Contoso\MonApp"
            valueName="FeatureXEnabled">
      <parentCategory ref="monapp:MonApp" />
      <supportedOn ref="windows:SUPPORTED_Win10" />
      <enabledValue><decimal value="1" /></enabledValue>
      <disabledValue><decimal value="0" /></disabledValue>
    </policy>
  </policies>

</policyDefinitions>
```

:material-file-document-outline: Le fichier ADML correspondant (`MonApp.adml`) contient les chaines de texte referencees par `$(string....)`. Il est place dans `fr-FR\MonApp.adml` et `en-US\MonApp.adml`.

### Tester un ADMX personnalise avant deploiement

Avant de pousser un ADMX personnalise vers le Central Store, testez-le localement :

```powershell
# Test a custom ADMX by copying it to the local PolicyDefinitions (non-domain testing)
$testAdmx = "C:\dev\MonApp\MonApp.admx"
$testAdml = "C:\dev\MonApp\fr-FR\MonApp.adml"
$localStore = "$env:SystemRoot\PolicyDefinitions"

# Validate XML structure first
try {
    $null = [xml](Get-Content $testAdmx -Encoding UTF8 -Raw)
    Write-Host "XML valid: MonApp.admx" -ForegroundColor Green
} catch {
    Write-Host "XML error in MonApp.admx: $_" -ForegroundColor Red
    exit 1
}

# Copy to local store for GPMC testing
Copy-Item $testAdmx $localStore -Force
Copy-Item $testAdml "$localStore\fr-FR\" -Force
Write-Host "Copied to local PolicyDefinitions. Reopen GPMC to test."
```

Ouvrez GPMC, creez une GPO de test, et verifiez que vos parametres apparaissent sous la categorie declaree. Si tout est correct, deployez via le depot Git.

!!! quote "En resume"
    Les ADMX personnalises suivent exactement le meme schema que les ADMX Microsoft. Validez le XML, testez localement, puis deploiement via Git. Un ADMX maison bien concu est indiscernable d'un ADMX Microsoft dans GPMC.

---

## :material-link-variant: Références croisées

| Sujet | Référence |
|-------|-----------|
| Format ADMX en detail (namespace, categories, policies) | La Bible GPO — Ch. 05 — ADMX/ADML |
| Mise a jour ADMX Windows 11 par version | Les GPO pour les Nuls — Ch. 14 |
| Automatisation du deploiement ADMX en CI/CD | Ch. 23 — Automatisation CI/CD |
| Sauvegarde et migration SYSVOL | Ch. 05 — Sauvegarde et migration |
| Gestion des GPO avec PowerShell | Ch. 03 — PowerShell GroupPolicy module |


!!! quote "En résumé"
    - À relire : Format ADMX en detail (namespace, categories, policies) → La Bible GPO — Ch. 05 — ADMX/ADML.
    - À relire : Mise a jour ADMX Windows 11 par version → Les GPO pour les Nuls — Ch. 14.
    - À relire : Automatisation du deploiement ADMX en CI/CD → Ch. 23 — Automatisation CI/CD.
    - À relire : Sauvegarde et migration SYSVOL → Ch. 05 — Sauvegarde et migration.
    - Ces renvois prolongent le chapitre avec des mécanismes complémentaires ou des cas d’usage voisins.
---
*Ce chapitre couvre la creation, la maintenance et la securisation du Central Store en environnement d'entreprise. Le chapitre suivant traite de la sauvegarde et de la migration des GPO — dont la sauvegarde complementaire du Central Store.*
