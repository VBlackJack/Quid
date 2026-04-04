---
tags:
  - registre
  - débutant
  - introduction
---

# C'est quoi la base de registre ?

!!! abstract "Ce que vous allez apprendre"
    - Ce qu'est la base de registre en termes simples
    - Pourquoi elle existe
    - Ce qu'elle contient (et ce qu'elle ne contient pas)
    - Si c'est dangereux d'y toucher
    - Ou elle se trouve sur votre disque dur

---

## Voyons d'abord un exemple concret

Appuyez sur ++win+r++, tapez `regedit`, puis naviguez jusqu'a :

```
HKEY_CURRENT_USER\Control Panel\Desktop
```

Cherchez la valeur **Wallpaper** dans la liste a droite. Vous voyez le chemin vers votre fond d'ecran actuel -- par exemple `C:\Users\VotreNom\Pictures\plage.jpg`.

Maintenant, ouvrez les **Parametres Windows** (++win+i++) et changez votre fond d'ecran. Revenez dans Regedit, appuyez sur ++f5++ pour rafraichir : la valeur **Wallpaper** a change. Windows vient de "noter" votre nouveau choix ici.

C'est ca, la **base de registre** : l'endroit ou Windows ecrit ses reglages pour s'en souvenir, meme apres un redemarrage.

!!! quote "En resume"
    - Quand vous changez un reglage (fond d'ecran, langue, etc.), Windows le "note" quelque part pour s'en souvenir apres un redemarrage.
    - Cet endroit, c'est la **base de registre**. Vous pouvez le voir vous-meme dans Regedit.

---

## L'analogie du carnet de reglages

Imaginez que votre ordinateur possede un immense carnet de notes. Dedans, il inscrit **tous** ses reglages :

- :material-image: Quel fond d'ecran afficher
- :material-rocket-launch: Quels programmes lancer au demarrage
- :material-keyboard: Quelle est la langue du clavier
- :material-folder: Ou est installe tel logiciel
- :material-desktop-tower: Quel est le nom de votre PC sur le reseau

Ce carnet, c'est la **base de registre** (ou *registre* tout court).

!!! info "En anglais"
    On l'appelle le *Windows Registry*. Vous verrez souvent ce terme dans les tutoriels en ligne.

!!! quote "En resume"
    - La base de registre est un immense carnet ou Windows note **tous** ses reglages : fond d'ecran, programmes au demarrage, langue du clavier, etc.
    - On l'appelle aussi *Windows Registry* en anglais.

---

## Pourquoi ca existe ?

### Le bazar des annees 90

Dans les annees 90, chaque logiciel stockait ses reglages dans son propre petit fichier texte (les fichiers `.ini`).

C'est un peu comme si chaque commercant d'un marche notait ses prix sur sa propre feuille volante, dans un format different. Impossible de s'y retrouver !

Le probleme :

- :material-file-multiple: Des dizaines de fichiers de configuration eparpilles partout
- :material-format-list-bulleted: Aucune organisation commune
- :material-shield-off: Pas de protection : n'importe qui pouvait modifier n'importe quoi

### La solution de Microsoft

Microsoft a cree la base de registre pour **centraliser** tout ca dans un seul endroit structure, avec des regles de securite.

C'est comme remplacer toutes les feuilles volantes par **un grand classeur** bien organise, avec une serrure.

```mermaid
graph LR
    A["Avant : fichiers .ini<br>éparpillés partout"] -->|"Microsoft crée<br>le registre"| B["Après : un classeur<br>central et structuré"]
    style A fill:#ff6b6b,color:#fff
    style B fill:#51cf66,color:#fff
```

!!! quote "En resume"
    - Dans les annees 90, chaque logiciel stockait ses reglages dans ses propres fichiers `.ini`, sans organisation commune.
    - Microsoft a cree le registre pour **centraliser** tous les parametres dans un seul endroit structure et securise.

---

## Est-ce que c'est dangereux d'y toucher ?

Soyons honnetes : **oui, mais pas plus que de toucher au tableau electrique de votre maison**.

Si vous savez quel fusible actionner, tout ira bien. Si vous appuyez sur tous les boutons au hasard... vous risquez de couper le courant partout.

!!! tip "Bonne nouvelle"
    Contrairement a un tableau electrique, il est tres facile de **faire une sauvegarde** du registre avant d'y toucher. Si quelque chose ne va pas, il suffit de restaurer la sauvegarde. On verra comment au chapitre 5.

!!! danger "La regle d'or"
    Ne modifiez **jamais** quelque chose dans le registre si vous ne comprenez pas ce que ca fait. Toujours sauvegarder avant.

!!! quote "En resume"
    - Modifier le registre n'est pas dangereux si on **sauvegarde avant** et qu'on comprend ce qu'on fait.
    - En cas d'erreur, il suffit de restaurer la sauvegarde (on verra comment au chapitre 5).

---

## Ce que la base de registre contient

| Categorie | Exemples |
|-----------|----------|
| Reglages de Windows | Apparence, comportement, reseau, securite |
| Configuration des logiciels | Preferences de chaque application installee |
| Associations de fichiers | Quel programme ouvre les `.pdf`, les `.jpg`, etc. |
| Comptes utilisateurs | Chaque utilisateur et ses preferences |
| Services et pilotes | Les composants materiels et services systeme |

!!! quote "En resume"
    - Le registre contient les reglages de Windows, la configuration des logiciels, les associations de fichiers, les comptes utilisateurs et les services/pilotes.

## Ce qu'elle ne contient **pas**

!!! warning "Idees recues"
    Le registre ne contient **aucun** de vos fichiers personnels.

| Ce qui n'est PAS dedans | Ou c'est en realite |
|------------------------|---------------------|
| Vos documents, photos, videos | Dans vos dossiers personnels |
| Les logiciels eux-memes (les `.exe`) | Sur le disque dur (ex. `C:\Program Files`) |
| Vos mots de passe du navigateur | Dans le stockage securise du navigateur |

Pour reprendre notre analogie : le carnet de reglages note "le fond d'ecran est `plage.jpg`" mais ne contient pas la photo elle-meme.

!!! quote "En resume"
    - Le registre contient les **reglages** de Windows et des logiciels : apparence, associations de fichiers, comptes utilisateurs, services.
    - Il ne contient **pas** vos fichiers personnels (documents, photos, videos) ni les programmes eux-memes.

---

## Ou se trouve-t-elle sur le disque ?

La base de registre n'est pas un fichier unique. Elle est repartie en plusieurs fichiers caches :

| Emplacement | Ce qu'il contient |
|-------------|-------------------|
| `C:\Windows\System32\config\` | Les fichiers principaux du systeme |
| `C:\Users\VotreNom\NTUSER.DAT` | Votre configuration personnelle |

!!! info "Fichiers proteges"
    Ces fichiers sont **verrouilles** par Windows en permanence. Vous ne pouvez pas les ouvrir avec le Bloc-notes ou les copier tant que Windows fonctionne. C'est normal et c'est une protection.

!!! quote "En resume"
    - Le registre est reparti en plusieurs fichiers caches : les fichiers systeme dans `C:\Windows\System32\config\` et votre profil dans `C:\Users\VotreNom\NTUSER.DAT`.
    - Ces fichiers sont verrouilles par Windows et ne peuvent pas etre ouverts directement.

---

## Comment on y accede ?

Windows fournit un outil integre appele **Regedit** (l'editeur du registre).

C'est l'equivalent d'un explorateur de fichiers, mais au lieu de naviguer dans vos dossiers, vous naviguez dans les **reglages** de Windows.

Le chapitre suivant vous guide pas a pas dans vos premiers pas avec Regedit.

!!! quote "En resume"
    - On accede au registre avec **Regedit**, un outil integre a Windows qui fonctionne comme un explorateur de fichiers pour les reglages.
    - Le chapitre suivant vous guide pas a pas dans vos premiers pas avec cet outil.

---

!!! quote "En resume"
    | Question | Reponse |
    |----------|---------|
    | C'est quoi ? | La base de donnees centrale des reglages de Windows |
    | Une analogie ? | Un grand carnet ou Windows note tous ses reglages |
    | C'est dangereux ? | Pas si on sauvegarde avant et qu'on sait ce qu'on modifie |
    | Ca contient mes fichiers ? | Non, uniquement des parametres de configuration |
    | Ou c'est stocke ? | Dans des fichiers caches et proteges par Windows |
    | Comment on y accede ? | Avec l'outil **Regedit** integre a Windows |
