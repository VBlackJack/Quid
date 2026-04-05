---
description: Secured-Core PC via GPO — VBS, HVCI, Secure Boot, Credential Guard, vérification de parc et roadmap de déploiement
tags: [gpo-admins, secured-core, vbs, hvci, credential-guard, secure-boot, dma, device-guard]
---
# Secured-Core PC

!!! abstract "Ce que vous allez apprendre"
    - Comprendre ce qu'un **Secured-Core PC** protège réellement : démarrage, firmware, noyau et chemins DMA
    - Activer via GPO les réglages clés autour de **VBS**, **HVCI**, **Credential Guard** et du niveau de sécurité plateforme
    - Vérifier à grande échelle l'état du parc avec **`msinfo32`** et **PowerShell**
    - Articuler Secured-Core avec les autres briques de sécurité déjà présentes : **BitLocker**, **WDAC/AppLocker** et politiques endpoint
    - Construire une **roadmap réaliste** pour un parc existant sans transformer le premier pilote en panne de masse

!!! tip "Si vous ne retenez qu'une chose"
    **Secured-Core n'est pas une case à cocher Windows.**
    C'est l'alignement de quatre couches :
    **firmware**, **boot**, **virtualisation**, **intégrité du noyau**.
    Si le matériel n'est pas prêt, la GPO ne "créera" pas la sécurité manquante.
    Elle ne fera que révéler vos incompatibilités.

---

## :material-shield-check: Qu'est-ce qu'un Secured-Core PC ?

### Une définition utile pour l'admin

Un **Secured-Core PC** est une machine conçue pour activer de façon cohérente plusieurs protections matérielles et logicielles contre les attaques qui ciblent le démarrage ou le noyau.

L'idée n'est pas seulement de "durcir Windows".

L'idée est de protéger la machine **avant** qu'un malware de haut niveau ait la main.

### Les quatre piliers à retenir

| Pilier | Rôle |
|---|---|
| **Virtualization-Based Security (VBS)** | Isole des composants sensibles dans un environnement virtualisé protégé |
| **Hypervisor-Protected Code Integrity (HVCI)** | Empêche du code noyau non approuvé de s'exécuter facilement |
| **Secure Boot** | Vérifie la chaîne de démarrage avant chargement de l'OS |
| **System Guard / DRTM** | Réinitialise la confiance à l'exécution et protège le lancement sécurisé |

### Pourquoi c'est important

Quand une attaque réussit **avant** ou **sous** l'OS, vos contrôles traditionnels deviennent moins utiles.

Exemples :

- **bootkits**
- **rootkits firmware**
- **DMA attacks**

Ces attaques cherchent le même objectif :

prendre position assez bas dans la pile pour contourner la sécurité plus haute.

### Ce que protège chaque pilier

| Menace | Pilier principalement impliqué | Effet attendu |
|---|---|---|
| Bootkit | Secure Boot | Empêche un chargeur non approuvé de démarrer |
| Rootkit noyau | HVCI | Réduit la possibilité de charger du code noyau non valide |
| Compromission précoce du démarrage | System Guard / DRTM | Réétablit une racine de confiance de mesure |
| Vol ou altération de secrets sensibles en mémoire | VBS / Credential Guard | Isole des éléments sensibles hors du contexte standard du noyau |
| DMA non contrôlé | Kernel DMA Protection + IOMMU | Réduit l'exposition des chemins mémoire directs |

### Le piège de vocabulaire

On entend souvent :

"On va activer Secured-Core."

Ce n'est pas faux.

Mais ce n'est pas assez précis.

En réalité, vous allez activer et vérifier un **ensemble** de mécanismes :

- certains via **UEFI / BIOS**
- certains via **Windows**
- certains via **GPO**

### Matériel requis

Pour viser une posture Secured-Core crédible, retenez au minimum :

- **TPM 2.0**
- **UEFI Secure Boot**
- CPU avec **IOMMU / VT-d** ou équivalent
- **Kernel DMA Protection**
- support **DRTM**

Sans cela, la GPO peut s'appliquer partiellement.

Mais votre machine ne sera pas réellement à la hauteur de la promesse Secured-Core.

### L'importance du firmware

Le firmware n'est pas un détail.

C'est l'endroit où se gagne ou se perd une bonne partie du sujet.

Si les options suivantes sont désactivées dans l'UEFI :

- virtualisation
- IOMMU
- Secure Boot
- DRTM / Secure Launch

alors le reste du projet patine.

### Secured-Core et Windows 11

Le sujet est particulièrement cohérent sur Windows 11, parce que :

- le matériel récent l'expose mieux
- Microsoft a renforcé la ligne de base sécurité
- les OEM publient davantage de profils compatibles

### Liste de postes certifiés

Pour les modèles officiellement compatibles, appuyez-vous sur les listes de matériels **Secured-Core PC** publiées par Microsoft et par les constructeurs partenaires.

Ne prenez pas pour argent comptant un "compatible VBS" dans une fiche commerciale.

Ce n'est pas la même chose qu'un **poste réellement validé Secured-Core**.

### Le bon angle de décision

La question utile n'est pas :

"Est-ce que mon poste peut démarrer avec Secure Boot ?"

La vraie question est :

"Est-ce que ce poste peut activer ensemble VBS, HVCI, protections DMA et Secure Launch sans casser mon exploitation ?"

### À ne pas confondre

| Élément | Ce que c'est | Ce que ce n'est pas |
|---|---|---|
| Secured-Core PC | Un socle matériel + firmware + OS + politique | Un simple profil logiciel |
| VBS | Une isolation par virtualisation | Un antivirus |
| HVCI | Un contrôle d'intégrité du code noyau | Un filtrage applicatif complet |
| WDAC | Une politique d'autorisation d'applications | Un substitut à HVCI |

### Ce que vous gagnez côté sécurité

Avec un parc correctement aligné :

- meilleure résistance aux charges malveillantes très basses dans la pile
- meilleur comportement face aux drivers douteux
- meilleure cohérence avec BitLocker, Credential Guard et WDAC

### Ce que vous perdez si vous foncez sans pilote

Vous risquez :

- des incompatibilités driver
- des postes qui démarrent mais perdent des fonctions
- des surprises côté périphériques DMA
- des tickets support "la machine est lente depuis le durcissement"

Secured-Core récompense la rigueur.

Il punit les déploiements "clic suivant".

!!! quote "En résumé"
    - Un Secured-Core PC combine **matériel**, **firmware**, **boot** et **protections noyau**.
    - Les quatre piliers à retenir sont **VBS**, **HVCI**, **Secure Boot** et **System Guard / DRTM**.
    - Les menaces visées sont les **bootkits**, **rootkits firmware** et **DMA attacks**.
    - Sans **TPM 2.0**, **Secure Boot**, **IOMMU** et **Kernel DMA Protection**, la posture est incomplète.
    - Une GPO peut activer des réglages, mais elle ne remplace jamais un **matériel non compatible**.

---

## :material-cog: Activer les fonctionnalités via GPO

### Le nœud GPO à retenir

Le chemin principal est :

`Computer Configuration > Administrative Templates > System > Device Guard`

Dans la pratique, plusieurs options critiques se concentrent autour de :

- `Turn On Virtualization Based Security`
- les sous-options de plateforme
- HVCI
- Credential Guard
- Secure Launch / System Guard

### Tableau des paramètres GPO

| Fonctionnalité | Chemin GPO | Paramètre | Valeur |
|---|---|---|---|
| VBS | System > Device Guard | Turn On Virtualization Based Security | Enabled |
| HVCI | System > Device Guard | Virtualization Based Protection of Code Integrity | Enabled with UEFI lock |
| Secure Boot | System > Device Guard | Select Platform Security Level | Secure Boot and DMA Protection |
| Credential Guard | System > Device Guard | Credential Guard Configuration | Enabled with UEFI lock |
| System Guard | (firmware/UEFI level) | — | Activé dans UEFI |

### Ce que ce tableau veut dire dans la vraie vie

Toutes ces lignes ne correspondent pas toujours à des policies distinctes visibles comme des cases indépendantes selon la version d'ADMX.

Sur beaucoup de versions, **`Turn On Virtualization Based Security`** expose plusieurs sous-réglages dans un même écran.

Ne vous laissez pas piéger par le rendu.

Le point important, c'est le **résultat fonctionnel**.

### `Turn On Virtualization Based Security`

C'est la policy socle.

Sans elle, le reste n'a pas d'assise.

En l'activant, vous ouvrez la voie à :

- VBS
- Secure Launch
- HVCI
- Credential Guard

si le matériel suit.

### `Select Platform Security Level`

La valeur recommandée ici est :

`Secure Boot and DMA Protection`

Pourquoi pas simplement Secure Boot ?

Parce que vous voulez aussi traiter la surface **DMA**.

Si votre matériel ne supporte pas correctement cette exigence, il vaut mieux le découvrir sur l'OU pilote.

### HVCI

HVCI est souvent perçu comme "Memory Integrity".

C'est une bonne approximation côté exploitation.

Son rôle :

durcir l'exécution de code en mode noyau.

C'est souvent la brique qui révèle des drivers vieillissants.

### Credential Guard

Credential Guard protège les secrets d'authentification en les isolant via VBS.

Dans un projet Secured-Core, il s'insère naturellement.

Mais il mérite plus de précautions que la case "Enabled" ne le laisse penser.

Pourquoi ?

Parce qu'il interagit avec :

- les workflows d'administration
- les outils de support
- certaines dépendances héritées

### System Guard / Secure Launch

Cette partie dépend fortement du firmware.

La GPO ne peut pas "inventer" Secure Launch si le matériel ou l'UEFI n'expose pas correctement la capacité.

D'où l'importance de tester :

- un même modèle de PC
- un même niveau de firmware
- un même jeu de drivers

### `!!! warning "UEFI lock"`

!!! warning "UEFI lock"
    Une fois activé avec **UEFI lock**, il peut devenir impossible de désactiver la fonctionnalité à distance.
    En pratique, il faut souvent une action locale avec accès physique à l'UEFI pour revenir en arrière.
    Conclusion simple :
    **tester d'abord en OU pilote**.

### `!!! danger "Production"`

!!! danger "Production"
    Ne jamais déployer **Credential Guard avec UEFI lock** sans avoir sauvegardé et vérifié les **clés de récupération BitLocker**.
    Le couplage Secure Boot + VBS + changement de posture firmware peut transformer un problème de pilote en incident de récupération si vos fondamentaux BitLocker ne sont pas prêts.

### Approche de déploiement recommandée

Créez des GPO distinctes ou des anneaux distincts selon votre niveau de risque.

Exemple :

- `SecuredCore-Baseline-Pilot`
- `SecuredCore-HVCI-Audit`
- `SecuredCore-UEFILock-Prod`

Ce n'est pas de la bureaucratie.

C'est ce qui vous permet de savoir **qui a reçu quoi**.

### Séparer "activation" et "verrouillage"

Très bon réflexe :

1. activer les fonctions **sans lock**
2. valider compatibilité
3. passer au **UEFI lock** plus tard

Cette séparation réduit beaucoup le risque de rollback pénible.

### Ce qu'il faut vérifier avant de lier la GPO

Avant lien large :

- BitLocker recovery keys présentes et testées
- firmware à jour sur le modèle ciblé
- inventaire des drivers sensibles
- population pilote documentée
- runbook rollback rédigé

### Le faux bon plan à éviter

Mettre VBS, HVCI et Credential Guard dans la même vieille GPO "Sécurité poste".

Vous perdez :

- la lisibilité
- la capacité d'isoler un problème
- la possibilité de faire des anneaux

### Lecture rapide de `gpresult`

Quand la GPO a bien pris, vous devez voir des entrées Device Guard sous :

- `EnableVirtualizationBasedSecurity`
- `RequirePlatformSecurityFeatures`
- `HypervisorEnforcedCodeIntegrity`
- `LsaCfgFlags`

L'absence d'une valeur attendue oriente immédiatement votre diagnostic.

!!! quote "En résumé"
    - Le nœud clé est **System > Device Guard**.
    - **VBS** est la base ; **HVCI**, **Credential Guard** et **Secure Launch** s'appuient dessus.
    - La valeur de plateforme recommandée est **Secure Boot and DMA Protection**.
    - **UEFI lock** doit être traité comme un verrouillage quasi irréversible à distance.
    - Le design sain sépare **activation**, **validation** et **verrouillage**.

---

## :material-magnify: Vérifier l'état Secured-Core sur le parc

### Vérification locale la plus simple

L'outil le plus banal reste souvent le meilleur point de départ :

`msinfo32.exe`

Les champs à regarder :

- `Virtualization-based security`
- `Device Encryption Support`
- `Secure Boot State`
- `Kernel DMA Protection`

### Ce que vous cherchez dans `msinfo32`

Lecture simplifiée :

| Champ | Valeur attendue |
|---|---|
| Secure Boot State | On |
| Kernel DMA Protection | On |
| Virtualization-based security | Running |
| Services running | Hypervisor enforced Code Integrity, Credential Guard ou Secure Launch selon posture |

### Vérification de masse PowerShell

Le script suivant donne une vue simple de l'état d'un parc Windows 11.

```powershell title="Contrôler VBS, HVCI et Credential Guard à distance"
$computers = Get-ADComputer -Filter { OperatingSystem -like "*Windows 11*" } |
    Select-Object -ExpandProperty Name

foreach ($pc in $computers) {
    $vbs = Get-CimInstance -ComputerName $pc -ClassName Win32_DeviceGuard `
        -Namespace root\Microsoft\Windows\DeviceGuard -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        Computer              = $pc
        VBSStatus             = $vbs.VirtualizationBasedSecurityStatus
        HVCIStatus            = $vbs.CodeIntegrityPolicyEnforcementStatus
        CredentialGuardStatus = $vbs.SecurityServicesRunning -contains 1
    }
}
```

### Comment lire ce script

Il ne vous donne pas un joli "Secured-Core = Yes/No".

Il vous donne mieux :

les briques.

Et c'est ce que vous devez piloter.

### Interprétation simple

| Valeur remontée | Lecture terrain |
|---|---|
| `VBSStatus` absent | Poste inaccessible, WMI/CIM bloqué ou namespace indisponible |
| `VBSStatus` actif | VBS tourne réellement |
| `HVCIStatus` actif | Memory Integrity / HVCI est en place |
| `CredentialGuardStatus = True` | Credential Guard est lancé |

### Ajoutez le contexte matériel

Une remontée logicielle positive ne remplace pas le contrôle matériel.

Sur les modèles pilotes, vérifiez aussi :

- options UEFI
- version de BIOS
- version de drivers critiques

Sinon vous risquez de confondre :

- un poste compatible
- un poste juste "pas encore cassé"

### Contrôle complémentaire

Vous pouvez aussi compléter avec :

```powershell title="Contrôler le TPM local"
Get-Tpm
```

et côté interface :

- **Windows Security**
- **Device security**
- **Core isolation**

### Indicateurs à exporter pour un pilote

Sur un premier anneau, exportez au minimum :

- nom du poste
- modèle matériel
- version BIOS
- état VBS
- état HVCI
- état Credential Guard
- présence BitLocker

### Ce que vous voulez voir avant élargissement

Avant de sortir de l'OU pilote :

- zero ou très peu d'incompatibilités driver
- VBS actif sur la grande majorité du lot
- HVCI cohérent avec votre politique
- aucune surprise BitLocker

### Ce que vous ne devez pas faire

Ne jugez pas l'état du parc uniquement via :

- l'absence de tickets
- une stratégie "Enabled" dans GPMC

Une GPO **appliquée** n'est pas une fonctionnalité **effective**.

### Checklist de validation

1. `gpresult` confirme la GPO
2. `msinfo32` confirme l'état local
3. PowerShell confirme l'état à distance
4. le support valide qu'aucun pilote métier n'est cassé

### Si les résultats sont partiels

N'élargissez pas pour "voir en grand".

Faites l'inverse :

- segmentez par modèle
- segmentez par version de firmware
- segmentez par population

C'est souvent la seule façon d'isoler l'origine réelle d'un écart.

!!! quote "En résumé"
    - `msinfo32.exe` reste le point de contrôle local le plus rapide.
    - Le script `Win32_DeviceGuard` donne une lecture exploitable de **VBS**, **HVCI** et **Credential Guard** sur le parc.
    - Une GPO appliquée n'est pas une preuve suffisante ; il faut vérifier la **fonction active**.
    - Les écarts doivent être regroupés par **modèle**, **BIOS** et **driver**, pas uniquement par OU.
    - Un pilote n'est validé que si la sécurité **et** l'exploitation restent stables.

---

## :material-compare: Interaction avec les GPO de sécurité existantes

### Secured-Core ne remplace pas tout

C'est une erreur d'architecture fréquente.

On active Secured-Core.

Puis on suppose que le reste devient redondant.

Faux.

Secured-Core protège surtout :

- le boot
- le firmware
- le noyau

Il ne remplace pas les contrôles de politique métier ou applicative.

### Secured-Core + AppLocker / WDAC

La bonne lecture est :

- **WDAC / AppLocker** contrôlent **ce qui peut s'exécuter**
- **HVCI** protège **comment le code noyau est accepté**

Les deux sont complémentaires.

WDAC n'annule pas le besoin d'HVCI.

HVCI n'annule pas le besoin de WDAC.

### Pourquoi WDAC est souvent plus cohérent qu'AppLocker ici

AppLocker reste utile.

Mais dans une posture moderne Windows 11, **WDAC** s'aligne mieux avec l'objectif de contrôle fort et avec la logique de protection de bas niveau.

Si vous avez déjà WDAC en projet, Secured-Core renforce la cohérence d'ensemble.

### Secured-Core + BitLocker

Le mariage est naturel.

Secure Boot et TPM 2.0 soutiennent déjà beaucoup de ce que BitLocker attend.

Autrement dit :

quand BitLocker est bien géré, vous avez déjà une partie du terrain préparé.

Mais attention :

un changement firmware ou un verrouillage UEFI mal préparé peut exiger une **clé de récupération**.

D'où la règle :

BitLocker recovery d'abord.

UEFI lock ensuite.

### Secured-Core + Credential Guard

Credential Guard est déjà une brique de sécurité à part entière.

Dans ce chapitre, ce qui compte surtout, c'est sa dépendance au socle :

- VBS
- UEFI
- compatibilité plateforme

Et surtout :

la question du **UEFI lock**.

### Ne dupliquez pas les politiques contradictoires

Exemple typique :

une GPO de sécurité historique désactive ou contourne certains éléments requis par Device Guard.

Puis votre nouvelle GPO Secured-Core tente de les réactiver.

Résultat :

- comportement incohérent
- difficulté de lecture dans `gpresult`
- support confus

### Audit à faire avant déploiement

Relisez vos GPO existantes sur :

- Device Guard
- LSA
- Credential Guard
- options de virtualisation
- règles de durcissement driver

Le but n'est pas de "tout jeter".

Le but est de retirer les contradictions.

### `!!! check "Point de contrôle"`

!!! check "Point de contrôle"
    - Vos politiques **WDAC/AppLocker** sont-elles pensées comme **complémentaires** et non concurrentes de HVCI ?
    - Les **clés de récupération BitLocker** sont-elles sauvegardées et testées avant tout passage en **UEFI lock** ?
    - Avez-vous audité les anciennes GPO **Device Guard / Credential Guard** pour éviter les doublons et contradictions ?

### Ligne de gouvernance recommandée

Conservez une frontière simple :

- GPO Secured-Core pour le socle hardware/boot/kernel
- WDAC/AppLocker pour le contrôle d'exécution
- BitLocker pour la protection des données au repos

Quand chaque couche a un propriétaire clair, le diagnostic devient beaucoup plus simple.

!!! quote "En résumé"
    - Secured-Core protège le **socle bas niveau** ; il ne remplace pas WDAC, AppLocker ou BitLocker.
    - **WDAC** et **HVCI** sont complémentaires.
    - **BitLocker** s'aligne très bien avec Secured-Core, mais impose une discipline forte sur les **clés de récupération**.
    - Le plus grand risque est la **contradiction entre anciennes GPO de sécurité et nouvelles GPO Device Guard**.
    - Une frontière de responsabilité claire entre couches réduit fortement le coût de support.

---

## :material-road: Roadmap Secured-Core pour un parc existant

### Visez une trajectoire, pas un drapeau

Un parc existant n'arrive presque jamais à "Secured-Core complet" en un seul mouvement.

Il faut une progression.

Sinon vous mélangez :

- compatibilité matérielle
- compatibilité drivers
- verrouillage UEFI
- récupération BitLocker

et vous perdez la main.

### Tableau de maturité

| Étape | Action | Prérequis |
|---|---|---|
| 1 | Activer Secure Boot + TPM 2.0 | UEFI + matériel compatible |
| 2 | Activer VBS via GPO (sans lock) | Étape 1 |
| 3 | Activer HVCI en mode audit | Étape 2 |
| 4 | Activer Credential Guard sans lock | Étape 2 |
| 5 | Passer HVCI + CG en UEFI lock | Test pilote validé |
| 6 | Labellisation Secured-Core PC | Toutes étapes validées |

### Étape 1 : Secure Boot + TPM 2.0

Sans cette base, le reste n'a pas de sens.

Commencez par un inventaire matériel.

Classez les postes :

- compatibles sans action
- compatibles après réglage UEFI
- incompatibles

### Étape 2 : VBS sans lock

C'est votre premier vrai test logiciel.

Le but n'est pas encore de verrouiller.

Le but est de mesurer :

- le comportement
- la performance
- les incompatibilités

### Étape 3 : HVCI en mode audit

Cette étape est excellente pour faire remonter les problèmes de drivers sans casser trop tôt.

Traitez les résultats comme un backlog d'assainissement :

- pilotes à mettre à jour
- logiciels à retirer
- modèles à exclure temporairement

### Étape 4 : Credential Guard sans lock

Même logique.

Vous validez l'usage réel avant de rendre le rollback coûteux.

Cette étape permet aussi au support de s'habituer au nouveau comportement.

### Étape 5 : UEFI lock

Ne passez ici que si :

- le pilote est propre
- les recovery keys BitLocker sont validées
- le modèle matériel est homogène
- le runbook de support est prêt

UEFI lock n'est pas une médaille.

C'est un engagement d'exploitation.

### Étape 6 : labelliser le socle

À ce stade, vous pouvez documenter que la flotte ou le modèle ciblé répond à votre définition interne de "Secured-Core prêt".

Ne transformez pas ce label en slogan vague.

Définissez des critères mesurables.

Exemple :

- Secure Boot = On
- TPM 2.0 = présent
- VBS = Running
- HVCI = actif
- Credential Guard = actif
- aucune incompatibilité driver ouverte

### Modèle d'anneaux recommandé

| Anneau | Population |
|---|---|
| 0 | IT sécurité et ingénierie |
| 1 | Support de proximité |
| 2 | Utilisateurs bureautiques standard |
| 3 | Populations VIP ou sensibles |

### Critères de sortie d'anneau

Ne passez pas à l'anneau suivant si vous avez encore :

- problèmes BitLocker non compris
- pilotes critiques non validés
- support non formé
- écarts massifs par modèle

### Ce qu'il faut documenter à chaque étape

Minimum :

- modèles testés
- versions BIOS
- GPO appliquées
- incidents observés
- décisions d'exclusion temporaire

### Le vrai objectif

Le vrai objectif n'est pas d'écrire "Secured-Core" dans un slide.

Le vrai objectif est de rendre le parc plus résistant **sans rendre l'exploitation aveugle**.

### Erreur à éviter absolument

Passer de l'étape 1 à l'étape 5 dans le même changement.

Vous perdez alors :

- la capacité de savoir ce qui casse
- la capacité de revenir proprement
- la confiance du support

!!! quote "En résumé"
    - Un parc existant doit suivre une **roadmap de maturité**, pas un déploiement monolithique.
    - **Secure Boot + TPM**, puis **VBS**, puis **HVCI/Credential Guard**, puis **UEFI lock** : cet ordre réduit le risque.
    - Le passage en **UEFI lock** est une décision d'exploitation, pas une simple amélioration de score sécurité.
    - Les anneaux et critères de sortie évitent de transformer le pilote en loterie.
    - Le label "Secured-Core" n'a de valeur que si vos critères sont **mesurables et vérifiés**.
