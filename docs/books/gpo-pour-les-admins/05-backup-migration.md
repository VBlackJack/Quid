---
description: "Sauvegarder, restaurer et migrer des GPO entre domaines : procedures de production, script automatise avec retention, table de migration inter-domaine et checklist de validation."
tags:
  - gpo
  - administration
  - sauvegarde
  - restauration
  - migration
---

# Sauvegarde, restauration et migration des GPO

!!! abstract "Ce que vous allez pouvoir faire"
    - Identifier exactement ce que `Backup-GPO` sauvegarde — et ce qu'il ne sauvegarde **pas** — pour ne pas etre surpris le jour J
    - Deployer un script de sauvegarde automatisee hebdomadaire avec retention configurable, pret pour le Task Scheduler
    - Restaurer une GPO en production en suivant une procedure etape par etape qui minimise l'impact metier
    - Migrer des GPO entre domaines (test vers production, fusion de societes) avec une table de migration pour remapper les chemins et les SID
    - Valider une restauration avec une checklist carre avant de fermer le ticket d'incident

!!! tip "Si vous ne retenez qu'une chose"
    `Backup-GPO` sauvegarde les parametres mais **PAS les liens OU**. Apres une restauration, il faut recréer les liens manuellement. Testez toujours une restauration en OU pilote avant de toucher a la production.

---

## Ce que Backup-GPO sauvegarde (et ne sauvegarde pas)

### :material-check-circle-outline: Ce qui est sauvegarde

`Backup-GPO` capture tout ce qui constitue la **definition** d'une GPO.

Les **paramètres configurés** sont integralement inclus : `registry.pol` (extensions registre), `GptTmpl.inf` (parametres de securite), les XML des Preferences, les scripts, etc.

Les **metadonnees** sont aussi sauvegardees : nom de la GPO, description, GUID d'origine, commentaires, et date de creation.

Les **permissions de la GPO** sont incluses — c'est-a-dire le filtrage de securite (les ACL sur l'objet GPO) et la delegation. Les groupes sont stockes par SID.

La **reference** aux filtres WMI associes est egalement presente. Attention : c'est le nom du filtre, pas son contenu.

### :material-close-circle-outline: Ce qui n'est PAS sauvegarde

Les **liens OU** ne sont pas dans la GPO elle-meme. Une GPO restauree n'est liee a aucune OU. Vous devrez recreer tous les liens manuellement.

Le **contenu des filtres WMI** n'est pas sauvegarde. Seule la reference par nom est conservee. Si le filtre WMI a ete supprime, il faut le recreer separement.

Les **membres des groupes AD** utilises dans le filtrage de securite ne sont evidemment pas dans la GPO — ils vivent dans Active Directory.

### Tableau recapitulatif

| Element | Sauvegarde | Notes |
|---------|:----------:|-------|
| Parametres (registry.pol, GptTmpl.inf, Preferences XML) | :white_check_mark: | Complet |
| Scripts (logon, logoff, startup, shutdown) | :white_check_mark: | Fichiers inclus |
| Nom et description | :white_check_mark: | |
| GUID de la GPO | :white_check_mark: | Utilise pour retrouver la GPO |
| Filtrage de securite (ACL) | :white_check_mark: | Groupes par SID |
| Delegation | :white_check_mark: | |
| Commentaires | :white_check_mark: | |
| Liens OU | :x: | A recreer manuellement |
| Filtre WMI (contenu de la requete) | :x: | Reference par nom seulement |
| Membres des groupes AD | :x: | Dans AD, pas dans la GPO |

!!! quote "En resume"
    `Backup-GPO` sauvegarde tout ce qui est **a l'interieur** de la GPO. Ce qui est **exterieur** — les liens, les membres de groupes, le contenu des filtres WMI — n'est pas touche. Gardez cette distinction en tete pour chaque operation de restauration.

---

## Sauvegarder une GPO

### :material-content-save: Sauvegarde d'une GPO unique

Le cas le plus courant : vous allez modifier une GPO sensible et vous voulez un point de retour propre.

```powershell
# Backup a single GPO before making changes
$backupPath = "\\serveur\GPO-Backups\$(Get-Date -Format 'yyyy-MM-dd')"
New-Item -ItemType Directory -Path $backupPath -Force | Out-Null

Backup-GPO -Name "SEC-Postes-Baseline" `
           -Path $backupPath `
           -Comment "Avant modification securite endpoint — incident INC-2026-042"
```

Le parametre `-Comment` est crucial. Il apparait dans `Backup.xml` et vous permettra de retrouver rapidement le bon backup des semaines plus tard.

### :material-content-save-all: Sauvegarde de toutes les GPO avec rapport

Pour une sauvegarde complete du domaine — avant une migration, un audit, ou simplement en routine.

```powershell
# Backup ALL GPOs and export a manifest CSV
$backupPath = "\\serveur\GPO-Backups\$(Get-Date -Format 'yyyy-MM-dd')"
New-Item -ItemType Directory -Path $backupPath -Force | Out-Null

$results = Backup-GPO -All -Path $backupPath
$results | Select-Object DisplayName, Id, BackupDirectory |
    Export-Csv "$backupPath\backup-manifest.csv" -NoTypeInformation

Write-Host "Backed up $($results.Count) GPOs to $backupPath"
```

```title="Resultat attendu"
Backed up 47 GPOs to \\serveur\GPO-Backups\2026-04-05
```

Le fichier `backup-manifest.csv` est votre index de navigation. Il contient le nom lisible de chaque GPO et le GUID du dossier de backup correspondant.

### :material-folder-open: Structure des dossiers de sauvegarde

Chaque GPO sauvegardee obtient son propre sous-dossier nomme avec un GUID de backup (different du GUID de la GPO).

```
\\serveur\GPO-Backups\2026-04-05\
├── {BACKUP-GUID-1}\
│   ├── Backup.xml          ← metadata (GPO name, GUID, date, comment)
│   ├── bkupInfo.xml        ← backup information
│   └── gpreport\
│       └── gpreport.xml    ← full GPO settings report
├── {BACKUP-GUID-2}\
│   └── ...
└── backup-manifest.csv     ← custom index (our addition)
```

`Backup.xml` contient le nom de la GPO, son GUID original, la date et le commentaire. C'est le premier fichier a ouvrir pour identifier un backup.

`gpreport.xml` est un rapport complet de tous les parametres — lisible dans un navigateur si vous le renommez en `.html` (ou presque).

!!! quote "En resume"
    Une sauvegarde unitaire avant modification + un commentaire `Backup.xml` lisible = retrouver le bon point de retour en moins de 30 secondes. Ne sautez jamais le commentaire.

---

## Script de sauvegarde automatisee

### :material-calendar-clock: Objectif du script

Ce script est prevu pour tourner chaque semaine via le Task Scheduler, avec un compte de service Domain Admin. Il sauvegarde toutes les GPO, ecrit un log horodate, et supprime les sauvegardes plus anciennes que la retention configuree.

```powershell
# Weekly GPO Backup Script with log and retention cleanup
# Schedule: Task Scheduler, every Sunday 02:00
# Run as: Domain Admin service account (read access to all GPOs + write to backup share)

param(
    [string]$BackupRoot = "\\serveur\GPO-Backups",
    [int]$RetentionDays = 30
)

$date = Get-Date -Format "yyyy-MM-dd"
$backupPath = Join-Path $BackupRoot $date
$logPath    = Join-Path $BackupRoot "backup.log"

function Write-Log {
    param(
        [string]$Message,
        [string]$Level = "INFO"
    )
    $entry = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') [$Level] $Message"
    Add-Content -Path $logPath -Value $entry
    Write-Host $entry
}

# --- Create backup directory ---
New-Item -ItemType Directory -Path $backupPath -Force | Out-Null
Write-Log "Starting GPO backup to $backupPath"

# --- Backup all GPOs ---
try {
    $results = Backup-GPO -All -Path $backupPath -ErrorAction Stop
    Write-Log "Successfully backed up $($results.Count) GPOs"

    # Save manifest for easy navigation
    $results | Select-Object DisplayName, Id, BackupDirectory |
        Export-Csv "$backupPath\backup-manifest.csv" -NoTypeInformation

    Write-Log "Manifest saved: $backupPath\backup-manifest.csv"
} catch {
    Write-Log "BACKUP FAILED: $_" -Level "ERROR"
    exit 1
}

# --- Cleanup old backups beyond retention window ---
$cutoff = (Get-Date).AddDays(-$RetentionDays)

Get-ChildItem $BackupRoot -Directory |
    Where-Object {
        $_.Name -match '^\d{4}-\d{2}-\d{2}$' -and
        $_.LastWriteTime -lt $cutoff
    } |
    ForEach-Object {
        Remove-Item $_.FullName -Recurse -Force
        Write-Log "Deleted old backup folder: $($_.Name)"
    }

Write-Log "Backup complete. Retention policy: last $RetentionDays days."
```

```title="Resultat attendu (log backup.log)"
2026-04-06 02:00:01 [INFO] Starting GPO backup to \\serveur\GPO-Backups\2026-04-06
2026-04-06 02:00:14 [INFO] Successfully backed up 47 GPOs
2026-04-06 02:00:14 [INFO] Manifest saved: \\serveur\GPO-Backups\2026-04-06\backup-manifest.csv
2026-04-06 02:00:14 [INFO] Deleted old backup folder: 2026-03-06
2026-04-06 02:00:14 [INFO] Backup complete. Retention policy: last 30 days.
```

### :material-cog: Configuration du Task Scheduler

Trois points importants pour la tache planifiee.

Le compte d'execution doit etre un **compte de service dedié** avec les droits Domain Admin (ou au minimum les droits de lecture sur toutes les GPO + ecriture sur le partage de backup). N'utilisez pas votre compte personnel.

Cochez **"Run whether user is logged on or not"** et **"Run with highest privileges"**. La tache ne doit jamais rater parce que la session est fermee.

Ajoutez une **alerte par e-mail** ou une supervision sur le fichier `backup.log` pour detecter les entrees `[ERROR]`. Une sauvegarde silencieusement echouee est pire qu'une sauvegarde absente.

!!! warning "Tester la retention avant la mise en production"
    Validez la logique de retention sur un repertoire de test avant de l'activer sur les vraies sauvegardes. Le filtre `^\d{4}-\d{2}-\d{2}$` ne supprime que les dossiers au format date — les dossiers avec d'autres noms sont proteges.

!!! quote "En resume"
    Ce script est le minimum viable pour une organisation de taille moyenne. La retention a 30 jours couvre les incidents decouverts tardivement. Adaptez `$RetentionDays` selon votre politique de conservation interne.

---

## Restaurer une GPO

### :material-history: Deux scenarios de restauration

**Scenario 1 — La GPO existe encore mais ses parametres sont incorrects.**
C'est le cas apres une mauvaise modification ou une erreur de configuration. `Restore-GPO` ecrase les parametres actuels avec ceux du backup. La GPO conserve son GUID et ses liens OU existants.

**Scenario 2 — La GPO a ete supprimee.**
La restoration recreé la GPO avec ses parametres, mais elle obtient un **nouveau GUID**. Les liens OU doivent etre recreés manuellement.

### :material-restore: Restauration — Scenario 1 (GPO existante)

```powershell
# Restore a GPO that still exists but has wrong settings
# Point to the backup folder for the date you want to roll back to
$backupPath = "\\serveur\GPO-Backups\2026-04-01"

Restore-GPO -Name "SEC-Postes-Baseline" -Path $backupPath
```

```title="Resultat attendu"
DisplayName      : SEC-Postes-Baseline
DomainName       : contoso.local
Owner            : CONTOSO\Domain Admins
Id               : {xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}
GpoStatus        : AllSettingsEnabled
```

La commande ne demande pas de confirmation. L'ecrasement est immediat.

!!! danger "Restore-GPO ecrase sans avertissement"
    `Restore-GPO` sur une GPO existante ecrase **tous** les parametres immediatement, sans prompt de confirmation. Il n'y a pas de `--WhatIf` utile ici. Restaurez toujours dans une **OU pilote** en premier. Verifiez. Puis restaurez en production.

### :material-plus-circle: Restauration — Scenario 2 (GPO supprimee)

```powershell
# The original GPO no longer exists — restore by BackupId (found in backup-manifest.csv or Backup.xml)
$backupPath = "\\serveur\GPO-Backups\2026-04-01"

Restore-GPO -BackupId "{BACKUP-GUID}" `
            -Path $backupPath `
            -Name "SEC-Postes-Baseline"

# The restored GPO has a NEW GUID — recreate the OU links manually
New-GPLink -Name "SEC-Postes-Baseline" `
           -Target "OU=Postes-de-travail,DC=contoso,DC=local"
```

```title="Resultat attendu"
DisplayName : SEC-Postes-Baseline
Id          : {yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy}   ← nouveau GUID
GpoStatus   : AllSettingsEnabled
```

!!! warning "Le GUID change apres restauration d'une GPO supprimee"
    Le nouveau GUID differe de l'original. Tout ce qui reference l'ancien GUID doit etre mis a jour : scripts de monitoring, alertes SIEM, documentation interne, eventuellement des GPO qui referençaient cette GPO via WMI. Notez l'ancien et le nouveau GUID dans le ticket d'incident.

### :material-list-status: Procedure etape par etape

Suivez ces etapes dans l'ordre. Ne brulez pas les etapes meme sous pression.

**Etape 1 — Identifier le bon backup.**
Ouvrez `backup-manifest.csv` du dossier de backup cible. Retrouvez la GPO par son nom et notez le `BackupDirectory` (GUID du dossier). Ou ouvrez `Backup.xml` directement et lisez `<GPODisplayName>` et `<BackupDate>`.

**Etape 2 — Restaurer dans une OU pilote.**
Avant de toucher la production, restaurez dans une OU de test separee et verifiez visuellement les parametres dans GPMC. Si l'OU pilote n'existe pas, creez-en une temporairement.

**Etape 3 — Valider les parametres.**
Ouvrez la GPO dans GPMC et parcourez les sections modifiees. Comparez avec le rapport d'incident ou la documentation attendue.

**Etape 4 — Restaurer en production.**
Si la validation est concluante, lancez la restauration sur la GPO de production.

**Etape 5 — Recreer les liens OU si necessaire.**
Uniquement si la GPO avait ete supprimee (Scenario 2). Utilisez `New-GPLink` ou la GPMC.

**Etape 6 — Forcer la mise a jour et verifier.**
Lancez `gpupdate /force` sur une machine pilote et verifiez avec `gpresult /r` que la GPO s'applique. Verifiez egalement les effets concrets (profil de pare-feu, cles de registre, etc.).

**Etape 7 — Documenter.**
Notez dans le ticket : qui a restaure, quand, depuis quel backup, pour quelle raison, et quel GUID a ete genere si la GPO etait supprimee.

!!! quote "En resume"
    La restauration en deux temps (pilote puis production) prend 10 minutes de plus. Elle vous epargne une deuxieme restauration en urgence si le backup n'etait pas le bon. Pilote toujours.

---

## Migration inter-domaine

### :material-swap-horizontal: Quand migrer des GPO ?

Les cas les plus frequents en entreprise : deploiement de GPO depuis un domaine de test vers la production, ou consolidation de domaines lors d'une fusion d'entreprise.

La difficulte principale : les GPO peuvent contenir des references specifiques au domaine source — chemins UNC (`\\test.contoso.local\partage\`), SID de groupes de securite, noms d'utilisateurs. Ces references doivent etre remappees vers le domaine cible.

### :material-domain: Copy-GPO (meme foret Active Directory)

Si les deux domaines appartiennent a la meme foret, `Copy-GPO` est la commande la plus simple.

```powershell
# Copy a GPO from a test domain to the production domain (same forest)
Copy-GPO -SourceGpoName "SEC-Postes-Baseline" `
         -SourceDomain "test.contoso.local" `
         -TargetName "SEC-Postes-Baseline" `
         -TargetDomain "contoso.local" `
         -CopyAcls:$false
```

Le parametre `-CopyAcls:$false` est important. Les SID des groupes different entre domaines. Copier les ACL telles quelles produirait un filtrage de securite reference des SID inconnus dans le domaine cible — la GPO s'appliquerait a personne ou a tout le monde.

Apres la copie, reconfigurez manuellement le filtrage de securite dans GPMC avec les groupes du domaine cible.

### :material-table-edit: Import-GPO avec table de migration

Pour une migration entre forets distinctes, ou quand la GPO contient des chemins UNC ou des references a des groupes specifiques au domaine source, utilisez `Import-GPO` avec une table de migration.

```powershell
# Import GPO settings from a backup into an existing target GPO
Import-GPO -BackupGpoName "SEC-Postes-Baseline" `
           -Path "\\serveur\GPO-Backups\2026-04-01" `
           -TargetName "SEC-Postes-Baseline" `
           -MigrationTable "C:\Temp\contoso-migration.migtable" `
           -CreateIfNeeded
```

```title="Resultat attendu"
DisplayName : SEC-Postes-Baseline
DomainName  : contoso.local
Id          : {zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz}
GpoStatus   : AllSettingsEnabled
```

Le parametre `-CreateIfNeeded` cree la GPO dans le domaine cible si elle n'existe pas encore. Sans ce flag, la commande echoue si la GPO n'existe pas.

### :material-file-table: Contenu d'une table de migration

La table `.migtable` mappe chaque reference du domaine source vers son equivalent dans le domaine cible. Elle couvre trois types de references.

**Chemins UNC** — par exemple, les scripts ou les dossiers de redirection.

| Source | Destination |
|--------|-------------|
| `\\test.contoso.local\netlogon\` | `\\contoso.local\netlogon\` |
| `\\fileserver-test\partage\` | `\\fileserver-prod\partage\` |

**Groupes de securite** — reference par nom ou SID.

| Source | Destination |
|--------|-------------|
| `TEST\GRP-Postes-Travail` | `CONTOSO\GRP-Postes-Travail` |
| `TEST\Domain Admins` | `CONTOSO\Domain Admins` |

**Utilisateurs individuels** (rare, mais possible dans les GPO de preferences).

| Source | Destination |
|--------|-------------|
| `test\jdupont` | `contoso\jdupont` |

### :material-tools: Creer la table de migration

!!! warning "Outil graphique uniquement"
    La table de migration se cree uniquement via **GPMC** (interface graphique). Il n'existe pas de cmdlet PowerShell pour la creer ou la modifier. Clic droit sur une GPO → **Migrate** → **Migration Table Editor**.

    L'outil peut scanner un backup de GPO existant et prelever automatiquement toutes les references detactables. C'est le point de depart recommande — vous n'avez plus qu'a renseigner les colonnes de destination.

Une fois la table creee et sauvegardee (`.migtable`), elle est reutilisable pour toutes les GPO du meme flux de migration. Versionnez-la dans votre depot Git de configuration.

!!! quote "En resume"
    `Copy-GPO` pour la meme foret sans references de domaine. `Import-GPO` + table de migration pour tout le reste. La table de migration est l'etape que les admins oublient le plus souvent — et la source numero un d'echecs silencieux apres migration.

---

## Checklist de validation post-restauration

### :material-clipboard-check: A executer avant de fermer le ticket

Utilisez cette liste apres chaque restauration, sans exception. Copiez-la dans votre ticket d'incident.

```
Validation post-restauration — SEC-Postes-Baseline — 2026-04-05

[ ] GPO visible dans GPMC avec le bon nom
[ ] Parametres verifies visuellement dans GPMC (sections concernees)
[ ] Liens OU recrees si GPO etait supprimee
[ ] Filtrage de securite verifie (groupes corrects dans le bon domaine)
[ ] Filtre WMI verifie (nom correct, contenu du filtre intact)
[ ] gpresult /r sur machine pilote confirme l'application de la GPO
[ ] Parametres effectifs verifies (ex: Get-NetFirewallProfile, cles registre)
[ ] Ancien GUID et nouveau GUID notes dans le ticket (si GPO supprimee)
[ ] Monitoring/SIEM mis a jour si reference a l'ancien GUID
[ ] Incident documente : qui, quand, depuis quel backup, pourquoi
```

### :material-console: Commandes de verification rapide

Verifier que la GPO s'applique bien a une machine pilote.

```powershell
# Force a policy refresh and display applied GPOs on a remote machine
Invoke-Command -ComputerName "PC-PILOTE-01" -ScriptBlock {
    gpupdate /force | Out-Null
    gpresult /r
}
```

```title="Resultat attendu (extrait)"
Applied Group Policy Objects
-----------------------------
    SEC-Postes-Baseline
    Default Domain Policy
```

Verifier un parametre specifique apres restauration d'une GPO de securite firewall.

```powershell
# Verify firewall profile settings as configured by the restored GPO
Invoke-Command -ComputerName "PC-PILOTE-01" -ScriptBlock {
    Get-NetFirewallProfile | Select-Object Name, Enabled, DefaultInboundAction
}
```

```title="Resultat attendu"
Name    Enabled DefaultInboundAction
----    ------- --------------------
Domain  True    Block
Private True    Block
Public  True    Block
```

Verifier une cle de registre deployee par une GPO Preferences.

```powershell
# Verify a registry value deployed via GPO Preferences
Invoke-Command -ComputerName "PC-PILOTE-01" -ScriptBlock {
    Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" `
                     -Name "SearchList" -ErrorAction SilentlyContinue
}
```

!!! quote "En resume"
    Une checklist a la con qu'on cochait a la va-vite, c'est 30 minutes de moins par incident. Une checklist ignoree, c'est potentiellement un deuxieme incident deux jours plus tard parce que le filtre WMI n'avait pas ete reverifie.

---

## Pieges en production

### :material-bomb: Pieges courants et comment les eviter

**Restaurer directement en production.**
Le reflexe "je suis certain du backup" est dangereux. `Restore-GPO` n'a pas de mode dry-run. Toujours passer par une OU de test. Toujours.

**Oublier de recreer les liens apres restauration d'une GPO supprimee.**
La GPO est restauree, les parametres sont la, mais rien ne s'applique parce que la GPO n'est liee a aucune OU. Ce bug est particulierement sournois car `gpresult /r` ne montrera tout simplement pas la GPO.

**Ignorer le changement de GUID.**
Apres restauration d'une GPO supprimee, l'ancien GUID ne pointe plus sur rien. Si des alertes de monitoring, des rapports SIEM ou des scripts referençaient ce GUID, ils cesseront de fonctionner silencieusement.

**Restaurer le mauvais backup.**
Plusieurs backups de la meme GPO pour des dates differentes. Sans commentaire dans `Backup-GPO`, impossible de savoir lequel choisir. D'ou l'importance du parametre `-Comment` systematique.

**Ne pas verifier les filtres WMI.**
Le backup conserve seulement le nom du filtre WMI, pas son contenu. Si le filtre a ete supprime ou renomme entre-temps, la GPO restauree pointe sur un filtre inexistant — aucune alerte dans GPMC, mais la GPO s'appliquera a des machines inattendues (ou a aucune).

!!! danger "Absence de prompt de confirmation"
    `Restore-GPO`, `Import-GPO` et `Copy-GPO` n'ont pas de prompt de confirmation et n'implementent pas correctement `-WhatIf`. Une mauvaise cible dans le script et vous ecrasez la mauvaise GPO sans aucun avertissement. Relisez vos commandes avant d'appuyer sur Entree.

!!! warning "Filtres WMI non sauvegardes"
    Si un filtre WMI associe a la GPO a ete supprime, la restauration de la GPO ne le restaure pas. Vous devez recreer le filtre WMI manuellement dans GPMC (onglet **WMI Filters**) puis le reassocier a la GPO restauree. Verifiez systematiquement l'etat des filtres WMI dans votre checklist post-restauration.

!!! quote "En resume"
    La majorite des incidents post-restauration provient de trois causes : liens OU non recrees, mauvais GUID reference dans le monitoring, filtre WMI manquant. Votre checklist couvre les trois. Utilisez-la.

---

## Cross-references

| Sujet | Reference |
|-------|-----------|
| Cmdlets PowerShell GroupPolicy (Backup-GPO, Restore-GPO, Import-GPO) | Ch. 03 — PowerShell GroupPolicy module |
| Sauvegarde GPO — concepts debutant | Les GPO pour les Nuls — Ch. 11 |
| Automatisation sauvegarde en pipeline CI/CD | Ch. 23 — Automatisation CI/CD |
| Structure interne des backups GPO et SYSVOL | La Bible GPO — Ch. 04 — SYSVOL |
| Audit et conformite des GPO | Ch. 06 — Audit et conformite |
