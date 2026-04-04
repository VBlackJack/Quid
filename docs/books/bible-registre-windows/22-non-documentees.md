---
tags:
  - registre
  - non documenté
  - tweaks
  - avancé
---

# Cles non documentees

!!! abstract "Ce que vous allez apprendre"
    - Ce que signifie "cle non documentee" et comment elles sont decouvertes
    - Des tweaks d'interface pour personnaliser Windows en profondeur
    - Des optimisations de performances pour le reseau, le multimedia et le systeme
    - Des reglages de confidentialite pour limiter la telemetrie
    - Des cles utiles pour les developpeurs
    - Des ajustements reseau avances

!!! danger "Avertissement"
    Les cles listees ici ne sont **pas documentees officiellement** par Microsoft. Elles peuvent changer ou disparaitre a toute mise a jour de Windows. Creez **toujours** un point de restauration avant d'appliquer ces modifications.

---

## Qu'est-ce qu'une cle "non documentee" ?

Une cle non documentee est une entree du registre qui **n'apparait pas** dans la documentation officielle de Microsoft (Microsoft Learn, MSDN, KB articles). Elle a ete decouverte par :

- **Reverse engineering** -- desassemblage des binaires Windows
- **Process Monitor** -- observation des acces registre en temps reel
- **Recherche communautaire** -- experimentation systematique par des passionnes
- **Fuites internes** -- builds Insider et symboles de debug

!!! info "Non documentee ne veut pas dire interdite"
    Ces cles sont presentes dans le systeme. Microsoft ne les documente pas car elle se reserve le droit de les modifier sans preavis. Les utiliser est legal, mais vous le faites **a vos risques**.

!!! info "En resume"
    - Une cle non documentee est une entree du registre absente de la documentation officielle Microsoft
    - Elles sont decouvertes par reverse engineering, Process Monitor ou recherche communautaire
    - Microsoft peut les modifier ou les supprimer a toute mise a jour sans preavis

---

## Interface utilisateur

### Messages detailles au demarrage

Par defaut, Windows affiche "Bienvenue" pendant le demarrage. Vous pouvez activer des messages detailles qui montrent exactement ce que fait le systeme.

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `VerboseStatus` | REG_DWORD | `1` | Affiche les etapes detaillees au boot |

!!! tip "Utile pour le diagnostic"
    Au lieu de "Bienvenue", vous verrez des messages comme "Applying Computer Settings...", "Starting service X...". Indispensable pour diagnostiquer un demarrage lent.

**Versions** : Windows 7, 8, 10, 11

**Effets secondaires** : aucun. Modification purement cosmetique.

---

### Desactiver l'ecran de verrouillage

L'ecran de verrouillage (celui avec l'heure et la photo avant l'ecran de connexion) peut etre desactive.

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\Personalization
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `NoLockScreen` | REG_DWORD | `1` | Supprime l'ecran de verrouillage |

!!! warning "Windows 11"
    Sur Windows 11 version 22H2+, cette cle est **ignoree** sur les editions Home. Elle fonctionne toujours sur Pro/Enterprise.

**Versions** : Windows 8, 10, 11 (Pro/Enterprise)

**Effets secondaires** : les notifications de l'ecran de verrouillage ne sont plus visibles.

---

### Desactiver Copilot (Windows 11)

```
HKCU\Software\Policies\Microsoft\Windows\WindowsCopilot
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `TurnOffWindowsCopilot` | REG_DWORD | `1` | Desactive Copilot |

Pour une desactivation machine entiere (necessite les droits administrateur) :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsCopilot
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `TurnOffWindowsCopilot` | REG_DWORD | `1` | Desactive Copilot pour tous les utilisateurs |

**Versions** : Windows 11 23H2+

**Effets secondaires** : l'icone Copilot disparait de la barre des taches. La fonctionnalite est totalement inactive.

!!! info "Redemarrage"
    Un redemarrage ou une fermeture de session est necessaire.

---

### Desactiver la recherche web dans le menu Demarrer

Quand vous tapez dans le menu Demarrer, Windows envoie votre recherche a Bing. Pour desactiver :

```
HKCU\Software\Policies\Microsoft\Windows\Explorer
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `DisableSearchBoxSuggestions` | REG_DWORD | `1` | Supprime les resultats web de la recherche |

Alternative plus ancienne (Windows 10) :

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Search
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `BingSearchEnabled` | REG_DWORD | `0` | Desactive Bing dans la recherche |
| `CortanaConsent` | REG_DWORD | `0` | Supprime le consentement Cortana |

**Versions** : Windows 10 (toutes), Windows 11

**Effets secondaires** : la recherche ne retourne que des resultats locaux (applications, fichiers, parametres).

---

### Personnalisation de la barre des taches (Windows 11)

Windows 11 a introduit de nouvelles valeurs pour controler la barre des taches :

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced
```

| Valeur | Type | Donnees possibles | Effet |
|--------|:----:|:-----------------:|-------|
| `TaskbarSi` | REG_DWORD | `0` = petit, `1` = moyen, `2` = grand | Taille de la barre des taches |
| `TaskbarAl` | REG_DWORD | `0` = gauche, `1` = centre | Alignement des icones |
| `TaskbarMn` | REG_DWORD | `0` = masque, `1` = affiche | Bouton Chat (Teams) |
| `ShowCopilotButton` | REG_DWORD | `0` = masque, `1` = affiche | Bouton Copilot |

!!! warning "TaskbarSi"
    La valeur `TaskbarSi` a ete **supprimee** dans les builds recentes de Windows 11 (24H2+). Elle fonctionnait dans les premieres versions 22H2. Verifiez votre build avant de l'appliquer.

**Versions** : Windows 11 (variable selon la build)

---

### Desactiver les widgets

```
HKLM\SOFTWARE\Policies\Microsoft\Dsh
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `AllowNewsAndInterests` | REG_DWORD | `0` | Desactive le panneau Widgets |

Alternative pour l'utilisateur courant :

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `TaskbarDa` | REG_DWORD | `0` | Masque le bouton Widgets de la barre des taches |

**Versions** : Windows 11

**Effets secondaires** : le processus `Widgets.exe` ne se lance plus au demarrage.

---

### Desactiver la section "Recommandations" du menu Demarrer

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\Explorer
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `HideRecommendedSection` | REG_DWORD | `1` | Masque les recommandations dans le menu Demarrer |

!!! info "Edition requise"
    Cette politique fonctionne uniquement sur les editions **Pro et Enterprise**. Sur Home, elle est ignoree.

**Versions** : Windows 11 22H2+

!!! info "En resume"
    - Les cles d'interface permettent de personnaliser le demarrage (VerboseStatus), la barre des taches (TaskbarAl, TaskbarSi) et le menu Demarrer
    - Copilot, la recherche web Bing, les widgets et les recommandations peuvent etre desactives via des cles de strategie
    - Ces modifications sont a faible risque et generalement reversibles par suppression de la valeur

---

## Performances

### Desactiver les mitigations Spectre/Meltdown

!!! danger "Risque de securite majeur"
    Desactiver ces mitigations expose votre systeme a des **attaques spectre/meltdown**. Ne le faites que sur des machines de test isolees ou pour du benchmarking.

```
HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `FeatureSettingsOverride` | REG_DWORD | `3` | Desactive les mitigations Spectre |
| `FeatureSettingsOverrideMask` | REG_DWORD | `3` | Masque associe |

Pour reactiver (valeurs par defaut) :

| Valeur | Donnees |
|--------|:-------:|
| `FeatureSettingsOverride` | `0` |
| `FeatureSettingsOverrideMask` | `3` |

**Versions** : Windows 10 1803+, Windows 11

**Effets secondaires** : gain de performances mesurable (5-15 % selon la charge), mais **exposition a des vulnerabilites CPU**.

---

### MMCSS pour le multimedia

Le service *Multimedia Class Scheduler Service* (MMCSS) donne la priorite aux threads multimedia. Vous pouvez ajuster ses parametres :

```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile
```

| Valeur | Type | Donnees par defaut | Description |
|--------|:----:|:------------------:|-------------|
| `NetworkThrottlingIndex` | REG_DWORD | `0x0000000a` (10) | Limite de paquets reseau par ms pendant le multimedia |
| `SystemResponsiveness` | REG_DWORD | `0x00000014` (20) | % CPU reserve au systeme (le reste va au multimedia) |

!!! tip "Pour l'audio professionnel"
    Passez `SystemResponsiveness` a `0` pour donner la priorite maximale au multimedia. Attention : les taches systeme (antivirus, indexation) seront ralenties.

```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\Tasks\Pro Audio
```

| Valeur | Type | Donnees recommandees | Description |
|--------|:----:|:--------------------:|-------------|
| `Priority` | REG_DWORD | `1` | Priorite du thread (1 = maximale) |
| `Scheduling Category` | REG_SZ | `High` | Categorie de planification |
| `SFIO Priority` | REG_SZ | `High` | Priorite des E/S fichiers |

**Versions** : Windows Vista+, 10, 11

---

### Desactiver le Network Throttling

Windows limite par defaut le debit reseau pour les applications multimedia. Pour supprimer cette limite :

```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `NetworkThrottlingIndex` | REG_DWORD | `0xffffffff` | Desactive completement le throttling reseau |

**Versions** : Windows Vista+, 10, 11

**Effets secondaires** : la lecture multimedia peut subir des micro-coupures si le reseau est tres sollicite.

---

### Desactiver l'algorithme de Nagle

L'algorithme de Nagle regroupe les petits paquets TCP pour optimiser la bande passante. Cela ajoute de la latence, problematique pour les jeux en ligne.

```
HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\{GUID-de-votre-interface}
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `TcpAckFrequency` | REG_DWORD | `1` | Acquitte chaque paquet immediatement |
| `TCPNoDelay` | REG_DWORD | `1` | Desactive l'algorithme de Nagle |

!!! example "Trouver le GUID de votre interface reseau"
    ```powershell
    # List network interface GUIDs
    Get-NetAdapter | Select-Object Name, InterfaceGuid | Format-Table -AutoSize
    ```

    ```title="Resultat attendu"
    Name         InterfaceGuid
    ----         -------------
    Ethernet     {A1B2C3D4-E5F6-7890-ABCD-EF1234567890}
    Wi-Fi        {12345678-ABCD-EF01-2345-678901234567}
    ```

**Versions** : Windows 7, 8, 10, 11

**Effets secondaires** : augmentation marginale de l'utilisation de bande passante. Gain de latence notable pour les jeux (reduction de 10-50 ms).

---

### Game Mode et GPU Scheduling

```
HKCU\Software\Microsoft\GameBar
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `AllowAutoGameMode` | REG_DWORD | `1` | Active le mode jeu automatique |
| `AutoGameModeEnabled` | REG_DWORD | `1` | Confirme l'activation |

GPU scheduling accelere par le materiel :

```
HKLM\SYSTEM\CurrentControlSet\Control\GraphicsDrivers
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `HwSchMode` | REG_DWORD | `2` | Active le GPU scheduling materiel |

Valeurs possibles :

| Donnees | Mode |
|:-------:|------|
| `1` | Desactive |
| `2` | Active |

**Versions** : Windows 10 2004+, Windows 11

**Effets secondaires** : necessite un GPU compatible (NVIDIA 10xx+, AMD RX 5000+, Intel Arc). Peut causer des instabilites avec certains anciens pilotes.

---

### SvcHostSplitThresholdInKB

Windows lance les services dans des processus `svchost.exe` partages ou individuels selon la memoire disponible. Ce seuil controle la separation :

```
HKLM\SYSTEM\CurrentControlSet\Control
```

| Valeur | Type | Donnees par defaut | Description |
|--------|:----:|:------------------:|-------------|
| `SvcHostSplitThresholdInKB` | REG_DWORD | RAM en Ko | Seuil de separation des services svchost |

!!! info "Comment ca marche"
    Si votre PC a 16 Go de RAM, la valeur par defaut est `16777216` (16 Go en Ko). Chaque service obtient son propre `svchost.exe`. Pour forcer le regroupement (moins de processus, moins de RAM utilisee) :

    ```powershell
    # Force service grouping (reduce svchost process count)
    reg add "HKLM\SYSTEM\CurrentControlSet\Control" /v SvcHostSplitThresholdInKB /t REG_DWORD /d 0x4000000 /f
    ```

    Cela force le regroupement a moins que le systeme ait plus de 64 Go de RAM.

**Versions** : Windows 10 1703+, Windows 11

**Effets secondaires** : le regroupement des services reduit l'isolation en cas de crash. Si un service plante, il peut entrainer les autres services du meme processus.

!!! info "En resume"
    - Desactiver les mitigations Spectre/Meltdown offre un gain de 5-15% mais expose a des vulnerabilites CPU critiques
    - MMCSS et le Network Throttling controlent la priorite multimedia et le debit reseau
    - L'algorithme de Nagle peut etre desactive par interface reseau pour reduire la latence en jeu
    - SvcHostSplitThresholdInKB controle le regroupement des services svchost

---

## Confidentialite

### Desactiver la telemetrie

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `AllowTelemetry` | REG_DWORD | `0` | Niveau minimal de telemetrie (Security) |

Niveaux disponibles :

| Donnees | Niveau | Description |
|:-------:|--------|-------------|
| `0` | Security | Minimum absolu (Enterprise/Education uniquement) |
| `1` | Basic | Donnees de diagnostic de base |
| `2` | Enhanced | Donnees ameliorees (supprime en Windows 11) |
| `3` | Full | Donnees completes (valeur par defaut) |

!!! warning "Edition Home/Pro"
    Sur les editions Home et Pro, le niveau `0` est **eleve automatiquement** a `1` (Basic). Le niveau Security n'est disponible que sur Enterprise et Education.

**Versions** : Windows 10, 11

---

### Desactiver l'identifiant publicitaire

```
HKCU\Software\Microsoft\Windows\CurrentVersion\AdvertisingInfo
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `Enabled` | REG_DWORD | `0` | Desactive l'identifiant publicitaire |

**Versions** : Windows 10, 11

**Effets secondaires** : les publicites dans les applications seront generiques (non ciblees).

---

### Desactiver l'historique d'activite

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\System
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `EnableActivityFeed` | REG_DWORD | `0` | Desactive le flux d'activites |
| `PublishUserActivities` | REG_DWORD | `0` | Empeche l'envoi d'activites au cloud |
| `UploadUserActivities` | REG_DWORD | `0` | Empeche le telechargement d'activites |

**Versions** : Windows 10 1803+, Windows 11

---

### Desactiver les suggestions de contenu

```
HKCU\Software\Microsoft\Windows\CurrentVersion\ContentDeliveryManager
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `SubscribedContent-338393Enabled` | REG_DWORD | `0` | Desactive les suggestions dans Parametres |
| `SubscribedContent-353694Enabled` | REG_DWORD | `0` | Desactive les suggestions dans le menu Demarrer |
| `SubscribedContent-353696Enabled` | REG_DWORD | `0` | Desactive les suggestions dans le menu Demarrer |
| `SilentInstalledAppsEnabled` | REG_DWORD | `0` | Empeche l'installation silencieuse d'apps |
| `SystemPaneSuggestionsEnabled` | REG_DWORD | `0` | Desactive les suggestions systeme |
| `SoftLandingEnabled` | REG_DWORD | `0` | Desactive les conseils Windows |

**Versions** : Windows 10, 11

---

### Desactiver les notifications de feedback

```
HKCU\Software\Microsoft\Siuf\Rules
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `NumberOfSIUFInPeriod` | REG_DWORD | `0` | Supprime les demandes de feedback |

**Versions** : Windows 10, 11

**Effets secondaires** : aucun. Vous ne verrez plus "Evaluez votre experience Windows".

---

### Desactiver Connected User Experiences

Le service *Connected User Experiences and Telemetry* (`DiagTrack`) collecte et envoie des donnees diagnostiques.

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `DisableEnterpriseAuthProxy` | REG_DWORD | `1` | Empeche l'authentification proxy pour la telemetrie |

Pour desactiver le service completement :

```powershell
# Disable the DiagTrack service
reg add "HKLM\SYSTEM\CurrentControlSet\Services\DiagTrack" /v Start /t REG_DWORD /d 4 /f
```

```title="Resultat attendu"
The operation completed successfully.
```

| Donnees `Start` | Mode |
|:-------:|------|
| `2` | Automatique (defaut) |
| `3` | Manuel |
| `4` | Desactive |

**Versions** : Windows 10, 11

**Effets secondaires** : les rapports de probleme Windows cessent de fonctionner. Certaines mises a jour de fonctionnalites peuvent echouer.

!!! info "En resume"
    - AllowTelemetry controle le niveau de donnees envoyees a Microsoft (0 = minimum, 3 = complet)
    - L'identifiant publicitaire, l'historique d'activite et les suggestions de contenu sont desactivables individuellement
    - Le service DiagTrack peut etre desactive completement mais impacte les rapports de probleme et certaines mises a jour

---

## Comportement systeme

### Desactiver le redemarrage automatique apres mise a jour

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `NoAutoRebootWithLoggedOnUsers` | REG_DWORD | `1` | Empeche le redemarrage si un utilisateur est connecte |

Complement :

```
HKLM\SOFTWARE\Microsoft\WindowsUpdate\UX\Settings
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `ActiveHoursStart` | REG_DWORD | `8` | Debut des heures actives (8h) |
| `ActiveHoursEnd` | REG_DWORD | `23` | Fin des heures actives (23h) |

**Versions** : Windows 10, 11

---

### Modifier le proprietaire et l'organisation enregistres

```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `RegisteredOwner` | REG_SZ | `Votre Nom` | Proprietaire affiche dans winver |
| `RegisteredOrganization` | REG_SZ | `Votre Societe` | Organisation affichee dans winver |

!!! example "Verification"
    ```powershell
    reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v RegisteredOwner
    ```

    ```title="Resultat attendu"
    HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion
        RegisteredOwner    REG_SZ    Votre Nom
    ```

**Versions** : toutes les versions de Windows

---

### Desactiver l'ecriture USB (WriteProtect)

Empeche toute ecriture sur les cles USB :

```
HKLM\SYSTEM\CurrentControlSet\Control\StorageDevicePolicies
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `WriteProtect` | REG_DWORD | `1` | Bloque l'ecriture sur les peripheriques USB |

!!! tip "La cle n'existe pas par defaut"
    Vous devez creer la cle `StorageDevicePolicies` si elle n'existe pas, puis creer la valeur `WriteProtect`.

**Versions** : Windows 7, 8, 10, 11

**Effets secondaires** : **tous** les peripheriques de stockage USB deviennent en lecture seule. Utile dans un contexte de securite d'entreprise.

---

### Desactiver l'AutoRun / AutoPlay

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `NoDriveTypeAutoRun` | REG_DWORD | `0xFF` | Desactive l'AutoRun sur tous les types de lecteurs |

Valeurs granulaires :

| Bit | Type de lecteur |
|:---:|-----------------|
| `0x04` | Lecteurs amovibles |
| `0x08` | Disques durs fixes |
| `0x10` | Lecteurs reseau |
| `0x20` | Lecteurs CD/DVD |
| `0x40` | Lecteurs RAM |
| `0xFF` | Tous les lecteurs |

**Versions** : Windows XP+, 7, 8, 10, 11

**Effets secondaires** : les CD/DVD d'installation ne se lancent plus automatiquement. Recommande pour la securite.

---

### Forcer un serveur NTP specifique

```
HKLM\SYSTEM\CurrentControlSet\Services\W32Time\Parameters
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `NtpServer` | REG_SZ | `pool.ntp.org,0x9` | Serveur NTP a utiliser |
| `Type` | REG_SZ | `NTP` | Force la synchronisation NTP pure |

!!! info "Le suffixe ,0x9"
    Le `0x9` apres le nom du serveur est un flag qui indique `SpecialPollInterval` + `Client`. C'est le mode recommande pour un poste autonome.

**Versions** : toutes les versions de Windows

---

### Activer/desactiver l'hibernation via le registre

```
HKLM\SYSTEM\CurrentControlSet\Control\Power
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `HibernateEnabled` | REG_DWORD | `0` | Desactive l'hibernation |
| `HibernateEnabled` | REG_DWORD | `1` | Active l'hibernation |

!!! tip "Commande equivalente plus simple"
    ```batch
    powercfg /hibernate off
    ```
    Cette commande modifie la meme cle et supprime le fichier `hiberfil.sys` (qui peut peser plusieurs Go).

**Versions** : Windows Vista+, 7, 8, 10, 11

!!! info "En resume"
    - Le redemarrage automatique apres mise a jour peut etre bloque par NoAutoRebootWithLoggedOnUsers
    - WriteProtect empeche toute ecriture sur les peripheriques USB (utile en securite d'entreprise)
    - NoDriveTypeAutoRun desactive l'AutoRun sur tous les types de lecteurs pour prevenir les infections
    - L'hibernation et le serveur NTP sont controlables via le registre

---

## Developpeur

### Activer les chemins longs (260 caracteres)

Windows limite historiquement les chemins a 260 caracteres (`MAX_PATH`). Cette limite peut etre levee :

```
HKLM\SYSTEM\CurrentControlSet\Control\FileSystem
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `LongPathsEnabled` | REG_DWORD | `1` | Autorise les chemins depassant 260 caracteres |

!!! warning "Application par application"
    L'application doit **aussi** declarer le support des chemins longs dans son manifeste (`longPathAware` = `true`). La cle registre seule ne suffit pas. Cependant, les outils modernes (Git, Node.js, Python) supportent deja les chemins longs.

**Versions** : Windows 10 1607+, Windows 11

---

### Activer UTF-8 par defaut

```
HKLM\SYSTEM\CurrentControlSet\Control\Nls\CodePage
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `ACP` | REG_SZ | `65001` | Page de code ANSI = UTF-8 |
| `OEMCP` | REG_SZ | `65001` | Page de code OEM = UTF-8 |
| `MACCP` | REG_SZ | `65001` | Page de code Mac = UTF-8 |

!!! danger "Instabilite potentielle"
    Modifier directement ces cles peut rendre certaines anciennes applications **inutilisables** (affichage de caracteres corrompus). Preferez la methode officielle :

    Parametres > Heure et langue > Langue et region > Parametres d'administration > Modifier les parametres regionaux > Cochez "Beta : utiliser Unicode UTF-8".

    Cette methode modifie les memes cles mais verifie la compatibilite.

**Versions** : Windows 10 1903+, Windows 11

---

### Mode developpeur

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `AllowDevelopmentWithoutDevLicense` | REG_DWORD | `1` | Active le mode developpeur |
| `AllowAllTrustedApps` | REG_DWORD | `1` | Active le sideloading |

**Versions** : Windows 10, 11

**Effets secondaires** : permet l'installation d'applications non signees par le Store, active Windows Device Portal, active Device Discovery.

---

### Configuration WSL

Le sous-systeme Windows pour Linux (WSL) stocke certains parametres dans le registre :

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Lxss
```

| Valeur | Type | Description |
|--------|:----:|-------------|
| `DefaultDistribution` | REG_SZ | GUID de la distribution par defaut |
| `DefaultVersion` | REG_DWORD | Version WSL par defaut (`1` ou `2`) |

Chaque distribution installee possede une sous-cle avec son propre GUID :

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Lxss\{GUID}
```

| Valeur | Type | Description |
|--------|:----:|-------------|
| `DistributionName` | REG_SZ | Nom de la distribution (ex: `Ubuntu`) |
| `BasePath` | REG_SZ | Chemin vers les fichiers de la distribution |
| `State` | REG_DWORD | Etat (`1` = installee, `3` = en cours d'installation) |
| `Version` | REG_DWORD | Version WSL de cette distribution |

!!! example "Lister les distributions WSL via le registre"
    ```powershell
    # List WSL distributions from the registry
    Get-ChildItem "HKCU:\Software\Microsoft\Windows\CurrentVersion\Lxss" |
        Get-ItemProperty |
        Select-Object DistributionName, BasePath, Version |
        Format-Table -AutoSize
    ```

    ```title="Resultat attendu"
    DistributionName BasePath                                              Version
    ---------------- --------                                              -------
    Ubuntu           C:\Users\User\AppData\Local\Packages\Canonical...         2
    Debian           C:\Users\User\AppData\Local\Packages\TheDebian...         2
    ```

**Versions** : Windows 10 1709+ (WSL1), Windows 10 2004+ (WSL2), Windows 11

---

### Virtualisation imbriquee Hyper-V

Pour activer la virtualisation imbriquee (executer Hyper-V dans une VM Hyper-V) :

```powershell
# Enable nested virtualization for a VM (run on the host)
Set-VMProcessor -VMName "MaVM" -ExposeVirtualizationExtensions $true
```

```title="Resultat attendu"
Aucune sortie si la commande reussit.
```

Le parametre est stocke dans la configuration de la VM, pas directement dans le registre. Cependant, la fonctionnalite globale Hyper-V est controlee par :

```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Virtualization
```

| Valeur | Type | Description |
|--------|:----:|-------------|
| `MinVmVersionForCpuBasedMitigations` | REG_SZ | Version minimale de VM pour les mitigations CPU |

**Versions** : Windows 10 Pro/Enterprise, Windows 11 Pro/Enterprise, Windows Server 2016+

!!! info "En resume"
    - LongPathsEnabled leve la limite historique de 260 caracteres sur les chemins de fichiers
    - Passer les pages de code a 65001 active UTF-8 par defaut mais peut casser des applications anciennes
    - Le mode developpeur et le sideloading se controlent via AppModelUnlock
    - WSL stocke la configuration de ses distributions dans le registre sous HKCU\...\Lxss

---

## Reseau

### Desactiver IPv6

!!! warning "Recommandation Microsoft"
    Microsoft **deconseille** de desactiver IPv6. Certains composants Windows dependent d'IPv6 meme en reseau IPv4 pur (Teredo, ISATAP, loopback). Ne desactivez que si vous avez un probleme specifique.

```
HKLM\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `DisabledComponents` | REG_DWORD | `0xFF` | Desactive completement IPv6 |

Valeurs granulaires :

| Donnees | Effet |
|:-------:|-------|
| `0x01` | Desactive IPv6 sur toutes les interfaces non-tunnel |
| `0x10` | Desactive IPv6 sur toutes les interfaces tunnel |
| `0x20` | Desactive IPv6 sur toutes les interfaces sauf loopback |
| `0xFF` | Desactive IPv6 completement |

**Versions** : Windows Vista+, 7, 8, 10, 11

**Effets secondaires** : peut casser DirectAccess, Teredo, le partage reseau domestique, et certaines applications Microsoft 365.

---

### Desactiver LLMNR

LLMNR (*Link-Local Multicast Name Resolution*) est un protocole de resolution de noms qui peut etre exploite pour des attaques de type *relay* :

```
HKLM\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `EnableMulticast` | REG_DWORD | `0` | Desactive LLMNR |

**Versions** : Windows Vista+, 7, 8, 10, 11

**Effets secondaires** : la resolution de noms locaux peut echouer dans des environnements sans DNS. Recommande en entreprise ou les noms sont resolus par un serveur DNS.

---

### Activer DNS over HTTPS (DoH)

```
HKLM\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `EnableAutoDoh` | REG_DWORD | `2` | Active DoH automatique |

Niveaux :

| Donnees | Comportement |
|:-------:|-------------|
| `0` | DoH desactive |
| `1` | DoH si disponible (fallback DNS classique) |
| `2` | DoH obligatoire (echoue si DoH indisponible) |

!!! info "Serveurs DNS compatibles"
    DoH ne fonctionne qu'avec des serveurs DNS qui supportent le protocole. Windows integre une liste :

    - Cloudflare : `1.1.1.1`, `1.0.0.1`
    - Google : `8.8.8.8`, `8.8.4.4`
    - Quad9 : `9.9.9.9`, `149.112.112.112`

**Versions** : Windows 10 21H2+, Windows 11

---

### Desactiver NetBIOS

NetBIOS est un protocole ancien qui represente un risque de securite en reseau :

```
HKLM\SYSTEM\CurrentControlSet\Services\NetBT\Parameters
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `NodeType` | REG_DWORD | `2` | Desactive NetBIOS sur toutes les interfaces |

Valeurs possibles :

| Donnees | Mode |
|:-------:|------|
| `1` | B-node (broadcast) |
| `2` | P-node (point-to-point, desactive broadcast NetBIOS) |
| `4` | M-node (mixte) |
| `8` | H-node (hybride, defaut) |

Pour desactiver NetBIOS par interface specifique :

```
HKLM\SYSTEM\CurrentControlSet\Services\NetBT\Parameters\Interfaces\Tcpip_{GUID}
```

| Valeur | Type | Donnees | Effet |
|--------|:----:|:-------:|-------|
| `NetbiosOptions` | REG_DWORD | `2` | Desactive NetBIOS sur cette interface |

**Versions** : Windows 2000+, 7, 8, 10, 11

**Effets secondaires** : les anciens peripheriques reseau (imprimantes, NAS) qui dependent de NetBIOS peuvent devenir inaccessibles par nom.

---

!!! info "En resume"
    Les cles non documentees offrent un controle fin de Windows, mais avec des risques :

    | Categorie | Risque moyen | Exemples |
    |-----------|:------------:|----------|
    | Interface | Faible | VerboseStatus, TaskbarAl, Widgets |
    | Confidentialite | Faible | AllowTelemetry, AdvertisingInfo |
    | Performances | Moyen | Nagle, NetworkThrottling, MMCSS |
    | Securite | Eleve | Spectre mitigations, WriteProtect |
    | Developpeur | Faible | LongPaths, UTF-8, WSL |
    | Reseau | Moyen | IPv6, LLMNR, NetBIOS |

    **Regles d'or** :

    1. Creez toujours un point de restauration avant
    2. Ne modifiez qu'une cle a la fois
    3. Notez la valeur d'origine pour pouvoir revenir en arriere
    4. Verifiez la compatibilite avec votre version de Windows
