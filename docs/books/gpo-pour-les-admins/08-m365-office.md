---
description: Microsoft 365 Apps et Office via GPO — ADMX, macros, OneDrive KFM, Teams, sécurité Office
tags: [gpo-admins, office, m365, onedrive, macros, admx]
---
# Microsoft 365 Apps (Office) via GPO

!!! abstract "Ce que vous allez apprendre"
    - Telecharger et installer les ADMX Office dans le Central Store sans collision de namespace
    - Configurer la politique de macros en production : blocage par defaut avec exception signee uniquement
    - Deployer OneDrive Known Folder Move (KFM) silencieux via deux cles de registre et un GUID de tenant
    - Comprendre pourquoi Teams nouvelle generation echappe aux restrictions SRP/AppLocker classiques et comment le prendre en compte
    - Eviter le pieges le plus frequent : autoriser les macros pour un groupe sans mesurer le risque ransomware

!!! tip "Si vous ne retenez qu'une chose"
    Les macros activees sur une seule machine compromise suffisent pour propager un ransomware sur tout le reseau. La politique correcte est `Disable all macros except digitally signed macros` — pas "demander confirmation". Signature obligatoire, pas de negotiation.

---

## :material-microsoft-office: Contexte de production

Depuis le passage a Microsoft 365 Apps for Enterprise (anciennement Office 365 ProPlus), la gestion par GPO reste le seul vecteur de politique fiable pour les postes joints au domaine on-premises.

Intune peut gerer Office sur les postes hybrides ou full-cloud, mais en environnement hybride, la GPO et Intune peuvent coexister — avec un risque de conflit si les memes cles de registre sont ecrites par les deux. Ce chapitre couvre uniquement le perimetre GPO on-premises.

:material-information-outline: Les parametres Office via GPO ecrivent sous `HKCU\SOFTWARE\Policies\Microsoft\Office\` et `HKLM\SOFTWARE\Policies\Microsoft\Office\`. Ces cles ont priorite sur les preferences utilisateur et ne peuvent pas etre ecrasees par l'application elle-meme — contrairement aux cles sous `HKCU\SOFTWARE\Microsoft\Office\`.


!!! quote "En résumé"
    - Ce chapitre couvre uniquement le perimetre GPO on-premises.
    - Les parametres Office via GPO ecrivent sous HKCU\SOFTWARE\Policies\Microsoft\Office\ et HKLM\SOFTWARE\Policies\Microsoft\Office\.
    - Le contexte de production fixe les contraintes réelles de réseau, de portée et d’exploitation qui gouvernent tout le chapitre.
    - Retenez les hypothèses opérationnelles avant de choisir un modèle de liaison ou de déploiement.
---

## :material-download-box-outline: ADMX Office : telechargement et installation

### Ou telecharger les modeles

Microsoft publie les ADMX Office separement des ADMX Windows. Le paquet s'appelle **Administrative Template files (ADMX/ADML) for Microsoft 365 Apps for Enterprise**.

Telechargez depuis le Microsoft Download Center :

```
https://www.microsoft.com/en-us/download/details.aspx?id=49030
```

Le package en cours (2024) produit un dossier `admx\` apres extraction. Il contient plus de 40 fichiers ADMX.

:material-alert-outline: Ne confondez pas ce paquet avec "Office ADMX for Office 2016/2019 on-prem". Pour Microsoft 365 Apps, utilisez le paquet lie ci-dessus — il couvre la version `16.0.x` mais est maintenu par Microsoft pour les fonctionnalites 365.

### Structure du paquet Office

Apres extraction, le dossier `admx\` a la structure suivante :

```
admx/
├── office16.admx               (parametres communs a toutes les apps Office)
├── excel16.admx
├── word16.admx
├── powerpoint16.admx
├── outlook16.admx
├── access16.admx
├── onenote16.admx
├── publisher16.admx
├── project16.admx
├── visio16.admx
├── teams16.admx
├── onedrive.admx               (OneDrive — peut aussi etre telecharge separement)
├── lync16.admx                 (Skype for Business legacy)
├── ... (environ 40 fichiers au total)
├── en-US/
│   └── *.adml
└── fr-FR/
    └── *.adml
```

Le fichier central est `office16.admx`. Il declare le namespace `Microsoft.Policies.Office16v2` dont dependent les fichiers applicatifs. **Il doit etre present avant tous les autres** — sinon GPMC ignore les fichiers applicatifs sans erreur visible.

### Namespace et dependances

Le namespace principal d'Office est `Microsoft.Policies.Office16v2`. Chaque fichier applicatif (`excel16.admx`, `word16.admx`, etc.) le reference comme namespace parent.

Si `office16.admx` est absent du Central Store et que vous copiez uniquement `excel16.admx`, Excel disparait de GPMC — silencieusement.

| Fichier | Namespace declare | Depend de |
|---------|-------------------|-----------|
| `office16.admx` | `Microsoft.Policies.Office16v2` | Aucune |
| `excel16.admx` | `Microsoft.Policies.Excel16` | `office16.admx` |
| `word16.admx` | `Microsoft.Policies.Word16` | `office16.admx` |
| `outlook16.admx` | `Microsoft.Policies.Outlook16` | `office16.admx` |
| `teams16.admx` | `Microsoft.Policies.Teams` | `office16.admx` |
| `onedrive.admx` | `Microsoft.OneDrive` | Aucune |

### Copier les ADMX dans le Central Store

```powershell title="Deploy-OfficeAdmx.ps1"
# Deploy Office ADMX templates to the Central Store
param(
    [Parameter(Mandatory)]
    [string]$OfficeAdmxSource,   # Path to the extracted admx\ folder
    [string]$Domain = (Get-ADDomain).DNSRoot
)

$centralStore = "\\$Domain\SYSVOL\$Domain\Policies\PolicyDefinitions"

if (-not (Test-Path $centralStore)) {
    throw "Central Store not found at: $centralStore. Create it first (see chapter 04)."
}

$added   = 0
$updated = 0
$skipped = 0

# Copy root-level ADMX files
Get-ChildItem $OfficeAdmxSource -Filter "*.admx" | ForEach-Object {
    $dest    = Join-Path $centralStore $_.Name
    $srcHash = (Get-FileHash $_.FullName -Algorithm SHA256).Hash

    if (Test-Path $dest) {
        $dstHash = (Get-FileHash $dest -Algorithm SHA256).Hash
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

# Copy language subfolders
Get-ChildItem $OfficeAdmxSource -Directory | ForEach-Object {
    $destLang = Join-Path $centralStore $_.Name
    New-Item -ItemType Directory -Path $destLang -Force | Out-Null
    Get-ChildItem $_.FullName -Filter "*.adml" | ForEach-Object {
        Copy-Item $_.FullName (Join-Path $destLang $_.Name) -Force
    }
    Write-Host "  LANG    : $($_.Name) processed."
}

Write-Host "`nOffice ADMX deployment complete." -ForegroundColor Green
Write-Host "Added: $added | Updated: $updated | Skipped (identical): $skipped"
```

```title="Resultat attendu"
  ADDED   : office16.admx
  ADDED   : excel16.admx
  ADDED   : word16.admx
  ADDED   : outlook16.admx
  ADDED   : powerpoint16.admx
  ADDED   : teams16.admx
  ADDED   : onedrive.admx
  ... (40+ files total)
  LANG    : en-US processed.
  LANG    : fr-FR processed.

Office ADMX deployment complete.
Added: 43 | Updated: 0 | Skipped (identical): 0
```

### Verifier l'installation dans GPMC

Apres avoir relance GPMC, editez une GPO et naviguez vers :

```
Configuration utilisateur > Modeles d'administration > Microsoft Office 2016
```

Si le noeud "Microsoft Office 2016" apparait avec ses sous-noeuds applicatifs (Excel 2016, Word 2016, Outlook 2016, etc.), l'installation est correcte.

:material-eye-check-outline: Si le noeud n'apparait pas, verifiez que `office16.admx` est bien a la racine du Central Store — pas dans un sous-dossier — et que son ADML est present dans `en-US\` ou `fr-FR\`.

!!! quote "En resume"
    - Le paquet ADMX Office se telecharge depuis le Microsoft Download Center (chercher "Administrative Template files ADMX ADML for Microsoft 365 Apps")
    - `office16.admx` est le fichier pivot : tous les autres en dependent, il doit etre copie en premier
    - Le namespace principal est `Microsoft.Policies.Office16v2`
    - OneDrive possede son propre namespace independant (`Microsoft.OneDrive`) — pas de dependance vers `office16.admx`

---

## :material-shield-lock-outline: Securite Office par GPO

### Politique de macros : le parametre le plus critique

La politique de macros Office est le seul parametre qui fait reellement la difference entre un parc qui resiste aux ransomwares vehicules par Office et un parc qui ne resiste pas.

Il existe quatre niveaux de configuration macro dans le Trust Center Office. Via GPO, le parametre cle est :

```
Configuration utilisateur > Modeles d'administration
  > Microsoft Excel 2016 > Options Excel > Securite > Centre de gestion de la confidentialite
    > Parametres des macros VBA
```

(Reproduire le chemin identique pour Word, PowerPoint, Outlook, Access.)

Les quatre valeurs possibles du parametre **"Parametres des macros VBA"** :

| Valeur | DWORD | Comportement |
|--------|-------|--------------|
| `Disable all macros without notification` | `4` | Macros bloquees, aucun message |
| `Disable all macros with notification` | `2` | Macros bloquees, barre jaune |
| `Disable all macros except digitally signed macros` | `3` | Seules les macros signees s'executent |
| `Enable all macros` | `1` | **Ne jamais utiliser en production** |

**La valeur recommandee en production est `3`** (`Disable all macros except digitally signed macros`).

La valeur `2` (notification) est dangereuse : elle invite l'utilisateur a cliquer "Activer". En pratique, les utilisateurs cliquent. Toujours.

:material-alert-circle-outline: La cle de registre correspondante est :
`HKCU\SOFTWARE\Policies\Microsoft\Office\16.0\<app>\Security\VBAWarnings`

Remplacez `<app>` par `excel`, `word`, `powerpoint`, `outlook`, `access`.

### Configurer la politique de macros via GPO

Dans l'editeur de GPO, pour chaque application :

```
Configuration utilisateur > Modeles d'administration
  > Microsoft Word 2016 > Options Word > Securite > Centre de gestion de la confidentialite
    > Parametres des macros VBA
    → Activer
    → Desactiver toutes les macros a l'exception des macros signees numeriquement
```

Vous devez repeter cette configuration pour chaque application Office dans votre perimetre.

Script PowerShell pour verifier l'etat deploye sur un poste :

```powershell title="Check-OfficeMacroPolicy.ps1"
# Check VBA macro policy for all Office apps on the current machine
$officeApps = @("excel", "word", "powerpoint", "outlook", "access", "publisher")
$officeVer  = "16.0"

$results = foreach ($app in $officeApps) {
    $regPath = "HKCU:\SOFTWARE\Policies\Microsoft\Office\$officeVer\$app\Security"
    $value   = $null

    if (Test-Path $regPath) {
        $value = Get-ItemProperty -Path $regPath -Name "VBAWarnings" -ErrorAction SilentlyContinue
    }

    $level = switch ($value.VBAWarnings) {
        1 { "ENABLED — CRITICAL RISK" }
        2 { "Disabled with notification (user can enable)" }
        3 { "Disabled except signed (recommended)" }
        4 { "Disabled without notification" }
        default { "Not configured (uses user preference)" }
    }

    [PSCustomObject]@{
        Application = $app
        VBAWarnings = $value.VBAWarnings
        PolicyLevel = $level
    }
}

$results | Format-Table -AutoSize
```

```title="Resultat attendu (configuration correcte)"
Application  VBAWarnings PolicyLevel
-----------  ----------- -----------
excel                  3 Disabled except signed (recommended)
word                   3 Disabled except signed (recommended)
powerpoint             3 Disabled except signed (recommended)
outlook                3 Disabled except signed (recommended)
access                 3 Disabled except signed (recommended)
publisher              3 Disabled except signed (recommended)
```

### Protected View et contenu externe

**Protected View** (mode protege) ouvre les fichiers provenant d'Internet, de messagerie, ou d'emplacements potentiellement dangereux dans un bac a sable en lecture seule.

Chemin GPMC :

```
Configuration utilisateur > Modeles d'administration
  > Microsoft Word 2016 > Options Word > Securite > Centre de gestion de la confidentialite
    > Parametres du mode protege
```

Parametres a activer :

| Parametre | Valeur recommandee | Cle de registre |
|-----------|-------------------|-----------------|
| Activer le mode protege pour les fichiers provenant d'Internet | **Activer** | `HKCU\...\Word\Security\ProtectedView\DisableInternetFilesInPV` = `0` |
| Activer le mode protege pour les pieces jointes Outlook | **Activer** | `HKCU\...\Word\Security\ProtectedView\DisableAttachmentsInPV` = `0` |
| Activer le mode protege pour les emplacements non fiables | **Activer** | `HKCU\...\Word\Security\ProtectedView\DisableUnsafeLocationsInPV` = `0` |

:material-shield-check-outline: La logique de ces cles est inversee : `Disable = 0` signifie que la protection est **active**. C'est un piege classique lors d'une verification manuelle.

**Block External Content** empeche Office de charger des ressources distantes (images, feuilles de calcul liees, DDE) a l'ouverture d'un fichier.

```
Configuration utilisateur > Modeles d'administration
  > Microsoft Office 2016 > Options Office > Securite > Centre de gestion de la confidentialite
    > Bloquer le contenu externe
    → Tout bloquer
```

Cle de registre : `HKCU\SOFTWARE\Policies\Microsoft\Office\16.0\common\Security\BlockContentExecutionFromInternet` = `1`

### Mark of the Web (MOTW) et Office

Le **Mark of the Web** est un flux de donnees NTFS alternatif (`Zone.Identifier`) appose par Windows sur les fichiers telecharges depuis Internet. Office lit ce flux et active automatiquement Protected View.

Les archives ZIP desarchivees par l'explorateur Windows propagent le MOTW aux fichiers contenus — depuis Windows 11 22H2. Les archives desarchivees par des outils tiers (7-Zip versions anciennes, WinRAR) ne propagent pas toujours le MOTW.

:material-alert-outline: Un fichier Office malveillant extrait par un outil tiers sans propagation MOTW arrive sur le poste **sans la protection Protected View**. La politique GPO de macros reste le dernier filet de securite dans ce cas.

!!! danger "Production — macros et ransomware"
    Un attaquant qui convainc un utilisateur d'ouvrir un document Word avec une macro activee dispose d'un shell utilisateur sur le poste en quelques secondes. La notification "Activer les macros" (`VBAWarnings = 2`) n'est pas une protection — c'est une invitation deguisee. En production, configurez uniquement `VBAWarnings = 3` et gerez vos macros legitimes via des certificats de signature de code.

!!! quote "En resume"
    - Macros : valeur `3` (`Disable all except digitally signed`) — pas de compromis
    - Protected View : activer les trois parametres (Internet, pieces jointes, emplacements non fiables)
    - Block External Content : activer pour tous les tenants
    - MOTW : complement de defense, pas un remplacement — des outils tiers peuvent ne pas le propager

---

## :material-folder-sync-outline: OneDrive Known Folder Move (KFM)

### Contexte de production

OneDrive KFM redirige automatiquement Bureau, Documents et Images vers OneDrive. Pour l'utilisateur, rien ne change visuellement. Pour l'entreprise, les fichiers utilisateur sont synchronises dans le cloud et accessibles depuis n'importe quelle machine.

En production, KFM via GPO remplace avantageusement la redirection de dossiers classique (Folder Redirection vers un partage reseau) : pas de dependance a un serveur de fichiers on-premises, pas de probleme de latence sur les VPN.

:material-key-outline: Le prerequis incontournable est le **GUID du tenant Azure AD / Microsoft 365**. Sans ce GUID, la GPO KFM ne fonctionne pas — OneDrive ne sait pas vers quel tenant synchroniser.

### Trouver le GUID du tenant

```powershell title="Get-TenantId.ps1"
# Retrieve the Microsoft 365 tenant ID (requires AzureAD or MSOnline module)
# Method 1: via AzureAD module
try {
    Connect-AzureAD -ErrorAction Stop | Out-Null
    $tenantId = (Get-AzureADTenantDetail).ObjectId
    Write-Host "Tenant ID (AzureAD): $tenantId"
} catch {
    Write-Warning "AzureAD module not available or not connected."
}

# Method 2: unauthenticated — works with any tenant domain name
$domain = "contoso.com"   # Replace with your tenant's primary domain
$uri    = "https://login.microsoftonline.com/$domain/.well-known/openid-configuration"
$resp   = Invoke-RestMethod -Uri $uri
$tenantId = ($resp.token_endpoint -split "/")[3]
Write-Host "Tenant ID (unauthenticated): $tenantId"
```

```title="Resultat attendu"
Tenant ID (unauthenticated): xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Conservez ce GUID — il est utilise dans toutes les cles de registre KFM ci-dessous.

### Les trois cles GPO de KFM

Toutes les cles OneDrive GPO se trouvent sous :
`HKLM\SOFTWARE\Policies\Microsoft\OneDrive`

Chemin GPMC pour les parametres OneDrive :

```
Configuration ordinateur > Modeles d'administration
  > OneDrive
```

| Parametre GPO | Nom de la valeur | Type | Description |
|---------------|------------------|------|-------------|
| Silently move Windows known folders to OneDrive | `KFMSilentOptIn` | REG_SZ | GUID du tenant — active KFM silencieux |
| Prevent users from redirecting their Windows known folders to their PC | `KFMBlockOptOut` | DWORD | `1` — empeche l'utilisateur d'annuler |
| Prompt users to move Windows known folders to OneDrive | `KFMSilentOptInWithNotification` | DWORD | `1` — affiche une notification apres la migration |

**Ordre de deploiement recommande :**

1. D'abord `KFMSilentOptIn` seul, en pilote sur 50 machines
2. Verifier que la synchronisation s'effectue correctement
3. Ajouter `KFMSilentOptInWithNotification` pour informer les utilisateurs
4. Ajouter `KFMBlockOptOut` une fois la migration validee sur l'ensemble du parc

### Configuration pas a pas dans GPMC

**Etape 1 — Activer KFM silencieux**

```
Configuration ordinateur > Modeles d'administration > OneDrive
  > Silently move Windows known folders to OneDrive
  → Activer
  → Tenant ID : xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  → Show notification to users after folders have been redirected : Activer (recommande)
```

**Etape 2 — Bloquer le retour arriere utilisateur (apres validation pilote)**

```
Configuration ordinateur > Modeles d'administration > OneDrive
  > Prevent users from redirecting their Windows known folders to their PC
  → Activer
```

**Etape 3 — Verifier le resultat sur un poste pilote**

```powershell title="Check-OneDriveKFM.ps1"
# Verify OneDrive KFM policy keys on the local machine
$regPath = "HKLM:\SOFTWARE\Policies\Microsoft\OneDrive"

if (-not (Test-Path $regPath)) {
    Write-Warning "OneDrive policy registry key not found. GPO not applied yet."
    exit 1
}

$kfmOpt       = (Get-ItemProperty $regPath -Name "KFMSilentOptIn"            -ErrorAction SilentlyContinue).KFMSilentOptIn
$kfmBlock     = (Get-ItemProperty $regPath -Name "KFMBlockOptOut"            -ErrorAction SilentlyContinue).KFMBlockOptOut
$kfmNotif     = (Get-ItemProperty $regPath -Name "KFMSilentOptInWithNotification" -ErrorAction SilentlyContinue).KFMSilentOptInWithNotification

Write-Host "KFMSilentOptIn (Tenant GUID) : $kfmOpt"
Write-Host "KFMBlockOptOut               : $kfmBlock"
Write-Host "KFMSilentOptInWithNotification: $kfmNotif"

# Check actual folder backup status via OneDrive registry
$odStatus = "HKCU:\SOFTWARE\Microsoft\OneDrive\Accounts\Business1"
if (Test-Path $odStatus) {
    $kfmStatus = Get-ItemProperty "$odStatus\ScopeIdToMountPoints" -ErrorAction SilentlyContinue
    Write-Host "`nOneDrive Business1 account present."
} else {
    Write-Warning "No OneDrive Business1 account found on this machine."
}
```

```title="Resultat attendu"
KFMSilentOptIn (Tenant GUID) : xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
KFMBlockOptOut               : 1
KFMSilentOptInWithNotification: 1

OneDrive Business1 account present.
```

### Verifier le statut de synchronisation KFM

Apres deploiement, verifiez que les dossiers connus sont bien dans OneDrive :

```powershell title="Check-KFMFolderStatus.ps1"
# Check whether known folders are backed up to OneDrive on the current user session
$knownFolders = @{
    Desktop   = [Environment]::GetFolderPath("Desktop")
    Documents = [Environment]::GetFolderPath("MyDocuments")
    Pictures  = [Environment]::GetFolderPath("MyPictures")
}

foreach ($folder in $knownFolders.GetEnumerator()) {
    $path = $folder.Value
    $inOneDrive = $path -like "*OneDrive*"
    $status     = if ($inOneDrive) { "OK — in OneDrive" } else { "NOT MIGRATED" }
    Write-Host "[$status] $($folder.Key): $path"
}
```

```title="Resultat attendu (migration reussie)"
[OK — in OneDrive] Desktop  : C:\Users\jdupont\OneDrive - Contoso\Desktop
[OK — in OneDrive] Documents: C:\Users\jdupont\OneDrive - Contoso\Documents
[OK — in OneDrive] Pictures : C:\Users\jdupont\OneDrive - Contoso\Pictures
```

!!! warning "Conflit avec Folder Redirection classique"
    Si vous avez deja une GPO de Folder Redirection qui redirige Bureau ou Documents vers un partage reseau (`\\srv-fichiers\users\%username%\`), KFM et Folder Redirection **entrent en conflit**. Le comportement est imprevisible — l'un des deux gagne selon l'ordre d'application. Desactivez la Folder Redirection avant d'activer KFM. Ne faites jamais coexister les deux sur les memes machines.

!!! quote "En resume"
    - KFM requiert le GUID du tenant — recuperez-le via l'endpoint OpenID Connect de votre domaine
    - Trois cles sous `HKLM\SOFTWARE\Policies\Microsoft\OneDrive` : `KFMSilentOptIn` (GUID), `KFMBlockOptOut` (1), `KFMSilentOptInWithNotification` (1)
    - Deployez progressivement : pilote sans blocage, puis ajout du blocage apres validation
    - Folder Redirection classique et KFM sont incompatibles — choisissez l'un ou l'autre

---

## :material-microsoft-teams: Teams nouvelle generation et GPO

### Pourquoi Teams pose un probleme aux administrateurs GPO

L'ancienne version de Teams (Teams Classic) s'installait dans `%LOCALAPPDATA%\Microsoft\Teams\` comme une application Win32. Les GPO de restriction logicielle (SRP) et AppLocker la controlaient via son chemin d'executable.

Teams nouvelle generation (Teams 2.0 / "new Teams") est une application **MSIX** deployee dans le Windows Store for Business ou via Intune. Elle s'execute dans le contexte utilisateur depuis un chemin virtualisé du type :

```
C:\Program Files\WindowsApps\MSTeams_<version>_x64__8wekyb3d8bbwe\ms-teams.exe
```

:material-alert-circle-outline: **Les regles AppLocker basees sur des chemins de dossier ne fonctionnent pas pour MSIX.** Les applications MSIX s'executent depuis `C:\Program Files\WindowsApps\`, un dossier protege par ACL dont le chemin change a chaque mise a jour.

### Approche GPO pour Teams MSIX

**Option 1 : Regles Publisher AppLocker (recommandee)**

AppLocker supporte des regles basees sur l'editeur (signature numerique) pour les applications MSIX. C'est la seule approche fiable car elle ne depend pas du chemin.

```
Configuration ordinateur > Parametres Windows > Parametres de securite
  > Politiques de controle des applications
    > AppLocker > Packaged App Rules
    → Nouvelle regle > Allow > Publisher
    → Editeur : O=MICROSOFT CORPORATION, L=REDMOND, S=WASHINGTON, C=US
    → Nom du package : MSTeams
```

Cela autorise toutes les versions signees par Microsoft sous le nom de package `MSTeams`.

**Option 2 : Gerer Teams via les ADMX Office**

Le fichier `teams16.admx` fournit des parametres de configuration Teams accessibles via GPO :

```
Configuration ordinateur > Modeles d'administration > Microsoft Teams
```

Parametres utiles en production :

| Parametre | Chemin | Usage |
|-----------|--------|-------|
| Disable auto-update for Microsoft Teams | `HKLM\SOFTWARE\Policies\Microsoft\Office\16.0\teams\` | Bloquer les mises a jour automatiques |
| Prevent Microsoft Teams from starting automatically | Politique utilisateur Teams | Empecher le demarrage au login |

```powershell title="Check-TeamsPolicy.ps1"
# Verify Teams GPO policies on the local machine
$teamsRegPath = "HKLM:\SOFTWARE\Policies\Microsoft\Office\16.0\teams"

if (Test-Path $teamsRegPath) {
    $props = Get-ItemProperty $teamsRegPath
    Write-Host "Teams policy keys found:"
    $props | Format-List
} else {
    Write-Host "No Teams policy keys under HKLM. Checking HKCU..."
    $teamsRegPathUser = "HKCU:\SOFTWARE\Policies\Microsoft\Office\16.0\teams"
    if (Test-Path $teamsRegPathUser) {
        Get-ItemProperty $teamsRegPathUser | Format-List
    } else {
        Write-Warning "No Teams policy keys found. ADMX Teams16 may not be applied."
    }
}
```

### SRP et Teams — le pieges de l'heritage

Si votre environnement utilise **Software Restriction Policies (SRP)** en mode "Deny by default", Teams nouvelle generation est bloque par defaut — mais le message d'erreur est generique et les utilisateurs ne comprennent pas pourquoi Teams ne s'ouvre pas.

Les SRP evaluent le chemin de l'executable. Teams MSIX s'execute depuis `C:\Program Files\WindowsApps\` — chemin qui peut etre couvert par une regle d'exception `%PROGRAMFILES%\WindowsApps\` si elle existe.

:material-wrench-outline: Avant de migrer vers Teams nouvelle generation, **auditez vos regles SRP et AppLocker** pour identifier les regles qui pourraient bloquer les applications MSIX. Preferez les regles Publisher AppLocker aux regles de chemin pour toutes les applications MSIX Microsoft.

!!! warning "Teams Classic en coexistence"
    Pendant la periode de transition, Teams Classic et Teams nouvelle generation peuvent coexister. Les deux executables sont differents, les chemins sont differents. Une regle AppLocker qui autorise Teams Classic n'autorise pas automatiquement Teams nouvelle generation. Verifiez les deux regles.

!!! quote "En resume"
    - Teams nouvelle generation est MSIX — les regles AppLocker de chemin ne fonctionnent pas
    - Utilisez des regles Publisher AppLocker (`MSTeams`, editeur Microsoft) pour la gestion par signature
    - `teams16.admx` permet de configurer certains comportements Teams (demarrage automatique, mises a jour)
    - En SRP "Deny by default", verifiez explicitement que `C:\Program Files\WindowsApps\` est couvert par une exception

---

## :material-license: Activation et licences — ne pas perturber le token M365

### Comment Microsoft 365 Apps s'active

Microsoft 365 Apps utilise un **token d'activation OAuth2** stocke dans le profil utilisateur et renforce par le Service de licences Click-to-Run. Ce mecanisme est distinct de l'activation KMS/MAK utilisee pour Windows et Office volume perpetuel.

Les parametres d'activation sont visibles sous :
`HKLM\SOFTWARE\Microsoft\Office\ClickToRun\Configuration`

```powershell title="Check-M365Activation.ps1"
# Check Microsoft 365 Apps activation and channel configuration
$ctrcPath = "HKLM:\SOFTWARE\Microsoft\Office\ClickToRun\Configuration"

if (-not (Test-Path $ctrcPath)) {
    Write-Warning "Office Click-to-Run not found. Office may not be installed via C2R."
    exit 1
}

$config = Get-ItemProperty $ctrcPath

Write-Host "Product IDs    : $($config.ProductReleaseIds)"
Write-Host "Update Channel : $($config.UpdateChannel)"
Write-Host "Version        : $($config.VersionToReport)"
Write-Host "Update URL     : $($config.UpdateUrl)"
Write-Host "License Status : $($config.LicensingType)"
```

```title="Resultat attendu"
Product IDs    : O365ProPlusRetail
Update Channel : http://officecdn.microsoft.com/pr/492350f6-3a01-4f97-b9c0-c7c6ddf67d60
Version        : 16.0.18730.20164
Update URL     :
License Status : User
```

### Ce que les GPO ne doivent pas faire

Les GPO peuvent ecrire des cles sous `HKLM\SOFTWARE\Policies\Microsoft\Office\16.0\common\`. En revanche, elles ne doivent **jamais** modifier les cles sous `HKLM\SOFTWARE\Microsoft\Office\ClickToRun\Configuration` directement.

Ces cles sont gerees par le service Click-to-Run. Une modification directe peut corrompre l'activation, forcer une reactivation, ou provoquer un etat "Produit sans licence" visible par l'utilisateur.

:material-alert-octagon-outline: **Ne jamais configurer les parametres Click-to-Run (UpdateChannel, ProductReleaseIds) via une GPO Preferences > Windows Settings > Registry.** Utilisez l'outil `setup.exe /configure` avec un fichier `configuration.xml` ou Microsoft Intune pour les parametres C2R.

### Verifier l'absence de conflit GPO/C2R

```powershell title="Check-OfficePolicyConflict.ps1"
# Detect potential GPO keys that could conflict with Click-to-Run activation
$potentialConflicts = @(
    "HKLM:\SOFTWARE\Policies\Microsoft\Office\16.0\common\Licensing",
    "HKLM:\SOFTWARE\Policies\Microsoft\Office\16.0\common\OfficeUpdate"
)

foreach ($key in $potentialConflicts) {
    if (Test-Path $key) {
        Write-Warning "Potential conflict key found: $key"
        Get-ItemProperty $key | Format-List
    } else {
        Write-Host "[OK] Not found: $key"
    }
}
```

```title="Resultat attendu (aucun conflit)"
[OK] Not found: HKLM:\SOFTWARE\Policies\Microsoft\Office\16.0\common\Licensing
[OK] Not found: HKLM:\SOFTWARE\Policies\Microsoft\Office\16.0\common\OfficeUpdate
```

!!! quote "En resume"
    - L'activation M365 est geree par le service Click-to-Run — ne pas y toucher via GPO
    - Les cles sous `HKLM\SOFTWARE\Microsoft\Office\ClickToRun\Configuration` sont hors perimetre GPO
    - Utilisez `setup.exe /configure` ou Intune pour les parametres C2R
    - Un conflit de cles peut provoquer un etat "Produit sans licence" sur les postes

---

## :material-bomb: Piege de production : macros et filtrage de securite

### Le scenario

Un administrateur recoit une demande : "Le service comptabilite a besoin de macros Excel pour ses outils metier. Pouvez-vous activer les macros pour eux ?"

La reaction naturelle : creer une GPO dediee, l'appliquer uniquement au groupe `GRP-Comptabilite` via le filtrage de securite, et configurer `VBAWarnings = 1` (macros activees) sur cette GPO.

Resultat : 15 postes de comptabilite avec macros entierement activees.

### Pourquoi c'est dangereux

**Un seul poste compromis du groupe comptabilite** avec macros activees suffit pour executer un payload VBA en memoire, pivoter sur le reseau, et chiffrer les partages accessibles.

Les macros activees via GPO ne distinguent pas les macros legitimes des macros malveillantes. Elles ouvrent l'execution de code arbitraire a quiconque peut faire ouvrir un fichier Office a un utilisateur du groupe.

:material-virus-outline: Les campagnes de ransomware Emotet, Qakbot et IcedID ont massivement exploite les documents Office avec macros. Leur vecteur initial : un email avec un document Word attache, adresse a un utilisateur dont le profil (comptabilite, RH, direction) indique une probabilite elevee de macros activees.

### La bonne approche

Si des macros legitimes sont indispensables, la seule voie acceptable est :

1. **Signer les macros** avec un certificat de signature de code interne (PKI d'entreprise) ou commercial
2. **Deployer le certificat** dans le magasin "Editeurs de confiance" via GPO : `Configuration ordinateur > Parametres Windows > Parametres de securite > Politiques de cles publiques > Editeurs de confiance`
3. **Configurer `VBAWarnings = 3`** (macros non signees bloquees, macros signees par un editeur de confiance executees)

```
Configuration ordinateur > Parametres Windows > Parametres de securite
  > Politiques de cles publiques > Editeurs de confiance
  → Ajouter : votre certificat de signature de code interne
```

Cette approche garantit que seules **vos** macros s'executent — pas celles d'un attaquant.

### Verifier les editeurs de confiance configures

```powershell title="Check-TrustedPublishers.ps1"
# List certificates in the Trusted Publishers store (machine level)
$store = New-Object System.Security.Cryptography.X509Certificates.X509Store(
    "TrustedPublisher",
    "LocalMachine"
)
$store.Open("ReadOnly")

if ($store.Certificates.Count -eq 0) {
    Write-Warning "No certificates in Trusted Publishers store (LocalMachine)."
    Write-Warning "If VBAWarnings = 3, ALL macros will be blocked (no trusted publisher)."
} else {
    Write-Host "Trusted Publishers (LocalMachine) — $($store.Certificates.Count) certificate(s):"
    foreach ($cert in $store.Certificates) {
        Write-Host "  Subject : $($cert.Subject)"
        Write-Host "  Issuer  : $($cert.Issuer)"
        Write-Host "  Expires : $($cert.NotAfter)"
        Write-Host ""
    }
}

$store.Close()
```

```title="Resultat attendu (configuration correcte)"
Trusted Publishers (LocalMachine) — 1 certificate(s):
  Subject : CN=Contoso Code Signing, O=Contoso Corp, C=FR
  Issuer  : CN=Contoso Internal CA, O=Contoso Corp, C=FR
  Expires : 12/31/2026 23:59:59
```

!!! danger "Production — activer les macros pour un groupe = vecteur ransomware"
    Autoriser `VBAWarnings = 1` (toutes macros activees) sur un groupe de postes, meme restreint, cree un sous-perimetre a haut risque dans votre domaine. Un seul clic sur un document malveillant dans ce groupe peut lancer un ransomware avec les droits de l'utilisateur — acces aux partages reseau inclus. La signature de code est la seule mitigation acceptable. Si signer les macros est "trop complique", le vrai probleme est la gestion des outils metier, pas la politique GPO.

!!! quote "En resume"
    - `VBAWarnings = 1` (macros activees) n'est jamais la bonne reponse, meme pour un sous-groupe
    - La bonne reponse : signer les macros + deployer le certificat de signature dans "Editeurs de confiance" via GPO
    - `VBAWarnings = 3` + certificat de confiance = les macros legitimes fonctionnent, les malveillantes sont bloquees
    - Un seul poste avec macros activees dans un groupe est un vecteur de mouvement lateral pour un ransomware

---

## :material-clipboard-check-outline: Verification post-deploiement

### Checklist de validation en une commande

```powershell title="Invoke-OfficeGPOAudit.ps1"
# Full Office GPO audit — run on a representative workstation after GPO deployment
$report = @()
$officeApps = @("excel", "word", "powerpoint", "outlook", "access")
$officeVer  = "16.0"

# --- Macro policy check ---
foreach ($app in $officeApps) {
    $regPath = "HKCU:\SOFTWARE\Policies\Microsoft\Office\$officeVer\$app\Security"
    $val = $null
    if (Test-Path $regPath) {
        $val = (Get-ItemProperty $regPath -Name "VBAWarnings" -ErrorAction SilentlyContinue).VBAWarnings
    }
    $status = if ($val -eq 3) { "OK" } elseif ($val -eq $null) { "NOT SET" } else { "REVIEW" }
    $report += [PSCustomObject]@{ Check = "MacroPolicy-$app"; Value = $val; Status = $status }
}

# --- OneDrive KFM check ---
$kfmPath = "HKLM:\SOFTWARE\Policies\Microsoft\OneDrive"
$kfmVal  = if (Test-Path $kfmPath) { (Get-ItemProperty $kfmPath -ErrorAction SilentlyContinue).KFMSilentOptIn } else { $null }
$report += [PSCustomObject]@{ Check = "OneDrive-KFMSilentOptIn"; Value = $kfmVal; Status = if ($kfmVal) { "OK" } else { "NOT SET" } }

$kfmBlock = if (Test-Path $kfmPath) { (Get-ItemProperty $kfmPath -ErrorAction SilentlyContinue).KFMBlockOptOut } else { $null }
$report += [PSCustomObject]@{ Check = "OneDrive-KFMBlockOptOut"; Value = $kfmBlock; Status = if ($kfmBlock -eq 1) { "OK" } else { "NOT SET" } }

# --- Known folders location ---
$desktopPath   = [Environment]::GetFolderPath("Desktop")
$documentsPath = [Environment]::GetFolderPath("MyDocuments")
$report += [PSCustomObject]@{ Check = "KFM-Desktop";   Value = $desktopPath;   Status = if ($desktopPath   -like "*OneDrive*") { "OK" } else { "NOT MIGRATED" } }
$report += [PSCustomObject]@{ Check = "KFM-Documents"; Value = $documentsPath; Status = if ($documentsPath -like "*OneDrive*") { "OK" } else { "NOT MIGRATED" } }

# --- Output ---
Write-Host "`n=== Office GPO Audit ===" -ForegroundColor Cyan
$report | Format-Table -AutoSize

$issues = $report | Where-Object { $_.Status -ne "OK" }
if ($issues.Count -eq 0) {
    Write-Host "All checks passed." -ForegroundColor Green
} else {
    Write-Host "$($issues.Count) item(s) require attention:" -ForegroundColor Yellow
    $issues | Format-Table -AutoSize
}
```

```title="Resultat attendu"
=== Office GPO Audit ===

Check                     Value                                          Status
-----                     -----                                          ------
MacroPolicy-excel         3                                              OK
MacroPolicy-word          3                                              OK
MacroPolicy-powerpoint    3                                              OK
MacroPolicy-outlook       3                                              OK
MacroPolicy-access        3                                              OK
OneDrive-KFMSilentOptIn   xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx          OK
OneDrive-KFMBlockOptOut   1                                              OK
KFM-Desktop               C:\Users\jdupont\OneDrive - Contoso\Desktop   OK
KFM-Documents             C:\Users\jdupont\OneDrive - Contoso\Documents OK

All checks passed.
```

### Verifier l'application des GPO sur un poste

```powershell title="Check-GPOApplicationStatus.ps1"
# Run gpresult and filter for Office-related GPOs
$gpresult = gpresult /scope user /r 2>&1
$officeLines = $gpresult | Select-String -Pattern "office|onedrive|teams" -CaseSensitive:$false

Write-Host "=== GPOs matching Office/OneDrive/Teams ===" -ForegroundColor Cyan
$officeLines | ForEach-Object { Write-Host $_.Line }

# Also check last GPO refresh time
$lastRefresh = (Get-Item "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine" -ErrorAction SilentlyContinue)
Write-Host "`nFor full HTML report, run: gpresult /h C:\Temp\gpresult.html"
```


!!! quote "En résumé"
    - Validez toujours le résultat sur un poste ou un utilisateur réellement dans le périmètre avant d’élargir.
    - Conservez les commandes et résultats de contrôle comme preuve de conformité post-déploiement.
    - Retenez surtout ce qui change la portée, l’ordre d’application ou le résultat final observé.
    - Ce résumé sert à vérifier que vous avez retenu le mécanisme, sa portée et sa conséquence pratique.
---

## :material-link-variant: References croisees

| Sujet | Reference |
|-------|-----------|
| Format ADMX en detail (namespace, dependances, structure XML) | [La Bible GPO — Ch. 05 — ADMX/ADML](../bible-gpo/05-admx-adml.md) |
| Creer le Central Store et y deployer les ADMX tiers | [Ch. 04 — Central Store](04-central-store.md) |
| Signature de code et PKI interne pour macros | Ch. 15 — Certificats et PKI |
| AppLocker et regles Publisher pour MSIX | Ch. 14 — Securite endpoint |
| Deploiement Office via MECM/SCCM en complement GPO | Ch. 20 — SCCM/MECM |


!!! quote "En résumé"
    - À relire : Format ADMX en detail (namespace, dependances, structure XML) → La Bible GPO — Ch. 05 — ADMX/ADML.
    - À relire : Creer le Central Store et y deployer les ADMX tiers → Ch. 04 — Central Store.
    - À relire : Signature de code et PKI interne pour macros → Ch. 15 — Certificats et PKI.
    - À relire : AppLocker et regles Publisher pour MSIX → Ch. 14 — Securite endpoint.
    - À relire : La Bible GPO — Ch. 05 — ADMX/ADML.
---
*Ce chapitre couvre la gestion de Microsoft 365 Apps en environnement on-premises via GPO : ADMX, securite des macros, KFM OneDrive, Teams nouvelle generation et conflits d'activation. Le chapitre suivant traite des navigateurs web (Edge, Chrome, Firefox) geres par GPO.*
