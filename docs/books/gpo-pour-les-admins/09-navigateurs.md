---
description: Navigateurs Edge, Chrome, Firefox via GPO — ADMX, sécurité, extensions, proxy, homepage
tags: [gpo-admins, edge, chrome, firefox, navigateurs, admx]
---

# Navigateurs (Edge, Chrome, Firefox) via GPO

!!! abstract "Ce que vous allez apprendre"
    - Deployer les ADMX officiels d'Edge, Chrome et Firefox dans le Central Store et configurer les parametres de securite critiques
    - Forcer la page d'accueil, desactiver le gestionnaire de mots de passe et le mode InPrivate/Incognito en entreprise
    - Gerer les extensions : liste d'autorisation, liste de blocage et installation silencieuse par navigateur
    - Verifier depuis PowerShell que les politiques sont bien appliquees sur les postes cibles
    - Identifier et eviter les pieges de production les plus frequents (extension ID incorrect, Chrome non managé, Firefox sans ADMX)

!!! tip "Si vous ne retenez qu'une chose"
    Les trois navigateurs ecrivent leurs politiques GPO dans le registre sous `HKLM\SOFTWARE\Policies\`. Edge utilise `Microsoft\Edge\`, Chrome utilise `Google\Chrome\`, Firefox utilise `Mozilla\Firefox\` (avec ADMX communautaire) ou lit directement `C:\Program Files\Mozilla Firefox\distribution\policies.json`. Comprendre cette structure suffit pour diagnostiquer n'importe quel probleme de politique navigateur.

---

## :material-microsoft-edge: Microsoft Edge

### Obtenir les ADMX officiels

Microsoft distribue les modeles ADMX d'Edge dans un package dedie, distinct des modeles Windows. Ils ne sont **pas** inclus dans le package "Administrative Templates for Windows".

Telechargez depuis [Microsoft Edge for Business](https://www.microsoft.com/en-us/edge/business/download). Selectionnez "Policy Files" et choisissez la version stable la plus recente.

:material-information-outline: Le package contient deux fichiers ADMX critiques : `msedge.admx` (parametres du navigateur) et `msedgeupdate.admx` (gestion des mises a jour). Les deux sont necessaires. Les ADML correspondants se trouvent dans les sous-dossiers de langue (`en-US/`, `fr-FR/`).

Copiez les fichiers dans le Central Store. Consultez [04-central-store.md](04-central-store.md) pour la procedure complete.

```powershell title="Verification de la presence des ADMX Edge dans le Central Store"
# Verify Edge ADMX files are present in the Central Store
$domain       = (Get-ADDomain).DNSRoot
$centralStore = "\\$domain\SYSVOL\$domain\Policies\PolicyDefinitions"

$required = @(
    "msedge.admx",
    "msedgeupdate.admx"
)

foreach ($file in $required) {
    $path   = Join-Path $centralStore $file
    $status = if (Test-Path $path) { "OK      " } else { "MISSING " }
    Write-Host "[$status] $file"
}
```

```title="Resultat attendu"
[OK      ] msedge.admx
[OK      ] msedgeupdate.admx
```

### Chemin de base dans le registre

Toutes les politiques Edge ecrites par GPO atterrissent sous :

```
HKLM\SOFTWARE\Policies\Microsoft\Edge\
```

Les politiques utilisateur (Computer vs User Configuration dans GPMC) peuvent aussi ecrire sous `HKCU\SOFTWARE\Policies\Microsoft\Edge\`. Preferez toujours la configuration **ordinateur** (`HKLM`) en entreprise — elle s'applique quel que soit l'utilisateur connecte.

### Chemin GPMC

Dans l'editeur de strategie de groupe, les parametres Edge se trouvent sous :

```
Configuration ordinateur
  └── Modeles d'administration
        └── Microsoft Edge
```

et

```
Configuration ordinateur
  └── Modeles d'administration
        └── Microsoft Edge Update
```

### Parametres de securite critiques

Les parametres suivants sont a activer en priorite dans toute GPO de durcissement Edge entreprise.

**SmartScreen** bloque les sites de phishing et les telechargements malveillants. Activez les deux entrees.

| Parametre GPMC | Valeur | Cle de registre |
|---|---|---|
| Configurer Microsoft Defender SmartScreen | Activer | `HKLM\...\Edge\SmartScreenEnabled` = `1` (DWORD) |
| Empecher le contournement des avertissements SmartScreen | Activer | `HKLM\...\Edge\PreventSmartScreenPromptOverride` = `1` |
| Empecher le contournement SmartScreen pour les fichiers | Activer | `HKLM\...\Edge\PreventSmartScreenPromptOverrideForFiles` = `1` |

**Isolation des sites** — activez `SitePerProcess`. Ce parametre cree un processus de rendu distinct pour chaque site, ce qui limite la propagation d'une exploitation de faille.

| Parametre GPMC | Valeur | Cle de registre |
|---|---|---|
| Activer l'isolation des sites pour chaque site | Activer | `HKLM\...\Edge\SitePerProcess` = `1` |

**Gestionnaire de mots de passe** — desactivez le stockage de mots de passe dans le navigateur en entreprise. Les mots de passe doivent etre dans un coffre-fort d'entreprise.

| Parametre GPMC | Valeur | Cle de registre |
|---|---|---|
| Activer l'enregistrement des mots de passe dans le Gestionnaire de mots de passe | Desactiver | `HKLM\...\Edge\PasswordManagerEnabled` = `0` |

**Mode InPrivate** — desactivez la navigation InPrivate pour eviter le contournement des controles de proxy et de journalisation.

| Parametre GPMC | Valeur | Cle de registre |
|---|---|---|
| Disponibilite du mode InPrivate | Desactiver (valeur 2) | `HKLM\...\Edge\InPrivateModeAvailability` = `2` |

:material-alert-outline: La valeur `2` signifie "Force disable" — l'icone disparait et le raccourci `Ctrl+Shift+N` ne fonctionne plus. La valeur `1` signifie "Force enable". La valeur `0` (defaut) laisse le choix a l'utilisateur.

### Page d'accueil et URL de demarrage

Deux mecanismes distincts coexistent dans Edge : la **page d'accueil** (bouton Home) et les **onglets de demarrage** (ouverture d'Edge). Configurez les deux.

Chemin GPMC : `Configuration ordinateur > Modeles d'administration > Microsoft Edge`

| Parametre | Valeur a configurer | Cle de registre |
|---|---|---|
| Configurer l'URL de la page d'accueil | URL intranet (ex: `https://intranet.contoso.com`) | `HKLM\...\Edge\HomepageLocation` |
| Activer le bouton de page d'accueil sur la barre d'outils | Activer | `HKLM\...\Edge\ShowHomeButton` = `1` |
| Action au demarrage | Ouvrir une liste de URLs (valeur 4) | `HKLM\...\Edge\RestoreOnStartup` = `4` |
| Pages a ouvrir au demarrage | URL intranet | `HKLM\...\Edge\RestoreOnStartupURLs\1` |

!!! quote "En resume"
    Edge stocke toutes ses politiques sous `HKLM\SOFTWARE\Policies\Microsoft\Edge\`. Les ADMX officiels (`msedge.admx`, `msedgeupdate.admx`) s'ajoutent au Central Store manuellement — ils ne font pas partie du pack Windows. Activez en priorite SmartScreen (avec blocage du contournement), SitePerProcess, et desactivez le gestionnaire de mots de passe et le mode InPrivate.

---

## :material-google-chrome: Google Chrome

### Obtenir les ADMX officiels

Google publie les modeles ADMX Chrome dans le package "Chrome Browser Cloud Management". Telechargez depuis [Google Chrome Enterprise](https://chromeenterprise.google/browser/download/).

Le fichier principal est `chrome.admx`, accompagne de son ADML. Copiez-les dans le Central Store de la meme maniere qu'Edge.

:material-information-outline: Contrairement a Edge, Chrome n'a pas de fichier ADMX separe pour les mises a jour dans le pack de base. La gestion des mises a jour se fait via Google Update (package separe) ou via Chrome Browser Cloud Management.

### Chemin de base dans le registre

```
HKLM\SOFTWARE\Policies\Google\Chrome\
```

La philosophie est identique a Edge : configuration ordinateur via `HKLM`, valeurs nommees identiques ou tres proches. Chrome a ete le modele qu'Edge a suivi lors de sa conception.

### Chemin GPMC

```
Configuration ordinateur
  └── Modeles d'administration
        └── Google
              └── Google Chrome
```

### Parametres de securite critiques

| Parametre GPMC | Valeur | Cle de registre |
|---|---|---|
| Activer Google Safe Browsing | Activer | `HKLM\...\Chrome\SafeBrowsingEnabled` = `1` |
| Empecher de contourner les avertissements Safe Browsing | Activer | `HKLM\...\Chrome\SafeBrowsingProceduralWarningEnabled` = `1` |
| Activer l'isolation des sites pour chaque site | Activer | `HKLM\...\Chrome\SitePerProcess` = `1` |
| Desactiver le gestionnaire de mots de passe | Desactiver | `HKLM\...\Chrome\PasswordManagerEnabled` = `0` |
| Mode navigation privee | Desactiver (valeur 2) | `HKLM\...\Chrome\IncognitoModeAvailability` = `2` |

:material-check-circle-outline: Chrome et Edge partagent la meme logique de valeurs pour `IncognitoModeAvailability` / `InPrivateModeAvailability` : 0 = autorise, 1 = force, 2 = desactive. Ce parallelisme facilite les audits comparatifs.

### Page d'accueil et URL de demarrage

| Parametre GPMC | Valeur | Cle de registre |
|---|---|---|
| Configurer l'URL de la page d'accueil | URL intranet | `HKLM\...\Chrome\HomepageLocation` |
| Afficher le bouton Accueil | Activer | `HKLM\...\Chrome\ShowHomeButton` = `1` |
| Action au demarrage | Ouvrir une liste de URLs (valeur 4) | `HKLM\...\Chrome\RestoreOnStartup` = `4` |
| URLs a ouvrir au demarrage | URL intranet | `HKLM\...\Chrome\RestoreOnStartupURLs\1` |

!!! quote "En resume"
    Chrome suit exactement la meme philosophie qu'Edge. Les noms de cles de registre sont souvent identiques, seul le chemin parent change (`Google\Chrome` vs `Microsoft\Edge`). Activez Safe Browsing, SitePerProcess, desactivez le gestionnaire de mots de passe et le mode incognito.

---

## :material-firefox: Firefox

### Deux approches, une seule recommandation

Firefox n'est pas un produit Microsoft — Mozilla gere ses propres outils d'administration. Deux approches coexistent en entreprise.

**Approche 1 : ADMX communautaire** (mozilla.admx). Des ADMX non officiels existent, maintenus par la communaute. Ils permettent de configurer Firefox via GPMC exactement comme Edge ou Chrome. Les parametres atterrissent sous `HKLM\SOFTWARE\Policies\Mozilla\Firefox\`.

**Approche 2 : policies.json** (recommandee par Mozilla). Firefox lit nativement un fichier JSON deposé dans `C:\Program Files\Mozilla Firefox\distribution\policies.json`. C'est l'approche officielle Mozilla, independante des ADMX.

:material-star-circle-outline: En production, preferez `policies.json` pour Firefox. Le fichier est plus simple a maintenir, plus lisible, et ne depend pas d'un tiers pour les ADMX. Deployer le fichier via GPP (Group Policy Preferences > Files) est la methode standard.

### Structure de policies.json

Le fichier `policies.json` est un objet JSON avec une cle racine `policies`. Voici un exemple minimal de durcissement.

```json title="C:\\Program Files\\Mozilla Firefox\\distribution\\policies.json"
{
  "policies": {
    "DisableMasterPasswordCreation": false,
    "OfferToSaveLogins": false,
    "PasswordManagerEnabled": false,
    "PrivateBrowsingModeAvailability": 2,
    "DisablePrivateBrowsing": true,
    "Homepage": {
      "URL": "https://intranet.contoso.com",
      "Locked": true,
      "StartPage": "homepage"
    },
    "SmartScreen": true,
    "Proxy": {
      "Mode": "system"
    },
    "BlockAboutConfig": true,
    "BlockAboutSupport": false
  }
}
```

:material-alert-outline: Le champ `DisablePrivateBrowsing` est booleen. La valeur `true` desactive entierement le mode navigation privee. Pas de valeur numerique ici — c'est une difference importante par rapport a Chrome et Edge.

### Deployer policies.json via GPP Files

Dans GPMC, utilisez Group Policy Preferences > Files pour deployer `policies.json` sur tous les postes.

Chemin GPMC :

```
Configuration ordinateur
  └── Preferences
        └── Parametres Windows
              └── Fichiers
```

Creez une nouvelle entree de fichier :

| Champ | Valeur |
|---|---|
| Action | Remplacer |
| Fichier source | `\\contoso.com\NETLOGON\Firefox\policies.json` |
| Fichier de destination | `C:\Program Files\Mozilla Firefox\distribution\policies.json` |
| Creer le dossier parent | Oui |

:material-information-outline: Stockez le fichier `policies.json` dans `NETLOGON` ou dans un partage DFS dedié aux ressources GPP. Cela garantit la replication sur tous les DC et la disponibilite depuis n'importe quel site.

!!! warning "A surveiller — chemin de distribution"
    Si Firefox est installe dans un chemin non standard (ex: `C:\Program Files (x86)\Mozilla Firefox\`), la politique ne s'applique pas. Verifiez le chemin reel d'installation avant de deployer. Vous pouvez utiliser GPP Files avec une variable d'environnement si les chemins sont heterogenes : `%ProgramFiles%\Mozilla Firefox\distribution\policies.json` — mais testez que GPP resout correctement la variable sur les postes 32 bits.

!!! quote "En resume"
    Firefox utilise `policies.json` comme meccanisme natif de politique. Deployer ce fichier via GPP Files est la methode la plus robuste — pas de dependance a des ADMX tiers, lisibilite maximale. Stockez le fichier source dans NETLOGON pour la replication automatique.

---

## :material-table-compare: Tableau comparatif des trois navigateurs

Ce tableau recense les parametres communs et leur equivalent dans chaque navigateur.

### Proxy

| Parametre | Edge | Chrome | Firefox (policies.json) |
|---|---|---|---|
| Utiliser les parametres systeme | `HKLM\...\Edge\ProxyMode` = `"system"` | `HKLM\...\Chrome\ProxyMode` = `"system"` | `"Proxy": {"Mode": "system"}` |
| Proxy fixe | `ProxyMode` = `"fixed_servers"` + `ProxyServer` | Identique | `"Mode": "manual"` + `"HTTPProxy"` |
| Script PAC | `ProxyMode` = `"pac_script"` + `ProxyPacUrl` | Identique | `"Mode": "pac_script"` + `"AutoConfigURL"` |
| Desactiver le proxy | `ProxyMode` = `"direct"` | Identique | `"Mode": "none"` |

### Page d'accueil

| Parametre | Edge (registre) | Chrome (registre) | Firefox (policies.json) |
|---|---|---|---|
| URL | `HomepageLocation` | `HomepageLocation` | `"Homepage": {"URL": "..."}` |
| Verrouillage | `HomepageIsNewTabPage` = `0` | `HomepageIsNewTabPage` = `0` | `"Homepage": {"Locked": true}` |
| Bouton Home visible | `ShowHomeButton` = `1` | `ShowHomeButton` = `1` | N/A (toujours visible si Homepage defini) |

### Gestion des extensions

| Parametre | Edge (registre) | Chrome (registre) | Firefox (policies.json) |
|---|---|---|---|
| Liste de blocage | `ExtensionInstallBlocklist\` | `ExtensionInstallBlocklist\` | `"ExtensionSettings": {"*": {"blocked": true}}` |
| Liste d'autorisation | `ExtensionInstallAllowlist\` | `ExtensionInstallAllowlist\` | `"ExtensionSettings": {"id": {"installation_mode": "allowed"}}` |
| Installation forcee | `ExtensionInstallForcelist\` | `ExtensionInstallForcelist\` | `"Extensions": {"Install": ["url"]}` |

### Mises a jour

| Parametre | Edge | Chrome | Firefox |
|---|---|---|---|
| Desactiver les MAJ auto | `msedgeupdate.admx` → `Update{GUID}` | Google Update ADMX | `"DisableAppUpdate": true` dans policies.json |
| Canal de mise a jour | `msedgeupdate.admx` → `TargetChannel` | Google Update ADMX | Non applicable (canal fixe par MSI) |

### Mots de passe

| Parametre | Edge (registre) | Chrome (registre) | Firefox (policies.json) |
|---|---|---|---|
| Desactiver le gestionnaire | `PasswordManagerEnabled` = `0` | `PasswordManagerEnabled` = `0` | `"PasswordManagerEnabled": false` |

### Navigation privee / incognito

| Parametre | Edge (registre) | Chrome (registre) | Firefox (policies.json) |
|---|---|---|---|
| Desactiver | `InPrivateModeAvailability` = `2` | `IncognitoModeAvailability` = `2` | `"DisablePrivateBrowsing": true` |

---

## :material-puzzle-plus: Gestion des extensions

### Edge — liste de blocage, liste d'autorisation, installation forcee

Les politiques d'extension Edge s'ecrivent dans des sous-cles de registre numerotees (valeurs `1`, `2`, `3`...).

**Bloquer toutes les extensions sauf la liste d'autorisation** — c'est le modele recommande en entreprise.

Chemin GPMC : `Configuration ordinateur > Modeles d'administration > Microsoft Edge > Extensions`

| Parametre | Action | Registre |
|---|---|---|
| Bloquer l'installation des extensions | Activer avec valeur `*` | `HKLM\...\Edge\ExtensionInstallBlocklist\1` = `*` |
| Autoriser des extensions specifiques | Liste d'IDs | `HKLM\...\Edge\ExtensionInstallAllowlist\1` = `<extension-id>` |
| Forcer l'installation d'extensions | ID + URL update | `HKLM\...\Edge\ExtensionInstallForcelist\1` = `<id>;<update_url>` |

L'URL de mise a jour pour les extensions du Microsoft Edge Add-ons Store est :

```
https://edge.microsoft.com/extensionwebstorebase/v1/crx
```

Exemple pour forcer l'extension "Microsoft Editor" (ID : `gpaiobkfhnonedkhhfjpmhdalgeoebfa`) :

```
gpaiobkfhnonedkhhfjpmhdalgeoebfa;https://edge.microsoft.com/extensionwebstorebase/v1/crx
```

### Chrome — meme logique, URL differente

Le mecanisme est identique pour Chrome. Seul le chemin de registre et l'URL de mise a jour changent.

Chemin GPMC : `Configuration ordinateur > Modeles d'administration > Google > Google Chrome > Extensions`

L'URL de mise a jour pour le Chrome Web Store est :

```
https://clients2.google.com/service/update2/crx
```

Exemple pour forcer l'extension "uBlock Origin" (ID : `cjpalhdlnbpafiamejdnhcphjbkeiagm`) :

```
cjpalhdlnbpafiamejdnhcphjbkeiagm;https://clients2.google.com/service/update2/crx
```

Registre resultant :

```
HKLM\SOFTWARE\Policies\Google\Chrome\ExtensionInstallForcelist\1
= "cjpalhdlnbpafiamejdnhcphjbkeiagm;https://clients2.google.com/service/update2/crx"
```

### Firefox — via policies.json

Firefox gere les extensions via la cle `ExtensionSettings` dans `policies.json`. La syntaxe est plus expressive — elle combine blocage, autorisation et installation forcee en un seul objet.

```json title="Extrait policies.json — gestion des extensions Firefox"
{
  "policies": {
    "ExtensionSettings": {
      "*": {
        "installation_mode": "blocked"
      },
      "uBlock0@raymondhill.net": {
        "installation_mode": "force_installed",
        "install_url": "https://addons.mozilla.org/firefox/downloads/latest/ublock-origin/latest.xpi"
      },
      "firefox@dashlane.com": {
        "installation_mode": "allowed",
        "install_url": "https://addons.mozilla.org/firefox/downloads/latest/dashlane/latest.xpi"
      }
    }
  }
}
```

:material-information-outline: L'ID d'une extension Firefox est son **email-like ID** (ex: `uBlock0@raymondhill.net`), visible dans la page `about:debugging` ou dans le manifest de l'extension. Ce n'est pas un hash hexadecimal comme pour Chrome/Edge.

!!! danger "Production — Extension ID incorrect : echec silencieux"
    Si l'ID d'extension dans `ExtensionInstallForcelist` (Chrome/Edge) ou `ExtensionSettings` (Firefox) est errone, le navigateur ne signale **aucune erreur**. La politique est appliquee, la valeur de registre existe, mais l'extension n'est jamais installee.

    Verifiez toujours l'ID en inspectant directement la page de l'extension dans son store respectif : l'ID est dans l'URL. Pour Chrome : `https://chrome.google.com/webstore/detail/<nom>/<ID>`. Pour Edge : `https://microsoftedge.microsoft.com/addons/detail/<nom>/<ID>`. Pour Firefox : l'ID est dans l'onglet "Plus d'informations" de la page addons.mozilla.org.

    Testez toujours sur un poste pilote avant de deployer a grande echelle.

!!! quote "En resume"
    Edge et Chrome partagent la meme mecanique d'extension via le registre. Firefox utilise `ExtensionSettings` dans `policies.json`. Dans tous les cas, commencez par bloquer tout (`*` = blocked), puis autorisez ou forcez des IDs specifiques. Un ID incorrect produit un echec silencieux — toujours verifier depuis le store officiel.

---

## :material-shield-check-outline: Verification post-deploiement

### Verifier les politiques appliquees sur un poste cible

PowerShell peut inspecter directement les cles de registre pour confirmer l'application des politiques. Ce script est concu pour s'executer en remote via `Invoke-Command`.

```powershell title="Verification des politiques navigateurs sur un poste cible"
# Check browser policies applied on a remote workstation
param(
    [Parameter(Mandatory)]
    [string]$ComputerName
)

Invoke-Command -ComputerName $ComputerName -ScriptBlock {

    $results = @()

    # Helper function to read a registry value safely
    function Get-RegValue {
        param([string]$Path, [string]$Name)
        try {
            $val = Get-ItemProperty -Path $Path -Name $Name -ErrorAction Stop
            return $val.$Name
        } catch {
            return $null
        }
    }

    # --- Microsoft Edge ---
    $edgeBase = "HKLM:\SOFTWARE\Policies\Microsoft\Edge"
    $results += [PSCustomObject]@{
        Browser  = "Edge"
        Setting  = "SmartScreenEnabled"
        Value    = Get-RegValue $edgeBase "SmartScreenEnabled"
        Expected = 1
    }
    $results += [PSCustomObject]@{
        Browser  = "Edge"
        Setting  = "PasswordManagerEnabled"
        Value    = Get-RegValue $edgeBase "PasswordManagerEnabled"
        Expected = 0
    }
    $results += [PSCustomObject]@{
        Browser  = "Edge"
        Setting  = "InPrivateModeAvailability"
        Value    = Get-RegValue $edgeBase "InPrivateModeAvailability"
        Expected = 2
    }
    $results += [PSCustomObject]@{
        Browser  = "Edge"
        Setting  = "SitePerProcess"
        Value    = Get-RegValue $edgeBase "SitePerProcess"
        Expected = 1
    }

    # --- Google Chrome ---
    $chromeBase = "HKLM:\SOFTWARE\Policies\Google\Chrome"
    $results += [PSCustomObject]@{
        Browser  = "Chrome"
        Setting  = "SafeBrowsingEnabled"
        Value    = Get-RegValue $chromeBase "SafeBrowsingEnabled"
        Expected = 1
    }
    $results += [PSCustomObject]@{
        Browser  = "Chrome"
        Setting  = "PasswordManagerEnabled"
        Value    = Get-RegValue $chromeBase "PasswordManagerEnabled"
        Expected = 0
    }
    $results += [PSCustomObject]@{
        Browser  = "Chrome"
        Setting  = "IncognitoModeAvailability"
        Value    = Get-RegValue $chromeBase "IncognitoModeAvailability"
        Expected = 2
    }

    # --- Firefox policies.json ---
    $ffPolicy = "C:\Program Files\Mozilla Firefox\distribution\policies.json"
    $ffExists = Test-Path $ffPolicy
    $results += [PSCustomObject]@{
        Browser  = "Firefox"
        Setting  = "policies.json present"
        Value    = if ($ffExists) { 1 } else { 0 }
        Expected = 1
    }

    # Display results with pass/fail indicator
    $results | ForEach-Object {
        $pass   = ($null -ne $_.Value) -and ($_.Value -eq $_.Expected)
        $status = if ($pass) { "PASS" } else { "FAIL" }
        $color  = if ($pass) { "Green" } else { "Red" }
        Write-Host ("[$status] {0,-10} {1,-35} Value={2} Expected={3}" -f `
            $_.Browser, $_.Setting, $_.Value, $_.Expected) -ForegroundColor $color
    }
}
```

```title="Resultat attendu (toutes politiques appliquees)"
[PASS] Edge       SmartScreenEnabled              Value=1 Expected=1
[PASS] Edge       PasswordManagerEnabled           Value=0 Expected=0
[PASS] Edge       InPrivateModeAvailability        Value=2 Expected=2
[PASS] Edge       SitePerProcess                   Value=1 Expected=1
[PASS] Chrome     SafeBrowsingEnabled              Value=1 Expected=1
[PASS] Chrome     PasswordManagerEnabled           Value=0 Expected=0
[PASS] Chrome     IncognitoModeAvailability        Value=2 Expected=2
[PASS] Firefox    policies.json present            Value=1 Expected=1
```

### Verifier les extensions forcees

Ce script verifie que les extensions configurees en `ExtensionInstallForcelist` sont bien presentes dans le registre du poste cible.

```powershell title="Verification des extensions forcees Edge et Chrome"
# Verify force-installed extension registry entries on a remote workstation
param(
    [Parameter(Mandatory)]
    [string]$ComputerName
)

Invoke-Command -ComputerName $ComputerName -ScriptBlock {

    Write-Host "`n=== Edge ExtensionInstallForcelist ===" -ForegroundColor Cyan
    $edgePath = "HKLM:\SOFTWARE\Policies\Microsoft\Edge\ExtensionInstallForcelist"
    if (Test-Path $edgePath) {
        Get-ItemProperty $edgePath | Select-Object -Property * -ExcludeProperty PS* |
            Get-Member -MemberType NoteProperty | ForEach-Object {
                Write-Host "  [$($_.Name)] $(((Get-ItemProperty $edgePath).$($_.Name)).Split(';')[0])"
            }
    } else {
        Write-Host "  (no force-installed extensions configured)" -ForegroundColor Yellow
    }

    Write-Host "`n=== Chrome ExtensionInstallForcelist ===" -ForegroundColor Cyan
    $chromePath = "HKLM:\SOFTWARE\Policies\Google\Chrome\ExtensionInstallForcelist"
    if (Test-Path $chromePath) {
        Get-ItemProperty $chromePath | Select-Object -Property * -ExcludeProperty PS* |
            Get-Member -MemberType NoteProperty | ForEach-Object {
                Write-Host "  [$($_.Name)] $(((Get-ItemProperty $chromePath).$($_.Name)).Split(';')[0])"
            }
    } else {
        Write-Host "  (no force-installed extensions configured)" -ForegroundColor Yellow
    }
}
```

```title="Resultat attendu"
=== Edge ExtensionInstallForcelist ===
  [1] gpaiobkfhnonedkhhfjpmhdalgeoebfa

=== Chrome ExtensionInstallForcelist ===
  [1] cjpalhdlnbpafiamejdnhcphjbkeiagm
```

### Verifier la page d'accueil configuree

```powershell title="Verification de la page d'accueil Edge et Chrome"
# Check homepage policy on a remote workstation
param(
    [Parameter(Mandatory)]
    [string]$ComputerName
)

Invoke-Command -ComputerName $ComputerName -ScriptBlock {

    $edgeHome  = (Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Edge" `
                    -Name "HomepageLocation" -ErrorAction SilentlyContinue).HomepageLocation
    $chromeHome = (Get-ItemProperty "HKLM:\SOFTWARE\Policies\Google\Chrome" `
                    -Name "HomepageLocation" -ErrorAction SilentlyContinue).HomepageLocation

    Write-Host "Edge   HomepageLocation : $edgeHome"
    Write-Host "Chrome HomepageLocation : $chromeHome"
}
```

```title="Resultat attendu"
Edge   HomepageLocation : https://intranet.contoso.com
Chrome HomepageLocation : https://intranet.contoso.com
```

!!! quote "En resume"
    Toujours verifier les politiques apres deploiement. Le script de verification remote centralise les controles Edge, Chrome et Firefox en un seul appel. Un `FAIL` sur une valeur `null` signifie que la GPO ne s'est pas appliquee — verifiez le filtrage de securite et les liens (links) de la GPO. Un `FAIL` sur une valeur presente indique un conflit de GPO ou une politique locale plus specifique.

---

## :material-alert-circle: Pieges de production

### Chrome non manage — extension forcee silencieusement ignoree

C'est le piege le plus frequent sur les parcs heterogenes.

**Scenario** : vous creez une GPO avec `ExtensionInstallForcelist` pour Chrome. Vous la liez a l'OU des postes. Vous forcez `gpupdate /force`. Rien ne se passe.

**Cause** : Chrome installe via le MSI entreprise respecte les politiques GPO. Chrome installe via le setup utilisateur standard (installer du site google.com) ou via une mise a jour automatique non geree peut **ignorer les politiques de registre HKLM** selon sa version et son mode d'installation.

:material-alert-outline: Verifiez dans `chrome://policy` sur un poste cible. Si la colonne "Source" indique "Platform" pour vos politiques, elles sont lues. Si la page est vide ou que vos politiques n'y figurent pas, Chrome n'est pas en mode "managed".

**Diagnostic** :

```powershell title="Verification du mode managed Chrome"
# Check if Chrome is running in managed mode on the local machine
$chromePolicies = "HKLM:\SOFTWARE\Policies\Google\Chrome"
$chromeEnroll   = "HKLM:\SOFTWARE\Google\Chrome\BrowserEnrollment"

$policiesExist = Test-Path $chromePolicies
$enrollExist   = Test-Path $chromeEnroll

Write-Host "Policies registry key exists : $policiesExist"
Write-Host "Enrollment key exists        : $enrollExist"

# Check chrome://policy equivalent via GPO report
$chromeInstall = Get-ItemProperty "HKLM:\SOFTWARE\Google\Update\Clients\{8A69D345-D564-463C-AFF1-A69D9E530F96}" `
    -ErrorAction SilentlyContinue
Write-Host "Chrome installed version     : $($chromeInstall.pv)"
```

**Solution** : deployez Chrome via le MSI entreprise disponible sur [Google Chrome Enterprise](https://chromeenterprise.google/browser/download/). Ce MSI enregistre Chrome comme application geree et respecte les politiques HKLM.

!!! danger "Production — ne jamais deployer Chrome via le setup consommateur"
    Le setup `ChromeSetup.exe` du site grand public et le MSI entreprise produisent des installations fonctionnellement identiques pour l'utilisateur, mais le comportement vis-a-vis des GPO est different. Seul le MSI entreprise garantit la lecture des politiques HKLM. Auditez votre parc avec un inventaire SCCM/Intune pour identifier les postes en setup consommateur avant de deployer des politiques d'extension.

### Conflit de GPO navigateur

Si plusieurs GPO configurent le meme parametre navigateur (par exemple `HomepageLocation` a la fois dans une GPO de domaine et une GPO d'OU), la GPO la plus proche de l'objet cible l'emporte (principe de precedence GPO standard).

:material-information-outline: Utilisez `Resultant Set of Policy` (RSoP) ou `gpresult /h rapport.html` pour identifier quelle GPO applique effectivement chaque parametre navigateur.

### Firefox et la mise a jour silencieuse de policies.json

Firefox met a jour son dossier `Program Files` lors des mises a jour automatiques. Le dossier `distribution/` est preserve — mais si Firefox est desinstalle et reinstalle (scenario de migration), le dossier `distribution/` est supprime.

:material-alert-outline: Assurez-vous que la GPP Files qui deploie `policies.json` est configuree en action "Remplacer" et non "Creer". L'action "Creer" ne fait rien si le fichier existe deja, mais ne le recreera pas apres une reinstallation propre. L'action "Remplacer" garantit la presence du fichier a chaque application de la GPO.

!!! warning "A surveiller — Firefox ESR vs Firefox standard"
    En entreprise, deployez **Firefox ESR** (Extended Support Release). Le cycle de support est de 12 a 14 mois, contre 4 semaines pour Firefox standard. Les politiques `policies.json` sont identiques entre les deux versions — seule la frequence des mises a jour change. Verifiez que votre GPP Files cible le bon chemin selon la version deployee (`Mozilla Firefox` vs `Mozilla Firefox ESR`).

!!! quote "En resume"
    Le piege principal est Chrome non manage qui ignore silencieusement les politiques GPO. Deployer Chrome via le MSI entreprise est la seule garantie. Pour Firefox, configurez GPP Files en action "Remplacer" pour survivre aux reinstallations. En cas de doute sur quelle GPO s'applique, `gpresult /h` reste votre meilleur outil de diagnostic.

---

## :material-link-variant: References et ressources

### Fichiers ADMX — ou les telecharger

| Navigateur | Fichiers ADMX | URL de telechargement |
|---|---|---|
| Microsoft Edge | `msedge.admx`, `msedgeupdate.admx` | [Microsoft Edge for Business](https://www.microsoft.com/en-us/edge/business/download) |
| Google Chrome | `chrome.admx` | [Google Chrome Enterprise](https://chromeenterprise.google/browser/download/) |
| Firefox | `mozilla.admx` (communautaire) | [GitHub mozilla/policy-templates](https://github.com/mozilla/policy-templates) |

### Pages de diagnostic navigateur

| Navigateur | Page de diagnostic des politiques |
|---|---|
| Edge | `edge://policy` |
| Chrome | `chrome://policy` |
| Firefox | `about:policies` |

Ces pages affichent toutes les politiques actives, leur source (GPO, MDM, cloud) et leur valeur effective. C'est la premiere chose a consulter lors d'un probleme de politique navigateur.

### Chapitres lies

- [04-central-store.md](04-central-store.md) — deployer les ADMX Edge et Chrome dans le Central Store
- [../bible-gpo/05-admx-adml.md](../bible-gpo/05-admx-adml.md) — fonctionnement interne des fichiers ADMX/ADML

### Documentation officielle

| Ressource | URL |
|---|---|
| Edge policy reference | https://learn.microsoft.com/en-us/deployedge/microsoft-edge-policies |
| Chrome policy reference | https://chromeenterprise.google/policies/ |
| Firefox policy templates | https://github.com/mozilla/policy-templates |
| Firefox policies.json reference | https://mozilla.github.io/policy-templates/ |
