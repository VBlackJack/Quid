---
tags:
  - registre
  - débutant
  - Windows 11
  - personnalisation
  - compatibilité
---

# Le registre et Windows 11

!!! abstract "Ce que vous allez apprendre"
    - Ce qui a change avec Windows 11 par rapport a Windows 10
    - Les 6 modifications du registre les plus demandees sous Windows 11
    - Comment verifier si votre PC est compatible Windows 11 via le registre
    - Les nouvelles cles specifiques a Windows 11 (Snap, Copilot, confidentialite)
    - Les anciennes astuces de Windows 10 qui ne fonctionnent plus
    - Ce que les mises a jour 22H2, 23H2 et 24H2 ont change

!!! warning "Rappel essentiel"
    Avant **chaque** modification, exportez la cle concernee (clic droit > Exporter). Les modifications de Windows 11 touchent souvent des cles sensibles de l'Explorateur. Relisez le chapitre 5 si besoin.

---

## Ce qui a change avec Windows 11

Windows 11 n'est pas juste une couche de peinture sur Windows 10. Microsoft a **reecrit** une bonne partie de l'interface :

| Nouveaute | Description |
|-----------|-------------|
| :material-dock-bottom: Barre des taches centree | Les icones sont centrees par defaut, comme sur macOS |
| :material-menu: Nouveau menu Demarrer | Plus de tuiles dynamiques, remplacees par des applications epinglees et des "Recommandations" |
| :material-rounded-corner: Coins arrondis | Toutes les fenetres ont des coins arrondis |
| :material-widgets: Widgets | Un panneau lateral avec la meteo, l'actualite, la bourse... |
| :material-dock-window: Snap Layouts | Survolez le bouton d'agrandissement pour organiser vos fenetres |
| :material-cursor-default-click: Nouveau menu contextuel | Le clic droit a ete simplifie (et beaucoup d'options cachees) |

Le probleme : certains de ces changements ne plaisent pas a tout le monde, et Microsoft ne propose **pas toujours** de reglage dans les Parametres pour revenir en arriere. C'est la que le registre entre en jeu.

!!! quote "En resume"
    - Windows 11 a reecrit une bonne partie de l'interface : barre des taches centree, nouveau menu Demarrer, coins arrondis, Snap Layouts et menu contextuel simplifie.
    - Certains de ces changements n'ont pas d'option de retour en arriere dans les Parametres, d'ou l'utilite du registre.

---

## Les modifications les plus demandees

### Modification 1 : Restaurer le menu contextuel classique

C'est **la** modification la plus populaire de Windows 11. Le nouveau menu contextuel (clic droit) cache la plupart des options derriere un bouton "Afficher plus d'options". Si vous en avez assez de ce clic supplementaire :

!!! example "Essayez vous-meme"
    **Cle a creer** :
    ```
    HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32
    ```

    **Etapes dans Regedit** :

    1. Naviguez vers `HKCU\Software\Classes\CLSID`
    2. Clic droit sur `CLSID` → **Nouveau** → **Cle**
    3. Nommez-la exactement : `{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}`
    4. Dans cette cle, creez une sous-cle nommee `InprocServer32`
    5. Dans `InprocServer32`, double-cliquez sur la valeur `(Par defaut)`
    6. Laissez le champ **Donnees de la valeur** completement **vide** et cliquez OK
    7. Redemarrez l'Explorateur (voir ci-dessous)

    **Commande alternative** :

    ```powershell
    # Restore classic right-click context menu in Windows 11
    reg add "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32" /f /ve
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

    **Redemarrer l'Explorateur** :

    ```powershell
    # Restart Windows Explorer to apply changes
    Stop-Process -Name explorer -Force; Start-Process explorer
    ```

    ```title="Resultat attendu"
    L'Explorateur redemarre automatiquement. La barre des taches disparait
    brievement puis reapparait. C'est normal !
    ```

!!! tip "Pour annuler"
    Supprimez toute la cle `{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}` dans Regedit, puis redemarrez l'Explorateur.

    ```powershell
    # Revert to Windows 11 modern context menu
    reg delete "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}" /f
    Stop-Process -Name explorer -Force; Start-Process explorer
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    L'Explorateur redemarre et le menu contextuel moderne revient.
    ```

---

### Modification 2 : Desactiver les widgets

Le panneau de widgets (meteo, actualites, bourse) consomme des ressources et s'ouvre parfois par accident quand vous survolez le coin gauche de la barre des taches.

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKLM\SOFTWARE\Policies\Microsoft\Dsh
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `AllowNewsAndInterests` | REG_DWORD | `0` | Desactive les widgets |

    ```powershell
    # Disable widgets panel in Windows 11
    reg add "HKLM\SOFTWARE\Policies\Microsoft\Dsh" /v AllowNewsAndInterests /t REG_DWORD /d 0 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

    **Redemarrage necessaire** : fermez et rouvrez votre session, ou redemarrez le PC.

!!! tip "Pour annuler"
    ```powershell
    # Re-enable widgets panel
    reg delete "HKLM\SOFTWARE\Policies\Microsoft\Dsh" /v AllowNewsAndInterests /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

---

### Modification 3 : Aligner la barre des taches a gauche

Par defaut, Windows 11 centre les icones de la barre des taches. Pour les remettre a gauche, comme sous Windows 10 :

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `TaskbarAl` | REG_DWORD | `0` | Aligne a gauche |
    | `TaskbarAl` | REG_DWORD | `1` | Centre (par defaut) |

    ```powershell
    # Align taskbar icons to the left
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced" /v TaskbarAl /t REG_DWORD /d 0 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

    **Redemarrage necessaire** : non, le changement s'applique immediatement.

!!! tip "Pour annuler"
    ```powershell
    # Center taskbar icons (default)
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced" /v TaskbarAl /t REG_DWORD /d 1 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

---

### Modification 4 : Desactiver les resultats de recherche mis en avant

La barre de recherche affiche par defaut des "highlights" : des suggestions, des evenements du jour, des tendances. Pour avoir une recherche sobre et rapide :

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKCU\Software\Microsoft\Windows\CurrentVersion\SearchSettings
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `IsDynamicSearchBoxEnabled` | REG_DWORD | `0` | Desactive les suggestions visuelles |
    | `IsDynamicSearchBoxEnabled` | REG_DWORD | `1` | Active (par defaut) |

    ```powershell
    # Disable search highlights
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\SearchSettings" /v IsDynamicSearchBoxEnabled /t REG_DWORD /d 0 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

    **Redemarrage necessaire** : non, le changement s'applique au prochain clic sur la recherche.

!!! tip "Pour annuler"
    ```powershell
    # Re-enable search highlights
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\SearchSettings" /v IsDynamicSearchBoxEnabled /t REG_DWORD /d 1 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

---

### Modification 5 : Masquer les "Recommandations" du menu Demarrer

La section "Recommandations" en bas du menu Demarrer affiche les fichiers recemment ouverts et les applications recemment installees. Certains trouvent ca intrusif.

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKCU\Software\Policies\Microsoft\Windows\Explorer
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `HideRecommendedSection` | REG_DWORD | `1` | Masque la section Recommandations |
    | `HideRecommendedSection` | REG_DWORD | `0` | Affiche (par defaut) |

    ```powershell
    # Hide Recommended section in Start menu
    reg add "HKCU\Software\Policies\Microsoft\Windows\Explorer" /v HideRecommendedSection /t REG_DWORD /d 1 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

    **Redemarrage necessaire** : fermez et rouvrez votre session pour voir le changement.

!!! note "Efficacite variable"
    Cette modification fonctionne de maniere fiable sur les editions Pro et Enterprise. Sur Windows Home, le resultat peut varier selon la version de Windows 11 installee.

!!! tip "Pour annuler"
    ```powershell
    # Show Recommended section in Start menu
    reg delete "HKCU\Software\Policies\Microsoft\Windows\Explorer" /v HideRecommendedSection /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

---

### Modification 6 : Ne jamais regrouper les fenetres dans la barre des taches

Par defaut, Windows 11 regroupe toutes les fenetres d'une meme application sous un seul bouton. Si vous preferez voir chaque fenetre separement :

!!! example "Essayez vous-meme"
    **Cle** :
    ```
    HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced
    ```

    **Modification** :

    | Valeur | Type | Donnees | Effet |
    |--------|:----:|:-------:|-------|
    | `TaskbarGlomLevel` | REG_DWORD | `0` | Toujours regrouper (par defaut) |
    | `TaskbarGlomLevel` | REG_DWORD | `1` | Regrouper quand la barre est pleine |
    | `TaskbarGlomLevel` | REG_DWORD | `2` | Ne jamais regrouper |

    ```powershell
    # Never combine taskbar buttons
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced" /v TaskbarGlomLevel /t REG_DWORD /d 2 /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

    **Redemarrage necessaire** : redemarrez l'Explorateur.

    ```powershell
    # Restart Explorer to apply taskbar changes
    Stop-Process -Name explorer -Force; Start-Process explorer
    ```

    ```title="Resultat attendu"
    L'Explorateur redemarre automatiquement. La barre des taches disparait
    brievement puis reapparait. C'est normal !
    ```

!!! warning "Versions de Windows 11"
    Cette valeur de registre ne fonctionnait **pas** dans les premieres versions de Windows 11 (21H2 et debut 22H2). Microsoft a reintroduit cette fonctionnalite nativement dans les Parametres a partir de **Windows 11 23H2**. Si vous etes sur une version recente, utilisez plutot : Parametres → Personnalisation → Barre des taches → Comportements de la barre des taches.

!!! tip "Pour annuler"
    ```powershell
    # Always combine taskbar buttons (default)
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced" /v TaskbarGlomLevel /t REG_DWORD /d 0 /f
    Stop-Process -Name explorer -Force; Start-Process explorer
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    L'Explorateur redemarre et les boutons sont a nouveau regroupes.
    ```

!!! quote "En resume"
    - Les 6 modifications les plus populaires : menu contextuel classique, desactiver les widgets, aligner la barre a gauche, desactiver les highlights de recherche, masquer les Recommandations, ne jamais regrouper les fenetres.
    - Chaque modification est reversible : soit en supprimant la cle creee, soit en remettant la valeur par defaut.

---

## Verifier la compatibilite Windows 11

Windows 11 exige des composants materiels specifiques que votre PC ne possede pas forcement. Vous pouvez verifier certaines de ces exigences directement dans le registre.

### TPM 2.0

Le TPM (Trusted Platform Module) est une puce de securite presente sur les cartes meres recentes. Windows 11 l'exige en version 2.0.

```
HKLM\SYSTEM\CurrentControlSet\Control\IntegrityServices
```

| Valeur | Ce qu'elle indique |
|--------|-------------------|
| `UEFI` | Presente si le demarrage UEFI est actif ; utile pour le measured boot et certaines fonctions de securite, mais ce n'est pas une preuve directe de la presence d'un TPM 2.0 |

!!! tip "Verification plus simple"
    La methode la plus fiable pour verifier le TPM reste ++win+r++ → `tpm.msc` ou la cmdlet PowerShell `Get-Tpm`. Le registre peut completer le diagnostic, pas remplacer ces outils.

### Secure Boot

Le demarrage securise (Secure Boot) empeche le chargement de logiciels non signes au demarrage.

```
HKLM\SYSTEM\CurrentControlSet\Control\SecureBoot\State
```

| Valeur | Type | Donnees | Signification |
|--------|:----:|:-------:|---------------|
| `UEFISecureBootEnabled` | REG_DWORD | `1` | Secure Boot actif |
| `UEFISecureBootEnabled` | REG_DWORD | `0` | Secure Boot inactif |

```powershell
# Check Secure Boot status
reg query "HKLM\SYSTEM\CurrentControlSet\Control\SecureBoot\State" /v UEFISecureBootEnabled
```

```title="Resultat attendu"
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecureBoot\State
    UEFISecureBootEnabled    REG_DWORD    0x1
```

### Contourner les exigences materielles

Il existe une cle de registre qui permet d'installer Windows 11 sur un PC qui ne repond pas aux exigences minimales :

```
HKLM\SYSTEM\Setup\MoSetup
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `AllowUpgradesWithUnsupportedTPMOrCPU` | REG_DWORD | `1` | Autorise la mise a niveau malgre un TPM ou CPU non supporte |

```powershell
# Allow Windows 11 upgrade on unsupported hardware
reg add "HKLM\SYSTEM\Setup\MoSetup" /v AllowUpgradesWithUnsupportedTPMOrCPU /t REG_DWORD /d 1 /f
```

```title="Resultat attendu"
L'operation a reussi.
```

!!! danger "Risques importants"
    Microsoft a **officiellement prevenu** que les PC non compatibles :

    - Ne recevront **peut-etre pas** toutes les mises a jour de securite
    - Pourraient rencontrer des **problemes de stabilite**
    - Ne sont **pas couverts** par le support technique

    De plus, cette astuce ne fonctionne que pour les **mises a niveau** (upgrade depuis Windows 10). Pour une installation propre, il faut d'autres manipulations plus complexes.

    **Utilisez cette option uniquement si vous comprenez et acceptez ces risques.**

!!! quote "En resume"
    - Le TPM 2.0 et Secure Boot sont verifiables dans le registre, mais `tpm.msc` reste la methode la plus fiable.
    - La cle `AllowUpgradesWithUnsupportedTPMOrCPU` permet de contourner les exigences, mais avec des risques de stabilite et de support.

---

## Nouveautes Windows 11 dans le registre

Windows 11 a introduit de nouvelles fonctionnalites avec leurs propres cles de registre.

### Snap Layouts (disposition des fenetres)

Les Snap Layouts sont le menu qui apparait quand vous survolez le bouton d'agrandissement d'une fenetre. Si ce menu vous gene :

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `EnableSnapAssistFlyout` | REG_DWORD | `1` | Active les Snap Layouts (par defaut) |
| `EnableSnapAssistFlyout` | REG_DWORD | `0` | Desactive le menu de disposition |

```powershell
# Disable Snap Layouts flyout on maximize button hover
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced" /v EnableSnapAssistFlyout /t REG_DWORD /d 0 /f
```

```title="Resultat attendu"
L'operation a reussi.
```

### Ne pas deranger / Assistance a la concentration

Windows 11 a remplace l'ancienne "Assistance a la concentration" (Focus Assist) par un mode "Ne pas deranger" plus simple. Les reglages se trouvent dans :

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Notifications\Settings
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `NOC_GLOBAL_SETTING_ALLOW_TOASTS_ABOVE_LOCK` | REG_DWORD | `0` | Bloque les notifications sur l'ecran de verrouillage |
| `NOC_GLOBAL_SETTING_TOASTS_ENABLED` | REG_DWORD | `0` | Desactive toutes les notifications |

### Confidentialite

Windows 11 collecte des donnees de diagnostic et affiche des publicites personnalisees. Voici les cles pour reprendre le controle :

**Historique d'activite** :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\System
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `PublishUserActivities` | REG_DWORD | `0` | Desactive le partage de l'historique d'activite |

**Donnees de diagnostic** :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `AllowTelemetry` | REG_DWORD | `1` | Niveau minimum de telemetrie (obligatoire) |

!!! note "Limite"
    Vous ne pouvez pas mettre `AllowTelemetry` a `0` sur les editions Home et Pro. La valeur `1` (Donnees de diagnostic obligatoires) est le minimum autorise. Seules les editions **Enterprise** et **Education** permettent la valeur `0`.

**Identifiant publicitaire** :

```
HKCU\Software\Microsoft\Windows\CurrentVersion\AdvertisingInfo
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `Enabled` | REG_DWORD | `0` | Desactive l'identifiant publicitaire |

```powershell
# Disable advertising ID tracking
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\AdvertisingInfo" /v Enabled /t REG_DWORD /d 0 /f
```

```title="Resultat attendu"
L'operation a reussi.
```

### Copilot

Microsoft a ajoute un assistant Copilot integre a Windows 11. Pour le desactiver :

```
HKCU\Software\Policies\Microsoft\Windows\WindowsCopilot
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `TurnOffWindowsCopilot` | REG_DWORD | `1` | Desactive completement Copilot |

```powershell
# Disable Windows Copilot
reg add "HKCU\Software\Policies\Microsoft\Windows\WindowsCopilot" /v TurnOffWindowsCopilot /t REG_DWORD /d 1 /f
```

```title="Resultat attendu"
L'operation a reussi.
```

**Redemarrage necessaire** : fermez et rouvrez votre session.

!!! tip "Pour annuler"
    ```powershell
    # Re-enable Windows Copilot
    reg delete "HKCU\Software\Policies\Microsoft\Windows\WindowsCopilot" /v TurnOffWindowsCopilot /f
    ```

    ```title="Resultat attendu"
    L'operation a reussi.
    ```

!!! quote "En resume"
    - Windows 11 a introduit de nouvelles cles pour Snap Layouts, le mode "Ne pas deranger", la confidentialite (telemetrie, identifiant publicitaire) et Copilot.
    - Ces nouvelles fonctionnalites peuvent etre desactivees ou ajustees via le registre quand les Parametres ne proposent pas l'option.

---

## Ce qui ne marche plus

Attention : certaines astuces de registre tres populaires sous Windows 10 **ne fonctionnent plus** sous Windows 11. Microsoft a reecrit une grande partie de l'Explorateur (`explorer.exe`) avec un nouveau code, et les anciennes cles sont tout simplement ignorees.

| Ancienne astuce (Windows 10) | Statut sous Windows 11 |
|------------------------------|----------------------|
| Deplacer la barre des taches en haut / a gauche / a droite | :material-close-circle:{ .red } Ne fonctionne plus |
| Petites icones dans la barre des taches | :material-close-circle:{ .red } Option supprimee |
| Menu Demarrer classique (via registre) | :material-close-circle:{ .red } Ne fonctionne plus |
| Tuiles dynamiques (Live Tiles) | :material-close-circle:{ .red } Supprimees definitivement |
| Chronologie (Timeline) | :material-close-circle:{ .red } Fonctionnalite retiree |
| Mode tablette | :material-close-circle:{ .red } Remplace par un comportement automatique |

!!! warning "Pourquoi ces astuces ne marchent plus ?"
    Windows 11 utilise un **nouveau Shell** (le composant qui gere le Bureau, la barre des taches et le menu Demarrer). Le code a ete reecrit depuis zero pour certaines parties. Les anciennes cles de registre pointent vers du code qui **n'existe plus**.

    C'est comme essayer d'utiliser la telecommande d'une ancienne television sur un modele completement different : les boutons sont les memes, mais l'electronique a l'interieur a change.

!!! tip "Avant d'appliquer une astuce trouvee en ligne"
    Verifiez toujours la **date** de l'article ou du tutoriel. Si c'est un article ecrit pour Windows 10 (avant octobre 2021), il y a de fortes chances que l'astuce ne fonctionne pas sous Windows 11. Consultez le chapitre 10 pour apprendre a evaluer la fiabilite des tutoriels.

!!! quote "En resume"
    - Plusieurs astuces populaires de Windows 10 ne fonctionnent plus : deplacer la barre des taches, petites icones, menu Demarrer classique, tuiles dynamiques.
    - Le Shell de Windows 11 a ete reecrit ; les anciennes cles de registre pointent vers du code qui n'existe plus.

---

## Mises a jour majeures

Windows 11 recoit des mises a jour majeures une fois par an. Chaque version peut modifier le comportement de certaines cles de registre.

### Comment verifier votre version

```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion
```

| Valeur | Ce qu'elle indique |
|--------|-------------------|
| `DisplayVersion` | Votre version (ex : `23H2`, `24H2`) |
| `CurrentBuild` | Le numero de build exact (ex : `22631`) |
| `UBR` | Le numero de revision apres les mises a jour cumulatives |

```powershell
# Check Windows 11 version details
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v DisplayVersion
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v CurrentBuild
```

```title="Resultat attendu"
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion
    DisplayVersion    REG_SZ    24H2

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion
    CurrentBuild    REG_SZ    26100
```

### Windows 11 22H2 (septembre 2022)

La premiere grosse mise a jour de Windows 11. Changements notables :

- :material-folder: L'Explorateur de fichiers a recu des onglets (pas de cle registre pour les desactiver proprement)
- :material-menu: Le menu de debordement de la barre des taches a ete modifie
- :material-drag: Le glisser-deposer sur la barre des taches est revenu (il manquait a la sortie de Windows 11)

### Windows 11 23H2 (octobre 2023)

La mise a jour qui a rendu beaucoup d'astuces registre **obsoletes** (dans le bon sens) :

- :material-dock-bottom: Le "Ne jamais regrouper" les boutons de la barre des taches est revenu **nativement** dans les Parametres — plus besoin du hack `TaskbarGlomLevel`
- :material-robot: Copilot est apparu comme fonctionnalite integree (d'ou la cle `TurnOffWindowsCopilot`)
- :material-palette: Nouvelles options de personnalisation du menu Demarrer accessibles depuis les Parametres

### Windows 11 24H2 (octobre 2024)

La version la plus recente au moment de l'ecriture. Changements importants :

- :material-shield-check: Nouvelles exigences de securite renforcees pour certaines fonctionnalites
- :material-robot-off: Copilot est devenu une **application separee** (et non plus integre a la barre des taches) — la cle registre `TurnOffWindowsCopilot` reste fonctionnelle
- :material-cellphone-link: Nouvelles cles pour la fonctionnalite "Phone Link" et le partage avec les appareils mobiles
- :material-update: Mecanisme de mise a jour rapide ("checkpoint updates") qui reduit le temps d'installation

!!! note "Les versions evoluent"
    Chaque mise a jour peut rendre certaines astuces obsoletes ou en creer de nouvelles. C'est normal. Avant d'appliquer une modification, verifiez toujours qu'elle correspond a **votre** version de Windows 11.

!!! quote "En resume"
    - Verifiez votre version de Windows 11 avec `DisplayVersion` et `CurrentBuild` dans `HKLM\...\Windows NT\CurrentVersion`.
    - Chaque mise a jour majeure (22H2, 23H2, 24H2) peut rendre certaines astuces obsoletes ou en introduire de nouvelles : verifiez toujours la compatibilite avec votre version.

---

## Envie d'aller plus loin ?

Ce chapitre vous a donne les bases des modifications les plus courantes pour Windows 11. Si vous souhaitez approfondir :

- Le **chapitre 18 de la Bible du Registre** couvre l'evolution du registre a travers toutes les versions de Windows, y compris les differences techniques entre Windows 10 et Windows 11.
- Le **chapitre 22 de la Bible du Registre** explore les cles non documentees, y compris celles specifiques a Windows 11 qui ne sont dans aucune documentation officielle de Microsoft.

!!! quote "En resume"
    - Pour approfondir, la Bible du Registre couvre l'evolution a travers toutes les versions de Windows (chapitre 18) et les cles non documentees de Windows 11 (chapitre 22).

---

!!! quote "En resume"
    | Question | Reponse |
    |----------|---------|
    | Le registre est-il different sous Windows 11 ? | La structure est la meme, mais de nouvelles cles sont apparues et certaines anciennes sont ignorees |
    | Quel est le hack le plus populaire ? | Restaurer le menu contextuel classique (cle CLSID InprocServer32) |
    | Mon PC est-il compatible Windows 11 ? | Verifiez le TPM et Secure Boot dans le registre, ou utilisez `tpm.msc` |
    | Peut-on contourner les exigences ? | Oui via `AllowUpgradesWithUnsupportedTPMOrCPU`, mais avec des risques |
    | Les astuces Windows 10 marchent-elles ? | Pas toutes — le Shell a ete reecrit, certaines cles sont ignorees |
    | Comment connaitre ma version ? | `DisplayVersion` dans `HKLM\...\Windows NT\CurrentVersion` |
