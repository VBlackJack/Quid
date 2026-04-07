---
tags:
  - registre
  - débutant
  - PowerShell
---

# PowerShell et le registre : les bases

!!! abstract "Ce que vous allez apprendre"
    - Pourquoi PowerShell est un outil precieux pour manipuler le registre
    - Comment ouvrir PowerShell et taper votre premiere commande
    - Comment naviguer dans le registre comme dans des dossiers
    - Comment lire, modifier, creer et supprimer des valeurs
    - Les 5 commandes essentielles a retenir

!!! warning "Rappel essentiel"
    Avant **chaque** modification du registre, faites une sauvegarde. Meme avec PowerShell, il n'y a pas de Ctrl+Z. Le chapitre 5 vous explique toutes les methodes de sauvegarde.

---

## Pourquoi PowerShell ?

Imaginez que vous etes devant votre television. Pour changer de chaine, vous avez deux options :

- **Option 1** : vous lever du canape, marcher jusqu'a la tele, appuyer sur le bouton, revenir vous asseoir.
- **Option 2** : prendre la telecommande et appuyer sur un bouton.

Regedit, c'est l'option 1. Ca marche tres bien, mais il faut tout faire a la main : cliquer, naviguer, chercher la bonne cle, double-cliquer, modifier, valider...

PowerShell, c'est la telecommande. Vous tapez une commande, et c'est fait.

### Ce que PowerShell peut faire que Regedit ne peut pas

| Besoin | Regedit | PowerShell |
|--------|:-------:|:----------:|
| Modifier une seule valeur | Oui | Oui |
| Modifier 50 valeurs d'un coup | Non (une par une) | Oui (une seule commande) |
| Lire une valeur sans ouvrir de fenetre | Non | Oui |
| Automatiser des changements recurrents | Non | Oui (scripts) |
| Appliquer les memes reglages sur 10 PC | Non (fichiers .reg) | Oui (beaucoup plus flexible) |

!!! tip "N'ayez pas peur de la fenetre bleue"
    PowerShell ressemble a un ecran de hacker dans les films, mais en realite c'est juste un endroit ou vous tapez des phrases que l'ordinateur comprend. Pas besoin d'etre informaticien. Si vous savez copier-coller, vous savez utiliser PowerShell.

!!! quote "En resume"
    - PowerShell est une telecommande pour le registre : plus rapide que Regedit pour les taches repetitives et l'automatisation.
    - Il permet de modifier 50 valeurs d'un coup, d'automatiser des changements et de lancer des scripts sur plusieurs PC.

---

## Ouvrir PowerShell

Il y a plusieurs facons d'ouvrir PowerShell. Choisissez celle qui vous convient.

### Methode 1 : le menu Demarrer

!!! example "Pas a pas"
    1. Cliquer sur le bouton **Demarrer** (l'icone Windows en bas a gauche)
    2. Taper `powershell`
    3. Cliquer sur **Windows PowerShell** dans les resultats

### Methode 2 : le raccourci Win+X

!!! example "Pas a pas"
    1. Appuyer sur les touches ++windows+x++ en meme temps
    2. Choisir **Terminal** ou **Windows PowerShell** dans le menu qui apparait

### Methode 3 : clic droit sur Demarrer

!!! example "Pas a pas"
    1. Faire un **clic droit** sur le bouton Demarrer
    2. Choisir **Terminal** ou **Windows PowerShell**

---

### PowerShell normal ou Administrateur ?

Vous verrez parfois deux options :

- **Windows PowerShell** : mode normal, suffisant pour lire le registre et modifier les cles de votre profil utilisateur (`HKCU:`).
- **Windows PowerShell (Admin)** : mode administrateur, necessaire pour modifier les cles systeme (`HKLM:`).

!!! note "Comment savoir lequel utiliser ?"
    - Pour **lire** le registre : le mode normal suffit presque toujours.
    - Pour **modifier** des cles dans `HKLM:` (la machine) : il faut le mode Administrateur.
    - Pour **modifier** des cles dans `HKCU:` (votre profil) : le mode normal suffit.

    Si une commande vous repond "Acces refuse", c'est probablement que vous devez relancer PowerShell en mode Administrateur.

---

### Ce que vous voyez a l'ecran

Une fois PowerShell ouvert, vous voyez une fenetre avec un fond bleu fonce (ou noir selon votre version de Windows). Le texte ressemble a ceci :

```title="Ce que vous voyez"
PS C:\Users\VotreNom>
```

Le `PS` signifie "PowerShell". `C:\Users\VotreNom` est votre emplacement actuel, comme quand vous etes dans un dossier de l'Explorateur de fichiers.

Le curseur clignote apres le `>`. C'est la que vous tapez vos commandes.

### Votre premiere commande

Tapez ceci et appuyez sur ++enter++ :

```powershell
# Display the current user name
whoami
```

```title="Resultat attendu"
votrepc\votrenom
```

Voila, vous venez de taper votre premiere commande PowerShell. Elle affiche simplement votre nom d'utilisateur. Rien de dangereux, rien de complique.

!!! tip "Copier-coller dans PowerShell"
    Pour coller une commande dans PowerShell, faites un **clic droit** dans la fenetre. Pas besoin de Ctrl+V (meme si ca marche aussi dans les versions recentes de Windows).

!!! quote "En resume"
    - Trois facons d'ouvrir PowerShell : menu Demarrer, ++win+x++, ou clic droit sur Demarrer.
    - Le mode **normal** suffit pour lire le registre et modifier HKCU ; le mode **Administrateur** est necessaire pour modifier HKLM.

---

## Naviguer dans le registre avec PowerShell

### Le registre comme un disque dur

Vous connaissez les lettres de lecteur : `C:` pour votre disque principal, `D:` pour un deuxieme disque, etc.

PowerShell utilise le meme principe pour le registre :

| Lettre de lecteur | Ce que c'est |
|-------------------|--------------|
| `C:` | Votre disque dur |
| `D:` | Un autre disque |
| `HKLM:` | La ruche **HKEY_LOCAL_MACHINE** (reglages de la machine) |
| `HKCU:` | La ruche **HKEY_CURRENT_USER** (reglages de votre profil) |

Vous pouvez "entrer" dans le registre exactement comme vous entrez dans un dossier.

### Se deplacer dans le registre

Tapez cette commande pour entrer dans la ruche de votre profil utilisateur :

```powershell
# Navigate to the Software key in current user hive
cd HKCU:\Software
```

```title="Resultat attendu"
PS HKCU:\Software>
```

Vous voyez ? Le chemin a change. Vous etes maintenant "dans" la cle `Software` du registre, exactement comme si vous aviez ouvert un dossier.

### Lister le contenu

Pour voir ce qu'il y a dans la cle ou vous etes, tapez :

```powershell
# List all subkeys in the current location
dir
```

```title="Resultat attendu"
    Hive: HKEY_CURRENT_USER\Software

Name                           Property
----                           --------
7-Zip
Adobe
Google
Microsoft
Mozilla
```

!!! note "dir ou Get-ChildItem ?"
    `dir` est un raccourci. La commande complete est `Get-ChildItem`. Les deux font exactement la meme chose. Utilisez celle que vous preferez.

---

### Exercice : lire et modifier le registre en PowerShell

!!! example "Essayez vous-meme"
    Naviguez jusqu'a la cle qui contient des informations sur votre version de Windows :

    ```powershell
    # Navigate to the Windows version information key
    cd "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
    ```

    ```title="Resultat attendu"
    PS HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion>
    ```

    Puis listez le contenu :

    ```powershell
    # List everything in this key
    dir
    ```

    ```title="Resultat attendu"
        Hive: HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion

    Name                           Property
    ----                           --------
    AppCompatFlags
    Drivers
    Fonts
    Image File Execution Options
    ProfileList
    Winlogon
    ...
    ```

    Vous verrez les sous-cles liees a votre installation de Windows. Ne modifiez rien ici, on ne fait que regarder.

Pour revenir a votre dossier habituel, tapez :

```powershell
# Go back to the C: drive
cd C:\
```

```title="Resultat attendu"
PS C:\>
```

!!! quote "En resume"
    - Le registre se parcourt comme un disque dur : `HKCU:` et `HKLM:` sont des lecteurs, `cd` pour naviguer, `dir` pour lister.
    - Vous pouvez entrer dans une cle avec `cd HKCU:\Software` et voir son contenu avec `dir`.

---

## Lire une valeur du registre

La commande principale pour lire une valeur est `Get-ItemProperty`. C'est comme double-cliquer sur une valeur dans Regedit pour voir son contenu, mais sans avoir a naviguer manuellement.

### Exemple 1 : lire le nom de votre PC

```powershell
# Read the active computer name from the registry
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\ComputerName\ActiveComputerName"
```

```title="Resultat attendu"
ComputerName    : MON-PC
PSPath          : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\ComputerName\ActiveComputerName
PSParentPath    : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\ComputerName
PSChildName     : ActiveComputerName
PSProvider      : Microsoft.PowerShell.Core\Registry
```

!!! note "C'est quoi toutes ces lignes ?"
    PowerShell affiche des informations supplementaires (les lignes `PS...`). La ligne qui vous interesse est `ComputerName : MON-PC`. Les autres lignes sont des details techniques que vous pouvez ignorer.

---

### Exemple 2 : lire la version de Windows

Pour lire une valeur specifique, ajoutez `-Name` suivi du nom de la valeur :

```powershell
# Read only the ProductName value
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" -Name ProductName
```

```title="Resultat attendu"
ProductName : Windows 11 Pro
```

**Decortiquons la commande** :

| Partie | Role |
|--------|------|
| `Get-ItemProperty` | La commande : "lis une propriete" |
| `"HKLM:\SOFTWARE\..."` | Le chemin vers la cle (entre guillemets car il contient des espaces) |
| `-Name ProductName` | Le nom precis de la valeur a lire |

!!! tip "Avec ou sans -Name ?"
    - **Sans** `-Name` : PowerShell affiche **toutes** les valeurs de la cle.
    - **Avec** `-Name` : PowerShell affiche **uniquement** la valeur demandee. Pratique quand une cle contient beaucoup de valeurs.

---

### Exemple 3 : lister les programmes au demarrage

```powershell
# List all programs that run at startup for the current user
Get-ItemProperty "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
```

```title="Resultat attendu"
SecurityHealth  : C:\Windows\system32\SecurityHealthSystray.exe
OneDrive        : "C:\Program Files\Microsoft OneDrive\OneDrive.exe" /background
Discord         : "C:\Users\VotreNom\AppData\Local\Discord\Update.exe" --processStart Discord.exe
```

Chaque ligne montre un programme qui se lance automatiquement quand vous ouvrez votre session Windows. Vous reconnaissez peut-etre certains d'entre eux.

!!! quote "En resume"
    - `Get-ItemProperty` lit les valeurs d'une cle ; ajoutez `-Name` pour cibler une valeur specifique.
    - Sans `-Name`, la commande affiche **toutes** les valeurs de la cle d'un coup.

---

## Modifier une valeur

!!! danger "Attention !"
    Modifier le registre peut affecter le fonctionnement de Windows. Suivez **toujours** le protocole en 3 etapes : sauvegarder, modifier, verifier.

### Le protocole en 3 etapes

#### Etape 1 : sauvegarder la cle

Avant de modifier quoi que ce soit, exportez la cle. Vous connaissez deja `reg export` ou l'export via Regedit (chapitre 5). Voici la version en ligne de commande :

```powershell
# Export the key to a backup file on the Desktop
reg export "HKCU\Control Panel\Mouse" "$env:USERPROFILE\Desktop\sauvegarde_mouse.reg"
```

```title="Resultat attendu"
L'operation a reussi.
```

#### Etape 2 : modifier la valeur

Prenons un exemple concret. Dans le chapitre 4, vous avez peut-etre modifie le delai de survol de la barre des taches (`MouseHoverTime`). Faisons la meme chose avec PowerShell :

```powershell
# Change the mouse hover time to 200 milliseconds
Set-ItemProperty -Path "HKCU:\Control Panel\Mouse" -Name "MouseHoverTime" -Value "200"
```

```title="Resultat attendu"
Aucune sortie si la commande reussit. C'est normal !
```

Aucun message ne s'affiche si tout va bien. Pas de message = pas d'erreur.

#### Etape 3 : verifier le changement

```powershell
# Verify the value was changed correctly
Get-ItemProperty "HKCU:\Control Panel\Mouse" -Name "MouseHoverTime"
```

```title="Resultat attendu"
MouseHoverTime : 200
```

La valeur affichee correspond bien a ce que vous avez tape. La modification est faite.

!!! tip "Si quelque chose ne va pas"
    Double-cliquez sur le fichier `.reg` que vous avez exporte a l'etape 1. Windows vous demandera de confirmer la restauration, et tout reviendra comme avant.

!!! quote "En resume"
    - Le protocole en 3 etapes : **sauvegarder** (`reg export`), **modifier** (`Set-ItemProperty`), **verifier** (`Get-ItemProperty`).
    - Aucun message apres `Set-ItemProperty` = pas d'erreur. Verifiez toujours avec `Get-ItemProperty` ensuite.

---

## Creer et supprimer

### Creer une nouvelle cle

Une cle, c'est comme un dossier. Pour en creer une :

```powershell
# Create a new key called "MonTest" under HKCU:\Software
New-Item -Path "HKCU:\Software\MonTest"
```

```title="Resultat attendu"
    Hive: HKEY_CURRENT_USER\Software

Name                           Property
----                           --------
MonTest
```

La cle `MonTest` existe maintenant dans le registre. Vous pouvez la voir dans Regedit si vous naviguez vers `HKCU\Software`.

---

### Creer une nouvelle valeur

Une valeur, c'est comme un fichier dans un dossier. Pour en creer une dans la cle que vous venez de faire :

```powershell
# Create a new string value inside the MonTest key
New-ItemProperty -Path "HKCU:\Software\MonTest" -Name "MaValeur" -Value "Bonjour" -PropertyType String
```

```title="Resultat attendu"
MaValeur     : Bonjour
PSPath       : Microsoft.PowerShell.Core\Registry::HKEY_CURRENT_USER\Software\MonTest
PSParentPath : Microsoft.PowerShell.Core\Registry::HKEY_CURRENT_USER\Software
PSChildName  : MonTest
PSProvider   : Microsoft.PowerShell.Core\Registry
```

!!! note "Les types de valeur"
    Le parametre `-PropertyType` accepte les memes types que Regedit :

    | PropertyType | Equivalent Regedit | Utilisation |
    |-------------|-------------------|-------------|
    | `String` | REG_SZ | Texte simple |
    | `DWord` | REG_DWORD | Nombre entier |
    | `ExpandString` | REG_EXPAND_SZ | Texte avec variables (`%USERPROFILE%`) |
    | `MultiString` | REG_MULTI_SZ | Liste de textes |
    | `Binary` | REG_BINARY | Donnees binaires |

---

### Supprimer une valeur

```powershell
# Delete the value "MaValeur" from the MonTest key
Remove-ItemProperty -Path "HKCU:\Software\MonTest" -Name "MaValeur"
```

```title="Resultat attendu"
Aucune sortie si la commande reussit. C'est normal !
```

Aucun message si tout va bien. La valeur a disparu.

### Supprimer une cle

```powershell
# Delete the entire MonTest key and everything inside it
Remove-Item -Path "HKCU:\Software\MonTest" -Recurse
```

```title="Resultat attendu"
Aucune sortie si la commande reussit. C'est normal !
```

!!! danger "La suppression est definitive"
    Il n'y a pas de corbeille pour le registre. Une fois supprime, c'est parti. Assurez-vous d'avoir une sauvegarde si vous supprimez quelque chose d'important.

!!! note "Le parametre -Recurse"
    Si la cle contient des sous-cles, PowerShell vous demandera une confirmation. Le parametre `-Recurse` lui dit de tout supprimer sans poser de question. Utilisez-le uniquement si vous etes sur de vouloir tout effacer.

!!! quote "En resume"
    - `New-Item` cree une cle, `New-ItemProperty` cree une valeur (avec `-PropertyType` pour preciser le type).
    - `Remove-ItemProperty` supprime une valeur, `Remove-Item -Recurse` supprime une cle entiere. La suppression est definitive.

---

## Les 5 commandes a retenir

Voici votre aide-memoire. Pas besoin de tout retenir par coeur : gardez cette page dans vos favoris.

| Commande | Ce qu'elle fait | Equivalent dans Regedit |
|----------|----------------|------------------------|
| `Get-ItemProperty` | Lire une valeur | Double-cliquer sur une valeur |
| `Set-ItemProperty` | Modifier une valeur | Modifier les donnees d'une valeur |
| `New-Item` | Creer une cle | Clic droit > Nouveau > Cle |
| `New-ItemProperty` | Creer une valeur | Clic droit > Nouveau > Valeur chaine |
| `Remove-ItemProperty` | Supprimer une valeur | Clic droit sur une valeur > Supprimer |

!!! tip "Un sixieme bonus"
    `Remove-Item` supprime une **cle** entiere (avec tout ce qu'elle contient). C'est comme supprimer un dossier entier dans l'Explorateur de fichiers.

!!! quote "En resume"
    - Les 5 commandes essentielles : `Get-ItemProperty` (lire), `Set-ItemProperty` (modifier), `New-Item` (creer une cle), `New-ItemProperty` (creer une valeur), `Remove-ItemProperty` (supprimer une valeur).
    - `Remove-Item` (bonus) supprime une cle entiere avec tout son contenu.

---

## Exercices pratiques

### Exercice 1 : lire le numero de build de Windows

Le numero de build est une information technique qui identifie la version exacte de votre Windows. Les forums de support le demandent souvent.

!!! example "Essayez vous-meme"
    **Objectif** : afficher le numero de build de votre Windows.

    **Indice** : la valeur s'appelle `CurrentBuildNumber` et se trouve dans la cle de version de Windows.

    ??? note "Solution (cliquez pour reveler)"
        ```powershell
        # Read the Windows build number
        Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" -Name CurrentBuildNumber
        ```

        ```title="Resultat attendu"
        CurrentBuildNumber : 22631
        ```

        Votre numero sera different, c'est normal. Il depend de votre version de Windows et des mises a jour installees.

---

### Exercice 2 : lister les programmes au demarrage

!!! example "Essayez vous-meme"
    **Objectif** : voir tous les programmes qui se lancent automatiquement quand vous ouvrez votre session.

    **Indice** : la cle `Run` contient les programmes de demarrage pour votre profil utilisateur.

    ??? note "Solution (cliquez pour reveler)"
        ```powershell
        # List all startup programs for the current user
        Get-ItemProperty "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
        ```

        ```title="Resultat attendu"
        SecurityHealth  : C:\Windows\system32\SecurityHealthSystray.exe
        OneDrive        : "C:\Program Files\Microsoft OneDrive\OneDrive.exe" /background
        Discord         : "C:\Users\VotreNom\AppData\Local\Discord\Update.exe" --processStart Discord.exe
        ```

        Vous verrez la liste des programmes avec leurs chemins. Si la liste est vide, c'est que vous n'avez aucun programme configure pour se lancer au demarrage via le registre (ils peuvent aussi etre configures ailleurs).

        Pour voir aussi les programmes au demarrage pour **tous les utilisateurs** (necessite le mode Administrateur) :

        ```powershell
        # List startup programs for all users (requires admin)
        Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
        ```

        ```title="Resultat attendu"
        SecurityHealth  : C:\Windows\system32\SecurityHealthSystray.exe
        VMware User Process : "C:\Program Files\VMware\VMware Tools\vmtoolsd.exe" -n vmusr
        ```

---

### Exercice 3 : creer, lire et supprimer (le cycle complet)

Cet exercice vous fait pratiquer toutes les commandes dans un environnement sans risque.

!!! example "Essayez vous-meme"
    **Objectif** : creer une cle de test, y ajouter une valeur, la lire, puis tout nettoyer.

    ??? note "Solution (cliquez pour reveler)"
        **Etape 1** : creer une cle de test.

        ```powershell
        # Create a test key
        New-Item -Path "HKCU:\Software\MonTestPowerShell"
        ```

        ```title="Resultat attendu"
            Hive: HKEY_CURRENT_USER\Software

        Name                           Property
        ----                           --------
        MonTestPowerShell
        ```

        **Etape 2** : ajouter une valeur dans cette cle.

        ```powershell
        # Add a string value to the test key
        New-ItemProperty -Path "HKCU:\Software\MonTestPowerShell" -Name "Auteur" -Value "Moi" -PropertyType String
        ```

        ```title="Resultat attendu"
        Auteur       : Moi
        PSPath       : Microsoft.PowerShell.Core\Registry::HKEY_CURRENT_USER\Software\MonTestPowerShell
        PSParentPath : Microsoft.PowerShell.Core\Registry::HKEY_CURRENT_USER\Software
        PSChildName  : MonTestPowerShell
        PSProvider   : Microsoft.PowerShell.Core\Registry
        ```

        **Etape 3** : lire la valeur pour verifier.

        ```powershell
        # Read back the value
        Get-ItemProperty "HKCU:\Software\MonTestPowerShell" -Name "Auteur"
        ```

        ```title="Resultat attendu"
        Auteur : Moi
        ```

        **Etape 4** : modifier la valeur.

        ```powershell
        # Change the value
        Set-ItemProperty -Path "HKCU:\Software\MonTestPowerShell" -Name "Auteur" -Value "Mon nom"
        ```

        ```title="Resultat attendu"
        Aucune sortie si la commande reussit. C'est normal !
        ```

        **Etape 5** : verifier la modification.

        ```powershell
        # Verify the change
        Get-ItemProperty "HKCU:\Software\MonTestPowerShell" -Name "Auteur"
        ```

        ```title="Resultat attendu"
        Auteur : Mon nom
        ```

        **Etape 6** : nettoyer en supprimant tout.

        ```powershell
        # Delete the entire test key
        Remove-Item -Path "HKCU:\Software\MonTestPowerShell" -Recurse
        ```

        ```title="Resultat attendu"
        Aucune sortie si la commande reussit. C'est normal !
        ```

        **Etape 7** : verifier que la cle n'existe plus.

        ```powershell
        # Try to read the deleted key (should produce an error)
        Get-ItemProperty "HKCU:\Software\MonTestPowerShell"
        ```

        ```title="Resultat attendu"
        Get-ItemProperty : Cannot find path 'HKCU:\Software\MonTestPowerShell' because it does not exist.
        ```

        Si vous voyez ce message d'erreur, c'est que tout a bien ete nettoye. Bien joue.

!!! quote "En resume"
    - L'exercice 1 lit le numero de build Windows, l'exercice 2 liste les programmes au demarrage, et l'exercice 3 pratique le cycle complet : creer, lire, modifier, supprimer.
    - Creez des cles de test sous `HKCU:\Software` pour vous entrainer sans risque.

---

## Envie d'aller plus loin ?

Ce chapitre vous a donne les bases : lire, modifier, creer et supprimer des valeurs du registre avec PowerShell. C'est deja beaucoup.

Si vous voulez aller plus loin, voici ce que vous pouvez explorer :

!!! note "Dans la Bible du Registre Windows"
    - **Chapitre 10 — Scripts et automatisation** : apprenez a ecrire des scripts PowerShell qui enchainent plusieurs modifications automatiquement. Ideal pour configurer un nouveau PC en quelques secondes.
    - **Chapitre 25 — API avancees** : pour les plus curieux, decouvrez comment interagir avec le registre depuis des programmes et des outils avances.

!!! quote "En resume"
    - Pour aller plus loin, la Bible du Registre couvre les scripts avances (chapitre 10) et les API avancees (chapitre 25).

---

!!! quote "En resume"
    - PowerShell est une **telecommande** pour le registre : plus rapide et plus puissant que Regedit pour les taches repetitives.
    - Le registre se manipule comme des **dossiers et fichiers** : `cd` pour naviguer, `dir` pour lister, `Get-ItemProperty` pour lire.
    - Les 5 commandes essentielles sont : `Get-ItemProperty`, `Set-ItemProperty`, `New-Item`, `New-ItemProperty`, `Remove-ItemProperty`.
    - **Sauvegardez toujours** avant de modifier quoi que ce soit. Le protocole en 3 etapes (sauvegarder, modifier, verifier) est votre meilleur allie.
    - N'ayez pas peur d'experimenter : creez des cles de test sous `HKCU:\Software` pour vous entrainer sans risque.
