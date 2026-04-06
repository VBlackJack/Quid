---
description: "Cinq mini-projets GPO de difficulte croissante pour consolider les acquis du livre et passer de la theorie a la pratique."
tags:
  - gpo
  - debutant
  - projets
  - pratique
---
# Mini-projets GPO : mettre en pratique

!!! abstract "Ce que vous allez apprendre"
    - Creer une GPO de securite de reference applicable a n'importe quel poste de travail
    - Configurer un poste kiosk verrouille pour un usage public ou de reception
    - Ecrire un script PowerShell de diagnostic GPO automatise et planifiable
    - Deployer une configuration navigateur standardisee via les modeles ADMX
    - Construire un kit GPO complet pour demarrer un nouvel environnement Active Directory


!!! tip "Si vous ne retenez qu'une chose"
    Un mini-projet GPO réussi part d'un besoin concret, d'un périmètre limité et d'un critère de validation observable.

---

## Introduction

:material-rocket-launch: Felicitations. Vous etes au chapitre 15, le dernier de ce livre.

Vous avez appris ce qu'est une GPO, comment la creer, la lier, la sauvegarder, la diagnostiquer, et comment eviter les pieges classiques. Vous connaissez l'heritage, le filtrage, les modeles d'administration et les parametres de securite.

Il est temps de **tout mettre en pratique**.

Ce chapitre ne presente pas de nouveaux concepts. Il s'agit de **faire**. Chaque projet vous demande de reprendre des notions deja vues et de les combiner pour construire quelque chose de concret et d'utile.

### Les 5 projets

| Projet | Titre | Difficulte | Chapitres requis |
|:------:|-------|:----------:|-----------------|
| 1 | Securiser un poste de travail | ⭐ | Ch. 7 |
| 2 | Configurer un poste kiosk | ⭐⭐ | Ch. 7, 8 |
| 3 | Script de diagnostic automatise | ⭐⭐ | Ch. 12 |
| 4 | Deployer une configuration navigateur | ⭐⭐ | Ch. 9, 10 |
| 5 | Kit GPO pour nouvel environnement AD | ⭐⭐⭐ | Tous |

Chaque projet suit la meme structure : objectif, pre-requis, pas a pas, verification, et pistes pour aller plus loin.

!!! tip "Conseils avant de commencer"
    Commencez toujours par le **projet 1**, meme si vous vous sentez a l'aise. Les projets suivants s'appuient dessus.

    Si vous n'avez pas d'environnement AD de test, vous pouvez utiliser une machine virtuelle avec Windows Server. Un labo sur VirtualBox ou Hyper-V suffit amplement.

!!! warning "Sauvegarder avant tout"
    Le chapitre 11 est votre meilleur ami. Avant de commencer chaque projet, sauvegardez vos GPO existantes. C'est une habitude, pas une option.

!!! quote "En resume"
    - Ce chapitre propose 5 projets pratiques de complexite croissante.
    - Ils combinent les competences des 14 chapitres precedents.
    - L'objectif est de construire des outils reels et utilisables, pas juste de s'exercer sur des exemples fictifs.

---

## Projet 1 — Securiser un poste de travail :material-shield-check: (⭐)

### Objectif

Creer une GPO nomme **SEC-Postes-Baseline** qui applique un socle de securite minimal sur n'importe quel poste de travail du domaine.

C'est la GPO la plus importante de toute infrastructure. Elle pose les fondations. Sans elle, vos postes sont vulnerables des leur integration au domaine.

### Ce que cette GPO va configurer

| Parametre | Valeur cible |
|-----------|-------------|
| Longueur minimale du mot de passe | 12 caracteres |
| Complexite du mot de passe | Activee |
| Duree de vie maximale du mot de passe | 90 jours |
| Seuil de verrouillage du compte | 5 tentatives |
| Duree de verrouillage | 30 minutes |
| Audit : echecs de connexion | Active |
| Audit : gestion des comptes | Active |
| Autorun USB | Desactive |
| Pare-feu Windows (tous profils) | Active |
| Service Remote Registry | Desactive |

### Pre-requis

- Acces a la console GPMC (`gpmc.msc`)
- Droits d'administration sur le domaine
- Une OU de test dans Active Directory (vous allez en creer une)

### :material-list-box: Pas a pas

#### Etape 1 — Creer l'OU de test

Dans la console **Utilisateurs et ordinateurs Active Directory** (`dsa.msc`) :

1. Clic droit sur le domaine (ex : `contoso.local`)
2. **Nouveau** > **Unite d'organisation**
3. Nommez-la `Postes-Pilote`
4. Laissez la case "Proteger contre la suppression accidentelle" cochee

Cette OU servira a tester les GPO avant de les deployer plus largement.

#### Etape 2 — Deplacer un poste de test

1. Dans `dsa.msc`, trouvez votre poste de test (il est probablement dans `Computers`)
2. Clic droit > **Deplacer**
3. Selectionnez `Postes-Pilote`

#### Etape 3 — Creer la GPO

1. Ouvrez `gpmc.msc`
2. Clic droit sur l'OU `Postes-Pilote`
3. **Creer un objet GPO dans ce domaine, et le lier ici...**
4. Nommez-la `SEC-Postes-Baseline`
5. Clic droit sur la GPO > **Modifier**

#### Etape 4 — Configurer la strategie de mot de passe

!!! info "Rappel important"
    Les strategies de mot de passe **s'appliquent uniquement si la GPO est liee a la racine du domaine**. Ici, pour le projet, on les configure dans la GPO de test pour voir le comportement local — mais en production, cette partie va dans une GPO liee au domaine.

Naviguez vers :

```
Configuration ordinateur
  └── Parametres Windows
        └── Parametres de securite
              └── Strategies de comptes
                    └── Strategie de mot de passe
```

Configurez :

- **Longueur minimale du mot de passe** → `12`
- **Le mot de passe doit respecter des exigences de complexite** → `Activer`
- **Duree de vie maximale du mot de passe** → `90`

Puis allez dans **Strategie de verrouillage du compte** :

- **Seuil de verrouillage du compte** → `5`
- **Duree de verrouillage du compte** → `30`
- **Reinitialiser le compteur de verrouillages apres** → `30`

#### Etape 5 — Configurer l'audit

Naviguez vers :

```
Configuration ordinateur
  └── Parametres Windows
        └── Parametres de securite
              └── Strategies locales
                    └── Strategie d'audit
```

Activez **Echec** (au minimum) pour :

- **Auditer les evenements de connexion** → `Echec`
- **Auditer la gestion des comptes** → `Succes, Echec`

#### Etape 6 — Desactiver l'autorun USB

Naviguez vers :

```
Configuration ordinateur
  └── Modeles d'administration
        └── Composants Windows
              └── Strategies d'execution automatique
```

Double-cliquez sur **Desactiver l'execution automatique** :
- Selectionnez **Active**
- Dans le menu deroulant : **Tous les lecteurs**

#### Etape 7 — Activer le pare-feu Windows

Naviguez vers :

```
Configuration ordinateur
  └── Parametres Windows
        └── Parametres de securite
              └── Pare-feu Windows Defender avec fonctions avancees de securite
```

1. Cliquez sur **Proprietes du Pare-feu Windows Defender**
2. Dans chaque onglet (Domaine, Prive, Public) :
   - **Etat du pare-feu** → `Actif (recommande)`
   - **Connexions entrantes** → `Bloquer (par defaut)`

#### Etape 8 — Desactiver le service Remote Registry

Naviguez vers :

```
Configuration ordinateur
  └── Parametres Windows
        └── Parametres de securite
              └── Services systeme
```

Trouvez **Remote Registry** dans la liste :
- Double-cliquez dessus
- Cochez **Definir ce parametre de strategie**
- Mode de demarrage : **Desactive**

#### Etape 9 — Appliquer et verifier

Sur le poste de test, ouvrez une invite de commandes en tant qu'administrateur et forcez l'application :

```cmd
gpupdate /force
```

Attendez le message de confirmation, puis verifiez le resultat :

```cmd
gpresult /r
```

Vous devez voir `SEC-Postes-Baseline` dans la liste des GPO appliquees.

### :material-check-circle: Verification avec PowerShell

Une fois la GPO appliquee, verifiez que les parametres sont bien en place sur le poste :

```powershell
# Verify security baseline settings on the local machine

# Check password policy (domain-level settings)
Write-Host "=== Password Policy ===" -ForegroundColor Cyan
net accounts

# Check audit policy for logon events
Write-Host "`n=== Audit Policy (Logon) ===" -ForegroundColor Cyan
auditpol /get /subcategory:"Logon"

# Check firewall status across all profiles
Write-Host "`n=== Windows Firewall Profiles ===" -ForegroundColor Cyan
Get-NetFirewallProfile | Select-Object Name, Enabled, DefaultInboundAction |
    Format-Table -AutoSize

# Check Remote Registry service status
Write-Host "`n=== Remote Registry Service ===" -ForegroundColor Cyan
Get-Service -Name RemoteRegistry | Select-Object Name, Status, StartType
```

**Resultat attendu :**

- `net accounts` affiche une longueur minimale de 12 et un seuil de verrouillage de 5
- Les trois profils de pare-feu affichent `Enabled : True`
- Le service `RemoteRegistry` est en `Disabled` ou `Stopped`

### Pour aller plus loin

- Ajoutez les parametres d'audit avances (via `auditpol`) pour couvrir les acces aux objets et les changements de privileges
- Liez cette GPO a toutes vos OU de postes de travail, pas seulement `Postes-Pilote`
- Combinez-la avec le projet 5 pour construire une infrastructure GPO complete

!!! quote "En resume"
    - La GPO `SEC-Postes-Baseline` configure les fondations de securite : mot de passe, verrouillage, audit, pare-feu, autorun et Remote Registry.
    - Elle se cree en moins de 30 minutes et se deploie sur autant de postes que vous voulez en changeant juste la liaison d'OU.
    - Verifiez toujours avec `gpresult /r` et les commandes PowerShell fournis.

---

## Projet 2 — Configurer un poste kiosk :material-monitor-lock: (⭐⭐)

### Objectif

Transformer un poste Windows standard en **terminal kiosk** : un poste accessible au public, a la reception, ou dans un espace de travail partage, ou l'utilisateur peut faire une seule chose bien definie — et rien d'autre.

### Ce que cette GPO va verrouiller

| Fonctionnalite | Effet de la restriction |
|----------------|------------------------|
| Fond d'ecran | Image imposee (logo entreprise) |
| Gestionnaire des taches | Inaccessible via Ctrl+Alt+Suppr |
| Icones du bureau | Masquees (sauf Corbeille) |
| Boite d'execution | Win+R desactive |
| Options Ctrl+Alt+Suppr | Verrouillage, changement mdp, deconnexion masques |
| Lecteurs dans l'Explorateur | Masques |
| Arret depuis le menu Demarrer | Desactive |
| Ecran de veille | Actif avec verrouillage apres 5 minutes |

### Pre-requis

- Avoir complete le projet 1 (notion d'OU et de GPO)
- Un compte utilisateur standard de test (pas un compte admin)
- Une OU `Postes-Kiosk` dans Active Directory

### :material-list-box: Pas a pas

#### Etape 1 — Creer l'OU et la GPO

Dans `dsa.msc`, creez une nouvelle OU `Postes-Kiosk` sous la racine du domaine.

Dans `gpmc.msc` :
1. Clic droit sur `Postes-Kiosk`
2. Creez une GPO nommee `CFG-Postes-Kiosk-Restrictions`
3. Modifiez-la

#### Etape 2 — Imposer le fond d'ecran

Naviguez vers :

```
Configuration utilisateur
  └── Modeles d'administration
        └── Bureau
              └── Bureau
```

Cherchez **Papier peint du Bureau** :
- **Activer**
- Chemin de l'image : `\\contoso.local\NETLOGON\kiosk-wallpaper.jpg`
- Style : `Etirer`

!!! tip "Ou stocker l'image ?"
    Le partage `NETLOGON` du controleur de domaine est accessible en lecture par tous les postes du domaine. C'est l'endroit ideal pour stocker les ressources partagees comme un fond d'ecran d'entreprise.

#### Etape 3 — Desactiver le Gestionnaire des taches

Naviguez vers :

```
Configuration utilisateur
  └── Modeles d'administration
        └── Systeme
              └── Options Ctrl+Alt+Suppr
```

Activez **Supprimer le Gestionnaire des taches**.

#### Etape 4 — Masquer les icones du bureau

Naviguez vers :

```
Configuration utilisateur
  └── Modeles d'administration
        └── Bureau
```

Activez les parametres suivants :
- **Masquer et desactiver tous les elements du bureau**

!!! tip "Alternative plus fine"
    Si vous voulez garder certaines icones (raccourci vers l'application kiosk par exemple), utilisez plutot les parametres individuels dans la meme section plutot que de tout masquer d'un coup.

#### Etape 5 — Desactiver la boite d'execution

Naviguez vers :

```
Configuration utilisateur
  └── Modeles d'administration
        └── Menu Demarrer et Barre des taches
```

Activez **Supprimer le menu Executer du menu Demarrer**.

#### Etape 6 — Restreindre les options Ctrl+Alt+Suppr

Dans la meme section **Options Ctrl+Alt+Suppr** qu'a l'etape 3, activez :
- **Supprimer Verrouiller l'ordinateur**
- **Supprimer Modifier un mot de passe**
- **Supprimer Se deconnecter**

#### Etape 7 — Masquer les lecteurs dans l'Explorateur

Naviguez vers :

```
Configuration utilisateur
  └── Modeles d'administration
        └── Composants Windows
              └── Explorateur de fichiers
```

Activez **Masquer ces lecteurs specifiques dans Poste de travail**.
Selectionnez l'option qui correspond a vos lecteurs (ou "Restreindre tous les lecteurs").

#### Etape 8 — Desactiver l'arret depuis le menu Demarrer

Naviguez vers :

```
Configuration utilisateur
  └── Modeles d'administration
        └── Menu Demarrer et Barre des taches
```

Activez **Supprimer et empecher l'acces aux commandes Arreter, Redemarrer, Mettre en veille et Hiberner**.

#### Etape 9 — Configurer l'ecran de veille avec mot de passe

Naviguez vers :

```
Configuration utilisateur
  └── Modeles d'administration
        └── Panneau de configuration
              └── Personnalisation
```

Configurez ces trois parametres :
- **Activer l'ecran de veille** → `Activer`
- **Proteger l'ecran de veille par mot de passe** → `Activer`
- **Temporisation de l'ecran de veille** → `Activer`, valeur : `300` (secondes = 5 minutes)

#### Etape 10 — Tester avec un compte standard

!!! danger "Cette etape est obligatoire"
    Testez **absolument** avec un compte utilisateur standard, pas un compte administrateur. Les restrictions d'utilisateur ne s'appliquent pas aux admins locaux. Vous risqueriez de croire que tout fonctionne alors que l'utilisateur final verra quelque chose de completement different.

1. Connectez-vous sur le poste kiosk avec le compte standard de test
2. Verifiez que chaque restriction est en place
3. Verifiez surtout que l'utilisateur peut **encore faire ce qu'il est cense faire**

```cmd
gpresult /r /user contoso\utilisateur-kiosk
```

### Pour aller plus loin

- Combinez cette GPO avec une politique de redemarrage automatique nocturne (tache planifiee + GPO de preferences) pour que le kiosk soit toujours frais le matin
- Utilisez **Assigned Access** (dans les parametres Windows) pour aller encore plus loin et limiter l'acces a une seule application

!!! warning "Le kiosk doit rester utilisable"
    Un kiosk trop verrouille est inutile. Avant de deployer, listez precisement ce que l'utilisateur doit pouvoir faire et testez chaque action. La liste des restrictions vient **apres** la liste des besoins, jamais avant.

!!! quote "En resume"
    - La GPO `CFG-Postes-Kiosk-Restrictions` verrouille l'interface utilisateur tout en laissant le poste fonctionnel pour son usage prevu.
    - Toutes les restrictions sont dans la partie **Configuration utilisateur**, pas Configuration ordinateur.
    - Testez obligatoirement avec un compte standard — les admins echappent a ces restrictions.

---

## Projet 3 — Script de diagnostic GPO automatise :material-script-text: (⭐⭐)

### Objectif

Ecrire un script PowerShell qui genere automatiquement un rapport de diagnostic GPO complet pour n'importe quel poste du domaine — puis le planifier pour qu'il tourne chaque semaine sans intervention manuelle.

Ce script vous fera gagner un temps considerable lorsque vous devrez diagnostiquer des problemes GPO a distance.

### Pre-requis

- PowerShell 5.1 ou superieur
- Droits d'administration sur les postes cibles
- Avoir vu le chapitre 12 (gpresult et le rapport HTML)

### Le script principal

Creez un fichier `gpo-diagnostic.ps1` dans `C:\Scripts\` (creez le dossier si necessaire) :

```powershell
# GPO Diagnostic Script
# Generates a full HTML report and console summary for a given computer/user
# Usage: .\gpo-diagnostic.ps1 [-Computer <name>] [-User <name>]

param(
    [string]$Computer = $env:COMPUTERNAME,
    [string]$User     = $env:USERNAME
)

# Create output directory if it does not exist
$outputDir = "C:\Temp\GPO-Reports"
if (-not (Test-Path $outputDir)) {
    New-Item -ItemType Directory -Path $outputDir | Out-Null
}

$timestamp  = Get-Date -Format 'yyyyMMdd-HHmm'
$reportPath = "$outputDir\GPO-Diagnostic-$Computer-$timestamp.html"

Write-Host "Generating GPO diagnostic report..." -ForegroundColor Cyan
Write-Host "  Computer : $Computer" -ForegroundColor White
Write-Host "  User     : $User" -ForegroundColor White
Write-Host "  Output   : $reportPath" -ForegroundColor White
Write-Host ""

# Generate full HTML report
gpresult /S $Computer /USER $User /H $reportPath /F 2>&1 | Out-Null

# Check whether the report was created successfully
if (Test-Path $reportPath) {
    Write-Host "Report generated successfully." -ForegroundColor Green
    Start-Process $reportPath
} else {
    Write-Host "ERROR: Failed to generate report." -ForegroundColor Red
    Write-Host "Verify that you have admin rights on $Computer and that it is reachable." -ForegroundColor Yellow
    exit 1
}

# Print a quick console summary
Write-Host "`n=== Quick Console Summary ===" -ForegroundColor Cyan
gpresult /S $Computer /USER $User /R 2>&1
```

### Le script bonus : verifier les erreurs GPO recentes

Ce deuxieme script inspecte le journal d'evenements GPO du poste cible et remonte les erreurs des dernieres 24 heures :

```powershell
# Check GPO error events from the last 24 hours on a given computer
# Usage: .\gpo-events.ps1 [-Computer <name>]

param(
    [string]$Computer = $env:COMPUTERNAME
)

$cutoff   = (Get-Date).AddHours(-24)
$logName  = "Microsoft-Windows-GroupPolicy/Operational"

Write-Host "Checking GPO error events on $Computer (last 24h)..." -ForegroundColor Cyan

$events = Get-WinEvent -ComputerName $Computer `
              -LogName $logName `
              -ErrorAction SilentlyContinue |
          Where-Object {
              $_.LevelDisplayName -eq "Error" -and
              $_.TimeCreated -gt $cutoff
          }

if ($events) {
    Write-Host "Found $($events.Count) error event(s):" -ForegroundColor Yellow
    $events |
        Select-Object TimeCreated, Id, Message |
        Format-Table -AutoSize -Wrap
} else {
    Write-Host "No GPO errors in the last 24 hours. All good." -ForegroundColor Green
}
```

### :material-list-box: Pas a pas — planifier le script comme tache automatique

#### Etape 1 — Preparer le script

Enregistrez `gpo-diagnostic.ps1` dans `C:\Scripts\gpo-diagnostic.ps1`.

#### Etape 2 — Ouvrir le Planificateur de taches

Tapez `taskschd.msc` dans la barre de recherche Windows.

#### Etape 3 — Creer la tache

1. Dans le panneau de gauche, clic droit sur **Bibliotheque du Planificateur de taches**
2. **Creer une tache de base...**
3. Nom : `GPO-Diagnostic-Hebdomadaire`
4. Description : `Weekly GPO diagnostic report for this computer`

#### Etape 4 — Configurer le declencheur

- **Chaque semaine**, le **lundi** a **08h00**

#### Etape 5 — Configurer l'action

- **Demarrer un programme**
- Programme : `powershell.exe`
- Arguments :

```
-ExecutionPolicy Bypass -NonInteractive -File "C:\Scripts\gpo-diagnostic.ps1" -Computer %COMPUTERNAME%
```

#### Etape 6 — Verifier la tache

Vous pouvez tester immediatement en clic droit > **Executer** sur la tache. Le rapport HTML doit apparaitre dans `C:\Temp\GPO-Reports\`.

### Pour aller plus loin

- Modifiez le script pour envoyer le rapport par e-mail (commande `Send-MailMessage` ou module Exchange/Graph)
- Ajoutez un parametre `-OutputFormat CSV` pour exporter les erreurs dans un tableau croise
- Deployez le script et la tache planifiee via... une GPO (preferences > taches planifiees, chapitre 10)

!!! quote "En resume"
    - Le script `gpo-diagnostic.ps1` genere un rapport HTML complet avec `gpresult /H` et affiche un resume console.
    - Le script bonus inspecte le journal d'evenements GPO pour les erreurs recentes.
    - La tache planifiee garantit un diagnostic automatique chaque semaine sans intervention manuelle.

---

## Projet 4 — Deployer une configuration navigateur entreprise :material-web: (⭐⭐)

### Objectif

Standardiser la configuration de **Microsoft Edge** sur tous les postes du domaine via une GPO : page d'accueil, proxy, gestionnaire de mots de passe, SmartScreen et onglet nouvel onglet.

### Pre-requis

- Les modeles ADMX Microsoft Edge installes dans le **magasin central** (Central Store)
- Avoir lu le chapitre 9 (modeles d'administration) et le chapitre 10 (preferences)
- Avoir Edge installe sur les postes cibles

!!! info "Ou trouver les modeles ADMX Edge ?"
    Telechargez les modeles depuis la page officielle Microsoft : recherchez "Microsoft Edge administrative templates" sur docs.microsoft.com. Vous obtenez un fichier ZIP contenant `msedge.admx`, `msedgeupdate.admx` et les fichiers de langue `.adml`.

    Copiez-les dans `\\contoso.local\SYSVOL\contoso.local\Policies\PolicyDefinitions\` (le magasin central). Si ce dossier n'existe pas, creez-le — Windows GPMC l'utilisera automatiquement.

### Ce que cette GPO va configurer

| Parametre | Valeur cible |
|-----------|-------------|
| Page d'accueil | `https://intranet.contoso.local` |
| Enregistrement des mots de passe | Desactive |
| Parametre proxy | Fichier PAC (configurable) |
| Extensions tierces | Bloquees par defaut |
| Onglet nouvel onglet | Portail entreprise |
| Microsoft Defender SmartScreen | Active |

### :material-list-box: Pas a pas

#### Etape 1 — Creer la GPO

Dans `gpmc.msc` :
1. Clic droit sur votre OU de postes de travail
2. Creez une GPO `CFG-Edge-Configuration-Entreprise`
3. Modifiez-la

#### Etape 2 — Configurer la page d'accueil

Naviguez vers :

```
Configuration ordinateur
  └── Modeles d'administration
        └── Microsoft Edge
```

Double-cliquez sur **Configurer l'URL de la page d'accueil** :
- **Activer**
- Valeur : `https://intranet.contoso.local`

Activez aussi **Definir la page d'accueil comme page Nouvel onglet** si vous souhaitez que le nouvel onglet ouvre aussi l'intranet.

#### Etape 3 — Desactiver le gestionnaire de mots de passe

Dans la meme section **Microsoft Edge** :

Double-cliquez sur **Activer l'enregistrement des mots de passe dans le gestionnaire de mots de passe** :
- **Desactiver**

!!! warning "Pourquoi desactiver le gestionnaire de mots de passe ?"
    En environnement d'entreprise, les mots de passe doivent etre geres par un coffre-fort centralise (CyberArk, Bitwarden for Business, etc.) et non stockes localement dans le navigateur. Le stockage local est un risque en cas de vol ou de compromission du poste.

#### Etape 4 — Configurer le proxy

Naviguez vers :

```
Configuration ordinateur
  └── Modeles d'administration
        └── Microsoft Edge
              └── Serveur proxy
```

Double-cliquez sur **Configurer les parametres du proxy** :
- **Activer**
- Mode proxy : `pac_script`
- URL du script PAC : `http://proxy.contoso.local/proxy.pac`

#### Etape 5 — Bloquer les extensions non approuvees

Dans **Microsoft Edge > Extensions** :

Activez **Conserver les extensions installees lors du deprovisionnement** puis cherchez **Contr\^oler les extensions pouvant etre installees** :
- **Activer**
- Extensions bloquees : `*` (toutes)
- Exceptions : listez les IDs des extensions approuvees

!!! tip "Comment trouver l'ID d'une extension Edge ?"
    Dans Edge, allez dans `edge://extensions`, activez le **Mode developpeur**. Chaque extension affiche alors son ID unique (une chaine de 32 caracteres).

#### Etape 6 — Activer SmartScreen

Dans **Microsoft Edge** :

Double-cliquez sur **Configurer Microsoft Defender SmartScreen** :
- **Activer**

#### Etape 7 — Verifier l'application

Apres `gpupdate /force` sur un poste de test, verifiez que les cles de registre ont bien ete ecrites :

```powershell
# Check Edge GPO registry keys on the local machine
$edgePolicyPath = "HKLM:\SOFTWARE\Policies\Microsoft\Edge"

if (Test-Path $edgePolicyPath) {
    Write-Host "Edge GPO policies found:" -ForegroundColor Green
    Get-ItemProperty -Path $edgePolicyPath |
        Select-Object * -ExcludeProperty PSPath, PSParentPath, PSChildName, PSDrive, PSProvider |
        Format-List
} else {
    Write-Host "No Edge GPO policies detected at $edgePolicyPath" -ForegroundColor Yellow
    Write-Host "Check that the ADMX templates are installed and the GPO is linked." -ForegroundColor White
}
```

**Resultat attendu :** Vous verrez les valeurs `HomepageLocation`, `PasswordManagerEnabled`, `SmartScreenEnabled`, etc.

### Note sur les autres navigateurs

La meme approche fonctionne pour **Google Chrome** (ADMX disponibles sur le site Chrome Enterprise) et **Mozilla Firefox** (qui utilise un fichier `policies.json` dans son dossier d'installation plutot que les modeles ADMX).

### Pour aller plus loin

- Ajoutez les parametres de mise a jour automatique d'Edge via `msedgeupdate.admx`
- Configurez le **mode IE** pour les applications legacy qui necessitent Internet Explorer
- Deployez une liste de sites de confiance pour le SSO (authentification unique)

!!! quote "En resume"
    - La GPO `CFG-Edge-Configuration-Entreprise` standardise Edge via les modeles ADMX officiels Microsoft.
    - Les parametres se configurent sous `Configuration ordinateur > Modeles d'administration > Microsoft Edge`.
    - Verifiez l'application en inspectant les cles `HKLM:\SOFTWARE\Policies\Microsoft\Edge` avec PowerShell.

---

## Projet 5 — Kit GPO pour nouvel environnement AD :material-domain: (⭐⭐⭐)

### Objectif

Construire un **kit de demarrage GPO complet** pour un nouveau domaine Active Directory. Ce kit comprend la structure d'OU recommandee et les 5 GPO essentielles que tout domaine devrait avoir des le premier jour.

C'est le projet le plus ambitieux. Il reprend tout ce que vous avez vu dans ce livre.

### Pre-requis

- Avoir complete les projets 1 a 4
- Droits d'administration de domaine
- Module PowerShell Active Directory installe (`Install-WindowsFeature RSAT-AD-PowerShell`)

### La structure OU recommandee

```
contoso.local
├── Postes-de-travail
│   ├── Postes-Pilote          ← OU de test, jamais en production
│   ├── Postes-Bureautique
│   └── Postes-Applications
├── Utilisateurs
│   ├── Utilisateurs-Standard
│   └── Utilisateurs-Admin
└── Serveurs
    ├── Serveurs-Infrastructure
    └── Serveurs-Applications
```

Cette structure separe clairement les postes, les utilisateurs et les serveurs. Elle vous permet de lier des GPO differentes a chaque categorie sans conflit.

### Les 5 GPO essentielles

| GPO | Liee a | Contenu |
|-----|--------|---------|
| `SEC-Domaine-MotDePasse` | Racine du domaine | Strategie de mot de passe, verrouillage, Kerberos |
| `SEC-Postes-Baseline` | `Postes-de-travail` | Pare-feu, audit, autorun, Remote Registry |
| `CFG-Postes-WindowsUpdate` | `Postes-de-travail` | WSUS ou Windows Update for Business |
| `CFG-Utilisateurs-Bureau` | `Utilisateurs` | Fond d'ecran, ecran de veille, menu Demarrer |
| `SEC-Admins-Restrictions` | `Utilisateurs-Admin` | Restrictions supplementaires + audit renforce |

### :material-list-box: Pas a pas

#### Etape 1 — Creer la structure OU avec PowerShell

```powershell
# Create starter OU structure for a new Active Directory domain
# Requires the Active Directory PowerShell module
Import-Module ActiveDirectory

$domain = (Get-ADDomain).DistinguishedName
Write-Host "Domain DN: $domain" -ForegroundColor Cyan

# Top-level OUs
$topOUs = @(
    "Postes-de-travail",
    "Utilisateurs",
    "Serveurs"
)

foreach ($ou in $topOUs) {
    try {
        New-ADOrganizationalUnit -Name $ou -Path $domain -ProtectedFromAccidentalDeletion $true
        Write-Host "Created OU: $ou" -ForegroundColor Green
    } catch {
        Write-Host "OU already exists or error: $ou — $($_.Exception.Message)" -ForegroundColor Yellow
    }
}

# Sub-OUs
$subOUs = @(
    @{ Name = "Postes-Pilote";            Parent = "Postes-de-travail" },
    @{ Name = "Postes-Bureautique";       Parent = "Postes-de-travail" },
    @{ Name = "Postes-Applications";      Parent = "Postes-de-travail" },
    @{ Name = "Utilisateurs-Standard";    Parent = "Utilisateurs"      },
    @{ Name = "Utilisateurs-Admin";       Parent = "Utilisateurs"      },
    @{ Name = "Serveurs-Infrastructure";  Parent = "Serveurs"          },
    @{ Name = "Serveurs-Applications";    Parent = "Serveurs"          }
)

foreach ($subOU in $subOUs) {
    $parentPath = "OU=$($subOU.Parent),$domain"
    try {
        New-ADOrganizationalUnit -Name $subOU.Name -Path $parentPath -ProtectedFromAccidentalDeletion $true
        Write-Host "Created sub-OU: $($subOU.Name) under $($subOU.Parent)" -ForegroundColor Green
    } catch {
        Write-Host "Sub-OU already exists or error: $($subOU.Name) — $($_.Exception.Message)" -ForegroundColor Yellow
    }
}

Write-Host "`nOU structure creation complete." -ForegroundColor Cyan
```

#### Etape 2 — Creer la GPO de mot de passe domaine

Dans `gpmc.msc`, clic droit sur **la racine du domaine** (pas une OU) :

1. Creez `SEC-Domaine-MotDePasse`
2. Configurez (chemin : `Config. ordinateur > Parametres Windows > Parametres de securite > Strategies de comptes`) :
   - Longueur minimale : `14` caracteres
   - Complexite : `Activee`
   - Duree de vie max : `90` jours
   - Seuil de verrouillage : `5`
   - Duree de verrouillage : `30` min
   - Reinitialisation du compteur : `30` min

!!! warning "Une seule GPO pour le mot de passe domaine"
    N'ajoutez rien d'autre dans cette GPO. Elle est liee au domaine entier. Moins elle contient de parametres, moins elle presente de risques de conflit ou d'effet de bord.

#### Etape 3 — Creer la GPO de securite des postes

Reportez-vous au **projet 1** de ce chapitre. La GPO `SEC-Postes-Baseline` est exactement ce qu'il faut creer ici. Liez-la a l'OU `Postes-de-travail`.

#### Etape 4 — Creer la GPO Windows Update

Naviguez vers :

```
Configuration ordinateur
  └── Modeles d'administration
        └── Composants Windows
              └── Windows Update
                    └── Gerer l'experience utilisateur de mise a jour de fin de service
```

Configurez selon votre infrastructure :
- **Avec WSUS** : pointez vers l'adresse de votre serveur WSUS
- **Sans WSUS** : utilisez les parametres **Windows Update for Business** pour definir un anneau de mise a jour

Ajoutez aussi les heures actives pour eviter les redefontes forcees pendant les heures de travail :
- **Heures actives** : `08:00` a `18:00`

Liez cette GPO a `Postes-de-travail`.

#### Etape 5 — Creer la GPO de bureau utilisateur

Clic droit sur l'OU `Utilisateurs` > Creez `CFG-Utilisateurs-Bureau`.

Configurez dans **Configuration utilisateur** :
- Fond d'ecran imposé (logo entreprise)
- Ecran de veille avec mot de passe apres 10 minutes
- Masquer les lecteurs reseau non mappes dans l'Explorateur

#### Etape 6 — Creer la GPO de restrictions admin

Clic droit sur `Utilisateurs-Admin` > Creez `SEC-Admins-Restrictions`.

Cette GPO ajoute des restrictions supplementaires pour les comptes administrateurs :
- Audit renforce de toutes les connexions (succes et echec)
- Interdiction d'utiliser les partages reseau non autorises
- Activation de Windows Defender Credential Guard (si l'infrastructure le permet)

#### Etape 7 — Nommer, documenter et sauvegarder

Pour chaque GPO creee :
1. Clic droit > **Proprietes** > onglet **Commentaire**
2. Ecrivez une description : objectif, date de creation, auteur, parametres principaux
3. Sauvegardez toutes les GPO (chapitre 11)

```powershell
# List all GPOs with their descriptions to verify documentation
Import-Module GroupPolicy

Get-GPO -All |
    Select-Object DisplayName, GpoStatus, Description,
                  @{N="Created"; E={$_.CreationTime.ToString("dd/MM/yyyy")}},
                  @{N="Modified"; E={$_.ModificationTime.ToString("dd/MM/yyyy")}} |
    Format-Table -AutoSize
```

**Resultat attendu :** Chaque GPO doit avoir une description non vide.

### Les conventions de nommage

Adoptez ces prefixes des le premier jour. Vous remercier plus tard.

| Prefixe | Usage | Exemple |
|---------|-------|---------|
| `SEC-` | Securite, restrictions | `SEC-Postes-Baseline` |
| `CFG-` | Configuration, preferences | `CFG-Edge-Entreprise` |
| `INF-` | Infrastructure, services | `INF-WSUS-Configuration` |
| `USR-` | Restrictions specifiques utilisateurs | `USR-Kiosk-Restrictions` |

### Les bonnes pratiques du kit

:material-check: **Une GPO, un objectif.** Ne melangez pas securite et configuration dans la meme GPO. Vous ne pourrez pas les desactiver independamment.

:material-check: **Gardez une GPO vide de reference.** Creez une GPO `_MODELE-GPO-Vide` sans rien dedans. Elle servira de point de depart pour de nouvelles GPO.

:material-check: **Documentez le jour J.** La description d'une GPO, c'est comme le commentaire dans un script : si vous ne l'ecrivez pas maintenant, vous ne l'ecrirez jamais.

:material-check: **Automatisez la sauvegarde.** Combinez le script du projet 3 avec une sauvegarde GPO planifiee. Chaque lundi matin : rapport de diagnostic + sauvegarde automatique.

### Pour aller plus loin

Une fois ce kit en place, explorez les **baselines de securite Microsoft** : ce sont des ensembles de GPO pre-configures fournis gratuitement par Microsoft Security Compliance Toolkit. Ils couvrent Windows 11, Windows Server 2022, Office, Edge, et bien d'autres. C'est le point de depart ideal pour une politique de securite rigoureuse.

!!! quote "En resume"
    - Le kit GPO comprend 5 GPO essentielles et une structure OU coherente, deploye en moins de 2 heures.
    - La structure OU se cree en une commande PowerShell. Chaque GPO a un perimetre unique et clairement documente.
    - Adoptez les prefixes `SEC-`, `CFG-`, `INF-`, `USR-` des le premier jour : votre infrastructure vous remerciera dans 3 ans.

---

## Et maintenant ?

:material-party-popper: **Vous avez termine "Les GPO pour les Nuls".**

Faites une pause. Serieusement.

Vous avez parcouru 15 chapitres. Vous avez appris ce qu'est une GPO, comment la creer, la lier, la sauvegarder, la diagnostiquer et la deployer. Vous savez utiliser `gpmc.msc`, `gpresult`, `gpupdate`, les modeles d'administration, les parametres de securite, et vous avez maintenant 5 projets concrets a votre actif.

Ce n'est pas rien.

### Ce que vous maitrisez desormais

| Chapitre | Competence acquise |
|:--------:|--------------------|
| 1-3 | Comprendre les GPO et utiliser gpedit.msc |
| 4-6 | Creer, lier et heriter des GPO dans un domaine AD |
| 7-10 | Configurer la securite, le bureau, Windows Update et les preferences |
| 11-13 | Sauvegarder, diagnostiquer et eviter les erreurs classiques |
| 14 | Adapter les GPO a Windows 11 |
| 15 | Mettre en pratique avec 5 projets reels |

### La suite du voyage

Ce livre couvre les fondamentaux. Mais les GPO sont un sujet vaste. Voici ou aller selon votre prochain objectif.

---
#### :material-book-open-page-variant: Vous voulez comprendre pourquoi les GPO fonctionnent ainsi ?

Plongez dans **La Bible des Strategies de Groupe**.

Ce livre couvre l'architecture interne : les Client Side Extensions (CSE), le fonctionnement de SYSVOL, le format `registry.pol`, les baselines de securite Microsoft Security Compliance Toolkit, les strategies avancees de filtrage et bien plus.

C'est la reference technique pour ceux qui veulent comprendre non seulement **quoi configurer**, mais **pourquoi et comment ca marche en dessous**.

[Ouvrir La Bible des Strategies de Groupe](../bible-gpo/index.md){ .md-button .md-button--primary }

---
#### :material-briefcase-outline: Vous etes administrateur systeme et vous voulez des cas concrets ?

Consultez **Les GPO pour les Administrateurs**.

Ce livre s'adresse aux pros en production. Il couvre les scenarios reels : deploiement Office 365, migration vers Intune, configuration d'entreprise pour GlobalProtect et WSUS, gestion des navigateurs a grande echelle, integration CI/CD pour les GPO, et bien d'autres.

[Ouvrir Les GPO pour les Administrateurs](../gpo-pour-les-admins/index.md){ .md-button }

---
### Un dernier conseil

:material-lightbulb: Les GPO ne s'apprennent vraiment qu'en les pratiquant. Le meilleur investissement que vous puissiez faire maintenant, c'est de monter un labo : une machine virtuelle Windows Server, quelques postes clients, et un domaine de test.

Cassez des choses. Diagnostiquez. Recommencez.

Chaque erreur dans un labo est une erreur que vous n'aurez pas en production.

Bonne route.

!!! quote "En resume"
    - Vous avez complete les 15 chapitres de "Les GPO pour les Nuls" et maitrisez les fondamentaux des strategies de groupe Windows.
    - Pour approfondir l'architecture interne, rendez-vous dans **La Bible des Strategies de Groupe**.
    - Pour des scenarios d'administration d'entreprise, rendez-vous dans **Les GPO pour les Administrateurs**.
    - La meilleure facon de progresser reste de pratiquer dans un labo de test.
