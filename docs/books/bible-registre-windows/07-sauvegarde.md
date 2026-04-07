---
tags:
  - registre
  - sauvegarde
  - restauration
---

# Sauvegarde et restauration

## Ce que vous allez apprendre

- Exporter des branches du registre au format `.reg` ou en copie binaire
- Restaurer le registre depuis un fichier de sauvegarde
- Sauver un systeme qui ne demarre plus grace a la restauration hors ligne
- Charger et modifier une ruche sans demarrer le systeme concerne

---

## Exporter au format .reg

Pensez au format `.reg` comme a une **photocopie** de votre registre : facile a faire, facile a relire, mais sans les tampons officiels (les permissions).

=== "Regedit"

    1. Selectionnez la cle a exporter
    2. **Fichier** > **Exporter**
    3. Choisissez la plage d'exportation (branche selectionnee ou tout le registre)
    4. Enregistrez le fichier `.reg`

=== "Ligne de commande"

    ```batch
    rem Export a specific branch
    reg export "HKCU\Software\MonApp" "%USERPROFILE%\Desktop\monapp_backup.reg" /y
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

    ```batch
    rem Export the entire HKCU hive
    reg export HKCU "%USERPROFILE%\Desktop\hkcu_backup.reg" /y
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

!!! warning "Le format .reg ne conserve pas les permissions"
    Un fichier `.reg` ne sauvegarde **pas** les ACL (listes de controle d'acces) des cles. C'est comme photocopier un document sans son tampon notarie : le contenu est la, mais pas les droits. Pour une sauvegarde complete, utilisez `reg save`.

!!! info "En resume"
    - `reg export` cree un fichier `.reg` lisible en texte brut
    - Utilisable via Regedit (menu Exporter) ou en ligne de commande
    - Les permissions (ACL) ne sont **pas** conservees dans ce format

---

## Sauvegarde binaire avec `reg save`

La commande `reg save` cree une **copie conforme** de la ruche : contenu + permissions. C'est l'equivalent numerique d'un coffre-fort plutot que d'une photocopie.

```batch
rem Save the SYSTEM hive (requires administrator rights)
reg save "HKLM\SYSTEM" "%USERPROFILE%\Desktop\SYSTEM.hiv" /y
```

```title="Resultat attendu"
L'operation a reussi.
```

```batch
rem Save the SOFTWARE hive
reg save "HKLM\SOFTWARE" "%USERPROFILE%\Desktop\SOFTWARE.hiv" /y

rem Save the current user hive
reg save "HKCU" "%USERPROFILE%\Desktop\NTUSER.hiv" /y
```

```title="Resultat attendu"
L'operation a reussi.
L'operation a reussi.
```

!!! tip "Quand utiliser quel format ?"
    | Situation | Format | Commande |
    |-----------|--------|----------|
    | Sauvegarder les preferences d'une application | `.reg` | `reg export` |
    | Sauvegarder une ruche entiere avec ses permissions | `.hiv` | `reg save` |
    | Transporter un reglage entre deux PC | `.reg` | `reg export` |
    | Preparer une restauration systeme complete | `.hiv` | `reg save` |

!!! info "En resume"
    - `reg save` cree une copie binaire complete (contenu + permissions)
    - Necessite des droits administrateur
    - Format `.hiv` pour les sauvegardes de ruches entieres, `.reg` pour les branches specifiques

---

## Points de restauration systeme

Windows cree automatiquement des **instantanes** du registre (comme des sauvegardes automatiques dans un jeu video) :

- Avant l'installation de pilotes
- Avant certaines mises a jour Windows
- Manuellement via **Creer un point de restauration** dans les parametres

Les copies sont stockees dans `%SystemRoot%\System32\config\RegBack\`.

!!! danger "Desactive par defaut depuis Windows 10 version 1803"
    Microsoft a desactive la sauvegarde automatique dans `RegBack` a partir de Windows 10 1803. Le dossier existe mais reste vide.

!!! tip "Reactiver RegBack"
    ```
    Cle    : HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Configuration Manager
    Valeur : EnablePeriodicBackup
    Type   : REG_DWORD
    Donnee : 1
    ```

    Verifiez avec :

    ```batch
    reg query "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Configuration Manager" /v EnablePeriodicBackup
    ```

    ```title="Resultat attendu"
    HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Configuration Manager
        EnablePeriodicBackup    REG_DWORD    0x1
    ```

    Un redemarrage est necessaire. La tache planifiee `RegIdleBackup` effectuera alors les sauvegardes.

!!! info "En resume"
    - **`.reg`** = photocopie lisible, sans permissions
    - **`.hiv`** = copie conforme binaire, avec permissions
    - **Points de restauration** = sauvegardes automatiques (desactivees par defaut depuis 1803)
    - Activez `EnablePeriodicBackup` pour retrouver les sauvegardes dans `RegBack`

---

## DISM et la sauvegarde du registre

DISM (Deployment Image Servicing and Management) est l'outil de Microsoft pour capturer et deployer des images systeme. Quand vous capturez une image avec DISM, le registre est inclus **integralement** -- ruches, permissions et tout le reste.

### Capturer une image systeme

```batch
rem Capture a full system image (run from WinPE or recovery environment)
dism /Capture-Image /ImageFile:D:\Backup\system.wim /CaptureDir:C:\ /Name:"Backup complet" /Compress:max
```

```title="Resultat attendu"
Outil Gestion et maintenance des images de deploiement
Version : 10.0.22621.1

Capture de l'image
[==========================100.0%==========================]
L'operation a reussi.
```

Cette commande cree un fichier `.wim` contenant la totalite du volume `C:\`, y compris `C:\Windows\System32\config\` ou resident les ruches.

### Restaurer une image systeme

```batch
rem Apply the captured image to a volume
dism /Apply-Image /ImageFile:D:\Backup\system.wim /Index:1 /ApplyDir:C:\
```

```title="Resultat attendu"
Outil Gestion et maintenance des images de deploiement
Version : 10.0.22621.1

Application de l'image
[==========================100.0%==========================]
L'operation a reussi.
```

!!! warning "DISM ne cible pas le registre seul"
    DISM capture ou restaure un **volume entier**. Il n'est pas possible de n'extraire que le registre depuis un `.wim` sans monter l'image au prealable. Pour une restauration ciblee du registre, preferez `reg save` / `reg restore`.

### Montage hors ligne et registre

DISM peut monter une image pour la modifier sans l'appliquer :

```batch
rem Mount an image for offline editing
dism /Mount-Wim /WimFile:D:\Backup\system.wim /Index:1 /MountDir:C:\Mount

rem The hive files are now accessible at C:\Mount\Windows\System32\config\
rem You can load them with reg load for modification

rem Unmount and save changes
dism /Unmount-Wim /MountDir:C:\Mount /Commit
```

```title="Resultat attendu"
Outil Gestion et maintenance des images de deploiement
Version : 10.0.22621.1

Montage de l'image
[==========================100.0%==========================]
L'operation a reussi.
```

Sous le capot, DISM monte le systeme de fichiers de l'image et donne acces aux fichiers de ruche. Vous pouvez alors utiliser `reg load` pour modifier le registre de l'image hors ligne.

### Comparaison des methodes de sauvegarde

| Methode | Portee | Permissions | Format | Restauration granulaire |
|---------|--------|-------------|--------|-------------------------|
| `reg export` | Branche specifique | Non | `.reg` (texte) | Oui (par valeur) |
| `reg save` | Ruche entiere | Oui | `.hiv` (binaire) | Non (tout ou rien) |
| DISM | Volume complet | Oui | `.wim` (image) | Non (volume entier) |
| Point de restauration | Systeme | Oui | Interne | Non (rollback global) |

!!! info "En resume"
    - DISM capture un volume entier (registre inclus) au format `.wim`
    - Il n'est pas possible d'extraire le registre seul sans monter l'image
    - Le montage hors ligne (`/Mount-Wim`) donne acces aux fichiers de ruche pour modification via `reg load`

---

## VSS (Volume Shadow Copy Service) et le registre

Le service VSS cree des **instantanes coherents** du volume, y compris les fichiers de ruche, meme s'ils sont verrouilles par le systeme. C'est comme photographier une page en cours d'ecriture sans que la plume ne bouge.

### Comment VSS capture le registre

Quand un instantane VSS est declenche, le Configuration Manager (le composant noyau qui gere le registre) vide ses tampons sur disque via un **writer VSS** dedie. Cela garantit que les fichiers de ruche dans l'instantane sont dans un etat coherent.

### Creer et lister des instantanes

```batch
rem Create a shadow copy of volume C:
vssadmin create shadow /for=C:
```

```title="Resultat attendu"
Shadow Copy ID: {12345678-abcd-1234-abcd-123456789abc}
Shadow Copy Volume Name: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1
```

```batch
rem List existing shadow copies
vssadmin list shadows
```

```title="Resultat attendu"
vssadmin 1.1 - Outil en ligne de commande d'administration du service de cliches instantanes du volume

Contenu de l'ID du jeu de copies shadow : {abcdef12-3456-7890-abcd-ef1234567890}
   ID du cliche instantane : {12345678-abcd-1234-abcd-123456789abc}
      Volume d'origine : (C:)\\?\Volume{...}\
      Volume du cliche instantane : \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1
      Ordinateur d'origine : DESKTOP-PC
      Ordinateur de service : DESKTOP-PC
      Fournisseur : 'Microsoft Software Shadow Copy provider 1.0'
      Type : ClientAccessible
```

### Acceder aux ruches depuis un instantane

```powershell
# List available shadow copies
$shadows = Get-WmiObject Win32_ShadowCopy
$shadows | Select-Object InstallDate, DeviceObject, VolumeName | Format-Table

# Create a symbolic link to access the shadow copy contents
$latestShadow = ($shadows | Sort-Object InstallDate -Descending | Select-Object -First 1).DeviceObject
cmd /c mklink /d C:\ShadowMount "$latestShadow\"

# The hive files are now accessible at:
# C:\ShadowMount\Windows\System32\config\SYSTEM
# C:\ShadowMount\Windows\System32\config\SOFTWARE
# Load them with reg load for inspection

# Clean up the symbolic link when done
cmd /c rmdir C:\ShadowMount
```

```title="Resultat attendu"
InstallDate           DeviceObject                                               VolumeName
-----------           ------------                                               ----------
20260401120000.000000 \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1            \\?\Volume{...}\
Junction created for C:\ShadowMount <<===>> \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\
```

### Logiciels de sauvegarde entreprise

Les solutions professionnelles comme Veeam, Commvault ou Azure Backup s'appuient sur VSS pour garantir la coherence du registre :

| Solution | Methode | Coherence registre |
|----------|---------|-------------------|
| Veeam Backup | VSS snapshot + CBT (Changed Block Tracking) | Garantie par le writer VSS |
| Commvault | VSS snapshot + agent systeme | Garantie par le writer VSS |
| Azure Backup | VSS snapshot + transfert vers Azure Recovery Vault | Garantie par le writer VSS |
| Windows Server Backup | VSS snapshot integre | Garantie par le writer VSS |

!!! info "VSS et les fichiers de ruche verrouilles"
    Le systeme verrouille en permanence les fichiers de ruche (`SYSTEM`, `SOFTWARE`, `SAM`, etc.). Les outils de copie classiques ne peuvent pas les lire. VSS contourne cette limitation en capturant un instantane au niveau du volume, ce qui inclut les fichiers verrouilles dans leur etat coherent.

!!! info "En resume"
    - VSS capture des instantanes coherents du volume, y compris les ruches verrouillees
    - Le writer VSS du Configuration Manager garantit l'integrite des fichiers de ruche
    - Les solutions de sauvegarde entreprise (Veeam, Azure Backup, etc.) s'appuient toutes sur VSS

---

## Restaurer depuis un fichier .reg

Restaurer un `.reg`, c'est comme **coller par-dessus** : les valeurs existantes sont ecrasees, mais les autres cles ne sont pas touchees.

=== "Double-clic"

    Double-cliquez sur le fichier `.reg` et confirmez l'import dans la boite de dialogue.

=== "Ligne de commande"

    ```batch
    rem Silent import
    reg import monapp_backup.reg
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

=== "Regedit"

    **Fichier** > **Importer** > selectionnez le fichier `.reg`.

!!! info "En resume"
    - L'import `.reg` ecrase les valeurs existantes mais ne supprime pas les autres cles
    - Trois methodes : double-clic, `reg import` en ligne de commande, ou menu Fichier de Regedit

---

## Restaurer une ruche binaire

```batch
rem Restore a hive (the hive must not be in use)
reg restore "HKLM\SYSTEM" SYSTEM.hiv
```

```title="Resultat attendu"
L'operation a reussi.
```

!!! danger "Operation destructrice"
    `reg restore` **remplace integralement** le contenu de la cle cible. Imaginez coller un ancien disque dur a la place de l'actuel : tout ce qui a change depuis la sauvegarde est perdu. Sur une ruche systeme, cela peut rendre la machine instable.

!!! info "En resume"
    - `reg restore` remplace integralement la cle cible avec le contenu du fichier `.hiv`
    - Operation destructrice : toutes les modifications depuis la sauvegarde sont perdues

---

## Restauration hors ligne (quand Windows ne demarre plus)

Quand votre PC refuse de demarrer, c'est comme une voiture en panne : il faut passer par le garage (l'environnement de recuperation) pour intervenir.

```mermaid
graph LR
    A[PC ne demarre pas] --> B[Demarrer sur USB / WinRE]
    B --> C[Ouvrir invite de commandes]
    C --> D[Identifier la lettre du disque]
    D --> E[Remplacer les ruches corrompues]
    E --> F[Redemarrer normalement]
```

**Etape 1** -- Demarrez sur le support de recuperation (WinRE, cle USB d'installation ou live CD).

**Etape 2** -- Ouvrez une invite de commandes et identifiez la lettre du lecteur Windows (souvent `D:` en mode recuperation, pas `C:`).

**Etape 3** -- Sauvegardez les ruches corrompues avant de les remplacer :

```batch
rem Back up the corrupted hives first
copy D:\Windows\System32\config\SYSTEM D:\Windows\System32\config\SYSTEM.broken
copy D:\Windows\System32\config\SOFTWARE D:\Windows\System32\config\SOFTWARE.broken
```

```title="Resultat attendu"
        1 fichier(s) copie(s).
        1 fichier(s) copie(s).
```

**Etape 4** -- Restaurez depuis RegBack (si disponible) :

```batch
rem Restore from RegBack
copy D:\Windows\System32\config\RegBack\SYSTEM D:\Windows\System32\config\SYSTEM
copy D:\Windows\System32\config\RegBack\SOFTWARE D:\Windows\System32\config\SOFTWARE
```

```title="Resultat attendu"
        1 fichier(s) copie(s).
```

!!! info "Comment trouver la bonne lettre de lecteur ?"
    En mode recuperation, les lettres de lecteur sont souvent differentes. Tapez `diskpart`, puis `list volume` pour voir la correspondance. Le volume Windows est generalement celui avec le dossier `Windows\System32`.

!!! info "En resume"
    - Demarrez sur un support de recuperation (WinRE ou USB) pour acceder aux fichiers de ruche
    - La lettre du lecteur Windows est souvent differente en mode recuperation (utiliser `diskpart` pour la trouver)
    - Sauvegardez les ruches corrompues avant de les remplacer depuis `RegBack`

---

## Charger une ruche hors ligne

Charger une ruche hors ligne, c'est comme **ouvrir le coffre-fort d'un autre PC sur votre bureau** : vous pouvez voir et modifier son contenu sans demarrer la machine.

=== "Regedit"

    1. Selectionnez `HKEY_LOCAL_MACHINE` ou `HKEY_USERS`
    2. **Fichier** > **Charger la ruche**
    3. Selectionnez le fichier de ruche (`NTUSER.DAT`, `SYSTEM`, etc.)
    4. Donnez un nom temporaire a la ruche montee
    5. Apres modification : **Fichier** > **Decharger la ruche**

=== "Ligne de commande"

    ```batch
    rem Load a hive under a temporary name
    reg load "HKU\OfflineUser" "D:\Users\Jean\NTUSER.DAT"
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

    ```batch
    rem Make changes
    reg add "HKU\OfflineUser\Software\MonApp" /v Activer /t REG_DWORD /d 0 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

    ```batch
    rem Unload the hive (mandatory to release the file)
    reg unload "HKU\OfflineUser"
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

!!! warning "Toujours decharger la ruche apres modification"
    Le fichier reste **verrouille** tant que la ruche est montee. Si vous oubliez de la decharger, l'utilisateur concerne ne pourra pas se connecter -- son `NTUSER.DAT` sera bloque.

!!! info "En resume"
    - **Import `.reg`** = colle les valeurs par-dessus l'existant (non destructif)
    - **`reg restore`** = remplacement total de la cle (destructif)
    - **Restauration hors ligne** = derniere chance quand Windows ne demarre plus
    - **Chargement de ruche** = permet de modifier le registre d'un autre utilisateur ou systeme
    - **Toujours** decharger la ruche apres utilisation (`reg unload`)

---

## Strategies de sauvegarde en entreprise

En environnement professionnel, la sauvegarde du registre ne doit pas etre un reflexe ponctuel mais un **processus planifie et documente**.

### Frequence de sauvegarde

| Contexte | Frequence recommandee |
|----------|----------------------|
| Postes de travail standards | Hebdomadaire |
| Serveurs de production | Quotidienne |
| Avant toute modification manuelle | Systematique (avant/apres) |
| Avant deploiement de GPO ou patches | Systematique |

### Quelles ruches sauvegarder ?

| Ruche | Fichier | Contenu critique |
|-------|---------|------------------|
| SYSTEM | `%SystemRoot%\System32\config\SYSTEM` | Pilotes, services, configuration materielle |
| SOFTWARE | `%SystemRoot%\System32\config\SOFTWARE` | Applications installees, parametres systeme |
| SAM | `%SystemRoot%\System32\config\SAM` | Comptes utilisateurs locaux et mots de passe |
| SECURITY | `%SystemRoot%\System32\config\SECURITY` | Strategies de securite locales, secrets LSA |
| DEFAULT | `%SystemRoot%\System32\config\DEFAULT` | Profil par defaut des nouveaux utilisateurs |
| NTUSER.DAT | `%UserProfile%\NTUSER.DAT` | Preferences de chaque utilisateur (un par profil) |

### Ou stocker les sauvegardes ?

| Destination | Avantage | Inconvenient |
|-------------|----------|--------------|
| Dossier local | Rapide, disponible hors reseau | Perdu si le disque tombe en panne |
| Partage reseau (UNC) | Protege contre la panne locale | Necessite une connectivite reseau |
| Azure Blob Storage | Haute disponibilite, geo-redondance | Necessite une configuration initiale |
| Support amovible | Isolation totale (air gap) | Gestion manuelle |

### Politique de retention

Conservez au minimum **3 generations** de sauvegardes. En cas de corruption silencieuse, la derniere sauvegarde peut elle-meme etre corrompue. Avoir plusieurs points de restauration dans le temps permet de remonter a un etat sain.

### Script d'automatisation PowerShell

```powershell
# Automated registry backup script
# Schedule via Task Scheduler for daily execution

$backupRoot  = "\\FileServer\Backups\Registry\$env:COMPUTERNAME"
$dateStamp   = Get-Date -Format "yyyy-MM-dd"
$backupDir   = Join-Path $backupRoot $dateStamp
$hivePath    = "$env:SystemRoot\System32\config"
$hiveNames   = @("SYSTEM", "SOFTWARE", "SAM", "SECURITY", "DEFAULT")
$retainDays  = 21

# Create the backup directory
New-Item -ItemType Directory -Path $backupDir -Force | Out-Null

# Export each hive
foreach ($hive in $hiveNames) {
    $dest = Join-Path $backupDir "$hive.hiv"
    reg save "HKLM\$hive" $dest /y 2>&1 | Out-Null
    Write-Output "Saved $hive -> $dest"
}

# Backup current user hive
$ntuserDest = Join-Path $backupDir "NTUSER.hiv"
reg save "HKCU" $ntuserDest /y 2>&1 | Out-Null
Write-Output "Saved HKCU -> $ntuserDest"

# Purge old backups beyond retention period
Get-ChildItem $backupRoot -Directory | Where-Object {
    $_.CreationTime -lt (Get-Date).AddDays(-$retainDays)
} | Remove-Item -Recurse -Force

Write-Output "Backup complete. Retention: $retainDays days."
```

```title="Resultat attendu"
Saved SYSTEM -> \\FileServer\Backups\Registry\DESKTOP-PC\2026-04-04\SYSTEM.hiv
Saved SOFTWARE -> \\FileServer\Backups\Registry\DESKTOP-PC\2026-04-04\SOFTWARE.hiv
Saved SAM -> \\FileServer\Backups\Registry\DESKTOP-PC\2026-04-04\SAM.hiv
Saved SECURITY -> \\FileServer\Backups\Registry\DESKTOP-PC\2026-04-04\SECURITY.hiv
Saved DEFAULT -> \\FileServer\Backups\Registry\DESKTOP-PC\2026-04-04\DEFAULT.hiv
Saved HKCU -> \\FileServer\Backups\Registry\DESKTOP-PC\2026-04-04\NTUSER.hiv
Backup complete. Retention: 21 days.
```

!!! tip "Planification avec le Planificateur de taches"
    Enregistrez ce script sous `C:\Scripts\Backup-Registry.ps1` et creez une tache planifiee :

    ```batch
    schtasks /create /tn "RegistryBackup" /tr "powershell -ExecutionPolicy Bypass -File C:\Scripts\Backup-Registry.ps1" /sc daily /st 02:00 /ru SYSTEM /rl HIGHEST
    ```

    ```title="Resultat attendu"
    REUSSITE : la tache planifiee "RegistryBackup" a ete creee.
    ```

### Conformite et audit

Certaines reglementations imposent la conservation d'instantanes du registre dans le cadre de pistes d'audit :

| Reglementation | Exigence liee au registre |
|----------------|--------------------------|
| PCI-DSS | Journalisation des modifications de configuration systeme |
| SOX (Sarbanes-Oxley) | Tracabilite des changements de parametres financiers |
| RGPD / GDPR | Prouver que les parametres de securite etaient appliques a un instant T |
| ISO 27001 | Gestion des changements documentee et reversible |

!!! info "Combiner sauvegarde et audit"
    La sauvegarde du registre seule ne suffit pas pour la conformite. Associez-la a l'audit SACL (voir le chapitre [Optimisation et maintenance](09-optimisation.md#audit-du-registre-via-la-sacl)) pour disposer a la fois d'un historique des etats et d'un journal des modifications.

!!! info "En resume"
    - Planifiez des sauvegardes regulieres (quotidiennes pour les serveurs, hebdomadaires pour les postes)
    - Sauvegardez toutes les ruches critiques : SYSTEM, SOFTWARE, SAM, SECURITY, DEFAULT, NTUSER.DAT
    - Conservez au minimum 3 generations de sauvegardes pour couvrir les corruptions silencieuses
    - Combinez la sauvegarde avec l'audit SACL pour la conformite reglementaire
