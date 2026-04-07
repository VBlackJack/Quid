---
description: Zero Trust et GPO — GPO comme brique ZT, Credential Guard, WDAC, ASR, BitLocker, LAPS, roadmap modernisation
tags: [gpo-admins, zero-trust, credential-guard, wdac, asr, roadmap, intune, modernisation]
---
# Zero Trust et GPO : le futur de la configuration

!!! abstract "Ce que couvre ce chapitre"
    - Comprendre pourquoi Zero Trust ne remplace pas les GPO mais redéfinit leur rôle dans une stratégie de sécurité globale
    - Identifier les contrôles GPO qui contribuent directement aux trois principes ZT — Verify Explicitly, Use Least Privilege, Assume Breach
    - Configurer Credential Guard, WDAC, les règles ASR, BitLocker et LAPS dans une logique Zero Trust, avec les event IDs à surveiller
    - Construire une architecture hybride GPO + Intune qui atteint la vérification d'état de conformité au niveau du poste
    - Lire la roadmap de modernisation : ce que les GPO feront toujours mieux que MDM, et le chemin vers Settings Catalog

!!! tip "Si vous ne retenez qu'une chose"
    Zero Trust n'est pas une technologie — c'est un **principe de vérification continue**. Les GPO sont un outil de configuration, pas de vérification. Elles restent essentielles pour durcir l'OS. Mais atteindre le Zero Trust complet requiert d'ajouter une couche de vérification d'identité et d'état (Intune + Conditional Access) **au-dessus** des GPO, pas à leur place.

---

## :material-shield-half-full: Zero Trust et GPO : deux logiques différentes

### Le modèle château-fort des GPO

Les GPO ont été conçues dans un modèle périmétrique.

La machine est jointe au domaine — on lui fait confiance. Elle reçoit des politiques au démarrage et à l'ouverture de session. Une fois dans le périmètre, la confiance est implicite.

C'est la logique du **château-fort** : le pont-levis (l'authentification AD) filtre les entrées. Tout ce qui est à l'intérieur est considéré comme sûr.

### Zero Trust remet en question cette logique

Le modèle Zero Trust part d'un postulat inverse : **ne jamais faire confiance, toujours vérifier**.

Peu importe si la machine est sur le réseau interne. Peu importe si elle est jointe au domaine. L'accès à une ressource doit être conditionné à une vérification explicite : qui est l'utilisateur, dans quel état est son poste, depuis quel contexte tente-t-il d'accéder.

Les trois principes fondateurs :

| Principe ZT | Signification concrète |
|---|---|
| **Verify Explicitly** | Authentifier et autoriser toujours, en tenant compte de tous les signaux disponibles |
| **Use Least Privilege** | Limiter l'accès utilisateur au strict nécessaire, avec JIT et JEA quand possible |
| **Assume Breach** | Concevoir les contrôles en partant du principe que l'attaquant est déjà à l'intérieur |

### GPO restent pertinentes — mais leur rôle change

!!! info "La distinction fondamentale"
    Zero Trust est une stratégie de **vérification**. La gestion de configuration OS est un **prérequis** de cette vérification. Durcir un poste via GPO n'est pas contraire au Zero Trust — c'est une condition nécessaire mais insuffisante.

Les GPO continuent d'avoir un rôle central pour :

- **La configuration de l'OS** : Credential Guard, BitLocker, ASR, restrictions de démarrage
- **Le durcissement AD côté DC** : Kerberos policy, NTLM restrictions, audit de sécurité
- **Les politiques core AD** : ces politiques ne peuvent pas être déployées par MDM

Ce qui manque aux GPO seules : elles ne vérifient pas dynamiquement l'état du poste **au moment d'une demande d'accès**. Elles configurent, mais ne conditionnent pas.

!!! quote "En résumé"
    - Les GPO opèrent dans une logique périmétrique — la confiance est accordée à la jonction de domaine
    - Zero Trust exige une vérification continue, indépendante de l'appartenance réseau
    - Les GPO restent le meilleur outil de configuration OS et de durcissement AD
    - Le Zero Trust complet requiert une couche de vérification supplémentaire (Intune + Conditional Access)

---

## :material-lock-check: Contributions des GPO au Zero Trust

### La carte de correspondance GPO → principe ZT

Chaque contrôle GPO peut être mappé sur l'un des trois principes Zero Trust. Le tableau ci-dessous est la colonne vertébrale de ce chapitre.

| Principe ZT | Contrôle GPO | Ce qu'il prévient |
|---|---|---|
| **Assume Breach** | Credential Guard | Vol de credentials NTLM/Kerberos par dump LSASS |
| **Assume Breach** | WDAC (Windows Defender Application Control) | Exécution de code non autorisé, binaires malveillants |
| **Assume Breach** | ASR Rules | Vecteurs d'exploitation connus (macros, scripts, LSASS) |
| **Verify Explicitly** | BitLocker + TPM | Accès aux données sur machine physiquement compromise |
| **Use Least Privilege** | LAPS v2 | Mouvement latéral via compte administrateur local partagé |
| **Verify Explicitly** | Windows Hello for Business | Authentification forte sans mot de passe |
| **Assume Breach** | Audit et Event Forwarding | Détection post-incident, visibilité sur les indicateurs de compromission |

Les sections suivantes détaillent chaque contrôle avec son chemin GPO, sa valeur de registre et l'event ID à surveiller.


!!! quote "En résumé"
    - Chaque contrôle GPO peut être mappé sur l'un des trois principes Zero Trust.
    - Le tableau ci-dessous est la colonne vertébrale de ce chapitre.
    - Les sections suivantes détaillent chaque contrôle avec son chemin GPO, sa valeur de registre et l'event ID à surveiller.
    - Assume Breach : Vol de credentials NTLM/Kerberos par dump LSASS.
    - Assume Breach : Exécution de code non autorisé, binaires malveillants.
---

## :material-shield-key: Credential Guard — principe ZT : Assume Breach

### Ce que Credential Guard protège

Credential Guard isole les dérivés de credentials (hachages NTLM, tickets Kerberos) dans un **conteneur VBS (Virtualization-Based Security)** inaccessible au système d'exploitation principal.

Sans Credential Guard, un attaquant ayant obtenu les droits SYSTEM peut exécuter Mimikatz et extraire les credentials de tous les utilisateurs connectés depuis `lsass.exe`. C'est le vecteur de mouvement latéral le plus courant dans les compromissions AD.

Avec Credential Guard actif, `lsass.exe` ne contient plus les dérivés — ils sont dans le conteneur VBS. L'extraction échoue.

### Prérequis matériels et OS

| Prérequis | Valeur requise |
|---|---|
| UEFI Secure Boot | Activé |
| TPM 2.0 | Présent et activé |
| Virtualisation CPU (Intel VT-x / AMD-V) | Activée dans le BIOS |
| SLAT (Second Level Address Translation) | Requis (Intel EPT / AMD RVI) |
| OS | Windows 10 Enterprise/Education 1607+ ou Windows 11 |
| Intégrité du code VBS | Activée (HVCI recommandé) |

### :material-folder-cog: Chemin GPO — Credential Guard

```
Computer Configuration → Policies → Administrative Templates
  → System → Device Guard
    → Turn On Virtualization Based Security
```

Paramètres à configurer dans cette GPO :

| Paramètre | Valeur recommandée |
|---|---|
| Select Platform Security Level | Secure Boot and DMA Protection |
| Virtualization Based Protection of Code Integrity | Enabled with UEFI lock |
| Credential Guard Configuration | **Enabled with UEFI lock** |
| Secure Launch Configuration | Enabled |

!!! warning "UEFI lock — irréversible sans accès physique"
    Le mode **"Enabled with UEFI lock"** écrit la configuration dans les variables UEFI du firmware. Pour désactiver Credential Guard ensuite, il faut accès physique à la machine et passage dans le BIOS. En production, utilisez d'abord le mode "Enabled without lock" sur un groupe pilote, puis basculez en UEFI lock après validation.

### :material-powershell: Vérification post-déploiement

```powershell title="Verify-CredentialGuard.ps1"
# Verify Credential Guard activation status on remote machines
param(
    [string[]]$Computers = @($env:COMPUTERNAME)
)

foreach ($computer in $Computers) {
    $result = Invoke-Command -ComputerName $computer -ScriptBlock {
        $vbs  = Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard
        $cgRunning = $vbs.SecurityServicesRunning -contains 1   # 1 = Credential Guard
        $cgConfig  = $vbs.SecurityServicesConfigured -contains 1

        [PSCustomObject]@{
            Computer             = $env:COMPUTERNAME
            VBSEnabled           = $vbs.VirtualizationBasedSecurityStatus -eq 2  # 2 = Running
            CredGuardConfigured  = $cgConfig
            CredGuardRunning     = $cgRunning
            SecureBootState      = $vbs.RequiredSecurityProperties -contains 1
        }
    } -ErrorAction SilentlyContinue

    $result
}
```

```title="Résultat attendu"
Computer    VBSEnabled  CredGuardConfigured  CredGuardRunning  SecureBootState
--------    ----------  -------------------  ----------------  ---------------
WKS-001     True        True                 True              True
WKS-002     True        True                 False             True    # config OK mais pas encore actif — reboot requis
WKS-003     False       False                False             False   # matériel non compatible
```

### Event IDs à surveiller

| Event ID | Source | Signification |
|---|---|---|
| **3065** | Microsoft-Windows-Kernel-Boot | VBS activé au démarrage |
| **3066** | Microsoft-Windows-Kernel-Boot | VBS — service de sécurité démarré |
| **3033** | Microsoft-Windows-CodeIntegrity | Pilote refusé par HVCI |
| **3063** | Microsoft-Windows-CodeIntegrity | Tentative de chargement de pilote non signé |

!!! quote "En résumé"
    - Credential Guard isole les dérivés NTLM/Kerberos dans un conteneur VBS — Mimikatz ne peut plus les extraire
    - GPO : `System → Device Guard → Turn On Virtualization Based Security` — Credential Guard : **Enabled with UEFI lock**
    - Vérifiez avec `Win32_DeviceGuard` : `SecurityServicesRunning` doit contenir 1
    - Event IDs : 3065/3066 pour VBS, 3033/3063 pour HVCI

---

## :material-application-cog: WDAC — principe ZT : Assume Breach

### Application Control comme brique Zero Trust

WDAC (Windows Defender Application Control) est le successeur d'AppLocker pour le contrôle d'exécution.

Son principe Zero Trust : **ne jamais exécuter ce qui n'est pas explicitement autorisé**. C'est l'implémentation technique de "Assume Breach" — même si un attaquant obtient un foothold, il ne peut pas exécuter ses outils si ceux-ci ne sont pas dans la liste blanche.

WDAC opère en mode kernel via le service CI (Code Integrity). Il est plus robuste qu'AppLocker car il ne peut pas être contourné par un processus en espace utilisateur.

### Architecture d'une politique WDAC

Une politique WDAC définit :

- **Les fichiers autorisés** : par signature, par chemin, par hash, par attribut de fichier
- **Les éditeurs de confiance** : Microsoft, éditeurs tiers signés
- **Les modes d'opération** : Audit (journalise), Enforce (bloque)

La politique est compilée en fichier binaire (`.cip`) et déployée via GPO ou MDM.

### :material-folder-cog: Chemin GPO — déploiement WDAC

```
Computer Configuration → Policies → Administrative Templates
  → System → Device Guard
    → Deploy Windows Defender Application Control
```

Paramètre : **Deploy Windows Defender Application Control** → chemin UNC vers le fichier `.cip`.

```
\\domain.local\NETLOGON\WDAC\{PolicyId}.cip
```

!!! info "Créer la politique de base"
    Microsoft fournit des politiques de base dans `C:\Windows\schemas\CodeIntegrity\ExamplePolicies\`. La politique `DefaultWindows_Audit.xml` est le point de départ recommandé — elle autorise tout ce qui est signé Microsoft. Compilez-la avec `ConvertFrom-CIPolicy`, déployez en mode Audit, analysez les événements 3076 pendant deux semaines, puis passez en Enforce.

### :material-powershell: Générer et déployer une politique WDAC de base

```powershell title="Deploy-WDACBaselinePolicy.ps1"
# Generate a WDAC policy based on Microsoft's DefaultWindows baseline (Audit mode first)
# Requires: Windows 10 1903+ or Windows 11 — run on a reference machine

param(
    [string]$PolicyDir      = "C:\WDAC\Policies",
    [string]$SharePath      = "\\domain.local\NETLOGON\WDAC",
    [switch]$EnforceMode    = $false  # Start in Audit mode — set to $true only after audit validation
)

if (-not (Test-Path $PolicyDir)) { New-Item -ItemType Directory -Path $PolicyDir | Out-Null }

$baselineXml  = "C:\Windows\schemas\CodeIntegrity\ExamplePolicies\DefaultWindows_Audit.xml"
$workingXml   = "$PolicyDir\WDAC-Baseline.xml"
$compiledCip  = "$PolicyDir\WDAC-Baseline.cip"

# Copy baseline policy
Copy-Item $baselineXml $workingXml -Force

if ($EnforceMode) {
    # Switch to Enforce mode — use only after audit validation
    Set-RuleOption -FilePath $workingXml -Option 3 -Delete  # Remove Audit Mode option
    Write-Host "Policy set to ENFORCE mode."
} else {
    Set-RuleOption -FilePath $workingXml -Option 3           # Ensure Audit Mode is set
    Write-Host "Policy set to AUDIT mode."
}

# Retrieve Policy ID from XML
$policyId = ([xml](Get-Content $workingXml)).SiPolicy.PolicyID -replace '[{}]'

# Compile policy
ConvertFrom-CIPolicy -XmlFilePath $workingXml -BinaryFilePath $compiledCip

# Deploy to NETLOGON share
if (-not (Test-Path $SharePath)) { New-Item -ItemType Directory -Path $SharePath | Out-Null }
Copy-Item $compiledCip "$SharePath\$policyId.cip" -Force

Write-Host "WDAC policy deployed: $SharePath\$policyId.cip"
Write-Host "Now configure GPO: Computer Config > Administrative Templates > System > Device Guard"
Write-Host "  -> 'Deploy Windows Defender Application Control' = $SharePath\$policyId.cip"
```

### Event IDs à surveiller

| Event ID | Source | Mode | Signification |
|---|---|---|---|
| **3076** | Microsoft-Windows-CodeIntegrity/Operational | Audit | Fichier qui aurait été bloqué en Enforce |
| **3077** | Microsoft-Windows-CodeIntegrity/Operational | Enforce | Fichier bloqué par la politique |
| **3089** | Microsoft-Windows-CodeIntegrity/Operational | Audit/Enforce | Fichier autorisé par la politique |
| **3099** | Microsoft-Windows-CodeIntegrity/Operational | — | Politique WDAC chargée au démarrage |

!!! quote "En résumé"
    - WDAC bloque l'exécution de tout binaire non autorisé — implémentation technique de "Assume Breach"
    - GPO : `System → Device Guard → Deploy Windows Defender Application Control` → chemin UNC vers `.cip`
    - Toujours démarrer en mode Audit (event 3076) — analyser deux semaines avant Enforce
    - Microsoft fournit des politiques de base dans `C:\Windows\schemas\CodeIntegrity\ExamplePolicies\`

---

## :material-wall: ASR Rules — principe ZT : Assume Breach

### ASR comme filet de sécurité comportemental

Les règles ASR (Attack Surface Reduction) sont détaillées dans le chapitre 14. Dans un contexte Zero Trust, leur rôle est précis : **bloquer les vecteurs d'exploitation les plus courants** avant qu'un code malveillant puisse s'exécuter.

ASR opère sur des comportements — une macro Office qui tente de lancer `cmd.exe`, un script PowerShell obfusqué, une tentative de dump de LSASS. Ces comportements sont des indicateurs de compromission en cours.

### Les règles ASR à priorité ZT

Dans un projet Zero Trust, certaines règles ASR ont une priorité immédiate :

| GUID | Règle | Principe ZT | Event Block / Audit |
|---|---|---|---|
| `9E6C4E1F-7D60-472F-BA1A-A39EF669E4B0` | Block credential stealing from LSASS | Assume Breach | 1121 / 1122 |
| `D1E49AAC-8F56-4280-B9BA-993A6D77406C` | Block process creations from PSExec and WMI | Assume Breach | 1121 / 1122 |
| `C1DB55AB-C21A-4637-BB3F-A12568109D35` | Advanced protection against ransomware | Assume Breach | 1121 / 1122 |
| `56A863A9-875E-4185-98A7-B882C64B5CE5` | Block abuse of exploited vulnerable signed drivers | Assume Breach | 1121 / 1122 |
| `D4F940AB-401B-4EFC-AADC-AD5F3C50688A` | Block all Office apps from creating child processes | Assume Breach | 1121 / 1122 |

!!! info "Voir le chapitre 14 pour le déploiement complet"
    Les 16 règles ASR, leurs GUIDs, et la méthodologie de déploiement progressif (Audit puis Block) sont documentés dans [le chapitre 14 — Sécurité endpoint via GPO](14-securite-endpoint.md). Ce chapitre se concentre sur leur positionnement dans une stratégie Zero Trust.

!!! quote "En résumé"
    - ASR bloque les comportements d'exploitation les plus courants — indépendamment des signatures antivirus
    - Les règles LSASS (9E6C...) et PSExec (D1E4...) sont les priorités ZT pour prévenir le mouvement latéral
    - Déploiement : toujours Audit (mode 2) en premier, Block (mode 1) après validation — voir chapitre 14
    - Event 1121 = bloc effectif, event 1122 = détection en audit

---

## :material-harddisk: BitLocker — principe ZT : Verify Explicitly

### BitLocker et la confiance au niveau du hardware

Dans un modèle Zero Trust, la confiance ne peut pas reposer uniquement sur l'identité réseau. Si une machine est volée ou son disque extrait, les données doivent rester inaccessibles.

BitLocker avec TPM implémente le principe "Verify Explicitly" au niveau matériel : le déchiffrement du volume n'est possible que si la chaîne de démarrage est intacte (UEFI, Secure Boot, bootloader, noyau).

Un attaquant qui extrait le disque et le monte sur une autre machine ne peut pas lire les données — le TPM de la machine source est absent.

### Le lien BitLocker → Conditional Access

BitLocker est également un signal de conformité exploitable par Intune et Conditional Access.

Une politique Conditional Access peut conditionner l'accès à une ressource cloud (Exchange Online, SharePoint) à l'état de chiffrement du poste. Si BitLocker est désactivé, l'accès est bloqué — ou redirigé vers un flux de remédiation.

Ce signal ne remonte que si la machine est co-managée par Intune. La GPO BitLocker chiffre le poste. Intune vérifie l'état et l'expose à Conditional Access.

!!! info "Déploiement BitLocker complet"
    Le déploiement complet de BitLocker via GPO — paramètres, script de remédiation au démarrage, vérification de l'escrow AD — est documenté dans [le chapitre 17 — BitLocker et LAPS via GPO](17-bitlocker-laps.md).

### Event IDs BitLocker à surveiller en contexte ZT

| Event ID | Source | Signification ZT |
|---|---|---|
| **24594** | Microsoft-Windows-BitLocker-API/Management | Volume chiffré avec succès |
| **24621** | Microsoft-Windows-BitLocker-API/Management | Clé de récupération sauvegardée dans AD |
| **24620** | Microsoft-Windows-BitLocker-API/Management | Échec de sauvegarde de la clé — à investiguer immédiatement |
| **853** | Microsoft-Windows-BitLocker-Driver | Tentative d'accès sans clé valide — tentative d'intrusion physique |

!!! quote "En résumé"
    - BitLocker + TPM protège les données sur machine physiquement compromise — "Verify Explicitly" au niveau hardware
    - La chaîne TPM garantit que le déchiffrement ne fonctionne que sur la machine d'origine avec la chaîne de démarrage intacte
    - Signal de conformité : Intune peut exposer l'état BitLocker à Conditional Access pour bloquer les postes non chiffrés
    - Event 24620 = clé non sauvegardée dans AD — priorité opérationnelle immédiate

---

## :material-account-lock: LAPS — principe ZT : Use Least Privilege

### Le mouvement latéral par compte administrateur partagé

Dans la majorité des compromissions AD documentées par Microsoft DART, l'un des vecteurs initiaux est le **compte administrateur local partagé**.

Scénario classique : le compte `.\Administrator` a le même mot de passe sur 500 postes. L'attaquant compromet un poste, extrait le hash, et peut s'authentifier sur tous les autres avec `Pass-the-Hash`. Le mouvement latéral est trivial.

LAPS résout ce problème en générant un **mot de passe unique par machine**, stocké dans AD et accessible uniquement aux opérateurs autorisés.

### LAPS v2 — Use Least Privilege natif

Windows LAPS (LAPS v2, intégré depuis Windows 11 22H2 et KB5025175) ajoute par rapport à LAPS v1 :

- Chiffrement du mot de passe dans AD (attribut `msLAPS-EncryptedPassword`)
- Historique des mots de passe (jusqu'à 12 entrées)
- Rotation automatique post-utilisation
- Intégration native avec Intune pour les postes AAD-joined

Les paramètres GPO LAPS v2 sont dans :

```
Computer Configuration → Policies → Administrative Templates
  → System → LAPS
```

| Paramètre LAPS v2 | Recommandation ZT |
|---|---|
| Password Settings → Password Complexity | Large letters + small letters + numbers + specials |
| Password Settings → Password Length | 20+ caractères |
| Password Settings → Password Age (Days) | 30 jours maximum |
| Do not allow password expiration time longer than required by policy | **Activé** |
| Post-authentication actions | Reset password AND log off the managed account (mode 3) |
| Enable password encryption | **Activé** (LAPS v2 uniquement) |

### :material-powershell: Vérification de la couverture LAPS sur le parc

```powershell title="Get-LAPSCoverage.ps1"
# Audit LAPS v2 deployment coverage across the domain
# Reports machines without a LAPS password (either not deployed or attribute empty)
param(
    [string]$SearchBase = (Get-ADDomain).DistinguishedName,
    [int]$MaxPasswordAgeDays = 30
)

$cutoff = (Get-Date).AddDays(-$MaxPasswordAgeDays)

$computers = Get-ADComputer -Filter { OperatingSystem -like "*Windows*" } `
    -SearchBase $SearchBase `
    -Properties msLAPS-PasswordExpirationTime, msLAPS-EncryptedPassword, `
                ms-Mcs-AdmPwdExpirationTime, ms-Mcs-AdmPwd

foreach ($computer in $computers) {
    # Detect LAPS version in use
    $hasLAPSv2  = $null -ne $computer.'msLAPS-EncryptedPassword'
    $hasLAPSv1  = $null -ne $computer.'ms-Mcs-AdmPwd' -and $computer.'ms-Mcs-AdmPwd' -ne ''
    $lapsVersion = if ($hasLAPSv2) { "v2" } elseif ($hasLAPSv1) { "v1" } else { "None" }

    # Check expiration
    $expiration  = if ($hasLAPSv2) {
        $computer.'msLAPS-PasswordExpirationTime'
    } elseif ($hasLAPSv1) {
        $computer.'ms-Mcs-AdmPwdExpirationTime'
    } else { $null }

    $isExpired = $expiration -and ([datetime]::FromFileTime($expiration) -lt $cutoff)

    [PSCustomObject]@{
        Computer    = $computer.Name
        LAPSVersion = $lapsVersion
        HasPassword = ($hasLAPSv2 -or $hasLAPSv1)
        Expired     = $isExpired
        ExpiresAt   = if ($expiration) { [datetime]::FromFileTime($expiration) } else { $null }
    }
} | Sort-Object LAPSVersion, Computer
```

```title="Résultat attendu"
Computer   LAPSVersion  HasPassword  Expired  ExpiresAt
--------   -----------  -----------  -------  ---------
WKS-001    v2           True         False    2026-05-01 07:00:00
WKS-002    v2           True         True     2026-03-01 07:00:00   # rotation en retard
WKS-003    v1           True         False    2026-04-20 08:00:00   # migrer vers v2
WKS-004    None         False        False                          # LAPS non déployé
```

!!! quote "En résumé"
    - LAPS élimine le vecteur Pass-the-Hash en donnant un mot de passe unique par machine — "Use Least Privilege" appliqué au compte local
    - LAPS v2 ajoute le chiffrement AD, l'historique et la rotation post-utilisation — priorité sur v1
    - Paramètre critique : `Post-authentication actions = Reset password AND log off` — rotation immédiate après toute utilisation
    - Vérifiez la couverture : toute machine `LAPSVersion = None` est un risque de mouvement latéral actif

---

## :material-check-decagram: Verify Explicitly — le maillon manquant des GPO seules

### Ce que les GPO ne font pas

Les GPO configurent le poste. Elles ne vérifient pas l'identité de l'utilisateur au moment d'une demande d'accès.

Une GPO s'applique au démarrage et toutes les 90 minutes. Elle ne sait pas si, entre deux cycles, le poste a été compromis, si l'utilisateur tente d'accéder depuis un réseau non conforme, ou si la session en cours présente des signaux d'anomalie.

"Verify Explicitly" requiert une évaluation **dynamique** et **contextuelle**. Les GPO opèrent en **batch périodique**.

### Ce qu'Intune + Conditional Access apportent

Intune ajoute la couche manquante :

- **Conformité continue** : Intune évalue l'état du poste (BitLocker, Defender, OS version, WDAC) et génère un signal de conformité en temps réel
- **Conditional Access** : Azure AD conditionne l'accès à chaque ressource cloud au signal de conformité Intune
- **Hybrid Join** : la machine est à la fois dans AD (GPO s'appliquent) et dans AAD (Intune évalue la conformité)

Le flux complet pour un accès à Exchange Online dans une architecture ZT hybride :

```
Utilisateur → Demande accès Exchange Online
  ↓
Azure AD évalue Conditional Access
  ↓
  Vérifie : identité MFA confirmée ?
  Vérifie : appareil conforme Intune ?
    → BitLocker actif ?
    → Defender actif et à jour ?
    → OS version dans la fenêtre de support ?
    → Aucune app non-conforme ?
  ↓
  Si tout OK → Accès accordé (token)
  Si non-conforme → Blocage ou redirection remédiation
```

### Architecture hybride GPO + Intune

La coexistence GPO + Intune (co-management) est documentée dans le [chapitre 18 — Azure AD Hybrid Join](18-azure-ad-hybrid.md). L'élément clé : la GPO qui bloque MDM doit être absente.

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM
  DisableMDMEnrollment = 0   (ou clé absente)
```

Si `DisableMDMEnrollment = 1` est présent dans une GPO, le co-management est silencieusement impossible. Intune ne peut pas évaluer la conformité. Conditional Access ne reçoit pas le signal. La boucle ZT est brisée.

!!! warning "Vérifiez cette clé avant tout projet Conditional Access"
    ```powershell title="Check-MDMEnrollmentBlock.ps1"
    # Check whether a GPO is blocking MDM enrollment across the domain
    Get-ADComputer -Filter { OperatingSystem -like "*Windows*" } |
        Select-Object -ExpandProperty Name |
        ForEach-Object {
            $val = Invoke-Command -ComputerName $_ -ScriptBlock {
                (Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM" `
                    -Name DisableMDMEnrollment -ErrorAction SilentlyContinue).DisableMDMEnrollment
            } -ErrorAction SilentlyContinue
            [PSCustomObject]@{ Computer = $_; MDMBlocked = ($val -eq 1) }
        } | Where-Object MDMBlocked
    ```

!!! quote "En résumé"
    - Les GPO ne peuvent pas implémenter "Verify Explicitly" seules — elles configurent mais ne vérifient pas dynamiquement
    - Intune + Conditional Access ajoutent la vérification continue de conformité postes et identités
    - Prérequis absolu : `DisableMDMEnrollment` doit être absent ou à 0 sur tous les postes cibles
    - Le Hybrid Join est le point d'entrée — voir [chapitre 18](18-azure-ad-hybrid.md) pour l'implémentation

---

## :material-table-split: Architecture hybride : matrice de responsabilité

### Qui gère quoi dans une architecture GPO + Intune + AAD

Dans une architecture Zero Trust hybride mature, les trois outils ont des responsabilités distinctes et non redondantes.

| Catégorie de contrôle | Outil | Méthode | Vérification |
|---|---|---|---|
| Configuration OS core (Credential Guard, WDAC) | **GPO** | ADMX via GPMC | `gpresult /r`, event logs |
| Durcissement DC et Kerberos policy | **GPO** | Politiques Default Domain / DC Policy | `Get-ADDefaultDomainPasswordPolicy` |
| BitLocker — chiffrement du volume | **GPO** | `BitLocker Drive Encryption.admx` | `Get-BitLockerVolume` + escrow AD |
| LAPS — mot de passe local unique | **GPO** | `LAPS.admx` | `Get-LapsADPassword` |
| ASR Rules — blocage comportemental | **GPO** ou **Intune** | ADMX ou Settings Catalog | Event 1121/1122 |
| Conformité postes (état temps réel) | **Intune** | Compliance Policy | Portail Intune / Graph API |
| Conditional Access — contrôle d'accès | **Azure AD** | Policies CA dans le portail AAD | Sign-in logs AAD |
| Gestion identités et MFA | **Azure AD** | Authentication Methods | AAD Audit logs |
| SYSVOL scripts, logon scripts | **GPO** | Scripts PowerShell déployés via GPO | `gpresult`, logs applicatifs |
| Applications legacy (MSI, MSP) | **GPO** (SWI) ou **SCCM** | Software Installation | SCCM reporting |

### Les quatre règles de répartition

**Règle 1** : Les GPO gèrent tout ce qui touche à l'infrastructure AD core (DC, Kerberos, NTLM, SYSVOL). Intune ne peut pas toucher aux contrôleurs de domaine.

**Règle 2** : Intune gère la conformité et la remédiation des postes cloud-attached. Les GPO gèrent la configuration initiale et les politiques qui doivent s'appliquer hors connexion Internet.

**Règle 3** : Évitez la redondance GPO + Intune sur le même paramètre. Le "last writer wins" s'applique au registre — une GPO et une policy Intune qui configurent le même paramètre génèrent des conflits silencieux.

**Règle 4** : La source de vérité pour l'état de conformité est Intune. La source de vérité pour la configuration AD est la GPO. Ne les inversez pas.

!!! info "Pour aller plus loin sur la migration GPO → Intune"
    La stratégie de migration progressive, l'outil Group Policy Analytics de Microsoft et la gestion des conflits GPO/MDM sont documentés dans [le chapitre 19 — Migration GPO vers Intune](19-migration-intune.md).

!!! quote "En résumé"
    - GPO : configuration OS, durcissement DC, politiques offline — ce qu'Intune ne peut pas faire
    - Intune : conformité temps réel, signal Conditional Access, postes AAD-joined uniquement
    - Azure AD : identité, MFA, Conditional Access — la boucle de vérification dynamique
    - Évitez la double configuration GPO + Intune sur le même paramètre — conflit silencieux garanti

---

## :material-road: Roadmap de modernisation : GPO → Settings Catalog

### La trajectoire Microsoft

Microsoft ne déprécie pas les GPO pour les environnements Active Directory on-premises. Ce point est clair dans la documentation officielle et dans les communications des équipes produit.

La trajectoire réelle est différente : Microsoft investit dans le **Settings Catalog** d'Intune comme interface de convergence. Les paramètres ADMX sont progressivement répliqués dans le Settings Catalog pour permettre aux environnements cloud-first de gérer les mêmes politiques via MDM.

Ce que cela signifie en pratique : les GPO restent le meilleur outil pour les machines on-prem AD-joined. Le Settings Catalog devient le meilleur outil pour les machines AAD-joined (Intune-only). Pour les environnements hybrides, les deux coexistent.

### Ce que les GPO feront toujours mieux que MDM

!!! warning "Ce périmètre ne sera pas remplacé par MDM — horizon 2030+"
    Certaines fonctionnalités sont structurellement liées à Active Directory et ne peuvent pas être remplacées par MDM :

    - **Configuration des contrôleurs de domaine** : les DC ne sont pas gérés par Intune
    - **Kerberos policy** : `Computer Configuration → Windows Settings → Security Settings → Account Policies → Kerberos Policy` — pas d'équivalent MDM
    - **NTLM restrictions** : `LAN Manager Authentication Level`, restrictions NTLM — spécifiques à l'infrastructure AD
    - **Fine-Grained Password Policies** : gérées via PSO (Password Settings Objects), pas via MDM
    - **SYSVOL scripts et logon scripts** : les scripts de connexion déployés via GPO n'ont pas d'équivalent Intune complet
    - **Filtrage par sécurité et WMI filtering** : la granularité de ciblage des GPO n'est pas reproduite dans Intune

### Roadmap de migration en cinq étapes

| Étape | État actuel | État cible | Outil de migration | Horizon |
|---|---|---|---|---|
| **1 — Inventaire** | GPO existantes non documentées | Cartographie complète avec Group Policy Analytics | Group Policy Analytics (Intune portal) | 1-3 mois |
| **2 — Audit de conformité** | Paramètres non suivis, no baseline | Security Baseline Microsoft déployée | Microsoft Security Baseline (GPO + Intune) | 1-2 mois |
| **3 — Hybrid Join** | Machines AD-only | Co-management GPO + Intune | Azure AD Connect + GPO MDM enrollment | 3-6 mois |
| **4 — Migration Settings Catalog** | GPO ADMX pour paramètres supportés MDM | Settings Catalog Intune (postes AAD-joined) | Group Policy Analytics + Migration report | 6-18 mois |
| **5 — Cloud-first** | Nouveaux postes GPO + Intune | Nouveaux postes Intune-only (AAD-joined) | Autopilot + Settings Catalog | En continu |

### :material-powershell: Group Policy Analytics — export pour la migration

```powershell title="Export-GPOForMigrationAnalysis.ps1"
# Export all GPOs to XML for import into Intune Group Policy Analytics
# Group Policy Analytics identifies which settings are MDM-supported
param(
    [string]$ExportDir = "C:\GPO-Migration-Export",
    [string]$Domain    = (Get-ADDomain).DNSRoot
)

if (-not (Test-Path $ExportDir)) { New-Item -ItemType Directory -Path $ExportDir | Out-Null }

$gpos = Get-GPO -All -Domain $Domain

$report = foreach ($gpo in $gpos) {
    $xmlFile = "$ExportDir\$($gpo.DisplayName -replace '[\\/:*?"<>|]', '_').xml"
    try {
        # Export GPO settings as XML (importable into Group Policy Analytics)
        Get-GPOReport -Guid $gpo.Id -ReportType Xml -Path $xmlFile
        [PSCustomObject]@{
            GPOName  = $gpo.DisplayName
            Status   = "Exported"
            XmlPath  = $xmlFile
            LastMod  = $gpo.ModificationTime
            Linked   = ($gpo.GpoStatus -ne 'AllSettingsDisabled')
        }
    } catch {
        [PSCustomObject]@{
            GPOName  = $gpo.DisplayName
            Status   = "ERROR: $_"
            XmlPath  = $null
            LastMod  = $gpo.ModificationTime
            Linked   = $null
        }
    }
}

$report | Format-Table -AutoSize
Write-Host ""
Write-Host "Total GPOs exported: $(($report | Where-Object Status -eq 'Exported').Count)"
Write-Host "Upload XML files to: Intune portal > Devices > Group Policy Analytics > Import"
```

### Le Settings Catalog comme interface de convergence

Le Settings Catalog d'Intune regroupe aujourd'hui plus de 5 000 paramètres. Microsoft continue d'y ajouter les paramètres ADMX les plus utilisés.

La convergence est réelle, mais partielle. En 2026, le Settings Catalog couvre environ 70% des paramètres GPO courants pour les postes Windows 10/11. Les 30% restants incluent les paramètres DC, les politiques AD core, et les paramètres applicatifs moins courants.

!!! info "Voir aussi"
    La convergence MDM/GPO au niveau des standards ADMX est analysée dans [La Bible des GPO — Chapitre 25 : MDM et convergence](../bible-gpo/25-mdm-convergence.md).

!!! quote "En résumé"
    - Microsoft ne déprécie pas les GPO pour l'AD on-prem — horizon 2030+ et au-delà
    - Le Settings Catalog est l'interface de convergence pour les postes cloud-first — 70% de couverture en 2026
    - Ce que MDM ne remplacera jamais : configuration DC, Kerberos policy, NTLM restrictions, SYSVOL scripts
    - Roadmap en 5 étapes : inventaire → baseline → Hybrid Join → Settings Catalog → cloud-first

---

## :material-telescope: Vision 2026+ et conclusion

### Un bilan honnête

Ce livre a été construit autour d'un postulat opérationnel : les GPO ne disparaissent pas, et les maîtriser est une compétence stratégique durable.

Le paysage de 2026 confirme ce postulat.

La majorité des organisations disposent d'un Active Directory actif. Les migrations Intune-only sont encore minoritaires. Et même dans les environnements cloud-first, les GPO restent nécessaires dès qu'un contrôleur de domaine est présent — ne serait-ce que pour Kerberos policy et la configuration des DC eux-mêmes.

La question n'est pas "GPO ou Intune". La question est **"quel outil pour quel périmètre"**.

### Ce que ce livre vous a donné

En parcourant ces 25 chapitres, vous avez acquis les outils pour :

- **Concevoir** une architecture GPO d'entreprise qui résiste à la croissance et aux audits (chapitres 1 à 7)
- **Déployer** les applications métier critiques — Office, navigateurs, Windows Update, RDS, PKI (chapitres 8 à 16)
- **Sécuriser** le parc avec BitLocker, LAPS, Credential Guard, ASR, WDAC (chapitres 14, 15, 17)
- **Migrer** progressivement vers le cloud sans interruption de service (chapitres 18, 19, 20)
- **Opérer** en autonomie : dépannage de terrain, CI/CD, sécurité des GPO elles-mêmes (chapitres 22, 23, 24)

### Ce que Zero Trust change pour votre pratique

Zero Trust n'invalide pas ces compétences — il les contextualise.

Les GPO que vous configurez deviennent des **briques de conformité** dans une évaluation continue. Credential Guard ne vaut que si vous avez une stratégie de détection des tentatives de dump LSASS. BitLocker ne vaut que si l'état de chiffrement remonte à Intune et déclenche le bon Conditional Access. LAPS ne vaut que si la rotation post-utilisation est activée.

La maîtrise technique est le prérequis. L'architecture de vérification continue est ce qui transforme cette maîtrise en posture de sécurité réelle.

### Continuer à progresser

La sécurité Windows et les GPO évoluent rapidement. Quelques ressources pour rester à niveau :

| Ressource | Contenu | URL |
|---|---|---|
| Microsoft Security Baselines | Référence GPO officielle mise à jour à chaque version Windows | `aka.ms/baselines` |
| Microsoft Security Blog | Annonces ASR, WDAC, nouvelles menaces | `microsoft.com/security/blog` |
| MSRC (Security Response Center) | Vulnérabilités actives et mitigations GPO | `msrc.microsoft.com` |
| CIS Benchmarks Windows | Références de durcissement tierces comparables aux baselines Microsoft | `cisecurity.org` |
| GitHub microsoft/securitybaselines | Scripts d'application des baselines | GitHub |
| Active Directory Security (adsecurity.org) | Sean Metcalf — référence AD attack paths | `adsecurity.org` |

!!! info "Les Microsoft Security Baselines — votre point de départ pour chaque nouveau build"
    Microsoft publie des Security Baselines pour chaque version majeure de Windows et Windows Server. Ces baselines sont des GPO exportées, prêtes à importer dans GPMC. Elles intègrent les recommandations de l'équipe sécurité Microsoft et couvrent des centaines de paramètres. Avant tout déploiement sur un nouveau build OS, importez la baseline correspondante et comparez-la à votre configuration existante.

### Un dernier mot sur l'outillage

Les GPO resteront l'outil de référence pour la configuration AD on-prem dans toute votre carrière d'administrateur.

Ce qui changera, c'est la surface qu'elles couvrent — progressivement réduite au périmètre AD core tandis que le cloud absorbe la gestion des postes modernes. Ce qui ne changera pas, c'est la rigueur nécessaire pour les concevoir, les documenter et les opérer.

Un parc mal gouverné par GPO accumule de la dette de configuration. Cette dette devient de la surface d'attaque. La différence entre un administrateur compétent et un administrateur expert tient souvent à la discipline qu'il applique à ce qui est invisible — les GPO qui s'appliquent silencieusement, toutes les 90 minutes, sur chaque machine du domaine.

Maîtriser ce mécanisme, c'est maîtriser l'infrastructure.


!!! quote "En résumé"
    - Le paysage de 2026 confirme ce postulat.
    - La majorité des organisations disposent d'un Active Directory actif.
    - Les migrations Intune-only sont encore minoritaires.
    - Concevoir une architecture GPO d'entreprise qui résiste à la croissance et aux audits (chapitres 1 à 7).
    - Déployer les applications métier critiques — Office, navigateurs, Windows Update, RDS, PKI (chapitres 8 à 16).
---
!!! quote "En résumé — chapitre final"
    - Zero Trust + GPO : les GPO durcissent l'OS (Assume Breach), Intune + AAD vérifient la conformité (Verify Explicitly), LAPS minimise les privilèges (Use Least Privilege)
    - Les GPO ne disparaissent pas : configuration DC, Kerberos policy, NTLM, SYSVOL — périmètre permanent
    - La roadmap réaliste : Hybrid Join → Settings Catalog pour les nouvelles machines → cloud-first progressif
    - Ce livre vous a équipé pour opérer, sécuriser et faire évoluer une infrastructure GPO d'entreprise dans un monde hybride
    - La compétence GPO reste stratégique — elle évolue vers une brique de conformité dans une architecture Zero Trust, pas vers l'obsolescence
