---
description: "Mettre en place une gouvernance GPO solide en entreprise : modele de delegation par roles, permissions GPMC, audit des droits, AGPM et documentation de production."
tags:
  - gpo
  - administration
  - gouvernance
  - delegation
---

# Gouvernance et délégation des GPO

!!! abstract "Ce que vous allez pouvoir faire"
    - Construire un modèle de délégation GPO à quatre rôles, sans toucher au groupe Domain Admins
    - Déléguer la création, l'édition et la liaison des GPO à des équipes distinctes avec des commandes PowerShell exactes
    - Auditer l'ensemble des permissions GPO de votre domaine et exporter le résultat en CSV
    - Identifier et corriger les cinq failles de gouvernance les plus courantes en production
    - Mettre en place une documentation minimale viable sans AGPM ni outil tiers

---

## Pourquoi la gouvernance GPO est un sujet à part entière

Une GPO liée à la racine du domaine avec des droits d'édition accordés à tout le monde, c'est une bombe à retardement. En production, les incidents GPO ne viennent presque jamais d'une mauvaise configuration technique — ils viennent d'une absence de gouvernance.

Quelqu'un a modifié une GPO un vendredi soir. Personne ne sait qui. Le lundi matin, 300 postes n'appliquent plus la configuration proxy. Il faut deux heures pour retrouver le coupable dans les journaux d'événements, en supposant que l'audit soit activé.

Ce chapitre construit le cadre qui évite ce scénario. Il s'applique dès 5 administrateurs touchant aux GPO, et devient critique à partir de 10.

:material-alert-circle-outline: **Ce chapitre suppose que vous avez lu le chapitre 1** (Architecture d'entreprise). La gouvernance s'appuie sur une structure d'OU cohérente et une convention de nommage en place.

!!! quote "En résumé"
    La gouvernance GPO n'est pas optionnelle au-delà de 3 personnes qui touchent aux GPO. L'absence de cadre explicite crée une situation où tout le monde peut tout faire — et où personne ne sait ce que les autres ont fait.

---

## :material-account-multiple-outline: Le modèle de délégation à quatre rôles

### Pourquoi quatre rôles distincts

GPMC expose un modèle de permissions qui ne correspond pas aux groupes AD intégrés. Vous devez le construire explicitement.

Le principe fondamental est la **séparation des pouvoirs** : celui qui peut modifier une GPO ne doit pas être celui qui décide de l'activer en production. Celui qui crée des GPO ne doit pas pouvoir les lier n'importe où.

| Rôle | Permissions requises | Portée typique |
|------|---------------------|----------------|
| **GPO Creator** | Membre de `Group Policy Creator Owners` | Équipe infrastructure |
| **GPO Editor** | Read + Edit settings sur les GPO spécifiques | Équipe applicative (ses GPO uniquement) |
| **GPO Approver** | Full Control sur les GPO (avec AGPM) | Responsable technique ou lead infra |
| **GPO Linker** | Permission de lier une GPO sur une OU spécifique | Administrateur local de l'OU |

### La séparation Edit / Link est clé

Un **GPO Editor** peut modifier les paramètres d'une GPO — mais ne peut pas la lier à une OU. Il ne peut donc pas déclencher lui-même l'application en production.

Un **GPO Linker** peut lier une GPO à une OU — mais ne voit pas et ne peut pas modifier ses paramètres. Il valide que la GPO va au bon endroit, pas ce qu'elle contient.

Cette séparation reproduit en miniature le principe des quatre yeux utilisé en sécurité bancaire.

### Les groupes AD à créer

Ces groupes n'existent pas par défaut. Créez-les explicitement :

```powershell
# Create delegation groups for GPO governance
New-ADGroup -Name "GRP-GPO-Creators"     -GroupScope Global -GroupCategory Security `
    -Description "Members can create new GPOs in GPMC"
New-ADGroup -Name "GRP-GPO-Approvers"   -GroupScope Global -GroupCategory Security `
    -Description "Full control over GPOs — production approval gate"
New-ADGroup -Name "GRP-GPO-Linkers"     -GroupScope Global -GroupCategory Security `
    -Description "Can link GPOs to designated OUs"
New-ADGroup -Name "GRP-Equipe-PDT"      -GroupScope Global -GroupCategory Security `
    -Description "Workstation team — editor on workstation GPOs only"
New-ADGroup -Name "GRP-Equipe-Serveurs" -GroupScope Global -GroupCategory Security `
    -Description "Server team — editor on server GPOs only"
```

```title="Résultat attendu"
New-ADGroup : les cinq groupes sont créés dans le conteneur Users par défaut.
Déplacez-les dans l'OU Groupes-de-securite définie au chapitre 1.
```

!!! quote "En résumé"
    Quatre rôles, cinq groupes. La séparation Edit/Link est le cœur du modèle : un éditeur qui ne peut pas lier ne peut pas déclencher un incident de production seul.

---

## :material-plus-circle-outline: Déléguer la création de GPO

### Le comportement par défaut

Par défaut, seuls les membres de `Domain Admins` et de `Group Policy Creator Owners` peuvent créer des GPO.

Donner `Domain Admins` à une équipe infrastructure pour qu'elle puisse créer des GPO est une erreur classique. C'est équivalent à donner les clés de la voiture et du coffre-fort au même personnel.

### Ajouter le groupe au bon endroit

```powershell
# Grant GPO creation rights to the infrastructure team
Add-ADGroupMember -Identity "Group Policy Creator Owners" `
    -Members "GRP-GPO-Creators"
```

```title="Résultat attendu"
Le groupe GRP-GPO-Creators est maintenant membre de Group Policy Creator Owners.
Ses membres peuvent créer de nouvelles GPO dans GPMC sans être Domain Admins.
```

### Le problème CREATOR OWNER

Quand un membre de `Group Policy Creator Owners` crée une GPO, AD lui attribue automatiquement une ACE `Full Control` via `CREATOR OWNER`. Cette permission reste active indéfiniment — même après que l'administrateur a quitté l'entreprise.

Au bout de 6 mois, des comptes d'ex-employés conservent `Full Control` sur des GPO de production. C'est une faille de sécurité documentée, pas un cas théorique.

:material-alert-outline: La correction est obligatoire en production. Après chaque création de GPO, retirez `CREATOR OWNER` :

```powershell
# Remove CREATOR OWNER from a GPO immediately after creation
# Run this right after New-GPO or after any new GPO is created via GPMC
$gpoName = "APP-Edge-ConfigEntreprise"

Set-GPPermissions -Name $gpoName `
    -TargetName "CREATOR OWNER" `
    -TargetType Group `
    -PermissionLevel None
```

```title="Résultat attendu"
La commande ne retourne rien si elle réussit.
Vérification : Get-GPPermissions -Name "APP-Edge-ConfigEntreprise" -All
CREATOR OWNER ne doit plus apparaître dans la liste.
```

### Automatiser le nettoyage CREATOR OWNER

En pratique, la correction manuelle est oubliée. Intégrez ce nettoyage dans votre script de création de GPO :

```powershell
# Wrapper function for GPO creation with automatic CREATOR OWNER cleanup
function New-ManagedGPO {
    param(
        [Parameter(Mandatory)] [string] $Name,
        [Parameter(Mandatory)] [string] $Comment,
        [string] $Domain = $env:USERDNSDOMAIN
    )

    # Create the GPO
    $gpo = New-GPO -Name $Name -Domain $Domain -Comment $Comment

    # Wait for AD replication before modifying permissions
    Start-Sleep -Seconds 2

    # Remove CREATOR OWNER — mandatory governance step
    Set-GPPermissions -Guid $gpo.Id `
        -TargetName "CREATOR OWNER" `
        -TargetType Group `
        -PermissionLevel None

    # Grant Full Control to the approvers group
    Set-GPPermissions -Guid $gpo.Id `
        -TargetName "GRP-GPO-Approvers" `
        -TargetType Group `
        -PermissionLevel GpoEditDeleteModifySecurity

    Write-Host "GPO created and secured: $Name" -ForegroundColor Green
    return $gpo
}

# Usage
New-ManagedGPO -Name "APP-Teams-ConfigEntreprise" `
               -Comment "Teams : configuration enterprise. Créé: 2026-04-05."
```

```title="Résultat attendu"
GPO created and secured: APP-Teams-ConfigEntreprise
```

!!! danger "Production — CREATOR OWNER sur toutes les GPO existantes"
    Si vous reprenez un environnement sans gouvernance, `CREATOR OWNER` est probablement présent sur toutes vos GPO existantes. Ne corrigez pas à la main. Utilisez le script d'audit de la section dédiée plus bas.

!!! quote "En résumé"
    `Group Policy Creator Owners` donne le droit de créer des GPO sans être Domain Admin. Mais chaque création déclenche une ACE `CREATOR OWNER Full Control` qu'il faut retirer immédiatement. Automatisez ce nettoyage dans une fonction wrapper.

---

## :material-pencil-outline: Déléguer l'édition d'une GPO spécifique

### Le principe de l'édition ciblée

Une équipe applicative ne doit pas pouvoir modifier toutes les GPO du domaine. Elle doit pouvoir modifier **ses GPO uniquement**.

GPMC permet de définir des permissions d'édition au niveau de chaque GPO. C'est précis, auditable, et révocable.

### Les niveaux de permission disponibles

| Niveau `Set-GPPermissions` | Accès réel |
|---------------------------|------------|
| `GpoRead` | Lecture seule — voir les paramètres sans les modifier |
| `GpoApply` | Appliquer la stratégie — utilisé pour le filtrage de sécurité |
| `GpoEdit` | Modifier les paramètres — ne peut pas lier ni supprimer |
| `GpoEditDeleteModifySecurity` | Contrôle total — peut tout faire sur cette GPO |
| `None` | Supprime toute permission existante |

:material-information-outline: `GpoApply` est le niveau attribué au groupe `Utilisateurs Authentifiés` par défaut. Il ne donne pas accès aux paramètres — il détermine qui reçoit la stratégie.

### Accorder les droits d'édition

```powershell
# Grant edit rights to the workstation team on their specific GPO
Set-GPPermissions -Name "APP-Edge-ConfigEntreprise" `
    -TargetName "GRP-Equipe-PDT" `
    -TargetType Group `
    -PermissionLevel GpoEdit
```

```title="Résultat attendu"
La commande retourne un objet GPPermission.
L'équipe PDT peut maintenant ouvrir la GPO dans GPMC et modifier ses paramètres.
Elle ne peut pas la lier, la renommer, ni la supprimer.
```

### Attribuer les droits d'édition sur plusieurs GPO d'un coup

```powershell
# Grant GpoEdit to a team on all GPOs matching a naming pattern
$team  = "GRP-Equipe-PDT"
$pattern = "APP-*"      # All application GPOs belong to this team

Get-GPO -All | Where-Object { $_.DisplayName -like $pattern } | ForEach-Object {
    Set-GPPermissions -Guid $_.Id `
        -TargetName $team `
        -TargetType Group `
        -PermissionLevel GpoEdit
    Write-Host "Edit granted on: $($_.DisplayName)"
}
```

```title="Résultat attendu"
Edit granted on: APP-Edge-ConfigEntreprise
Edit granted on: APP-Chrome-ConfigEntreprise
Edit granted on: APP-Teams-ConfigEntreprise
```

### Révoquer les droits d'édition

La révocation est aussi simple que l'attribution. C'est utile quand un prestataire intervient ponctuellement sur une GPO :

```powershell
# Revoke all permissions for a specific trustee on a specific GPO
Set-GPPermissions -Name "APP-Edge-ConfigEntreprise" `
    -TargetName "GRP-Prestataire-Externe" `
    -TargetType Group `
    -PermissionLevel None
```

```title="Résultat attendu"
Le groupe prestataire n'apparaît plus dans les permissions de la GPO.
À exécuter dès la fin de l'intervention.
```

!!! warning "À surveiller — GpoRead vs GpoApply"
    Ces deux niveaux sont souvent confondus. `GpoRead` permet de **voir** les paramètres dans GPMC. `GpoApply` détermine **qui reçoit** la stratégie — c'est le filtrage de sécurité. Une machine peut recevoir une GPO (GpoApply) sans qu'aucun administrateur ne puisse la lire (sans GpoRead). Ces deux axes sont indépendants.

!!! quote "En résumé"
    `GpoEdit` est le niveau minimal pour qu'une équipe puisse modifier les paramètres d'une GPO. Attribuez-le GPO par GPO, jamais en global. Révoquez-le dès que l'accès n'est plus nécessaire.

---

## :material-link-variant-plus: Déléguer la liaison d'une GPO à une OU

### La subtilité : la permission est sur l'OU, pas sur la GPO

L'édition d'une GPO se délègue sur la GPO elle-même. La liaison d'une GPO à une OU se délègue sur **l'objet OU dans AD** — pas dans GPMC.

C'est la source de confusion la plus fréquente sur ce sujet. Un administrateur qui cherche le droit de lier dans les propriétés de la GPO ne le trouvera pas. Il est dans l'ACL de l'OU.

### Délégation via la GPMC (interface graphique)

Dans GPMC, clic droit sur l'OU cible → "Délégation" → bouton "Ajouter" → sélectionner le groupe → choisir "Lier les objets de stratégie de groupe".

C'est la méthode la plus simple pour une délégation ponctuelle. Pour une délégation à l'échelle, utilisez PowerShell.

### Délégation via PowerShell (méthode production)

La délégation de liaison passe par la modification directe de l'ACL AD sur l'OU. Le GUID de l'attribut `gPLink` est un GUID de schéma AD fixe — identique sur tous les domaines Windows :

```powershell
# Delegate GPO linking permission on a specific OU
# The gPLink attribute GUID is fixed across all Windows AD schemas
Import-Module ActiveDirectory

$ouDN    = "OU=Postes-Standard,DC=contoso,DC=local"
$group   = "GRP-GPO-Linkers"

# Retrieve current ACL from the OU object
$ou      = Get-ADOrganizationalUnit -Identity $ouDN
$acl     = Get-Acl -Path "AD:\$($ou.DistinguishedName)"

# Build the ACE: WriteProperty on gPLink attribute
$identity  = (Get-ADGroup $group).SID
$adRight   = [System.DirectoryServices.ActiveDirectoryRights]::WriteProperty
$type      = [System.Security.AccessControl.AccessControlType]::Allow
$gplinkGuid = New-Object Guid "f30e3bc1-9ff0-11d1-b603-0000f80367c1"  # gPLink schema GUID

$ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule `
    $identity, $adRight, $type, $gplinkGuid

$acl.AddAccessRule($ace)
Set-Acl -Path "AD:\$($ou.DistinguishedName)" -AclObject $acl

Write-Host "GPO linking delegated on $ouDN to $group" -ForegroundColor Green
```

```title="Résultat attendu"
GPO linking delegated on OU=Postes-Standard,DC=contoso,DC=local to GRP-GPO-Linkers
```

### Vérifier la délégation

```powershell
# Verify that gPLink write permission is correctly set on the OU
$ouDN  = "OU=Postes-Standard,DC=contoso,DC=local"
$acl   = Get-Acl -Path "AD:\$ouDN"

$acl.Access | Where-Object {
    $_.ObjectType -eq [Guid]"f30e3bc1-9ff0-11d1-b603-0000f80367c1"
} | Select-Object IdentityReference, ActiveDirectoryRights, AccessControlType
```

```title="Résultat attendu"
IdentityReference            ActiveDirectoryRights  AccessControlType
-----------------            ---------------------  -----------------
CONTOSO\GRP-GPO-Linkers      WriteProperty          Allow
```

### Déléguer la liaison sur toutes les OU d'un niveau

```powershell
# Delegate GPO linking on all OUs directly under Postes-de-travail
$parentOU = "OU=Postes-de-travail,DC=contoso,DC=local"
$group    = "GRP-GPO-Linkers"
$identity = (Get-ADGroup $group).SID
$adRight  = [System.DirectoryServices.ActiveDirectoryRights]::WriteProperty
$type     = [System.Security.AccessControl.AccessControlType]::Allow
$gplinkGuid = New-Object Guid "f30e3bc1-9ff0-11d1-b603-0000f80367c1"

Get-ADOrganizationalUnit -SearchBase $parentOU -SearchScope OneLevel -Filter * |
    ForEach-Object {
        $acl = Get-Acl -Path "AD:\$($_.DistinguishedName)"
        $ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule `
            $identity, $adRight, $type, $gplinkGuid
        $acl.AddAccessRule($ace)
        Set-Acl -Path "AD:\$($_.DistinguishedName)" -AclObject $acl
        Write-Host "Delegated: $($_.DistinguishedName)"
    }
```

```title="Résultat attendu"
Delegated: OU=Postes-Pilote,OU=Postes-de-travail,DC=contoso,DC=local
Delegated: OU=Postes-Standard,OU=Postes-de-travail,DC=contoso,DC=local
Delegated: OU=Postes-Kiosk,OU=Postes-de-travail,DC=contoso,DC=local
Delegated: OU=Postes-Mobiles,OU=Postes-de-travail,DC=contoso,DC=local
```

!!! danger "Production — Ne pas confondre liaison et édition"
    Un administrateur avec droit de liaison sur une OU peut lier **n'importe quelle GPO** à cette OU — y compris des GPO qu'il n'a pas créées et dont il ne connaît pas le contenu. La délégation de liaison sans gouvernance sur les GPO elles-mêmes est aussi dangereuse que la délégation d'édition sans contrôle des liaisons.

!!! quote "En résumé"
    Le droit de liaison se délègue sur l'OU dans AD, via l'attribut `gPLink`. Il est totalement indépendant du droit d'édition de la GPO. Un Linker sans restriction peut lier n'importe quelle GPO — délimitez toujours le périmètre d'OU concerné.

---

## :material-shield-search: Auditer les permissions GPO

### Pourquoi l'audit est non négociable

Dans un environnement sans audit régulier, les permissions GPO dérivent. Des groupes temporaires restent actifs. Des prestataires conservent des accès après la fin de mission. `CREATOR OWNER` prolifère.

Un audit trimestriel prend 10 minutes avec les scripts ci-dessous. Un incident post-dérive peut prendre deux jours.

### Exporter toutes les permissions GPO

```powershell
# Full GPO permissions audit — exports all trustees and permission levels
Get-GPO -All | ForEach-Object {
    $gpoName = $_.DisplayName
    Get-GPPermissions -Guid $_.Id -All | ForEach-Object {
        [PSCustomObject]@{
            GPO        = $gpoName
            Trustee    = $_.Trustee.Name
            TrusteeType = $_.Trustee.SidType
            Permission = $_.Permission
            Inherited  = $_.Inherited
        }
    }
} | Export-Csv "C:\Temp\gpo-permissions-audit.csv" -NoTypeInformation -Encoding UTF8

Write-Host "Audit exported: C:\Temp\gpo-permissions-audit.csv" -ForegroundColor Green
```

```title="Résultat attendu"
Audit exported: C:\Temp\gpo-permissions-audit.csv

Le fichier contient une ligne par (GPO × Trustee).
Ouvrez dans Excel et filtrez la colonne Permission pour identifier les anomalies.
```

### Détecter les permissions trop larges

Ce script cherche les cas où des comptes ou groupes non administrateurs ont des droits d'édition ou de contrôle total :

```powershell
# Find GPOs where non-admin groups have Edit or higher permissions
$adminGroups = @(
    "Domain Admins",
    "Enterprise Admins",
    "SYSTEM",
    "GRP-GPO-Approvers"   # adjust to your environment
)

Get-GPO -All | ForEach-Object {
    $gpoName = $_.DisplayName
    Get-GPPermissions -Guid $_.Id -All |
        Where-Object {
            $_.Permission -in @("GpoEdit", "GpoEditDeleteModifySecurity") -and
            $_.Trustee.Name -notin $adminGroups
        } |
        ForEach-Object {
            [PSCustomObject]@{
                GPO      = $gpoName
                Trustee  = $_.Trustee.Name
                Level    = $_.Permission
                Flag     = "REVIEW REQUIRED"
            }
        }
} | Sort-Object GPO | Format-Table -AutoSize
```

```title="Résultat attendu"
GPO                          Trustee                    Level    Flag
---                          -------                    -----    ----
APP-Edge-ConfigEntreprise    CONTOSO\jdupont            GpoEdit  REVIEW REQUIRED
CFG-Postes-Reseau            CONTOSO\GRP-Prestataire-X  GpoEditDeleteModifySecurity  REVIEW REQUIRED
```

### Détecter CREATOR OWNER sur toutes les GPO

```powershell
# Find all GPOs where CREATOR OWNER still has explicit permissions
$creatorOwnerGPOs = Get-GPO -All | ForEach-Object {
    $perms = Get-GPPermissions -Guid $_.Id -All -ErrorAction SilentlyContinue
    $co    = $perms | Where-Object { $_.Trustee.Name -eq "CREATOR OWNER" }
    if ($co) {
        [PSCustomObject]@{
            GPO        = $_.DisplayName
            GUID       = $_.Id
            Permission = $co.Permission
            Created    = $_.CreationTime.ToString("yyyy-MM-dd")
            Modified   = $_.ModificationTime.ToString("yyyy-MM-dd")
        }
    }
}

$creatorOwnerGPOs | Format-Table -AutoSize

Write-Host "`n$($creatorOwnerGPOs.Count) GPO(s) with CREATOR OWNER permissions." -ForegroundColor Yellow
```

```title="Résultat attendu"
GPO                          GUID       Permission                    Created    Modified
---                          ----       ----------                    -------    --------
CFG-Postes-Reseau            {3f4a...}  GpoEditDeleteModifySecurity   2025-03-01 2025-03-01
APP-Edge-ConfigEntreprise    {7c2b...}  GpoEditDeleteModifySecurity   2025-09-12 2026-01-20
SEC-Postes-Baseline          {1a9c...}  GpoEditDeleteModifySecurity   2024-06-05 2025-11-30

3 GPO(s) with CREATOR OWNER permissions.
```

### Corriger CREATOR OWNER en masse

```powershell
# Remove CREATOR OWNER from all GPOs — run after audit review
Get-GPO -All | ForEach-Object {
    $perms = Get-GPPermissions -Guid $_.Id -All -ErrorAction SilentlyContinue
    if ($perms | Where-Object { $_.Trustee.Name -eq "CREATOR OWNER" }) {
        Set-GPPermissions -Guid $_.Id `
            -TargetName "CREATOR OWNER" `
            -TargetType Group `
            -PermissionLevel None
        Write-Host "Fixed: $($_.DisplayName)" -ForegroundColor Green
    }
}
```

```title="Résultat attendu"
Fixed: CFG-Postes-Reseau
Fixed: APP-Edge-ConfigEntreprise
Fixed: SEC-Postes-Baseline
```

### Détecter les GPO sans propriétaire explicite

```powershell
# List GPOs whose Owner is not a known admin group
$expectedOwners = @("Domain Admins", "Enterprise Admins", "CONTOSO\Infra-Team")

Get-GPO -All | Where-Object { $_.Owner -notin $expectedOwners } |
    Select-Object DisplayName, Owner, CreationTime |
    Sort-Object Owner |
    Format-Table -AutoSize
```

```title="Résultat attendu"
DisplayName               Owner               CreationTime
-----------               -----               ------------
GPO-test-proxy-ancien     CONTOSO\jmartin     01/06/2023 10:15:00
APP-Teams-v1-obsolete     CONTOSO\plegrand    15/09/2024 08:30:00
```

!!! quote "En résumé"
    Un audit GPO complet nécessite trois scripts : export global des permissions, détection des droits d'édition hors groupes admin, et détection de `CREATOR OWNER`. Planifiez ces trois scripts en tâche trimestrielle. Le résultat CSV s'ouvre dans Excel en 30 secondes.

---

## :material-source-branch: AGPM : workflow de validation structuré

### Ce qu'AGPM apporte

AGPM (Advanced Group Policy Management) est un composant optionnel du Microsoft Desktop Optimization Pack. Il ajoute un workflow de gestion des changements par-dessus GPMC :

1. **Draft** — L'éditeur crée ou modifie une GPO dans un environnement hors-ligne (archive AGPM)
2. **Pending** — L'éditeur soumet la modification pour validation
3. **Approved** — L'approbateur valide et déclenche le déploiement
4. **Deployed** — La GPO est active en production

Ce workflow rend chaque modification traçable, réversible et approbable avant mise en production.

### Quand AGPM est justifié

AGPM répond à des besoins spécifiques. Ne le déployez pas par défaut — il ajoute de la complexité opérationnelle.

| Contexte | AGPM justifié ? |
|----------|-----------------|
| Environnement régulé (ISO 27001, SOC2, PCI-DSS) | Oui — traçabilité obligatoire |
| Plus de 5 administrateurs modifiant des GPO | Oui — contrôle des conflits |
| Exigence de validation avant production | Oui — workflow formel |
| Équipe infra de 2-3 personnes, < 50 GPO | Non — overhead disproportionné |
| Environnement sans MDOP disponible | Non — coût de licence |

### L'alternative sans AGPM : Git + PowerShell

Pour les environnements sans AGPM, la version GPO + Git est une alternative viable. Elle est traitée en détail au chapitre 23 (Automatisation CI/CD). Le principe :

```powershell
# Export all GPOs to a folder for Git versioning
# Run this before and after any change as part of your change process
$exportPath = "C:\GPO-Backup\$(Get-Date -Format 'yyyy-MM-dd_HH-mm')"
New-Item -ItemType Directory -Path $exportPath -Force | Out-Null

Get-GPO -All | ForEach-Object {
    $gpoFolder = Join-Path $exportPath $_.DisplayName.Replace(" ", "_")
    Backup-GPO -Guid $_.Id -Path $exportPath | Out-Null
}

Write-Host "GPOs exported to: $exportPath" -ForegroundColor Green
# Commit this folder to Git — change history becomes your audit trail
```

```title="Résultat attendu"
GPOs exported to: C:\GPO-Backup\2026-04-05_14-30
```

:material-information-outline: La combinaison backup GPO + Git + tags de commit par numéro de ticket d'incident reproduit 80 % de la valeur d'AGPM sans le coût de déploiement.

!!! quote "En résumé"
    AGPM est pertinent dans les environnements régulés ou les grandes équipes. Pour les autres, un workflow Git sur les exports GPO fournit la traçabilité essentielle à un coût opérationnel minimal.

---

## :material-text-box-outline: Documenter les GPO

### Le minimum viable : le champ Description

GPMC expose un champ `Description` (aussi accessible via `-Comment` en PowerShell) sur chaque GPO. C'est le seul emplacement de documentation garanti accessible sans outil tiers, visible depuis GPMC, et conservé dans les exports/migrations.

Remplissez-le. Toujours. Dès la création.

```powershell
# Set or update the description on a GPO — minimum viable documentation
Set-GPO -Name "SEC-Postes-Baseline" -Comment "Baseline de securite pour tous les postes de travail.
Cree: 2026-01-15 par J.Dupont (ticket CHG-2026-001).
Lie a: OU=Postes-de-travail.
Modifiee: 2026-04-05 — Ajout desactivation USB autorun (incident INC-2026-042).
Sections actives: Computer uniquement."
```

```title="Résultat attendu"
Le champ Description est visible dans GPMC en cliquant sur la GPO → onglet Details.
Il est également exporté dans les rapports HTML et XML de Get-GPOReport.
```

### Le template de description standard

Standardisez le format pour que chaque GPO soit documentée de la même manière :

```
[Objet]     Configuration Edge pour les postes de travail.
[Cree]      2026-01-15 — J.Dupont — CHG-2026-001
[Lie a]     OU=Postes-de-travail
[Sections]  Computer uniquement (User désactivé)
[Modifs]
  2026-04-05 — J.Martin — INC-2026-042 : ajout règle SmartScreen
  2026-03-10 — J.Dupont — CHG-2026-089 : mise à jour page de démarrage
```

```powershell
# Bulk-check which GPOs have an empty description
Get-GPO -All | Where-Object { [string]::IsNullOrWhiteSpace($_.Description) } |
    Select-Object DisplayName, CreationTime, ModificationTime |
    Sort-Object CreationTime |
    Format-Table -AutoSize
```

```title="Résultat attendu"
DisplayName               CreationTime           ModificationTime
-----------               ------------           ----------------
GPO-test-proxy            01/06/2023 10:15:00    01/06/2023 10:15:00
APP-Teams-ConfigEntreprise 05/04/2026 09:30:00   05/04/2026 09:30:00
```

### Le nommage comme première ligne de documentation

Un nom GPO bien construit communique son objet sans qu'on ait à l'ouvrir. C'est la documentation la plus fiable parce qu'elle est impossible à oublier — le nom s'affiche partout.

`SEC-Postes-Baseline` communique : domaine sécurité, cible postes, type baseline.

`GPO-Config-Final-v3` ne communique rien d'exploitable 6 mois plus tard.

La convention de nommage est définie au chapitre 1. Ce rappel est intentionnel : la gouvernance commence par le nom.

!!! quote "En résumé"
    Le champ Description de GPMC est le seul emplacement de documentation universellement accessible. Définissez un template de deux à cinq lignes et appliquez-le à toutes les GPO, y compris les existantes. Un script de détection des GPO sans description permet de prioriser le rattrapage.

---

## :material-alert-outline: Pièges en production

### Piège 1 — CREATOR OWNER actif depuis la création

!!! danger "Production — Vecteur de persistance non documenté"
    `CREATOR OWNER` est la faille de gouvernance la plus fréquente et la moins visible. Par conception AD, toute GPO créée accorde `Full Control` à son créateur via cet héritage — même si le créateur ne fait plus partie de l'organisation.

    Un ex-employé dont le compte AD est désactivé mais non supprimé conserve `Full Control` sur toutes les GPO qu'il a créées. Si le compte est réactivé pour une raison quelconque (audit, migration), la porte est ouverte.

    **Fréquence observée en production :** dans les environnements sans gouvernance, 100 % des GPO créées manuellement ont encore `CREATOR OWNER` actif.

    **Correction :** utilisez le script d'audit de la section précédente. Exécutez la correction en masse. Intégrez la suppression dans `New-ManagedGPO`.

---

### Piège 2 — Group Policy Creator Owners trop peuplé

!!! warning "À surveiller — Groupe à haut privilège souvent oublié"
    `Group Policy Creator Owners` est l'un des groupes AD à plus fort impact et l'un des moins surveillés. Ses membres peuvent créer des GPO dans n'importe quelle OU du domaine.

    **Règle de sécurité :** ce groupe ne doit jamais contenir plus de 3 à 5 comptes nominatifs. Préférez des comptes de service dédiés à l'automatisation plutôt que des comptes personnels.

    **Audit rapide :**

```powershell
# List all members of Group Policy Creator Owners
Get-ADGroupMember -Identity "Group Policy Creator Owners" -Recursive |
    Select-Object Name, SamAccountName, objectClass |
    Format-Table -AutoSize
```

```title="Résultat attendu"
Name            SamAccountName  objectClass
----            --------------  -----------
Julien Dupont   jdupont         user
Prestataire X   ext-prestatx    user
GRP-GPO-Creators GRP-GPO-Creat  group
```

---

### Piège 3 — GPO non liées avec des paramètres actifs

!!! warning "À surveiller — Mine dormante en production"
    Une GPO non liée n'est pas appliquée — mais elle n'est pas inoffensive. Ses paramètres sont là, prêts à s'appliquer si quelqu'un la lie par erreur à la mauvaise OU.

    **Scénario réel :** une GPO de test créée pour valider une configuration de proxy restrictive est oubliée sans être supprimée. Six mois plus tard, un administrateur junior cherche à "activer la GPO proxy" et voit cette GPO dans GPMC. Il la lie à `Postes-de-travail`. 300 postes perdent l'accès internet.

    **Règle :** toute GPO non liée depuis plus de 30 jours doit être soit liée définitivement soit supprimée.

```powershell
# Find all unlinked GPOs older than 30 days
$cutoff = (Get-Date).AddDays(-30)

Get-GPO -All | ForEach-Object {
    $report = [xml](Get-GPOReport -Guid $_.Id -ReportType Xml)
    if (-not $report.GPO.LinksTo -and $_.CreationTime -lt $cutoff) {
        [PSCustomObject]@{
            Name        = $_.DisplayName
            Created     = $_.CreationTime.ToString("yyyy-MM-dd")
            LastModified = $_.ModificationTime.ToString("yyyy-MM-dd")
            DaysOld     = [int]((Get-Date) - $_.CreationTime).TotalDays
        }
    }
} | Sort-Object DaysOld -Descending | Format-Table -AutoSize
```

```title="Résultat attendu"
Name                    Created    LastModified  DaysOld
----                    -------    ------------  -------
GPO-test-proxy-restrict 2025-06-01 2025-06-02    308
APP-Teams-v1-draft      2025-11-15 2025-11-15    141
```

---

### Piège 4 — Délégation accordée à des comptes nominatifs

!!! warning "À surveiller — Délégation non révocable en cas de départ"
    Attribuer des droits GPO directement à `jdupont` plutôt qu'à `GRP-Equipe-PDT` crée une dépendance à un compte individuel.

    Quand `jdupont` quitte l'entreprise, son compte est désactivé — mais ses permissions GPO restent dans les ACL jusqu'à ce que quelqu'un les retire explicitement. Active Directory ne nettoie pas automatiquement les ACE des comptes désactivés.

    **Règle absolue :** délégation uniquement sur des groupes, jamais sur des comptes individuels. Si vous trouvez des ACE nominatives dans votre audit, migrez-les vers des groupes avant de désactiver le compte concerné.

---

### Piège 5 — Droits d'édition accordés sans date d'expiration

!!! warning "À surveiller — Les accès temporaires deviennent permanents"
    "Je te donne accès à cette GPO pour l'intervention de la semaine prochaine" — et six mois plus tard, le prestataire a encore `GpoEdit` sur quinze GPO de production.

    AD n'a pas de mécanisme natif d'expiration des ACE GPO. La gestion est manuelle.

    **Pratique recommandée :** créez un ticket de révocation pour chaque accès temporaire accordé, planifié à la date de fin de mission. Intégrez la vérification des accès temporaires dans votre audit trimestriel.

```powershell
# Quick check: find groups with "prestataire" or "externe" in their name having edit rights
Get-GPO -All | ForEach-Object {
    $gpoName = $_.DisplayName
    Get-GPPermissions -Guid $_.Id -All |
        Where-Object {
            $_.Trustee.Name -match "prestataire|externe|vendor|contractor" -and
            $_.Permission -in @("GpoEdit", "GpoEditDeleteModifySecurity")
        } |
        ForEach-Object {
            [PSCustomObject]@{
                GPO     = $gpoName
                Trustee = $_.Trustee.Name
                Level   = $_.Permission
                Action  = "REVOKE IF MISSION ENDED"
            }
        }
} | Format-Table -AutoSize
```

```title="Résultat attendu"
GPO                       Trustee                      Level    Action
---                       -------                      -----    ------
APP-Edge-ConfigEntreprise CONTOSO\GRP-Prestataire-Ext  GpoEdit  REVOKE IF MISSION ENDED
```

!!! quote "En résumé"
    Les cinq pièges de production se résument à deux principes : automatisez le nettoyage de `CREATOR OWNER` dès la création, et n'accordez jamais de droits GPO à des comptes individuels. Le reste (GPO orphelines, groupe trop peuplé, accès temporaires) se traite dans l'audit trimestriel.

---

## :material-table: Matrice des responsabilités (RACI simplifié)

Pour documenter qui fait quoi dans votre équipe, utilisez cette matrice comme point de départ. Adaptez les noms de rôles à votre organisation :

| Action | Infra Lead | GPO Creator | GPO Editor | GPO Linker | Ops |
|--------|-----------|-------------|------------|------------|-----|
| Créer une GPO | A | R | — | — | I |
| Modifier les paramètres | A | — | R | — | I |
| Lier une GPO à une OU pilote | A | — | — | R | I |
| Valider en pilote (48h) | R | — | C | — | I |
| Lier une GPO en production | A | — | — | R | I |
| Supprimer une GPO | R | — | — | — | I |
| Auditer les permissions | R | — | — | — | C |
| Documenter (Description) | A | R | R | — | I |

*R = Responsable, A = Approbateur, C = Consulté, I = Informé*

---

## :material-book-open-outline: Références croisées

| Sujet | Référence |
|-------|-----------|
| Architecture et structure des OU | Ch. 01 — Architecture d'entreprise |
| PowerShell GroupPolicy (cmdlets complets) | Ch. 03 — PowerShell GroupPolicy module |
| Sauvegarde et restauration des GPO | Ch. 05 — Sauvegarde, restauration et migration |
| Audit et conformité GPO | Ch. 06 — Audit et conformité |
| Automatisation CI/CD et versioning Git | Ch. 23 — Automatisation CI/CD |
| Sécurisation des GPO elles-mêmes | Ch. 24 — Sécurité des GPO |
| Filtrage de sécurité (Apply vs Read) | La Bible GPO — Ch. 09 — Filtrage |
| LSDOU et ordre d'application | La Bible GPO — Ch. 08 — Héritage LSDOU |
