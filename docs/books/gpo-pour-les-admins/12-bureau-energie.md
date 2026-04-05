---
description: Bureau et énergie via GPO — fond d'écran, barre des tâches Win11, kiosk Assigned Access, plans d'énergie
tags: [gpo-admins, bureau, energie, kiosk, assigned-access, fond-ecran, windows-11]
---

# Bureau et énergie via GPO

!!! abstract "Ce que vous allez apprendre"
    - Imposer un **fond d'écran, des icônes du bureau et une barre des tâches Windows 11** unifiés via GPO, avec les chemins ADMX exacts et les clés de registre correspondantes
    - Configurer l'**économiseur d'écran avec verrouillage** et éviter les conflits silencieux avec les plans d'énergie et Modern Standby
    - Déployer et cibler les **plans d'alimentation** par GPO avec les GUIDs natifs, et comprendre pourquoi forcer "Haute Performance" sur les laptops est une faute de production
    - Construire un **kiosk Assigned Access** (mono-application ou multi-applications) déployé via GPO, avec le compte dédié, le fichier XML de configuration et la vérification post-déploiement
    - Gérer la **mise à l'échelle HiDPI** pour les applications legacy non-DPI-aware via les clés de compatibilité et les surcharges de registre

!!! tip "Si vous ne retenez qu'une chose"
    Le plan d'alimentation **Haute Performance** est la première cause de batterie morte en 3 heures sur les laptops d'entreprise après un déploiement GPO de masse. Ciblez **toujours** les plans d'énergie agressifs avec une ILT sur le type de châssis (requête WMI `Win32_SystemEnclosure`) pour exclure les portables — avant d'appuyer sur le bouton.

---

## Contexte de production

La gestion du bureau via GPO couvre un spectre large : de la simple charte graphique (fond d'écran corporate) à la mise sous tutelle complète du poste (kiosk Assigned Access). Ces deux extrêmes mobilisent des outils différents — ADMX Desktop pour le fond d'écran, Assigned Access XML pour le kiosk — mais partagent la même logique de ciblage.

Windows 11 a rendu plusieurs paramètres de bureau moins accessibles qu'avant. La barre des tâches n'accepte plus de fichier LayoutModification.xml comme sous Windows 10 : le paramètre GPO `Start/StartLayout` reste actif pour le menu Démarrer, mais la personnalisation de la barre des tâches Win11 passe désormais par des clés de registre directes. Ce chapitre documente la situation réelle en production, pas l'idéal de la documentation Microsoft.

---

## :material-monitor-dashboard: Bureau géré : fond d'écran, icônes, barre des tâches Win11

### :material-image: Fond d'écran via GPO

Le fond d'écran est l'une des personnalisations les plus demandées en entreprise. Il s'impose via une GPO utilisateur.

**Chemin GPMC :**

```
Configuration utilisateur
  └─ Stratégies
       └─ Modèles d'administration
            └─ Bureau
                 └─ Bureau
                      └─ Papier peint du Bureau
```

**Paramètres à configurer :**

| Paramètre | Valeur recommandée |
|-----------|-------------------|
| Chemin du papier peint | `\\srv-infra\sysvol\Contoso.com\wallpapers\corporate.jpg` ou chemin UNC vers partage DFS |
| Style du papier peint | `Fill` (remplit l'écran, recommandé pour les résolutions multiples) |

**Clés de registre correspondantes :**

| Clé | Type | Valeur |
|-----|------|--------|
| `HKCU\SOFTWARE\Policies\Microsoft\Windows\Personalization\LockScreenImage` | REG_SZ | Chemin UNC ou local |
| `HKCU\Control Panel\Desktop\Wallpaper` | REG_SZ | Chemin UNC ou local |
| `HKCU\Control Panel\Desktop\WallpaperStyle` | REG_SZ | `10` (Fill), `6` (Fit), `2` (Stretch), `0` (Center) |

!!! warning "À surveiller — chemin UNC et disponibilité réseau"
    Si le fond d'écran pointe vers un chemin UNC, la résolution doit être accessible au moment de l'ouverture de session. Un poste hors réseau (VPN non connecté au démarrage) affichera un bureau noir. Privilégiez un chemin local copié via GPP `File` ou utilisez `\\%LOGONSERVER%\sysvol\...` avec un fallback local configuré via script de démarrage.

**Approche par GPP (plus robuste) :**

Pour les environnements où la connectivité réseau n'est pas garantie au logon, copiez d'abord le fichier localement via une préférence GPP, puis pointez la clé vers le chemin local.

```powershell title="Script de démarrage GPO — copie locale du fond d'écran"
# Deploy wallpaper to local disk before user logon
$source = "\\srv-infra\sysvol\Contoso.com\wallpapers\corporate.jpg"
$dest   = "C:\Windows\Web\Wallpaper\Corporate\corporate.jpg"

if (-not (Test-Path (Split-Path $dest))) {
    New-Item -ItemType Directory -Path (Split-Path $dest) -Force | Out-Null
}

if (-not (Test-Path $dest)) {
    Copy-Item -Path $source -Destination $dest -Force
}
```

---

### :material-desktop-classic: Icônes du bureau — restreindre via ADMX

Les icônes système (Corbeille, Ce PC, Réseau) se contrôlent indépendamment du fond d'écran. Les clés ADMX `Desktop` permettent de masquer ou forcer chaque icône.

**Chemin GPMC :**

```
Configuration utilisateur
  └─ Stratégies
       └─ Modèles d'administration
            └─ Bureau
                 └─ Bureau
```

**Paramètres ADMX disponibles :**

| Paramètre ADMX | Effet | Chemin de registre |
|----------------|-------|-------------------|
| Masquer l'icône Corbeille sur le Bureau | Supprime la Corbeille du bureau | `HKCU\SOFTWARE\Policies\Microsoft\Windows\Explorer\NoRecycleBinIcon` = 1 |
| Masquer l'icône Ce PC sur le Bureau | Supprime "Ce PC" | `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer\NoDesktopIcons` (bit 0) |
| Interdire à l'utilisateur d'ajouter des icônes sur le Bureau | Verrouille entièrement le bureau | `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer\NoAddingComponents` = 1 |
| Interdire à l'utilisateur de supprimer des icônes du Bureau | Empêche la suppression | `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer\NoClosing` = 1 |

!!! tip "Kiosk vs restriction partielle"
    Sur un poste standard, masquez uniquement la Corbeille et Ce PC pour la charte graphique. Si vous avez besoin d'un verrouillage total du bureau, utilisez Assigned Access (voir section dédiée ci-dessous) — les paramètres ADMX Desktop ne suffisent pas pour un kiosk production.

---

### :material-dock-bottom: Barre des tâches Windows 11 via GPO

Windows 11 a rompu la compatibilité avec le fichier `LayoutModification.xml` utilisé sous Windows 10 pour personnaliser la barre des tâches. En Windows 11, la personnalisation est gérée par des clés de registre sous `HKCU\SOFTWARE\Policies\Microsoft\Windows\Explorer`.

**Widgets (Actualités et intérêts) — désactivation :**

Le widget "Actualités et intérêts" en bas à gauche se désactive via une clé HKLM, donc applicable via GPO ordinateur sans dépendance au contexte utilisateur.

| Clé | Type | Valeur |
|-----|------|--------|
| `HKLM\SOFTWARE\Policies\Microsoft\Dsh\AllowNewsAndInterests` | REG_DWORD | `0` (désactivé) |

**Chemin GPMC (si ADMX Windows 11 disponible) :**

```
Configuration ordinateur
  └─ Stratégies
       └─ Modèles d'administration
            └─ Composants Windows
                 └─ Widgets
                      └─ Autoriser les widgets
```

**Valeur :** Désactivé

**Menu Démarrer — déployer un layout personnalisé (Windows 11 22H2+) :**

Le paramètre `Start/StartLayout` fonctionne toujours pour imposer un layout de menu Démarrer sur Windows 11. Il accepte un fichier JSON exporté depuis un poste de référence.

```powershell title="Export du layout Démarrer depuis un poste de référence"
# Export Start layout from reference machine (Windows 11 22H2+)
Export-StartLayout -Path "C:\Temp\StartLayout.json"
```

```
Configuration utilisateur
  └─ Stratégies
       └─ Modèles d'administration
            └─ Menu Démarrer et barre des tâches
                 └─ Disposition du menu Démarrer
```

Pointez vers le fichier JSON sur un partage accessible au logon.

**Autres restrictions barre des tâches utiles en production :**

| Clé de registre | Type | Valeur | Effet |
|-----------------|------|--------|-------|
| `HKCU\SOFTWARE\Policies\Microsoft\Windows\Explorer\NoPinningToTaskbar` | REG_DWORD | `1` | Interdit l'épinglage d'applications |
| `HKCU\SOFTWARE\Policies\Microsoft\Windows\Explorer\TaskbarNoNotification` | REG_DWORD | `1` | Masque la zone de notification |
| `HKCU\SOFTWARE\Policies\Microsoft\Windows\Explorer\NoTaskbarSystemNotification` | REG_DWORD | `1` | Supprime les bulles de notification système |
| `HKLM\SOFTWARE\Policies\Microsoft\Windows\Explorer\HideSCAMeetNow` | REG_DWORD | `1` | Masque le bouton "Se réunir maintenant" (Teams legacy) |

!!! quote "En résumé"
    - Fond d'écran : ADMX Desktop — `HKCU\SOFTWARE\Policies\Microsoft\Windows\Personalization\LockScreenImage` — pensez au fallback local si l'UNC peut être inaccessible au logon
    - Icônes système : ADMX Desktop, clés `NoRecycleBinIcon` et `NoDesktopIcons`
    - Widgets Windows 11 : `HKLM\SOFTWARE\Policies\Microsoft\Dsh\AllowNewsAndInterests = 0`
    - Layout Démarrer : fichier JSON + GPO `Start/StartLayout` — valable Windows 11 22H2+
    - La barre des tâches Win11 ne s'impose plus via LayoutModification.xml : passez par les clés de registre `HKCU\SOFTWARE\Policies\Microsoft\Windows\Explorer`

---

## :material-lock-clock: Économiseur d'écran et verrouillage

### :material-cog-outline: Paramètres GPMC de l'économiseur d'écran

L'économiseur d'écran avec verrouillage est le mécanisme traditionnel de sécurité de poste inactif. Son chemin GPMC est :

```
Configuration utilisateur
  └─ Stratégies
       └─ Modèles d'administration
            └─ Panneau de configuration
                 └─ Personnalisation
```

**Paramètres essentiels :**

| Paramètre ADMX | Valeur recommandée | Clé de registre |
|----------------|-------------------|----------------|
| Activer l'économiseur d'écran | Activé | `HKCU\SOFTWARE\Policies\Microsoft\Windows\Control Panel\Desktop\ScreenSaveActive` = `1` |
| Délai d'activation de l'économiseur d'écran | `900` (15 min) | `HKCU\SOFTWARE\Policies\Microsoft\Windows\Control Panel\Desktop\ScreenSaveTimeOut` = `900` |
| Protéger l'économiseur d'écran par un mot de passe | Activé | `HKCU\SOFTWARE\Policies\Microsoft\Windows\Control Panel\Desktop\ScreenSaverIsSecure` = `1` |
| Nom de l'économiseur d'écran | `scrnsave.scr` (écran noir, recommandé) | `HKCU\SOFTWARE\Policies\Microsoft\Windows\Control Panel\Desktop\SCRNSAVE.EXE` |

!!! tip "Économiseur d'écran recommandé en production"
    Utilisez `scrnsave.scr` (écran noir pur) plutôt que les économiseurs graphiques. Les économiseurs graphiques consomment GPU et CPU inutilement. En environnement RDS, un économiseur graphique sur 50 sessions simultanées représente une charge non négligeable.

---

### :material-power-sleep: Conflit entre économiseur d'écran et Modern Standby

C'est un piège de production courant que la documentation Microsoft mentionne rarement clairement.

**Comportement standard (S3 sleep) :**

Sur les postes avec un état de veille traditionnel S3, l'économiseur d'écran se déclenche selon le délai configuré, puis Windows entre en veille après le délai du plan d'énergie. Les deux coexistent sans conflit.

**Comportement Modern Standby (S0ix) :**

Sur les machines modernes (Surface Pro, laptops Intel Tiger Lake+, ARM), Windows utilise **Connected Standby (S0ix)** au lieu de S3. Dans ce mode :

- L'économiseur d'écran peut **ne jamais se déclencher** si le plan d'énergie fait entrer la machine en S0ix avant le délai de l'économiseur
- Le verrouillage par l'économiseur d'écran ne se produit alors **pas du tout**
- Windows active à la place le verrouillage par délai d'inactivité via `SignInOptions`

**Solution recommandée en environnement Modern Standby :**

Remplacez l'économiseur d'écran par le verrouillage natif Windows :

```
Configuration ordinateur
  └─ Stratégies
       └─ Modèles d'administration
            └─ Composants Windows
                 └─ Options de connexion Windows
                      └─ Exiger une connexion automatique sur le verrouillage de session
```

Et configurez le délai de verrouillage via :

| Clé | Type | Valeur |
|-----|------|--------|
| `HKLM\SOFTWARE\Policies\Microsoft\Windows\Personalization\NoLockScreen` | REG_DWORD | `0` (laisser l'écran de verrouillage actif) |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\InactivityTimeoutSecs` | REG_DWORD | `900` (15 min en secondes) |

!!! danger "Production — économiseur d'écran ≠ verrouillage sur Modern Standby"
    Si votre audit de sécurité exige un verrouillage après 15 minutes d'inactivité, vérifiez le type de veille de vos machines cibles avec `powercfg /availablesleepstates`. Sur les machines S0ix, un économiseur d'écran GPO peut donner une fausse impression de conformité alors que le verrouillage ne se produit jamais. Utilisez `InactivityTimeoutSecs` comme source de vérité.

!!! quote "En résumé"
    - Économiseur d'écran : ADMX Personnalisation — délai dans `ScreenSaveTimeOut`, mot de passe dans `ScreenSaverIsSecure`
    - Sur Modern Standby (S0ix) : l'économiseur d'écran peut ne jamais se déclencher — utilisez `InactivityTimeoutSecs` à la place
    - `scrnsave.scr` (écran noir) est le seul économiseur recommandé en production
    - Vérifiez le mode veille avec `powercfg /availablesleepstates` avant de valider la configuration

---

## :material-lightning-bolt: Plans d'énergie via GPO

### :material-battery-charging: Les trois GUIDs natifs Windows

Windows gère les plans d'énergie par des GUIDs. Ces GUIDs sont identiques sur toutes les installations Windows — ils ne sont pas générés aléatoirement.

| Plan | GUID | Usage recommandé |
|------|------|-----------------|
| Équilibré (Balanced) | `381b4222-f694-41f0-9685-ff5bb260df2e` | Postes de travail standard, laptops — **valeur par défaut** |
| Haute Performance | `8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c` | Serveurs, postes de travail fixes haute charge |
| Économie d'énergie | `a1841308-3541-4fab-bc81-f71556f20b4a` | Postes en veille prolongée, bornes avec contrainte électrique |

!!! warning "À surveiller — plans d'énergie créés par OEM"
    Les fabricants (Dell, HP, Lenovo) créent souvent leurs propres plans d'énergie avec des GUIDs différents. `powercfg /list` peut retourner 5 ou 6 plans sur un parc hétérogène. Les GUIDs ci-dessus sont les plans **natifs Windows** et sont garantis présents sur toute installation Windows 10/11.

---

### :material-cog: Chemin GPMC — Power Management ADMX

Les paramètres de plans d'énergie sont dans l'ADMX `Power` :

```
Configuration ordinateur
  └─ Stratégies
       └─ Modèles d'administration
            └─ Système
                 └─ Gestion de l'alimentation
```

Les sous-noeuds disponibles :

| Sous-noeud | Contenu |
|-----------|---------|
| Paramètres de mise en veille | Délai de mise en veille sur batterie/secteur, hibernation |
| Paramètres d'affichage | Délai d'extinction de l'écran |
| Paramètres de boutons et couvercle | Comportement bouton d'alimentation, fermeture du couvercle |
| Notifications | Alertes niveau de batterie |

**Pour forcer un plan spécifique**, le paramètre GPMC est :

```
Configuration ordinateur
  └─ Stratégies
       └─ Modèles d'administration
            └─ Système
                 └─ Gestion de l'alimentation
                      └─ Spécifier un plan d'alimentation actif personnalisé
```

**Valeur :** GUID du plan entre accolades — ex. `{381b4222-f694-41f0-9685-ff5bb260df2e}`

**Clé de registre correspondante :**

| Clé | Type | Valeur |
|-----|------|--------|
| `HKLM\SOFTWARE\Policies\Microsoft\Power\PowerSettings\ActivePowerScheme` | REG_SZ | GUID du plan |

---

### :material-powershell: Déploiement par script PowerShell (alternative GPP Registry)

Si l'ADMX Power Management ne couvre pas votre besoin exact, utilisez un script de démarrage GPO.

```powershell title="Forcer le plan Équilibré via script de démarrage"
# Set power plan to Balanced on all machines
$balanced_guid = "381b4222-f694-41f0-9685-ff5bb260df2e"

# Check if plan exists (may not be present on custom OEM images)
$plan = powercfg /list | Select-String -Pattern $balanced_guid
if ($plan) {
    powercfg /setactive $balanced_guid
    Write-EventLog -LogName Application -Source "GPO-Power" `
        -EventId 1001 -EntryType Information `
        -Message "Power plan set to Balanced: $balanced_guid"
} else {
    # Restore native plans if OEM image removed them
    powercfg /restoredefaultschemes
    powercfg /setactive $balanced_guid
}
```

```powershell title="Résultat attendu"
# powercfg /getactivescheme
# Power Scheme GUID: 381b4222-f694-41f0-9685-ff5bb260df2e  (Balanced)
```

---

### :material-hibernate: Fast Startup et interaction avec BitLocker

**Fast Startup** (démarrage rapide) est activé par défaut sur Windows 10/11. Il hybride l'arrêt et la mise en veille : lors de l'arrêt, le noyau est mis en hibernation pour accélérer le prochain démarrage.

Interaction critique avec BitLocker :

- Sur certains firmwares, Fast Startup empêche BitLocker de déclencher le contrôle TPM au démarrage
- En cas d'arrêt et redémarrage depuis un autre OS (dual-boot, WinPE), la partition système peut rester dans un état hybride

**Désactiver Fast Startup via GPO :**

```
Configuration ordinateur
  └─ Stratégies
       └─ Modèles d'administration
            └─ Système
                 └─ Gestion de l'alimentation
                      └─ Paramètres de mise en veille
                           └─ Exiger un mot de passe à la sortie du mode veille (secteur)
```

Ou directement via la clé de registre :

| Clé | Type | Valeur |
|-----|------|--------|
| `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Power\HiberbootEnabled` | REG_DWORD | `0` (désactivé) |

!!! danger "Production — Fast Startup + BitLocker + dual-boot"
    Si votre environnement inclut des postes avec dual-boot ou des démarrages depuis WinPE (PXE, MDT), désactivez Fast Startup systématiquement. Un arrêt avec Fast Startup actif laisse la partition NTFS dans un état "sale" pour les autres OS — et BitLocker peut se retrouver en mode de récupération au prochain démarrage Windows.

---

### :material-laptop-off: Le piège critique : Haute Performance sur les laptops

C'est la faute de production la plus fréquente dans ce domaine. L'explication complète mérite une attention particulière.

Le plan **Haute Performance** désactive toutes les optimisations d'économie d'énergie du processeur (`ACPI\PSD` = 0), maintient les cœurs au maximum de fréquence en permanence, et supprime les états de veille profonde. Sur un serveur ou un desktop, c'est le comportement voulu.

Sur un laptop de 60 Wh, ce plan consomme typiquement 25-45 W en charge bureautique, soit **une autonomie de 1h30 à 3h maximum**. Les utilisateurs constatent que leur laptop "ne tient plus la batterie" sans comprendre pourquoi — et le ticket arrive en DSI.

**Solution : ciblage ILT par type de châssis**

Utilisez un filtre ILT WMI sur la GPO qui force le plan Haute Performance pour n'appliquer la GPO qu'aux postes de type Desktop.

**Requête WMI ILT à utiliser :**

```
SELECT * FROM Win32_SystemEnclosure WHERE ChassisTypes="3" OR ChassisTypes="4" OR ChassisTypes="5" OR ChassisTypes="6" OR ChassisTypes="7"
```

| ChassisType | Type de machine |
|-------------|----------------|
| 3 | Desktop |
| 4 | Low Profile Desktop |
| 5 | Pizza Box |
| 6 | Mini Tower |
| 7 | Tower |
| 8 | Portable |
| 9 | Laptop |
| 10 | Notebook |

Dans GPMC, créez un filtre WMI sur la GPO "Haute Performance" et collez la requête ci-dessus. La GPO ne s'appliquera qu'aux machines dont `Win32_SystemEnclosure.ChassisTypes` correspond à un facteur de forme desktop.

```powershell title="Vérification locale du type de châssis"
# Check chassis type on a machine before targeting
(Get-WmiObject -Class Win32_SystemEnclosure).ChassisTypes
```

```powershell title="Résultat attendu"
# Desktop : [3] ou [4] ou [6] ou [7]
# Laptop  : [8] ou [9] ou [10]
```

!!! danger "Production — valider la requête WMI avant déploiement"
    Les valeurs `ChassisTypes` retournées par `Win32_SystemEnclosure` sont un tableau, pas une valeur scalaire. Sur certains constructeurs (notamment Dell), la valeur peut être `{8}` (avec accolades) plutôt que `8`. Validez la requête WMI sur un échantillon de votre parc avant de lier la GPO en production — sinon elle peut s'appliquer à zéro machine, ou à toutes.

!!! quote "En résumé"
    - Trois GUIDs natifs : Balanced `381b4222...`, High Performance `8c5e7fda...`, Power Saver `a1841308...`
    - ADMX Power Management : `Configuration ordinateur > Stratégies > Modèles d'administration > Système > Gestion de l'alimentation`
    - Fast Startup + BitLocker + dual-boot = problèmes de récupération : désactivez `HiberbootEnabled` sur ces postes
    - Haute Performance sur laptops = batterie morte en 3h : utilisez un filtre ILT WMI `Win32_SystemEnclosure ChassisTypes` pour ne cibler que les desktops

---

## :material-monitor-lock: Kiosk mode — Assigned Access

### :material-information-outline: Contexte et cas d'usage

Assigned Access est le mécanisme natif Windows pour verrouiller un compte à une seule application (mono-app kiosk) ou à un ensemble d'applications définies (multi-app kiosk). Les cas d'usage production typiques :

- Bornes d'accueil (application web en kiosk Edge, application de billetterie)
- Postes de production partagés (logiciel métier unique, sans accès au bureau)
- Postes de consultation (document viewer, catalogue produit)
- Remplacement de solutions de kiosk tierces onéreuses

Assigned Access s'intègre naturellement avec le Loopback Processing GPO (voir [chapitre sur le Loopback](../bible-gpo/10-loopback.md)) et avec la gestion des profils RDS (voir [chapitre RDS](11-rds.md)).

---

### :material-account-lock: Étape 1 — Créer le compte kiosk dédié

Le compte kiosk doit être un compte **local** dédié, sans droits administrateurs, avec un mot de passe qui n'expire pas. Ne réutilisez pas un compte AD existant — le compte AD peut être sujet à des GPO de l'OU utilisateur qui interféreront avec Assigned Access.

```powershell title="Création du compte kiosk local via script de démarrage GPO"
# Create dedicated local kiosk account
$kiosk_user = "KioskUser"
$kiosk_pass = ConvertTo-SecureString "K!0sk@2026!" -AsPlainText -Force

# Check if account already exists
if (-not (Get-LocalUser -Name $kiosk_user -ErrorAction SilentlyContinue)) {
    New-LocalUser -Name $kiosk_user `
                  -Password $kiosk_pass `
                  -FullName "Kiosk Account" `
                  -Description "Assigned Access kiosk account - DO NOT DELETE" `
                  -PasswordNeverExpires `
                  -UserMayNotChangePassword
    Add-LocalGroupMember -Group "Utilisateurs" -Member $kiosk_user
    Write-EventLog -LogName Application -Source "GPO-Kiosk" `
        -EventId 1010 -EntryType Information `
        -Message "Kiosk account created: $kiosk_user"
}
```

!!! warning "À surveiller — compte kiosk et GPO utilisateur AD"
    Si vous devez utiliser un compte AD (ex : contrainte d'audit), placez ce compte dans une OU dédiée avec un minimum de GPO utilisateur liées — idéalement zéro, sauf la GPO Assigned Access elle-même. Tout paramètre GPO utilisateur non prévu peut bloquer le lancement de l'application kiosk.

---

### :material-xml: Étape 2 — Configurer le fichier XML Assigned Access

#### Mono-application kiosk (single-app)

Le format XML single-app est simple. L'exemple ci-dessous configure Microsoft Edge en mode kiosk avec une URL cible.

```xml title="AssignedAccess_SingleApp.xml"
<?xml version="1.0" encoding="utf-8"?>
<AssignedAccessConfiguration
    xmlns="http://schemas.microsoft.com/AssignedAccess/2017/config"
    xmlns:rs5="http://schemas.microsoft.com/AssignedAccess/201810/config">
  <Profiles>
    <Profile Id="{9A2A490F-10F6-4764-974A-43B19E722C23}">
      <KioskModeApp AppUserModelId="Microsoft.MicrosoftEdge.Stable_8wekyb3d8bbwe!App"
                    rs5:ClassicAppPath=""
                    rs5:ClassicAppArguments="" />
    </Profile>
  </Profiles>
  <Configs>
    <Config>
      <Account>KioskUser</Account>
      <DefaultProfile Id="{9A2A490F-10F6-4764-974A-43B19E722C23}"/>
    </Config>
  </Configs>
</AssignedAccessConfiguration>
```

!!! tip "AppUserModelId pour Edge kiosk"
    Pour Edge en mode kiosk public (navigation sans historique, sans barre d'adresse modifiable), utilisez le paramètre Edge `--kiosk https://votre-url.com --edge-kiosk-type=public-browsing` via `ClassicAppArguments`. Cela donne un navigateur entièrement verrouillé sur une URL.

#### Multi-application kiosk

Le format multi-app requiert Windows 10 Enterprise/Education ou Windows 11 Pro for Workstations/Enterprise. Il permet de définir un shell personnalisé avec plusieurs applications autorisées.

```xml title="AssignedAccess_MultiApp.xml (structure de base)"
<?xml version="1.0" encoding="utf-8"?>
<AssignedAccessConfiguration
    xmlns="http://schemas.microsoft.com/AssignedAccess/2017/config"
    xmlns:rs5="http://schemas.microsoft.com/AssignedAccess/201810/config"
    xmlns:v3="http://schemas.microsoft.com/AssignedAccess/2020/config">
  <Profiles>
    <Profile Id="{A9A0C2C5-3B3D-4C4E-8F2A-1B2C3D4E5F60}">
      <AllAppsList>
        <AllowedApps>
          <!-- Microsoft Edge (Chromium) -->
          <App AppUserModelId="Microsoft.MicrosoftEdge.Stable_8wekyb3d8bbwe!App" rs5:AutoLaunch="true" />
          <!-- Calculator (example of allowed UWP app) -->
          <App AppUserModelId="Microsoft.WindowsCalculator_8wekyb3d8bbwe!App" />
          <!-- Classic Win32 app example -->
          <App rs5:ClassicAppPath="C:\Program Files\CompanyApp\app.exe" />
        </AllowedApps>
      </AllAppsList>
      <v3:Taskbar ShowTaskbar="true"/>
      <StartLayout>
        <![CDATA[
          <LayoutModificationTemplate
              xmlns="http://schemas.microsoft.com/Start/2014/LayoutModification"
              xmlns:defaultlayout="http://schemas.microsoft.com/Start/2014/FullDefaultLayout"
              xmlns:start="http://schemas.microsoft.com/Start/2014/StartLayout"
              Version="1">
            <LayoutOptions StartTileGroupCellWidth="6" />
            <DefaultLayoutOverride>
              <StartLayoutCollection>
                <defaultlayout:StartLayout GroupCellWidth="6">
                  <start:Group Name="Applications">
                    <start:Tile Size="2x2" Column="0" Row="0"
                                AppUserModelID="Microsoft.MicrosoftEdge.Stable_8wekyb3d8bbwe!App"/>
                  </start:Group>
                </defaultlayout:StartLayout>
              </StartLayoutCollection>
            </DefaultLayoutOverride>
          </LayoutModificationTemplate>
        ]]>
      </StartLayout>
    </Profile>
  </Profiles>
  <Configs>
    <Config>
      <Account>KioskUser</Account>
      <DefaultProfile Id="{A9A0C2C5-3B3D-4C4E-8F2A-1B2C3D4E5F60}"/>
    </Config>
  </Configs>
</AssignedAccessConfiguration>
```

---

### :material-cog-transfer: Étape 3 — Déployer la configuration Assigned Access via GPO

La configuration Assigned Access est stockée dans le registre. Déployez-la via GPP Registry ou un script de démarrage ordinateur.

**Méthode 1 : Script de démarrage GPO (recommandé pour les fichiers XML complexes)**

```powershell title="Script de démarrage GPO — déploiement Assigned Access"
# Deploy Assigned Access XML configuration
$xml_source = "\\srv-infra\sysvol\Contoso.com\kiosk\AssignedAccess_SingleApp.xml"
$xml_local  = "C:\Windows\System32\AssignedAccess\kiosk_config.xml"

# Ensure destination directory exists
if (-not (Test-Path (Split-Path $xml_local))) {
    New-Item -ItemType Directory -Path (Split-Path $xml_local) -Force | Out-Null
}

# Copy and apply configuration
Copy-Item -Path $xml_source -Destination $xml_local -Force

# Apply via CIM (Windows 10 1803+ / Windows 11)
$namespace = "root\cimv2\mdm\dmmap"
$class_name = "MDM_AssignedAccess"

try {
    $assigned_access = Get-CimInstance -Namespace $namespace -ClassName $class_name
    $new_config = Get-Content -Path $xml_local -Raw
    Set-CimInstance -CimInstance $assigned_access -Property @{ Configuration = $new_config }
    Write-EventLog -LogName Application -Source "GPO-Kiosk" `
        -EventId 1011 -EntryType Information `
        -Message "Assigned Access configuration applied from: $xml_local"
} catch {
    Write-EventLog -LogName Application -Source "GPO-Kiosk" `
        -EventId 1012 -EntryType Error `
        -Message "Failed to apply Assigned Access: $_"
}
```

**Méthode 2 : GPP Registry (pour configuration simple)**

La configuration Assigned Access peut être écrite directement dans le registre pour un déploiement sans script :

| Clé | Type | Valeur |
|-----|------|--------|
| `HKLM\SOFTWARE\Microsoft\Windows\AssignedAccess\Configuration` | REG_SZ | Contenu XML de configuration (encodé en une seule ligne) |

!!! danger "Production — Assigned Access et Loopback Processing"
    Sur les machines kiosk en domaine, activez systématiquement le **Loopback Processing en mode Replace** (voir [chapitre Loopback](../bible-gpo/10-loopback.md)) sur l'OU des machines kiosk. Sans Loopback, les GPO utilisateur de l'OU du compte `KioskUser` (ou son OU AD) peuvent s'appliquer en complément et déverrouiller des accès inattendus. Avec Replace, seules les GPO liées à l'OU de la machine kiosk sont actives pendant la session.

---

### :material-check-circle: Étape 4 — Vérification post-déploiement

```powershell title="Vérification de la configuration Assigned Access"
# Check current Assigned Access configuration via CIM
$namespace = "root\cimv2\mdm\dmmap"
$class_name = "MDM_AssignedAccess"

$config = Get-CimInstance -Namespace $namespace -ClassName $class_name
if ($config.Configuration) {
    Write-Host "[OK] Assigned Access is configured" -ForegroundColor Green
    # Parse to check the kiosk account
    [xml]$xml = $config.Configuration
    $account = $xml.AssignedAccessConfiguration.Configs.Config.Account
    Write-Host "[OK] Kiosk account: $account" -ForegroundColor Green
} else {
    Write-Host "[FAIL] Assigned Access is NOT configured on this machine" -ForegroundColor Red
}
```

```powershell title="Résultat attendu"
# [OK] Assigned Access is configured
# [OK] Kiosk account: KioskUser
```

**Vérification des Event IDs à surveiller :**

| Event ID | Source | Signification |
|----------|--------|--------------|
| `400` | `Microsoft-Windows-AssignedAccess/Admin` | Démarrage de la session kiosk réussi |
| `500` | `Microsoft-Windows-AssignedAccess/Admin` | Fermeture de la session kiosk |
| `31000` | `Microsoft-Windows-AssignedAccess/Operational` | Erreur de chargement du profil kiosk |

```powershell title="Lecture des événements Assigned Access"
# Check Assigned Access events on target machine
Get-WinEvent -LogName "Microsoft-Windows-AssignedAccess/Operational" `
             -MaxEvents 20 | Format-Table TimeCreated, Id, Message -AutoSize
```

!!! quote "En résumé"
    - Compte kiosk local dédié, sans expiration de mot de passe, sans droits admin
    - XML Assigned Access : single-app (plus simple, Edge kiosk) ou multi-app (Enterprise/Education requis)
    - Déploiement via CIM `MDM_AssignedAccess` ou GPP Registry — le script CIM est plus robuste
    - Loopback Processing Replace **obligatoire** sur l'OU des machines kiosk pour bloquer les GPO utilisateur parasites
    - Events à surveiller : `400` (session kiosk démarrée), `31000` (erreur de profil)

---

## :material-magnify-plus: HiDPI et mise à l'échelle pour les applications legacy

### :material-monitor-eye: Le problème des applications non-DPI-aware

Windows 10/11 sur écrans haute résolution (4K, 2560×1440) applique une mise à l'échelle automatique. Les applications modernes qui déclarent être DPI-aware gèrent leur propre mise à l'échelle — tout va bien.

Les applications **non-DPI-aware** (applications Win32 legacy, applications développées avant Windows 8.1) sont "virtualisées" par Windows : Windows leur présente une résolution fictive et agrandit le rendu au niveau GDI. Résultat : polices et boutons flous, voire interface illisible.

Ce problème est particulièrement fréquent dans les parcs avec des applications métier vieillissantes — ERP, logiciels de CFAO, terminaux AS/400.

---

### :material-cog-outline: Paramètre GPO SystemDpiSettings

Windows 11 introduit `SystemDpiSettings` pour forcer le comportement de mise à l'échelle DPI au niveau système.

| Clé | Type | Valeur |
|-----|------|--------|
| `HKLM\SOFTWARE\Policies\Microsoft\Windows\Display\EnablePerProcessSystemDpiForceDisable` | REG_DWORD | `1` (désactive la virtualisation DPI par processus) |
| `HKCU\Control Panel\Desktop\DpiScalingVer` | REG_DWORD | `4096` (force la mise à l'échelle système) |

**Paramètre GPMC pour le comportement DPI de l'écran de connexion :**

```
Configuration ordinateur
  └─ Stratégies
       └─ Modèles d'administration
            └─ Système
                 └─ Affichage
                      └─ Paramètres de mise à l'échelle DPI
```

---

### :material-application-cog: Surcharge DPI par application via clés de compatibilité

Windows permet de forcer un comportement DPI spécifique pour une application individuelle via les clés de compatibilité de registre. Cette approche est chirurgicale : elle n'affecte que l'application ciblée.

```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\Layers
```

| Valeur | Effet |
|--------|-------|
| `DPIUNAWARE` | Force l'application en mode non-DPI-aware (virtualisation GDI) |
| `GDIDPISCALING DPIUNAWARE` | Active la mise à l'échelle GDI pour une application non-DPI-aware |
| `HIGHDPIAWARE` | Déclare l'application comme DPI-aware (désactive la virtualisation) |

**Exemple de déploiement via GPP Registry :**

Pour forcer `GDIDPISCALING` sur une application legacy `legacyapp.exe` :

| Clé | Nom de valeur | Type | Données |
|-----|--------------|------|---------|
| `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\Layers` | `C:\Program Files\LegacyApp\legacyapp.exe` | REG_SZ | `GDIDPISCALING DPIUNAWARE` |

```powershell title="Déploiement de la surcharge DPI via script GPO"
# Force GDI DPI scaling for legacy application
$app_path = "C:\Program Files\LegacyApp\legacyapp.exe"
$reg_key  = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\Layers"

if (-not (Test-Path $reg_key)) {
    New-Item -Path $reg_key -Force | Out-Null
}

Set-ItemProperty -Path $reg_key -Name $app_path -Value "GDIDPISCALING DPIUNAWARE" -Type String
Write-Host "[OK] DPI scaling override applied for: $app_path" -ForegroundColor Green
```

```powershell title="Résultat attendu"
# [OK] DPI scaling override applied for: C:\Program Files\LegacyApp\legacyapp.exe
```

!!! warning "À surveiller — applications 64 bits vs 32 bits sur Windows 64 bits"
    Sur un Windows 64 bits, les applications 32 bits sont dans `C:\Program Files (x86)\`. Le chemin dans la clé `AppCompatFlags\Layers` doit correspondre **exactement** au chemin réel de l'exécutable — y compris l'espace dans `Program Files (x86)`. Une erreur de chemin entraîne un échec silencieux : la surcharge n'est pas appliquée.

!!! tip "Détecter les applications non-DPI-aware sur un parc"
    L'outil `procmon.exe` (Sysinternals) filtre sur `DpiAwareness` lors du lancement d'une application. En production, le plus simple est de collecter les retours utilisateurs ("polices floues sur...") et d'appliquer la surcharge ciblée plutôt que de faire un audit exhaustif.

!!! quote "En résumé"
    - Applications non-DPI-aware : Windows applique une virtualisation GDI — résultat flou sur écrans HiDPI
    - Surcharge par application : clé `AppCompatFlags\Layers` avec valeur `GDIDPISCALING DPIUNAWARE`
    - Le chemin de l'exécutable dans la clé doit être exact (attention aux applications 32 bits dans `Program Files (x86)`)
    - `SystemDpiSettings` GPO agit au niveau système — réservé aux environnements où toutes les applications sont legacy

---

## :material-check-all: Vérification globale de la configuration bureau et énergie

Une fois les GPO bureau et énergie déployées, un script de vérification centralisé permet de valider l'état de chaque poste.

```powershell title="Audit bureau et énergie — vérification post-déploiement"
# Comprehensive audit: desktop GPO and power plan configuration
param(
    [string]$ExpectedPlanGuid = "381b4222-f694-41f0-9685-ff5bb260df2e",
    [string]$ExpectedWallpaper = "C:\Windows\Web\Wallpaper\Corporate\corporate.jpg"
)

$results = @()

# --- Power plan check ---
$active_plan = powercfg /getactivescheme
if ($active_plan -match $ExpectedPlanGuid) {
    $results += "[OK] Power plan: Balanced ($ExpectedPlanGuid)"
} else {
    $results += "[FAIL] Power plan mismatch: $active_plan"
}

# --- Fast Startup check ---
$hiberboot = (Get-ItemProperty `
    "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Power" `
    -Name HiberbootEnabled -ErrorAction SilentlyContinue).HiberbootEnabled
if ($hiberboot -eq 0) {
    $results += "[OK] Fast Startup: disabled"
} elseif ($null -eq $hiberboot) {
    $results += "[WARN] Fast Startup: registry key not found (default = enabled)"
} else {
    $results += "[FAIL] Fast Startup: ENABLED (HiberbootEnabled=$hiberboot)"
}

# --- Wallpaper policy check ---
$wp_policy = (Get-ItemProperty `
    "HKCU:\SOFTWARE\Policies\Microsoft\Windows\Personalization" `
    -Name LockScreenImage -ErrorAction SilentlyContinue).LockScreenImage
if ($wp_policy) {
    $results += "[OK] Wallpaper policy defined: $wp_policy"
} else {
    $results += "[WARN] Wallpaper policy: not configured via GPO"
}

# --- Widgets disable check ---
$widgets = (Get-ItemProperty `
    "HKLM:\SOFTWARE\Policies\Microsoft\Dsh" `
    -Name AllowNewsAndInterests -ErrorAction SilentlyContinue).AllowNewsAndInterests
if ($widgets -eq 0) {
    $results += "[OK] Widgets: disabled"
} else {
    $results += "[WARN] Widgets: not disabled by policy"
}

# --- Assigned Access check ---
try {
    $aa_config = Get-CimInstance -Namespace "root\cimv2\mdm\dmmap" `
                                 -ClassName "MDM_AssignedAccess" `
                                 -ErrorAction Stop
    if ($aa_config.Configuration) {
        $results += "[OK] Assigned Access: configured"
    } else {
        $results += "[INFO] Assigned Access: not configured (non-kiosk machine)"
    }
} catch {
    $results += "[INFO] Assigned Access: CIM class not available"
}

# --- Chassis type (for power plan targeting validation) ---
$chassis = (Get-WmiObject Win32_SystemEnclosure).ChassisTypes
$results += "[INFO] Chassis type: $chassis"

# --- Output ---
$results | ForEach-Object {
    $color = if ($_ -match "^\[OK\]") { "Green" }
             elseif ($_ -match "^\[FAIL\]") { "Red" }
             elseif ($_ -match "^\[WARN\]") { "Yellow" }
             else { "Cyan" }
    Write-Host $_ -ForegroundColor $color
}
```

```powershell title="Résultat attendu — poste desktop standard"
# [OK] Power plan: Balanced (381b4222-f694-41f0-9685-ff5bb260df2e)
# [OK] Fast Startup: disabled
# [OK] Wallpaper policy defined: C:\Windows\Web\Wallpaper\Corporate\corporate.jpg
# [OK] Widgets: disabled
# [INFO] Assigned Access: not configured (non-kiosk machine)
# [INFO] Chassis type: 3
```

---

## Références croisées

- [Loopback Processing GPO](../bible-gpo/10-loopback.md) — indispensable pour les kiosks Assigned Access en domaine
- [Remote Desktop Services via GPO](11-rds.md) — gestion des bureaux en environnement RDS, profils FSLogix, restrictions de session
