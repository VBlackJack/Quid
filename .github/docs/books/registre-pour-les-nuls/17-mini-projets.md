---
tags:
  - registre
  - débutant
  - projets
  - PowerShell
  - automatisation
  - sauvegarde
---

# Mini-projets : votre boite a outils

!!! abstract "Ce que vous allez apprendre"
    - Comment creer un fichier `.reg` de personnalisation complet (et son fichier d'annulation)
    - Comment ecrire un script PowerShell qui analyse les programmes au demarrage
    - Comment construire un outil de diagnostic systeme en une commande
    - Comment automatiser la sauvegarde mensuelle de vos cles importantes
    - Comment assembler tout ca dans un "kit de premier secours"

!!! warning "Rappel essentiel"
    Tous ces projets sont **sans danger** et **reversibles**, a condition de suivre les instructions. Avant de commencer, assurez-vous d'avoir une sauvegarde recente de votre registre (chapitre 5). Dans le doute, testez d'abord dans une machine virtuelle.

---

## Introduction

Vous avez traverse 16 chapitres. Vous savez ce qu'est le registre, comment le naviguer, comment le sauvegarder, comment le modifier, comment utiliser les fichiers `.reg` et PowerShell. Il est temps de **mettre tout ca ensemble**.

Chaque projet ci-dessous combine des competences de plusieurs chapitres. Le but n'est pas de vous faire decouvrir quelque chose de nouveau, mais de vous faire **pratiquer** ce que vous avez appris en construisant des outils que vous pourrez utiliser au quotidien.

Les projets sont classes par difficulte croissante :

| Projet | Difficulte | Temps estime | Chapitres utilises |
|--------|:----------:|:------------:|-------------------|
| 1. Fichier `.reg` de personnalisation | :material-star: | 15 min | 4, 7, 8 |
| 2. Script de nettoyage du demarrage | :material-star::material-star: | 20 min | 9, 13 |
| 3. Diagnostic systeme | :material-star::material-star: | 20 min | 2, 13 |
| 4. Sauvegarde automatique mensuelle | :material-star::material-star::material-star: | 30 min | 5, 13 |
| 5. Kit de premier secours | :material-star: | 15 min | Tous |

!!! quote "En resume"
    - Ce chapitre propose 5 projets concrets de difficulte croissante, combinant les competences des 16 chapitres precedents.
    - L'objectif est de **pratiquer** en construisant des outils utilisables au quotidien.

---

## Projet 1 : Mon fichier .reg de personnalisation

### Objectif

Creer un **unique fichier `.reg`** qui applique d'un seul clic toutes vos personnalisations preferees. Plus besoin de naviguer dans 5 menus differents apres chaque reinstallation de Windows.

### Ce que le fichier va faire

| Modification | Effet |
|-------------|-------|
| Activer le mode sombre (applications et systeme) | Interface sombre partout |
| Afficher les extensions de fichiers | Voir `.docx`, `.pdf`, `.exe` au lieu de noms tronques |
| Ouvrir l'Explorateur sur "Ce PC" | Au lieu du dossier "Acces rapide" |
| Desactiver les points forts de la recherche | Plus de suggestions d'actualites dans la barre de recherche |
| Menu contextuel classique (Windows 11) | Retrouver le clic droit complet sans cliquer sur "Afficher plus d'options" |

### Le fichier .reg complet

Creez un nouveau fichier texte sur votre Bureau. Nommez-le `ma-personnalisation.reg`. Ouvrez-le avec le Bloc-notes et collez le contenu suivant :

```reg
Windows Registry Editor Version 5.00

; ===================================================
; Personal customization file
; Apply all favorite settings in one click
; ===================================================

; --- Dark mode for applications and system ---
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize]
"AppsUseLightTheme"=dword:00000000
"SystemUsesLightTheme"=dword:00000000

; --- Show file extensions in Explorer ---
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"HideFileExt"=dword:00000000

; --- Open Explorer on "This PC" instead of Quick Access ---
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"LaunchTo"=dword:00000001

; --- Disable search highlights (news and suggestions) ---
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\SearchSettings]
"IsDynamicSearchBoxEnabled"=dword:00000000

; --- Classic context menu on Windows 11 ---
[HKEY_CURRENT_USER\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32]
@=""
```

### Comment l'appliquer

1. **Enregistrez** le fichier (attention a l'extension : `.reg`, pas `.reg.txt`)
2. **Double-cliquez** sur le fichier
3. Windows vous demande confirmation : cliquez **Oui**
4. Un message confirme que les cles ont ete ajoutees
5. **Redemarrez l'Explorateur** : ouvrez le Gestionnaire des taches (++ctrl+shift+esc++), trouvez "Explorateur Windows", clic droit > **Redemarrer**

!!! tip "Verifier l'extension du fichier"
    Si les extensions sont masquees, le fichier pourrait s'appeler `ma-personnalisation.reg.txt` sans que vous le sachiez. Pour eviter ca, dans le Bloc-notes, choisissez **Fichier > Enregistrer sous**, puis dans le menu **Type**, selectionnez **Tous les fichiers (*.*)** et tapez le nom avec `.reg` a la fin.

### Le fichier d'annulation

C'est la bonne pratique : pour chaque fichier `.reg` qui modifie, creez un fichier `.reg` qui **annule**. Nommez-le `annuler-personnalisation.reg` :

```reg
Windows Registry Editor Version 5.00

; ===================================================
; Undo file: revert all customizations to defaults
; ===================================================

; --- Restore light mode ---
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize]
"AppsUseLightTheme"=dword:00000001
"SystemUsesLightTheme"=dword:00000001

; --- Hide file extensions (Windows default) ---
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"HideFileExt"=dword:00000001

; --- Open Explorer on Quick Access (Windows default) ---
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"LaunchTo"=dword:00000002

; --- Re-enable search highlights ---
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\SearchSettings]
"IsDynamicSearchBoxEnabled"=dword:00000001

; --- Remove classic context menu override (restore Windows 11 menu) ---
[-HKEY_CURRENT_USER\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}]
```

!!! note "Le tiret devant le chemin"
    La ligne `[-HKEY_CURRENT_USER\...]` avec un tiret au debut supprime toute la cle. C'est la syntaxe pour "effacer cette cle et tout ce qu'elle contient". On a vu ca au chapitre 8.

!!! quote "En resume"
    - Un fichier `.reg` de personnalisation applique tous vos reglages preferes en un seul double-clic apres chaque reinstallation.
    - Creez toujours un fichier d'annulation correspondant pour pouvoir revenir aux valeurs par defaut.

---

## Projet 2 : Script de nettoyage du demarrage

### Objectif

Creer un script PowerShell qui **liste tous les programmes lances au demarrage** de Windows, verifie si leurs fichiers existent toujours, et signale les entrees suspectes.

### Pourquoi c'est utile

Au fil du temps, votre PC accumule des programmes au demarrage. Certains sont legitimes, d'autres sont des restes de logiciels desinstalles dont le chemin pointe vers un fichier qui n'existe plus. Ces "fantomes" ralentissent le demarrage sans rien apporter.

### Le script complet

Creez un fichier `verifier-demarrage.ps1` sur votre Bureau et collez-y le contenu suivant :

```powershell
# Startup Programs Diagnostic Script
# Scans Run keys and identifies entries with missing executables

$registryPaths = @(
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run",
    "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run",
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\RunOnce",
    "HKLM:\Software\Microsoft\Windows\CurrentVersion\RunOnce"
)

$totalEntries = 0
$suspiciousEntries = 0

foreach ($path in $registryPaths) {
    Write-Host "`n=== $path ===" -ForegroundColor Cyan

    if (-not (Test-Path $path)) {
        Write-Host "  (this key does not exist)" -ForegroundColor DarkGray
        continue
    }

    $entries = Get-ItemProperty -Path $path -ErrorAction SilentlyContinue
    $names = $entries.PSObject.Properties |
        Where-Object { $_.Name -notlike "PS*" }

    foreach ($entry in $names) {
        $totalEntries++
        $value = $entry.Value

        # Extract the executable path (handle quotes and arguments)
        $exePath = $value -replace '"', '' -replace '\s+/.*', '' -replace '\s+-.*', ''

        if (Test-Path $exePath) {
            Write-Host "  [OK] $($entry.Name)" -ForegroundColor Green
            Write-Host "       $value" -ForegroundColor DarkGray
        } else {
            $suspiciousEntries++
            Write-Host "  [??] $($entry.Name)" -ForegroundColor Yellow
            Write-Host "       $value" -ForegroundColor Yellow
            Write-Host "       -> File not found at expected path" -ForegroundColor Red
        }
    }
}

Write-Host "`n--- Summary ---" -ForegroundColor Cyan
Write-Host "Total startup entries: $totalEntries"
Write-Host "Suspicious entries:    $suspiciousEntries" -ForegroundColor $(
    if ($suspiciousEntries -gt 0) { "Yellow" } else { "Green" }
)
Write-Host "`nPress any key to close..."
$null = $Host.UI.RawUI.ReadKey("NoEcho,IncludeKeyDown")
```

### Comment l'executer

1. Ouvrez PowerShell (++win+x++ > **Terminal** ou **Windows PowerShell**)
2. Si c'est la premiere fois que vous executez un script, autorisez l'execution :
   ```powershell
   # Allow scripts to run for the current session only
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
   ```
   ```title="Resultat attendu"
   Aucune sortie si la commande reussit. C'est normal !
   ```
3. Naviguez vers votre Bureau :
   ```powershell
   cd "$env:USERPROFILE\Desktop"
   ```
   ```title="Resultat attendu"
   PS C:\Users\VotreNom\Desktop>
   ```
4. Lancez le script :
   ```powershell
   .\verifier-demarrage.ps1
   ```

### Comprendre les resultats

```title="Resultat attendu"
=== HKCU:\Software\Microsoft\Windows\CurrentVersion\Run ===
  [OK] SecurityHealth
       "C:\Windows\System32\SecurityHealthSystray.exe"
  [OK] OneDrive
       "C:\Users\Jean\AppData\Local\Microsoft\OneDrive\OneDrive.exe" /background
  [??] AncienLogiciel
       "C:\Program Files\Logiciel\app.exe" --start
       -> File not found at expected path

--- Summary ---
Total startup entries: 3
Suspicious entries:    1
```

- **[OK]** : le programme existe bien a l'emplacement indique. Rien a faire.
- **[??]** : le fichier n'a pas ete trouve. C'est probablement un reste de logiciel desinstalle. Vous pouvez supprimer cette entree dans Regedit en toute securite.

!!! warning "Ne supprimez pas les entrees que vous ne reconnaissez pas"
    "Suspect" ne veut pas dire "dangereux". Avant de supprimer une entree, faites une recherche sur Internet avec le nom du programme. Si c'est un composant Windows ou un pilote, laissez-le tranquille.

!!! quote "En resume"
    - Le script parcourt les 4 cles `Run` et `RunOnce` et verifie si chaque executable existe encore sur le disque.
    - Les entrees marquees `[??]` sont probablement des restes de logiciels desinstalles, supprimables en toute securite apres verification.

---

## Projet 3 : Diagnostic systeme en un clic

### Objectif

Creer un script PowerShell qui affiche les **informations essentielles de votre systeme** en lisant directement dans le registre. Pas besoin d'ouvrir 5 fenetres differentes.

### Le script complet

Creez un fichier `diagnostic-systeme.ps1` sur votre Bureau :

```powershell
# System Diagnostic Script
# Reads key system information from the registry

Write-Host "========================================" -ForegroundColor Cyan
Write-Host "       SYSTEM DIAGNOSTIC REPORT         " -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""

# Computer name
$computerName = (Get-ItemProperty `
    "HKLM:\SYSTEM\CurrentControlSet\Control\ComputerName\ActiveComputerName"
).ComputerName
Write-Host "Computer name:     $computerName"

# Windows version and build
$winVer = Get-ItemProperty `
    "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
$displayVersion = $winVer.DisplayVersion
$buildNumber = $winVer.CurrentBuildNumber
$ubr = $winVer.UBR
Write-Host "Windows version:   $displayVersion (Build $buildNumber.$ubr)"
Write-Host "Product name:      $($winVer.ProductName)"

# Installation date (convert Unix timestamp to readable date)
$installDate = $winVer.InstallDate
if ($installDate) {
    $readableDate = (Get-Date "1970-01-01").AddSeconds($installDate).ToLocalTime()
    Write-Host "Install date:      $($readableDate.ToString('yyyy-MM-dd HH:mm'))"
}

# Registered owner
$owner = $winVer.RegisteredOwner
if ($owner) {
    Write-Host "Registered owner:  $owner"
} else {
    Write-Host "Registered owner:  (not set)"
}

Write-Host ""

# Startup programs count
$startupCount = 0
$runPaths = @(
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run",
    "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run"
)
foreach ($runPath in $runPaths) {
    if (Test-Path $runPath) {
        $props = (Get-ItemProperty $runPath).PSObject.Properties |
            Where-Object { $_.Name -notlike "PS*" }
        $startupCount += ($props | Measure-Object).Count
    }
}
Write-Host "Startup programs:  $startupCount"

# Antivirus status (from Security Center, requires admin for full details)
Write-Host ""
Write-Host "--- Security ---" -ForegroundColor Cyan
try {
    $avProducts = Get-CimInstance -Namespace "root\SecurityCenter2" `
        -ClassName AntiVirusProduct -ErrorAction Stop
    foreach ($av in $avProducts) {
        Write-Host "Antivirus:         $($av.displayName)"
    }
} catch {
    Write-Host "Antivirus:         (unable to query Security Center)"
}

# Windows Defender status from registry
$defenderPath = "HKLM:\SOFTWARE\Microsoft\Windows Defender"
if (Test-Path $defenderPath) {
    $rtProtection = (Get-ItemProperty `
        "$defenderPath\Real-Time Protection" -ErrorAction SilentlyContinue
    ).DisableRealtimeMonitoring
    if ($rtProtection -eq 1) {
        Write-Host "Real-time protect: DISABLED" -ForegroundColor Red
    } else {
        Write-Host "Real-time protect: Enabled" -ForegroundColor Green
    }
}

Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "Report generated:  $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')"
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "`nPress any key to close..."
$null = $Host.UI.RawUI.ReadKey("NoEcho,IncludeKeyDown")
```

### Comment l'executer

Meme methode que le projet 2 :

1. Ouvrez PowerShell
2. Autorisez l'execution si necessaire :
   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
   ```
   ```title="Resultat attendu"
   Aucune sortie si la commande reussit. C'est normal !
   ```
3. Lancez le script :
   ```powershell
   & "$env:USERPROFILE\Desktop\diagnostic-systeme.ps1"
   ```

### Comprendre les resultats

```title="Resultat attendu"
========================================
       SYSTEM DIAGNOSTIC REPORT
========================================

Computer name:     PC-DE-JEAN
Windows version:   24H2 (Build 26100.3775)
Product name:      Windows 11 Pro
Install date:      2025-01-15 14:30
Registered owner:  Jean Dupont

Startup programs:  7

--- Security ---
Antivirus:         Windows Defender
Real-time protect: Enabled

========================================
Report generated:  2026-04-03 10:30:00
========================================
```

Chaque information vient directement du registre. Vous retrouverez ces cles si vous les cherchez manuellement dans Regedit — le script ne fait que lire ce qui est deja la.

!!! quote "En resume"
    - Le script lit directement dans le registre les informations essentielles : nom du PC, version de Windows, date d'installation, programmes au demarrage et etat de l'antivirus.
    - C'est un rapport rapide en une seule commande, sans ouvrir plusieurs fenetres.

---

## Projet 4 : Sauvegarde automatique mensuelle

### Objectif

Creer un script PowerShell qui **exporte automatiquement** les cles les plus importantes du registre dans un dossier date, puis programmer ce script pour qu'il s'execute **une fois par mois** grace au Planificateur de taches.

### Le script de sauvegarde

Creez un fichier `sauvegarde-registre.ps1` dans un dossier permanent (par exemple `C:\Scripts\`) :

```powershell
# Monthly Registry Backup Script
# Exports important registry keys to a dated folder

# Create backup folder with current date
$backupRoot = "$env:USERPROFILE\Documents\Sauvegardes-Registre"
$dateFolder = Get-Date -Format "yyyy-MM-dd"
$backupPath = Join-Path $backupRoot $dateFolder

if (-not (Test-Path $backupPath)) {
    New-Item -Path $backupPath -ItemType Directory -Force | Out-Null
}

# Define keys to back up (name = filename, path = registry key)
$keysToBackup = @(
    @{ Name = "startup-user";       Path = "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" },
    @{ Name = "startup-machine";    Path = "HKLM\Software\Microsoft\Windows\CurrentVersion\Run" },
    @{ Name = "uninstall";          Path = "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall" },
    @{ Name = "explorer-advanced";  Path = "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced" },
    @{ Name = "current-version";    Path = "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" }
)

$successCount = 0

Write-Host "Registry backup starting..." -ForegroundColor Cyan
Write-Host "Destination: $backupPath" -ForegroundColor DarkGray
Write-Host ""

foreach ($key in $keysToBackup) {
    $outputFile = Join-Path $backupPath "$($key.Name).reg"

    # Use reg.exe export to create .reg files
    $result = & reg export $key.Path $outputFile /y 2>&1

    if ($LASTEXITCODE -eq 0) {
        $successCount++
        Write-Host "  [OK] $($key.Name)" -ForegroundColor Green
    } else {
        Write-Host "  [FAIL] $($key.Name): $result" -ForegroundColor Red
    }
}

Write-Host ""
Write-Host "--- Summary ---" -ForegroundColor Cyan
Write-Host "Keys backed up: $successCount / $($keysToBackup.Count)"
Write-Host "Location:       $backupPath"
Write-Host "Total size:     $([math]::Round(((Get-ChildItem $backupPath -File | Measure-Object Length -Sum).Sum / 1KB), 1)) KB"
Write-Host ""
Write-Host "Backup complete." -ForegroundColor Green
```

### Programmer la sauvegarde avec le Planificateur de taches

Avant de planifier, **testez le script manuellement** pour verifier qu'il fonctionne :

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
& "C:\Scripts\sauvegarde-registre.ps1"
```

```title="Resultat attendu"
Registry backup starting...
Destination: C:\Users\Jean\Documents\Sauvegardes-Registre\2026-04-03

  [OK] startup-user
  [OK] startup-machine
  [OK] uninstall
  [OK] explorer-advanced
  [OK] current-version

--- Summary ---
Keys backed up: 5 / 5
Location:       C:\Users\Jean\Documents\Sauvegardes-Registre\2026-04-03
Total size:     847.3 KB

Backup complete.
```

Verifiez que le dossier `Documents\Sauvegardes-Registre\2026-04-03\` a bien ete cree avec les fichiers `.reg`.

Maintenant, programmons l'execution automatique :

!!! example "Pas a pas : Planificateur de taches"

    **1. Ouvrir le Planificateur de taches**

    - Appuyez sur ++win+r++, tapez `taskschd.msc`, appuyez sur ++enter++
    - La fenetre du Planificateur de taches s'ouvre

    **2. Creer une nouvelle tache**

    - Dans le panneau de droite, cliquez sur **Creer une tache...** (pas "Creer une tache de base")
    - Onglet **General** :
        - **Nom** : `Sauvegarde mensuelle du registre`
        - **Description** : `Exporte les cles importantes du registre le 1er de chaque mois`
        - Cochez **Executer avec les autorisations maximales**
        - Configurez pour : **Windows 10** (fonctionne aussi pour Windows 11)

    **3. Configurer le declencheur (quand)**

    - Cliquez sur l'onglet **Declencheurs** > **Nouveau...**
    - **Commencer la tache** : Selon une planification
    - **Parametres** : Tous les mois
    - **Jour** : cochez **1** (premier jour du mois)
    - **Heure** : `09:00:00` (ou l'heure de votre choix)
    - Cochez tous les **mois** de l'annee
    - Cliquez **OK**

    **4. Configurer l'action (quoi)**

    - Cliquez sur l'onglet **Actions** > **Nouvelle...**
    - **Action** : Demarrer un programme
    - **Programme/script** : `powershell.exe`
    - **Ajouter des arguments** :
      ```
      -ExecutionPolicy Bypass -File "C:\Scripts\sauvegarde-registre.ps1"
      ```
    - Cliquez **OK**

    **5. Conditions (optionnel)**

    - Onglet **Conditions** :
        - Decochez "Ne demarrer la tache que si l'ordinateur est sur secteur" (pour que ca marche aussi sur batterie)

    **6. Parametres supplementaires**

    - Onglet **Parametres** :
        - Cochez "Executer la tache des que possible si un demarrage planifie est manque"
        - Cela garantit que la sauvegarde se fera meme si le PC etait eteint le 1er du mois

    **7. Valider**

    - Cliquez **OK**
    - Windows peut vous demander votre mot de passe. Entrez-le et validez.

!!! tip "Tester la tache planifiee"
    Dans le Planificateur de taches, trouvez votre tache dans la **Bibliotheque du Planificateur de taches**, faites un clic droit > **Executer**. Verifiez qu'un nouveau dossier date apparait dans `Documents\Sauvegardes-Registre\`.

### Gerer l'espace disque

Les sauvegardes s'accumulent mois apres mois. Chaque sauvegarde fait generalement entre 500 Ko et 2 Mo. Au bout d'un an, ca reste raisonnable, mais pensez a supprimer les plus anciennes de temps en temps.

!!! quote "En resume"
    - Le script exporte automatiquement les cles importantes du registre dans un dossier date sous `Documents\Sauvegardes-Registre\`.
    - Le Planificateur de taches permet de programmer cette sauvegarde pour qu'elle s'execute automatiquement le 1er de chaque mois.

---

## Projet 5 : Creer un "kit de premier secours"

### Objectif

Rassembler tous les outils que vous avez crees dans un **dossier unique**, bien organise, que vous pouvez copier sur une cle USB. En cas de probleme, vous aurez tout sous la main.

### Structure du dossier

```
Kit-Registre/
├── 01-personnalisation/
│   ├── ma-personnalisation.reg
│   └── annuler-personnalisation.reg
├── 02-diagnostic/
│   ├── verifier-demarrage.ps1
│   └── diagnostic-systeme.ps1
├── 03-sauvegarde/
│   └── sauvegarde-registre.ps1
└── LISEZ-MOI.txt
```

### Le fichier LISEZ-MOI.txt

Creez un fichier `LISEZ-MOI.txt` a la racine du dossier `Kit-Registre` :

```title="LISEZ-MOI.txt"
=============================================
  KIT DE PREMIER SECOURS - REGISTRE WINDOWS
=============================================

Contenu de ce kit :

1. PERSONNALISATION (dossier 01-personnalisation/)
   - ma-personnalisation.reg
     Applique vos reglages preferes : mode sombre, extensions
     de fichiers visibles, Explorateur sur "Ce PC", menu
     contextuel classique sous Windows 11.
     -> Double-cliquez pour appliquer.

   - annuler-personnalisation.reg
     Annule toutes les modifications ci-dessus et restaure
     les reglages par defaut de Windows.
     -> Double-cliquez pour restaurer.

2. DIAGNOSTIC (dossier 02-diagnostic/)
   - verifier-demarrage.ps1
     Liste les programmes qui se lancent au demarrage et
     signale ceux dont le fichier n'existe plus.
     -> Clic droit > "Executer avec PowerShell"

   - diagnostic-systeme.ps1
     Affiche les informations essentielles de votre systeme :
     nom du PC, version de Windows, date d'installation,
     nombre de programmes au demarrage, antivirus.
     -> Clic droit > "Executer avec PowerShell"

3. SAUVEGARDE (dossier 03-sauvegarde/)
   - sauvegarde-registre.ps1
     Exporte les cles importantes du registre dans un
     dossier date sous Documents\Sauvegardes-Registre\.
     -> Clic droit > "Executer avec PowerShell"

---------------------------------------------
EN CAS DE PROBLEME :
1. Ne paniquez pas.
2. Redemarrez le PC. Ca resout souvent le probleme.
3. Si le probleme persiste, restaurez la derniere
   sauvegarde du registre (voir chapitre 5 du livre).
4. Si vous avez modifie le registre recemment, utilisez
   le fichier annuler-personnalisation.reg.
=============================================
```

### Assembler le kit pas a pas

!!! example "Pas a pas"

    **1. Creer la structure de dossiers**

    Creez un dossier `Kit-Registre` sur votre Bureau, puis les trois sous-dossiers :

    ```powershell
    # Create the kit folder structure on the Desktop
    $kitPath = "$env:USERPROFILE\Desktop\Kit-Registre"
    New-Item -Path "$kitPath\01-personnalisation" -ItemType Directory -Force
    New-Item -Path "$kitPath\02-diagnostic" -ItemType Directory -Force
    New-Item -Path "$kitPath\03-sauvegarde" -ItemType Directory -Force
    ```

    ```title="Resultat attendu"
        Directory: C:\Users\VotreNom\Desktop\Kit-Registre

    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    d-----        2026-04-04  10:00            01-personnalisation
    d-----        2026-04-04  10:00            02-diagnostic
    d-----        2026-04-04  10:00            03-sauvegarde
    ```

    **2. Copier vos fichiers**

    Si vous avez cree les fichiers des projets precedents sur votre Bureau :

    ```powershell
    # Copy project files into the kit
    $desktop = "$env:USERPROFILE\Desktop"
    $kitPath = "$desktop\Kit-Registre"

    Copy-Item "$desktop\ma-personnalisation.reg"      "$kitPath\01-personnalisation\"
    Copy-Item "$desktop\annuler-personnalisation.reg"  "$kitPath\01-personnalisation\"
    Copy-Item "$desktop\verifier-demarrage.ps1"        "$kitPath\02-diagnostic\"
    Copy-Item "$desktop\diagnostic-systeme.ps1"        "$kitPath\02-diagnostic\"
    Copy-Item "$desktop\sauvegarde-registre.ps1"       "$kitPath\03-sauvegarde\"
    ```

    ```title="Resultat attendu"
    Aucune sortie si la commande reussit. C'est normal !
    Les fichiers sont copies dans les sous-dossiers du kit.
    ```

    **3. Creer le fichier LISEZ-MOI.txt**

    Copiez le texte de la section precedente dans un fichier `LISEZ-MOI.txt` a la racine du dossier `Kit-Registre`.

    **4. Copier sur une cle USB**

    Branchez une cle USB, puis copiez simplement le dossier `Kit-Registre` dessus. Vous pouvez le faire a la souris (copier-coller) ou en PowerShell :

    ```powershell
    # Copy the kit to a USB drive (replace E: with your drive letter)
    Copy-Item -Path "$env:USERPROFILE\Desktop\Kit-Registre" `
              -Destination "E:\Kit-Registre" `
              -Recurse -Force
    ```

    ```title="Resultat attendu"
    Aucune sortie si la commande reussit. C'est normal !
    Le dossier Kit-Registre est copie sur la cle USB.
    ```

### Quand utiliser le kit

| Situation | Outil a utiliser |
|-----------|-----------------|
| Vous venez de reinstaller Windows | `ma-personnalisation.reg` |
| Un programme ne se lance plus | `verifier-demarrage.ps1` pour verifier les entrees de demarrage |
| Vous voulez connaitre l'etat de votre systeme | `diagnostic-systeme.ps1` |
| Vous voulez sauvegarder avant une grosse modification | `sauvegarde-registre.ps1` |
| Quelque chose ne va plus apres une modification | `annuler-personnalisation.reg` puis redemarrer |

!!! quote "En resume"
    - Le kit de premier secours rassemble tous les outils dans un dossier unique copiable sur cle USB : fichiers de personnalisation, scripts de diagnostic et script de sauvegarde.
    - Un fichier `LISEZ-MOI.txt` documente l'utilisation de chaque outil et la marche a suivre en cas de probleme.

---

## Et maintenant ?

Vous voila arrive au bout de ce livre. Prenons un instant pour mesurer le chemin parcouru.

Au chapitre 1, le registre etait une boite noire mysterieuse. Vous ne saviez pas ce que c'etait, ni a quoi ca servait, et vous aviez probablement un peu peur d'y toucher.

Maintenant, vous savez :

- :material-check: **Ce qu'est** le registre et comment il est structure
- :material-check: **Naviguer** dans Regedit comme dans l'Explorateur de fichiers
- :material-check: **Sauvegarder** avant chaque modification (et restaurer si ca tourne mal)
- :material-check: **Modifier** des valeurs, creer des cles, supprimer des entrees
- :material-check: **Creer des fichiers `.reg`** pour automatiser vos changements
- :material-check: **Utiliser PowerShell** pour lire et ecrire dans le registre
- :material-check: **Diagnostiquer** des problemes courants
- :material-check: **Evaluer** la fiabilite des tutoriels que vous trouvez en ligne
- :material-check: **Construire vos propres outils** pour le quotidien

C'est beaucoup. Vous etes passe de "c'est quoi le registre ?" a "je construis mes propres scripts de diagnostic".

### Pour aller plus loin

Si le sujet vous passionne et que vous voulez approfondir, le livre *La Bible du Registre Windows* couvre les aspects avances : les API Windows, les strategies de groupe en detail, le registre en entreprise, WMI, les taches planifiees avancees, et bien d'autres sujets.

### Le mot de la fin

Le registre n'est pas dangereux. Ce qui est dangereux, c'est de le modifier sans comprendre ce qu'on fait et sans sauvegarde.

Vous comprenez maintenant ce que vous faites. Et vous avez vos sauvegardes.

Alors experimentez. Essayez des choses. Cassez des trucs (dans une machine virtuelle, de preference). C'est comme ca qu'on apprend.

!!! quote "En resume"
    Vous avez construit cinq outils concrets : un fichier de personnalisation, un analyseur de demarrage, un diagnostic systeme, une sauvegarde automatique, et un kit de premier secours. Ces projets combinent tout ce que vous avez appris au fil des 16 chapitres precedents. Le registre n'a plus de secret pour vous — ou en tout cas, beaucoup moins qu'avant. Bonne exploration.
