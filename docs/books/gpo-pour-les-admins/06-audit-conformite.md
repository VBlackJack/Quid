---
description: "Auditer les GPO en place, prouver la conformite aux referentiels de securite (ISO 27001, SOC2, HDS) et detecter toute modification non autorisee grace aux evenements Windows et aux scripts PowerShell."
tags:
  - gpo
  - administration
  - audit
  - conformite
  - securite
---

# Audit et conformite des GPO

!!! abstract "Ce que vous allez pouvoir faire"
    - Generer un rapport HTML ou XML exhaustif de toutes les GPO du domaine en une commande, exploitable par un auditeur externe
    - Detecter toute modification sur une GPO grace aux evenements Windows 5136 et 4719, sans deployer AGPM ni outil tiers
    - Comparer une GPO de production avec une baseline de securite documentee et produire un rapport de conformite pret a etre joint a un audit ISO 27001 ou SOC2
    - Centraliser les evenements GPO vers un SIEM via Windows Event Collector avec un filtre XML pret a l'emploi
    - Identifier les GPO orphelines, sans description et non modifiees depuis plus d'un an — les trois signaux d'une hygiene GPO degradee

!!! tip "Si vous ne retenez qu'une chose"
    L'evenement **5136** dans le journal Security enregistre chaque modification sur un objet Active Directory — y compris les GPO (`groupPolicyContainer`). Combine avec l'evenement **4719** (politique d'audit modifiee), ces deux ID vous donnent une piste d'audit complete des changements GPO **sans AGPM**. Mais ils n'existent que si "Audit Directory Service Changes" est active sur les controleurs de domaine. Verifiez ce point avant tout.

---

## Pourquoi auditer les GPO

### :material-shield-check: Les GPO comme surface d'attaque

Les GPO sont les vecteurs de configuration les plus puissants de votre Active Directory. Une GPO malveillante ou mal configuree peut desactiver le pare-feu, affaiblir la politique de mots de passe, ou deployer un script de demarrage sur tous les postes du domaine.

Pour un attaquant ayant obtenu les droits suffisants, modifier une GPO est une methode discrete et persistante d'installation d'une backdoor. Sans audit actif, cette modification peut rester invisible pendant des semaines.

### :material-file-document-check: Les exigences reglementaires

Les referentiels ISO 27001, SOC2 Type II et HDS exigent tous de pouvoir demontrer que les configurations de securite sont **maintenues dans le temps** et que toute modification est **tracee et justifiee**.

Lors d'un audit externe, la question "qui a modifie cette GPO de securite, et quand ?" doit recevoir une reponse factuelle et documentee. "On ne sait pas" ou "on n'avait pas active l'audit" constituent des non-conformites majeures.

### :material-timeline-question: Ce que l'audit GPO doit couvrir

Un dispositif d'audit GPO complet repond a quatre questions.

**Qui a modifie ?** — Nom du compte ayant effectue la modification (via Event 5136).

**Quoi a change ?** — Attribut LDAP modifie sur l'objet `groupPolicyContainer` et, via LGPO.exe, la valeur avant/apres.

**Quand ?** — Horodatage exact de la modification.

**La GPO est-elle conforme a la baseline attendue ?** — Comparaison entre les parametres actuels et les valeurs requises par votre politique de securite.

!!! quote "En resume"
    Sans audit GPO, une investigation d'incident de securite ne peut pas determiner si un attaquant a modifie une GPO pour affaiblir les defenses. L'activation des evenements 5136 et 4719 est le prerequis zero de tout dispositif de securite serieux sur Active Directory.

---

## Audit natif : Get-GPOReport

### :material-file-code: Rapport HTML de toutes les GPO

La commande la plus directe pour produire une vue complete de toutes les GPO du domaine. Le rapport HTML est lisible dans n'importe quel navigateur et peut etre joint a un dossier d'audit.

```powershell
# Generate a single HTML report for all GPOs, sorted alphabetically
$reportPath = "C:\Temp\GPO-Report-$(Get-Date -Format 'yyyyMMdd').html"
Get-GPO -All | Sort-Object DisplayName | ForEach-Object {
    Get-GPOReport -Guid $_.Id -ReportType Html | Out-File $reportPath -Append
}
Write-Host "Report saved to $reportPath"
```

```title="Resultat attendu"
Report saved to C:\Temp\GPO-Report-20260405.html
```

Le fichier genere concatene le rapport de chaque GPO. Chaque section commence par le nom, la date de modification, et le statut de la GPO avant de detailler les parametres configures.

!!! warning "Rapport HTML concatene non structure"
    La concatenation de rapports HTML individuels produit un fichier valide mais sans en-tete HTML global. Pour un rapport structure avec navigation, privilegiez le format XML (voir section suivante) et transformez-le via XSLT ou parsez-le en PowerShell.

### :material-xml: Extraction de parametres via le format XML

Le format XML de `Get-GPOReport` est la base de tous les scripts de conformite. Il permet de cibler precisement un parametre dans une ou plusieurs GPO sans lire l'integralite du rapport.

Cet exemple extrait les parametres de politique de securite (`SecurityOptions`) de toutes les GPO du domaine pour les consolider dans un tableau de conformite.

```powershell
# Extract security settings from all GPOs for a compliance report
$ns = @{ q1 = "http://www.microsoft.com/GroupPolicy/Settings/Security" }

$complianceData = Get-GPO -All | ForEach-Object {
    $gpo = $_
    $xml = [xml](Get-GPOReport -Guid $gpo.Id -ReportType Xml)

    $secSettings = $xml | Select-Xml -XPath "//q1:SecurityOptions/q1:Display" -Namespace $ns
    if ($secSettings) {
        $secSettings | ForEach-Object {
            [PSCustomObject]@{
                GPO     = $gpo.DisplayName
                Setting = $_.Node.Name
                Value   = $_.Node.DisplayString
            }
        }
    }
}

$complianceData | Sort-Object GPO, Setting | Format-Table -AutoSize
```

```title="Resultat attendu (extrait)"
GPO                      Setting                              Value
---                      -------                              -----
SEC-Postes-Baseline      Minimum password length              12
SEC-Postes-Baseline      Password must meet complexity req.   Enabled
SEC-Postes-Baseline      Account lockout threshold            5
SEC-Serveurs-Production  Minimum password length              16
```

### :material-export: Export CSV pour Excel ou SIEM

Pour un audit formel, exportez les donnees en CSV plutot que de les afficher a l'ecran.

```powershell
# Export all GPO settings to CSV for audit documentation
$reportPath = "C:\Temp\GPO-Compliance-$(Get-Date -Format 'yyyyMMdd').csv"

Get-GPO -All | ForEach-Object {
    $gpo = $_
    [PSCustomObject]@{
        Name             = $gpo.DisplayName
        GUID             = $gpo.Id
        Status           = $gpo.GpoStatus
        CreatedDate      = $gpo.CreationTime.ToString("yyyy-MM-dd")
        LastModifiedDate = $gpo.ModificationTime.ToString("yyyy-MM-dd")
        Description      = $gpo.Description
        WMIFilter        = if ($gpo.WmiFilter) { $gpo.WmiFilter.Name } else { "" }
    }
} | Export-Csv $reportPath -NoTypeInformation

Write-Host "Exported $((Get-GPO -All).Count) GPOs to $reportPath"
```

```title="Resultat attendu"
Exported 47 GPOs to C:\Temp\GPO-Compliance-20260405.csv
```

!!! quote "En resume"
    `Get-GPOReport -ReportType Xml` est le point d'entree de tout script de conformite. Le XML expose la totalite des parametres d'une GPO de maniere parseable. La combinaison XPath + namespace `q1` est le pattern a retenir pour cibler n'importe quelle section du rapport.

---

## Surveillance des modifications GPO : Event ID 5136

### :material-eye: Comment Windows trace les modifications AD

Chaque objet Active Directory — y compris les GPO, representees dans LDAP comme des objets `groupPolicyContainer` — genere un evenement **5136** dans le journal Security du controleur de domaine chaque fois qu'un de ses attributs est modifie.

Cet evenement contient : le nom du compte auteur de la modification, le nom LDAP de l'attribut modifie, la nouvelle valeur, et l'horodatage exact.

### :material-cog: Prerequis : activer "Audit Directory Service Changes"

L'evenement 5136 n'est pas genere par defaut. Il necessite l'activation de la sous-categorie d'audit **"Audit Directory Service Changes"** via la politique d'audit avancee.

Chemin dans la GPO :
`Computer Configuration → Policies → Windows Settings → Security Settings → Advanced Audit Policy Configuration → DS Access → Audit Directory Service Changes`

Configurez cette sous-categorie sur **Success** (au minimum) ou **Success and Failure** sur tous les controleurs de domaine. La politique doit etre liee au niveau de la DC OU.

```powershell
# Verify that Audit Directory Service Changes is enabled on the PDC
$dcPDC = (Get-ADDomain).PDCEmulator

Invoke-Command -ComputerName $dcPDC -ScriptBlock {
    auditpol /get /subcategory:"Directory Service Changes"
}
```

```title="Resultat attendu"
System audit policy
Category/Subcategory                      Setting
DS Access
  Directory Service Changes               Success and Failure
```

!!! danger "5136 non active par defaut"
    L'evenement 5136 n'est **pas genere** si "Audit Directory Service Changes" n'est pas active. Sans cette configuration, toutes les modifications GPO sont silencieuses. Verifiez systematiquement ce point sur chaque nouveau controleur de domaine promu — la politique d'audit avancee ne se propage pas toujours correctement.

### :material-magnify: Requete PowerShell : modifications GPO des 7 derniers jours

```powershell
# Find all GPO modifications in the last 7 days from the PDC Security log
$startDate = (Get-Date).AddDays(-7)
$dcPDC     = (Get-ADDomain).PDCEmulator

Get-WinEvent -ComputerName $dcPDC -FilterHashtable @{
    LogName   = 'Security'
    Id        = 5136
    StartTime = $startDate
} | Where-Object {
    $_.Message -match "groupPolicyContainer"
} | Select-Object TimeCreated,
    @{N="Object";     E={ ($_.Message -split "`n" | Where-Object { $_ -match "Object Name:" })     -replace ".*Object Name:\s*","" }},
    @{N="Attribute";  E={ ($_.Message -split "`n" | Where-Object { $_ -match "LDAP Display Name:" }) -replace ".*LDAP Display Name:\s*","" }},
    @{N="ModifiedBy"; E={ ($_.Message -split "`n" | Where-Object { $_ -match "Account Name:" })[0]  -replace ".*Account Name:\s*","" }} |
    Format-Table -AutoSize -Wrap
```

```title="Resultat attendu (extrait)"
TimeCreated           Object                                       Attribute       ModifiedBy
-----------           ------                                       ---------       ----------
2026-04-04 14:32:11   CN={GUID},CN=Policies,CN=System,DC=...       versionNumber   CONTOSO\jdupont
2026-04-03 09:17:44   CN={GUID},CN=Policies,CN=System,DC=...       gPCFileSysPath  CONTOSO\adminsvc
```

### :material-translate: Decoder le GUID vers un nom de GPO

Le champ `Object` contient le GUID de la GPO au format LDAP. Pour obtenir le nom lisible :

```powershell
# Resolve a GPO GUID to its display name
$guid = "{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}"  # replace with GUID from event 5136
Get-GPO -Guid $guid.Trim("{}") | Select-Object DisplayName, ModificationTime, GpoStatus
```

```title="Resultat attendu"
DisplayName       ModificationTime       GpoStatus
-----------       ----------------       ---------
SEC-Postes-Baseline   2026-04-04 14:32:11    AllSettingsEnabled
```

### :material-table: Les attributs LDAP importants a surveiller

| Attribut LDAP | Signification | Niveau d'alerte |
|---|---|---|
| `versionNumber` | Numero de version de la GPO incremente a chaque modification | Haute |
| `gPCFileSysPath` | Chemin SYSVOL de la GPO — ne doit pas changer | Critique |
| `nTSecurityDescriptor` | Permissions sur l'objet GPO modifiees | Critique |
| `gPCMachineExtensionNames` | Extensions CSE utilisees cote ordinateur | Haute |
| `gPCUserExtensionNames` | Extensions CSE utilisees cote utilisateur | Haute |
| `displayName` | Nom de la GPO renomme | Moyenne |

!!! quote "En resume"
    L'evenement 5136 filtres sur `groupPolicyContainer` est votre principale source de verite pour les modifications GPO. Le GUID dans le champ Object se resout en nom lisible avec `Get-GPO -Guid`. Ciblez toujours le PDC Emulator pour les requetes d'audit — c'est lui qui recoit les ecritures AD en priorite.

---

## Event ID 4719 : changement de politique d'audit

### :material-alert-circle: Pourquoi cet evenement est critique

L'evenement **4719** est declenche lorsque la politique d'audit systeme est modifiee. C'est un signal d'alerte fort : si quelqu'un desactive la sous-categorie "Directory Service Changes", les evenements 5136 cessent d'etre generes — et les modifications GPO deviennent silencieuses.

Un attaquant suffisamment avance commencera souvent par desactiver les logs d'audit avant d'agir. L'evenement 4719 est donc la premiere ligne de defense de votre dispositif d'audit.

### :material-console: Interroger les evenements 4719

```powershell
# Review the 50 most recent audit policy changes on the PDC
$dcPDC = (Get-ADDomain).PDCEmulator

Get-WinEvent -ComputerName $dcPDC -FilterHashtable @{
    LogName = 'Security'
    Id      = 4719
} -MaxEvents 50 | Select-Object TimeCreated, Message | Format-Table -AutoSize -Wrap
```

```title="Resultat attendu (exemple sans incident)"
TimeCreated             Message
-----------             -------
2026-04-01 08:00:03     System audit policy was changed. ...
```

### :material-bell-ring: Creer une alerte sur 4719

En production, l'evenement 4719 doit declencher une alerte immmediate. Si vous n'avez pas encore de SIEM, un script de surveillance simple peut envoyer un mail via `Send-MailMessage`.

```powershell
# Monitor for unexpected audit policy changes and alert by email
$dcPDC     = (Get-ADDomain).PDCEmulator
$cutoff    = (Get-Date).AddMinutes(-60)  # check the last hour

$events = Get-WinEvent -ComputerName $dcPDC -FilterHashtable @{
    LogName   = 'Security'
    Id        = 4719
    StartTime = $cutoff
} -ErrorAction SilentlyContinue

if ($events) {
    $body = $events | Select-Object TimeCreated, Message |
        Out-String

    Send-MailMessage `
        -To      "soc@contoso.local" `
        -From    "monitor@contoso.local" `
        -Subject "[ALERTE] Politique d'audit modifiee sur $dcPDC" `
        -Body    $body `
        -SmtpServer "smtp.contoso.local"
}
```

!!! danger "4719 sans explication = incident de securite"
    Un evenement 4719 inattendu — en dehors des fenetres de maintenance planifiees — doit etre traite comme un incident de securite actif jusqu'a preuve du contraire. La modification de la politique d'audit est une etape classique d'un mouvement lateral avance.

!!! quote "En resume"
    Monitorez 4719 avec au moins autant de rigueur que 5136. Si 4719 survient sans cause documentee, supposez une compromission et lancez une investigation. En SIEM, correlez 4719 avec les connexions RDP et les elevations de privileges dans la meme fenetre temporelle.

---

## Script de conformite baseline

### :material-clipboard-list-outline: Principe du controle de conformite

Un script de conformite compare les parametres actuels d'une GPO a une liste de valeurs requises — la "baseline". Il produit un rapport indiquant si chaque parametre est conforme, non conforme ou absent.

Ce type de rapport est exactement ce qu'un auditeur ISO 27001 ou SOC2 demande pour verifier que les controles de securite sont en place.

### :material-function: Fonction Test-GPOCompliance

```powershell
# Compliance check: verify a named GPO contains required registry-based settings
function Test-GPOCompliance {
    param(
        [Parameter(Mandatory)]
        [string]$GPOName,

        [Parameter(Mandatory)]
        [hashtable]$RequiredSettings
    )

    $xml     = [xml](Get-GPOReport -Name $GPOName -ReportType Xml)
    $results = @()

    foreach ($setting in $RequiredSettings.GetEnumerator()) {
        # Search for the registry key path in GPO XML
        $found = $xml | Select-Xml -XPath "//RegistrySetting/KeyPath[contains(text(),'$($setting.Key)')]"
        $status = if ($found) { "COMPLIANT" } else { "MISSING" }

        $results += [PSCustomObject]@{
            Setting  = $setting.Key
            Expected = $setting.Value
            Status   = $status
        }
    }

    return $results
}
```

### :material-play: Utilisation du script

```powershell
# Define required security baseline settings and run the check
$baseline = @{
    "EnableFirewall"    = "1"
    "DisableAutorun"    = "1"
    "MinPasswordLength" = "12"
    "NTLMv2Only"        = "1"
    "DisableWinRM"      = "0"   # WinRM doit rester actif pour la gestion
}

$report = Test-GPOCompliance -GPOName "SEC-Postes-Baseline" -RequiredSettings $baseline
$report | Format-Table -AutoSize

# Summary
$compliant  = ($report | Where-Object Status -eq "COMPLIANT").Count
$missing    = ($report | Where-Object Status -eq "MISSING").Count
Write-Host "Conformite : $compliant/$($report.Count) parametres OK — $missing manquants"
```

```title="Resultat attendu"
Setting             Expected  Status
-------             --------  ------
EnableFirewall      1         COMPLIANT
DisableAutorun      1         COMPLIANT
MinPasswordLength   12        MISSING
NTLMv2Only          1         COMPLIANT
DisableWinRM        0         COMPLIANT

Conformite : 4/5 parametres OK — 1 manquants
```

### :material-file-export: Export du rapport de conformite

```powershell
# Export compliance report to CSV for audit documentation
$reportPath = "C:\Temp\Compliance-Report-$(Get-Date -Format 'yyyyMMdd').csv"
$report | Export-Csv $reportPath -NoTypeInformation
Write-Host "Compliance report saved to $reportPath"
```

!!! warning "Limites de Test-GPOCompliance"
    Cette fonction recherche les cles dans les `RegistrySetting` de la GPO. Les parametres deployes via les **Security Settings** (GptTmpl.inf) comme la politique de mots de passe, les droits utilisateurs ou le pare-feu Windows Defender sont dans un namespace XML different. Pour ces parametres, utilisez le namespace `q1` comme dans les exemples de la section `Get-GPOReport`.

!!! quote "En resume"
    La fonction `Test-GPOCompliance` est un point de depart. En production, enrichissez-la avec la verification des Security Settings via le namespace XML adequat, et exportez le rapport en CSV pour l'archiver dans votre GED ou votre outil de ticketing.

---

## Intégration SIEM : Windows Event Collector

### :material-server-network: Architecture WEC pour les evenements GPO

Windows Event Collector (WEC) permet de centraliser les evenements des controleurs de domaine vers un collecteur unique, sans installer d'agent tiers. Le SIEM interroge ensuite uniquement le collecteur.

Cette architecture reduit la charge sur les DC et garantit que les evenements sont preserves meme si le journal Security d'un DC est efface.

### :material-xml: Filtre WEF pour les evenements GPO

Ce fichier de souscription WEF (Windows Event Forwarding) collecte uniquement les evenements liés aux GPO, en filtrant sur la classe d'objet `groupPolicyContainer`.

```xml
<!-- WEF Subscription filter for GPO audit events -->
<!-- Deploy on the WEC server via wecutil cs subscription.xml -->
<Subscription>
  <SubscriptionType>SourceInitiated</SubscriptionType>
  <Description>GPO audit events from all Domain Controllers</Description>
  <SubscriptionId>GPO-Audit</SubscriptionId>
  <Enabled>true</Enabled>
  <Uri>http://schemas.microsoft.com/wbem/wsman/1/windows/EventLog</Uri>
  <ConfigurationMode>Custom</ConfigurationMode>
  <Delivery Mode="Push">
    <Batching>
      <MaxLatencyTime>900000</MaxLatencyTime>
    </Batching>
    <PushSettings>
      <Heartbeat Interval="1800000"/>
    </PushSettings>
  </Delivery>
  <Query>
    <![CDATA[
      <QueryList>
        <Query Id="0">
          <Select Path="Security">
            *[System[(EventID=5136 or EventID=5137 or EventID=5141 or EventID=4719 or EventID=4670)]]
            and
            *[EventData[Data[@Name='ObjectClass']='groupPolicyContainer']]
          </Select>
        </Query>
      </QueryList>
    ]]>
  </Query>
  <ReadExistingEvents>false</ReadExistingEvents>
  <TransportSecurity>HTTPS</TransportSecurity>
  <ContentFormat>RenderedText</ContentFormat>
  <Locale Language="fr-FR"/>
</Subscription>
```

### :material-table: Tableau des Event IDs GPO a transmettre au SIEM

| Event ID | Journal | Signification | Priorite SIEM |
|----------|---------|---------------|---------------|
| 5136 | Security | Objet AD modifie — attribut GPO change | Haute |
| 5137 | Security | Objet AD cree — nouvelle GPO creee | Haute |
| 5141 | Security | Objet AD supprime — GPO supprimee | Critique |
| 4719 | Security | Politique d'audit systeme modifiee | Critique |
| 4670 | Security | Permissions d'un objet AD modifiees | Haute |

### :material-cog-play: Deployer la souscription WEF

```powershell
# Deploy the WEF subscription on the WEC server
# Run on the WEC server as Domain Admin
wecutil cs "C:\WEF\GPO-Audit-Subscription.xml"

# Verify the subscription is active
wecutil gs GPO-Audit

# List all active subscriptions
wecutil es
```

```title="Resultat attendu (wecutil es)"
GPO-Audit
Security-Critical-Events
```

!!! warning "Journal Security plein"
    Activer toutes les sous-categories d'audit sur un grand domaine peut remplir le journal Security d'un DC en quelques heures. Configurez une retention minimale de **128 Mo** sur les DC — la recommandation pour un environnement enterprise est **4 Go**. Utilisez WEC pour centraliser les evenements et alleger la pression sur les journaux locaux des DC.

!!! quote "En resume"
    WEC + un filtre XPath sur `groupPolicyContainer` est la solution la plus simple pour centraliser les evenements GPO sans agent. Les cinq Event IDs du tableau couvrent l'ensemble du cycle de vie d'une GPO : creation, modification, suppression, changement de permissions, changement de politique d'audit.

---

## Audit avec LGPO.exe

### :material-tools: Ce que LGPO.exe apporte

`LGPO.exe` est un outil Microsoft de la **Security Compliance Toolkit**. Il peut exporter les parametres d'une GPO au format texte parseable, ce qui permet de faire des **comparaisons diff** entre deux etats de la meme GPO.

C'est l'approche la plus fiable pour repondre a la question "qu'est-ce qui a change exactement dans cette GPO ?" quand l'evenement 5136 indique seulement que `versionNumber` a ete incremente.

### :material-download: Telecharger LGPO.exe

LGPO.exe est disponible sur le site officiel de Microsoft dans le package **Security Compliance Toolkit**. Placez l'executable dans `C:\Tools\LGPO\` ou ajoutez son dossier au PATH systeme.

```powershell
# Verify LGPO.exe is accessible
Get-Command lgpo.exe -ErrorAction SilentlyContinue |
    Select-Object Name, Source, Version
```

### :material-camera: Creer un snapshot des GPO au format texte

```powershell
# Export all GPO Registry.pol files to parseable text — GPO snapshot
$snapshotPath = "C:\Temp\GPO-Snapshot-$(Get-Date -Format 'yyyyMMdd')"
New-Item -ItemType Directory -Path $snapshotPath -Force | Out-Null

$domain = $env:USERDNSDOMAIN

Get-GPO -All | ForEach-Object {
    $guid    = $_.Id.ToString()
    $gptPath = "\\$domain\SYSVOL\$domain\Policies\{$guid}\Machine\Registry.pol"

    if (Test-Path $gptPath) {
        # Sanitize GPO name for use as filename
        $safeName = $_.DisplayName -replace '[\\/:*?"<>|]', '_'
        $outFile  = "$snapshotPath\$safeName.txt"

        lgpo.exe /parse /m $gptPath > $outFile
        Write-Host "Exported: $($_.DisplayName)"
    }
}

Write-Host "Snapshot saved to $snapshotPath"
```

```title="Resultat attendu"
Exported: Default Domain Policy
Exported: SEC-Postes-Baseline
Exported: SEC-Serveurs-Production
Exported: GPO-WSUS-Configuration
...
Snapshot saved to C:\Temp\GPO-Snapshot-20260405
```

### :material-compare: Comparer deux snapshots

Apres un evenement 5136 signalant une modification, comparez le snapshot actuel avec le snapshot de reference pour identifier precisement la valeur modifiee.

```powershell
# Compare two snapshots to identify changed settings
$snap1 = "C:\Temp\GPO-Snapshot-20260401"
$snap2 = "C:\Temp\GPO-Snapshot-20260405"

# Compare a specific GPO file
$file1 = "$snap1\SEC-Postes-Baseline.txt"
$file2 = "$snap2\SEC-Postes-Baseline.txt"

if ((Test-Path $file1) -and (Test-Path $file2)) {
    $diff = Compare-Object (Get-Content $file1) (Get-Content $file2)
    if ($diff) {
        $diff | Format-Table -AutoSize
    } else {
        Write-Host "No changes detected in SEC-Postes-Baseline"
    }
}
```

```title="Resultat attendu (exemple avec modification)"
InputObject                                      SideIndicator
-----------                                      -------------
MACHINE\Software\Policies\Microsoft\...\Value=0  <=
MACHINE\Software\Policies\Microsoft\...\Value=1  =>
```

Les lignes `<=` representent les valeurs dans le snapshot de reference. Les lignes `=>` representent les valeurs actuelles. Une difference sur une cle de securite doit declencher une investigation.

!!! tip "Versionner les snapshots avec Git"
    Versionnez vos snapshots dans un depot Git prive. `git diff` produit une sortie diff beaucoup plus lisible que `Compare-Object` et conserve l'historique complet des modifications. Ajoutez la creation du snapshot a votre pipeline de sauvegarde hebdomadaire GPO.

!!! quote "En resume"
    LGPO.exe + snapshots Git = visibilite complete sur "ce qui a change" dans les GPO, au niveau du parametre individuel. L'evenement 5136 vous dit qu'une modification a eu lieu, le diff LGPO vous dit laquelle.

---

## Rapport de GPO orphelines et non documentees

### :material-broom: Les trois signaux d'hygiene GPO degradee

Un parc GPO mal maintenu accumule trois types de problemes qui affectent a la fois les performances (traitement GPO plus long) et la securite (configurations imprevisibles).

**GPO non liee** — La GPO existe dans AD mais n'est liee a aucune OU. Elle consomme de l'espace SYSVOL et de la bande passante de replication sans s'appliquer nulle part. Risque : une GPO orpheline peut etre deliberement reliee par un attaquant.

**GPO sans description** — Impossible de savoir a quoi elle sert, qui l'a creee, ou si elle est encore utile. Risque : personne ne la supprime par peur de casser quelque chose.

**GPO non modifiee depuis plus d'un an** — Peut indiquer qu'elle est obsolete, ou au contraire qu'elle est critique et stable. Dans les deux cas, elle doit etre documentee et validee.

### :material-magnify-scan: Script d'identification

```powershell
# Identify GPOs with hygiene issues: unlinked, no description, or stale
$cutoffDate = (Get-Date).AddDays(-365)

$issues = Get-GPO -All | ForEach-Object {
    $gpo = $_
    $xml = [xml](Get-GPOReport -Guid $gpo.Id -ReportType Xml)

    # LinksTo is present in the XML only if the GPO has at least one link
    $isLinked = $null -ne $xml.GPO.LinksTo

    [PSCustomObject]@{
        Name        = $gpo.DisplayName
        GUID        = $gpo.Id
        Unlinked    = -not $isLinked
        NoDesc      = [string]::IsNullOrWhiteSpace($gpo.Description)
        Stale       = $gpo.ModificationTime -lt $cutoffDate
        LastModified = $gpo.ModificationTime.ToString("yyyy-MM-dd")
        Status      = $gpo.GpoStatus
    }
} | Where-Object { $_.Unlinked -or $_.NoDesc -or $_.Stale }

# Display results
$issues | Format-Table Name, Unlinked, NoDesc, Stale, LastModified, Status -AutoSize

# Summary
Write-Host ""
Write-Host "GPOs avec problemes d'hygiene : $($issues.Count)"
Write-Host "  Non liees   : $(($issues | Where-Object Unlinked).Count)"
Write-Host "  Sans desc.  : $(($issues | Where-Object NoDesc).Count)"
Write-Host "  Non maj +1an: $(($issues | Where-Object Stale).Count)"
```

```title="Resultat attendu (exemple)"
Name                          Unlinked  NoDesc  Stale   LastModified  Status
----                          --------  ------  -----   ------------  ------
OLD-IE-Settings               True      True    True    2019-03-14    AllSettingsEnabled
TEST-GPO-NePasSupprimer        True      True    False   2025-11-02    AllSettingsEnabled
SEC-Postes-Baseline           False     False   False   2026-04-04    AllSettingsEnabled

GPOs avec problemes d'hygiene : 2
  Non liees   : 2
  Sans desc.  : 2
  Non maj +1an: 1
```

### :material-export: Exporter pour le rapport d'audit

```powershell
# Export hygiene report to CSV for audit
$reportPath = "C:\Temp\GPO-Hygiene-$(Get-Date -Format 'yyyyMMdd').csv"
$issues | Export-Csv $reportPath -NoTypeInformation
Write-Host "Hygiene report saved to $reportPath — $($issues.Count) items to review"
```

!!! warning "Ne supprimez pas sans validation"
    Avant de supprimer une GPO identifiee comme "non liee" ou "obsolete", verifiez qu'elle n'est pas referencee dans des scripts ou des outils de monitoring par son GUID. Consultez egalement l'equipe applicative responsable du perimetre concerne. La suppression d'une GPO est irreversible sans backup.

!!! quote "En resume"
    Planifiez ce script en tache mensuelle et exportez le CSV vers votre outil de ticketing. Chaque item genere un ticket de validation. C'est la methode la plus rapide pour maintenir un parc GPO propre et auditables sans revue manuelle fastidieuse.

---

## Pieges en production

### :material-bomb: Pieges courants et comment les eviter

**Journal Security plein apres activation de l'audit.**
L'activation de toutes les sous-categories d'audit sur un domaine de taille significative peut saturer le journal Security d'un DC en quelques heures, provoquant l'arret de la journalisation des nouveaux evenements. Augmentez la taille maximale du journal (4 Go recommande) et deployez WEC avant d'activer les sous-categories.

**Interroger tous les DC au lieu du PDC.**
Les requetes d'evenements 5136 doivent cibler en priorite le **PDC Emulator** — c'est lui qui recoit les ecritures Active Directory. Interroger tous les DC produit des doublons et ralentit les scripts. Utilisez `(Get-ADDomain).PDCEmulator` pour cibler automatiquement le bon DC.

**Confondre la date de modification de la GPO avec la date de modification des parametres.**
`Get-GPO` retourne `ModificationTime` qui correspond au dernier incrément de `versionNumber`. Un simple changement de description ou de commentaire incrementé la version. Pour identifier ce qui a vraiment change, croisez avec l'evenement 5136 et les snapshots LGPO.

**Namespace XML incorrect dans Get-GPOReport.**
Les parametres de securite (password policy, audit policy, user rights) sont dans le namespace `http://www.microsoft.com/GroupPolicy/Settings/Security`. Les parametres de registre sont dans un namespace different. Un XPath sans namespace produit toujours zero resultat — sans message d'erreur.

!!! warning "Journal Security plein"
    Configurez une taille de journal minimum de **128 Mo** sur tous les DC, et **4 Go** sur les DC de production. Verifiez cette configuration apres chaque promotion DC — la politique de taille du journal ne s'applique pas toujours automatiquement. Utilisez WEC pour centraliser les evenements et reduire la pression sur les journaux locaux.

!!! danger "5136 non active par defaut"
    L'evenement 5136 necessite l'activation explicite de "Audit Directory Service Changes" dans la politique d'audit avancee. Sans cette activation, les modifications GPO sont **entierement silencieuses**. Verifiez que cette sous-categorie est configuree sur tous les DC, y compris les DC nouvellement promus, et incluez cette verification dans votre checklist de promotion DC.

!!! quote "En resume"
    Les deux pieges les plus couteux en production : journal Security plein qui stoppe la journalisation silencieusement, et 5136 non active qui rend les modifications GPO invisibles. Verifiez ces deux points avant de presenter votre dispositif d'audit a un auditeur externe.

---

## Cross-references

| Sujet | Reference |
|-------|-----------|
| Gouvernance et delegation des droits GPO | Ch. 02 — Gouvernance |
| Cmdlets PowerShell GroupPolicy (Get-GPOReport, Get-GPO) | Ch. 03 — PowerShell GroupPolicy module |
| Sauvegarde et restauration des GPO | Ch. 05 — Sauvegarde et restauration |
| Baselines de securite Microsoft (SCT) | La Bible GPO — Ch. 22 — Baselines |
| Audit avancé et strategies d'audit subcategories | La Bible GPO — Ch. 13 — Securite avancee |
| Automatisation CI/CD — audit en pipeline | Ch. 23 — Automatisation CI/CD |
