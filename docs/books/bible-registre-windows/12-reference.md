---
tags:
  - registre
  - référence
  - clés
---

# Reference des cles

## Ce que vous allez apprendre

- Retrouver rapidement les cles du registre les plus utilisees
- Connaitre les valeurs importantes pour chaque categorie (systeme, demarrage, apparence, reseau, securite)
- Appliquer des personnalisations courantes avec les commandes correspondantes

!!! info "Cette page est un aide-memoire"
    Utilisez-la comme reference rapide. Chaque section est independante : allez directement a celle qui vous interesse.

---

## Informations systeme

### Version de Windows

C'est la **carte d'identite** de votre installation Windows.

```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion
```

| Valeur | Description | Exemple |
|--------|-------------|---------|
| `ProductName` | Nom commercial | `Windows 11 Pro` |
| `DisplayVersion` | Version d'affichage | `23H2` |
| `CurrentBuildNumber` | Numero de build | `22631` |
| `UBR` | Revision de mise a jour (*Update Build Revision*) | `3007` |
| `EditionID` | Edition | `Professional` |
| `InstallDate` | Horodatage d'installation (UNIX time) | `1680000000` |
| `RegisteredOwner` | Proprietaire enregistre | `Jean Dupont` |
| `RegisteredOrganization` | Organisation enregistree | `ACME Corp` |

!!! example "Lire la version de Windows en une commande"
    ```batch
    reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v ProductName
    ```

    ```title="Resultat attendu"
    HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion
        ProductName    REG_SZ    Windows 11 Pro
    ```

### Nom de la machine

Deux emplacements distincts stockent le nom de la machine : un pour NetBIOS, un pour DNS.

**Nom NetBIOS :**

```
HKLM\SYSTEM\CurrentControlSet\Control\ComputerName\ComputerName
```

| Valeur | Description |
|--------|-------------|
| `ComputerName` | Nom NetBIOS de la machine (15 caracteres max) |

**Nom d'hote DNS :**

```
HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters
```

| Valeur | Description |
|--------|-------------|
| `Hostname` | Nom d'hote DNS actuel |
| `NV Hostname` | Nom d'hote permanent (survit au redemarrage) |

!!! tip "NetBIOS vs DNS : quelle difference ?"
    Le nom **NetBIOS** est l'ancien systeme de nommage Windows (limite a 15 caracteres). Le nom **DNS** est le standard moderne. Les deux sont generalement identiques, mais ils peuvent differer dans certaines configurations.

!!! info "En resume"
    - Version de Windows : `HKLM\...\CurrentVersion` (ProductName, CurrentBuildNumber)
    - Nom de machine : deux emplacements (ComputerName pour NetBIOS, Tcpip\Parameters pour DNS)

---

## Programmes au demarrage

C'est ici que Windows sait quels programmes lancer automatiquement -- comme une liste de courses pour le demarrage.

### Demarrage par utilisateur

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce
```

### Demarrage machine (tous les utilisateurs)

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
```

### Demarrage precoce (avant l'ouverture de session)

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunServicesOnce
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunServices
```

!!! warning "Cles obsoletes"
    `RunServices` et `RunServicesOnce` sont des reliquats des anciennes versions Windows 9x/NT. Ils ne doivent plus etre utilises comme mecanisme de demarrage sur les versions modernes de Windows ; preferez `Run`, `RunOnce`, les services ou le Planificateur de taches selon le besoin reel.

!!! example "Lister les programmes au demarrage de l'utilisateur"
    ```batch
    reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"
    ```

    ```title="Resultat attendu"
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
        Discord    REG_SZ    "C:\Users\User\AppData\Local\Discord\Update.exe" --processStart Discord.exe
        Steam      REG_SZ    "C:\Program Files (x86)\Steam\steam.exe" /silentlogin
    ```

!!! tip "RunOnce : execution unique"
    Les entrees `RunOnce` sont **supprimees apres leur execution**. Astuce : prefixer le nom de la valeur par `!` empeche la suppression en cas d'echec.

    ```batch
    rem Add a RunOnce entry that persists on failure
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce" /v "!SetupFinalize" /t REG_SZ /d "C:\setup\finalize.bat" /f
    ```

    ```title="Resultat attendu"
    The operation completed successfully.
    ```

!!! info "En resume"
    - `HKCU\...\Run` = programmes de l'utilisateur courant
    - `HKLM\...\Run` = programmes pour tous les utilisateurs
    - `RunOnce` = execution unique, puis suppression de l'entree
    - Prefixe `!` = conserver l'entree si l'execution echoue

---

## Desinstallation des programmes

C'est le catalogue de tous les logiciels installes -- ce que vous voyez dans "Programmes et fonctionnalites".

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{GUID ou NomApp}
HKCU\Software\Microsoft\Windows\CurrentVersion\Uninstall\{GUID ou NomApp}
HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\{GUID ou NomApp}
```

!!! info "Pourquoi trois emplacements ?"
    | Emplacement | Contient |
    |-------------|----------|
    | `HKLM\...\Uninstall` | Applications 64 bits installees pour tous les utilisateurs |
    | `HKLM\...\WOW6432Node\...\Uninstall` | Applications 32 bits installees pour tous les utilisateurs |
    | `HKCU\...\Uninstall` | Applications installees pour l'utilisateur courant uniquement |

| Valeur | Description | Exemple |
|--------|-------------|---------|
| `DisplayName` | Nom affiche | `Mozilla Firefox` |
| `DisplayVersion` | Version affichee | `125.0.1` |
| `Publisher` | Editeur | `Mozilla` |
| `InstallLocation` | Repertoire d'installation | `C:\Program Files\Mozilla Firefox` |
| `UninstallString` | Commande de desinstallation | `"C:\Program Files\...\uninstall.exe"` |
| `InstallDate` | Date d'installation | `20260402` (format YYYYMMDD) |
| `EstimatedSize` | Taille estimee en Ko | `215040` |

!!! example "Lister tous les programmes installes"
    ```powershell
    # List installed programs from all three locations
    $paths = @(
        "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*",
        "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*",
        "HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*"
    )
    Get-ItemProperty $paths -ErrorAction SilentlyContinue |
        Where-Object DisplayName |
        Select-Object DisplayName, DisplayVersion, Publisher |
        Sort-Object DisplayName | Format-Table -AutoSize
    ```

    ```title="Resultat attendu"
    DisplayName          DisplayVersion Publisher
    -----------          -------------- ---------
    7-Zip 24.09          24.09          Igor Pavlov
    Mozilla Firefox      125.0.1        Mozilla
    Visual Studio Code   1.88.1         Microsoft Corporation
    ```

!!! info "En resume"
    - Trois emplacements `Uninstall` : 64 bits (`HKLM`), 32 bits (`WOW6432Node`), utilisateur (`HKCU`)
    - `DisplayName`, `UninstallString` et `InstallLocation` sont les valeurs les plus utiles
    - PowerShell permet de lister tous les programmes en interrogeant les trois chemins

---

## Apparence et personnalisation

### Theme sombre / clair

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize
```

| Valeur | Type | `0` | `1` |
|--------|------|-----|-----|
| `AppsUseLightTheme` | REG_DWORD | Sombre (applications) | Clair (applications) |
| `SystemUsesLightTheme` | REG_DWORD | Sombre (barre des taches) | Clair (barre des taches) |

!!! example "Activer le theme sombre complet"
    ```batch
    rem Set dark mode for apps
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize" /v AppsUseLightTheme /t REG_DWORD /d 0 /f

    rem Set dark mode for taskbar
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize" /v SystemUsesLightTheme /t REG_DWORD /d 0 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

    Deconnectez-vous puis reconnectez-vous pour appliquer le changement.

### Fond d'ecran

```
HKCU\Control Panel\Desktop
```

| Valeur | Description | Valeurs possibles |
|--------|-------------|-------------------|
| `WallPaper` | Chemin vers l'image | `C:\Users\Jean\Pictures\fond.jpg` |
| `WallpaperStyle` | Style d'affichage | `0` = centre, `2` = etire, `6` = ajuste, `10` = rempli, `22` = etendu |
| `TileWallpaper` | Mode mosaique | `0` = non, `1` = oui |

!!! info "En resume"
    - Theme sombre/clair : `Themes\Personalize` avec `AppsUseLightTheme` et `SystemUsesLightTheme`
    - Fond d'ecran : `Control Panel\Desktop` avec `WallPaper` et `WallpaperStyle`
    - Deconnexion/reconnexion necessaire pour appliquer les changements de theme

---

## Explorateur de fichiers

Les reglages de l'Explorateur sont stockes dans une seule cle -- c'est le **tableau de bord** de l'affichage des fichiers.

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced
```

| Valeur | Type | Description | Valeurs |
|--------|------|-------------|---------|
| `Hidden` | REG_DWORD | Fichiers caches | `1` = afficher, `2` = masquer |
| `HideFileExt` | REG_DWORD | Extensions de fichiers | `0` = afficher, `1` = masquer |
| `LaunchTo` | REG_DWORD | Page d'accueil | `1` = Ce PC, `2` = Acces rapide, `3` = Telechargements |
| `ShowCompColor` | REG_DWORD | Coloriser les fichiers compresses | `1` = oui |
| `TaskbarAl` | REG_DWORD | Alignement barre des taches (Win 11) | `0` = gauche, `1` = centre |
| `TaskbarSi` | REG_DWORD | Taille barre des taches | `0` = petit, `1` = moyen, `2` = grand |

!!! example "Afficher les extensions et fichiers caches"
    ```batch
    rem Show file extensions
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced" /v HideFileExt /t REG_DWORD /d 0 /f

    rem Show hidden files
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced" /v Hidden /t REG_DWORD /d 1 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

    Redemarrez l'Explorateur pour appliquer :

    ```batch
    taskkill /f /im explorer.exe && start explorer.exe
    ```

    ```title="Resultat attendu"
    L'Explorateur se ferme puis redemarre.
    ```

!!! info "En resume"
    - `Explorer\Advanced` controle l'affichage des fichiers caches, extensions et barre des taches
    - `HideFileExt = 0` et `Hidden = 1` sont les reglages recommandes pour les utilisateurs avances
    - Redemarrez l'Explorateur (`taskkill /f /im explorer.exe && start explorer.exe`) pour appliquer

---

## Menu contextuel

### Restaurer le menu classique (Windows 11)

Windows 11 a remplace le menu contextuel par une version simplifiee. Pour retrouver le menu complet de Windows 10 :

```
HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32
```

Creez cette cle avec la valeur par defaut vide (`""`) :

```batch
rem Restore the classic context menu in Windows 11
reg add "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32" /ve /t REG_SZ /d "" /f
```

```title="Resultat attendu"
L'operation a reussi.
```

Redemarrez l'Explorateur pour appliquer.

!!! tip "Pour revenir au menu Windows 11"
    ```batch
    rem Remove the override to restore the Windows 11 menu
    reg delete "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}" /f
    ```

    ```title="Resultat attendu"
    The operation completed successfully.
    ```

### Ajouter une entree au menu contextuel

Vous pouvez ajouter vos propres actions au clic droit -- comme ajouter un raccourci sur votre bureau, mais directement dans le menu.

```
HKCR\*\shell\MonAction
    (Par defaut) = "Mon action personnalisee"
    Icon = "chemin\vers\icone.ico"

HKCR\*\shell\MonAction\command
    (Par defaut) = "chemin\vers\app.exe \"%1\""
```

!!! example "Ajouter 'Ouvrir avec Notepad++' au menu contextuel"
    ```batch
    rem Create the menu entry
    reg add "HKCR\*\shell\NotepadPP" /ve /t REG_SZ /d "Ouvrir avec Notepad++" /f
    reg add "HKCR\*\shell\NotepadPP" /v Icon /t REG_SZ /d "C:\Program Files\Notepad++\notepad++.exe,0" /f

    rem Create the command
    reg add "HKCR\*\shell\NotepadPP\command" /ve /t REG_SZ /d "\"C:\Program Files\Notepad++\notepad++.exe\" \"%%1\"" /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

!!! info "En resume"
    - Menu classique Windows 11 : creer la cle `InprocServer32` avec valeur vide sous le CLSID dedie
    - Ajouter une entree personnalisee : creer une sous-cle sous `HKCR\*\shell\` avec une sous-cle `command`
    - Redemarrer l'Explorateur pour appliquer les modifications

---

## Reseau et Internet

### Desactiver IPv6

```
HKLM\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters
```

| Valeur | Type | Description |
|--------|------|-------------|
| `DisabledComponents` | REG_DWORD | `0xFF` = desactiver IPv6 sur toutes les interfaces |

```batch
rem Disable IPv6 on all interfaces
reg add "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" /v DisabledComponents /t REG_DWORD /d 0xFF /f
```

```title="Resultat attendu"
L'operation a reussi.
```

Un redemarrage est necessaire pour appliquer.

### Configuration DNS

```
HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters
```

| Valeur | Type | Description |
|--------|------|-------------|
| `SearchList` | REG_SZ | Suffixes DNS de recherche (separes par des virgules) |
| `UseDomainNameDevolution` | REG_DWORD | `1` = activer la devolution de nom DNS |

!!! info "En resume"
    - IPv6 : `DisabledComponents` dans `Tcpip6\Parameters` (`0xFF` pour desactiver, redemarrage requis)
    - DNS : `SearchList` dans `Tcpip\Parameters` pour les suffixes de recherche

---

## Securite

### UAC (Controle de compte d'utilisateur)

L'UAC est le **vigile** de Windows : il demande confirmation avant toute action administrative.

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
```

| Valeur | Type | Description |
|--------|------|-------------|
| `EnableLUA` | REG_DWORD | `1` = UAC active, `0` = desactive |
| `ConsentPromptBehaviorAdmin` | REG_DWORD | Comportement de l'invite pour les administrateurs |
| `PromptOnSecureDesktop` | REG_DWORD | `1` = invite sur le bureau securise |

**Valeurs de `ConsentPromptBehaviorAdmin`** :

| Valeur | Comportement |
|--------|-------------|
| `0` | Elever sans demander (dangereux) |
| `1` | Demander les identifiants sur le bureau securise |
| `2` | Demander le consentement sur le bureau securise |
| `3` | Demander les identifiants |
| `4` | Demander le consentement |
| `5` | Demander le consentement pour les binaires non-Windows (defaut) |

!!! danger "Ne desactivez jamais l'UAC"
    Mettre `EnableLUA` a `0` desactive toute protection contre l'elevation de privileges. C'est comme retirer la serrure de votre porte d'entree.

### Windows Defender

```
HKLM\SOFTWARE\Policies\Microsoft\Windows Defender
```

| Valeur | Type | Description |
|--------|------|-------------|
| `DisableAntiSpyware` | REG_DWORD | Valeur legacy historiquement utilisee pour desactiver Defender ; a surveiller, mais non fiable sur les plateformes modernes |

!!! danger "Cle surveillee par la protection anti-sabotage"
    Cette cle est souvent exploitee par les malwares, mais elle doit etre lue comme un **indicateur** de sabotage ou de configuration obsolete, pas comme une preuve suffisante que Defender est desactive. Sur les versions recentes de Windows, Tamper Protection la bloque souvent et Microsoft la traite comme un mecanisme legacy sur de nombreux endpoints modernes.

### Strategie de mots de passe

```
HKLM\SAM\SAM\Domains\Account
```

!!! warning "Acces restreint"
    La ruche SAM est protegee par des permissions strictes. Meme en tant qu'administrateur, vous devez modifier les permissions manuellement dans Regedit pour y acceder. Utilisez plutot les outils dedies (`net accounts`, `secpol.msc`) pour gerer les strategies de mots de passe.

!!! info "En resume"
    - **Systeme** : `CurrentVersion` pour la version, `ComputerName` pour le nom
    - **Demarrage** : `Run` / `RunOnce` sous `HKCU` (utilisateur) et `HKLM` (machine)
    - **Applications** : trois emplacements `Uninstall` (64 bits, 32 bits, utilisateur)
    - **Apparence** : `Themes\Personalize` pour le mode sombre, `Explorer\Advanced` pour l'Explorateur
    - **Reseau** : `Tcpip6\Parameters` pour IPv6, `Tcpip\Parameters` pour DNS
    - **Securite** : `Policies\System` pour l'UAC, `Windows Defender` pour l'antivirus
