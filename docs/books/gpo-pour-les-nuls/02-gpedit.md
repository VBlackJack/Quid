---
description: "Apprenez a ouvrir et naviguer dans l'editeur de strategies de groupe locales (gpedit.msc) : interface, categories, etats des parametres et exercice pratique."
tags:
  - gpo
  - debutant
  - gpedit
---
# L'editeur de strategies locales (gpedit.msc)

!!! abstract "Ce que vous allez apprendre"
    - Comment ouvrir gpedit.msc et ce que vous voyez quand il demarre
    - Comment naviguer dans l'interface a deux panneaux (comme l'Explorateur de fichiers)
    - La difference entre Configuration ordinateur et Configuration utilisateur
    - Les trois etats d'un parametre (Non configure, Active, Desactive) et pourquoi ca compte
    - Comment trouver un parametre precis sans chercher pendant une heure


!!! tip "Si vous ne retenez qu'une chose"
    `gpedit.msc` ne pilote que la stratégie locale du poste : pratique pour apprendre, insuffisant pour administrer un domaine.

---

## Qu'est-ce que gpedit.msc ?

Avant de l'ouvrir, un mot rapide sur ce qu'est cet outil.

**gpedit.msc** est l'editeur de strategies de groupe **locales**. Le mot "locales" est important : cet editeur ne gere que les regles de **votre poste**. Il ne touche pas aux autres ordinateurs du reseau.

C'est un tableau de bord qui vous permet d'activer ou desactiver des centaines de reglages Windows, sans avoir a fouiller dans le registre vous-meme. Vous cochez une case, et Windows applique la regle. Pas besoin de taper des commandes compliquees ou de naviguer dans le registre.

Pensez-y comme le panneau de controle d'un avion. Chaque interrupteur correspond a une fonction precise. Gpedit vous donne acces a ces interrupteurs, avec une description de ce que chacun fait.

!!! info "Locale vs domaine : quelle difference ?"
    En entreprise, les administrateurs utilisent la console GPMC pour creer des GPO de **domaine** qui s'appliquent a des dizaines ou des centaines de machines en meme temps. L'editeur gpedit, lui, ne gere que la machine sur laquelle il est ouvert. C'est une gestion "un par un".

    Pour la suite de ce livre, on commence par gpedit (la gestion locale), et on passera a la console GPMC (la gestion centralisee en entreprise) au [chapitre 4](04-gpmc-premiers-pas.md).


!!! quote "En résumé"
    - Avant de l'ouvrir, un mot rapide sur ce qu'est cet outil.
    - gpedit.msc est l'editeur de strategies de groupe locales.
    - Le mot "locales" est important : cet editeur ne gere que les regles de votre poste.
---

## Ouvrir gpedit.msc

### La methode rapide

1. Appuyez sur ++win+r++ (la touche Windows + la lettre R)
2. Tapez `gpedit.msc`
3. Appuyez sur ++enter++

C'est tout. Pas de mot de passe a saisir, pas de confirmation administrateur a valider (contrairement a Regedit). L'editeur s'ouvre directement.

!!! info "Pourquoi pas de confirmation administrateur ?"
    Contrairement a Regedit, gpedit ne demande pas d'autorisation UAC (la fenetre "Voulez-vous autoriser cette application..."). C'est parce que l'editeur de strategies **locales** ne modifie que la configuration de **votre** machine. Il n'a pas besoin d'un niveau d'acces special pour s'ouvrir. En revanche, certains parametres a l'interieur ne prendront effet que si vous etes administrateur du poste.

!!! warning "Ouverture sans UAC != droits d'ecriture"
    L'ouverture de `gpedit.msc` ne demande pas d'elevation, mais l'enregistrement de la plupart des changements en **Configuration ordinateur** exige bien des droits administrateur locaux. Sans ces droits, vous pouvez parcourir l'editeur, pas appliquer toutes les modifications.

### Via la recherche Windows

1. Cliquez sur la barre de recherche dans la barre des taches (ou appuyez sur ++win+s++)
2. Tapez `gpedit`
3. Cliquez sur **Modifier la strategie de groupe**

!!! tip "Raccourci du quotidien"
    ++win+r++ puis `gpedit.msc` est la methode la plus rapide. Si vous travaillez souvent avec les GPO, ce raccourci deviendra un reflexe. Pas besoin de chercher dans les menus.

### Ce que vous voyez a l'ouverture

Une fenetre s'affiche avec le titre **Editeur de strategie de groupe locale**. Elle ressemble a ceci :

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Editeur de strategie de groupe locale                                   │
│  Fichier   Action   Affichage   ?                                        │
├───────────────────────────────┬──────────────────────────────────────────┤
│  ▼ Strategie Ordinateur local │  Selectionnez un element pour afficher  │
│    ▶ Configuration ordinateur │  sa description.                        │
│    ▶ Configuration utilisateur│                                         │
│                               │                                         │
│                               │                                         │
│                               │                                         │
│                               │                                         │
└───────────────────────────────┴──────────────────────────────────────────┘
```

Deux branches dans le panneau de gauche. Un panneau vide a droite. C'est normal : il faut d'abord cliquer sur quelque chose a gauche pour que le contenu apparaisse a droite.

Ne vous laissez pas impressionner par l'aspect "vieux Windows" de l'interface. L'outil utilise la console MMC (Microsoft Management Console), une technologie qui date un peu visuellement mais qui est extremement stable et fiable.

!!! info "MMC, c'est quoi ?"
    MMC (Microsoft Management Console) est le "cadre" dans lequel s'affichent de nombreux outils d'administration de Windows : gestion des disques, gestion des services, gestion des utilisateurs... et bien sur gpedit. C'est pour ca que tous ces outils ont le meme look "annees 2000". Le contenu change, mais le cadre reste le meme.

!!! warning "L'editeur ne s'ouvre pas ?"
    Si rien ne se passe ou si vous obtenez un message d'erreur, deux causes possibles :

    - Vous etes sur une edition **Windows Home** (on en parle plus loin dans ce chapitre)
    - Vous avez fait une faute de frappe : verifiez bien `gpedit.msc` (pas `gpoedit`, pas `gpedit.exe`)

!!! quote "En resume"
    - gpedit.msc est l'editeur de strategies de groupe **locales** : il ne gere que votre poste.
    - Pour l'ouvrir : ++win+r++ puis tapez `gpedit.msc` et appuyez sur ++enter++.
    - Aucune autorisation administrateur n'est demandee a l'ouverture, mais beaucoup de changements exigent quand meme des droits admin pour etre enregistres.
    - Deux branches principales apparaissent : **Configuration ordinateur** et **Configuration utilisateur**.

---

## L'interface : deux colonnes, un arbre

### L'analogie de l'Explorateur de fichiers

Vous connaissez l'Explorateur de fichiers de Windows ? Le panneau de gauche affiche vos dossiers, le panneau de droite affiche leur contenu.

L'editeur gpedit fonctionne **exactement** de la meme maniere. Sauf qu'au lieu de dossiers et fichiers, vous naviguez dans des **categories** et des **parametres de configuration**.

| | Explorateur de fichiers | gpedit.msc |
|---|:-----------------------:|:----------:|
| Structure | :material-folder: Dossiers | :material-folder-cog: Categories |
| Contenu | :material-file-document: Fichiers | :material-cog: Parametres (regles) |
| Ouvrir | Double-cliquer ouvre le fichier | Double-cliquer ouvre le parametre |
| Modifier | Modifier un fichier | Modifier une regle Windows |
| Panneau gauche | Arborescence de dossiers | Arborescence de categories |
| Panneau droit | Liste de fichiers | Liste de parametres |

Si vous savez naviguer dans l'Explorateur de fichiers, vous savez deja naviguer dans gpedit. Le reflexe est exactement le meme : cliquer a gauche pour naviguer, regarder a droite pour voir le contenu.

### Le panneau de gauche : l'arborescence

C'est votre **plan de navigation**. Cliquez sur les fleches `▶` pour deployer les categories, exactement comme vous ouvrez des sous-dossiers.

L'arborescence forme un arbre logique. Chaque branche se subdivise en sous-branches, qui se subdivisent encore.

Pensez a un arbre genealogique : le tronc se divise en deux grosses branches (Configuration ordinateur et Configuration utilisateur), qui se divisent en branches moyennes (Modeles d'administration, Parametres Windows...), qui elles-memes portent des dizaines de petites branches (Bureau, Systeme, Reseau...).

| Action au clavier | Resultat |
|-------------------|----------|
| ++arrow-right++ | Deployer la branche selectionnee |
| ++arrow-left++ | Replier la branche selectionnee |
| ++arrow-down++ | Descendre d'un element |
| ++arrow-up++ | Monter d'un element |
| ++enter++ | Ouvrir le parametre selectionne dans le panneau droit |

!!! info "Combien de parametres ?"
    L'editeur gpedit contient **plusieurs milliers** de parametres, organises dans des centaines de categories. Sur une installation Windows 11 standard, vous pouvez trouver plus de 3 000 parametres dans les Modeles d'administration seuls. Pas de panique : vous n'en utiliserez jamais qu'une petite fraction. L'important, c'est de savoir **ou chercher**.

### Le panneau de droite : le contenu

Quand vous cliquez sur une categorie a gauche, le panneau de droite affiche la **liste des parametres** de cette categorie. Chaque ligne montre :

| Colonne | Ce qu'elle indique |
|---------|-------------------|
| **Parametre** | Le nom du reglage |
| **Etat** | Non configure, Active ou Desactive |
| **Commentaire** | Une description optionnelle (souvent vide) |

Pour ouvrir un parametre et voir ce qu'il fait, il suffit de **double-cliquer** dessus.

### La fenetre de configuration d'un parametre

Quand vous double-cliquez sur un parametre, une fenetre de configuration s'ouvre. Elle contient toujours les memes elements :

| Element | Ou le trouver | A quoi ca sert |
|---------|---------------|----------------|
| :material-radiobox-marked: Boutons radio (3 etats) | En haut de la fenetre | Choisir l'etat du parametre (Non configure, Active, Desactive) |
| :material-text-box: Options supplementaires | Au milieu (pas toujours present) | Configurer des details quand le parametre est "Active" (ex. choisir une valeur, un chemin, etc.) |
| :material-information: Description / Aide | En bas ou dans un onglet separe | Expliquer en detail ce que fait le parametre |
| :material-check: Boutons OK / Annuler | En bas | Appliquer ou annuler les modifications |

Certains parametres n'ont que les trois boutons radio. D'autres, quand ils sont mis sur "Active", font apparaitre des options supplementaires (un champ texte, une liste deroulante, des cases a cocher...).

!!! tip "L'onglet Aide"
    Quand vous double-cliquez sur un parametre, une fenetre s'ouvre. Ne regardez pas que les boutons radio en haut ! En bas, il y a un **onglet Aide** (ou un panneau "Pris en charge sur" et "Description") qui explique en detail ce que fait le parametre, quelles versions de Windows sont compatibles, et parfois meme le chemin de registre correspondant.

!!! quote "En resume"
    - gpedit fonctionne comme l'Explorateur de fichiers : categories a gauche, parametres a droite.
    - Double-cliquer sur un parametre ouvre sa fenetre de configuration.
    - L'onglet **Aide** ou la **Description** dans chaque parametre explique ce qu'il fait.

---

## Configuration ordinateur vs Configuration utilisateur

### Les deux grandes branches

En haut de l'arborescence, vous avez vu deux branches. C'est la premiere decision a prendre quand vous cherchez un parametre : est-ce que je veux agir sur **la machine** ou sur **la personne** ?

| Branche | Ce qu'elle controle | Analogie |
|---------|--------------------|---------:|
| :material-desktop-tower: **Configuration ordinateur** | Les reglages de la **machine**, quel que soit l'utilisateur connecte | Le reglement interieur d'un batiment : il s'applique a tous ceux qui entrent |
| :material-account: **Configuration utilisateur** | Les reglages qui suivent **la personne** connectee | Les preferences personnelles d'un employe : il les retrouve quel que soit le bureau ou il s'installe |

### Des exemples concrets

Voici comment savoir dans quelle branche chercher :

| Je veux... | Branche | Pourquoi |
|------------|---------|----------|
| Desactiver les cles USB sur le PC | :material-desktop-tower: Configuration ordinateur | Ca concerne le materiel de la machine, pas un utilisateur precis |
| Empecher l'acces au Panneau de configuration | :material-account: Configuration utilisateur | On veut restreindre ce que **la personne** peut faire |
| Configurer Windows Update | :material-desktop-tower: Configuration ordinateur | Les mises a jour concernent la machine entiere |
| Masquer certaines icones du bureau | :material-account: Configuration utilisateur | C'est l'apparence du bureau de **l'utilisateur** qui change |
| Definir une politique de mot de passe | :material-desktop-tower: Configuration ordinateur | La securite s'applique a toute la machine |
| Rediriger le dossier Documents | :material-account: Configuration utilisateur | Chaque utilisateur a ses propres documents |

!!! warning "Quand le meme parametre existe aux deux endroits"
    Certains parametres existent dans les **deux** branches. Dans ce cas, la **Configuration ordinateur** l'emporte generalement. C'est logique : le reglement du batiment est prioritaire sur les preferences personnelles.

### L'analogie de l'hotel

Pour bien ancrer la difference, pensez a un hotel :

- :material-desktop-tower: **Configuration ordinateur** = les regles de l'hotel. Le reglement interieur affiche dans le hall s'applique a **tous** les clients, sans exception. Interdiction de fumer dans les chambres, heure limite du petit-dejeuner, piscine fermee apres 22h. Peu importe qui occupe la chambre 12 : les regles sont les memes.

- :material-account: **Configuration utilisateur** = les preferences du client. La temperature de la chambre, les oreillers supplementaires, le reveil programme a 7h. Ces preferences suivent le client : s'il change de chambre, l'hotel s'adapte a ses preferences dans la nouvelle chambre aussi.

### En pratique

Quand vous ne savez pas ou chercher, demandez-vous : **"Est-ce que ce reglage concerne la machine ou la personne ?"**

- Si c'est la machine (materiel, securite, mises a jour, services) → :material-desktop-tower: Configuration ordinateur
- Si c'est la personne (bureau, menus, restrictions d'interface) → :material-account: Configuration utilisateur

!!! tip "L'astuce du doute"
    Si vous hesitez vraiment, commencez par **Configuration ordinateur**. Les parametres les plus critiques (securite, mises a jour, peripheriques) s'y trouvent. Et si le parametre n'y est pas, allez voir dans **Configuration utilisateur**.

!!! quote "En resume"
    - **Configuration ordinateur** : reglages de la machine, appliques quel que soit l'utilisateur connecte. Prioritaire en cas de conflit.
    - **Configuration utilisateur** : reglages lies a la personne connectee, qui la suivent d'un poste a l'autre.
    - Pour savoir ou chercher, posez-vous la question : "machine ou personne ?".

---

## Modeles d'administration

### Ou se trouvent les reglages utiles ?

Sous chaque branche (Configuration ordinateur et Configuration utilisateur), vous trouvez **trois sous-branches** :

| Sous-branche | Ce qu'elle contient | Pour qui |
|--------------|--------------------|---------:|
| :material-cog: Parametres du logiciel | Deploiement de logiciels via MSI | Principalement pour l'entreprise (Active Directory) |
| :material-shield-lock: Parametres Windows | Securite, scripts de connexion, audit | Reglages systeme plus techniques |
| :material-file-document-edit: **Modeles d'administration** | La grande majorite des reglages du quotidien | **Vous !** C'est ici que vous passerez 90 % de votre temps |

**Modeles d'administration**, c'est le tresor de gpedit. C'est la que se trouvent les milliers de parametres que vous pouvez activer ou desactiver pour personnaliser le comportement de Windows et de ses composants.

Pour reprendre notre analogie de l'hotel : les **Parametres du logiciel** et **Parametres Windows** sont les coulisses techniques (la chaufferie, le systeme electrique). Les **Modeles d'administration**, c'est le tableau de bord de la reception, avec tous les interrupteurs accessibles pour le personnel.

!!! info "Pourquoi 'Modeles d'administration' ?"
    Ce nom un peu abstrait vient du fait que ces reglages sont definis par des fichiers appeles **ADMX** (des fichiers XML qui decrivent chaque parametre). Chaque fichier ADMX represente un "modele" de configuration. Mais pour l'utilisation quotidienne, retenez juste : c'est l'endroit ou on trouve les reglages.

### Les grandes categories

Quand vous deployez **Modeles d'administration** dans l'arborescence, vous decouvrez plusieurs categories. Voici les plus importantes :

| Categorie | Ce qu'on y trouve | Exemple utile |
|-----------|-------------------|---------------|
| :material-monitor: **Bureau** | Fond d'ecran, icones du bureau, Active Desktop | Forcer un fond d'ecran d'entreprise |
| :material-menu: **Menu Demarrer et barre des taches** | Apparence et comportement du menu Demarrer | Masquer le bouton "Eteindre" |
| :material-cog-outline: **Systeme** | Strategie de groupe elle-meme, scripts, connexion, profils | Empecher l'acces a l'invite de commandes |
| :material-ethernet: **Reseau** | Connexions reseau, pare-feu, DNS, QoS | Interdire la modification des parametres reseau |
| :material-puzzle: **Composants Windows** | Windows Update, Defender, Edge, OneDrive, BitLocker... | Configurer les mises a jour automatiques |
| :material-wrench: **Panneau de configuration** | Affichage du Panneau de configuration, ajout/suppression | Masquer certains elements du Panneau de configuration |

!!! tip "Composants Windows : la categorie fourre-tout"
    **Composants Windows** est de loin la categorie la plus fournie. Elle contient des sous-categories pour pratiquement chaque fonctionnalite de Windows : Windows Update, Windows Defender, OneDrive, Internet Explorer, Edge, BitLocker, Recherche Windows, etc. C'est souvent la qu'il faut aller fouiller.

### Visualiser l'arborescence

Voici comment se presente la structure sous **Configuration ordinateur** :

```
▼ Configuration ordinateur
  ▼ Modeles d'administration
    ▶ Bureau
    ▶ Composants Windows
    ▶ Imprimantes
    ▶ Menu Demarrer et barre des taches
    ▶ Panneau de configuration
    ▶ Reseau
    ▶ Systeme
    ▶ Tous les parametres
```

La structure sous **Configuration utilisateur** est quasi identique, avec quelques differences mineures (certaines categories n'existent que dans l'une ou l'autre branche).

!!! tip "Ne pas confondre les deux arborescences"
    Attention : le fait que les categories portent les memes noms sous les deux branches peut preter a confusion. Le parametre "Bureau > Fond d'ecran" sous **Configuration ordinateur** et sous **Configuration utilisateur** sont **deux parametres differents**, meme s'ils portent un nom similaire. Verifiez toujours que vous etes dans la bonne branche (regardez le chemin en haut du panneau droit).

!!! info "Le dossier 'Tous les parametres'"
    Tout en bas de la liste, vous trouverez un dossier appele **Tous les parametres**. Il affiche **tous** les parametres de la branche dans une seule liste, sans les classer par categorie. Utile quand vous savez ce que vous cherchez et que vous voulez utiliser le tri ou le filtre.

!!! quote "En resume"
    - **Modeles d'administration** est la sous-branche ou vous passerez 90 % de votre temps. C'est la que vivent les reglages du quotidien.
    - Les categories principales sont : Bureau, Systeme, Reseau, Composants Windows, Panneau de configuration et Menu Demarrer.
    - **Composants Windows** est la categorie la plus fournie, avec des sous-categories pour chaque fonctionnalite (Windows Update, Defender, Edge, etc.).

---

## Les trois etats d'un parametre

### L'analogie de l'interrupteur a trois positions

C'est le concept le plus important de ce chapitre. Si vous ne devez retenir qu'une seule chose de cette page, c'est celle-ci.

Chaque parametre dans gpedit a exactement **trois** etats possibles. Pas deux. Trois.

Imaginez un interrupteur special avec trois positions :

- :material-circle-outline: **Position neutre** (au milieu) : l'interrupteur ne fait rien, c'est comme s'il n'existait pas
- :material-toggle-switch: **Position ON** : la regle est appliquee
- :material-toggle-switch-off: **Position OFF** : la regle est explicitement desactivee

Ces trois positions correspondent aux trois etats de gpedit :

| Etat | Icone dans gpedit | Ce que ca fait | Analogie |
|------|:-----------------:|----------------|----------|
| **Non configure** | :material-circle-outline: | Le parametre n'est pas touche par la GPO. Windows utilise son comportement par defaut. | L'interrupteur est en position neutre : on ne dit rien |
| **Active** | :material-check-circle: | La regle est **en vigueur**. Le parametre est force a la valeur choisie. | L'interrupteur est sur ON : la regle s'applique |
| **Desactive** | :material-close-circle: | La regle est **explicitement eteinte**. Le parametre est force a etre inactif. | L'interrupteur est sur OFF : on dit explicitement "non" |

### Pourquoi trois etats et pas deux ?

Vous vous demandez peut-etre : pourquoi pas un simple interrupteur ON/OFF, comme partout ailleurs ?

La raison est **l'heritage**. En entreprise, plusieurs GPO peuvent s'empiler les unes sur les autres (on verra ca en detail au chapitre 6). Il faut pouvoir dire "je ne me prononce pas sur ce parametre" pour laisser une GPO de niveau superieur (ou inferieur) decider.

Avec un simple ON/OFF, chaque GPO serait obligee de prendre position sur **chaque** parametre. Avec plusieurs milliers de parametres, ca deviendrait ingerable. Avec trois etats, une GPO peut choisir de ne rien dire sur un sujet et laisser une autre GPO prendre la decision. C'est beaucoup plus souple.

!!! info "Un detail qui compte"
    Quand vous ouvrez gpedit pour la premiere fois sur une machine fraiche, **tous** les parametres sont en "Non configure". C'est normal. Ca veut dire que la strategie locale ne force rien : Windows utilise ses comportements par defaut partout.

### La confusion classique : Non configure vs Desactive

C'est LE piege numero un pour les debutants. "Non configure" et "Desactive" semblent vouloir dire la meme chose, mais ce n'est **pas du tout** le cas.

!!! danger "Ce n'est pas la meme chose !"
    **Non configure** = "Je ne me prononce pas". Le parametre garde son comportement par defaut.

    **Desactive** = "Je dis explicitement NON". Le parametre est force a etre inactif, meme si une autre GPO essaie de l'activer.

Prenons un exemple concret pour bien comprendre :

| Parametre | Non configure | Active | Desactive |
|-----------|:------------:|:------:|:---------:|
| "Empecher l'acces au Panneau de configuration" | L'utilisateur peut acceder au Panneau de configuration (comportement par defaut) | L'acces est **bloque** | L'acces est **explicitement autorise**, meme si une autre GPO tente de le bloquer |

!!! example "Pas a pas : voir les trois etats"
    1. Ouvrez gpedit (++win+r++ puis `gpedit.msc` puis ++enter++)
    2. Naviguez vers : **Configuration utilisateur** → **Modeles d'administration** → **Panneau de configuration**
    3. Double-cliquez sur **Interdire l'acces au Panneau de configuration et aux parametres du PC**
    4. Vous voyez les trois boutons radio : **Non configure**, **Active**, **Desactive**
    5. Lisez la description en bas de la fenetre pour comprendre ce que fait ce parametre
    6. Cliquez sur **Annuler** (on ne modifie rien pour l'instant !)

### Un deuxieme exemple pour bien comprendre

Imaginons le parametre **"Desactiver Windows Update"** :

| Etat | Ce qui se passe concretement |
|------|------------------------------|
| **Non configure** | Windows Update fonctionne normalement, selon ses reglages par defaut. La GPO ne dit rien a ce sujet. |
| **Active** | Windows Update est **desactive**. Les mises a jour ne se telechargent plus automatiquement. |
| **Desactive** | Windows Update est **explicitement maintenu actif**. Meme si quelqu'un essaie de le couper par un autre moyen, cette GPO le "protege". |

Vous voyez la nuance ? "Non configure" et "Desactive" donnent ici le meme resultat visible (Windows Update fonctionne), mais la **raison** est differente. Dans un cas, la GPO est silencieuse. Dans l'autre, elle crie "je veux que ca reste actif !".

### Quand utiliser Desactive plutot que Non configure ?

Dans la plupart des cas, **Non configure** suffit. Vous utilisez **Desactive** quand vous voulez **forcer** un comportement, meme si une autre GPO (par exemple une GPO du domaine en entreprise) essaie d'imposer autre chose.

C'est un concept qui deviendra plus clair au chapitre 6, quand on parlera de l'heritage des GPO en entreprise.

Pour reprendre l'analogie de l'hotel :

- **Non configure** = le client n'a rien demande pour la temperature de sa chambre. L'hotel applique la temperature par defaut.
- **Active** = le client a demande 18 degres. L'hotel applique 18 degres.
- **Desactive** = le client a explicitement dit "je ne veux pas de chauffage, meme si la politique de l'hotel est de chauffer toutes les chambres a 22 degres". L'hotel respecte sa demande.

!!! tip "La regle simple"
    - Vous ne voulez rien changer → laissez sur **Non configure**
    - Vous voulez activer une regle → mettez sur **Active**
    - Vous voulez empecher qu'une regle soit activee (par une autre GPO, par exemple) → mettez sur **Desactive**

!!! quote "En resume"
    - Chaque parametre a **trois** etats : Non configure, Active, Desactive.
    - **Non configure** = la GPO ne touche pas a ce parametre (comportement par defaut).
    - **Active** = la regle est en vigueur (le parametre est force).
    - **Desactive** = la regle est explicitement desactivee (force a "non", meme si une autre GPO dit le contraire).
    - Ne confondez jamais "Non configure" et "Desactive" : le premier est neutre, le second est actif.

---

## Naviguer dans les categories

### Les categories principales en detail

Maintenant que vous comprenez la structure generale, voici un tour d'horizon des categories que vous croiserez le plus souvent sous **Modeles d'administration**.

| Categorie | Ce qu'on y trouve | Exemple de parametre utile |
|-----------|-------------------|----------------------------|
| :material-monitor: **Bureau** | Fond d'ecran, icones, Active Desktop, economiseur d'ecran | Forcer un fond d'ecran specifique |
| :material-menu: **Menu Demarrer et barre des taches** | Elements visibles dans le menu Demarrer, comportement de la barre des taches | Supprimer les applications recemment ajoutees du menu Demarrer |
| :material-cog-outline: **Systeme** | Scripts de demarrage/arret, connexion, strategie de groupe, gestion de l'alimentation, profils utilisateurs | Empecher l'acces a l'invite de commandes |
| :material-ethernet: **Reseau** | Connexions reseau, pare-feu, DNS client, fichiers hors connexion | Interdire la modification des parametres de connexion |
| :material-puzzle: **Composants Windows** | Sous-categories pour chaque fonctionnalite : Windows Update, Defender Antivirus, Chiffrement BitLocker, Edge, OneDrive, Recherche, etc. | Configurer les mises a jour automatiques |
| :material-wrench: **Panneau de configuration** | Visibilite des elements du Panneau de configuration, programmes par defaut | Masquer des elements specifiques du Panneau de configuration |
| :material-printer: **Imprimantes** | Recherche d'imprimantes, publication dans Active Directory | Empecher l'ajout d'imprimantes |

### Difference entre les deux branches

Les categories ne sont pas exactement les memes sous Configuration ordinateur et Configuration utilisateur. Voici les differences principales :

| Categorie | Configuration ordinateur | Configuration utilisateur |
|-----------|:------------------------:|:------------------------:|
| Bureau | :material-check: Oui | :material-check: Oui |
| Systeme | :material-check: Oui | :material-check: Oui |
| Reseau | :material-check: Oui | :material-check: Oui |
| Composants Windows | :material-check: Oui | :material-check: Oui |
| Menu Demarrer et barre des taches | :material-check: Oui | :material-check: Oui |
| Panneau de configuration | :material-check: Oui | :material-check: Oui |
| Imprimantes | :material-check: Oui | :material-close: Non |
| Dossiers partages | :material-close: Non | :material-check: Oui |

Les differences sont mineures, mais elles existent. Ne soyez pas surpris si une categorie que vous voyez sous Configuration ordinateur n'apparait pas sous Configuration utilisateur (et inversement).

Le nombre de sous-categories peut aussi varier selon les logiciels installes et les fichiers ADMX presents sur votre systeme. Par exemple, si vous installez les modeles ADMX pour Microsoft Office, de nouvelles categories apparaitront automatiquement.

### Zoom sur "Composants Windows"

La categorie **Composants Windows** merite qu'on s'y attarde, car c'est la plus dense. Voici ses sous-categories les plus utiles :

| Sous-categorie | Ce qu'elle gere |
|----------------|----------------|
| :material-download: **Windows Update** | Planification des mises a jour, redemarrage automatique, canal de mise a jour |
| :material-shield-check: **Antivirus Windows Defender** | Analyse en temps reel, exclusions, mises a jour des definitions |
| :material-lock: **Chiffrement de lecteur BitLocker** | Chiffrement des disques, methodes d'authentification |
| :material-web: **Microsoft Edge** | Page d'accueil, moteur de recherche, extensions autorisees |
| :material-cloud: **OneDrive** | Synchronisation, fichiers a la demande |
| :material-magnify: **Recherche** | Cortana, indexation, recherche web |
| :material-message-text: **Collecte de donnees et versions d'evaluation** | Telemetrie, donnees de diagnostic |
| :material-remote-desktop: **Services Bureau a distance** | Connexion a distance, sessions RDP |
| :material-apps: **Store** | Microsoft Store, installation d'applications |

### Astuce de navigation

Quand une categorie contient trop de sous-categories, utilisez la **barre de defilement** du panneau gauche. Et n'oubliez pas : cliquer sur la fleche `▶` deploie la categorie sans l'ouvrir dans le panneau de droite. Pour afficher les parametres d'une categorie dans le panneau de droite, il faut **cliquer sur le nom** de la categorie.

!!! warning "La fleche vs le nom"
    C'est une subtilite qui fait perdre du temps a beaucoup de debutants :

    - :material-chevron-right: **Cliquer sur la fleche** `▶` deploie les sous-categories dans le panneau de gauche (comme ouvrir un dossier pour voir ses sous-dossiers)
    - :material-cursor-default-click: **Cliquer sur le nom** de la categorie affiche les parametres dans le panneau de droite (comme afficher le contenu d'un dossier)

    Ce sont deux actions differentes. Pour voir les parametres, cliquez sur le **nom**.

!!! tip "Le raccourci 'Tous les parametres'"
    Si vous connaissez une partie du nom du parametre que vous cherchez, cliquez directement sur le dossier **Tous les parametres** (en bas de la liste des categories). Tous les parametres apparaissent dans une seule liste. Cliquez ensuite sur l'en-tete de colonne **Parametre** pour trier par ordre alphabetique, ce qui facilite la recherche.

!!! quote "En resume"
    - Les categories les plus utiles au quotidien sont : **Systeme**, **Composants Windows**, **Bureau** et **Menu Demarrer**.
    - **Composants Windows** est la categorie la plus fournie avec des sous-categories pour chaque fonctionnalite de Windows (Update, Defender, Edge, BitLocker, etc.).
    - Le dossier **Tous les parametres** affiche tout en une seule liste, pratique pour trier et chercher.

---

## Chercher un parametre

### Le probleme

Avec plusieurs milliers de parametres repartis dans des dizaines de categories, trouver celui que vous cherchez peut ressembler a chercher une aiguille dans une botte de foin.

Heureusement, il existe plusieurs techniques. Aucune n'est parfaite, mais en les combinant, vous trouverez toujours ce que vous cherchez.

### Methode 1 : le filtre integre

Gpedit dispose d'une fonction de **filtrage** accessible depuis le menu.

!!! example "Pas a pas : activer le filtre"
    1. Dans gpedit, cliquez sur le menu **Action** en haut de la fenetre
    2. Cliquez sur **Options de filtre...**
    3. Cochez **Activer le filtre par mot cle**
    4. Tapez le mot cle que vous cherchez (par exemple `fond d'ecran` ou `Windows Update`)
    5. Selectionnez **"Dans le titre du parametre de strategie"** ou **"Dans le texte de l'aide"** selon vos besoins
    6. Cliquez sur **OK**
    7. Seuls les parametres correspondant au filtre seront affiches

Pour desactiver le filtre et retrouver tous les parametres, retournez dans **Action** → **Options de filtre...** et decochez la case.

!!! danger "N'oubliez pas de desactiver le filtre !"
    Une erreur classique : vous activez le filtre, vous trouvez votre parametre, puis vous continuez a naviguer... et vous ne voyez presque plus rien dans les categories. C'est parce que le filtre est toujours actif ! Pensez a le desactiver quand vous avez fini votre recherche.

!!! warning "Le filtre a ses limites"
    Le filtre de gpedit est basique. Il ne fonctionne que sur du texte exact (pas de recherche approximative). Si vous cherchez "ecran de veille" et que le parametre s'appelle "economiseur d'ecran", vous ne le trouverez pas. Essayez plusieurs mots cles differents.

    De plus, le filtre ne cherche que dans la **branche actuellement selectionnee**. Si vous cherchez dans Configuration ordinateur, vous ne verrez pas les resultats de Configuration utilisateur.

### Methode 2 : la vue "Tous les parametres"

Comme mentionne plus haut, le dossier **Tous les parametres** (au bas de l'arborescence de chaque branche, sous Modeles d'administration) affiche **tout** dans une liste unique.

1. Cliquez sur **Tous les parametres** dans le panneau de gauche
2. Dans le panneau de droite, cliquez sur l'en-tete de colonne **Parametre** pour trier par ordre alphabetique
3. Faites defiler jusqu'au parametre souhaite

C'est moins elegant que le filtre, mais ca a l'avantage d'etre simple et de ne rien masquer.

!!! tip "Tri par etat"
    Vous pouvez aussi cliquer sur l'en-tete de colonne **Etat** pour trier par etat. C'est tres pratique pour voir d'un coup d'oeil **tous les parametres que vous avez modifies** (ceux qui ne sont pas "Non configure").

### Methode 3 : chercher sur Internet d'abord

Souvent, la methode la plus efficace est de chercher sur Internet le nom exact du parametre avant d'aller le chercher dans gpedit.

Par exemple, cherchez "gpo desactiver cortana" dans votre moteur de recherche. Les resultats vous donneront le **chemin complet** dans l'arborescence, que vous n'aurez plus qu'a suivre clic par clic.

!!! tip "La bonne requete de recherche"
    Pour des resultats pertinents, utilisez ces formulations :

    - `gpo [ce que vous voulez faire]` → ex. `gpo desactiver cortana`
    - `gpedit [nom du parametre]` → ex. `gpedit fond d'ecran`
    - `group policy [nom en anglais]` → ex. `group policy disable USB storage`

    Les resultats en anglais sont souvent plus complets et plus a jour. Le nom du parametre dans gpedit varie selon la langue de votre Windows, mais le chemin dans l'arborescence reste identique.

### Methode 4 : le chemin de registre

Chaque parametre de gpedit correspond a une **cle de registre**. Quand vous double-cliquez sur un parametre, le panneau d'aide indique souvent le chemin de registre concerne.

L'inverse est vrai aussi : si vous connaissez un chemin de registre depuis un tutoriel, vous pouvez chercher le parametre gpedit correspondant. Cela fait le lien avec le livre [Le Registre pour les Nuls](../registre-pour-les-nuls/index.md).

### Quelle methode choisir ?

| Methode | Rapidite | Precision | Quand l'utiliser |
|---------|:--------:|:---------:|-----------------|
| :material-filter: Filtre integre | Moyenne | Moyenne | Vous connaissez un mot du nom du parametre |
| :material-format-list-bulleted: Tous les parametres | Lente | Haute | Vous voulez parcourir ou trier la liste complete |
| :material-web: Recherche Internet | Rapide | Haute | Vous savez ce que vous voulez faire mais pas le nom du parametre |
| :material-key: Chemin de registre | Variable | Tres haute | Vous venez d'un tutoriel registre et voulez l'equivalent GPO |

Dans la pratique, la plupart des professionnels utilisent la **recherche Internet** en premier, puis naviguent directement vers le bon chemin dans gpedit. C'est la combinaison la plus efficace.

!!! info "GPO et registre : un duo indissociable"
    Sous le capot, la plupart des parametres de gpedit ecrivent tout simplement des valeurs dans la base de registre. L'editeur gpedit est une interface graphique conviviale qui vous evite de manipuler le registre a la main. Le resultat final est le meme.

    Par exemple, activer le parametre "Empecher l'acces au Panneau de configuration" dans gpedit revient exactement a creer la valeur `NoControlPanel = 1` dans la cle de registre `HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer`. Gpedit fait ce travail pour vous, sans que vous ayez a toucher au registre.

!!! quote "En resume"
    - Utilisez le **filtre** (menu Action → Options de filtre) pour chercher un parametre par mot cle.
    - Le dossier **Tous les parametres** permet de trier et parcourir l'integralite des parametres dans une seule liste.
    - Chercher le **nom du parametre sur Internet** est souvent la methode la plus rapide.
    - Chaque parametre gpedit correspond a une cle de registre : l'onglet Aide indique le chemin.

---

## gpedit n'est pas disponible sur Windows Home

### Le probleme

Si vous avez tente d'ouvrir gpedit et que vous avez vu ce message :

```
Windows ne trouve pas 'gpedit.msc'. Verifiez que vous avez
tape le nom correctement, puis reessayez.
```

...c'est que vous utilisez une edition **Home** (Famille) de Windows. Ne paniquez pas : ce n'est pas un probleme de votre machine, c'est une limitation normale de l'edition.

!!! danger "gpedit.msc est absent de Windows Home"
    Les editions Windows **Home** (Windows 10 Famille, Windows 11 Famille) ne contiennent **pas** l'editeur de strategies de groupe. C'est une limitation volontaire de Microsoft pour differencier les editions grand public des editions professionnelles.

### Comment verifier votre edition

!!! example "Pas a pas : verifier votre edition de Windows"
    1. Appuyez sur ++win+r++
    2. Tapez `winver`
    3. Appuyez sur ++enter++
    4. La fenetre affiche votre edition : **Pro**, **Enterprise**, **Education** (gpedit disponible) ou **Home** (gpedit absent)

Voici un recapitulatif des editions et de la disponibilite de gpedit :

| Edition Windows | gpedit.msc disponible | Public cible |
|----------------|:---------------------:|--------------|
| Windows Home (Famille) | :material-close: Non | Grand public |
| Windows Pro | :material-check: Oui | Professionnels et petites entreprises |
| Windows Enterprise | :material-check: Oui | Grandes entreprises |
| Windows Education | :material-check: Oui | Etablissements scolaires |
| Windows Server | :material-check: Oui | Serveurs |

### Les alternatives

Si vous etes sur Windows Home, vous avez trois options :

| Option | Avantage | Inconvenient |
|--------|----------|-------------|
| :material-pencil: **Modifier le registre directement** | Gratuit, toujours disponible, meme resultat | Plus technique, risque d'erreur plus eleve |
| :material-tools: **Utiliser Policy Plus** | Interface identique a gpedit, gratuit, open-source | Outil tiers non officiel, necessite une installation |
| :material-arrow-up-bold: **Passer a Windows Pro** | Acces complet a gpedit et tous les outils d'entreprise | Payant (licence de mise a niveau requise) |

### Policy Plus : une alternative gratuite

**Policy Plus** est un outil open-source qui reproduit l'interface de gpedit sur les editions Home. Il lit et ecrit les memes parametres que le vrai gpedit.

Il se presente sous la forme d'un fichier executable unique, sans installation. Vous le telechargez, vous le lancez, et vous retrouvez une interface tres similaire a celle de gpedit.

!!! warning "Outil non officiel"
    Policy Plus n'est pas un produit Microsoft. Il fonctionne bien dans la plupart des cas, mais il n'est pas garanti d'etre toujours a jour avec les dernieres versions de Windows. Utilisez-le en connaissance de cause.

!!! tip "Quand utiliser Policy Plus ?"
    Si vous etes sur Windows Home et que vous avez besoin de modifier **un ou deux** parametres GPO, la modification directe du registre est plus simple. Si vous avez besoin de modifier **plusieurs** parametres ou de les explorer visuellement, Policy Plus est un meilleur choix.

### La modification directe du registre

Puisque chaque parametre gpedit correspond a une cle de registre, vous pouvez obtenir le meme resultat en modifiant directement le registre avec Regedit. C'est la methode la plus universelle : elle fonctionne sur **toutes** les editions de Windows.

Le livre [Le Registre pour les Nuls](../registre-pour-les-nuls/index.md) vous guide pas a pas dans ces modifications.

!!! info "L'analogie du raccourci et du chemin"
    Gpedit est comme un **raccourci convivial** vers le registre. Quand vous activez un parametre dans gpedit, Windows ecrit la valeur correspondante dans le registre. Modifier le registre directement, c'est prendre le **chemin long** a pied, mais on arrive au meme endroit. Sur Windows Home, le raccourci n'existe pas, mais le chemin, lui, est toujours la.

!!! quote "En resume"
    - gpedit.msc est **absent** de Windows Home (Famille). C'est une limitation de l'edition.
    - Pour verifier votre edition : ++win+r++ puis `winver`.
    - Alternatives : modifier le registre directement, utiliser Policy Plus (gratuit, open-source) ou passer a Windows Pro.
    - Modifier le registre directement donne le meme resultat qu'utiliser gpedit, mais demande plus de prudence.

---

## Exercice pratique : explorer sans toucher

### Objectif

L'objectif de cet exercice est de vous familiariser avec la navigation dans gpedit, de lire la description d'un parametre et d'observer ses trois etats. Vous n'allez **rien modifier**. Juste explorer.

!!! warning "Rappel important"
    Pendant tout cet exercice, vous ne cliquerez **jamais** sur OK. Uniquement sur **Annuler**. L'exploration est sans risque tant que vous ne validez rien.

!!! example "Pas a pas : votre premier voyage dans gpedit"
    **Etape 1 : Ouvrir gpedit**

    Appuyez sur ++win+r++, tapez `gpedit.msc`, appuyez sur ++enter++.

---
    **Etape 2 : Naviguer vers un parametre**

    Dans le panneau de gauche, suivez ce chemin en cliquant sur les fleches :

    1. :material-chevron-right: **Configuration utilisateur**
    2. :material-chevron-right: **Modeles d'administration**
    3. :material-chevron-right: **Systeme**
    4. Cliquez sur le nom **Systeme** (pas juste sur la fleche) pour afficher les parametres dans le panneau de droite

---
    **Etape 3 : Trouver le parametre**

    Dans la liste du panneau de droite, cherchez :

    **"Empecher l'acces a l'invite de commandes"**

---
    **Etape 4 : Lire la description**

    Double-cliquez sur ce parametre. Une fenetre s'ouvre.

    Observez :

    - Les **trois boutons radio** en haut : Non configure, Active, Desactive
    - Le panneau de **description** ou l'onglet **Aide** en bas qui explique ce que fait ce parametre
    - La mention **"Pris en charge sur"** qui indique les versions de Windows compatibles

---
    **Etape 5 : Ne rien changer**

    Cliquez sur **Annuler** pour fermer la fenetre sans rien modifier.

---
    **Etape 6 : Explorer une autre categorie**

    Maintenant, naviguez vers :

    1. :material-chevron-right: **Configuration ordinateur**
    2. :material-chevron-right: **Modeles d'administration**
    3. :material-chevron-right: **Composants Windows**
    4. :material-chevron-right: **Windows Update**

    Parcourez les parametres. Lisez les descriptions de ceux qui vous intriguent. Reperez ceux qui sont "Non configure" (la grande majorite).

---
    **Etape 7 : Observer un parametre avec des options**

    Toujours dans Windows Update, cherchez un parametre qui, quand on le met sur "Active", affiche des options supplementaires (une liste deroulante, un champ de texte, des cases a cocher). Par exemple, cherchez **"Configuration du service Mises a jour automatiques"**.

    Double-cliquez dessus, selectionnez **Active** (juste pour voir les options qui apparaissent, sans cliquer sur OK !). Vous verrez des menus deroulants pour configurer le comportement de Windows Update : jour de la semaine, heure de l'installation, etc.

    Notez comme ces options n'apparaissent **que** quand le parametre est sur "Active". C'est logique : si le parametre n'est pas active, il n'y a rien a configurer.

    Cliquez sur **Annuler** quand vous avez fini d'observer.

Bravo, vous venez de naviguer dans gpedit comme un pro ! :material-party-popper:

### Defi bonus : trouver un parametre par vous-meme

Si vous voulez aller plus loin, essayez de trouver **seul** le parametre suivant :

**"Supprimer l'acces aux fonctionnalites de l'Historique des fichiers"**

Quelques indices :

- C'est un parametre qui concerne un **composant** de Windows
- Il se trouve sous **Configuration ordinateur**
- Regardez dans la categorie **Composants Windows**

!!! tip "Solution"
    Le chemin complet est : **Configuration ordinateur** → **Modeles d'administration** → **Composants Windows** → **Historique des fichiers**. Si vous l'avez trouve sans cette aide, vous maitrisez deja la navigation !

### Ce que vous avez appris en pratiquant

| Vous avez fait... | Ce que ca vous apprend |
|-------------------|----------------------|
| Ouvert gpedit avec ++win+r++ | Le reflexe d'ouverture rapide |
| Navigue dans l'arborescence | La logique des categories et sous-categories |
| Double-clique sur un parametre | Comment ouvrir et lire un parametre |
| Lu la description et l'aide | Ou trouver des informations fiables sur chaque parametre |
| Clique sur Annuler | La discipline de ne rien modifier tant qu'on ne sait pas ce qu'on fait |

!!! quote "En resume"
    - Vous savez maintenant naviguer dans gpedit, trouver un parametre et lire sa description.
    - La discipline la plus importante : **lire avant de modifier**, et cliquer sur Annuler quand on est la juste pour explorer.
    - La navigation dans gpedit, c'est comme la navigation dans l'Explorateur de fichiers : une fois qu'on a compris la logique, on ne l'oublie plus.

---

## Les erreurs courantes du debutant

Avant de passer au chapitre suivant, voici les pieges dans lesquels tombent la plupart des debutants. Les connaitre a l'avance, c'est s'eviter des heures de frustration.

| Erreur | Pourquoi c'est un probleme | Ce qu'il faut faire |
|--------|---------------------------|---------------------|
| Confondre "Non configure" et "Desactive" | Le comportement est different, surtout en entreprise avec plusieurs GPO | Relire la section "Les trois etats" ci-dessus |
| Modifier un parametre sans lire sa description | On risque d'activer quelque chose qu'on ne comprend pas | Toujours lire l'onglet Aide / Description **avant** de cliquer sur OK |
| Chercher dans la mauvaise branche | Un parametre "utilisateur" n'apparait pas dans "ordinateur" et vice-versa | Se demander "machine ou personne ?" avant de naviguer |
| Oublier de cliquer sur le **nom** de la categorie | Cliquer sur la fleche deploie l'arbre mais n'affiche rien a droite | Cliquer sur le texte de la categorie, pas sur la fleche |
| Penser que gpedit existe sur Windows Home | On perd du temps a chercher un outil qui n'est pas installe | Verifier son edition avec `winver` |
| Appliquer un parametre sans comprendre l'impact | On peut verrouiller l'acces a des fonctionnalites essentielles | Lire la description, tester sur un poste non critique |
| Oublier de desactiver le filtre de recherche | On ne voit plus qu'une partie des parametres sans comprendre pourquoi | Verifier dans Action → Options de filtre si le filtre est actif |

!!! danger "La regle d'or de ce chapitre"
    Ne modifiez **rien** tant que vous n'etes pas a l'aise avec la navigation. L'exploration ne presente aucun risque : ouvrir un parametre et cliquer sur Annuler ne change rien du tout. En revanche, la modification, si elle est faite sans comprehension, peut verrouiller des fonctionnalites de votre poste.

    Un exemple classique : activer "Interdire l'acces au Panneau de configuration" sans realiser que ca vous empechera aussi d'acceder aux **Parametres** de Windows. Resultat : vous ne pouvez plus configurer votre WiFi, votre ecran ou vos peripheriques. Le chapitre 3 vous guidera dans vos premieres modifications en toute securite, avec des parametres sans risque.

!!! quote "En resume"
    - Les erreurs les plus frequentes : confondre les etats, chercher dans la mauvaise branche, ne pas lire la description avant de modifier.
    - Explorer gpedit est **totalement** sans risque tant que vous cliquez sur Annuler. Modifier sans comprendre ne l'est pas.
    - En cas de doute, la regle est simple : **ne touchez pas**. Lisez, observez, et revenez quand vous serez pret.

---

## Recapitulatif du chapitre

!!! quote "En resume"

    | Concept | Ce qu'il faut retenir |
    |---------|----------------------|
    | Qu'est-ce que gpedit | L'editeur de strategies de groupe **locales** (votre poste uniquement) |
    | Ouvrir gpedit | ++win+r++ puis `gpedit.msc` puis ++enter++ |
    | L'interface | Deux panneaux : categories a gauche, parametres a droite (comme l'Explorateur de fichiers) |
    | Configuration ordinateur | Reglages de la **machine**, appliques a tous les utilisateurs |
    | Configuration utilisateur | Reglages de la **personne** connectee |
    | Modeles d'administration | La section ou vous passerez 90 % de votre temps |
    | Les trois etats | Non configure (neutre), Active (ON), Desactive (OFF) |
    | Non configure vs Desactive | "Je ne me prononce pas" vs "Je dis explicitement non" |
    | Fenetre de parametre | Boutons radio en haut, options au milieu (si Active), description en bas |
    | Chercher un parametre | Filtre (Action → Options de filtre), dossier Tous les parametres, ou recherche Internet |
    | Windows Home | gpedit absent ; alternatives : registre direct, Policy Plus ou upgrade vers Pro |

---

## La suite

Vous savez maintenant **ouvrir** gpedit, **naviguer** dans ses categories, **lire** la description d'un parametre et comprendre ses **trois etats**. Il est temps de passer a l'action.

:material-arrow-right: Dans le [chapitre 3 : 10 modifications utiles avec gpedit](03-modifications-utiles.md), vous allez appliquer vos premiers reglages concrets -- en toute securite. On y va pas a pas, avec des explications pour chaque parametre.

!!! info "Besoin de reviser ?"
    Si les concepts de ce chapitre ne sont pas encore clairs, relisez les sections qui vous posent probleme avant de continuer. Il n'y a aucune urgence. Mieux vaut bien comprendre la navigation et les trois etats avant de commencer a modifier quoi que ce soit.

!!! tip "Memo a garder sous la main"
    Les trois reflexes a retenir avant de passer au chapitre suivant :

    1. ++win+r++ → `gpedit.msc` → ++enter++ pour ouvrir
    2. "Machine ou personne ?" pour choisir la bonne branche
    3. Toujours lire la description **avant** de modifier un parametre

!!! quote "En résumé"
    - win+r → gpedit.msc → enter pour ouvrir.
    - "Machine ou personne ?" pour choisir la bonne branche.
    - Toujours lire la description avant de modifier un parametre.
    - Il est temps de passer a l'action.
    - On y va pas a pas, avec des explications pour chaque parametre.
