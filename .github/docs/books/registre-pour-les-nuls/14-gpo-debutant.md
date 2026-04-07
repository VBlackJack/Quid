---
tags:
  - registre
  - débutant
  - stratégie de groupe
  - gpedit
  - GPO
---

# Strategies de groupe pour debutants

!!! abstract "Ce que vous allez apprendre"
    - Ce qu'est une strategie de groupe (Group Policy)
    - Comment ouvrir et utiliser l'editeur gpedit.msc
    - 3 exemples concrets de strategies avec leur chemin exact
    - Le lien direct entre les strategies de groupe et le registre
    - Les 10 strategies les plus utiles a appliquer manuellement sur Windows Home
    - Ce qu'il faut savoir si votre PC est gere par une entreprise

!!! warning "Rappel essentiel"
    Avant **chaque** modification du registre, exportez la cle concernee (clic droit > Exporter). C'est votre filet de securite. Relisez le chapitre 5 si besoin.

---

## C'est quoi une strategie de groupe ?

### L'analogie de l'ecole

Imaginez une ecole. Il y a des regles qui s'appliquent a **tout le monde** dans l'etablissement : pas de telephone en classe, uniforme obligatoire, horaires fixes. Ces regles viennent de la **direction** (l'equivalent de la configuration **Ordinateur**).

Ensuite, chaque professeur peut ajouter ses propres regles pour sa classe : pas de calculatrice pendant le controle, places attribuees. Ces regles s'appliquent uniquement aux **eleves de cette classe** (l'equivalent de la configuration **Utilisateur**).

Les strategies de groupe, c'est exactement ca : un systeme qui permet a Windows de **forcer** certains reglages, soit pour tout l'ordinateur, soit pour un utilisateur en particulier.

### Pourquoi ca existe ?

Vous avez peut-etre deja remarque que certains reglages sont **grises** sur votre PC de travail. Impossible de changer le fond d'ecran, de desactiver l'antivirus ou d'installer un logiciel. Ce n'est pas un bug : c'est une **strategie de groupe** qui vous en empeche.

!!! tip "En anglais"
    On parle de **Group Policy** ou **GPO** (Group Policy Object). Vous verrez souvent ces termes dans les tutoriels.

### Les deux grands types

| Type | S'applique a... | Exemple |
|------|-----------------|---------|
| **Configuration ordinateur** | Tout le PC, quel que soit l'utilisateur connecte | Desactiver les cles USB |
| **Configuration utilisateur** | Un utilisateur en particulier | Interdire l'acces au Panneau de configuration |

!!! note "Qui gagne en cas de conflit ?"
    Si une strategie **ordinateur** et une strategie **utilisateur** se contredisent, c'est generalement la strategie **ordinateur** qui l'emporte. C'est logique : la regle du directeur d'ecole est prioritaire sur celle du professeur.

!!! quote "En resume"
    - Une strategie de groupe (GPO) est un mecanisme qui permet a Windows de **forcer** certains reglages, soit pour tout l'ordinateur, soit pour un utilisateur.
    - Il y a deux types : **Configuration ordinateur** (s'applique a tout le PC) et **Configuration utilisateur** (s'applique a un utilisateur).

---

## gpedit.msc : l'editeur de strategies locales

### Comment l'ouvrir

!!! example "Essayez vous-meme"
    1. Appuyez sur ++win+r++ pour ouvrir la fenetre **Executer**
    2. Tapez `gpedit.msc`
    3. Appuyez sur ++enter++

    L'editeur de strategies de groupe locale s'ouvre.

!!! danger "Windows Home : gpedit n'est pas disponible"
    Si vous avez une edition **Windows Home** (Famille), vous obtiendrez une erreur. L'editeur gpedit.msc est reserve aux editions **Pro**, **Enterprise** et **Education**.

    Pas de panique : la section "gpedit n'est pas disponible sur Windows Home" plus bas vous explique comment faire les memes modifications directement dans le registre.

### L'interface

L'editeur ressemble a l'Explorateur de fichiers, avec un panneau de gauche (les dossiers) et un panneau de droite (le contenu).

```
┌─ Editeur de strategie de groupe locale ──────────────────────────┐
│                                                                   │
│  ▼ Strategie Ordinateur local                                     │
│    ▼ Configuration ordinateur    │  Parametre        Etat         │
│      ▶ Parametres du logiciel   │  ────────────     ──────        │
│      ▶ Parametres Windows       │  Reglage 1        Non configure │
│      ▼ Modeles d'administration │  Reglage 2        Active        │
│        ▶ Panneau de config...   │  Reglage 3        Desactive     │
│        ▶ Reseau                 │                                  │
│        ▶ Systeme                │                                  │
│    ▼ Configuration utilisateur   │                                 │
│      ▶ Parametres du logiciel   │                                  │
│      ▶ Parametres Windows       │                                  │
│      ▼ Modeles d'administration │                                  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Les trois etats d'un parametre

Quand vous double-cliquez sur un parametre, une fenetre s'ouvre avec trois choix :

| Etat | Signification |
|------|---------------|
| **Non configure** | Windows utilise son comportement par defaut. Aucune ecriture dans le registre. |
| **Active** | Le reglage est **force**. Windows ecrit une valeur dans le registre. |
| **Desactive** | Le reglage est **explicitement bloque**. Windows ecrit une valeur differente dans le registre. |

!!! tip "Non configure ≠ Desactive"
    C'est une source de confusion frequente. "Non configure" signifie "je ne touche a rien, Windows se comporte normalement". "Desactive" signifie "je bloque volontairement cette fonctionnalite".

### Naviguer dans gpedit

L'arborescence peut sembler intimidante au debut. Voici quelques reperes :

| Dossier | Ce qu'il contient |
|---------|-------------------|
| Parametres du logiciel | Reglages pour les applications deployees par un administrateur |
| Parametres Windows | Scripts de demarrage/fermeture et parametres de securite |
| Modeles d'administration | **C'est ici que se trouvent 90% des reglages utiles** |

Le dossier **Modeles d'administration** est celui que vous utiliserez le plus souvent. Il contient des centaines de parametres organises par categorie : Panneau de configuration, Bureau, Reseau, Systeme, Composants Windows...

!!! tip "Astuce : le filtre"
    Il y a des centaines de parametres. Pour retrouver un reglage precis, utilisez le menu **Action** → **Options de filtrage** dans gpedit. Vous pouvez filtrer par mot-cle pour trouver rapidement ce que vous cherchez.

!!! quote "En resume"
    - Ouvrez gpedit avec ++win+r++ puis `gpedit.msc` (editions Pro/Enterprise/Education uniquement, pas disponible sur Home).
    - Les trois etats d'un parametre : **Non configure** (comportement par defaut), **Active** (force), **Desactive** (bloque). Le dossier **Modeles d'administration** contient 90% des reglages utiles.

---

## 3 exemples concrets

Pour chaque exemple, on vous donne le chemin exact dans gpedit **et** la cle de registre correspondante. Ainsi, vous pouvez utiliser la methode qui vous convient.

### Exemple 1 : Interdire l'acces au Panneau de configuration

**Chemin dans gpedit** :

```
Configuration utilisateur
  → Modeles d'administration
    → Panneau de configuration
      → Interdire l'acces au Panneau de configuration et aux parametres PC
```

**Ce que ca fait** : quand un utilisateur essaie d'ouvrir le Panneau de configuration ou les Parametres Windows, il recoit un message "Cette operation a ete annulee en raison de restrictions...".

**Ce que ca ecrit dans le registre** :

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer
    "NoControlPanel" = dword:00000001
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `NoControlPanel` | REG_DWORD | `1` | Bloque l'acces au Panneau de configuration |
| `NoControlPanel` | REG_DWORD | `0` | Retablit l'acces |

!!! tip "Verification rapide"
    Apres avoir active cette strategie, essayez d'ouvrir les Parametres Windows (++win+i++). Vous verrez un message d'erreur. Pour verifier dans le registre :

    ```powershell
    # Check if Control Panel access is blocked
    reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" /v NoControlPanel
    ```

    ```title="Resultat attendu"
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer
        NoControlPanel    REG_DWORD    0x1
    ```

---

### Exemple 2 : Desactiver le telechargement automatique de pilotes

**Chemin dans gpedit** :

```
Configuration ordinateur
  → Modeles d'administration
    → Systeme
      → Installation de peripheriques
        → Restrictions d'installation de peripheriques
          → Empecher l'installation de peripheriques non decrits
            par d'autres parametres de strategie
```

**Ce que ca fait** : Windows ne telecharge plus automatiquement les pilotes via Windows Update. Utile si une mise a jour de pilote cause des problemes.

**Ce que ca ecrit dans le registre** :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\DeviceInstall\Restrictions
    "DenyUnspecified" = dword:00000001
```

---

### Exemple 3 : Forcer un fond d'ecran

**Chemin dans gpedit** :

```
Configuration utilisateur
  → Modeles d'administration
    → Bureau
      → Bureau
        → Papier peint du Bureau
```

**Ce que ca fait** : tous les utilisateurs ont le meme fond d'ecran, et ils ne peuvent pas le changer.

**Ce que ca ecrit dans le registre** :

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\System
    "Wallpaper"      = "C:\Chemin\Vers\Image.jpg"
    "WallpaperStyle" = "2"
```

| Valeur de WallpaperStyle | Comportement |
|:------------------------:|--------------|
| `0` | Centre |
| `2` | Etire |
| `6` | Ajuste |
| `10` | Remplir |

!!! quote "En resume"
    - Chaque parametre dans gpedit correspond a une **valeur dans le registre** : bloquer le Panneau de configuration (`NoControlPanel`), desactiver le telechargement de pilotes (`DenyUnspecified`), forcer un fond d'ecran (`Wallpaper`).
    - Pour chaque exemple, le chemin dans gpedit et la cle du registre sont fournis pour les deux methodes.

---

## Le lien avec le registre

### Ce qui se passe en coulisses

Quand vous activez un parametre dans gpedit.msc, Windows ne fait qu'une seule chose : **ecrire une valeur dans le registre**, dans une zone speciale reservee aux strategies.

Les strategies s'ecrivent dans des sous-cles `Policies` :

| Portee | Emplacement dans le registre |
|--------|------------------------------|
| Ordinateur | `HKLM\SOFTWARE\Policies\` |
| Utilisateur | `HKCU\Software\Policies\` |

!!! note "Priorite des Policies"
    Les cles sous `Policies` ont **priorite** sur les cles normales. Si une valeur existe a la fois dans `HKCU\Software\Microsoft\...` et dans `HKCU\Software\Policies\Microsoft\...`, c'est la version `Policies` qui gagne. C'est pour ca que certains reglages apparaissent grises dans les Parametres : Windows detecte qu'une strategie est en place et vous empeche de la contredire.

### Visualiser le mecanisme

```
┌─────────────────────────────────────┐
│         gpedit.msc                  │
│  "Activer le parametre X"          │
└──────────────┬──────────────────────┘
               │ ecrit dans
               ▼
┌─────────────────────────────────────┐
│         Registre                    │
│  HKLM\SOFTWARE\Policies\...        │
│  ou HKCU\Software\Policies\...     │
└──────────────┬──────────────────────┘
               │ lu par
               ▼
┌─────────────────────────────────────┐
│         Windows                     │
│  Applique le reglage force          │
│  (et grise l'option dans les        │
│   Parametres)                       │
└─────────────────────────────────────┘
```

### Exercice de verification

!!! example "Essayez vous-meme"
    1. Ouvrez **gpedit.msc**
    2. Allez dans : Configuration utilisateur → Modeles d'administration → Bureau → Bureau → Papier peint du Bureau
    3. Activez le parametre et entrez un chemin d'image (par exemple `C:\Windows\Web\Wallpaper\Windows\img0.jpg`)
    4. Cliquez OK
    5. Maintenant, ouvrez **Regedit** (++win+r++ → `regedit`)
    6. Naviguez vers : `HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\System`
    7. Vous voyez la valeur `Wallpaper` avec le chemin que vous venez d'entrer

    Vous venez de voir la **preuve directe** que gpedit ecrit dans le registre.

    Pour annuler : retournez dans gpedit et remettez le parametre sur "Non configure". La cle du registre disparait.

!!! quote "En resume"
    - Activer un parametre dans gpedit ne fait qu'ecrire une valeur dans les cles `Policies` du registre (`HKLM\SOFTWARE\Policies\` ou `HKCU\Software\Policies\`).
    - Les cles `Policies` ont **priorite** sur les cles normales, c'est pourquoi certains reglages apparaissent grises dans les Parametres.

---

## gpedit n'est pas disponible sur Windows Home

Si vous avez Windows Home (Famille), pas de gpedit.msc. Mais pas de panique : puisque les strategies de groupe ne font qu'ecrire dans le registre, vous pouvez **ecrire directement** les memes valeurs avec Regedit.

!!! tip "Meme resultat, methode differente"
    C'est comme envoyer un courrier : gpedit est le facteur, le registre est la boite aux lettres. Sans facteur, vous pouvez deposer le courrier vous-meme.

### Les 10 strategies les plus utiles

Voici les 10 modifications les plus couramment appliquees via gpedit, avec leur equivalent registre pour Windows Home :

| # | Description | Cle du registre | Valeur | Type | Donnees |
|:-:|-------------|-----------------|--------|:----:|:-------:|
| 1 | Bloquer l'acces au Panneau de configuration | `HKCU\...\Policies\Explorer` | `NoControlPanel` | REG_DWORD | `1` |
| 2 | Desactiver le redemarrage auto apres Windows Update | `HKLM\...\Policies\Microsoft\Windows\WindowsUpdate\AU` | `NoAutoRebootWithLoggedOnUsers` | REG_DWORD | `1` |
| 3 | Masquer certains lecteurs dans l'Explorateur | `HKCU\...\Policies\Explorer` | `NoDrives` | REG_DWORD | voir note |
| 4 | Desactiver le stockage USB | `HKLM\SYSTEM\CurrentControlSet\Services\USBSTOR` | `Start` | REG_DWORD | `4` |
| 5 | Desactiver l'ecran de verrouillage | `HKLM\SOFTWARE\Policies\Microsoft\Windows\Personalization` | `NoLockScreen` | REG_DWORD | `1` |
| 6 | Forcer la complexite des mots de passe | — | — | — | voir note |
| 7 | Desactiver Cortana / Copilot | `HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search` | `AllowCortana` | REG_DWORD | `0` |
| 8 | Forcer un fond d'ecran | `HKCU\...\Policies\System` | `Wallpaper` | REG_SZ | chemin |
| 9 | Desactiver les conseils et suggestions Windows | `HKCU\Software\Microsoft\Windows\CurrentVersion\ContentDeliveryManager` | `SubscribedContent-338389Enabled` | REG_DWORD | `0` |

!!! info "Approche GPO standard"
    `USBSTOR\Start = 4` fonctionne comme coupe-circuit bas niveau, mais l'approche GPO la plus propre reste en general **System > Removable Storage Access** ou les **Device Installation Restrictions**. Ces politiques sont plus explicites, plus auditables et moins susceptibles d'avoir des effets de bord sur d'autres peripheriques USB.
| 10 | Bloquer le Microsoft Store | `HKLM\SOFTWARE\Policies\Microsoft\WindowsStore` | `RemoveWindowsStore` | REG_DWORD | `1` |

!!! note "Notes sur le tableau"
    **Ligne 3 (Masquer des lecteurs)** : la valeur `NoDrives` utilise un masque binaire. Pour masquer le lecteur D: uniquement, la valeur est `8`. Pour masquer C: et D:, c'est `12`. Le calcul : A=1, B=2, C=4, D=8, E=16...

    **Ligne 6 (Complexite des mots de passe)** : cette strategie ne passe **pas** par le registre. Elle utilise un outil different : `secpol.msc` (Strategie de securite locale). C'est une exception notable. Et `secpol.msc` n'est pas non plus disponible sur Windows Home.

!!! example "Essayez vous-meme"
    Pour appliquer la modification n°1 (bloquer le Panneau de configuration) sur Windows Home :

    ```powershell
    # Block access to Control Panel and Settings
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" /v NoControlPanel /t REG_DWORD /d 1 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

    Pour annuler :

    ```powershell
    # Restore access to Control Panel and Settings
    reg delete "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" /v NoControlPanel /f
    ```

    ```title="Resultat attendu"
    The operation completed successfully.
    ```

!!! quote "En resume"
    - Sur Windows Home, gpedit n'est pas disponible, mais vous pouvez ecrire les memes valeurs directement dans le registre.
    - Les 10 strategies les plus utiles sont listees avec leur cle de registre equivalente, applicables sur toutes les editions de Windows.

---

## Attention aux entreprises

### Votre PC de travail, c'est different

Si votre PC est fourni par votre employeur, les strategies de groupe ne viennent probablement **pas** de votre machine locale. Elles arrivent d'un **serveur central** appele Active Directory (ou Entra ID / Intune pour les environnements modernes).

!!! info "L'analogie"
    Imaginez que le reglement de votre ecole est decide par le rectorat. Meme si le directeur de l'ecole efface le reglement du tableau, le rectorat en renverra une copie. C'est pareil avec Active Directory : les strategies sont reappliquees automatiquement.

### Comment savoir si votre PC est gere ?

Quelques indices qui ne trompent pas :

- Certains reglages dans les Parametres sont **grises** avec le message "Certains parametres sont geres par votre organisation"
- Vous ne pouvez pas installer certains logiciels
- Votre fond d'ecran ou ecran de verrouillage est impose
- Des logiciels d'entreprise (antivirus, VPN) sont installes sans que vous les ayez choisis

### Verifier les strategies appliquees

Pour voir les strategies qui s'appliquent a votre PC, ouvrez une invite de commandes **en administrateur** et tapez :

```batch
REM Display applied Group Policies summary
gpresult /r
```

```title="Resultat attendu"
RESULTATS DE LA STRATEGIE DE GROUPE
------------------------------------
Nom de l'ordinateur : PC-BUREAU
Nom du domaine      : ENTREPRISE.LOCAL

CONFIGURATION ORDINATEUR
  Objets de strategie de groupe appliques :
    Default Domain Policy
    Securite-Postes-Travail
    Restrictions-USB

CONFIGURATION UTILISATEUR
  Objets de strategie de groupe appliques :
    Default Domain Policy
    Bureau-Standard-Employes
```

!!! tip "Pour un rapport detaille"
    Vous pouvez generer un rapport HTML complet avec :

    ```batch
    REM Generate detailed Group Policy report as HTML
    gpresult /h C:\Users\%USERNAME%\Desktop\rapport-gpo.html
    ```

    ```title="Resultat attendu"
    Un fichier rapport-gpo.html est cree sur le Bureau.
    ```

    Ouvrez ensuite le fichier `rapport-gpo.html` sur votre Bureau. Il contient toutes les strategies appliquees avec leurs valeurs, de maniere beaucoup plus lisible.

### Ne tentez pas de contourner

!!! danger "Important"
    Les strategies de groupe venant d'Active Directory se **reappliquent automatiquement** toutes les 90 minutes environ (et a chaque demarrage du PC). Meme si vous modifiez le registre manuellement, vos changements seront **ecrases** au prochain cycle de rafraichissement.

    De plus, tenter de contourner les strategies de securite de votre entreprise peut etre considere comme une **faute professionnelle**. Ne le faites pas.

!!! note "Pour aller plus loin"
    Le chapitre 20 de la Bible du Registre couvre les strategies de groupe en profondeur : heritage, filtrage WMI, strategies de bouclage, et diagnostic avance.

!!! quote "En resume"
    - Sur un PC d'entreprise, les strategies viennent d'**Active Directory** et se reappliquent automatiquement toutes les 90 minutes. Modifier le registre manuellement sera ecrase.
    - Utilisez `gpresult /r` pour voir les strategies appliquees a votre PC. Ne tentez **pas** de contourner les strategies de votre entreprise.

---

!!! quote "En resume"
    | Question | Reponse |
    |----------|---------|
    | C'est quoi une GPO ? | Un mecanisme qui permet a Windows de forcer des reglages |
    | Comment on y accede ? | Via `gpedit.msc` (editions Pro/Enterprise/Education uniquement) |
    | Quel lien avec le registre ? | Chaque strategie ecrit une valeur dans les cles `Policies` du registre |
    | Et sur Windows Home ? | Vous pouvez ecrire les memes valeurs directement dans le registre |
    | Et en entreprise ? | Les strategies viennent d'Active Directory et se reappliquent toutes les 90 minutes |
