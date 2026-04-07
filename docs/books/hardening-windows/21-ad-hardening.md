---
description: "Durcissement de l'Active Directory : AdminSDHolder, Kerberoasting, delegation, tiering model, LDAP signing et rotation KRBTGT."
tags:
  - hardening
  - active-directory
  - kerberos
  - tiering
---

# Active Directory hardening

!!! abstract "Ce que vous allez apprendre"
    - Pourquoi le durcissement AD est une priorité même dans un parc Windows déjà baseliné.
    - Comment réduire les chemins d'attaque liés à Kerberos, aux délégations, à DCSync et aux ACL.
    - Quels contrôles mettre en place sur LDAP, KRBTGT, les DCs et le modèle d'administration.

Active Directory est la racine opérationnelle de la plupart des environnements Windows.
Un poste peut être réinstallé, un serveur membre peut être reconstruit, mais un annuaire compromis remet en cause les identités, les tickets Kerberos, les GPO, les certificats et les droits d'administration.

Le durcissement AD ne consiste pas à empiler des réglages isolés.
Il consiste à réduire les chemins qui permettent de devenir administrateur du domaine sans l'être explicitement : mauvaises ACL, délégations trop larges, comptes de service faibles, protocoles LDAP non signés, DCs exposés et hygiène KRBTGT oubliée.

!!! quote "En résumé"
    Durcir AD, c'est protéger la source d'autorité des identités Windows.
    Les baselines poste et serveur restent utiles, mais elles ne compensent pas un annuaire trop permissif.

---

## Pourquoi durcir Active Directory

Un attaquant n'a pas toujours besoin d'exploiter une faille système.
Dans un domaine AD, il peut souvent progresser par abus de configuration :

- un compte de service avec SPN et mot de passe faible ;
- une délégation Kerberos non contrainte ;
- une ACL qui donne des droits de réplication à un groupe trop large ;
- un administrateur de domaine qui ouvre une session sur un poste utilisateur ;
- un DC qui expose des services inutiles ;
- une CA ADCS qui émet des certificats d'authentification trop librement.

Le durcissement doit donc combiner trois angles :

| Angle | Question | Exemple de contrôle |
|---|---|---|
| Identité | Qui peut devenir qui ? | Groupes privilégiés, AdminSDHolder, DCSync |
| Protocole | Que peut-on négocier ? | LDAP signing, channel binding, NTLM, Kerberos |
| Administration | Où les privilèges circulent-ils ? | Tiering model, PAW, séparation des comptes |

!!! warning "Erreur de cadrage"
    Une forêt AD n'est pas "interne donc sûre".
    Elle est la cible principale dès qu'un attaquant obtient un premier accès domaine.

!!! quote "En résumé"
    L'objectif n'est pas de rendre AD parfait.
    L'objectif est de rendre les chemins d'escalade visibles, limités et surveillés.

---

## AdminSDHolder et SDProp

`AdminSDHolder` est un objet spécial qui sert de modèle de sécurité pour les comptes et groupes protégés.
Le processus `SDProp` recopie périodiquement son descripteur de sécurité sur les objets membres de groupes protégés, comme `Domain Admins`, `Enterprise Admins` ou `Administrators`.

Ce mécanisme protège les objets sensibles contre des délégations OU trop larges.
Il peut aussi masquer un problème : une ACL ajoutée sur `AdminSDHolder` se propage ensuite aux objets protégés.

### Contrôles à effectuer

```powershell title="Lister les objets protégés par adminCount"
# Requires the Active Directory module
Get-ADObject -LDAPFilter "(adminCount=1)" `
    -Properties adminCount, distinguishedName |
    Select-Object Name, ObjectClass, DistinguishedName |
    Sort-Object ObjectClass, Name
```

```powershell title="Lire les ACL de AdminSDHolder"
$domainDN = (Get-ADDomain).DistinguishedName
$adminSdHolder = "CN=AdminSDHolder,CN=System,$domainDN"

Get-Acl "AD:\$adminSdHolder" |
    Select-Object -ExpandProperty Access |
    Select-Object IdentityReference, ActiveDirectoryRights, AccessControlType, IsInherited |
    Sort-Object IdentityReference
```

| Signal | Interprétation |
|---|---|
| Compte métier avec `adminCount=1` | Ancien privilège ou appartenance à nettoyer |
| ACE non héritée inattendue sur `AdminSDHolder` | Risque de persistance privilégiée |
| Groupe support dans ACL AdminSDHolder | Délégation trop large sur comptes protégés |

!!! tip "Routine saine"
    Auditez `AdminSDHolder` après chaque changement d'équipe IAM, après un incident, et avant tout audit externe.
    Une ACL parasite sur cet objet est une persistance discrète.

!!! quote "En résumé"
    `AdminSDHolder` protège les comptes privilégiés, mais il peut aussi propager une mauvaise délégation.
    Ses ACL doivent être aussi contrôlées qu'un groupe `Domain Admins`.

---

## Prévention DCSync

Une attaque DCSync abuse des droits de réplication AD pour demander les secrets du domaine comme le ferait un contrôleur de domaine.
Les droits critiques sont notamment :

- `Replicating Directory Changes` ;
- `Replicating Directory Changes All` ;
- `Replicating Directory Changes In Filtered Set`.

Ces droits doivent rester limités aux DCs et aux services explicitement prévus, par exemple certains outils de synchronisation d'identité.

```powershell title="Rechercher les droits de réplication sur la racine du domaine"
# Requires the Active Directory module and the AD: provider
$domainDN = (Get-ADDomain).DistinguishedName
$acl = Get-Acl "AD:\$domainDN"

$replicationRights = @{
    "1131f6aa-9c07-11d1-f79f-00c04fc2dcd2" = "Replicating Directory Changes"
    "1131f6ad-9c07-11d1-f79f-00c04fc2dcd2" = "Replicating Directory Changes All"
    "89e95b76-444d-4c62-991a-0facbeda640c" = "Replicating Directory Changes In Filtered Set"
}

$acl.Access |
    Where-Object { $replicationRights.ContainsKey($_.ObjectType.ToString()) } |
    Select-Object IdentityReference, ActiveDirectoryRights,
        @{N="ReplicationRight";E={$replicationRights[$_.ObjectType.ToString()]}},
        AccessControlType, IsInherited
```

!!! warning "Validation nécessaire"
    Les GUID des droits étendus AD varient moins que les libellés, mais les scripts d'audit doivent être testés dans un lab.
    En production, comparez toujours la sortie avec `dsacls` ou un outil IAM maîtrisé avant de supprimer une ACE.

Contrôles complémentaires :

- alerter sur les événements de réplication inhabituels depuis une machine qui n'est pas DC ;
- limiter les comptes de synchronisation à une OU ou un besoin précis quand l'outil le permet ;
- retirer les droits de réplication aux groupes historiques devenus inutiles ;
- protéger les comptes de synchronisation comme des comptes Tier 0.

!!! quote "En résumé"
    DCSync n'a pas besoin d'un shell sur un DC.
    Il suffit de droits de réplication trop larges, donc ces ACL doivent faire partie du contrôle Tier 0.

---

## Kerberoasting et AS-REP Roasting

Kerberoasting cible les comptes avec SPN.
Un utilisateur authentifié peut demander un ticket de service, puis tenter de casser hors ligne le secret du compte de service associé.

AS-REP Roasting cible les comptes dont l'option `Do not require Kerberos preauthentication` est activée.
Dans ce cas, un attaquant peut obtenir une donnée chiffrée exploitable hors ligne sans connaître le mot de passe.

```powershell title="Identifier les comptes avec SPN"
Get-ADUser -LDAPFilter "(servicePrincipalName=*)" `
    -Properties servicePrincipalName, PasswordLastSet, Enabled |
    Select-Object SamAccountName, Enabled, PasswordLastSet, ServicePrincipalName
```

```powershell title="Identifier les comptes sans pré-authentification Kerberos"
Get-ADUser -Filter { DoesNotRequirePreAuth -eq $true } `
    -Properties DoesNotRequirePreAuth, PasswordLastSet |
    Select-Object SamAccountName, Enabled, PasswordLastSet, DoesNotRequirePreAuth
```

| Risque | Réponse saine |
|---|---|
| Compte de service utilisateur avec SPN | Migrer vers gMSA quand possible |
| Mot de passe ancien et SPN exposé | Rotation, longueur élevée, coffre de secrets |
| `DoesNotRequirePreAuth = True` | Désactiver sauf exception documentée |
| RC4 encore utilisé | Prioriser AES, désactiver les dépendances legacy |

!!! tip "Indicateur simple"
    Un compte avec SPN, mot de passe ancien et privilège élevé doit être traité comme un risque prioritaire.
    La longueur du mot de passe compte autant que la fréquence de rotation.

!!! quote "En résumé"
    Kerberoasting et AS-REP Roasting exploitent surtout l'hygiène des comptes.
    Les SPN doivent être inventoriés, les mots de passe longs, et les exceptions Kerberos documentées.

---

## gMSA pour les services

Les `group Managed Service Accounts` réduisent le risque des comptes de service classiques.
Le mot de passe est long, géré par AD et changé automatiquement.
L'usage est limité aux machines autorisées.

```powershell title="Créer un gMSA pour un service applicatif"
# Requires a KDS root key already present in the forest
New-ADServiceAccount -Name "gmsa-webapp01" `
    -DNSHostName "gmsa-webapp01.corp.local" `
    -PrincipalsAllowedToRetrieveManagedPassword "GG-WebApp01-Servers" `
    -KerberosEncryptionType AES128,AES256
```

```powershell title="Installer et tester le gMSA sur un serveur membre"
Install-ADServiceAccount gmsa-webapp01
Test-ADServiceAccount gmsa-webapp01
```

!!! warning "Préproduction obligatoire"
    Tous les services et applications ne supportent pas un gMSA.
    Testez le démarrage, les tâches planifiées, les accès réseau et les SPN avant migration.

!!! quote "En résumé"
    Un gMSA remplace avantageusement un compte de service utilisateur quand l'application le supporte.
    Il réduit le risque Kerberoasting, mais ne supprime pas le besoin de limiter les privilèges du service.

---

## Délégation Kerberos : constrained, RBCD et unconstrained

La délégation Kerberos permet à un service d'agir pour le compte d'un utilisateur vers un autre service.
Mal maîtrisée, elle devient un chemin d'escalade.

| Type | Risque | Position recommandée |
|---|---|---|
| Unconstrained delegation | Très élevé : délégation vers n'importe quel service | À supprimer sauf exception historique temporaire |
| Constrained delegation | Risque borné aux SPN déclarés | Acceptable avec revue régulière |
| Resource-based constrained delegation (RBCD) | Contrôlée côté ressource cible | Utile, mais ACL à surveiller strictement |

```powershell title="Lister les ordinateurs avec délégation non contrainte"
Get-ADComputer -Filter { TrustedForDelegation -eq $true } `
    -Properties TrustedForDelegation |
    Select-Object Name, DistinguishedName, TrustedForDelegation
```

```powershell title="Lister les délégations contraintes classiques"
Get-ADComputer -Filter * -Properties msDS-AllowedToDelegateTo |
    Where-Object { $_."msDS-AllowedToDelegateTo" } |
    Select-Object Name, msDS-AllowedToDelegateTo
```

```powershell title="Lister les objets avec RBCD configurée"
Get-ADComputer -Filter * -Properties msDS-AllowedToActOnBehalfOfOtherIdentity |
    Where-Object { $_."msDS-AllowedToActOnBehalfOfOtherIdentity" } |
    Select-Object Name, DistinguishedName
```

!!! danger "Contrôleurs de domaine"
    Un DC ou un serveur Tier 0 ne doit pas être exposé à une délégation non contrainte.
    La délégation est un sujet d'architecture, pas un simple paramètre applicatif.

!!! quote "En résumé"
    Supprimez la délégation non contrainte, documentez la délégation contrainte et surveillez les ACL RBCD.
    Une exception de délégation doit avoir un propriétaire et une date de revue.

---

## Rotation KRBTGT

Le compte `krbtgt` signe les tickets Kerberos du domaine.
Après une compromission de secrets AD, sa rotation est indispensable pour invalider des tickets forgés.

La rotation doit se faire en deux changements de mot de passe, espacés de manière contrôlée, afin de laisser les tickets légitimes expirer entre les deux.

```powershell title="Lire la date de dernier changement KRBTGT"
Get-ADUser krbtgt -Properties PasswordLastSet |
    Select-Object SamAccountName, PasswordLastSet
```

!!! warning "Pas une clé registre"
    `MaxTicketAge` n'est pas un paramètre registre local.
    C'est une politique Kerberos du domaine, visible dans les paramètres de sécurité du domaine et cohérente avec la durée de vie des tickets.

Séquence opérationnelle :

1. Vérifier la santé AD : réplication, DCs hors ligne, sauvegardes, supervision Kerberos.
2. Changer le mot de passe `krbtgt` une première fois.
3. Attendre au moins la durée de vie maximale des tickets Kerberos, selon la politique du domaine.
4. Changer le mot de passe `krbtgt` une deuxième fois.
5. Surveiller les erreurs Kerberos et les services sensibles.

!!! danger "Ne pas improviser"
    Une double rotation KRBTGT mal planifiée peut casser l'authentification.
    Utilisez un runbook, validez la réplication et tenez compte des DCs restaurés ou hors ligne.

!!! quote "En résumé"
    KRBTGT se tourne en deux temps, avec réplication saine et fenêtre de surveillance.
    Ce n'est pas un réglage quotidien, c'est une opération de sécurité majeure.

---

## LDAP Signing et Channel Binding

LDAP signing empêche les échanges LDAP non signés.
LDAP channel binding renforce LDAPS contre certains scénarios de relais en liant l'authentification au canal TLS.

| Paramètre | Emplacement registre | Valeur de durcissement |
|---|---|---|
| Signature LDAP serveur | `HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Parameters\LDAPServerIntegrity` | `2` |
| Channel Binding serveur | `HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Parameters\LdapEnforceChannelBinding` | `2` |
| Signature LDAP client | `HKLM\SYSTEM\CurrentControlSet\Services\LDAP\LDAPClientIntegrity` | `2` |

!!! warning "Piloter avant de forcer"
    Le passage en `Require signing` ou `Always` peut casser des applications LDAP legacy.
    Activez l'audit, traitez les clients qui négocient encore en clair ou sans signature, puis forcez.

Événements utiles côté DC :

| Event ID | Usage |
|---|---|
| `2886` | Le serveur n'exige pas la signature LDAP |
| `2887` | Nombre de connexions LDAP non signées |
| `2888` | Refus de connexions non signées quand l'exigence est active |
| `2889` | Détail des clients LDAP non signés, si le diagnostic est configuré |

!!! quote "En résumé"
    LDAP signing et channel binding réduisent fortement le relais et les liaisons LDAP faibles.
    La difficulté n'est pas technique : elle consiste à identifier les clients legacy avant le basculement strict.

---

## Print Spooler sur les contrôleurs de domaine

Le service Print Spooler n'a généralement pas de raison d'être actif sur un contrôleur de domaine.
Il a déjà été impliqué dans des chaînes d'attaque et augmente inutilement la surface exposée des DCs.

```powershell title="Désactiver le spooler sur les DCs via PowerShell"
Get-ADDomainController -Filter * |
    ForEach-Object {
        Invoke-Command -ComputerName $_.HostName -ScriptBlock {
            Stop-Service Spooler -Force
            Set-Service Spooler -StartupType Disabled
        }
    }
```

!!! tip "Déploiement propre"
    Préférez une GPO liée à l'OU `Domain Controllers` pour définir le service `Print Spooler` en `Disabled`.
    Le script sert surtout à l'audit ou à une remédiation contrôlée.

!!! quote "En résumé"
    Sauf besoin exceptionnel documenté, le Print Spooler doit être désactivé sur les DCs.
    Un DC ne doit pas porter un service d'impression par habitude.

---

## Tiering model

Le modèle de tiering sépare les identités et les machines selon leur niveau de contrôle.
L'objectif est d'empêcher un privilège Tier 0 de circuler vers un poste ou serveur moins sensible.

| Tier | Contenu | Règle de base |
|---|---|---|
| Tier 0 | AD, DCs, PKI racine, Entra Connect, comptes d'admin domaine | Admin uniquement depuis postes d'administration dédiés |
| Tier 1 | Serveurs membres et applications critiques | Comptes séparés, pas de droits Tier 0 |
| Tier 2 | Postes utilisateurs | Administration locale limitée, pas de comptes Tier 0 |

Contrôles pratiques :

- interdire l'ouverture de session des admins Tier 0 sur les postes Tier 1 et Tier 2 ;
- utiliser des comptes distincts pour chaque tier ;
- dédier des postes d'administration protégés aux tâches Tier 0 ;
- journaliser les ouvertures de session privilégiées ;
- supprimer les droits locaux hérités des groupes trop larges.

!!! warning "Le tiering est un comportement"
    Créer trois groupes ne suffit pas.
    Le modèle ne fonctionne que si les administrateurs cessent réellement d'utiliser les mêmes comptes partout.

!!! quote "En résumé"
    Le tiering réduit la propagation des identifiants privilégiés.
    C'est une règle d'exploitation quotidienne, pas un schéma théorique.

---

## Synthèse et checklist

| Contrôle | Priorité | Preuve attendue |
|---|---|---|
| Revue `AdminSDHolder` | Haute | ACL documentée, aucune ACE inattendue |
| Revue `adminCount=1` | Haute | Liste courte et justifiée |
| Droits DCSync | Critique | Uniquement DCs et comptes de synchronisation approuvés |
| Comptes avec SPN | Haute | gMSA ou mot de passe long, rotation documentée |
| AS-REP Roasting | Haute | Aucune exception sans justification |
| Délégation non contrainte | Critique | Supprimée ou exception temporaire validée |
| KRBTGT | Critique après incident | Runbook double rotation testé |
| LDAP signing/channel binding | Haute | Audit puis enforcement |
| Print Spooler sur DCs | Haute | Service désactivé sauf exception |
| Tiering | Critique | Comptes et postes séparés par tier |

!!! quote "En résumé"
    Le durcissement AD commence par les droits Tier 0, les comptes de service, Kerberos et LDAP.
    Les réglages doivent être audités en continu, car une seule délégation oubliée peut rouvrir un chemin d'escalade.

!!! tip "Voir aussi"
    - :material-book-cog: [Active Directory Domain Services](../registre-pour-les-admins/05-ad-ds.md) — Registre Admins
    - :material-shield-lock: [Securite des strategies de groupe](../bible-gpo/13-securite-strategies.md) — Bible GPO
    - :material-book-open-variant: [Securite moderne](../bible-registre-windows/16-securite-moderne.md) — Bible Registre
