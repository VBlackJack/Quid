---
description: "Methodologie et principes pour concevoir une architecture GPO robuste, lisible et maintenable a l'echelle de l'entreprise."
tags:
  - gpo
  - administration
  - architecture
  - entreprise
---
# Concevoir une architecture GPO d'entreprise

!!! abstract "Ce que vous allez pouvoir faire"
    - Concevoir une structure d'OU adaptee au deploiement GPO, independante de l'organigramme
    - Etablir une convention de nommage qui parle a toute l'equipe, 6 mois plus tard
    - Decider combien de GPO creer et comment les regrouper par fonction
    - Auditer un parc existant pour identifier les GPO inutiles, non liees ou en conflit
    - Eviter les trois pieges de production qui ralentissent les ouvertures de session et bloquent le diagnostic


!!! tip "Si vous ne retenez qu'une chose"
    Une architecture GPO d'entreprise tient si le scope, le nommage et les niveaux de liaison restent lisibles avant même d'ouvrir l'éditeur.

---

## Pourquoi l'architecture GPO est un sujet de production

Dans un environnement de moins de 20 GPO, l'architecture n'a pas d'importance. On s'en sort avec des noms approximatifs et quelques liens mal places.

A partir de 50 GPO, les problemes emergent. Les ouvertures de session s'allongent. Deux politiques se contredisent sur la configuration du proxy. Personne ne sait plus quelle GPO desactive l'ecran de veille.

A 200 GPO — ce qui arrive vite dans un parc de 500 postes non structure — les temps de logon depassent 90 secondes, le debug prend des heures, et la migration vers Intune devient un projet de plusieurs mois a cause de la dette technique accumulee.

Ce chapitre donne une methodologie pour eviter ce scenario. Les principes sont applies en production sur des environnements de 200 a 5 000 postes.

:material-alert-circle-outline: **Ce chapitre suppose que vous connaissez deja le fonctionnement de base des GPO** (LSDOU, GPMC, filtrage par groupe de securite). L'accent est mis sur les decisions d'architecture, pas sur la manipulation dans la console.

!!! quote "En resume"
    Une architecture GPO non planifiee devient une dette technique. Elle se paie en temps de logon, en conflits impossibles a diagnostiquer, et en nuits blanchies avant une migration.

---

## :material-sitemap: Principe 1 — Aligner les GPO sur les OU, pas sur l'organigramme

### L'erreur la plus frequente

La majorite des architectures GPO mal concues partagent le meme defaut : elles reproduisent l'organigramme de l'entreprise.

On trouve des OU `RH`, `Finance`, `IT`, `Direction`. Les GPO sont liees a ces OU. Ca semble logique jusqu'au jour ou il faut configurer le proxy differentement pour les postes kiosk de la RH et les postes standards des memes utilisateurs.

**Le probleme fondamental :** les objets AD sont de deux natures tres differentes — les ordinateurs et les utilisateurs — et la GPO traite ces deux natures dans des contextes separes.

`Computer Configuration` s'applique au demarrage de la machine, avant toute ouverture de session. `User Configuration` s'applique au moment du logon de l'utilisateur. Ce sont deux phases de traitement differentes, avec des agents differents et des timings differents.

Une OU `RH` qui contient a la fois des comptes utilisateurs et des comptes ordinateurs force a ecrire des GPO qui melangent les deux configurations. Le resultat : des parametres qui s'appliquent aux mauvais objets, des filtres WMI complexes pour compenser, et un diagnostic impossible.

### La segmentation technique, pas fonctionnelle

La regle est simple : **les OU doivent refleter la nature technique des objets, pas leur rattachement metier**.

Un poste de travail standard a les memes besoins de configuration qu'il soit en RH ou en Finance. C'est le profil technique qui compte : est-ce un poste standard, un poste kiosk, un poste mobile ?

Structure d'OU recommandee pour un environnement de 200 a 2 000 objets :

```
domain.local
├── Postes-de-travail
│   ├── Postes-Pilote          ← toujours avoir une OU de test
│   ├── Postes-Standard
│   ├── Postes-Kiosk
│   └── Postes-Mobiles
├── Utilisateurs
│   ├── Users-Standard
│   ├── Users-Admins
│   └── Users-Service          ← comptes de service, pas des humains
├── Serveurs
│   ├── Serveurs-Infra         ← DC, DNS, DHCP
│   ├── Serveurs-Applications
│   └── Serveurs-DMZ
└── Groupes-de-securite        ← isoler les groupes des objets cibles
```

### L'OU Postes-Pilote est obligatoire

Toute architecture sans OU pilote est incomplete. Sans OU pilote, chaque deploiement est un deploiement en production.

L'OU `Postes-Pilote` doit contenir entre 5 et 10 machines representatives du parc : au moins un poste de chaque profil materiel ou logiciel present en production. Les GPO y sont liees en premier, avec au moins 48h d'observation avant le deploiement en production.

### Le cas des organisations multi-sites

Dans un environnement multi-sites, la question est : faut-il une OU par site ou une OU par profil technique ?

La reponse depend du nombre de configurations specifiques au site. Si 95 % des parametres sont communs entre les sites, la segmentation par profil technique est suffisante — le ciblage par site se fait avec des filtres WMI ou des groupes de securite.

Si des configurations reseau (proxy, DNS, lecteurs reseau) sont specifiques a chaque site, une segmentation hybride est possible :

```
Postes-de-travail
├── Postes-Standard
│   ├── Site-Paris
│   ├── Site-Lyon
│   └── Site-Bordeaux
└── Postes-Pilote
```

:material-information-outline: La segmentation par site multiplie le nombre d'OU et de GPO. Ne l'utilisez que si les differences de configuration entre sites sont reelles et stables.

!!! warning "A surveiller — Les OU trop profondes"
    Chaque niveau d'OU supplementaire augmente le temps de traitement LSDOU. Au-dela de 5 niveaux d'imbrication, les performances de traitement GPO se degradent de maniere measurable. Restez aussi plat que possible.

!!! quote "En resume"
    Structurez les OU selon la nature technique des objets (standard, kiosk, mobile, serveur), pas selon l'organigramme. Une OU Postes-Pilote est non-negociable. La segmentation par site n'est justifiee que si les differences de configuration entre sites sont significatives.

---

## :material-tag-outline: Principe 2 — Convention de nommage rigoureuse

### Pourquoi le nom d'une GPO est critique

Le nom d'une GPO est la seule information visible dans GPMC sans ouvrir la GPO. Un nom bien choisi permet de repondre en 3 secondes a la question : "Quelle GPO gere la configuration Edge sur les postes pilotes ?".

Un nom mal choisi — `GPO-test`, `NouvelleGPO`, `GPO-FINAL-v3` — force a ouvrir chaque GPO pour comprendre ce qu'elle fait.

Apres 6 mois sans convention, personne n'ose toucher a `GPO-237` parce que personne ne sait ce qu'elle fait.

### Schema de prefixes recommande

Le nom doit communiquer trois informations : le domaine fonctionnel, la cible, et le sujet.

| Prefixe | Domaine | Exemple |
|---------|---------|---------|
| `SEC-` | Securite | `SEC-Postes-Baseline` |
| `CFG-` | Configuration | `CFG-Postes-WindowsUpdate` |
| `INF-` | Infrastructure | `INF-Serveurs-Audit` |
| `USR-` | Utilisateurs | `USR-Standard-Bureau` |
| `APP-` | Applications tierces | `APP-Edge-Configuration` |
| `KSK-` | Kiosk / restrictions renforcees | `KSK-Accueil-Restrictions` |

Le pattern complet est : `{PREFIXE}-{CIBLE}-{SUJET}[-{VERSION}]`

La version facultative (`-Ring1`, `-v2`, `-Pilote`) ne sert qu'a distinguer des variantes d'une meme politique — par exemple des rings Windows Update.

### Exemples de noms corrects

```
SEC-Postes-Baseline              ← securite socle pour tous les postes
SEC-Serveurs-Baseline            ← securite socle pour tous les serveurs
CFG-Postes-WUfB-Ring1            ← Windows Update ring 1 (pilotes)
CFG-Postes-WUfB-Ring2            ← Windows Update ring 2 (production)
CFG-Postes-EnvironnementBureau   ← fond d'ecran, economiseur, barre des taches
CFG-Postes-Reseau                ← proxy, suffixes DNS, lecteurs reseau
APP-Edge-ConfigEntreprise        ← Edge : page de demarrage, extensions, SmartScreen
APP-Chrome-ConfigEntreprise      ← Chrome : idem
USR-Admins-RestrictionsRenforcees  ← limitations supplementaires pour les admins
KSK-Accueil-Restrictions         ← kiosk reception : restrictions maximales
INF-Serveurs-Audit               ← politique d'audit avancee pour les serveurs
```

### Ce qu'il faut eviter absolument

Les noms a bannir en production :

- `GPO-test`, `test`, `TEST-2` : apres 3 mois, plus personne ne sait si c'est encore utilise
- `GPO-FINAL`, `GPO-FINAL-v2`, `GPO-definitive` : il y aura toujours un v3
- `GPO-237`, `GPO-1045` : numeros sans signification
- Noms descriptifs du parametre plutot que de l'usage : `GPO-DesactiverUSB`, `GPO-FondEcranBleu`

:material-lightbulb-outline: Un nom descriptif du parametre (`GPO-DesactiverUSB`) devient faux des que la GPO est enrichie d'autres parametres. Un nom base sur l'usage (`SEC-Postes-Baseline`) reste exact indefiniment.

### Renommer les GPO existantes

PowerShell permet de renommer les GPO sans impact sur les liaisons. La GPO est identifiee par son GUID en interne — le nom est juste un label.

```powershell
# Rename a GPO without breaking any existing links
Rename-GPO -Name "GPO-test-proxy" -TargetName "CFG-Postes-Reseau"
```

```title="Resultat attendu"
DisplayName      : CFG-Postes-Reseau
DomainName       : corp.local
Owner            : CORP\Domain Admins
Id               : {a1b2c3d4-e5f6-7890-abcd-ef1234567890}
GpoStatus        : AllSettingsEnabled
Description      :
CreationTime     : 01/03/2025 14:22:10
ModificationTime : 05/04/2026 09:15:33
```

!!! quote "En resume"
    Le schema `{PREFIXE}-{CIBLE}-{SUJET}` rend l'architecture auto-documentee. Un administrateur qui arrive 6 mois apres comprend l'architecture en parcourant la liste des GPO dans GPMC. Le renommage ne casse pas les liaisons.

---

## :material-layers-outline: Principe 3 — Une GPO par fonction, pas par parametre

### L'anti-pattern GPO atomique

L'erreur inverse du nommage est de creer une GPO pour chaque parametre : `GPO-Wallpaper`, `GPO-Screensaver`, `GPO-Proxy`, `GPO-Audit`.

Ce pattern semble methodique. Il devient cauchemardesque a partir de 50 parametres.

**Impact concret sur les performances :** a chaque ouverture de session, le client interroge SYSVOL pour chaque GPO liee. Chaque GPO = une requete LDAP + un acces SYSVOL. 200 GPO atomiques sur un lien reseau lent = 90 secondes de logon.

**Impact sur le debug :** `gpresult /r` retourne 200 lignes. Retrouver quel parametre vient de quelle GPO devient une recherche binaire manuelle.

### La regle de regroupement

La regle est : **meme audience + meme cycle de vie = meme GPO**.

Si deux parametres s'appliquent aux memes objets et evoluent ensemble (deployes en meme temps, modifies ensemble, desactives ensemble), ils appartiennent a la meme GPO.

| GPO regroupee | Ce qu'elle contient |
|--------------|---------------------|
| `CFG-Postes-EnvironnementBureau` | Fond d'ecran + economiseur + barre des taches + menu Demarrer |
| `CFG-Postes-Reseau` | Proxy + suffixes DNS + lecteurs reseau mappes |
| `SEC-Postes-Baseline` | Pare-feu + autorun + politique de mot de passe + audit |
| `APP-Edge-ConfigEntreprise` | Page de demarrage + extensions + proxy + SmartScreen |

### L'exception valide : le risque de deploiement

Un parametre a risque eleve doit rester dans sa propre GPO.

Exemple : vous voulez ajouter une restriction `AppLocker` a votre `SEC-Postes-Baseline` existante. `AppLocker` peut bloquer des applications legitimes si mal configure. Si vous l'integrez dans la baseline, un rollback force a desactiver toute la baseline — ce qui retire aussi le pare-feu et la politique d'audit.

La regle pratique : **si le rollback d'un parametre necessite de desactiver d'autres parametres non lies, ce parametre doit etre dans une GPO separee**.

```
SEC-Postes-Baseline       ← stable, mature, deploye depuis 2 ans
SEC-Postes-AppLocker      ← nouveau, risque, peut etre desactive seul
```

### Combien de GPO pour 500 postes ?

Un environnement de 500 postes bien structure tourne avec 25 a 40 GPO. Au-dela de 50, commencez a auditer les doublons et les GPO atomiques.

:material-chart-bar: Regle empirique : si une GPO contient moins de 3 parametres configures et qu'elle n'a pas de raison technique d'exister seule, elle est un candidat au merge.

!!! warning "A surveiller — GPO avec Computer ET User config actives"
    Activer les deux sections (Computer Configuration et User Configuration) dans une meme GPO force le client a traiter les deux sections a chaque cycle — meme si l'objet cible n'est qu'un ordinateur. **Desactivez la section inutilisee** dans les proprietes de chaque GPO. Ca reduit le temps de traitement et les requetes SYSVOL.

    Dans GPMC : clic droit sur la GPO → Proprietes → onglet Details → Status GPO → choisir "User configuration settings disabled" ou "Computer configuration settings disabled".

!!! quote "En resume"
    Regroupez les parametres par audience et cycle de vie, pas par categorie logique. Une GPO qui contient le fond d'ecran et l'economiseur est saine. Une GPO qui contient l'economiseur seul est un signe d'architecture atomique. Desactivez toujours la section non utilisee de chaque GPO.

---

## :material-link-variant: Principe 4 — Les regles de liaison

### Ne jamais lier a la racine du domaine (sauf exceptions)

Une GPO liee a la racine du domaine s'applique a tous les objets du domaine : postes, serveurs, utilisateurs, comptes de service. L'impact d'une erreur est maximal.

Seules deux GPO peuvent justifier une liaison a la racine :

1. **Default Domain Policy** : politique de mot de passe et Kerberos. C'est la seule GPO qui peut configurer la `Password Policy` au niveau domaine.
2. **SEC-Domaine-Baseline** : un parametre de securite qui doit genuinement s'appliquer a chaque objet du domaine sans exception — typiquement la signature SMB ou le NTLMv2.

Tout le reste doit etre lie au niveau OU.

### Lier le plus bas possible dans la hierarchie

Plus une GPO est liee bas dans la hierarchie des OU, plus son scope est precis et plus son impact en cas d'erreur est limite.

Une GPO liee a `Postes-de-travail\Postes-Standard` ne touche que les postes standard. Une GPO liee a `Postes-de-travail` touche tous les sous-types de postes. Choisissez le niveau le plus specifique compatible avec votre besoin.

### Eviter les liaisons multiples

Une meme GPO liee a 5 OU differentes cree des dependances invisibles. Si vous modifiez la GPO pour repondre au besoin d'une OU, vous impactez silencieusement les 4 autres.

:material-alert-outline: Si vous vous surprenez a lier la meme GPO a plusieurs OU non contigues, c'est un signal que la GPO doit etre scindee ou que les OU doivent etre reorganisees.

### Jeu de GPO de reference pour 500 postes

| GPO | Liee a | Portee | Priorite |
|-----|--------|--------|----------|
| `SEC-Domaine-MotDePasse` | Racine domaine | Tous | 1 |
| `SEC-Postes-Baseline` | Postes-de-travail | Ordinateurs | 2 |
| `CFG-Postes-WUfB-Ring2` | Postes-Standard | Ordinateurs | 3 |
| `CFG-Postes-WUfB-Ring1` | Postes-Pilote | Ordinateurs | 3 |
| `CFG-Postes-EnvironnementBureau` | Utilisateurs | Utilisateurs | 4 |
| `CFG-Postes-Reseau` | Postes-de-travail | Ordinateurs | 5 |
| `APP-Edge-ConfigEntreprise` | Postes-de-travail | Ordinateurs | 6 |
| `USR-Admins-RestrictionsRenforcees` | Users-Admins | Utilisateurs | 1 |
| `KSK-Accueil-Restrictions` | Postes-Kiosk | Ordinateurs + Utilisateurs (loopback) | 1 |
| `SEC-Serveurs-Baseline` | Serveurs | Ordinateurs | 1 |
| `INF-Serveurs-Audit` | Serveurs-Infra | Ordinateurs | 2 |

### Le cas du Loopback pour les kiosks

Les postes kiosk et les serveurs RDS necessitent que la configuration utilisateur soit determinee par la machine, pas par le compte utilisateur. C'est le mode Loopback.

Sans Loopback, un utilisateur standard qui se connecte sur un kiosk recoit ses GPO utilisateur normales — et non les restrictions du kiosk.

Activation dans GPMC : `Configuration ordinateur → Modeles d'administration → Systeme → Strategie de groupe → Configurer le mode de traitement en mode Loopback de la strategie de groupe de l'utilisateur` → Mode **Remplacer** pour les kiosks, mode **Fusionner** pour RDS.

!!! quote "En resume"
    Liez a la racine du domaine uniquement pour la politique de mot de passe et les parametres qui s'appliquent sans exception a tous les objets. Pour tout le reste, liez au niveau OU le plus bas et le plus specifique. Evitez les liaisons multiples de la meme GPO.

---

## :material-magnify: Auditer l'existant avec PowerShell

Avant de concevoir ou de restructurer une architecture GPO, l'etat des lieux est indispensable. Ces scripts s'executent depuis n'importe quel poste avec le module `GroupPolicy` installe (RSAT).

### Inventaire complet des GPO

```powershell
# Full GPO inventory with link status and metadata
Get-GPO -All | ForEach-Object {
    $report = [xml](Get-GPOReport -Guid $_.Id -ReportType Xml)
    $links = $report.GPO.LinksTo | ForEach-Object { $_.SOMPath }

    [PSCustomObject]@{
        Name        = $_.DisplayName
        Status      = $_.GpoStatus
        Created     = $_.CreationTime.ToString("yyyy-MM-dd")
        Modified    = $_.ModificationTime.ToString("yyyy-MM-dd")
        Links       = if ($links) { ($links -join "; ") } else { "UNLINKED" }
        ComputerSec = if ($report.GPO.Computer.ExtensionData) { "Active" } else { "Empty" }
        UserSec     = if ($report.GPO.User.ExtensionData)     { "Active" } else { "Empty" }
    }
} | Sort-Object Links, Name |
    Export-Csv -Path "C:\Temp\gpo-inventory.csv" -NoTypeInformation -Encoding UTF8
```

```title="Resultat attendu"
Fichier C:\Temp\gpo-inventory.csv cree avec une ligne par GPO.
Colonnes : Name, Status, Created, Modified, Links, ComputerSec, UserSec.
Les GPO non liees apparaissent avec Links = UNLINKED.
```

### Trouver les GPO non liees

Les GPO non liees sont du bruit pur. Elles ne s'appliquent a rien mais sont quand meme chargees par certains outils de diagnostic, et elles encombrent GPMC.

```powershell
# Find all unlinked GPOs
Get-GPO -All | ForEach-Object {
    $report = [xml](Get-GPOReport -Guid $_.Id -ReportType Xml)
    if (-not $report.GPO.LinksTo) {
        [PSCustomObject]@{
            Name     = $_.DisplayName
            GUID     = $_.Id
            Created  = $_.CreationTime.ToString("yyyy-MM-dd")
            Modified = $_.ModificationTime.ToString("yyyy-MM-dd")
            Status   = "UNLINKED"
        }
    }
} | Sort-Object Modified | Format-Table -AutoSize
```

```title="Resultat attendu"
Name                          GUID                                   Created    Modified   Status
----                          ----                                   -------    --------   ------
GPO-test                      {3f4a...}                              2024-01-15 2024-01-16 UNLINKED
GPO-proxy-ancien              {7c2b...}                              2023-06-01 2023-06-01 UNLINKED
Default Domain Controllers... {6ac1...}                              2022-09-01 2022-09-01 UNLINKED
```

### Trouver les GPO avec les deux sections actives

Les GPO qui ont Computer Configuration et User Configuration toutes les deux actives alors qu'une seule section contient des parametres configurent deux cycles de traitement inutilement.

```powershell
# Find GPOs with both sections enabled but only one containing settings
Get-GPO -All | ForEach-Object {
    $report = [xml](Get-GPOReport -Guid $_.Id -ReportType Xml)
    $hasComputer = [bool]$report.GPO.Computer.ExtensionData
    $hasUser     = [bool]$report.GPO.User.ExtensionData
    $status      = $_.GpoStatus

    # Flag GPOs where status doesn't match actual content
    if ($hasComputer -and -not $hasUser -and $status -ne "UserSettingsDisabled") {
        [PSCustomObject]@{
            Name            = $_.DisplayName
            Recommendation  = "Disable User Configuration (empty)"
            CurrentStatus   = $status
        }
    } elseif ($hasUser -and -not $hasComputer -and $status -ne "ComputerSettingsDisabled") {
        [PSCustomObject]@{
            Name            = $_.DisplayName
            Recommendation  = "Disable Computer Configuration (empty)"
            CurrentStatus   = $status
        }
    }
} | Format-Table -AutoSize
```

```title="Resultat attendu"
Name                          Recommendation                          CurrentStatus
----                          --------------                          -------------
CFG-Postes-Reseau             Disable User Configuration (empty)      AllSettingsEnabled
APP-Edge-ConfigEntreprise     Disable User Configuration (empty)      AllSettingsEnabled
USR-Standard-Bureau           Disable Computer Configuration (empty)  AllSettingsEnabled
```

### Detecter les liaisons multiples

```powershell
# Find GPOs linked to more than 2 OUs (potential architecture smell)
Get-GPO -All | ForEach-Object {
    $report = [xml](Get-GPOReport -Guid $_.Id -ReportType Xml)
    $links  = @($report.GPO.LinksTo | ForEach-Object { $_.SOMPath })

    if ($links.Count -gt 2) {
        [PSCustomObject]@{
            Name      = $_.DisplayName
            LinkCount = $links.Count
            Links     = $links -join " | "
        }
    }
} | Sort-Object LinkCount -Descending | Format-Table -AutoSize
```

```title="Resultat attendu"
Name                    LinkCount Links
----                    --------- -----
CFG-Postes-Reseau-OLD   5         dc=corp,dc=local | OU=RH | OU=Finance | OU=IT | OU=Serveurs
GPO-wallpaper-v2        3         OU=Postes-Standard | OU=Postes-Pilote | OU=Postes-Mobiles
```

### Exporter le rapport vers Excel

```powershell
# Export full audit to Excel-compatible CSV with semicolon separator
Get-GPO -All | ForEach-Object {
    $report  = [xml](Get-GPOReport -Guid $_.Id -ReportType Xml)
    $links   = @($report.GPO.LinksTo | ForEach-Object { $_.SOMPath })
    $enabled = @($report.GPO.LinksTo | Where-Object { $_.Enabled -eq "true" } |
                  ForEach-Object { $_.SOMPath })

    [PSCustomObject]@{
        Nom             = $_.DisplayName
        GUID            = $_.Id
        Statut          = $_.GpoStatus
        Creation        = $_.CreationTime.ToString("yyyy-MM-dd")
        DerniereModif   = $_.ModificationTime.ToString("yyyy-MM-dd")
        NbLiaisons      = $links.Count
        LiaisonsActives = $enabled.Count
        Liaisons        = $links -join " | "
        SectionOrdi     = if ($report.GPO.Computer.ExtensionData) { "Active" } else { "Vide" }
        SectionUser     = if ($report.GPO.User.ExtensionData)     { "Active" } else { "Vide" }
    }
} | Sort-Object NbLiaisons, Nom |
    Export-Csv -Path "C:\Temp\gpo-audit-complet.csv" -Delimiter ";" -NoTypeInformation -Encoding UTF8

Write-Host "Audit exporte : C:\Temp\gpo-audit-complet.csv" -ForegroundColor Green
```

```title="Resultat attendu"
Audit exporte : C:\Temp\gpo-audit-complet.csv
```

!!! quote "En resume"
    L'audit avant restructuration est obligatoire. Les scripts ci-dessus identifient les GPO non liees (a supprimer), les sections inutiles a desactiver, et les liaisons multiples a consolider. Exportez en CSV et triez par colonne NbLiaisons pour prioriser le travail.

---

## :material-shield-alert-outline: Pièges en production

Cette section documente les erreurs les plus courantes observees sur des environnements reels. Chaque piege a cause au moins un incident de production documentable.

---
### Piege 1 — Enforced active par defaut

!!! danger "Production — Ne jamais utiliser Enforced par defaut"
    **Enforced** (anciennement "No Override") empeche les OU enfants de bloquer l'heritage d'une GPO. C'est un outil de dernier recours, pas une option de confort.

    **Impact sur le diagnostic :** quand Enforced est active, `gpresult /r` ne le signale pas clairement. Un administrateur qui debug depuis l'OU enfant voit une politique s'appliquer sans comprendre d'ou elle vient. Le seul moyen de le detecter est de regarder l'onglet "Linked Group Policy Objects" de chaque OU parente dans GPMC.

    **Regle :** utilisez Enforced uniquement quand vous avez une OU enfant avec Block Inheritance et qu'une politique de securite critique doit quand meme s'y appliquer. Documentez chaque usage dans la description de la GPO.

---
### Piege 2 — Block Inheritance sans documentation

!!! warning "A surveiller — Block Inheritance casse le diagnostic"
    Block Inheritance active sur une OU empeche toutes les GPO des OU parentes de s'y appliquer (sauf celles avec Enforced). C'est parfois legitime pour une OU DMZ ou une OU de quarantaine.

    **Le probleme :** un administrateur qui debug un probleme sur une machine dans cette OU n'a pas de moyen evident de savoir que Block Inheritance est active. `gpresult /r` ne liste que les GPO qui s'appliquent — pas celles qui sont bloquees.

    **Diagnostic :** depuis GPMC, l'icone de l'OU affiche un point d'exclamation bleu quand Block Inheritance est active. Ce n'est pas visible depuis la machine elle-meme.

    **Regle :** chaque OU avec Block Inheritance doit avoir une description explicite dans ses proprietes AD. Exemple : `"Block Inheritance active — DMZ — justification : isolation reseau, voir ticket INC-4521"`.

---
### Piege 3 — CREATOR OWNER en production

!!! warning "A surveiller — CREATOR OWNER conserve Full Control"
    Par defaut dans Active Directory, le compte qui cree une GPO obtient une ACE `Full Control` via `CREATOR OWNER`. En production, cela signifie que n'importe quel membre du groupe `Group Policy Creator Owners` qui cree une GPO en garde le controle permanent — independamment des droits delegues.

    **Consequence :** un administrateur junior peut creer une GPO, la delever (remettre) a un administrateur senior, et conserver Full Control pour la modifier ulterieurement. C'est une faille de gouvernance.

    **Correction :** en production, retirez l'ACE `CREATOR OWNER` et remplacez par une ACE explicite `Domain Admins - Full Control`.

```powershell
# Find all GPOs where CREATOR OWNER still has permissions
Get-GPO -All | ForEach-Object {
    $gpo   = $_
    $perms = Get-GPPermissions -Guid $gpo.Id -All -ErrorAction SilentlyContinue
    $co    = $perms | Where-Object { $_.Trustee.Name -eq "CREATOR OWNER" }
    if ($co) {
        [PSCustomObject]@{
            GPO         = $gpo.DisplayName
            GUID        = $gpo.Id
            Permission  = $co.Permission
            Issue       = "CREATOR OWNER has explicit permissions"
        }
    }
} | Format-Table -AutoSize
```

```title="Resultat attendu"
GPO                           GUID       Permission         Issue
---                           ----       ----------         -----
CFG-Postes-Reseau             {3f4a...}  GpoEditDeleteModifySecurity  CREATOR OWNER has explicit permissions
APP-Edge-ConfigEntreprise     {7c2b...}  GpoEditDeleteModifySecurity  CREATOR OWNER has explicit permissions
```

```powershell
# Remove CREATOR OWNER from a specific GPO
# WARNING: run on a test GPO first — this modifies ACLs directly
$gpoName = "CFG-Postes-Reseau"
$gpo = Get-GPO -Name $gpoName
$path = "AD:\CN={$($gpo.Id)},CN=Policies,CN=System,DC=corp,DC=local"
$acl  = Get-Acl -Path $path

# Remove all ACEs for CREATOR OWNER
$creatorOwner = New-Object System.Security.Principal.NTAccount("CREATOR OWNER")
$acl.PurgeAccessRules($creatorOwner)
Set-Acl -Path $path -AclObject $acl

Write-Host "CREATOR OWNER removed from: $gpoName" -ForegroundColor Yellow
```

```title="Resultat attendu"
CREATOR OWNER removed from: CFG-Postes-Reseau
```

---
### Piege 4 — GPO desactivee encore liee

!!! warning "A surveiller — GPO desactivee = pas supprimee"
    Desactiver une GPO (`GpoStatus = AllSettingsDisabled`) ne supprime pas la liaison. La GPO reste visible dans GPMC et est toujours listee dans les resultats `gpresult /r` — avec le statut "disabled". Ca cree de la confusion.

    Apres validation qu'une GPO desactivee n'est plus necessaire, supprimez-la completement. GPMC propose de supprimer la GPO ET ses liaisons en une seule operation.

---
### Piege 5 — L'ordre des liens et la priorite

!!! warning "A surveiller — La priorite des liens n'est pas intuitive"
    Dans GPMC, pour une OU donnee, les GPO sont listees avec un numero d'ordre. La GPO avec l'**ordre le plus bas** (1) a la **priorite la plus haute** et ses parametres ecrasent ceux des GPO avec un ordre plus eleve.

    C'est contre-intuitif : "ordre 1" signifie "applique en dernier donc gagne". De nombreux administrateurs ont l'habitude inverse.

    Regle mnemotechnique : **ordre 1 = dernier a parler = il a raison**.

!!! quote "En resume"
    Les quatre pieges critiques sont : Enforced active sans raison, Block Inheritance non documente, CREATOR OWNER non retire, et GPO desactivees non supprimees. Auditez ces quatre points des que vous reprenez un environnement existant.

---

## :material-book-open-outline: Références croisées

| Sujet | Reference |
|-------|-----------|
| Delegation et roles GPO | Ch. 02 — Gouvernance et delegation |
| Automatisation avec PowerShell | Ch. 03 — PowerShell GroupPolicy |
| Sauvegarde et migration | Ch. 05 — Backup et migration |
| LSDOU et ordre d'application | La Bible GPO — Ch. 08 — Heritage LSDOU |
| Loopback pour kiosks et RDS | La Bible GPO — Ch. 10 — Loopback |
| Introduction debutant GPO | Les GPO pour les Nuls — Ch. 06 — Heritage |
| Windows Update en entreprise | Ch. 10 — Windows Update (WUfB) |
| Securite endpoint | Ch. 14 — Securite endpoint |

!!! quote "En résumé"
    - À relire : Delegation et roles GPO → Ch. 02 — Gouvernance et delegation.
    - À relire : Automatisation avec PowerShell → Ch. 03 — PowerShell GroupPolicy.
    - À relire : Sauvegarde et migration → Ch. 05 — Backup et migration.
    - À relire : LSDOU et ordre d'application → La Bible GPO — Ch. 08 — Heritage LSDOU.
    - Ces renvois prolongent le chapitre avec des mécanismes complémentaires ou des cas d’usage voisins.
