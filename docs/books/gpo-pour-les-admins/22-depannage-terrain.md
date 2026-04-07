---
description: Dépannage GPO terrain — 15 cas réels avec symptôme, diagnostic, cause racine, solution et prévention
tags: [gpo-admins, depannage, troubleshooting, cas-reels, diagnostic, rsop]
---
# Dépannage terrain : 15 cas réels

!!! abstract "Ce que couvre ce chapitre"
    - **15 cas réels** de dysfonctionnements GPO rencontrés en production, de l'incident mineur au blocage critique
    - Pour chaque cas : symptôme observé, environnement, diagnostic pas à pas, cause racine identifiée, solution appliquée et règle de prévention
    - Les **outils de diagnostic** adaptés à chaque situation : `gpresult`, `Get-WinEvent`, `Test-ComputerSecureChannel`, `dfsrdiag`, `manage-bde` et leurs options clés
    - Les pièges les plus fréquents — WMI Filter mal écrits, BitLocker post-chiffrement, profils RDS corrompus — avec les commandes de remédiation prêtes à l'emploi

!!! tip "Si vous ne retenez qu'une chose"
    Avant toute investigation approfondie, lancez `gpresult /r` sur la machine cible **en tant qu'administrateur local**. Ce rapport de deux pages vous dira immédiatement quelles GPO sont appliquées, lesquelles sont refusées et pourquoi. Quatre-vingt pour cent des cas de ce chapitre auraient pu être diagnostiqués en moins de deux minutes avec ce seul outil.

---

## :material-table: Index des 15 cas

| # | Titre | Catégorie de cause | Outil clé |
|---|-------|--------------------|-----------|
| 1 | GPO de sécurité non appliquée sur 30 % des postes | Authentification / compte machine | `Test-ComputerSecureChannel` |
| 2 | Logon lent (45 s) après migration Windows 11 | Script non signé / CSE Scripts | `Get-WinEvent` (EventID 4018) |
| 3 | Paramètre de registre "revient" après modification manuelle | GPP Replace — effet tattoo | GPMC — action Update |
| 4 | AppLocker bloque une application interne après mise à jour | Règle Hash invalidée | `Get-AppLockerFileInformation` |
| 5 | Default Domain Policy corrompue | Modification directe de la DDP | `Restore-GPO` / `dcgpofix` |
| 6 | GPO liée mais non appliquée à un utilisateur | Filtrage de sécurité manquant | `gpresult /r` — Denied GPOs |
| 7 | Nouveau DC n'applique pas les GPO correctement | SYSVOL DFS-R non initialisé | `dfsrdiag ReplicationState` |
| 8 | WMI Filter provoque timeout et ralentit tous les logons | Requête `Win32_Product` | Rewrite WQL |
| 9 | BitLocker recovery key non sauvegardée dans AD | GPO appliquée post-chiffrement | `manage-bde -protectors -adbackup` |
| 10 | Redirection de dossiers cassée après renommage serveur | Chemin UNC codé en dur dans GPO | Espace de noms DFS |
| 11 | LAPS password non généré sur un subset de machines | Permission manquante sur `ms-Mcs-AdmPwd` | `Set-AdmPwdComputerSelfPermission` |
| 12 | GPO de firewall bloque RDP après déploiement | Block All sans exception RDP | `netsh advfirewall reset` |
| 13 | Profils itinérants corrompus sur sessions RDS | Déconnexion forcée pendant sauvegarde | FSLogix Profile Container |
| 14 | Migration GPO vers Intune — un paramètre manque | Pas de CSP équivalent (MDM gap) | `Get-MDMGroupPolicyAnalytics` |
| 15 | GPO Enforced écrase une exception métier légitime | Suremploi de Enforced | Filtrage de sécurité + Enforced |


!!! quote "En résumé"
    - 1 : Test-ComputerSecureChannel.
    - 2 : Get-WinEvent (EventID 4018).
    - 3 : GPMC — action Update.
    - 4 : Get-AppLockerFileInformation.
    - Cette synthèse condense index des 15 cas en aide de décision rapide.
---

## :material-wrench-clock: Les 15 cas

=== "Cas 1 — GPO sécurité manquante sur 30 % des postes"

    **Symptôme :** Une GPO de sécurité critique (restrictions USB, Defender config) s'applique sur 70 % des machines du domaine. Les 30 % restants n'ont aucune trace de la GPO dans `gpresult /r`.

    **Environnement :** Domaine AD unique, 400 postes, GPO liée à la racine du domaine, sans WMI Filter ni filtrage de sécurité restrictif.

    **Diagnostic pas à pas :**

    1. Exécutez `gpresult /r /scope computer` sur une machine affectée. La sortie indique `ACCESS DENIED` dans la section RSOP ou la GPO apparaît dans "Denied GPOs" sans raison de filtrage visible.
    2. Consultez l'Observateur d'événements : `Applications and Services Logs > Microsoft > Windows > GroupPolicy > Operational`. Cherchez l'EventID **1030** (accès SYSVOL refusé) ou **1006** (erreur Kerberos).
    3. Testez le canal sécurisé de la machine :

    ```powershell title="Test-SecureChannel.ps1"
    # Test the machine's secure channel to the domain controller
    Test-ComputerSecureChannel -Verbose
    ```

    ```title="Résultat sur machine affectée"
    VERBOSE: Performing the operation "Test-ComputerSecureChannel" on target "WKS-042".
    False
    ```

    4. Confirmez l'expiration du compte machine dans AD :

    ```powershell title="Check-MachineAccount.ps1"
    # Check if machine account password is expired (>30 days default)
    $computer = Get-ADComputer "WKS-042" -Properties PasswordLastSet, Enabled
    $computer | Select-Object Name, PasswordLastSet, Enabled
    ```

    **Cause racine :** Le compte machine Active Directory a un mot de passe expiré ou corrompu. Kerberos ne peut plus authentifier le poste auprès du DC. Le service `gpsvc` n'obtient pas de ticket Kerberos valide et ne peut pas accéder à `\\DOMAIN\SYSVOL`. La GPO ne peut pas être lue.

    !!! warning "Piège classique"
        Les machines déconnectées longtemps (congés, stockage) ou clonées sans `sysprep` sont les premières victimes. Le compte machine n'a pas renouvelé son mot de passe depuis plus de 30 jours.

    **Solution :**

    ```powershell title="Repair-SecureChannel.ps1"
    # Repair the secure channel — must run on the affected machine as local admin
    Test-ComputerSecureChannel -Repair -Credential (Get-Credential "DOMAIN\AdminAccount")

    # Force a GP refresh after repair
    gpupdate /force
    ```

    Si le canal ne peut pas être réparé à distance (machine hors réseau) :

    ```powershell title="Reset-MachineAccountFromDC.ps1"
    # Reset machine account password from a DC (alternative approach)
    Reset-ComputerMachinePassword -Server "DC01.domain.local" -Credential (Get-Credential)
    ```

    **Prévention :** Activez la surveillance automatique du canal sécurisé dans votre supervision. Un script hebdomadaire qui exécute `Test-ComputerSecureChannel` sur l'ensemble du parc et alerte sur les `False` détecte le problème avant que les GPO ne cessent de s'appliquer.

---
=== "Cas 2 — Logon lent (45 s) après migration Windows 11"

    **Symptôme :** Après une migration de Windows 10 vers Windows 11 22H2, les utilisateurs signalent des logons de 40 à 50 secondes. L'écran reste bloqué sur "Applying Group Policy Scripts" pendant toute cette durée.

    **Environnement :** Scripts PowerShell de logon déployés via GPO (Configuration utilisateur > Paramètres Windows > Scripts > Logon). Scripts existants non modifiés depuis 3 ans.

    **Diagnostic pas à pas :**

    1. Consultez le journal GP Operational sur la machine lente :

    ```powershell title="Get-GPScriptsEvent.ps1"
    # Retrieve GP Scripts CSE events — look for 4018 (long processing)
    Get-WinEvent -LogName "Microsoft-Windows-GroupPolicy/Operational" |
        Where-Object { $_.Id -in @(4018, 4016, 4017) } |
        Select-Object TimeCreated, Id, Message |
        Format-List
    ```

    2. L'EventID **4018** indique le temps de traitement du CSE Scripts. Repérez une durée de 40 000 ms ou plus.

    3. Vérifiez la politique d'exécution en vigueur et l'état de signature du script :

    ```powershell title="Check-ScriptSignature.ps1"
    # Check effective execution policy and script signature status
    Get-ExecutionPolicy -List | Format-Table -AutoSize

    # Verify script digital signature
    Get-AuthenticodeSignature -FilePath "\\DOMAIN\SYSVOL\domain.local\scripts\logon.ps1" |
        Select-Object Status, SignerCertificate, Path
    ```

    **Cause racine :** Windows 11 active **DevicePolicyManager** avec des contrôles renforcés sur les scripts de stratégie de groupe. Un script PowerShell non signé déclenche une vérification OCSP/CRL auprès d'une autorité de certification, avec un timeout de 40 secondes si le CA n'est pas joignable au moment du logon (avant que le réseau soit pleinement disponible).

    !!! danger "Production — DevicePolicyManager sous Windows 11"
        Ce comportement est intentionnel et introduit progressivement depuis Windows 11 22H2. Ne pas signer les scripts de logon n'était qu'une "mauvaise pratique" sous Windows 10 — sous Windows 11, c'est un blocage opérationnel.

    **Solution — Option A : signer le script**

    ```powershell title="Sign-LogonScript.ps1"
    # Sign the logon script with an internal code-signing certificate
    $cert = Get-ChildItem Cert:\CurrentUser\My -CodeSigningCert | Select-Object -First 1
    Set-AuthenticodeSignature -FilePath "\\DOMAIN\SYSVOL\domain.local\scripts\logon.ps1" `
                              -Certificate $cert `
                              -TimestampServer "http://timestamp.domain.local"
    ```

    **Solution — Option B : migrer vers GPP Immediate Task**

    Si le script est simple (mapping lecteur, variable d'environnement), remplacez-le par une **GPP Scheduled Task** en mode "Run immediately when task is created" — les tâches GPP ne sont pas soumises à la vérification de signature.

    **Prévention :** Signez tous les scripts GPO lors de leur création. Ajoutez la vérification de signature dans votre pipeline CI/CD GPO. Testez toujours sur Windows 11 avant le déploiement en production.

---
=== "Cas 3 — Paramètre de registre qui « revient » après modif manuelle"

    **Symptôme :** Un utilisateur ou un technicien modifie une valeur de registre sur un poste. 90 minutes plus tard (parfois au redémarrage), la valeur est restaurée à sa valeur d'origine. L'utilisateur signale une GPO "buggée" qui annule ses modifications.

    **Environnement :** Paramètre déployé via **GPP Registry** (Group Policy Preferences, pas une vraie GPO de sécurité), action configurée sur "Replace".

    **Diagnostic pas à pas :**

    1. Identifiez la source du paramètre dans GPMC. Dans l'onglet **Settings** de la GPO, cherchez sous "Registry (preferences)".
    2. Notez l'action configurée pour l'entrée : **Create**, **Replace**, **Update** ou **Delete**.
    3. Vérifiez le cycle de rafraîchissement en arrière-plan (90 min par défaut) :

    ```powershell title="Check-GPRefreshInterval.ps1"
    # Check background refresh interval from registry (in minutes)
    $refreshPath = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\GPExtensions"
    Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\System" `
        -Name GroupPolicyRefreshTime -ErrorAction SilentlyContinue
    ```

    **Cause racine :** L'action **Replace** sur une entrée GPP Registry supprime et recrée la valeur à chaque cycle de rafraîchissement GPO (toutes les 90 minutes en arrière-plan). Ce comportement est documenté et voulu — mais il est contre-intuitif pour les utilisateurs et les techniciens qui s'attendent à ce que les préférences soient des suggestions, pas des ordres.

    !!! tip "GPP : la distinction critique entre les 4 actions"
        | Action | Comportement | Usage |
        |--------|-------------|-------|
        | **Create** | Crée uniquement si absent | Valeur par défaut initiale |
        | **Replace** | Supprime et recrée à chaque cycle | Enforcement strict |
        | **Update** | Crée ou modifie, ne supprime pas | Valeur suggérée modifiable |
        | **Delete** | Supprime la valeur | Nettoyage |

    **Solution :** Dans GPMC, modifiez l'action de l'entrée GPP Registry de **Replace** vers **Update**. Avec Update, la valeur est définie si elle n'existe pas ou est absente, mais une modification manuelle survit au cycle de rafraîchissement suivant.

    Si l'enforcement est intentionnel (valeur de sécurité qui doit toujours être définie), documentez ce comportement et expliquez-le à l'équipe support pour éviter les tickets récurrents.

    **Prévention :** Nommez vos GPP en indiquant l'action : `GPP-Registry-REPLACE-ProxySettings` vs `GPP-Registry-UPDATE-DefaultBrowser`. La distinction est visible dans GPMC et évite les confusions.

---
=== "Cas 4 — AppLocker bloque une appli interne après mise à jour"

    **Symptôme :** Après une mise à jour d'une application interne (v2.1.4 → v2.2.0), AppLocker bloque l'exécution sur tous les postes concernés. L'EventID **8004** apparaît dans `Applications and Services Logs > Microsoft > Windows > AppLocker > EXE and DLL`.

    **Environnement :** AppLocker déployé via GPO, règle de type **Hash** basée sur le hash SHA256 de l'exécutable v2.1.4.

    **Diagnostic pas à pas :**

    1. Consultez les événements AppLocker sur une machine bloquée :

    ```powershell title="Get-AppLockerBlockEvents.ps1"
    # Retrieve AppLocker block events (8004 = blocked, 8003 = would be blocked in audit)
    Get-WinEvent -LogName "Microsoft-Windows-AppLocker/EXE and DLL" |
        Where-Object { $_.Id -eq 8004 } |
        Select-Object TimeCreated,
                      @{N="File";E={$_.Properties[1].Value}},
                      @{N="User";E={$_.Properties[4].Value}} |
        Format-Table -AutoSize
    ```

    2. Identifiez le hash attendu par AppLocker vs le hash réel du nouveau binaire :

    ```powershell title="Get-FileHash-Comparison.ps1"
    # Get hash of the new binary to compare with AppLocker rule
    Get-AppLockerFileInformation -Path "C:\App\Metier\app.exe" | Format-List
    Get-FileHash -Path "C:\App\Metier\app.exe" -Algorithm SHA256
    ```

    **Cause racine :** Une règle **Hash** est liée au hash cryptographique exact du fichier. Toute modification du binaire (mise à jour, recompilation, patch) invalide la règle. Les règles Hash sont fiables mais ne survivent pas aux mises à jour — c'est leur limitation fondamentale.

    !!! warning "Règle Hash vs règle Publisher"
        Une règle Hash nécessite une mise à jour manuelle à chaque version. Une règle Publisher (basée sur le certificat de signature de l'éditeur) survit aux mises à jour tant que l'éditeur ne change pas son certificat.

    **Solution :** Remplacez la règle Hash par une règle **Publisher** :

    ```powershell title="Create-AppLockerPublisherRule.ps1"
    # Extract publisher information from the new executable
    Get-AppLockerFileInformation -Path "C:\App\Metier\app.exe" |
        Select-Object -ExpandProperty Publisher

    # Generate an AppLocker policy based on publisher for the file
    $policy = New-AppLockerPolicy -FileInformation (
        Get-AppLockerFileInformation -Path "C:\App\Metier\app.exe"
    ) -RuleType Publisher -User "Everyone" -RuleNamePrefix "InternalApp"

    # Review the generated XML before applying
    $policy.ToXml()
    ```

    Si l'exécutable n'est pas signé (pas de certificat éditeur disponible), utilisez une règle **Path** avec un chemin fixe plutôt qu'une règle Hash — moins sécurisée mais maintenable.

    **Prévention :** Auditez vos règles AppLocker et identifiez toutes les règles Hash avec `Get-AppLockerPolicy -Effective | Select-Xml "//FileHashRule"`. Migrez-les systématiquement vers des règles Publisher lorsque l'application est signée.

---
=== "Cas 5 — Default Domain Policy corrompue"

    **Symptôme :** Les politiques de mot de passe et de verrouillage de compte ne s'appliquent plus correctement. Certains DCs affichent des paramètres différents. La GPMC affiche des erreurs à l'ouverture de la DDP ou le `versionNumber` est incohérent entre AD et SYSVOL.

    **Environnement :** Domaine mature (8+ ans), plusieurs administrateurs avaient un accès direct à la GPMC. La DDP a été modifiée directement au lieu d'utiliser une GPO dédiée.

    **Diagnostic pas à pas :**

    1. Vérifiez la cohérence de version entre AD et SYSVOL :

    ```powershell title="Check-DDP-Version.ps1"
    # Compare DDP version number in AD vs SYSVOL
    $ddpGuid = "{31B2F340-016D-11D2-945F-00C04FB984F9}"
    $domainDN = (Get-ADDomain).DistinguishedName

    # Version in AD
    $adVersion = (Get-ADObject "CN={$ddpGuid},CN=Policies,CN=System,$domainDN" `
        -Properties versionNumber).versionNumber
    Write-Host "AD version     : $adVersion"

    # Version in SYSVOL (parse GPT.INI)
    $sysvolPath = "\\$env:USERDNSDOMAIN\SYSVOL\$env:USERDNSDOMAIN\Policies\{$ddpGuid}\GPT.INI"
    $sysvolContent = Get-Content $sysvolPath
    Write-Host "SYSVOL content : $sysvolContent"
    ```

    2. Exportez la DDP actuelle avant toute intervention :

    ```powershell title="Backup-DDP.ps1"
    # Always backup before any remediation
    $backupPath = "C:\GPO_Backups\$(Get-Date -Format 'yyyyMMdd_HHmmss')"
    New-Item -ItemType Directory -Path $backupPath -Force | Out-Null
    Backup-GPO -Name "Default Domain Policy" -Path $backupPath
    Write-Host "Backup saved to: $backupPath"
    ```

    **Cause racine :** Des modifications directes et répétées de la DDP ont désynchronisé le `versionNumber` entre l'objet AD et le fichier `GPT.INI` dans SYSVOL. Les DCs ne savent plus quelle version est authoritative. Dans les cas graves, des paramètres ont été supprimés ou écrasés par des modifications concurrentes.

    !!! danger "Production — Ne jamais modifier la DDP directement"
        La **Default Domain Policy** et la **Default Domain Controllers Policy** ne doivent contenir que les paramètres qu'elles définissent par défaut (politique de mot de passe, audit de base). Tout le reste doit être dans des GPO dédiées. C'est une règle Microsoft et une leçon apprise par tous ceux qui ont corrompu une DDP en production.

    **Solution — Option A : restaurer depuis la sauvegarde**

    ```powershell title="Restore-DDP.ps1"
    # Restore Default Domain Policy from backup
    $backupPath = "C:\GPO_Backups\20240115_143022"  # use your actual backup path

    Restore-GPO -Name "Default Domain Policy" -Path $backupPath
    ```

    **Solution — Option B : recréer depuis zéro (dernier recours)**

    ```powershell title="Reset-DDP-LastResort.ps1"
    # LAST RESORT ONLY — dcgpofix recreates DDP and DDCP to factory defaults
    # This REMOVES all customizations — document what you need to restore first
    dcgpofix /ignoreschema /target:Domain
    ```

    !!! danger "Production — dcgpofix"
        `dcgpofix /target:Domain` remet la DDP **exactement** aux valeurs d'usine. Tous vos paramètres personnalisés (politique de mot de passe fine, restrictions, audit) sont effacés. Documentez la configuration existante avant d'exécuter cette commande.

    **Prévention :** Verrouillez la DDP avec une politique de délégation : retirez les droits "Edit settings" de tous les comptes sauf un compte de service dédié. Créez une GPO `CORP-Security-PasswordPolicy` séparée pour tout ajout.

---
=== "Cas 6 — GPO liée mais non appliquée à un utilisateur"

    **Symptôme :** Un utilisateur spécifique ne reçoit pas les paramètres d'une GPO pourtant liée à son OU. Ses collègues dans la même OU l'ont. `gpresult /r /user DOMAIN\username` ne montre pas la GPO dans la liste appliquée.

    **Environnement :** GPO configurée avec filtrage de sécurité personnalisé (groupe spécifique au lieu de "Authenticated Users").

    **Diagnostic pas à pas :**

    1. Exécutez `gpresult` pour l'utilisateur cible et cherchez la section "Denied GPOs" :

    ```powershell title="Get-DeniedGPOs.ps1"
    # Get full RSOP report and save to HTML for analysis
    gpresult /h "C:\Temp\gpresult-$(whoami -replace '\\','-').html" /f

    # Or quick console output for the specific user
    gpresult /r /user "DOMAIN\username" /scope user
    ```

    2. La sortie indique la GPO dans "Denied GPOs" avec le motif **"Inaccessible"** ou **"Denied (Security)"**.

    3. Vérifiez les permissions de la GPO :

    ```powershell title="Check-GPO-Permissions.ps1"
    # List all security permissions on the GPO
    $gpoName = "Corp-Software-Deploy"
    Get-GPPermission -Name $gpoName -All | Format-Table -AutoSize

    # Check if a specific user has Apply Group Policy permission
    Get-GPPermission -Name $gpoName -TargetName "DOMAIN\username" -TargetType User `
        -ErrorAction SilentlyContinue
    ```

    **Cause racine :** Le filtrage de sécurité de la GPO a été modifié pour s'appliquer à un groupe spécifique (`GRP-GPO-Deploy` par exemple). L'utilisateur n'est pas membre de ce groupe. Sans la permission "Apply Group Policy", la GPO est évaluée mais refusée lors de l'application.

    !!! tip "Diagnostic rapide"
        Dans GPMC, onglet **Scope** de la GPO, section **Security Filtering** : si vous voyez un groupe au lieu de "Authenticated Users", vérifiez immédiatement si l'utilisateur affecté est membre de ce groupe.

    **Solution :**

    ```powershell title="Fix-GPO-SecurityFilter.ps1"
    # Add the user or their group to the GPO security filter
    # Option A: Add the user directly (not recommended for large environments)
    Set-GPPermission -Name "Corp-Software-Deploy" `
                     -TargetName "DOMAIN\username" `
                     -TargetType User `
                     -PermissionLevel GpoApply

    # Option B: Add the user to the existing group (preferred)
    Add-ADGroupMember -Identity "GRP-GPO-Deploy" -Members "username"
    # Token refresh required — user must log off and log on again
    ```

    **Prévention :** Documentez chaque GPO avec filtrage de sécurité non standard dans votre registre GPO. Ajoutez un commentaire GPMC (`Set-GPO -Name "..." -Comment "..."`) indiquant le groupe ciblé et la raison du filtrage.

---
=== "Cas 7 — Nouveau DC n'applique pas les GPO correctement"

    **Symptôme :** Un nouveau contrôleur de domaine vient d'être promu. Les postes qui l'utilisent comme DC préféré ne reçoivent pas les GPO, ou reçoivent une version obsolète. L'Observateur d'événements montre l'EventID **4602** absent du journal DFS Replication.

    **Environnement :** Nouveau DC ajouté dans un domaine existant, migration de FRS vers DFS-R en cours ou récemment terminée.

    **Diagnostic pas à pas :**

    1. Vérifiez l'état de la réplication DFS-R sur le nouveau DC :

    ```powershell title="Check-DFSR-State.ps1"
    # Check DFS-R replication state for SYSVOL
    dfsrdiag ReplicationState /member:NEWDC01

    # Check if SYSVOL is shared and accessible
    net share | findstr /i "sysvol netlogon"
    ```

    2. Vérifiez l'état global de la migration SYSVOL :

    ```powershell title="Check-SYSVOL-Migration.ps1"
    # Check global migration state (3 = Eliminated / DFS-R active)
    dfsrmig /getglobalstate

    # Check per-DC migration state
    dfsrmig /getmigrationstate
    ```

    3. Cherchez l'EventID 4602 dans le journal DFS Replication (indique que DFS-R a initialisé SYSVOL depuis un partenaire) :

    ```powershell title="Check-DFSR-Events.ps1"
    # Look for EventID 4602 (SYSVOL initialized) and 4604 (replication started)
    Get-WinEvent -LogName "DFS Replication" |
        Where-Object { $_.Id -in @(4602, 4604, 4614) } |
        Select-Object TimeCreated, Id, Message |
        Format-List
    ```

    **Cause racine :** Le nouveau DC n'a pas encore reçu l'initialisation DFS-R complète de SYSVOL. L'EventID 4602 ("DFS Replication initialized SYSVOL") est absent, ce qui signifie que le contenu SYSVOL (dont les GPO) n'a pas encore été répliqué depuis les DCs existants.

    !!! warning "FRS vs DFS-R"
        Si le domaine utilise encore FRS (File Replication Service), le diagnostic est différent : vérifiez avec `repadmin /showrepl` et consultez le journal FRS dans l'Observateur d'événements. FRS est obsolète depuis Windows Server 2008 R2 — planifiez la migration vers DFS-R.

    **Solution :**

    ```powershell title="Force-DFSR-Sync.ps1"
    # Force DFS-R to sync SYSVOL from an authoritative partner
    # Step 1: Set the authoritative DC
    dfsrmig /setglobalstate 3

    # Step 2: Force replication from a known-good DC
    repadmin /syncall NEWDC01 /AdeP

    # Step 3: Verify SYSVOL is now shared (may take 5-10 minutes)
    dfsrdiag ReplicationState /member:NEWDC01

    # Step 4: Check SYSVOL share is accessible
    Test-Path "\\NEWDC01\SYSVOL"
    ```

    **Prévention :** Après chaque promotion de DC, attendez la confirmation de l'EventID 4602 avant de router des clients vers ce DC. Ajoutez une vérification `Test-Path "\\NEWDC\SYSVOL"` dans votre runbook de promotion.

---
=== "Cas 8 — WMI Filter provoque timeout et ralentit les logons"

    **Symptôme :** Depuis l'ajout d'un WMI Filter sur une GPO, les logons prennent 60 à 90 secondes supplémentaires sur toutes les machines ciblées par cette GPO — pas seulement celles où le filtre devrait retourner False.

    **Environnement :** WMI Filter créé pour cibler uniquement les machines ayant une application spécifique installée. La requête WQL utilise `Win32_Product`.

    **Diagnostic pas à pas :**

    1. Identifiez la requête WQL dans le WMI Filter (GPMC > WMI Filters) :

    ```wql
    SELECT * FROM Win32_Product WHERE Name = "MyApp"
    ```

    2. Mesurez le temps d'exécution de la requête WMI sur une machine cible :

    ```powershell title="Test-WMIFilterPerf.ps1"
    # Measure execution time of the WMI query used in the filter
    Measure-Command {
        Get-WmiObject -Query "SELECT * FROM Win32_Product WHERE Name = 'MyApp'"
    }

    # Compare with the alternative query
    Measure-Command {
        Get-WmiObject -Query "SELECT * FROM Win32_InstalledWin32Program WHERE Name = 'MyApp'"
    }
    ```

    3. Comparez les temps. `Win32_Product` retourne souvent 30 à 60 secondes. `Win32_InstalledWin32Program` retourne en moins d'une seconde.

    **Cause racine :** La classe WMI `Win32_Product` est notoire pour son comportement destructeur : chaque requête déclenche une **vérification d'intégrité MSI** (`msiexec /fp`) sur tous les packages installés. Sur un poste avec 50 applications, cette opération prend 30 à 60 secondes et peut même déclencher des réparations de logiciels non désirées.

    !!! danger "Production — Win32_Product interdit en production"
        N'utilisez jamais `Win32_Product` dans un WMI Filter GPO, un script de logon, ou un outil de supervision. Cette classe WMI est documentée par Microsoft comme ne devant pas être utilisée pour des requêtes d'inventaire — elle reconfigure les MSI à chaque interrogation. Des cas de réparations intempestives de logiciels en pleine journée ont été causés par des requêtes `Win32_Product` lancées depuis SCCM.

    **Solution :** Remplacez la requête WQL par une alternative basée sur le registre :

    ```powershell title="Better-WMI-Filter-Options.ps1"
    # Option A: Win32_InstalledWin32Program (faster, no MSI reconfiguration)
    # WQL: SELECT * FROM Win32_InstalledWin32Program WHERE Name = 'MyApp'

    # Option B: Registry-based detection (fastest — direct registry query)
    # WQL: SELECT * FROM RegistryValueChangeEvent ...
    # Better: Use ILT (Item-Level Targeting) in GPP instead of WMI Filter

    # Option C: Check registry directly in WMI Filter
    # WQL query for registry key existence:
    $wqlQuery = @"
    SELECT * FROM Win32_RegistryKey
    WHERE Hive = 'HKEY_LOCAL_MACHINE'
    AND KeyPath = 'SOFTWARE\\MyApp'
    "@
    Get-WmiObject -Query $wqlQuery
    ```

    **Prévention :** Banissez `Win32_Product` de tous vos WMI Filters et scripts. Préférez le ciblage au niveau des éléments GPP (Item-Level Targeting) qui offre des conditions similaires sans requête WMI coûteuse.

---
=== "Cas 9 — BitLocker recovery key non sauvegardée dans AD"

    **Symptôme :** Un poste BitLocker demande la clé de récupération au démarrage. La clé n'est pas dans AD DS (attribut `msFVE-RecoveryPassword` absent). La GPO BitLocker est bien appliquée selon `gpresult`.

    **Environnement :** GPO BitLocker déployée après que les postes ont déjà été chiffrés (chiffrement manuel ou via un script précédent).

    **Diagnostic pas à pas :**

    1. Vérifiez si la clé existe dans AD :

    ```powershell title="Find-BitLockerKeyInAD.ps1"
    # Search for BitLocker recovery key in AD for a specific computer
    $computerName = "WKS-042"
    $computer = Get-ADComputer $computerName

    Get-ADObject -Filter { objectClass -eq "msFVE-RecoveryInformation" } `
                 -SearchBase $computer.DistinguishedName `
                 -Properties msFVE-RecoveryPassword, msFVE-RecoveryGuid |
        Select-Object Name, msFVE-RecoveryPassword
    ```

    2. Sur la machine cible, listez les protecteurs BitLocker actuels :

    ```powershell title="Get-BitLockerProtectors.ps1"
    # List all BitLocker key protectors on the system drive
    manage-bde -protectors -get C:
    ```

    3. Notez le GUID du protecteur de type "RecoveryPassword".

    **Cause racine :** La GPO BitLocker configure l'escrow automatique des clés vers AD — mais uniquement pour les **nouveaux chiffrements** effectués après l'application de la GPO. Les postes déjà chiffrés avant le déploiement de la GPO n'ont pas re-soumis leurs clés de récupération existantes. L'escrow n'est pas rétroactif.

    !!! warning "Piège de déploiement BitLocker"
        Déployer une GPO BitLocker sur un parc existant ne garantit pas que les clés sont dans AD. Vérifiez systématiquement l'attribut `msFVE-RecoveryPassword` sur un échantillon représentatif après le déploiement.

    **Solution — Pour les postes joints à AD on-prem :**

    ```powershell title="Backup-BitLockerKeyToAD.ps1"
    # Get the recovery key protector ID
    $keyId = (Get-BitLockerVolume -MountPoint C:).KeyProtector |
        Where-Object { $_.KeyProtectorType -eq "RecoveryPassword" } |
        Select-Object -ExpandProperty KeyProtectorId

    # Back up the key to AD DS
    manage-bde -protectors -adbackup C: -id $keyId

    # Verify the backup succeeded (check AD)
    Backup-BitLockerKeyProtector -MountPoint "C:" -KeyProtectorId $keyId
    ```

    **Solution — Pour les postes joints à Azure AD / Hybrid :**

    ```powershell title="Backup-BitLockerKeyToAAD.ps1"
    # Back up BitLocker key to Azure Active Directory
    $keyId = (Get-BitLockerVolume -MountPoint C:).KeyProtector |
        Where-Object { $_.KeyProtectorType -eq "RecoveryPassword" } |
        Select-Object -ExpandProperty KeyProtectorId

    BackupToAAD-BitLockerKeyProtector -MountPoint "C:" -KeyProtectorId $keyId
    ```

    **Prévention :** Créez un script de remédiation de masse qui parcourt tous les postes, vérifie si `msFVE-RecoveryPassword` est vide dans AD, et effectue le backup automatiquement. Planifiez ce script en tâche GPP sur le parc entier.

---
=== "Cas 10 — Redirection de dossiers cassée après renommage serveur"

    **Symptôme :** Après le renommage ou le remplacement d'un serveur de fichiers, les utilisateurs ne voient plus leurs documents habituels. La GPO de redirection de dossiers pointe vers `\\OLDSERVER\Profiles$\%USERNAME%`. Le nouveau serveur se nomme `NEWSERVER`.

    **Environnement :** Redirection de dossiers (Documents, Desktop) configurée avec un chemin UNC statique dans la GPO. Pas d'espace de noms DFS en place.

    **Diagnostic pas à pas :**

    1. Identifiez le chemin actuel dans la GPO :

    ```powershell title="Get-FolderRedirectionPath.ps1"
    # Get folder redirection settings from the GPO (read GPO XML report)
    $gpoName = "Corp-FolderRedirection"
    $report = Get-GPOReport -Name $gpoName -ReportType Xml

    # Parse the XML to find redirect paths
    [xml]$xml = $report
    $xml.GPO.User.ExtensionData | Where-Object { $_.Name -like "*Folder*" }
    ```

    2. Vérifiez si les dossiers utilisateurs existent encore sur l'ancien chemin.
    3. Vérifiez l'accès au nouveau serveur depuis un poste client.

    **Cause racine :** Le chemin UNC statique dans la GPO est lié à un nom de serveur physique. Tout renommage, remplacement ou déplacement du serveur invalide immédiatement les chemins pour tous les utilisateurs actifs. Les documents sont toujours présents sur l'ancien chemin mais inaccessibles si le serveur a été retiré.

    !!! danger "Production — UNC statiques dans les GPO de redirection"
        Un chemin `\\SERVEUR\Partage` dans une GPO de redirection de dossiers est une bombe à retardement. La moindre opération d'infrastructure (renommage, migration, virtualisation) crée une rupture de service immédiate pour tous les utilisateurs.

    **Solution — Procédure de migration :**

    ```powershell title="Migrate-FolderRedirection.ps1"
    # Step 1: Copy data from old to new server with full fidelity
    robocopy "\\OLDSERVER\Profiles$" "\\NEWSERVER\Profiles$" `
        /E /COPYALL /SEC /LOG:"C:\Temp\robocopy-profiles.log" /TEE /NP

    # Step 2: Create DFS namespace (prevents future recurrence)
    # Install DFS role if needed
    Install-WindowsFeature -Name FS-DFS-Namespace, FS-DFS-Replication, RSAT-DFS-Mgmt-Con

    # Create the namespace
    New-DfsnRoot -TargetPath "\\NEWSERVER\Profiles$" `
                 -Path "\\domain.local\Profiles" `
                 -Type DomainV2

    # Step 3: Update GPO to use DFS namespace path
    # Change \\OLDSERVER\Profiles$ to \\domain.local\Profiles in the GPO
    # Do this in GPMC: User Config > Policies > Windows Settings > Folder Redirection
    ```

    ```powershell title="Verify-Migration.ps1"
    # Verify users can access their documents via DFS path
    $testUsers = Get-ADUser -Filter * -SearchBase "OU=Users,DC=domain,DC=local" |
        Select-Object -First 10 -ExpandProperty SamAccountName

    foreach ($user in $testUsers) {
        $path = "\\domain.local\Profiles\$user\Documents"
        [PSCustomObject]@{
            User      = $user
            PathExists = Test-Path $path
            ItemCount  = (Get-ChildItem $path -ErrorAction SilentlyContinue | Measure-Object).Count
        }
    }
    ```

    **Prévention :** N'utilisez jamais de nom de serveur physique dans une GPO de redirection. Créez systématiquement un espace de noms DFS (`\\domain.local\Profiles`) et pointez-y la GPO. Le jour d'une migration serveur, vous n'aurez qu'à mettre à jour la cible DFS — sans toucher à la GPO.

---
=== "Cas 11 — LAPS password non généré sur un subset de machines"

    **Symptôme :** LAPS est déployé via GPO et fonctionne sur 80 % du parc. Sur les machines d'une OU spécifique, l'attribut `ms-Mcs-AdmPwd` reste vide. L'extension LAPS est bien installée sur ces machines.

    **Environnement :** LAPS (Legacy LAPS avec l'extension CSE `AdmPwd.dll`), OU ciblée récemment créée par déplacement de machines depuis une autre OU.

    **Diagnostic pas à pas :**

    1. Vérifiez que la machine tente d'écrire la valeur (journal d'événements Application) :

    ```powershell title="Get-LAPSEvents.ps1"
    # Check LAPS events in Application log (source: AdmPwd)
    Get-WinEvent -LogName Application |
        Where-Object { $_.ProviderName -eq "AdmPwd" } |
        Select-Object TimeCreated, Id, Message |
        Format-List
    ```

    2. Vérifiez les permissions AD sur l'OU cible :

    ```powershell title="Check-LAPS-Permissions.ps1"
    # Import the LAPS PowerShell module
    Import-Module AdmPwd.PS

    # Find OUs where LAPS self-write permission is missing
    Find-AdmPwdExtendedRights -Identity "OU=NewOU,DC=domain,DC=local"
    ```

    3. Comparez avec une OU fonctionnelle :

    ```powershell title="Compare-LAPS-Permissions.ps1"
    # Check self-permission on working OU
    Find-AdmPwdExtendedRights -Identity "OU=WorkingOU,DC=domain,DC=local"

    # Check self-permission on broken OU
    Find-AdmPwdExtendedRights -Identity "OU=NewOU,DC=domain,DC=local"
    ```

    **Cause racine :** La commande `Set-AdmPwdComputerSelfPermission` n'a pas été exécutée sur la nouvelle OU. Cette commande accorde aux comptes machine le droit d'écriture sur leur propre attribut `ms-Mcs-AdmPwd` — sans elle, le CSE LAPS tente d'écrire le mot de passe mais obtient "Access Denied" silencieusement.

    !!! tip "Nouvelle OU = nouvelle permission LAPS"
        Chaque nouvelle OU qui doit bénéficier de LAPS nécessite l'exécution de `Set-AdmPwdComputerSelfPermission`. Cette étape est souvent oubliée lors des réorganisations d'OU. Ajoutez-la à votre checklist de création d'OU.

    **Solution :**

    ```powershell title="Fix-LAPS-Permissions.ps1"
    Import-Module AdmPwd.PS

    # Grant self-write permission on the target OU
    Set-AdmPwdComputerSelfPermission -OrgUnit "OU=NewOU,DC=domain,DC=local"

    # Verify the permission was applied
    Find-AdmPwdExtendedRights -Identity "OU=NewOU,DC=domain,DC=local"

    # Force LAPS to generate password on affected machines
    # (trigger GP refresh — LAPS generates on next cycle)
    Invoke-Command -ComputerName (
        Get-ADComputer -Filter * -SearchBase "OU=NewOU,DC=domain,DC=local" |
        Select-Object -ExpandProperty Name
    ) -ScriptBlock { gpupdate /force } -ErrorAction SilentlyContinue
    ```

    **Prévention :** Créez un script de vérification hebdomadaire qui liste toutes les OU contenant des objets Computer et vérifie que `Set-AdmPwdComputerSelfPermission` a été appliqué. Intégrez l'exécution de cette commande dans votre procédure standard de création d'OU.

---
=== "Cas 12 — GPO de firewall bloque RDP après déploiement"

    **Symptôme :** Après le déploiement d'une nouvelle GPO de pare-feu Windows Defender, les administrateurs ne peuvent plus se connecter en RDP aux serveurs membres. L'accès est impossible à distance — y compris pour corriger la GPO.

    **Environnement :** GPO "Block All Inbound" déployée en urgence sur une OU de serveurs sans audit préalable. Aucune règle d'exception explicite pour RDP.

    **Diagnostic pas à pas :**

    1. Vérifiez l'état du pare-feu depuis la console du serveur (accès physique ou console iDRAC/ILO) :

    ```powershell title="Check-Firewall-State.ps1"
    # Check current firewall profile status and rules
    Get-NetFirewallProfile | Select-Object Name, Enabled, DefaultInboundAction

    # List all inbound Allow rules for RDP
    Get-NetFirewallRule -Direction Inbound -Enabled True |
        Where-Object { $_.DisplayName -like "*Remote Desktop*" -or $_.DisplayName -like "*RDP*" } |
        Select-Object DisplayName, Action, Enabled
    ```

    2. Identifiez si la règle vient d'une GPO ou d'un paramètre local :

    ```powershell title="Check-Firewall-PolicySource.ps1"
    # Check if firewall policy is being enforced by GPO
    netsh advfirewall show allprofiles | findstr /i "policy"
    ```

    **Cause racine :** La GPO configure le profil de pare-feu en **Block All Inbound** sans créer d'exception pour RDP (port TCP 3389). Déployer en mode "enforced" sans audit préalable — ni liste blanche des flux entrants légitimes — bloque instantanément la capacité d'administration à distance.

    !!! danger "Production — Firewall GPO sans audit préalable"
        Ne déployez jamais une GPO de pare-feu en mode Block sans avoir d'abord activé l'audit pendant 48 heures minimum et analysé les connexions entrantes légitimes. Un déploiement aveugle peut rendre un parc entier inaccessible à distance.

    **Solution — Récupération d'urgence :**

    ```powershell title="Emergency-Firewall-Recovery.ps1"
    # EMERGENCY: Run on the console of the affected server
    # Option A: Reset to defaults (removes GPO firewall settings too)
    netsh advfirewall reset

    # Option B: Add RDP allow rule without removing other settings
    netsh advfirewall firewall add rule `
        name="Emergency-RDP-Allow" `
        dir=in `
        action=allow `
        protocol=TCP `
        localport=3389 `
        remoteip=10.0.0.0/8
    ```

    **Solution — GPO corrective (appliquer via une GPO de priorité plus haute) :**

    ```powershell title="Create-EmergencyRDP-GPO.ps1"
    # Create an emergency GPO to allow RDP — link at domain level with higher priority
    $gpo = New-GPO -Name "EMERGENCY-Allow-RDP"

    # Set the inbound RDP firewall rule via registry
    Set-GPRegistryValue -Name "EMERGENCY-Allow-RDP" `
        -Key "HKLM\SOFTWARE\Policies\Microsoft\WindowsFirewall\FirewallRules" `
        -ValueName "Emergency-RDP-In" `
        -Type String `
        -Value "v2.31|Action=Allow|Active=TRUE|Dir=In|Protocol=6|LPort=3389|Name=Emergency-RDP|"

    New-GPLink -Name "EMERGENCY-Allow-RDP" -Target "OU=Servers,DC=domain,DC=local" -LinkEnabled Yes
    ```

    **Prévention :** Utilisez systématiquement un mode **Audit Only** (log uniquement, pas de block) pendant 48 heures avant de passer en Block. Intégrez toujours une règle Allow RDP explicite pour les plages IP d'administration avant tout déploiement de GPO firewall.

---
=== "Cas 13 — Profils itinérants corrompus sur sessions RDS"

    **Symptôme :** Des utilisateurs RDS ne peuvent plus ouvrir de session ou obtiennent un profil temporaire. L'Observateur d'événements montre l'EventID **1509** ("Windows cannot copy file") ou **1511** ("Windows cannot find the local profile"). L'ancienne session s'est terminée par une déconnexion forcée.

    **Environnement :** Profils itinérants (`roaming profiles`) stockés sur un serveur de fichiers, serveur RDS sous Windows Server 2019.

    **Diagnostic pas à pas :**

    1. Identifiez le profil corrompu dans le journal des événements :

    ```powershell title="Get-ProfileErrors.ps1"
    # Find profile-related errors in the Application log
    Get-WinEvent -LogName Application |
        Where-Object { $_.Id -in @(1509, 1511, 1515, 1521) } |
        Select-Object TimeCreated, Id, Message |
        Format-List
    ```

    2. Vérifiez l'état du fichier `ntuser.dat` du profil affecté :

    ```powershell title="Check-ProfileFiles.ps1"
    # List profile files including ntuser.dat and transaction logs
    $profilePath = "\\FILESERVER\Profiles$\username"
    Get-ChildItem $profilePath -Force |
        Where-Object { $_.Name -like "ntuser*" } |
        Select-Object Name, Length, LastWriteTime
    ```

    3. La présence de `ntuser.dat.LOG1` ou `ntuser.dat.LOG2` avec une taille anormale indique une transaction de registre non appliquée — signe de corruption.

    **Cause racine :** Une déconnexion forcée (coupure réseau, timeout, kill session) pendant la sauvegarde du profil laisse `ntuser.dat` dans un état intermédiaire. Les fichiers de transaction `.LOG1`/`.LOG2` contiennent des modifications non commitées. Windows ne peut plus charger le profil correctement.

    !!! warning "Profils itinérants sur RDS : un héritage à remplacer"
        Les profils itinérants classiques sont fragiles par design : une seule session mal terminée suffit à les corrompre. Sur RDS, FSLogix Profile Container est la solution recommandée par Microsoft depuis 2019 — le profil est un VHD(X) monté, pas une copie réseau.

    **Solution — Récupération immédiate :**

    ```powershell title="Fix-CorruptProfile.ps1"
    # Step 1: Rename the corrupt profile (do NOT delete — user data is inside)
    $oldProfile = "\\FILESERVER\Profiles$\username"
    $backupProfile = "\\FILESERVER\Profiles$\username.corrupt_$(Get-Date -Format 'yyyyMMdd')"
    Rename-Item -Path $oldProfile -NewName $backupProfile

    # Step 2: Create a new empty profile directory
    New-Item -ItemType Directory -Path $oldProfile

    # Step 3: The user will get a fresh profile on next logon
    # Step 4: After confirmation the new profile works, migrate user data manually
    ```

    **Solution — Migration vers FSLogix (recommandé) :**

    ```powershell title="Deploy-FSLogix-GPO.ps1"
    # Key FSLogix registry settings to deploy via GPO
    # HKLM\SOFTWARE\FSLogix\Profiles
    $fslogixSettings = @{
        Enabled          = 1
        VHDLocations     = "\\FILESERVER\FSLogix\%username%"
        DeleteLocalProfileWhenVHDShouldApply = 1
        FlipFlopProfileDirectoryName         = 1
        SizeInMBs        = 30000
    }
    # Deploy these via GPP Registry with action Replace
    ```

    **Prévention :** Migrez vers FSLogix Profile Container sur tous les environnements RDS. Activez les sessions fantômes pour pouvoir reprendre les sessions déconnectées sans corruption. Configurez un timeout de déconnexion raisonnable (30 minutes) plutôt qu'infini.

---
=== "Cas 14 — Migration GPO vers Intune — un paramètre manque"

    **Symptôme :** Lors d'une migration de GPO vers Intune (via l'outil Group Policy Analytics), un paramètre spécifique est signalé comme n'ayant pas d'équivalent MDM. Ce paramètre est pourtant critique pour la sécurité ou la conformité.

    **Environnement :** Migration hybride AD + Intune, utilisation de l'outil d'analyse GPMC dans Intune (Group Policy Analytics).

    **Diagnostic pas à pas :**

    1. Analysez les GPO dans Intune pour identifier les paramètres sans CSP :

    ```powershell title="Get-MDMGapAnalysis.ps1"
    # Export GPO to XML for analysis
    $gpoName = "Corp-Security-Baseline"
    $exportPath = "C:\Temp\$gpoName.xml"
    Get-GPOReport -Name $gpoName -ReportType Xml -Path $exportPath

    # Use the Intune Group Policy Analytics API to identify MDM gaps
    # (requires Microsoft.Graph module)
    Import-Module Microsoft.Graph.Beta.DeviceManagement

    # Upload GPO XML to Intune for analysis
    # Navigate to: Intune > Devices > Group Policy Analytics > Import
    ```

    2. Identifiez spécifiquement les paramètres en statut "Not supported" ou "Deprecated" :

    ```powershell title="Analyze-MDMSupport.ps1"
    # Use the built-in cmdlet to analyze GPO migration readiness
    # Requires Windows 10/11 with Intune management tools
    $result = Get-MDMGroupPolicyAnalytics -GroupPolicyObjectId (
        Get-GPO -Name "Corp-Security-Baseline"
    ).Id

    # Filter for unsupported settings
    $result | Where-Object { $_.MigrationReadiness -ne "Supported" } |
        Select-Object SettingName, MigrationReadiness, Notes |
        Format-Table -AutoSize
    ```

    3. Documentez chaque paramètre sans équivalent MDM avec son chemin de registre, sa valeur et sa justification métier.

    **Cause racine :** Tous les paramètres GPO n'ont pas d'équivalent CSP (Configuration Service Provider) dans le monde MDM. Microsoft comble progressivement ces lacunes, mais en 2024-2025, certains paramètres administratifs avancés (pilotes de périphériques spécifiques, paramètres réseau bas niveau, certaines politiques de sécurité Legacy) n'ont pas encore de CSP correspondant.

    !!! tip "MDM gap : ni bug, ni échec"
        Un paramètre sans équivalent MDM n'est pas un obstacle à la migration — c'est une décision d'architecture. Documentez-le, maintenez la GPO pour ce paramètre précis, et révisez à chaque mise à jour Windows.

    **Solution :** Adoptez une approche hybride documentée :

    ```powershell title="Document-MDMGap.ps1"
    # Document MDM gaps in a tracking CSV
    $gapRecord = [PSCustomObject]@{
        Date           = Get-Date -Format "yyyy-MM-dd"
        GPOName        = "Corp-Security-Baseline"
        SettingPath    = "Computer Config\Admin Templates\System\..."
        SettingName    = "SpecificSetting"
        RegistryPath   = "HKLM\SOFTWARE\Policies\..."
        RegistryValue  = "1"
        MDMStatus      = "No CSP equivalent"
        Decision       = "Keep as GPO — review Q1 2025"
        Owner          = "Security Team"
    }

    $gapRecord | Export-Csv "C:\Admin\MDM-Gap-Register.csv" -Append -NoTypeInformation
    ```

    - Conservez la GPO source uniquement pour les paramètres sans CSP.
    - Migrez tout le reste vers des profils de configuration Intune.
    - Révisez le registre des gaps à chaque release majeure de Windows.

    **Prévention :** Avant de planifier une migration GPO → Intune, exécutez systematiquement Group Policy Analytics sur l'ensemble de votre parc GPO. Identifiez le pourcentage de couverture MDM et planifiez le traitement des gaps en amont — pas en cours de migration.

---
=== "Cas 15 — GPO Enforced écrase une exception métier légitime"

    **Symptôme :** Une GPO de sécurité marquée **Enforced** empêche une OU métier d'appliquer son exception locale. L'équipe métier a une GPO qui désactive un paramètre bloquant pour leur application spécifique — mais l'Enforced écrase tout.

    **Environnement :** GPO de sécurité liée à la racine du domaine avec Enforced activé. OU métier avec une GPO locale en conflit. Les deux GPO configurent le même paramètre de registre avec des valeurs opposées.

    **Diagnostic pas à pas :**

    1. Identifiez les GPO en conflit dans `gpresult` :

    ```powershell title="Analyze-GPO-Conflict.ps1"
    # Get full RSOP data including winning GPO for each setting
    $rsop = Get-GPResultantSetOfPolicy -Computer "WKS-METIER-01" -ReportType Xml
    # Or use command-line
    gpresult /h "C:\Temp\rsop-conflict.html" /f /scope user
    ```

    2. Dans le rapport HTML, cherchez les paramètres en conflit : la colonne "Winning GPO" indique quelle GPO a remporté le conflit et pourquoi.

    3. Vérifiez l'état Enforced et Block Inheritance dans la hiérarchie :

    ```powershell title="Check-Enforced-Links.ps1"
    # List all GPO links with their Enforced status
    $domain = Get-ADDomain
    $allLinks = Get-GPInheritance -Target $domain.DistinguishedName

    $allLinks.GpoLinks | Select-Object DisplayName, Enabled, Enforced, Order |
        Sort-Object Order | Format-Table -AutoSize
    ```

    **Cause racine :** L'utilisation de **Enforced** (anciennement "No Override") a été généralisée par l'équipe sécurité pour garantir l'application de leurs paramètres. Mais certaines exceptions métier légitimes — une application qui nécessite l'activation d'un protocole spécifique, une configuration réseau particulière — sont bloquées par cette rigidité.

    !!! warning "Enforced : outil chirurgical, pas marteau"
        Enforced doit être utilisé pour des paramètres véritablement non négociables (politique de mot de passe, chiffrement). L'appliquer systématiquement à toutes les GPO de sécurité crée des conflits légitimes et pousse les équipes métier à trouver des contournements (scripts locaux, modifications manuelles) qui sont pires sur le plan de la sécurité.

    **Solution : Enforced + filtrage de sécurité**

    La combinaison Enforced + Security Filtering permet d'exclure des machines ou utilisateurs spécifiques de l'application d'une GPO Enforced :

    ```powershell title="Configure-EnforcedWithException.ps1"
    # Step 1: Create a security group for the exception
    New-ADGroup -Name "GRP-Exception-AppMetier" `
                -GroupScope Global `
                -GroupCategory Security `
                -Description "Machines excluded from GPO-Security-Enforced for AppMetier"

    # Add exception machines to the group
    Add-ADGroupMember -Identity "GRP-Exception-AppMetier" `
                      -Members (Get-ADComputer -Filter { Name -like "METIER-*" })

    # Step 2: Add DENY "Apply Group Policy" permission to the exception group on the Enforced GPO
    # Note: DENY on Apply Group Policy overrides Enforced for members of this group
    Set-GPPermission -Name "Corp-Security-Enforced-Policy" `
                     -TargetName "GRP-Exception-AppMetier" `
                     -TargetType Group `
                     -PermissionLevel GpoDenyApply

    # Step 3: Document the exception
    Set-GPO -Name "Corp-Security-Enforced-Policy" `
            -Comment "Enforced. Exception: GRP-Exception-AppMetier (AppMetier compatibility). Review: Q2 2025."
    ```

    ```powershell title="Verify-Exception.ps1"
    # Verify the exception group members are excluded from the GPO
    Get-GPPermission -Name "Corp-Security-Enforced-Policy" -All |
        Where-Object { $_.Permission -eq "GpoDenyApply" } |
        Format-Table -AutoSize
    ```

    **Prévention :** Créez un processus formel de demande d'exception GPO : formulaire documenté, validation sécurité, durée limitée avec date de révision. Chaque exception doit être dans le commentaire GPMC avec son propriétaire et sa date d'expiration. Sans processus, les exceptions s'accumulent sans jamais être révisées.


!!! quote "En résumé"
    - === "Cas 1 — GPO sécurité manquante sur 30 % des postes".
    - Symptôme : Une GPO de sécurité critique (restrictions USB, Defender config) s'applique sur 70 % des machines du domaine.
    - Les 30 % restants n'ont aucune trace de la GPO dans gpresult /r.
    - Exécutez gpresult /r /scope computer sur une machine affectée.
    - La sortie indique ACCESS DENIED dans la section RSOP ou la GPO apparaît dans "Denied GPOs" sans raison de filtrage visible.
---

## :material-toolbox: Boîte à outils de diagnostic

Référence rapide des outils utilisés dans ce chapitre.

| Outil | Commande type | Usage principal |
|-------|--------------|-----------------|
| `gpresult` | `gpresult /r` ou `gpresult /h rapport.html /f` | Premier réflexe — liste GPO appliquées, refusées, et pourquoi |
| `Get-WinEvent` | `Get-WinEvent -LogName "Microsoft-Windows-GroupPolicy/Operational"` | Chronologie détaillée de chaque application GPO, durées des CSE, erreurs |
| `gpupdate` | `gpupdate /force` ou `gpupdate /boot` | Déclencher un rafraîchissement immédiat (+ redémarrage si nécessaire) |
| `Test-ComputerSecureChannel` | `Test-ComputerSecureChannel -Repair` | Diagnostiquer et réparer un canal Kerberos machine brisé |
| `dfsrdiag` | `dfsrdiag ReplicationState /member:DC01` | Vérifier l'état de réplication SYSVOL sur un DC |
| `manage-bde` | `manage-bde -protectors -adbackup C: -id {GUID}` | Forcer la sauvegarde d'une clé BitLocker existante dans AD |
| `Get-AppLockerFileInformation` | `Get-AppLockerFileInformation -Path app.exe` | Extraire les métadonnées (hash, publisher) pour créer des règles AppLocker |
| `Restore-GPO` | `Restore-GPO -Name "DDP" -Path C:\Backups` | Restaurer une GPO depuis une sauvegarde `Backup-GPO` |
| `dcgpofix` | `dcgpofix /ignoreschema /target:Domain` | Dernier recours — recrée DDP et DDCP aux valeurs d'usine |
| `Set-AdmPwdComputerSelfPermission` | `Set-AdmPwdComputerSelfPermission -OrgUnit "OU=..."` | Accorder la permission d'écriture LAPS aux comptes machine d'une OU |
| `Get-MDMGroupPolicyAnalytics` | Via Intune portal ou MS Graph | Identifier les paramètres GPO sans équivalent CSP MDM |

---
!!! quote "En résumé"
    - `gpresult /r` résout 80 % des cas en deux minutes — c'est votre premier réflexe systématique
    - Les 15 cas de ce chapitre couvrent les causes racines les plus fréquentes : canal machine brisé, filtrage de sécurité, permissions SYSVOL/OU, scripts non signés, WMI Filter mal écrits
    - La prévention vaut toujours mieux que la remédiation : DFS Namespace pour les redirections, Publisher rules pour AppLocker, audit avant Block pour les GPO firewall
    - La migration vers Intune ne supprime pas les GPO du jour au lendemain — gérez les MDM gaps avec rigueur et documentation
    - Chaque cas de ce chapitre a une version plus grave que celle décrite. La différence entre l'incident mineur et la coupure de production, c'est souvent un backup récent et une procédure de restauration testée.

---
*Voir aussi :*

- [`../bible-gpo/20-rsop-diagnostic.md`](../bible-gpo/20-rsop-diagnostic.md) — outils RSOP et gpresult en détail
- [`../gpo-pour-les-nuls/13-erreurs.md`](../gpo-pour-les-nuls/13-erreurs.md) — erreurs débutants fréquentes
