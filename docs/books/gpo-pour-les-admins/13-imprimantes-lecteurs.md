---
description: Imprimantes, lecteurs réseau et partages via GPO — GPP Printers, Drive Maps, DFS, ILT ciblage, Print Management
tags: [gpo-admins, imprimantes, lecteurs, gpp, drive-maps, dfs, print-management, ilt]
---
# Imprimantes, lecteurs réseau et partages

!!! abstract "Ce que vous allez apprendre"
    - Déployer des imprimantes par département via GPP Printers avec ciblage ILT précis, sans toucher aux scripts de logon
    - Utiliser Print Management ADMX comme alternative simplifiée et comprendre quand choisir l'une ou l'autre méthode
    - Remplacer tous vos `net use` de logon script par des GPP Drive Maps avec variables d'environnement et ciblage par groupe AD
    - Configurer les paramètres client DFS via GPO pour contrôler les timeouts et le cache de références
    - Éviter le piège classique de la lettre `Z:` mappée pour tout le monde — l'escalade au DSI garantie si le ciblage ILT est absent

!!! tip "Si vous ne retenez qu'une chose"
    **Tout GPP Drive Map ou GPP Printer sans ciblage ILT s'applique à tous les utilisateurs du périmètre de la GPO sans exception.** Un lecteur `Z:` mappé sur l'UNC `\\srv-fichiers\direction` visible sur le poste d'un stagiaire n'est pas un bug GPO — c'est un administrateur qui a oublié l'ILT. Toujours ajouter une condition de groupe de sécurité en premier réflexe.

---

## Contexte de production

Les imprimantes et les lecteurs réseau sont les deux sujets qui génèrent le plus de tickets de support au moment des migrations de sites ou de renouvellements de parc. La raison est presque toujours la même : des scripts de logon écrits il y a dix ans, dont personne ne comprend plus la logique, qui tombent silencieusement en erreur sur les nouvelles versions de Windows.

GPP Printers et GPP Drive Maps sont les remplaçants officiels de ces scripts. Ils s'administrent depuis la GPMC, journalisent leurs erreurs dans l'Observateur d'événements, et supportent le ciblage ILT nativement. La migration depuis les scripts `net use` prend une demi-journée ; les gains de maintenabilité durent des années.


!!! quote "En résumé"
    - GPP Printers et GPP Drive Maps sont les remplaçants officiels de ces scripts.
    - La migration depuis les scripts net use prend une demi-journée ; les gains de maintenabilité durent des années.
    - Le contexte de production fixe les contraintes réelles de réseau, de portée et d’exploitation qui gouvernent tout le chapitre.
    - Retenez les hypothèses opérationnelles avant de choisir un modèle de liaison ou de déploiement.
---

## GPP Printers : déploiement d'imprimantes par département

### :material-printer: Les trois types d'imprimantes GPP

GPP Printers propose trois types de connexions, chacun adapté à un scénario distinct.

| Type GPP | Cas d'usage | Port / Protocole | Remarque |
|----------|-------------|-----------------|----------|
| **Shared Printer** | Imprimante publiée sur un print server Windows | UNC `\\srv-print\HP-RDC` | Type le plus courant en environnement AD |
| **TCP/IP Printer** | Imprimante directement accessible sur le réseau (IP fixe) | Port TCP/IP standard (9100) | Nécessite le pilote installé localement ou via GPO |
| **Local Printer** | Imprimante attachée localement sur un poste spécifique | LPT / COM / USB | Usage rare, kiosques ou postes industriels |

En entreprise, **Shared Printer est le choix par défaut**. Il s'appuie sur le print server pour la distribution des pilotes, ce qui évite de déployer les pilotes manuellement sur chaque poste.

### :material-cog-outline: Les quatre actions CRUD

Chaque item GPP Printers dispose d'une action qui détermine son comportement au moment de l'application.

| Action | Comportement | Quand l'utiliser |
|--------|-------------|-----------------|
| **Create** | Crée la connexion si elle n'existe pas. Ne modifie pas si elle existe déjà. | Déploiement initial |
| **Replace** | Supprime puis recrée — force la mise à jour du pilote. | Migration de print server |
| **Update** | Crée si absent, met à jour si présent (paramètres uniquement). | Modifications de config sans changer le pilote |
| **Delete** | Supprime la connexion imprimante. | Décommissionnement |

L'action **Replace** est utile lors d'une migration de print server (changement de nom UNC). Elle garantit que l'ancienne connexion est bien supprimée avant la création de la nouvelle.

### :material-map-marker-path: Chemin GPMC exact

Les items GPP Printers se trouvent dans la section **utilisateur** de la GPO (sauf cas de déploiement machine) :

```
Configuration utilisateur
  > Préférences
    > Paramètres du panneau de configuration
      > Imprimantes
```

Pour un déploiement **machine** (imprimante disponible quel que soit l'utilisateur connecté, par exemple une imprimante de département dans un open space) :

```
Configuration ordinateur
  > Préférences
    > Paramètres du panneau de configuration
      > Imprimantes
```

### :material-walk: Déploiement pas à pas : imprimante par département avec ILT

L'objectif : déployer l'imprimante `\\srv-print\HP-Compta` uniquement pour les membres du groupe AD `GRP-Compta`.

**Étape 1 — Créer un item GPP Printer**

1. Ouvrez la GPMC, éditez la GPO cible.
2. Naviguez jusqu'à `Configuration utilisateur > Préférences > Paramètres du panneau de configuration > Imprimantes`.
3. Clic droit > `Nouveau` > `Imprimante partagée`.
4. Renseignez :
   - **Action** : `Update`
   - **Chemin de partage** : `\\srv-print\HP-Compta`
   - **Définir cette imprimante comme imprimante par défaut** : cochez selon besoin
5. Ne cliquez pas encore sur OK.

**Étape 2 — Ajouter le ciblage ILT**

6. Cliquez sur l'onglet **Ciblage commun** (Common tab).
7. Cliquez sur **Ciblage...** (Targeting...).
8. Dans l'éditeur de ciblage, cliquez sur `Nouvel élément` > `Groupe de sécurité`.
9. Renseignez le groupe : `GRP-Compta` (ou son SID si vous préférez la robustesse au renommage).
10. Cliquez sur **OK** deux fois pour valider.

!!! warning "À surveiller"
    L'éditeur ILT utilise l'opérateur **ET** entre les conditions par défaut. Si vous ajoutez un filtre de groupe ET un filtre de site AD, seuls les membres du groupe situés dans ce site recevront l'imprimante. Vérifiez la logique booléenne en lisant la phrase récapitulative en bas de l'éditeur ILT.

**Étape 3 — Forcer l'application et vérifier**

```powershell title="Force-GPUpdate-And-Verify.ps1"
# Force GPO refresh on a target machine and check printer mappings
# Run from an elevated PowerShell session or via remote PSSession

param(
    [string]$ComputerName = $env:COMPUTERNAME
)

# Force Group Policy update
Invoke-Command -ComputerName $ComputerName -ScriptBlock {
    gpupdate /force /target:user | Out-Null
    Write-Host "GPO refresh complete on $env:COMPUTERNAME"
}

# List installed printers on the target machine
Invoke-Command -ComputerName $ComputerName -ScriptBlock {
    Get-Printer | Select-Object Name, DriverName, PortName, Shared |
        Sort-Object Name |
        Format-Table -AutoSize
}
```

```title="Résultat attendu"
GPO refresh complete on WS-COMPTA-01

Name                  DriverName             PortName       Shared
----                  ----------             --------       ------
HP-Compta             HP Universal Printing  \\srv-print\.. False
Microsoft Print to PDF Microsoft Print To PDF PORTPROMPT:  False
```

### :material-history: Comparaison avec les scripts logon `net use`

| Critère | Script logon (`net use lpt1`) | GPP Printers |
|---------|------------------------------|-------------|
| Visibilité GPMC | Aucune — script externe | Native dans la GPMC |
| Journalisation des erreurs | Aucune (sauf `echo` dans le script) | Observateur d'événements (GPP log) |
| Ciblage par groupe AD | Manuel dans le script | ILT natif |
| Imprimante par défaut | Manuel (`RUNDLL32 printui.dll`) | Case à cocher dans l'item |
| Support Windows 11 | Fonctionnel mais déprécié | Recommandé par Microsoft |
| Action Replace (migration) | Nécessite `net use /delete` puis `net use` | Action Replace intégrée |

Les scripts `net use lpt1` fonctionnent encore sur Windows 11. Mais ils ne journalisent rien, ne supportent pas l'ILT nativement, et sont impossibles à auditer depuis la GPMC. La migration vers GPP Printers est systématiquement recommandée lors d'un renouvellement de parc.

!!! quote "En résumé"
    - GPP Printers Shared Printer est le type standard pour les imprimantes de print server.
    - L'action Update convient au déploiement initial ; Replace est indispensable lors d'une migration de print server.
    - Le ciblage ILT par groupe AD est non négociable — sans lui, toutes les imprimantes s'appliquent à tout le monde.
    - Les scripts `net use lpt1` peuvent être migrés en GPP Printers sans interruption de service.

---

## Print Management ADMX : déploiement depuis le snap-in

### :material-printer-settings: Le snap-in Print Management

Print Management (`printmanagement.msc`) est le snap-in MMC dédié à la gestion centralisée des serveurs d'impression. Il propose une fonctionnalité native de déploiement via GPO, distincte de GPP Printers.

Le chemin GPMC correspondant :

```
Configuration utilisateur
  > Stratégies
    > Paramètres Windows
      > Imprimantes déployées
```

Cette section est alimentée **depuis le snap-in Print Management lui-même**, pas manuellement via la GPMC. La procédure :

1. Ouvrez `printmanagement.msc`.
2. Développez le nœud de votre print server.
3. Clic droit sur l'imprimante à déployer > `Déployer avec une stratégie de groupe...`.
4. Sélectionnez la GPO cible et indiquez si le déploiement est **par ordinateur** ou **par utilisateur**.
5. Cliquez sur `Ajouter`, puis `OK`.

### :material-scale-balance: GPP Printers vs Print Management ADMX

| Critère | GPP Printers | Print Management ADMX |
|---------|-------------|----------------------|
| Ciblage ILT | Oui (complet) | Non — pas de ciblage natif |
| Imprimante par défaut | Oui | Non |
| Administration depuis GPMC | Oui | Partielle (via snap-in Print Mgmt) |
| Support des variables (`%USERNAME%`) | Oui | Non |
| Simplicité de déploiement massif | Nécessite un item par imprimante | Déploiement depuis Print Mgmt en quelques clics |
| Audit centralisé | Observateur d'événements GPP | Journal Print Management |

**Quand utiliser Print Management ADMX :** déploiement massif d'imprimantes identiques pour tous les utilisateurs d'une OU, sans besoin de ciblage fin. Typiquement les imprimantes partagées de couloir ou de salle de réunion accessibles à tous.

**Quand utiliser GPP Printers :** dès qu'un ciblage par groupe, par site, ou par machine est nécessaire. C'est le cas dans 80% des environnements multi-départements.

### :material-backup-restore: Backup et migration avec `printbrm.exe`

Lors d'une migration de print server (remplacement physique, changement de nom), `printbrm.exe` permet de sauvegarder l'intégralité de la configuration d'un serveur d'impression et de la restaurer sur le nouveau serveur.

```powershell title="PrintServer-Migration.ps1"
# Backup all printers, drivers, ports and queues from a print server
# Run elevated on the SOURCE print server

# Step 1: Export the entire print server configuration
printbrm.exe -B -S \\srv-print-old -F "C:\PrintBackup\print-server-backup.printerExport"

Write-Host "Backup complete. File: C:\PrintBackup\print-server-backup.printerExport"
Write-Host "Transfer this file to the new print server, then run the restore command."
```

```powershell title="PrintServer-Restore.ps1"
# Restore print server configuration on the TARGET server
# Run elevated on the DESTINATION print server

printbrm.exe -R -S \\srv-print-new -F "\\srv-print-old\C$\PrintBackup\print-server-backup.printerExport"

Write-Host "Restore complete. Verify printers with: Get-Printer -ComputerName srv-print-new"
```

!!! danger "Production"
    `printbrm.exe` restaure les pilotes tels qu'ils ont été exportés. Si le nouveau serveur tourne sur une version de Windows Server différente (ex : 2016 → 2022), certains pilotes V3 peuvent ne pas se charger correctement. Vérifiez la compatibilité des pilotes avant la migration et prévoyez un téléchargement des versions V4 depuis les sites constructeurs. Testez la restauration sur un serveur de staging **avant** la nuit de bascule.

!!! quote "En résumé"
    - Print Management ADMX est plus simple pour un déploiement massif sans ciblage fin.
    - GPP Printers est indispensable dès qu'un ciblage ILT est requis.
    - `printbrm.exe` est l'outil officiel de migration de print server — testez toujours la restauration sur un environnement de staging.

---

## GPP Drive Maps : remplacer les `net use` de logon script

### :material-map: Fonctionnement des GPP Drive Maps

GPP Drive Maps est l'équivalent GPO des commandes `net use Z: \\srv-fichiers\partage`. Il gère les lettres de lecteur réseau dans la section Préférences de la GPO, avec journalisation native et support ILT.

Chemin GPMC :

```
Configuration utilisateur
  > Préférences
    > Paramètres Windows
      > Mappages de lecteurs
```

### :material-table: Paramètres clés d'un item Drive Map

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| **Action** | Create / Replace / Update / Delete | Même logique que GPP Printers |
| **Lettre de lecteur** | `Z:` | Lettre attribuée |
| **Chemin** | `\\srv-fichiers\partage` | UNC ou chemin DFS |
| **Etiquette** | `Partage commun` | Nom affiché dans l'explorateur Windows |
| **Reconnecter** | Coché | Reconnecte au démarrage de session — à cocher systématiquement |
| **Masquer/Afficher** | Afficher ce lecteur | Visibilité dans l'explorateur |

### :material-variable: Variables d'environnement dans les chemins UNC

GPP Drive Maps supporte les variables d'environnement Windows directement dans le champ chemin. Cela permet de construire des chemins dynamiques sans créer un item par utilisateur.

| Variable | Exemple de chemin | Résultats |
|----------|------------------|-----------|
| `%USERNAME%` | `\\srv-fichiers\users\%USERNAME%` | `\\srv-fichiers\users\jdupont` |
| `%USERDOMAIN%` | `\\srv-fichiers\%USERDOMAIN%\shared` | `\\srv-fichiers\contoso\shared` |
| `%COMPUTERNAME%` | `\\srv-fichiers\machines\%COMPUTERNAME%` | `\\srv-fichiers\machines\WS-001` |
| `%LOGONSERVER%` | `%LOGONSERVER%\netlogon` | `\\DC-PARIS\netlogon` |

!!! warning "À surveiller"
    Les variables d'environnement GPP sont résolues **au moment de l'application de la GPO**, côté client. Si `%USERNAME%` n'est pas encore disponible dans le contexte d'exécution (rare sur les GPO utilisateur, mais possible sur les GPO machine), le chemin sera littéralement `\\srv-fichiers\users\%USERNAME%` — ce qui générera une erreur de connexion silencieuse. Vérifiez toujours en regardant les logs GPP dans l'Observateur d'événements.

### :material-walk: Déploiement pas à pas : lecteur Z: par département avec ILT

Objectif : mapper `Z:` sur `\\srv-fichiers\compta` uniquement pour les membres de `GRP-Compta`.

**Étape 1 — Créer l'item Drive Map**

1. GPMC > éditez la GPO cible.
2. `Configuration utilisateur > Préférences > Paramètres Windows > Mappages de lecteurs`.
3. Clic droit > `Nouveau` > `Lecteur mappé`.
4. Renseignez :
   - **Action** : `Update`
   - **Lettre** : `Z:`
   - **Chemin** : `\\srv-fichiers\compta`
   - **Etiquette** : `Comptabilité`
   - **Reconnecter** : coché
5. Onglet **Ciblage commun** > `Ciblage...`.

**Étape 2 — Configurer le ciblage ILT**

6. `Nouvel élément` > `Groupe de sécurité`.
7. Groupe : `GRP-Compta`.
8. Cliquez **OK** pour fermer l'éditeur ILT.
9. Cliquez **OK** pour valider l'item.

**Étape 3 — Vérification post-déploiement**

```powershell title="Verify-DriveMaps.ps1"
# Verify mapped drives on a remote workstation after GPO application
# Run from a machine with RSAT and remote PS access

param(
    [string]$ComputerName = $env:COMPUTERNAME,
    [string]$UserName     = ""
)

# List all PSDrives of type FileSystem on the remote machine
Invoke-Command -ComputerName $ComputerName -ScriptBlock {
    Get-PSDrive -PSProvider FileSystem |
        Where-Object { $_.Root -like '\\*' } |
        Select-Object Name, Root, Description |
        Format-Table -AutoSize
}

# Check the GPP Drive Maps log in Event Viewer (event source: Group Policy Preferences Client)
Invoke-Command -ComputerName $ComputerName -ScriptBlock {
    Get-WinEvent -LogName "Microsoft-Windows-GroupPolicy/Operational" |
        Where-Object { $_.Message -match "Drive Maps" } |
        Select-Object TimeCreated, LevelDisplayName, Message |
        Select-Object -First 10
}
```

```title="Résultat attendu"
Name  Root                     Description
----  ----                     -----------
C     C:\
Z     \\srv-fichiers\compta    Comptabilité
```

### :material-wifi-off: Comportement des lecteurs déconnectés

Quand un lecteur mappé via GPP n'est pas joignable au moment de la connexion (poste en déplacement, VPN non connecté, serveur en maintenance), Windows affiche le lecteur avec une icône rouge en croix dans l'explorateur.

Ce comportement est normal et attendu. Le lecteur se reconnecte automatiquement dès que le chemin UNC devient accessible, sans nécessiter de déconnexion/reconnexion de session.

L'option **Reconnecter** dans l'item GPP contrôle cette tentative de reconnexion automatique. Il est fortement recommandé de la cocher pour tous les lecteurs de données.

!!! danger "Production"
    Un Drive Map avec **action Create** et **Reconnecter non coché** crée le lecteur une seule fois au premier logon et ne tente jamais de le reconnecter. Après un redémarrage sur un réseau différent (VPN, WiFi visiteur), le lecteur disparaît définitivement jusqu'au prochain `gpupdate`. Utilisez toujours **action Update + Reconnecter coché** pour les lecteurs de production.

!!! quote "En résumé"
    - GPP Drive Maps remplace avantageusement `net use` dans les scripts logon.
    - Les variables `%USERNAME%`, `%USERDOMAIN%` permettent des chemins dynamiques sans multiplication d'items.
    - ILT par groupe de sécurité est obligatoire pour tout lecteur non universel.
    - Action Update + Reconnecter coché est la combinaison de référence pour la production.

---

## GPP Shares : partages sur les clients

### :material-folder-network: Cas d'usage des GPP Shares

GPP Shares ne gère pas les connexions à des partages distants — c'est le rôle de Drive Maps. GPP Shares crée ou modifie des **partages locaux sur les machines cibles** elles-mêmes.

Les cas d'usage sont limités mais réels :

- **Kiosque ou poste de collecte** : créer un partage local `\\kiosque-01\depot` temporaire que d'autres machines du réseau peuvent utiliser.
- **Standardisation des partages admin** : forcer le partage `C$` avec des permissions spécifiques sur des serveurs membres.
- **Déploiement d'un partage applicatif local** : certaines applications legacy s'attendent à trouver un chemin UNC local.

Chemin GPMC :

```
Configuration ordinateur
  > Préférences
    > Paramètres Windows
      > Partages réseau
```

### :material-lock-check: NTFS vs permissions de partage : la distinction critique

GPP Shares configure les **permissions de partage** (Share permissions), pas les permissions NTFS. Ce sont deux couches de sécurité indépendantes.

| Couche | Portée | Configurée via |
|--------|--------|----------------|
| **Permissions de partage** | Accès réseau uniquement | GPP Shares, `net share`, GPMC |
| **Permissions NTFS** | Accès local ET réseau | Explorateur > Propriétés > Sécurité, icacls, GPP Files |

En pratique, l'accès effectif est **le plus restrictif des deux**. Si les permissions de partage autorisent `Tout le monde : Contrôle total` mais que les ACL NTFS refusent l'accès, l'utilisateur sera bloqué.

La bonne pratique : laisser les permissions de partage larges (`Tout le monde : Lecture` ou `Tout le monde : Contrôle total`) et contrôler finement via les ACL NTFS. Cela simplifie la gestion et évite les conflits entre les deux couches.

!!! warning "À surveiller"
    GPP Shares ne peut pas modifier les ACL NTFS d'un dossier. Pour combiner création d'un partage ET configuration des permissions NTFS, utilisez GPP Shares pour la couche partage et un script PowerShell via GPP Scheduled Tasks (ou GPP Files) pour configurer les ACL NTFS avec `icacls` ou `Set-Acl`.

!!! quote "En résumé"
    - GPP Shares crée des partages locaux sur les machines — pas des connexions à des partages distants.
    - Cas d'usage principaux : kiosques, postes de collecte, standardisation des partages admin.
    - GPP Shares ne touche pas aux ACL NTFS — utilisez un script séparé pour les deux couches.

---

## DFS et GPO : configuration du client

### :material-sitemap: Paramètres client DFS configurables via GPO

DFS (Distributed File System) est transparent pour les utilisateurs finaux dans la majorité des cas. Mais pour les environnements multi-sites avec des latences élevées ou des Offline Files activés, plusieurs paramètres client DFS méritent d'être ajustés via GPO.

Les paramètres DFS client se trouvent dans le registre à :

```
HKLM\SYSTEM\CurrentControlSet\Services\Mup\Parameters
```

Les valeurs les plus utiles :

| Valeur de registre | Type | Valeur par défaut | Description |
|-------------------|------|------------------|-------------|
| `DisableDfs` | REG_DWORD | `0` | Désactive entièrement le client DFS si mis à `1` |
| `SysvopTimeout` | REG_DWORD | `30` | Timeout (en secondes) pour les opérations SYSVOL DFS |
| `ReferralCacheSizeInEntries` | REG_DWORD | `1000` | Nombre maximum d'entrées dans le cache de références DFS |

Pour les paramètres de namespace DFS (Timeout, intervalle de re-référence), le chemin est différent :

```
HKLM\SOFTWARE\Policies\Microsoft\System\DFSClient
```

| Valeur de registre | Type | Description |
|-------------------|------|-------------|
| `DfsDnsConfig` | REG_DWORD | `1` = force la résolution DNS plutôt que NetBIOS pour les références DFS |
| `MaxCacheEntryAgeInSeconds` | REG_DWORD | Durée de vie du cache de références (défaut : 300 secondes) |
| `MaxReferralPerFolderTarget` | REG_DWORD | Nombre maximum de cibles par référence de dossier |

### :material-cog: Configurer les paramètres DFS client via GPP Registry

Les paramètres DFS client ne disposent pas de modèles ADMX natifs pour toutes les valeurs. On les configure via GPP Registry.

Chemin GPMC :

```
Configuration ordinateur
  > Préférences
    > Paramètres Windows
      > Registre
```

```powershell title="Set-DFSClientGPO.ps1"
# Configure DFS client registry values via Set-GPRegistryValue
# Run on a domain controller or machine with RSAT GroupPolicy module

#Requires -Modules GroupPolicy

$GPOName = "GPO-DFS-Client-Settings"
$gpo = Get-GPO -Name $GPOName -ErrorAction Stop

# Force DNS resolution for DFS referrals (recommended in multi-site environments)
Set-GPRegistryValue -Guid $gpo.Id `
    -Key       "HKLM\SOFTWARE\Policies\Microsoft\System\DFSClient" `
    -ValueName "DfsDnsConfig" `
    -Type      DWord `
    -Value     1

# Extend referral cache lifetime to 10 minutes (600 seconds) to reduce DC load
Set-GPRegistryValue -Guid $gpo.Id `
    -Key       "HKLM\SOFTWARE\Policies\Microsoft\System\DFSClient" `
    -ValueName "MaxCacheEntryAgeInSeconds" `
    -Type      DWord `
    -Value     600

Write-Host "DFS client GPO values set on '$GPOName'." -ForegroundColor Green
Write-Host "Verify on client with: reg query HKLM\SOFTWARE\Policies\Microsoft\System\DFSClient"
```

### :material-sync-alert: Offline Files + DFS : la combinaison dangereuse

Offline Files (CSC — Client Side Caching) et DFS Namespaces peuvent coexister, mais leur interaction est complexe et source d'incidents.

Le problème principal : quand un utilisateur travaille hors connexion via Offline Files sur un chemin DFS, et qu'une cible DFS bascule vers un autre serveur (failover DFS), la synchronisation au retour en ligne peut échouer silencieusement ou produire des conflits de version.

!!! danger "Production"
    **Ne jamais activer Offline Files sur des partages DFS qui ont plusieurs cibles actives en réplication DFS-R.** La combinaison Offline Files + DFS-R multi-cible avec synchronisation simultanée est un vecteur connu de corruption de données. Si vous avez besoin d'accès hors ligne sur des namespaces DFS, utilisez **Work Folders** (Windows Server 2012 R2+) ou **OneDrive Known Folder Move** — ces solutions sont conçues pour la synchronisation. Offline Files sur DFS ne l'est pas.

    Si vous maintenez une configuration Offline Files + DFS existante, surveillez le journal `Microsoft-Windows-OfflineFiles/Operational` pour détecter les conflits de synchronisation.

!!! quote "En résumé"
    - Les paramètres client DFS se configurent via GPP Registry sur `HKLM\SYSTEM\CurrentControlSet\Services\Mup\Parameters` et `HKLM\SOFTWARE\Policies\Microsoft\System\DFSClient`.
    - `DfsDnsConfig = 1` est recommandé dans les environnements multi-sites pour éviter les résolutions NetBIOS aléatoires.
    - N'activez jamais Offline Files sur des partages DFS multi-cibles en DFS-R — le risque de corruption de données est réel.

---

## Le piège classique : Drive Map sans ILT et l'escalade garantie

### :material-alert-decagram: Le scénario

Un administrateur crée une GPO `GPO-Lecteurs-Direction` liée à l'OU `Computers` racine du domaine. Il ajoute un item Drive Map : `Z:` → `\\srv-fichiers\direction`. Pas de ciblage ILT. Filtrage de sécurité par défaut : `Utilisateurs authentifiés`.

Résultat : **le lecteur `Z:` pointant vers `\\srv-fichiers\direction` est mappé sur tous les postes du domaine**, y compris les postes des stagiaires, des techniciens d'accueil et du personnel d'entretien.

Deux heures plus tard, un stagiaire qui explore son poste ouvre `Z:` et tombe sur les fichiers de rémunération de la direction. Le DSI reçoit un appel.

Ce scénario se produit plusieurs fois par an dans des organisations de toutes tailles.

### :material-shield-check: La configuration ILT correcte

La règle absolue : **tout Drive Map sur un partage sensible doit avoir un ciblage ILT avec condition de groupe de sécurité**.

Configuration ILT correcte pour l'exemple ci-dessus :

1. Item Drive Map `Z:` → `\\srv-fichiers\direction`.
2. Onglet **Ciblage commun** > **Ciblage...**.
3. `Nouvel élément` > `Groupe de sécurité`.
4. Groupe : `GRP-Direction` (groupe AD dédié).
5. Vérifiez la phrase récapitulative : *"Le groupe de sécurité est GRP-Direction"*.
6. OK > OK.

```powershell title="Audit-DriveMapsWithoutILT.ps1"
# Audit all GPO Drive Map items and flag those without ILT targeting
# Requires: RSAT GroupPolicy module, access to SYSVOL

#Requires -Modules GroupPolicy

$domain   = (Get-ADDomain).DNSRoot
$allGPOs  = Get-GPO -All -Domain $domain

$results = foreach ($gpo in $allGPOs) {
    # Get the GPO report as XML to inspect Preferences items
    try {
        [xml]$report = Get-GPOReport -Guid $gpo.Id -ReportType Xml
    } catch {
        continue
    }

    # Look for Drive Maps items in User Preferences
    $driveMaps = $report.SelectNodes(
        "//q1:DriveMapSettings",
        (@{ q1 = "http://www.microsoft.com/GroupPolicy/Settings/DriveMaps" })
    )

    foreach ($dm in $driveMaps) {
        $hasFilters = $dm.SelectNodes(".//q1:Filters", @{ q1 = "http://www.microsoft.com/GroupPolicy/Settings/DriveMaps" }).Count

        if ($hasFilters -eq 0) {
            [PSCustomObject]@{
                GPOName      = $gpo.DisplayName
                DriveLetter  = $dm.Properties.letter
                UNCPath      = $dm.Properties.path
                Action       = $dm.Properties.action
                HasILT       = $false
                Warning      = "NO ILT TARGETING — applies to ALL users in scope"
            }
        }
    }
}

if ($results) {
    Write-Warning "Found $($results.Count) Drive Map item(s) without ILT targeting:"
    $results | Format-Table -AutoSize
} else {
    Write-Host "All Drive Map items have ILT targeting configured." -ForegroundColor Green
}
```

```title="Résultat attendu — audit avec problèmes détectés"
WARNING: Found 2 Drive Map item(s) without ILT targeting:

GPOName                DriveLetter  UNCPath                   Action  HasILT  Warning
-------                -----------  -------                   ------  ------  -------
GPO-Lecteurs-Direction Z            \\srv-fichiers\direction  Update  False   NO ILT TARGETING — applies to ALL users in scope
GPO-Lecteurs-RH        Y            \\srv-fichiers\rh         Create  False   NO ILT TARGETING — applies to ALL users in scope
```

### :material-checklist: Liste de contrôle anti-escalade DSI

Avant de lier une GPO contenant des Drive Maps ou des GPP Printers à une OU :

- [ ] Chaque item GPP dispose d'un ciblage ILT avec au minimum un filtre de groupe de sécurité
- [ ] Le groupe de sécurité utilisé contient uniquement les utilisateurs légitimes du partage
- [ ] La GPO est liée à l'OU la plus restrictive possible (pas à la racine du domaine sauf exception justifiée)
- [ ] Le filtrage de sécurité de la GPO a été contrôlé (remplacer `Utilisateurs authentifiés` par un groupe machine si GPO machine)
- [ ] Un test sur un utilisateur pilote a été réalisé avant déploiement général
- [ ] Le script d'audit `Audit-DriveMapsWithoutILT.ps1` a été exécuté et ne retourne aucun avertissement

!!! danger "Production"
    La lettre `Z:` est traditionnellement choisie pour les lecteurs réseau de direction ou de partage commun. C'est aussi la lettre que les utilisateurs remarquent en premier dans l'explorateur. Si `Z:` apparaît sur un poste qui ne devrait pas l'avoir, la probabilité que l'utilisateur l'explore est élevée. Réservez les lettres `Y:`, `Z:` aux partages vraiment sensibles et doublez le ciblage ILT avec un filtrage de sécurité GPO sur le groupe concerné.

!!! quote "En résumé"
    - Un Drive Map sans ILT = données sensibles potentiellement visibles par tous les utilisateurs du scope de la GPO.
    - Tout item GPP sur un partage non universel doit avoir un ciblage ILT de groupe de sécurité.
    - Le script d'audit ci-dessus détecte les Drive Maps sans ILT — intégrez-le dans vos revues de configuration GPO.
    - Doublez le ciblage ILT avec un filtrage de sécurité GPO pour les partages à accès restreint.

---

## Vérification globale post-déploiement

### :material-magnify: Commandes de vérification terrain

Après le déploiement d'une GPO contenant des imprimantes ou des lecteurs, ces commandes permettent de valider rapidement l'état sur une machine cible.

```powershell title="GPP-PostDeploy-Verification.ps1"
# Post-deployment verification script for GPP Printers and Drive Maps
# Run from an elevated session on the target workstation or via remote PS

param(
    [string]$ComputerName = $env:COMPUTERNAME
)

Write-Host "=== GPP Deployment Verification on $ComputerName ===" -ForegroundColor Cyan

# 1. Force GP refresh
Write-Host "`n[1] Forcing Group Policy refresh..." -ForegroundColor Yellow
Invoke-Command -ComputerName $ComputerName -ScriptBlock { gpupdate /force /wait:0 | Out-Null }

# 2. Check applied GPOs
Write-Host "`n[2] Applied GPOs (Computer):" -ForegroundColor Yellow
Invoke-Command -ComputerName $ComputerName -ScriptBlock {
    Get-GPResultantSetOfPolicy -ReportType Html -Path "$env:TEMP\rsop.html" | Out-Null
    gpresult /r /scope:computer 2>&1 | Select-String "Applied Group Policy" -A 10
}

# 3. List mapped drives
Write-Host "`n[3] Mapped network drives:" -ForegroundColor Yellow
Invoke-Command -ComputerName $ComputerName -ScriptBlock {
    Get-PSDrive -PSProvider FileSystem |
        Where-Object { $_.Root -like '\\*' } |
        Select-Object Name, Root, Description |
        Format-Table -AutoSize
}

# 4. List installed printers
Write-Host "`n[4] Installed printers:" -ForegroundColor Yellow
Invoke-Command -ComputerName $ComputerName -ScriptBlock {
    Get-Printer | Select-Object Name, DriverName, PortName, Default |
        Format-Table -AutoSize
}

# 5. Check GPP event log for errors
Write-Host "`n[5] GPP errors in event log (last 20 events):" -ForegroundColor Yellow
Invoke-Command -ComputerName $ComputerName -ScriptBlock {
    Get-WinEvent -LogName "Microsoft-Windows-GroupPolicy/Operational" -MaxEvents 50 -ErrorAction SilentlyContinue |
        Where-Object { $_.LevelDisplayName -in @("Error", "Warning") } |
        Select-Object TimeCreated, LevelDisplayName, Message |
        Select-Object -First 20 |
        Format-List
}

Write-Host "`n=== Verification complete ===" -ForegroundColor Cyan
```

### :material-bug: Journaux GPP dans l'Observateur d'événements

Les erreurs GPP (Drive Maps, Printers, Shares) sont consignées dans :

```
Journaux des applications et des services
  > Microsoft
    > Windows
      > GroupPolicy
        > Operational
```

Les ID d'événements les plus utiles pour le diagnostic :

| ID événement | Niveau | Signification |
|-------------|--------|--------------|
| `4016` | Information | Démarrage du traitement d'une extension GPP |
| `4017` | Information | Fin du traitement d'une extension GPP — succès |
| `7016` | Erreur | Échec de traitement d'une extension GPP — contient le message d'erreur détaillé |
| `5016` | Avertissement | Traitement GPP terminé avec des avertissements |

Quand un Drive Map ou une imprimante ne se déploie pas, commencez toujours par filtrer sur les événements `7016` dans ce journal — le message d'erreur contient le nom de l'item GPP et le code d'erreur Windows exact.


!!! quote "En résumé"
    - Les erreurs GPP (Drive Maps, Printers, Shares) sont consignées dans.
    - Les ID d'événements les plus utiles pour le diagnostic.
    - Validez toujours le résultat sur un poste ou un utilisateur réellement dans le périmètre avant d’élargir.
    - Conservez les commandes et résultats de contrôle comme preuve de conformité post-déploiement.
---

## Références croisées

- [Bible GPO — Chapitre 11 : Préférences GPP](../bible-gpo/11-preferences-gpp.md) — Mécanique complète des Préférences GPO (actions CRUD, ordre d'application, comportement Remove)
- [Bible GPO — Chapitre 12 : ILT Ciblage](../bible-gpo/12-ilt.md) — Tous les types de conditions ILT et leur combinaison avec les opérateurs booléens

!!! quote "En résumé"
    - À relire : Bible GPO — Chapitre 11 : Préférences GPP.
    - À relire : Bible GPO — Chapitre 12 : ILT Ciblage.
    - Ces renvois prolongent le chapitre avec des mécanismes complémentaires ou des cas d’usage voisins.
    - Gardez ces chapitres sous la main pour le diagnostic ou la conception d’une GPO liée à ce thème.
