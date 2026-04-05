---
description: "Personnaliser le bureau, la barre des taches et le menu Demarrer de Windows via les strategies de groupe."
tags:
  - gpo
  - debutant
  - bureau
---

# Gerer le bureau et le menu Demarrer

!!! abstract "Ce que vous allez apprendre"
    - Imposer un fond d'ecran et un ecran de verrouillage uniformes a tous les postes
    - Verrouiller et configurer la barre des taches via GPO
    - Deployer un menu Demarrer personnalise sur Windows 10 (XML) et Windows 11 (JSON)
    - Controler les icones du bureau, l'economiseur d'ecran et les themes
    - Creer une GPO complete qui standardise l'environnement de bureau

---

## Pourquoi gerer le bureau par GPO ?

Imaginez une ecole ou chaque eleve porterait les vetements qu'il veut. Certains arriveraient en costume, d'autres en pyjama, et le jour de la photo de classe, ce serait le chaos. C'est pour ca que beaucoup d'ecoles imposent un **uniforme** : tout le monde a la meme base, c'est plus simple a gerer, et on evite les distractions.

Le bureau Windows, c'est pareil. Sans GPO, chaque employe personnalise son poste a sa sauce. Jean-Pierre met une photo de son chat en fond d'ecran, Sylvie installe 47 icones sur le bureau, et Kevin change les couleurs du systeme en vert fluo. Quand le support informatique doit intervenir sur un poste, il perd du temps a chercher ses reperes dans un environnement qu'il ne reconnait pas.

Avec une GPO de personnalisation du bureau, vous obtenez :

- **Uniformite visuelle** : tous les postes ont le meme aspect professionnel
- **Image de marque** : le fond d'ecran affiche le logo de l'entreprise
- **Securite** : l'economiseur d'ecran se verrouille apres quelques minutes d'inactivite
- **Support simplifie** : le technicien retrouve le meme environnement sur chaque poste
- **Moins de distractions** : les utilisateurs se concentrent sur leur travail, pas sur la decoration

!!! tip "Ce n'est pas que cosmetique"
    Gerer le bureau par GPO n'est pas une lubie de maniaque du controle. Le verrouillage automatique de l'ecran est une **exigence de securite** dans la plupart des referentiels (ISO 27001, ANSSI, RGPD). Un poste non verrouille dans un open space, c'est une porte ouverte sur les donnees de l'entreprise.

!!! quote "En resume"
    - Sans GPO, chaque poste a un bureau different, ce qui complique le support et l'image de marque.
    - Les GPO de bureau permettent d'uniformiser l'apparence, renforcer la securite et simplifier la maintenance.
    - Le verrouillage automatique de l'ecran n'est pas optionnel, c'est une exigence de securite.

---

## Fond d'ecran et ecran de verrouillage

Ce sont les deux parametres de personnalisation les plus courants. Ils sont souvent confondus, mais ce sont **deux reglages distincts** dans des emplacements differents.

- Le **fond d'ecran** (ou papier peint) est l'image affichee sur le bureau Windows, derriere les icones.
- L'**ecran de verrouillage** est l'image affichee quand le poste est verrouille (avant la saisie du mot de passe).

### :material-image: Preparer l'image

Avant de configurer quoi que ce soit dans la GPO, il faut preparer l'image et la rendre **accessible a tous les postes du reseau**.

L'image doit etre stockee sur un **partage reseau** accessible en lecture par tous les utilisateurs. Si vous placez l'image sur le disque local d'un seul PC, les autres postes ne pourront pas y acceder. Logique, mais c'est une erreur tres frequente.

Le chemin UNC doit ressembler a :

```
\\serveur\partage\images\fond-ecran-entreprise.jpg
```

!!! warning "Permissions du partage"
    Le groupe **Utilisateurs du domaine** (ou **Utilisateurs authentifies**) doit avoir les permissions de **lecture** sur le partage et sur le dossier NTFS. Si un seul poste ne peut pas lire l'image, il affichera un fond d'ecran noir ou le fond par defaut de Windows.

    Pour tester : connectez-vous sur un poste de test avec un compte utilisateur standard et essayez d'ouvrir le chemin UNC dans l'Explorateur de fichiers. Si l'image s'affiche, c'est bon.

Voici les formats d'image pris en charge par Windows :

| Format | Extension | Recommande ? | Remarques |
|:------:|:---------:|:------------:|:---------:|
| JPEG | `.jpg` / `.jpeg` | :material-check-circle: Oui | Le plus courant, bon compromis taille/qualite |
| PNG | `.png` | :material-check-circle: Oui | Meilleure qualite, fichier plus volumineux |
| BMP | `.bmp` | :material-close-circle: Non | Fichier tres volumineux, aucun avantage |
| GIF | `.gif` | :material-close-circle: Non | Qualite insuffisante pour un fond d'ecran |
| TIFF | `.tif` / `.tiff` | :material-close-circle: Non | Non supporte pour le fond d'ecran par GPO |

Et les resolutions recommandees :

| Resolution | Ratio | Usage typique |
|:----------:|:-----:|:-------------:|
| 1920 x 1080 | 16:9 | Ecrans Full HD (le standard actuel) |
| 2560 x 1440 | 16:9 | Ecrans QHD |
| 3840 x 2160 | 16:9 | Ecrans 4K |
| 1920 x 1200 | 16:10 | Ecrans professionnels au format 16:10 |

!!! tip "Une seule image pour tous"
    Utilisez une image en **1920 x 1080** minimum. Windows l'etirera automatiquement sur les ecrans plus grands. C'est mieux qu'une image trop petite qui sera pixelisee. Si vous avez un parc heterogene, une image en 3840 x 2160 sera reduite sur les petits ecrans sans perte de qualite (mais le fichier sera plus lourd a telecharger via le reseau).

### :material-wallpaper: Configurer le fond d'ecran

Le fond d'ecran se configure dans la section **utilisateur** de la GPO.

| Element | Valeur |
|---------|--------|
| **Chemin GPO** | Configuration utilisateur → Strategies → Modeles d'administration → Bureau → Bureau |
| **Parametre** | Papier peint du Bureau |
| **Action** | Activer |
| **Champ "Nom du papier peint"** | Chemin UNC vers l'image (ex. : `\\serveur\partage\images\fond-ecran.jpg`) |
| **Champ "Style du papier peint"** | Remplissage (recommande pour les ecrans modernes) |

!!! example "Pas a pas"

    **Etape 1** : Ouvrir la console GPMC.

    Appuyez sur ++win+r++, tapez `gpmc.msc` et appuyez sur ++enter++.

    **Etape 2** : Creer ou modifier la GPO.

    Si vous suivez ce chapitre dans l'ordre, creez une nouvelle GPO nommee `CFG-Utilisateurs-EnvironnementBureau` dans le conteneur **Objets de strategie de groupe**.

    **Etape 3** : Modifier la GPO.

    Clic droit sur la GPO > **Modifier...**

    **Etape 4** : Naviguer vers le parametre.

    Developpez : **Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Bureau** > **Bureau**

    **Etape 5** : Configurer le fond d'ecran.

    1. Double-cliquez sur **Papier peint du Bureau**
    2. Selectionnez **Active**
    3. Dans le champ **Nom du papier peint**, entrez le chemin UNC :
       `\\serveur\partage\images\fond-ecran-entreprise.jpg`
    4. Dans la liste deroulante **Style du papier peint**, selectionnez **Remplissage**
    5. Cliquez **OK**

Les differents styles de papier peint fonctionnent ainsi :

| Style | Comportement |
|:-----:|:------------:|
| Centre | L'image est placee au centre de l'ecran, sans redimensionnement |
| Mosaique | L'image est repetee pour remplir l'ecran |
| Etirement | L'image est etiree pour couvrir tout l'ecran (peut deformer) |
| Remplissage | L'image est agrandie proportionnellement pour remplir l'ecran (recommande) |
| Ajustement | L'image est reduite proportionnellement pour tenir entierement dans l'ecran |

### :material-lock-outline: Configurer l'ecran de verrouillage

L'ecran de verrouillage se configure dans la section **ordinateur** de la GPO. C'est une difference importante : le fond d'ecran est un parametre **utilisateur**, l'ecran de verrouillage est un parametre **ordinateur**.

| Element | Valeur |
|---------|--------|
| **Chemin GPO** | Configuration ordinateur → Strategies → Modeles d'administration → Panneau de configuration → Personnalisation |
| **Parametre** | Forcer une image d'ecran de verrouillage et d'ouverture de session par defaut |
| **Action** | Activer |
| **Champ "Chemin de l'image"** | Chemin local ou UNC vers l'image |

!!! example "Pas a pas"

    **Etape 1** : Dans l'editeur de GPO (meme GPO que precedemment), naviguer vers :

    **Configuration ordinateur** > **Strategies** > **Modeles d'administration** > **Panneau de configuration** > **Personnalisation**

    **Etape 2** : Configurer l'ecran de verrouillage.

    1. Double-cliquez sur **Forcer une image d'ecran de verrouillage et d'ouverture de session par defaut**
    2. Selectionnez **Active**
    3. Dans le champ **Chemin de l'image de l'ecran de verrouillage**, entrez :
       `\\serveur\partage\images\ecran-verrouillage.jpg`
    4. Cliquez **OK**

    **Etape 3** (optionnel) : Empecher la modification de l'ecran de verrouillage.

    Dans le meme dossier **Personnalisation**, double-cliquez sur **Empecher la modification de l'image de l'ecran de verrouillage**, selectionnez **Active**, puis cliquez **OK**. Cela empeche les utilisateurs de remplacer l'image que vous avez imposee.

!!! warning "Chemin local vs chemin UNC pour l'ecran de verrouillage"
    Contrairement au fond d'ecran, l'ecran de verrouillage peut poser probleme avec un chemin UNC si le poste n'a pas encore de connexion reseau au moment de l'affichage (par exemple avant l'ouverture de session). Dans ce cas, copiez l'image en local sur chaque poste via un script de demarrage ou une preference de GPO, puis utilisez un chemin local comme `C:\Windows\Web\Wallpaper\entreprise\verrouillage.jpg`.

!!! quote "En resume"
    - Le **fond d'ecran** est un parametre utilisateur : Configuration utilisateur → Bureau → Bureau → Papier peint du Bureau.
    - L'**ecran de verrouillage** est un parametre ordinateur : Configuration ordinateur → Personnalisation → Forcer une image d'ecran de verrouillage.
    - L'image doit etre accessible en lecture par tous les utilisateurs via un chemin UNC.
    - Utilisez le format JPEG ou PNG en 1920 x 1080 minimum.
    - Testez toujours l'acces au partage avec un compte utilisateur standard avant de deployer.

---

## Barre des taches

La barre des taches est la bande horizontale en bas de l'ecran. Elle contient le bouton Demarrer, les icones epinglees, la zone de notification et l'horloge. Par GPO, on peut verrouiller sa configuration pour empecher les utilisateurs de la modifier.

### :material-pin: Les parametres essentiels

Tous les parametres de la barre des taches se trouvent dans :

**Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Menu Demarrer et barre des taches**

Voici les plus utiles :

| Parametre | Effet | Recommandation |
|:---------:|:-----:|:--------------:|
| Verrouiller la barre des taches | Empeche le deplacement et le redimensionnement | :material-check-circle: Activer |
| Supprimer l'acces au menu contextuel de la barre des taches | Desactive le clic droit sur la barre des taches | :material-check-circle: Activer |
| Masquer la zone de notification | Cache toutes les icones de la zone de notification | :material-close-circle: A eviter en general |
| Masquer le bouton Affichage des taches | Supprime le bouton "Task View" (les bureaux virtuels) | :material-check-circle: Activer si non utilise |
| Masquer l'icone des Personnes de la barre des taches | Supprime l'icone "Contacts" | :material-check-circle: Activer (rarement utile en entreprise) |
| Supprimer les notifications et le Centre de notifications | Cache le Centre de notifications | Selon le besoin |
| Ne pas autoriser l'epinglage d'applications sur la barre des taches | Empeche d'ajouter ou retirer des raccourcis | Selon le besoin |

### :material-cog: Configurer la barre des taches

!!! example "Pas a pas"

    Dans l'editeur de GPO, naviguez vers :

    **Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Menu Demarrer et barre des taches**

    **Parametre 1 : Verrouiller la barre des taches**

    1. Double-cliquez sur **Verrouiller la barre des taches**
    2. Selectionnez **Active**
    3. Cliquez **OK**

    **Parametre 2 : Supprimer le menu contextuel**

    1. Double-cliquez sur **Supprimer l'acces au menu contextuel de la barre des taches**
    2. Selectionnez **Active**
    3. Cliquez **OK**

    **Parametre 3 : Masquer Affichage des taches**

    1. Double-cliquez sur **Masquer le bouton Affichage des taches**
    2. Selectionnez **Active**
    3. Cliquez **OK**

    **Parametre 4 : Masquer l'icone Personnes**

    1. Double-cliquez sur **Masquer l'icone des Personnes de la barre des taches**
    2. Selectionnez **Active**
    3. Cliquez **OK**

!!! info "Windows 10 vs Windows 11"
    Certains de ces parametres n'existent plus sous Windows 11 car Microsoft a retire les fonctionnalites correspondantes. Par exemple, l'icone **Personnes** n'existe plus sous Windows 11, et la barre des taches ne peut plus etre deplacement sur les cotes de l'ecran. La GPO reste sans effet dans ce cas — elle ne provoque pas d'erreur, elle est simplement ignoree.

!!! quote "En resume"
    - Les parametres de la barre des taches se trouvent dans Configuration utilisateur → Menu Demarrer et barre des taches.
    - **Verrouiller la barre des taches** et **supprimer le menu contextuel** sont les deux parametres les plus courants en entreprise.
    - Un parametre qui n'existe plus dans la version de Windows du poste est simplement ignore, sans erreur.

---

## Menu Demarrer : layout Windows 10

Le menu Demarrer de Windows 10 est compose de **tuiles** (les fameux carres colores). Par GPO, on peut imposer un agencement precis de ces tuiles pour que tous les utilisateurs aient les memes applications epinglees au meme endroit.

C'est comme un plan de table dans un restaurant : au lieu de laisser chaque convive s'asseoir ou il veut, on decide a l'avance qui va ou. Le resultat est ordonne et previsible.

### :material-file-xml-box: Le principe : un fichier XML

Sous Windows 10, le layout du menu Demarrer est defini dans un **fichier XML**. La methode est la suivante :

1. **Personnaliser** manuellement le menu Demarrer sur un poste de reference
2. **Exporter** la configuration dans un fichier XML
3. **Deployer** ce fichier XML via GPO

### :material-export: Exporter le layout

Sur le poste de reference (un PC Windows 10 configure comme vous le souhaitez) :

1. Personnalisez le menu Demarrer : ajoutez, supprimez et reorganisez les tuiles
2. Ouvrez **PowerShell en tant qu'administrateur**
3. Executez la commande suivante :

```powershell
# Export the current Start menu layout to an XML file
Export-StartLayout -Path "C:\StartLayout.xml"
```

Le fichier genere contient la description complete de votre menu Demarrer. Voici un exemple simplifie :

```xml
<!-- Simplified Start menu layout for Windows 10 -->
<LayoutModificationTemplate
    xmlns:defaultlayout="http://schemas.microsoft.com/Start/2014/FullDefaultLayout"
    xmlns:start="http://schemas.microsoft.com/Start/2014/StartLayout"
    xmlns="http://schemas.microsoft.com/Start/2014/LayoutModification"
    Version="1">
    <LayoutOptions StartTileGroupCellWidth="6" />
    <DefaultLayoutOverride>
        <StartLayoutCollection>
            <defaultlayout:StartLayout GroupCellWidth="6">
                <start:Group Name="Bureautique">
                    <start:DesktopApplicationTile
                        Size="2x2"
                        Column="0"
                        Row="0"
                        DesktopApplicationLinkPath="%ALLUSERSPROFILE%\Microsoft\Windows\Start Menu\Programs\Word.lnk" />
                    <start:DesktopApplicationTile
                        Size="2x2"
                        Column="2"
                        Row="0"
                        DesktopApplicationLinkPath="%ALLUSERSPROFILE%\Microsoft\Windows\Start Menu\Programs\Excel.lnk" />
                    <start:DesktopApplicationTile
                        Size="2x2"
                        Column="4"
                        Row="0"
                        DesktopApplicationLinkPath="%ALLUSERSPROFILE%\Microsoft\Windows\Start Menu\Programs\Outlook.lnk" />
                </start:Group>
                <start:Group Name="Outils">
                    <start:Tile
                        Size="2x2"
                        Column="0"
                        Row="0"
                        AppUserModelID="Microsoft.WindowsCalculator_8wekyb3d8bbwe!App" />
                    <start:Tile
                        Size="2x2"
                        Column="2"
                        Row="0"
                        AppUserModelID="Microsoft.WindowsNotepad_8wekyb3d8bbwe!App" />
                </start:Group>
            </defaultlayout:StartLayout>
        </StartLayoutCollection>
    </DefaultLayoutOverride>
</LayoutModificationTemplate>
```

!!! info "Deux types de tuiles"
    Dans le fichier XML, vous trouverez deux types d'elements :

    - `<start:Tile>` : pour les applications UWP (du Microsoft Store). Identifiees par leur `AppUserModelID`.
    - `<start:DesktopApplicationTile>` : pour les applications de bureau classiques (installees en `.exe` ou `.msi`). Identifiees par le chemin vers leur raccourci `.lnk`.

### :material-upload: Deployer le layout via GPO

Une fois le fichier XML pret :

1. Copiez le fichier XML sur un partage reseau accessible en lecture par tous les utilisateurs (par exemple `\\serveur\partage\config\StartLayout.xml`)
2. Dans l'editeur de GPO, naviguez vers :

    **Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Menu Demarrer et barre des taches**

3. Double-cliquez sur **Disposition de l'ecran de demarrage**
4. Selectionnez **Active**
5. Dans le champ **Fichier de disposition de l'ecran de demarrage**, entrez le chemin UNC :
   `\\serveur\partage\config\StartLayout.xml`
6. Cliquez **OK**

### :material-lock-open-variant: Layout complet vs layout partiel

Par defaut, le layout exporte est **complet** : il verrouille entierement le menu Demarrer. Les utilisateurs ne peuvent ni ajouter ni supprimer de tuiles. C'est l'option la plus restrictive.

Si vous voulez laisser un peu de liberte aux utilisateurs, vous pouvez creer un **layout partiel**. Pour cela, ajoutez `LayoutCustomizationRestrictionType="OnlySpecifiedGroups"` dans le fichier XML :

```xml
<!-- Partial layout: only the specified groups are locked -->
<DefaultLayoutOverride LayoutCustomizationRestrictionType="OnlySpecifiedGroups">
    <StartLayoutCollection>
        <!-- Groups defined here are locked -->
        <!-- Users can add their own groups in the remaining space -->
    </StartLayoutCollection>
</DefaultLayoutOverride>
```

Avec cette option, les groupes de tuiles que vous definissez sont verrouilles, mais l'utilisateur peut ajouter ses propres groupes a cote. C'est un bon compromis entre controle et flexibilite.

| Mode | Comportement | Cas d'usage |
|:----:|:------------:|:-----------:|
| Layout complet (par defaut) | Le menu Demarrer est entierement verrouille | Postes en libre-service, bornes, kiosques |
| Layout partiel (`OnlySpecifiedGroups`) | Les groupes definis sont verrouilles, l'utilisateur peut ajouter les siens | Postes de bureau classiques en entreprise |

!!! warning "L'utilisateur ne peut pas modifier les groupes verrouilles"
    Meme en mode partiel, les groupes que vous avez definis dans le fichier XML ne peuvent pas etre modifies par l'utilisateur. Les tuiles sont grises si l'utilisateur essaie de les deplacer. C'est voulu : les applications de l'entreprise restent visibles, et l'utilisateur organise ses applications personnelles autour.

!!! quote "En resume"
    - Sous Windows 10, le menu Demarrer est personnalise via un fichier XML.
    - Workflow : personnaliser un poste de reference → exporter avec `Export-StartLayout` → deployer via GPO.
    - Le layout complet verrouille tout. Le layout partiel (`OnlySpecifiedGroups`) laisse de la liberte aux utilisateurs.
    - Le fichier XML doit etre sur un partage reseau accessible en lecture.

---

## Menu Demarrer : layout Windows 11

:material-alert-circle: Windows 11 a completement change le menu Demarrer. Fini les tuiles, place a une liste d'**applications epinglees** et une section **Recommandations**. La consequence : le fichier XML de Windows 10 **ne fonctionne pas** sous Windows 11. Il faut utiliser un fichier **JSON** a la place.

C'est comme si vous aviez appris a cuisiner avec un four a gaz, et qu'on vous donnait soudainement un four electrique. Les ingredients sont les memes, mais la methode change completement.

### :material-code-json: Le principe : un fichier JSON

Le layout du menu Demarrer sous Windows 11 est defini dans un fichier JSON beaucoup plus simple que l'XML de Windows 10. Il contient la liste des applications epinglees, identifiees par leur `desktopAppId` (applications classiques) ou leur `packagedAppId` (applications du Store).

Voici un exemple :

```json
{
    "pinnedList": [
        { "desktopAppId": "MSEdge" },
        { "desktopAppId": "Microsoft.Office.WINWORD.EXE.15" },
        { "desktopAppId": "Microsoft.Office.EXCEL.EXE.15" },
        { "desktopAppId": "Microsoft.Office.OUTLOOK.EXE.15" },
        { "desktopAppId": "Microsoft.Office.POWERPNT.EXE.15" },
        { "packagedAppId": "Microsoft.WindowsCalculator_8wekyb3d8bbwe!App" },
        { "packagedAppId": "Microsoft.WindowsNotepad_8wekyb3d8bbwe!App" },
        { "packagedAppId": "windows.immersivecontrolpanel_cw5n1h2txyewy!microsoft.windows.immersivecontrolpanel" },
        { "desktopAppId": "Microsoft.Windows.Explorer" }
    ]
}
```

C'est beaucoup plus lisible que le XML, non ?

### :material-identifier: Trouver les identifiants des applications

Pour trouver le `desktopAppId` ou le `packagedAppId` d'une application, la methode la plus fiable est d'exporter la configuration actuelle.

Sous Windows 11 22H2 et superieur, vous pouvez utiliser PowerShell :

```powershell
# Export the current Start menu layout (Windows 11)
# The layout is stored in a binary file per user
$startLayoutPath = "$env:LOCALAPPDATA\Packages\Microsoft.Windows.StartMenuExperienceHost_cw5n1h2txyewy\LocalState\start2.bin"

# Copy to a working location for inspection
Copy-Item $startLayoutPath -Destination "C:\start2-backup.bin"
```

!!! tip "Methode alternative : Intune"
    Si vous gerez aussi des postes via Microsoft Intune (Endpoint Manager), vous pouvez deployer le layout JSON via un **profil de configuration** au lieu d'une GPO. C'est la methode recommandee par Microsoft pour les postes geres en mode cloud. La GPO reste pertinente pour les environnements Active Directory traditionnels.

### :material-upload: Deployer le layout via GPO

Le deploiement sous Windows 11 passe par un parametre different de celui de Windows 10 :

1. Copiez le fichier JSON sur un partage reseau (par exemple `\\serveur\partage\config\StartLayout.json`)
2. Dans l'editeur de GPO, naviguez vers :

    **Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Menu Demarrer et barre des taches**

3. Double-cliquez sur **Configurer la disposition de l'ecran de demarrage**
4. Selectionnez **Active**
5. Dans le champ de texte, collez le contenu JSON directement (pas le chemin du fichier, mais le contenu)
6. Cliquez **OK**

!!! warning "Contenu JSON, pas chemin de fichier"
    Contrairement a Windows 10 ou l'on renseigne un **chemin** vers le fichier XML, la GPO Windows 11 attend le **contenu brut** du fichier JSON. Copiez-collez le JSON directement dans le champ texte du parametre. C'est une source de confusion frequente.

### :material-compare: Comparaison Windows 10 vs Windows 11

| Element | Windows 10 | Windows 11 |
|:-------:|:----------:|:----------:|
| **Format du layout** | XML | JSON |
| **Structure du menu** | Tuiles dans des groupes | Liste d'applications epinglees |
| **Tailles des elements** | 4 tailles de tuiles (petite, moyenne, large, grande) | Taille unique |
| **Parametre GPO** | Disposition de l'ecran de demarrage | Configurer la disposition de l'ecran de demarrage |
| **Champ du parametre** | Chemin vers le fichier XML | Contenu JSON brut |
| **Layout partiel** | Oui (`OnlySpecifiedGroups`) | Non (le layout est toujours complet) |
| **Export PowerShell** | `Export-StartLayout` | Pas de cmdlet officiel equivalent |
| **Applications vivantes (Live Tiles)** | Oui | Non (supprimees) |

!!! info "Poste mixte Windows 10 et 11 ?"
    Si votre parc contient a la fois des postes Windows 10 et Windows 11, vous aurez besoin de **deux GPO distinctes** avec un **filtrage WMI** pour cibler la bonne version. Utilisez le filtre `SELECT * FROM Win32_OperatingSystem WHERE Version LIKE "10.0.2%"` pour Windows 11 (les versions 22xxx et superieures) et `SELECT * FROM Win32_OperatingSystem WHERE Version LIKE "10.0.1%"` pour Windows 10. Le filtrage WMI est aborde dans les chapitres avances.

!!! quote "En resume"
    - Windows 11 utilise un fichier JSON, pas le XML de Windows 10. Les deux sont incompatibles.
    - Le JSON est plus simple : une liste d'applications epinglees avec leur identifiant.
    - La GPO Windows 11 attend le **contenu JSON** dans le champ texte, pas un chemin de fichier.
    - Pour un parc mixte, il faut deux GPO separees avec filtrage WMI.

---

## Icones du bureau

Les icones du bureau sont un autre element que l'on peut controler par GPO. Certaines entreprises preferent un bureau **completement vide** (plus propre, moins de distractions), d'autres veulent simplement masquer certaines icones systeme comme la Corbeille ou le raccourci Reseau.

### :material-desktop-mac: Les parametres disponibles

Les parametres des icones du bureau se trouvent dans :

**Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Bureau**

| Parametre | Chemin exact | Effet |
|:---------:|:------------:|:-----:|
| Masquer et desactiver tous les elements du bureau | Bureau | Supprime **toutes** les icones, fichiers et dossiers du bureau |
| Supprimer l'icone de la Corbeille du bureau | Bureau | La Corbeille n'apparait plus sur le bureau |
| Supprimer l'icone Voisinage reseau du bureau | Bureau | Le raccourci Reseau disparait du bureau |
| Interdire a l'utilisateur de modifier le chemin des dossiers Mes documents | Bureau | Empeche la redirection manuelle de Mes documents |
| Empecher l'ajout, le glisser-deposer et la fermeture des barres d'outils de la barre des taches | Bureau | Empeche la personnalisation des barres d'outils |

!!! example "Pas a pas"

    Pour masquer toutes les icones du bureau :

    1. Dans l'editeur de GPO, naviguez vers :
       **Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Bureau**
    2. Double-cliquez sur **Masquer et desactiver tous les elements du bureau**
    3. Selectionnez **Active**
    4. Cliquez **OK**

    Pour masquer uniquement la Corbeille :

    1. Dans le meme dossier **Bureau**, double-cliquez sur **Supprimer l'icone de la Corbeille du bureau**
    2. Selectionnez **Active**
    3. Cliquez **OK**

!!! warning "Masquer toutes les icones = tout masquer"
    Le parametre **Masquer et desactiver tous les elements du bureau** ne masque pas seulement les icones systeme. Il masque **tout** : les fichiers, les dossiers, les raccourcis que l'utilisateur a places sur le bureau. C'est radical. Les fichiers ne sont pas supprimes (ils sont toujours dans `C:\Users\utilisateur\Desktop`), mais ils deviennent invisibles. Si vos utilisateurs stockent des fichiers sur le bureau (beaucoup le font), cette option peut generer des tickets de support en cascade.

!!! tip "L'alternative douce"
    Plutot que de masquer toutes les icones d'un coup, commencez par masquer uniquement les icones systeme inutiles (Corbeille, Reseau) et laissez les raccourcis et fichiers de l'utilisateur visibles. C'est moins brutal et tout aussi propre visuellement.

!!! quote "En resume"
    - Les icones du bureau se gerent dans Configuration utilisateur → Modeles d'administration → Bureau.
    - **Masquer tous les elements** est radical : fichiers et dossiers de l'utilisateur deviennent invisibles.
    - Preferez masquer individuellement les icones systeme (Corbeille, Reseau) pour un nettoyage en douceur.
    - Les fichiers masques ne sont pas supprimes, ils restent sur le disque.

---

## Economiseur d'ecran avec verrouillage

C'est probablement le parametre de securite le plus important de ce chapitre. Un poste non verrouille dans un open space, c'est comme laisser la porte de votre maison grande ouverte avec un panneau "Entrez, servez-vous". N'importe qui peut acceder aux mails, aux documents confidentiels, voire aux applications metier avec les droits de la personne absente.

La GPO d'economiseur d'ecran permet de forcer :

1. L'**activation** de l'economiseur d'ecran
2. Le **verrouillage** de la session au reveil (saisie du mot de passe obligatoire)
3. Le **delai** d'inactivite avant le declenchement

### :material-shield-lock: Les trois parametres cles

Ils se trouvent tous dans le meme dossier :

**Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Panneau de configuration** > **Personnalisation**

| Parametre | Action | Effet |
|:---------:|:------:|:-----:|
| Activer l'economiseur d'ecran | Activer | Force l'utilisation d'un economiseur d'ecran |
| Proteger l'economiseur d'ecran par mot de passe | Activer | Exige un mot de passe au reveil |
| Delai de l'economiseur d'ecran | Activer + definir la valeur | Nombre de secondes d'inactivite avant le declenchement |

!!! example "Pas a pas"

    **Etape 1** : Dans l'editeur de GPO, naviguez vers :

    **Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Panneau de configuration** > **Personnalisation**

    **Etape 2** : Activer l'economiseur d'ecran.

    1. Double-cliquez sur **Activer l'economiseur d'ecran**
    2. Selectionnez **Active**
    3. Cliquez **OK**

    **Etape 3** : Exiger le mot de passe au reveil.

    1. Double-cliquez sur **Proteger l'economiseur d'ecran par mot de passe**
    2. Selectionnez **Active**
    3. Cliquez **OK**

    **Etape 4** : Definir le delai d'inactivite.

    1. Double-cliquez sur **Delai de l'economiseur d'ecran**
    2. Selectionnez **Active**
    3. Dans le champ **Secondes**, entrez `900` (= 15 minutes)
    4. Cliquez **OK**

    **Etape 5** (optionnel) : Forcer un economiseur specifique.

    1. Double-cliquez sur **Forcer un economiseur d'ecran specifique**
    2. Selectionnez **Active**
    3. Dans le champ **Nom du fichier de l'economiseur d'ecran**, entrez : `scrnsave.scr` (ecran noir)
    4. Cliquez **OK**

!!! info "Quel delai choisir ?"
    Il n'y a pas de reponse universelle, mais voici les valeurs les plus courantes :

    | Delai | Secondes | Usage |
    |:-----:|:--------:|:-----:|
    | 5 minutes | 300 | Environnement tres sensible (salle serveur, finance) |
    | 10 minutes | 600 | Bon compromis securite/confort |
    | 15 minutes | 900 | Standard en entreprise (recommandation ANSSI) |
    | 30 minutes | 1800 | Environnement a faible risque |

    Un delai trop court agacera les utilisateurs (l'ecran se verrouille pendant qu'ils lisent un document long). Un delai trop long laisse le poste ouvert trop longtemps. **15 minutes** (900 secondes) est le bon compromis pour la plupart des entreprises.

### :material-check-decagram: Verifier l'application

Sur le poste de test, apres un `gpupdate /force` :

1. Ouvrez les **Parametres** > **Personnalisation** > **Ecran de verrouillage** > **Parametres de l'economiseur d'ecran**
2. Les champs doivent etre grises (non modifiables par l'utilisateur)
3. L'economiseur d'ecran est active avec le delai que vous avez defini
4. La case **A la reprise, afficher l'ecran de connexion** est cochee et grisee

Si les champs sont toujours modifiables, verifiez avec `gpresult /r` que la GPO est bien appliquee. Le probleme le plus courant est un oubli de liaison de la GPO a l'OU contenant l'utilisateur de test.

!!! quote "En resume"
    - Trois parametres a configurer ensemble : activer l'economiseur, exiger le mot de passe, definir le delai.
    - Tous dans Configuration utilisateur → Panneau de configuration → Personnalisation.
    - 15 minutes (900 secondes) est le delai standard recommande par l'ANSSI.
    - Verifiez que les champs sont grises dans les Parametres du poste apres `gpupdate /force`.

---

## Themes et couleurs

Les themes Windows regroupent le fond d'ecran, les couleurs, les sons et le curseur dans un ensemble coherent. Par GPO, on peut forcer un theme specifique ou empecher les utilisateurs de le modifier.

### :material-palette: Les parametres disponibles

Les parametres de themes se trouvent dans deux emplacements :

**Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Panneau de configuration** > **Personnalisation**

| Parametre | Effet |
|:---------:|:-----:|
| Charger un theme specifique | Force le chargement d'un fichier `.theme` |
| Empecher la modification du theme | L'utilisateur ne peut plus changer le theme |
| Empecher la modification de la couleur et de l'apparence | Verrouille les couleurs |
| Empecher la modification de l'arriere-plan du bureau | Empeche le changement du fond d'ecran (complementaire au parametre "Papier peint du Bureau") |
| Empecher la modification des sons | Verrouille le schema sonore |

### :material-brush: Forcer un theme

Pour forcer un theme specifique :

1. Creez ou recuperez un fichier `.theme` (les themes par defaut de Windows se trouvent dans `C:\Windows\Resources\Themes\`)
2. Placez le fichier sur un partage reseau : `\\serveur\partage\config\entreprise.theme`
3. Dans l'editeur de GPO :
    1. Double-cliquez sur **Charger un theme specifique**
    2. Selectionnez **Active**
    3. Dans le champ **Chemin du fichier de theme**, entrez le chemin UNC
    4. Cliquez **OK**
4. Activez aussi **Empecher la modification du theme** pour que l'utilisateur ne puisse pas le remplacer

!!! warning "Limites des GPO pour les themes"
    Le controle des themes par GPO est relativement **limite**. Certains elements de personnalisation, comme la **couleur d'accentuation** de Windows 10/11 (la couleur des barres de titre, du menu Demarrer et de la barre des taches), ne disposent pas de parametre GPO natif.

    Pour aller plus loin, il faut passer par les **preferences de registre** dans la GPO. On peut definir la valeur `AccentColor` dans `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Accent`. Ce sujet est aborde au chapitre 10 sur les preferences de GPO.

!!! tip "Mode sombre par GPO ?"
    Le mode sombre (Dark Mode) de Windows 10/11 ne dispose pas non plus de parametre GPO natif. Il faut passer par une preference de registre :

    - Cle : `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Themes\Personalize`
    - Valeur : `AppsUseLightTheme` = `0` (DWORD) pour le mode sombre des applications
    - Valeur : `SystemUsesLightTheme` = `0` (DWORD) pour le mode sombre du systeme

    Ces manipulations de registre via preferences GPO seront detaillees au chapitre 10.

!!! quote "En resume"
    - Les parametres de themes sont dans Configuration utilisateur → Personnalisation.
    - On peut forcer un fichier `.theme` et empecher l'utilisateur de le modifier.
    - La couleur d'accentuation et le mode sombre n'ont **pas** de parametre GPO natif — il faut passer par les preferences de registre (chapitre 10).
    - Le controle des themes par GPO est plus limite que les autres parametres vus dans ce chapitre.

---

## Exercice pratique

Il est temps de mettre tout ca en pratique. On va creer une GPO complete qui standardise l'environnement de bureau pour tous les utilisateurs d'une OU.

**Objectif** : creer la GPO `CFG-Utilisateurs-EnvironnementBureau` qui :

- :material-image: Impose un fond d'ecran d'entreprise
- :material-pin: Verrouille la barre des taches
- :material-shield-lock: Force l'economiseur d'ecran avec verrouillage apres 15 minutes
- :material-desktop-mac: Masque les icones systeme inutiles du bureau

**Duree estimee** : 15-20 minutes.

**Prerequis** :

- L'OU de test `OU-Test-GPO` est en place (chapitre 5)
- Un poste et un utilisateur de test sont dans cette OU
- Une image de fond d'ecran est disponible sur un partage reseau

### Etape 1 : Creer la GPO

1. Ouvrez `gpmc.msc`
2. Naviguez vers **Objets de strategie de groupe**
3. Clic droit > **Nouveau**
4. Nom : `CFG-Utilisateurs-EnvironnementBureau`
5. Cliquez **OK**

### Etape 2 : Configurer le fond d'ecran

1. Clic droit sur `CFG-Utilisateurs-EnvironnementBureau` > **Modifier...**
2. Naviguez vers : **Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Bureau** > **Bureau**
3. Double-cliquez sur **Papier peint du Bureau**
4. Selectionnez **Active**
5. Nom du papier peint : `\\serveur\partage\images\fond-ecran-entreprise.jpg`
6. Style du papier peint : **Remplissage**
7. Cliquez **OK**

### Etape 3 : Verrouiller la barre des taches

1. Naviguez vers : **Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Menu Demarrer et barre des taches**
2. Double-cliquez sur **Verrouiller la barre des taches** → **Active** → **OK**
3. Double-cliquez sur **Supprimer l'acces au menu contextuel de la barre des taches** → **Active** → **OK**
4. Double-cliquez sur **Masquer le bouton Affichage des taches** → **Active** → **OK**

### Etape 4 : Configurer l'economiseur d'ecran

1. Naviguez vers : **Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Panneau de configuration** > **Personnalisation**
2. Double-cliquez sur **Activer l'economiseur d'ecran** → **Active** → **OK**
3. Double-cliquez sur **Proteger l'economiseur d'ecran par mot de passe** → **Active** → **OK**
4. Double-cliquez sur **Delai de l'economiseur d'ecran** → **Active** → Secondes : `900` → **OK**

### Etape 5 : Masquer les icones systeme

1. Naviguez vers : **Configuration utilisateur** > **Strategies** > **Modeles d'administration** > **Bureau**
2. Double-cliquez sur **Supprimer l'icone de la Corbeille du bureau** → **Active** → **OK**
3. Double-cliquez sur **Supprimer l'icone Voisinage reseau du bureau** → **Active** → **OK**

### Etape 6 : Lier a l'OU de test

1. Fermez l'editeur de GPO
2. Dans `gpmc.msc`, naviguez vers **OU-Test-GPO**
3. Clic droit > **Lier un objet de strategie de groupe existant...**
4. Selectionnez `CFG-Utilisateurs-EnvironnementBureau`
5. Cliquez **OK**

### Etape 7 : Verifier

Sur le poste de test, connecte avec le compte utilisateur de test :

```cmd
gpupdate /force
```

Puis fermez et rouvrez la session (certains parametres ne prennent effet qu'a l'ouverture de session).

Verifiez les points suivants :

| Element | Verification | Resultat attendu |
|:-------:|:------------:|:----------------:|
| Fond d'ecran | Regarder le bureau | L'image d'entreprise est affichee |
| Barre des taches | Essayer un clic droit sur la barre | Le menu contextuel ne s'affiche pas |
| Barre des taches | Essayer de deplacer la barre | Impossible (verrouillee) |
| Affichage des taches | Chercher le bouton | Le bouton n'est plus visible |
| Corbeille | Regarder le bureau | La Corbeille n'est plus visible |
| Reseau | Regarder le bureau | L'icone Reseau n'est plus visible |
| Economiseur d'ecran | Ouvrir Parametres > Personnalisation > Ecran de verrouillage > Economiseur d'ecran | Les champs sont grises, delai = 15 minutes |

Confirmez avec `gpresult /r /scope:user` :

```cmd
gpresult /r /scope:user
```

```title="Resultat attendu"
RESULTAT UTILISATEUR
--------------------

    Objets de strategie de groupe appliques
    ----------------------------------------
        CFG-Utilisateurs-EnvironnementBureau
        Default Domain Policy
```

!!! danger "Ca ne fonctionne pas ?"
    Verifiez dans l'ordre :

    1. La GPO est-elle **liee** a l'OU ? (Verifiez dans `gpmc.msc`)
    2. Le **compte utilisateur** de test est-il dans la bonne OU ?
    3. Le partage reseau du fond d'ecran est-il accessible ? (Testez avec `dir \\serveur\partage\images\`)
    4. La GPO apparait-elle dans `gpresult /r` ? Si elle est dans "Objets refuses", verifiez les filtres de securite.
    5. Avez-vous **ferme et rouvert la session** apres le `gpupdate /force` ?

!!! quote "En resume"
    - Vous avez cree une GPO complete qui standardise le bureau : fond d'ecran, barre des taches, economiseur d'ecran et icones.
    - 7 etapes : creer, fond d'ecran, barre des taches, economiseur, icones, lier, verifier.
    - `gpresult /r` confirme l'application. Une fermeture de session peut etre necessaire.

---

## Recapitulatif complet

Voici le tableau de synthese de tous les parametres couverts dans ce chapitre :

| Parametre | Chemin GPO | Section |
|:---------:|:----------:|:-------:|
| Papier peint du Bureau | Config. utilisateur → Bureau → Bureau | Fond d'ecran |
| Forcer une image d'ecran de verrouillage | Config. ordinateur → Personnalisation | Ecran de verrouillage |
| Verrouiller la barre des taches | Config. utilisateur → Menu Demarrer et barre des taches | Barre des taches |
| Supprimer le menu contextuel de la barre des taches | Config. utilisateur → Menu Demarrer et barre des taches | Barre des taches |
| Masquer le bouton Affichage des taches | Config. utilisateur → Menu Demarrer et barre des taches | Barre des taches |
| Masquer l'icone Personnes | Config. utilisateur → Menu Demarrer et barre des taches | Barre des taches |
| Disposition de l'ecran de demarrage (W10) | Config. utilisateur → Menu Demarrer et barre des taches | Menu Demarrer |
| Configurer la disposition (W11) | Config. utilisateur → Menu Demarrer et barre des taches | Menu Demarrer |
| Masquer tous les elements du bureau | Config. utilisateur → Bureau | Icones |
| Supprimer la Corbeille | Config. utilisateur → Bureau | Icones |
| Supprimer l'icone Reseau | Config. utilisateur → Bureau | Icones |
| Activer l'economiseur d'ecran | Config. utilisateur → Personnalisation | Economiseur |
| Proteger par mot de passe | Config. utilisateur → Personnalisation | Economiseur |
| Delai de l'economiseur d'ecran | Config. utilisateur → Personnalisation | Economiseur |
| Charger un theme specifique | Config. utilisateur → Personnalisation | Themes |
| Empecher la modification du theme | Config. utilisateur → Personnalisation | Themes |

!!! tip "Conseil de nomenclature"
    Regroupez tous ces parametres dans une seule GPO nommee `CFG-Utilisateurs-EnvironnementBureau` plutot que de creer une GPO par parametre. Cela facilite la maintenance et reduit le nombre d'objets a gerer. En revanche, si certains parametres doivent s'appliquer a des groupes differents (par exemple, un fond d'ecran different par departement), creez des GPO separees.

---

## Et maintenant ?

Vous savez desormais controler l'apparence et le comportement du bureau Windows par GPO. Vos postes ont tous le meme fond d'ecran, la barre des taches est verrouillee, et l'economiseur d'ecran protege les sessions inactives.

Mais il reste un sujet critique que tout administrateur doit maitriser : les **mises a jour Windows**. Comment s'assurer que tous les postes installent les correctifs de securite a temps ? Comment eviter qu'une mise a jour ne casse une application metier ? Comment planifier les redemarrages pour ne pas interrompre le travail des utilisateurs ?

C'est exactement ce que couvre le **chapitre 9 : Configurer Windows Update par GPO**.
