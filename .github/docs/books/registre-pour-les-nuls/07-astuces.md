---
tags:
  - registre
  - débutant
  - astuces
---

# Astuces pratiques

!!! abstract "Ce que vous allez apprendre"
    - Des astuces de personnalisation du Bureau
    - Des reglages pour ameliorer la reactivite de Windows
    - Des ajustements reseau utiles
    - Des tweaks de securite pour proteger votre PC
    - Comment appliquer plusieurs modifications d'un coup avec un fichier `.reg`

!!! warning "Rappel"
    Avant **chaque** modification, sauvegardez la cle concernee (clic droit > Exporter). C'est la regle d'or. Chapitre 5 si vous avez besoin d'un rappel.

---

## Personnalisation du Bureau

### Changer le delai de survol de la barre des taches

Quand vous survolez une icone dans la barre des taches, Windows attend un peu avant d'afficher l'apercu de la fenetre. Vous pouvez raccourcir ce delai.

!!! example "Essayez vous-meme"
    **Cle** : collez dans la barre d'adresse de Regedit :
    ```
    HKEY_CURRENT_USER\Control Panel\Mouse
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `MouseHoverTime` | REG_SZ | `200` | Reduit le delai de 400 ms a 200 ms |

    ```title="Resultat attendu"

    survolez une icone dans la barre des taches. L'apercu apparait plus vite qu'avant.

    ```

    **Valeur par defaut** : `400` (pour revenir en arriere).

---

### Degrouper les fenetres dans la barre des taches

Par defaut, Windows regroupe toutes les fenetres d'une meme application sous un seul bouton. Si vous preferez voir chaque fenetre separement :

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced
    ```

    **Modification** :

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `TaskbarGlomLevel` | REG_DWORD | `2` | Chaque fenetre a son propre bouton |

!!! note "Compatibilite"
    `TaskbarGlomLevel` est surtout pertinent pour Windows 10. Sous Windows 11, la nouvelle barre des taches ignore ou limite ce reglage selon la version et le shell actif.

Les trois options possibles :

    | Valeur | Comportement |
    |:------:|:------------:|
    | `0` | Toujours regrouper |
    | `1` | Regrouper seulement si la barre est pleine |
    | `2` | Ne jamais regrouper |

    ```title="Resultat attendu"

    fermez et rouvrez votre session (ou redemarrez l'explorateur). Chaque fenetre a maintenant son propre bouton dans la barre des taches.

    ```

!!! info "Redemarrage necessaire"
    Fermez et rouvrez la session, ou redemarrez l'explorateur pour appliquer ce changement.

---

### Ajouter "Ouvrir une invite de commandes ici" au menu contextuel

Vous voulez pouvoir faire un clic droit dans un dossier et ouvrir directement une invite de commandes a cet emplacement ? Voici comment.

!!! example "Essayez vous-meme"
    **Etape 1** : Naviguez vers :
    ```
    HKEY_CLASSES_ROOT\Directory\Background\shell
    ```

    **Etape 2** : Clic droit sur `shell` > **Nouveau** > **Cle** > nommez-la `cmd`

    **Etape 3** : Cliquez sur la cle `cmd`. Double-cliquez sur **(Par defaut)** et entrez :
    ```
    Ouvrir une invite de commandes ici
    ```

    **Etape 4** : Clic droit sur `cmd` > **Nouveau** > **Cle** > nommez-la `command`

    **Etape 5** : Cliquez sur `command`. Double-cliquez sur **(Par defaut)** et entrez :
    ```
    cmd.exe /s /k pushd "%V"
    ```

    ```title="Resultat attendu"

    faites un clic droit sur le fond d'un dossier dans l'explorateur. Vous voyez maintenant "Ouvrir une invite de commandes ici" dans le menu.

    ```

!!! quote "En resume"
    - Vous pouvez personnaliser le delai de survol de la barre des taches (`MouseHoverTime`), degrouper les fenetres (`TaskbarGlomLevel`) et ajouter des options au menu contextuel.
    - Ces modifications se font dans `HKCU` et sont sans risque pour les autres utilisateurs.

---

## Performances

### Supprimer le delai de demarrage des applications

Windows impose un delai au lancement des programmes configures pour demarrer automatiquement. Ce delai peut etre supprime pour accelerer le demarrage.

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Serialize
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `StartupDelayInMSec` | REG_DWORD | `0` | Supprime le delai de demarrage |

    !!! tip "La cle n'existe probablement pas"
        La cle `Serialize` n'existe pas par defaut. Pas de panique, il faut la creer :

        1. Clic droit sur `Explorer` > **Nouveau** > **Cle** > nommez-la `Serialize`
        2. Clic droit dans le panneau de droite > **Nouveau** > **Valeur DWORD (32 bits)**
        3. Nommez-la `StartupDelayInMSec`
        4. Laissez la valeur a `0`

    ```title="Resultat attendu"

    au prochain redemarrage, vos programmes de demarrage se lancent plus rapidement.

    ```

---

### Accelerer l'apparition des sous-menus

Quand vous survolez un menu qui a des sous-menus, il y a un petit delai avant que le sous-menu apparaisse. Vous pouvez le reduire.

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKEY_CURRENT_USER\Control Panel\Desktop
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `MenuShowDelay` | REG_SZ | `100` | Reduit le delai de 400 ms a 100 ms |

    ```title="Resultat attendu"

    les menus et sous-menus s'ouvrent presque instantanement.

    ```

    **Valeur par defaut** : `400` (pour revenir en arriere).

!!! quote "En resume"
    - Vous pouvez supprimer le delai de demarrage des applications (`StartupDelayInMSec` = `0`) et accelerer l'affichage des sous-menus (`MenuShowDelay`).
    - Ces reglages rendent Windows plus reactif sans effets secondaires.

---

## Reseau

### Desactiver le proxy manuel

Si vous n'utilisez pas de **proxy manuel**, vous pouvez desactiver cette configuration pour eviter des tentatives de connexion inutiles.

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Internet Settings
    ```

    **Modification** :

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `ProxyEnable` | REG_DWORD | `0` | Desactive uniquement le proxy manuel configure dans Internet Settings |

    ```title="Resultat attendu"

    votre systeme n'essaie plus d'utiliser un proxy manuel obsolete.

!!! note "WPAD et detection automatique"
    `ProxyEnable = 0` **ne desactive pas** la detection automatique de proxy (WPAD / option "Detecter automatiquement les parametres"). Pour cela, il faut agir sur d'autres parametres de configuration reseau ou sur la configuration WinHTTP / navigateur.

    ```

!!! note "Utilisateurs en entreprise"
    Si vous etes sur un reseau d'entreprise qui utilise un proxy, **ne touchez pas** a ce reglage. Votre connexion Internet cesserait de fonctionner.

---

### Ajuster la taille du cache DNS

Le cache DNS stocke les adresses des sites web que vous avez visites, pour ne pas avoir a les rechercher a chaque fois. Vous pouvez ajuster sa duree de vie.

!!! example "Essayez vous-meme"
    **Cle** (necessite les droits administrateur) :
    ```
    HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters
    ```

    **Modifications** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `MaxCacheEntryTtlLimit` | REG_DWORD | `86400` | Duree max du cache : 24h (en secondes) |
    | `MaxNegativeCacheTtl` | REG_DWORD | `300` | Duree du cache negatif : 5 min (en secondes) |

    !!! info "C'est quoi le cache negatif ?"
        Quand un site web est introuvable, Windows se souvient de cette "non-reponse" pendant un certain temps. C'est le cache negatif. Reduire sa duree permet de reessayer plus vite si le site revient en ligne.

!!! quote "En resume"
    - `ProxyEnable = 0` desactive le **proxy manuel**, pas la detection automatique WPAD.
    - Le cache DNS peut etre ajuste pour optimiser la resolution des noms de domaine.

---

## Securite

### Verrouiller le fond d'ecran

Empecher la modification du fond d'ecran. Utile en entreprise, en salle informatique ou si vos enfants changent le fond d'ecran toutes les 5 minutes.

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\ActiveDesktop
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `NoChangingWallPaper` | REG_DWORD | `1` | Interdit le changement de fond d'ecran |

    ```title="Resultat attendu"

    dans les parametres de personnalisation, l'option de fond d'ecran est grisee.

    ```

    Pour annuler : remettre la valeur a `0` ou supprimer la valeur.

---

### Masquer un lecteur dans l'explorateur

Vous pouvez cacher un lecteur (par exemple `D:`) dans l'explorateur sans le desactiver. Les fichiers restent accessibles, mais le lecteur n'apparait plus dans "Ce PC".

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `NoDrives` | REG_DWORD | Voir le tableau ci-dessous | Masque les lecteurs specifies |

    **Tableau des valeurs par lecteur** :

    | Lecteur | Valeur decimale |
    |:-------:|:---------------:|
    | A: | 1 |
    | B: | 2 |
    | C: | 4 |
    | D: | 8 |
    | E: | 16 |

    Pour masquer **plusieurs** lecteurs, additionnez les valeurs.

    Par exemple : masquer `D:` et `E:` = 8 + 16 = **24**.

!!! info "Juste cache, pas bloque"
    Le lecteur est **invisible** dans l'explorateur mais reste **accessible** en tapant son chemin directement (ex. : `D:\` dans la barre d'adresse).

    Pour bloquer reellement l'acces, il faut modifier les permissions NTFS (ce qui depasse le cadre de ce livre).

!!! quote "En resume"
    - Vous pouvez verrouiller le fond d'ecran (`NoChangingWallPaper`) et masquer des lecteurs dans l'Explorateur (`NoDrives`).
    - Masquer un lecteur le rend invisible mais pas inaccessible : le chemin direct fonctionne toujours.

---

## Appliquer plusieurs modifications d'un coup

Si vous avez plusieurs modifications a faire, vous pouvez creer un **fichier `.reg`** qui les applique toutes en un double-clic.

C'est comme ecrire une liste de courses au lieu de faire un aller-retour au magasin pour chaque article.

### Comment creer un fichier .reg

!!! example "Pas a pas"
    1. Ouvrir le **Bloc-notes** (++win+r++ puis `notepad`)
    2. Ecrire le contenu (voir l'exemple ci-dessous)
    3. **Fichier** > **Enregistrer sous...**
    4. Dans **Type**, choisir `Tous les fichiers (*.*)`
    5. Nommer le fichier avec l'extension `.reg` (ex. : `mes_tweaks.reg`)
    6. Cliquer **Enregistrer**

### Exemple de fichier .reg

```ini
Windows Registry Editor Version 5.00

; Show file extensions
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"HideFileExt"=dword:00000000

; Open explorer on "This PC"
[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"LaunchTo"=dword:00000001

; Reduce menu delay
[HKEY_CURRENT_USER\Control Panel\Desktop]
"MenuShowDelay"="100"
```

### Comment l'utiliser

**Double-cliquer** sur le fichier `.reg` et confirmer l'import. Toutes les modifications s'appliquent d'un coup.

!!! warning "Premiere ligne obligatoire"
    Le fichier **doit** commencer par `Windows Registry Editor Version 5.00`. Sans cette ligne, Windows ne reconnait pas le fichier comme un fichier de registre valide.

!!! tip "Les commentaires"
    Les lignes qui commencent par `;` sont des commentaires. Elles sont ignorees par Windows mais vous aident a comprendre ce que fait chaque section. Utilisez-les abondamment !

!!! quote "En resume"
    - Un fichier `.reg` permet d'appliquer **plusieurs modifications** en un seul double-clic, comme une liste de courses.
    - Le fichier doit commencer par `Windows Registry Editor Version 5.00` et les commentaires (`;`) sont recommandes pour documenter chaque modification.

---

!!! quote "En resume"
    | Categorie | Astuce phare | Difficulte |
    |:---------:|:------------:|:----------:|
    | Bureau | Degrouper les fenetres | Facile |
    | Bureau | Menu contextuel personnalise | Moyen |
    | Performance | Supprimer le delai de demarrage | Facile |
    | Performance | Accelerer les menus | Facile |
    | Reseau | Desactiver la detection de proxy | Facile |
    | Securite | Masquer un lecteur | Moyen |
    | Productivite | Fichier `.reg` pour tout automatiser | Moyen |
