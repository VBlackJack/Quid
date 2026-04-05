---
description: Certificats et PKI via GPO — auto-enrollment, racines de confiance, CTL, révocation OCSP/CRL, NTAuth
tags: [gpo-admins, pki, certificats, auto-enrollment, ocsp, ntauth, crl]
---
# Certificats et PKI via GPO

!!! abstract "Ce que vous allez apprendre"
    - Configurer l'auto-enrollment pour que les machines et les utilisateurs obtiennent leurs certificats sans intervention manuelle
    - Distribuer des certificats racines et intermédiaires via GPO pour que tous les postes fassent confiance à votre CA interne
    - Déployer les Trusted Publishers et les CTL pour autoriser l'exécution de code signé en interne
    - Contrôler le comportement de vérification de révocation (OCSP/CRL) et éviter les blocages TLS en production
    - Publier le certificat CA dans le store NTAuth — prérequis absolu pour l'authentification par carte à puce

!!! tip "Si vous ne retenez qu'une chose"
    Une PKI d'entreprise ne fonctionne correctement que si **trois conditions sont réunies simultanément** : les machines font confiance à la CA (Trusted Root), les certificats sont distribués automatiquement (auto-enrollment), et le store NTAuth contient le certificat CA (authentification AD). Si l'une de ces trois pièces manque, les symptômes sont cryptiques — TLS qui échoue, logon smartcard refusé, scripts bloqués par SmartScreen — et le diagnostic prend des heures.

---

## Contexte de production

Les GPO sont le seul mécanisme natif pour distribuer la configuration PKI à grande échelle dans Active Directory.

Sans GPO PKI, chaque nouveau poste doit recevoir manuellement les certificats racines, les paramètres de révocation et les droits d'auto-enrollment. Sur un parc de 500 machines, c'est un gouffre opérationnel. Avec des GPO bien configurées, un poste joint au domaine hérite de la totalité de la configuration PKI en moins de 90 minutes après son premier redémarrage.


!!! quote "En résumé"
    - Les GPO sont le seul mécanisme natif pour distribuer la configuration PKI à grande échelle dans Active Directory.
    - Sur un parc de 500 machines, c'est un gouffre opérationnel.
    - Le contexte de production fixe les contraintes réelles de réseau, de portée et d’exploitation qui gouvernent tout le chapitre.
    - Retenez les hypothèses opérationnelles avant de choisir un modèle de liaison ou de déploiement.
---

## Auto-enrollment via GPO

### :material-certificate: Qu'est-ce que l'auto-enrollment

L'auto-enrollment est le mécanisme par lequel un poste ou un utilisateur obtient automatiquement un certificat depuis une CA d'entreprise, sans aucune intervention de l'administrateur ou de l'utilisateur.

Il couvre le cycle complet : demande initiale, renouvellement avant expiration, et suppression lorsque le template est retiré. C'est la base de toute PKI d'entreprise fonctionnelle.

### :material-check-circle: Conditions préalables obligatoires

Trois conditions doivent être réunies avant de configurer l'auto-enrollment via GPO.

**Enterprise CA obligatoire.** L'auto-enrollment ne fonctionne qu'avec une CA d'entreprise (Enterprise CA) intégrée à Active Directory. Une Standalone CA ne publie pas ses templates dans LDAP et ne peut pas répondre aux requêtes d'auto-enrollment.

**Permission `Autoenroll` sur le template.** Le template de certificat doit accorder la permission `Autoenroll` au groupe de sécurité cible (typiquement `Domain Computers` pour les templates machine, `Domain Users` pour les templates utilisateur). Cette permission s'ajoute dans la MMC `certtmpl.msc`, onglet Security du template.

**Template publié sur la CA.** Le template doit être publié sur la CA via `certsrv.msc` → Certificate Templates → New → Certificate Template to Issue. Un template existant dans LDAP mais non publié est invisible pour l'auto-enrollment.

!!! danger "Production — checklist avant activation"
    Avant d'activer l'auto-enrollment en GPO, vérifiez ces trois points dans l'ordre. Beaucoup d'admins activent la GPO, voient que rien ne se passe, et concluent que la PKI est cassée — alors que le template n'a tout simplement pas été publié sur la CA.

### :material-folder-cog: Configuration GPMC — Computer Configuration

L'auto-enrollment machine se configure via :

`Computer Configuration → Policies → Windows Settings → Security Settings → Public Key Policies → Certificate Services Client - Auto-Enrollment`

Double-cliquez sur le paramètre pour accéder à la boîte de dialogue.

| Paramètre | Valeur recommandée |
|---|---|
| Configuration Model | Enabled |
| Renew expired certificates, update pending certificates, and remove revoked certificates | Coché |
| Update certificates that use certificate templates | Coché |

La case **"Update certificates that use certificate templates"** est critique. Sans elle, les renouvellements basés sur des templates v2 ou v3 ne se déclenchent pas.

### :material-folder-account: Configuration GPMC — User Configuration

Le paramètre existe aussi en User Configuration. Il couvre les certificats utilisateur (signature S/MIME, authentification utilisateur, etc.).

`User Configuration → Policies → Windows Settings → Security Settings → Public Key Policies → Certificate Services Client - Auto-Enrollment`

Les mêmes cases doivent être cochées. Les deux nœuds (Computer et User) sont indépendants — activez les deux si votre PKI émet des certificats machine **et** utilisateur.

### :material-console: Vérification post-déploiement

Après application de la GPO, forcez le rafraîchissement et attendez quelques minutes.

```powershell title="Force GPO refresh and trigger certificate enrollment"
# Force GPO refresh on the local machine
gpupdate /force

# Trigger certificate auto-enrollment manually (useful for immediate testing)
certutil -pulse
```

```title="Résultat attendu"
CertUtil: -pulse command completed successfully.
```

Vérifiez ensuite la présence du certificat dans le store machine.

```powershell title="Verify enrolled machine certificate"
# List certificates in the local machine personal store
Get-ChildItem Cert:\LocalMachine\My | Select-Object Subject, Issuer, NotAfter, Thumbprint |
    Format-Table -AutoSize
```

```title="Résultat attendu"
Subject                        Issuer                   NotAfter             Thumbprint
-------                        ------                   --------             ----------
CN=POSTE01.contoso.local       CN=CONTOSO-CA, DC=...    2027-04-05 10:00:00  A1B2C3D4...
```

Pour les certificats utilisateur, interrogez le store personnel de l'utilisateur courant.

```powershell title="Verify enrolled user certificate"
Get-ChildItem Cert:\CurrentUser\My | Select-Object Subject, Issuer, NotAfter |
    Format-Table -AutoSize
```

Vous pouvez aussi utiliser `certutil -viewstore My` pour une vue plus détaillée incluant le template utilisé.

```powershell title="Detailed view with certutil"
# View machine personal store with template information
certutil -viewstore "My"

# View all certificates from a specific CA
certutil -store "My" | Select-String -Pattern "(Issuer|Template|NotAfter)"
```

### :material-clock-alert: Cycle de renouvellement automatique

Windows déclenche l'auto-enrollment au démarrage de la machine et à chaque application de la GPO (toutes les 90-120 minutes). Il vérifie aussi si un certificat arrive à expiration dans moins de **10% de sa durée de vie totale** — pour un certificat d'un an, le renouvellement est tenté à partir de 36 jours avant l'expiration.

!!! warning "À surveiller — Event ID 82"
    Si l'auto-enrollment échoue silencieusement, vérifiez le journal `Application` pour l'Event ID **82** (source : `AutoEnrollment`). Cet événement indique pourquoi la demande a échoué — permission manquante, template non publié, CA inaccessible. C'est le point de départ systématique de tout diagnostic auto-enrollment.

```powershell title="Check auto-enrollment events"
# Look for auto-enrollment failures in the Application log
Get-WinEvent -FilterHashtable @{
    LogName  = 'Application'
    ProviderName = 'AutoEnrollment'
} -MaxEvents 20 | Select-Object TimeCreated, Id, Message | Format-Table -Wrap -AutoSize
```

!!! quote "En résumé"
    - L'auto-enrollment nécessite une Enterprise CA, la permission `Autoenroll` sur le template, et le template publié sur la CA
    - Activez le paramètre en **Computer Configuration ET User Configuration** si vous émettez les deux types de certificats
    - L'Event ID 82 dans le journal Application est votre premier réflexe en cas d'échec silencieux
    - `certutil -pulse` force un cycle d'enrollment immédiat pour les tests

---

## Distribution des certificats racines et intermédiaires

### :material-certificate-outline: Pourquoi distribuer via GPO

Par défaut, un poste Windows ne fait confiance qu'aux CA du programme Microsoft Root Certificate Program.

Votre CA interne n'en fait pas partie. Sans distribution explicite, chaque accès HTTPS à un service interne (intranet, Outlook Web App, portail applicatif) génère une alerte de certificat. Les postes non membres du domaine ou nouvellement joints n'ont pas ce certificat racine.

La GPO est le mécanisme le plus fiable et le plus rapide pour propager ce certificat à l'ensemble du parc.

### :material-folder-cog: Chemin GPMC

`Computer Configuration → Policies → Windows Settings → Security Settings → Public Key Policies → Trusted Root Certification Authorities`

Cliquez-droit sur le nœud → **Import...** pour lancer l'assistant d'import.

### :material-import: Procédure d'import pas à pas

**Étape 1.** Exportez le certificat racine de votre CA depuis `certsrv.msc` → tâche "Download a CA certificate" → format DER. Le fichier `.cer` est votre source.

**Étape 2.** Dans la GPO ciblée, naviguez jusqu'au nœud `Trusted Root Certification Authorities` ci-dessus.

**Étape 3.** Cliquez-droit → Import → sélectionnez votre fichier `.cer`. L'assistant vous demande confirmation du store de destination — laissez "Trusted Root Certification Authorities".

**Étape 4.** Le certificat apparaît dans le volet droit de la GPMC. Vérifiez le nom et l'empreinte avant de fermer.

Pour les CA intermédiaires, la procédure est identique mais le nœud cible est :

`Computer Configuration → Policies → Windows Settings → Security Settings → Public Key Policies → Intermediate Certification Authorities`

### :material-registry: Ce qui se passe sur la machine

Lorsque la GPO s'applique, le certificat est écrit dans la base de registre Windows :

```
HKLM\SOFTWARE\Microsoft\SystemCertificates\ROOT\Certificates\<Thumbprint>
```

Chaque sous-clé représente un certificat. La valeur `Blob` contient le certificat encodé en DER.

Pour les CA intermédiaires, le store de destination est :

```
HKLM\SOFTWARE\Microsoft\SystemCertificates\CA\Certificates\<Thumbprint>
```

### :material-console: Vérification

```powershell title="Verify root CA certificate in machine trust store"
# Check if the internal CA certificate is trusted on the machine
$thumbprint = "A1B2C3D4E5F6..."  # replace with your CA certificate thumbprint

$cert = Get-ChildItem Cert:\LocalMachine\Root |
    Where-Object { $_.Thumbprint -eq $thumbprint }

if ($cert) {
    $cert | Select-Object Subject, Issuer, NotAfter, Thumbprint
    Write-Host "CA certificate is trusted on this machine."
} else {
    Write-Host "WARNING: CA certificate not found in Trusted Root store." -ForegroundColor Red
}
```

```title="Résultat attendu"
Subject    : CN=CONTOSO-CA, DC=contoso, DC=local
Issuer     : CN=CONTOSO-CA, DC=contoso, DC=local
NotAfter   : 2035-01-01 12:00:00
Thumbprint : A1B2C3D4E5F6...
CA certificate is trusted on this machine.
```

Vous pouvez aussi lister tous les certificats racines distribués via GPO en comparant le store machine avec les certificats issus de votre CA.

```powershell title="List all certificates from internal CA in root store"
# List all root certificates issued by internal CA
$internalCA = "CONTOSO-CA"  # replace with your CA common name

Get-ChildItem Cert:\LocalMachine\Root |
    Where-Object { $_.Issuer -match $internalCA } |
    Select-Object Subject, NotAfter, Thumbprint |
    Format-Table -AutoSize
```

!!! warning "À surveiller — certificat expiré dans le store"
    Si vous distribuez un certificat racine via GPO et que ce certificat expire, il **reste dans le store machine** jusqu'à la prochaine application GPO après que vous l'ayez retiré de la GPO. Un certificat racine expiré dans le store Trust provoque des erreurs de validation de chaîne cryptiques. Auditez la date d'expiration de vos CA racines au moins une fois par an.

!!! quote "En résumé"
    - Importez le certificat CA (`.cer`) dans `Trusted Root Certification Authorities` de la GPO
    - Le certificat atterrit dans `HKLM\SOFTWARE\Microsoft\SystemCertificates\ROOT\Certificates\`
    - Vérifiez avec `Get-ChildItem Cert:\LocalMachine\Root` filtré sur l'empreinte
    - Un certificat distribué via GPO reste en place jusqu'à ce que vous le retiriez de la GPO — pensez-y avant renouvellement de CA

---

## Trusted Publishers et Certificate Trust List (CTL)

### :material-shield-key: Cas d'usage : code signing interne

Votre équipe développe des scripts PowerShell ou des exécutables signés avec un certificat issu de votre CA interne.

Sans configuration supplémentaire, SmartScreen et AppLocker ne reconnaissent pas ce certificat comme une source fiable — même si la CA est dans le store Trusted Root. Ils vérifient le store **Trusted Publishers**, qui est distinct.

La distribution du certificat de signature dans le store Trusted Publishers, via GPO, est le mécanisme qui fait passer vos binaires internes de "bloqué par SmartScreen" à "autorisé sans invite".

### :material-folder-cog: Nœud GPMC pour Trusted Publishers

`Computer Configuration → Policies → Windows Settings → Security Settings → Public Key Policies → Trusted Publishers`

La procédure d'import est identique à celle des racines de confiance. Importez le certificat de **l'émetteur** (le certificat CA qui a signé le certificat de code signing), pas le certificat de code signing lui-même.

### :material-compare: Trusted Root vs Trusted Publishers — quand utiliser quoi

| Store | Usage | Exemple |
|---|---|---|
| Trusted Root | Faire confiance à la chaîne TLS/SSL | Intranet HTTPS, Outlook Web App |
| Trusted Publishers | Autoriser l'exécution de code signé | Scripts PowerShell, .exe internes |
| Intermediate CA | Compléter la chaîne de validation | CA intermédiaire entre root et leaf |

Un certificat dans Trusted Root ne suffit pas pour débloquer l'exécution de code. AppLocker vérifie Trusted Publishers indépendamment.

### :material-list-status: Certificate Trust List (CTL)

Une CTL est une liste signée de certificats de confiance. Elle permet de restreindre dynamiquement les CA autorisées pour une connexion TLS donnée — sans modifier le store global.

En entreprise, les CTL sont utiles pour contrôler quelles CA tierces sont autorisées pour des contextes spécifiques (exemple : seule votre CA interne est autorisée pour l'authentification mutuelle 802.1X).

La mise à jour de la CTL sur Windows se fait via le mécanisme **Certificate Trust List Update** (Windows Update) ou manuellement avec `certutil`.

```powershell title="Force CTL cache refresh"
# Force Windows to refresh the Certificate Trust List from Windows Update
certutil -setreg Chain\ChainCacheResyncFiletime @now

# Alternatively, force certificate store flush
certutil -urlcache * delete
```

```title="Résultat attendu"
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Cryptography\OID\EncodingType 0\CertDllCreateCertificateChainEngine\Config
  ChainCacheResyncFiletime REG_BINARY = <timestamp>
CertUtil: -setreg command completed successfully.
```

!!! warning "À surveiller — CTL et environnement hors ligne"
    Dans un environnement sans accès Internet (OT, réseau isolé), Windows tente de télécharger la CTL Microsoft depuis `ctldl.windowsupdate.com`. Si cette URL est inaccessible, la validation des certificats peut être retardée de plusieurs secondes — ou échouer si le délai d'attente est court. Configurez les paramètres OCSP/CRL en conséquence (section suivante).

!!! quote "En résumé"
    - Distribuez le certificat de la CA de code signing dans `Trusted Publishers` pour débloquer SmartScreen et AppLocker
    - Trusted Root et Trusted Publishers sont indépendants — les deux doivent être configurés pour du code signing interne
    - `certutil -setreg Chain\ChainCacheResyncFiletime @now` force un rafraîchissement du cache CTL

---

## Vérification de révocation : OCSP et CRL

### :material-shield-refresh: Pourquoi la révocation est critique

Quand Windows valide un certificat, il vérifie qu'il n'a pas été révoqué. Il utilise pour cela soit la **CRL** (Certificate Revocation List — une liste téléchargée périodiquement), soit **OCSP** (Online Certificate Status Protocol — une requête en temps réel).

Si la CRL est inaccessible ou expirée, Windows peut bloquer toutes les connexions TLS vers les services internes — même si les certificats sont parfaitement valides. C'est un scénario de production réel et dévastateur.

### :material-compare: OCSP vs CRL

| Mécanisme | Fonctionnement | Avantages | Inconvénients |
|---|---|---|---|
| CRL | Fichier téléchargé périodiquement | Simple, fonctionne hors ligne si cache valide | Mise à jour périodique, peut être volumineuse |
| OCSP | Requête HTTP en temps réel à chaque validation | Révocation quasi-instantanée, léger | Dépend de la disponibilité du répondeur OCSP |
| OCSP Stapling | Le serveur TLS inclut la réponse OCSP dans le handshake | Pas de requête client vers le répondeur | Nécessite support côté serveur |

En environnement d'entreprise, configurez les deux. OCSP en priorité, CRL en fallback.

### :material-registry: Paramètres de registre pour la révocation

Windows expose les paramètres de révocation dans :

```
HKLM\SOFTWARE\Policies\Microsoft\SystemCertificates\
```

Les valeurs clés à connaître et à configurer via GPO (Administrative Templates → System → Internet Communication Management) ou directement en registre :

| Clé de registre | Valeur | Effet |
|---|---|---|
| `HKLM\SOFTWARE\Microsoft\Cryptography\OID\...\CertDllCreateCertificateChainEngine\Config` | `ChainRevAccumulativeUrlRetrievalTimeoutMilliseconds` | Timeout global de toutes les requêtes de révocation (ms) |
| `ChainUrlRetrievalTimeoutMilliseconds` | `15000` (défaut) | Timeout par URL individuelle |
| `ChainRevocationCheckCacheEndTime` | Valeur de temps | Durée maximale de mise en cache d'une réponse CRL |

Pour forcer la préférence OCSP sur CRL quand les deux sont disponibles :

```powershell title="Configure OCSP preference over CRL via registry"
# Prefer OCSP over CRL for certificate revocation checking
$regPath = "HKLM:\SOFTWARE\Microsoft\Cryptography\OID\EncodingType 0\CertDllCreateCertificateChainEngine\Config"

# Set OCSP retrieval timeout (in milliseconds)
Set-ItemProperty -Path $regPath -Name "ChainUrlRetrievalTimeoutMilliseconds" -Value 10000 -Type DWord

# Reduce CRL cache resync time to detect revocations faster
Set-ItemProperty -Path $regPath -Name "ChainRevAccumulativeUrlRetrievalTimeoutMilliseconds" -Value 20000 -Type DWord

Write-Host "Revocation check timeouts configured."
```

### :material-alert: CRL inaccessible en production — procédure d'urgence

Une CRL inaccessible (serveur CRL tombé, réseau coupé, certificat CRL expiré) bloque immédiatement toutes les connexions TLS qui passent par votre CA interne.

**Diagnostic immédiat :**

```powershell title="Diagnose CRL accessibility"
# Check CRL URL accessibility from the affected machine
$crlUrl = "http://crl.contoso.local/CONTOSO-CA.crl"  # replace with your CRL URL

try {
    $response = Invoke-WebRequest -Uri $crlUrl -Method Head -TimeoutSec 5
    Write-Host "CRL accessible — HTTP $($response.StatusCode)" -ForegroundColor Green
} catch {
    Write-Host "CRL INACCESSIBLE: $($_.Exception.Message)" -ForegroundColor Red
}

# Check CRL cache validity on the local machine
certutil -urlcache CRL
```

**Si la CRL est inaccessible et que vous devez maintenir le service :**

```powershell title="Emergency — disable revocation checking (temporary)"
# EMERGENCY ONLY — disable revocation checking temporarily
# This reduces security. Document the incident and re-enable immediately after CRL is restored.
$regPath = "HKLM:\SOFTWARE\Microsoft\Cryptography\OID\EncodingType 0\CertDllCreateCertificateChainEngine\Config"

# Set a very long timeout to use cached CRL instead of failing
Set-ItemProperty -Path $regPath -Name "ChainUrlRetrievalTimeoutMilliseconds" -Value 1 -Type DWord

# After CRL restoration, reset to default
# Set-ItemProperty -Path $regPath -Name "ChainUrlRetrievalTimeoutMilliseconds" -Value 15000 -Type DWord
```

!!! danger "Production — CRL expirée = service coupé"
    Un certificat CRL a lui-même une date d'expiration. Si la CRL expire (la CA ne l'a pas régénérée), Windows refuse de valider **tous** les certificats émis par cette CA — peu importe que les certificats individuels soient valides. Sur une PKI d'entreprise, configurez une alerte sur la date `Next Update` de votre CRL. La durée de vie recommandée est 7 jours pour la CRL base, 24 heures pour la Delta CRL.

```powershell title="Check CRL expiry date"
# Check the expiry date of the CRL published by your CA
$crlUrl = "http://crl.contoso.local/CONTOSO-CA.crl"  # replace with your CRL URL

$webClient = New-Object System.Net.WebClient
$crlBytes  = $webClient.DownloadData($crlUrl)

$crl = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2
# CRL parsing requires certutil on Windows — use command line
certutil -url $crlUrl
```

```powershell title="Get CRL NextUpdate via certutil"
# Parse CRL metadata including NextUpdate date
certutil -dump "\\CA-SERVER\CertData\CONTOSO-CA.crl" | Select-String -Pattern "NextUpdate|Issuer"
```

```title="Résultat attendu"
Issuer: CN=CONTOSO-CA, DC=contoso, DC=local
NextUpdate: 2026/04/12 10:00
```

!!! quote "En résumé"
    - OCSP est préférable à CRL pour la rapidité de révocation — configurez les deux
    - `ChainUrlRetrievalTimeoutMilliseconds` contrôle le timeout de vérification — réduisez-le si vos connexions TLS sont lentes à établir
    - Une CRL expirée bloque tous les certificats de la CA — monitorez la date `Next Update`
    - En urgence, `certutil -urlcache CRL` montre ce qui est en cache sur la machine

---

## Store NTAuth : prérequis pour l'authentification par certificat

### :material-smart-card: Rôle du store NTAuth

Le store **NTAuth** est un objet LDAP stocké dans Active Directory qui liste les CA dont les certificats sont autorisés pour l'authentification Windows (smartcard logon, S4U2Self, authentification mutuelle LDAPS).

Quand un utilisateur s'authentifie avec un certificat (carte à puce ou certificat logiciel), le contrôleur de domaine vérifie que la CA ayant émis ce certificat est présente dans NTAuth. Si elle ne l'est pas, l'authentification échoue avec une erreur générique.

NTAuth est **distinct** du store Trusted Root. Un certificat peut être dans Trusted Root sans être dans NTAuth — et dans ce cas, la connexion HTTPS fonctionne mais le logon smartcard échoue.

### :material-location-enter: Publier le certificat CA dans NTAuth

```powershell title="Publish CA certificate to NTAuth store in Active Directory"
# Publish the CA certificate to the NTAuth store
# Run as Domain Admin or Enterprise Admin on a Domain Controller

$certPath = "C:\Temp\CONTOSO-CA.cer"  # path to your CA certificate file

# Add the CA certificate to the NTAuth store in Active Directory
certutil -dspublish -f $certPath NTAuthCA

Write-Host "CA certificate published to NTAuth store."
```

```title="Résultat attendu"
Certificate added to DS store.
CertUtil: -dspublish command completed successfully.
```

Si votre CA est une Enterprise CA intégrée à AD, cette publication est normalement effectuée automatiquement lors de l'installation. Vérifiez quand même — notamment après migration de CA ou renouvellement de certificat CA.

### :material-magnify: Vérifier le contenu du store NTAuth

```powershell title="Verify NTAuth store content"
# List all CA certificates in the NTAuth store
certutil -viewdelstore "ldap:///CN=NTAuthCertificates,CN=Public Key Services,CN=Services,CN=Configuration,DC=contoso,DC=local?cACertificate?base?objectClass=certificationAuthority"
```

```title="Résultat attendu"
================ Certificate 0 ================
Issuer: CN=CONTOSO-CA, DC=contoso, DC=local
Subject: CN=CONTOSO-CA, DC=contoso, DC=local
NotAfter: 2035-01-01 12:00:00
Thumbprint(sha1): A1B2C3D4E5F6...
CertUtil: -viewdelstore command completed successfully.
```

Vous pouvez aussi utiliser PowerShell pour vérifier via LDAP.

```powershell title="Check NTAuth via LDAP PowerShell"
# Query NTAuth store via Active Directory PowerShell
$configNC = (Get-ADRootDSE).configurationNamingContext
$ntauthDN = "CN=NTAuthCertificates,CN=Public Key Services,CN=Services,$configNC"

$ntauthObj = Get-ADObject -Identity $ntauthDN -Properties cACertificate

if ($ntauthObj.cACertificate) {
    Write-Host "NTAuth store contains $($ntauthObj.cACertificate.Count) certificate(s):"
    foreach ($certBytes in $ntauthObj.cACertificate) {
        $cert = [System.Security.Cryptography.X509Certificates.X509Certificate2]::new($certBytes)
        $cert | Select-Object Subject, NotAfter, Thumbprint
    }
} else {
    Write-Host "WARNING: NTAuth store is empty. Smartcard logon will fail." -ForegroundColor Red
}
```

```title="Résultat attendu"
NTAuth store contains 1 certificate(s):

Subject    : CN=CONTOSO-CA, DC=contoso, DC=local
NotAfter   : 2035-01-01 12:00:00
Thumbprint : A1B2C3D4E5F6...
```

### :material-sync: Propagation vers les machines

Le store NTAuth est répliqué vers toutes les machines via la Partition de Configuration d'Active Directory. Windows met en cache le contenu dans :

```
HKLM\SOFTWARE\Microsoft\EnterpriseCertificates\NTAuth\Certificates\
```

La mise à jour du cache se produit à chaque traitement GPO ou au bout de 8 heures. Pour forcer la synchronisation immédiate :

```powershell title="Force NTAuth cache refresh on local machine"
# Force NTAuth store synchronization from Active Directory
certutil -enterprise -addstore NTAuth "C:\Temp\CONTOSO-CA.cer"
```

!!! danger "Production — renouvellement de CA sans republication NTAuth"
    Quand vous renouvelez le certificat de votre CA (événement rare mais inévitable), le nouveau certificat CA doit être publié dans NTAuth avec `certutil -dspublish -f <nouveau-cert.cer> NTAuthCA`. L'ancien certificat reste valide pour les certificats qu'il a émis, mais les nouvelles émissions nécessitent le nouveau certificat CA dans NTAuth. L'oubli de cette étape provoque un échec de tous les logons smartcard après le renouvellement — souvent découvert en production un lundi matin.

!!! quote "En résumé"
    - NTAuth est distinct de Trusted Root — les deux doivent contenir le certificat CA pour un fonctionnement complet
    - `certutil -dspublish -f <cert.cer> NTAuthCA` publie dans AD
    - `certutil -viewdelstore "ldap:///CN=NTAuthCertificates..."` vérifie le contenu
    - Après renouvellement de CA, republication NTAuth obligatoire — documentez-la dans votre runbook PKI

---

## Pièges de production

### :material-bomb: Certificat expiré distribué via GPO

Si vous retirez un certificat d'une GPO mais que vous oubliez de sauvegarder la GPO et de la lier à nouveau, le certificat expiré reste dans le store machine jusqu'à la prochaine application GPO — potentiellement plusieurs jours.

Un certificat expiré dans le store Trusted Root peut provoquer des erreurs de validation en cascade, car Windows tente de construire une chaîne passant par ce certificat.

**Script de détection des certificats expirés dans les stores machine :**

```powershell title="Detect expired certificates in machine stores"
# Detect expired certificates in all machine certificate stores
$stores = @("Root", "CA", "My", "TrustedPublisher")
$now    = Get-Date

foreach ($storeName in $stores) {
    $store = New-Object System.Security.Cryptography.X509Certificates.X509Store($storeName, "LocalMachine")
    $store.Open("ReadOnly")

    $expired = $store.Certificates |
        Where-Object { $_.NotAfter -lt $now } |
        Select-Object @{N="Store";E={$storeName}}, Subject, NotAfter, Thumbprint

    if ($expired) {
        Write-Host "EXPIRED certificates in store '$storeName':" -ForegroundColor Yellow
        $expired | Format-Table -AutoSize
    }

    $store.Close()
}
```

```title="Résultat attendu (si certificats expirés détectés)"
EXPIRED certificates in store 'Root':

Store  Subject                          NotAfter             Thumbprint
-----  -------                          --------             ----------
Root   CN=OLD-CONTOSO-CA, DC=...        2023-06-01 12:00:00  DEADBEEF...
```

**Remédiation — suppression d'un certificat expiré sur un parc :**

```powershell title="Remove expired certificate from machine store via GPO"
# Remove an expired certificate from the machine Trusted Root store
# Deploy as a startup script or via Scheduled Task GPO

$thumbprintToRemove = "DEADBEEF1234..."  # replace with the expired cert thumbprint

$store = New-Object System.Security.Cryptography.X509Certificates.X509Store("Root", "LocalMachine")
$store.Open("ReadWrite")

$certToRemove = $store.Certificates |
    Where-Object { $_.Thumbprint -eq $thumbprintToRemove }

if ($certToRemove) {
    $store.Remove($certToRemove)
    Write-Host "Expired certificate removed: $thumbprintToRemove"
} else {
    Write-Host "Certificate not found in store — already removed or wrong thumbprint."
}

$store.Close()
```

### :material-bomb: Auto-enrollment sans vérification des permissions de template — échecs silencieux

C'est le piège le plus fréquent. L'admin active l'auto-enrollment en GPO, mais oublie d'accorder la permission `Autoenroll` sur le template. Résultat : aucun certificat n'est émis, aucune erreur visible dans GPMC, et l'utilisateur ne voit rien.

Le seul signal est l'**Event ID 82** dans le journal Application de la machine cliente.

```powershell title="Check auto-enrollment failures across multiple machines"
# Check for auto-enrollment failures on a remote machine
$targetMachine = "POSTE01"  # replace with target machine name

Invoke-Command -ComputerName $targetMachine -ScriptBlock {
    Get-WinEvent -FilterHashtable @{
        LogName      = 'Application'
        ProviderName = 'AutoEnrollment'
        Id           = 82
    } -MaxEvents 10 -ErrorAction SilentlyContinue |
    Select-Object TimeCreated, Message |
    Format-Table -Wrap -AutoSize
}
```

```title="Résultat attendu (si problème de permission)"
TimeCreated          Message
-----------          -------
2026-04-05 08:12:34  Automatic certificate enrollment for local system failed (0x80070005)
                     Access is denied. ...
```

**Vérification des permissions sur le template :**

```powershell title="Check template permissions for auto-enrollment"
# List security permissions on a certificate template
$templateName = "ComputerAuthentication"  # replace with your template name

$templateDN = "CN=$templateName,CN=Certificate Templates,CN=Public Key Services,CN=Services,$((Get-ADRootDSE).configurationNamingContext)"

$acl = Get-Acl "AD:$templateDN"
$acl.Access |
    Where-Object { $_.ActiveDirectoryRights -match "ExtendedRight" } |
    Select-Object IdentityReference, ActiveDirectoryRights, AccessControlType |
    Format-Table -AutoSize
```

```title="Résultat attendu (permissions correctes)"
IdentityReference        ActiveDirectoryRights   AccessControlType
-----------------        ---------------------   -----------------
CONTOSO\Domain Computers ExtendedRight           Allow
NT AUTHORITY\Authenticated Users  ExtendedRight  Allow
```

!!! danger "Production — double vérification avant activation de l'auto-enrollment"
    Avant d'activer l'auto-enrollment en GPO pour un nouveau template, testez manuellement depuis un poste de test avec `certutil -pulse` et vérifiez immédiatement l'Event ID 82. Un échec silencieux sur 2 000 postes sans aucun ticket d'incident est le pire scénario — l'absence de certificats ne se remarque qu'au moment où un service en dépend.

!!! quote "En résumé"
    - Un certificat expiré distribué via GPO reste en place jusqu'au prochain cycle GPO après retrait — nettoyez proactivement avec le script de détection
    - L'Event ID 82 (source `AutoEnrollment`) est le seul signal d'un échec d'auto-enrollment — monitorez-le en production
    - Testez `certutil -pulse` sur un poste de test avant tout déploiement d'auto-enrollment sur le parc complet

---

## Vérification globale post-déploiement

### :material-clipboard-check: Checklist de validation PKI complète

Après déploiement ou modification de votre configuration PKI via GPO, exécutez cette séquence de validation sur un poste représentatif.

```powershell title="Full PKI deployment validation checklist"
# Full PKI validation script — run on a target machine after GPO refresh
param(
    [string]$InternalCaThumbprint = "A1B2C3D4...",  # replace with your CA thumbprint
    [string]$InternalCaName      = "CONTOSO-CA"      # replace with your CA CN
)

$results = @()

# 1. Check Trusted Root store
$rootCert = Get-ChildItem Cert:\LocalMachine\Root |
    Where-Object { $_.Thumbprint -eq $InternalCaThumbprint }
$results += [PSCustomObject]@{
    Check  = "CA in Trusted Root store"
    Status = if ($rootCert) { "OK" } else { "FAIL" }
}

# 2. Check auto-enrollment — any certificate from internal CA in machine store
$enrolledCert = Get-ChildItem Cert:\LocalMachine\My |
    Where-Object { $_.Issuer -match $InternalCaName }
$results += [PSCustomObject]@{
    Check  = "Machine certificate enrolled from internal CA"
    Status = if ($enrolledCert) { "OK" } else { "FAIL — check Event ID 82" }
}

# 3. Check Trusted Publishers store
$trustedPub = Get-ChildItem Cert:\LocalMachine\TrustedPublisher |
    Where-Object { $_.Issuer -match $InternalCaName -or $_.Subject -match $InternalCaName }
$results += [PSCustomObject]@{
    Check  = "CA or signing cert in Trusted Publishers"
    Status = if ($trustedPub) { "OK" } else { "WARNING — code signing may be blocked" }
}

# 4. Check for auto-enrollment failures in last 24h
$ae82 = Get-WinEvent -FilterHashtable @{
    LogName      = 'Application'
    ProviderName = 'AutoEnrollment'
    Id           = 82
    StartTime    = (Get-Date).AddHours(-24)
} -ErrorAction SilentlyContinue

$results += [PSCustomObject]@{
    Check  = "No auto-enrollment failures (Event ID 82) in last 24h"
    Status = if (-not $ae82) { "OK" } else { "FAIL — $($ae82.Count) failure(s) found" }
}

# 5. Check for expired certificates in Trusted Root
$expiredRoot = Get-ChildItem Cert:\LocalMachine\Root |
    Where-Object { $_.NotAfter -lt (Get-Date) }
$results += [PSCustomObject]@{
    Check  = "No expired certificates in Trusted Root"
    Status = if (-not $expiredRoot) { "OK" } else { "WARNING — $($expiredRoot.Count) expired cert(s)" }
}

# Display summary
$results | Format-Table -AutoSize

$fails = $results | Where-Object { $_.Status -match "FAIL|WARNING" }
if ($fails) {
    Write-Host "`nAction required on $($fails.Count) check(s). Review items above." -ForegroundColor Yellow
} else {
    Write-Host "`nAll PKI checks passed." -ForegroundColor Green
}
```

```title="Résultat attendu"
Check                                              Status
-----                                              ------
CA in Trusted Root store                           OK
Machine certificate enrolled from internal CA      OK
CA or signing cert in Trusted Publishers           OK
No auto-enrollment failures (Event ID 82) in 24h  OK
No expired certificates in Trusted Root            OK

All PKI checks passed.
```


!!! quote "En résumé"
    - Validez toujours le résultat sur un poste ou un utilisateur réellement dans le périmètre avant d’élargir.
    - Conservez les commandes et résultats de contrôle comme preuve de conformité post-déploiement.
    - Cette section fixe l’essentiel à retenir sur vérification globale post-déploiement.
    - Retenez surtout ce qui change la portée, l’ordre d’application ou le résultat final observé.
    - Ce résumé sert à vérifier que vous avez retenu le mécanisme, sa portée et sa conséquence pratique.
---

## Cross-références

| Sujet | Référence |
|---|---|
| 802.1X Wi-Fi — nécessite certificat machine (PKI prérequis) | Ch. 16 — Wi-Fi, VPN et 802.1X |
| BitLocker avec récupération par certificat | Ch. 17 — BitLocker et LAPS |
| EFS Recovery Agent et certificats de récupération | La Bible GPO — Ch. 13 — Sécurité avancée |

!!! quote "En résumé"
    - À relire : 802.1X Wi-Fi — nécessite certificat machine (PKI prérequis) → Ch. 16 — Wi-Fi, VPN et 802.1X.
    - À relire : BitLocker avec récupération par certificat → Ch. 17 — BitLocker et LAPS.
    - À relire : EFS Recovery Agent et certificats de récupération → La Bible GPO — Ch. 13 — Sécurité avancée.
    - Ces renvois prolongent le chapitre avec des mécanismes complémentaires ou des cas d’usage voisins.
    - Gardez ces chapitres sous la main pour le diagnostic ou la conception d’une GPO liée à ce thème.
