---
tags:
  - registre
  - optimisation
  - maintenance
---

# Optimisation et maintenance

## Ce que vous allez apprendre

- Evaluer la taille de vos ruches et detecter un gonflement anormal
- Nettoyer le registre en toute securite (et pourquoi les "nettoyeurs" sont inutiles)
- Surveiller les modifications du registre en temps reel
- Compacter une ruche pour recuperer l'espace perdu

---

## Taille du registre

Le registre est charge **integralement en memoire** au demarrage. Pensez-y comme une valise : plus elle est lourde, plus il faut de temps pour la soulever (demarrage) et plus elle prend de place dans le coffre (RAM).

### Verifier la taille des ruches

```powershell
# List hive file sizes
$hivePath = "$env:SystemRoot\System32\config"
Get-ChildItem $hivePath -File | Where-Object {
    $_.Name -match "^(SYSTEM|SOFTWARE|SAM|SECURITY|DEFAULT)$"
} | Select-Object Name, @{N="SizeMB"; E={[math]::Round($_.Length / 1MB, 2)}}
```

```title="Resultat attendu"
Name     SizeMB
----     ------
DEFAULT    0.26
SAM        0.05
SECURITY   0.03
SOFTWARE  87.50
SYSTEM    18.75
```

### Tailles typiques sur une installation propre

| Ruche | Taille normale | Seuil d'alerte |
|-------|---------------|----------------|
| SOFTWARE | 50 -- 150 Mo | > 200 Mo |
| SYSTEM | 10 -- 40 Mo | > 60 Mo |
| NTUSER.DAT | 2 -- 20 Mo | > 50 Mo |
| SAM | < 1 Mo | > 5 Mo |
| SECURITY | < 1 Mo | > 5 Mo |

### Pourquoi le registre gonfle-t-il ?

Comme un tiroir qui accumule les vieux papiers, le registre grossit au fil du temps :

- **Applications mal desinstallees** : cles orphelines sous `HKLM\SOFTWARE` et `HKCU\Software`
- **Pilotes accumules** : entrees obsoletes dans `HKLM\SYSTEM\CurrentControlSet\Enum`
- **Profils utilisateurs** : donnees temporaires stockees dans `NTUSER.DAT`
- **Mises a jour Windows** : cles de migration et de compatibilite empilees

!!! info "En resume"
    - Le registre est charge en RAM : taille = impact sur les performances
    - Verifiez regulierement avec le script PowerShell ci-dessus
    - Un `SOFTWARE` > 200 Mo ou un `NTUSER.DAT` > 50 Mo meritent attention

---

## Nettoyage du registre

!!! danger "Toujours sauvegarder avant de nettoyer"
    Supprimer une cle en cours d'utilisation peut provoquer des dysfonctionnements graves. Exportez la branche concernee avec `reg export` **avant** toute suppression.

### Les seules operations de nettoyage recommandees

Le nettoyage du registre, c'est comme la chirurgie : on n'opere que quand c'est necessaire, et on sait exactement ce qu'on fait.

| Operation | Methode |
|-----------|---------|
| Application mal desinstallee | Utiliser l'outil de desinstallation officiel de l'editeur |
| Profils utilisateurs orphelins | **Parametres** > **Systeme** > **Comptes** > **Profils utilisateurs** |
| Programmes au demarrage obsoletes | Verifier `HKCU\...\Run` et `RunOnce` |
| Associations de fichiers cassees | **Parametres** > **Applications** > **Applications par defaut** |

!!! example "Verifier les programmes au demarrage"
    ```batch
    rem List user startup entries
    reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"
    ```

    ```title="Resultat attendu"
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
        Discord    REG_SZ    "C:\Users\User\AppData\Local\Discord\Update.exe" --processStart Discord.exe
        Steam      REG_SZ    "C:\Program Files (x86)\Steam\steam.exe" /silentlogin
    ```

    Si une entree pointe vers un programme desinstalle, supprimez-la :

    ```batch
    rem Remove a specific startup entry
    reg delete "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v NomObsolete /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

### Les "nettoyeurs de registre" : oubliez-les

Les utilitaires comme CCleaner, Wise Registry Cleaner, etc. :

- Offrent un gain de performance **negligeable voire nul** sur les systemes modernes
- **Risquent de supprimer** des cles necessaires au fonctionnement du systeme
- Ne sont **pas recommandes** par Microsoft

!!! quote "Position officielle de Microsoft"
    Microsoft ne supporte pas l'utilisation de nettoyeurs de registre et ne peut pas garantir que les problemes resultant de l'utilisation de ces utilitaires puissent etre resolus.

!!! info "En resume"
    - Nettoyez uniquement de maniere ciblee (desinstallation propre, entrees de demarrage obsoletes)
    - Les nettoyeurs tiers sont inutiles et potentiellement dangereux
    - Toujours sauvegarder avant toute suppression

---

## Cles de gestion memoire et performance systeme

La cle suivante controle le comportement de la memoire virtuelle et du gestionnaire de memoire du noyau :

```
HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management
```

| Valeur | Type | Description | Defaut |
|--------|------|-------------|--------|
| `ClearPageFileAtShutdown` | REG_DWORD | Efface le fichier d'echange a l'arret (`1` = oui). Empeche qu'un attaquant recupere des donnees sensibles depuis le pagefile. | 0 |
| `DisablePagingExecutive` | REG_DWORD | Empeche le noyau d'etre pagine sur disque (`1` = tout en RAM). Ameliore la reactivite au prix d'une utilisation RAM accrue. | 0 |
| `LargeSystemCache` | REG_DWORD | Alloue plus de memoire au cache du systeme de fichiers (`1` = mode serveur). Utile pour les serveurs de fichiers. | 0 |
| `NonPagedPoolSize` | REG_DWORD | Taille du pool non pagine en octets. La valeur `0` laisse Windows calculer automatiquement. | 0 (auto) |
| `PagedPoolSize` | REG_DWORD | Taille du pool pagine en octets. La valeur `0` laisse Windows calculer automatiquement. | 0 (auto) |
| `IoPageLockLimit` | REG_DWORD | Nombre maximal d'octets verrouilles en memoire pour les E/S. La valeur `0` utilise la formule interne. | 0 (auto) |

!!! warning "Ne modifiez pas les tailles de pool manuellement"
    Les valeurs `NonPagedPoolSize`, `PagedPoolSize` et `IoPageLockLimit` sont auto-calculees par le noyau en fonction de la RAM disponible. Les modifier manuellement avec des valeurs inadaptees peut provoquer des ecrans bleus (`DRIVER_IRQL_NOT_LESS_OR_EQUAL`) ou une degradation des performances.

```powershell
# Check current memory management values
$path = "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management"
Get-ItemProperty $path | Select-Object ClearPageFileAtShutdown,
    DisablePagingExecutive, LargeSystemCache | Format-List
```

```title="Resultat attendu"
ClearPageFileAtShutdown : 0
DisablePagingExecutive  : 0
LargeSystemCache        : 0
```

!!! tip "Recommandation pour les postes de travail avec beaucoup de RAM"
    Sur une machine disposant de 32 Go de RAM ou plus, activer `DisablePagingExecutive` (`1`) peut ameliorer la reactivite du systeme en empechant le noyau d'etre deplace vers le fichier d'echange. Sur les machines avec moins de 16 Go, laissez la valeur par defaut.

!!! info "En resume"
    - La cle `Memory Management` controle le fichier d'echange, le cache systeme et les pools memoire
    - `ClearPageFileAtShutdown` et `DisablePagingExecutive` sont les seules valeurs a ajuster manuellement
    - Ne modifiez jamais les tailles de pool (`NonPagedPoolSize`, etc.) : le noyau les calcule automatiquement

---

## Prefetch, Superfetch et SysMain

Windows utilise des mecanismes de **precharge** pour accelerer le demarrage et le lancement des applications. Ces parametres sont controles par le registre.

### PrefetchParameters

```
HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management\PrefetchParameters
```

| Valeur | Type | Options |
|--------|------|---------|
| `EnablePrefetcher` | REG_DWORD | `0` = desactive, `1` = applications uniquement, `2` = demarrage uniquement, `3` = les deux |
| `EnableSuperfetch` | REG_DWORD | `0` = desactive, `1` = active |

### SysMain (nom moderne du service)

Depuis Windows 10, le service Superfetch a ete renomme **SysMain** :

```
HKLM\SYSTEM\CurrentControlSet\Services\SysMain
```

| Valeur | Type | Description |
|--------|------|-------------|
| `Start` | REG_DWORD | `2` = automatique, `3` = manuel, `4` = desactive |

### Impact selon le type de disque

| Scenario | HDD | SSD |
|----------|-----|-----|
| Prefetch (demarrage) | Gain notable (~15-30% plus rapide) | Gain marginal (SSD deja rapide) |
| Prefetch (applications) | Gain notable | Gain marginal |
| Superfetch/SysMain | Tres benefique (precharge en RAM) | Peut causer des ecritures inutiles |
| Recommandation | Activer tout (`3` + `1`) | Laisser par defaut ou desactiver SysMain |

```powershell
# Check Prefetch and Superfetch settings
$path = "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management\PrefetchParameters"
Get-ItemProperty $path | Select-Object EnablePrefetcher, EnableSuperfetch
```

```title="Resultat attendu"
EnablePrefetcher : 3
EnableSuperfetch : 1
```

!!! info "Laisser les valeurs par defaut dans la plupart des cas"
    Windows ajuste automatiquement le comportement de SysMain en fonction du type de stockage detecte. Desactiver ces mecanismes sur un HDD degrade significativement les temps de demarrage et de lancement des applications.

!!! info "En resume"
    - `EnablePrefetcher` et `EnableSuperfetch` controlent la precharge des applications et du demarrage
    - Le service SysMain (ex-Superfetch) est le nom moderne du mecanisme de precharge
    - Sur HDD : laisser active (gain significatif) ; sur SSD : desactiver SysMain peut reduire les ecritures inutiles

---

## Parametres de performance NTFS

Le systeme de fichiers NTFS expose plusieurs parametres de performance via le registre :

```
HKLM\SYSTEM\CurrentControlSet\Control\FileSystem
```

| Valeur | Type | Description | Defaut |
|--------|------|-------------|--------|
| `NtfsDisableLastAccessUpdate` | REG_DWORD | Desactive la mise a jour de l'horodatage "dernier acces" (`1` = desactive). Reduit les ecritures disque. | `0x80000003` sur Windows 10 1803+ (gestion systeme, dernier acces generalement desactive) |
| `NtfsDisable8dot3NameCreation` | REG_DWORD | Desactive la creation des noms courts 8.3 (ex : `PROGRA~1`). Ameliore les performances de creation de fichiers et economise l'espace de nommage. | Variable |
| `NtfsAllowExtendedCharacterIn8dot3Name` | REG_DWORD | Autorise les caracteres etendus dans les noms courts 8.3. | 0 |
| `NtfsMemoryUsage` | REG_DWORD | Controle la taille du jeu de travail NTFS. `1` = normal, `2` = augmente (plus de cache paginable). | 1 |

!!! note "Valeur par defaut moderne"
    Depuis Windows 10 1803, la valeur par defaut observee pour `NtfsDisableLastAccessUpdate` est souvent `0x80000003`. Ce comportement laisse Windows gerer l'horodatage "dernier acces" de facon interne et revient, en pratique, a ne plus le mettre a jour comme sur les versions anciennes.

### Valeurs recommandees selon le contexte

| Parametre | Poste de travail | Serveur de fichiers | SSD haute performance |
|-----------|------------------|---------------------|----------------------|
| `NtfsDisableLastAccessUpdate` | `1` (desactive) | `1` (desactive) | `1` (desactive) |
| `NtfsDisable8dot3NameCreation` | `1` (desactive) | `1` (desactive) | `1` (desactive) |
| `NtfsMemoryUsage` | `1` (normal) | `2` (augmente) | `1` (normal) |

```powershell
# Check NTFS performance settings
$path = "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem"
Get-ItemProperty $path | Select-Object NtfsDisableLastAccessUpdate,
    NtfsDisable8dot3NameCreation, NtfsMemoryUsage | Format-List
```

```title="Resultat attendu"
NtfsDisableLastAccessUpdate  : 1
NtfsDisable8dot3NameCreation : 1
NtfsMemoryUsage              : 1
```

!!! tip "Desactiver les noms 8.3 sur les volumes avec beaucoup de fichiers"
    Sur un volume contenant des centaines de milliers de fichiers, la generation des noms courts 8.3 peut ralentir significativement les operations de creation. Desactivez-la avec :

    ```batch
    rem Disable 8.3 name creation on all volumes
    fsutil 8dot3name set 1
    ```

    ```title="Resultat attendu"
    Le parametre 8dot3name pour le systeme a la valeur : 1 (desactive pour tous les volumes)
    ```

!!! warning "Compatibilite des noms courts"
    Certaines anciennes applications (installeurs 16 bits, scripts batch avec des chemins `~1`) dependent des noms courts 8.3. Testez avant de desactiver en production.

!!! info "En resume"
    - `NtfsDisableLastAccessUpdate` et `NtfsDisable8dot3NameCreation` reduisent les ecritures disque inutiles
    - Desactiver les noms courts 8.3 ameliore les performances sur les volumes avec beaucoup de fichiers
    - `NtfsMemoryUsage = 2` augmente le cache NTFS (utile pour les serveurs de fichiers)

---

## Priorite des processus et MMCSS

Le service **MMCSS** (Multimedia Class Scheduler Service) garantit que les applications multimedia (audio, video, jeux) obtiennent une priorite CPU suffisante pour eviter les saccades.

### SystemProfile

```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile
```

| Valeur | Type | Description | Defaut |
|--------|------|-------------|--------|
| `SystemResponsiveness` | REG_DWORD | Pourcentage de CPU **reserve aux taches non-multimedia**. Une valeur de `20` signifie que 80% du CPU est disponible pour le multimedia. | 20 |
| `NetworkThrottlingIndex` | REG_DWORD | Nombre de paquets reseau traites par milliseconde. La valeur par defaut (`10`) limite le debit pour privilegier le multimedia. Mettre `0xFFFFFFFF` pour desactiver la limitation. | 10 |

```powershell
# Check MMCSS settings
$path = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile"
Get-ItemProperty $path | Select-Object SystemResponsiveness, NetworkThrottlingIndex
```

```title="Resultat attendu"
SystemResponsiveness    : 20
NetworkThrottlingIndex  : 10
```

### Taches MMCSS

Chaque profil de tache definit la priorite CPU et E/S pour une categorie d'application :

```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\Tasks
```

| Sous-cle | Application | Priorite par defaut |
|----------|-------------|---------------------|
| `Audio` | Lecture audio, enregistrement | Haute (priorite temps reel) |
| `Games` | Jeux video | Moyenne-haute |
| `Pro Audio` | Studios d'enregistrement (DAW) | Tres haute |
| `Playback` | Lecture video | Haute |
| `Window Manager` | Compositing du bureau (DWM) | Moyenne |

Chaque sous-cle contient :

| Valeur | Type | Description |
|--------|------|-------------|
| `Affinity` | REG_DWORD | Masque d'affinite CPU (0 = tous les coeurs) |
| `Background Only` | REG_SZ | `True` = la tache ne recoit une priorite elevee qu'en arriere-plan |
| `Priority` | REG_DWORD | Priorite de la tache (1 a 8, 8 etant la plus haute) |
| `Scheduling Category` | REG_SZ | `High`, `Medium` ou `Low` |
| `SFIO Priority` | REG_SZ | Priorite E/S : `High`, `Normal` ou `Low` |

!!! tip "Optimisation pour le gaming"
    Pour maximiser la reactivite en jeu, certains utilisateurs avances ajustent `SystemResponsiveness` a `0` (tout le CPU pour le multimedia) et `NetworkThrottlingIndex` a `0xFFFFFFFF` (pas de limitation reseau). Ces modifications sont detaillees dans le chapitre [Cles non documentees](22-non-documentees.md).

!!! warning "N'ajustez MMCSS que si necessaire"
    Les valeurs par defaut conviennent a la majorite des usages. Modifier `SystemResponsiveness` a `0` peut rendre le systeme moins reactif pour les taches de fond (antivirus, indexation, mises a jour).

!!! info "En resume"
    - MMCSS garantit la priorite CPU et E/S pour les applications multimedia (audio, video, jeux)
    - `SystemResponsiveness` definit le pourcentage de CPU reserve aux taches non-multimedia
    - `NetworkThrottlingIndex` limite le debit reseau pour privilegier le multimedia
    - Les valeurs par defaut conviennent a la majorite des usages

---

## Surveiller les modifications

### Process Monitor (Sysinternals)

Process Monitor est une **camera de surveillance** pour votre registre : il capture en temps reel toutes les operations de lecture, ecriture et suppression.

1. Telechargez Process Monitor depuis Sysinternals
2. Filtrez par type d'operation : `RegSetValue`, `RegCreateKey`, `RegDeleteKey`
3. Filtrez par processus pour isoler une application specifique

!!! tip "Filtres les plus utiles"
    | Filtre | Valeur | Usage |
    |--------|--------|-------|
    | `Operation` contains `Reg` | Affiche uniquement les operations registre |
    | `Result` is `ACCESS DENIED` | Trouve les problemes de permissions |
    | `Result` is `NAME NOT FOUND` | Trouve les cles manquantes |
    | `Process Name` is `app.exe` | Isole une application specifique |

### Audit du registre via la SACL

Pour un suivi permanent (pas seulement le temps d'une session Process Monitor), configurez l'audit Windows. C'est comme installer une alarme sur une porte : chaque ouverture est enregistree dans le journal.

**Etape 1** -- Activez la strategie d'audit :

```powershell
# Enable registry audit policy
auditpol /set /subcategory:"Registry" /success:enable /failure:enable
```

```title="Resultat attendu"
La commande a ete executee.
```

**Etape 2** -- Configurez l'audit sur la cle cible :

1. Ouvrez Regedit
2. Clic droit sur la cle a surveiller > **Autorisations** > **Avance** > **Audit**
3. Ajoutez une entree d'audit pour l'utilisateur ou groupe concerne
4. Selectionnez les operations a auditer (ecriture, suppression)

**Etape 3** -- Consultez les evenements dans le journal de securite :

| ID | Evenement |
|----|-----------|
| 4657 | Modification d'une valeur du registre |
| 4656 | Handle demande sur un objet du registre |
| 4663 | Tentative d'acces a un objet du registre |

!!! example "Consulter les evenements d'audit"
    ```powershell
    # View recent registry audit events
    Get-WinEvent -FilterHashtable @{LogName="Security"; Id=4657} -MaxEvents 5 |
        Select-Object TimeCreated, Message | Format-List
    ```

    ```title="Resultat attendu"
    TimeCreated : 04/04/2026 14:32:10
    Message     : Une valeur du Registre a ete modifiee.
                  Sujet : DESKTOP-PC\Admin
                  Nom de l'objet : \REGISTRY\MACHINE\SOFTWARE\MonApp
                  Nom de la valeur : Config
                  Nouvelle valeur : production
    ```

!!! info "En resume"
    - **Process Monitor** = camera en temps reel (temporaire, pendant le diagnostic)
    - **Audit SACL** = alarme permanente (enregistrement dans le journal de securite)
    - Evenement 4657 = la cle a ete modifiee

---

## Compactage des ruches

Le registre fonctionne comme un **fichier a trous** : quand vous supprimez des cles, l'espace est marque comme libre mais le fichier ne retrecit pas. C'est comme arracher des pages d'un classeur sans reduire l'epaisseur de la reliure.

Pour compacter une ruche (exporter puis reimporter) :

```batch
rem Export the hive to a new file
reg save "HKLM\SOFTWARE" software_export.hiv /y

rem Reimport the compacted hive
reg restore "HKLM\SOFTWARE" software_export.hiv
```

```title="Resultat attendu"
L'operation a reussi.
L'operation a reussi.
```

!!! warning "Operation delicate"
    Le compactage d'une ruche systeme **en cours d'utilisation** est risque. Cette operation est bien plus sure sur une ruche chargee hors ligne (voir le chapitre [Sauvegarde et restauration](07-sauvegarde.md#charger-une-ruche-hors-ligne)).

!!! tip "Methode plus sure : compactage hors ligne"
    1. Demarrez en mode recuperation
    2. Chargez la ruche avec `reg load`
    3. Exportez-la avec `reg save`
    4. Rechargez le fichier compact

!!! info "En resume"
    - La suppression de cles ne reduit pas la taille du fichier
    - Le compactage consiste a exporter puis reimporter la ruche
    - Preferez le faire hors ligne pour eviter les risques
