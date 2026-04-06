---
description: "Adapter vos strategies de groupe a Windows 11 : nouveaux ADMX, menu Demarrer JSON, barre des taches et Copilot."
tags:
  - gpo
  - debutant
  - windows11
---
# GPO et Windows 11

!!! abstract "Ce que vous allez apprendre"
    - Pourquoi les GPO Windows 10 fonctionnent (presque) toutes sur Windows 11, et dans quels cas elles echouent
    - Comment mettre a jour les modeles ADMX dans le magasin central pour gerer les nouveaux parametres Windows 11
    - Deployer un menu Demarrer personnalise avec la nouvelle methode JSON (l'ancien XML ne fonctionne plus)
    - Controler les nouveaux elements de la barre des taches : Widgets, Copilot, bouton Chat Teams
    - Construire une checklist de migration GPO avant de basculer votre parc vers Windows 11


!!! tip "Si vous ne retenez qu'une chose"
    Sous Windows 11, la version des ADMX et l'identification précise des applications conditionnent la réussite des politiques modernes.

---

## Windows 11 et GPO : la bonne nouvelle d'abord

Avant de parler des differences, rassurons-nous : **la grande majorite de vos GPO existantes continuent de fonctionner sans modification sur Windows 11**.

Les mecanismes fondamentaux n'ont pas change. Le moteur de traitement des GPO est le meme, le protocole LDAP pour recuperer les objets est identique, et `gpresult` fonctionne exactement comme avant.

Concretement, vos parametres de securite, vos restrictions logicielles, vos configurations d'impression, vos scripts de demarrage... tout cela s'applique sur Windows 11 sans toucher une seule ligne.

!!! info "Compatibilite descendante preservee"
    Microsoft s'est engage depuis des decennies sur la compatibilite descendante des GPO. Une GPO ecrite pour Windows XP peut encore s'appliquer sur Windows 11 si le parametre existe toujours. Ce n'est pas un accident, c'est une politique deliberee de Microsoft pour proteger les investissements des entreprises.

Alors ou est le probleme ? Dans deux zones precises : le **menu Demarrer** et certains **elements de l'interface utilisateur** qui ont ete profondement repenses dans Windows 11.

!!! quote "En resume"
    - La tres grande majorite des GPO Windows 10 fonctionnent sans modification sur Windows 11.
    - Les mecanismes fondamentaux (LDAP, traitement des GPO, gpresult) sont identiques.
    - Deux zones posent probleme : le menu Demarrer (rupture complete) et certains elements de la barre des taches.

---

## Mettre a jour les modeles ADMX pour Windows 11

### Pourquoi les ADMX ont-ils besoin d'etre mis a jour ?

Les modeles ADMX sont les "dictionnaires" que la console GPMC utilise pour afficher les parametres disponibles dans l'editeur de GPO.

Quand Microsoft ajoute de nouveaux parametres dans Windows 11 (le bouton Copilot, les Widgets, Snap Layouts...), ces parametres ne sont pas visibles dans votre console GPMC tant que vous n'avez pas installe les **modeles administratifs Windows 11** correspondants.

Sans mise a jour des ADMX, le parametre existe bien dans Windows 11, mais il est invisible depuis votre console d'administration. C'est comme essayer de cuisiner une recette dont vous ne connaissez pas les ingredients.

!!! warning "Magasin central ou magasin local ?"
    Si vous utilisez un **magasin central** (recommande en entreprise), les ADMX sont stockes dans SYSVOL et partages avec tous vos controleurs de domaine. C'est le seul endroit a mettre a jour. Si vous n'avez pas encore configure de magasin central, vos ADMX sont stockes localement sur chaque machine d'administration — situation a eviter.

    Le magasin central se trouve ici : `\\<domaine>\SYSVOL\<domaine>\Policies\PolicyDefinitions\`

### Ou telecharger les modeles Windows 11

Les modeles administratifs officiels sont disponibles sur le **Centre de telechargement Microsoft** sous le nom "Administrative Templates (.admx) for Windows 11".

Cherchez "Windows 11 Administrative Templates" sur le site de Microsoft. Le fichier telechargeable est un MSI ou un ZIP qui contient les fichiers `.admx` et les sous-dossiers de langue (dont `fr-FR`).

!!! tip "Verifier la version des modeles"
    Microsoft publie des modeles a chaque version majeure de Windows 11 (22H2, 23H2, 24H2...). Assurez-vous de telecharger la version correspondant a la **version la plus recente de Windows 11** presente dans votre parc. Les modeles sont compatibles vers le bas : des modeles 24H2 peuvent gerer des postes 22H2.

### Installer les modeles dans le magasin central

!!! example "Pas a pas"

    **Etape 1** : Telecharger les modeles administratifs Windows 11.

    Rendez-vous sur le Centre de telechargement Microsoft et telechargez les "Administrative Templates (.admx) for Windows 11". Executez le MSI ou extrayez le ZIP dans un dossier temporaire, par exemple `C:\Temp\ADMX-Win11\`.

    **Etape 2** : Identifier votre magasin central.

    Ouvrez l'Explorateur de fichiers et naviguez vers :
    ```
    \\<votre-domaine>\SYSVOL\<votre-domaine>\Policies\PolicyDefinitions\
    ```
    Si ce dossier n'existe pas, le magasin central n'est pas configure. Creez-le d'abord.

    **Etape 3** : Copier les fichiers ADMX.

    Copiez tous les fichiers `.admx` depuis le dossier temporaire vers `PolicyDefinitions\`. Acceptez le remplacement des fichiers existants.

    **Etape 4** : Copier les fichiers de langue.

    Copiez le contenu du dossier `fr-FR\` (fichiers `.adml`) vers `PolicyDefinitions\fr-FR\`. Acceptez le remplacement.

    **Etape 5** : Verifier dans la console GPMC.

    Ouvrez une GPO en edition. Les nouveaux parametres Windows 11 doivent maintenant apparaitre dans l'arborescence. Aucun redemarrage n'est necessaire.

!!! info "Pas de redemarrage requis"
    La console GPMC recharge les modeles ADMX a chaque ouverture d'une GPO. Inutile de redemarrer quoi que ce soit apres la copie des fichiers.

### Verifier la version des ADMX en place avec PowerShell

Avant d'intervenir, il est utile de savoir ce qui est deja present dans le magasin central. Ce script affiche les dix fichiers ADMX les plus recemment modifies.

```powershell
# Check ADMX version in the Central Store
# Lists the 10 most recently modified ADMX files
$centralStore = "\\$env:USERDNSDOMAIN\SYSVOL\$env:USERDNSDOMAIN\Policies\PolicyDefinitions"

if (Test-Path $centralStore) {
    Write-Host "Central Store found: $centralStore" -ForegroundColor Green
    Write-Host ""
    Write-Host "Most recently updated templates:" -ForegroundColor Cyan

    Get-ChildItem $centralStore -Filter "*.admx" |
        Select-Object Name, LastWriteTime |
        Sort-Object LastWriteTime -Descending |
        Select-Object -First 10 |
        Format-Table -AutoSize
} else {
    Write-Warning "Central Store not found at: $centralStore"
    Write-Host "You may be using local templates only." -ForegroundColor Yellow
}
```

Pour verifier specifiquement que les modeles Windows 11 sont presents, cherchez ces fichiers caracteristiques :

| Fichier ADMX | Parametres couverts |
|:-------------|:--------------------|
| `WindowsAI.admx` | Copilot Windows, Recall |
| `NewsAndInterests.admx` | Widget Meteo et actualites (Widgets) |
| `Taskbar.admx` | Bouttons de la barre des taches Windows 11 |
| `StartMenu.admx` | Menu Demarrer (JSON layout) |
| `SnapLayout.admx` | Snap Layouts et disposition des fenetres |

Si ces fichiers sont absents ou dates d'avant 2022, vos modeles ne couvrent pas Windows 11.

!!! quote "En resume"
    - Les ADMX sont les "dictionnaires" de l'editeur GPO : sans les modeles Windows 11, les nouveaux parametres sont invisibles.
    - Copiez les fichiers `.admx` dans `PolicyDefinitions\` et les `.adml` dans `PolicyDefinitions\fr-FR\` sur SYSVOL.
    - Aucun redemarrage requis : la GPMC recharge les modeles automatiquement.
    - Verifiez la presence de `WindowsAI.admx`, `NewsAndInterests.admx` et `Taskbar.admx` pour confirmer que les modeles Windows 11 sont bien en place.

---

## Le menu Demarrer de Windows 11 : le grand changement

### Une rupture, pas une evolution

C'est le sujet qui fait le plus parler en entreprise. Le menu Demarrer de Windows 11 n'est **pas une version amelioree** du menu Windows 10 : c'est un composant entierement different, reecrit from scratch par Microsoft.

Consequence directe pour les administrateurs : la **GPO de disposition du menu Demarrer Windows 10 ne fonctionne pas sur Windows 11**. Si vous l'appliquez sur un poste Windows 11, elle sera ignoree silencieusement.

| | Windows 10 | Windows 11 |
|:-:|:----------:|:----------:|
| Format de configuration | XML | JSON |
| Parametre GPO | "Configurer la disposition de l'ecran de demarrage" | "Configurer la disposition du menu Demarrer" |
| Emplacement | Config. utilisateur → Bureau | Config. utilisateur → Menu Demarrer et barre des taches |
| Compatibilite croisee | Non applicable sur Win 11 | Non applicable sur Win 10 |

### Pourquoi Microsoft a change le format ?

Le format XML Windows 10 etait puissant mais tres rigide. Il controlait toute la disposition : les tuiles, leur taille, leur emplacement exact, les groupes. L'administrateur avait tout ou rien.

Windows 11 supprime les tuiles vivantes. Le menu Demarrer est reduit a une grille de **raccourcis epingles**. Le format JSON reflette cette simplification : vous fournissez une liste ordonnee d'applications a epingler, et Windows 11 les dispose dans sa grille.

!!! info "Epingler n'est pas tout controler"
    La GPO Windows 11 permet de definir les applications **epinglees** dans le menu Demarrer. Elle ne permet pas de contr controler la disposition exacte (quel raccourci dans quelle case), ni la section "Recommandations" du bas. C'est volontaire : Microsoft considere que cette section doit rester personnelle.

### Le format JSON explique

Voici la structure de base d'un fichier JSON de disposition du menu Demarrer Windows 11 :

```json
{
  "pinnedList": [
    {
      "packagedAppId": "Microsoft.WindowsNotepad_8wekyb3d8bbwe!App"
    },
    {
      "packagedAppId": "Microsoft.WindowsCalculator_8wekyb3d8bbwe!App"
    },
    {
      "packagedAppId": "Microsoft.WindowsStore_8wekyb3d8bbwe!App"
    },
    {
      "desktopAppId": "MSEdge"
    },
    {
      "desktopAppId": "Microsoft.Office.WINWORD.EXE.15"
    }
  ]
}
```

Les deux types d'entrees sont :

| Propriete | Usage | Exemple |
|:----------|:------|:--------|
| `packagedAppId` | Applications du Microsoft Store (packagées, MSIX) | `Microsoft.WindowsNotepad_8wekyb3d8bbwe!App` |
| `desktopAppId` | Applications Win32 classiques (EXE) | `MSEdge`, `Microsoft.Office.WINWORD.EXE.15` |

!!! tip "Trouver l'identifiant d'une application packagee"
    Ouvrez PowerShell et executez la commande suivante pour lister toutes les applications packagees et leurs identifiants :

    ```powershell
    # List all installed packaged app IDs for the current user
    Get-StartApps | Select-Object Name, AppId | Sort-Object Name | Format-Table -AutoSize
    ```

    La colonne `AppId` donne exactement la valeur a utiliser dans `packagedAppId`.

!!! tip "Trouver l'identifiant d'une application Win32"
    Pour les applications classiques, l'identifiant correspond souvent au nom du raccourci dans le menu Demarrer sans extension. Verifiez dans `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\`. Le nom du fichier `.lnk` (sans l'extension) est generalement l'identifiant.

    Pour Microsoft Edge, l'identifiant est simplement `MSEdge`.

### Exemple complet : menu Demarrer d'entreprise

Voici un JSON representatif d'un menu Demarrer standardise pour une entreprise utilisant Microsoft 365 :

```json
{
  "pinnedList": [
    {
      "desktopAppId": "MSEdge"
    },
    {
      "packagedAppId": "Microsoft.OutlookForWindows_8wekyb3d8bbwe!Microsoft.OutlookforWindows"
    },
    {
      "desktopAppId": "Microsoft.Office.WINWORD.EXE.15"
    },
    {
      "desktopAppId": "Microsoft.Office.EXCEL.EXE.15"
    },
    {
      "desktopAppId": "Microsoft.Office.POWERPNT.EXE.15"
    },
    {
      "packagedAppId": "Microsoft.MicrosoftTeams_8wekyb3d8bbwe!MSTeams"
    },
    {
      "packagedAppId": "Microsoft.WindowsNotepad_8wekyb3d8bbwe!App"
    },
    {
      "packagedAppId": "Microsoft.WindowsCalculator_8wekyb3d8bbwe!App"
    },
    {
      "desktopAppId": "Microsoft.Windows.ControlPanel"
    }
  ]
}
```

### Appliquer le JSON via GPO

!!! example "Pas a pas"

    **Etape 1** : Preparer le fichier JSON.

    Creez votre fichier JSON et enregistrez-le dans un partage reseau accessible par tous les utilisateurs, par exemple `\\serveur\partage\gpo\startmenu-win11.json`.

    **Etape 2** : Ouvrir la GPO ciblee.

    Dans la GPMC, ouvrez en edition la GPO qui gere vos utilisateurs (ou creez-en une dediee, par exemple `CFG-Utilisateurs-MenuDemarrer-Win11`).

    **Etape 3** : Naviguer vers le parametre.

    Developpez : **Configuration utilisateur** → **Strategies** → **Modeles d'administration** → **Menu Demarrer et barre des taches**

    **Etape 4** : Ouvrir le parametre.

    Double-cliquez sur **Configurer la disposition du menu Demarrer**.

    **Etape 5** : Activer et saisir le chemin.

    Selectionnez **Activer**, puis dans le champ de texte, entrez le chemin UNC complet vers votre fichier JSON :
    ```
    \\serveur\partage\gpo\startmenu-win11.json
    ```
    Cliquez sur **OK**.

    **Etape 6** : Forcer l'application et tester.

    Sur un poste Windows 11 de test, ouvrez une invite de commandes et executez `gpupdate /force`. Deconnectez puis reconnectez la session utilisateur. Le menu Demarrer doit afficher vos applications epinglees.

!!! warning "Cette GPO n'a aucun effet sur Windows 10"
    La GPO JSON est ignoree sur Windows 10. Elle ne cause pas d'erreur, elle est simplement sans effet. Si votre parc est mixte (Windows 10 et Windows 11), vous devez maintenir **deux GPO distinctes** : l'une avec le XML pour Windows 10, l'autre avec le JSON pour Windows 11. Utilisez des filtres WMI pour cibler chaque version de Windows.

!!! quote "En resume"
    - Le menu Demarrer Windows 11 est entierement different de celui de Windows 10 : l'ancien format XML est inoperant.
    - La nouvelle GPO utilise un fichier JSON qui liste les applications a epingler (`packagedAppId` ou `desktopAppId`).
    - Le chemin de la GPO est : Configuration utilisateur → Modeles d'administration → Menu Demarrer et barre des taches → Configurer la disposition du menu Demarrer.
    - Pour un parc mixte, maintenez deux GPO distinctes et utilisez des filtres WMI pour les cibler.

---

## Barre des taches de Windows 11 : nouveautes et restrictions

### Ce qui a disparu

Windows 11 a supprime plusieurs options de personnalisation de la barre des taches qui existaient depuis des annees. Certains administrateurs ont eu la mauvaise surprise de constater que des GPO qui fonctionnaient depuis Windows 7 n'avaient plus aucun effet.

La restriction la plus connue : **il n'est plus possible de deplacer la barre des taches en haut, a gauche ou a droite via GPO** (ni meme via le registre de maniere officielle). Windows 11 impose la barre en bas.

### Ce qui est nouveau

En contrepartie, Windows 11 introduit de nouvelles GPO pour controler les elements specifiques a son interface.

| Parametre | Chemin GPO | Nouveaute Win 11 ? |
|:----------|:-----------|:------------------:|
| Masquer le bouton "Vue des taches" | Config. util. → Menu Demarrer et barre des taches → Masquer le bouton Vue des taches | :material-check: Oui |
| Masquer les Widgets | Config. ordi. → Composants Windows → Widgets → Autoriser les widgets | :material-check: Oui |
| Masquer le bouton Chat (Teams) | Config. util. → Menu Demarrer et barre des taches → Masquer le bouton Chat | :material-check: Oui |
| Desactiver Copilot | Config. util. → Composants Windows → Copilot Windows → Desactiver Copilot Windows | :material-check: Oui |
| Verrouiller la barre des taches | Config. util. → Menu Demarrer et barre des taches → Verrouiller la barre des taches | :material-minus: Existait |
| Supprimer la zone de notification | Config. util. → Menu Demarrer et barre des taches → Supprimer la zone de notification | :material-minus: Existait |
| Masquer l'horloge | Config. util. → Menu Demarrer et barre des taches → Supprimer l'horloge | :material-minus: Existait |

!!! warning "La position de la barre est verrouilee"
    Sur Windows 11, la barre des taches ne peut officiellement etre placee qu'en bas. Les GPO qui tentaient de la deplacer via le registre (valeur `TaskbarSi` dans `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\StuckRects3`) ne fonctionnent plus de maniere fiable. Partez du principe que la barre est en bas et adaptez vos procedures de support en consequence.

### Comparaison Windows 10 vs Windows 11 pour la barre des taches

| Fonctionnalite | Windows 10 | Windows 11 |
|:---------------|:----------:|:----------:|
| Deplacer la barre (haut/gauche/droite) via GPO | :material-check-circle: Oui | :material-close-circle: Non |
| Masquer la barre des taches | :material-check-circle: Oui | :material-check-circle: Oui |
| Verrouiller la barre | :material-check-circle: Oui | :material-check-circle: Oui |
| Controler les icones systeme | :material-check-circle: Oui | :material-check-circle: Oui |
| Masquer le bouton "Vue des taches" | :material-close-circle: Non | :material-check-circle: Oui |
| Masquer les Widgets | :material-close-circle: Non | :material-check-circle: Oui |
| Masquer le bouton Chat Teams | :material-close-circle: Non | :material-check-circle: Oui |
| Desactiver Copilot | :material-close-circle: Non | :material-check-circle: Oui |

!!! quote "En resume"
    - Windows 11 supprime la possibilite de positionner la barre des taches autrement qu'en bas.
    - De nouveaux parametres GPO permettent de masquer : Vue des taches, Widgets, bouton Chat Teams et Copilot.
    - Les GPO de verrouillage et de masquage de la barre existantes continuent de fonctionner sans modification.

---

## Widgets et Copilot

### Les Widgets : gestion de l'information sur le bureau

Les Widgets sont le panneau lateral qui affiche la meteo, les actualites, les resultats sportifs et d'autres informations personnalisees. Dans un contexte professionnel, ils peuvent etre une source de distraction ou poser des questions de confidentialite (ils envoient des donnees a Microsoft).

Pour les desactiver :

| Element | Valeur |
|:--------|:-------|
| **Section** | Configuration ordinateur |
| **Chemin** | Strategies → Modeles d'administration → Composants Windows → Widgets |
| **Parametre** | Autoriser les widgets |
| **Action** | Desactiver |

!!! info "Configuration ordinateur, pas utilisateur"
    Ce parametre se trouve dans la section **Configuration ordinateur**, pas Configuration utilisateur. Cela signifie qu'il s'applique au poste, independamment de quel utilisateur est connecte. C'est le comportement souhaite en entreprise : les Widgets sont absents pour tout le monde.

### Copilot Windows : l'assistant IA integre

Copilot Windows est l'assistant IA integre a Windows 11 (apparu dans la version 23H2). Dans un contexte professionnel, sa desactivation peut etre requise pour des raisons de securite (envoi de donnees vers le cloud Microsoft) ou de conformite.

Pour le desactiver :

| Element | Valeur |
|:--------|:-------|
| **Section** | Configuration utilisateur |
| **Chemin** | Strategies → Modeles d'administration → Composants Windows → Copilot Windows |
| **Parametre** | Desactiver Copilot Windows |
| **Action** | Activer (activer le parametre "Desactiver Copilot") |

!!! warning "Attention au paradoxe activer/desactiver"
    La logique peut paraitre contre-intuitive : pour **desactiver** Copilot, il faut **Activer** le parametre intitule "Desactiver Copilot Windows". C'est la convention Microsoft pour les parametres de restriction : le parametre decrit ce qu'il fait quand il est active. Lisez toujours le nom du parametre et l'action souhaitee ensemble.

!!! warning "Windows 11 24H2 et versions ulterieures"
    Le **Copilot Windows** historique est progressivement remplace par une experience plus proche d'une application / web app. La GPO **Desactiver Copilot Windows** reste pertinente sur 23H2, mais sur 24H2+ il faut aussi verifier la gestion de l'application Copilot, du Microsoft Store et des experiences M365 associees.

### Desactiver Copilot et les Widgets en une seule GPO

!!! example "Pas a pas"

    **Etape 1** : Creer une GPO dediee.

    Dans la GPMC, creez une GPO nommee `CFG-Ordinateurs-Win11-Restrictions` et liez-la a l'OU qui contient vos postes Windows 11.

    **Etape 2** : Desactiver les Widgets.

    Editez la GPO et naviguez vers :
    **Configuration ordinateur** → **Strategies** → **Modeles d'administration** → **Composants Windows** → **Widgets**

    Double-cliquez sur **Autoriser les widgets** et selectionnez **Desactiver**.

    **Etape 3** : Desactiver Copilot.

    Naviguez vers :
    **Configuration utilisateur** → **Strategies** → **Modeles d'administration** → **Composants Windows** → **Copilot Windows**

    Double-cliquez sur **Desactiver Copilot Windows** et selectionnez **Activer**.

    **Etape 4** : Masquer le bouton Chat Teams.

    Naviguez vers :
    **Configuration utilisateur** → **Strategies** → **Modeles d'administration** → **Menu Demarrer et barre des taches**

    Double-cliquez sur **Masquer le bouton Chat** et selectionnez **Activer**.

    **Etape 5** : Appliquer et tester.

    Sur un poste Windows 11 de test, executez `gpupdate /force`, puis verifiez que les Widgets, Copilot et le bouton Chat ont disparu de la barre des taches.

!!! tip "Recall et autres fonctionnalites IA"
    Windows 11 24H2 a introduit **Recall** (fonctionnalite de capture d'ecran automatique pour la recherche). Elle aussi peut etre desactivee via GPO : cherchez le parametre **Desactiver la fonctionnalite Recall** dans **Composants Windows → Recall de Windows**. Cette fonctionnalite etant recente, verifiez que vos modeles ADMX sont a jour avant de chercher ce parametre.

!!! quote "En resume"
    - Les Widgets se desactivent via Configuration **ordinateur** → Composants Windows → Widgets → Autoriser les widgets → Desactiver.
    - Copilot se desactive via Configuration **utilisateur** → Composants Windows → Copilot Windows → Desactiver Copilot Windows → Activer.
    - Regroupez ces restrictions dans une GPO dediee liee aux postes Windows 11 pour garder vos configurations lisibles.

---

## Snap Layouts et multitache

### Qu'est-ce que Snap Layouts ?

Snap Layouts est la fonctionnalite qui apparait quand vous survolez le bouton "Agrandir" d'une fenetre. Windows 11 propose alors plusieurs dispositions predefinies pour organiser plusieurs fenetres cote a cote.

C'est une fonctionnalite de productivite appreciee par beaucoup d'utilisateurs. Mais dans certains environnements (postes kiosque, applications plein ecran, interfaces simplifiees), vous pouvez vouloir la desactiver.

### Desactiver Snap Layouts

| Element | Valeur |
|:--------|:-------|
| **Section** | Configuration utilisateur |
| **Chemin** | Strategies → Modeles d'administration → Systeme → Fonctionnalites Snap |
| **Parametre** | Desactiver Snap |
| **Action** | Activer |

!!! info "Snap existait avant Windows 11"
    La fonctionnalite Snap (ancrage de fenetres) existait depuis Windows 7. Seul le volet "Snap Layouts" (le menu de disposition au survol du bouton Agrandir) est nouveau dans Windows 11. Le parametre GPO "Desactiver Snap" desactive l'ensemble de la fonctionnalite, y compris le glisser-deposer vers les bords de l'ecran.

### Autres parametres de multitache

Windows 11 offre quelques parametres supplementaires pour controler le comportement du multitache :

| Parametre | Chemin | Effet |
|:----------|:-------|:------|
| Desactiver Snap | Config. util. → Systeme → Fonctionnalites Snap | Desactive entierement le systeme d'ancrage de fenetres |
| Masquer le bouton "Vue des taches" | Config. util. → Menu Demarrer et barre des taches | Supprime le bouton Vue des taches de la barre |
| Desactiver les bureaux virtuels | Config. util. → Systeme | Empeche la creation de bureaux virtuels supplementaires |

!!! tip "Postes kiosque et fenetrage"
    Si vous gerez des postes kiosque (caisse, borne d'accueil, poste de production), desactiver Snap et la Vue des taches reduit les risques qu'un utilisateur reorganise accidentellement l'affichage ou accede a des fenetres cachees.

!!! quote "En resume"
    - Snap Layouts est une nouveaute Windows 11 qui peut etre desactivee via : Config. utilisateur → Systeme → Fonctionnalites Snap → Desactiver Snap → Activer.
    - Combinez cette restriction avec le masquage de la Vue des taches pour les postes kiosque.
    - La desactivation de Snap est globale : elle supprime tout l'ancrage de fenetres, pas seulement le menu de disposition.

---

## OneDrive et Microsoft Teams integres

### OneDrive Known Folder Move

Windows 11 pousse activement les utilisateurs a activer la **sauvegarde automatique des dossiers** Bureau, Documents et Images vers OneDrive (fonctionnalite "Known Folder Move" ou KFM). Cette redirection peut etre configuree, forcee ou bloquee via GPO.

!!! info "OneDrive a ses propres ADMX"
    Les parametres OneDrive ne sont pas dans les modeles Windows standard. Ils font partie du **client OneDrive** (OneDrive.exe). Microsoft fournit des modeles ADMX specifiques a OneDrive sur sa documentation officielle. Cherchez "OneDrive ADMX" dans la documentation Microsoft.

Les parametres KFM les plus courants se trouvent sous :
**Configuration ordinateur** → **Modeles d'administration** → **OneDrive**

| Parametre KFM | Effet |
|:--------------|:------|
| Desactiver la migration KFM | Empeche OneDrive de proposer la sauvegarde automatique des dossiers |
| Forcer la migration KFM sans invite | Redirige silencieusement Bureau/Documents/Images vers OneDrive |
| Inviter l'utilisateur a activer KFM | Affiche une notification pour inviter l'utilisateur a activer KFM |
| Empecher l'utilisateur de modifier KFM | Verouille la configuration KFM, l'utilisateur ne peut plus la modifier |

!!! warning "KFM modifie les chemins des dossiers"
    Quand KFM est active, `C:\Users\jean\Desktop` devient un lien symbolique vers `C:\Users\jean\OneDrive - Entreprise\Desktop`. Ca peut surprendre certaines applications qui utilisent des chemins codifes en dur. Testez avant de deployer a grande echelle.

### Microsoft Teams integre a Windows 11

Windows 11 a integre une version allégée de Microsoft Teams directement dans la barre des taches (le bouton Chat). Si votre organisation utilise une autre solution de communication, ou la version complete de Teams via Microsoft 365, ce bouton integre est superflu.

Pour masquer le bouton Chat Teams :

| Element | Valeur |
|:--------|:-------|
| **Section** | Configuration utilisateur |
| **Chemin** | Strategies → Modeles d'administration → Menu Demarrer et barre des taches |
| **Parametre** | Masquer le bouton Chat |
| **Action** | Activer |

!!! tip "Teams integre vs Teams Microsoft 365"
    L'application Teams "integree" a Windows 11 est distincte de l'application Teams Microsoft 365 que vous deployez via votre abonnement. Masquer le bouton Chat n'affecte pas du tout Teams Microsoft 365. Les deux coexistent independamment.

!!! quote "En resume"
    - OneDrive Known Folder Move (KFM) permet de forcer la sauvegarde des dossiers utilisateur vers le cloud, configurable via des ADMX OneDrive specifiques.
    - Le bouton Chat Teams integre peut etre masque via : Config. utilisateur → Menu Demarrer et barre des taches → Masquer le bouton Chat → Activer.
    - Ces deux fonctionnalites sont independantes l'une de l'autre et peuvent etre gerees separement.

---

## Compatibilite : les GPO Windows 10 sur Windows 11

### La matrice de compatibilite

Voici un tableau de reference pour les parametres les plus courants, indiquant leur comportement lorsqu'ils sont appliques sur Windows 11.

| Domaine | Parametre | Fonctionne sur Win 11 ? | Remarque |
|:--------|:----------|:-----------------------:|:---------|
| **Menu Demarrer** | Configurer la disposition (XML) | :material-close-circle: Non | Ignore silencieusement. Utiliser le parametre JSON a la place |
| **Menu Demarrer** | Supprimer le menu Demarrer et la barre des taches | :material-check-circle: Oui | Fonctionne sans modification |
| **Barre des taches** | Verrouiller la barre des taches | :material-check-circle: Oui | Fonctionne sans modification |
| **Barre des taches** | Positionner la barre (haut/gauche/droite) | :material-close-circle: Non | La barre est toujours en bas dans Win 11 |
| **Barre des taches** | Masquer les icones de la zone de notification | :material-check-circle: Oui | Fonctionne sans modification |
| **Bureau** | Fond d'ecran | :material-check-circle: Oui | Fonctionne sans modification |
| **Bureau** | Ecran de verrouillage | :material-check-circle: Oui | Fonctionne sans modification |
| **Securite** | Strategie de mot de passe | :material-check-circle: Oui | Inchange |
| **Securite** | Audit et journalisation | :material-check-circle: Oui | Inchange |
| **Restrictions logicielles** | AppLocker / WDAC | :material-check-circle: Oui | Inchange |
| **Windows Update** | WSUS et WUfB | :material-check-circle: Oui | Inchange (les noms des parametres sont identiques) |
| **Internet Explorer** | Tous les parametres IE | :material-close-circle: Non | IE n'existe plus dans Win 11, utilisez les parametres Edge |
| **Edge** | Parametres Microsoft Edge | :material-check-circle: Oui | Utiliser les ADMX Edge separement |
| **Imprimantes** | Deploiement d'imprimantes | :material-check-circle: Oui | Inchange |
| **Scripts** | Scripts de demarrage / connexion | :material-check-circle: Oui | Inchange |
| **Preferences GPO** | Variables d'environnement, registre, fichiers | :material-check-circle: Oui | Inchange |
| **Nouvelles fonctionnalites** | Widgets, Copilot, Snap Layouts | :material-minus-circle: Nouveau | Necessitent les modeles ADMX Windows 11 |

### Le cas special d'Internet Explorer

Windows 11 ne contient plus Internet Explorer. Toutes les GPO relatives a IE (securite des zones, parametres avances, page d'accueil via IE...) sont sans effet sur Windows 11.

Si vous avez des applications qui dependent d'IE, elles passent par le **mode IE d'Edge** (Internet Explorer Mode). Ce mode a ses propres parametres GPO, dans les modeles ADMX de Microsoft Edge.

!!! warning "Ne laissez pas des GPO IE orphelines"
    Des GPO IE appliquees sur des postes Windows 11 ne causent pas d'erreur, mais elles occupent inutilement du temps de traitement et surtout de la bande passante reseau a chaque connexion. Apres la migration Windows 11, faites le menage dans vos GPO et desactivez ou supprimez celles qui ne visent que IE.

!!! quote "En resume"
    - La grande majorite des GPO Windows 10 fonctionnent sur Windows 11 sans modification.
    - Deux exceptions majeures : le menu Demarrer XML (nouveau format JSON requis) et la position de la barre des taches.
    - Internet Explorer n'existe plus : toutes les GPO IE sont inoperantes sur Windows 11.
    - Les nouvelles fonctionnalites (Widgets, Copilot, Snap Layouts) necessitent les modeles ADMX Windows 11.

---

## Verifier la version des ADMX appliquees

### Pourquoi cette verification est importante

Avant de commencer a configurer des parametres Windows 11, il faut s'assurer que vos modeles ADMX sont bien a jour. Si un parametre n'apparait pas dans votre editeur GPO, la cause est presque toujours des ADMX obsoletes.

Ce script PowerShell plus complet permet de diagnostiquer l'etat de votre magasin central et de detecter si les modeles Windows 11 sont en place :

```powershell
# Diagnose the ADMX Central Store version and Windows 11 readiness
# Run this script from a domain-joined machine with read access to SYSVOL

$domain        = $env:USERDNSDOMAIN
$centralStore  = "\\$domain\SYSVOL\$domain\Policies\PolicyDefinitions"

Write-Host "=== ADMX Central Store Diagnostics ===" -ForegroundColor Cyan
Write-Host "Domain   : $domain"
Write-Host "Path     : $centralStore"
Write-Host ""

# Check if the central store exists
if (-not (Test-Path $centralStore)) {
    Write-Warning "Central Store not found. Local templates may be in use."
    Write-Host "Expected path: $centralStore" -ForegroundColor Red
    exit 1
}

# Count total ADMX files
$allAdmx = Get-ChildItem $centralStore -Filter "*.admx" -ErrorAction Stop
Write-Host "Total ADMX files : $($allAdmx.Count)" -ForegroundColor Green

# Show the 10 most recently updated files
Write-Host ""
Write-Host "--- Most recently updated ADMX templates ---" -ForegroundColor Yellow
$allAdmx |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 10 |
    Format-Table Name, LastWriteTime -AutoSize

# Check for Windows 11-specific templates
Write-Host "--- Windows 11 template availability ---" -ForegroundColor Yellow
$win11Templates = @(
    "WindowsAI.admx",
    "NewsAndInterests.admx",
    "Taskbar.admx",
    "SnapLayout.admx"
)

foreach ($template in $win11Templates) {
    $path = Join-Path $centralStore $template
    if (Test-Path $path) {
        $file = Get-Item $path
        Write-Host "  [OK] $template  (modified: $($file.LastWriteTime.ToString('yyyy-MM-dd')))" -ForegroundColor Green
    } else {
        Write-Host "  [MISSING] $template" -ForegroundColor Red
    }
}

# Check French language files
Write-Host ""
Write-Host "--- French (fr-FR) language files ---" -ForegroundColor Yellow
$frStore = Join-Path $centralStore "fr-FR"
if (Test-Path $frStore) {
    $admlCount = (Get-ChildItem $frStore -Filter "*.adml").Count
    Write-Host "  fr-FR folder found with $admlCount .adml files" -ForegroundColor Green
} else {
    Write-Warning "  fr-FR language folder not found in Central Store"
}
```

### Interpreter les resultats

| Resultat | Signification | Action |
|:---------|:--------------|:-------|
| Tous les modeles Win 11 sont `[OK]` | Votre magasin est pret pour Windows 11 | Aucune action requise |
| Un ou plusieurs modeles sont `[MISSING]` | Les ADMX Windows 11 ne sont pas en place | Telechargez et installez les modeles Microsoft |
| La date des fichiers est anterieure a 2022 | Les modeles sont ceux de Windows 10 | Mettez a jour vers les modeles Windows 11 |
| Le dossier `fr-FR` est manquant | Les descriptions seront en anglais | Copiez les fichiers `.adml` du dossier `fr-FR` |
| Le magasin central n'existe pas | Vous utilisez les modeles locaux | Creez le magasin central (bonne pratique) |

!!! quote "En resume"
    - Verifiez l'etat de votre magasin ADMX avant toute configuration Windows 11.
    - Le script PowerShell detaille ci-dessus diagnostique automatiquement la presence des modeles cles.
    - Les fichiers a surveiller : `WindowsAI.admx`, `NewsAndInterests.admx`, `Taskbar.admx`, `SnapLayout.admx`.
    - Un magasin central manquant est un probleme de fond a resoudre independamment.

---

## Checklist de migration Windows 10 vers Windows 11

### Avant de deployer Windows 11 en production

Une migration reussie commence par un audit de l'existant. Voici la checklist pratique pour les administrateurs qui preparent le basculement de leur parc vers Windows 11.

#### :material-checkbox-marked-outline: Phase 1 — Preparation des outils d'administration

- [ ] Telecharger et installer les modeles ADMX Windows 11 dans le magasin central SYSVOL
- [ ] Verifier la presence des fichiers `WindowsAI.admx`, `NewsAndInterests.admx`, `Taskbar.admx`
- [ ] Verifier les fichiers de langue `fr-FR` correspondants
- [ ] Executer le script de diagnostic PowerShell pour confirmer la readiness du magasin central
- [ ] Mettre a jour la console GPMC si necessaire (Windows Server 2019/2022 ou RSAT recents)

#### :material-checkbox-marked-outline: Phase 2 — Audit des GPO existantes

- [ ] Identifier toutes les GPO qui configurent le menu Demarrer (format XML Windows 10)
- [ ] Identifier toutes les GPO qui configurent la position de la barre des taches
- [ ] Identifier toutes les GPO relatives a Internet Explorer
- [ ] Lister les GPO qui utilisent des filtres WMI bases sur la version de Windows

#### :material-checkbox-marked-outline: Phase 3 — Creation des nouvelles GPO Windows 11

- [ ] Creer une GPO de menu Demarrer Windows 11 (format JSON) et la lier aux postes Windows 11
- [ ] Creer une GPO de restrictions Windows 11 (Widgets, Copilot, Chat Teams si necessaire)
- [ ] Ajouter des filtres WMI aux GPO Windows 10 pour exclure les postes Windows 11
- [ ] Ajouter des filtres WMI aux nouvelles GPO Windows 11 pour cibler uniquement Windows 11

#### :material-checkbox-marked-outline: Phase 4 — Tests sur poste pilote

- [ ] Deployer Windows 11 sur un ou plusieurs postes pilotes representatifs
- [ ] Appliquer toutes les GPO et executer `gpupdate /force` + `gpresult /h rapport.html`
- [ ] Verifier visuellement le menu Demarrer, la barre des taches, l'absence de Copilot/Widgets
- [ ] Tester les applications metier sur le poste pilote
- [ ] Valider les scripts de demarrage et de connexion
- [ ] Verifier la connexion a l'imprimante reseau et aux partages
- [ ] Tester le verrouillage de session et la politique de mot de passe

#### :material-checkbox-marked-outline: Phase 5 — Deploiement progressif

- [ ] Deployer sur un groupe pilote elargi (5 a 10% du parc)
- [ ] Surveiller les evenements GPO dans l'observateur d'evenements (canal Operational)
- [ ] Collecter les retours utilisateurs et corriger les anomalies
- [ ] Planifier le deploiement general par vagues (par site, par departement...)
- [ ] Mettre a jour la documentation des procedures de support

!!! tip "Le filtre WMI pour cibler Windows 11"
    Pour que vos GPO Windows 11 ne s'appliquent qu'aux postes Windows 11 (et pas aux Windows 10), utilisez ce filtre WMI :

    ```sql
    SELECT * FROM Win32_OperatingSystem
    WHERE Version LIKE "10.0.2%" AND ProductType = "1"
    ```

    Le numero de build de Windows 11 commence par `10.0.2` (ex. : 10.0.22000 pour la version initiale, 10.0.22621 pour 22H2, 10.0.22631 pour 23H2). Windows 10 utilise des builds `10.0.1xxxx`. Ce filtre discrimine les deux de maniere fiable.

!!! warning "Ne supprimez pas les GPO Windows 10 trop tot"
    Pendant la periode de coexistence (Windows 10 et Windows 11 dans le meme parc), maintenez vos deux sets de GPO en parallele. Ne supprimez les GPO Windows 10 qu'une fois que le dernier poste Windows 10 a ete migre ou retire du domaine.

!!! quote "En resume"
    - Une migration GPO reussie se prepare en cinq phases : outils, audit, creation, tests pilote, deploiement progressif.
    - Les filtres WMI permettent de cibler selectivement Windows 10 et Windows 11 avec des GPO differentes.
    - Ne supprimez pas les GPO Windows 10 tant que des postes Windows 10 sont encore dans le parc.
    - Utilisez `gpresult /h rapport.html` sur un poste pilote pour valider l'application correcte de toutes les GPO.

---

## Resume du chapitre

Vous avez maintenant une vision complete des differences entre GPO Windows 10 et Windows 11, et les outils pour gerer les deux en parallele. Voici le tableau de synthese des points cles.

| Sujet | Situation sur Windows 11 | Action requise |
|:------|:------------------------|:---------------|
| **Modeles ADMX** | De nouveaux fichiers sont necessaires pour les fonctionnalites Win 11 | Telecharger et copier dans le magasin central SYSVOL |
| **Menu Demarrer** | La GPO XML Windows 10 est ignoree | Creer une GPO JSON via le nouveau parametre dedié |
| **Barre des taches (position)** | Blocage en bas, aucune GPO ne peut la deplacer | Adapter les procedures support, pas d'alternative GPO |
| **Barre des taches (boutons)** | Nouveaux parametres pour Widgets, Copilot, Chat | Configurer via les nouvelles GPO si besoin |
| **Widgets** | Fonctionnalite nouvelle avec parametre GPO dedie | Desactiver via Config. ordi. → Composants Windows → Widgets |
| **Copilot** | Fonctionnalite nouvelle avec parametre GPO dedie | Desactiver via Config. util. → Composants Windows → Copilot Windows |
| **Snap Layouts** | Fonctionnalite nouvelle desactivable par GPO | Desactiver via Config. util. → Systeme → Fonctionnalites Snap |
| **Internet Explorer** | N'existe plus dans Windows 11 | Desactiver les GPO IE, migrer vers les GPO Edge |
| **Securite, scripts, impression** | Inchanges, compatibilite totale | Aucune action requise |
| **Windows Update / WSUS** | Inchange, compatibilite totale | Aucune action requise |
| **Filtres WMI** | Indispensables en parc mixte | Creer des filtres separant Win 10 et Win 11 |

### :material-school: Les trois reflexes a retenir

**Reflexe 1** : Avant de configurer quoi que ce soit de Windows 11, verifiez que vos modeles ADMX sont a jour.

**Reflexe 2** : Menu Demarrer Windows 11 = JSON. Menu Demarrer Windows 10 = XML. Les deux ne sont pas interchangeables.

**Reflexe 3** : Utilisez des filtres WMI pour cibler precisement chaque version de Windows. Ne laissez pas des GPO s'appliquer sur les mauvais systemes.


!!! quote "En résumé"
    - Modeles ADMX : Telecharger et copier dans le magasin central SYSVOL.
    - Menu Demarrer : Creer une GPO JSON via le nouveau parametre dedié.
    - Barre des taches (position) : Adapter les procedures support, pas d'alternative GPO.
    - Barre des taches (boutons) : Configurer via les nouvelles GPO si besoin.
    - Voici le tableau de synthese des points cles.
---
!!! info "Chapitre suivant : Mini-projets GPO"
    Dans le **chapitre 15 — Mini-projets GPO**, vous mettrez en pratique tout ce que vous avez appris dans ce livre a travers des scenarios complets et realistes : deploiement d'un poste standardise de A a Z, securisation d'une salle de formation, configuration d'une station kiosque, et gestion d'un parc mixte Windows 10 / Windows 11. Des etapes guidees, des GPO a creer, des scripts a executer. Le meilleur moyen de consolider vos connaissances.
