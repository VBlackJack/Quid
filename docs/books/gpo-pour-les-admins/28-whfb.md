---
description: Windows Hello for Business via GPO — modèles de confiance, Cloud Kerberos Trust, TPM, PIN, déploiement pilote et diagnostic
tags: [gpo-admins, whfb, windows-hello-for-business, passwordless, tpm, kerberos, entra, hybrid]
---
# Windows Hello for Business

!!! abstract "Ce que vous allez apprendre"
    - Comprendre pourquoi **Windows Hello for Business (WHfB)** réduit des risques que vos GPO de mots de passe ne savent pas éliminer : phishing, pass-the-hash et réutilisation de secrets
    - Distinguer clairement les trois modèles de déploiement : **Key Trust**, **Certificate Trust** et **Cloud Kerberos Trust**
    - Configurer les **paramètres GPO essentiels** sans mélanger politique PIN, politique mot de passe et règles MDM Intune
    - Déployer **Cloud Kerberos Trust** sur un pilote Windows Server 2022 / Windows 11 avec une check-list terrain exploitable
    - Diagnostiquer rapidement un poste qui ne s'enrôle pas grâce aux **event IDs**, à `Get-Tpm`, à `dsregcmd` et à l'attribut `msDS-KeyCredentialLink`

!!! tip "Si vous ne retenez qu'une chose"
    **WHfB n'envoie pas un secret réutilisable sur le réseau.**
    Le poste conserve une **clé privée liée au TPM**.
    L'utilisateur déverrouille cette clé avec un **geste local** : PIN, empreinte ou visage.
    Le mot de passe ne disparaît pas toujours de l'annuaire, mais il sort du **chemin d'authentification quotidien**.

---

## :material-shield-key: WHfB : pourquoi remplacer les mots de passe

### Le vrai problème n'est pas "le mot de passe faible"

Le problème n'est pas uniquement la longueur du mot de passe.

Le problème, c'est qu'un mot de passe est un **secret partagé**.

Dès qu'il est connu par l'utilisateur, il peut être :

- saisi sur un faux portail
- intercepté sur un poste compromis
- rejoué sur plusieurs services
- transformé en hash réutilisable dans une attaque latérale

Autrement dit :

vous ne défendez pas seulement une chaîne de caractères.

Vous défendez une **matière première d'attaque**.

### Le modèle de menace à garder en tête

Trois familles d'attaque expliquent à elles seules pourquoi WHfB existe.

| Menace | Ce que l'attaquant cherche | Pourquoi le mot de passe aide l'attaquant | Ce que WHfB change |
|---|---|---|---|
| **Phishing** | Récupérer le secret de connexion | L'utilisateur peut taper son mot de passe sur un faux site | Le geste local ne se transmet pas au service distant |
| **Pass-the-Hash** | Réutiliser un hash NTLM capturé | Le hash devient une preuve d'identité exploitable | WHfB retire l'usage quotidien du mot de passe du flux principal |
| **Credential stuffing** | Tester un secret recyclé sur plusieurs services | Les utilisateurs réemploient les mêmes secrets | Une clé liée au poste ne se "rejoue" pas comme un mot de passe |

### WHfB, en une phrase opérationnelle

WHfB repose sur deux éléments.

Le premier :

une **clé privée** créée sur l'appareil et idéalement protégée par le **TPM**.

Le second :

un **geste local** qui autorise l'usage de cette clé.

Ce geste peut être :

- un **PIN**
- une **empreinte**
- une **reconnaissance faciale**

Le point critique :

le geste n'est pas le secret de réseau.

Le PIN ne "voyage" pas.

L'empreinte ne "voyage" pas.

Le visage ne "voyage" pas.

L'authentification réseau s'appuie sur la **clé** et sur le modèle de confiance choisi.

### Ce que le TPM apporte vraiment

Le TPM ne rend pas une architecture magique.

Il rend l'extraction du matériel cryptographique beaucoup plus difficile.

En pratique :

- la clé privée est liée au matériel
- les protections anti-bruteforce sont locales
- la copie simple du secret sur un autre poste devient non triviale

Sur un parc moderne Windows 11, c'est la ligne de base attendue.

Si vous autorisez WHfB sans TPM, vous perdez précisément une partie de la promesse de sécurité qui justifie le projet.

### Le PIN n'est pas un mini mot de passe

C'est une confusion classique.

Le **mot de passe AD** et le **PIN WHfB** ne protègent pas la même chose.

Le mot de passe sert à authentifier un utilisateur contre une identité.

Le PIN sert à **déverrouiller localement** l'usage d'une clé déjà liée à l'appareil.

Conséquence pratique :

- un mot de passe faible met potentiellement en danger plusieurs services
- un PIN compromis ne suffit pas sans l'appareil
- un PIN de 6 chiffres peut être acceptable là où un mot de passe de 6 caractères ne l'est pas

Ce n'est pas parce que le PIN est "court".

C'est parce que son **périmètre d'abus** n'est pas le même.

### Password, NTLM, Kerberos, WHfB : ne mélangez pas tout

Ces termes se croisent souvent dans les discussions de migration.

Ils ne sont pourtant pas équivalents.

| Élément | Nature | Secret réutilisable sur le réseau | Résistance au phishing | Résistance au pass-the-hash | Rôle principal |
|---|---|---|---|---|---|
| **Password** | Secret partagé | Oui | Faible | Faible | Authentification utilisateur de base |
| **NTLM** | Protocole basé sur challenge/réponse dérivé du secret | Indirectement oui, via hash | Faible | Faible à moyenne | Compatibilité legacy |
| **Kerberos** | Protocole à tickets | Mieux que NTLM, mais dépend du secret initial | Meilleure | Meilleure | SSO AD moderne |
| **WHfB** | Credential asymétrique lié à l'appareil | Non, pas de secret utilisateur transmis comme tel | Forte | Forte | Connexion passwordless ou password-reduced |

### Ce que vous gagnez côté exploitation

Avec WHfB bien déployé :

- moins de tickets liés aux oublis de mot de passe
- moins de dépendance aux portails de reset d'urgence
- moins de surface pour le phishing classique
- un meilleur alignement avec Conditional Access et les scénarios passwordless

### Ce que vous ne gagnez pas automatiquement

WHfB n'efface pas d'un coup :

- les applications legacy en NTLM
- les comptes privilégiés mal gérés
- les postes sans TPM
- les workflows RDP/VDI qui attendent encore un certificat ou un autre mécanisme

WHfB n'est pas une baguette magique.

C'est un **changement de primitive d'authentification**.

### Décision simple pour un admin

Si votre priorité est :

- réduire le phishing utilisateur
- diminuer la dépendance au mot de passe
- rester compatible avec un AD hybride moderne

alors WHfB mérite d'être traité comme un **chantier de sécurité**, pas comme une simple option de confort Windows 11.

!!! quote "En résumé"
    - Le mot de passe est un **secret partagé** ; c'est la racine du problème.
    - WHfB remplace l'usage quotidien du mot de passe par une **clé privée liée au poste**.
    - Le geste utilisateur reste **local** : PIN, empreinte ou visage.
    - Le TPM 2.0 n'est pas un bonus marketing ; c'est la fondation du modèle.
    - Le bon angle de décision n'est pas "est-ce plus pratique ?", mais **"est-ce que je retire un secret rejouable de mon flux d'authentification ?"**

---

## :material-sitemap: Les trois modèles de déploiement

### Première règle : choisissez le modèle avant la GPO

Beaucoup d'échecs WHfB viennent d'un ordre inversé.

L'équipe active la GPO.

Puis elle se demande comment le poste doit authentifier l'utilisateur on-prem.

C'est trop tard.

Le bon ordre est :

1. choisir le **modèle de confiance**
2. valider les **prérequis d'infrastructure**
3. déployer la **GPO ou la policy MDM**

### Tableau comparatif

| Modèle | Infrastructure requise | Token émis | Cas d'usage |
|---|---|---|---|
| Key Trust | DC WS2016+, TPM 2.0 | Kerberos | AD on-prem pur |
| Certificate Trust | PKI + ADFS ou Azure AD | Certificat | Mixte, smart card compat |
| Cloud Kerberos Trust | Azure AD Connect + DC WS2016+ | Kerberos hybride | Hybrid Join (recommandé) |

### 1. Key Trust

Key Trust est le modèle historique "simple" côté PKI.

Il n'exige pas de certificat utilisateur WHfB.

En revanche, il s'appuie sur :

- la synchronisation de clé publique
- l'attribut **`msDS-KeyCredentialLink`**
- des DC compatibles

Ce modèle reste pertinent dans deux cas :

- environnement **AD on-prem pur**
- besoin de rester hors dépendance cloud pour le plan principal

Mais sur un parc moderne hybride, ce n'est plus le premier choix.

### 2. Certificate Trust

Certificate Trust existe pour les environnements qui doivent rester compatibles avec :

- des applications qui veulent un **certificat**
- des scénarios de type **smart card**
- des contraintes historiques autour de la PKI et d'AD FS

Il fonctionne.

Mais il coûte plus cher à opérer.

Pourquoi ?

Parce que vous ajoutez :

- une chaîne PKI
- une logique d'inscription de certificats
- des points de contrôle supplémentaires

Si vous n'avez pas un besoin explicite de certificat, ce modèle complexifie souvent inutilement le projet.

### 3. Cloud Kerberos Trust

Cloud Kerberos Trust est le modèle à regarder en premier pour un environnement hybride moderne.

Microsoft le présente aujourd'hui comme le **modèle recommandé** face à Key Trust quand vous n'avez pas d'exigence de certificat.

Ce qu'il simplifie :

- pas de PKI à déployer pour WHfB
- pas de synchronisation de clés publiques WHfB vers AD pour l'accès on-prem
- meilleure cohérence avec Microsoft Entra ID et les déploiements Windows 11

### Recommandation pratique pour WS2022 + Win11

Si votre socle cible est :

- **Windows Server 2022**
- **Windows 11**
- **Hybrid Join**
- **Microsoft Entra Connect Sync**

alors la recommandation par défaut est :

**Cloud Kerberos Trust**.

Ne gardez Key Trust qu'avec une raison claire.

Ne partez sur Certificate Trust qu'avec une exigence métier documentée.

### Les prérequis DC à ne pas rater

Deux idées doivent rester séparées dans votre tête.

**`msDS-KeyCredentialLink`**

Cet attribut sert à stocker les informations liées au credential pour les scénarios Key Trust et pour les contrôles d'inventaire d'objets déjà enrôlés.

Il fait partie des éléments que vous devez connaître même si vous ciblez Cloud Kerberos Trust, ne serait-ce que pour comprendre l'existant.

**Correctifs DC**

La documentation Microsoft actuelle publie des prérequis minimums pour Cloud Kerberos Trust :

- Windows Server 2016 avec le KB publié pour ce scénario
- Windows Server 2019 avec le KB publié pour ce scénario
- Windows Server 2022 ou plus récent

Sur un parc 2026, la bonne lecture opérationnelle est simple :

si vous avez encore des DC 2016 ou 2019, vérifiez explicitement qu'ils portent les correctifs requis par la documentation WHfB en vigueur.

### Pourquoi le brief "KB4534307" mérite vérification

Vous verrez parfois circuler des numéros de KB légèrement différents selon les billets, les traductions ou les notes de migration.

Ne recopiez pas un KB de mémoire dans votre runbook.

Vérifiez la page Microsoft Learn active au moment du déploiement.

C'est particulièrement vrai si vous opérez un domaine mixte 2016/2019/2022.

### Cloud Kerberos Trust n'est pas "pour tout le monde"

Deux garde-fous importants :

- il ne répond pas au besoin de **certificat on-prem** si vous avez des applications qui l'exigent
- il n'est pas le bon modèle pour un **pur AD on-prem** sans dépendance Entra

Autrement dit :

recommandé ne veut pas dire universel.

### Mini aide-mémoire

| Si votre contexte ressemble à... | Modèle à évaluer d'abord |
|---|---|
| AD on-prem sans stratégie cloud | **Key Trust** |
| Hybrid Join moderne Windows 11 | **Cloud Kerberos Trust** |
| Applications ou middleware exigeant un certificat | **Certificate Trust** |

### Erreur de design fréquente

Vouloir faire cohabiter les trois modèles "au cas où".

Mauvaise idée.

Vous devez choisir un **modèle principal**.

Ensuite :

- documenter les exceptions
- piloter les OU pilotes
- éviter les policies contradictoires

!!! quote "En résumé"
    - Il existe trois modèles : **Key Trust**, **Certificate Trust** et **Cloud Kerberos Trust**.
    - Pour un parc **WS2022 + Windows 11 + Hybrid Join**, **Cloud Kerberos Trust** est le choix par défaut recommandé.
    - `msDS-KeyCredentialLink` reste un attribut important à connaître pour l'inventaire et les scénarios Key Trust.
    - Les prérequis DC ne se devinent pas : vérifiez les **versions et KB Microsoft Learn** encore valides au moment du déploiement.
    - Le pire design est le mélange implicite de modèles sans règle de priorité.

---

## :material-cog: Configuration GPO pour WHfB

### Le chemin GPMC à retenir

Le nœud principal est :

`Computer Configuration > Administrative Templates > Windows Components > Windows Hello for Business`

Une partie des réglages PIN n'est **pas** dans ce nœud.

Ils sont sous :

`Computer Configuration > Administrative Templates > System > PIN Complexity`

Si vous ne séparez pas ces deux zones, vous allez croire qu'il "manque" des paramètres.

### Table des paramètres clés

| Paramètre | Valeur recommandée | Remarque |
|---|---|---|
| Use Windows Hello for Business | Enabled | Déploiement obligatoire |
| Use cloud Kerberos trust for on-premises authentication | Enabled | Obligatoire pour Cloud Kerberos Trust |
| Use a hardware security device | Enabled | TPM 2.0 requis en pratique |
| Use certificate for on-premises authentication | Depends | **Certificate Trust uniquement** |
| Enable enhanced anti-spoofing | Enabled | Biométrie Win11 |
| Use PIN Recovery | Enabled | Self-service reset |
| Minimum PIN length | 6 | Recommandation CIS / valeur par défaut moderne minimale |

### Lecture correcte de ce tableau

Ne lisez pas ce tableau comme un copier-coller aveugle.

Lisez-le comme une **matrice de posture**.

Exemple :

si vous ciblez Cloud Kerberos Trust, alors :

- `Use cloud Kerberos trust for on-premises authentication` = **Enabled**
- `Use certificate for on-premises authentication` = **Not Configured** ou **Disabled**

Si vous activez les deux à la fois, le comportement ne sera pas celui que vous attendez.

La documentation Microsoft actuelle indique que le mode **Certificate Trust** prend la priorité sur Cloud Kerberos Trust quand la policy certificat est activée.

### `Use Windows Hello for Business`

Cette setting fait plus que "permettre l'option".

En **Enabled**, vous forcez l'enrôlement dans le périmètre visé.

En **Not Configured**, vous laissez plus de latitude au poste et à l'utilisateur.

Pour un déploiement d'entreprise, l'ambiguïté n'aide pas.

Si vous voulez industrialiser, activez explicitement.

### `Use a hardware security device`

Sur le terrain, c'est la policy qui sépare un vrai déploiement de sécurité d'un déploiement "ça marche sur mon portable".

Activez-la.

Puis vérifiez les TPM.

Ne poussez pas cette GPO à l'échelle si votre parc mélange :

- vieux portables sans TPM utilisable
- TPM désactivés en firmware
- machines virtuelles non alignées avec votre politique

### `Enable enhanced anti-spoofing`

Cette policy concerne surtout la biométrie faciale.

Elle durcit le contrôle des capteurs compatibles.

Sur Windows 11, c'est cohérent avec une posture moderne.

Mais gardez en tête l'effet réel :

certains matériels biométriques plus anciens peuvent cesser d'être utilisables.

### `Use PIN Recovery`

Sans PIN Recovery, l'expérience oubli de PIN devient plus brutale.

L'utilisateur peut devoir recréer son conteneur selon le contexte et repasser certaines étapes.

Avec PIN Recovery activé, vous améliorez :

- l'autonomie utilisateur
- la continuité opérationnelle
- la réduction des tickets support basiques

### `Minimum PIN length`

Le minimum recommandé ici est **6**.

Ce n'est pas une concession faible.

C'est une valeur cohérente avec le fait que le PIN ne s'utilise qu'en **local** sur l'appareil lié.

Si vous montez plus haut, faites-le pour une raison documentée :

- exigences réglementaires
- population privilégiée
- borne partagée avec contrainte spécifique

### Politique PIN et politique mot de passe : deux mondes

Ne reliez pas mentalement le PIN aux règles du mot de passe AD.

La politique de mot de passe agit sur :

- l'authentification de compte
- la rotation du secret partagé
- la surface d'exposition du compte dans d'autres services

La politique PIN agit sur :

- la **déverrouillage local** du credential WHfB
- la complexité locale de ce geste
- le comportement de l'expérience utilisateur

Vous pouvez donc avoir :

- mot de passe AD long et complexe
- PIN de 6 caractères

sans incohérence de sécurité.

### Où configurer la complexité PIN

Le piège le plus fréquent :

les admins cherchent tous les réglages WHfB dans le nœud Windows Hello for Business.

Or la **complexité du PIN** se gère surtout sous :

`Computer Configuration > Administrative Templates > System > PIN Complexity`

Paramètres typiques à revoir :

- Minimum PIN length
- Maximum PIN length
- Require digits
- Require uppercase letters
- Require lowercase letters
- Require special characters
- History
- Expiration

### Recommandation d'exploitation

Pour la majorité des parcs bureautiques :

- minimum 6
- chiffres autorisés
- pas d'obligation de caractères spéciaux
- pas d'expiration artificielle du PIN sauf exigence réglementaire forte

Pourquoi ?

Parce qu'un PIN trop "administratif" casse l'adoption sans bénéfice proportionnel.

### `!!! warning "Conflit MDM"`

!!! warning "Conflit MDM"
    Si le poste est aussi géré par Intune, **n'appliquez pas les mêmes réglages WHfB dans les deux canaux**.
    Sur les politiques WHfB, la documentation Microsoft actuelle indique que la **GPO peut prendre le dessus** sur les réglages Intune correspondants.
    Sur d'autres domaines ADMX-backed en environnement hybride, vous rencontrerez souvent une logique de **dernier écrit**.
    La règle d'exploitation reste la même :
    **une frontière de responsabilité claire, zéro duplication**.

### Filtrage et ciblage

Le bon déploiement GPO WHfB ne vise pas "tout le domaine" le premier jour.

Utilisez :

- une OU pilote
- ou un filtrage par **security group**

L'objectif n'est pas uniquement le confort.

L'objectif est de contrôler :

- le volume de TPM non conformes
- les incidents biométriques
- les applications métiers qui tolèrent mal le changement de méthode d'ouverture de session

### Petite checklist avant lien GPO

Avant de lier la GPO :

- Central Store à jour avec les bons `Passport.admx` / `Passport.adml`
- parc pilote identifié
- modèle de confiance documenté
- conflit Intune exclu
- support prêt à répondre au premier enrôlement

!!! quote "En résumé"
    - Le nœud principal GPMC est **Windows Hello for Business**, mais la complexité PIN vit surtout sous **System > PIN Complexity**.
    - Pour **Cloud Kerberos Trust**, activez la policy dédiée et laissez la policy certificat hors jeu.
    - `Use a hardware security device` doit être activée si vous voulez une vraie posture matérielle.
    - Un **PIN** n'est pas un mot de passe bis ; sa politique se raisonne différemment.
    - En environnement hybride, le plus gros risque n'est pas le manque de setting, c'est le **double pilotage GPO + Intune**.

---

## :material-list-box-outline: Déploiement étape par étape (Cloud Kerberos Trust)

### Logique générale

Le déploiement Cloud Kerberos Trust tient en une idée simple :

vous préparez d'abord le **côté identité**.

Ensuite seulement vous ouvrez l'enrôlement côté poste.

Si vous inversez l'ordre, les utilisateurs voient l'invite WHfB sans que l'infrastructure ne soit prête.

Résultat :

- enrôlements partiels
- messages peu parlants
- faux diagnostics "TPM" alors que le problème est côté trust

### `!!! example "Pas à pas"`

!!! example "Pas à pas"
    1. Vérifier **Azure AD Connect version ≥ 2.1.1.0** ou, mieux, une version supportée actuelle de **Microsoft Entra Connect Sync**.
    2. Activer l'objet Kerberos côté domaine avec le module Microsoft Entra Kerberos et la cmdlet `Set-AzureADKerberosServer`.
    3. Créer et lier la **GPO WHfB** sur l'OU cible, avec Cloud Kerberos Trust activé.
    4. Vérifier le **TPM** sur les postes pilotes avec `Get-Tpm`.
    5. Forcer `gpupdate /force`, puis **redémarrer**.
    6. Vérifier l'enrôlement dans le journal **Microsoft-Windows-HelloForBusiness** avec l'**Event ID 358**.

### Étape 1 : contrôler Microsoft Entra Connect Sync

Le but n'est pas de faire un audit exhaustif du connecteur.

Le but est de répondre à trois questions :

- la synchro identités fonctionne-t-elle ?
- la version est-elle dans une branche supportée ?
- les objets utilisateurs et appareils sont-ils correctement présents côté Entra ?

Le libellé historique "Azure AD Connect" existe encore dans beaucoup de runbooks.

Le produit s'appelle désormais **Microsoft Entra Connect Sync**.

Gardez cette correspondance en tête quand vous lisez une vieille procédure.

### Étape 2 : créer l'objet Microsoft Entra Kerberos

Cloud Kerberos Trust s'appuie sur **Microsoft Entra Kerberos**.

Concrètement, vous devez créer l'objet serveur Kerberos correspondant dans le domaine.

Exemple de séquence :

```powershell title="Activer Microsoft Entra Kerberos pour le domaine"
# Exemple de principe : adaptez les modules et le compte d'administration à votre contexte
Import-Module AzureADHybridAuthenticationManagement

$domain = "corp.contoso.com"
$upn    = "admin-whfb@contoso.com"

Set-AzureADKerberosServer -Domain $domain -UserPrincipalName $upn
```

Après exécution, vérifiez qu'un objet de type **AzureADKerberos** existe côté domaine.

### Étape 3 : créer la GPO dédiée

N'ajoutez pas WHfB dans une énorme GPO de sécurité générique.

Créez une GPO dédiée, par exemple :

`WHFB-CloudKerberosTrust-Pilot`

Pourquoi ?

Parce que vous voulez pouvoir :

- la lier ou la délier vite
- la filtrer proprement
- lire `gpresult` sans pollution

### Étape 4 : vérifier les TPM

Un pilote WHfB sans contrôle TPM est une perte de temps.

Sur chaque poste pilote :

```powershell title="Contrôler le TPM"
Get-Tpm
```

Lecture minimale :

- `TpmPresent = True`
- `TpmReady = True`
- `ManagedAuthLevel` cohérent avec votre configuration

Si le TPM n'est pas prêt, n'accusez pas la GPO.

Le problème est plus bas dans la pile.

### Étape 5 : appliquer la policy

Sur le poste cible :

```powershell title="Forcer l'application puis redémarrer"
gpupdate /force
shutdown /r /t 0
```

L'oubli du redémarrage fausse beaucoup de validations.

Vous croyez tester l'état final.

En réalité, vous testez un état intermédiaire.

### Étape 6 : vérifier le join et le prérequis cloud trust

Avant même de regarder HelloForBusiness, vérifiez le statut d'inscription :

```powershell title="Contrôler Hybrid Join et MDM"
dsregcmd /status
```

Cherchez au minimum :

- `AzureAdJoined` ou `DomainJoined` selon le scénario attendu
- `DeviceAuthStatus`
- `SSO State`
- `MdmUrl` si vous mélangez avec Intune

Sur un poste Hybrid Join, l'absence de prérequis côté Entra Kerberos se voit souvent ici avant même que l'utilisateur ne vous remonte "le PIN ne marche pas".

### Étape 7 : laisser l'utilisateur s'enrôler

L'enrôlement WHfB démarre après ouverture de session si les prérequis sont bons.

Expliquez à l'utilisateur pilote ce qu'il va voir :

- proposition de configurer un PIN
- éventuellement proposition biométrique
- demande de MFA selon le flux

Si vous ne préparez pas l'utilisateur, il annule l'assistant.

Puis vous diagnostiquez un "échec technique" qui est en fait un refus humain.

### Vérification AD côté parc

Pour repérer les objets qui portent déjà des données d'enrôlement :

```powershell title="Résultat attendu : liste postes WHfB enrollés"
Get-ADComputer -Filter * -Properties msDS-KeyCredentialLink |
    Where-Object { $_.'msDS-KeyCredentialLink' } |
    Select-Object Name, DistinguishedName
```

Ce script est surtout utile pour :

- inventorier un existant Key Trust
- repérer des objets déjà passés par un enrôlement
- préparer une migration ou un nettoyage

Pour Cloud Kerberos Trust, ne tirez pas de conclusion abusive :

l'absence de `msDS-KeyCredentialLink` n'est pas, à elle seule, une preuve d'échec du modèle.

### Ce que vous devez valider côté poste

Validation minimale :

| Contrôle | Outil | Résultat attendu |
|---|---|---|
| TPM prêt | `Get-Tpm` | `TpmPresent=True`, `TpmReady=True` |
| GPO appliquée | `gpresult /r /scope computer` | GPO WHfB visible |
| Join correct | `dsregcmd /status` | Hybrid Join cohérent |
| Enrôlement | Event Viewer | Event ID `358` |

### Séquence de déploiement recommandée

Ne ciblez pas 500 postes d'un coup.

Découpez en anneaux :

1. IT / admins non privilégiés
2. support de proximité
3. population bureautique standard
4. portables VIP
5. comptes sensibles avec exigences spécifiques

### Cas à sortir du premier pilote

Excluez du premier anneau :

- comptes admin Tier 0
- VDI/RDS avec exigences RDP particulières
- postes sans biométrie mais avec contraintes métiers de smart card
- utilisateurs qui changent de poste en permanence sans modèle clair

### Point d'attention RDP / VDI

Cloud Kerberos Trust ne couvre pas tous les scénarios RDP/VDI fournis comme credential.

Si votre besoin principal est le RDP avec credential fourni au serveur distant, vérifiez le design cible avant généralisation.

Ce n'est pas un bug.

C'est une limite de scénario à traiter en amont.

### Après le pilote

Quand le pilote est stable :

- conservez la GPO dédiée
- élargissez le groupe de sécurité
- gardez les comptes à contraintes fortes dans une phase séparée

!!! quote "En résumé"
    - En Cloud Kerberos Trust, on prépare d'abord **Microsoft Entra Kerberos**, puis la **GPO**, puis le **poste**.
    - `Set-AzureADKerberosServer`, `Get-Tpm`, `gpupdate`, `dsregcmd` et l'event log forment la boîte à outils minimum.
    - `msDS-KeyCredentialLink` est utile pour l'inventaire, mais ne doit pas être surinterprété dans un modèle Cloud Kerberos Trust.
    - Un pilote propre repose sur une **GPO dédiée**, un **groupe pilote** et un **support briefé**.
    - Le plus grand accélérateur de succès n'est pas technique : c'est la **discipline de séquencement**.

---

## :material-magnify: Diagnostic et event IDs

### Commencez par le journal correct

Pour WHfB, ne cherchez pas au hasard dans le journal Système.

Allez directement dans :

`Applications and Services Logs > Microsoft > Windows > HelloForBusiness > Operational`

La source visible est généralement :

`Microsoft-Windows-HelloForBusiness`

### Table des event IDs utiles

| Event ID | Signification | Action |
|---|---|---|
| 358 | Enrollment réussi | OK |
| 360 | Enrollment échoué | Vérifier TPM + GPO |
| 362 | PIN reset réussi | OK |
| 100 | WHfB désactivé par policy | Vérifier GPO/MDM conflit |

### Comment lire ces événements

**358**

Bonne nouvelle.

Le poste a réussi l'enrôlement.

Votre travail n'est pas fini pour autant.

Vérifiez encore :

- que l'utilisateur arrive bien à se reconnecter
- que l'accès aux ressources on-prem fonctionne
- que le support sait distinguer oubli de PIN et problème d'enrôlement

**360**

Ne concluez pas trop vite à un problème "Hello".

Commencez par les fondamentaux :

- TPM absent ou non prêt
- GPO non appliquée
- mauvais modèle de trust
- poste pas correctement Hybrid Join

**362**

C'est un signal utile en exploitation.

Il confirme qu'un scénario de récupération de PIN a fonctionné.

Si vous voyez beaucoup de `362` après déploiement, cela peut aussi indiquer :

- une mauvaise préparation utilisateur
- une politique PIN trop exigeante
- une mauvaise compréhension du geste attendu

**100**

Celui-ci oriente directement vers la gouvernance de policy.

Posez-vous tout de suite la question :

qui pilote réellement WHfB sur ce poste ?

### Commandes utiles de diagnostic

```powershell title="Lire rapidement les derniers événements WHfB"
Get-WinEvent -LogName "Microsoft-Windows-HelloForBusiness/Operational" -MaxEvents 20 |
    Select-Object TimeCreated, Id, LevelDisplayName, Message
```

```powershell title="Contrôler la présence de la GPO sur le poste"
gpresult /r /scope computer
```

```powershell title="Contrôler l'état du TPM"
Get-Tpm
```

```powershell title="Contrôler l'inscription Entra / Hybrid Join"
dsregcmd /status
```

### Ordre de diagnostic recommandé

Quand un poste échoue :

1. `Get-Tpm`
2. `gpresult`
3. `dsregcmd /status`
4. `Get-WinEvent`

Cet ordre évite de passer vingt minutes dans l'Event Viewer pour découvrir ensuite que le TPM est désactivé dans l'UEFI.

### Symptômes classiques et lecture rapide

| Symptôme | Cause probable | Première action |
|---|---|---|
| L'utilisateur ne voit jamais la proposition WHfB | Policy absente ou désactivée | Vérifier `gpresult` |
| L'assistant démarre puis échoue | TPM / trust / prereq Entra | Vérifier `Get-Tpm` puis `dsregcmd` |
| Le PIN est créé mais l'accès on-prem échoue | Modèle de trust ou prérequis Kerberos incomplet | Vérifier objet Entra Kerberos et DC |
| Le poste reçoit des règles contradictoires | Double pilotage GPO / Intune | Retirer le chevauchement |

### N'oubliez pas la partie humaine

Un ticket "WHfB ne marche pas" couvre souvent des réalités différentes :

- l'utilisateur a annulé l'enrôlement
- il a oublié son PIN
- la webcam n'est pas compatible anti-spoofing
- il est hors ligne de vue DC lors du premier usage en Hybrid Join

Votre runbook support doit distinguer ces cas.

Sinon tout finit en "bug WHfB".

### `!!! check "Point de contrôle"`

!!! check "Point de contrôle"
    - Le poste pilote a-t-il un **TPM prêt** et une **GPO WHfB visible** dans `gpresult` ?
    - Le **modèle de confiance** choisi est-il documenté noir sur blanc, sans policy concurrente activée ?
    - Les événements `358`, `360`, `362` et `100` sont-ils intégrés dans votre **runbook support** ?

### Décision d'exploitation après incident

Si vous devez interrompre un pilote :

- retirez le groupe de sécurité cible
- laissez la GPO intacte pour analyse
- documentez le motif exact

Ne "réparez" pas en empilant plus de paramètres.

WHfB se corrige rarement par ajout de complexité.

Il se corrige en retirant l'ambiguïté :

- modèle clair
- prérequis clairs
- policy claire

!!! quote "En résumé"
    - Le journal à regarder est **Microsoft-Windows-HelloForBusiness/Operational**.
    - Les événements `358`, `360`, `362` et `100` couvrent déjà l'essentiel du pilotage terrain.
    - L'ordre de diagnostic correct est : **TPM**, **GPO**, **join**, **events**.
    - Un grand nombre d'incidents WHfB sont en réalité des problèmes de **séquencement** ou de **double gouvernance**.
    - Votre objectif n'est pas seulement d'enrôler ; votre objectif est de **rendre le diagnostic banal** pour le support.
