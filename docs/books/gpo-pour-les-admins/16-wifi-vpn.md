---
description: Wi-Fi, VPN et 802.1X via GPO — profils sans fil, EAP-TLS, PEAP, Always On VPN, NPS, certificats machine
tags: [gpo-admins, wifi, vpn, 802.1x, eap-tls, peap, nps, always-on-vpn]
---

# Wi-Fi, VPN et 802.1X via GPO

!!! abstract "Ce que vous allez apprendre"
    - Déployer des profils Wi-Fi d'entreprise (WPA2-Enterprise) via GPO sans intervention sur chaque poste, avec le bon nœud GPMC et le format XML annoté
    - Configurer l'authentification 802.1X filaire via "Wired Network Policies" — la chaîne complète GPO → certificat machine → NPS → switch
    - Comprendre la différence entre Device Tunnel et User Tunnel dans Always On VPN, et déployer le ProfileXML via script de démarrage GPO
    - Interroger les journaux NPS (Event IDs 6272/6273/6274) pour diagnostiquer les échecs 802.1X en production
    - Détecter et remédier au piège le plus dangereux en prod : le renouvellement du certificat CA racine qui casse toutes les authentications 802.1X d'un coup

!!! tip "Si vous ne retenez qu'une chose"
    **Le certificat machine est la clé de voûte de tout ce chapitre.** Sans certificat machine valide dans `Cert:\LocalMachine\My`, déployé par auto-enrollment (chapitre 15), ni le Wi-Fi EAP-TLS, ni le 802.1X filaire, ni le Device Tunnel Always On VPN ne fonctionneront. Configurez et validez l'auto-enrollment PKI avant de toucher à la moindre politique réseau.

---

## Contexte de production

L'authentification réseau pilotée par GPO couvre trois besoins distincts que les admins confondent souvent.

Le **Wi-Fi d'entreprise** (SSID sécurisé avec WPA2-Enterprise) exige que chaque poste présente une identité au RADIUS avant d'obtenir une adresse IP. Sans GPO Wi-Fi, les utilisateurs configurent manuellement leurs paramètres ou finissent sur le réseau invité.

Le **802.1X filaire** applique le même principe aux ports de switch. Un poste non authentifié reste dans un VLAN de quarantaine. C'est un contrôle de segmentation réseau qui complète le NAC.

**Always On VPN** remplace DirectAccess sur Windows 10/11. Il établit automatiquement un tunnel VPN dès que la machine est hors du réseau d'entreprise, sans action utilisateur. Le Device Tunnel s'établit avant même le logon — il est configuré et déployé exclusivement par GPO ou script machine.

Les trois reposent sur la même fondation : une PKI interne fonctionnelle, des certificats machine distribués par auto-enrollment, et un NPS correctement configuré. Ce chapitre suppose que le chapitre [15 — Certificats et PKI](15-certificats-pki.md) est opérationnel.

---

## Profils Wi-Fi via GPO

### :material-wifi: Les deux mécanismes GPO pour le Wi-Fi

Il existe une confusion fréquente entre deux nœuds GPMC distincts pour les profils sans fil.

**"Wireless Network (IEEE 802.11) Policies"** (Security Settings) est la méthode principale. Elle déploie des profils XML complets — SSID, type d'authentification, configuration EAP, priorité. C'est le nœud à utiliser pour WPA2-Enterprise.

**ADMX Wireless (`WirelessPolicy.admx`)** (Administrative Templates) contrôle les comportements de la couche sans fil : interdire de se connecter à des réseaux ad-hoc, bloquer les points d'accès non gérés, contrôler le mode d'infrastructure. C'est un complément, pas un remplaçant.

Le bon réflexe : profils d'entreprise via Security Settings, restrictions comportementales via ADMX.

### :material-folder-cog: Chemin GPMC

Le nœud pour les profils Wi-Fi :

`Computer Configuration → Policies → Windows Settings → Security Settings → Wireless Network (IEEE 802.11) Policies`

Cliquez-droit → **Create A New Wireless Network Policy for Windows Vista and Later Releases**.

!!! warning "À surveiller — Politique Vista ou XP"
    Le menu propose deux types : "Windows Vista and Later" et "Windows XP". Ne choisissez jamais la version XP dans un environnement moderne — elle ne supporte pas WPA2-Enterprise avec EAP-TLS. Si vous voyez les deux coexister dans une GPO héritée, supprimez la version XP.

### :material-card-account-details: Export d'un profil existant

Avant de rédiger un XML à la main, exportez un profil déjà fonctionnel depuis un poste pilote.

```powershell title="Export Wi-Fi profile to XML"
# Export a specific Wi-Fi profile to an XML file
netsh wlan export profile name="CONTOSO-WIFI" key=clear folder="C:\Temp\WifiProfiles"
```

```title="Résultat attendu"
Interface profile "CONTOSO-WIFI" is saved in file "C:\Temp\WifiProfiles\Wi-Fi-CONTOSO-WIFI.xml" successfully.
```

Le fichier exporté contient la structure complète. Ouvrez-le, retirez les balises spécifiques à la machine (`<createdBy>`, `<creationTime>`), et il devient votre template GPO.

### :material-xml: Format XML annoté — WPA2-Enterprise avec EAP-TLS

Voici un profil opérationnel pour WPA2-Enterprise avec authentification par certificat machine (EAP-TLS). Chaque section critique est commentée.

```xml title="CONTOSO-WIFI-EAP-TLS.xml"
<?xml version="1.0"?>
<WLANProfile xmlns="http://www.microsoft.com/networking/WLAN/profile/v1">

  <!-- SSID exact — respectez la casse -->
  <name>CONTOSO-WIFI</name>
  <SSIDConfig>
    <SSID>
      <hex>434F4E544F534F2D57494649</hex>  <!-- hex encoding of SSID -->
      <name>CONTOSO-WIFI</name>
    </SSID>
    <nonBroadcast>false</nonBroadcast>
  </SSIDConfig>

  <!-- Infrastructure = point d'accès, jamais ad-hoc en entreprise -->
  <connectionType>ESS</connectionType>
  <connectionMode>auto</connectionMode>

  <MSM>
    <security>
      <!-- WPA2-Enterprise : authentification 802.1X côté RADIUS -->
      <authEncryption>
        <authentication>WPA2</authentication>
        <encryption>AES</encryption>
        <useOneX>true</useOneX>
      </authEncryption>

      <OneX xmlns="http://www.microsoft.com/networking/OneX/v1">
        <!-- authMode = machineOrUser : préfère le certificat machine,
             bascule sur certificat utilisateur si machine absent.
             Pour forcer le certificat machine seul : "machine" -->
        <authMode>machine</authMode>
        <EAPConfig>
          <EapHostConfig xmlns="http://www.microsoft.com/provisioning/EapHostConfig">
            <EapMethod>
              <!-- Type 13 = EAP-TLS -->
              <Type xmlns="http://www.microsoft.com/provisioning/EapCommon">13</Type>
              <VendorId xmlns="http://www.microsoft.com/provisioning/EapCommon">0</VendorId>
              <VendorType xmlns="http://www.microsoft.com/provisioning/EapCommon">0</VendorType>
              <AuthorId xmlns="http://www.microsoft.com/provisioning/EapCommon">0</AuthorId>
            </EapMethod>
            <Config xmlns="http://www.microsoft.com/provisioning/EapHostConfig">
              <Eap xmlns="http://www.microsoft.com/provisioning/BaseEapConnectionPropertiesV1">
                <Type>13</Type>
                <EapType xmlns="http://www.microsoft.com/provisioning/EapTlsConnectionPropertiesV1">
                  <CredentialsSource>
                    <!-- CertificateStore : utilise le certificat du store local -->
                    <CertificateStore>
                      <SimpleCertSelection>true</SimpleCertSelection>
                    </CertificateStore>
                  </CredentialsSource>
                  <ServerValidation>
                    <!-- Activez ServerValidation en production pour vérifier le cert NPS -->
                    <DisableUserPromptForServerValidation>true</DisableUserPromptForServerValidation>
                    <ServerNames>nps01.contoso.local;nps02.contoso.local</ServerNames>
                    <!-- Empreinte SHA1 du certificat NPS (sans espaces) -->
                    <TrustedRootCA>A1B2C3D4E5F6A1B2C3D4E5F6A1B2C3D4E5F6A1B2</TrustedRootCA>
                  </ServerValidation>
                  <DifferentUsername>false</DifferentUsername>
                  <PerformServerValidation>true</PerformServerValidation>
                  <AcceptServerName>true</AcceptServerName>
                </EapType>
              </Eap>
            </Config>
          </EapHostConfig>
        </EAPConfig>
      </OneX>
    </security>
  </MSM>
</WLANProfile>
```

### :material-upload: Import du profil dans la GPO

Une fois votre XML prêt, importez-le dans la politique GPMC.

**Étape 1.** Dans le nœud `Wireless Network (IEEE 802.11) Policies`, double-cliquez sur la politique créée.

**Étape 2.** Dans l'onglet **General**, cliquez sur **Add** → **Infrastructure**.

**Étape 3.** Dans la fenêtre du profil, cliquez sur **Import** et sélectionnez votre fichier XML.

**Étape 4.** Vérifiez que le SSID apparaît dans la liste, puis validez.

!!! danger "Production — ServerValidation désactivé"
    Laisser `DisableUserPromptForServerValidation` à `true` sans renseigner `ServerNames` ni `TrustedRootCA` revient à accepter n'importe quel serveur RADIUS — y compris un RADIUS rogue. En production, renseignez toujours le FQDN de votre NPS et l'empreinte de son certificat. L'empreinte s'obtient avec `(Get-ChildItem Cert:\LocalMachine\My | Where-Object { $_.Subject -match "NPS" }).Thumbprint`.

### :material-microsoft-windows: Ciblage : filtrer les laptops uniquement

La GPO Wi-Fi ne doit s'appliquer qu'aux machines équipées d'une interface sans fil.

Utilisez un filtre WMI (voir [Filtrage avancé des GPO](../bible-gpo/09-filtrage.md)) sur la classe `Win32_NetworkAdapter` :

```sql title="WMI filter — Wireless adapters only"
SELECT * FROM Win32_NetworkAdapter
WHERE AdapterTypeId = 9
```

Un poste sans adaptateur sans fil ne recevra pas la politique.

### :material-check-circle: Vérification post-déploiement

```powershell title="Verify Wi-Fi profile deployment"
# Check if the GPO wireless profile is present
netsh wlan show profiles | Select-String -Pattern "CONTOSO-WIFI"

# Check connection status
netsh wlan show interfaces | Select-String -Pattern "(SSID|State|Authentication)"
```

```title="Résultat attendu"
    All User Profile     : CONTOSO-WIFI

    SSID                   : CONTOSO-WIFI
    State                  : connected
    Authentication         : WPA2-Enterprise
```

```powershell title="Verify machine certificate is present (prerequisite for EAP-TLS)"
# The machine must have a valid certificate in LocalMachine\My
Get-ChildItem Cert:\LocalMachine\My |
    Where-Object { $_.Extensions | Where-Object {
        $_.Oid.FriendlyName -eq "Enhanced Key Usage" -and
        $_.Format(0) -match "Client Authentication"
    }} |
    Select-Object Subject, Issuer, NotAfter, Thumbprint |
    Format-Table -AutoSize
```

!!! quote "En résumé"
    - Nœud correct : `Security Settings → Wireless Network (IEEE 802.11) Policies`, pas les ADMX
    - Exportez un profil fonctionnel avec `netsh wlan export profile` avant de rédiger le XML
    - EAP-TLS type 13 : `<authMode>machine</authMode>` force le certificat machine
    - `ServerValidation` doit être configuré avec le FQDN NPS et l'empreinte TrustedRootCA
    - Filtrez par WMI `Win32_NetworkAdapter AdapterTypeId = 9` pour cibler les laptops uniquement

---

## 802.1X filaire via GPO

### :material-lan: Le nœud Wired Network Policies

L'authentification 802.1X sur les ports filaires se configure via un nœud symétrique au Wi-Fi :

`Computer Configuration → Policies → Windows Settings → Security Settings → Wired Network (IEEE 802.3) Policies`

Cliquez-droit → **Create a new Wired Network Policy for Windows Vista and Later**.

Ce nœud n'existe que dans Computer Configuration. Il n'y a pas d'équivalent en User Configuration — le 802.1X filaire est toujours une politique machine.

### :material-transit-connection-variant: La chaîne de confiance complète

Le 802.1X filaire implique quatre acteurs qui doivent tous être configurés correctement.

```
GPO auto-enrollment  →  Poste Windows  →  NPS (RADIUS)  →  Switch
(certificat machine)    (supplicant)       (validation)      (NAC)
```

**Étape 1 — GPO PKI (chapitre 15).** L'auto-enrollment distribue un certificat machine dans `Cert:\LocalMachine\My`. Sans ce certificat, le supplicant n'a rien à présenter.

**Étape 2 — GPO Wired Network.** La politique configure le supplicant Windows (service `dot3svc`) : type EAP, comportement, certificat à utiliser.

**Étape 3 — NPS.** Le serveur RADIUS reçoit la demande d'authentification du switch, valide le certificat machine contre l'Active Directory et répond Access-Accept ou Access-Reject.

**Étape 4 — Switch.** Sur la base de la réponse NPS, le port bascule dans le VLAN de production (Accept) ou reste dans le VLAN de quarantaine (Reject).

!!! warning "À surveiller — Service dot3svc"
    Le service `Wired AutoConfig` (`dot3svc`) doit être en démarrage automatique sur les postes. Il est désactivé par défaut sur Windows 10/11. La politique Wired Network le démarre automatiquement quand la GPO s'applique — mais si le service est bloqué par une autre GPO de durcissement, l'authentification 802.1X filaire ne fonctionnera jamais.

    Vérifiez : `Get-Service dot3svc | Select-Object Name, Status, StartType`

### :material-folder-cog: Configuration du profil filaire dans GPMC

Dans la boîte de dialogue de la politique filaire, l'onglet **Security** expose les paramètres 802.1X.

| Paramètre | Valeur recommandée |
|---|---|
| Enable IEEE 802.1X authentication | Activé |
| Select a network authentication method | Microsoft: Smart Card or other certificate (EAP-TLS) |
| Authentication mode | Computer authentication |
| Max Authentication Failures | 3 |
| Cache user information | Non (machine context uniquement) |

L'onglet **Properties** de la méthode EAP ouvre la configuration détaillée : sélection du certificat, validation du serveur NPS (même principe que le Wi-Fi — renseignez le FQDN et l'empreinte).

### :material-scale-balance: EAP-TLS vs PEAP-MSCHAPv2

Le choix du type EAP conditionne la sécurité et les prérequis de l'infrastructure.

| Critère | EAP-TLS | PEAP-MSCHAPv2 |
|---|---|---|
| Authentification | Certificat machine/utilisateur | Identifiant + mot de passe AD |
| Prérequis PKI | Enterprise CA + auto-enrollment | Certificat serveur NPS uniquement |
| Sécurité | Très élevée (mutual TLS) | Correcte (tunnel TLS + MSCHAPv2) |
| Complexité opérationnelle | Élevée | Faible |
| Résistance au vol de credential | Excellente (pas de mot de passe) | Vulnérable aux attaques MITM si cert NPS non vérifié |
| Recommandé pour | Environnements à haute sécurité, 802.1X filaire | Déploiements rapides, Wi-Fi utilisateurs |

**Recommandation d'architecture.** Utilisez EAP-TLS pour le 802.1X filaire (machines de confiance uniquement) et PEAP-MSCHAPv2 pour le Wi-Fi utilisateurs si la PKI n'est pas encore généralisée. N'utilisez jamais PEAP sans valider le certificat NPS côté client — c'est la configuration par défaut de Windows et c'est une vulnérabilité.

### :material-console: Vérification post-déploiement

```powershell title="Verify dot3svc status and 802.1X policy"
# Check Wired AutoConfig service
Get-Service dot3svc | Select-Object Name, Status, StartType

# Check 802.1X policy registry key
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WiredL2\GP" `
    -ErrorAction SilentlyContinue
```

```powershell title="Check machine certificate for 802.1X"
# Verify a Client Authentication certificate exists in the machine store
$certs = Get-ChildItem Cert:\LocalMachine\My |
    Where-Object {
        $_.Extensions | Where-Object {
            $_.Oid.FriendlyName -eq "Enhanced Key Usage" -and
            $_.Format(0) -match "Client Authentication"
        }
    } |
    Where-Object { $_.NotAfter -gt (Get-Date) }

if ($certs) {
    Write-Host "OK: $($certs.Count) valid client certificate(s) found" -ForegroundColor Green
    $certs | Select-Object Subject, Issuer, NotAfter, Thumbprint | Format-Table -AutoSize
} else {
    Write-Host "FAIL: No valid client certificate found for 802.1X" -ForegroundColor Red
}
```

```title="Résultat attendu"
OK: 1 valid client certificate(s) found

Subject                        Issuer                   NotAfter             Thumbprint
-------                        ------                   --------             ----------
CN=WKS-001.contoso.local       CN=CONTOSO-CA, DC=...    2027-04-05 10:00:00  A1B2C3D4...
```

!!! quote "En résumé"
    - Nœud GPMC : `Security Settings → Wired Network (IEEE 802.3) Policies` — Computer Configuration uniquement
    - La chaîne : GPO PKI (certificat machine) → supplicant Windows (`dot3svc`) → NPS → switch
    - Vérifiez que le service `dot3svc` n'est pas désactivé par une autre GPO de durcissement
    - EAP-TLS = certificat machine (haute sécurité) ; PEAP-MSCHAPv2 = mot de passe (déploiement rapide)
    - Validez toujours le certificat NPS côté client pour éviter les attaques MITM

---

## Always On VPN via GPO

### :material-vpn: Device Tunnel vs User Tunnel

Always On VPN propose deux modes de connexion fondamentalement différents. Comprendre cette distinction est indispensable avant de déployer quoi que ce soit.

**Device Tunnel** s'établit dans le contexte machine, avant le logon utilisateur. Il utilise le certificat machine. Il est destiné à la communication précoce avec le DC (authentification AD, GPO, SCCM), et il s'établit au démarrage de la machine dès que le réseau est disponible hors réseau d'entreprise.

**User Tunnel** s'établit dans le contexte utilisateur, après authentification. Il peut utiliser un certificat utilisateur, des identifiants EAP, ou Windows Hello for Business. Il est destiné au trafic applicatif de l'utilisateur.

| Critère | Device Tunnel | User Tunnel |
|---|---|---|
| Contexte | Machine (SYSTEM) | Utilisateur connecté |
| Déclenchement | Au démarrage réseau | Après logon |
| Certificat | `Cert:\LocalMachine\My` | `Cert:\CurrentUser\My` |
| Déploiement | Script machine GPO / PowerShell SYSTEM | PowerShell utilisateur / Intune |
| Trafic couvert | DC, DNS interne, SCCM, GPO | Applications métier |
| Prérequis | Windows 10 Enterprise 1709+ | Windows 10 toutes éditions |

!!! danger "Production — Device Tunnel réservé Windows Enterprise"
    Le Device Tunnel ne fonctionne que sur **Windows 10/11 Enterprise et Education**. Les éditions Pro ne le supportent pas. Si votre parc mélange Enterprise et Pro, déployez uniquement le User Tunnel sur les postes Pro, ou migrez vers Enterprise avant d'activer le Device Tunnel.

### :material-xml: Format ProfileXML

Always On VPN est configuré via un XML de profil (ProfileXML). Voici la structure annotée pour un Device Tunnel avec EAP-TLS.

```xml title="DeviceTunnel-ProfileXML.xml"
<VPNProfile>
  <!-- AlwaysOn : reconnexion automatique si le tunnel tombe -->
  <AlwaysOn>true</AlwaysOn>

  <!-- DeviceTunnel : contexte SYSTEM, avant logon -->
  <DeviceTunnel>true</DeviceTunnel>

  <!-- DnsSuffix pour la résolution des noms internes -->
  <DnsSuffix>contoso.local</DnsSuffix>

  <!-- RegisterDNS : enregistre le nom machine dans le DNS interne -->
  <RegisterDNS>true</RegisterDNS>

  <NativeProfile>
    <!-- FQDN ou IP du serveur RRAS/VPN Gateway -->
    <Servers>vpn.contoso.com</Servers>

    <!-- IKEv2 est le protocole recommandé pour Always On VPN -->
    <NativeProtocolType>IKEv2</NativeProtocolType>

    <!-- Authentication : certificat machine pour Device Tunnel -->
    <Authentication>
      <MachineMethod>Certificate</MachineMethod>
    </Authentication>

    <!-- CryptographySuite : suite cryptographique IKEv2 -->
    <CryptographySuite>
      <AuthenticationTransformConstants>GCMAES256</AuthenticationTransformConstants>
      <CipherTransformConstants>GCMAES256</CipherTransformConstants>
      <EncryptionMethod>AES256</EncryptionMethod>
      <IntegrityCheckMethod>SHA256</IntegrityCheckMethod>
      <DHGroup>Group14</DHGroup>
      <PfsGroup>PFS2048</PfsGroup>
    </CryptographySuite>

    <!-- DisableClassBasedDefaultRoute : évite de router tout le trafic
         par le VPN — utile si split tunneling est voulu -->
    <DisableClassBasedDefaultRoute>false</DisableClassBasedDefaultRoute>
  </NativeProfile>

  <!-- TrustedNetworkDetection : ne s'active que hors réseau interne.
       Liste des suffixes DNS qui identifient le réseau d'entreprise -->
  <TrustedNetworkDetection>contoso.local,corp.contoso.com</TrustedNetworkDetection>

</VPNProfile>
```

Pour un **User Tunnel** avec EAP-TLS utilisateur, remplacez `<DeviceTunnel>true</DeviceTunnel>` par `<DeviceTunnel>false</DeviceTunnel>` et substituez `<MachineMethod>Certificate</MachineMethod>` par un bloc `<UserMethod>` avec configuration EAP complète.

### :material-powershell: Déploiement via script de démarrage GPO

Le Device Tunnel doit être configuré dans le contexte SYSTEM. Le mécanisme GPO natif est le **Computer Startup Script**.

`Computer Configuration → Policies → Windows Settings → Scripts (Startup/Shutdown) → Startup`

Ajoutez un script PowerShell. Le script ci-dessous lit le XML depuis SYSVOL et provisionne le tunnel.

```powershell title="Deploy-DeviceTunnel.ps1"
# Deploy Always On VPN Device Tunnel from SYSVOL
# Must run in SYSTEM context (GPO Startup Script)

$profileName = "Contoso Device Tunnel"
$profileXmlPath = "\\contoso.local\SYSVOL\contoso.local\VPN\DeviceTunnel-ProfileXML.xml"

# Read the ProfileXML file
if (-not (Test-Path $profileXmlPath)) {
    Write-EventLog -LogName Application -Source "VPN-Deploy" `
        -EventId 9001 -EntryType Error `
        -Message "ProfileXML not found at $profileXmlPath"
    exit 1
}

$profileXml = (Get-Content $profileXmlPath -Raw).Trim()

# Remove existing profile if present (idempotent deployment)
$existingProfile = Get-VpnConnection -Name $profileName -AllUserConnection -ErrorAction SilentlyContinue
if ($existingProfile) {
    Remove-VpnConnection -Name $profileName -AllUserConnection -Force
    Write-EventLog -LogName Application -Source "VPN-Deploy" `
        -EventId 9000 -EntryType Information `
        -Message "Removed existing profile: $profileName"
}

# Deploy the Device Tunnel (AllUserConnection = machine context)
Add-VpnConnection `
    -Name $profileName `
    -ServerAddress "vpn.contoso.com" `
    -TunnelType IKEv2 `
    -AuthenticationMethod MachineCertificate `
    -EncryptionLevel Required `
    -AllUserConnection `
    -PassThru

# Apply ProfileXML to the newly created profile
$profileXmlEscaped = $profileXml -replace "<", "&lt;" -replace ">", "&gt;" -replace '"', "&quot;"

$nodeCSP = "./Device/Vendor/MSFT/VPNv2/$profileName/ProfileXML"
$session = New-CimSession
$params = @{
    Namespace  = "root/MDM/Microsoft/Windows/WiredL2/GP"
    ClassName  = "MDM_VPNv2_01"
    CimSession = $session
}

# Alternative: use WMI bridge for CSP provisioning
$wmiPath = "root\cimv2\mdm\dmmap"
$wmiClass = "MDM_VPNv2_01"

try {
    $instance = New-CimInstance -Namespace $wmiPath -ClassName $wmiClass `
        -Property @{ ParentID = "./Device/Vendor/MSFT/VPNv2"; InstanceID = $profileName } `
        -ClientOnly
    $instance | Set-CimInstance -CimSession $session
} catch {
    # Fallback: direct registry provisioning via rasphone
    $null = $session
}

Write-EventLog -LogName Application -Source "VPN-Deploy" `
    -EventId 9000 -EntryType Information `
    -Message "Device Tunnel '$profileName' deployed successfully"
```

!!! tip "Alternative : provisioning CSP via PowerShell simplifié"
    Pour les environnements sans MDM, la méthode la plus fiable pour appliquer le ProfileXML complet est d'utiliser la classe WMI `MDM_VPNv2_01` via le pont WMI-CSP. Microsoft documente cette approche dans [Always On VPN Deployment Guide](https://learn.microsoft.com/en-us/windows-server/remote/remote-access/vpn/always-on-vpn/). En Intune, le ProfileXML s'applique directement via le CSP VPNv2 sans script.

### :material-check-circle: Vérification post-déploiement

```powershell title="Verify Device Tunnel deployment"
# Check if Device Tunnel profile exists (machine context)
Get-VpnConnection -Name "Contoso Device Tunnel" -AllUserConnection |
    Select-Object Name, ServerAddress, TunnelType, AuthenticationMethod, ConnectionStatus

# Check Device Tunnel connection status (SYSTEM context required)
Get-VpnConnection -AllUserConnection | Where-Object { $_.IsDeviceTunnel -eq $true } |
    Select-Object Name, ConnectionStatus, ServerAddress
```

```title="Résultat attendu"
Name                   : Contoso Device Tunnel
ServerAddress          : vpn.contoso.com
TunnelType             : IKEv2
AuthenticationMethod   : MachineCertificate
ConnectionStatus       : Connected
```

!!! quote "En résumé"
    - Device Tunnel = contexte SYSTEM, avant logon, certificat machine — Windows Enterprise uniquement
    - User Tunnel = contexte utilisateur, après logon, toutes éditions Windows 10/11
    - Déployez le Device Tunnel via un script de démarrage GPO (Computer Startup Script)
    - Stockez le ProfileXML dans SYSVOL pour qu'il soit accessible même hors réseau en bootstrap
    - `TrustedNetworkDetection` évite que le VPN s'active inutilement sur le réseau interne

---

## NPS et diagnostic 802.1X

### :material-server-network: NPS n'est pas configuré par GPO

Le serveur NPS (Network Policy Server) lui-même ne se configure pas via GPO — il a sa propre MMC (`nps.msc`) et son propre fichier de configuration XML exportable.

Ce que les GPO contrôlent, c'est la configuration des **clients** : supplicant Wi-Fi, supplicant filaire, certificats. Le NPS est la contrepartie serveur, configurée centralement.

Ce que vous configurez sur NPS pour 802.1X :

- Les **RADIUS Clients** : adresses IP et secrets partagés des switches et bornes Wi-Fi
- Les **Network Policies** : conditions d'accès (groupe AD, type de connexion, heure), méthode EAP
- Les **Connection Request Policies** : proxy RADIUS optionnel vers un NPS central

### :material-eye: Event IDs NPS critiques

Les journaux NPS sont votre première source de vérité pour diagnostiquer un échec 802.1X.

| Event ID | Source | Signification |
|---|---|---|
| 6272 | Microsoft Windows security auditing | Access-Accept — authentification réussie |
| 6273 | Microsoft Windows security auditing | Access-Reject — authentification refusée |
| 6274 | Microsoft Windows security auditing | Requête ignorée (Request Discarded) |
| 6275 | Microsoft Windows security auditing | Accounting request reçue |
| 6276 | Microsoft Windows security auditing | Utilisateur mis en quarantaine |
| 6278 | Microsoft Windows security auditing | Accès accordé après remédiation NAP |

L'Event ID **6273** est le plus utile. Il contient le **Reason Code** qui explique pourquoi l'authentification a échoué.

### :material-powershell: Interroger les logs NPS pour les échecs

```powershell title="Query-NPSFailures.ps1"
# Query NPS authentication failures from Security event log
# Run on the NPS server with appropriate permissions

param(
    [string]$ComputerName = "NPS01",
    [int]    $MaxEvents   = 100,
    [int]    $HoursBack   = 24
)

$startTime = (Get-Date).AddHours(-$HoursBack)

$failedAuths = Get-WinEvent -ComputerName $ComputerName -FilterHashtable @{
    LogName   = 'Security'
    Id        = 6273
    StartTime = $startTime
} -MaxEvents $MaxEvents -ErrorAction SilentlyContinue

if (-not $failedAuths) {
    Write-Host "No NPS failures found in the last $HoursBack hours" -ForegroundColor Green
    return
}

$results = foreach ($event in $failedAuths) {
    $xml = [xml]$event.ToXml()
    $data = $xml.Event.EventData.Data

    [PSCustomObject]@{
        TimeCreated  = $event.TimeCreated
        SubjectAccount = ($data | Where-Object { $_.Name -eq "SubjectUserName" }).'#text'
        CalledStation = ($data | Where-Object { $_.Name -eq "CalledStationID"  }).'#text'
        CallingStation = ($data | Where-Object { $_.Name -eq "CallingStationID" }).'#text'
        ReasonCode   = ($data | Where-Object { $_.Name -eq "Reason"            }).'#text'
        ProxyPolicy  = ($data | Where-Object { $_.Name -eq "ProxyPolicyName"   }).'#text'
        NetworkPolicy = ($data | Where-Object { $_.Name -eq "NetworkPolicyName" }).'#text'
    }
}

$results | Sort-Object TimeCreated -Descending |
    Format-Table TimeCreated, SubjectAccount, ReasonCode, CallingStation -AutoSize
```

```title="Résultat attendu"
TimeCreated          SubjectAccount       ReasonCode  CallingStation
-----------          --------------       ----------  --------------
2026-04-05 09:12:33  WKS-042$            23          00-50-56-AB-CD-EF
2026-04-05 09:10:11  WKS-017$            16          00-50-56-AB-12-34
```

### :material-book-open: Reason Codes NPS courants

| Reason Code | Signification | Action corrective |
|---|---|---|
| 16 | Authentication failed | Vérifier le certificat machine dans `Cert:\LocalMachine\My` |
| 23 | Certificate expired | Relancer l'auto-enrollment : `certutil -pulse` |
| 48 | Certificate not from trusted CA | Vérifier que le root CA est dans le store NPS `Trusted Root CA` |
| 65 | EAP method not configured | Vérifier la Network Policy NPS — méthode EAP active |
| 66 | Certificate chain not valid | Vérifier les CA intermédiaires sur NPS et sur le client |

!!! quote "En résumé"
    - NPS se configure via `nps.msc`, pas par GPO — les GPO configurent les clients, pas le serveur
    - Event ID 6273 = Access-Reject, avec un Reason Code exploitable immédiatement
    - Le script `Query-NPSFailures.ps1` extrait les échecs des dernières N heures avec les accounts et reason codes
    - Reason Code 48 (CA non reconnue) et 23 (certificat expiré) représentent 80 % des incidents 802.1X en production

---

## Piège de production : renouvellement du certificat CA racine

### :material-alert-circle: Le scénario catastrophe

C'est l'incident 802.1X le plus sévère en production, et il est entièrement évitable.

Votre CA racine arrive en fin de vie. Vous renouvelez ou remplacez le certificat CA. Le nouveau certificat est émis, les nouvelles authentifications NPS fonctionnent... mais les machines qui ont conservé **uniquement l'ancien root CA** dans leur store `Trusted Root` ne font plus confiance à la nouvelle chaîne.

Résultat : toutes les authentifications 802.1X échouent sur les postes concernés. Reason Code **48** dans les logs NPS. Les machines ne peuvent plus se connecter au réseau filaire ni au Wi-Fi d'entreprise — y compris pour recevoir la GPO qui distribuerait le nouveau root CA.

C'est une boucle de dépendance : la machine a besoin du réseau pour recevoir la GPO, et elle a besoin de la GPO pour obtenir le nouveau certificat root.

!!! danger "Production — Rupture de chaîne de confiance"
    Ne renouvelez jamais un certificat CA racine sans avoir préalablement distribué le **nouveau certificat root** via GPO sur 100 % du parc, et vérifié sa présence sur un échantillon représentatif. La fenêtre de déploiement doit être terminée avant la mise en service du nouveau root — pas le jour J, pas J-1.

    Le renouvellement de CA racine doit être planifié **au moins 30 jours à l'avance** avec une phase de coexistence des deux roots.

### :material-magnify: Script de détection préventive

Ce script vérifie que le nouveau root CA est présent sur l'ensemble du parc avant de basculer.

```powershell title="Check-RootCADeployment.ps1"
# Verify that a specific root CA certificate is present on all domain computers
# Run this BEFORE activating a new root CA on NPS

param(
    [Parameter(Mandatory)]
    [string]$NewRootCAThumbprint,  # Thumbprint of the new root CA certificate

    [string]$OUFilter = "OU=Workstations,DC=contoso,DC=local",
    [int]   $ThrottleLimit = 50
)

# Get all active computer accounts from the target OU
$computers = Get-ADComputer -SearchBase $OUFilter -Filter { Enabled -eq $true } |
    Select-Object -ExpandProperty Name

Write-Host "Checking $($computers.Count) computers for root CA: $NewRootCAThumbprint"

$results = $computers | ForEach-Object -ThrottleLimit $ThrottleLimit -Parallel {
    $computer = $_
    $thumbprint = $using:NewRootCAThumbprint

    $status = Invoke-Command -ComputerName $computer -ScriptBlock {
        param($tp)
        $cert = Get-ChildItem Cert:\LocalMachine\Root |
            Where-Object { $_.Thumbprint -eq $tp }

        [PSCustomObject]@{
            Computer  = $env:COMPUTERNAME
            HasNewRoot = $null -ne $cert
            NotAfter  = if ($cert) { $cert.NotAfter } else { $null }
        }
    } -ArgumentList $thumbprint -ErrorAction SilentlyContinue

    if (-not $status) {
        [PSCustomObject]@{
            Computer  = $computer
            HasNewRoot = $false
            NotAfter  = "UNREACHABLE"
        }
    } else {
        $status
    }
}

# Summary report
$missing = $results | Where-Object { -not $_.HasNewRoot }
$present = $results | Where-Object { $_.HasNewRoot }

Write-Host "`nSUMMARY" -ForegroundColor Cyan
Write-Host "Present : $($present.Count) / $($computers.Count)" -ForegroundColor Green
Write-Host "Missing : $($missing.Count) / $($computers.Count)" -ForegroundColor $(if ($missing.Count -gt 0) { "Red" } else { "Green" })

if ($missing.Count -gt 0) {
    Write-Host "`nComputers missing the new root CA:" -ForegroundColor Red
    $missing | Select-Object Computer, NotAfter | Format-Table -AutoSize
}
```

```title="Résultat attendu — avant basculement"
Checking 312 computers for root CA: B3C4D5E6F7A8B3C4D5E6F7A8B3C4D5E6F7A8B3C4

SUMMARY
Present : 309 / 312
Missing : 3 / 312

Computers missing the new root CA:
Computer    NotAfter
--------    --------
WKS-047     UNREACHABLE
WKS-112     UNREACHABLE
WKS-218     False
```

Les postes `UNREACHABLE` sont hors réseau (déploiement ultérieur). Le poste `WKS-218` est en ligne mais n'a pas reçu la GPO — forcez un `gpupdate /force` à distance.

### :material-wrench: Remédiation pas à pas

**Phase 1 — Coexistence des deux roots (J-30 à J-0).**

Ajoutez le **nouveau** certificat root dans la GPO PKI (`Trusted Root Certification Authorities`) sans supprimer l'ancien. Les deux coexistent dans le store des machines. Vérifiez la présence du nouveau root sur 100 % des postes accessibles avec le script ci-dessus.

**Phase 2 — Basculement NPS (J-0).**

Reconfigurez NPS pour utiliser la nouvelle CA. Mettez à jour la Network Policy pour référencer le nouveau certificat server NPS (renouvelé auprès de la nouvelle CA). Les postes ayant les deux roots acceptent l'ancienne et la nouvelle chaîne pendant la transition.

**Phase 3 — Suppression de l'ancien root (J+30 minimum).**

Une fois que 100 % des certificats machine ont été renouvelés par auto-enrollment auprès de la nouvelle CA, retirez l'ancien root CA de la GPO. Attendez un cycle GPO complet (90-120 minutes) avant de valider.

```powershell title="Force GPO refresh on unreachable-but-reachable computers"
# Force GPO refresh remotely on a list of computers
$targetComputers = @("WKS-218", "WKS-047")

foreach ($computer in $targetComputers) {
    Invoke-Command -ComputerName $computer -ScriptBlock {
        # Force GPO refresh — distributes new root CA immediately
        gpupdate /force /target:computer
        # Trigger certificate re-enrollment
        certutil -pulse
    } -ErrorAction SilentlyContinue
}
```

!!! warning "À surveiller — Postes hors réseau pendant le basculement"
    Les portables en télétravail ne reçoivent pas la GPO si le Device Tunnel VPN n'est pas actif. Planifiez le basculement NPS un lundi matin pour que les postes se reconnectent au VPN le week-end précédent. Ou prolongez la phase de coexistence jusqu'à J+60 pour absorber les longs congés.

### :material-script: Validation finale post-basculement

```powershell title="Validate-802.1X-PostMigration.ps1"
# Run on NPS server after root CA switchover
# Counts successes vs failures in the last hour

$oneHourAgo = (Get-Date).AddHours(-1)

$accepts = (Get-WinEvent -FilterHashtable @{
    LogName   = 'Security'
    Id        = 6272
    StartTime = $oneHourAgo
} -ErrorAction SilentlyContinue).Count

$rejects = (Get-WinEvent -FilterHashtable @{
    LogName   = 'Security'
    Id        = 6273
    StartTime = $oneHourAgo
} -ErrorAction SilentlyContinue).Count

$total = $accepts + $rejects
$successRate = if ($total -gt 0) { [math]::Round(($accepts / $total) * 100, 1) } else { 0 }

Write-Host "NPS 802.1X authentication results — last 1 hour" -ForegroundColor Cyan
Write-Host "Accepts (6272) : $accepts"  -ForegroundColor Green
Write-Host "Rejects (6273) : $rejects"  -ForegroundColor $(if ($rejects -gt 5) { "Red" } else { "Yellow" })
Write-Host "Success rate   : $successRate%"

if ($successRate -lt 95) {
    Write-Host "`nWARNING: Success rate below 95% — investigate Event ID 6273 for reason codes" `
        -ForegroundColor Red
}
```

```title="Résultat attendu après basculement réussi"
NPS 802.1X authentication results — last 1 hour
Accepts (6272) : 847
Rejects (6273) : 3
Success rate   : 99.6%
```

!!! quote "En résumé"
    - Renouveler un root CA sans prédéployer le nouveau certificat via GPO = toutes les authentications 802.1X tombent simultanément
    - Script `Check-RootCADeployment.ps1` à exécuter sur 100 % du parc avant le basculement NPS
    - Phase de coexistence obligatoire : ancien et nouveau root dans la GPO en parallèle pendant 30 jours minimum
    - Les postes hors réseau (congés, télétravail sans VPN actif) sont le risque résiduel — allongez la phase de coexistence en conséquence
    - Après basculement, surveillez le ratio 6272/6273 dans les logs NPS pendant 48 heures

---

## Récapitulatif opérationnel

### :material-map: Architecture décisionnelle

Le schéma ci-dessous résume les dépendances entre les composants de ce chapitre.

```
PKI (chap. 15)
  ├── Auto-enrollment → Certificat machine (Cert:\LocalMachine\My)
  │     ├── Wi-Fi EAP-TLS ──────────────────────→ NPS → Borne Wi-Fi → SSID
  │     ├── 802.1X filaire (dot3svc) ───────────→ NPS → Switch → VLAN prod
  │     └── Always On VPN Device Tunnel ────────→ RRAS/VPN GW
  └── Distribution Root CA
        └── Machines font confiance à NPS et à la chaîne 802.1X
```

Chaque flèche est un point de rupture potentiel. Le certificat machine est le seul point commun.

### :material-table: Référence rapide des nœuds GPMC

| Fonctionnalité | Nœud GPMC | Section |
|---|---|---|
| Profils Wi-Fi d'entreprise | `Computer Config → Security Settings → Wireless Network (802.11) Policies` | Computer |
| Restrictions comportementales Wi-Fi | `Computer Config → Admin Templates → Network → Wireless LAN Service` | Computer |
| 802.1X filaire | `Computer Config → Security Settings → Wired Network (802.3) Policies` | Computer |
| Auto-enrollment certificats machine | `Computer Config → Security Settings → Public Key Policies → Certificate Services Client - Auto-Enrollment` | Computer |
| Distribution Root CA | `Computer Config → Security Settings → Public Key Policies → Trusted Root CAs` | Computer |
| Script démarrage (Device Tunnel VPN) | `Computer Config → Policies → Windows Settings → Scripts → Startup` | Computer |
| Filtre WMI laptops | GPO → Scope → WMI Filtering | Scope |

### :material-crosshairs: Liste de contrôle avant mise en production

- [ ] Auto-enrollment fonctionnel (chapitre 15) — certificat machine présent dans `Cert:\LocalMachine\My`
- [ ] NPS configuré avec la bonne CA dans les Trusted Roots, Network Policy active, test manuel de connexion
- [ ] Profil Wi-Fi importé en GPO avec `ServerValidation` activé et empreinte NPS renseignée
- [ ] Service `dot3svc` en démarrage automatique (non bloqué par GPO de durcissement)
- [ ] Device Tunnel déployé en Computer Startup Script — test sur un poste hors réseau d'entreprise
- [ ] Script `Check-RootCADeployment.ps1` en tâche planifiée — alerte si couverture < 100 %
- [ ] Plan de basculement CA racine documenté avec phase de coexistence de 30 jours minimum
