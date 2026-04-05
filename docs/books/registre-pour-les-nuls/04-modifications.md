---
tags:
  - registre
  - débutant
  - modifications
---

# Modifications simples et utiles

!!! abstract "Ce que vous allez apprendre"
    - Comment modifier une valeur existante (texte ou nombre)
    - Comment creer une nouvelle valeur ou une nouvelle cle
    - Comment supprimer une valeur ou une cle
    - 5 modifications utiles que vous pouvez faire tout de suite
    - 5 autres modifications utiles pour arriver a 10 exemples concrets
    - Quand faut-il redemarrer apres un changement

!!! warning "Avant toute chose : sauvegardez !"
    Avant **chaque** modification, faites une sauvegarde. C'est la ceinture de securite.

    **Methode rapide** : dans Regedit, faites un clic droit sur la cle que vous allez modifier > **Exporter** > enregistrez le fichier `.reg` sur votre Bureau.

    Le chapitre 5 detaille toutes les methodes de sauvegarde.

---

## Modifier une valeur existante

### Modifier du texte (REG_SZ)

!!! example "Pas a pas"
    1. Naviguer vers la cle contenant la valeur
    2. **Double-cliquer** sur la valeur dans le panneau de droite
    3. Une petite fenetre s'ouvre avec le champ **Donnees de la valeur**
    4. Modifier le texte
    5. Cliquer **OK**

    **Ce que vous voyez** :
    ```
    ┌─ Modifier la chaîne ──────────────────────┐
    │                                            │
    │  Nom de la valeur : MonReglage             │
    │                                            │
    │  Données de la valeur :                    │
    │  ┌──────────────────────────────────────┐  │
    │  │ ancien texte → nouveau texte         │  │
    │  └──────────────────────────────────────┘  │
    │                                            │
    │              [ OK ]  [ Annuler ]           │
    └────────────────────────────────────────────┘
    ```

---

### Modifier un nombre (REG_DWORD)

!!! example "Pas a pas"
    1. **Double-cliquer** sur la valeur DWORD
    2. Dans la fenetre qui s'ouvre, choisir la **base** :
        - **Hexadecimal** : pour les valeurs comme `0x00000001`
        - **Decimal** : pour les nombres classiques (`1`, `2`, `3`...)
    3. Entrer la nouvelle valeur
    4. Cliquer **OK**

    **Ce que vous voyez** :
    ```
    ┌─ Modifier la valeur DWORD ────────────────┐
    │                                            │
    │  Nom de la valeur : HideFileExt            │
    │                                            │
    │  Données de la valeur :                    │
    │  ┌──────────────────────────────────────┐  │
    │  │ 0                                    │  │
    │  └──────────────────────────────────────┘  │
    │                                            │
    │  Base : ○ Hexadécimal  ● Décimal           │
    │                                            │
    │              [ OK ]  [ Annuler ]           │
    └────────────────────────────────────────────┘
    ```

!!! tip "Pas de prise de tete avec l'hexadecimal"
    La plupart des tutoriels utilisent des valeurs simples comme `0` ou `1`. Dans ce cas, le resultat est **identique** en hexadecimal et en decimal. Selectionnez "Decimal" et tapez le nombre, tout simplement.

!!! quote "En resume"
    - Pour modifier une valeur existante, **double-cliquez** dessus dans le panneau de droite et changez les donnees.
    - Pour les DWORD, selectionnez **Decimal** dans la fenetre de modification pour utiliser des nombres classiques.

---

## Creer une nouvelle valeur

Parfois un tutoriel vous demande de **creer** une valeur qui n'existe pas encore. Voici comment :

1. Naviguer vers la cle ou creer la valeur
2. **Clic droit** dans le panneau de droite (dans une zone vide)
3. **Nouveau** > choisir le type :
    - **Valeur chaine** pour du texte (`REG_SZ`)
    - **Valeur DWORD (32 bits)** pour un nombre
4. Taper le **nom exact** de la valeur (attention aux majuscules !)
5. **Double-cliquer** dessus pour entrer les donnees

!!! warning "Le nom doit etre exact"
    Si le tutoriel dit de creer `StartupDelayInMSec`, tapez exactement ca. Pas `startupdelayinmsec`, pas `Startup Delay`. Les noms sont sensibles a la casse dans certains cas.

!!! quote "En resume"
    - Pour creer une valeur : clic droit dans le panneau de droite > **Nouveau** > choisir le type, puis taper le nom exact.
    - Le nom doit correspondre **exactement** a ce que le tutoriel indique (majuscules/minuscules comprises).

---

## Creer une nouvelle cle

1. **Clic droit** sur la cle parente dans le panneau de gauche
2. **Nouveau** > **Cle**
3. Taper le nom de la cle
4. Appuyer sur ++enter++

C'est comme creer un nouveau dossier dans l'explorateur de fichiers.

!!! quote "En resume"
    - Pour creer une cle : clic droit sur la cle parente > **Nouveau** > **Cle**, puis taper le nom.
    - C'est l'equivalent de creer un nouveau dossier dans l'Explorateur de fichiers.

---

## Renommer une valeur ou une cle

Regedit n'a pas de gros bouton "Renommer", mais la fonction existe bien.

C'est le meme principe que dans l'Explorateur : on clique sur l'element, puis on change son nom directement.

### Renommer une valeur

1. **Clic droit** sur la valeur > **Renommer**
2. Taper le nouveau nom
3. Appuyer sur ++enter++

### Renommer une cle

1. **Clic droit** sur la cle dans le panneau gauche > **Renommer**
2. Taper le nouveau nom
3. Appuyer sur ++enter++

!!! warning "Renommer une cle systeme peut casser les references qui pointent vers elle par son ancien nom. Reservez cette operation aux cles que vous avez creees vous-meme."
    Si un programme attend exactement l'ancien nom, il ne retrouvera plus la cle. C'est comme changer le nom d'une rue sans prevenir le GPS.

!!! quote "En resume"
    - Pour renommer une valeur ou une cle, utilisez simplement **Clic droit > Renommer**.
    - Renommer une cle est une operation plus sensible que renommer une valeur.
    - Limitez cette manipulation aux cles que vous comprenez ou que vous avez creees vous-meme.

---

## Supprimer une valeur ou une cle

| Quoi | Comment |
|------|---------|
| Une valeur | Clic droit sur la valeur > **Supprimer** > confirmer |
| Une cle | Clic droit sur la cle > **Supprimer** > confirmer |

!!! danger "Pas de corbeille !"
    La suppression dans Regedit est **immediate et definitive**. Il n'y a pas de corbeille. C'est pour ca qu'on sauvegarde **avant**.

    Supprimer une cle supprime aussi **toutes** ses sous-cles et valeurs. C'est comme supprimer un dossier avec tout son contenu.

!!! quote "En resume"
    - La suppression dans Regedit est **immediate et definitive** : il n'y a pas de corbeille.
    - Supprimer une cle supprime aussi tout ce qu'elle contient (sous-cles et valeurs).

---

## Modifier un REG_EXPAND_SZ ou un REG_MULTI_SZ

Ces deux noms font un peu peur au debut, mais ils restent simples quand on les ramene a des exemples concrets.

### REG_EXPAND_SZ : texte avec variable d'environnement

- Cela ressemble souvent a `%USERPROFILE%\Documents`
- Vous le modifiez exactement comme un `REG_SZ` : **double-clic**, puis changement du texte
- La partie `%VARIABLE%` est remplacee par Windows au moment de l'utilisation
- Exemple : si vous voyez `%SystemRoot%\system32\cmd.exe`, Windows remplace `%SystemRoot%` par `C:\Windows`

### REG_MULTI_SZ : texte multi-lignes

- Ce type contient plusieurs lignes de texte empilees
- Quand vous double-cliquez dessus, une fenetre multi-lignes s'ouvre
- Chaque ligne represente une entree distincte dans la liste
- Pour ajouter une ligne, appuyez sur ++enter++ a la fin de la derniere, puis tapez la nouvelle entree

!!! tip "Pour les debutants, REG_EXPAND_SZ se modifie exactement comme du texte normal. Laissez les `%VARIABLE%` tels quels sauf si vous savez exactement ce que vous changez."
    Si vous remplacez une variable d'environnement par un chemin incorrect, Windows risque de ne plus retrouver le bon dossier ou le bon programme.

!!! quote "En resume"
    - `REG_EXPAND_SZ` est du texte normal avec des variables comme `%USERPROFILE%` ou `%SystemRoot%`.
    - `REG_MULTI_SZ` contient plusieurs lignes, une entree par ligne.
    - Dans les deux cas, prenez le reflexe de modifier le minimum necessaire et de laisser intact ce que vous ne comprenez pas encore.

---

## 10 modifications utiles au quotidien

Voici 10 modifications concretes que vous pouvez essayer. Pour chacune :

1. Sauvegardez la cle (clic droit > Exporter)
2. Faites la modification
3. Verifiez le resultat

---

### 1. Afficher les extensions de fichiers

Par defaut, Windows cache les extensions. Vous voyez `document` au lieu de `document.pdf`. C'est genant et parfois dangereux (un virus peut se cacher derriere `photo.jpg.exe`).

!!! example "Essayez vous-meme"
    **Cle** : collez dans la barre d'adresse de Regedit :
    ```
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `HideFileExt` | REG_DWORD | `0` | Les extensions deviennent visibles |

    ```title="Resultat attendu"

    dans l'explorateur, vos fichiers affichent maintenant `.pdf`, `.docx`, `.jpg`, etc.

    ```

    Pour annuler : remettre la valeur a `1`.

---

### 2. Ouvrir l'explorateur sur "Ce PC"

Par defaut, l'explorateur s'ouvre sur l'acces rapide. Si vous preferez voir directement vos disques :

!!! example "Essayez vous-meme"
    **Cle** (la meme que ci-dessus) :
    ```
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `LaunchTo` | REG_DWORD | `1` | L'explorateur s'ouvre sur "Ce PC" |

    ```title="Resultat attendu"

    ouvrez l'explorateur (++win+e++), vous voyez directement vos disques C:, D:, etc.

    ```

    Pour annuler : remettre la valeur a `2`.

---

### 3. Activer le theme sombre

Passer Windows et vos applications en mode sombre :

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize
    ```

    **Modifications** (deux valeurs a changer) :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `AppsUseLightTheme` | REG_DWORD | `0` | Applications en mode sombre |
    | `SystemUsesLightTheme` | REG_DWORD | `0` | Barre des taches en mode sombre |

    ```title="Resultat attendu"

    l'interface passe en theme sombre quasi immediatement.

    ```

    Pour annuler : remettre les deux valeurs a `1`.

---

### 4. Desactiver les animations de fenetres

Rendre Windows plus reactif en supprimant les animations quand vous reduisez ou agrandissez une fenetre :

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKEY_CURRENT_USER\Control Panel\Desktop\WindowMetrics
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `MinAnimate` | REG_SZ | `0` | Les fenetres apparaissent/disparaissent instantanement |

    ```title="Resultat attendu"

    reduisez une fenetre dans la barre des taches. Plus d'animation, c'est instantane.

    ```

    Pour annuler : remettre la valeur a `1`.

---

### 5. Restaurer le menu contextuel classique (Windows 11)

Windows 11 a remplace le menu du clic droit par une version simplifiee. Pour retrouver l'ancien menu complet :

!!! example "Essayez vous-meme"
    Celui-ci est un peu plus complexe. On va **creer** des cles qui n'existent pas.

    **Etape 1** : Naviguez vers :
    ```
    HKEY_CURRENT_USER\Software\Classes\CLSID
    ```

    **Etape 2** : Clic droit sur `CLSID` > **Nouveau** > **Cle**

    **Etape 3** : Nommez-la exactement :
    ```
    {86ca1aa0-34aa-4e8b-a509-50c905bae2a2}
    ```

    **Etape 4** : Clic droit sur cette nouvelle cle > **Nouveau** > **Cle** > nommez-la `InprocServer32`

    **Etape 5** : Cliquez sur `InprocServer32`, puis double-cliquez sur **(Par defaut)** dans le panneau de droite. Laissez les donnees **vides**. Cliquez **OK**.

    **Etape 6** : Redemarrez l'explorateur ou le PC.

    ```title="Resultat attendu"

    le clic droit affiche maintenant l'ancien menu complet directement.

    ```

    Pour annuler : supprimez la cle `{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}` et redemarrez.

---

### 6. Desactiver l'ecran de verrouillage

Sur un PC fixe ou un ordinateur utilise uniquement a la maison, l'ecran de verrouillage peut sembler etre une etape de plus avant l'ecran de connexion.

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKLM\SOFTWARE\Policies\Microsoft\Windows\Personalization
    ```

    Si la cle `Personalization` n'existe pas, creez-la d'abord.

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `NoLockScreen` | REG_DWORD | `1` | Windows saute l'ecran de verrouillage et va directement vers l'ecran de connexion |

    ```title="Resultat attendu"

    au demarrage ou a la sortie de veille, vous voyez directement l'ecran de connexion.

    ```

    Pour annuler : remettre la valeur a `0` ou supprimer la valeur.

!!! note "Necessite des droits administrateur (HKLM)."

---

### 7. Afficher les fichiers caches

Tres pratique quand vous suivez un tutoriel et qu'un dossier "a disparu" alors qu'il est simplement masque par Windows.

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `Hidden` | REG_DWORD | `1` | Les fichiers et dossiers caches deviennent visibles dans l'Explorateur |

    ```title="Resultat attendu"

    les dossiers habituellement invisibles apparaissent avec une icone un peu plus discrete.

    ```

    Pour annuler : remettre la valeur a `2`.

---

### 8. Desactiver les suggestions de recherche dans le menu Demarrer

Si vous voulez que la recherche du menu Demarrer se concentre sur votre PC, ce reglage supprime les suggestions web quand vous tapez.

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Explorer
    ```

    Si la cle `Explorer` n'existe pas, creez-la d'abord.

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `DisableSearchBoxSuggestions` | REG_DWORD | `1` | Plus de suggestions web lorsque vous tapez dans le menu Demarrer |

    ```title="Resultat attendu"

    la recherche du menu Demarrer devient plus sobre et moins chargee en propositions venues du web.

    ```

    Pour annuler : supprimer la valeur ou la remettre a `0`.

---

### 9. Accelerer le menu de demarrage (reduire le delai)

Windows attend un petit instant avant d'afficher certains menus. Ce delai est discret, mais on le ressent vite quand on aime que tout s'ouvre immediatement.

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKEY_CURRENT_USER\Control Panel\Desktop
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `MenuShowDelay` | REG_SZ | `0` | Les menus apparaissent instantanement au lieu d'attendre environ 400 ms |

    ```title="Resultat attendu"

    les menus s'affichent presque sans delai quand vous les ouvrez.

    ```

    Pour annuler : remettre la valeur a `400`.

---

### 10. Desactiver le son de demarrage Windows

Utile si vous allumez souvent votre PC tot le matin, en reunion, ou dans une piece silencieuse.

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Authentication\LogonUI\BootAnimation
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `DisableStartupSound` | REG_DWORD | `1` | Le son de demarrage Windows ne joue plus |

    ```title="Resultat attendu"

    au prochain demarrage, Windows reste silencieux.

    ```

    Pour annuler : remettre la valeur a `0`.

!!! note "Necessite des droits administrateur (HKLM)."

!!! quote "En resume"
    - Cinq modifications utiles a tester : afficher les extensions, ouvrir l'Explorateur sur "Ce PC", activer le theme sombre, desactiver les animations, restaurer le menu contextuel classique.
    - Cinq autres modifications utiles : desactiver l'ecran de verrouillage, afficher les fichiers caches, couper les suggestions de recherche, accelerer l'affichage des menus, desactiver le son de demarrage.
    - Pour chaque modification, **sauvegardez d'abord** (clic droit > Exporter), puis faites le changement et verifiez le resultat.

---

## Quand faut-il redemarrer ?

Apres une modification, le changement n'est pas toujours visible immediatement. Voici un guide :

| Ce que vous avez modifie | Ce qu'il faut faire |
|--------------------------|---------------------|
| Apparence (theme, fond d'ecran) | Souvent immediat, parfois fermer/rouvrir la session |
| Reglages de l'explorateur | Redemarrer l'explorateur (voir ci-dessous) |
| Un service Windows | Redemarrer le service ou le PC |
| Configuration reseau | Redemarrer le service reseau ou le PC |
| Parametres de securite | Redemarrer le PC |

!!! tip "Redemarrer l'explorateur rapidement"
    Ouvrez une invite de commandes (++win+r++ puis `cmd`) et tapez :

    ```batch
    taskkill /f /im explorer.exe
    ```

    ```title="Resultat attendu"
    SUCCESS: The process "explorer.exe" has been terminated.
    ```

    L'ecran devient noir (c'est normal !). Puis tapez :

    ```batch
    explorer.exe
    ```

    ```title="Resultat attendu"
    Une nouvelle fenetre de l'Explorateur apparait.
    ```

    Tout revient. C'est beaucoup plus rapide qu'un redemarrage complet du PC.

!!! quote "En resume"
    - Certaines modifications sont immediates, d'autres necessitent un **redemarrage** de l'Explorateur, de la session ou du PC.
    - Pour redemarrer l'Explorateur rapidement : `taskkill /f /im explorer.exe` puis `explorer.exe` dans l'invite de commandes.

---

!!! quote "En resume"
    - **Toujours sauvegarder** avant de modifier (clic droit > Exporter)
    - **Double-clic** sur une valeur pour la modifier
    - **Clic droit > Nouveau** pour creer une valeur ou une cle
    - **Clic droit > Supprimer** pour effacer (attention, pas de corbeille !)
    - Certaines modifications necessitent un **redemarrage** pour prendre effet
    - En cas de probleme, importez votre fichier `.reg` de sauvegarde
