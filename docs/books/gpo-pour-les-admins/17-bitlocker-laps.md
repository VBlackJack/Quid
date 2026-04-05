---
description: BitLocker et LAPS via GPO — déploiement enterprise, recovery keys AD, LAPS v1/v2, JIT administration
tags: [gpo-admins, bitlocker, laps, local-admin, recovery-key, jit, privilege]
---
# BitLocker et LAPS via GPO

!!! abstract "Ce que vous allez apprendre"
    - Déployer BitLocker sur l'ensemble du parc via GPO, avec un script de détection d'état et un mécanisme de remédiation automatique pour les machines non encore chiffrées
    - Configurer l'escrow des clés de récupération dans Active Directory et vérifier que chaque machine a bien sauvegardé sa clé avant l'application de l'enforcement
    - Déployer LAPS v1 (legacy) avec extension du schéma AD, ACL sur l'attribut `ms-Mcs-AdmPwd`, et lecture via `Get-AdmPwdPassword`
    - Déployer LAPS v2 (Windows LAPS, intégré dans Windows 11 22H2+ et KB5025175) avec chiffrement des mots de passe, historique, et table complète des paramètres GPO
    - Intégrer LAPS avec une stratégie JIT (Just-In-Time) pour limiter la fenêtre d'exposition du compte administrateur local

!!! tip "Si vous ne retenez qu'une chose"
    Ne déployez jamais LAPS sans vérifier en amont que les ACL permettent au service desk de lire les mots de passe générés. La configuration la plus courante en production : LAPS est déployé, les mots de passe sont générés, mais **aucun opérateur ne peut les lire** — et la découverte se fait lors du premier incident nécessitant l'accès local à une machine. Vérifiez les ACL avant de pousser les GPO.

---

## Contexte de production

BitLocker et LAPS répondent à deux vecteurs de risque complémentaires : le vol physique d'un poste (BitLocker) et la compromission par mouvement latéral via un compte administrateur local partagé (LAPS).

Les deux sont souvent déployés en même temps, lors d'une initiative de durcissement du parc. Ils s'appuient sur Active Directory pour stocker les secrets (clé de récupération, mot de passe local), ce qui impose une rigueur particulière sur les ACL AD et sur la séquence de déploiement.


!!! quote "En résumé"
    - Les deux sont souvent déployés en même temps, lors d'une initiative de durcissement du parc.
    - Le contexte de production fixe les contraintes réelles de réseau, de portée et d’exploitation qui gouvernent tout le chapitre.
    - Retenez les hypothèses opérationnelles avant de choisir un modèle de liaison ou de déploiement.
---

## :material-lock: BitLocker — déploiement enterprise via GPO

### Stratégie de déploiement : qui chiffre quoi, quand

BitLocker via GPO ne chiffre pas automatiquement les disques au moment de l'application de la GPO. La GPO définit la **politique** ; c'est l'utilisateur, le démarrage ou un script qui déclenche le chiffrement réel.

L'approche recommandée en production se déroule en trois temps.

**Temps 1 — GPO de configuration** : définit les paramètres BitLocker (TPM, PIN, algorithme, escrow AD) sans activer le chiffrement.

**Temps 2 — Script de startup** : détecte les volumes non chiffrés et lance `Enable-BitLocker` en arrière-plan lors du démarrage suivant.

**Temps 3 — GPO d'enforcement** : une fois que l'escrow est vérifié sur 95 %+ du parc, active le blocage des machines non chiffrées (accès réseau restreint ou notification SCCM).

### :material-folder-cog: Chemin GPMC — paramètres BitLocker

Les paramètres BitLocker sont dans les modèles ADMX `VolumeEncryption.admx` :

```
Computer Configuration → Policies → Administrative Templates
  → Windows Components → BitLocker Drive Encryption
```

Les sous-nœuds critiques sont :

| Nœud GPMC | Portée |
|---|---|
| `BitLocker Drive Encryption` | Paramètres globaux (algorithme, escrow) |
| `Operating System Drives` | Disque système (C:), prérequis TPM |
| `Fixed Data Drives` | Disques de données fixes |
| `Removable Data Drives` | Clés USB, disques externes |

### :material-table: Paramètres GPO recommandés pour le disque système

| Paramètre | Nœud | Valeur recommandée |
|---|---|---|
| Choose drive encryption method and cipher strength | `BitLocker Drive Encryption` | XTS-AES 256 (Windows 10+) |
| Store BitLocker recovery information in AD DS | `Operating System Drives` | **Activé** — Store recovery passwords and key packages |
| Do not enable BitLocker until recovery information is stored to AD | `Operating System Drives` | **Activé** |
| Require additional authentication at startup | `Operating System Drives` | Activé — Allow BitLocker without a compatible TPM : **Non** |
| Configure TPM platform validation profile | `Operating System Drives` | Default (PCR 0, 2, 4, 11) |
| Choose how BitLocker-protected OS drives can be recovered | `Operating System Drives` | Allow 48-digit recovery password : **Activé** |

!!! danger "Production — L'escrow AD avant tout"
    Le paramètre **"Do not enable BitLocker until recovery information is stored to AD DS"** est non-négociable. Sans lui, un poste peut se chiffrer avec un TPM qui échoue au redémarrage suivant, et la clé de récupération n'est nulle part. La machine est irrécupérable sans reformatage. Activez toujours ce paramètre avant tout déploiement à grande échelle.

### :material-powershell: Script de détection d'état BitLocker — inventaire du parc

Avant de pousser le chiffrement, mesurez l'état réel du parc. Ce script interroge chaque machine et retourne l'état de chiffrement du volume C:.

```powershell title="Get-BitLockerFleetStatus.ps1"
# Retrieve BitLocker encryption status across the entire domain fleet
param(
    [string]$OU = "",          # Optional: limit to a specific OU (DistinguishedName)
    [string]$OutputCsv = ""    # Optional: export results to CSV
)

$filter = if ($OU) {
    { OperatingSystem -like "*Windows*" }
} else {
    { OperatingSystem -like "*Windows*" }
}

$searchBase = if ($OU) { @{ SearchBase = $OU } } else { @{} }

$computers = Get-ADComputer -Filter $filter @searchBase |
    Select-Object -ExpandProperty Name

$results = foreach ($computer in $computers) {
    try {
        $status = Invoke-Command -ComputerName $computer -ScriptBlock {
            $vol = Get-BitLockerVolume -MountPoint "C:" -ErrorAction Stop
            [PSCustomObject]@{
                Computer          = $env:COMPUTERNAME
                EncryptionStatus  = $vol.VolumeStatus          # FullyEncrypted, FullyDecrypted, EncryptionInProgress
                ProtectionStatus  = $vol.ProtectionStatus      # On, Off
                EncryptionMethod  = $vol.EncryptionMethod
                KeyProtectors     = ($vol.KeyProtector.KeyProtectorType) -join ", "
                PercentEncrypted  = $vol.EncryptionPercentage
            }
        } -ErrorAction Stop
        $status
    } catch {
        [PSCustomObject]@{
            Computer          = $computer
            EncryptionStatus  = "ERROR: $_"
            ProtectionStatus  = "Unknown"
            EncryptionMethod  = "Unknown"
            KeyProtectors     = "Unknown"
            PercentEncrypted  = -1
        }
    }
}

$results | Format-Table -AutoSize

if ($OutputCsv) {
    $results | Export-Csv -Path $OutputCsv -NoTypeInformation -Encoding UTF8
    Write-Host "Exported to $OutputCsv"
}
```

```title="Résultat attendu"
Computer     EncryptionStatus  ProtectionStatus  EncryptionMethod  KeyProtectors         PercentEncrypted
--------     ----------------  ----------------  ----------------  -------------         ----------------
WKS-001      FullyEncrypted    On                XtsAes256         Tpm, RecoveryPassword 100
WKS-002      FullyDecrypted    Off               None                                    0
WKS-003      EncryptionInProgress On             XtsAes256         Tpm, RecoveryPassword 47
```

### :material-rocket-launch: Remédiation automatique — script de startup GPO

Pour chiffrer les machines qui ne le sont pas encore, déployez un script PowerShell via GPO en tant que **Computer Startup Script**.

Chemin GPMC :

```
Computer Configuration → Policies → Windows Settings → Scripts (Startup/Shutdown)
  → Startup → PowerShell Scripts
```

```powershell title="Start-BitLockerRemediation.ps1"
# BitLocker remediation startup script — encrypts C: if not already encrypted
# Deploy via GPO: Computer Configuration > Windows Settings > Scripts > Startup

$logPath = "C:\Windows\Logs\BitLocker-Remediation.log"
$vol     = Get-BitLockerVolume -MountPoint "C:" -ErrorAction Stop

function Write-Log {
    param([string]$Message)
    $entry = "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] $Message"
    Add-Content -Path $logPath -Value $entry -Encoding UTF8
}

# Skip if already fully encrypted
if ($vol.VolumeStatus -eq "FullyEncrypted" -and $vol.ProtectionStatus -eq "On") {
    Write-Log "BitLocker already active — no action required."
    exit 0
}

# Check TPM availability
$tpm = Get-Tpm -ErrorAction SilentlyContinue
if (-not $tpm -or -not $tpm.TpmReady) {
    Write-Log "ERROR: TPM not ready — cannot enable BitLocker. TPMPresent=$($tpm.TpmPresent) TpmReady=$($tpm.TpmReady)"
    exit 1
}

Write-Log "Starting BitLocker encryption on C: — TPM ready."

try {
    # Add TPM key protector
    Add-BitLockerKeyProtector -MountPoint "C:" -TpmProtector -ErrorAction Stop |
        Out-Null

    # Add recovery password protector (will be escrowed to AD by GPO policy)
    $keyResult = Add-BitLockerKeyProtector -MountPoint "C:" -RecoveryPasswordProtector -ErrorAction Stop
    $recoveryKeyId = $keyResult.KeyProtector |
        Where-Object { $_.KeyProtectorType -eq "RecoveryPassword" } |
        Select-Object -ExpandProperty KeyProtectorId

    Write-Log "Key protectors added. Recovery Key ID: $recoveryKeyId"

    # Back up recovery key to AD before starting encryption
    Backup-BitLockerKeyProtector -MountPoint "C:" -KeyProtectorId $recoveryKeyId -ErrorAction Stop
    Write-Log "Recovery key escrowed to Active Directory successfully."

    # Start encryption (SkipHardwareTest allows encryption without reboot cycle)
    Enable-BitLocker -MountPoint "C:" -EncryptionMethod XtsAes256 -SkipHardwareTest -ErrorAction Stop |
        Out-Null

    Write-Log "Encryption started. Status: $(( Get-BitLockerVolume -MountPoint 'C:' ).VolumeStatus)"
} catch {
    Write-Log "ERROR during BitLocker activation: $_"
    exit 1
}
```

!!! warning "À surveiller — SkipHardwareTest"
    Le flag `-SkipHardwareTest` permet de démarrer le chiffrement sans redémarrage préalable pour le test matériel. Il est utile en déploiement automatisé mais contourne la vérification du TPM au boot. Utilisez-le uniquement si vous avez validé la compatibilité TPM sur les modèles de votre parc.

### :material-key-variant: Vérification de l'escrow — clé sauvegardée dans AD

Avant d'activer toute forme d'enforcement, vérifiez que les clés sont bien dans AD.

```powershell title="Verify-BitLockerEscrow.ps1"
# Verify that each computer has a BitLocker recovery key stored in Active Directory
param(
    [string]$SearchBase = (Get-ADDomain).DistinguishedName
)

$computers = Get-ADComputer -Filter { OperatingSystem -like "*Windows*" } `
    -SearchBase $SearchBase | Select-Object -ExpandProperty DistinguishedName

foreach ($dn in $computers) {
    $cn = ($dn -split ',')[0] -replace '^CN='

    # BitLocker recovery objects are stored as child objects of the computer object
    $recoveryObjects = Get-ADObject -Filter { objectClass -eq "msFVE-RecoveryInformation" } `
        -SearchBase $dn -Properties msFVE-RecoveryPassword, Created -ErrorAction SilentlyContinue

    [PSCustomObject]@{
        Computer      = $cn
        KeysInAD      = ($recoveryObjects | Measure-Object).Count
        LatestBackup  = ($recoveryObjects | Sort-Object Created -Descending | Select-Object -First 1).Created
    }
}
```

```title="Résultat attendu"
Computer   KeysInAD  LatestBackup
--------   --------  ------------
WKS-001    1         2026-03-15 08:12:44
WKS-002    0                               # pas encore chiffré — à surveiller
WKS-003    2         2026-04-01 07:30:11   # 2 clés = rotation effectuée
```

Toute machine avec `KeysInAD = 0` doit être traitée avant l'enforcement.

!!! quote "En résumé"
    - GPO BitLocker : nœud `Computer Configuration → … → BitLocker Drive Encryption → Operating System Drives`
    - Algorithme recommandé : XTS-AES 256 — configurable dans le nœud parent
    - Escrow AD : activez **"Do not enable BitLocker until recovery info is stored to AD"** — obligatoire
    - Inventaire préalable : `Get-BitLockerVolume` en remoting sur le parc avant tout déploiement
    - Séquence : GPO de config → script startup pour remédiation → vérification escrow → enforcement

---

## :material-account-key: LAPS v1 (legacy) — Local Administrator Password Solution

### Pourquoi LAPS v1 est encore présent partout

LAPS v1 (Microsoft LAPS, aka `AdmPwd.dll`) a été publié en 2015 et reste déployé dans la très grande majorité des environnements Active Directory.

Il fonctionne via une extension de schéma AD qui ajoute deux attributs au schéma de la classe `Computer`. L'extension de schéma est irréversible — elle ne peut pas être annulée. C'est une opération à planifier avec soin.

### :material-database-cog: Extension du schéma AD

L'extension du schéma ajoute deux attributs :

| Attribut AD | OID | Contenu |
|---|---|---|
| `ms-Mcs-AdmPwd` | 1.2.840.113556.1.8000.2554.50051 | Mot de passe en clair (confidentiel) |
| `ms-Mcs-AdmPwdExpirationTime` | 1.2.840.113556.1.8000.2554.50052 | Date d'expiration (FileTime 64-bit) |

L'extension se réalise avec le module PowerShell `AdmPwd.PS` inclus dans le package LAPS.

```powershell title="Extend-ADSchema-LAPS-v1.ps1"
# Extend Active Directory schema for LAPS v1 (AdmPwd)
# Must be run as Schema Admin on the Schema Master DC

Import-Module AdmPwd.PS

# Update the AD schema — this operation is irreversible
Update-AdmPwdADSchema
```

```title="Résultat attendu"
Name                   DistinguishedName
----                   -----------------
ms-Mcs-AdmPwd          CN=ms-Mcs-AdmPwd,CN=Schema,...
ms-Mcs-AdmPwdExpiration CN=ms-Mcs-AdmPwdExpirationTime,CN=Schema,...
```

### :material-shield-lock: ACL sur l'attribut — qui peut lire le mot de passe

Par défaut, après l'extension, **personne** ne peut lire `ms-Mcs-AdmPwd` (attribut confidentiel). Il faut accorder explicitement la permission de lecture aux groupes qui en ont besoin (service desk, équipe sécurité).

```powershell title="Set-LAPSv1-ReadPermissions.ps1"
# Grant read permission on ms-Mcs-AdmPwd attribute to a specific group
# Run once per OU that contains LAPS-managed computers

param(
    [Parameter(Mandatory)]
    [string]$OUDistinguishedName,   # e.g. "OU=Workstations,DC=contoso,DC=local"

    [Parameter(Mandatory)]
    [string]$AllowedGroup           # e.g. "CONTOSO\ServiceDesk-L1"
)

Import-Module AdmPwd.PS

Set-AdmPwdReadPasswordPermission -OrgUnit $OUDistinguishedName -AllowedPrincipals $AllowedGroup
```

```title="Résultat attendu"
OrganizationalUnit                                  Identity             Rights
------------------                                  --------             ------
OU=Workstations,DC=contoso,DC=local                 CONTOSO\ServiceDesk  ReadProperty (ms-Mcs-AdmPwd)
```

Répétez cette opération pour chaque OU contenant des machines gérées par LAPS.

### :material-folder-cog: GPO LAPS v1 — paramètres ADMX

Le modèle `AdmPwd.admx` s'installe avec le package LAPS. Copiez-le dans le Central Store.

Chemin GPMC :

```
Computer Configuration → Policies → Administrative Templates → LAPS
```

| Paramètre | Valeur recommandée |
|---|---|
| Enable local admin password management | **Activé** |
| Password Settings — Complexity | Large letters + small letters + numbers + special characters |
| Password Settings — Length | **14** (minimum) |
| Password Settings — Age (days) | **30** |
| Name of administrator account to manage | Laisser vide pour gérer le compte built-in (`Administrator`, RID 500) |
| Do not allow password expiration time longer than required | **Activé** |

### :material-magnify: Lecture du mot de passe LAPS v1

```powershell title="Get-LAPSv1-Password.ps1"
# Retrieve LAPS v1 password for a specific computer from Active Directory
param(
    [Parameter(Mandatory)]
    [string]$ComputerName
)

Import-Module AdmPwd.PS

Get-AdmPwdPassword -ComputerName $ComputerName | Select-Object ComputerName, Password, ExpirationTimestamp
```

```title="Résultat attendu"
ComputerName  Password          ExpirationTimestamp
------------  --------          -------------------
WKS-001       Xk9#mPqR!2nLv     2026-05-05 08:00:00
```

!!! danger "Production — Attribut confidentiel mais pas chiffré"
    En LAPS v1, `ms-Mcs-AdmPwd` contient le mot de passe **en clair** dans AD. Toute personne disposant des droits de lecture sur cet attribut (via les ACL AD) peut le lire, y compris depuis `ldapsearch` ou tout outil LDAP. Assurez-vous que le groupe autorisé à lire l'attribut est **strictement limité** et audité.

### :material-table: LAPS v1 vs LAPS v2 — tableau de décision pour la migration

| Critère | LAPS v1 (AdmPwd) | LAPS v2 (Windows LAPS) |
|---|---|---|
| Distribution | Package MSI à déployer | Intégré Windows 11 22H2+ / KB5025175 |
| Attribut AD | `ms-Mcs-AdmPwd` (clair) | `msLAPS-EncryptedPassword` (chiffré) |
| Chiffrement du mot de passe | Non | Oui (DPAPI-based, clé dans AD) |
| Historique des mots de passe | Non | Oui (configurable) |
| Compte gérable | Built-in Admin (RID 500) | Tout compte local |
| Support Entra ID (Azure AD) | Non | Oui |
| Compatibilité OS | Windows Vista+ | Windows 10 20H2+ avec KB |
| Intégration GPO native | Via ADMX externe | ADMX natif Windows |
| Audit des lectures | Limité (Event 4662) | Event dédié (EventID 10020+) |

**Migrer vers LAPS v2 si** : vous êtes sur Windows 11 22H2+ sur la majorité du parc, ou si vous avez besoin du chiffrement des mots de passe, de l'historique, ou du support Entra ID.

**Garder LAPS v1 si** : parc hétérogène avec beaucoup de Windows 10 older builds, ou si la migration ne peut pas se planifier à court terme.

!!! quote "En résumé"
    - Extension de schéma LAPS v1 : `Update-AdmPwdADSchema` — opération **irréversible**
    - ACL : `Set-AdmPwdReadPasswordPermission` par OU — à faire avant tout déploiement GPO
    - GPO : nœud `Administrative Templates → LAPS` via `AdmPwd.admx`
    - Lecture : `Get-AdmPwdPassword -ComputerName <machine>` — module `AdmPwd.PS` requis
    - v1 vs v2 : le critère décisif est la présence de KB5025175 (Windows LAPS) sur le parc

---

## :material-lock-smart: LAPS v2 (Windows LAPS) — intégré Windows 11 22H2+

### Ce qui change avec Windows LAPS

Windows LAPS est une réécriture complète, intégrée directement dans Windows depuis la version 22H2 (et backportée sur Windows 10 et Server via KB5025175 d'avril 2023).

Il n'y a plus de MSI à déployer. Le CSE (Client-Side Extension) est natif dans l'OS. La configuration se fait entièrement via GPO ou MDM Policy CSP.

!!! info "Prérequis LAPS v2 par OS"
    | Système | Prérequis | Natif |
    |---|---|---|
    | Windows Server 2022 | KB5025175 (avril 2023) ou ultérieur | Oui (post-patch) |
    | Windows Server 2019 | KB5025172 (avril 2023) ou ultérieur | Oui (post-patch) |
    | Windows 11 21H2+ | Intégré depuis 22H2 | Oui |
    | Windows 10 20H2+ | KB5025221 (avril 2023) ou ultérieur | Oui (post-patch) |

    Sur **Windows Server 2022**, LAPS v2 (Windows LAPS) est disponible nativement
    après l'installation des mises à jour d'avril 2023. Aucun agent MSI supplémentaire
    n'est requis, contrairement à LAPS v1 (AdmPwd.exe).

### :material-database-cog: Extension du schéma AD pour LAPS v2

Même si le client est intégré, le schéma AD doit être étendu pour les nouveaux attributs. Cette opération est distincte de celle de LAPS v1 — les deux extensions coexistent.

```powershell title="Extend-ADSchema-LAPS-v2.ps1"
# Extend Active Directory schema for Windows LAPS (v2)
# Requires AD PowerShell module and Schema Admin rights

Import-Module ActiveDirectory

# Update schema — adds msLAPS-* attributes to the Computer class
Update-LapsADSchema -Verbose
```

```title="Résultat attendu"
VERBOSE: Updating AD schema for Windows LAPS...
VERBOSE: Added attribute msLAPS-Password
VERBOSE: Added attribute msLAPS-EncryptedPassword
VERBOSE: Added attribute msLAPS-PasswordExpirationTime
VERBOSE: Added attribute msLAPS-EncryptedPasswordHistory
VERBOSE: Schema update complete.
```

Les quatre nouveaux attributs :

| Attribut | Usage |
|---|---|
| `msLAPS-Password` | Mot de passe en clair (si chiffrement désactivé) |
| `msLAPS-EncryptedPassword` | Mot de passe chiffré (recommandé) |
| `msLAPS-PasswordExpirationTime` | Date d'expiration |
| `msLAPS-EncryptedPasswordHistory` | Historique des mots de passe chiffrés |

### :material-folder-cog: GPO LAPS v2 — chemin GPMC et paramètres

Le nœud GPO Windows LAPS est natif dans les modèles ADMX Windows 11 22H2+. Si votre Central Store est basé sur des modèles plus anciens, copiez les fichiers `LAPS.admx` et `LAPS.adml` depuis un poste à jour.

Chemin GPMC :

```
Computer Configuration → Policies → Administrative Templates
  → System → LAPS
```

### :material-table: Tous les paramètres GPO Windows LAPS avec chemins de registre

| Paramètre GPMC | Registre | Valeurs / Notes |
|---|---|---|
| Configure password backup directory | `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\LAPS\BackupDirectory` | `0` = Disabled, `1` = AD, `2` = Entra ID |
| Password Settings — Complexity | `HKLM\…\LAPS\PasswordComplexity` | `1`=Upper, `2`=+Lower, `3`=+Digits, `4`=+Special |
| Password Settings — Length | `HKLM\…\LAPS\PasswordLength` | Défaut : 14, Recommandé : **20** |
| Password Settings — Age (days) | `HKLM\…\LAPS\PasswordAgeDays` | Défaut : 30, plage : 1–365 |
| Enable password encryption | `HKLM\…\LAPS\ADPasswordEncryptionEnabled` | `1` = activé (recommandé) |
| Password encryption — authorized decryptors | `HKLM\…\LAPS\ADPasswordEncryptionPrincipal` | SID ou nom du groupe autorisé |
| Number of passwords in history | `HKLM\…\LAPS\PasswordHistorySize` | `0` = aucun historique, max : 12 |
| Configure size of encrypted password history | `HKLM\…\LAPS\EncryptedPasswordHistorySize` | Nombre d'entrées historique chiffré |
| Configure post-authentication actions | `HKLM\…\LAPS\PostAuthenticationActions` | `1`=Reset pwd, `3`=Reset+Logoff, `5`=Reset+Reboot |
| Post-authentication reset delay (hours) | `HKLM\…\LAPS\PostAuthenticationResetDelay` | Délai après utilisation avant rotation |
| Name of administrator account to manage | `HKLM\…\LAPS\AdministratorAccountName` | Vide = compte Built-in (RID 500) |
| Do not allow password expiration time longer than required | `HKLM\…\LAPS\PasswordExpirationProtectionEnabled` | `1` = activé |
| Enable logging and tracing | `HKLM\…\LAPS\LoggingEnabled` | `1` pour debug — désactiver en production |

Le chemin de registre de base est :

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\LAPS\
```

### :material-shield-lock: ACL pour LAPS v2

La commande équivalente à `Set-AdmPwdReadPasswordPermission` pour LAPS v2 :

```powershell title="Set-LAPSv2-ReadPermissions.ps1"
# Grant read permission on Windows LAPS encrypted password attribute to a group
param(
    [Parameter(Mandatory)]
    [string]$OUDistinguishedName,

    [Parameter(Mandatory)]
    [string]$AllowedGroup
)

# Set-LapsADReadPasswordPermission is part of the LAPS PowerShell module (Windows LAPS)
Set-LapsADReadPasswordPermission -Identity $OUDistinguishedName -AllowedPrincipals $AllowedGroup
```

### :material-magnify: Lecture du mot de passe LAPS v2

```powershell title="Get-LAPSv2-Password.ps1"
# Retrieve Windows LAPS (v2) password for a specific computer
param(
    [Parameter(Mandatory)]
    [string]$ComputerName
)

# Get-LapsADPassword requires the LAPS module (included with Windows LAPS feature)
Get-LapsADPassword -Identity $ComputerName -AsPlainText | Select-Object ComputerName, Password, ExpirationTimestamp
```

```title="Résultat attendu"
ComputerName  Password              ExpirationTimestamp
------------  --------              -------------------
WKS-001       R#7kPmXq!9nLvZw2      2026-05-05 08:00:00
```

!!! quote "En résumé"
    - Windows LAPS est natif sur Windows 11 22H2+ et KB5025175 — pas de MSI
    - Extension schéma : `Update-LapsADSchema` — distincte de l'extension LAPS v1
    - GPO : nœud `System → LAPS` — registre sous `HKLM\…\Policies\LAPS\`
    - Activez le chiffrement (`ADPasswordEncryptionEnabled = 1`) — c'est le différenciateur clé vs v1
    - Actions post-authentification : `PostAuthenticationActions = 3` (reset + logoff) en production

---

## :material-swap-horizontal: Coexistence LAPS v1 et v2 — migration progressive

### Le problème de coexistence

Si Windows LAPS (v2) est installé sur une machine **et** que la GPO LAPS v1 (AdmPwd) est active, les deux CSE tentent de gérer le compte administrateur local.

**Windows LAPS v2 a priorité** sur LAPS v1 si les deux sont configurés sur la même machine. La machine ignore alors les paramètres LAPS v1. Cela peut créer une confusion opérationnelle : le service desk tente de lire le mot de passe via `Get-AdmPwdPassword` mais le compte a été changé par LAPS v2 depuis un moment.

### :material-powershell: Détection de la version LAPS active sur une machine

```powershell title="Get-LAPSActiveVersion.ps1"
# Detect which LAPS version (v1, v2, or both) is active on a set of computers
param(
    [string[]]$ComputerNames = @($env:COMPUTERNAME)
)

foreach ($computer in $ComputerNames) {
    $result = Invoke-Command -ComputerName $computer -ScriptBlock {
        # Check for Windows LAPS (v2) — present as a Windows capability
        $lapsV2Present = (Get-WindowsCapability -Online -Name "LAPS*" -ErrorAction SilentlyContinue |
            Where-Object { $_.State -eq "Installed" }).Count -gt 0

        # Check for LAPS v1 CSE DLL
        $lapsV1Present = Test-Path "C:\Program Files\LAPS\CSE\AdmPwd.dll"

        # Check which policy is actually active via registry
        $v2PolicyActive = Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\LAPS"
        $v1PolicyActive = Test-Path "HKLM:\SOFTWARE\Policies\Microsoft Services\AdmPwd"

        [PSCustomObject]@{
            Computer      = $env:COMPUTERNAME
            LAPSv1_CSE    = $lapsV1Present
            LAPSv2_CSE    = $lapsV2Present
            LAPSv1_Policy = $v1PolicyActive
            LAPSv2_Policy = $v2PolicyActive
            ActiveVersion = if ($lapsV2Present -and $v2PolicyActive) { "v2 (Windows LAPS)" }
                            elseif ($lapsV1Present -and $v1PolicyActive) { "v1 (AdmPwd)" }
                            elseif ($lapsV1Present -and $v2PolicyActive) { "v2 overrides v1" }
                            else { "None" }
        }
    } -ErrorAction SilentlyContinue

    $result
}
```

```title="Résultat attendu"
Computer  LAPSv1_CSE  LAPSv2_CSE  LAPSv1_Policy  LAPSv2_Policy  ActiveVersion
--------  ----------  ----------  -------------  -------------  -------------
WKS-001   True        True        True           True           v2 overrides v1
WKS-002   True        False       True           False          v1 (AdmPwd)
WKS-011   False       True        False          True           v2 (Windows LAPS)
```

### Stratégie de migration recommandée

La migration de LAPS v1 vers v2 se fait OU par OU, jamais en masse.

**Étape 1** : Étendez le schéma LAPS v2 (`Update-LapsADSchema`) — sans impact sur les machines v1 existantes.

**Étape 2** : Créez une GPO Windows LAPS et liez-la à une OU pilote (20-50 machines).

**Étape 3** : Vérifiez que les mots de passe sont bien générés et lisibles via `Get-LapsADPassword`.

**Étape 4** : Sur les machines pilotes, désactivez la GPO LAPS v1 et désinstallez le MSI AdmPwd.

**Étape 5** : Étendez OU par OU jusqu'à couverture complète, puis révoquez la GPO LAPS v1 globale.

!!! warning "À surveiller — Deux clés dans AD pendant la migration"
    Pendant la période de coexistence, une machine peut avoir à la fois `ms-Mcs-AdmPwd` (v1) et `msLAPS-EncryptedPassword` (v2) dans AD. Le mot de passe actif est celui de v2 — mais le service desk peut lire l'ancien de v1 par erreur. Formez l'équipe sur la distinction et indiquez la version active dans votre CMDB ou la description de l'OU.

!!! quote "En résumé"
    - Windows LAPS v2 prend le dessus si les deux sont configurés — LAPS v1 devient inactif silencieusement
    - Détection : vérifiez `HKLM:\…\Policies\LAPS` (v2) vs `HKLM:\SOFTWARE\Policies\Microsoft Services\AdmPwd` (v1)
    - Migration : OU par OU, pas en masse — validez v2 avant de désactiver v1
    - Les deux extensions de schéma (v1 et v2) peuvent coexister sans conflit dans AD

---

## :material-timer-lock: Intégration JIT — Just-In-Time administration avec LAPS

### Le modèle JIT : LAPS + fenêtre d'accès temporaire

LAPS fournit le **secret** (le mot de passe du compte administrateur local). Mais LAPS seul ne limite pas la **durée** pendant laquelle ce mot de passe est utilisable par un opérateur.

L'intégration JIT complète le dispositif : LAPS fournit le mot de passe, et une solution PAM/PIM accorde une **fenêtre d'accès temporaire** (30 min, 2h, 4h) après approbation. Une fois la fenêtre expirée, l'accès est révoqué **et** le mot de passe LAPS est immédiatement tourné.

### :material-flow: Workflow d'accès JIT + LAPS

```
Technicien → Demande d'accès (PAM/PIM portal)
                ↓
       Approbation (manager ou auto pour L1)
                ↓
       PAM accorde la fenêtre (ex: 2h)
                ↓
    Technicien lit le mot de passe LAPS via Get-LapsADPassword
                ↓
    Connexion locale avec le compte Administrator (mot de passe LAPS)
                ↓
    Fin de fenêtre → PAM révoque + déclenche rotation LAPS
                ↓
    Mot de passe LAPS changé → accès impossible sans nouvelle demande
```

### :material-microsoft: Microsoft PAM vs solutions tierces

| Fonctionnalité | Microsoft PAM (MIM) | Entra PIM (cloud) | Solutions tierces (CyberArk, BeyondTrust) |
|---|---|---|---|
| Fenêtres d'accès temporaire | Oui (shadow groups) | Oui (eligible roles) | Oui (checkout/checkin) |
| Intégration LAPS v1 | Manuelle | Limitée | Native selon produit |
| Intégration Windows LAPS v2 | Partielle | Oui (Entra-joined) | Native selon produit |
| Approbation multi-niveaux | Oui (MIM workflows) | Oui | Oui |
| Enregistrement de session | Non (natif) | Non (natif) | Oui (session recording) |
| Complexité de déploiement | Élevée (MIM) | Faible (cloud) | Moyenne à élevée |
| Adapté pour AD on-premises | Oui | Hybride uniquement | Oui |

**Recommandation pour AD on-premises pur** : Microsoft PAM avec MIM (complexe mais natif) ou CyberArk/BeyondTrust si budget disponible.

**Recommandation pour environnement hybride (AD + Entra ID)** : Entra PIM pour les rôles cloud + Windows LAPS v2 pour les machines, avec la fonctionnalité `PostAuthenticationActions` pour forcer la rotation après utilisation.

### :material-timer-cog: Configuration LAPS v2 pour rotation post-utilisation (JIT simplifié)

En l'absence d'une vraie solution PAM, Windows LAPS v2 propose un mécanisme de rotation automatique après authentification via `PostAuthenticationActions`.

```
Computer Configuration → Policies → Administrative Templates → System → LAPS
  → Configure post-authentication actions
```

| Valeur | Comportement après authentification avec le compte LAPS |
|---|---|
| `1` | Réinitialise uniquement le mot de passe |
| `3` | Réinitialise le mot de passe **et** déconnecte la session |
| `5` | Réinitialise le mot de passe **et** redémarre la machine |

La valeur `3` est la plus équilibrée pour un JIT simplifié : le mot de passe change dès que la session est fermée.

Le délai entre la fin de session et la rotation se configure via `PostAuthenticationResetDelay` (en heures, défaut : 24h — **à réduire à 1h** en production).

!!! warning "À surveiller — PostAuthenticationActions en production"
    La valeur `5` (reset + reboot) peut surprendre les utilisateurs si elle s'applique à des machines autres que les serveurs. Limitez la GPO avec `PostAuthenticationActions = 5` aux OUs serveurs uniquement, et utilisez `3` (reset + logoff) pour les postes de travail.

!!! quote "En résumé"
    - LAPS = secret ; JIT = fenêtre d'accès temporaire — les deux ensemble forment le modèle de moindre privilège complet
    - Windows LAPS v2 `PostAuthenticationActions = 3` fournit un JIT simplifié sans solution PAM dédiée
    - `PostAuthenticationResetDelay` : réduire à 1h en production (défaut 24h trop long)
    - Pour un vrai JIT avec approbation : Microsoft PAM (MIM) on-prem ou Entra PIM en hybride

---

## :material-bug: Piège production — LAPS déployé mais illisible depuis le helpdesk

### Le scénario classique

C'est le piège le plus fréquent du déploiement LAPS. La séquence d'événements :

1. GPO LAPS déployée et validée techniquement (mots de passe générés dans AD).
2. Premier incident nécessitant l'accès local à une machine.
3. Le technicien L1 tente de lire le mot de passe : **accès refusé**.
4. Escalade en urgence vers l'admin AD, qui découvre que les ACL n'ont jamais été configurées pour le groupe helpdesk.

Les mots de passe sont là, dans AD, mais **aucun groupe opérationnel n'a la permission de les lire**.

### :material-magnify: Script de détection — vérifier qui peut lire les attributs LAPS

```powershell title="Audit-LAPSReadPermissions.ps1"
# Audit which security principals have read permissions on LAPS password attributes
param(
    [Parameter(Mandatory)]
    [string]$OUDistinguishedName
)

# Load the AD module
Import-Module ActiveDirectory

$ou = Get-ADOrganizationalUnit -Identity $OUDistinguishedName
$acl = Get-Acl -Path "AD:\$($ou.DistinguishedName)"

# LAPS v1 attribute GUID
$lapsV1Guid  = [guid]"bf967950-0de6-11d0-a285-00aa003049e2"  # ms-Mcs-AdmPwd
# LAPS v2 attribute GUID
$lapsV2Guid  = [guid]"f8c1a0d0-4c2a-11d5-a4ca-00c04fd430c8"  # msLAPS-EncryptedPassword (example GUID)

$lapsAcls = $acl.Access | Where-Object {
    $_.ActiveDirectoryRights -match "ReadProperty" -and
    ($_.ObjectType -eq $lapsV1Guid -or $_.ObjectType -eq $lapsV2Guid -or
     $_.ObjectType -eq [guid]::Empty)   # empty GUID = all properties
}

if ($lapsAcls) {
    Write-Host "Principals with LAPS read access on OU '$OUDistinguishedName':" -ForegroundColor Green
    $lapsAcls | Select-Object IdentityReference, ActiveDirectoryRights, ObjectType |
        Format-Table -AutoSize
} else {
    Write-Host "WARNING: No explicit LAPS read permissions found on OU '$OUDistinguishedName'" -ForegroundColor Red
    Write-Host "The helpdesk will NOT be able to read LAPS passwords for computers in this OU."
}
```

```title="Résultat attendu (problème détecté)"
WARNING: No explicit LAPS read permissions found on OU 'OU=Workstations,DC=contoso,DC=local'
The helpdesk will NOT be able to read LAPS passwords for computers in this OU.
```

### :material-wrench: Remédiation — accorder les permissions de lecture

**Pour LAPS v1 :**

```powershell title="Remediate-LAPSv1-ReadPermissions.ps1"
# Grant LAPS v1 read permissions to helpdesk group on all workstation OUs
param(
    [Parameter(Mandatory)]
    [string[]]$OUList,

    [Parameter(Mandatory)]
    [string]$HelpdeskGroup    # e.g. "CONTOSO\Helpdesk-LAPS-Readers"
)

Import-Module AdmPwd.PS

foreach ($ou in $OUList) {
    Write-Host "Setting LAPS v1 read permission on: $ou"
    Set-AdmPwdReadPasswordPermission -OrgUnit $ou -AllowedPrincipals $HelpdeskGroup
    Write-Host "  Done." -ForegroundColor Green
}
```

**Pour LAPS v2 (Windows LAPS) :**

```powershell title="Remediate-LAPSv2-ReadPermissions.ps1"
# Grant Windows LAPS (v2) read permissions to helpdesk group on all workstation OUs
param(
    [Parameter(Mandatory)]
    [string[]]$OUList,

    [Parameter(Mandatory)]
    [string]$HelpdeskGroup
)

foreach ($ou in $OUList) {
    Write-Host "Setting LAPS v2 read permission on: $ou"
    Set-LapsADReadPasswordPermission -Identity $ou -AllowedPrincipals $HelpdeskGroup
    Write-Host "  Done." -ForegroundColor Green
}
```

### :material-check-all: Vérification complète post-remédiation

```powershell title="Verify-LAPSAccess-E2E.ps1"
# End-to-end verification: attempt to read LAPS password as a helpdesk user
# Run this script as a member of the helpdesk group (not as Domain Admin)
param(
    [Parameter(Mandatory)]
    [string]$TestComputerName
)

Write-Host "Testing LAPS access for computer: $TestComputerName" -ForegroundColor Cyan
Write-Host "Running as: $([System.Security.Principal.WindowsIdentity]::GetCurrent().Name)"
Write-Host ""

# Test LAPS v1
try {
    Import-Module AdmPwd.PS -ErrorAction Stop
    $v1 = Get-AdmPwdPassword -ComputerName $TestComputerName -ErrorAction Stop
    Write-Host "LAPS v1 read: OK — Password expires $($v1.ExpirationTimestamp)" -ForegroundColor Green
} catch {
    Write-Host "LAPS v1 read: FAILED — $_" -ForegroundColor Red
}

# Test LAPS v2
try {
    $v2 = Get-LapsADPassword -Identity $TestComputerName -AsPlainText -ErrorAction Stop
    Write-Host "LAPS v2 read: OK — Password expires $($v2.ExpirationTimestamp)" -ForegroundColor Green
} catch {
    Write-Host "LAPS v2 read: FAILED — $_" -ForegroundColor Red
}
```

!!! danger "Production — Testez avec un compte helpdesk, pas Domain Admin"
    Le test de lecture LAPS **doit être réalisé avec un compte membre du groupe helpdesk**, pas avec un compte Domain Admin. Un Domain Admin peut toujours lire `ms-Mcs-AdmPwd` via les droits implicites — ce test ne valide pas que le groupe opérationnel a bien accès.

!!! quote "En résumé"
    - Déployer LAPS sans ACL helpdesk = mots de passe générés mais inexploitables en production
    - Détection : audit des ACL sur l'OU avec `Get-Acl -Path "AD:\<OU>"`
    - Remédiation v1 : `Set-AdmPwdReadPasswordPermission` — remédiation v2 : `Set-LapsADReadPasswordPermission`
    - Validation : test de lecture avec un compte du groupe helpdesk, pas Domain Admin
    - À faire **avant** la mise en production de LAPS, pas après le premier incident

---

## :material-check-circle: Vérification globale post-déploiement

Après avoir déployé BitLocker et LAPS, exécutez cette vérification synthétique sur un échantillon représentatif du parc.

```powershell title="Verify-BitLockerLAPS-Deployment.ps1"
# Comprehensive post-deployment check: BitLocker status + LAPS password presence
param(
    [string[]]$SampleComputers    # Provide a representative sample
)

foreach ($computer in $SampleComputers) {
    $result = Invoke-Command -ComputerName $computer -ScriptBlock {
        # BitLocker status
        $bl = Get-BitLockerVolume -MountPoint "C:" -ErrorAction SilentlyContinue
        $blStatus = if ($bl) { "$($bl.VolumeStatus) / Protection: $($bl.ProtectionStatus)" } else { "WMI error" }

        # LAPS v2 policy registry key
        $lapsV2Active = Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\LAPS"

        # LAPS v1 policy registry key
        $lapsV1Active = Test-Path "HKLM:\SOFTWARE\Policies\Microsoft Services\AdmPwd"

        [PSCustomObject]@{
            Computer      = $env:COMPUTERNAME
            BitLocker     = $blStatus
            LAPSv1_Active = $lapsV1Active
            LAPSv2_Active = $lapsV2Active
            LastGPOApply  = (Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" `
                                -Name "SyncForegroundPolicy" -ErrorAction SilentlyContinue) |
                            Select-Object -ExpandProperty SyncForegroundPolicy
        }
    } -ErrorAction SilentlyContinue

    $result
}
```

```title="Résultat attendu"
Computer  BitLocker                              LAPSv1_Active  LAPSv2_Active
--------  ---------                              -------------  -------------
WKS-001   FullyEncrypted / Protection: On        False          True
WKS-002   FullyEncrypted / Protection: On        False          True
WKS-003   FullyDecrypted / Protection: Off       False          True           # BitLocker à corriger
WKS-010   FullyEncrypted / Protection: On        True           True           # migration v1→v2 en cours
```


!!! quote "En résumé"
    - Après avoir déployé BitLocker et LAPS, exécutez cette vérification synthétique sur un échantillon représentatif du parc.
    - Validez toujours le résultat sur un poste ou un utilisateur réellement dans le périmètre avant d’élargir.
    - Conservez les commandes et résultats de contrôle comme preuve de conformité post-déploiement.
---

## Références croisées

- [BitLocker — fonctionnement interne et TPM](../bible-gpo/16-bitlocker.md) — pour comprendre les modes de protection TPM, les PCR et le recovery flow avant de déployer
- [Sécurité endpoint via GPO](14-securite-endpoint.md) — Credential Guard, LSASS protection et Exploit Protection complètent le durcissement des postes
- [Sécurité des GPO elles-mêmes](24-securite-gpo.md) — protéger l'accès à la GPO LAPS et aux droits de lecture AD contre toute élévation non autorisée

!!! quote "En résumé"
    - À relire : BitLocker — fonctionnement interne et TPM.
    - À relire : Sécurité endpoint via GPO.
    - À relire : Sécurité des GPO elles-mêmes.
    - Ces renvois prolongent le chapitre avec des mécanismes complémentaires ou des cas d’usage voisins.
    - Gardez ces chapitres sous la main pour le diagnostic ou la conception d’une GPO liée à ce thème.
