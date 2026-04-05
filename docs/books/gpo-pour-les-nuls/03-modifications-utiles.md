---
description: "Dix modifications pratiques et immediates a realiser avec l'editeur de strategies de groupe locales (gpedit.msc)."
tags:
  - gpo
  - debutant
  - modifications
---
# 10 modifications utiles avec gpedit

!!! abstract "Ce que vous allez apprendre"
    - Comment appliquer 10 modifications concretes avec gpedit.msc
    - Comment verifier que chaque modification a bien pris effet
    - Comment annuler proprement un changement
    - Quelles modifications touchent l'ordinateur et lesquelles touchent l'utilisateur
    - Comment garder une trace de tout ce que vous changez


!!! tip "Si vous ne retenez qu'une chose"
    Avant toute modification, notez le réglage d'origine et validez le résultat avec `gpupdate /force` puis un contrôle visible sur le poste.

---

## Petit rappel : gpedit.msc, c'est quoi ?

Si vous arrivez directement sur ce chapitre, un petit rappel s'impose. **gpedit.msc** est l'editeur de strategies de groupe **locales** de Windows. Il permet de modifier le comportement de Windows sur **un seul PC**, sans avoir besoin d'un serveur Active Directory.

Pour l'ouvrir : ++win+r++ → tapez `gpedit.msc` → ++enter++.

L'interface se divise en deux grandes branches :

- **Configuration ordinateur** : les parametres qui s'appliquent a la **machine entiere**, quel que soit l'utilisateur connecte.
- **Configuration utilisateur** : les parametres qui s'appliquent au **profil de l'utilisateur** qui est connecte.

Dans ce chapitre, on va toucher aux deux branches. Pour chaque modification, on precisera laquelle est concernee.

!!! warning "Pas disponible sur Windows Home"
    L'editeur gpedit.msc est disponible uniquement sur les editions **Pro**, **Enterprise** et **Education** de Windows. Si vous avez Windows Home, ce chapitre ne fonctionnera pas pour vous — vous obtiendrez un message d'erreur au lancement de gpedit.msc.

    Pour verifier votre edition : ++win+r++ → tapez `winver` → ++enter++. La fenetre qui s'ouvre indique votre edition. Le chapitre 2 explique tout ca en detail.


!!! quote "En résumé"
    - Si vous arrivez directement sur ce chapitre, un petit rappel s'impose.
    - gpedit.msc est l'editeur de strategies de groupe locales de Windows.
    - Il permet de modifier le comportement de Windows sur un seul PC, sans avoir besoin d'un serveur Active Directory.
    - Configuration ordinateur : les parametres qui s'appliquent a la machine entiere, quel que soit l'utilisateur connecte.
    - Configuration utilisateur : les parametres qui s'appliquent au profil de l'utilisateur qui est connecte.
---

## Avant de commencer : notez tout !

!!! warning "Gardez un journal de vos modifications"
    Avant de toucher a quoi que ce soit, ouvrez un simple fichier texte. Pour chaque modification, notez :

    - **La date**
    - **Le chemin exact** du parametre
    - **L'etat d'origine** (Non configure, Active, Desactive)
    - **Ce que vous avez change**

    Pourquoi ? Parce que dans trois semaines, quand quelque chose ne fonctionnera plus comme prevu, vous serez **tres content** de retrouver cette liste.

    Pas besoin d'un outil complique. Un fichier `mes-modifications-gpo.txt` sur le Bureau fait parfaitement l'affaire.

Bonne nouvelle : contrairement au registre ou une erreur peut avoir des consequences graves, gpedit offre un filet de securite naturel. Chaque parametre a trois etats possibles, et revenir en arriere est toujours simple.

- **Non configure** : Windows utilise son comportement par defaut. C'est l'etat initial de tous les parametres.
- **Active** : la strategie est appliquee. Windows respecte la regle que vous definissez.
- **Desactive** : la strategie est explicitement desactivee. Ce n'est **pas** la meme chose que "Non configure" — on y reviendra.

Pour annuler une modification, il suffit de remettre le parametre sur **Non configure**. C'est simple et sans risque.

Une derniere chose avant de plonger : apres avoir modifie un parametre, il peut etre necessaire de **fermer et rouvrir la session**, voire de **redemarrer le PC**, pour que le changement prenne effet. La plupart des parametres des "Modeles d'administration" prennent effet immediatement ou apres un `gpupdate /force`. Les parametres de securite peuvent necessiter un redemarrage.

!!! tip "Astuce : forcer l'application immediate"
    Apres chaque modification, vous pouvez forcer Windows a appliquer les strategies immediatement au lieu d'attendre. Ouvrez une invite de commandes (++win+r++ → `cmd`) et tapez :

    ```batch
    gpupdate /force
    ```

    Vous verrez un message comme :

    ```
    Mise à jour de la stratégie de l'ordinateur effectuée.
    Mise à jour de la stratégie de l'utilisateur effectuée.
    ```

    Certains parametres necessitent quand meme une fermeture/reouverture de session ou un redemarrage, mais cette commande accelere les choses pour la majorite des cas.

Maintenant, passons aux choses serieuses. On commence par les modifications les plus simples et on monte progressivement en complexite.

Chaque modification suit le meme format :

1. **Pourquoi** c'est utile (le contexte)
2. **Ou** trouver le parametre (le chemin exact dans gpedit)
3. **Comment** l'appliquer (pas a pas)
4. **Comment** verifier et annuler


!!! quote "En résumé"
    - Pourquoi ?
    - Parce que dans trois semaines, quand quelque chose ne fonctionnera plus comme prevu, vous serez tres content de retrouver cette liste.
    - Pas besoin d'un outil complique.
    - La date.
    - Le chemin exact du parametre.
---

## 1. Interdire l'acces au Panneau de configuration

Vous partagez un PC avec d'autres personnes ? Vous ne voulez pas qu'elles aillent fouiller dans les reglages systeme ? Cette strategie verrouille completement l'acces au Panneau de configuration et a l'application Parametres.

C'est probablement la premiere strategie que les administrateurs activent sur les postes partages. Et pour cause : un utilisateur curieux dans les Parametres peut desactiver l'antivirus, changer la configuration reseau ou modifier les parametres de mise a jour. Autant de choses qu'on prefere eviter.

| Element | Valeur |
|---------|--------|
| **Chemin** | Configuration utilisateur → Modeles d'administration → Panneau de configuration |
| **Parametre** | Interdire l'acces au Panneau de configuration et aux parametres du PC |
| **Action** | Activer |
| **Effet** | Le Panneau de configuration et l'application Parametres deviennent inaccessibles |

!!! example "Essayez vous-meme"
    1. Ouvrir gpedit.msc (++win+r++ → `gpedit.msc`)
    2. Naviguer vers **Configuration utilisateur → Modeles d'administration → Panneau de configuration**
    3. Double-cliquer sur **Interdire l'acces au Panneau de configuration et aux parametres du PC**
    4. Selectionner **Active**
    5. Cliquer sur **OK**

    Pour verifier : essayez d'ouvrir les **Parametres** (++win+i++) ou le **Panneau de configuration** (++win+r++ → `control`). Un message d'erreur devrait apparaitre :

    ```
    Cette operation a ete annulee en raison de restrictions
    en vigueur sur cet ordinateur. Contactez votre
    administrateur systeme.
    ```

    Pour annuler : remettre le parametre sur **Non configure**

!!! info "Ce parametre s'applique a l'utilisateur courant"
    Il se trouve dans la branche **Configuration utilisateur**. Les autres comptes utilisateurs du meme PC ne sont pas affectes. Si vous voulez bloquer tout le monde, il faudra l'appliquer par GPO en domaine (on verra ca dans les chapitres suivants).

Notez que ce parametre bloque aussi bien l'ancien **Panneau de configuration** (l'interface classique avec les icones) que la nouvelle application **Parametres** (l'interface moderne de Windows 10/11). Un seul parametre, deux interfaces bloquees.

!!! quote "En resume"
    - Ce parametre bloque l'acces au Panneau de configuration **et** a l'application Parametres.
    - Il s'applique uniquement a l'utilisateur courant, pas aux autres comptes du PC.

Premiere modification terminee. Vous voyez le principe ? On navigue, on active, on verifie. C'est toujours le meme schema. Les 9 suivantes fonctionnent exactement pareil.

!!! tip "Mise a jour immediate"
    Ce parametre prend effet immediatement apres avoir clique sur OK. Pas besoin de redemarrer le PC ni de fermer la session. Vous pouvez tester tout de suite.

---

## 2. Desactiver le telechargement automatique de pilotes

Windows a une facheuse habitude : il adore telecharger des pilotes tout seul via Windows Update. Parfois, c'est formidable — vous branchez un nouveau peripherique, et hop, ca fonctionne en quelques secondes sans rien faire. Parfois, c'est un cauchemar — Windows remplace votre pilote NVIDIA optimise par un pilote generique Microsoft, et votre ecran passe en resolution 800x600. Ou votre imprimante de bureau cesse de fonctionner du jour au lendemain.

Si vous avez deja vecu le moment ou votre ecran passe en 800x600 apres une mise a jour, ou celui ou votre webcam a cesse de fonctionner juste avant une reunion importante, vous comprenez l'interet de cette strategie.

| Element | Valeur |
|---------|--------|
| **Chemin** | Configuration ordinateur → Modeles d'administration → Systeme → Installation de peripheriques |
| **Parametre** | Empecher la recuperation de metadonnees de peripheriques sur Internet |
| **Action** | Activer |
| **Effet** | Windows ne telecharge plus automatiquement les pilotes et metadonnees depuis Internet |

!!! example "Essayez vous-meme"
    1. Ouvrir gpedit.msc (++win+r++ → `gpedit.msc`)
    2. Naviguer vers **Configuration ordinateur → Modeles d'administration → Systeme → Installation de peripheriques**
    3. Double-cliquer sur **Empecher la recuperation de metadonnees de peripheriques sur Internet**
    4. Selectionner **Active**
    5. Cliquer sur **OK**

    Pour verifier : branchez un peripherique USB inconnu (une webcam, une cle USB Wi-Fi, etc.). Windows ne devrait plus chercher de pilote en ligne automatiquement. Vous verrez un message dans le Gestionnaire de peripheriques indiquant que le peripherique n'a pas pu etre configure automatiquement.

    Pour annuler : remettre le parametre sur **Non configure**

!!! warning "Attention aux consequences"
    En activant cette strategie, vous devrez installer les pilotes **manuellement** pour les nouveaux peripheriques. Gardez les pilotes de vos imprimantes, cles USB Wi-Fi et autres peripheriques a portee de main (sur une cle USB, par exemple).

C'est un parametre de la branche **Configuration ordinateur**, ce qui signifie qu'il s'applique a **tous les utilisateurs** de la machine. Peu importe qui est connecte, Windows ne telechargera plus de pilotes tout seul.

!!! tip "Dans quel cas activer ce parametre ?"
    Ce parametre est particulierement utile dans ces situations :

    - Vous avez une **carte graphique** avec des pilotes proprietaires (NVIDIA, AMD) et vous ne voulez pas que Windows installe un pilote generique par-dessus
    - Vous gerez un **parc informatique** et vous voulez controler precisement les versions de pilotes deployees
    - Vous avez des **peripheriques industriels** ou specialises dont les pilotes generiques ne fonctionnent pas correctement

    En revanche, si vous etes un utilisateur lambda qui branche simplement une souris ou un clavier, laissez ce parametre tranquille — le telechargement automatique vous simplifie la vie.

!!! quote "En resume"
    - Windows ne telecharge plus de pilotes automatiquement depuis Internet.
    - Pensez a garder vos pilotes a disposition pour les installations manuelles.

Les deux premieres modifications etaient des blocages "invisibles" — on empeche quelque chose de se produire. La suivante est beaucoup plus visuelle : on va changer ce que l'utilisateur voit sur son ecran.

---

## 3. Forcer un fond d'ecran

Besoin d'imposer un fond d'ecran precis ? Les cas d'usage sont nombreux :

- En **entreprise**, pour afficher le logo de la societe ou un rappel de securite
- En **famille**, pour eviter les fonds d'ecran inappropries sur le PC du salon
- Sur un **poste de demonstration**, pour maintenir une apparence professionnelle
- Dans un **laboratoire informatique**, pour identifier visuellement les postes (fond rouge pour les postes admin, bleu pour les postes etudiants, etc.)

Ce parametre est un grand classique de l'administration Windows. Il impose une image et empeche l'utilisateur de la changer. L'option de personnalisation du fond d'ecran dans les Parametres sera completement grisee.

| Element | Valeur |
|---------|--------|
| **Chemin** | Configuration utilisateur → Modeles d'administration → Bureau → Bureau |
| **Parametre** | Papier peint du Bureau |
| **Action** | Activer |
| **Effet** | Impose l'image specifiee comme fond d'ecran, sans possibilite de le changer |

!!! example "Essayez vous-meme"
    1. Ouvrir gpedit.msc (++win+r++ → `gpedit.msc`)
    2. Naviguer vers **Configuration utilisateur → Modeles d'administration → Bureau → Bureau**
    3. Double-cliquer sur **Papier peint du Bureau**
    4. Selectionner **Active**
    5. Dans le champ **Nom du papier peint**, saisir le chemin complet de l'image, par exemple :
        ```
        C:\Windows\Web\Wallpaper\Windows\img0.jpg
        ```
    6. Choisir le **Style du papier peint** :
        - **Remplir** : l'image remplit l'ecran (peut etre rognee)
        - **Ajuster** : l'image est redimensionnee pour tenir dans l'ecran (barres noires possibles)
        - **Etirer** : l'image est etiree pour remplir l'ecran (peut etre deformee)
        - **Mosaique** : l'image est repetee en mosaique
        - **Centrer** : l'image est centree sans redimensionnement
    7. Cliquer sur **OK**

    Pour verifier : faites un clic droit sur le Bureau → **Personnaliser**. L'option de fond d'ecran devrait etre grisee et l'image imposee affichee.

    Pour annuler : remettre le parametre sur **Non configure**

!!! tip "Formats d'image acceptes"
    Windows accepte les fichiers `.jpg`, `.jpeg`, `.bmp` et `.png`. Utilisez un chemin **local** (pas un chemin reseau) pour eviter les problemes au demarrage si le reseau n'est pas encore disponible.

    Pour les ecrans haute resolution (Full HD et au-dela), utilisez une image d'au moins **1920x1080 pixels**. Pour les ecrans 4K, visez **3840x2160 pixels**. Sinon, l'image sera floue ou pixelisee, ce qui donne un rendu peu professionnel.

Attention au double dossier **Bureau → Bureau** dans le chemin. Ce n'est pas une erreur, c'est vraiment la structure de l'arborescence dans gpedit. Le premier "Bureau" est la categorie, le second contient les parametres specifiques au Bureau Windows.

!!! info "Et si l'image n'existe pas ?"
    Si le chemin pointe vers un fichier qui n'existe pas (faute de frappe, fichier supprime, disque inaccessible), Windows affichera simplement un fond d'ecran **noir uni**. Ce n'est pas un plantage, c'est le comportement normal. Corrigez le chemin dans gpedit pour retrouver votre image.

!!! quote "En resume"
    - Le parametre impose un fond d'ecran precis et empeche l'utilisateur de le changer.
    - Utilisez un chemin local vers un fichier image `.jpg` ou `.png` d'au moins 1920x1080.

Vous remarquerez que ce parametre, contrairement a la plupart des autres, demande de **saisir des informations supplementaires** (le chemin de l'image et le style). C'est le cas de nombreux parametres dans gpedit : certains sont de simples interrupteurs (on/off), d'autres offrent des options de configuration plus detaillees.

---

## 4. Masquer les lecteurs dans l'Explorateur

Vous avez un disque de sauvegarde que les utilisateurs n'ont pas besoin de voir ? Ou un lecteur reseau que vous voulez garder discret ? Cette strategie masque les lecteurs dans le Poste de travail.

C'est particulierement utile dans plusieurs scenarios :

- Sur les **postes partages** ou les **bornes en libre-service** : les utilisateurs voient uniquement le lecteur C: et rien d'autre
- Sur les PC avec un **disque de sauvegarde** : plus de risque qu'un utilisateur supprime accidentellement des fichiers sur votre disque D:
- En **entreprise** avec des lecteurs reseau : masquer les lecteurs qui ne sont pas pertinents pour l'utilisateur simplifie l'interface

| Element | Valeur |
|---------|--------|
| **Chemin** | Configuration utilisateur → Modeles d'administration → Composants Windows → Explorateur de fichiers |
| **Parametre** | Masquer ces lecteurs specifies dans le Poste de travail |
| **Action** | Activer |
| **Effet** | Les lecteurs selectionnes disparaissent de l'Explorateur de fichiers |

!!! example "Essayez vous-meme"
    1. Ouvrir gpedit.msc (++win+r++ → `gpedit.msc`)
    2. Naviguer vers **Configuration utilisateur → Modeles d'administration → Composants Windows → Explorateur de fichiers**
    3. Double-cliquer sur **Masquer ces lecteurs specifies dans le Poste de travail**
    4. Selectionner **Active**
    5. Dans le menu deroulant, choisir la combinaison de lecteurs a masquer. Les options proposees sont :
        - Limiter les lecteurs A et B uniquement
        - Limiter les lecteurs C et D uniquement
        - Limiter les lecteurs A, B, C et D uniquement
        - Limiter tous les lecteurs
        - Ne pas limiter les lecteurs
    6. Cliquer sur **OK**

    Pour verifier : ouvrez l'Explorateur de fichiers (++win+e++) et verifiez que les lecteurs selectionnes ont disparu de la vue "Ce PC".

    Pour annuler : remettre le parametre sur **Non configure**

!!! info "Masquer n'est pas interdire"
    Les lecteurs sont **masques**, pas **bloques**. Un utilisateur avise peut toujours y acceder en tapant le chemin directement dans la barre d'adresse (par exemple `D:\`). Pour vraiment bloquer l'acces, il faut utiliser le parametre **Empecher l'acces aux lecteurs a partir du Poste de travail** qui se trouve juste en dessous dans la meme section.

    La difference est importante :

    | Parametre | Effet |
    |-----------|-------|
    | **Masquer** les lecteurs | Le lecteur disparait de l'Explorateur, mais reste accessible via son chemin |
    | **Empecher l'acces** aux lecteurs | Le lecteur est visible mais totalement inaccessible |

    En environnement professionnel, on combine generalement les deux pour un verrouillage complet : les lecteurs sont a la fois masques (invisibles dans l'Explorateur) **et** inaccessibles (bloques meme en tapant le chemin directement). Ceinture et bretelles.

!!! quote "En resume"
    - Les lecteurs selectionnes disparaissent de l'Explorateur, mais restent accessibles par leur chemin direct.
    - Pour un blocage reel, combinez avec le parametre **Empecher l'acces aux lecteurs**.

On passe a une modification un peu plus "musclee" : le blocage de programmes. C'est le genre de parametre qui impressionne les collegues quand on leur montre.

---

## 5. Interdire l'execution de certains programmes

Vous voulez empecher un utilisateur de lancer certaines applications ? Bloquer un jeu installe en douce sur un poste de travail, empecher l'utilisation d'un navigateur non approuve par la politique de securite, ou interdire l'acces a l'invite de commandes ? C'est possible, et c'est etonnamment simple.

Imaginez un poste dans une salle informatique d'ecole. Vous ne voulez pas que les eleves lancent Steam ou Fortnite entre deux exercices. Ou dans une entreprise, vous ne voulez pas que les employes utilisent Spotify sur le PC pro, ou qu'ils lancent un logiciel telecharge depuis un site douteux. Cette strategie fait exactement ca.

Le principe est simple : une liste noire. Vous listez les programmes interdits, tout le reste est autorise.

| Element | Valeur |
|---------|--------|
| **Chemin** | Configuration utilisateur → Modeles d'administration → Systeme |
| **Parametre** | Ne pas executer les applications Windows specifiees |
| **Action** | Activer |
| **Effet** | Les programmes listes sont bloques au lancement |

!!! example "Essayez vous-meme"
    1. Ouvrir gpedit.msc (++win+r++ → `gpedit.msc`)
    2. Naviguer vers **Configuration utilisateur → Modeles d'administration → Systeme**
    3. Double-cliquer sur **Ne pas executer les applications Windows specifiees**
    4. Selectionner **Active**
    5. Cliquer sur le bouton **Afficher...**
    6. Dans la liste, ajouter le **nom de l'executable** a bloquer, par exemple :
        ```
        notepad.exe
        ```
    7. Vous pouvez ajouter plusieurs programmes, un par ligne :
        ```
        notepad.exe
        mspaint.exe
        calc.exe
        ```
    8. Cliquer sur **OK** deux fois

    Pour verifier : essayez de lancer le programme bloque (via le menu Demarrer, un raccourci, ou directement le fichier `.exe`). Un message d'erreur devrait s'afficher :

    ```
    Cette operation a ete annulee en raison de restrictions
    en vigueur sur cet ordinateur. Contactez votre
    administrateur systeme.
    ```

    Pour annuler : remettre le parametre sur **Non configure**

!!! warning "C'est le nom de l'executable qui compte"
    Vous devez entrer le **nom exact du fichier .exe**, pas le nom du raccourci ou le nom affiche dans le menu Demarrer. Quelques exemples courants :

    | Application | Nom de l'executable |
    |-------------|:-------------------:|
    | Google Chrome | `chrome.exe` |
    | Mozilla Firefox | `firefox.exe` |
    | Microsoft Edge | `msedge.exe` |
    | Bloc-notes | `notepad.exe` |
    | Paint | `mspaint.exe` |
    | Invite de commandes | `cmd.exe` |
    | PowerShell | `powershell.exe` |

    Si vous ne connaissez pas le nom de l'executable, deux methodes :

    - Faites un clic droit sur le raccourci → **Proprietes** → regardez le champ **Cible**
    - Ouvrez le **Gestionnaire des taches** (++ctrl+shift+esc++), trouvez l'application dans la liste, faites un clic droit → **Ouvrir l'emplacement du fichier** : le nom du fichier `.exe` est affiche dans l'Explorateur

Un detail important : cette methode est une liste **noire**. Vous bloquez ce que vous listez, tout le reste est autorise. Si vous preferez une approche inverse — tout bloquer sauf ce que vous autorisez explicitement — utilisez plutot le parametre **Executer uniquement les applications Windows specifiees**, juste en dessous dans la meme section.

L'approche "liste blanche" est beaucoup plus restrictive mais aussi beaucoup plus securisee. A vous de choisir selon votre contexte.

!!! info "Limites de cette methode"
    Le blocage par nom d'executable a une faiblesse : si l'utilisateur renomme le fichier `.exe`, le blocage ne fonctionne plus. Par exemple, si vous bloquez `chrome.exe` et que l'utilisateur copie le fichier en `navigateur.exe`, il pourra le lancer.

    Pour une securite plus robuste, les entreprises utilisent **AppLocker** ou **Windows Defender Application Control (WDAC)**, qui bloquent par signature numerique ou par hash du fichier. Mais c'est un sujet avance qu'on abordera plus tard.

!!! quote "En resume"
    - Ajoutez les noms des fichiers `.exe` a bloquer dans la liste (un par ligne).
    - Pour une approche inverse (tout bloquer sauf une liste), utilisez **Executer uniquement les applications Windows specifiees**.

On a couvert la moitie des modifications. Les cinq suivantes touchent a la securite et au confort d'utilisation. On continue.

---

## 6. Desactiver les notifications toast

Les notifications qui apparaissent en bas a droite de l'ecran (ou en haut a droite sur Windows 11) peuvent etre utiles au quotidien. Mais sur un PC de presentation, un kiosque ou simplement si elles vous distraient, cette strategie les fait taire.

Imaginez : vous etes en pleine presentation devant des clients. Votre ecran est projete. Et la, une notification Teams s'affiche avec un message de votre collegue qui dit "T'as vu la tete du directeur ce matin ?". Voila pourquoi ce parametre existe.

Le terme **toast** vient du fait que ces notifications "sautent" du bas de l'ecran (ou du haut sur Windows 11) comme un toast qui sort du grille-pain. Oui, c'est vraiment l'origine du nom. Les developpeurs Microsoft avaient le sens de l'humour ce jour-la.

| Element | Valeur |
|---------|--------|
| **Chemin** | Configuration utilisateur → Modeles d'administration → Menu Demarrer et barre des taches → Notifications |
| **Parametre** | Desactiver les notifications toast |
| **Action** | Activer |
| **Effet** | Plus aucune notification toast ne s'affiche a l'ecran |

!!! example "Essayez vous-meme"
    1. Ouvrir gpedit.msc (++win+r++ → `gpedit.msc`)
    2. Naviguer vers **Configuration utilisateur → Modeles d'administration → Menu Demarrer et barre des taches → Notifications**
    3. Double-cliquer sur **Desactiver les notifications toast**
    4. Selectionner **Active**
    5. Cliquer sur **OK**

    Pour verifier : declenchez une notification (par exemple, changez le volume sonore ou recevez un e-mail). Aucune bulle de notification ne devrait apparaitre a l'ecran.

    Pour annuler : remettre le parametre sur **Non configure**

!!! info "Les notifications ne sont pas perdues"
    Meme si les notifications toast ne s'affichent plus a l'ecran, elles restent consultables dans le **Centre de notifications**. Sur Windows 10, cliquez sur l'icone en bas a droite de la barre des taches. Sur Windows 11, appuyez sur ++win+n++ ou cliquez sur la date et l'heure.

    Les applications continuent de recevoir leurs notifications normalement — c'est juste l'affichage en pop-up qui est desactive.

Ce parametre est le favori des administrateurs qui gerent des salles de reunion ou des postes de demonstration. Zero distraction, zero risque de message embarrassant.

!!! tip "Alternatives plus fines"
    Si vous ne voulez pas desactiver **toutes** les notifications mais seulement celles de certaines applications specifiques, ce n'est pas dans gpedit que ca se passe. Allez plutot dans **Parametres** (++win+i++) **→ Systeme → Notifications** et desactivez les notifications application par application.

    La strategie de groupe est un outil "tout ou rien" pour les notifications. C'est sa force (simplicite) et sa limite (pas de granularite).

!!! quote "En resume"
    - Les notifications toast disparaissent de l'ecran mais restent dans le Centre de notifications.
    - Ideal pour les postes de presentation ou les environnements qui necessitent zero distraction.

On entre maintenant dans le domaine de la **securite**. Les quatre dernieres modifications sont celles que tout administrateur systeme connait par coeur. Elles sont un peu plus "serieuses" que les precedentes — on touche aux mots de passe, au verrouillage de session, a l'acces au registre — mais elles restent tout aussi simples a mettre en place.

La securite, ca n'a pas besoin d'etre complique. Souvent, quelques parametres bien choisis suffisent a couvrir 80% des risques courants. Et les quatre modifications qui suivent font partie de ces "quelques parametres".

---

## 7. Forcer le verrouillage de session apres inactivite

Vous quittez votre poste pour un cafe et vous oubliez de verrouiller votre session ? Cette strategie le fait pour vous. Apres un certain temps d'inactivite — pas de mouvement de souris, pas de frappe clavier — Windows verrouille automatiquement l'ecran.

C'est un classique absolu de la securite en entreprise. Meme si l'utilisateur oublie de taper ++win+l++ avant de quitter son poste, la session se verrouille toute seule.

| Element | Valeur |
|---------|--------|
| **Chemin** | Configuration ordinateur → Parametres Windows → Parametres de securite → Strategies locales → Options de securite |
| **Parametre** | Ouverture de session interactive : limite d'inactivite de l'ordinateur |
| **Action** | Definir une valeur en secondes |
| **Effet** | La session se verrouille automatiquement apres le delai specifie |

!!! example "Essayez vous-meme"
    1. Ouvrir gpedit.msc (++win+r++ → `gpedit.msc`)
    2. Naviguer vers **Configuration ordinateur → Parametres Windows → Parametres de securite → Strategies locales → Options de securite**
    3. Double-cliquer sur **Ouverture de session interactive : limite d'inactivite de l'ordinateur**
    4. Entrer une valeur en **secondes**, par exemple :
        - `300` pour 5 minutes
        - `600` pour 10 minutes
        - `900` pour 15 minutes
    5. Cliquer sur **OK**

    Pour verifier : ne touchez plus a la souris ni au clavier pendant le delai configure. L'ecran de verrouillage devrait apparaitre automatiquement.

    **Astuce pour tester rapidement** : mettez temporairement la valeur a `60` (1 minute). Attendez sans rien toucher. L'ecran se verrouille ? Ca fonctionne. Remettez ensuite la valeur souhaitee (par exemple `900` pour 15 minutes).

    Pour annuler : remettre la valeur a `0` (desactive le verrouillage automatique)

!!! tip "Quel delai choisir ?"
    Voici quelques recommandations selon le contexte :

    | Contexte | Delai recommande | En secondes |
    |----------|:----------------:|:-----------:|
    | Poste en libre acces (bibliotheque, borne) | 5 minutes | `300` |
    | Bureau standard en entreprise | 10 minutes | `600` |
    | Norme courante en entreprise | 15 minutes | `900` |
    | Poste de developpeur (compilations longues) | 30 minutes | `1800` |

    Le bon delai est un compromis entre securite et confort. Trop court, et les utilisateurs devront retaper leur mot de passe toutes les cinq minutes — bonjour la productivite. Trop long, et le poste reste expose pendant des heures apres le depart de l'utilisateur.

Ce parametre se trouve dans la branche **Configuration ordinateur**. Il s'applique donc a **tous les utilisateurs** de la machine, quel que soit le compte connecte. C'est exactement le comportement souhaite : la securite ne devrait pas dependre de la bonne volonte de chaque utilisateur.

Vous aurez peut-etre remarque que ce parametre ne se trouve pas dans "Modeles d'administration" comme les precedents, mais dans "Parametres Windows → Parametres de securite". C'est une section differente de gpedit, dediee aux reglages de securite. L'interface est un peu differente (pas d'onglet "Expliquer"), mais le principe reste le meme.

!!! info "Difference avec l'ecran de veille"
    Ne confondez pas ce parametre avec l'ecran de veille securise. L'ecran de veille est un parametre **utilisateur** (chaque utilisateur peut le configurer differemment). Le verrouillage par inactivite via les Options de securite est un parametre **machine** — il s'impose a tout le monde.

    Les deux mecanismes peuvent coexister sans conflit. Mais le verrouillage par inactivite via les Options de securite est plus fiable pour une raison simple : l'utilisateur ne peut pas le desactiver depuis les Parametres. L'ecran de veille, lui, peut etre modifie par n'importe quel utilisateur depuis la personnalisation du Bureau.

!!! quote "En resume"
    - Definissez la duree en secondes (300 = 5 min, 600 = 10 min, 900 = 15 min).
    - Pour desactiver, remettez la valeur a `0`.

---

## 8. Empecher les utilisateurs d'acceder au registre

Le registre Windows est un outil puissant — mais aussi un outil dangereux entre de mauvaises mains. Cette strategie bloque l'acces a regedit.exe pour les utilisateurs qui n'en ont pas besoin.

Si vous avez des utilisateurs qui aiment "optimiser" leur PC en suivant des tutoriels trouves sur Internet — du genre "supprimez cette cle pour booster votre PC de 300%" — ce parametre pourrait vous eviter bien des appels au support et des reinstallations le vendredi a 17h.

| Element | Valeur |
|---------|--------|
| **Chemin** | Configuration utilisateur → Modeles d'administration → Systeme |
| **Parametre** | Empecher l'acces aux outils de modification du Registre |
| **Action** | Activer |
| **Effet** | L'utilisateur ne peut plus ouvrir regedit.exe ni executer de fichiers .reg |

!!! example "Essayez vous-meme"
    1. Ouvrir gpedit.msc (++win+r++ → `gpedit.msc`)
    2. Naviguer vers **Configuration utilisateur → Modeles d'administration → Systeme**
    3. Double-cliquer sur **Empecher l'acces aux outils de modification du Registre**
    4. Selectionner **Active**
    5. Dans l'option **Desactiver l'execution silencieuse de regedit ?**, choisir :
        - **Oui** : bloque aussi l'importation silencieuse de fichiers `.reg` (recommande)
        - **Non** : bloque uniquement l'interface graphique de regedit
    6. Cliquer sur **OK**

    Pour verifier : essayez d'ouvrir regedit (++win+r++ → `regedit`). Un message devrait s'afficher :

    ```
    L'edition du Registre a ete desactivee par votre
    administrateur.
    ```

    Pour annuler : remettre le parametre sur **Non configure**

!!! danger "Ne vous enfermez pas dehors !"
    Si vous activez ce parametre pour **votre propre compte** sans avoir un autre compte administrateur, vous ne pourrez plus acceder au registre. C'est un piege classique.

    Avant d'activer ce parametre, assurez-vous d'au moins une de ces conditions :

    - Vous avez un **autre compte administrateur** sur la machine
    - Vous faites ce changement via une **GPO de domaine** sur des comptes utilisateurs standards
    - Vous savez que **gpedit.msc reste accessible** meme quand regedit est bloque (c'est le cas)

    Si vous vous etes piege, pas de panique : rouvrez gpedit.msc et remettez le parametre sur **Non configure**. L'editeur de strategies de groupe n'est pas affecte par ce blocage.

Petite subtilite : l'option "Desactiver l'execution silencieuse de regedit" merite votre attention. Si vous la laissez sur **Non**, un utilisateur pourrait toujours importer un fichier `.reg` en double-cliquant dessus — les modifications du registre contenues dans le fichier seraient appliquees sans que regedit ne s'ouvre visuellement. Mettez-la sur **Oui** pour un blocage complet.

C'est une distinction importante. "L'execution silencieuse" designe l'importation de fichiers `.reg` via la ligne de commande (`regedit /s fichier.reg`) ou par double-clic, sans que l'interface graphique de regedit ne s'ouvre. Les modifications du registre sont appliquees en arriere-plan, sans aucune confirmation. Bloquer uniquement l'interface graphique tout en laissant l'importation silencieuse active, c'est comme fermer la porte d'entree tout en laissant la fenetre grande ouverte.

!!! quote "En resume"
    - Bloque l'acces a regedit et optionnellement l'importation de fichiers `.reg`.
    - Gpedit.msc reste accessible — c'est votre porte de secours si vous vous etes piege.

Plus que deux modifications. La suivante est la plus complete de toutes — elle regroupe plusieurs parametres dans une meme section. Et la derniere est celle que tout le monde attend avec impatience.

---

## 9. Configurer la politique de mot de passe

Soyons honnetes : la plupart des utilisateurs choisissent des mots de passe lamentables. "123456", "motdepasse", "azerty", le prenom du chien, la date de naissance du petit dernier... Vous connaissez la chanson.

Les statistiques sont sans appel : les mots de passe les plus utilises au monde restent "123456", "password" et "qwerty" — et ce, annee apres annee. Dans une entreprise, un seul mot de passe faible peut compromettre tout le reseau. Un attaquant n'a besoin que d'un point d'entree. Cette section de gpedit regroupe plusieurs parametres pour imposer une politique de mots de passe solide.

Contrairement aux modifications precedentes qui sont de simples interrupteurs, celle-ci est une **famille de parametres** qui travaillent ensemble pour definir une politique coherente. On va les configurer un par un.

| Element | Valeur |
|---------|--------|
| **Chemin** | Configuration ordinateur → Parametres Windows → Parametres de securite → Strategies de comptes → Strategie de mot de passe |
| **Parametre** | Plusieurs parametres dans cette section |
| **Action** | Configurer chaque parametre individuellement |
| **Effet** | Les mots de passe doivent respecter les regles definies |

Voici les parametres disponibles dans cette section et les valeurs recommandees :

| Parametre | Description | Valeur recommandee |
|-----------|-------------|:------------------:|
| Longueur minimale du mot de passe | Nombre minimum de caracteres | `12` |
| Le mot de passe doit respecter des exigences de complexite | Majuscules, minuscules, chiffres, symboles | **Active** |
| Duree de vie maximale du mot de passe | Nombre de jours avant expiration | `90` |
| Duree de vie minimale du mot de passe | Nombre de jours avant de pouvoir changer | `1` |
| Conserver l'historique des mots de passe | Nombre d'anciens mots de passe memorises | `5` |

!!! example "Essayez vous-meme"
    1. Ouvrir gpedit.msc (++win+r++ → `gpedit.msc`)
    2. Naviguer vers **Configuration ordinateur → Parametres Windows → Parametres de securite → Strategies de comptes → Strategie de mot de passe**
    3. Double-cliquer sur **Longueur minimale du mot de passe**
    4. Entrer `12` comme nombre minimum de caracteres
    5. Cliquer sur **OK**
    6. Double-cliquer sur **Le mot de passe doit respecter des exigences de complexite**
    7. Selectionner **Active**
    8. Cliquer sur **OK**
    9. Repetez pour les autres parametres selon vos besoins

    Pour verifier : essayez de changer le mot de passe de votre compte (++ctrl+alt+del++ → **Modifier un mot de passe**) en utilisant un mot de passe court comme "abc". Windows devrait refuser avec un message expliquant que le mot de passe ne respecte pas les exigences.

    Vous pouvez aussi creer un nouveau compte utilisateur local (Parametres → Comptes → Famille et autres utilisateurs) avec un mot de passe faible pour tester.

    Pour annuler : remettre chaque parametre sur sa valeur d'origine (longueur a `0`, complexite sur **Desactive**)

!!! warning "Attention a votre propre mot de passe"
    Si vous activez ces regles et que votre propre mot de passe ne les respecte pas, Windows ne vous forcera pas a le changer immediatement. Mais au prochain changement de mot de passe (volontaire ou a expiration), le nouveau mot de passe devra respecter les nouvelles regles. Assurez-vous de vous en souvenir !

!!! info "Complexite : qu'est-ce que Windows exige exactement ?"
    Quand l'exigence de complexite est activee, le mot de passe doit contenir des caracteres d'au moins **trois** de ces quatre categories :

    - Lettres majuscules (A-Z)
    - Lettres minuscules (a-z)
    - Chiffres (0-9)
    - Symboles (!, @, #, $, %, etc.)

    De plus, le mot de passe **ne doit pas** contenir le nom du compte utilisateur ni plus de deux caracteres consecutifs du nom complet de l'utilisateur.

    Quelques exemples concrets pour un utilisateur nomme "Jean Dupont" :

    | Mot de passe | Verdict | Raison |
    |-------------|:-------:|--------|
    | `Jean2024!` | Refuse | Contient "Jean" (nom du compte) |
    | `monmotdepasse` | Refuse | Pas de majuscule, pas de chiffre, pas de symbole |
    | `Abc123` | Refuse | Trop court (moins de 12 caracteres si longueur minimale configuree) |
    | `Tr0mbone#42bleu` | Accepte | Longueur suffisante, 4 categories presentes, pas de nom |

Pourquoi une **duree de vie minimale** de 1 jour ? C'est pour empecher un utilisateur malin de changer son mot de passe 5 fois de suite pour "epuiser" l'historique et remettre son ancien mot de passe. Avec un delai d'un jour entre chaque changement, cette ruse devient tres penible.

!!! tip "Tendance actuelle : des mots de passe longs plutot que complexes"
    Les recommandations evoluent. Le NIST (l'organisme americain de reference en cybersecurite) recommande desormais de privilegier la **longueur** plutot que la complexite. Un mot de passe comme `MonChatMangeDesCroquettes` (25 caracteres, facile a retenir) est plus securise que `Tr0mb!42` (8 caracteres, difficile a retenir).

    Si vous pouvez, augmentez la longueur minimale a **14 ou 16 caracteres** et envisagez de desactiver l'exigence de complexite. Vos utilisateurs vous remercieront.

!!! quote "En resume"
    - Configurez une longueur minimale de 12 caracteres et activez les exigences de complexite.
    - Ces regles s'appliquent a **tous les comptes locaux** de la machine.

Derniere modification, et pas des moindres. Celle-ci, c'est celle qui sauvera votre travail non sauvegarde un jour ou l'autre.

---

## 10. Desactiver le redemarrage automatique apres mise a jour

On a tous vecu ca au moins une fois. Certains d'entre nous l'ont vecu plusieurs fois. Et certains ont appris la dure lecon avec un document important non sauvegarde.

Voici le scenario : vous etes en plein travail. Vous partez cinq minutes chercher un cafe. Quand vous revenez, votre ecran affiche "Mise a jour en cours... ne pas eteindre votre ordinateur". Travail non sauvegarde ? Perdu. Onglets de navigateur ? Disparus. Bonne humeur ? Evaporee.

Le pire, c'est que Windows vous avait prevenu. Mais la notification etait apparue pendant votre pause dejeuner, et le compte a rebours s'est ecoule sans que vous la voyiez. Quand vous etes revenu, le redemarrage etait deja en cours.

Ce scenario est tellement courant qu'il a engendre des milliers de memes sur Internet. Et une strategie de groupe pour y remedier.

Cette strategie empeche ce scenario cauchemardesque. Les mises a jour continuent a s'installer normalement, mais c'est **vous** qui choisissez le moment du redemarrage. Plus de surprise.

| Element | Valeur |
|---------|--------|
| **Chemin** | Configuration ordinateur → Modeles d'administration → Composants Windows → Windows Update → Gestion de l'experience utilisateur final |
| **Parametre** | Pas de redemarrage automatique avec des utilisateurs connectes pour les installations de mises a jour automatiques planifiees |
| **Action** | Activer |
| **Effet** | Windows ne redemarrera plus automatiquement tant qu'un utilisateur est connecte |

!!! example "Essayez vous-meme"
    1. Ouvrir gpedit.msc (++win+r++ → `gpedit.msc`)
    2. Naviguer vers **Configuration ordinateur → Modeles d'administration → Composants Windows → Windows Update → Gestion de l'experience utilisateur final**
    3. Double-cliquer sur **Pas de redemarrage automatique avec des utilisateurs connectes pour les installations de mises a jour automatiques planifiees**
    4. Selectionner **Active**
    5. Cliquer sur **OK**

    Pour verifier : lors de la prochaine mise a jour, Windows devrait vous **demander** de redemarrer au lieu de le faire tout seul. Vous verrez une notification vous invitant a planifier le redemarrage au lieu d'un redemarrage surprise.

    Pour verifier immediatement que le parametre est bien pris en compte, vous pouvez lancer `gpupdate /force` dans une invite de commandes, puis verifier avec `gpresult /r` que la strategie apparait dans les resultats.

    Pour annuler : remettre le parametre sur **Non configure**

!!! warning "N'oubliez pas de redemarrer manuellement"
    Cette strategie empeche le redemarrage **automatique**, pas les mises a jour elles-memes. Les mises a jour continueront a se telecharger et a s'installer en arriere-plan. Mais c'est **vous** qui decidez quand redemarrer.

    Prenez l'habitude de redemarrer regulierement — au moins une fois par semaine — pour appliquer les correctifs de securite. Un PC qui n'est jamais redemarrage accumule les mises a jour en attente et devient de plus en plus vulnerable. Les correctifs de securite ne protegent votre PC que s'ils sont effectivement appliques — et pour beaucoup d'entre eux, le redemarrage est l'etape finale indispensable.

Le chemin de navigation pour ce parametre est assez long. Si vous avez du mal a le trouver, voici une astuce : dans gpedit.msc, vous pouvez utiliser le filtre. Cliquez sur **Action → Filtrage...** et cherchez les mots "redemarrage automatique" pour trouver le parametre plus rapidement.

Ce parametre est un favori absolu des utilisateurs avances. C'est souvent le tout premier qu'on active sur une nouvelle installation de Windows.

!!! info "Versions de Windows et sous-dossiers"
    Sur certaines versions de Windows 10/11, le sous-dossier **Gestion de l'experience utilisateur final** peut s'appeler differemment ou etre absent. Si vous ne le trouvez pas, cherchez le parametre directement dans le dossier **Windows Update**. Le nom du parametre reste le meme.

    Autre point : sur Windows 11 23H2 et versions ulterieures, Windows peut se montrer plus insistant avec les redemarrages. Ce parametre reste efficace — il empeche bien le redemarrage automatique — mais Windows affichera des rappels de plus en plus frequents et insistants pour vous inciter a redemarrer volontairement. C'est le compromis : plus de redemarrages surprises, mais des rappels reguliers.

!!! quote "En resume"
    - Windows ne redemarrera plus automatiquement pendant que vous travaillez.
    - Pensez a redemarrer manuellement et regulierement pour appliquer les mises a jour de securite.

---

## Tableau recapitulatif

Vous avez traverse les 10 modifications. Bravo ! Voici un tableau qui resume tout en un coup d'oeil. Gardez-le sous la main comme reference rapide :

| # | Modification | Branche | Difficulte |
|:-:|-------------|---------|:----------:|
| 1 | Interdire l'acces au Panneau de configuration | Utilisateur | :material-star: |
| 2 | Desactiver le telechargement automatique de pilotes | Ordinateur | :material-star: |
| 3 | Forcer un fond d'ecran | Utilisateur | :material-star::material-star: |
| 4 | Masquer les lecteurs dans l'Explorateur | Utilisateur | :material-star: |
| 5 | Interdire l'execution de certains programmes | Utilisateur | :material-star::material-star: |
| 6 | Desactiver les notifications toast | Utilisateur | :material-star: |
| 7 | Forcer le verrouillage apres inactivite | Ordinateur | :material-star: |
| 8 | Empecher l'acces au registre | Utilisateur | :material-star::material-star: |
| 9 | Configurer la politique de mot de passe | Ordinateur | :material-star::material-star::material-star: |
| 10 | Desactiver le redemarrage auto apres MAJ | Ordinateur | :material-star: |

Quelques observations sur ce tableau :

- **6 modifications sur 10** se trouvent dans la branche **Configuration utilisateur**. Ce sont celles qui controlent le comportement d'un profil specifique. Si vous faites une erreur, vous pouvez toujours vous connecter avec un autre compte pour corriger.
- **4 modifications** se trouvent dans la branche **Configuration ordinateur**. Elles affectent le systeme pour **tous** les utilisateurs de la machine. Soyez un peu plus prudent avec celles-ci.
- La plupart sont de difficulte :material-star: — un simple clic suffit. Seule la politique de mot de passe (:material-star::material-star::material-star:) demande de configurer plusieurs parametres.
- Les modifications :material-star::material-star: demandent de saisir des informations supplementaires (liste de programmes, chemin d'image, option de configuration).

Si vous debutez, commencez par les modifications :material-star: pour vous familiariser avec l'interface. Une fois a l'aise, passez aux :material-star::material-star: puis aux :material-star::material-star::material-star:.

Vous pouvez aussi imprimer ce tableau ou le garder en signet — c'est une reference utile quand on veut retrouver rapidement un parametre.

!!! tip "Combiner les modifications"
    Ces 10 modifications ne sont pas independantes les unes des autres. En les combinant intelligemment, vous pouvez creer des profils de securite adaptes a votre contexte :

    - **Poste de presentation** : modifications 3 (fond d'ecran) + 6 (notifications) + 7 (verrouillage)
    - **Poste en libre-service** : modifications 1 (panneau de config) + 4 (lecteurs) + 5 (programmes) + 7 (verrouillage) + 8 (registre)
    - **Poste de travail standard** : modifications 7 (verrouillage) + 9 (mot de passe) + 10 (redemarrage)
    - **Poste de developpeur** : modification 2 (pilotes) + 10 (redemarrage) — les developpeurs ont besoin de liberte, mais pas de redemarrages surprises en pleine compilation

Et surtout : toutes ces modifications sont **reversibles**. Il suffit de remettre le parametre sur **Non configure** pour retrouver le comportement par defaut de Windows.


!!! quote "En résumé"
    - # : Difficulte.
    - 1 : material-star.
    - 2 : material-star.
    - 3 : material-star::material-star.
    - 6 modifications sur 10 se trouvent dans la branche Configuration utilisateur.
---

## Et apres ?

Vous avez maintenant 10 modifications concretes dans votre boite a outils. Mais gpedit.msc contient des **milliers** de parametres. Ceux qu'on a vu ici ne sont que la partie emergee de l'iceberg.

Quelques pistes pour aller plus loin par vous-meme :

- **Explorez les descriptions** : dans gpedit, quand vous double-cliquez sur un parametre, l'onglet **Expliquer** (ou **Commentaire** selon la version de Windows) contient une description detaillee de ce que fait le parametre, quelles versions de Windows le supportent, et quels sont les effets de chaque option. Lisez-la, c'est une mine d'informations souvent meconnue.
- **Commencez par "Configuration utilisateur"** : les parametres utilisateur sont moins risques que les parametres ordinateur. Si quelque chose tourne mal, il suffit de se connecter avec un autre compte pour corriger. C'est votre filet de securite.
- **Ne touchez pas a ce que vous ne comprenez pas** : c'est la regle d'or. Si la description d'un parametre ne vous parle pas, laissez-le tranquille. Mieux vaut un parametre non configure qu'un parametre mal configure.
- **Testez sur un compte secondaire** : si vous avez peur de casser quelque chose, creez un compte utilisateur local de test. Appliquez vos modifications dessus, verifiez le resultat, et seulement ensuite appliquez au compte reel. C'est la methode utilisee par les professionnels — meme les administrateurs experimentes testent avant de deployer en production.

!!! info "Non configure vs Desactive : quelle difference ?"
    On l'a mentionne au debut du chapitre, et c'est le moment d'y revenir. La difference est subtile mais importante :

    - **Non configure** : Windows se comporte comme si la strategie **n'existait pas**. C'est l'etat par defaut, neutre.
    - **Desactive** : Windows **empeche activement** le comportement decrit par la strategie.

    Exemple concret avec "Interdire l'acces au Panneau de configuration" :

    | Etat | Effet |
    |------|-------|
    | **Non configure** | Le Panneau de configuration est accessible (comportement par defaut de Windows) |
    | **Active** | Le Panneau de configuration est **bloque** |
    | **Desactive** | Le Panneau de configuration est **explicitement autorise**, meme si une autre strategie essaie de le bloquer |

    En pratique, pour la plupart des parametres locaux (gpedit sur un seul PC), "Non configure" et "Desactive" ont le meme effet visible. La difference devient cruciale en environnement Active Directory, quand plusieurs GPO de differents niveaux (domaine, site, OU) peuvent se superposer et se contredire. "Desactive" a le pouvoir d'annuler un "Active" venu d'une GPO de niveau superieur.

    Mais c'est un sujet pour les chapitres suivants. Pour l'instant, retenez simplement : pour annuler vos modifications, utilisez **Non configure**.

---
!!! tip "Tenez un journal de modifications"
    Creez un fichier texte simple et notez chaque modification que vous faites. Voici un modele que vous pouvez copier-coller :

    ```text
    === Journal des modifications GPO ===

    Date : 2025-01-15
    Parametre : Desactiver les notifications toast
    Chemin : Config utilisateur > Modeles d'admin > Menu Demarrer > Notifications
    Etat avant : Non configure
    Etat apres : Active
    Raison : PC de presentation en salle de reunion
---
    Date : 2025-01-15
    Parametre : Verrouillage apres inactivite
    Chemin : Config ordinateur > Parametres Windows > Securite > Options de securite
    Etat avant : 0
    Etat apres : 900
    Raison : Securisation du poste d'accueil
---
    Date : 2025-01-20
    Parametre : Longueur minimale du mot de passe
    Chemin : Config ordinateur > Parametres Windows > Securite > Strategie de mot de passe
    Etat avant : 0
    Etat apres : 12
    Raison : Renforcement de la politique de securite
    ```

    Ce petit reflexe vous fera gagner des **heures** de diagnostic le jour ou quelque chose ne fonctionne plus comme prevu. Un probleme apparait ? Consultez votre journal, identifiez la derniere modification, annulez-la, et testez a nouveau.

!!! info "Pour les plus organises : utilisez gpresult"
    Si vous voulez voir un rapport complet de toutes les strategies appliquees sur votre machine, ouvrez une invite de commandes en **administrateur** et tapez :

    ```batch
    gpresult /h C:\Users\%USERNAME%\Desktop\rapport-gpo.html
    ```

    Ca genere un fichier HTML sur votre Bureau avec un rapport detaille et lisible de toutes les strategies en vigueur. Ouvrez-le dans votre navigateur et vous verrez :

    - Les strategies **ordinateur** appliquees
    - Les strategies **utilisateur** appliquees
    - La date et l'heure de la derniere mise a jour des strategies
    - Les eventuelles erreurs d'application

    Tres pratique pour retrouver une modification quand on a oublie de tenir son journal. Ou pour diagnostiquer un probleme sur le poste d'un collegue.

---
!!! quote "En resume"
    - Gpedit.msc permet de modifier le comportement de Windows sans toucher au registre directement.
    - Chaque parametre a trois etats : **Non configure**, **Active**, **Desactive**.
    - Pour annuler, remettez simplement le parametre sur **Non configure**.
    - **Notez toujours** ce que vous changez, quand, et pourquoi.
    - Les parametres de la branche **Utilisateur** n'affectent que le profil courant.
    - Les parametres de la branche **Ordinateur** affectent tous les utilisateurs de la machine.
    - En cas de doute, remettez tout sur **Non configure** — c'est le bouton "reset" de gpedit.

Vous avez desormais 10 modifications concretes sous la main, un journal pour les suivre, la commande `gpupdate /force` pour appliquer vos changements immediatement, et la commande `gpresult` pour les retrouver. Pas mal pour un seul chapitre, non ?

---
Dans le [chapitre suivant](04-gpmc-premiers-pas.md), on quitte le monde du PC local pour decouvrir la **console de gestion des strategies de groupe (GPMC)** — l'outil qui permet de gerer les GPO a l'echelle d'un domaine Active Directory. On passera de la gestion d'un seul PC a la gestion de centaines de machines.

C'est le grand saut. Mais avec les bases solides que vous venez d'acquerir dans ce chapitre — naviguer dans gpedit, activer des parametres, verifier les effets, annuler proprement — vous avez tous les reflexes necessaires pour aborder la suite sereinement.
