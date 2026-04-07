---
description: Monter un lab Active Directory pour pratiquer les GPO et le Registre Windows
tags: [lab, active-directory, hyper-v, virtualisation, pratique]
---

# Monter son lab AD - Guide complet

!!! abstract "Ce que vous allez construire"
    - Un domaine AD minimal mais realiste pour tester GPO, registre, DNS et SYSVOL
    - Deux DC pour valider la replication DFS-R et les comportements multi-serveurs
    - Deux clients Windows 11 pour comparer application machine et utilisateur
    - Un environnement jetable a snapshots, ideal avant chaque chapitre pratique
    - Une base de remise a zero pour refaire les exercices sans dette technique

!!! tip "Si vous ne retenez qu'une chose"
    N'investissez pas d'abord dans plus de machines virtuelles. Investissez dans un plan de nommage simple, des snapshots propres et des GPO de test prefixees `LAB-`. C'est cela qui rend un lab reutilisable.

## 1. Prerequis materiels

Le minimum utile pour un vrai lab GPO/registre reste modeste, mais il ne faut pas sous-estimer la RAM. Les DC, les clients Windows 11, les snapshots et l'I/O SYSVOL se comportent nettement mieux sur SSD.

Conditions de base :

- CPU x64 avec virtualisation activee dans l'UEFI/BIOS (`Intel VT-x/VT-d` ou `AMD-V`)
- Support SLAT recommande pour Hyper-V
- SSD ou NVMe vivement recommande
- 16 Go de RAM comme point de depart realiste

| Budget / confort | RAM hote | CPU | Stockage | Ce que vous pouvez faire |
|------------------|----------|-----|----------|--------------------------|
| Minimum viable | 16 Go | 4 coeurs / 8 threads | 250 Go SSD | 1 DC + 1 client + snapshots limites |
| Recommande | 32 Go | 6 a 8 coeurs | 500 Go SSD/NVMe | 2 DC + 2 clients + tests confortables |
| Large | 64 Go+ | 8 coeurs+ | 1 To NVMe | Plusieurs branches de snapshots et scenarios paralleles |

!!! warning "Piege classique"
    Un disque dur mecanique transforme un bon lab en faux negatif permanent : demarrage lent, SYSVOL lent, login lent, et vous accusez a tort les GPO.

## 2. Choix de l'hyperviseur

| Hyperviseur | Points forts | Limites | Bon choix si... |
|-------------|--------------|---------|-----------------|
| Hyper-V | Integre a Windows 10/11 Pro, tres stable, checkpoints propres, switch virtuel simple | Pas disponible sur Home, conflits possibles avec certains hyperviseurs tiers | Votre poste est deja sous Windows Pro/Enterprise |
| VirtualBox | Gratuit, tres simple a prendre en main, portable | Performances et integration parfois moins propres sur Windows 11 | Vous voulez du gratuit et du leger |
| VMware Workstation | Tres mature, excellente compatibilite, snapshots solides | Payant selon edition ou contexte | Vous privilegiez le confort d'usage et la maturite |

!!! example "Pas a pas - si vous etes sous Windows 10/11 Pro"
    1. Activez Hyper-V.
    2. Creez un vSwitch interne dedie au lab.
    3. Donnez au reseau du lab une plage simple, par exemple `192.168.100.0/24`.
    4. Isolez ce reseau du LAN de production si vous debutez.

## 3. VMs a creer

| VM | OS | vCPU | RAM | Disque | Role |
|----|----|------|-----|--------|------|
| `DC01` | Windows Server 2022 | 2 | 4 Go | 60 Go | Premier DC, DNS, PDC Emulator, authorite GPO |
| `DC02` | Windows Server 2022 | 2 | 4 Go | 60 Go | Deuxieme DC pour replication AD et DFS-R |
| `CLIENT01` | Windows 11 | 2 | 4 Go | 64 Go | Poste de test standard utilisateur |
| `CLIENT02` | Windows 11 | 2 | 4 Go | 64 Go | Poste de comparaison et A/B testing |

Conseils simples :

- Conservez des noms courts et fixes.
- Utilisez des IP statiques pour les DC.
- Laissez les clients en DHCP si vous voulez aussi tester les options reseau.

## 4. Installation pas a pas de `DC01`

Choisissez un nom de domaine purement labo, par exemple `lab.local` ou `corp.lab`. Evitez de reutiliser un vrai domaine de production.

!!! example "Pas a pas - promotion de DC01"
    1. Installez Windows Server 2022 sur `DC01`.
    2. Renommez le serveur, fixez une IP statique et pointez le DNS sur lui-meme.
    3. Installez le role AD DS avec les outils.
    4. Promouvez le serveur en nouveau domaine ou nouvelle foret.
    5. Choisissez un niveau fonctionnel domaine et foret au minimum Windows Server 2016.
    6. Verifiez que DNS, `NETLOGON` et `SYSVOL` sont presents avant d'aller plus loin.

Exemple PowerShell :

```powershell
# Install AD DS and management tools
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Promote the server as a new forest root
Install-ADDSForest `
    -DomainName "lab.local" `
    -DomainNetbiosName "LAB" `
    -InstallDNS `
    -ForestMode WinThreshold `
    -DomainMode WinThreshold `
    -SafeModeAdministratorPassword (Read-Host -AsSecureString)
```

Verification immediate apres reboot :

```powershell
Get-Service NTDS, DNS
Get-SmbShare SYSVOL, NETLOGON
dcdiag /q
```

!!! warning "Ne continuez pas tant que SYSVOL n'est pas la"
    Si `SYSVOL` ou `NETLOGON` manquent, ne joignez aucun client. Corrigez d'abord le DC.

## 5. Jonction des clients au domaine

Une fois `DC01` stable, creez d'abord votre structure de test. C'est ce qui rendra les chapitres GPO plus lisibles.

Structure minimale recommandee :

- `OU=LAB`
- `OU=Clients,OU=LAB`
- `OU=Admins,OU=LAB`
- `OU=Utilisateurs,OU=LAB`
- `OU=Serveurs,OU=LAB`

Comptes utiles :

- `lab.admin` : admin de labo, separe du compte `Administrator`
- `u.test1` : utilisateur standard
- `u.test2` : utilisateur de comparaison

!!! example "Pas a pas - jonction de CLIENT01 et CLIENT02"
    1. Sur `DC01`, creez les OUs de test.
    2. Creez `u.test1` et `u.test2`.
    3. Pointez `CLIENT01` et `CLIENT02` vers le DNS de `DC01`.
    4. Joignez les deux machines au domaine `lab.local`.
    5. Deplacez-les dans `OU=Clients,OU=LAB`.
    6. Connectez-vous avec `LAB\\u.test1`, puis lancez `gpresult /r`.

Exemple PowerShell cote DC :

```powershell
Import-Module ActiveDirectory

$base = "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "LAB" -Path $base -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Clients" -Path "OU=LAB,$base"
New-ADOrganizationalUnit -Name "Admins" -Path "OU=LAB,$base"
New-ADOrganizationalUnit -Name "Utilisateurs" -Path "OU=LAB,$base"

New-ADUser -Name "u.test1" -SamAccountName "u.test1" `
    -AccountPassword (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force) `
    -Enabled $true -Path "OU=Utilisateurs,OU=LAB,$base"
```

## 5bis. Promouvoir DC02 comme deuxieme controleur de domaine

Le deuxième DC transforme le lab en vrai terrain de test AD : réplication, SYSVOL multi-DC, bascule DNS et diagnostic de cohérence GPO.

```powershell title="Promouvoir DC02"
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Install-ADDSDomainController `
    -DomainName "lab.local" `
    -SiteName "Default-First-Site-Name" `
    -InstallDns:$true `
    -Credential (Get-Credential "LAB\Administrator") `
    -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password") `
    -Force:$true
```

Après la réplication initiale :

```powershell
repadmin /replsummary
dfsrdiag pollad
```

Avantages lab :

- tester la réplication AD ;
- vérifier DFS-R SYSVOL ;
- simuler la perte d'un DC ;
- observer le comportement GPO quand plusieurs DC répondent.

### Creer le Central Store

```powershell title="Creer le Central Store dans SYSVOL"
$sysvolPath = "\\lab.local\SYSVOL\lab.local\Policies\PolicyDefinitions"
New-Item -Path $sysvolPath -ItemType Directory -Force
Copy-Item "C:\Windows\PolicyDefinitions\*" $sysvolPath -Recurse -Force
```

Après copie, GPMC utilise automatiquement le Central Store au lieu des fichiers ADMX locaux.

Vérification : ouvrez GPMC, éditez une GPO, puis regardez les modèles d'administration.
Le titre doit indiquer que les modèles viennent du **Central Store**.

### Installer RSAT sur CLIENT01

```powershell title="Installer les outils RSAT"
Get-WindowsCapability -Online -Name "RSAT*" |
    Where-Object { $_.State -ne "Installed" } |
    Add-WindowsCapability -Online
```

RSAT permet de gérer AD, DNS, DHCP et GPMC depuis un poste client.
Dans le lab, c'est utile pour pratiquer l'administration sans ouvrir directement une session sur le DC.

### Creer les GPO de test

```powershell title="Creer les GPO de base pour le lab"
# Security baseline
New-GPO -Name "LAB-Security-Baseline" -Comment "Baseline securite pour le lab"
New-GPLink -Name "LAB-Security-Baseline" -Target "OU=LAB,DC=lab,DC=local"

# Desktop settings
New-GPO -Name "LAB-Desktop-Settings" -Comment "Parametres bureau pour le lab"
New-GPLink -Name "LAB-Desktop-Settings" -Target "OU=LAB,DC=lab,DC=local"
```

Recommandation : préfixez toutes les GPO de lab avec `LAB-`.
Le script de nettoyage plus bas supprime uniquement les GPO `LAB-*`, ce qui limite les accidents.

### Extension cloud : Entra ID free tenant

Un lab cloud-hybride minimal permet de tester les notions Hybrid Join, synchronisation d'identités et enrollment sans connecter le domaine à la production.

Étapes de principe :

1. Créer un tenant Entra ID gratuit depuis `entra.microsoft.com`
2. Installer Microsoft Entra Connect Sync ou Entra Cloud Sync
3. Synchroniser quelques comptes AD de test vers Entra ID
4. Tester Hybrid Join et les scénarios d'enrollment disponibles dans votre licence

!!! warning "Ne jamais connecter le lab au tenant de production"
    Un lab AD doit rester jetable.
    Ne le synchronisez pas avec un tenant Entra ID de production, même "juste pour tester".

Limitation : un tenant gratuit ne fournit pas les licences Intune.
Il permet de tester la synchronisation et certains prérequis cloud, mais pas tout le cycle de gestion Intune en production.

## 6. Snapshots strategiques

Les snapshots sont votre veritable gain de temps. Faites-en peu, mais au bon moment.

| Nom de snapshot | Quand le prendre | Pourquoi |
|-----------------|------------------|----------|
| `00-clean-install` | Juste apres installation de l'OS | Revenir a une VM brute |
| `10-dc01-promoted` | Apres promotion de `DC01` et validation `SYSVOL` | Eviter de refaire tout AD DS |
| `20-clients-joined` | Apres jonction de `CLIENT01` et `CLIENT02` | Base ideale avant les premiers chapitres GPO |
| `30-dc02-ready` | Apres ajout de `DC02` et replication stable | Tester DFS-R, multi-DC et pannes |
| `40-before-lab-xyz` | Avant chaque gros exercice | Pouvoir refaire le scenario a l'identique |

!!! warning "Sur les DC"
    Les snapshots de DC sont acceptables en lab, mais gardez-les pedagogiques et courts. En production, ce serait un autre sujet.

## 7. Scripts de reinitialisation

Le meilleur standard est simple : tout ce qui est jetable porte un prefixe `LAB-`. Vos GPO, groupes et OUs de test doivent suivre cette regle.

!!! example "Pas a pas - remise a zero logique du lab"
    1. Supprimez les GPO de test prefixees `LAB-`.
    2. Nettoyez les liens restants.
    3. Videz les OUs jetables, mais ne touchez ni aux GPO par defaut ni aux comptes d'administration.
    4. Recreez la structure minimale.

Exemple PowerShell de nettoyage :

```powershell
Import-Module GroupPolicy
Import-Module ActiveDirectory

$domainDn = "DC=lab,DC=local"
$labRoot  = "OU=LAB,$domainDn"
$keepOus  = @(
    "OU=Clients,$labRoot",
    "OU=Admins,$labRoot",
    "OU=Utilisateurs,$labRoot",
    "OU=Serveurs,$labRoot"
)

# Remove test GPOs only - never touch default GPOs
Get-GPO -All |
    Where-Object { $_.DisplayName -like "LAB-*" } |
    ForEach-Object { Remove-GPO -Guid $_.Id -Confirm:$false }

# Remove child OUs not in the baseline list
Get-ADOrganizationalUnit -SearchBase $labRoot -SearchScope OneLevel -Filter * |
    Where-Object { $_.DistinguishedName -notin $keepOus } |
    Remove-ADOrganizationalUnit -Recursive -Confirm:$false

# Disable or remove test accounts prefixed LAB-
Get-ADUser -SearchBase "OU=Utilisateurs,$labRoot" -Filter 'SamAccountName -like "LAB-*"' |
    Remove-ADUser -Confirm:$false
```

!!! warning "Prefixe obligatoire"
    Sans convention de nommage (`LAB-`), votre script de nettoyage finira un jour par supprimer un objet que vous vouliez garder.

## 8. Ressources complementaires

- [Windows Server 2022 Evaluation Center (180 jours)](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022)
- [Telecharger l'ISO Windows 11](https://www.microsoft.com/software-download/windows11)
- [Windows Server technical documentation](https://learn.microsoft.com/windows-server/)
- [Installer Active Directory Domain Services](https://learn.microsoft.com/windows-server/identity/ad-ds/deploy/install-active-directory-domain-services--level-100-)
- [Installer Hyper-V sur Windows](https://learn.microsoft.com/windows-server/virtualization/hyper-v/get-started/install-hyper-v)

## Checklist de fin

- `DC01` et `DC02` se resolvent mutuellement en DNS
- `SYSVOL` et `NETLOGON` existent sur les deux DC
- `CLIENT01` et `CLIENT02` sont dans `OU=Clients,OU=LAB`
- `gpresult /r` retourne quelque chose de propre sur les deux clients
- Un snapshot `20-clients-joined` existe avant vos vrais TP

!!! tip "Ordre ideal"
    Faites d'abord fonctionner un petit lab propre. Ajoutez la complexite ensuite. Deux DC et deux clients bien nommes valent mieux que huit VMs mal tenues.
