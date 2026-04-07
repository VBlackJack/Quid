---
description: GPO multi-sites et multi-forêts — sites AD, DFS-R, trusts inter-forêts, architecture cross-domain
tags: [gpo-admins, multi-sites, multi-forêts, dfs-r, sites-ad, trusts]
---
# GPO multi-sites et multi-forêts

!!! abstract "Ce que vous allez apprendre"
    - Lier une GPO à un Site AD dans GPMC et comprendre les implications sur la réplication WAN
    - Appliquer des GPO dans plusieurs domaines d'une même forêt avec `Copy-GPO` et une table de migration
    - Comprendre pourquoi les GPO ne peuvent **pas** franchir une frontière de forêt, même avec un trust complet
    - Diagnostiquer les problèmes de convergence SYSVOL DFS-R sur un lien WAN avec `dfsrdiag` et les Event IDs clés
    - Anticiper et éviter le piège de la création de GPO sur le mauvais DC en environnement multi-sites

!!! tip "Si vous ne retenez qu'une chose"
    Une GPO s'applique **uniquement dans la forêt où elle a été créée**. Les trusts inter-forêts permettent l'authentification et l'accès aux ressources — ils ne permettent pas à une GPO de traverser la frontière de forêt. C'est une limitation architecturale du service GP Client, pas une configuration manquante.

---

## Contexte de production

Les environnements multi-sites sont la norme dans les organisations de taille moyenne et grande : un siège social, plusieurs agences régionales, parfois des pays distincts. Chaque site a des contraintes réseau, des DNS suffixes, des proxies, parfois des fuseaux horaires différents.

Les problèmes arrivent quand on essaie d'appliquer une configuration centralisée sans tenir compte de l'architecture AD Sites & Services. Ou, à l'inverse, quand on crée des GPO site-spécifiques sans comprendre comment le GPC et le GPT se répliquent — et ce que ça implique pour les utilisateurs qui ouvrent une session pendant une fenêtre de latence WAN.

Ce chapitre couvre les quatre situations que vous rencontrerez en production : GPO liées à un Site, GPO cross-domaine dans une même forêt, GPO cross-forêt (spoiler : ça n'existe pas), et la gestion de la convergence SYSVOL sur liens étendus.


!!! quote "En résumé"
    - Chaque site a des contraintes réseau, des DNS suffixes, des proxies, parfois des fuseaux horaires différents.
    - Le contexte de production fixe les contraintes réelles de réseau, de portée et d’exploitation qui gouvernent tout le chapitre.
    - Retenez les hypothèses opérationnelles avant de choisir un modèle de liaison ou de déploiement.
---

## GPO liées à un Site AD

### :material-map-marker-radius: Pourquoi lier une GPO à un Site

Les GPO sont le plus souvent liées à des OU — c'est le cas standard. Mais Active Directory permet aussi de lier une GPO à un **Site AD**, c'est-à-dire à un objet de la partition `CN=Sites,CN=Configuration`.

L'intérêt est de cibler des postes **par leur emplacement réseau** plutôt que par leur position dans l'arborescence OU.

Les cas d'usage courants :

| Besoin | Exemple concret |
|--------|-----------------|
| Fond d'écran avec nom du site | `Siège Paris — Zone sécurisée` |
| Proxy HTTP spécifique au site | `proxy-paris.corp.example.com:3128` |
| Suffixe DNS local | `paris.corp.example.com` |
| Serveurs d'impression locaux | Redirection vers les imprimantes du bâtiment |
| Configuration NTP spécifique | Serveur de temps local pour un site isolé |

!!! warning "À surveiller — Site AD ≠ OU géographique"
    Un objet Site AD regroupe des plages IP. Une OU géographique regroupe des objets AD. Les deux ne sont pas équivalents. Un poste dans l'OU `Paris` peut très bien se connecter depuis Lyon s'il est nomade. La GPO liée au Site s'applique selon **l'IP au moment de la connexion**, pas selon la position dans l'OU. Choisissez votre stratégie de liaison en connaissance de cause.

### :material-link: Lier une GPO à un Site dans GPMC

La liaison à un Site se fait dans GPMC via le nœud **Sites**, pas via le nœud **Domains**. Ce nœud est masqué par défaut.

**Chemin GPMC pour afficher les Sites :**

```
Group Policy Management
└── Forest: corp.example.com
    └── Sites          ← clic droit → "Show Sites..." pour l'afficher
        ├── Paris
        ├── Lyon
        └── Bordeaux
```

Étapes dans GPMC :

1. Dans GPMC, clic droit sur **Sites** → **Show Sites...**
2. Cochez le ou les sites à afficher, cliquez OK.
3. Sélectionnez le site cible (ex: `Paris`).
4. Onglet **Linked Group Policy Objects** → **Link an Existing GPO...**
5. Sélectionnez la GPO à lier.

### :material-powershell: Lier une GPO à un Site en PowerShell

Le DN d'un Site est dans la partition de Configuration, pas dans la partition du domaine. La syntaxe exacte de `-Target` est critique.

```powershell
# Link a GPO to an AD Site
# The target DN is always in CN=Sites,CN=Configuration,DC=...
New-GPLink -Name "SITE-Paris-Proxy" `
           -Target "CN=Paris,CN=Sites,CN=Configuration,DC=corp,DC=example,DC=com" `
           -LinkEnabled Yes `
           -Enforced No
```

```title="Résultat attendu"
GpoId       : {xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}
DisplayName : SITE-Paris-Proxy
Enabled     : True
Enforced    : False
Target      : CN=Paris,CN=Sites,CN=Configuration,DC=corp,DC=example,DC=com
Order       : 1
```

Pour retrouver le DN exact d'un site sans l'avoir mémorisé :

```powershell
# List all AD Sites with their distinguished names
Get-ADReplicationSite -Filter * | Select-Object Name, DistinguishedName
```

```title="Résultat attendu"
Name        DistinguishedName
----        -----------------
Paris       CN=Paris,CN=Sites,CN=Configuration,DC=corp,DC=example,DC=com
Lyon        CN=Lyon,CN=Sites,CN=Configuration,DC=corp,DC=example,DC=com
Bordeaux    CN=Bordeaux,CN=Sites,CN=Configuration,DC=corp,DC=example,DC=com
```

### :material-transit-connection-variant: Implications sur la réplication WAN

C'est le point critique que les admins sous-estiment systématiquement.

Une GPO se compose de deux parties distinctes qui ne se répliquent pas de la même façon :

| Composant | Stockage | Réplication |
|-----------|----------|-------------|
| **GPC** (Group Policy Container) | Objet AD dans la partition du domaine | Réplication AD standard (selon les liens de sites) |
| **GPT** (Group Policy Template) | Dossier dans `SYSVOL\Policies\{GUID}\` | DFS-R entre tous les DC du domaine |

La **partition de Configuration** (qui contient les objets Sites) se réplique entre **tous les DC de la forêt**, pas seulement ceux du domaine. Elle est globale.

Le **GPT dans SYSVOL** est téléchargé depuis le **DC le plus proche** au moment de l'application de la GPO. Si ce DC local n'a pas encore reçu le GPT (latence DFS-R sur lien WAN), le client obtient l'ancienne version.

!!! danger "Production — Latence asymétrique GPC/GPT"
    Vous modifiez une GPO liée au Site de Lyon depuis Paris. Le GPC est mis à jour sur le PDC Emulator (Paris) et repliqué rapidement vers Lyon. Mais le GPT dans SYSVOL peut mettre plusieurs minutes à atteindre le DC de Lyon via DFS-R sur un lien WAN chargé. Résultat : les postes de Lyon lisent un GPC qui référence la version N+1, mais téléchargent le GPT de la version N. L'application de la GPO est incohérente. Un `gpupdate /force` quelques minutes plus tard résout le problème — mais les utilisateurs connectés pendant cette fenêtre ont reçu une configuration partielle.

!!! quote "En résumé"
    - Les Sites AD permettent de cibler des postes par plage IP, indépendamment de leur OU.
    - La liaison se fait dans le nœud **Sites** de GPMC (masqué par défaut).
    - Le DN cible est toujours dans `CN=Sites,CN=Configuration,DC=...`.
    - Le GPT est téléchargé depuis le DC local — la latence DFS-R sur WAN peut créer une fenêtre d'incohérence.

---

## GPO multi-domaines dans la même forêt

### :material-domain: Comportement natif : une GPO reste dans son domaine

Une GPO est créée dans un domaine précis. Elle est stockée dans la partition du domaine (`CN=Policies,CN=System,DC=...`) et son GPT vit dans le SYSVOL de ce domaine.

**Une GPO ne s'applique qu'aux objets du domaine dans lequel elle a été créée.** C'est un comportement natif, sans exception.

Si vous avez une forêt `corp.example.com` avec deux domaines — `emea.corp.example.com` et `apac.corp.example.com` — une GPO créée dans `emea` ne peut pas s'appliquer aux postes d'`apac`, même si les deux domaines sont dans la même forêt.

### :material-copy: Stratégie : Copy-GPO entre domaines de la même forêt

La solution officielle est de dupliquer la GPO dans chaque domaine cible. `Copy-GPO` fait exactement ça pour les domaines d'une même forêt.

```powershell
# Copy a GPO from one domain to another within the same forest
Copy-GPO -SourceGpoName "SEC-Baseline-Postes" `
         -SourceDomain "emea.corp.example.com" `
         -TargetName "SEC-Baseline-Postes" `
         -TargetDomain "apac.corp.example.com" `
         -CopyAcls:$false
```

Le paramètre `-CopyAcls:$false` est obligatoire quand les domaines source et cible ont des groupes différents. Les SID ne sont pas portables entre domaines.

Après la copie, configurez manuellement le filtrage de sécurité dans GPMC pour `apac.corp.example.com` avec les groupes du domaine cible.

### :material-table-edit: Table de migration pour les références cross-domaine

Si la GPO contient des références spécifiques au domaine source — chemins UNC, noms de groupes, suffixes de domaine dans des paramètres — une table de migration est nécessaire.

La table mappe les références source vers leurs équivalents dans le domaine cible.

```powershell
# Import GPO settings into the target domain using a migration table
Import-GPO -BackupGpoName "SEC-Baseline-Postes" `
           -Path "\\serveur\GPO-Backups\2026-04-05" `
           -TargetName "SEC-Baseline-Postes" `
           -TargetDomain "apac.corp.example.com" `
           -MigrationTable "C:\Temp\emea-to-apac.migtable" `
           -CreateIfNeeded
```

```title="Résultat attendu"
DisplayName : SEC-Baseline-Postes
DomainName  : apac.corp.example.com
Id          : {yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy}
GpoStatus   : AllSettingsEnabled
```

Exemples de mappings dans la table de migration :

| Référence source (EMEA) | Référence cible (APAC) |
|-------------------------|------------------------|
| `\\emea-fs01\netlogon\` | `\\apac-fs01\netlogon\` |
| `EMEA\GRP-Postes-Travail` | `APAC\GRP-Postes-Travail` |
| `emea.corp.example.com` | `apac.corp.example.com` |

!!! warning "À surveiller — Table de migration créée uniquement dans GPMC"
    Il n'existe pas de cmdlet PowerShell pour créer ou éditer un fichier `.migtable`. La création se fait exclusivement via l'interface graphique GPMC : clic droit sur une GPO → **Migrate** → **Migration Table Editor**. L'outil peut scanner un backup existant pour pré-remplir les colonnes source automatiquement.

### :material-shield-account: Filtrage de sécurité cross-domaine

Les groupes utilisés dans le filtrage de sécurité doivent exister dans le domaine où la GPO est liée. Un groupe `EMEA\GRP-Postes-Travail` n'est pas visible dans `apac.corp.example.com` pour le filtrage.

Deux options :

1. **Groupes universels** — Un groupe universel est visible dans toute la forêt. Créez un groupe universel dans la forêt et utilisez-le dans le filtrage de sécurité des deux domaines. C'est la solution la plus propre pour un déploiement uniforme.

2. **Groupes locaux de domaine** — Créez un groupe équivalent dans chaque domaine cible et appliquez le même filtrage localement. Plus de maintenance, mais utile si les périmètres sont différents.

!!! quote "En résumé"
    - Une GPO ne traverse pas les frontières de domaine, même dans la même forêt.
    - `Copy-GPO` duplique la GPO dans le domaine cible en une commande.
    - `Import-GPO` + table de migration résout les références spécifiques au domaine source.
    - Les groupes universels simplifient le filtrage de sécurité cross-domaine.
    - Voir aussi : [Chapitre 05 — Sauvegarde et migration](05-backup-migration.md) pour le détail du workflow `Copy-GPO` / `Import-GPO`.

---

## GPO multi-forêts : la limitation fondamentale

### :material-forest: Une GPO ne peut pas traverser une forêt

C'est la limite architecturale la plus importante de ce chapitre — et la plus souvent mal comprise.

**Une GPO ne peut jamais s'appliquer à des objets d'une autre forêt.** Pas avec `Copy-GPO`, pas avec un trust, pas avec une configuration avancée. Il n'existe aucune exception à cette règle.

### :material-cog: Pourquoi — Le fonctionnement du GP Client Service

Quand un poste Windows applique des GPO, le service **GP Client Service** (`gpsvc`) suit ce processus :

1. Contacte le DC de son propre domaine pour obtenir la liste des GPO applicables.
2. Télécharge les GPT depuis le SYSVOL de son propre domaine.
3. Applique les paramètres localement.

Le service ne consulte **jamais** l'AD d'une autre forêt. Il ne connaît pas l'existence d'un trust inter-forêts au moment du traitement des GPO. Cette contrainte est dans le design du protocole, pas dans une configuration Active Directory.

### :material-alert: Le modèle Forest de ressources

Un cas qui revient régulièrement : le **Resource Forest** (forêt de ressources + forêt de comptes).

Dans ce modèle :

- La **forêt de comptes** contient les utilisateurs et groupes.
- La **forêt de ressources** contient les serveurs, applications, et postes de travail.
- Un trust inter-forêts (généralement bidirectionnel) permet aux utilisateurs de la forêt de comptes de s'authentifier sur les ressources de la forêt de ressources.

La question revient toujours : "Peut-on appliquer une GPO de la forêt de comptes aux postes de la forêt de ressources ?"

**Non.** Les postes appartiennent à la forêt de ressources. Le GP Client Service de ces postes consulte uniquement le DC de la forêt de ressources. Les GPO de la forêt de comptes sont totalement invisibles pour eux.

| Ce qu'un trust inter-forêts permet | Ce qu'un trust inter-forêts ne permet pas |
|------------------------------------|------------------------------------------|
| Authentification cross-forêt | Application de GPO cross-forêt |
| Accès aux ressources partagées | Consultation de l'AD de l'autre forêt par gpsvc |
| Utilisation de groupes universels cross-forêt | Héritage de paramètres GPO |
| Recherche de ressources dans l'autre forêt | Téléchargement du GPT depuis l'autre SYSVOL |

!!! danger "Production — L'incident classique du trust"
    Un admin configure un trust inter-forêts entre `corp-a.example.com` et `corp-b.example.com`. Il est convaincu que les postes de `corp-b` vont maintenant recevoir les GPO de `corp-a`. Il ne crée aucune GPO dans `corp-b`. Les postes de `corp-b` restent sans configuration de sécurité pendant trois semaines avant que l'incident soit détecté lors d'un audit. Le trust n'a aucun effet sur le traitement des GPO.

### :material-wrench: Alternatives opérationnelles

Puisqu'on ne peut pas croiser les forêts, les alternatives sont :

**Option 1 — Duplication manuelle avec `Copy-GPO` + table de migration.**
Exportez la GPO depuis la forêt source, importez-la dans la forêt cible avec une table de migration adaptée. Maintenez les deux GPO en parallèle. C'est le travail supplémentaire à absorber dans votre gouvernance.

**Option 2 — Console GPMC multi-forêts.**
GPMC peut se connecter à plusieurs forêts simultanément (clic droit sur **Forest** → **Add Forest**). Cela permet de gérer les GPO des deux forêts depuis une seule console — mais les GPO restent dans leur forêt d'origine.

**Option 3 — Outils tiers.**
Des outils comme AGPM (Advanced Group Policy Management) de Microsoft ou des solutions tierces permettent de gérer le cycle de vie des GPO à l'échelle multi-forêts, mais chaque GPO reste dans sa forêt. Ils automatisent la synchronisation, pas le franchissement de frontière.

!!! quote "En résumé"
    - Il n'existe aucun moyen de faire appliquer une GPO cross-forêt.
    - Le GP Client Service ne consulte que l'AD de sa propre forêt.
    - Le Resource Forest n'est pas une exception — les GPO doivent exister dans la forêt de ressources.
    - La solution opérationnelle est toujours la duplication + table de migration, gérée par votre gouvernance.

---

## Trusts inter-forêts et GPO

### :material-handshake: Ce qu'un trust fait — et ce qu'il ne fait pas

Un trust inter-forêts est un accord d'authentification entre deux forêts. Il dit : "Les utilisateurs de la forêt A peuvent prouver leur identité auprès des ressources de la forêt B."

Ce que le trust permet :

- Authentification Kerberos cross-forêt (via le SID History ou les noms UPN)
- Accès aux partages réseau, applications, services
- Utilisation de groupes universels cross-forêt dans les ACL

Ce que le trust ne fait pas — jamais, sans exception :

- Permettre à une GPO de s'appliquer dans l'autre forêt
- Exposer l'AD de la forêt source au GP Client de la forêt cible
- Rendre les SYSVOL des deux forêts accessibles l'un à l'autre pour le traitement des GPO

### :material-alert-circle: L'idée reçue et l'incident qui suit

L'idée reçue la plus répandue : "Avec un trust bidirectionnel complet entre deux forêts, les GPO peuvent circuler."

Ce raisonnement vient d'une confusion entre **authentification** et **application des stratégies**. Le trust résout l'authentification. L'application des GPO est un mécanisme distinct qui ne connaît pas le trust.

En pratique, l'incident qui suit ce raisonnement ressemble à ceci :

1. Fusion d'entreprise. Deux forêts AD coexistent.
2. Un trust bidirectionnel est mis en place pour que les utilisateurs des deux entités puissent accéder aux ressources partagées.
3. L'équipe infra décide de ne pas dupliquer les GPO de sécurité dans la forêt B "parce que le trust va les propager".
4. Six semaines plus tard, l'audit de sécurité révèle que les postes de la forêt B n'ont aucune configuration de sécurité appliquée.
5. La reconstruction prend deux jours.

!!! danger "Production — Ne jamais présumer qu'un trust propage les GPO"
    Après la mise en place d'un trust inter-forêts, validez systématiquement avec `gpresult /r` sur un poste de chaque forêt que les GPO attendues s'appliquent bien. Ne présumez pas. La commande prend 10 secondes. L'incident correctif prend deux jours.

```powershell
# Validate applied GPOs on a remote machine in the other forest
# Run this from a machine that has access cross-forest
Invoke-Command -ComputerName "PC-ForetB-01.corp-b.example.com" `
               -Credential (Get-Credential) `
               -ScriptBlock { gpresult /r }
```

```title="Résultat attendu — Si les GPO de la forêt B sont bien configurées"
Applied Group Policy Objects
-----------------------------
    SEC-Baseline-Postes-B
    Default Domain Policy

The following GPOs were not applied because they were filtered out
--------------------------------------------------------------------
    (none)
```

!!! quote "En résumé"
    - Les trusts inter-forêts ne propagent pas les GPO. Cette phrase doit être dans votre documentation de fusion.
    - Authentification et application des GPO sont deux mécanismes indépendants.
    - Après un trust, validez avec `gpresult /r` sur des postes des deux forêts.
    - Voir aussi : [Chapitre 02 — Gouvernance](02-gouvernance.md) pour la gestion multi-domaine dans le cadre d'une fusion.

---

## DFS-R multi-sites et convergence SYSVOL

### :material-sync: Le problème de la latence SYSVOL sur WAN

Quand vous modifiez une GPO, les changements sont écrits sur le **PDC Emulator** du domaine. DFS-R réplique ensuite le contenu du SYSVOL vers tous les autres DC du domaine.

Sur un réseau local, cette réplication est quasi-instantanée. Sur un lien WAN à 10 Mbps entre Paris et un site distant en province, la réplication peut prendre plusieurs minutes — voire plusieurs dizaines de minutes si le lien est saturé.

Le scénario problématique :

1. Un admin modifie une GPO urgente à Paris (PDC Emulator).
2. Il demande à un utilisateur de Lyon de faire `gpupdate /force`.
3. Le DC local de Lyon n'a pas encore reçu le GPT mis à jour.
4. L'utilisateur applique l'ancienne version de la GPO.
5. L'admin pense que la GPO est défectueuse. Elle ne l'est pas — elle n'est simplement pas encore arrivée.

### :material-eye: Event IDs DFS-R à surveiller

Ces Event IDs dans le journal **DFS Replication** (Applications and Services Logs) signalent des problèmes de réplication SYSVOL.

| Event ID | Niveau | Signification | Action |
|----------|--------|---------------|--------|
| **4602** | Information | DFS-R a terminé la synchronisation initiale du SYSVOL | Normal au démarrage d'un DC |
| **5014** | Warning | DFS-R a détecté un conflit de réplication | Identifier le fichier en conflit |
| **2213** | Error | Journal DFS-R corrompu (journal wrap) | Intervention manuelle requise |
| **4012** | Error | DFS-R a perdu le contact avec un partenaire depuis trop longtemps | Vérifier la connectivité réseau |
| **5002** | Error | Le dossier répliqué a été mis hors service par DFS-R | Intervention manuelle d'urgence |

!!! danger "Production — Event ID 2213 et 5002 : arrêt immédiat de la réplication"
    L'Event ID 2213 (journal wrap) et l'Event ID 5002 (dossier mis hors service) indiquent que DFS-R a **suspendu la réplication** pour ce partenaire. Les GPO ne se répliquent plus vers ce DC. Les utilisateurs du site local continuent d'appliquer des GPO potentiellement obsolètes, sans aucune alerte visible pour eux. Ces deux Event IDs doivent déclencher une alerte supervisée prioritaire.

### :material-console: Diagnostiquer l'état DFS-R avec dfsrdiag

`dfsrdiag` est l'outil en ligne de commande pour diagnostiquer l'état de DFS-R sur un DC.

**Vérifier l'état du partenariat DFS-R entre deux DC :**

```powershell
# Check DFS-R replication health from the local DC's perspective
dfsrdiag ReplicationState /rgname:"Domain System Volume"
```

```title="Résultat attendu — Réplication saine"
Replication group name: Domain System Volume
State               : Active
Replication members : 3
```

**Vérifier le backlog entre deux DC (nombre de fichiers en attente de réplication) :**

```powershell
# Check the replication backlog between two domain controllers
# A high backlog means SYSVOL updates are queued and not yet delivered
dfsrdiag Backlog `
    /rgname:"Domain System Volume" `
    /rfname:"SYSVOL Share" `
    /sendingmember:DC-Paris01 `
    /receivingmember:DC-Lyon01
```

```title="Résultat attendu — Backlog nul (sain)"
Backlog File Count: 0
```

```title="Résultat attendu — Backlog présent (alerte)"
Backlog File Count: 14
The following files are waiting for replication:
  registry.pol
  GPT.ini
  ...
```

Un backlog non nul signifie que des mises à jour GPO sont en attente. Ne forcez pas `gpupdate` sur les postes distants avant que le backlog soit à 0.

**Forcer une synchronisation DFS-R :**

```powershell
# Trigger a DFS-R synchronization poll on a remote DC
# Use this after confirming backlog, not as a first response
Invoke-Command -ComputerName DC-Lyon01 -ScriptBlock {
    dfsrdiag PollAD /member:DC-Lyon01
}
```

```title="Résultat attendu"
DFS Replication Service successfully polled Active Directory.
```

### :material-powershell: Surveiller l'état DFS-R de tous les DC en PowerShell

Pour une vue d'ensemble rapide avant un déploiement urgent :

```powershell
# Check DFS-R replication state on all domain controllers
# Run before forcing gpupdate on remote sites after a critical GPO change
$domainControllers = (Get-ADDomainController -Filter *).HostName

foreach ($dc in $domainControllers) {
    Write-Host "`n=== $dc ===" -ForegroundColor Cyan
    try {
        Invoke-Command -ComputerName $dc -ScriptBlock {
            dfsrdiag ReplicationState /rgname:"Domain System Volume"
        } -ErrorAction Stop
    } catch {
        Write-Warning "Cannot reach $dc : $_"
    }
}
```

```title="Résultat attendu"
=== DC-Paris01.corp.example.com ===
State : Active
...
=== DC-Lyon01.corp.example.com ===
State : Active
...
=== DC-Bordeaux01.corp.example.com ===
State : Active
...
```

### :material-alert: Impact sur l'application des GPO

Le scénario le plus dangereux en multi-sites :

1. Un utilisateur ouvre sa session sur un poste du site de Bordeaux.
2. Le DC local de Bordeaux n'a pas encore reçu la dernière version du GPT (DFS-R en cours).
3. Le poste télécharge le GPT depuis le DC de Bordeaux — version N-1.
4. La GPO s'applique, mais avec les anciens paramètres.
5. L'utilisateur se retrouve avec une configuration incorrecte (proxy manquant, restriction logicielle absente, etc.).

Ce problème est **transitoire** mais peut avoir des conséquences de sécurité si la GPO manquante est une baseline de sécurité.

!!! warning "À surveiller — gpresult ne montre pas la version du GPT appliqué"
    `gpresult /r` liste les GPO appliquées par nom — il ne montre pas le numéro de version. Un poste peut afficher "SEC-Baseline-Postes" dans ses GPO appliquées tout en ayant reçu la version N-1. Pour vérifier la version, comparez le `gPCUserExtensionNames` ou consultez `GPT.ini` dans le SYSVOL local du DC.

```powershell
# Compare GPT.ini version on two DCs to detect replication lag
$gpoGuid = "{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}"

$versionParis = Invoke-Command -ComputerName DC-Paris01 -ScriptBlock {
    Get-Content "\\DC-Paris01\SYSVOL\corp.example.com\Policies\$using:gpoGuid\GPT.ini"
}

$versionLyon = Invoke-Command -ComputerName DC-Lyon01 -ScriptBlock {
    Get-Content "\\DC-Lyon01\SYSVOL\corp.example.com\Policies\$using:gpoGuid\GPT.ini"
}

Write-Host "Paris GPT.ini: $versionParis"
Write-Host "Lyon GPT.ini:  $versionLyon"
```

```title="Résultat attendu — Versions identiques (convergence atteinte)"
Paris GPT.ini: [General]
               Version=65537
Lyon GPT.ini:  [General]
               Version=65537
```

```title="Résultat attendu — Versions différentes (réplication en cours)"
Paris GPT.ini: [General]
               Version=65538
Lyon GPT.ini:  [General]
               Version=65537
```

!!! quote "En résumé"
    - La latence DFS-R sur WAN crée une fenêtre d'incohérence entre la création de la GPO et son application effective sur les sites distants.
    - `dfsrdiag Backlog` mesure le nombre de fichiers en attente — à vérifier avant tout `gpupdate` forcé à distance.
    - Les Event IDs 2213 et 5002 signalent un arrêt de réplication : alertes prioritaires obligatoires.
    - Comparer `GPT.ini` entre DC permet de confirmer la convergence sur une GPO spécifique.
    - Voir aussi : [La Bible GPO — Chapitre 04 — SYSVOL](../bible-gpo/04-sysvol.md) pour les mécanismes internes de DFS-R.

---

## Piège de production : créer une GPO depuis le mauvais DC

### :material-bomb: Le scénario

Vous êtes en déplacement à Lyon. Vous ouvrez GPMC depuis un poste local et vous vous connectez au DC local de Lyon. Vous créez une nouvelle GPO urgente et vous la liez à une OU.

La GPO est créée — mais elle a été écrite sur le DC de Lyon, pas sur le PDC Emulator (qui est à Paris).

### :material-transit-connection: Ce qui se passe ensuite

Active Directory réplique les nouvelles GPO depuis le DC où elles ont été créées vers les autres DC, y compris le PDC Emulator. Dans un réseau local, c'est transparent. Sur un lien WAN, cela peut prendre plusieurs minutes.

Mais le vrai problème est plus subtil : GPMC, par défaut, affiche l'état des GPO tel que vu par le DC auquel il est connecté. Si ce DC est en avance sur la réplication, vous voyez une GPO que les autres DC ne connaissent pas encore.

Les postes du site de Lyon vont appliquer la GPO rapidement (leur DC local l'a). Les postes des autres sites attendent la réplication AD + DFS-R.

!!! danger "Production — Toujours créer les GPO sur le PDC Emulator"
    La bonne pratique est de **toujours pointer GPMC vers le PDC Emulator** pour créer et modifier des GPO. Le PDC Emulator est l'autorité de référence pour les GPO dans le domaine. Créer une GPO sur un autre DC est techniquement possible — mais la fenêtre de réplication est imprévisible et les symptômes sont difficiles à diagnostiquer.

    Dans GPMC : clic droit sur le domaine → **Change Domain Controller** → **PDC Emulator**.

```powershell
# Identify the PDC Emulator for the domain before creating GPOs
(Get-ADDomain).PDCEmulator
```

```title="Résultat attendu"
DC-Paris01.corp.example.com
```

```powershell
# Create a GPO and explicitly target the PDC Emulator
# This ensures immediate visibility across the domain
New-GPO -Name "SITE-Lyon-Config-Proxy" `
        -Server "DC-Paris01.corp.example.com"

New-GPLink -Name "SITE-Lyon-Config-Proxy" `
           -Target "OU=Postes-Lyon,DC=corp,DC=example,DC=com" `
           -Server "DC-Paris01.corp.example.com"
```

```title="Résultat attendu"
DisplayName      : SITE-Lyon-Config-Proxy
DomainName       : corp.example.com
Owner            : CORP\Domain Admins
Id               : {zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz}
GpoStatus        : AllSettingsEnabled
```

Le paramètre `-Server` sur `New-GPO` et `New-GPLink` force l'écriture sur un DC précis. Utilisez toujours le PDC Emulator.

### :material-clock-alert: Délai constaté en production

En environnement multi-sites réel, sur un lien WAN standard :

| Étape | Délai typique |
|-------|---------------|
| Réplication AD (GPC) entre DC du même site | < 15 secondes |
| Réplication AD (GPC) inter-sites (selon schedule de réplication) | 15 min à plusieurs heures selon configuration |
| Réplication DFS-R (GPT) sur lien WAN 10 Mbps saturé | 5 à 30 minutes |
| Réplication DFS-R (GPT) sur lien WAN 100 Mbps sain | 1 à 5 minutes |

!!! warning "À surveiller — Les schedules de réplication inter-sites AD"
    La réplication AD entre sites est contrôlée par les **Connection Objects** dans `CN=Sites,CN=Configuration`. Par défaut, la réplication inter-sites est schedulée toutes les 3 heures (avec un minimum configurable de 15 minutes). Une GPO créée sur le DC de Lyon à 14h peut n'atteindre le DC de Bordeaux qu'à 17h si aucun schedule n'a été configuré manuellement. Vérifiez vos schedules de réplication dans **Active Directory Sites and Services**.

!!! quote "En résumé"
    - Créez toujours les GPO en pointant GPMC (ou `-Server`) vers le PDC Emulator.
    - La réplication inter-sites AD suit un schedule — potentiellement plusieurs heures.
    - La réplication DFS-R du GPT dépend de la bande passante disponible sur le WAN.
    - Identifiez votre PDC Emulator avec `(Get-ADDomain).PDCEmulator` avant chaque opération urgente.

---

## Vérification post-déploiement

### :material-clipboard-check: Validation d'un déploiement multi-sites

Après tout déploiement de GPO en environnement multi-sites, exécutez cette séquence de vérification.

**Étape 1 — Confirmer la convergence DFS-R sur tous les DC distants.**

```powershell
# Verify DFS-R backlog is zero on all remote DCs after GPO change
$gpoGuid = "{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}"
$remoteDCs = @("DC-Lyon01", "DC-Bordeaux01", "DC-Toulouse01")

foreach ($dc in $remoteDCs) {
    $result = dfsrdiag Backlog `
        /rgname:"Domain System Volume" `
        /rfname:"SYSVOL Share" `
        /sendingmember:DC-Paris01 `
        /receivingmember:$dc

    $backlogCount = ($result | Select-String "Backlog File Count").ToString() -replace ".*: ",""
    Write-Host "$dc — Backlog: $backlogCount"
}
```

```title="Résultat attendu — Tous les sites convergés"
DC-Lyon01 — Backlog: 0
DC-Bordeaux01 — Backlog: 0
DC-Toulouse01 — Backlog: 0
```

**Étape 2 — Valider l'application sur un poste pilote de chaque site.**

```powershell
# Validate GPO application on a pilot workstation per site
$pilotMachines = @{
    "Lyon"     = "PC-PILOTE-LYON-01"
    "Bordeaux" = "PC-PILOTE-BDX-01"
    "Toulouse" = "PC-PILOTE-TLS-01"
}

foreach ($site in $pilotMachines.Keys) {
    $machine = $pilotMachines[$site]
    Write-Host "`n=== Pilote $site : $machine ===" -ForegroundColor Cyan

    Invoke-Command -ComputerName $machine -ScriptBlock {
        gpupdate /force | Out-Null
        $result = gpresult /r 2>&1
        $appliedGPOs = $result | Select-String "Applied Group Policy Objects" -A 10
        Write-Output $appliedGPOs
    }
}
```

```title="Résultat attendu"
=== Pilote Lyon : PC-PILOTE-LYON-01 ===
Applied Group Policy Objects
-----------------------------
    SITE-Lyon-Config-Proxy
    SEC-Baseline-Postes
    Default Domain Policy

=== Pilote Bordeaux : PC-PILOTE-BDX-01 ===
Applied Group Policy Objects
-----------------------------
    SEC-Baseline-Postes
    Default Domain Policy
```

**Étape 3 — Vérifier les Event IDs DFS-R sur les DC distants.**

```powershell
# Check for DFS-R error events on remote DCs in the last 2 hours
$remoteDCs = @("DC-Lyon01", "DC-Bordeaux01")
$criticalEventIDs = @(2213, 4012, 5002)
$since = (Get-Date).AddHours(-2)

foreach ($dc in $remoteDCs) {
    $events = Get-WinEvent -ComputerName $dc -FilterHashtable @{
        LogName   = "DFS Replication"
        Id        = $criticalEventIDs
        StartTime = $since
    } -ErrorAction SilentlyContinue

    if ($events) {
        Write-Warning "$dc — $($events.Count) DFS-R critical event(s) detected!"
        $events | Select-Object TimeCreated, Id, Message | Format-Table -AutoSize
    } else {
        Write-Host "$dc — No DFS-R critical events in the last 2 hours." -ForegroundColor Green
    }
}
```

```title="Résultat attendu — Environnement sain"
DC-Lyon01 — No DFS-R critical events in the last 2 hours.
DC-Bordeaux01 — No DFS-R critical events in the last 2 hours.
```


!!! quote "En résumé"
    - Après tout déploiement de GPO en environnement multi-sites, exécutez cette séquence de vérification.
    - Étape 1 — Confirmer la convergence DFS-R sur tous les DC distants.
    - Validez toujours le résultat sur un poste ou un utilisateur réellement dans le périmètre avant d’élargir.
    - Conservez les commandes et résultats de contrôle comme preuve de conformité post-déploiement.
---

## Cross-références

| Sujet | Référence |
|-------|-----------|
| Mécanismes internes DFS-R et structure SYSVOL | [La Bible GPO — Ch. 04 — SYSVOL](../bible-gpo/04-sysvol.md) |
| Copy-GPO et Import-GPO avec table de migration | [Ch. 05 — Sauvegarde et migration](05-backup-migration.md) |
| Gouvernance multi-domaine et séparation des rôles | [Ch. 02 — Gouvernance](02-gouvernance.md) |
| PowerShell GroupPolicy module — New-GPLink, New-GPO | [Ch. 03 — PowerShell GroupPolicy](03-powershell-gpo.md) |

!!! quote "En résumé"
    - À relire : Mécanismes internes DFS-R et structure SYSVOL → La Bible GPO — Ch. 04 — SYSVOL.
    - À relire : Copy-GPO et Import-GPO avec table de migration → Ch. 05 — Sauvegarde et migration.
    - À relire : Gouvernance multi-domaine et séparation des rôles → Ch. 02 — Gouvernance.
    - À relire : PowerShell GroupPolicy module — New-GPLink, New-GPO → Ch. 03 — PowerShell GroupPolicy.
    - À relire : La Bible GPO — Ch. 04 — SYSVOL.
