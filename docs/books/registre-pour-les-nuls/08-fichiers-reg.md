---
tags:
  - registre
  - débutant
  - fichiers .reg
---

# Comprendre les fichiers .reg

!!! abstract "Ce que vous allez apprendre"
    - Ce qu'est un fichier `.reg` et a quoi il sert
    - Comment lire et comprendre chaque ligne d'un fichier `.reg`
    - Comment creer votre propre fichier `.reg` de zero
    - Les types de donnees et leur syntaxe
    - Comment supprimer des cles et des valeurs avec un fichier `.reg`
    - Comment importer et exporter en toute securite
    - Des exemples concrets a tester tout de suite

---

## C'est quoi un fichier .reg ?

Imaginez une **fiche recette de cuisine**. Au lieu de vous rappeler chaque ingredient et chaque etape par coeur, vous ecrivez tout sur une fiche. La prochaine fois, vous n'avez qu'a suivre la fiche.

Un fichier `.reg`, c'est exactement ca : une **recette ecrite** pour le registre. Au lieu de cliquer manuellement dans Regedit, vous ecrivez les modifications dans un fichier texte, et Windows les applique en un clic.

!!! info "Pourquoi c'est utile"
    - Appliquer 10 modifications en **une seule fois** au lieu de 10 manipulations manuelles
    - **Partager** une configuration avec un collegue
    - **Sauvegarder** une modification pour la re-appliquer apres une reinstallation
    - **Documenter** ce que vous avez change (le fichier est la preuve)

!!! quote "En resume"
    - Un fichier `.reg` est une **recette ecrite** pour le registre : il applique des modifications en un clic au lieu de manipulations manuelles dans Regedit.
    - Il permet d'appliquer plusieurs modifications a la fois, de partager une configuration et de documenter les changements.

---

## Anatomie d'un fichier .reg

Voici un fichier `.reg` complet. Ne paniquez pas, on va le decrypter ligne par ligne.

```ini
Windows Registry Editor Version 5.00

; Enable Dark Mode for the current user
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize]
"AppsUseLightTheme"=dword:00000000
"SystemUsesLightTheme"=dword:00000000
```

### Ligne par ligne

| Ligne | Signification |
|-------|---------------|
| `Windows Registry Editor Version 5.00` | **En-tete obligatoire**. Dit a Windows "ceci est un fichier de registre". Sans cette ligne, le fichier ne fonctionnera pas. |
| *(ligne vide)* | Separation visuelle. Les lignes vides sont ignorees. |
| `; Enable Dark Mode...` | **Commentaire**. Commence par `;`. Ignore par Windows, utile pour vous. |
| `[HKEY_CURRENT_USER\...]` | **Chemin de la cle**. Entre crochets. C'est le "dossier" dans lequel on va travailler. |
| `"AppsUseLightTheme"=dword:00000000` | **Valeur a creer ou modifier**. Nom entre guillemets, signe `=`, type `:`, puis la donnee. |

!!! warning "L'en-tete est obligatoire"
    La toute premiere ligne **doit** etre `Windows Registry Editor Version 5.00`. Pas d'espace avant, pas de ligne vide avant. C'est la carte d'identite du fichier.

    Si vous oubliez cette ligne, Windows ne reconnaitra pas le fichier.

!!! quote "En resume"
    - Un fichier `.reg` commence obligatoirement par `Windows Registry Editor Version 5.00`.
    - Les chemins de cles sont entre `[crochets]`, les valeurs au format `"Nom"=type:donnees`, et les commentaires commencent par `;`.

---

## Les types de donnees dans un fichier .reg

Chaque valeur du registre a un **type**. Voici les plus courants avec leur syntaxe dans un fichier `.reg` :

### REG_SZ (chaine de texte)

Le type le plus simple. Un texte entre guillemets.

```ini
"MonNom"="Jean Dupont"
```

!!! tip "Les guillemets dans le texte"
    Si votre texte contient des guillemets, echappez-les avec `\"` :

    ```ini
    "MonChemin"="C:\\Program Files\\Mon App\\app.exe"
    ```

    Les anti-slashes aussi doivent etre doubles : `\\` au lieu de `\`.

---

### REG_DWORD (nombre entier 32 bits)

Un nombre en hexadecimal, prefixe par `dword:`.

```ini
"MonNombre"=dword:00000001
```

| Valeur courante | Hexadecimal | Decimal |
|:---------------:|:-----------:|:-------:|
| Desactive | `dword:00000000` | 0 |
| Active | `dword:00000001` | 1 |
| Exemple 255 | `dword:000000ff` | 255 |

!!! info "Hexadecimal ?"
    Pas de panique. Pour les cas courants, vous n'utiliserez que `00000000` (zero) et `00000001` (un). C'est comme un interrupteur : eteint ou allume.

---

### REG_BINARY (donnees binaires)

Des octets en hexadecimal, separes par des virgules, prefixes par `hex:`.

```ini
"MesDonnees"=hex:01,a2,ff,00,4b
```

!!! note "Rarement utilise manuellement"
    Les valeurs binaires sont generalement gerees par les applications elles-memes. Vous n'aurez presque jamais a en ecrire a la main.

---

### REG_EXPAND_SZ (chaine avec variables d'environnement)

Comme REG_SZ, mais peut contenir des variables comme `%USERPROFILE%`. Prefixe par `hex(2):` suivi de la chaine encodee en UTF-16 LE.

```ini
"MonChemin"=hex(2):25,00,55,00,53,00,45,00,52,00,50,00,52,00,4f,00,\
  46,00,49,00,4c,00,45,00,25,00,5c,00,44,00,6f,00,63,00,75,00,6d,00,\
  65,00,6e,00,74,00,73,00,00,00
```

!!! tip "Astuce pratique"
    L'encodage hexadecimal de REG_EXPAND_SZ est penible a ecrire a la main. Methode recommandee :

    1. Creez la valeur dans Regedit manuellement
    2. Exportez la cle (clic droit > Exporter)
    3. Le fichier `.reg` exporte contiendra le bon encodage

---

### REG_MULTI_SZ (liste de chaines)

Une liste de textes, prefixee par `hex(7):`, encodee en UTF-16 LE avec des separateurs nuls.

```ini
"MaListe"=hex(7):50,00,72,00,65,00,6d,00,69,00,65,00,72,00,00,00,\
  44,00,65,00,75,00,78,00,69,00,e8,00,6d,00,65,00,00,00,00,00
```

!!! note "Meme conseil"
    Pour les types `hex(2)` et `hex(7)`, exportez depuis Regedit plutot que d'encoder a la main. C'est beaucoup plus fiable.

---

### REG_QWORD (nombre entier 64 bits)

Prefixe par `hex(b):`, encode en little-endian.

```ini
"MonGrandNombre"=hex(b):01,00,00,00,00,00,00,00
```

---

### Tableau recapitulatif

| Type registre | Prefixe dans .reg | Exemple |
|---------------|-------------------|---------|
| REG_SZ | `"..."` (guillemets) | `"Nom"="Valeur"` |
| REG_DWORD | `dword:` | `"Actif"=dword:00000001` |
| REG_BINARY | `hex:` | `"Data"=hex:01,ff,00` |
| REG_EXPAND_SZ | `hex(2):` | *(encodage hexadecimal)* |
| REG_MULTI_SZ | `hex(7):` | *(encodage hexadecimal)* |
| REG_QWORD | `hex(b):` | `"Big"=hex(b):01,00,...` |

!!! quote "En resume"
    - Les deux types les plus courants dans un fichier `.reg` sont `"texte"` (REG_SZ) et `dword:00000001` (REG_DWORD).
    - Les types complexes (REG_EXPAND_SZ, REG_MULTI_SZ) utilisent un encodage hexadecimal (`hex(2):`, `hex(7):`) ; exportez depuis Regedit plutot que d'encoder a la main.

---

## Creer votre premier fichier .reg

### Etape par etape

**Objectif** : creer un fichier `.reg` qui active le mode sombre de Windows.

**Etape 1** -- Ouvrir le Bloc-notes :

Appuyez sur ++win+r++, tapez `notepad`, puis ++enter++.

**Etape 2** -- Ecrire le contenu :

```ini
Windows Registry Editor Version 5.00

; Activate Dark Mode for apps and system
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize]
"AppsUseLightTheme"=dword:00000000
"SystemUsesLightTheme"=dword:00000000
```

**Etape 3** -- Enregistrer correctement :

1. Fichier > Enregistrer sous
2. Dans "Type", selectionnez **Tous les fichiers (\*.\*)**
3. Nommez le fichier `dark-mode.reg`
4. Enregistrez sur le Bureau

!!! danger "Piege classique"
    Si vous laissez "Fichiers texte" dans le type, le fichier sera enregistre comme `dark-mode.reg.txt`. Windows ne le reconnaitra pas comme un fichier de registre.

    Verifiez toujours que votre fichier a bien l'icone de registre (un cube bleu) sur le Bureau.

**Etape 4** -- L'icone doit ressembler a ca :

```
📄 dark-mode.reg  →  ❌ Mauvais (icone texte)
🟦 dark-mode.reg  →  ✅ Bon (icone registre)
```

!!! quote "En resume"
    - Pour creer un fichier `.reg` : ouvrez le Bloc-notes, ecrivez le contenu, et enregistrez avec le type **Tous les fichiers** en nommant le fichier `.reg`.
    - Le piege classique : si le type reste sur "Fichiers texte", le fichier sera enregistre comme `.reg.txt` et ne fonctionnera pas.

---

## Supprimer des valeurs et des cles

### Supprimer une valeur

Pour supprimer une valeur, mettez un signe `-` (moins) apres le `=` :

```ini
Windows Registry Editor Version 5.00

; Delete a specific value
[HKEY_CURRENT_USER\Software\MonApp]
"ValeurASupprimer"=-
```

Cela supprime la valeur `ValeurASupprimer` de la cle `MonApp`, sans toucher aux autres valeurs.

---

### Supprimer une cle entiere

Pour supprimer une cle entiere (et toutes ses valeurs et sous-cles), placez un `-` devant le chemin :

```ini
Windows Registry Editor Version 5.00

; Delete the entire MonApp key and all its contents
[-HKEY_CURRENT_USER\Software\MonApp]
```

!!! danger "Attention : c'est irreversible"
    La suppression d'une cle supprime **tout** ce qu'elle contient : valeurs, sous-cles, et les sous-cles des sous-cles. C'est comme supprimer un dossier et tout son contenu.

    Exportez toujours la cle avant de la supprimer.

!!! quote "En resume"
    - Pour supprimer une valeur : `"NomValeur"=-` (signe moins apres le `=`).
    - Pour supprimer une cle entiere : `[-HKEY_...\Chemin]` (tiret devant le chemin entre crochets). C'est irreversible.

---

## Importer un fichier .reg

### Methode 1 : double-clic

1. Double-cliquez sur le fichier `.reg`
2. Windows affiche un avertissement de securite
3. Cliquez **Oui** pour confirmer

Vous verrez :

```
Voulez-vous vraiment continuer ?
L'ajout d'informations peut involontairement modifier ou supprimer
des valeurs et empêcher le fonctionnement correct de certains composants.
```

Puis si tout va bien :

```
Les informations contenues dans [chemin]\dark-mode.reg ont été
inscrites dans le registre.
```

---

### Methode 2 : en ligne de commande

```batch
reg import "C:\Users\User\Desktop\dark-mode.reg"
```

```title="Resultat attendu"
L'operation a reussi.
```

!!! tip "Avantage de la ligne de commande"
    Pas de boite de dialogue de confirmation. Utile pour les scripts d'automatisation.

---

### Methode 3 : clic droit > Fusionner

Faites un clic droit sur le fichier `.reg` et selectionnez **Fusionner**. C'est equivalent au double-clic.

!!! quote "En resume"
    - Trois facons d'importer : **double-clic**, commande `reg import`, ou clic droit > **Fusionner**.
    - La ligne de commande n'affiche pas de boite de confirmation, ce qui est pratique pour les scripts.

---

## Exporter depuis Regedit

L'export est l'operation inverse : extraire des cles du registre vers un fichier `.reg`.

### Comment exporter

1. Ouvrez Regedit (++win+r++ > `regedit` > ++enter++)
2. Naviguez vers la cle souhaitee
3. Clic droit sur la cle > **Exporter**
4. Choisissez un emplacement et un nom
5. Cliquez **Enregistrer**

Le fichier genere contiendra toutes les valeurs de la cle et de ses sous-cles.

!!! example "Exemple d'export"
    Si vous exportez `HKCU\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize`, vous obtiendrez un fichier comme :

    ```ini
    Windows Registry Editor Version 5.00

    [HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize]
    "AppsUseLightTheme"=dword:00000000
    "ColorPrevalence"=dword:00000001
    "EnableTransparency"=dword:00000001
    "SystemUsesLightTheme"=dword:00000000
    ```

    Ce fichier est votre **sauvegarde**. En cas de probleme, double-cliquez dessus pour restaurer.

!!! quote "En resume"
    - Pour exporter : clic droit sur une cle dans Regedit > **Exporter**, choisir un emplacement et enregistrer.
    - Le fichier exporte contient toutes les valeurs de la cle et de ses sous-cles, et sert de **sauvegarde** restaurable en un double-clic.

---

## Checklist de securite avant import

Avant d'importer **n'importe quel** fichier `.reg`, suivez cette liste :

!!! warning "Verifiez TOUT avant d'importer"
    - [ ] Ouvrez le fichier avec le Bloc-notes et lisez chaque ligne
    - [ ] La premiere ligne est bien `Windows Registry Editor Version 5.00`
    - [ ] Vous comprenez chaque chemin de cle modifie
    - [ ] Vous comprenez chaque valeur modifiee
    - [ ] Il n'y a pas de suppression de cle (`[-HKEY_...]`) que vous n'avez pas prevue
    - [ ] Le fichier ne modifie pas `HKLM\SYSTEM` ou `HKLM\SECURITY` (zones critiques)
    - [ ] Vous avez cree un **point de restauration** ou exporte les cles concernees

!!! quote "En resume"
    - Avant d'importer un fichier `.reg`, ouvrez-le avec le Bloc-notes et verifiez chaque ligne : en-tete, chemins de cles, valeurs, et absence de suppressions imprevues.
    - Mefiez-vous des fichiers qui modifient `HKLM\SYSTEM` ou `HKLM\SECURITY` (zones critiques).

---

## Exemples pratiques

### Exemple 1 : activer le mode sombre

```ini
Windows Registry Editor Version 5.00

; Switch Windows to Dark Mode
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize]
"AppsUseLightTheme"=dword:00000000
"SystemUsesLightTheme"=dword:00000000
```

**Resultat** : les applications et la barre des taches passent en mode sombre. Effet immediat, pas de redemarrage necessaire.

---

### Exemple 2 : afficher les extensions de fichiers

Par defaut, Windows masque les extensions (.txt, .exe, .pdf). C'est un risque de securite car un fichier `photo.jpg.exe` apparait comme `photo.jpg`.

```ini
Windows Registry Editor Version 5.00

; Show file extensions in File Explorer
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"HideFileExt"=dword:00000000
```

**Resultat** : ouvrez l'Explorateur de fichiers. Les extensions sont maintenant visibles.

Pour revenir en arriere :

```ini
Windows Registry Editor Version 5.00

; Hide file extensions (restore default)
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"HideFileExt"=dword:00000001
```

---

### Exemple 3 : afficher les fichiers caches

```ini
Windows Registry Editor Version 5.00

; Show hidden files and folders
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"Hidden"=dword:00000001
"ShowSuperHidden"=dword:00000001
```

| Valeur | Effet |
|--------|-------|
| `Hidden` = `1` | Affiche les fichiers caches |
| `ShowSuperHidden` = `1` | Affiche les fichiers systeme proteges |

---

### Exemple 4 : un fichier .reg avec plusieurs cles

Vous pouvez modifier plusieurs cles dans un seul fichier. Separez-les par une ligne vide :

```ini
Windows Registry Editor Version 5.00

; Dark Mode
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize]
"AppsUseLightTheme"=dword:00000000
"SystemUsesLightTheme"=dword:00000000

; Show file extensions
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"HideFileExt"=dword:00000000

; Show hidden files
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"Hidden"=dword:00000001
```

!!! tip "Un fichier = une configuration complete"
    Creez un fichier `.reg` avec **tous** vos reglages preferes. Apres chaque reinstallation de Windows, un double-clic et c'est fait.

!!! quote "En resume"
    - Les exemples couvrent le mode sombre, l'affichage des extensions, les fichiers caches et la combinaison de plusieurs cles dans un seul fichier `.reg`.
    - Un fichier `.reg` bien construit permet de re-appliquer tous vos reglages preferes en un double-clic apres une reinstallation.

---

## Erreurs courantes

### Oubli de l'en-tete

```ini
; THIS WILL NOT WORK - missing header
[HKEY_CURRENT_USER\Software\Test]
"MaValeur"=dword:00000001
```

Windows ne reconnaitra pas ce fichier. Ajoutez toujours `Windows Registry Editor Version 5.00` en premiere ligne.

---

### Anti-slashes simples dans les chemins de texte

```ini
; WRONG - single backslashes in string value
"MonChemin"="C:\Program Files\MonApp"

; CORRECT - double backslashes in string value
"MonChemin"="C:\\Program Files\\MonApp"
```

!!! info "Quand doubler les anti-slashes"
    Uniquement dans les **valeurs texte** (entre guillemets). Les chemins de **cles** (entre crochets) utilisent des anti-slashes simples :

    ```ini
    [HKEY_CURRENT_USER\Software\MonApp]  ← simples (chemin de cle)
    "Dossier"="C:\\Users\\Jean"          ← doubles (valeur texte)
    ```

---

### Espace avant le signe egal

```ini
; WRONG - space before the equal sign
"MaValeur" =dword:00000001

; CORRECT - no space
"MaValeur"=dword:00000001
```

---

### Mauvaise casse du prefixe

```ini
; WRONG - uppercase DWORD
"MaValeur"=DWORD:00000001

; CORRECT - lowercase dword
"MaValeur"=dword:00000001
```

Le prefixe de type (`dword:`, `hex:`) doit etre en **minuscules**.

---

### Guillemets manquants autour du nom

```ini
; WRONG - missing quotes around value name
MaValeur=dword:00000001

; CORRECT - quoted value name
"MaValeur"=dword:00000001
```

!!! quote "En resume"
    - Les erreurs les plus frequentes : oubli de l'en-tete, anti-slashes simples dans les valeurs texte (il faut les doubler), espaces avant le `=`, prefixe en majuscules.
    - Les noms de valeurs doivent toujours etre entre **guillemets** et les prefixes de type (`dword:`, `hex:`) en **minuscules**.

---

## Lignes longues et continuation

Quand une valeur hexadecimale est tres longue, vous pouvez la decouper sur plusieurs lignes avec un anti-slash `\` :

```ini
"LongueValeur"=hex:01,02,03,04,05,06,07,08,09,0a,0b,0c,0d,0e,0f,10,\
  11,12,13,14,15,16,17,18,19,1a,1b,1c,1d,1e,1f,20
```

Le `\` a la fin de la ligne indique que la suite est sur la ligne suivante. L'espace en debut de ligne suivante est ignore.

!!! quote "En resume"
    - Les valeurs hexadecimales longues peuvent etre decoupees sur plusieurs lignes avec un anti-slash `\` a la fin de chaque ligne.
    - L'espace en debut de ligne suivante est ignore par Windows.

---

!!! quote "En resume"
    - Un fichier `.reg` est un fichier texte qui contient des instructions pour le registre
    - La premiere ligne **doit** etre `Windows Registry Editor Version 5.00`
    - Les commentaires commencent par `;`
    - Les chemins de cles sont entre `[crochets]`
    - Les valeurs sont au format `"Nom"=type:donnees`
    - Le prefixe `-` supprime une valeur ou une cle
    - Double-cliquez pour importer, clic droit > Exporter pour sauvegarder
    - **Toujours** verifier le contenu avant d'importer
    - **Toujours** sauvegarder les cles avant de les modifier

---

!!! tip "Pour aller plus loin"
    Les outils avances de manipulation du registre (API Win32, .NET, scripts de deploiement) sont couverts dans [La Bible — Outils et methodes d'acces](../bible-registre-windows/05-outils.md).
