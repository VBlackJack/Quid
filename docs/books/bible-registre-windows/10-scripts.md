---
tags:
  - registre
  - scripts
  - automatisation
  - powershell
---

# Scripts et automatisation

## Ce que vous allez apprendre

- Rechercher dans le registre avec PowerShell (cles, noms, valeurs)
- Comparer deux instantanes du registre pour detecter les changements
- Surveiller une cle en temps reel
- Generer des fichiers `.reg` dynamiquement
- Deployer des modifications a grande echelle via GPO ou PowerShell Remoting

---

## Recherche recursive dans le registre

Chercher une valeur dans le registre avec Regedit, c'est comme chercher un mot dans un dictionnaire page par page. PowerShell permet de faire cette recherche **instantanement**.

### La fonction `Search-Registry`

```powershell
function Search-Registry {
    param(
        [string]$Path = "HKLM:\SOFTWARE",
        [string]$Pattern,
        [switch]$ValueNames,
        [switch]$ValueData
    )

    Get-ChildItem -Path $Path -Recurse -ErrorAction SilentlyContinue |
    ForEach-Object {
        $key = $_
        if ($ValueNames -or $ValueData) {
            $props = Get-ItemProperty -Path $key.PSPath -ErrorAction SilentlyContinue
            $props.PSObject.Properties | Where-Object {
                ($ValueNames -and $_.Name -match $Pattern) -or
                ($ValueData -and "$($_.Value)" -match $Pattern)
            } | ForEach-Object {
                [PSCustomObject]@{
                    Key   = $key.PSPath -replace "Microsoft\.PowerShell\.Core\\Registry::", ""
                    Name  = $_.Name
                    Value = $_.Value
                }
            }
        }
        elseif ($key.Name -match $Pattern) {
            [PSCustomObject]@{
                Key   = $key.PSPath -replace "Microsoft\.PowerShell\.Core\\Registry::", ""
                Name  = "(key)"
                Value = ""
            }
        }
    }
}
```

### Exemples d'utilisation

```powershell
# Search for values containing "MonApp" in the data
Search-Registry -Path "HKCU:\Software" -Pattern "MonApp" -ValueData
```

```title="Resultat attendu"
Key                                    Name      Value
---                                    ----      -----
HKCU\Software\MonApp                   InstallDir C:\Program Files\MonApp
HKCU\Software\MonApp                   Version    2.0.0
```

```powershell
# Search for value names matching a pattern
Search-Registry -Path "HKLM:\SOFTWARE" -Pattern "Install" -ValueNames
```

```title="Resultat attendu"
Key                                                  Name            Value
---                                                  ----            -----
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion       InstallDir      C:\WINDOWS
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion       InstallDate     1680000000
HKLM\SOFTWARE\MonApp                                 InstallPath     C:\Program Files\MonApp
```

```powershell
# Search for key names matching a pattern
Search-Registry -Path "HKCU:\Software" -Pattern "Mozilla"
```

```title="Resultat attendu"
Key                                    Name   Value
---                                    ----   -----
HKCU\Software\Mozilla                  (key)
HKCU\Software\Mozilla\Firefox          (key)
```

!!! tip "La recherche peut etre lente sur les grandes branches"
    Restreindre le parametre `-Path` a la branche la plus specifique possible accelere considerablement la recherche. Evitez de chercher dans `HKLM:\` sans filtre.

!!! info "En resume"
    - `-ValueData` : cherche dans les donnees des valeurs
    - `-ValueNames` : cherche dans les noms des valeurs
    - Sans switch : cherche dans les noms de cles
    - Toujours restreindre le chemin pour accelerer la recherche

---

## Comparaison d'instantanes

Imaginez prendre une **photo avant/apres** de votre registre pour voir exactement ce qu'une installation ou une configuration a change. C'est le principe de la comparaison d'instantanes.

### La fonction `Get-RegistrySnapshot`

```powershell
function Get-RegistrySnapshot {
    param([string]$Path)

    $snapshot = @{}
    Get-ChildItem -Path $Path -Recurse -ErrorAction SilentlyContinue |
    ForEach-Object {
        $keyPath = $_.PSPath -replace "Microsoft\.PowerShell\.Core\\Registry::", ""
        $props = Get-ItemProperty -Path $_.PSPath -ErrorAction SilentlyContinue
        $props.PSObject.Properties |
        Where-Object { $_.Name -notmatch "^PS" } |
        ForEach-Object {
            $snapshot["$keyPath\$($_.Name)"] = $_.Value
        }
    }
    return $snapshot
}
```

### Utilisation pas a pas

```powershell
# Step 1: Take a snapshot BEFORE changes
$before = Get-RegistrySnapshot "HKCU:\Software\MonApp"

# Step 2: Install or configure something here...

# Step 3: Take a snapshot AFTER changes
$after = Get-RegistrySnapshot "HKCU:\Software\MonApp"

# Step 4: Compare the two snapshots
$after.GetEnumerator() | Where-Object {
    -not $before.ContainsKey($_.Key) -or $before[$_.Key] -ne $_.Value
} | Format-Table Key, Value -AutoSize
```

```title="Resultat attendu"
Key                                        Value
---                                        -----
HKCU\Software\MonApp\Version               2.1.0
HKCU\Software\MonApp\LastUpdate            2026-04-02
HKCU\Software\MonApp\Settings\NewFeature   1
```

!!! info "Astuce : detecter aussi les suppressions"
    Le script ci-dessus ne montre que les ajouts et modifications. Pour detecter les cles **supprimees**, inversez la comparaison :

    ```powershell
    # Find deleted entries
    $before.GetEnumerator() | Where-Object {
        -not $after.ContainsKey($_.Key)
    } | Format-Table Key, Value -AutoSize
    ```

!!! info "En resume"
    - `Get-RegistrySnapshot` capture l'etat d'une branche du registre dans une table de hachage
    - Comparez deux instantanes (avant/apres) pour identifier les ajouts et modifications
    - Inversez la comparaison pour detecter les suppressions

---

## Surveillance en temps reel

Ce script surveille une cle et vous alerte des qu'elle change -- comme une alarme de porte qui sonne a chaque ouverture.

```powershell
# Watch for changes on a specific key (polling every 2 seconds)
$path = "HKLM:\SOFTWARE\MonApp"

$lastHash = (Get-ItemProperty $path | ConvertTo-Json | Get-FileHash -InputStream (
    [System.IO.MemoryStream]::new(
        [System.Text.Encoding]::UTF8.GetBytes(
            (Get-ItemProperty $path | ConvertTo-Json)
        )
    )
)).Hash

while ($true) {
    Start-Sleep -Seconds 2
    $currentHash = (Get-ItemProperty $path | ConvertTo-Json | Get-FileHash -InputStream (
        [System.IO.MemoryStream]::new(
            [System.Text.Encoding]::UTF8.GetBytes(
                (Get-ItemProperty $path | ConvertTo-Json)
            )
        )
    )).Hash
    if ($currentHash -ne $lastHash) {
        Write-Host "Registry key modified: $path" -ForegroundColor Yellow
        Get-ItemProperty $path
        $lastHash = $currentHash
    }
}
```

```title="Resultat attendu"
Registry key modified: HKLM:\SOFTWARE\MonApp
Config  : production
Version : 2.1.0
PSPath  : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SOFTWARE\MonApp
```

!!! warning "Ce script tourne indefiniment"
    Appuyez sur ++ctrl+c++ pour l'arreter. Il consomme peu de ressources grace au delai de 2 secondes entre chaque verification.

!!! info "En resume"
    - **Instantane** = photo du registre a un instant T, ideal pour analyser les installations
    - **Surveillance** = polling toutes les 2 secondes, alerte a chaque changement
    - Pensez a verifier aussi les suppressions en inversant la comparaison

---

## Fichiers .reg avances

### Scripts batch conditionnels

Un script batch peut lire le registre, prendre une decision, puis ecrire. C'est comme un GPS qui adapte l'itineraire selon le vehicule.

```batch
@echo off
setlocal enabledelayedexpansion

rem Check Windows version before applying settings
for /f "tokens=3" %%a in ('reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v CurrentBuildNumber 2^>nul ^| findstr CurrentBuildNumber') do (
    set BUILD=%%a
)

if !BUILD! GEQ 22000 (
    echo Windows 11 detected, applying specific settings...
    reg add "HKCU\Software\MonApp" /v W11Mode /t REG_DWORD /d 1 /f
) else (
    echo Windows 10 detected, applying legacy settings...
    reg add "HKCU\Software\MonApp" /v W11Mode /t REG_DWORD /d 0 /f
)
```

```title="Resultat attendu"
Windows 11 detected, applying specific settings...
L'operation a reussi.
```

### Generation dynamique de fichiers .reg

Plutot que d'ecrire des fichiers `.reg` a la main, generez-les par programme. C'est comme utiliser un publipostage au lieu de taper chaque lettre individuellement.

```powershell
function Export-RegFile {
    param(
        [string]$OutputPath,
        [hashtable]$Values
    )

    $content = "Windows Registry Editor Version 5.00`r`n`r`n"

    foreach ($key in $Values.Keys) {
        $content += "[$key]`r`n"
        foreach ($entry in $Values[$key]) {
            $name = $entry.Name
            switch ($entry.Type) {
                "REG_SZ"    { $content += "`"$name`"=`"$($entry.Data -replace '\\', '\\')`"`r`n" }
                "REG_DWORD" { $content += "`"$name`"=dword:$($entry.Data.ToString('x8'))`r`n" }
            }
        }
        $content += "`r`n"
    }

    Set-Content -Path $OutputPath -Value $content -Encoding Unicode
}
```

```powershell
# Usage: generate a .reg file programmatically
$regData = @{
    "HKEY_CURRENT_USER\Software\MonApp" = @(
        @{ Name = "Version";  Type = "REG_SZ";    Data = "2.0.0" }
        @{ Name = "Activer";  Type = "REG_DWORD"; Data = 1 }
    )
}
Export-RegFile -OutputPath "config.reg" -Values $regData
```

```title="Resultat attendu"
Aucune sortie si la commande reussit. Le fichier config.reg est cree dans le repertoire courant.
```

Contenu du fichier `config.reg` genere :

```reg
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Software\MonApp]
"Version"="2.0.0"
"Activer"=dword:00000001
```

!!! info "En resume"
    - Les scripts batch permettent d'appliquer des reglages conditionnels (version de Windows, etc.)
    - `Export-RegFile` genere des fichiers `.reg` a partir de donnees structurees
    - Utile pour creer des configurations reproductibles et versionnables

---

## Deploiement a grande echelle

### Via les strategies de groupe (GPO)

Les preferences GPO permettent de gerer le registre de maniere centralisee sur tout un parc de machines -- comme un reglement applique a toutes les succursales d'une entreprise.

**Configuration :**

1. Ouvrez **Gestion des strategies de groupe** (`gpmc.msc`)
2. Creez ou modifiez un GPO
3. Naviguez vers **Configuration ordinateur** (ou utilisateur) > **Preferences** > **Parametres Windows** > **Registre**
4. Ajoutez les elements de registre souhaites

**Avantages des preferences GPO :**

| Fonctionnalite | Detail |
|----------------|--------|
| Ciblage | Par UO, groupe de securite, ou filtre WMI |
| Actions | Creer, remplacer, mettre a jour ou supprimer |
| Application | Periodique et automatique |
| Journalisation | Centralisee dans les journaux Windows |

### Via PowerShell Remoting

Pour deployer rapidement sur quelques machines sans infrastructure GPO :

```powershell
$computers = @("PC001", "PC002", "PC003")
$scriptBlock = {
    Set-ItemProperty -Path "HKLM:\SOFTWARE\MonApp" -Name "Config" -Value "production" -Type String
}

Invoke-Command -ComputerName $computers -ScriptBlock $scriptBlock -Credential (Get-Credential)
```

```title="Resultat attendu"
PSComputerName    RunspaceId
--------------    ----------
PC001             a1b2c3d4-...
PC002             e5f6g7h8-...
PC003             i9j0k1l2-...
```

!!! tip "GPO vs PowerShell Remoting"
    | Critere | GPO | PowerShell Remoting |
    |---------|-----|---------------------|
    | Infrastructure requise | Active Directory | WinRM active sur les cibles |
    | Nombre de machines | Illimite | Adapte a quelques dizaines |
    | Application periodique | Automatique (toutes les 90 min) | Manuelle (ou via tache planifiee) |
    | Rollback | Suppression du GPO | Script inverse a deployer |

!!! info "En resume"
    - **GPO** = solution enterprise pour des centaines/milliers de machines
    - **PowerShell Remoting** = solution rapide pour quelques machines
    - Les deux approches sont preferables au service Remote Registry

---

## DSC : Registry comme ressource declarative

Desired State Configuration (DSC) permet de declarer l'etat souhaite du registre.
Au lieu d'ecrire un script imperatif qui "fait" des modifications, vous décrivez la configuration attendue.
Le moteur DSC applique ensuite l'écart si nécessaire.

```powershell title="Configuration DSC pour une cle registre"
Configuration RegistryBaseline {
    Node 'localhost' {
        Registry DisableIPv6 {
            Key       = 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters'
            ValueName = 'DisabledComponents'
            ValueData = '0xff'
            ValueType = 'Dword'
            Ensure    = 'Present'
        }
    }
}

RegistryBaseline
Start-DscConfiguration -Path .\RegistryBaseline -Wait -Force -Verbose
```

Commandes utiles :

| Commande | Usage |
|---|---|
| `Test-DscConfiguration` | Verifie si l'etat actuel correspond a l'etat declare |
| `Get-DscConfiguration` | Retourne l'etat actuel de toutes les ressources |
| `Start-DscConfiguration` | Applique la configuration compilee |

Avantages :

- idempotent : relancer la configuration ne double pas les modifications ;
- auditable : l'etat cible est declare explicitement ;
- versionnable en Git ;
- pratique pour une baseline stable.

!!! warning "DSC v2 et v3"
    DSC v2 côté Windows PowerShell 5.1 reste très présent dans les environnements existants.
    DSC v3 évolue vers un modèle refondu ; ne migrez pas un runbook sans valider les ressources disponibles et le moteur cible.

!!! info "En resume"
    DSC est utile quand vous voulez declarer une baseline registre et la tester de maniere idempotente.
    Pour un simple changement ponctuel, un script PowerShell classique reste souvent plus rapide.

---

## Gestion des erreurs dans les scripts registre

Dans un script de production, une erreur de registre ne doit pas passer inaperçue.
Utilisez `-ErrorAction Stop` pour transformer les erreurs non terminales PowerShell en exceptions capturables par `try/catch`.

```powershell title="Pattern standard try/catch pour le registre"
try {
    $value = Get-ItemPropertyValue -Path "HKLM:\SOFTWARE\MyApp" -Name "Setting" -ErrorAction Stop
}
catch [System.Management.Automation.ItemNotFoundException] {
    Write-Warning "Key does not exist: HKLM:\SOFTWARE\MyApp"
}
catch [System.Security.SecurityException] {
    Write-Warning "Access denied to HKLM:\SOFTWARE\MyApp"
}
catch {
    Write-Error "Unexpected error: $_"
}
```

| Erreur | Exception | Cause |
|--------|-----------|-------|
| Cle inexistante | `ItemNotFoundException` | Chemin incorrect |
| Acces refuse | `SecurityException` | Permissions ou TrustedInstaller |
| Type incorrect | `InvalidCastException` | Valeur `REG_DWORD` traitee comme string |

!!! tip "Production"
    Ajoutez `-ErrorAction Stop` sur les lectures et écritures critiques.
    Sans cela, un `catch` peut ne jamais s'exécuter alors que la commande affiche une erreur.

!!! info "En resume"
    Les scripts registre doivent echouer clairement.
    `try/catch` + `-ErrorAction Stop` rendent les erreurs exploitables dans les logs et les pipelines CI.

---

## Tests Pester pour le registre

Pester est le framework de test PowerShell standard.
Il permet de transformer une baseline registre en tests lisibles et répétables.

```powershell title="Test Pester : verifier une cle registre"
Describe "Registry baseline compliance" {
    It "DisabledComponents should be 0xff (IPv6 disabled)" {
        $value = Get-ItemPropertyValue "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" -Name "DisabledComponents"
        $value | Should -Be 255
    }

    It "RDP should be enabled (fDenyTSConnections = 0)" {
        $value = Get-ItemPropertyValue "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" -Name "fDenyTSConnections"
        $value | Should -Be 0
    }
}
```

Exécution :

```powershell
Invoke-Pester .\RegistryBaseline.Tests.ps1 -Output Detailed
```

Cas d'usage :

- validation de baseline après déploiement ;
- audit de conformité ;
- contrôle CI/CD d'une image de référence ;
- non-régression après modification d'un script registre.

!!! info "En resume"
    Pester transforme les attentes registre en tests automatisés.
    C'est le bon outil quand une baseline doit être prouvée, pas seulement appliquée.

---

!!! note "Premiers pas avec PowerShell"
    Si vous debutez avec PowerShell, commencez par [PowerShell et le registre : les bases](../registre-pour-les-nuls/13-powershell-bases.md) dans le guide pour debutants.
