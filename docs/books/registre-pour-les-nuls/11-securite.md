---
tags:
  - registre
  - débutant
  - sécurité
---

# Le registre et la securite

!!! abstract "Ce que vous allez apprendre"
    - Comment les malwares utilisent le registre pour persister
    - Comment verifier que votre demarrage est propre
    - Comment controler l'etat de Windows Defender via le registre
    - Des reglages de securite basiques a appliquer
    - Comment detecter les changements non autorises
    - Les bases des permissions du registre

---

## Comment les malwares utilisent le registre

Le registre est une cible de choix pour les malwares. Pourquoi ? Parce que c'est la **memoire a long terme** de Windows. Si un malware ecrit dans le registre, sa modification survit aux redemarrages.

!!! info "L'analogie"
    Imaginez que votre PC est une maison. Le registre, c'est le carnet d'adresses de l'entree. Un malware, c'est un cambrioleur qui ajoute son nom dans le carnet pour que le gardien le laisse entrer a chaque fois.

### Technique 1 : persistance via les cles Run

La technique la plus courante. Le malware ajoute une entree dans les cles de demarrage :

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
```

A chaque demarrage de Windows, le programme malveillant se lance automatiquement.

!!! example "Exemple de cle suspecte"
    ```
    HKCU\Software\Microsoft\Windows\CurrentVersion\Run
        "SystemUpdate"    REG_SZ    C:\Users\Jean\AppData\Local\Temp\svchost.exe
    ```

    **Pourquoi c'est suspect** :

    - Le vrai `svchost.exe` se trouve dans `C:\Windows\System32`, pas dans `Temp`
    - Le nom "SystemUpdate" est generique et trompeur
    - Le dossier `Temp` est un emplacement inhabituel pour un programme legitime

---

### Technique 2 : desactivation de Windows Defender

Un malware sophistique commence par **neutraliser l'antivirus** avant de s'installer :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows Defender
    "DisableAntiSpyware" = dword:00000001

HKLM\SYSTEM\CurrentControlSet\Services\WinDefend
    "Start" = dword:00000004
```

| Modification | Effet |
|-------------|-------|
| `DisableAntiSpyware` = `1` | Desactive le moteur anti-spyware |
| `Start` = `4` | Empeche le service Defender de demarrer |

!!! danger "Tamper Protection"
    Depuis Windows 10 version 1903, **Tamper Protection** empeche les modifications de ces cles, meme par un administrateur. Si ces valeurs sont presentes et que vous n'avez pas desactive Tamper Protection vous-meme, c'est un signe de compromission.

---

### Technique 3 : dissimulation dans CLSID

Les malwares avances se cachent derriere des CLSID (identifiants uniques de composants) :

```
HKCR\CLSID\{GUID-malveillant}\InProcServer32
    (Default) = "C:\ProgramData\malware.dll"
```

Le malware s'enregistre comme un composant COM legitime. Windows le charge automatiquement quand une application utilise ce CLSID.

!!! info "Difficile a detecter"
    Parmi les milliers de CLSID du registre, reperer un CLSID malveillant a l'oeil nu est quasiment impossible. C'est pourquoi les outils automatises (antivirus, Autoruns) sont indispensables.

---

### Technique 4 : modification des associations de fichiers

Le malware redirige les ouvertures de fichiers pour executer son code :

```
HKCR\.txt\shell\open\command
    (Default) = "C:\malware\fake-notepad.exe %1"
```

A chaque ouverture d'un fichier `.txt`, c'est le malware qui s'execute au lieu du Bloc-notes.

!!! quote "En resume"
    - Les malwares utilisent le registre pour persister : cles `Run` (demarrage automatique), desactivation de Defender, dissimulation dans les CLSID, et redirection des associations de fichiers.
    - Ces modifications survivent aux redemarrages, ce qui rend le registre une cible de choix pour les programmes malveillants.

---

## Verifier votre demarrage

### Methode 1 : le Gestionnaire des taches

La methode la plus simple pour un debutant :

1. Appuyez sur ++ctrl+shift+esc++
2. Onglet **Demarrage**
3. Examinez chaque entree

| Colonne | Ce qu'il faut verifier |
|---------|----------------------|
| Nom | Est-ce un nom que vous reconnaissez ? |
| Editeur | L'editeur est-il connu (Microsoft, Intel, NVIDIA...) ? |
| Statut | `Active` signifie qu'il se lance au demarrage |
| Impact demarrage | `Eleve` peut indiquer un programme gourmand |

!!! tip "Regle pratique"
    Si un programme dans la liste n'a **pas d'editeur** ou a un editeur inconnu, recherchez son nom sur Google. Exemple : `"nom_du_programme.exe" what is it`.

---

### Methode 2 : verification manuelle des cles Run

```powershell
# Check startup entries for the current user
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"
```

```title="Resultat attendu"
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
    SecurityHealth    REG_SZ    C:\Windows\System32\SecurityHealthSystray.exe
    OneDrive          REG_SZ    "C:\Program Files\Microsoft OneDrive\OneDrive.exe" /background
```

```powershell
# Check machine-wide startup entries
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
```

```title="Resultat attendu"
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
    SecurityHealth     REG_SZ    %windir%\system32\SecurityHealthSystray.exe
    RtkAudUService     REG_SZ    "C:\Windows\System32\RtkAudUService64.exe" -background
```

Verifiez aussi les cles `RunOnce` :

```powershell
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce"
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce"
```

```title="Resultat attendu"
Les entrees RunOnce s'affichent si elles existent. Sinon, Windows indique que la cle ou la valeur est introuvable.
```

!!! warning "Signaux d'alerte dans les cles Run"
    Mefiance si vous voyez :

    - Un chemin vers `%TEMP%`, `AppData\Local\Temp`, ou `C:\ProgramData` avec un `.exe` inconnu
    - Un nom generique comme `svchost`, `csrss`, `System` (noms de processus Windows copies)
    - Un chemin avec des caracteres etranges ou des espaces inhabituels
    - Une commande PowerShell encodee (`-EncodedCommand ...`)

---

### Methode 3 : Autoruns (outil Sysinternals)

L'outil **Autoruns** de Sysinternals est le **meilleur outil gratuit** pour inspecter tout ce qui se lance au demarrage.

1. Telechargez-le : `https://learn.microsoft.com/sysinternals/downloads/autoruns`
2. Lancez `Autoruns64.exe` en administrateur
3. Cliquez sur l'onglet **Everything**

Autoruns affiche **tous** les emplacements de demarrage, pas seulement les cles `Run` :

| Onglet | Ce qu'il montre |
|--------|----------------|
| Logon | Cles Run, dossier Startup |
| Explorer | Extensions shell, barres d'outils |
| Scheduled Tasks | Taches planifiees |
| Services | Services Windows |
| Drivers | Pilotes |
| Known DLLs | DLLs systeme |

!!! tip "Detecter les entrees suspectes"
    Autoruns colore les entrees :

    - **Rose** : l'entree n'a pas de signature numerique verifiee
    - **Jaune** : le fichier reference n'existe pas sur le disque

    Les entrees roses et jaunes meritent une investigation.

    Clic droit > **Search Online** recherche automatiquement l'entree sur Google.

!!! quote "En resume"
    - Trois methodes pour verifier le demarrage : le Gestionnaire des taches (le plus simple), la verification manuelle des cles `Run`/`RunOnce`, et l'outil **Autoruns** de Sysinternals (le plus complet).
    - Mefiez-vous des chemins vers `%TEMP%` ou `AppData\Local\Temp` et des noms generiques comme `svchost` dans les cles de demarrage.

---

## Verifier l'etat de Windows Defender

### Via le registre

```powershell
# Check if Windows Defender is active
reg query "HKLM\SOFTWARE\Microsoft\Windows Defender" /v DisableAntiSpyware
```

```title="Resultat attendu"
ERREUR : le nom ou la valeur specifie est introuvable.
```

!!! tip "L'absence de la valeur est une bonne nouvelle"
    Si `DisableAntiSpyware` **n'existe pas**, cela signifie que Defender est dans son etat par defaut (actif). Si elle existe et vaut `1`, Defender est desactive.

```powershell
# Check Tamper Protection status
reg query "HKLM\SOFTWARE\Microsoft\Windows Defender\Features" /v TamperProtection
```

```title="Resultat attendu"
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows Defender\Features
    TamperProtection    REG_DWORD    0x5
```

| Valeur | Signification |
|:------:|---------------|
| `0` | Desactive |
| `1` | Active par l'utilisateur |
| `5` | Active et geree par le systeme (optimal) |

---

### Via PowerShell (plus fiable)

```powershell
# Comprehensive Defender status check
Get-MpComputerStatus | Select-Object AntivirusEnabled, RealTimeProtectionEnabled, IsTamperProtected
```

```title="Resultat attendu"
AntivirusEnabled          : True
RealTimeProtectionEnabled : True
IsTamperProtected         : True
```

Si l'un de ces trois champs est `False` et que vous n'avez rien desactive, **investiguez immediatement**.

!!! quote "En resume"
    - Verifiez Defender via le registre (`DisableAntiSpyware` doit etre absent) ou via PowerShell (`Get-MpComputerStatus`).
    - `TamperProtection` a `5` signifie que la protection anti-falsification est active et geree par le systeme (optimal).

---

## Renforcement de securite pour debutants

### Desactiver l'AutoRun

L'AutoRun execute automatiquement un programme quand vous inseez une cle USB ou un CD. C'est un vecteur d'infection classique.

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `NoDriveTypeAutoRun` | REG_DWORD | `0xFF` | Desactive l'AutoRun sur tous les lecteurs |

!!! example "Commande rapide"
    ```powershell
    reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer" /v NoDriveTypeAutoRun /t REG_DWORD /d 255 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

**Pourquoi c'est important** : en 2008, le ver Conficker s'est propage massivement via l'AutoRun des cles USB. Depuis, desactiver l'AutoRun est considere comme une mesure de base.

---

### Desactiver le Bureau a distance (si vous ne l'utilisez pas)

Le Bureau a distance (RDP) permet de se connecter a votre PC depuis un autre ordinateur. Si vous ne l'utilisez pas, desactivez-le :

```
HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `fDenyTSConnections` | REG_DWORD | `1` | Bloque les connexions Bureau a distance |

!!! example "Commande rapide"
    ```powershell
    reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 1 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

!!! warning "Ne desactivez pas si..."
    ...vous utilisez le Bureau a distance pour acceder a votre PC depuis un autre endroit, ou si votre service informatique l'utilise pour la maintenance.

---

### Verifier que l'UAC est actif

Le Controle de Compte Utilisateur (UAC) est la fenetre qui vous demande "Voulez-vous autoriser cette application a apporter des modifications ?" C'est une **barriere de securite essentielle**.

```powershell
# Check UAC status
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" /v EnableLUA
```

```title="Resultat attendu"
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
    EnableLUA    REG_DWORD    0x1
```

| Valeur | Signification |
|:------:|---------------|
| `1` | UAC actif (recommande) |
| `0` | UAC desactive (dangereux) |

!!! danger "Ne desactivez JAMAIS l'UAC"
    Certains tutoriels recommandent de desactiver l'UAC "pour ne plus etre derange". C'est une **tres mauvaise idee**. Sans UAC, n'importe quel programme peut modifier votre systeme sans vous demander la permission.

    C'est comme retirer la serrure de votre porte d'entree parce que vous trouvez que c'est penible de tourner la cle.

---

### Desactiver l'execution de scripts PowerShell non signes (optionnel)

Par defaut, la politique d'execution PowerShell empeche l'execution de scripts telecharges. Verifiez qu'elle n'a pas ete assouplie :

```powershell
# Check PowerShell execution policy
reg query "HKLM\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell" /v ExecutionPolicy
```

```title="Resultat attendu"
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell
    ExecutionPolicy    REG_SZ    RemoteSigned
```

| Politique | Securite |
|-----------|----------|
| `Restricted` | Aucun script ne s'execute (le plus sur) |
| `RemoteSigned` | Seuls les scripts signes ou locaux s'executent (bon compromis) |
| `Unrestricted` | Tous les scripts s'executent (risque) |
| `Bypass` | Aucune restriction (dangereux) |

!!! quote "En resume"
    - Les reglages de base a appliquer : desactiver l'AutoRun (`NoDriveTypeAutoRun`), desactiver le Bureau a distance si inutilise (`fDenyTSConnections`), verifier que l'UAC est actif (`EnableLUA` = `1`).
    - Ne desactivez **jamais** l'UAC : c'est une barriere de securite essentielle contre les programmes malveillants.

---

## Detecter les changements non autorises

### Regshot : la methode avant/apres

**Regshot** est un outil gratuit qui prend un **instantane** du registre, puis un deuxieme apres un evenement. Il compare les deux et affiche les differences.

!!! info "L'analogie"
    C'est comme prendre une photo de votre chambre avant de partir, puis une deuxieme a votre retour. Vous comparez les deux photos pour voir si quelqu'un a touche a vos affaires.

**Utilisation** :

1. Telechargez Regshot : `https://sourceforge.net/projects/regshot/`
2. Lancez-le en administrateur

**Etape 1** -- Prenez le premier instantane :

- Cliquez sur **1st shot** > **Shot**
- Attendez la fin du scan (cela peut prendre 1-2 minutes)

**Etape 2** -- Faites l'action a analyser :

- Par exemple : installez un programme, ou lancez un fichier suspect dans une VM

**Etape 3** -- Prenez le deuxieme instantane :

- Cliquez sur **2nd shot** > **Shot**

**Etape 4** -- Comparez :

- Cliquez sur **Compare**
- Regshot genere un rapport HTML ou texte

Exemple de rapport :

```
Keys added: 12
Values added: 47
Values modified: 8
Keys deleted: 0

----------------------------------
Keys added:
----------------------------------
HKLM\SOFTWARE\MonNouveauProgramme
HKLM\SOFTWARE\MonNouveauProgramme\Settings
...

----------------------------------
Values added:
----------------------------------
HKCU\Software\Microsoft\Windows\CurrentVersion\Run\MonProgramme: "C:\Program Files\..."
...
```

!!! tip "Quand utiliser Regshot"
    - Avant/apres l'installation d'un programme
    - Avant/apres l'execution d'un fichier `.reg` telecharge
    - Pour comprendre ce qu'un logiciel modifie dans le registre

!!! quote "En resume"
    - **Regshot** prend deux instantanes du registre (avant et apres) et compare les differences : cles ajoutees, valeurs modifiees, cles supprimees.
    - C'est l'outil ideal pour verifier ce qu'un programme ou un fichier `.reg` modifie reellement dans le registre.

---

## Les permissions du registre

### Pourquoi certaines cles sont verrouillees

Vous avez peut-etre deja rencontre ce message :

> "Impossible d'enregistrer la modification. Erreur lors de l'ecriture..."

C'est le systeme de **permissions** du registre. Comme les fichiers sur votre disque, chaque cle du registre a des permissions qui controlent **qui** peut faire **quoi**.

!!! info "L'analogie"
    Dans une entreprise, tout le monde peut lire les affiches dans le hall d'entree (permissions de lecture). Mais seul le directeur peut modifier le panneau d'affichage (permissions d'ecriture). Et seul le proprietaire du batiment peut decider qui a le droit de faire quoi (permissions de controle total).

### Les niveaux de permission

| Permission | Signification |
|-----------|---------------|
| Lecture | Voir les valeurs (tout le monde l'a pour la plupart des cles) |
| Ecriture | Modifier ou creer des valeurs |
| Controle total | Tout faire, y compris modifier les permissions |
| Acces special | Permissions personnalisees et granulaires |

### Qui a quelles permissions ?

| Cle | Administrateurs | Utilisateur standard |
|-----|:---------------:|:--------------------:|
| `HKCU\...` | Controle total | Controle total (c'est **sa** ruche) |
| `HKLM\SOFTWARE` | Controle total | Lecture seule |
| `HKLM\SYSTEM` | Controle total | Lecture seule |
| `HKLM\SAM` | Controle total | Aucun acces |
| `HKLM\SECURITY` | Aucun acces* | Aucun acces |

*Meme les administrateurs ne peuvent pas lire `HKLM\SECURITY` directement. Seul le compte `SYSTEM` y a acces.

### Comment voir les permissions d'une cle

1. Dans Regedit, clic droit sur une cle
2. **Autorisations...**
3. La fenetre affiche les groupes et utilisateurs avec leurs niveaux d'acces

!!! warning "Ne modifiez pas les permissions"
    En tant que debutant, ne changez **jamais** les permissions d'une cle du registre. C'est une operation avancee qui peut rendre votre systeme instable si elle est mal faite. Cette section est pour **comprendre** le concept, pas pour le modifier.

!!! quote "En resume"
    - Chaque cle du registre a des permissions (lecture, ecriture, controle total) qui controlent qui peut faire quoi.
    - Les utilisateurs standard ont le controle total sur `HKCU` (leur espace) mais seulement la lecture sur `HKLM`. En tant que debutant, ne modifiez **jamais** les permissions.

---

## Cles "protegees" par Windows

Certaines cles sont protegees par des mecanismes supplementaires :

| Mecanisme | Ce qu'il protege |
|-----------|-----------------|
| **Tamper Protection** | Cles de Windows Defender |
| **Windows Resource Protection (WRP)** | Cles systeme critiques |
| **Trusted Installer** | Cles installees par Windows Update |

!!! info "TrustedInstaller"
    Le proprietaire de nombreuses cles systeme n'est pas `Administrateurs` mais `TrustedInstaller`, un compte special utilise par Windows Update. Meme un administrateur ne peut pas modifier ces cles sans d'abord **prendre possession** de la cle, ce qui est deconseille.

!!! quote "En resume"
    - Certaines cles sont protegees par des mecanismes supplementaires : **Tamper Protection** (Defender), **Windows Resource Protection** (cles systeme) et **TrustedInstaller** (Windows Update).
    - Meme un administrateur ne peut pas modifier les cles appartenant a TrustedInstaller sans en prendre possession, ce qui est deconseille.

---

## Tableau recapitulatif des menaces

| Menace | Technique registre | Comment detecter |
|--------|-------------------|-----------------|
| Demarrage automatique | Cles `Run` / `RunOnce` | Gestionnaire des taches, Autoruns |
| Desactivation antivirus | `DisableAntiSpyware` | `Get-MpComputerStatus` |
| Composant cache | CLSID dans `HKCR` | Autoruns, analyse antivirus |
| Redirection de fichiers | Associations dans `HKCR` | Ouverture d'un fichier test |
| Service malveillant | `HKLM\SYSTEM\Services` | Autoruns, `sc query` |
| Tache planifiee | Taches dans le registre | Planificateur de taches, Autoruns |

!!! quote "En resume"
    - Ce tableau recapitule les 6 techniques de menace (demarrage automatique, desactivation antivirus, composant cache, redirection de fichiers, service malveillant, tache planifiee) avec les outils de detection correspondants.
    - Les outils les plus utiles : Gestionnaire des taches, **Autoruns**, `Get-MpComputerStatus` et l'analyse antivirus complete.

---

!!! quote "En resume"
    - Les malwares utilisent le registre pour **persister** : cles Run, CLSID, services, associations de fichiers
    - Verifiez regulierement vos cles de demarrage avec le **Gestionnaire des taches** ou **Autoruns**
    - Confirmez que Windows Defender et **Tamper Protection** sont actifs
    - Appliquez les reglages de base : desactiver AutoRun, verifier UAC, controler le Bureau a distance
    - Utilisez **Regshot** pour comparer avant/apres une installation ou modification
    - Ne modifiez **jamais** les permissions du registre sauf si vous savez exactement ce que vous faites
    - En cas de doute sur une entree suspecte, lancez une analyse antivirus complete

---

!!! tip "Pour aller plus loin"
    Les mecanismes de securite avances (ACL, SACL, UAC Virtualization) sont detailles dans [La Bible — Securite et permissions](../bible-registre-windows/06-securite.md), et les technologies modernes (Credential Guard, Defender, BitLocker) dans [La Bible — Securite moderne](../bible-registre-windows/16-securite-moderne.md).
