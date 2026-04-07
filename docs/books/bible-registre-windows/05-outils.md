---
tags:
  - registre
  - outils
  - regedit
  - reg
---

# Outils et methodes d'acces

## Ce que vous allez apprendre

- Comment utiliser Regedit pour naviguer et modifier le registre
- Les commandes `reg.exe` essentielles (lecture, ecriture, sauvegarde)
- Comment manipuler le registre avec PowerShell
- Les fonctions de l'API Win32 et .NET
- Le format des fichiers `.reg` pour l'import/export

---

## Regedit : l'outil graphique integre

Le plus rapide pour explorer le registre visuellement.
Present sur **toutes les versions de Windows** depuis Windows 95.

**Lancement** : ++win+r++ puis `regedit` puis ++enter++

!!! tip "Barre d'adresse"
    Depuis Windows 10 version 1703, Regedit a une **barre d'adresse**. Collez un chemin de cle pour y acceder instantanement. Essayez avec :
    ```
    HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion
    ```

### Ce que Regedit sait faire

| Fonction | Comment |
|----------|---------|
| Naviguer dans l'arborescence | Panneau gauche, comme l'Explorateur de fichiers |
| Creer / modifier / supprimer | Clic droit sur une cle ou une valeur |
| Rechercher | ++ctrl+f++ -- cherche dans les cles, valeurs et donnees |
| Exporter en `.reg` | Fichier > Exporter (ou clic droit > Exporter) |
| Importer un `.reg` | Fichier > Importer (ou double-clic sur un fichier `.reg`) |
| Charger une ruche hors ligne | Fichier > Charger la ruche (selectionner un `NTUSER.DAT`) |
| Gerer les permissions | Clic droit sur une cle > Autorisations |
| Se connecter a un registre distant | Fichier > Se connecter au Registre reseau |

---

### Outils tiers notables

| Outil | Utilite |
|-------|---------|
| **Registry Explorer** (Eric Zimmerman) | Analyse forensique de ruches hors ligne, supporte les fichiers de transaction |
| **RegScanner** (NirSoft) | Recherche avancee avec filtres par type, taille et date |
| **Regshot** | Compare deux instantanes du registre pour detecter ce qui a change |
| **Process Monitor** (Sysinternals) | Surveillance en temps reel des acces au registre par les processus |

!!! example "Cas d'usage de Regshot"
    Vous installez un logiciel et vous voulez savoir **exactement** ce qu'il a modifie dans le registre ? Prenez un instantane avant l'installation, un apres, et Regshot vous montre la difference.

!!! info "En resume"
    Regedit est l'outil graphique de base pour explorer et modifier le registre. Pour des besoins avances (forensique, surveillance, comparaison), des outils tiers comme Registry Explorer, RegScanner, Regshot et Process Monitor completent l'arsenal.

---

## reg.exe : la ligne de commande

Outil en ligne de commande **integre a Windows**.
Ideal pour les scripts et l'automatisation.

### Lecture

```batch
rem Lire une valeur specifique
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v ProductName
```

```title="Resultat attendu"
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion
    ProductName    REG_SZ    Windows 11 Pro
```

```batch
rem Lister toutes les valeurs d'une cle
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
```

```title="Resultat attendu"
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion
    SystemRoot          REG_SZ    C:\Windows
    BuildBranch         REG_SZ    ni_release
    CurrentBuild        REG_SZ    22631
    ProductName         REG_SZ    Windows 11 Pro
    ...
```

```batch
rem Recherche recursive d'un mot dans les cles et valeurs
reg query "HKLM\SOFTWARE" /s /f "MonApp"
```

```title="Resultat attendu"
HKEY_LOCAL_MACHINE\SOFTWARE\MonApp
    (Default)    REG_SZ    Mon Application

End of search: 1 match(es) found.
```

---

### Ecriture

```batch
rem Creer ou modifier une valeur REG_SZ
reg add "HKCU\Software\MonApp" /v Param /t REG_SZ /d "valeur" /f
```

```title="Resultat attendu"
The operation completed successfully.
```

```batch
rem Creer une valeur REG_DWORD
reg add "HKCU\Software\MonApp" /v Activer /t REG_DWORD /d 1 /f
```

```title="Resultat attendu"
The operation completed successfully.
```

```batch
rem Supprimer une valeur
reg delete "HKCU\Software\MonApp" /v Param /f
```

```title="Resultat attendu"
The operation completed successfully.
```

```batch
rem Supprimer une cle et tout son contenu
reg delete "HKCU\Software\MonApp" /f
```

```title="Resultat attendu"
The operation completed successfully.
```

!!! warning "Le flag /f"
    Le flag `/f` (**force**) supprime la demande de confirmation. Sans ce flag, `reg.exe` vous demande "Etes-vous sur ?". Dans un script, utilisez toujours `/f`. En mode interactif, retirez-le pour eviter les erreurs.

---

### Sauvegarde et restauration

=== "Export / Import (.reg)"

    ```batch
    rem Exporter une branche vers un fichier .reg (texte lisible)
    reg export "HKCU\Software\MonApp" sauvegarde.reg
    ```

    ```title="Resultat attendu"
    The operation completed successfully.
    ```

    ```batch
    rem Importer un fichier .reg
    reg import sauvegarde.reg
    ```

    ```title="Resultat attendu"
    The operation completed successfully.
    ```

=== "Save / Restore (.hiv)"

    ```batch
    rem Sauvegarder une ruche entiere (format binaire brut)
    reg save "HKLM\SYSTEM" system_backup.hiv
    ```

    ```title="Resultat attendu"
    The operation completed successfully.
    ```

    ```batch
    rem Restaurer une ruche (ATTENTION : ecrase tout !)
    reg restore "HKLM\SYSTEM" system_backup.hiv
    ```

    ```title="Resultat attendu"
    The operation completed successfully.
    ```

!!! danger "reg save vs reg export"
    `reg export` cree un fichier `.reg` **texte** lisible et modifiable. `reg save` cree un fichier `.hiv` **binaire** brut (copie exacte de la ruche). Utilisez `export` pour des sauvegardes partielles et `save` pour des sauvegardes completes de ruche.

!!! info "En resume"
    `reg.exe` couvre tous les besoins : `query` pour lire, `add` pour ecrire, `delete` pour supprimer, `export/import` pour le format texte `.reg`, et `save/restore` pour le format binaire `.hiv`. Le flag `/f` supprime les confirmations.

---

## PowerShell : le registre comme un disque

PowerShell expose les ruches du registre comme des **lecteurs** (`HKLM:`, `HKCU:`).
Vous naviguez dedans exactement comme dans un systeme de fichiers.

### Navigation et lecture

```powershell
# Navigate like a file system
Set-Location HKLM:\SOFTWARE\Microsoft

# List subkeys (like 'dir')
Get-ChildItem HKLM:\SOFTWARE\Microsoft | Select-Object -First 5
```

```title="Resultat attendu"
    Hive: HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft

Name                           Property
----                           --------
.NETFramework                  ...
Active Setup                   ...
ASP.NET                        ...
Assistance                     ...
AuthHost                       ...
```

```powershell
# Read all values of a key
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" |
    Select-Object ProductName, CurrentBuild
```

```title="Resultat attendu"
ProductName    CurrentBuild
-----------    ------------
Windows 11 Pro 22631
```

```powershell
# Read a single value
Get-ItemPropertyValue "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" -Name ProductName
```

```title="Resultat attendu"
Windows 11 Pro
```

---

### Ecriture et suppression

```powershell
# Create a key
New-Item -Path "HKCU:\Software\MonApp" -Force
```

```title="Resultat attendu"
    Hive: HKEY_CURRENT_USER\Software

Name                           Property
----                           --------
MonApp
```

```powershell
# Create or update a value
Set-ItemProperty -Path "HKCU:\Software\MonApp" -Name "Activer" -Value 1 -Type DWord

# Delete a value
Remove-ItemProperty -Path "HKCU:\Software\MonApp" -Name "Activer"

# Delete a key and everything inside
Remove-Item -Path "HKCU:\Software\MonApp" -Recurse
```

```title="Resultat attendu"
La valeur est creee, puis supprimee. La cle est ensuite supprimee sans erreur.
```

---

### Acceder aux autres ruches

Par defaut, seuls `HKLM:` et `HKCU:` sont disponibles.
Pour acceder a `HKU` ou d'autres ruches :

```powershell
# Create a PSDrive for HKEY_USERS
New-PSDrive -Name HKU -PSProvider Registry -Root HKEY_USERS

# Now you can browse it
Get-ChildItem HKU:\ | Select-Object Name
```

```title="Resultat attendu"
Name
----
HKEY_USERS\.DEFAULT
HKEY_USERS\S-1-5-18
HKEY_USERS\S-1-5-19
HKEY_USERS\S-1-5-20
HKEY_USERS\S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX-1001
```

!!! info "En resume"
    PowerShell traite le registre comme un systeme de fichiers. `Get-ChildItem` liste les sous-cles, `Get-ItemProperty` lit les valeurs, `Set-ItemProperty` ecrit, `Remove-Item` supprime. Creez un `PSDrive` pour acceder aux ruches autres que HKLM et HKCU.

---

## Acces distant

### OpenSSH

Windows inclut `ssh.exe` comme client natif depuis Windows 10 1809. Le serveur `sshd` existe aussi comme fonctionnalite optionnelle et devient pertinent pour administrer des machines Windows depuis des postes Linux, macOS ou PowerShell 7.

```powershell title="Lire une cle registre via ssh.exe"
ssh admin@server01 "reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion /v ProductName"
```

Pour des operations registre complexes, privilegiez PowerShell Remoting over SSH :

```powershell title="Provider registre via SSH remoting"
Enter-PSSession -HostName server01 -UserName admin
Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion
```

Deux outils historiques restent utiles :

- `reg compare` compare deux sous-arbres de registre, pratique pour detecter une derive entre image de reference et poste cible ;
- `regini.exe` modifie le registre et les ACL en batch, mais sa syntaxe est cryptique : preferez PowerShell sauf contrainte legacy.

Pour la configuration complete d'OpenSSH et le choix WinRM vs SSH, voir [PowerShell Remoting](../registre-pour-les-admins/01-powershell-remoting.md).

!!! quote "En resume"
    - `ssh.exe` permet d'executer rapidement `reg query` a distance sans WinRM.
    - Pour les operations serieuses, utilisez PowerShell 7 remoting over SSH.
    - `reg compare` est utile pour comparer deux branches registre.
    - `regini.exe` existe encore pour les cas legacy, mais PowerShell reste preferable.

---

## API de programmation

### API Win32 (C/C++)

Les fonctions principales pour manipuler le registre depuis du code natif :

| Fonction | Role |
|----------|------|
| `RegOpenKeyEx` | Ouvrir une cle existante |
| `RegCreateKeyEx` | Creer une cle (ou ouvrir si elle existe) |
| `RegSetValueEx` | Ecrire une valeur |
| `RegQueryValueEx` | Lire une valeur |
| `RegDeleteKey` | Supprimer une cle |
| `RegDeleteValue` | Supprimer une valeur |
| `RegEnumKeyEx` | Enumerer les sous-cles |
| `RegEnumValue` | Enumerer les valeurs |
| `RegNotifyChangeKeyValue` | Surveiller les modifications d'une cle |
| `RegCloseKey` | Fermer un handle de cle |

!!! warning "Toujours fermer les handles"
    Chaque appel a `RegOpenKeyEx` ou `RegCreateKeyEx` ouvre un **handle**. Si vous oubliez de le fermer avec `RegCloseKey`, vous creez une fuite de ressources. C'est comme ouvrir un robinet sans jamais le refermer.

---

### API .NET (C#)

L'espace de noms `Microsoft.Win32` fournit les classes `Registry` et `RegistryKey` :

```csharp
using Microsoft.Win32;

// Read a value
using RegistryKey key = Registry.LocalMachine.OpenSubKey(
    @"SOFTWARE\Microsoft\Windows NT\CurrentVersion");
string productName = key?.GetValue("ProductName") as string;
// productName = "Windows 11 Pro"

// Write a value
using RegistryKey appKey = Registry.CurrentUser.CreateSubKey(@"Software\MonApp");
appKey.SetValue("Activer", 1, RegistryValueKind.DWord);
```

!!! tip "Le mot-cle using"
    Le `using` dans `using RegistryKey` garantit que le handle est **automatiquement ferme** a la fin du bloc. C'est l'equivalent .NET de `RegCloseKey`.

!!! info "En resume"
    - L'API Win32 (C/C++) utilise des fonctions comme `RegOpenKeyEx`, `RegQueryValueEx` et `RegSetValueEx`
    - L'API .NET (C#) fournit les classes `Registry` et `RegistryKey` dans `Microsoft.Win32`
    - Dans les deux cas, chaque handle ouvert doit etre ferme (`RegCloseKey` ou `using`) pour eviter les fuites de ressources

---

## Format de fichier .reg

Les fichiers `.reg` sont des fichiers **texte** permettant d'importer et d'exporter des donnees du registre.
Vous pouvez les creer manuellement ou les generer avec `reg export`.

### Exemple complet commente

```ini
Windows Registry Editor Version 5.00

; This is a comment — configuration for MonApp
[HKEY_CURRENT_USER\Software\MonApp]
"ParamTexte"="valeur texte"
"Activer"=dword:00000001
"Compteur64"=qword:00000000000003e8
"DonneesBinaires"=hex:48,65,6c,6c,6f
"ListeChaines"=hex(7):50,00,72,00,65,00,6d,00,69,00,65,00,72,00,00,00,\
  44,00,65,00,75,00,78,00,69,00,e8,00,6d,00,65,00,00,00,00,00

; Delete a specific value (dash after equals)
[HKEY_CURRENT_USER\Software\MonApp]
"AncienParametre"=-

; Delete an entire key (dash before the path)
[-HKEY_CURRENT_USER\Software\AncienneApp]
```

### Table de correspondance des prefixes

| Prefixe dans le fichier .reg | Type du registre |
|------------------------------|------------------|
| `"texte"` | `REG_SZ` |
| `dword:00000001` | `REG_DWORD` |
| `qword:00000000000003e8` | `REG_QWORD` |
| `hex:48,65,6c,6c,6f` | `REG_BINARY` |
| `hex(2):...` | `REG_EXPAND_SZ` |
| `hex(7):...` | `REG_MULTI_SZ` |

!!! tip "Ligne de continuation"
    Les lignes longues (comme `hex(7):`) peuvent etre coupees avec un `\` en fin de ligne. Le contenu continue sur la ligne suivante, indente de 2 espaces.

!!! danger "Double-clic = import"
    Double-cliquer sur un fichier `.reg` l'importe **immediatement** dans le registre (apres confirmation). Ne double-cliquez jamais sur un fichier `.reg` dont vous ne connaissez pas le contenu. Ouvrez-le d'abord dans un editeur de texte.

!!! info "En resume"
    Vous avez 4 facons d'acceder au registre : **Regedit** (graphique), **reg.exe** (ligne de commande), **PowerShell** (scriptable) et les **API** (C/C++, .NET). Les fichiers `.reg` permettent d'echanger des configurations sous forme de texte lisible. Chaque outil a ses forces : Regedit pour l'exploration, reg.exe pour les scripts batch, PowerShell pour l'automatisation avancee, et les API pour les applications.
