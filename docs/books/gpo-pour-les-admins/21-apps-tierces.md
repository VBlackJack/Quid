---
description: Applications tierces via GPO — Adobe, Java, 7-Zip, méthode générique ProcMon, AppLocker Publisher rules
tags: [gpo-admins, apps-tierces, adobe, java, 7-zip, gpp-registry, procmon]
---

# Applications tierces via GPO

!!! abstract "Ce que vous allez apprendre"
    - Deployer la configuration d'Adobe Acrobat/Reader via ADMX officiel ou GPP Registry en l'absence de modele
    - Verrouiller Java en entreprise : desactiver le plugin navigateur, deployer un `deployment.properties` centralise et gerer les listes de sites autorises
    - Appliquer la methode generique ProcMon pour configurer n'importe quelle application sans ADMX en capturant ses cles de registre
    - Creer des regles AppLocker de type Publisher basees sur le certificat editeur plutot que sur le hash — stables entre mises a jour
    - Eviter le piege du remplacement GPP qui ecrase les modifications utilisateur a chaque cycle

!!! tip "Si vous ne retenez qu'une chose"
    Toute application ecrit sa configuration dans le registre. ProcMon vous dit exactement quelles cles elle utilise. Une fois ces cles identifiees, GPP Registry les deploie sans ADMX, sans MSI customise, sans script. C'est la methode universelle — utilisez-la en premier.

---

## :material-file-pdf-box: Adobe Acrobat et Reader via GPO

### Contexte de production

Adobe est omniprésent dans les entreprises et sa surface d'attaque est importante. Le JavaScript PDF, les mises a jour automatiques qui consomment de la bande passante et les connexions vers les serveurs Adobe Analytics sont trois points de friction courants.

La gestion centralisee via GPO permet de traiter ces trois problemes en une seule GPO, appliquee a l'ensemble du parc simultanement.

:material-information-outline: Ce chapitre couvre **Adobe Acrobat** (version payante) et **Adobe Reader** (version gratuite). Les cles de registre different selon le produit et la version. Les exemples ci-dessous utilisent Acrobat DC / Reader DC (version `DC` = Document Cloud, la gamme actuelle).

### Obtenir les ADMX officiels Adobe

Adobe publie ses modeles ADMX sur son FTP public. Ils couvrent Reader et Acrobat DC.

```
ftp://ftp.adobe.com/pub/adobe/reader/
```

Pour Acrobat, le paquet s'appelle `AcrobatADMX.zip` (ou equivalent selon la version). Telechargez la version correspondant a la version deployee dans votre parc.

:material-alert-outline: Adobe met a jour les ADMX a chaque version majeure. Si votre parc est sur Acrobat 2020 et que vous deployez les ADMX DC, certains parametres n'auront pas d'effet. Verifiez la correspondance version/ADMX dans la release note du package.

Le paquet ADMX extrait produit deux fichiers principaux :

| Fichier | Contenu |
|---------|---------|
| `reader.admx` | Parametres Adobe Reader DC |
| `AcrobatStd.admx` / `AcrobatPro.admx` | Parametres Acrobat Standard / Pro |
| `en-US\reader.adml` | Libelles anglais correspondants |

Copiez ces fichiers dans le Central Store comme pour tout ADMX tiers (voir [04-central-store.md](04-central-store.md)).

### Chemin GPMC apres installation des ADMX

```
Configuration ordinateur
  └── Modeles d'administration
        └── Adobe Reader
              ├── Preferences
              ├── Security (Enhanced)
              └── ... (20+ categories)
```

### Parametres de securite critiques

Ces quatre parametres forment le socle minimal de durcissement pour tout deploiement Adobe en entreprise.

| Parametre GPMC | Valeur | Cle de registre |
|---|---|---|
| Disable JavaScript | Activer | `HKLM\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown\bDisableJavaScript` = `1` |
| Enable Protected Mode at startup | Activer | `HKLM\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown\bProtectedMode` = `1` |
| Enable Enhanced Security | Activer | `HKLM\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown\bEnhancedSecurityStandalone` = `1` |
| Disable Automatic Updates | Activer | `HKLM\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown\bUpdater` = `0` |

:material-check-circle-outline: Le Protected Mode isole le processus de rendu dans un bac a sable. C'est la protection la plus importante contre l'exploitation de failles PDF — ne la desactivez jamais sans raison documentee et approuvee.

### Deploiement si aucun ADMX n'est disponible

Si votre version d'Adobe n'a pas d'ADMX, ou si vous ne pouvez pas modifier le Central Store, deployez les memes valeurs via **GPP Registry**.

Creez un item GPP pour chaque cle sous :

```
HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown\
```

Le chemin `Policies\Adobe\` est traite par Adobe comme un emplacement de politique — les valeurs y sont lues en priorite et ne peuvent pas etre modifiees par l'utilisateur, meme si l'ADMX n'est pas charge.

```powershell title="Set-AdobeReaderPolicy.ps1"
# Deploy Adobe Reader DC hardening settings via registry
# These keys are read as policy by Adobe Reader DC even without ADMX
$policyBase = "HKLM:\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown"

if (-not (Test-Path $policyBase)) {
    New-Item -Path $policyBase -Force | Out-Null
}

$settings = @{
    bDisableJavaScript           = 1   # Disable JavaScript execution in PDFs
    bProtectedMode               = 1   # Enable Protected Mode (sandbox renderer)
    bEnhancedSecurityStandalone  = 1   # Enable Enhanced Security for local files
    bUpdater                     = 0   # Disable automatic updates
}

foreach ($name in $settings.Keys) {
    Set-ItemProperty -Path $policyBase -Name $name -Value $settings[$name] -Type DWord
    Write-Host "  SET: $name = $($settings[$name])"
}

Write-Host "`nAdobe Reader policy applied." -ForegroundColor Green
```

```title="Resultat attendu"
  SET: bDisableJavaScript = 1
  SET: bProtectedMode = 1
  SET: bEnhancedSecurityStandalone = 1
  SET: bUpdater = 0

Adobe Reader policy applied.
```

!!! warning "A surveiller"
    Adobe a change plusieurs noms de cles entre Reader XI, DC et la gamme Acrobat 2020/2024. Si vous migrez d'une version, verifiez que les cles de l'ancienne version sont supprimees — sinon les deux coexistent et la priorite n'est pas garantie.

### Verification post-deploiement

```powershell title="Verify-AdobePolicy.ps1"
# Verify Adobe Reader policy registry values on local machine
$policyBase = "HKLM:\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown"

$expected = @{
    bDisableJavaScript           = 1
    bProtectedMode               = 1
    bEnhancedSecurityStandalone  = 1
    bUpdater                     = 0
}

Write-Host "Adobe Reader DC — Policy verification" -ForegroundColor Cyan
Write-Host ("=" * 50)

foreach ($name in $expected.Keys) {
    try {
        $actual = (Get-ItemProperty -Path $policyBase -Name $name -ErrorAction Stop).$name
        $ok     = ($actual -eq $expected[$name])
        $status = if ($ok) { "OK  " } else { "FAIL" }
        $color  = if ($ok) { "Green" } else { "Red" }
        Write-Host "[$status] $name = $actual (expected: $($expected[$name]))" -ForegroundColor $color
    } catch {
        Write-Host "[MISS] $name — key not found" -ForegroundColor Red
    }
}
```

```title="Resultat attendu"
Adobe Reader DC — Policy verification
==================================================
[OK  ] bDisableJavaScript = 1 (expected: 1)
[OK  ] bProtectedMode = 1 (expected: 1)
[OK  ] bEnhancedSecurityStandalone = 1 (expected: 1)
[OK  ] bUpdater = 0 (expected: 0)
```

!!! quote "En resume"
    - Les ADMX Adobe sont disponibles sur le FTP public Adobe — utilisez la version correspondant a votre parc
    - Le chemin de politique est `HKLM\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown\`
    - Les quatre parametres prioritaires : JavaScript desactive, Protected Mode actif, Enhanced Security actif, mises a jour desactivees
    - En l'absence d'ADMX, GPP Registry sur ce meme chemin produit le meme effet car Adobe lit ce chemin comme politique

---

## :material-coffee: Java via GPO

### Contexte de production

Java est l'un des logiciels les plus difficiles a gerer en entreprise. Les raisons sont multiples : plusieurs versions peuvent coexister sur la meme machine, le plugin navigateur a ete abandonne mais des applications web legacy en dependent encore, et les mises a jour frequentes perturbent les environnements ou une version specifique est requise.

L'objectif GPO est simple : desactiver ce qui n'est pas utilise (plugin navigateur, Java Web Start pour les postes hors perimetre), et controler les sites autorises a utiliser Java pour les applications qui en ont besoin.

### Desactiver le plugin navigateur et Java Web Start

Java ne possede pas d'ADMX officiel pour ces parametres. Le mecanisme natif de Java est le fichier `deployment.properties`, deploye via GPP File ou GPP Registry.

Le fichier `deployment.properties` se trouve (selon la version et la configuration) dans :

```
C:\Windows\Sun\Java\Deployment\deployment.properties   (system-level)
C:\Users\<user>\AppData\LocalLow\Sun\Java\Deployment\deployment.properties   (user-level)
```

En entreprise, on cible le fichier **system-level** (`C:\Windows\Sun\Java\Deployment\`) — il a priorite sur le fichier utilisateur et s'applique a tous les comptes.

### Deployer deployment.properties via GPP File

Creez d'abord le fichier de configuration dans un partage SYSVOL ou un partage reseau accessible :

```ini title="deployment.properties"
# Java deployment configuration — managed by GPO
# System-level policy — overrides user-level settings

# Disable browser plugin (NPAPI) for all browsers
deployment.webjava.enabled=false

# Disable Java Web Start for all users
deployment.javaws.shortcut=NEVER
deployment.javaws.autodownload=NEVER

# Disable automatic Java update checks
deployment.expiration.check.enabled=false

# Security level: HIGH (blocks unsigned and self-signed apps unless in exception list)
deployment.security.level=HIGH

# Path to the exception site list (allow Java only on approved sites)
deployment.user.security.exception.sites=C:\Windows\Sun\Java\Deployment\exception.sites

# Lock these settings so users cannot override them
deployment.webjava.enabled.locked
deployment.security.level.locked
deployment.expiration.check.enabled.locked
```

Via GPP, creez une action de type **File** :

```
Category        : Computer Configuration > Preferences > Windows Settings > Files
Action          : Replace
Source file     : \\domain.contoso.com\NETLOGON\java\deployment.properties
Destination     : C:\Windows\Sun\Java\Deployment\deployment.properties
Suppress errors : No
```

:material-information-outline: L'action **Replace** ecrase le fichier a chaque cycle de GPO, ce qui garantit que la configuration reste coherente. Voir la section "Piege de production" en fin de chapitre pour comprendre quand "Replace" est problematique.

### Gerer la liste de sites Java autorises

Les applications web legacy qui necessitent Java doivent etre dans le fichier `exception.sites`. Deployez-le de la meme maniere :

```ini title="exception.sites"
# Java exception site list — managed by GPO
# One URL per line — exact match or wildcard prefix
https://erp.contoso.com
https://legacy-app.contoso.local
http://192.168.10.55
```

GPP File — action **Replace** :

```
Source file  : \\domain.contoso.com\NETLOGON\java\exception.sites
Destination  : C:\Windows\Sun\Java\Deployment\exception.sites
```

!!! danger "Production"
    Si `exception.sites` n'existe pas mais que `deployment.user.security.exception.sites` y fait reference, Java refuse de demarrer et affiche une erreur cryptique. Deployez toujours le fichier `exception.sites` en meme temps que `deployment.properties` — meme s'il est vide.

### Probleme des versions multiples de Java

Sur un parc enterprise, il est courant de trouver Java 8, Java 11 et Java 17 sur la meme machine selon les applications installees. Chaque version a son propre dossier dans `C:\Program Files\Java\`.

Le fichier `deployment.properties` system-level est partage par toutes les versions de la JRE Oracle. Cependant, les JDK Adoptium/Temurin (OpenJDK) ne lisent pas ce fichier — ils ont leur propre mecanisme de configuration.

| Distribution Java | Lit `deployment.properties` | Mecanisme alternatif |
|---|---|---|
| Oracle JRE/JDK 8 | Oui | — |
| Oracle JRE/JDK 11+ | Partiellement | `java.security` et flags JVM |
| Adoptium/Temurin (OpenJDK) | Non | `java.security`, variables d'environnement |
| Amazon Corretto | Non | Variables d'environnement |

:material-alert-outline: Si votre application legacy necessite Oracle Java 8 specifiquement, verrouillez la version avec `deployment.expiration.check.enabled=false` et **gerez les mises a jour manuellement** via SCCM/MECM ou un script planifie. Ne laissez pas Java se mettre a jour automatiquement si une version specifique est requise.

### Verification Java

```powershell title="Verify-JavaPolicy.ps1"
# Verify Java system-level deployment.properties is present and locked
$systemDeployment = "C:\Windows\Sun\Java\Deployment\deployment.properties"
$exceptionSites   = "C:\Windows\Sun\Java\Deployment\exception.sites"

Write-Host "Java Deployment Policy — Verification" -ForegroundColor Cyan
Write-Host ("=" * 50)

foreach ($file in @($systemDeployment, $exceptionSites)) {
    $exists = Test-Path $file
    $status = if ($exists) { "OK  " } else { "MISS" }
    $color  = if ($exists) { "Green" } else { "Red" }
    Write-Host "[$status] $file" -ForegroundColor $color
}

# Check key settings if file exists
if (Test-Path $systemDeployment) {
    $content = Get-Content $systemDeployment -Raw
    $checks  = @(
        "deployment.webjava.enabled=false",
        "deployment.security.level=HIGH",
        "deployment.webjava.enabled.locked",
        "deployment.security.level.locked"
    )
    Write-Host "`nContent checks:" -ForegroundColor Cyan
    foreach ($check in $checks) {
        $found  = $content -match [regex]::Escape($check)
        $status = if ($found) { "OK  " } else { "FAIL" }
        $color  = if ($found) { "Green" } else { "Red" }
        Write-Host "[$status] $check" -ForegroundColor $color
    }
}
```

```title="Resultat attendu"
Java Deployment Policy — Verification
==================================================
[OK  ] C:\Windows\Sun\Java\Deployment\deployment.properties
[OK  ] C:\Windows\Sun\Java\Deployment\exception.sites

Content checks:
[OK  ] deployment.webjava.enabled=false
[OK  ] deployment.security.level=HIGH
[OK  ] deployment.webjava.enabled.locked
[OK  ] deployment.security.level.locked
```

!!! quote "En resume"
    - Java n'a pas d'ADMX officiel — le vecteur de configuration est `deployment.properties` au niveau system
    - Deployez via GPP File en action Replace : `C:\Windows\Sun\Java\Deployment\deployment.properties`
    - Verrouillez les cles sensibles avec la syntaxe `cle.locked` (ligne sans valeur)
    - Deployez toujours `exception.sites` en meme temps — meme vide
    - Les JDK OpenJDK (Adoptium, Corretto) ne lisent pas ce fichier : gerez-les separement

---

## :material-magnify-scan: Apps sans ADMX — la methode generique ProcMon

### Contexte de production

La majorite des applications tierces n'ont pas d'ADMX. Pourtant, chacune stocke sa configuration quelque part : dans le registre, dans un fichier `.ini`, dans un fichier XML dans `AppData`.

La methode decrite ici fonctionne pour n'importe quelle application. Elle repose sur **Process Monitor** (ProcMon) de Sysinternals pour observer ce que l'application ecrit quand vous la configurez manuellement, puis sur GPP Registry pour repliquer ces ecritures sur l'ensemble du parc.

### Configurer ProcMon pour capturer les ecritures registre

**Etape 1 — Lancer ProcMon en tant qu'administrateur.**

Telechargez ProcMon depuis [Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) et lancez `Procmon.exe` avec les droits administrateur.

**Etape 2 — Appliquer un filtre precis avant de lancer la capture.**

Sans filtre, ProcMon capture des milliers d'evenements par seconde. Filtrez sur le processus cible :

```
Menu Filter > Filter... (Ctrl+L)

Condition 1:  Process Name  | is  | <nom_du_process.exe>  | Include
Condition 2:  Operation     | is  | RegSetValue           | Include
Condition 3:  Result        | is  | SUCCESS               | Include

Action: Add each condition, then OK
```

:material-information-outline: `RegSetValue` capture uniquement les ecritures de valeurs de registre. Ajoutez `RegCreateKey` si vous voulez aussi voir les creations de cles. Filtrez sur `SUCCESS` pour ignorer les tentatives en echec qui polluent la vue.

**Etape 3 — Reproduire la configuration manuellement.**

Avec ProcMon actif et le filtre en place :

1. Ouvrez l'application
2. Naviguez vers les parametres que vous voulez deployer
3. Changez chaque parametre un par un, en observant ProcMon en temps reel
4. Enregistrez les modifications dans l'application

**Etape 4 — Exporter les resultats.**

```
Menu File > Save... > Format: CSV > Save
```

Le CSV contient : horodatage, processus, PID, operation, chemin de registre, resultat, detail (nom et valeur).

**Etape 5 — Identifier les cles pertinentes.**

Ouvrez le CSV dans Excel ou PowerShell. Filtrez sur les chemins qui commencent par `HKCU\Software\<NomApp>` ou `HKLM\Software\<NomApp>`.

```powershell title="Parse-ProcmonCsv.ps1"
# Parse ProcMon CSV export to extract registry write operations for a given app path
param(
    [Parameter(Mandatory)] [string]$CsvPath,
    [Parameter(Mandatory)] [string]$RegistryPrefix   # e.g. "HKCU\Software\7-Zip"
)

Import-Csv $CsvPath | Where-Object {
    $_.Operation -eq "RegSetValue" -and
    $_.Path -like "$RegistryPrefix*" -and
    $_.Result -eq "SUCCESS"
} | Select-Object Path, @{N="ValueName"; E={$_.Detail -replace "^Name: ([^,]+).*", '$1'}},
                         @{N="Data";      E={$_.Detail -replace ".*Data: (.*)",    '$1'}} |
    Sort-Object Path, ValueName -Unique |
    Format-Table -AutoSize
```

```title="Resultat attendu (exemple pour 7-Zip)"
Path                                         ValueName          Data
----                                         ---------          ----
HKCU\Software\7-Zip\Options                  Lang               fr
HKCU\Software\7-Zip\Options                  ArcHistory         10
HKCU\Software\7-Zip\FM                       ShowDots           1
HKCU\Software\7-Zip\FM                       ShowRealFileIcons  1
```

### Exemple complet : 7-Zip via GPP Registry

**Scenario :** configurer 7-Zip pour tous les utilisateurs avec la langue en francais, l'historique d'archives limite a 10 entrees, et les associations de fichiers `.zip` et `.7z` activees.

**Etape 1 — Cles identifiees par ProcMon :**

| Chemin | Valeur | Type | Donnee |
|--------|--------|------|--------|
| `HKCU\Software\7-Zip\Options` | `Lang` | REG_SZ | `fr` |
| `HKCU\Software\7-Zip\Options` | `ArcHistory` | REG_DWORD | `10` |
| `HKCU\Software\7-Zip\Options` | `Level` | REG_DWORD | `5` (Normal) |
| `HKCU\Software\7-Zip\FM` | `ShowDots` | REG_DWORD | `1` |

**Etape 2 — Creer les items GPP Registry dans GPMC :**

```
GPO : Config-Apps-Tierces
  └── Configuration utilisateur
        └── Preferences
              └── Parametres Windows
                    └── Registry
```

Pour chaque cle, creez un item avec :

```
Action     : Update
Hive       : HKEY_CURRENT_USER
Key path   : Software\7-Zip\Options
Value name : Lang
Value type : REG_SZ
Value data : fr
```

Repliquez pour chaque cle du tableau ci-dessus.

**Etape 3 — Ciblage ILT pour le groupe pilote :**

Avant de deployer sur l'ensemble du parc, limitez la GPO a un groupe de test via le ciblage au niveau des elements (Item-Level Targeting) :

```
Chaque item GPP > Common tab > Item-level targeting > New Item
  Type    : Security Group
  Group   : GRP-Pilot-Apps-Tierces
  Action  : is member of
```

:material-check-circle-outline: Le ciblage ILT s'applique element par element. Un utilisateur hors du groupe pilote verra simplement les valeurs ignorees — sa configuration 7-Zip existante n'est pas touchee.

**Etape 4 — Valider sur le groupe pilote, puis supprimer le ciblage ILT pour deploiement large.**

!!! warning "A surveiller"
    7-Zip ecrit ses associations de fichiers sous `HKCU\Software\Classes\` en plus de `HKCU\Software\7-Zip\`. Les associations de fichiers peuvent etre gerees plus proprement via `Default Programs` (GPP Shortcuts ou OpenWith policy). N'utilisez pas GPP Registry pour forcer les associations de fichiers — le mecanisme Windows de gestion des associations peut les reecrire.

### Verification 7-Zip

```powershell title="Verify-7ZipPolicy.ps1"
# Verify 7-Zip GPP registry settings for current user
$optionsPath = "HKCU:\Software\7-Zip\Options"
$fmPath      = "HKCU:\Software\7-Zip\FM"

$expected = @{
    "$optionsPath|Lang"       = "fr"
    "$optionsPath|ArcHistory" = 10
    "$optionsPath|Level"      = 5
    "$fmPath|ShowDots"        = 1
}

Write-Host "7-Zip GPP Policy — Verification" -ForegroundColor Cyan
Write-Host ("=" * 45)

foreach ($key in $expected.Keys) {
    $parts    = $key -split "\|"
    $regPath  = $parts[0]
    $valName  = $parts[1]
    try {
        $actual = (Get-ItemProperty -Path $regPath -Name $valName -ErrorAction Stop).$valName
        $ok     = ($actual -eq $expected[$key])
        $status = if ($ok) { "OK  " } else { "FAIL" }
        $color  = if ($ok) { "Green" } else { "Red" }
        Write-Host "[$status] $regPath\$valName = $actual" -ForegroundColor $color
    } catch {
        Write-Host "[MISS] $regPath\$valName" -ForegroundColor Red
    }
}
```

```title="Resultat attendu"
7-Zip GPP Policy — Verification
=============================================
[OK  ] HKCU:\Software\7-Zip\Options\Lang = fr
[OK  ] HKCU:\Software\7-Zip\Options\ArcHistory = 10
[OK  ] HKCU:\Software\7-Zip\Options\Level = 5
[OK  ] HKCU:\Software\7-Zip\FM\ShowDots = 1
```

!!! quote "En resume"
    - ProcMon + filtre `RegSetValue + SUCCESS + Process Name` identifie toutes les cles ecrites par une application lors de sa configuration
    - GPP Registry avec action `Update` deploie ces cles sans ADMX
    - Testez toujours avec un ciblage ILT sur un groupe pilote avant le deploiement large
    - Les associations de fichiers ont leur propre mecanisme Windows — ne les forcez pas via GPP Registry simple

---

## :material-shield-lock-outline: AppLocker — regles Publisher pour les apps tierces

### Contexte de production

AppLocker est souvent configure avec des regles de type **Hash** pour les applications tierces : rapide a creer, mais cassante a chaque mise a jour de l'application. Le hash change a chaque version, donc la regle doit etre mise a jour avant chaque deploiement — sinon l'application est bloquee.

Les regles **Publisher** reposent sur le certificat de signature de l'editeur. Tant que l'application est signee par le meme editeur avec le meme certificat, la regle reste valide meme apres une mise a jour. C'est la methode recommandee pour les applications tierces actives dans la duree.

:material-information-outline: Pour AppLocker et WDAC en contexte general, consultez [../bible-gpo/15-applocker-wdac.md](../bible-gpo/15-applocker-wdac.md). Ce chapitre se concentre sur le workflow specifique aux applications tierces non-Microsoft.

### Extraire les informations Publisher d'un executable

Avant de creer une regle Publisher, extraire les informations de signature de l'exe cible :

```powershell title="Get-PublisherInfo.ps1"
# Extract publisher information from a signed executable for AppLocker rule creation
param(
    [Parameter(Mandatory)]
    [string]$ExecutablePath
)

# Method 1: Get-AppLockerFileInformation (requires RSAT or AppLocker feature)
Write-Host "=== AppLocker Publisher Info ===" -ForegroundColor Cyan
$info = Get-AppLockerFileInformation -Path $ExecutablePath
$info | Select-Object -ExpandProperty Publisher | Format-List

# Method 2: Signature details via Get-AuthenticodeSignature (always available)
Write-Host "`n=== Authenticode Signature ===" -ForegroundColor Cyan
$sig = Get-AuthenticodeSignature -FilePath $ExecutablePath
$sig.SignerCertificate | Select-Object Subject, Issuer, NotBefore, NotAfter |
    Format-List
```

```title="Resultat attendu (7-Zip 23.01 x64)"
=== AppLocker Publisher Info ===
PublisherName  : O=Igor Pavlov, L=, S=, C=
ProductName    : 7-Zip
BinaryName     : 7z.exe
LowSection     : 0.0.0.0
HighSection    : *

=== Authenticode Signature ===
Subject   : CN=Igor Pavlov, O=Igor Pavlov, L=, S=, C=
Issuer    : CN=DigiCert Trusted G4 Code Signing RSA4096 SHA384 2021 CA1, ...
NotBefore : 01/01/2023 00:00:00
NotAfter  : 01/01/2026 00:00:00
```

:material-alert-outline: **7-Zip est signe depuis la version 22.x.** Les versions anterieures (21.x et avant) ne sont pas signees — vous ne pouvez pas creer de regle Publisher pour elles. Mettez a jour vers une version signee avant de creer la regle.

### Creer une regle Publisher via GPMC

Chemin GPMC :

```
Configuration ordinateur
  └── Parametres Windows
        └── Parametres de securite
              └── Strategies de controle des applications
                    └── AppLocker
                          └── Regles de fichiers executables
```

Procedure dans GPMC :

1. Clic droit sur **Regles de fichiers executables** > **Creer une nouvelle regle**
2. Action : **Autoriser**
3. Utilisateur ou groupe : `Everyone` (ou le groupe cible)
4. Condition : **Editeur**
5. Cliquez **Parcourir** et selectionnez le fichier exe signe
6. GPMC remplit automatiquement les champs Publisher, Product name, File name, Version

Configurez les niveaux de granularite de la regle :

| Niveau de restriction | Regle valide si... |
|---|---|
| Editeur seulement (`O=Igor Pavlov`) | N'importe quel produit de cet editeur |
| Editeur + Produit (`7-Zip`) | N'importe quelle version de 7-Zip par cet editeur |
| Editeur + Produit + Fichier (`7z.exe`) | Ce fichier specifique de ce produit |
| Editeur + Produit + Fichier + Version (`>= 23.0`) | Ce fichier, version minimale exigee |

**Recommandation production :** choisissez **Editeur + Produit** comme niveau de restriction. Plus restrictif que "editeur seul" (pas tous les produits de l'editeur), plus stable que "fichier + version" (survit aux mises a jour).

### Creer la regle Publisher via PowerShell

```powershell title="New-AppLockerPublisherRule.ps1"
# Create an AppLocker Publisher rule for a signed third-party application
# Requires: Windows RSAT AppLocker feature, or run on a machine with AppLocker configured
param(
    [Parameter(Mandatory)] [string]$ExecutablePath,
    [Parameter(Mandatory)] [string]$RuleDescription,
    [string]$UserOrGroup = "Everyone"
)

# Extract publisher info
$fileInfo = Get-AppLockerFileInformation -Path $ExecutablePath

# Generate Publisher rule at product level (Publisher + ProductName, any version)
$rule = New-AppLockerPolicy -FileInformation $fileInfo `
            -RuleType Publisher `
            -User $UserOrGroup `
            -Optimize

Write-Host "Generated AppLocker rule XML:" -ForegroundColor Cyan
$rule.RuleCollections | Where-Object { $_.Count -gt 0 } |
    ForEach-Object { $_.GetXml() } | Format-List

# Export to file for import into GPO
$outputPath = Join-Path $PSScriptRoot "AppLocker-Rule-$((Split-Path $ExecutablePath -Leaf) -replace '\.exe').xml"
$rule.ToXml() | Out-File -FilePath $outputPath -Encoding UTF8
Write-Host "Rule exported to: $outputPath" -ForegroundColor Green
```

```title="Resultat attendu"
Generated AppLocker rule XML:
...
<FilePublisherRule Id="..." Name="7-Zip" ...>
  <Conditions>
    <FilePublisherCondition PublisherName="O=Igor Pavlov"
                            ProductName="7-Zip"
                            BinaryName="*"
                            FileVersion="0.0.0.0">
    </FilePublisherCondition>
  </Conditions>
</FilePublisherRule>

Rule exported to: C:\Scripts\AppLocker-Rule-7z.xml
```

Importez ensuite le XML dans la GPO :

```
AppLocker > Regles de fichiers executables > Clic droit > Importer la strategie
```

### Tester la regle en mode Audit avant enforcement

**Ne jamais activer AppLocker en mode Enforce sans passer par Audit.**

```
GPO > AppLocker > Configurer l'application de regles
  Fichiers executables : Auditer uniquement
```

En mode Audit, les applications non couvertes par une regle sont autorisees mais un evenement est cree dans l'observateur d'evenements :

```
Journal : Microsoft-Windows-AppLocker/EXE and DLL
Event ID: 8003   (application autorisee mais aurait ete bloquee)
Event ID: 8002   (application autorisee par une regle)
```

```powershell title="Get-AppLockerAuditEvents.ps1"
# Retrieve AppLocker audit events to identify uncovered executables
$events = Get-WinEvent -LogName "Microsoft-Windows-AppLocker/EXE and DLL" -ErrorAction SilentlyContinue |
    Where-Object { $_.Id -eq 8003 }

if (-not $events) {
    Write-Host "No audit violations found — all executables are covered." -ForegroundColor Green
} else {
    Write-Host "Executables that would be BLOCKED in Enforce mode:" -ForegroundColor Yellow
    $events | ForEach-Object {
        [xml]$xml  = $_.ToXml()
        $filePath  = ($xml.Event.UserData.RuleAndFileData.FilePath)
        $publisher = ($xml.Event.UserData.RuleAndFileData.PublisherName)
        Write-Host "  PATH: $filePath | PUBLISHER: $publisher"
    } | Sort-Object -Unique
}
```

!!! warning "A surveiller"
    AppLocker ne s'applique pas aux processus deja lances au moment de l'activation de la politique. Un reboot est necessaire apres passage de Audit a Enforce pour garantir l'application complete.

!!! quote "En resume"
    - Les regles Publisher AppLocker reposent sur le certificat editeur — elles survivent aux mises a jour contrairement aux regles Hash
    - Utilisez `Get-AppLockerFileInformation` pour extraire les informations Publisher avant de creer la regle
    - Le niveau recommande est **Editeur + Produit** : stable et pas trop permissif
    - Deployez toujours en mode **Audit** en premier — au moins une semaine — avant de passer en Enforce
    - 7-Zip est signe depuis la version 22.x — verifiez avant de creer une regle Publisher

---

## :material-bomb: Piege de production : GPP Replace ecrase les modifications utilisateur

### Le scenario

Vous deployez `deployment.properties` pour Java via GPP File avec l'action **Replace**. Un utilisateur (ou son equipe applicative) a besoin d'ajouter une URL dans `exception.sites` pour une application specifique. Il la modifie manuellement.

Au prochain cycle GPO (toutes les 90-120 minutes environ), GPP ecrase le fichier avec la version du SYSVOL. La modification de l'utilisateur est perdue — silencieusement. L'application cesse de fonctionner. L'utilisateur appelle le support.

Ce scenario se reproduit pour n'importe quel fichier deploye par GPP File en action Replace, mais aussi pour les cles GPP Registry en action Replace quand une application permet a l'utilisateur de modifier sa configuration.

### Pourquoi "Replace" fait ca

L'action **Replace** dans GPP signifie : "Supprimer l'element s'il existe, recreer depuis la source". Elle est executee a chaque rafraichissement GPO — pas seulement lors de la premiere application.

Comparaison des actions GPP pour File et Registry :

| Action GPP | Comportement |
|---|---|
| `Create` | Cree uniquement si absent — ne touche pas si existant |
| `Update` | Cree si absent, modifie les cles/valeurs specifiees, laisse les autres intactes |
| `Replace` | Ecrase completement a chaque cycle |
| `Delete` | Supprime l'element |

### La solution correcte

**Pour les fichiers de configuration partages :** utilisez deux fichiers distincts.

- `deployment.properties` : contenu entierement gere par GPP (action Replace — OK car c'est un fichier de politique pure)
- `exception.sites` : pointe vers un fichier que vous gerez de maniere centralisee dans le SYSVOL, et que vous mettez a jour explicitement quand une nouvelle URL est approuvee

En pratique, le processus devient :

```
1. Utilisateur soumet une demande d'exception Java pour URL X
2. Admin ajoute l'URL dans \\domain\NETLOGON\java\exception.sites
3. GPP distribue la version mise a jour lors du prochain cycle
4. Aucune modification locale n'est necessaire — et aucune modification locale n'est possible
```

**Pour les cles GPP Registry :** utilisez l'action **Update** plutot que **Replace**.

L'action Update ecrit ou met a jour uniquement les valeurs nommees dans l'item GPP. Elle ne supprime pas les autres valeurs presentes sous la meme cle. Si l'application cree d'autres valeurs que l'admin n'a pas listes dans GPP, elles ne sont pas touchees.

!!! danger "Production"
    Si vous avez besoin de supprimer des valeurs specifiques en plus d'en ecrire, utilisez un script PowerShell dans une GPO Preferences (Computer Configuration > Preferences > Control Panel Settings > Scheduled Tasks, ou un script de demarrage) plutot que d'utiliser Replace. Ce script peut supprimer precisement les cles non desirees sans toucher aux autres.

### Exemple : GPP Registry avec action Update pour 7-Zip

Au lieu de :

```
Action : Replace
Key    : HKCU\Software\7-Zip\Options
(ecrase toutes les valeurs — incluant celles que 7-Zip pourrait avoir crees depuis la derniere GPO)
```

Utilisez :

```
Action     : Update
Key        : HKCU\Software\7-Zip\Options
Value name : Lang
Value data : fr
```

Creez un item separe pour chaque valeur a gerer. L'action Update sur chaque item individuel ecrit uniquement cette valeur — les autres valeurs de la cle `Options` restent intactes.

!!! quote "En resume"
    - L'action GPP **Replace** s'execute a chaque cycle GPO — elle ecrase toujours, meme les modifications utilisateur legitimes
    - Preferez l'action **Update** pour les cles de registre : elle ecrit uniquement les valeurs listees, laisse les autres intactes
    - Pour les fichiers de configuration, gerez les sections "politiques" et les sections "configurables par exception" dans des fichiers distincts
    - La gouvernance autour des exceptions (qui peut demander, qui approuve, qui met a jour le fichier source) est aussi importante que la technique

---

## :material-link-variant: References croisees

| Sujet | Chapitre |
|---|---|
| GPP Registry — concept et structure | [../bible-gpo/11-preferences-gpp.md](../bible-gpo/11-preferences-gpp.md) |
| AppLocker et WDAC en detail | [../bible-gpo/15-applocker-wdac.md](../bible-gpo/15-applocker-wdac.md) |
| Meme approche ADMX pour les apps Microsoft | [08-m365-office.md](08-m365-office.md) |

---

## :material-clipboard-check-outline: Checklist operationnelle

Avant de considerer la GPO "apps tierces" comme prete pour la production :

- [ ] Adobe : ADMX charges dans le Central Store OU cles GPP Registry deployees sous `HKLM\SOFTWARE\Policies\Adobe\`
- [ ] Adobe : JavaScript desactive, Protected Mode actif, mises a jour desactivees
- [ ] Java : `deployment.properties` present sous `C:\Windows\Sun\Java\Deployment\`
- [ ] Java : `exception.sites` deploy en meme temps (meme vide)
- [ ] Java : cles critiques verrouillee avec la syntaxe `.locked`
- [ ] ProcMon utilise pour identifier les cles des apps sans ADMX avant toute tentative de GPP
- [ ] GPP Registry : action **Update** (pas Replace) pour les cles d'apps tierces
- [ ] AppLocker : regles Publisher creees a partir d'executables signes, verifiez la signature avant
- [ ] AppLocker : deploiement en mode **Audit** au moins 5 jours ouvres avant passage en Enforce
- [ ] AppLocker : audit des evenements 8003 valide (aucune exclusion necessaire non identifiee)
