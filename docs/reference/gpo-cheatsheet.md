---
description: "Reference rapide GPO : commandes, Event IDs, GUID CSE, emplacements critiques et cmdlets PowerShell."
tags:
  - gpo
  - cheatsheet
  - reference
---

# GPO - Aide-memoire de reference rapide

## Commandes essentielles

| Commande | Usage en une ligne |
|----------|--------------------|
| `gpupdate /force` | Force un rafraichissement immediat des strategies machine et utilisateur |
| `gpresult /r` | Affiche le resultat synthese des GPO appliquees et refusees |
| `gpresult /h report.html` | Genere un rapport HTML RSoP exploitable pour le diagnostic |
| `gpmc.msc` | Ouvre la console de gestion GPO de domaine |
| `gpedit.msc` | Ouvre l'editeur de strategie locale sur un poste standalone |
| `rsop.msc` | Lance un Resultant Set of Policy graphique sur la machine locale |
| `secedit /analyze /db C:\Temp\secedit.sdb` | Compare la politique de securite configuree et l'etat reel du systeme |

## Event IDs GPO critiques

| ID | Source / Log | Signification | Action recommandee |
|----|--------------|---------------|--------------------|
| 1030 | `GroupPolicy` / `System` | Le client n'obtient pas correctement la liste ou le contenu des GPO | Verifier canal securise, LDAP, DNS et acces SYSVOL |
| 1058 | `GroupPolicy` / `System` ou `Operational` | Echec d'acces au GPT ou a `gpt.ini` | Tester `\\domaine\SYSVOL`, les ACL, SMB et la resolution DNS |
| 4001 | `Microsoft-Windows-GroupPolicy/Operational` | Debut d'un cycle de traitement GPO | Utiliser l'`ActivityID` pour correler tout le cycle |
| 4016 | `Microsoft-Windows-GroupPolicy/Operational` | Debut du traitement d'une CSE | Noter le GUID ou le nom de l'extension qui part en traitement |
| 4017 | `Microsoft-Windows-GroupPolicy/Operational` | Fin d'une CSE avec duree | Comparer les durees entre machines pour isoler les lenteurs |
| 5016 | `Microsoft-Windows-GroupPolicy/Operational` | Traitement termine avec succes | Utiliser comme point de comparaison quand tout va bien |
| 5017 | `Microsoft-Windows-GroupPolicy/Operational` | Traitement partiel ou en erreur, souvent avec `HRESULT` | Lire le message brut, puis remonter vers la CSE et la GPO concernees |
| 5313 | `Microsoft-Windows-GroupPolicy/Operational` | GPO refusee par filtrage de securite | Verifier `Read` + `Apply Group Policy` sur l'objet GPO |
| 5320 | `Microsoft-Windows-GroupPolicy/Operational` | GPO ou CSE jugee non applicable selon le contexte | Lire le detail du message : WMI, ciblage ou optimisation `NoGPOListChanges` |
| 7016 | `Microsoft-Windows-GroupPolicy/Operational` ou `System` | Echec critique d'une CSE | Identifier la DLL / le GUID et verifier dependances, acces et contenu SYSVOL |

## CSE GUIDs principaux

| GUID | Nom | DLL | Usage |
|------|-----|-----|-------|
| `{35378EAC-683F-11D2-A89A-00C04FBBCFA2}` | Registry | `gpedit.dll` | Parametres ADMX et ecritures `registry.pol` |
| `{42B5FAAE-6536-11D2-AE5A-0000F87571E3}` | Scripts | `gpscript.dll` | Scripts de demarrage, ouverture et fermeture de session |
| `{827D319E-6EAC-11D2-A4EA-00C04F79F83A}` | Security | `scecli.dll` | Politiques de securite, droits, audit, options systeme |
| `{B1BE8D72-6EAC-11D2-A4EA-00C04F79F83A}` | Folder Redirection | `fdeploy.dll` | Redirection de dossiers utilisateur |
| `{C6DC5466-785A-11D2-84D0-00C04FB169F7}` | Software Installation (Machine) | `appmgmts.dll` | MSI attribues au demarrage |
| `{3610EDA5-77EF-11D2-8DC5-00C04FA31A66}` | Software Installation (User) | `appmgmts.dll` | MSI publies ou attribues a l'utilisateur |
| `{0F6B957E-509E-11D1-A7CC-0000F87571E3}` | Wireless Policy | `gptext.dll` | Wi-Fi, 802.1X et profils sans fil |
| `{A2E30F80-D7DE-11D2-BBDE-00C04F86AE3B}` | Audit Policy (Basic) | `auditcse.dll` | Ancien modele d'audit |
| `{FC491EF1-C4AA-4CE1-B329-414B101DB823}` | Audit Policy (Advanced) | `auditcse.dll` | Sous-categories d'audit via `auditpol` |
| `{0E28E245-9368-4853-AD84-6DA3BA35BB75}` | Group Policy Preferences | `gpprefcl.dll` | Moteur global GPP |
| `{169EBF44-942F-4C43-87CE-13C93996EBBE}` | GP Preferences - Computer | `gpprefcl.dll` | Preferences machine |
| `{AADCED64-746C-4633-A97C-D61349046527}` | GP Preferences - User | `gpprefcl.dll` | Preferences utilisateur |

## Emplacements critiques GPO

| Cle / emplacement | Type | Role |
|-------------------|------|------|
| `HKLM\SYSTEM\CurrentControlSet\Services\gpsvc` | Service | Configuration du service Group Policy Client |
| `HKLM\SYSTEM\CurrentControlSet\Services\gpsvc\Parameters\ServiceDll` | Valeur | Doit pointer vers `%SystemRoot%\System32\gpsvc.dll` |
| `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\GPExtensions\{GUID}` | Registre | Enregistrement des CSE, flags et DLL |
| `HKLM\SOFTWARE\Policies` | Registre | Parametres machine imposes par GPO / MDM |
| `HKCU\SOFTWARE\Policies` | Registre | Parametres utilisateur imposes par GPO / MDM |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies` | Registre | Emplacements historiques de politiques machine |
| `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies` | Registre | Emplacements historiques de politiques utilisateur |
| `%SystemRoot%\System32\GroupPolicy\Machine\Registry.pol` | Fichier local | Strategie locale machine |
| `%SystemRoot%\System32\GroupPolicy\User\Registry.pol` | Fichier local | Strategie locale utilisateur |
| `\\domaine\SYSVOL\domaine\Policies\{GUID-GPO}\Machine\Registry.pol` | Fichier SYSVOL | Strategie de domaine machine |
| `\\domaine\SYSVOL\domaine\Policies\{GUID-GPO}\User\Registry.pol` | Fichier SYSVOL | Strategie de domaine utilisateur |
| `Attribut AD gPLink` sur Site / Domaine / OU | AD | Liste des GPO liees, Link Order et flags |
| `Attribut AD gPCWQLFilter` + `CN=SOM,CN=WMIPolicy,CN=System,<DN>` | AD | Association et definition des filtres WMI |
| `Cle cible hors \Policies` | Tattoo GPP | Valeur laissee en place apres unlink si aucune action de nettoyage n'est prevue |

## Format `gPLink`

Format brut :

```text
[LDAP://CN={GUID-GPO},CN=Policies,CN=System,DC=contoso,DC=local;FLAG]
```

Flags :

| Flag | Etat du lien | Effet |
|------|--------------|-------|
| `0` | Active | GPO appliquee normalement |
| `1` | Desactive | GPO visible mais ignoree sur ce conteneur |
| `2` | Enforced | GPO forcee, ignore `Block Inheritance` |
| `3` | Desactive + Enforced | Cas limite, a eviter en pratique |

## Ordre LSDOU

```text
L  Local policy
S  Site
D  Domaine
O  OU parente
U  OU enfant / OU la plus proche

Ordre d'application normal :
Local -> Site -> Domaine -> OU parente -> OU enfant

Regle de lecture :
- Le dernier applique gagne.
- Au meme niveau, Link Order 1 gagne.
- Enforced passe au-dessus de Block Inheritance.
```

## Cmdlets PowerShell `GroupPolicy`

| Cmdlet | Usage en une ligne |
|--------|--------------------|
| `Backup-GPO` | Sauvegarde une GPO ou tout un domaine de GPO |
| `Copy-GPO` | Copie une GPO dans le meme domaine ou vers un autre |
| `Get-GPO` | Recupere une GPO par nom, GUID ou liste tout |
| `Get-GPOReport` | Genere un rapport XML ou HTML d'une GPO |
| `Get-GPInheritance` | Affiche les liens et l'heritage sur un conteneur AD |
| `Get-GPPermission` | Lit les ACL GPO du point de vue droits applicatifs |
| `Get-GPRegistryValue` | Lit une valeur presente dans `registry.pol` |
| `Get-GPResultantSetOfPolicy` | Produit un RSoP pour un poste ou un utilisateur |
| `Get-GPStarterGPO` | Liste les Starter GPO disponibles |
| `Import-GPO` | Importe une sauvegarde dans une GPO cible |
| `Invoke-GPUpdate` | Lance un `gpupdate` distant via WMI |
| `New-GPO` | Cree une nouvelle GPO vide |
| `New-GPLink` | Cree un lien de GPO vers un domaine, site ou OU |
| `New-GPStarterGPO` | Cree un Starter GPO |
| `Remove-GPO` | Supprime une GPO |
| `Remove-GPLink` | Retire un lien sans supprimer la GPO |
| `Remove-GPPrefRegistryValue` | Supprime un item GPP registre |
| `Remove-GPRegistryValue` | Supprime une valeur de `registry.pol` |
| `Rename-GPO` | Renomme une GPO quand la version du module le permet |
| `Restore-GPO` | Restaure une GPO depuis une sauvegarde |
| `Set-GPInheritance` | Active ou coupe l'heritage sur une OU |
| `Set-GPLink` | Modifie ordre, activation ou `Enforced` d'un lien |
| `Set-GPPermission` | Defini les droits `Read`, `Apply`, `Edit`, etc. |
| `Set-GPPrefRegistryValue` | Ecrit un item GPP de type registre |
| `Set-GPRegistryValue` | Ecrit une valeur brute dans `registry.pol` |

!!! note "Compatibilite module"
    Selon la version de RSAT ou de Windows Server, `Rename-GPO` peut ne pas etre expose de la meme facon. Si la cmdlet manque, passez par GPMC ou un module plus recent.
